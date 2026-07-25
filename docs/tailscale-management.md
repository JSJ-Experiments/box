# On-device Tailscale management

The custom Mihomo build provides per-device path ordering, concurrent path
probes, and loopback service forwarding through a Tailscale outbound. The Box
alpha module also includes:

- the standalone path UI;
- a public-key-only Dropbear SSH server;
- an optional rooted `adbd` TCP mode for ADB and scrcpy.

All management services are opt-in.

## 1. Pick one management outbound

Add `service-forwards` to **one** Tailscale proxy entry. Do not add it to every
exit-node proxy unless you intentionally want every tsnet node kept online.

```yaml
- name: phone-management-ts
  type: tailscale
  hostname: phone-management
  state-dir: tailscale/phone-management
  # Existing auth-key/control-url/etc. settings stay unchanged.
  connection-order: "https://example.net/connection-order.yaml"
  service-forwards:
    - name: mihomo-controller
      listen: 9090
      target: 127.0.0.1:9090
    - name: ssh
      listen: 8022
      target: 127.0.0.1:8022
```

Bind Mihomo's controller to loopback and set a non-empty secret:

```yaml
external-controller: 127.0.0.1:9090
secret: "replace-with-a-long-random-secret"
```

The service forward is reachable at the Tailscale IP of
`phone-management-ts`, subject to the tailnet ACL/grants. It does not open an
Android physical-network listener.

## 2. Path UI

Locally open:

```text
http://127.0.0.1:9090/tailscale-ui/
```

From another tailnet device open:

```text
http://MANAGEMENT_TAILSCALE_IP:9090/tailscale-ui/
```

Enter the Mihomo controller secret when prompted. The UI:

- applies orders to individual target devices, not exit-node definitions;
- writes local overrides to each outbound's
  `state-dir/connection-order-local.yaml`;
- treats an empty local order as explicit `AUTO`;
- leaves the downloaded `connection-order.yaml` cache separate;
- probes requested paths concurrently only while the page is visible.

Update only the UI, without restarting Mihomo:

```sh
su -c /data/adb/box/scripts/box.tool uptailscaleui
```

## 3. SSH

Copy an OpenSSH public key to the device and install it:

```sh
adb push ~/.ssh/id_ed25519.pub /data/local/tmp/id_ed25519.pub
adb shell su -c '/data/adb/box/scripts/box.tool sshkey /data/local/tmp/id_ed25519.pub'
```

Then set this in `/data/adb/box/settings.ini`:

```sh
ssh_enable="true"
```

Restart the management services or Box:

```sh
su -c '/data/adb/box/scripts/box.tool management restart'
```

Connect from another tailnet device:

```sh
ssh -p 8022 root@MANAGEMENT_TAILSCALE_IP
```

Password authentication is disabled at compile time. Dropbear binds only to
`127.0.0.1`; tsnet exposes it to the tailnet. Local SSH port forwarding remains
available, so a loopback-only Android service can also be reached with
`ssh -L`.

## 4. ADB and scrcpy

SSH cannot replace the ADB protocol, but it can securely carry a loopback ADB
connection. Enable loopback adbd only when needed:

```sh
adb_loopback_enable="true"
```

Restart the management services:

```sh
su -c '/data/adb/box/scripts/box.tool management restart'
```

On the client, keep this SSH tunnel open in one terminal:

```sh
ssh -N -o ExitOnForwardFailure=yes \
  -L 5555:127.0.0.1:5555 \
  -p 8022 root@MANAGEMENT_TAILSCALE_IP
```

Then use the forwarded ADB endpoint in another terminal:

```sh
adb connect 127.0.0.1:5555
scrcpy --serial 127.0.0.1:5555
```

Android's normal ADB RSA authorization still applies. Box requests
`service.adb.listen_addrs=tcp:localhost:5555`, which current AOSP adbd binds to
IPv4 loopback. Android Wireless Debugging independently opens its dynamic TLS
listener, so both listeners coexist afterward. Enabling the stable listener
requires one adbd restart, causing existing ADB sessions to disconnect briefly
and Android to choose a new Wireless Debugging port.

The stable loopback listener does not depend on Wi-Fi and remains usable
through SSH over any network that keeps the management tsnet node reachable.

Box also inserts firewall rules that reject port 5555 on non-loopback Android
interfaces as a fallback for older or vendor-modified adbd builds. Disabling
the setting and restarting restores the previous adbd properties. The former
`adb_tailnet_enable` and `adb_tailnet_port` setting names remain supported.

The SSH tunnel adds a small amount of latency and CPU use because traffic is
encrypted by both SSH and Tailscale, but scrcpy still works normally. It avoids
publishing an ADB port to every device allowed to reach the management tsnet
node. A direct `service-forwards` entry for port 5555 remains possible when
maximum throughput matters, but is not the recommended default.

## Battery impact

- A saved order by itself adds no periodic probing.
- The UI's live mode sends concurrent probes every five seconds only while the
  tab is visible; this can keep the radio awake during testing.
- A management outbound with service forwarding keeps one tsnet client online.
  Configure only one to avoid keeping every provider entry alive.
- Idle Dropbear is normally a small additional cost.
- Continuously running TCP adbd costs more and expands the attack surface, so
  it is disabled by default and is best enabled only for scrcpy sessions.

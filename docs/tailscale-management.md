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
    - name: adb
      listen: 5555
      target: 127.0.0.1:5555
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

ADB is separate from SSH. Enable it only when needed:

```sh
adb_tailnet_enable="true"
```

Then restart management services and connect:

```sh
adb connect MANAGEMENT_TAILSCALE_IP:5555
scrcpy --serial MANAGEMENT_TAILSCALE_IP:5555
```

Android's normal ADB RSA authorization still applies. Box requests
`service.adb.listen_addrs=tcp:127.0.0.1:5555` and also inserts firewall rules
that reject the port on non-loopback Android interfaces as a fallback for older
or vendor-modified adbd builds. Disabling the setting and restarting restores
the previous adbd properties.

## Battery impact

- A saved order by itself adds no periodic probing.
- The UI's live mode sends concurrent probes every five seconds only while the
  tab is visible; this can keep the radio awake during testing.
- A management outbound with service forwarding keeps one tsnet client online.
  Configure only one to avoid keeping every provider entry alive.
- Idle Dropbear is normally a small additional cost.
- Continuously running TCP adbd costs more and expands the attack surface, so
  it is disabled by default and is best enabled only for scrcpy sessions.

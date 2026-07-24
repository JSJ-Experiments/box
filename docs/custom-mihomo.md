# Custom Mihomo updates

This Box build uses the moving JSJ Mihomo alpha release by default. Updating the
core does not require reflashing the module.

```sh
/data/adb/box/scripts/box.tool upkernel mihomo
```

The updater downloads the binary and `checksums.txt`, verifies SHA-256, checks
that the new core can parse the active configuration, keeps the old binary in
`/data/adb/box/bin/backup/mihomo.bak`, and rolls back if the service cannot
restart.

Manual rollback:

```sh
/data/adb/box/scripts/box.tool rollbackkernel mihomo
```

The source is configured in `/data/adb/box/settings.ini`:

```sh
mihomo_custom_repo="JSJ-Experiments/mihomo"
mihomo_custom_release_tag="Prerelease-Alpha"
mihomo_custom_base_url=""
```

Set `mihomo_custom_repo=""` to use Box's original official Mihomo updater.
Alternatively, set `mihomo_custom_base_url` to a directory URL containing the
platform binary and `checksums.txt`.

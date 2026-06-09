# Minecraft on rootful Podman Quadlet

This configuration runs a supplied Minecraft server JAR directly on
`gcr.io/distroless/java25-debian13:nonroot`. Podman and systemd are rootful,
but the server process runs as uid/gid `65532`.

Restic runs directly on the host. The [official release Dockerfile][restic-dockerfile]
uses Alpine, so its container image is intentionally not used here.

## Layout

- `/var/lib/minecraft/data`: server JAR, configuration, world, and generated files
- `/var/lib/minecraft/backup`: local restic repository

## Install

Podman 5.x with Quadlet, cgroup v2, and `restic` in the system search path are
expected.

```sh
sudo install -d -m 0750 /etc/containers/systemd
sudo install -d -m 0750 /var/lib/minecraft/{data,backup}
sudo install -m 0644 quadlet/minecraft.container /etc/containers/systemd/minecraft.container
sudo install -m 0644 \
  systemd/minecraft-backup.service \
  systemd/minecraft-backup.timer \
  /etc/systemd/system/
sudo install -m 0644 config/server.properties /var/lib/minecraft/data/server.properties
sudo install -m 0644 config/eula.txt.example /var/lib/minecraft/data/eula.txt
sudo install -m 0600 config/rcon.yaml.example /var/lib/minecraft/data/rcon.yaml
sudo install -m 0644 /path/to/server.jar /var/lib/minecraft/data/server.jar
sudo chown -R 65532:65532 /var/lib/minecraft/data
```

Set one strong RCON password in both:

- `/var/lib/minecraft/data/server.properties`: `rcon.password=...`
- `/var/lib/minecraft/data/rcon.yaml`: `password: ...`

Review the Minecraft EULA, then change `/var/lib/minecraft/data/eula.txt` to
`eula=true`.

Initialize the local restic repository without a password:

```sh
sudo podman pull docker.io/itzg/rcon-cli:latest
sudo restic \
  --repo=/var/lib/minecraft/backup \
  --insecure-no-password \
  init
```

Load and start the units:

```sh
sudo systemctl daemon-reload
sudo systemctl start minecraft.service
sudo systemctl enable --now minecraft-backup.timer
sudo systemctl enable --now podman-auto-update.timer
```

Quadlet applies `[Install]` during generation. The generated
`minecraft.service` is therefore started at boot without running
`systemctl enable` against the generated unit.

## Networking

- Minecraft TCP: `0.0.0.0:25565` and `[::]:25565`
- RCON TCP: `127.0.0.1:25575` only

`server-ip` stays empty so the server listens on all interfaces inside the
container. The public bind addresses are controlled by Quadlet.

Ensure IPv6 is enabled on the host and allowed through the firewall. On hosts
where an IPv6 wildcard socket also claims IPv4, set
`net.ipv6.bindv6only=1`; otherwise the two explicit Podman publishes can
conflict.

## Backup

The daily timer runs at 04:00 local time with up to 15 minutes of random delay.
The timer activates `minecraft-backup.service` directly. Its `Type=oneshot`
commands run serially:

1. `save-off`
2. `save-all flush`
3. host restic backup with `--insecure-no-password`
4. `forget --keep-last=20 --prune`
5. `save-on`, including after failures

There is no target unit in this implementation, but the workflow can be split
across multiple Quadlet `.container` units. A timer can activate a target whose
`Wants=` pulls them in. `Requires=` and `After=` define the chain, `PartOf=`
ties their lifecycle to the target, and `StopWhenUnneeded=yes` makes the target
reusable by the next timer activation. Cleanup still needs its own ordering so
that `save-on` runs after both successful and failed jobs.

The single oneshot service is retained here because it expresses the same
serial transaction and failure cleanup without additional generated units.

Run and inspect a backup manually:

```sh
sudo systemctl start minecraft-backup.service
sudo journalctl -u minecraft-backup.service
```

## Validation

After installation, inspect the generated service before first start:

```sh
sudo /usr/lib/systemd/system-generators/podman-system-generator --dryrun
sudo systemd-analyze verify minecraft-backup.service minecraft-backup.timer
sudo podman auto-update --dry-run
```

[restic-dockerfile]: https://github.com/restic/restic/blob/master/docker/Dockerfile.release

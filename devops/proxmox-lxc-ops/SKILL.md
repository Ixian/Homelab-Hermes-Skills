---
name: proxmox-lxc-ops
description: Use when operating LXC containers on the HSB Proxmox host (proxmox-hsb, 192.168.68.92). Covers pct command patterns, the /opt/stacks Docker-in-LXC layout, and the two production LXCs (102 hsb-traefik, 103 hsb-cameras with GPU passthrough).
version: 1.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [homelab, proxmox, lxc, hsb, docker]
    related_skills: [homelab-ops, frigate-tune]
---

# Proxmox LXC Operations (HSB)

## Overview

HSB runs a single Proxmox VE host (`proxmox-hsb`, 192.168.68.92) that holds the HSB infrastructure as LXC containers. Two are production-load-bearing:

| VMID | Name | Role |
|---|---|---|
| 102 | `hsb-traefik` | Edge reverse proxy for HSB-local services |
| 103 | `hsb-cameras` | Frigate NVR + Scrypted, with Intel GPU passthrough for QuickSync transcoding |

Docker compose stacks inside each LXC live under `/opt/stacks/<stack>/`. Proxmox itself does not run Docker; that runs *inside* the LXCs. This nesting (Docker-in-LXC) is the single most common source of subtle bugs.

## When to Use

- Starting/stopping/restarting LXCs
- Updating Docker stacks inside the LXCs
- Inspecting GPU passthrough on 103 (Frigate/Scrypted hw accel)
- Snapshotting before risky changes
- Adding a new LXC

## Don't Use For

- HA config — that's on `ha2` (the Home Assistant host), not Proxmox. See `homelab-ops`.
- Frigate camera/zone config — that's *inside* LXC 103. See `frigate-tune`.
- TrueNAS at HSB (`truenas-hsb`) — separate physical box.

## Connection

```bash
ssh proxmox-hsb           # interactive
# Run a command directly inside an LXC from the Proxmox host:
ssh proxmox-hsb "pct exec 103 -- <cmd>"
# Drop into a shell inside the LXC:
ssh proxmox-hsb "pct enter 103"
```

`pct exec` is the workhorse — no need to `ssh hsb-cameras` separately for most ops.

## Common Workflows

### List LXCs and status
```bash
ssh proxmox-hsb "pct list"
```

### Start / stop / reboot
```bash
ssh proxmox-hsb "pct start 103"
ssh proxmox-hsb "pct stop 103"
ssh proxmox-hsb "pct reboot 103"   # graceful, sends SIGTERM first
ssh proxmox-hsb "pct shutdown 103 --forceStop 1 --timeout 60"
```

### Snapshot before risky change
```bash
ssh proxmox-hsb "pct snapshot 103 pre-frigate-017 --description 'Pre Frigate 0.17 upgrade'"
ssh proxmox-hsb "pct listsnapshot 103"
ssh proxmox-hsb "pct rollback 103 pre-frigate-017"   # rollback if needed
```

ZFS-backed LXCs snapshot near-instantly. Use them aggressively before config changes — it's the cheapest insurance you have.

### Update Docker stacks inside the LXC
```bash
ssh proxmox-hsb "pct exec 103 -- bash -c 'cd /opt/stacks/<stack> && docker compose pull && docker compose up -d'"
```

`docker compose` (v2, no hyphen) per project conventions.

### Tail logs of a container inside an LXC
```bash
ssh proxmox-hsb "pct exec 103 -- docker logs <container> --tail 100 -f"
```

### Inspect resource use
```bash
ssh proxmox-hsb "pct exec 103 -- top -bn1 | head -20"
ssh proxmox-hsb "pct config 103"   # CPU cores, memory, mounts, devices
```

### Edit a compose file
Per CLAUDE.md workflow, do this in the local clone, push, then deploy. There is no `Homelab-HSB/proxmox/...` mirror today; if the user starts tracking these stacks in git, mirror the same edit-local → push → cat | ssh pattern. Until then, edit on the LXC directly and tell the user it's not yet in git.

## LXC 103 Specifics (Cameras + GPU)

LXC 103 has Intel UHD Graphics (host: Core i5-12450H Alder Lake) passed through for Frigate hardware-accelerated decoding and Scrypted's HomeKit Secure Video re-encode. Verify GPU access:

```bash
ssh proxmox-hsb "pct exec 103 -- ls -la /dev/dri/"
# Expect: card0, renderD128 with crw-rw---- owned by root:video
ssh proxmox-hsb "pct exec 103 -- docker exec frigate ls -la /dev/dri/"
# Same inside the Frigate container — confirms compose passes through the device
```

If `/dev/dri` is missing inside the LXC, check `pct config 103` for `dev0:` lines mounting the GPU. Missing them means the LXC was reconfigured and GPU passthrough was lost. Add via the Proxmox web UI's "Resources" tab on the LXC (Add → Device Passthrough).

**Privileged vs unprivileged:** GPU passthrough to an unprivileged LXC requires the right cgroup and `lxc.idmap` UID/GID mappings. Don't switch 103 between privileged and unprivileged casually — it breaks the GPU mapping.

## LXC 102 Specifics (HSB Traefik)

This is HSB's local edge — handles cross-site routing inside HSB without going through Austin. Standard Traefik + compose pattern. See `traefik-route-add` for label conventions (same `chain-authelia@file` pattern if/when HSB gets Authelia; currently it may be a simpler chain).

## Adding a New LXC

Generic recipe (run from `proxmox-hsb`):

```bash
# Find next VMID (just pick the next free integer)
pveam list local                                # list available templates
# Create from a Debian 12 template (commonest)
pct create 104 local:vztmpl/debian-12-standard_12.7-1_amd64.tar.zst \
  --hostname new-service --cores 2 --memory 2048 --swap 1024 \
  --net0 name=eth0,bridge=vmbr0,ip=dhcp \
  --storage local-lvm --rootfs local-lvm:8 --unprivileged 1 --features nesting=1
pct start 104
pct exec 104 -- apt update && apt -y install docker.io docker-compose-plugin
```

`nesting=1` is required for Docker inside the LXC. Don't omit it.

## Pitfalls

### Docker-in-LXC nesting flag
If Docker fails to start inside a new LXC with `cgroup` errors, the LXC was created without `--features nesting=1`. Fix in `/etc/pve/lxc/<vmid>.conf` (`features: nesting=1`) and reboot the LXC.

### `pct exec` inherits root's PATH
Inside `pct exec`, the PATH is minimal. Use absolute paths or `bash -lc` to source `/etc/profile`. Same general failure mode as the Hermes terminal-tool PATH issue documented in `infrastructure.md`.

### Snapshots aren't backups
ZFS snapshots live on the same pool. If the pool dies, the snapshots die. For real backups use Proxmox Backup Server (PBS) or replication — neither is currently set up at HSB (`truenas-hsb` ZFS replication target is inactive). Treat snapshots as undo-buttons, not as DR.

### Memory ballooning surprises
LXCs share the host kernel, so OOM in an LXC can be triggered by host pressure even when the LXC's own memory cap isn't hit. Check `dmesg` on the host (`ssh proxmox-hsb "dmesg -T | tail -50"`), not just inside the LXC.

### Docker network conflicts with LXC bridge
The default Docker bridge `172.17.0.0/16` can collide with whatever the LXC's `vmbr0` is bridged to. If outbound from a container is flaky, check `docker network inspect bridge` and the host's routing.

### Restarting the LXC ≠ restarting Docker
`pct reboot 103` restarts the whole LXC and bounces Frigate too — long downtime. To just bounce a service, `pct exec 103 -- docker restart frigate`. Don't reboot the LXC for a container hiccup.

### MQTT credentials drift
Frigate's MQTT password in `/addon_configs/ccab4aaf_frigate/config.yml` (on ha2) and the matching user in HA's Mosquitto add-on are tracked separately. If you rotate Mosquitto's password, also update Frigate's — they won't realign on their own.

## Quick Reference

```
ssh proxmox-hsb                       # interactive
pct list                              # inventory
pct exec <id> -- <cmd>                # one-shot inside LXC
pct enter <id>                        # shell inside LXC
pct snapshot <id> <name>              # cheap insurance
pct rollback <id> <name>              # undo
pct config <id>                       # CPU/RAM/mounts/devices
/opt/stacks/<stack>/                  # Docker compose dir inside LXC
docker compose pull && docker compose up -d
```

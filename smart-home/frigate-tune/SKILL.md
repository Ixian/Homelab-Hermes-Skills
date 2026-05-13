---
name: frigate-tune
description: Use when editing Frigate NVR config — adding cameras, tuning zones, adjusting object detectors/filters, MQTT auth, or recording retention. Config lives on ha2 at /addon_configs/ccab4aaf_frigate/config.yml; runtime is inside LXC 103 on proxmox-hsb with Intel QuickSync passthrough.
version: 1.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [homelab, frigate, nvr, cameras, hsb, home-assistant]
    related_skills: [homelab-ops, proxmox-lxc-ops]
---

# Frigate NVR Tuning

## Overview

Frigate runs as a Home Assistant add-on on `ha2` (the HSB Home Assistant host). Hardware: Intel Core i5-12450H with UHD Graphics for OpenVINO object detection and QuickSync transcoding. Currently runs Frigate 0.17 (Feb 2026) with a custom-trained Frigate+ YOLOv9-t model on 6 cameras.

There is also a Frigate instance running inside LXC 103 (`hsb-cameras`) on `proxmox-hsb` per the HSB site map. **Confirm with the user which Frigate is in scope** before editing — the ha2 add-on is the primary; the LXC may be for development or a separate camera group.

This skill assumes the **ha2 add-on** Frigate as default.

## When to Use

- Add a new camera (RTSP URL, detect/record settings, zones)
- Adjust object filters (min_area, min_score, threshold)
- Change recording retention or motion mask
- Edit MQTT broker config
- Tune detector (OpenVINO/edgetpu/tensorrt) or swap the model
- Set up Frigate+ submissions

## Don't Use For

- HA-side notification automations triggered by Frigate events — those are in `Homelab-HSB/automations.yaml`
- Camera POE switch / network plumbing — that's OPNsense + the cam VLAN
- Adding go2rtc streams unrelated to Frigate — separate file

## File Locations

| File | Path on ha2 | Purpose |
|---|---|---|
| Main config | `/addon_configs/ccab4aaf_frigate/config.yml` | Cameras, zones, detectors, recording |
| Backup config | `/addon_configs/ccab4aaf_frigate/config.yml.bak-0.16` | Pre-0.17-upgrade snapshot |
| go2rtc | `/addon_configs/ccab4aaf_frigate/go2rtc_homekit.yml` | HomeKit Secure Video passthrough |
| Frigate DB | `/addon_configs/ccab4aaf_frigate/frigate.db` | Events DB (SQLite). Don't edit live. |
| Backup DB | `/addon_configs/ccab4aaf_frigate/backup.db` | Periodic Frigate-managed snapshot |

The add-on slug `ccab4aaf_frigate` is the immutable internal slug — don't try to rename it.

## Editing Workflow

1. Read the live file first per the source-of-truth rule:
   ```bash
   ssh ha2 "cat /addon_configs/ccab4aaf_frigate/config.yml"
   ```
2. Edit in `Homelab-HSB/frigate/config.yml` locally (if tracked) or paste a diff for the user to apply.
3. Validate with the Frigate config check before restart:
   ```bash
   ssh ha2 "ha addons restart ccab4aaf_frigate"
   ```
   Frigate will log a fatal at startup if the YAML is invalid. Watch the log:
   ```bash
   ssh ha2 "ha addons logs ccab4aaf_frigate | tail -100"
   ```
4. Confirm cameras come back online in the Frigate UI (`/frigate` in HA, or `frigate.maximumgamer.com` if exposed).

## Camera Block — Canonical Pattern

```yaml
cameras:
  driveway:
    ffmpeg:
      inputs:
        - path: rtsp://<user>:<pass>@<cam-ip>:554/Streaming/Channels/101
          roles: [record, detect]
        - path: rtsp://<user>:<pass>@<cam-ip>:554/Streaming/Channels/102
          roles: [audio]      # sub-stream for low-bandwidth uses
    detect:
      width: 1920
      height: 1080
      fps: 5                  # cap detection FPS, decode rest for recording
    record:
      enabled: true
      retain:
        days: 7
        mode: motion
      alerts:
        retain: { days: 30, mode: motion }
      detections:
        retain: { days: 14, mode: motion }
    snapshots:
      enabled: true
      retain: { default: 7 }
    motion:
      mask:
        - "0,0,0,200,300,200,300,0"   # street area, ignored
    zones:
      porch:
        coordinates: "..."
        objects: [person, package]
```

Common knobs:

- **`detect.fps`** — keep at 5 unless tracking fast-moving objects. CPU saver.
- **`record.retain.mode`** — `motion` (default), `active_objects` (only when tracking), `all` (24/7, big disk). Almost always `motion`.
- **`alerts` vs `detections`** — 0.17 split: alerts = high-confidence events you'd notify on; detections = everything tracked. Retain alerts longer than detections.
- **`zones`** — polygon coords from the Frigate UI's zone editor. Don't hand-edit unless you know the image dimensions.

## Detector + Model (current setup)

```yaml
detectors:
  ov:
    type: openvino
    device: CPU                   # uses Intel UHD via OpenVINO

model:
  path: plus://4fbed9863f895a3c2d4e0c73bce2a99c
  # Frigate+ YOLOv9-t fine-tuned model. 640x640. Model ID:
  # 80b729a0ad4596f2f6f08a8bc776129a
```

Don't swap models casually — the Frigate+ one is trained on this site's footage. To try a stock model temporarily, comment out `model:` (Frigate falls back to the built-in MobileDet).

## Object Filters (the high-leverage knob)

Per-object filters control false-positive rate. Pattern is `min_area`, `min_score`, `threshold` — see `objects.filters` block. Tighter values → fewer false alarms but more missed detections. Current tuning targets long-range yard wildlife so `min_area` is intentionally low for dogs.

To debug a noisy detection: temporarily raise `min_score` (e.g., person from 0.55 to 0.70) and watch alert volume. Don't go above 0.85 — you'll start missing real events.

## MQTT

```yaml
mqtt:
  enabled: true
  host: core-mosquitto             # HA Mosquitto add-on internal hostname
  user: frigate_mqtt
  password: <secret>
```

When rotating MQTT creds: update both Mosquitto's user (via HA UI) AND this file, restart Frigate. Drift is silent — Frigate keeps running but events stop appearing in HA.

## Hardware Acceleration Health Check

```bash
# Confirm OpenVINO is using the iGPU and not falling back to CPU-only
ssh ha2 "ha addons logs ccab4aaf_frigate | grep -iE 'openvino|gpu|cpu' | tail -20"

# Check decode load on iGPU (if intel-gpu-tools available)
ssh ha2 "intel_gpu_top -s 1"
```

If `Render/3D` engine is consistently idle, hardware decode isn't being used and you're burning CPU for nothing — re-check the `device` field and the add-on's hardware passthrough.

## Recording Disk Pressure

Recordings land on the ha2 host's storage. Check:

```bash
ssh ha2 "du -sh /share/frigate/recordings/ 2>/dev/null; df -h /share"
```

If disk fills, Frigate stops recording silently. Either tighten retention (per-camera), prune manually, or expand storage. HA's Frigate add-on doesn't auto-prune below the retention window — it manages the window, not free space.

## Pitfalls

### Add-on restart, not reload
There's no hot-reload. Every config change needs an add-on restart, which interrupts recording for 10–30s. Batch edits when possible.

### YAML indentation in `objects.filters`
This is the #1 silent break: a misindented filter dict gets attached to the wrong object and you wonder why person detection went haywire. Use `yamllint` locally before pushing.

### Zone coordinates change with resolution
If you change `detect.width/height`, all your hand-drawn zones break (they're pixel coords on the detect image). Re-draw via the UI after a resolution change.

### Frigate+ model ID rotation
The Frigate+ path has TWO IDs in current config (`plus://4fb...` and a comment about `80b729...`). The active one is the `plus://` URL. Don't confuse the two when uploading new training data.

### MQTT broken silently
Frigate doesn't crash if MQTT auth fails — it logs `MQTT connection refused` and keeps running. HA stops getting events. Always grep logs after an MQTT change.

### Backup files masquerading as configs
`config.yml.bak-0.16` and `backup.db` are old. Editing them does nothing. Always edit `config.yml` and `frigate.db` (and don't edit `frigate.db` live anyway).

### The HSB LXC 103 Frigate
If a second Frigate exists inside LXC 103 (`hsb-cameras`) per the site map, its config is at `/opt/stacks/frigate/config/config.yml` *inside the LXC* — accessed via `pct exec 103 -- cat ...` from `proxmox-hsb`. Don't edit one thinking it's the other. Confirm with the user which is the active NVR.

## Quick Reference

```
Config:          /addon_configs/ccab4aaf_frigate/config.yml   (on ha2)
Restart:         ssh ha2 "ha addons restart ccab4aaf_frigate"
Logs:            ssh ha2 "ha addons logs ccab4aaf_frigate"
Detector:        OpenVINO on iGPU
Model:           Frigate+ YOLOv9-t (custom, plus://4fb...)
Cameras:         6
HA add-on slug:  ccab4aaf_frigate
```

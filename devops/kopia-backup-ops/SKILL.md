---
name: kopia-backup-ops
description: Use when verifying, restoring, or modifying Kopia backups for the Austin stack. Covers the kopia server at backups.maximumgamer.com, the read-only source-mount pattern, DB-dump-then-snapshot strategy, and B2 destination — plus how to verify backups are actually working (the part everyone skips).
version: 1.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [homelab, kopia, backup, austin, b2, disaster-recovery]
    related_skills: [homelab-ops, traefik-route-add]
---

# Kopia Backup Operations (Austin → Backblaze B2)

## Overview

Austin's offsite backup is Kopia → Backblaze B2. The `kopia` container at `backups.maximumgamer.com` (behind Authelia) takes encrypted, deduplicated snapshots of a curated set of source directories mounted **read-only**, and pushes them to B2. Databases (Postgres, MariaDB, SQLite for Hermes) are dumped to disk first by sidecar containers, then Kopia snapshots the dumps — never the live DB files.

The chain looks like:

```
[live MariaDB/Postgres/SQLite]
        │ (dump on schedule)
        ▼
$DOCKERDIR/db-backups/                   ← intermediate JSON/SQL dumps
        │ (mounted :ro into kopia)
        ▼
kopia container ──(snapshot)──▶ B2 bucket (encrypted, deduped)
```

Plus: live files Kopia can snapshot safely (paperless originals, vaultwarden attachments, $SECRETSDIR, karakeep data, hermes home).

## When to Use

- "Are backups working?" (audit cadence, age of latest snapshot)
- Restoring a specific file from B2
- Adding a new source dir to the backup set
- Rotating encryption / repo passwords
- Investigating B2 cost growth

## Don't Use For

- HSB backups — different setup, currently no PBS or Kopia at HSB. The `truenas-hsb` ZFS replication target is inactive per memory.
- Snapshotting live SQLite/Postgres directly — always go through the dump sidecar.
- Restoring an *entire* host — this is file-level backup, not bare-metal.

## Compose Block (current)

In `ops.yml` at Austin (line ~599):

```yaml
kopia:
  image: kopia/kopia:latest
  hostname: kopia
  networks: [t2_proxy]            # Traefik-only, no gt_internal
  command: [server, start, --insecure, --disable-csrf-token-checks,
            --address=0.0.0.0:51515, --without-password]
  environment:
    KOPIA_PASSWORD: ${KOPIA_REPO_PASSWORD}
  volumes:
    - $APPDIR/kopia/config:/app/config
    - $APPDIR/kopia/cache:/app/cache
    - $APPDIR/kopia/logs:/app/logs
    - $APPDIR/kopia/tmp:/tmp:shared
    # Source dirs (all :ro)
    - $APPDIR/paperless-ngx/media/documents/originals:/sources/paperless-originals:ro
    - $APPDIR/vaultwarden/attachments:/sources/vaultwarden-attachments:ro
    - $SECRETSDIR:/sources/secrets:ro
    - $DOCKERDIR/db-backups:/sources/db-backups:ro
    - $APPDIR/karakeep/data:/sources/karakeep:ro
    - $APPDIR/hermes/home:/sources/hermes-home:ro
```

Note `--without-password`: Kopia's own auth is disabled because Authelia in front handles user auth. The `KOPIA_PASSWORD` env var is the **repository encryption password**, not the UI password — protects the B2-stored data.

## The DB-Dump Sidecars

Running containers (`docker ps`):

- `postgres-db-backup` — dumps the multi-DB Postgres (dify, dify_plugin, immich, paperless, etc.) to `$DOCKERDIR/db-backups/postgres/`
- `maria-db-backup` — dumps MariaDB (Nextcloud, Authelia) to `$DOCKERDIR/db-backups/mariadb/`
- `hermes-sqlite-backup` — alpine sidecar doing `sqlite3 state.db ".backup"` snapshots to `$DOCKERDIR/db-backups/hermes/`

These run on cron schedules inside their containers and write to the same shared volume Kopia reads from. **The schedules matter**: if a dump cron breaks but Kopia keeps running, you snapshot stale dumps for weeks before noticing.

Verify the freshness of dumps:

```bash
ssh homelab "ls -la /mnt/ssd-storage/apps/db-backups/postgres/ | tail -10"
ssh homelab "ls -la /mnt/ssd-storage/apps/db-backups/mariadb/ | tail -10"
ssh homelab "ls -la /mnt/ssd-storage/apps/db-backups/hermes/"
```

Modified time should be within the last 24 hours. If it isn't, the sidecar is broken — fix it before trusting Kopia.

## Audit: "Are backups working?"

The single highest-value question. Steps:

1. **Open the Kopia UI** at https://backups.maximumgamer.com. Authelia bounces you through.
2. **Snapshots tab.** Each source path should have a recent snapshot (within its configured policy — typically daily). Look at the *latest* timestamp column.
3. **Policies tab.** Confirm retention is sane (e.g., 30 daily, 12 monthly).
4. **Maintenance.** Check that quick + full maintenance ran recently. If full maintenance hasn't run in >7 days, the repo grows uncontrolled.

Or via CLI inside the container:

```bash
ssh homelab "docker exec kopia kopia snapshot list --all --max-results 30"
ssh homelab "docker exec kopia kopia maintenance info"
```

A backup you haven't restored from is not a backup. **Do a test restore at least quarterly** — see next section.

## Test Restore

Pick a non-critical file, restore it to a scratch dir, diff against the live copy. Example:

```bash
ssh homelab 'docker exec kopia bash -lc "
  mkdir -p /tmp/restore-test &&
  kopia snapshot list /sources/paperless-originals --max-results 1 &&
  kopia snapshot restore <snapshot-id-from-above>/<sample-file> /tmp/restore-test/
"'
# Then compare:
ssh homelab 'diff /tmp/restore-test/<file> /mnt/ssd-storage/apps/appdata/paperless-ngx/media/documents/originals/<file>'
```

If `diff` is empty, the backup deserialized cleanly. Calendar reminder for the next quarter; don't let this slip.

## Restore a Specific File for the User

```bash
# Find the snapshot
ssh homelab 'docker exec kopia kopia snapshot list /sources/<source-name>'
# Restore one file or subtree
ssh homelab 'docker exec kopia kopia restore <snapshot-id>/<path-inside> /tmp/restore/'
# Copy out
scp homelab:/var/lib/docker/volumes/.../tmp/restore/<file> /local/path
```

The Kopia UI's "Restore" button also works — point-and-click for ad-hoc.

## Adding a New Source

1. Decide if the data is **live-snapshot-safe** (JSON, YAML, MD, JPEG, etc.) or **needs a dump-first** (live DB, write-busy file).
2. Live-snapshot-safe: add a `:ro` volume mount under `/sources/<new-name>:ro` in `ops.yml`.
3. Dump-first: add a sidecar (model on `hermes-sqlite-backup`) that writes to `$DOCKERDIR/db-backups/<service>/`, then trust the existing db-backups mount.
4. Push compose change (per CLAUDE.md workflow), `docker compose up -d kopia`.
5. In the Kopia UI: create a snapshot policy for the new source path. Default policy applies if you don't override.
6. Force a first snapshot and check it succeeds.

## Pitfalls

### Snapshotting a live SQLite/Postgres file
Will produce a corrupted backup. Always go through dump sidecars. The hermes example is the template: `sqlite3 .backup` then snapshot the dump.

### Trusting `latest snapshot` without checking source freshness
Kopia happily snapshots an empty dump dir if the sidecar failed two weeks ago. Always check dump file mtimes as part of audits.

### `:ro` enforcement
All source mounts MUST be `:ro`. A `:rw` mount means a Kopia bug or compromise could destroy live data. Read-only is the airgap.

### B2 cost growth
B2 charges for storage + Class B/C operations. Aggressive snapshot retention with deep deletion churn (e.g., paperless re-OCRing files) explodes operations cost. If the B2 monthly bill jumps, look at `kopia maintenance info` and tighten retention.

### KOPIA_REPO_PASSWORD recovery
Lose this password, lose the backups. It lives in `.env` on the Austin host (gitignored) and also needs to be backed up out-of-band (password manager). If you rotate it, the repo needs to be re-encrypted (`kopia repository change-password`) and the new password stored everywhere.

### Authelia in front means raw API doesn't work from outside
`curl https://backups.maximumgamer.com/api/...` will be Authelia-redirected. To script against Kopia, either use `docker exec kopia kopia ...` from the host or bypass via an internal call from a host on `t2_proxy`.

### Hermes home is mounted live
`$APPDIR/hermes/home` is mounted `:ro` and snapshotted live. JSON/YAML/MD is fine. The live `state.db` SQLite is NOT — it's snapshotted via the sidecar. Don't add a separate mount of `state.db` and snapshot that directly — it will be inconsistent.

## Quick Reference

```
UI:                  https://backups.maximumgamer.com   (Authelia)
Repo password env:   $KOPIA_REPO_PASSWORD               (in /mnt/ssd-storage/apps/.env)
Source mounts:       /sources/*  (all :ro)
Dump intermediate:   /mnt/ssd-storage/apps/db-backups/  (writable by sidecars only)
Destination:         Backblaze B2 (configured in kopia repo)
Snapshot list CLI:   docker exec kopia kopia snapshot list --all
Maintenance check:   docker exec kopia kopia maintenance info
Test restore:        docker exec kopia kopia restore <id>/<path> /tmp/restore/
```

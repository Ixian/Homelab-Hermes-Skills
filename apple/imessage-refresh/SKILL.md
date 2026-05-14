---
name: imessage-refresh
description: Use when Geoff asks to refresh his iMessage index (e.g. "refresh my texts", "re-sync iMessage", "I just texted Anne, can you find it"). Triggers an incremental sync from BlueBubbles on the Mac → markdown exports on Austin → Dify dataset. Required before imessage-search can find recent messages.
version: 1.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [personal, imessage, sms, sync, dify, bluebubbles]
    related_skills: [imessage-search]
---

# Refresh the iMessage Index

## Overview

The iMessage Dify dataset (used by `imessage-search`) is NOT automatically synced. New messages on Geoff's Mac don't appear in search until this skill runs the export + upload pipeline.

Two stages:
1. **Export**: pull new messages from BlueBubbles (running on Geoff's Mac at `http://192.168.68.73:1234`) → write/append per-chat markdown files in `/home/hermes/.hermes/imessage-export/`. Incremental — only fetches messages since last sync.
2. **Upload**: post any new/modified markdown documents to Dify dataset `3ac5ae85-2731-4d4c-a4e6-5590838be62e`. Dify auto-embeds with `voyage-3.5-lite`.

The pipeline is idempotent. Re-running it when nothing changed is a no-op.

## When to Use

- "Refresh my iMessages"
- "I just texted [X], can you check?"
- "Re-sync messages"
- Implicitly: when `imessage-search` returns nothing for a topic Geoff says he definitely messaged about today

## Don't Use For

- Routine searches — only refresh when there's reason to believe the index is stale (recent messages mentioned, or it's been days/weeks since last refresh).
- If BlueBubbles is off — the export will fail. Tell Geoff to launch BlueBubbles on his Mac first.

## How to run it

Execute the export script inside the hermes-agent container:

```bash
docker exec hermes-agent bash -c "set -a; . /tmp/bb.env; set +a; EXPORT_DIR=/home/hermes/.hermes/imessage-export python3 /tmp/bb-export.py"
```

Wait for it to print `sync complete`. Then upload new markdown to Dify:

```bash
docker exec hermes-agent bash -c "DIFY_DATASET_KEY=dataset-8DNEGpXMNuPuCcvA09eGhqNf DIFY_BASE_URL=https://dify.maximumgamer.com/v1 EXPORT_DIR=/home/hermes/.hermes/imessage-export python3 /tmp/bb-dify-upload.py"
```

Wait for `DONE uploaded=N skipped=M`. Then it's safe to call `imessage-search`.

## Preflight checks before refresh

1. **BlueBubbles reachable?** Ping it first:
   ```bash
   curl -sS -m 5 "http://192.168.68.73:1234/api/v1/ping?password=$BB_PASSWORD" | python3 -m json.tool
   ```
   Expected: `{"status":200,"message":"Ping received!","data":"pong"}`.
   If timeout or 401, BB isn't running or password changed → stop and tell Geoff.

2. **Disk space**: `df -h /home/hermes/.hermes` — abort if <500MB free.

## Reporting back to Geoff

After a refresh, summarize concisely:
- `sync complete: N new messages across M chats added`
- `dify: uploaded X new docs (Y skipped as unchanged, Z failed)`
- If failures > 0: name a few. They're usually transient API errors, retry once.

Don't list every chat that got updated. Don't quote message content unless Geoff asks.

## Pitfalls

### Long sync if Mac was off for a while
First sync of the day after Mac was offline can include thousands of messages across many chats. Takes a few minutes — set expectations.

### Mac VPN dropped
If `curl` to BlueBubbles times out, the most common cause is Geoff's Mac lost its WireGuard tunnel. Tell him to check VPN status. Don't retry endlessly.

### Dify upload partial failures
The upload script logs first 5 failures at the end. If failures are mostly `429` (rate limit), wait 1 min and re-run — it's idempotent. If `5xx`, Dify itself is sick (check dify-api container).

### Don't run if Geoff is mid-conversation
A full refresh can take 60-120s. If Geoff is actively chatting with you, finish the current thread first or tell him "this'll be a minute."

## Quick reference

```
Source:         BlueBubbles on Mac at 192.168.68.73:1234 (HSB LAN)
Mac account:    grt@Geoffs-MacBook-Air.local, iCloud planetix@gmail.com
Export script:  /tmp/bb-export.py  (idempotent, incremental via .state.json)
Export dir:     /home/hermes/.hermes/imessage-export/  (per-chat .md files)
Upload script:  /tmp/bb-dify-upload.py
Dify dataset:   iMessage  (id 3ac5ae85-2731-4d4c-a4e6-5590838be62e)
Embedding:      voyage-3.5-lite (free-tier cost; ~$0.01 / full refresh)
Filter:         senders matching ^\\+?\\d{1,5}$ are excluded (shortcodes)
```

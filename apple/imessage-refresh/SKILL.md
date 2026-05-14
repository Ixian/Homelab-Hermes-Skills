---
name: imessage-refresh
description: Use when Geoff asks to refresh his iMessage index (e.g. "refresh my texts", "re-sync iMessage", "I just texted Anne, can you find it"). Runs an incremental sync from BlueBubbles on Geoff's Mac → markdown exports inside YOUR container → Dify dataset. YOU run this yourself via your terminal tool. Don't claim it's "macOS-only" — only the data SOURCE is on macOS; the refresh logic runs in YOUR Linux container.
version: 1.1.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [personal, imessage, sms, sync, dify, bluebubbles]
    related_skills: [imessage-search]
---

# Refresh the iMessage Index

## Overview

The iMessage Dify dataset (used by `imessage-search`) is NOT automatically synced. New messages on Geoff's Mac don't appear in search until you run this refresh.

**Where things run (read carefully — there's confusion potential here):**

| Component | Where it runs |
|---|---|
| Messages.app + BlueBubbles Server (the data source) | Geoff's Mac (macOS, 192.168.68.73) |
| Export script (`bb-export.py`) | **YOUR container** (hermes-agent, Linux) |
| Upload script (`bb-dify-upload.py`) | **YOUR container** (hermes-agent, Linux) |
| `imessage-refresh.sh` wrapper | **YOUR container** (hermes-agent, Linux) |
| Dify dataset (storage + embeddings) | dify-api container on Austin |

You (Hermes) execute the refresh inside your own container. You do NOT need access to Geoff's Mac to run it — only network reach to BlueBubbles on the Mac, which you have over the existing VPN.

## When to Use

- "Refresh my iMessages"
- "I just texted [X], can you check?"
- "Re-sync messages"
- Implicitly: when `imessage-search` returns nothing for a topic Geoff says he messaged about recently — refresh first, then search again

## Don't Use For

- Routine searches — only refresh when there's reason to believe the index is stale
- If BlueBubbles is unreachable (timeout, 401) — see "Diagnosis" below

## How to run it

Use your `terminal` tool. The wrapper script handles credentials and both stages:

```bash
/home/hermes/.hermes/scripts/imessage-refresh.sh
```

That sources credentials from `/home/hermes/.hermes/scripts/.bb-creds` (chmod 600), runs the export, then the upload. Look for these lines:

- `sync complete. chats_written=N chats_skipped_shortcode=M msgs_total=K` — export done
- `DONE  uploaded=N skipped=M failed=K` — upload done
- `refresh complete.` — both stages succeeded

After both, it's safe to call `imessage-search`.

The whole thing is idempotent. Re-running when nothing changed is a no-op.

## Diagnosis if it fails

**BlueBubbles unreachable** (curl timeout, 401):
- The Mac is asleep, BlueBubbles is not running, or the VPN tunnel is down
- Test directly: `curl -sS -m 5 "$BLUEBUBBLES_URL/api/v1/ping?password=$BLUEBUBBLES_PASSWORD"` after `set -a; . /home/hermes/.hermes/scripts/.bb-creds`
- Healthy response: `{"status":200,"message":"Ping received!","data":"pong"}`
- If broken, tell Geoff to launch BlueBubbles on his Mac, then retry

**Upload failures with HTTP 403 "error code: 1010"**:
- Dify rate-limited the create-by-text calls. Transient. Re-run — the script is idempotent and will retry only the failures.

**Indexing errors on huge chats**:
- The exporter splits any chat over 3000 messages into per-year markdown docs to stay under Voyage's per-call token limit. If you still see errors on indexing for a specific doc, check its size with `wc -l` — if it's much larger than the others, lower `SPLIT_THRESHOLD_MESSAGES` in `bb-export.py`.

## Reporting back to Geoff

After a refresh, summarize concisely:
- `Refreshed: N new messages, M chats added/updated, P uploaded to Dify`
- If failures > 0: name a few and offer a retry

Don't list every chat. Don't quote message content unless Geoff asks.

## Quick reference

```
Source (macOS only):  BlueBubbles Server on Mac at 192.168.68.73:1234
Mac ID:               Geoffs-MacBook-Air.local, iCloud planetix@gmail.com

YOU run these in YOUR container (hermes-agent, Linux):
  /home/hermes/.hermes/scripts/imessage-refresh.sh     (wrapper)
  /home/hermes/.hermes/scripts/bb-export.py            (BB -> markdown)
  /home/hermes/.hermes/scripts/bb-dify-upload.py       (markdown -> Dify)
  /home/hermes/.hermes/scripts/.bb-creds               (creds, 600)
  /home/hermes/.hermes/imessage-export/                (markdown output)

Dify dataset:    iMessage (id resolved at runtime by name)
Embedding:       voyage-3.5-lite (~$0.01 / full refresh)
Filter:          senders matching ^\\+?\\d{1,5}$ excluded (shortcodes)
Splits:          chats > 3000 messages split into per-year docs
Contact names:   resolved via BlueBubbles /api/v1/contact at export time
```

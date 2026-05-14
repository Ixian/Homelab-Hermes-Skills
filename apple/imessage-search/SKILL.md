---
name: imessage-search
description: Use when the user asks about anything in their iMessage/SMS history. Invokes the iMessage_Search Dify tool which returns relevant message excerpts from the indexed archive. Semantic search only — cannot filter by date or recency. The archive is a static snapshot (no auto-refresh).
version: 3.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [personal, imessage, sms, dify, rag]
---

# Search Geoff's iMessage / SMS History

## Overview

Geoff has an indexed semantic-search archive of his past iMessage/SMS history in Dify. The `iMessage_Search` tool (under the `homelab-docs` MCP, alongside `Homelab_Bot` and `Pool_Bot`) queries that archive.

**The archive is a STATIC SNAPSHOT.** There is no auto-refresh and no way to add new messages — the BlueBubbles ingest pipeline was removed. Whatever is in the index is what's searchable, frozen at the last snapshot date.

## When to Use

- "What did Anne say about the trip?"
- "Did I tell my mom about X?"
- "Find the address Sarah sent me"
- "When did we discuss the pool?"
- Cross-reference questions where the natural source is past texts

## When NOT to Use

- **Anything time-bounded** like "this week", "today", "most recent" — semantic search cannot filter by date. Results will be old hits that happen to be topically similar. Tell the user the archive is a static snapshot and offer to search by topic instead.
- Anything Geoff can find faster on his phone (current Messages.app)
- Anything Geoff says NOT to look at — respect that

## How to invoke

```
iMessage_Search(query="<natural-language topic>")
```

Returns up to ~8 chunks with chat name, similarity score, and message excerpts.

## Interpreting results

```
--- Excerpt 1 (chat: Sandra_Tognetti__2024__-__15124842106, score: 0.72) ---
**2024-06-12 18:30** `Sandra Tognetti (+15124842106)`: <message>
**2024-06-12 18:31** `me`: <reply>
```

- **score** ~0.4–0.85. Above 0.6 is usually relevant; below 0.4 is a stretch.
- **chat name** ends in `__<chat-guid>`. Large chats are split per-year so name includes year.
- **sender labels**: `me` = Geoff. Others show resolved contact names from the snapshot.

## Quick reference

```
Tool:            iMessage_Search (homelab-docs MCP)
Backend:         Dify dataset c9765049-2076-466b-9619-18e41fee7f9c
Embedding:       voyage-3.5-lite (semantic similarity)
Archive shape:   ~52k messages, ~900 chats, frozen snapshot
Refresh:         None. No incremental sync. Snapshot is read-only.
Excludes:        SMS shortcode senders (2FA / promos)
```

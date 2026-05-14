---
name: imessage-search
description: Use when the user asks about anything in their iMessage/SMS history. Two tools are available — iMessage_Search (semantic, Dify RAG over full archive — for TOPIC questions) and iMessage_Recent (live BlueBubbles query — for TIME-BOUNDED or recent-contact questions). Pick the right one. Semantic search cannot filter by date; live query cannot search by topic.
version: 2.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [personal, imessage, sms, dify, rag, bluebubbles]
    related_skills: [imessage-refresh]
---

# Querying Geoff's iMessage / SMS History

## Two tools, two different jobs

You have access to **two different tools** for iMessage queries. Pick the right one — they're not interchangeable.

### `iMessage_Search` — Dify semantic search

- **Use for TOPIC questions** across all history: "what did Anne say about the trip", "when did we discuss the pool", "find the address Sarah sent me"
- Returns top semantic-search chunks ranked by topical similarity to the query
- Covers the FULL archive (~52k messages, ~881 chats, ~902 documents — large chats split per-year)
- Indexed in Dify with voyage-3.5-lite embeddings
- **CANNOT filter by date.** Semantic search ranks by content similarity, not recency. Asking for "this week" or "recent" will return whatever's topically similar regardless of when sent.
- Args: `query` (natural language topic)

### `iMessage_Recent` — Live BlueBubbles query

- **Use for TIME-BOUNDED questions about a specific contact**: "what did Sandra text this week", "most recent messages from Bob", "did Anne text me today", "show me my last 20 messages with Mom"
- Hits BlueBubbles directly (no Dify) and pulls actual messages in chronological order
- Resolves the contact by name against Geoff's address book (66 contacts, partial name match OK if unique)
- Returns messages newest-first with timestamps and sender names
- **CANNOT search by topic.** It pulls ALL recent messages from the chat; you'll need to read them and find what matters.
- Args: `contact` (name, required), `days_back` (default 7, max 60), `limit` (default 50, max 200)
- Latency: ~3s first call (chat-list scan), ~0.5s for subsequent calls within 5 min (cached)

## Decision rules

| Question shape | Tool | Why |
|---|---|---|
| "What did X say about Y?" | `iMessage_Search` | Topic-driven |
| "When did we talk about Z?" | `iMessage_Search` | Topic-driven |
| "Find the address/phone/link X sent" | `iMessage_Search` | Topic-driven |
| "What did X text this week / today / recently?" | `iMessage_Recent` | Date-bounded |
| "Show me my last N messages with X" | `iMessage_Recent` | Recency-bounded |
| "Did X text me about Y today?" | `iMessage_Recent` THEN scan for Y | Date + topic — date filter first |
| "What's the dinner reservation time?" | `iMessage_Search` first; if no recent hits, `iMessage_Recent` | Topic primary |
| "Catch me up on X" | `iMessage_Recent` (broader days_back) | Recency-driven |

When in doubt: if the question mentions a TIME WINDOW (today, this week, recently, last N days), use `iMessage_Recent`. If it's about a TOPIC across time, use `iMessage_Search`. **Never use `iMessage_Search` for "this week" — it cannot filter by date.**

## When NOT to use either

- Anything you can answer from your own context or other tools (HA, file system, web search) without invoking message history — don't pollute responses with iMessage chunks unprompted
- Anything Geoff explicitly says NOT to look at — respect "don't read my texts" type instructions
- Group-wide questions where you'd expose one person's messages in a discussion that includes others; pause and confirm

## Interpreting `iMessage_Search` results

Each excerpt is formatted like:

```
--- Excerpt 1 (chat: Sandra_Tognetti__2024__-__15124842106, score: 0.72) ---
**2024-06-12 18:30** `Sandra Tognetti (+15124842106)`: <message>
**2024-06-12 18:31** `me`: <reply>
```

- **score** is semantic similarity, ~0.4-0.85. Above 0.6 is usually relevant.
- **chat name** is `<participant_name>__<year-if-split>__<chat-guid>` — recent chats from large conversations have a year suffix; small chats don't.
- **sender labels**: `me` = Geoff. Other senders have their real name resolved from his address book (e.g. `Sandra Tognetti (+15124842106)`).

## Interpreting `iMessage_Recent` results

Cleaner format, newest-first:

```
Last 12 message(s) with Sandra Tognetti (last 7 day(s), newest first):

**2026-05-14 16:36** `me`: I booked everything...
**2026-05-14 16:23** `Sandra Tognetti`: Did you want to do a couples massage...
...
```

- Senders are real names (already resolved). `me` = Geoff.
- If the response says "No 1:1 chat with X found", they may only be in group chats — `iMessage_Search` with their name is the fallback.
- If the response says "matches multiple contacts", you need a more specific name.

## Pitfalls

### Don't use `iMessage_Search` for date-bounded questions
The old version of this skill conflated the two. Semantic search returning older chunks for "this week" queries is the #1 way to look stupid. Use `iMessage_Recent` for anything time-bounded.

### Group chats aren't in `iMessage_Recent`'s 1:1 lookup
`iMessage_Recent` finds 1:1 conversations only. For "what did Sandra say in the family group chat this week," use `iMessage_Search` with her name + the group context as the query.

### Cache TTL is 5 min
Contact list and chat-index are cached in mcp-dify memory for 5 minutes. If a new contact was just added on the Mac, it might not be visible for up to 5 minutes. Rare edge case.

### `iMessage_Search` is only as fresh as the last `imessage-refresh` run
The Dify index isn't auto-synced. New messages don't appear in `iMessage_Search` until refresh. `iMessage_Recent` is always live.

### Geoff dislikes verbose preambles
Don't say "Let me search your iMessages..." then dump results. Just answer.

## Quick reference

```
TOPIC across years:           iMessage_Search(query="...")
RECENT from specific person:  iMessage_Recent(contact="Name", days_back=7)
Both available via:           homelab-docs MCP
Backend (Search):             Dify dataset c9765049-..., voyage-3.5-lite
Backend (Recent):             BlueBubbles REST @ 192.168.68.73:1234 (live)
Source corpus:                ~52k messages, ~881 chats from Geoff's Mac
Excludes:                     3-5 digit shortcode senders (2FA/promos)
```

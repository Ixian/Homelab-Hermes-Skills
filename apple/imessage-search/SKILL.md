---
name: imessage-search
description: Use when the user asks about anything that might be in their iMessage/SMS history — past conversations, what someone said about a topic, recalling dates/details/decisions discussed over text. Invokes the iMessage_Search Dify tool which returns relevant message excerpts from the full archive (exported via BlueBubbles, indexed in Dify). On-demand only; the archive is refreshed manually via the imessage-refresh skill.
version: 1.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [personal, imessage, sms, dify, rag, bluebubbles]
    related_skills: [imessage-refresh]
---

# Search Geoff's iMessage / SMS History

## Overview

Geoff's full iMessage + SMS + RCS history is exported from his Mac (via BlueBubbles) and indexed in Dify as a semantic-search dataset. This skill is your interface to that archive. When you need to answer a question that depends on what someone said over text, or when Geoff asks you to recall something from his messages, use the `iMessage_Search` tool exposed by the homelab-docs MCP.

The archive includes ~52,000 messages across ~880 chats. Senders that look like SMS shortcodes (3-5 digit numbers — 2FA codes, delivery alerts, promotional shortcodes) are excluded.

## When to Use

- "What did Anne say about the trip?"
- "Did I tell my mom about X?"
- "When did I last text Bob about the pool?"
- "Find the address Sarah sent me"
- Any question where the natural source of the answer is Geoff's text history
- Cross-reference questions: "What's the dinner reservation time? Check my texts."

## Don't Use For

- Anything you can answer from your own context or other tools (HA, file system, web search) without invoking the message archive — don't pollute response with iMessage chunks unprompted
- Anything Geoff explicitly says NOT to look at — respect "don't read my texts" type instructions
- Group-wide questions where you'd be exposing one person's messages to a discussion involving someone else; pause and confirm

## How to invoke

The `iMessage_Search` tool (under the `homelab-docs` MCP) takes a single `query` argument — natural language. Examples:

```
iMessage_Search(query="Anne trip planning Thanksgiving")
iMessage_Search(query="pool pump rpm what did Bob say")
iMessage_Search(query="dinner reservation address Saturday")
```

The tool returns up to 8 message excerpts (semantic-search, top-k=8) with chat names and similarity scores. Each excerpt is a chunk from one chat's markdown export.

## Interpreting results

Each excerpt is formatted like:

```
--- Excerpt 1 (chat: Anne_Smith__chat<guid>, score: 0.72) ---
**2026-04-12 18:30** `anne@icloud.com`: <message text>
**2026-04-12 18:31** `me`: <reply>
...
```

- **score** is the Dify semantic similarity score, typically 0.4-0.85. Above ~0.6 is usually relevant; below ~0.4 may be a stretch.
- **chat name** is `<participant>__<chat-guid>` — useful for telling apart multiple conversations with similar topics.
- **sender labels**: `me` = Geoff, otherwise the phone/email/handle.
- **timestamps** are in Geoff's local timezone (Central, America/Chicago).

When multiple excerpts come back, prefer recent + high-score. If the question is about a specific person, prioritize excerpts from chats where that person is a participant (chat name will hint).

## Refining queries

The retrieval is semantic, not keyword. Better queries describe the *topic* rather than the exact words.

- ❌ "find text containing 'meet at 3'"  → too narrow
- ✓ "scheduling a meeting time with Sarah this week"

If first result is empty or weak, try:
1. Broaden the topic ("Anne trip" instead of "Thanksgiving 2026 Anne flight 1132")
2. Try a different angle ("Anne planning travel" instead of "Anne airport")
3. Try the contact's name + the rough topic in one query

## Pitfalls

### Staleness
The Dify index is only as fresh as the last `imessage-refresh` run. There's no auto-sync. If Geoff asks about a very recent message, suggest running `imessage-refresh` first.

### Privacy: don't paraphrase sensitive content unprompted
Excerpts can contain personal/financial/medical info. When summarizing or quoting back, lean toward minimum-necessary: answer the actual question, don't dump the full excerpt unless Geoff asked to see it.

### Shortcodes excluded by design
2FA codes, USPS/FedEx delivery alerts, "STOP to opt out" promos are NOT in the index. Don't suggest the index can answer "what was that 2FA code from Amazon?" — it can't.

### Chat names can be cryptic
For group chats or unnamed conversations, the chat name will be a long string of phone numbers. The excerpts themselves show senders clearly though.

### Geoff dislikes verbose preambles
Don't say "Let me search your iMessages..." then dump results. Just answer the question and cite an excerpt if needed.

## Quick reference

```
Tool:           iMessage_Search (via homelab-docs MCP)
Input:          query: string (natural language)
Output:         Top-8 semantic-search excerpts, raw markdown chunks
Dataset id:     bc4a6ec7-8f72-4309-961d-63fa91de18c7 (Dify, "iMessage")
Refresh:        Manual via imessage-refresh skill
Source:         BlueBubbles on Geoff's Mac (HSB, 192.168.68.73)
Excludes:       3-5 digit shortcode senders (2FA / promos / delivery)
Volume:         ~52k messages, ~880 chats
```

---
name: compose-drift-check
description: Use to detect and resolve drift between the local Homelab-Austin git clone and the deployed Austin server. Compares compose files and recently-touched appdata configs, classifies drift, and offers four sync options. Always fetch origin first — this Mac is one of several clones and can be the stalest.
version: 1.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [homelab, docker-compose, drift, austin, deployment]
    related_skills: [homelab-ops, traefik-route-add]
---

# Detect and Resolve Compose/Config Drift

## Overview

The Austin Docker host is the live source of truth for running services. The `Homelab-Austin` git repo (cloned from Forgejo, mirrored to GitHub) is *supposed* to match. In practice, drift happens — someone edits on the server, a deploy step gets skipped, or this Mac falls behind another clone that pushed first. This skill catches the drift before the next deploy overwrites real work.

**Critical first move: `git fetch origin` before assuming anything.** This Mac is one of several machines syncing with Forgejo. The drift you're seeing might be your local clone being stale, not the server diverging.

## When to Use

- Before deploying any compose change ("am I about to overwrite something?")
- After someone reports a service behaving differently than the YAML says
- Routinely after a session where files were touched on the server (during incident response, etc.)
- As a pre-merge check on PRs touching `Homelab-Austin/compose/` or `appdata/`

## Workflow

### 1. Fetch origin first (non-negotiable)

```bash
cd /Users/grt/Github/Homelab/Homelab-Austin && git fetch origin
git log --oneline HEAD..origin/main   # what's upstream that you don't have
git log --oneline origin/main..HEAD   # what's local that origin doesn't have
```

If `HEAD..origin/main` is non-empty, **you are stale.** Pull (or rebase) before comparing to the server, otherwise you'll "fix" drift that's already resolved upstream.

### 2. Compose file diff

For each tracked file in `Homelab-Austin/compose/*.yml`:

```bash
diff <(cat Homelab-Austin/compose/<file>.yml) <(ssh homelab "cat /mnt/ssd-storage/apps/compose/<file>.yml")
```

Or in one pass:

```bash
for f in /Users/grt/Github/Homelab/Homelab-Austin/compose/*.yml; do
  name=$(basename "$f")
  [ "$name" = "projectx.yml" ] && continue   # excluded — see below
  diff -q "$f" <(ssh homelab "cat /mnt/ssd-storage/apps/compose/$name") \
    && echo "OK  $name" || echo "DIFF $name"
done
```

**Exclude `projectx.yml`** — intentionally untracked (experimental/disabled). It's expected to exist on the server but not in git.

### 3. Appdata config drift (scope it)

Appdata is huge; don't diff the whole tree. Limit to files git has touched recently:

```bash
cd Homelab-Austin && git log --since="1 week ago" --name-only --pretty=format: appdata/ \
  | sort -u | grep -v '^$' > /tmp/recent-appdata.txt
while read p; do
  diff -q "$p" <(ssh homelab "cat /mnt/ssd-storage/apps/$p") \
    && echo "OK  $p" || echo "DIFF $p"
done < /tmp/recent-appdata.txt
```

For each DIFF, run a real `diff -u` to see what changed.

### 4. Classify

Categorize every drifted file into one of:

| Class | Meaning | Default response |
|---|---|---|
| **Server-only** | Exists on server, not in git (and not in exclusion list) | Pull → workspace, commit |
| **Workspace-only** | In git, not on server | Push → server, then `docker compose up -d` |
| **Content diff, workspace newer** | Git has the change you want | Push → server |
| **Content diff, server newer** | Server has the truth | Pull → workspace, commit |
| **Both newer** | Real conflict | Hand-merge, ask user |
| **Expected drift** | projectx.yml, secrets, ephemeral state | Ignore, document the exception |

### 5. Offer the four sync options (prompt the user)

For each non-trivial drift, present:

- **A: Pull server → workspace** (server is right; git catches up)
- **B: Push workspace → server** (git is right; deploy)
- **C: Manual review** (show full diff, decide per file)
- **D: Ignore** (drift is expected; document why so it doesn't get flagged again)

Don't pick autonomously for compose files. For unambiguous appdata cases (e.g., a comment added on the server) **A** is usually right and can be batched with confirmation.

## Deploy Steps After a "Push to Server" Choice

Per CLAUDE.md workflow:

```bash
cat Homelab-Austin/compose/<file>.yml | ssh homelab "cat > /mnt/ssd-storage/apps/compose/<file>.yml"
ssh homelab "cd /mnt/ssd-storage/apps && docker compose up -d <service>"
```

Or for appdata:

```bash
cat Homelab-Austin/appdata/<path> | ssh homelab "cat > /mnt/ssd-storage/apps/appdata/<path>"
ssh homelab "docker restart <container>"
```

**Restart caveat (from .claude/rules/docker.md):** restarting `gluetun` requires also restarting `qbittorrent`.

## After Resolving Drift

Always:

1. Commit both ends — local change committed and pushed to both remotes (`git push origin && git push github`).
2. Re-run drift check; expect clean state.

## Pitfalls

### Trusting your local clone
This is the #1 failure mode. You see `git status` is clean and assume your tree is current. It might be 3 commits behind another machine that pushed earlier. **Always `git fetch origin` first.** Saved as a feedback memory for a reason.

### "Server is newer" is not always pull-worthy
If someone edited on the server in panic mode during an incident, the change might be a hotfix that needs cleanup before it goes into git. Read the diff, don't just batch-pull.

### Diffing the wrong remote
`origin` is Forgejo (`git.planetixian.com`). `github` is mirror. If you pull from `github` you might miss work pushed to Forgejo. Always fetch/pull from `origin`.

### Ignoring projectx.yml is deliberate
Don't add it to the diff loop. It's an experimental stack that intentionally lives only on the server.

### `.env` files
`.env` is gitignored but lives on the server. It will always "drift" from git's perspective (it's not there). Exclude `*.env` from any auto-diff.

### Restarts that take chains down
`docker compose up -d <service>` recreates with the new compose. For services that other services depend on (e.g., `postgres`, `redis`, `traefik`), that's a cascade. Read `depends_on` before restart; consider `docker compose up -d --no-deps <service>` for surgical replacement.

## One-Liner: Quick Health Check

```bash
cd /Users/grt/Github/Homelab/Homelab-Austin && git fetch origin -q && \
  git log --oneline HEAD..origin/main | head -5 && echo "--- compose drift ---" && \
  for f in compose/*.yml; do n=$(basename "$f"); [ "$n" = projectx.yml ] && continue; \
  diff -q "$f" <(ssh homelab "cat /mnt/ssd-storage/apps/compose/$n") 2>&1 | sed 's/^/  /'; done
```

Run this before any deploy session. If it's clean and `HEAD..origin/main` is empty, you can proceed.

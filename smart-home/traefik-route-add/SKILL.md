---
name: traefik-route-add
description: Use when adding a new web-facing service to the Austin Docker host. Scaffolds the Traefik labels, network anchor, Authelia chain, and compose-file placement so the new subdomain works on first restart without touching Traefik or Authelia config.
version: 1.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [homelab, traefik, authelia, docker-compose, austin]
    related_skills: [homelab-ops, compose-drift-check]
---

# Add a Traefik Route to the Austin Stack

## Overview

Austin runs ~50 services behind a single Traefik v3 reverse proxy with Authelia SSO and CrowdSec. The label conventions are uniform — every new service gets the same five Traefik labels, joins the same two networks, and inherits Authelia automatically via a wildcard rule. This skill scaffolds a new service block so the URL `https://<name>.maximumgamer.com` works the moment the container comes up.

**No Traefik or Authelia config changes are needed for new subdomains.** The middleware `chain-authelia@file` plus the `*.maximumgamer.com` wildcard rule in Authelia handles policy automatically (bypass from internal nets, two-factor from external).

## When to Use

- Adding a new container that needs an HTTPS URL on `maximumgamer.com`
- Migrating a service from direct port-binding to behind Traefik
- Setting up multi-route services (API on a path, UI on another)

## Don't Use For

- Internal-only services (use the MCP garden pattern from `mcp.yml` instead — `gt_internal` network only, no Traefik labels)
- Services on `planetixian.com` (different convention, used only for Pangolin/headscale/monitor — ask first)
- Adding new entrypoints, certs, or middleware chains — that's Traefik config, not a route add

## Required Inputs

Before writing any YAML, confirm:

| Input | Notes |
|---|---|
| Service name | lowercase, hyphen-separated. Used as `<name>-rtr`, `<name>-svc`, container_name |
| Internal container port | the port the app listens on inside the container (not the host) |
| Subdomain | usually `<name>` — `<name>.maximumgamer.com` |
| Compose file | which of `apps.yml / media.yml / ops.yml / utils.yml / security.yml / misc.yml / network.yml / dify.yml / hermes.yml / karakeep.yml / projectx.yml / mcp.yml / core.yml / manifest.yml` it belongs in (see "Compose file placement" below) |
| Auth needed? | Default yes (chain-authelia@file). Public APIs/webhooks may need a no-auth chain — flag to user |

## Canonical Service Block (simple, single route)

This is the Dify frontend pattern at `dify.yml:185-202`. Use it as the template for 90% of new services.

```yaml
  <name>:
    <<: *network-internal-and-security
    container_name: <name>
    image: <image>:<tag>
    restart: unless-stopped
    environment:
      TZ: $TZ
      PUID: $PUID
      PGID: $PGID
    volumes:
      - $APPDIR/<name>:/config
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.<name>-rtr.entrypoints=https"
      - "traefik.http.routers.<name>-rtr.rule=Host(`<name>.$DOMAINNAME0`)"
      - "traefik.http.routers.<name>-rtr.middlewares=chain-authelia@file"
      - "traefik.http.routers.<name>-rtr.service=<name>-svc"
      - "traefik.http.services.<name>-svc.loadbalancer.server.port=<internal-port>"
```

Key invariants:

- `<<: *network-internal-and-security` pulls in both networks (`t2_proxy` + `gt_internal`) plus `no-new-privileges`. The anchor is defined at `base.yml:137`. Do not redeclare `networks:` — that overrides the anchor and silently drops the proxy network.
- Five labels. Not four, not six. Anything else is multi-route territory.
- `$DOMAINNAME0` resolves to `maximumgamer.com` from `.env`. Never hardcode the domain.
- No `ports:` block. Exposing a host port bypasses Traefik and breaks the auth model.

## Multi-Route Service Block (API + UI, different paths)

When the same host name needs to fan out to multiple containers (or path-prefixed routes), use **priority** to disambiguate. Pattern is from Dify (`dify.yml:90-108` for the API, `185-202` for the catch-all frontend).

```yaml
    labels:
      - "traefik.enable=true"
      # Specific route — higher priority wins
      - "traefik.http.routers.<name>-api-rtr.entrypoints=https"
      - "traefik.http.routers.<name>-api-rtr.rule=Host(`<name>.$DOMAINNAME0`) && PathPrefix(`/api`)"
      - "traefik.http.routers.<name>-api-rtr.middlewares=chain-authelia@file"
      - "traefik.http.routers.<name>-api-rtr.priority=100"
      - "traefik.http.routers.<name>-api-rtr.service=<name>-api-svc"
      - "traefik.http.services.<name>-api-svc.loadbalancer.server.port=5001"
      # Catch-all UI — explicit low priority
      - "traefik.http.routers.<name>-rtr.entrypoints=https"
      - "traefik.http.routers.<name>-rtr.rule=Host(`<name>.$DOMAINNAME0`)"
      - "traefik.http.routers.<name>-rtr.middlewares=chain-authelia@file"
      - "traefik.http.routers.<name>-rtr.priority=1"
      - "traefik.http.routers.<name>-rtr.service=<name>-svc"
      - "traefik.http.services.<name>-svc.loadbalancer.server.port=3000"
```

The catch-all needs `priority=1` (Traefik's default priority is computed from rule length and will sometimes outrank your intended specific rule). Always set priority on both when there are multiple routers for one host.

## Public / No-Auth Routes

For webhooks, public APIs, or services Authelia can't intercept (most OAuth callbacks):

```
- "traefik.http.routers.<name>-rtr.middlewares=chain-no-auth@file"
```

Use sparingly. CrowdSec still applies (it watches the Traefik access log via `crowdsecurity/traefik`, not per-service labels). Confirm with the user that no auth is intended.

## Compose File Placement

The Austin stack is split by domain. Pick the file that already contains peers:

| File | Holds |
|---|---|
| `apps.yml` | General apps (Nextcloud, Vaultwarden, Paperless, etc.) |
| `media.yml` | *arr stack, qBittorrent, Plex/Jellyfin |
| `ops.yml` | Kopia, monitoring agents, db-backup sidecars |
| `utils.yml` | Heimdall, code-server, small tooling |
| `security.yml` | Traefik, Authelia, CrowdSec |
| `network.yml` | DNS, WireGuard, network plumbing |
| `dify.yml` | Dify stack (closed) |
| `hermes.yml` | Hermes stack (closed) |
| `mcp.yml` | MCP garden — internal-only, **not Traefik-exposed** |
| `core.yml` | Postgres, MariaDB, Redis (shared DBs) |

Edit the file in the local clone at `Homelab-Austin/compose/<file>.yml`, **never on the server directly** (per CLAUDE.md workflow).

## Reuse Before You Add

Before pulling in new infrastructure containers, check `core.yml`:

- **Postgres** (`ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`) — multi-DB, pgvector + vectorchord preloaded. Hosts dify, dify_plugin, immich, paperless. Add a new DB here rather than spinning another Postgres.
- **Redis** (`redis:8`) — AOF, 512MB cap, on `gt_internal`. No password currently. Reuse for queues/cache.
- **MariaDB** — used by Nextcloud + Authelia. Available for new MySQL services.

If your service needs a fresh DB, add the schema via `core.yml`'s init scripts. Don't add a per-service Postgres container.

## Deployment Workflow (per CLAUDE.md)

1. Edit `Homelab-Austin/compose/<file>.yml` locally.
2. Commit and push both remotes: `git push origin && git push github`.
3. Deploy: `cat Homelab-Austin/compose/<file>.yml | ssh homelab "cat > /mnt/ssd-storage/apps/compose/<file>.yml"`.
4. Reload the stack: `ssh homelab "cd /mnt/ssd-storage/apps && docker compose up -d <name>"`. Compose v2 (no hyphen).
5. Verify (see below).

## Verification

```bash
# Traefik picked up the router
ssh homelab "docker logs traefik 2>&1 | tail -50 | grep -i <name>"

# Container is on both networks
ssh homelab "docker inspect <name> --format '{{range \$k,\$v := .NetworkSettings.Networks}}{{\$k}} {{end}}'"
# Expect: gt_internal t2_proxy

# DNS resolves
dig +short <name>.maximumgamer.com
# Expect Austin's public IP (or LAN IP if you're inside the network and split-DNS is set)

# Auth challenge fires
curl -sI https://<name>.maximumgamer.com | head -5
# Expect 302 to https://auth.maximumgamer.com (or 200 from internal net per Authelia bypass)
```

Then load in a browser. Authelia redirects to login the first time; subsequent visits in the same SSO session pass through.

## Pitfalls

### Forgot the anchor merge
If you write `networks: [t2_proxy]` directly instead of `<<: *network-internal-and-security`, the container has no `gt_internal` access (so it can't reach Postgres/Redis) and no `no-new-privileges`. Always use the anchor unless you have a specific reason.

### Two routers with same host, no priority
Traefik 502s or routes inconsistently. Symptom: the API works sometimes, the UI works other times. Fix: explicit `priority=100` on the specific rule, `priority=1` on the catch-all.

### Port mismatch in label vs container
`loadbalancer.server.port` must be the **container's** listen port, not the host port. There is no host port (no `ports:` block); Traefik talks to the container over `t2_proxy`.

### Authelia 403 from external network
The wildcard rule is on `*.maximumgamer.com` — if you accidentally use a different domain (e.g. `planetixian.com`) the rule doesn't match and falls through to `default_policy: deny`. Stick to `$DOMAINNAME0`.

### CrowdSec banning yourself
CrowdSec watches Traefik access logs. Multiple failed Authelia logins from the same IP can ban you. Whitelist your VPN/LAN range or use the CrowdSec dashboard to release.

### Editing on the server
Per CLAUDE.md: "NEVER edit server files directly." Always edit `Homelab-Austin/compose/` locally, commit, push, then `cat | ssh ... cat >`. Otherwise the next deploy from git silently reverts your change.

### Forgetting the GitHub mirror
`git push origin && git push github` — both remotes per the git rule. Origin is Forgejo (`git.planetixian.com`), GitHub is mirror.

## Quick Reference Card

```
Networks anchor:  <<: *network-internal-and-security   (base.yml:137)
Auth middleware:  chain-authelia@file                   (no per-service config needed)
No-auth variant:  chain-no-auth@file                    (use sparingly; CrowdSec still applies)
Domain var:       $DOMAINNAME0  → maximumgamer.com
Internal port:    loadbalancer.server.port=<container-port>
Multi-route:      priority=100 on specific, priority=1 on catch-all
Compose v2:       docker compose up -d <name>          (no hyphen)
```

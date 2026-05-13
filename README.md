# Homelab-Hermes-Skills

Custom Hermes Agent skills for the homelab. Deployed to `/mnt/ssd-storage/apps/appdata/hermes/home/skills/` on the Austin Docker host. Mirror the directory layout in this repo to the deploy path 1:1.

## Layout

```
smart-home/
  traefik-route-add/SKILL.md         Add a new Traefik route with Authelia
  opnsense-ops/SKILL.md              Firewall/NAT/alias ops on both OPNsense boxes
  compose-drift-check/SKILL.md       Detect & resolve git ↔ server drift
  frigate-tune/SKILL.md              Frigate NVR config edits (ha2 add-on)
  wireguard-headscale-ops/SKILL.md   Cross-site VPN (WG + Headscale)

devops/
  proxmox-lxc-ops/SKILL.md           HSB Proxmox LXC ops (102, 103)
  kopia-backup-ops/SKILL.md          Kopia → B2 backup verification & restore
```

## Deploying a skill change

```bash
# Per skill (preserves uid 1000:1000 ownership)
scp <category>/<skill>/SKILL.md homelab:/mnt/ssd-storage/apps/appdata/hermes/home/skills/<category>/<skill>/SKILL.md
ssh homelab "chown 1000:1000 /mnt/ssd-storage/apps/appdata/hermes/home/skills/<category>/<skill>/SKILL.md"

# Or sync the whole tree (CAUTION: deletes server-side files not in repo)
# rsync -av --delete smart-home/ devops/ homelab:/mnt/ssd-storage/apps/appdata/hermes/home/skills/
```

Skills are read off disk by hermes-workspace at the start of each chat — no restart needed.

## Authoring conventions

Per `software-development/hermes-agent-skill-authoring` (upstream skill, ships with hermes-agent):

- Frontmatter `name`, `description` (≤1024 chars) required. Validator checks for them.
- Aim for 8–14k chars per SKILL.md. Past 20k, split into `references/*.md`.
- Sections: Overview → When to Use → Don't Use For → topic sections → Pitfalls → Quick Reference.

## Workflow

1. Edit a skill locally in this repo.
2. Commit & push both remotes: `git push origin && git push github`.
3. Deploy with the scp+chown one-liner above.
4. Test in a fresh Hermes chat (no restart needed).

Never edit the live server copy without mirroring back to this repo — drift is silent and you'll lose the change on next deploy.

---
name: wireguard-headscale-ops
description: Use when managing the cross-site VPN — direct WireGuard between Austin↔HSB OPNsense firewalls, and Headscale (oci-headscale, 172.30.0.0/24) for remote/mobile clients. Covers adding peers, advertising subnet routes, debugging connectivity, and the split-brain risk of two parallel mesh systems.
version: 1.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [homelab, wireguard, headscale, vpn, networking, oci]
    related_skills: [opnsense-ops, homelab-ops]
---

# Cross-Site VPN: WireGuard + Headscale

## Overview

Two VPN systems run in parallel:

1. **Site-to-site WireGuard** — direct tunnel between Austin OPNsense (192.168.86.1) and HSB OPNsense (192.168.68.1). Carries inter-site LAN traffic. Managed via the OPNsense `wireguard_manage` MCP on each firewall.
2. **Headscale tailnet** — coordinator at `oci-headscale` (Oracle Cloud, 129.146.52.207). Mesh on `172.30.0.0/24`. Used for remote clients (phone, laptops outside the homes) and for hosts that need a stable always-on identity (`homelab`, `proxmox-hsb`, `oci-server`).

These are independent. A host can be on neither, one, or both. **Austin↔HSB traffic uses the direct WG tunnel, not Headscale**, even though both firewalls are also Headscale-aware. Headscale's job is remote access *to* the homelab, not between the homelabs.

## When to Use

- Add a new Headscale node (preauth key, register, advertise routes)
- Add a new site-to-site WG peer (rare — usually only for a new site)
- Diagnose "I can ping from my phone but not from my laptop"
- Audit who's in the tailnet
- Rotate a Headscale auth key
- Move a host off ephemeral expiry

## Don't Use For

- DNS resolution for tailnet names — Headscale's MagicDNS handles that, but if it's broken, that's a Headscale config issue (look at `/home/ubuntu/headscale/config.yaml`).
- OPNsense firewall rules permitting WG-tunnel traffic — see `opnsense-ops`.

## Headscale (Remote Clients) — How It Works Here

- Coordinator container `headscale` runs via docker compose at `/home/ubuntu/headscale/` on `oci-headscale`.
- Behind Traefik (`headscale.planetixian.com` is implied; check the compose for the exact route).
- Single user: `grt`.
- Tailnet: `172.30.0.0/24` (IPv4) + `fd7a:115c:a1e0::/48` (IPv6).
- Current nodes (from `docker exec headscale headscale nodes list`):

| ID | Host | Tailnet IP | Role |
|---|---|---|---|
| 4 | oci-server | 172.30.0.4 | OCI hub presence |
| 5 | proxmox-hsb | 172.30.0.5 | HSB site presence |
| 7 | homelab | 172.30.0.7 | Austin Docker host |
| (others) | mobile, opnsense legacy, etc. | — | various |

### Add a new node

```bash
# On oci-headscale: mint a preauth key (24h, single-use, reusable=false)
ssh oci-headscale 'sudo docker exec headscale headscale preauthkeys create --user grt --expiration 24h'

# On the new host (Linux example with Tailscale client):
sudo tailscale up --login-server https://<headscale-url> --authkey <key>

# Verify
ssh oci-headscale 'sudo docker exec headscale headscale nodes list'
```

For nodes that should advertise their LAN to the rest of the tailnet (a "subnet router"):

```bash
# On the node:
sudo tailscale up --login-server <...> --authkey <key> \
  --advertise-routes=192.168.86.0/24
# On oci-headscale: approve the route
sudo docker exec headscale headscale routes list
sudo docker exec headscale headscale routes enable -r <route-id>
```

### Rotate / expire a node

```bash
ssh oci-headscale 'sudo docker exec headscale headscale nodes expire -i <node-id>'
# To delete entirely:
ssh oci-headscale 'sudo docker exec headscale headscale nodes delete -i <node-id>'
```

### Magic DNS / search domain

Configured in `/home/ubuntu/headscale/config.yaml` under `dns:`. If MagicDNS hostnames stop resolving on clients, restart the client first (it caches), then check the server config.

## Site-to-Site WireGuard (Austin ↔ HSB)

This is the data plane for cross-site app traffic — backups, monitoring, file sync. Managed entirely through the OPNsense web UI / MCP.

### Inspect peers

```
opnsense-austin: wireguard_manage action=list_peers
opnsense-hsb:    wireguard_manage action=list_peers
```

There should be exactly one peer on each side pointing at the other firewall's public WAN endpoint with the matching public keys.

### Add a third site

1. Generate a keypair (use the MCP's `wireguard_manage` action=generate_keypair, or `wg genkey | tee priv | wg pubkey`).
2. On each existing endpoint: add the new site as a peer with the new public key and AllowedIPs covering its LAN.
3. On the new site: configure the interface, add both existing peers.
4. Add firewall rules on each side permitting the appropriate traffic through the WG interface (default OPNsense rules don't auto-permit).
5. Apply on every box. `wireguard_manage` action=apply, then `firewall_manage` action=apply.
6. Verify with a ping across the new path.

### Don't double-route

If a host is *also* in Headscale and advertises its LAN there, you can end up with two paths for the same traffic. WG site-to-site should win because both firewalls have static routes for the peer LAN. Confirm:

```bash
ssh homelab "ip route get 192.168.68.10"   # should go via the WG tunnel, NOT 172.30.x
```

If routing prefers Headscale, the OPNsense routing for `192.168.68.0/22` is missing or has lower priority.

## Common Diagnostics

```bash
# From any tailnet client:
tailscale status                        # who's up
tailscale ping <hostname>               # direct or DERP-relayed
tailscale netcheck                      # NAT/STUN report

# On Headscale itself:
ssh oci-headscale 'sudo docker exec headscale headscale nodes list'
ssh oci-headscale 'sudo docker logs headscale --tail 100'

# On OPNsense (via MCP):
# wireguard_manage action=show_status   → handshake age, transfer counters
# diagnostics_manage action=list_states filter=<peer-ip>
```

A handshake older than 3 minutes on a busy tunnel means a problem (NAT timeout, key mismatch, firewall blocking). Healthy: handshake updates every 2 minutes when there's traffic.

## ACLs (Headscale)

ACLs live in `/home/ubuntu/headscale/config.yaml` under the `acl_policy_path` (or inline policy file). Defaults are often permissive between same-user nodes — fine for a single-user tailnet, but worth tightening if you ever add another user.

Reload after edit:

```bash
ssh oci-headscale 'sudo docker exec headscale headscale policy reload'
```

## Pitfalls

### Two systems, one IP space confusion
A node's Tailscale IP (172.30.0.x) is NOT the same as its LAN IP. When the user says "the Austin host is at 192.168.86.90", they mean the LAN side. Inside the tailnet it's 172.30.0.7. Read carefully.

### Subnet route not approved
You ran `tailscale up --advertise-routes=...` but didn't run `headscale routes enable -r <id>` afterward. The node thinks it's a subnet router; the coordinator doesn't broadcast the routes. Symptom: ping by tailnet IP works, by LAN IP fails.

### Ephemeral expiry
Some old preauth keys default to ephemeral; the node disappears after disconnect. Use `--expiration 24h` for the *key* but ensure the node itself isn't created ephemeral. Re-add with `headscale nodes register` if it drops.

### CGNAT and Pangolin
Austin sits behind CGNAT. Direct inbound WG from the internet won't work — that's why Pangolin (OCI tunnel) exists for inbound HTTPS to Austin. WG site-to-site uses the HSB side as the initiator since HSB has a real public IP.

### MagicDNS cache
A client renamed in Headscale takes a while to propagate. If `ping new-name` fails, restart tailscaled on the client.

### Firewall rules vs WG state
A WG handshake can succeed while traffic over the tunnel is blocked by OPNsense rules on the WG interface. Always test with `ping` from a host actually behind the firewall, not from the OPNsense shell.

### Don't push compose for headscale via cat | ssh blindly
The `/home/ubuntu/headscale/` directory has secrets (private keys, DERP keys). Treat it like the Austin secrets dir — never commit them. If the directory ever needs to be versioned, scrub secrets out first.

### `oci-headscale` is also behind Traefik+CrowdSec
The OCI box runs its own Traefik + CrowdSec (different from Austin's). Failed Headscale auth attempts get banned by CrowdSec there. If your phone suddenly can't reach Headscale, check the OCI Traefik logs.

## Quick Reference

```
Coordinator SSH:     ssh oci-headscale
Headscale CLI:       sudo docker exec headscale headscale <verb>
Tailnet IPv4:        172.30.0.0/24
Single user:         grt
Nodes list:          headscale nodes list
Preauth key:         headscale preauthkeys create --user grt --expiration 24h
Routes approve:      headscale routes enable -r <id>
Policy reload:       headscale policy reload

Site-to-site WG:     via opnsense-austin / opnsense-hsb MCPs
                     wireguard_manage action=list_peers/add_peer/apply
                     firewall_manage rules required for tunnel traffic

NAT exception:       Austin behind CGNAT → inbound via Pangolin (oci-pangolin)
                     HSB has real public IP → initiates site-to-site WG
```

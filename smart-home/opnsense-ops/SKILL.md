---
name: opnsense-ops
description: Use when changing firewall rules, NAT, aliases, or port forwards on either OPNsense (Austin 192.168.86.1 or HSB 192.168.68.1). Drives the opnsense-austin and opnsense-hsb MCP servers with a safe rule-change workflow and post-change verification.
version: 1.0.0
author: Geoff
license: MIT
metadata:
  hermes:
    tags: [homelab, opnsense, firewall, networking, austin, hsb]
    related_skills: [homelab-ops, wireguard-headscale-ops]
---

# OPNsense Operations (Austin + HSB)

## Overview

Two OPNsense firewalls front the homelab — `opnsense-austin` (192.168.86.1) and `opnsense-hsb` (192.168.68.1). Each is wired into Hermes via a dedicated MCP server (`mcp__opnsense-austin__*` and `mcp__opnsense-hsb__*`). Both expose the same surface area: firewall, NAT, aliases, interfaces, DHCP, Unbound, WireGuard, IDS, routing, etc.

The MCPs talk straight to the OPNsense API. **Every rule/alias write is a real change.** OPNsense uses a stage-then-apply model: changes are saved to the running config but don't take effect until you `firewall.apply` (or the relevant `*_apply` for the subsystem). Forget to apply and you'll think a rule is live when it isn't. Apply without verifying and you can lock yourself out.

## When to Use

- Adding/editing/removing firewall rules or NAT entries
- Managing aliases (host/network/port groups used by rules)
- Opening or closing a port forward
- Auditing what's exposed externally
- Inspecting interface state, routes, IPsec, WireGuard, or diagnostics
- Comparing rule sets between Austin and HSB

## Don't Use For

- Changing the LAN subnet, DNS resolver behavior, or routing topology — those have cascading effects beyond a rule add; plan first, ask user, do via web UI
- Firmware upgrades — use `firmware_manage` only after confirming a maintenance window
- Restoring a config — manual via web UI, with the user watching

## Two Firewalls, Two MCPs — Always Confirm Which

Both MCPs share verb names. The only difference is the prefix:

| Site | MCP prefix | LAN | Mgmt IP |
|---|---|---|---|
| Austin | `mcp__opnsense-austin__` | 192.168.86.0/24 | 192.168.86.1 |
| HSB | `mcp__opnsense-hsb__` | 192.168.68.0/22 | 192.168.68.1 |

**Before any write op, state which firewall in plain English** ("Adding a rule on Austin's WAN_PUBLIC interface…"). It is trivially easy to call the wrong MCP. The HSB net mask is `/22`, not `/24` — keep that in mind for aliases.

## Core Workflow: Add a Firewall Rule

1. **Read first.** `firewall_manage` with action=list (filter by interface) to see existing rules. Identify the right interface and a sensible position.
2. **Stage the change.** `firewall_manage` with action=add. Note that OPNsense rules use UUIDs; the returned UUID is what you address for later edits.
3. **Apply.** `firewall_manage action=apply` — this is the step people forget. Until you apply, the rule lives only in `config.xml`'s pending state.
4. **Verify.** Re-list with a filter that includes the new rule. Spot-check evaluation order. If the rule depends on an alias, ensure the alias was applied too (`firewall_manage action=apply` covers aliases too on most OPNsense versions, but `firewall_manage` may need a separate alias apply on older).
5. **Smoke test from outside.** From a host on the side the rule allows/blocks, attempt the actual traffic. Don't trust "the rule is in the list" as proof it works.

## Aliases: The Right Default

Almost every new rule should reference an alias, not a literal IP/port. Aliases are reusable, named, and far easier to audit later. Patterns:

- **Host alias** for any device referenced more than once (e.g., `homelab_host` = 192.168.86.90)
- **Network alias** for subnets (`hsb_lan` = 192.168.68.0/22, `headscale_net` = 172.30.0.0/24)
- **Port alias** for service port groups (`media_ports` = 32400, 8096, 7878, 8989)

Before adding a literal, search aliases (`firewall_manage` with action=list_aliases) — there's likely one already.

## NAT / Port Forwards

Port forwards are NAT rules with auto-added firewall pass rules. Workflow:

1. Confirm the inside host has a static DHCP lease (`dhcpv4_manage` or `kea_manage` — Austin uses Kea, HSB uses ISC) — otherwise the forward breaks on IP renewal.
2. Add the NAT rule (`firewall_manage` action=add_nat).
3. Apply.
4. Verify with `curl -v --connect-to <pubip>:<port>` from off-network.

**Default to NOT exposing services.** Almost everything goes through Traefik on Austin (HTTPS only via 443) which is itself behind a single NAT rule. New external exposure should be questioned: can it go through Traefik instead? Through Pangolin (OCI tunnel) for CGNAT scenarios? Direct port forwards bypass CrowdSec and Authelia.

## Cross-Reference: Headscale Subnet Routes

When a host advertises subnet routes via Headscale (e.g., Austin's `homelab` node advertises `192.168.86.0/24` to remote clients), the OPNsense routing table doesn't need to know — Headscale handles it inside the tailnet. But if you want Austin LAN devices to *reach* a Headscale subnet (e.g., HSB hosts via Austin → Tailscale), you need a static route on OPNsense pointing the `172.30.0.0/24` net to the host running tailscaled. See related skill `wireguard-headscale-ops`.

## Pitfalls

### Forgot to apply
Rule shows up in `list`, but `pf` still uses the old set. Symptom: traffic behaves like the rule doesn't exist. Always run `*_apply` after writes. The MCPs do not auto-apply.

### Wrong interface
"LAN" on Austin is `igb0` (or whatever your hardware shows). Easy to add a rule on `WAN` thinking it's `LAN`. Read the rule back and check the `interface` field.

### Floating rules vs interface rules
Floating rules apply across multiple interfaces with `quick` semantics. Mixing floating + interface rules in head-scratching order produces ghosts. Stick to interface rules unless you specifically need floating.

### Anti-lockout
Don't disable the anti-lockout rule on the management interface without a serial/console fallback. If you brick remote access, you're driving to Austin.

### Both firewalls drift
Austin and HSB are managed independently; they don't sync. If you add an outbound NAT rule on Austin, the matching one on HSB is *not* added automatically. When the user says "block X everywhere," apply twice and verify twice.

### Aliases vs rules apply
Most current OPNsense versions apply aliases as part of `firewall_apply`. On some older versions (or for `alias_util`), a separate alias reload is needed. If a new alias isn't matching, force an alias reload via `firewall_manage` (action=reconfigure_alias or equivalent — check the MCP's tool list).

## Useful MCP Verbs (quick map)

| Task | MCP verb pattern |
|---|---|
| List/add/edit firewall rule | `firewall_manage` action=list/add/edit/delete/apply |
| Aliases | `firewall_manage` action=list_aliases/add_alias |
| NAT / port forward | `firewall_manage` action=list_nat/add_nat |
| Interface state | `interfaces_manage` action=list/status |
| DHCP leases & static maps | `kea_manage` (Austin), `dhcpv4_manage` (HSB if still ISC) |
| DNS overrides | `unbound_manage` (host_overrides, domain_overrides) |
| WireGuard peers | `wireguard_manage` action=list_peers/add_peer/apply |
| Routes | `routes_manage` action=list/add/apply |
| Diagnostics (states, ARP, etc.) | `diagnostics_manage` |
| IDS rules/alerts | `ids_manage` |
| Cron-style scheduled tasks | `cron_manage` |
| System health, firmware status | `core_manage`, `firmware_manage` |

## Verification Snippets

```bash
# From an Austin LAN host, test a port-forwarded service externally
curl -v --connect-to <name>.maximumgamer.com:443:<wan-ip>:443 https://<name>.maximumgamer.com/

# Check a state for traffic actually traversing the firewall
# (via diagnostics MCP) — diagnostics_manage action=list_states filter=<pattern>

# Confirm rule order on an interface
# firewall_manage action=list interface=lan
```

## Workflow When Things Break

1. Read the rule order on the affected interface — first match wins (modulo `quick`).
2. Check states table for an existing state matching old rules (`diagnostics_manage`). Old states survive rule changes until they expire or you flush.
3. If a recent change is suspect: `firewall_manage action=apply` again, then re-test. If still broken, revert via the OPNsense web UI's config history — don't try to reverse-engineer through the MCP.
4. Last resort: console access via IPMI/serial. Don't push more rules trying to "fix" a brick.

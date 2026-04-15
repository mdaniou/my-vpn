# CLAUDE.md

See `README.md` for repo structure.

## Project

Self-hosted WireGuard VPN. RPi5 (Alpine Linux, diskless) in a DMZ behind pfSense.

## Docs

14 standalone Markdown guides in `docs/`, written one at a time in order (01 → 14). Each covers one component with exact commands, inline-commented configs, verification steps, and security rationale.

## Security principles (enforce in every doc)

1. Default deny everywhere
2. Least privilege per WireGuard peer
3. PresharedKey on every peer (post-quantum layer)
4. IPv6 disabled
5. Split tunnel only
6. SSH to RPi5 from MGMT VLAN only
7. DMZ → LAN rules enforced by pfSense (compromised gateway cannot reach LAN directly)

## Style

- Use clear placeholders for environment-specific values (e.g. `<WAN_IP>`, `<PEER_PUBKEY>`)
- Do not skip ahead or batch multiple docs unless explicitly asked

my-vpn main ? ❯ cat CLAUDE.md
# Project: Secure Homelab WireGuard VPN

## Goal

Build a fully self-hosted, zero-third-party-dependency WireGuard VPN infrastructure for a homelab, with **maximum security** as the primary design goal at every layer.

This repository contains step-by-step documentation (Markdown files) for deploying and hardening every component of the setup. Each document covers one component and must be self-contained, actionable, and security-focused.

## Architecture Overview

```
Internet
    │
    ▼
┌──────────────────────────────────────────────────┐
│                    pfSense                       │
│                                                  │
│  Port 1: WAN ──────── Internet uplink            │
│  Port 2: LAN ──────── Homelab devices            │
│  Port 3: DMZ ──────── RPi5 WireGuard gateway     │
│  Port 4: MGMT ─────── Admin access only          │
└──────────────────────────────────────────────────┘
```

**Traffic flow (remote access):**

```
Remote client
  → WAN (UDP 51820 only)
    → pfSense (forwards to DMZ only)
      → RPi5 WireGuard gateway (Alpine Linux)
        → pfSense (firewall rules, DMZ→LAN)
          → LAN devices (only allowed IP:port pairs)
```

The WireGuard gateway sits in an isolated DMZ so that even if it is compromised, the attacker is contained — pfSense still enforces all DMZ→LAN rules.

## Hardware Inventory

| Device             | Role                              | Network Zone |
|--------------------|-----------------------------------|--------------|
| pfSense (4-port)   | Firewall, router, DHCP, DNS       | All zones    |
| RPi5 #1            | WireGuard VPN gateway             | DMZ          |
| RPi5 #2            | Monitoring / IDS (future)         | LAN or DMZ   |
| Proxmox host       | VMs, containers, lab workloads    | LAN          |
| Mac Mini 2012 (×3) | Lab workloads                     | LAN          |
| RPi3B              | Pi-hole DNS / syslog collector    | LAN          |

## Network Zones

- **WAN** (pfSense port 1): Internet-facing. Only UDP 51820 allowed inbound, forwarded to DMZ.
- **DMZ** (pfSense port 3): Isolated zone for the WireGuard gateway RPi5. No direct LAN access except through pfSense firewall rules.
- **LAN** (pfSense port 2): Homelab devices. No inbound from WAN or DMZ unless explicitly allowed.
- **MGMT** (pfSense port 4): Management VLAN. Only zone allowed to SSH into pfSense and the RPi5. Accessible from a single hardened admin device.

## Security Principles (Apply to Every Document)

1. **Default deny** — Every firewall, every iptables ruleset, every config starts with "deny all" and adds explicit allows.
2. **Least privilege** — Each device, user, and VPN peer gets access only to the specific IP:port pairs it needs. No broad subnet access.
3. **Defense in depth** — Security does not rely on a single layer. The WireGuard gateway has its own iptables rules even though pfSense also filters traffic. Both must be compromised for an attacker to reach LAN.
4. **Minimal attack surface** — The RPi5 runs Alpine Linux in diskless mode with only the packages strictly required for WireGuard. No GUI, no unnecessary services, no extra software.
5. **No third-party dependencies** — No Tailscale, no cloud DNS, no external services. Static IP from ISP (or fallback DDNS on own domain). Everything is self-hosted and self-managed.
6. **Cryptographic hardening** — WireGuard with PresharedKey on every peer (post-quantum resistant layer). Keys generated locally, never transmitted over the network. Regular key rotation schedule.
7. **Auditability** — All configs are documented, all firewall rules are logged, all access is traceable.

## WAN IP Strategy

- **Primary**: Static IPv4 from ISP (Salt Switzerland) — eliminates DNS dependency entirely.
- **Fallback**: If static IP is unavailable, self-managed DDNS via own domain + pfSense built-in DDNS client (Cloudflare with scoped API token as minimal-trust option).

## Documentation Structure

Each document is a standalone guide for one component. They should be followed in order but remain independently readable.

```
docs/
├── 01-network-architecture.md     # Network topology, zones, IP plan, VLANs
├── 02-pfsense-base.md             # pfSense initial setup, interfaces, zones
├── 03-pfsense-firewall-rules.md   # All firewall rules: WAN, DMZ, LAN, MGMT
├── 04-pfsense-services.md         # DHCP, DNS (Unbound), NTP, logging, DDNS
├── 05-rpi5-alpine-install.md      # Alpine Linux install on RPi5 (diskless mode)
├── 06-rpi5-hardening.md           # OS hardening: sysctl, SSH, services, users
├── 07-wireguard-server.md         # WireGuard config on RPi5: interface, peers, keys
├── 08-rpi5-iptables.md            # Local firewall on RPi5 (defense in depth)
├── 09-wireguard-clients.md        # Client configs: laptop, phone, key management
├── 10-key-management.md           # Key generation, rotation schedule, revocation
├── 11-monitoring-logging.md       # Syslog, log shipping to RPi3B, alerting
├── 12-salt-router-config.md       # Salt Fiber Box: DMZ mode, port forwarding to pfSense
├── 13-maintenance-runbook.md      # Updates, key rotation, backup, recovery procedures
└── 14-security-checklist.md       # Final audit checklist to validate the full setup
```

## Document Format Requirements

Each markdown file must follow this structure:

```markdown
# <Component Name>

## Overview
Brief description of what this document covers and its role in the architecture.

## Prerequisites
What must be completed before this step (reference other docs by filename).

## Security Objectives
What specific security goals this component achieves.

## Steps
Numbered, actionable steps with exact commands and config blocks.
Every config block must include comments explaining security-relevant choices.

## Verification
How to verify the component is working correctly and securely.
Include specific test commands and expected outputs.

## Troubleshooting
Common issues and how to resolve them without compromising security.

## Security Notes
Explicit warnings about what NOT to do and why.
Document any trade-offs made and their security implications.
```

## Key Technical Decisions

- **OS for WireGuard gateway**: Alpine Linux (diskless mode, minimal packages, hardened kernel, ~5 running processes total)
- **WireGuard runs on RPi5, NOT on pfSense**: Isolation — if WireGuard is compromised, attacker is trapped in DMZ, not on the firewall itself
- **PresharedKey on every WireGuard peer**: Adds symmetric encryption layer on top of Curve25519 (post-quantum defense)
- **SSH to RPi5 only from MGMT VLAN**: Key-only auth, no password, restricted source IPs
- **IPv6 disabled on all interfaces**: Reduces attack surface, avoids accidental global addressability of LAN devices
- **Split tunnel only**: VPN clients route only homelab traffic through the tunnel, not all internet traffic

## Interaction Model

We will build this documentation **one file at a time, in order**. For each document:

1. I will tell you which document to write next.
2. You write the full document following the format above.
3. I will review and request changes if needed.
4. Once approved, we move to the next document.

Do not skip ahead or write multiple documents at once unless I ask.

## Tone and Style

- Write for a technically competent homelab operator (comfortable with CLI, networking, Linux).
- Be precise — use exact commands, exact config file paths, exact syntax.
- Explain the *why* behind every security decision, not just the *what*.
- Do not add unnecessary commentary or filler. Every sentence should be actionable or informative.
- Use code blocks for all commands and config files. Annotate configs with inline comments.
- When a config value is environment-specific (IP addresses, keys, hostnames), use clear placeholders like `<WAN_STATIC_IP>`, `<DMZ_SUBNET>`, `<PEER_PUBLIC_KEY>` and document what each placeholder means.


## more info

See `README.md` for repo structure.

## Project

Self-hosted WireGuard VPN. RPi5 (Alpine Linux, diskless) in a DMZ behind pfSense.

## Docs

14 standalone Markdown guides in `docs/`, written one at a time in order (01 → 14). Each covers one component with exact commands, inline-commented configs, verification steps, and security rationale.

## Security principles (enforce in every doc)

1. Default deny everywhere
2. Least privilege per WireGuard peer
3. PresharedKey on every peer (post-quantum layer)
4. IPv6 disabled
5. Split tunnel only
6. SSH to RPi5 from MGMT VLAN only
7. DMZ → LAN rules enforced by pfSense (compromised gateway cannot reach LAN directly)

## Style

- Use clear placeholders for environment-specific values (e.g. `<WAN_IP>`, `<PEER_PUBKEY>`)
- Do not skip ahead or batch multiple docs unless explicitly asked

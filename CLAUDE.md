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

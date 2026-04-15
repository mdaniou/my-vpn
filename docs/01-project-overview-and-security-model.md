# Project Overview and Security Model

## Overview

This document defines the scope, architecture, and security model for a fully self-hosted WireGuard VPN infrastructure built on personal homelab hardware. It is the foundation all subsequent documents build on — read it before any other guide.

The goal is remote access to homelab resources with **maximum security** and **zero external dependencies**: no Tailscale, no cloud relays, no third-party DNS services. Every component is self-hosted and self-managed.

## Prerequisites

None. This is the first document.

## Security Objectives

This project achieves the following security guarantees at the system level:

1. **Contained compromise** — If the WireGuard gateway is fully compromised, the attacker is trapped in the DMZ. pfSense continues to enforce all DMZ→LAN rules; no lateral movement to the LAN is possible without also compromising pfSense.
2. **No implicit trust** — Every network boundary requires explicit allow rules. Absence of a rule means denial.
3. **No third-party exposure** — No traffic metadata, keys, or credentials ever touch a cloud service. All cryptographic material is generated and stored locally.
4. **Minimal footprint** — The WireGuard host runs only the packages strictly necessary for VPN operation. Smaller codebase = smaller attack surface.
5. **Auditability** — Every firewall rule, every peer, every config file is documented with its security rationale.

---

## Architecture

### Physical Topology

```
Internet (IPv4 only)
    │
    ▼
┌──────────────────────────────────────────────────────┐
│                      pfSense                         │
│                                                      │
│  em0  WAN  ───── Internet uplink (static IPv4)       │
│  em1  LAN  ───── Homelab devices (LAN zone)          │
│  em2  DMZ  ───── WireGuard gateway RPi5 (DMZ zone)   │
│  em3  MGMT ───── Admin device only (MGMT zone)       │
└──────────────────────────────────────────────────────┘
         │                     │
         ▼                     ▼
   RPi5 #1                LAN devices
   WireGuard gateway      (Proxmox, Mac Minis,
   (DMZ zone)              RPi3B Pi-hole, etc.)
```

### Traffic Flow — Remote Access

```
Remote client (phone / laptop)
  └─► WAN UDP 51820 (only port open inbound)
        └─► pfSense NAT + WAN rule: forward to RPi5 DMZ IP
              └─► RPi5 WireGuard (decrypts, validates peer)
                    └─► pfSense DMZ→LAN rules (whitelist by IP:port)
                          └─► LAN resource (only explicitly allowed pairs)
```

Every hop is a hard enforcement boundary, not advisory routing. Bypassing any one hop does not grant access — the next hop still enforces independently.

---

## Hardware Inventory

| Device            | Role                                    | Zone        |
|-------------------|-----------------------------------------|-------------|
| pfSense (4-port)  | Firewall, router, NAT, DHCP, DNS        | All zones   |
| RPi5 #1           | WireGuard VPN gateway (Alpine, diskless) | DMZ         |
| RPi5 #2           | Monitoring / IDS (future scope)         | LAN or DMZ  |
| Proxmox host      | VMs and containers                      | LAN         |
| Mac Mini 2012 ×3  | Lab workloads                           | LAN         |
| RPi3B             | Pi-hole DNS + syslog collector          | LAN         |

---

## Network Zones

### WAN

- **Purpose**: Internet-facing uplink.
- **Inbound policy**: Block all except UDP 51820 (WireGuard), forwarded to RPi5 DMZ IP via NAT.
- **Outbound policy**: Permit (pfSense handles stateful tracking).
- **IP**: Static IPv4 from ISP (Salt Switzerland). No IPv6 — disabled at all layers.

### DMZ

- **Purpose**: Isolation zone for the WireGuard gateway.
- **Inbound from WAN**: Only forwarded WireGuard packets (post-NAT).
- **Outbound to LAN**: Only explicitly allowed IP:port pairs via pfSense DMZ→LAN rules.
- **Outbound to WAN**: Blocked except for DNS queries to pfSense (for hostname resolution if needed) and NTP.
- **Key constraint**: No route from DMZ to LAN except through pfSense firewall rules.

### LAN

- **Purpose**: Homelab devices.
- **Inbound from DMZ**: Only whitelisted destination IP:port pairs.
- **Inbound from WAN**: None. All WAN traffic terminates at the WireGuard gateway in DMZ.
- **Inbound from MGMT**: Admin access (SSH to managed devices).

### MGMT

- **Purpose**: Out-of-band management access to pfSense and RPi5.
- **Hosts**: Single hardened admin device.
- **Privilege**: Only zone permitted to SSH into pfSense and RPi5.
- **Outbound to DMZ**: SSH only (port 22, source-restricted to MGMT subnet).
- **Outbound to LAN**: Admin access to LAN devices.

---

## Security Principles

These principles apply to every document in this series. Deviations require explicit justification.

### 1. Default Deny

Every firewall, every iptables ruleset, every network boundary starts with an implicit or explicit deny-all. Rules are additive allowlists, never blocklists.

*Rationale*: A misconfigured allowlist leaves one service exposed. A misconfigured blocklist leaves everything exposed.

### 2. Least Privilege

Each WireGuard peer has an `AllowedIPs` list containing only the exact subnets or hosts it legitimately needs to reach. No peer receives broad LAN subnet access unless specifically required and documented.

*Rationale*: A stolen client config or compromised device can only reach what that specific peer was allowed to reach.

### 3. Defense in Depth

The WireGuard gateway runs its own iptables ruleset (doc 08) even though pfSense already filters DMZ traffic (doc 05). Both must be bypassed for an attacker to pivot to the LAN.

*Rationale*: No single control point is assumed infallible. Layered controls bound the impact of any one failure.

### 4. Minimal Attack Surface

The RPi5 runs Alpine Linux in diskless mode with no GUI, no unnecessary daemons, no package manager leftovers. Target: fewer than 10 listening sockets total on the WireGuard host.

*Rationale*: Every installed package and every open port is a potential vulnerability. Remove what is not needed.

### 5. No Third-Party Dependencies

No Tailscale, no Cloudflare Tunnel, no cloud key servers, no external services of any kind in the data path. The DDNS fallback (doc 04) uses pfSense's built-in client against a self-managed domain — no traffic leaves via a third-party relay.

*Rationale*: Third-party services introduce trust relationships outside your control. A compromised Tailscale or similar service can expose all peers.

### 6. Cryptographic Hardening

Every WireGuard peer pair uses a `PresharedKey` (32-byte symmetric key, generated locally). This adds a symmetric encryption layer on top of WireGuard's Curve25519 key exchange, providing a post-quantum defense against harvest-now/decrypt-later attacks.

*Rationale*: Curve25519 is not quantum-resistant. PresharedKey forces an attacker to break both the asymmetric handshake and the symmetric layer.

### 7. IPv6 Disabled

IPv6 is disabled at the kernel level on the RPi5 and suppressed in pfSense interfaces. No `AAAA` records are published. No IPv6 routing is configured.

*Rationale*: IPv6 can create unintended global addressability for LAN devices and bypasses IPv4-only firewall rules. Eliminating it removes an entire class of misconfiguration risk.

### 8. Split Tunnel Only

VPN clients route only homelab-bound traffic through the tunnel (`AllowedIPs` set to homelab subnets only). General internet traffic exits the client's local network directly.

*Rationale*: Full-tunnel VPN routes all client internet traffic through the homelab uplink, creating privacy exposure and unnecessary load. Split tunnel contains the VPN to its stated purpose.

### 9. SSH Restricted to MGMT VLAN

The RPi5 accepts SSH connections only from the MGMT subnet. pfSense enforces this at the DMZ firewall level. The RPi5 also enforces it via `sshd_config` (`ListenAddress` + `AllowUsers`). Key-only authentication — no passwords.

*Rationale*: SSH is the highest-privilege access vector to the gateway. Restricting it to a single, physically controlled VLAN eliminates remote brute-force and key-theft scenarios.

---

## WAN IP Strategy

| Method       | Mechanism                                      | When to use                          |
|--------------|------------------------------------------------|--------------------------------------|
| Static IPv4  | Direct IP in WireGuard client configs          | Primary — eliminates DNS dependency  |
| DDNS fallback| pfSense built-in DDNS → self-managed domain    | If ISP removes static IP assignment  |

The DDNS fallback uses a Cloudflare API token scoped to a single zone and record (minimum required permissions). pfSense updates the record directly — no third-party DDNS service.

---

## Key Technical Decisions

| Decision | Rationale |
|---|---|
| WireGuard on RPi5, not pfSense | Compromise of WireGuard process is contained to DMZ, not the firewall itself |
| Alpine Linux diskless | Minimizes installed packages, eliminates SD card wear, forces stateless config (easier to audit) |
| PresharedKey on every peer | Post-quantum layer; negligible performance cost |
| Static ISP IP primary | Removes DNS as a dependency and attack surface from the remote access path |
| iptables on RPi5 AND pfSense rules | Defense in depth; neither layer alone is sufficient |
| IPv6 fully disabled | Eliminates bypass vectors and unintended global exposure |

---

## Document Map

Follow the guides in order. Each document assumes all prior documents are complete.

| File | Component |
|---|---|
| `01-project-overview-and-security-model.md` | This document — scope, architecture, security model |
| `02-network-architecture-and-traffic-flow.md` | IP plan, subnets, VLANs, routing |
| `03-pfsense-dmz-setup.md` | pfSense interfaces, zones, basic routing |
| `04-pfsense-wan-rules-and-port-forwarding.md` | WAN firewall rules, NAT, port forward to RPi5 |
| `05-pfsense-dmz-to-lan-firewall-rules.md` | DMZ→LAN whitelist rules |
| `06-rpi5-hardware-and-alpine-diskless-install.md` | RPi5 hardware setup, Alpine diskless install |
| `07-alpine-hardening-and-network-config.md` | OS hardening, sysctl, package minimization |
| `08-wireguard-install-and-server-config.md` | WireGuard installation, server config, keys |
| `09-wireguard-peer-management.md` | Adding/removing peers, PresharedKey workflow |
| `10-wireguard-client-config.md` | Client configs for laptop, phone |
| `11-ssh-hardening-and-mgmt-vlan-access.md` | SSH config, key-only auth, MGMT restriction |
| `12-ipv6-disable-and-kernel-hardening.md` | IPv6 suppression, sysctl hardening on RPi5 |
| `13-verification-and-testing.md` | End-to-end test procedures, firewall validation |
| `14-maintenance-and-troubleshooting.md` | Key rotation, updates, recovery, runbook |

---

## Verification

This document is a reference, not a deployment step. Verification is done holistically in doc 13 after all components are deployed.

To confirm you understand the architecture before proceeding:

- Identify which physical pfSense port connects to the DMZ and its IP address.
- Confirm your ISP provides a static IPv4 (or prepare the DDNS fallback — see doc 04).
- Confirm the RPi5 that will serve as the WireGuard gateway is on the DMZ port.
- Confirm your admin device (for SSH access) is on the MGMT port or VLAN.

---

## Security Notes

**Do not put WireGuard on pfSense itself.** If the WireGuard process is exploited (e.g. via a malformed handshake), running it on pfSense gives the attacker direct control of the firewall. Running it on an isolated RPi5 in the DMZ limits the blast radius.

**Do not allow any DMZ→LAN rule with a broad destination.** Every DMZ→LAN rule must specify an exact destination IP and port. A rule permitting `any` destination in the LAN defeats the entire DMZ isolation model.

**Do not disable IPv6 only in userspace.** Disable it at the kernel level via `sysctl` (`net.ipv6.conf.all.disable_ipv6 = 1`) and verify with `ip -6 addr`. A userspace-only disable can be bypassed by applications that open raw sockets.

**Do not store private keys in version control.** All private keys and PresharedKeys generated during this setup must remain on the device that owns them. Never commit them to this repository or any backup that leaves your physical control.

# Network Architecture and Traffic Flow

## Overview

This document defines the complete network topology for the homelab WireGuard VPN: physical zones, IP addressing, VLAN assignments, and the exact path that VPN traffic takes from a remote client to a LAN device. Everything in subsequent documents maps back to this architecture.

## Prerequisites

- `01-project-overview-and-security-model.md` — security model and design goals.

## Security Objectives

- Isolate the WireGuard gateway in a DMZ so a compromised gateway cannot reach LAN directly.
- Enforce all DMZ→LAN access exclusively through pfSense firewall rules.
- Restrict management access to a dedicated MGMT zone reachable only from a single admin device.
- Disable IPv6 everywhere to eliminate accidental global addressability.
- Expose only UDP 51820 on the WAN interface — no other inbound port is open.

---

## Network Zones

Four physical interfaces on the pfSense firewall, each mapped to a dedicated zone:

| Zone | pfSense Port | Subnet | Purpose |
|------|-------------|--------|---------|
| WAN  | Port 1 | `<WAN_STATIC_IP>/32` (ISP) | Internet uplink |
| LAN  | Port 2 | `192.168.1.0/24` | Homelab devices |
| DMZ  | Port 3 | `192.168.2.0/24` | WireGuard gateway (RPi5) |
| MGMT | Port 4 | `192.168.4.0/24` | Admin access only |

**Placeholder reference:**

| Placeholder | Meaning |
|-------------|---------|
| `<WAN_STATIC_IP>` | Static IPv4 assigned by ISP (Salt Switzerland) |
| `<DMZ_GW>` | pfSense IP on the DMZ interface, e.g. `192.168.2.1` |
| `<RPI5_DMZ_IP>` | RPi5 static IP in the DMZ, e.g. `192.168.2.2` |
| `<WG_TUNNEL_SUBNET>` | WireGuard tunnel network, e.g. `10.0.0.0/24` |
| `<WG_SERVER_TUNNEL_IP>` | RPi5 WireGuard interface address, e.g. `10.0.0.1/24` |

---

## Physical Topology

```
Internet
    │
    ▼
┌──────────────────────────────────────────────────────┐
│                       pfSense                        │
│                                                      │
│  Port 1: WAN  ──── <WAN_STATIC_IP>  (Internet)       │
│  Port 2: LAN  ──── 192.168.1.1      (Homelab)        │
│  Port 3: DMZ  ──── <DMZ_GW>         (WG gateway)     │
│  Port 4: MGMT ──── 192.168.4.1      (Admin only)     │
└──────────────────────────────────────────────────────┘
          │                    │
          ▼                    ▼
   192.168.2.2           192.168.4.x
   RPi5 (WireGuard)      Admin workstation
   Alpine Linux
   diskless mode
```

---

## Device Inventory and IP Assignments

| Device | Zone | IP Address | Role |
|--------|------|-----------|------|
| pfSense | All | 192.168.1.1 / `<DMZ_GW>` / 192.168.4.1 | Firewall, router, DHCP, DNS |
| RPi5 #1 | DMZ | `<RPI5_DMZ_IP>` (static) | WireGuard VPN gateway |
| RPi5 #2 | LAN or DMZ | TBD | Monitoring / IDS (future) |
| Proxmox host | LAN | 192.168.1.x (static) | VMs, containers |
| Mac Mini ×3 | LAN | 192.168.1.x (DHCP reserved) | Lab workloads |
| RPi3B | LAN | 192.168.1.x (static) | Pi-hole DNS, syslog |
| Admin workstation | MGMT | 192.168.4.x (static) | Only device with SSH access to pfSense and RPi5 |

---

## WireGuard Tunnel Addressing

The WireGuard tunnel is a private overlay network, fully separate from all physical subnets:

| Endpoint | Tunnel IP |
|----------|-----------|
| RPi5 (server) | `<WG_SERVER_TUNNEL_IP>` (e.g. `10.0.0.1/24`) |
| Laptop peer | `10.0.0.2/32` |
| Phone peer | `10.0.0.3/32` |

Each peer is assigned a `/32` — no peer can claim another peer's address or route to other tunnel addresses unless explicitly configured. Split tunnel is enforced on the client: only `192.168.1.0/24` (LAN) and `10.0.0.0/24` (tunnel) are routed through the VPN.

---

## Traffic Flow: Remote Client → LAN Device

```
Remote client (laptop / phone)
  │
  │  UDP 51820 (WireGuard, encrypted)
  ▼
pfSense — WAN interface
  │  NAT + port forward: WAN:51820 → <RPI5_DMZ_IP>:51820
  ▼
RPi5 — DMZ interface (<RPI5_DMZ_IP>)
  │  WireGuard decrypts packet
  │  Source becomes peer tunnel IP (e.g. 10.0.0.2)
  ▼
RPi5 — WireGuard interface (wg0, <WG_SERVER_TUNNEL_IP>)
  │  iptables FORWARD: only allowed dst IP:port pairs pass
  │  MASQUERADE: source rewritten to <RPI5_DMZ_IP>
  ▼
pfSense — DMZ interface
  │  DMZ→LAN firewall rules: only explicit IP:port pairs allowed
  ▼
LAN device (e.g. Proxmox at 192.168.1.x:8006)
```

**Two independent enforcement layers:**

1. **RPi5 iptables** — forwards only to specific allowed destinations; drops everything else.
2. **pfSense DMZ→LAN rules** — independently enforces the same restrictions at the firewall level.

Both layers must be bypassed for an attacker to reach LAN from a compromised gateway. Neither layer trusts the other.

---

## Traffic Flow: LAN Device → Remote Client (Return)

```
LAN device
  ▼
pfSense — LAN→DMZ: return traffic to established sessions allowed
  ▼
RPi5 — DMZ interface
  ▼
RPi5 — wg0: WireGuard encrypts return packet
  ▼
pfSense — DMZ→WAN: outbound UDP 51820 allowed (stateful)
  ▼
Remote client
```

Return traffic is handled by stateful firewall tracking — no separate inbound LAN→DMZ rules are needed for established connections.

---

## Inter-Zone Communication Matrix

| Source | Destination | Allowed? | Protocol / Port |
|--------|------------|----------|----------------|
| WAN | DMZ (RPi5) | Yes | UDP 51820 only (port forward) |
| WAN | LAN | No | — |
| WAN | MGMT | No | — |
| DMZ | LAN | Restricted | Explicit IP:port pairs only (pfSense rules) |
| DMZ | WAN | Yes | Outbound only (return traffic) |
| DMZ | MGMT | No | — |
| LAN | DMZ | No (except return) | Stateful return only |
| LAN | WAN | Yes | Outbound (NAT) |
| MGMT | DMZ | Yes | TCP 22 (SSH to RPi5) |
| MGMT | LAN | Yes | Admin access |
| MGMT | pfSense | Yes | TCP 22, 443 (admin) |

Default deny: any traffic not in the above table is blocked at pfSense. The DMZ has no route to MGMT under any circumstances.

---

## IPv6 Policy

IPv6 is **disabled on all interfaces and devices**:

- pfSense: IPv6 set to "None" on all interfaces.
- RPi5: `net.ipv6.conf.all.disable_ipv6 = 1` in sysctl.
- WireGuard config: no IPv6 `AllowedIPs` entries.

Rationale: IPv6 link-local and global addresses would create additional attack surface and could accidentally expose LAN devices to global addressability. The complexity gain is zero for this use case.

---

## DDNS Fallback (if static IP unavailable)

Primary connection uses a static IPv4 from Salt Switzerland — no DNS lookup required, no third-party dependency.

If a static IP becomes unavailable, fallback is self-managed DDNS:
- Own domain, Cloudflare DNS with a scoped API token (Zone:DNS:Edit on the single record only).
- pfSense built-in DDNS client updates the A record on IP change.
- WireGuard clients use the hostname instead of `<WAN_STATIC_IP>`.

This fallback introduces a DNS dependency but still avoids any third-party VPN service.

---

## Verification

After completing pfSense setup (docs 03–05), verify zone isolation:

```bash
# From DMZ (RPi5) — must fail:
ping 192.168.1.1          # LAN gateway — blocked by pfSense DMZ→LAN rules
ping 192.168.4.1          # MGMT gateway — no route

# From LAN — must fail:
ping 192.168.2.2          # RPi5 DMZ IP — no inbound LAN→DMZ rule

# From MGMT workstation — must succeed:
ssh admin@192.168.2.2     # RPi5 via SSH

# From WAN (external):
# Only UDP 51820 should be reachable. All other probes must time out.
nmap -sU -p 51820 <WAN_STATIC_IP>   # expect: open
nmap -p 22,80,443 <WAN_STATIC_IP>   # expect: filtered
```

---

## Security Notes

- **Never place the RPi5 on the LAN interface.** The DMZ isolation is the primary containment boundary. If the gateway is on LAN, a compromise immediately exposes all LAN devices.
- **Never allow DMZ→MGMT traffic.** An attacker on the RPi5 must not be able to reach the admin workstation or pfSense management interface.
- **Do not use `any` as a source or destination in firewall rules.** Every rule in pfSense and iptables must name explicit IPs or subnets.
- **MGMT zone is not for general use.** Only the single hardened admin workstation belongs here. Do not attach other devices to the MGMT interface.
- **Static IP on RPi5 is mandatory.** The pfSense port forward targets `<RPI5_DMZ_IP>` directly. A DHCP-assigned address could change and silently break the VPN — or worse, forward traffic to the wrong host.

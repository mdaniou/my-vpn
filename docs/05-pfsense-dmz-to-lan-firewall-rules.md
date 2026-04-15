# pfSense DMZ to LAN Firewall Rules

## Overview

This document covers the firewall rules governing traffic flow from the DMZ (where the RPi5 WireGuard gateway lives) to the LAN (where homelab devices live). These rules are the second firewall layer that enforces least-privilege access for each VPN peer — even if the RPi5 is compromised, the attacker is contained within the DMZ by these rules.

The design principle: the RPi5 is trusted to handle WireGuard cryptography, but it is *not* trusted to decide what LAN resources VPN peers can access. That decision belongs exclusively to pfSense.

## Prerequisites

- `03-pfsense-dmz-setup.md` completed: DMZ interface configured, RPi5 at `<DMZ_RPi5_IP>`
- `04-pfsense-wan-rules-and-port-forwarding.md` completed: WAN rules in place
- `07-wireguard-install-and-server-config.md` completed (or at least planned): know the tunnel IPs assigned to each peer
  - WireGuard tunnel subnet: `10.0.0.0/24`
  - RPi5 (server): `10.0.0.1`
  - Laptop peer: `10.0.0.2`
  - Phone peer: `10.0.0.3`
- Know the LAN IPs of resources each peer needs to access:
  - Proxmox host: `<PROXMOX_IP>` (e.g. 192.168.1.10)
  - Pi-hole DNS: `<PIHOLE_IP>` (e.g. 192.168.1.2)
  - Mac Minis: `<MACMINI_IP_1>`, `<MACMINI_IP_2>`, `<MACMINI_IP_3>`

## Security Objectives

- Enforce least-privilege access: each VPN peer may reach only the specific LAN IP:port pairs it legitimately needs
- Block all DMZ → MGMT traffic (prevents lateral movement to the management network)
- Block peer-to-peer traffic within the WireGuard tunnel (peers cannot reach each other unless explicitly allowed)
- Stateful firewall handles return traffic automatically — no explicit allow rules needed for established connections
- Log all blocked DMZ traffic for audit and incident detection
- Default deny as the final rule: anything not explicitly allowed is dropped

## Steps

### 1. Understand pfSense Rule Evaluation Order

pfSense evaluates firewall rules **top to bottom, first match wins**. Order is critical:

- Protective block rules (DMZ → MGMT, peer-to-peer) must come **before** any allow rules
- The default block-all rule must be **last** (lowest priority)
- A misconfigured rule order can inadvertently allow traffic that should be blocked

All rules in this section are added under **Firewall > Rules > DMZ**.

### 2. Create an Alias for the WireGuard Tunnel Subnet

Using aliases avoids repeating `10.0.0.0/24` across many rules and makes changes easier.

Navigate to **Firewall > Aliases > IP**, click **Add**:

| Field | Value |
|-------|-------|
| Name | `WG_TUNNEL` |
| Type | `Network` |
| Network | `10.0.0.0/24` |
| Description | `WireGuard tunnel addresses` |

Also create per-peer aliases for clean rule descriptions:

| Name | Type | Value | Description |
|------|------|-------|-------------|
| `WG_LAPTOP` | Host | `10.0.0.2` | WireGuard laptop peer |
| `WG_PHONE` | Host | `10.0.0.3` | WireGuard phone peer |

Click **Save**, then **Apply Changes**.

### 3. Add Block Rule: DMZ → MGMT (Rule 1 — Highest Priority)

This rule prevents a compromised RPi5 from reaching the management network (where pfSense admin UI and admin workstation live).

Navigate to **Firewall > Rules > DMZ**, click **Add** (up arrow — adds to top):

| Field | Value | Reason |
|-------|-------|--------|
| Action | `Block` | Drop, do not reject (no response reveals less) |
| Interface | `DMZ` | |
| Protocol | `Any` | Block all protocols, not just TCP/UDP |
| Source | `DMZ subnets` | Anything originating from DMZ |
| Destination | `<MGMT_SUBNET>` (e.g. 192.168.4.0/24) | MGMT network |
| Log | checked | Alert on attempted MGMT access from DMZ |
| Description | `Block DMZ to MGMT - lateral movement prevention` | |

### 4. Add Block Rule: DMZ → DMZ (Rule 2)

Prevents the RPi5 from attacking other devices that might share the DMZ subnet in the future.

| Field | Value |
|-------|-------|
| Action | `Block` |
| Interface | `DMZ` |
| Protocol | `Any` |
| Source | `DMZ subnets` |
| Destination | `DMZ subnets` |
| Log | checked |
| Description | `Block intra-DMZ traffic` |

Exception: if you later add a second DMZ device that the RPi5 needs to communicate with, add an explicit allow rule **above** this block rule for that specific IP:port pair.

### 5. Add Per-Peer Allow Rules (Rules 3–N)

For each VPN peer, add specific allow rules granting access only to the resources that peer needs. The principle: if a peer doesn't have an explicit allow rule for a destination, it cannot reach it.

#### Rule: Laptop peer → Proxmox web UI

| Field | Value | Reason |
|-------|-------|--------|
| Action | `Pass` | |
| Interface | `DMZ` | |
| Protocol | `TCP` | HTTPS only |
| Source | `WG_LAPTOP` (10.0.0.2) | Restrict to this peer's tunnel IP |
| Destination | `<PROXMOX_IP>` | |
| Destination Port | `8006` | Proxmox web UI port |
| Description | `Laptop VPN → Proxmox HTTPS` | |

#### Rule: Laptop peer → Proxmox SSH

| Field | Value |
|-------|-------|
| Action | `Pass` |
| Protocol | `TCP` |
| Source | `WG_LAPTOP` |
| Destination | `<PROXMOX_IP>` |
| Destination Port | `22` |
| Description | `Laptop VPN → Proxmox SSH` |

#### Rule: Laptop peer → Pi-hole DNS

| Field | Value |
|-------|-------|
| Action | `Pass` |
| Protocol | `TCP/UDP` |
| Source | `WG_LAPTOP` |
| Destination | `<PIHOLE_IP>` |
| Destination Port | `53` |
| Description | `Laptop VPN → Pi-hole DNS` |

#### Rule: Phone peer → Pi-hole DNS only

The phone peer gets DNS resolution only — it has no reason to access Proxmox or SSH into anything.

| Field | Value |
|-------|-------|
| Action | `Pass` |
| Protocol | `TCP/UDP` |
| Source | `WG_PHONE` |
| Destination | `<PIHOLE_IP>` |
| Destination Port | `53` |
| Description | `Phone VPN → Pi-hole DNS` |

Add further per-peer rules as needed. Always apply the minimum required access. Resist the temptation to add `<LAN_SUBNET>` → any as a convenience rule.

### 6. Add the Default Block-All Rule (Last Rule)

This is the catch-all that drops any DMZ traffic not matched by a specific allow rule above.

| Field | Value |
|-------|-------|
| Action | `Block` |
| Interface | `DMZ` |
| Protocol | `Any` |
| Source | `any` |
| Destination | `any` |
| Log | checked |
| Description | `Default deny — DMZ to any` |

This rule must be the **last rule** (lowest position) in the DMZ rule list.

### 7. Full Rule Order Summary

After adding all rules, the DMZ rule list should read top to bottom:

| # | Action | Source | Destination | Port | Description |
|---|--------|--------|-------------|------|-------------|
| 1 | Block | DMZ subnets | `<MGMT_SUBNET>` | any | Block DMZ→MGMT |
| 2 | Block | DMZ subnets | DMZ subnets | any | Block intra-DMZ |
| 3 | Pass | `WG_LAPTOP` | `<PROXMOX_IP>` | 8006/TCP | Laptop→Proxmox UI |
| 4 | Pass | `WG_LAPTOP` | `<PROXMOX_IP>` | 22/TCP | Laptop→Proxmox SSH |
| 5 | Pass | `WG_LAPTOP` | `<PIHOLE_IP>` | 53/TCP+UDP | Laptop→DNS |
| 6 | Pass | `WG_PHONE` | `<PIHOLE_IP>` | 53/TCP+UDP | Phone→DNS |
| N | Block | any | any | any | Default deny |

### 8. Return Traffic (LAN → DMZ)

pfSense uses a stateful firewall. When a VPN peer initiates a TCP connection to a LAN device, pfSense creates a state entry. The LAN device's response packets match this state and are automatically passed back to the DMZ — no explicit LAN → DMZ allow rule is needed.

This means you only need to write rules for the *initiating* direction. Do not add open-ended LAN → DMZ allow rules, as this would allow LAN devices to initiate connections to the RPi5 (which is unnecessary and increases attack surface).

## Verification

### Confirm Rules Are Applied

```bash
# From pfSense shell (SSH from MGMT or Diagnostics > Command Prompt)
pfctl -s rules | grep -A2 "DMZ"
```

### Test Allowed Traffic (Should Pass)

From a connected WireGuard client (laptop, tunnel IP 10.0.0.2):

```bash
# DNS resolution via Pi-hole
dig @<PIHOLE_IP> example.com
# Expected: ANSWER SECTION with resolved IP, no timeout

# Proxmox UI reachability
curl -k https://<PROXMOX_IP>:8006
# Expected: HTML response (Proxmox login page)

# SSH to Proxmox
ssh root@<PROXMOX_IP>
# Expected: SSH prompt
```

### Test Blocked Traffic (Should Be Blocked)

From WireGuard client (laptop, 10.0.0.2):

```bash
# Attempt to reach MGMT network — must be blocked
ping <MGMT_SUBNET_GATEWAY>     # e.g. 192.168.4.1
# Expected: no response, request times out

# Attempt to reach a LAN device not in any allow rule
curl http://<MACMINI_IP_1>:80
# Expected: timeout (not connection refused — pfSense drops silently)

# Phone peer attempting SSH (not in phone's allow rules)
# From phone peer (10.0.0.3) — test via WireGuard tunnel:
# Expected: timeout
```

### Check pfSense Firewall Logs

Navigate to **Status > System Logs > Firewall**. Filter by interface = DMZ.

- Block entries for `<MGMT_SUBNET>` destinations confirm the MGMT protection rule fires.
- Block entries from `WG_PHONE` to `<PROXMOX_IP>` confirm phone peer is correctly restricted.

### Verify State Table

Navigate to **Diagnostics > States**. After a successful VPN connection:

- Should show states for `10.0.0.2 → <PIHOLE_IP>:53` (DNS queries)
- Should show states for `10.0.0.2 → <PROXMOX_IP>:8006` (if accessed)
- Should NOT show any states from `10.0.0.x → <MGMT_SUBNET>`

## Troubleshooting

**VPN peer connected but cannot reach any LAN device**
- Check that the peer's tunnel IP (`10.0.0.2` or `10.0.0.3`) is the source in the DMZ rules, not the DMZ IP of the RPi5 (`<DMZ_RPi5_IP>`).
- When WireGuard forwards a packet, the source IP is the peer's tunnel IP (e.g. `10.0.0.2`), not the RPi5's DMZ IP. If your rules target `<DMZ_RPi5_IP>` as source, they will never match.
- Enable logging on the default deny rule and check what source IP is appearing.

**Correct peer can reach a resource but wrong peer can also reach it**
- Verify the source alias in the allow rule is the specific peer tunnel IP, not the entire `WG_TUNNEL` subnet.
- Use **Diagnostics > States** to see which source IP established the connection.

**Traffic from one WireGuard peer reaches another peer**
- By default, WireGuard peers cannot reach each other unless the server's `AllowedIPs` and pfSense rules both permit it.
- If peer-to-peer traffic is unexpectedly passing, check that no allow rule has `WG_TUNNEL` as both source and destination.

**Connection drops after a few minutes**
- pfSense state table entries expire for idle UDP connections. This is expected behavior.
- WireGuard's `PersistentKeepalive = 25` on the client config prevents idle state expiry by sending a keepalive every 25 seconds. Verify this is set in client configs (see `10-wireguard-client-config.md`).

## Security Notes

**Source IP in DMZ rules is the tunnel IP, not the DMZ IP.** When WireGuard decapsulates a packet from peer 10.0.0.2 and forwards it toward the LAN, the source IP in the forwarded packet is `10.0.0.2`. pfSense sees this source IP when evaluating DMZ rules. This is why aliases `WG_LAPTOP`, `WG_PHONE` reference tunnel IPs, not the RPi5's DMZ IP.

**Never use `WG_TUNNEL` as a source for broad access.** Using `10.0.0.0/24` as source in an allow rule grants every current and future peer that access. Always use per-peer aliases.

**Adding a new peer requires both a wg0.conf change (doc 09) and a new pfSense rule.** The peer cannot reach anything until both are done. This is intentional — a peer with no pfSense rules has WireGuard connectivity to the tunnel but cannot reach any LAN resource.

**Do not add floating rules for DMZ traffic.** Floating rules apply across all interfaces and can inadvertently override interface-specific rules. Keep DMZ rules under the DMZ tab only.

**Reject vs Block**: Using `Block` (not `Reject`) drops packets silently without sending an ICMP response. This makes it slightly harder for an attacker inside the DMZ to enumerate what is and isn't reachable. `Reject` sends an immediate response that confirms the destination exists.

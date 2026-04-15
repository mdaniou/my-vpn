# pfSense WAN Rules and Port Forwarding

## Overview

This document covers the WAN-facing firewall rules and NAT port forwarding configuration on pfSense. The only inbound traffic permitted from the internet is UDP port 51820 (WireGuard), forwarded directly to the RPi5 gateway in the DMZ. All other inbound traffic is blocked by default.

This is the first line of defense: attacker traffic is either dropped here at the WAN edge, or forwarded only to the WireGuard daemon in the isolated DMZ — never to the LAN directly.

## Prerequisites

- `03-pfsense-dmz-setup.md` completed: pfSense interfaces configured (WAN, LAN, DMZ, MGMT), DMZ interface has static IP `<DMZ_GATEWAY_IP>`, RPi5 has static IP `<DMZ_RPi5_IP>` in the DMZ subnet
- WAN interface has a static IPv4 address `<WAN_STATIC_IP>` assigned by ISP (Salt Switzerland)
- pfSense admin UI accessible from MGMT VLAN

## Security Objectives

- Enforce default-deny on WAN: no unsolicited inbound traffic reaches any internal zone
- Expose exactly one service to the internet: UDP 51820 forwarded to DMZ RPi5 only
- Block all IPv6 on WAN to eliminate accidental global addressability
- Log all blocked WAN traffic for audit purposes
- Prevent RFC 1918 (private), bogon, and spoofed source addresses from entering via WAN

## Steps

### 1. Verify WAN Interface Configuration

Navigate to **Interfaces > WAN**.

Confirm:
- IPv4 Configuration Type: `Static IPv4`
- Static IPv4 Address: `<WAN_STATIC_IP>`
- IPv6 Configuration Type: `None` (critical — see Security Notes)
- Block private networks: `checked`
- Block bogon networks: `checked`

The "Block private networks" option drops packets sourced from RFC 1918 ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) and loopback. These addresses should never appear as source IPs on a legitimate WAN packet — their presence indicates spoofing or misconfiguration.

The "Block bogon networks" option drops packets sourced from IANA-reserved, unallocated, or special-use ranges. Bogons have no legitimate use as source addresses on the public internet.

Click **Save** if you changed anything, then **Apply Changes**.

### 2. Create the NAT Port Forward Rule

Navigate to **Firewall > NAT > Port Forward**.

Click **Add** (arrow pointing up — adds to top of list).

| Field | Value | Reason |
|-------|-------|--------|
| Interface | `WAN` | Applies to inbound WAN traffic only |
| Protocol | `UDP` | WireGuard uses only UDP — no TCP fallback |
| Destination | `WAN address` | Matches packets destined for the WAN IP |
| Destination port range | `51820` to `51820` | Exactly one port; no range |
| Redirect target IP | `<DMZ_RPi5_IP>` | The RPi5 in DMZ, not a LAN device |
| Redirect target port | `51820` | Same port — no translation needed |
| Description | `WireGuard VPN inbound` | |
| Filter rule association | `Add associated filter rule` | Automatically creates the allow rule |

Click **Save**, then **Apply Changes**.

Why a port forward rather than 1:1 NAT: 1:1 NAT maps all traffic destined for an external IP to an internal IP, which would expose the RPi5 to all inbound protocols. Port forward limits the mapping to exactly UDP 51820 — everything else on `<WAN_STATIC_IP>` is still dropped by the default WAN block rule.

### 3. Review the Auto-Created WAN Firewall Rule

pfSense automatically creates an associated firewall rule when you choose "Add associated filter rule" in the NAT configuration. Navigate to **Firewall > Rules > WAN** to verify it.

The auto-created rule should look like:

| Field | Value |
|-------|-------|
| Action | Pass |
| Interface | WAN |
| Protocol | UDP |
| Source | `any` |
| Destination | `<DMZ_RPi5_IP>` port `51820` |
| Description | `NAT WireGuard VPN inbound` |

This rule allows the forwarded packet (after NAT rewrites the destination from `<WAN_STATIC_IP>` to `<DMZ_RPi5_IP>`) to pass. Without this rule, the NAT translation would occur but the firewall would still block the packet.

**Do not modify the source to restrict by source IP** unless your clients always come from static IPs. Mobile clients and dynamic ISP customers will have changing IPs. WireGuard's cryptographic authentication handles client identity — the firewall's job here is only to allow the port open.

### 4. Verify the Default Block Rule

At the bottom of **Firewall > Rules > WAN**, pfSense includes a built-in default deny rule (shown in grey, labeled "Default deny rule"). This rule blocks all traffic that did not match any explicit allow rule above it.

pfSense evaluates rules top to bottom, first match wins. The explicit UDP 51820 pass rule fires first for WireGuard traffic; everything else falls through to the default deny.

You do not need to create an explicit block rule — the default deny is always present and cannot be removed.

### 5. Enable Logging on the Default Block Rule

The built-in default deny rule does not log by default. To enable logging:

Navigate to **Firewall > Rules > WAN**. The default deny rule is the grey row at the bottom. Click the pencil icon to edit it.

Check **Log packets that are handled by this rule**.

Click **Save**, then **Apply Changes**.

This generates a log entry in **Status > System Logs > Firewall** for every blocked WAN packet. Useful for:
- Detecting port scans and probes
- Auditing unexpected inbound traffic
- Confirming the block rule is firing correctly

### 6. Confirm IPv6 is Blocked on WAN

Navigate to **System > Advanced > Networking**.

Verify:
- **Allow IPv6**: unchecked
- **Prefer IPv4 over IPv6**: checked

Navigate to **Interfaces > WAN**:
- **IPv6 Configuration Type**: `None`

IPv6 must be disabled at the interface level, not just the system level, to prevent SLAAC address assignment and accidental IPv6 reachability. See `12-ipv6-disable-and-kernel-hardening.md` for the full IPv6 hardening procedure.

### 7. Configure pfSense Firewall Optimization

Navigate to **System > Advanced > Firewall & NAT**.

Recommended settings:

| Setting | Value | Reason |
|---------|-------|--------|
| Firewall Optimization Options | `Normal` | Balanced state table expiry |
| Firewall Maximum States | leave default or set to `100000` | Prevents state table exhaustion (DoS) |
| Firewall Maximum Table Entries | leave default | |
| Disable Firewall Scrubbing | unchecked | Scrubbing normalizes fragmented packets, prevents evasion |

Click **Save** if changed.

## Verification

### Confirm NAT Rule Is Active

```bash
# On pfSense shell (Diagnostics > Command Prompt or SSH from MGMT VLAN)
pfctl -s nat | grep 51820
# Expected output:
# rdr on em0 proto udp from any to <WAN_STATIC_IP>/32 port = 51820 -> <DMZ_RPi5_IP> port 51820
```

### Confirm Firewall Pass Rule Is Active

```bash
pfctl -s rules | grep 51820
# Expected output: a pass rule referencing UDP port 51820 destined for <DMZ_RPi5_IP>
```

### Test Port Reachability from Outside

From a device outside the network (mobile hotspot, external VPS, or after completing RPi5 WireGuard setup):

```bash
# UDP port scan — should show port 51820 open (WireGuard responds to handshakes)
nmap -sU -p 51820 <WAN_STATIC_IP>
# Expected: 51820/udp open|filtered  (WireGuard doesn't respond to unauthenticated probes)

# All other ports should be filtered/closed
nmap -p 1-1000 <WAN_STATIC_IP>
# Expected: all ports filtered (default deny rule blocking)
```

Note: WireGuard intentionally does not respond to unauthenticated packets, so nmap may show the port as `open|filtered` rather than `open`. This is correct behavior — it reduces the attack surface by not confirming the service is listening.

### Confirm Blocked Traffic Is Logged

Navigate to **Status > System Logs > Firewall**. After any inbound scan or connection attempt on a non-51820 port, you should see block entries with the WAN interface as source.

### Verify IPv6 Not Assigned on WAN

Navigate to **Status > Interfaces**. The WAN interface should show:
- IPv4 address: `<WAN_STATIC_IP>`
- IPv6 address: none (blank)

## Troubleshooting

**WireGuard clients cannot connect, NAT rule looks correct**
- Check that RPi5 WireGuard is actually running and listening: `ss -ulnp | grep 51820` on RPi5
- Use **Diagnostics > Packet Capture** on the WAN interface, filter for UDP port 51820, to confirm packets arrive at pfSense
- Check **Diagnostics > Packet Capture** on DMZ interface to confirm packets are being forwarded
- Verify the NAT rule is enabled (not greyed out)

**pfSense logs show WAN block hits for UDP 51820**
- The NAT rule processes traffic before the firewall rules. If the firewall block fires for port 51820, the NAT rule may be disabled or incorrectly configured.
- Confirm the auto-created allow rule in **Firewall > Rules > WAN** references `<DMZ_RPi5_IP>:51820` as destination (after NAT translation).

**Cannot reach pfSense admin UI after rule changes**
- The anti-lockout rule in pfSense automatically allows admin UI access from LAN — it cannot be accidentally removed via WAN rules.
- If locked out from MGMT, connect a device directly to the MGMT switch and access the UI from there.

**Inbound UDP 51820 passes but WireGuard handshake fails**
- Clock skew: WireGuard rejects handshakes if the timestamp is too far off. Verify NTP is configured on pfSense (**Services > NTP**) and on RPi5.
- Key mismatch: confirm the server public key in the client config matches the actual server key (`wg show wg0 public-key` on RPi5).

## Security Notes

**Never open additional ports on WAN.** Each open port is an attack surface. If you add services in the future (e.g. a web server), place them in the DMZ, not the LAN, and add explicit per-port rules.

**Do not use 1:1 NAT for the RPi5.** 1:1 NAT would expose all ports on `<DMZ_RPi5_IP>` to the internet, not just UDP 51820. Use port forward.

**Do not whitelist source IPs for the WireGuard rule.** WireGuard clients are mobile and have dynamic IPs. Source-IP filtering on WAN provides minimal security benefit here — WireGuard's cryptographic authentication is the correct mechanism for client identity. Restricting source IPs creates an operational burden without meaningful security improvement.

**Bogon blocking is not a substitute for the default deny rule.** Bogon filtering blocks known bad source ranges; the default deny rule blocks everything not explicitly allowed. Both are necessary.

**Log retention**: pfSense logs are in-memory by default and lost on reboot. Configure remote syslog (**Status > System Logs > Settings > Remote Logging**) to ship logs to a persistent collector (e.g., RPi3B syslog). See `11-monitoring-logging.md`.

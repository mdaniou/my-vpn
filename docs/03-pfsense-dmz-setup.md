# pfSense DMZ Interface and Zone Setup

## Overview

This document covers configuring pfSense to establish the DMZ network zone that will host the RPi5 WireGuard gateway. You will assign the physical interface, configure IP addressing, set DHCP for the DMZ, and apply interface-level hardening. The result is an isolated segment that the WireGuard gateway lives in — separated from both LAN and WAN — with pfSense as the sole enforcer of what crosses zone boundaries.

Firewall rules for WAN→DMZ and DMZ→LAN are covered in subsequent documents (`04` and `05`). This document stops at the interface and DHCP layer.

## Prerequisites

- `02-network-architecture-and-traffic-flow.md` completed: IP plan finalized, zones defined, physical port assignments decided.
- pfSense is installed, accessible via web UI from the MGMT interface.
- The physical port that will become `DMZ` is cabled to the RPi5's NIC (or the switch port feeding it).
- You know your DMZ subnet (this document uses `<DMZ_SUBNET>` — e.g. `192.168.50.0/24`) and the static IP you will assign to the RPi5 (`<RPI5_DMZ_IP>` — e.g. `192.168.50.10`).

## Security Objectives

- Isolate the WireGuard gateway on its own L3 segment so that a compromised RPi5 has no direct path to LAN.
- Assign the RPi5 a static IP via DHCP reservation so firewall rules can target it precisely.
- Block inter-VLAN routing at the interface level — pfSense must explicitly permit every flow; nothing is implicitly routed.
- Disable IPv6 on the DMZ interface to eliminate a parallel attack surface.

## Network Assumptions

Replace placeholders throughout with your actual values:

| Placeholder | Example | Meaning |
|---|---|---|
| `<DMZ_IFACE>` | `igb2` | pfSense NIC assigned to DMZ |
| `<DMZ_GW_IP>` | `192.168.50.1` | pfSense IP on the DMZ segment (gateway) |
| `<DMZ_SUBNET>` | `192.168.50.0/24` | DMZ network block |
| `<DMZ_SUBNET_MASK>` | `255.255.255.0` | Subnet mask for the DMZ |
| `<RPI5_DMZ_IP>` | `192.168.50.10` | Static IP leased to the RPi5 |
| `<RPI5_MAC>` | `dc:a6:32:xx:xx:xx` | RPi5 MAC address (from `ip link` on the device) |
| `<LAN_SUBNET>` | `192.168.10.0/24` | LAN network block (for reference in rules) |
| `<MGMT_SUBNET>` | `192.168.99.0/24` | MGMT network block (for reference in rules) |

## Steps

### 1. Identify the Physical Interface

Log into the pfSense web UI (`https://<MGMT_IP>`) and navigate to **Interfaces → Assignments**.

Identify which NIC (`igb0`, `igb1`, `igb2`, `igb3`, or similar) corresponds to the physical port cabled to your DMZ switch port. If unsure:

1. **Diagnostics → Ping** — ping a known MAC on that port, then check **Diagnostics → ARP Table** to confirm which interface learned the ARP entry.
2. Alternatively, use **Interfaces → Assignments → Network port** — each unassigned port shows its MAC. Match the MAC to the physical label on the appliance.

Note the interface identifier as `<DMZ_IFACE>`.

### 2. Assign the DMZ Interface

In **Interfaces → Assignments**:

1. In the **Available network ports** dropdown, select `<DMZ_IFACE>`.
2. Click **+ Add**.
3. pfSense assigns it as `OPT1` (or the next available `OPTn`). You will rename it in the next step.
4. Click **Save**.

### 3. Configure the DMZ Interface

Navigate to **Interfaces → OPT1** (or whichever `OPTn` was just assigned).

Set the following fields:

```
Enable:             ✓ (checked)
Description:        DMZ
IPv4 Configuration: Static IPv4
IPv4 Address:       <DMZ_GW_IP>  /  24      ← pfSense's own IP on this segment
IPv6 Configuration: None                    ← explicitly disabled; reduces attack surface
MTU:                (leave blank — inherit default 1500)
MSS:                (leave blank)
Speed/Duplex:       Default (autoselect)
```

**Why `None` for IPv6:** The RPi5 and all DMZ devices operate IPv4-only. Assigning an IPv6 address to this interface would create a reachable global or link-local address that bypasses IPv4 firewall rules unless separately locked down. Eliminating IPv6 at the interface level is simpler and safer.

Click **Save**, then **Apply Changes**.

### 4. Verify Interface Is Up

Navigate to **Interfaces → DMZ**. Confirm:

- Status shows **up**
- IPv4 address shows `<DMZ_GW_IP>/24`
- No IPv6 address is assigned

Also check **Status → Interfaces** — the DMZ row should show a MAC, the correct IP, and link status `up`.

If the interface shows `down`, confirm the cable is seated and the RPi5 (or a temporary test device) is plugged into the DMZ port — some NICs will not assert link without a connected peer.

### 5. Create DHCP Server for DMZ

Navigate to **Services → DHCP Server → DMZ**.

```
Enable DHCP server on DMZ interface:   ✓

Subnet:         <DMZ_SUBNET>           (auto-filled, read-only)
Subnet Mask:    <DMZ_SUBNET_MASK>      (auto-filled, read-only)
Range:          192.168.50.100  –  192.168.50.199   ← dynamic pool (for temporary devices)
Gateway:        <DMZ_GW_IP>
DNS Servers:    <DMZ_GW_IP>            ← pfSense Unbound resolver handles DNS
NTP Server:     <DMZ_GW_IP>           ← pfSense NTP server (configured in doc 04)
Domain name:    dmz.local             ← internal-only; never resolves externally
Lease time:     86400                 ← 24h; long enough to be stable, short enough to reclaim
```

**Why restrict the pool to `.100`–`.199`:** The lower range (`.2`–`.99`) is reserved for static DHCP assignments (like the RPi5 at `.10`) and any future DMZ devices with fixed addresses. Static assignments sit outside the dynamic pool so they are never accidentally handed out to an unexpected client.

Click **Save**.

### 6. Create a Static DHCP Reservation for the RPi5

Still in **Services → DHCP Server → DMZ**, scroll to the **DHCP Static Mappings** section at the bottom and click **+ Add**.

```
MAC Address:    <RPI5_MAC>
IP Address:     <RPI5_DMZ_IP>
Hostname:       rpi5-vpn-gw
Description:    WireGuard VPN gateway
```

Click **Save**, then **Apply Changes**.

**Why static assignment via DHCP rather than a manually configured static IP on the RPi5:** The firewall rules in docs 04 and 05 reference `<RPI5_DMZ_IP>` explicitly. If the RPi5 ever gets a different IP (misconfiguration, hardware swap), the DHCP server enforces the correct assignment based on MAC. You still configure the RPi5 to use DHCP, not a hardcoded IP, so you only manage the mapping in one place.

### 7. Disable IPv6 Router Advertisements on DMZ

Navigate to **Services → Router Advertisements → DMZ**.

```
Router Mode:    Disabled
```

Click **Save**.

This prevents pfSense from advertising an IPv6 default gateway on the DMZ segment, which would cause clients to auto-configure IPv6 addresses (SLAAC) even without a DHCPv6 server. Combined with the `None` IPv6 setting on the interface itself, this guarantees no IPv6 traffic will flow on the DMZ.

### 8. Confirm No Implicit Inter-Zone Routing

pfSense does **not** route between interfaces by default when you have firewall rules blocking traffic — but by default it *does* route if no rules are blocking. At this point you have not yet added DMZ firewall rules (that is doc 05), so the DMZ has pfSense's default `allow all` outbound rule.

**Immediately after this step**, proceed to doc 04 (WAN rules) and doc 05 (DMZ→LAN rules) before placing the RPi5 on the DMZ segment. Do not leave the DMZ with default-allow rules in a production environment.

To confirm the current (unsafe) default state:

1. Navigate to **Firewall → Rules → DMZ**.
2. You should see a default `IPv4 *` allow rule. This will be replaced in doc 05.

### 9. (Optional) Add a Descriptive Interface Group

If you have multiple DMZ-adjacent interfaces in the future, grouping them simplifies rule management. For now, skip this — you have a single DMZ interface and targeted per-interface rules are clearer.

## Verification

### Check Interface Assignment

**Interfaces → Assignments** — confirm `DMZ` maps to `<DMZ_IFACE>`.

### Check IP Configuration

**Status → Interfaces → DMZ**:

```
Status:   up
IPv4:     <DMZ_GW_IP>/24
IPv6:     (none)
```

### Check DHCP Lease

Boot (or reboot) the RPi5 with DHCP configured on its interface. Then check **Status → DHCP Leases**:

```
Interface   IP Address       MAC Address        Hostname       Type
DMZ         <RPI5_DMZ_IP>   <RPI5_MAC>         rpi5-vpn-gw    static
```

If the RPi5 is not yet installed, use any temporary device on the DMZ port to confirm the DHCP server is handing out addresses from the dynamic pool.

### Ping Test from pfSense to DMZ Gateway IP

**Diagnostics → Ping**:

```
Host:         <DMZ_GW_IP>
IP Protocol:  IPv4
Source:       DMZ
```

Expected: replies from `<DMZ_GW_IP>` (pfSense pinging itself on the DMZ interface). This confirms the interface is up and the IP is bound.

### Ping Test from RPi5 to pfSense DMZ Gateway

From the RPi5 (once booted into Alpine with a DHCP lease):

```sh
ping -c 4 <DMZ_GW_IP>
```

Expected output:

```
PING 192.168.50.1 (192.168.50.1): 56 data bytes
64 bytes from 192.168.50.1: seq=0 ttl=64 time=0.4 ms
...
4 packets transmitted, 4 received, 0% packet loss
```

### Confirm No IPv6 on DMZ

From the RPi5:

```sh
ip -6 addr show
```

Expected: no global or link-local IPv6 addresses on the DMZ-facing interface (only loopback `::1` is acceptable).

From pfSense shell (**Diagnostics → Command Prompt**):

```sh
ifconfig <DMZ_IFACE> | grep inet6
```

Expected: no output (no IPv6 addresses bound to the DMZ interface).

## Troubleshooting

### RPi5 Does Not Get a DHCP Lease

1. Confirm the cable is in the correct physical port (the one mapped to `<DMZ_IFACE>`).
2. Check **Status → DHCP Leases** — if a lease appears with the wrong IP, the MAC in the static mapping may be incorrect. Verify with `ip link show` on the RPi5.
3. Check **Status → System Logs → DHCP** for `DHCPDISCOVER` and `DHCPOFFER` messages. A `DISCOVER` with no `OFFER` means pfSense is not seeing the request — likely a cabling or VLAN trunk issue.

### Interface Shows `down` in pfSense

- Confirm a device is plugged into the DMZ port (some NICs require a link partner to assert up).
- Check **Diagnostics → ARP Table** — if the RPi5 has any ARP entries, pfSense is seeing layer-2 traffic.
- Reseat the cable; check switch port configuration if there is a managed switch between pfSense and the RPi5.

### RPi5 Gets an IP but Cannot Ping `<DMZ_GW_IP>`

- Confirm pfSense has `<DMZ_GW_IP>` bound: **Diagnostics → Command Prompt** → `ifconfig <DMZ_IFACE>`.
- Check firewall rules: **Firewall → Rules → DMZ** — if rules were accidentally configured before doc 05, they might be blocking ICMP. The default rule should allow all traffic at this stage.

### IPv6 Appearing on RPi5 Despite Configuration

- Ensure `net.ipv6.conf.all.disable_ipv6 = 1` is set in `/etc/sysctl.conf` on the RPi5 (covered in doc 07).
- Ensure router advertisements are disabled in pfSense: **Services → Router Advertisements → DMZ → Router Mode: Disabled**.

## Security Notes

**Do not leave the DMZ with default-allow rules.** pfSense ships with a permissive default rule on new interfaces. This document intentionally stops short of writing DMZ firewall rules — those are in docs 04 and 05 — but you must apply those rules before treating this setup as secure. A DMZ with a default-allow outbound rule provides no isolation benefit.

**Static DHCP assignment is not authentication.** A device can spoof the RPi5's MAC address and receive the `<RPI5_DMZ_IP>` reservation. The DHCP reservation is for operational convenience and rule precision, not for access control. The actual security boundary is the pfSense firewall rules that restrict what `<RPI5_DMZ_IP>` can reach.

**Do not enable the pfSense anti-lockout rule on the DMZ interface.** The anti-lockout rule is intended for LAN/MGMT, not DMZ. If it appears in **Firewall → Rules → DMZ**, remove it — it creates an implicit allow for the web UI that contradicts your default-deny posture.

**Keep the DMZ segment dedicated.** Do not place any device on the DMZ segment other than the RPi5 WireGuard gateway. The firewall rules in docs 04 and 05 are written assuming the only host in `<DMZ_SUBNET>` is the WireGuard gateway. Adding other devices broadens the blast radius if the WireGuard gateway is compromised.

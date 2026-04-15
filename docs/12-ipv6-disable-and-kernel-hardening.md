# IPv6 Disable and Kernel Hardening

## Overview

This document provides the complete procedure for disabling IPv6 on all components (pfSense interfaces and Alpine RPi5) and applying kernel-level network hardening to the RPi5. While individual steps appear in earlier documents, this document consolidates and verifies the full IPv6 disable across the entire stack.

IPv6 disable is a critical security measure for this setup: a device with IPv6 enabled and SLAAC active can receive a globally routable IPv6 address automatically, making it directly reachable from the internet and bypassing pfSense's IPv4 firewall rules entirely. This threat is real and not hypothetical — many ISPs now allocate /56 or /48 IPv6 prefixes to residential connections.

## Prerequisites

- `07-alpine-hardening-and-network-config.md` completed: base sysctl files in place on RPi5
- pfSense base configuration completed (docs 02–05)
- SSH access to RPi5 from MGMT VLAN

## Security Objectives

- No IPv6 addresses on any interface of any device in the setup (WAN, LAN, DMZ, MGMT, RPi5)
- IPv6 disabled at the kernel level on RPi5 (prevents SLAAC even if an application tries to enable it)
- IPv6 disabled at the interface level on pfSense (no SLAAC, no DHCPv6, no router advertisements processed)
- All kernel network hardening settings verified and persistent across reboots

## Steps

### Part 1: Disable IPv6 on pfSense

IPv6 must be disabled on each interface individually, plus at the system level.

#### 1.1 System-Level IPv6 Disable

Navigate to **System > Advanced > Networking**:

| Setting | Value | Reason |
|---------|-------|--------|
| Allow IPv6 | unchecked | System-wide disable; prevents any IPv6 forwarding |
| Prefer IPv4 over IPv6 | checked | Ensures IPv4 is used when both are available |
| IPv6 over IPv4 Tunneling | disabled | Do not create IPv6-in-IPv4 tunnels (6in4, Teredo, etc.) |

Click **Save**.

#### 1.2 WAN Interface IPv6

Navigate to **Interfaces > WAN**:

| Setting | Value |
|---------|-------|
| IPv6 Configuration Type | `None` |

Setting this to `None` prevents pfSense from:
- Sending DHCPv6 requests to the ISP
- Processing Router Advertisements (SLAAC)
- Assigning a global IPv6 address to the WAN interface

Without this, your WAN interface might receive a public IPv6 address from Salt Switzerland and begin routing IPv6 traffic — bypassing all WAN firewall rules that target IPv4 only.

Click **Save** → **Apply Changes**.

#### 1.3 LAN Interface IPv6

Navigate to **Interfaces > LAN**:

| Setting | Value |
|---------|-------|
| IPv6 Configuration Type | `None` |

Also navigate to **Services > DHCPv6 Server & RA**. For the LAN interface, ensure DHCPv6 is **disabled** and Router Advertisements mode is set to **Disabled**. This prevents LAN devices from receiving IPv6 addresses via SLAAC from pfSense.

#### 1.4 DMZ Interface IPv6

Navigate to **Interfaces > DMZ**:

| Setting | Value |
|---------|-------|
| IPv6 Configuration Type | `None` |

#### 1.5 MGMT Interface IPv6

Navigate to **Interfaces > MGMT**:

| Setting | Value |
|---------|-------|
| IPv6 Configuration Type | `None` |

#### 1.6 Verify No IPv6 Addresses Are Assigned

Navigate to **Status > Interfaces**. For each interface (WAN, LAN, DMZ, MGMT):
- IPv4 address: correct static or DHCP-assigned address shown
- IPv6 address: blank / "none"

If any interface shows an IPv6 address, the corresponding step above was not applied or requires a **Apply Changes** to take effect.

### Part 2: Disable IPv6 on Alpine RPi5

#### 2.1 sysctl Configuration

This should already exist from doc 07. Verify and ensure it is complete:

```bash
cat /etc/sysctl.d/99-no-ipv6.conf
```

Expected content (create or update if different):

```bash
cat > /etc/sysctl.d/99-no-ipv6.conf << 'EOF'
# Disable IPv6 on all present and future interfaces
# Reason: SLAAC can assign a globally routable IPv6 address automatically,
# bypassing pfSense's IPv4-based firewall rules. We disable it at the kernel
# level so no process or interface can enable it without kernel parameter change.

net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# Explicitly disable on known interfaces
net.ipv6.conf.eth0.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

# Disable IPv6 routing
net.ipv6.conf.all.forwarding = 0
net.ipv6.conf.all.accept_ra = 0
net.ipv6.conf.default.accept_ra = 0
EOF

# Apply immediately
sysctl -p /etc/sysctl.d/99-no-ipv6.conf
```

#### 2.2 Kernel Command Line (Boot-time Disable)

sysctl settings apply after the kernel has already initialized. Some IPv6 operations happen at kernel init, before sysctl is applied. The `ipv6.disable=1` kernel parameter disables IPv6 at the earliest possible point.

```bash
# Mount the RPi5 boot partition
mount /dev/mmcblk0p1 /mnt 2>/dev/null || mount | grep mmcblk0p1

# Read current cmdline
cat /mnt/cmdline.txt
# Example: console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 fsck.repair=yes rootwait

# Append ipv6.disable=1 — must remain a single line, no line breaks
# Use sed to append safely:
if ! grep -q 'ipv6.disable=1' /mnt/cmdline.txt; then
  sed -i 's/$/ ipv6.disable=1/' /mnt/cmdline.txt
fi

cat /mnt/cmdline.txt
# Verify: line ends with ... ipv6.disable=1

umount /mnt
```

#### 2.3 ip6tables — Block Any IPv6 That Leaks Through

As an additional safeguard, drop all IPv6 traffic in ip6tables even if some IPv6 were to be active:

```bash
# Drop all IPv6 traffic on all chains
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
ip6tables -P OUTPUT DROP

# Allow loopback (required for some local operations)
ip6tables -A INPUT -i lo -j ACCEPT
ip6tables -A OUTPUT -o lo -j ACCEPT

# Save rules
ip6tables-save > /etc/iptables/rules6-save

# Enable ip6tables service at boot
rc-update add ip6tables default
```

### Part 3: Full Kernel Hardening Verification on RPi5

This section verifies all hardening settings from doc 07 are active and persistent.

#### 3.1 Verify All sysctl Files Exist

```bash
ls /etc/sysctl.d/
# Expected:
# 99-forwarding.conf
# 99-hardening.conf
# 99-no-ipv6.conf
```

#### 3.2 Apply All Settings

```bash
# Apply all sysctl.d configs at once
sysctl --system

# Or per-file:
sysctl -p /etc/sysctl.d/99-no-ipv6.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-hardening.conf
```

#### 3.3 Verify Critical Values

```bash
# IPv6 disabled
sysctl net.ipv6.conf.all.disable_ipv6
# Expected: 1

sysctl net.ipv6.conf.eth0.disable_ipv6
# Expected: 1

# IP forwarding enabled (required for WireGuard)
sysctl net.ipv4.ip_forward
# Expected: 1

# Reverse path filtering
sysctl net.ipv4.conf.all.rp_filter
# Expected: 1

# ICMP redirects disabled
sysctl net.ipv4.conf.all.accept_redirects
# Expected: 0

sysctl net.ipv4.conf.all.send_redirects
# Expected: 0

# Source routing disabled
sysctl net.ipv4.conf.all.accept_source_route
# Expected: 0

# SYN cookies
sysctl net.ipv4.tcp_syncookies
# Expected: 1

# Kernel pointer restriction
sysctl kernel.kptr_restrict
# Expected: 2

# dmesg restriction
sysctl kernel.dmesg_restrict
# Expected: 1
```

#### 3.4 Verify sysctl Service Is Enabled at Boot

```bash
rc-update show | grep sysctl
# Expected: sysctl | boot
```

If not present:
```bash
rc-update add sysctl boot
```

### Part 4: Commit and Reboot Test

```bash
# Commit all changes
lbu commit -d

# Reboot
reboot
```

After reboot, run through Section 3.3 again to confirm all settings survive reboot.

## Verification

### RPi5 — No IPv6 Addresses

```bash
# Should show no IPv6 addresses on eth0 or wg0
ip -6 addr show
# Expected: only ::1/128 on lo (loopback) — or nothing if lo IPv6 is also disabled

# No IPv6 routes
ip -6 route show
# Expected: no output (or only local route if loopback IPv6 enabled)

# IPv6 connectivity fails
ping6 -c 2 ::1 2>&1 | head -3
# Expected: "Network is unreachable" or "connect: Network is unreachable"
```

### pfSense — No IPv6 Addresses

```bash
# From pfSense shell (Diagnostics > Command Prompt or SSH from MGMT)
ifconfig | grep inet6
# Expected: only fe80:: link-local on loopback if any — ideally nothing on external interfaces

# Or check Status > Interfaces in UI — all IPv6 fields should be blank
```

### Full IPv6 Leak Test

From an admin workstation connected to the VPN:

```bash
# Attempt IPv6 connectivity — should fail
curl -6 --max-time 5 https://ipv6.google.com
# Expected: "Couldn't connect to server" or "Network is unreachable"

# No IPv6 DNS resolution
dig AAAA ipv6.google.com @<PIHOLE_IP>
# Expected: NOERROR with empty ANSWER section, or Pi-hole may return NXDOMAIN
```

## Troubleshooting

**IPv6 address still appears on RPi5 eth0 after sysctl apply**

```bash
# Check if the address appeared before sysctl ran (SLAAC at boot)
# Confirm cmdline.txt has ipv6.disable=1
cat /mnt/cmdline.txt | grep ipv6.disable

# If sysctl disable works but boot assign persists:
# Manually remove the address temporarily
ip -6 addr del <IPV6_ADDR>/64 dev eth0
# Then fix cmdline.txt and reboot
```

**pfSense still shows IPv6 address on WAN after setting to None**

- Click **Apply Changes** in the Interfaces page — setting to None does not automatically release a DHCPv6 lease
- Navigate to **Status > DHCPv6 Leases** and release any active leases
- If the ISP pushed a prefix delegation, it may take a few minutes to expire after stopping DHCPv6

**After `sysctl --system`, getting "No such file or directory" errors**

- The sysctl key may not exist on this kernel version
- Check with `sysctl -a 2>/dev/null | grep <key_name>` to see what is available
- Remove unsupported keys from the config file

## Security Notes

**Why kernel cmdline disable is more reliable than sysctl alone**: sysctl runs as a userspace service, meaning there is a brief window at boot (between kernel init and sysctl service start) where IPv6 is enabled. During this window, the network interface could receive a Router Advertisement and auto-configure an IPv6 address. `ipv6.disable=1` in cmdline.txt prevents IPv6 from ever initializing, closing this window completely.

**Do not rely on pfSense IPv6 firewall rules as a substitute.** The correct approach is to not have IPv6 addresses at all, not to have IPv6 addresses and block all traffic. Firewall rules can have gaps; an interface with no IPv6 address has no IPv6 attack surface at all.

**LAN devices**: disabling IPv6 on pfSense prevents pfSense from distributing IPv6 prefixes to LAN devices via SLAAC/DHCPv6, but it does not prevent LAN devices from processing Router Advertisements from other sources. If you have any router or switch on the LAN that might send RAs, disable IPv6 on those devices too. The most complete protection is disabling IPv6 in each device's OS.

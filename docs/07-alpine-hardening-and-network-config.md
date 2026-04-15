# Alpine Hardening and Network Configuration

## Overview

This document hardens the Alpine Linux system on the RPi5 after the base install. Hardening covers three areas: reducing the running service footprint to the minimum, applying kernel-level network hardening via sysctl, and configuring SSH with strict access controls. The network interface is also locked to a static configuration.

All changes must be committed with `lbu commit -d` to survive reboots.

## Prerequisites

- `06-rpi5-hardware-and-alpine-diskless-install.md` completed: Alpine is running in diskless mode, static IP `<DMZ_RPi5_IP>`, packages installed
- Console or SSH access to RPi5

## Security Objectives

- Disable all non-essential services (reduce attack surface to only what WireGuard requires)
- Disable IPv6 at the kernel level (eliminates accidental global addressability)
- Enable IP forwarding (required for WireGuard to route packets between tunnel and LAN)
- Apply kernel hardening: prevent ICMP redirect attacks, source routing, SYN floods
- Harden SSH: key-only auth, no root login, bind only to DMZ interface, restrict to MGMT source IPs

## Steps

### 1. Audit and Disable Unnecessary Services

Check what Alpine starts by default:

```bash
rc-status --all
```

Expected output includes something like:
```
Dynamic Runlevel: hotplugged
Dynamic Runlevel: needed
Runlevel: sysinit
  devfs       [ started ]
  dmesg       [ started ]
  mdev        [ started ]
Runlevel: boot
  modules     [ started ]
  sysfs       [ started ]
  proc        [ started ]
  networking  [ started ]
  syslog      [ started ]
  lbu         [ started ]
  apkcache    [ started ]
  chronyd     [ started ]
Runlevel: default
  sshd        [ started ]
```

Services that should be running:
- `networking` — required (static IP)
- `sshd` — required (remote management)
- `chronyd` — required (NTP; WireGuard handshakes are time-sensitive)
- `syslog` — required (audit trail)
- `lbu` — required (diskless config management)

Services to remove if present (add to this list as needed):
```bash
# Remove avahi (mDNS — not needed, leaks host information)
rc-update del avahi-daemon 2>/dev/null

# Remove acpid if not needed for RPi5
# (RPi5 uses GPIO for power, not ACPI; acpid is a no-op but unnecessary)
rc-update del acpid 2>/dev/null
```

### 2. Network Interface Configuration

Verify `/etc/network/interfaces` has the correct static IP configuration. It should have been set by `setup-alpine`, but verify:

```bash
cat /etc/network/interfaces
```

It should contain:

```
# /etc/network/interfaces
# Static configuration for DMZ interface
# No IPv6 stanzas — IPv6 is disabled at the kernel level

auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
    address <DMZ_RPi5_IP>       # e.g. 192.168.3.2
    netmask 255.255.255.0       # /24
    gateway <DMZ_GATEWAY>       # pfSense DMZ interface, e.g. 192.168.3.1
```

If it differs, edit it to match the above, then restart networking:

```bash
rc-service networking restart
ip addr show eth0    # confirm address
```

### 3. Disable IPv6 via sysctl

Create the sysctl config file:

```bash
cat > /etc/sysctl.d/99-no-ipv6.conf << 'EOF'
# Disable IPv6 on all interfaces
# Reason: prevents accidental SLAAC address assignment and global IPv6 reachability,
# which could bypass pfSense IPv4-based firewall rules entirely.

net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
EOF
```

Apply immediately (without reboot):

```bash
sysctl -p /etc/sysctl.d/99-no-ipv6.conf
```

Also add to the kernel command line to disable IPv6 at the earliest possible point in boot, before the network subsystem initializes:

```bash
# The RPi5 boot cmdline is in /media/mmcblk0p1/cmdline.txt
# Mount the boot partition if not already mounted
mount /dev/mmcblk0p1 /mnt 2>/dev/null || true

# Read current cmdline
cat /mnt/cmdline.txt

# Add ipv6.disable=1 to the end of the single line in cmdline.txt
# Example current content: console=serial0,115200 console=tty1 root=/dev/mmcblk0p2 ...
# Append: ipv6.disable=1

# Edit manually to add ipv6.disable=1 at the end of the line:
# (Use vi or sed — be careful not to add a newline; cmdline.txt must be a single line)
sed -i 's/$/ ipv6.disable=1/' /mnt/cmdline.txt

# Verify
cat /mnt/cmdline.txt
# The line should end with: ... ipv6.disable=1

umount /mnt
```

### 4. Enable IP Forwarding

WireGuard acts as a router: it receives encrypted packets on the tunnel interface and forwards decrypted packets toward LAN. The Linux kernel must have IP forwarding enabled for this to work.

```bash
cat > /etc/sysctl.d/99-forwarding.conf << 'EOF'
# Enable IPv4 forwarding — required for WireGuard to route packets
# between the wg0 tunnel interface and eth0 (DMZ → LAN direction)
# Without this, the kernel drops forwarded packets and VPN clients
# cannot reach LAN resources even if pfSense allows the traffic.
net.ipv4.ip_forward = 1

# Do NOT enable IPv6 forwarding — IPv6 is disabled entirely
# net.ipv6.conf.all.forwarding = 0  (default, left unset)
EOF

sysctl -p /etc/sysctl.d/99-forwarding.conf
```

### 5. Kernel Network Hardening

```bash
cat > /etc/sysctl.d/99-hardening.conf << 'EOF'
# =========================================================
# Reverse path filtering
# Drops packets that arrive on the wrong interface relative
# to the routing table. Prevents source IP spoofing.
# An attacker on the WireGuard tunnel cannot spoof LAN IPs.
# =========================================================
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# =========================================================
# ICMP redirect handling
# Accepting ICMP redirects allows a device on the same segment
# to modify our routing table. An attacker in the DMZ could
# redirect our traffic through a malicious gateway.
# Sending redirects is also unnecessary for our use case.
# =========================================================
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# =========================================================
# Source routing
# IP source routing allows a packet to specify its own route.
# This can be used to bypass firewall rules by routing packets
# through trusted intermediate hosts.
# =========================================================
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# =========================================================
# SYN flood protection
# SYN cookies allow the kernel to handle SYN floods without
# exhausting the connection backlog. The RPi5 is internet-facing
# (via WireGuard UDP); legitimate TCP services are few but
# SYN cookies are low cost and always worth enabling.
# =========================================================
net.ipv4.tcp_syncookies = 1

# =========================================================
# Bogus ICMP error responses
# Some broken hosts send invalid ICMP error codes. Ignoring
# these prevents spurious errors from polluting logs and
# potentially confusing stateful firewall tracking.
# =========================================================
net.ipv4.icmp_ignore_bogus_error_responses = 1

# =========================================================
# Log martian packets
# Martians are packets with source addresses that are impossible
# given the routing context (e.g. a packet claiming to be from
# 192.168.1.x arriving on the WireGuard interface from the internet).
# Logging these helps detect misconfiguration or spoofing attempts.
# =========================================================
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# =========================================================
# Restrict dmesg to root
# Kernel ring buffer can contain sensitive information about
# device drivers, memory layout, and loaded modules.
# Non-root processes should not be able to read it.
# =========================================================
kernel.dmesg_restrict = 1

# =========================================================
# Disable magic SysRq key
# SysRq allows keyboard-triggered kernel operations (reboot,
# memory dumps). Unnecessary on a headless server appliance
# and disabling it removes one potential misuse vector.
# =========================================================
kernel.sysrq = 0

# =========================================================
# Hide kernel pointers
# /proc/kallsyms and similar interfaces expose kernel symbol
# addresses, which help attackers craft kernel exploits.
# kptr_restrict=2 hides these from all users including root.
# =========================================================
kernel.kptr_restrict = 2
EOF

sysctl -p /etc/sysctl.d/99-hardening.conf
```

### 6. Ensure sysctl Service Runs at Boot

Alpine applies `/etc/sysctl.d/*.conf` files via the `sysctl` openrc service:

```bash
rc-update add sysctl boot
rc-service sysctl start
```

Verify all settings are active:

```bash
sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1

sysctl net.ipv6.conf.all.disable_ipv6
# Expected: net.ipv6.conf.all.disable_ipv6 = 1

sysctl net.ipv4.conf.all.rp_filter
# Expected: net.ipv4.conf.all.rp_filter = 1
```

### 7. SSH Hardening

Edit `/etc/ssh/sshd_config`. Replace the defaults with this configuration:

```bash
cat > /etc/ssh/sshd_config << 'EOF'
# /etc/ssh/sshd_config — hardened SSH configuration for WireGuard gateway

# Only listen on the DMZ interface IP, not on 0.0.0.0
# Reason: limits SSH exposure to the DMZ network only.
# Even if an attacker gains a WireGuard tunnel IP, sshd
# is not listening on the wg0 interface.
ListenAddress <DMZ_RPi5_IP>

Port 22

# Disable root login entirely — use a named admin user
PermitRootLogin no

# Disable all password-based authentication methods
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no

# Enable public key authentication only
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Restrict to specific admin user(s)
# Replace <ADMIN_USER> with the actual non-root admin username
AllowUsers <ADMIN_USER>

# Prevent SSH tunneling — we don't need it and it could be abused
AllowTcpForwarding no
X11Forwarding no
AllowAgentForwarding no

# Disconnect idle sessions after 10 minutes (300s × 2 = 600s max idle)
ClientAliveInterval 300
ClientAliveCountMax 2

# Reduce the window during which a failed login attempt is counted
LoginGraceTime 30

# Limit authentication attempts per connection
MaxAuthTries 3

# Limit concurrent unauthenticated connections (prevents connection flood)
MaxStartups 3:50:10

# Disable unused authentication methods
UsePAM no
PermitEmptyPasswords no

# Protocol hardening
Protocol 2

# Restrict accepted key types to modern algorithms only
PubkeyAcceptedKeyTypes ssh-ed25519,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521

# Log level — INFO is sufficient; VERBOSE adds key fingerprints to auth log
LogLevel INFO
EOF
```

### 8. Create Admin User and Install SSH Key

```bash
# Create non-root admin user
adduser -D <ADMIN_USER>   # -D: no password (will use keys only)

# Create SSH authorized_keys directory
mkdir -p /home/<ADMIN_USER>/.ssh
chmod 700 /home/<ADMIN_USER>/.ssh

# On your admin workstation, generate an Ed25519 key if you don't have one:
# ssh-keygen -t ed25519 -C "admin@vpn-gw" -f ~/.ssh/vpn_gw

# Copy the PUBLIC key content (from ~/.ssh/vpn_gw.pub on your workstation)
# Paste it into the authorized_keys file on RPi5:
cat >> /home/<ADMIN_USER>/.ssh/authorized_keys << 'EOF'
ssh-ed25519 AAAA...<YOUR_PUBLIC_KEY_HERE> admin@vpn-gw
EOF

chmod 600 /home/<ADMIN_USER>/.ssh/authorized_keys
chown -R <ADMIN_USER>:<ADMIN_USER> /home/<ADMIN_USER>/.ssh
```

### 9. Test SSH Key Auth Before Disabling Password Auth

With the current `sshd_config` in place, restart sshd:

```bash
rc-service sshd restart
```

From your admin workstation (on MGMT VLAN), test key-based login:

```bash
ssh -i ~/.ssh/vpn_gw <ADMIN_USER>@<DMZ_RPi5_IP>
```

You must successfully log in before proceeding. If key auth fails, do not commit — troubleshoot from the console.

### 10. Commit All Changes

```bash
lbu commit -d
```

Verify what was committed:

```bash
lbu list | grep -E "sysctl|sshd|network|interfaces"
```

## Verification

```bash
# IPv6 disabled
ip -6 addr show
# Expected: no addresses on eth0 (only ::1 on lo if anything)

# IP forwarding enabled
sysctl net.ipv4.ip_forward
# Expected: 1

# rp_filter active
sysctl net.ipv4.conf.all.rp_filter
# Expected: 1

# ICMP redirects disabled
sysctl net.ipv4.conf.all.accept_redirects
# Expected: 0

# sshd listening only on DMZ IP
ss -tlnp | grep sshd
# Expected: 0.0.0.0:22 NOT shown; only <DMZ_RPi5_IP>:22

# sshd running
rc-status | grep sshd
# Expected: sshd [ started ]

# SSH login from MGMT VLAN with key
ssh -i ~/.ssh/vpn_gw <ADMIN_USER>@<DMZ_RPi5_IP>
# Expected: shell prompt (no password prompt)
```

## Troubleshooting

**sshd fails to start after editing sshd_config**

```bash
# Test the config for syntax errors before restarting
sshd -t
# If errors are shown, fix them before restarting the service
```

**Cannot SSH in with key after restart**

- Check `sshd -T | grep listenaddress` — confirm it matches `<DMZ_RPi5_IP>`
- Check `/var/log/auth.log` for the rejection reason: `tail -f /var/log/auth.log`
- Verify key permissions: `authorized_keys` must be `600`, `.ssh` must be `700`
- Verify `AllowUsers` matches the username you are connecting as

**sysctl settings not persisting after reboot**

- Verify `sysctl` service is in the boot runlevel: `rc-update show | grep sysctl`
- If not: `rc-update add sysctl boot`
- Verify the files exist in `/etc/sysctl.d/`: `ls /etc/sysctl.d/`
- Verify `lbu commit -d` was run after creating the sysctl files

## Security Notes

**`ListenAddress` binding is not a substitute for pfSense firewall rules.** sshd bound to `<DMZ_RPi5_IP>` will not accept connections on the wg0 tunnel address. However, if a misconfiguration re-enables listening on `0.0.0.0`, the pfSense DMZ rules (which block WireGuard tunnel IPs from reaching port 22) provide the second enforcement layer. Both layers must be correct.

**The `AllowUsers` directive is case-sensitive.** `AllowUsers Admin` and `AllowUsers admin` are different. Match the exact username as created in `/etc/passwd`.

**Never commit the admin user's private key.** The `authorized_keys` file (containing the public key) is safe to commit via lbu. The private key (`~/.ssh/vpn_gw`) lives only on the admin workstation and is never copied to the RPi5.

**Rotate SSH keys when admin personnel change.** If the admin workstation changes or is compromised, regenerate the SSH keypair, update `authorized_keys` on RPi5, and commit.

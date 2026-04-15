# RPi5 Hardware and Alpine Linux Diskless Install

## Overview

This document covers the physical hardware setup for the Raspberry Pi 5 WireGuard gateway and the installation of Alpine Linux in diskless mode. Diskless mode is a core security feature: the system runs entirely from RAM, the SD card holds only a compressed config overlay, and an attacker who exploits the running system cannot persist changes across a reboot without also having write access to the SD card.

After completing this document, you will have a minimal Alpine Linux system booted and ready for hardening and WireGuard installation.

## Prerequisites

- pfSense is fully configured with the DMZ interface and rules (docs 02–05)
- RPi5 is physically connected to the pfSense DMZ switch/port
- Admin workstation on MGMT VLAN with SSH access
- A separate device to write the SD card (Linux or macOS)

## Security Objectives

- Run the WireGuard gateway OS entirely from RAM (diskless), so exploits cannot persist across reboots
- Minimize installed packages to the absolute minimum required for WireGuard
- Disable IPv6 from the start
- Use a static IP (no DHCP client in production) to prevent IP changes that would break pfSense rules

## Steps

### 1. Hardware Requirements

| Component | Specification | Notes |
|-----------|---------------|-------|
| RPi5 | 4GB or 8GB RAM | Either is fine; Alpine diskless uses ~80MB RAM |
| SD card | ≥4GB, Class 10 or better | Holds Alpine image + config overlay only; no heavy I/O |
| Power supply | Official RPi5 27W USB-C PSU | Underpowering causes random crashes |
| Ethernet cable | Cat5e or better | RPi5 built-in GbE port connects to pfSense DMZ |
| MicroSD card reader | For initial SD write | Any USB reader works |

A case with passive cooling is recommended but not required. The RPi5 has no spinning disks — diskless mode means the SD card is read-only after boot, greatly extending its lifespan.

Do not use WiFi for the WireGuard gateway. The DMZ connection must use the wired Ethernet port for reliability and to simplify firewall rules.

### 2. Download Alpine Linux for RPi5

Alpine Linux for Raspberry Pi uses the `aarch64` architecture. Download the **Raspberry Pi** image from the official Alpine downloads page.

```bash
# On your admin workstation
# Replace x.x.x with the current Alpine version (check alpine.org for latest stable)
ALPINE_VERSION="3.21.3"
ALPINE_FILE="alpine-rpi-${ALPINE_VERSION}-aarch64.img.gz"

# Download the image
curl -LO "https://dl-cdn.alpinelinux.org/alpine/v${ALPINE_VERSION%.*}/releases/aarch64/${ALPINE_FILE}"

# Download the SHA256 checksum file
curl -LO "https://dl-cdn.alpinelinux.org/alpine/v${ALPINE_VERSION%.*}/releases/aarch64/${ALPINE_FILE}.sha256"

# Verify the checksum — output must say "OK"
sha256sum -c "${ALPINE_FILE}.sha256"
```

If the SHA256 check fails, do not proceed. Re-download the image. A corrupted or tampered image is a supply chain risk.

### 3. Write Alpine to SD Card

```bash
# Identify the SD card device — list block devices before and after insertion
lsblk

# The SD card will appear as /dev/sdX or /dev/mmcblkX
# Replace /dev/sdX with your actual device — double-check before proceeding
# This command DESTROYS all data on the target device

gunzip -c "${ALPINE_FILE}" | sudo dd of=/dev/sdX bs=4M status=progress conv=fsync

# Flush write buffers before ejecting
sync
```

The `conv=fsync` flag ensures all data is written to the card before `dd` exits. Do not remove the card until `sync` completes.

### 4. First Boot — Initial Setup

Insert the SD card into the RPi5 and connect:
- Ethernet cable to pfSense DMZ port
- HDMI monitor and USB keyboard (required only for initial setup; removed afterward)
- Power supply (RPi5 powers on when connected)

Alpine boots to a login prompt. Log in as `root` with no password.

### 5. Run the Alpine Setup Script

`setup-alpine` is a guided installer that configures the system. Run it:

```bash
setup-alpine
```

Answer each prompt as follows. Values marked `<PLACEHOLDER>` are environment-specific — substitute yours.

```
Keyboard layout: us
Variant: us

Hostname: vpn-gw
# Short, descriptive, no FQDN needed for a single-purpose appliance

Network interface to initialize: eth0
IP address: <DMZ_RPi5_IP>
# e.g. 192.168.3.2 — static, matches pfSense DMZ rules

Netmask: 255.255.255.0
# or /24 in CIDR notation

Gateway: <DMZ_GATEWAY>
# pfSense DMZ interface IP, e.g. 192.168.3.1

DNS domain: (leave blank or set to local domain)
DNS nameserver: <DNS_IP>
# Use pfSense LAN IP or Pi-hole IP
# Do NOT use 8.8.8.8 — the DMZ has no direct WAN access; DNS must route through pfSense

Root password: <STRONG_ROOT_PASSWORD>
# Will be locked/disabled after SSH keys are set up (doc 11)

Timezone: Europe/Zurich
# Or your local timezone

HTTP/FTP proxy: (leave blank — no proxy)

NTP client: chrony
NTP server: <DMZ_GATEWAY>
# Use pfSense as NTP source — the DMZ cannot reach WAN NTP servers directly
# pfSense must have NTP enabled (Services > NTP) and allow DMZ queries

SSH server: openssh

Which disk(s) would you like to use: none
# *** THIS IS THE CRITICAL CHOICE FOR DISKLESS MODE ***
# "none" means: do not install to disk, run from RAM
# The config overlay (lbu) is saved to the SD card's data partition separately

Enter where to store configs: (press Enter for default /dev/mmcblk0p1 or the SD card partition)

APK cache directory: /var/cache/apk
```

After answering all prompts, `setup-alpine` configures the system in RAM and returns to the shell.

### 6. Understanding Diskless Mode and lbu

In diskless mode, Alpine:
1. Boots the kernel and initramfs from the SD card FAT partition
2. Loads the system into RAM
3. Mounts an `apkovl` (Alpine Package Keeper overlay) archive from the SD card's data partition
4. All runtime changes (files, config) exist only in RAM

`lbu` (Local Backup Utility) manages the overlay:

```bash
# Save current in-RAM config state to SD card
lbu commit -d

# The -d flag includes packages installed via apk add
# Without -d, installed packages are not remembered across reboots

# View what will be committed (dry run)
lbu status

# List files currently tracked
lbu list
```

**Critical**: Every time you make a configuration change (sshd_config, WireGuard config, sysctl settings), you must run `lbu commit -d` to persist it. If you reboot without committing, all changes are lost.

This behavior is a security feature, not a bug: an exploit that modifies files in RAM is automatically undone on reboot, without any action from you.

### 7. Install Essential Packages

```bash
# Update package index
apk update

# Install packages needed for subsequent docs
# Keep this list minimal — every package is attack surface
apk add wireguard-tools iptables ip6tables chrony openssh util-linux

# Save the installed package list
lbu commit -d
```

Package rationale:
- `wireguard-tools`: provides `wg` and `wg-quick` commands; WireGuard kernel module is built into Alpine ≥3.17
- `iptables`: local firewall rules on the RPi5 (defense in depth)
- `ip6tables`: needed to explicitly block IPv6 (even though it's disabled via sysctl)
- `chrony`: NTP client (accurate time is required for WireGuard handshake validation)
- `openssh`: SSH server for remote management
- `util-linux`: provides `lsblk` and other diagnostic utilities

### 8. Verify Static IP and Network Connectivity

```bash
# Confirm IP assignment
ip addr show eth0
# Expected: inet <DMZ_RPi5_IP>/24

# Confirm gateway reachability (pfSense DMZ interface)
ping -c 3 <DMZ_GATEWAY>
# Expected: 3 packets received

# Confirm DNS resolution works
nslookup google.com <DNS_IP>
# Expected: resolved address (confirms pfSense/Pi-hole DNS is forwarding correctly)

# No IPv6 addresses should appear
ip -6 addr show
# Expected: only loopback (::1), no global or link-local addresses on eth0
```

### 9. Set Up lbu Exclusions

Some files should never be committed to the SD card because they contain secrets:

```bash
# Edit /etc/lbu/lbu.conf to exclude WireGuard private keys from lbu commits
# We will add private keys to a separate, manually backed-up location

# Add exclusions
echo "/etc/wireguard/server_private.key" >> /etc/lbu/exclude
echo "/etc/wireguard/psk_*.key" >> /etc/lbu/exclude

lbu commit -d
```

The WireGuard private key security trade-off is covered in detail in `08-wireguard-install-and-server-config.md`. For now, ensure the exclusion pattern is in place before generating any keys.

### 10. Commit and Reboot

```bash
# Final commit of initial configuration
lbu commit -d

# Reboot to verify the system comes up cleanly from the committed config
reboot
```

After reboot, log back in via console or SSH from the MGMT VLAN (once SSH is hardened in doc 11). Verify all services started correctly and the static IP is retained.

## Verification

After reboot, confirm the system state:

```bash
# Running services
rc-status
# Expected: sshd, chronyd, networking — all started

# Network config survived reboot
ip addr show eth0
# Expected: <DMZ_RPi5_IP>/24

# RAM usage — should be minimal (~80–120MB used)
free -m

# Installed packages persist
apk info | grep wireguard-tools
# Expected: wireguard-tools listed

# Disk is read-only (the SD card FAT partition)
mount | grep mmcblk
# Expected: /dev/mmcblk0p1 on /media/mmcblk0p1 type vfat (ro,...) 
# The boot partition is mounted read-only — correct
```

## Troubleshooting

**System does not retain config after reboot (reverts to default)**
- You did not run `lbu commit -d` before rebooting. Run `lbu status` to see uncommitted changes, make the config again, and commit.
- Verify the SD card data partition is writable during the commit: `lbu commit -d -v` (verbose output).

**Cannot ping pfSense DMZ gateway after setup**
- Verify the pfSense DMZ interface has the correct subnet and IP (`<DMZ_GATEWAY>`)
- Verify the Ethernet cable is connected to the correct pfSense port (DMZ, not LAN)
- Check pfSense **Status > Interfaces > DMZ** for link state

**`setup-alpine` chose DHCP instead of static IP**
- Edit `/etc/network/interfaces` manually:
  ```
  auto eth0
  iface eth0 inet static
      address <DMZ_RPi5_IP>
      netmask 255.255.255.0
      gateway <DMZ_GATEWAY>
  ```
- Restart networking: `rc-service networking restart`
- Commit: `lbu commit -d`

**NTP not syncing**
- Verify pfSense NTP service is running: **Services > NTP** in pfSense UI
- Verify the pfSense DMZ interface allows NTP (UDP 123) from `<DMZ_RPi5_IP>` — add a DMZ rule if needed
- Check chrony status: `chronyc tracking`

## Security Notes

**Diskless mode does not protect against SD card physical access.** The `apkovl` overlay on the SD card is not encrypted. Anyone with physical access to the SD card can read all committed config files. Keep the RPi5 in a physically secure location (server rack, locked cabinet).

**Do not write WireGuard private keys to lbu.** Private keys committed to the SD card persist across reboots and are readable by anyone with the SD card. The exclusion set in Step 9 prevents this. In `08-wireguard-install-and-server-config.md` we document the full key management strategy.

**Do not install additional packages beyond what is listed.** The security value of Alpine diskless comes partly from its minimal footprint. Every extra package is a potential exploit path. If you need a diagnostic tool temporarily, install it in RAM (`apk add <pkg>` without committing) and it will be gone after reboot.

**Remove HDMI and keyboard after setup is complete.** Physical access to the console bypasses SSH restrictions. Once remote SSH access is confirmed, disconnect the monitor and keyboard. If physical access is required for recovery, reconnect then — it should not be the normal management path.

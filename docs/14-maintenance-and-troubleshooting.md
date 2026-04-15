# Maintenance and Troubleshooting

## Overview

This document covers routine maintenance procedures and troubleshooting for the WireGuard VPN infrastructure: key rotation, OS updates, backups, log management, and a structured response to common failure modes and security incidents. It is the operational reference for keeping the system secure and functional after initial deployment.

This is the final document in the series. All components — pfSense, RPi5 Alpine diskless, WireGuard, iptables, and monitoring — must be fully deployed before these procedures apply.

---

## Prerequisites

All prior documents completed and verified:

- `01-project-overview-and-security-model.md` through `13-verification-and-testing.md`
- WireGuard running on RPi5 with at least one connected peer
- pfSense firewall rules in place (WAN, DMZ, LAN, MGMT)
- Alpine diskless with `lbu` configured and at least one clean `lbu commit -d` completed
- Syslog shipping to RPi3B (doc 11) operational

---

## Security Objectives

- Maintain cryptographic hygiene through scheduled key rotation
- Preserve integrity of the diskless Alpine system across updates and reboots
- Detect and contain peer compromise with minimal downtime for other peers
- Enable fast, clean recovery from RPi5 compromise without trusting any state on the device
- Ensure troubleshooting steps do not create temporary security holes (e.g. disabling all firewall rules)

---

## Steps

### 1. WireGuard PSK Rotation (Every 90 Days)

PSKs provide a symmetric post-quantum layer on top of WireGuard's Curve25519 key exchange. Rotate every 90 days per the schedule in doc 09.

**On the RPi5 (as root):**

```bash
# Step 1: Generate the new PSK for the target peer
wg genpsk > /etc/wireguard/psk_<PEER_NAME>_new.key
chmod 600 /etc/wireguard/psk_<PEER_NAME>_new.key

# Step 2: View the new PSK — you will need to send this to the peer operator
cat /etc/wireguard/psk_<PEER_NAME>_new.key
```

**Coordinate with the peer operator** before proceeding. The peer must have the new PSK config ready to apply. The handshake will fail during the brief window between server and client update — plan for 30–60 seconds of downtime for that peer.

```bash
# Step 3: Edit the server config to reference the new PSK
vi /etc/wireguard/wg0.conf

# Find the [Peer] block for <PEER_NAME> and update:
#   PresharedKey = <contents of psk_<PEER_NAME>_new.key>

# Step 4: Apply the change without restarting the interface (zero downtime for other peers)
wg syncconf wg0 <(wg-quick strip wg0)
```

**On the peer device** (coordinate timing — apply immediately after server update):

```ini
# In the peer's wg0.conf [Peer] block:
PresharedKey = <NEW_PSK>
```

Reload the peer's WireGuard interface after updating.

```bash
# Step 5: Verify handshake succeeds with the new PSK
wg show wg0
# Confirm "latest handshake" is recent (within the last few seconds)

# Step 6: Delete the old PSK file and rename the new one to canonical path
rm -f /etc/wireguard/psk_<PEER_NAME>.key
mv /etc/wireguard/psk_<PEER_NAME>_new.key /etc/wireguard/psk_<PEER_NAME>.key

# Step 7: Persist changes to the Alpine overlay
lbu commit -d

# Step 8: Log the rotation in your peer inventory
# Record: peer name, rotation date, operator who performed it
```

---

### 2. WireGuard Private Key Rotation (Annual or Post-Compromise)

Private key rotation requires all peer configs to be updated simultaneously because the server's public key changes. Plan a maintenance window.

**On the RPi5:**

```bash
# Step 1: Generate new server keypair
wg genkey > /etc/wireguard/server_private_new.key
chmod 600 /etc/wireguard/server_private_new.key
wg pubkey < /etc/wireguard/server_private_new.key > /etc/wireguard/server_public_new.key

# Step 2: Distribute the new PUBLIC key to all peer operators out-of-band
# (secure channel — Signal, age-encrypted file, GPG)
# Peers update their configs in advance but do NOT apply yet
cat /etc/wireguard/server_public_new.key
```

**Peer pre-staging (all peers):** Each peer operator updates their `[Peer]` block with the new `PublicKey` value but does NOT reload yet.

**Atomic switchover:**

```bash
# Step 3: Update wg0.conf on the server to use the new private key
vi /etc/wireguard/wg0.conf
# In [Interface]: PrivateKey = <contents of server_private_new.key>

# Step 4: Bring the interface down and back up with the new key
wg-quick down wg0
wg-quick up wg0

# Simultaneously: all peer operators reload their WireGuard interfaces
```

```bash
# Step 5: Verify all peers reconnect
wg show wg0
# Each peer should show a recent "latest handshake"

# Step 6: Rotate the old private key files
rm /etc/wireguard/server_private.key
mv /etc/wireguard/server_private_new.key /etc/wireguard/server_private.key
mv /etc/wireguard/server_public_new.key /etc/wireguard/server_public.key

# Step 7: Persist
lbu commit -d
```

---

### 3. Alpine Linux Package Updates

Alpine diskless does not update automatically. Run updates manually and validate WireGuard still functions afterward, since the kernel module version must match the running kernel.

```bash
# Step 1: Update package index and upgrade all packages
apk update && apk upgrade

# Step 2: Check if the kernel was updated (new kernel = module reload required)
apk info linux-rpi | grep -i version
# Compare against running kernel:
uname -r

# Step 3: If the kernel was upgraded, reboot to load the new kernel
# WireGuard's kernel module is in-tree since Linux 5.6 — it upgrades with the kernel
reboot

# Step 4: After reboot, verify WireGuard interface comes back up
wg show wg0
# Expected: interface listed, peers shown, listening on port 51820

# Step 5: Verify iptables rules are intact (restored by the init script on boot)
iptables -L -n -v --line-numbers
iptables -t nat -L -n -v

# Step 6: Persist the updated package state
lbu commit -d
```

If WireGuard fails to load after a kernel upgrade, the module may not have been included in the upgrade. Fix:

```bash
apk add wireguard-tools
modprobe wireguard
wg-quick up wg0
lbu commit -d
```

---

### 4. pfSense Updates

pfSense updates can modify firewall rule ordering or reset certain settings. Always back up first.

**Before updating:**

1. Navigate to **Diagnostics > Backup & Restore**
2. Click **Download configuration as XML** — save to a secure location off the firewall
3. Note the current version: **System > Update > System Update**

**Performing the update:**

1. Navigate to **System > Update > System Update**
2. Click **Confirm** — pfSense will download and apply the update, then reboot
3. Do not interrupt the process

**After update — verify these items in order:**

```
[ ] Interfaces are all assigned correctly (Status > Interfaces)
[ ] WAN NAT/port forward for UDP 51820 → <DMZ_RPi5_IP> is intact
     (Firewall > NAT > Port Forward)
[ ] WAN firewall rules: only UDP 51820 allowed inbound
     (Firewall > Rules > WAN)
[ ] DMZ rules: only WireGuard tunnel traffic to allowed LAN destinations
     (Firewall > Rules > DMZ)
[ ] MGMT rules: SSH to pfSense and <DMZ_RPi5_IP> only from MGMT subnet
     (Firewall > Rules > MGMT)
[ ] LAN rules: no inbound from WAN or DMZ unless explicitly allowed
[ ] DHCP leases intact (Services > DHCP Server)
[ ] Unbound DNS resolver running (Services > DNS Resolver)
[ ] pfSense itself can ping <DMZ_RPi5_IP>
[ ] A WireGuard peer can connect end-to-end
```

If a rule is missing after update, restore it manually from the XML backup or via the UI.

---

### 5. Alpine Diskless SD Card Backup

The Alpine diskless overlay (`.apkovl.tar.gz`) contains all committed config. Back it up regularly and before any major changes.

**Create a backup archive:**

```bash
# lbu package creates a standalone archive of all committed files
lbu package -v /tmp/alpine-backup-$(date +%Y%m%d).apkovl.tar.gz

# Verify the archive contents
tar -tzf /tmp/alpine-backup-$(date +%Y%m%d).apkovl.tar.gz | head -40
```

**Copy off-device** (from your admin workstation on MGMT VLAN):

```bash
scp root@<DMZ_RPi5_IP>:/tmp/alpine-backup-<DATE>.apkovl.tar.gz \
    /path/to/secure/backup/location/
```

**Restore procedure** (after SD card failure or wipe):

```bash
# Step 1: Boot a fresh Alpine diskless image on the RPi5
# Step 2: Copy the .apkovl.tar.gz onto the new SD card boot partition
#         Alpine init auto-restores if the .apkovl file is in the boot partition,
#         or restore manually:
lbu restore /path/to/alpine-backup-<DATE>.apkovl.tar.gz

# Step 3: Reboot — all committed configs will be applied on boot
reboot
```

The restored system will have all WireGuard configs, iptables init scripts, SSH keys, and syslog config exactly as they were when the backup was taken.

---

### 6. Log Review and Rotation

**pfSense firewall logs:**

- View in real-time: **Status > System Logs > Firewall**
- Blocked traffic is logged by the default deny rules — review daily for anomalies
- Export: **Status > System Logs > Settings** — configure remote syslog to RPi3B (doc 11)
- Log retention: pfSense keeps logs in RAM; persistent logging requires syslog export

Normal firewall log entries to expect:
- Blocked inbound WAN traffic to any port other than 51820 (internet background noise)
- Allowed UDP 51820 from `<WAN_STATIC_IP>` peers

Suspicious entries requiring investigation:
- Any outbound connection from `<DMZ_RPi5_IP>` to unexpected external IPs
- Traffic from DMZ toward LAN subnets other than the allowed WireGuard tunnel destinations
- SSH attempts to pfSense from outside MGMT VLAN

**Alpine syslog (WireGuard and system events):**

```bash
# On RPi5 — default syslog location
tail -f /var/log/messages

# Filter for WireGuard events
grep -i wireguard /var/log/messages

# Check for authentication failures
grep -i "authentication\|invalid\|failed" /var/log/messages
```

Normal WireGuard log entries:
- `wireguard: wg0: Peer <PUBKEY_PREFIX> sent handshake initiation`
- `wireguard: wg0: Peer <PUBKEY_PREFIX> sent handshake response`

Suspicious WireGuard log entries:
- Repeated handshake attempts from unknown public keys (probe or replay attempt)
- High volume of `invalid MAC` or `invalid handshake` messages

**Log rotation on Alpine:**

```bash
# Syslog rotates automatically; verify the config:
cat /etc/syslog-conf        # or /etc/syslog.conf depending on installed package

# If shipping to RPi3B (doc 11), logs are preserved there regardless of local rotation
```

---

## Verification

After any maintenance procedure, run this checklist before considering the task complete:

```bash
# 1. WireGuard interface is up
wg show wg0
# Expected: interface, public key, listening port (51820), one or more peers

# 2. All expected peers are present
wg show wg0 peers

# 3. Recent handshake for active peers (within last few minutes for connected peers)
wg show wg0 latest-handshakes

# 4. IP forwarding enabled
sysctl net.ipv4.ip_forward
# Expected: net.ipv4.ip_forward = 1

# 5. iptables rules intact
iptables -L FORWARD -n -v
# Expected: ACCEPT rules for wg0 traffic, DROP as default policy or final rule

# 6. lbu status is clean (no uncommitted changes)
lbu status
# Expected: no output (nothing uncommitted); if there are changes, commit them

# 7. pfSense: WAN port forward intact
# Diagnostics > Test Port — test UDP 51820 reachability

# 8. End-to-end: connect a WireGuard peer and ping a LAN device
ping -c 3 <LAN_DEVICE_IP>   # from peer device over tunnel
```

---

## Troubleshooting

### T1. WireGuard Peer Not Connecting

**Symptom:** Client reports "connection timeout" or `wg show` shows no recent handshake.

**Diagnosis checklist (work top-down):**

```bash
# 1. On RPi5: confirm WireGuard is listening on the expected port
ss -ulnp | grep 51820
# Expected: UNCONN  0  0  0.0.0.0:51820  0.0.0.0:*

# 2. Capture incoming packets on the RPi5 DMZ interface
tcpdump -i eth0 udp port 51820 -n
# Send a connection attempt from the client while capture is running
# If no packets arrive: pfSense NAT/port forward rule is broken or not forwarding to DMZ

# 3. If packets arrive but no handshake completes: check for key mismatch
wg show wg0
# Verify the peer's public key in wg0.conf exactly matches what the client has
# as the server's PublicKey in its [Peer] block

# 4. Check for clock skew — WireGuard rejects handshakes with timestamp drift > 3 minutes
date                          # on RPi5
# Compare against the client device clock
# Fix: ensure NTP is running on both sides
rc-service ntpd status        # on RPi5
ntpq -p                       # shows NTP sync status and stratum

# 5. AllowedIPs mismatch — server must have the client's tunnel IP in its AllowedIPs
wg show wg0 allowed-ips
# Each peer should show its assigned tunnel IP (e.g. 10.0.0.2/32) and nothing broader
# The client must have the server's tunnel IP and target LAN subnets in its AllowedIPs
```

**pfSense packet capture for port forward verification:**

Navigate to **Diagnostics > Packet Capture**, select the WAN interface, filter on port 51820, start capture, trigger a client connection attempt, stop capture, and inspect whether packets are arriving at WAN and being forwarded to `<DMZ_RPi5_IP>`.

---

### T2. VPN Connected but Cannot Reach LAN

**Symptom:** `wg show` shows a recent handshake and traffic counters incrementing, but pings to LAN IPs fail.

```bash
# 1. Confirm IP forwarding on RPi5
sysctl net.ipv4.ip_forward
# Must be 1 — if 0:
sysctl -w net.ipv4.ip_forward=1
# Ensure the sysctl.conf setting is committed: lbu commit -d

# 2. Check iptables FORWARD chain
iptables -L FORWARD -n -v --line-numbers
# Look for an ACCEPT rule for wg0 traffic
# If the default policy is DROP and no ACCEPT rule exists for wg0:
# add it (refer to doc 08 for the exact rule set)

# 3. Check MASQUERADE / NAT rule
iptables -t nat -L POSTROUTING -n -v
# Should show MASQUERADE for traffic leaving eth0 sourced from the WireGuard tunnel subnet

# 4. Check pfSense DMZ → LAN rules
# Firewall > Rules > DMZ
# Verify there is an ACCEPT rule for traffic from <DMZ_RPi5_IP> to the specific
# LAN destination IPs and ports that the peer needs to reach
# If missing: add the rule (refer to doc 05)

# 5. Traceroute from the peer to the LAN target to identify where packets drop
traceroute <LAN_DEVICE_IP>    # run on the peer device
# Hop 1: WireGuard server tunnel IP (10.0.0.1)
# Hop 2: pfSense DMZ interface (<DMZ_GW_IP>)
# Stops at hop 1 → RPi5 iptables or IP forwarding is the problem
# Stops at hop 2 → pfSense DMZ→LAN rule is the problem
```

---

### T3. SSH Locked Out of RPi5

**Symptom:** Cannot SSH from MGMT VLAN; connection refused or times out.

```bash
# From MGMT workstation — verify the MGMT → DMZ SSH rule in pfSense
# Firewall > Rules > MGMT — should allow TCP 22 from MGMT subnet to <DMZ_RPi5_IP>

# Test basic reachability
ping <DMZ_RPi5_IP>            # from MGMT workstation

# If reachable but SSH refused: sshd is not running on RPi5
# Requires physical console access
```

**Recovery via physical console (HDMI + keyboard attached to RPi5):**

```bash
# Log in at the console as root (no network required)

# Check sshd config for syntax errors
sshd -t

# Restart sshd
rc-service sshd restart

# Verify sshd is now listening
ss -tlnp | grep :22

# If you need to temporarily allow password auth to regain access from MGMT:
vi /etc/ssh/sshd_config
# Set: PasswordAuthentication yes
rc-service sshd restart
# Log in from MGMT, fix the underlying issue (restore authorized_keys, fix permissions)
# Then immediately revert:
vi /etc/ssh/sshd_config
# Set: PasswordAuthentication no
rc-service sshd restart
lbu commit -d
```

Do not leave `PasswordAuthentication yes` in place. Revert it immediately after regaining key-based access.

---

### T4. RPi5 Will Not Boot After lbu commit

**Symptom:** RPi5 boots to a busybox prompt or fails to apply the overlay; services do not start.

**Recovery:**

```bash
# Step 1: Boot a known-good Alpine image from a second SD card or USB stick

# Step 2: Mount the primary SD card
mkdir -p /mnt/sd
mount /dev/mmcblk0p1 /mnt/sd   # adjust device path as needed

# Step 3: Inspect the overlay archive for corruption
ls /mnt/sd/*.apkovl.tar.gz
tar -tzf /mnt/sd/<HOSTNAME>.apkovl.tar.gz | head -40
# If tar errors: the archive is corrupt

# Step 4: Restore from a known-good off-device backup
cp /path/to/backup/alpine-backup-<DATE>.apkovl.tar.gz \
   /mnt/sd/<HOSTNAME>.apkovl.tar.gz

# Step 5: Unmount and reboot with the primary SD card
umount /mnt/sd
# Boot the primary SD card — it will load the restored overlay
```

If no backup is available, rebuild from scratch using doc 06 (Alpine install) and re-apply all configs from docs 07–12.

---

### T5. pfSense Rule Accidentally Blocks Legitimate Traffic

**Symptom:** A device or peer that previously worked can no longer reach its destination after a pfSense rule change.

```bash
# Step 1: Identify the blocking rule via pfSense firewall logs
# Status > System Logs > Firewall
# Filter by source IP or destination IP/port
# The log entry shows the rule description and interface

# Step 2: Check existing connection states
# Diagnostics > States — search for the source IP
# An existing state may be passing or blocking traffic independently of current rules

# Step 3: Packet capture to confirm traffic is arriving at pfSense
# Diagnostics > Packet Capture — select the relevant interface, filter by IP/port

# Step 4: Identify and fix the specific incorrect rule
# Firewall > Rules > <interface>
# Do NOT disable all rules — find only the problematic rule and correct or delete it
```

**If you need to revert an entire set of rule changes:**

1. Navigate to **Diagnostics > Backup & Restore**
2. Restore the XML backup taken before the change
3. pfSense reloads all rules from the backup immediately

---

### T6. WireGuard Interface Stuck or Hung

**Symptom:** `wg show` returns no output or hangs; peers cannot connect; WireGuard process unresponsive.

```bash
# Step 1: Clean restart
wg-quick down wg0
wg-quick up wg0

# If wg-quick down hangs or fails:
# Step 2: Force-delete the interface and recreate it
ip link delete wg0
wg-quick up wg0

# If the module itself is in a bad state:
# Step 3: Remove and reload the kernel module
modprobe -r wireguard
modprobe wireguard
wg-quick up wg0

# Verify
wg show wg0
```

After recovery, check `/var/log/messages` for kernel errors related to the wireguard module. A pattern of recurring hangs may indicate a hardware issue or a kernel bug requiring an OS update.

---

### T7. Peer Shows Connected but No Traffic Passes

**Symptom:** `wg show` shows a recent handshake for the peer, TX/RX counters increment, but application traffic does not work.

```bash
# Step 1: Confirm counters are actually changing while traffic is sent
watch -n 2 wg show wg0
# Confirm "transfer" line shows increasing bytes in both directions

# Step 2: Check AllowedIPs on both sides — mismatches cause silent packet drops
wg show wg0 allowed-ips           # on RPi5 server
# Server must list the peer's tunnel IP (e.g. 10.0.0.2/32)
# Client AllowedIPs must include the target LAN subnet (e.g. 192.168.10.0/24)
# Packets for IPs not in AllowedIPs are dropped silently by the WireGuard kernel module

# Step 3: Check for MTU issues
# WireGuard encapsulation adds overhead; MTU 1420 avoids fragmentation on standard links
ip link show wg0
# Expected: mtu 1420

# If MTU is missing or wrong, add to wg0.conf [Interface] section:
vi /etc/wireguard/wg0.conf
# Add under [Interface]: MTU = 1420
wg-quick down wg0 && wg-quick up wg0
lbu commit -d

# Apply the same MTU = 1420 in the [Interface] section of the client config
```

---

## Security Incident Response

### IR1. Suspected Peer Compromise (Device Stolen or Key Leaked)

**Goal:** Remove the peer immediately, contain any ongoing access, and audit for damage.

```bash
# Step 1: Remove the peer from the running WireGuard interface immediately
# Takes effect instantly — no interface restart required, no impact on other peers
wg set wg0 peer <COMPROMISED_PEER_PUBKEY> remove

# Verify the peer is gone
wg show wg0 peers
# The compromised public key must not appear

# Step 2: Remove the peer from wg0.conf to prevent restoration on next start
vi /etc/wireguard/wg0.conf
# Delete the entire [Peer] block for the compromised peer

# Step 3: Remove any pfSense rules specific to this peer's tunnel IP
# Firewall > Rules > DMZ — delete any ACCEPT rules for the peer's tunnel IP (e.g. 10.0.0.X)

# Step 4: Delete the peer's PSK file
rm /etc/wireguard/psk_<PEER_NAME>.key

# Step 5: Rotate PSKs for ALL remaining peers
# PSKs are per-peer and not shared, but if the compromised device had access to
# other peer configs (e.g. a shared workstation), all PSKs must be treated as suspect
# Follow the PSK rotation procedure in Step 1 for each remaining peer

# Step 6: Persist all changes
lbu commit -d

# Step 7: Audit pfSense firewall logs for traffic from the compromised peer's tunnel IP
# Status > System Logs > Firewall — filter by source 10.0.0.X
# Look for: connections to unexpected LAN destinations, large data transfers, port scans
# Review as far back as logs allow (or the RPi3B syslog archive)

# Step 8: Update peer inventory — mark peer as revoked with date and reason
```

---

### IR2. Suspected RPi5 Compromise

**Goal:** Contain the compromise at the firewall layer, preserve evidence, and restore a clean system.

**Step 1 — Immediate containment at pfSense (do this before touching the RPi5):**

1. Navigate to **Firewall > Rules > DMZ**
2. Add a BLOCK rule at the top of the DMZ ruleset: block ALL traffic from `<DMZ_RPi5_IP>` to any destination
3. Apply the rule — this ensures the RPi5 cannot reach LAN even if under attacker control

Verify the block is in effect by attempting to reach a LAN device through the VPN from a peer — all tunnel traffic should now be blocked.

**Step 2 — Evidence preservation (before any remediation):**

```bash
# From pfSense — export all firewall logs immediately
# Status > System Logs > Firewall > Download
# Save with a timestamp — this is your primary evidence source

# From RPi3B syslog collector (doc 11)
# SSH to RPi3B and archive all relevant log files
ssh <RPi3B_IP>
tar -czf /tmp/incident-logs-$(date +%Y%m%d-%H%M).tar.gz \
    /var/log/remote/<DMZ_RPi5_IP>/
# Copy the archive off to your admin workstation
```

**Step 3 — Remediation:**

```bash
# Do NOT attempt to clean the running system — treat all state on it as compromised
# Diskless Alpine means wiping = power off + reflash SD card

# Power down the RPi5 (physically or via pfSense power management if available)

# Reflash the SD card with a clean Alpine diskless image (doc 06)

# Restore from a known-good backup ONLY IF the backup predates the compromise window
lbu restore <backup.apkovl.tar.gz>
# If the backup is suspect or unavailable: rebuild from scratch using docs 06–12

# Regenerate ALL WireGuard keys — server keypair and all PSKs
wg genkey > /etc/wireguard/server_private.key
chmod 600 /etc/wireguard/server_private.key
wg pubkey < /etc/wireguard/server_private.key > /etc/wireguard/server_public.key

for peer in <PEER1_NAME> <PEER2_NAME>; do
  wg genpsk > /etc/wireguard/psk_${peer}.key
  chmod 600 /etc/wireguard/psk_${peer}.key
done

# Distribute new server public key and PSKs to all peer operators
# Use out-of-band secure channels only — never use the VPN itself for this

# Test each peer connection before removing the containment block rule in pfSense

# Remove the temporary block rule once the clean system is verified end-to-end
# Firewall > Rules > DMZ — delete the emergency block rule
```

**Step 4 — Post-incident review:**

```
[ ] Audit pfSense logs for data exfiltration: large outbound transfers from
    <DMZ_RPi5_IP> to WAN during the compromise window
[ ] Review MGMT VLAN SSH logs: any successful SSH sessions from unexpected source IPs?
[ ] Assess whether any LAN devices need credential rotation if they were accessible
    through the compromised gateway during the window
[ ] Review and tighten any firewall rules that may have been too permissive
[ ] Document the incident: timeline, indicators of compromise, actions taken, lessons learned
```

---

## Security Notes

**Do not disable all firewall rules to troubleshoot.** Always identify and modify the specific rule causing the problem. Disabling all rules exposes the entire network. Use pfSense packet capture and firewall logs to pinpoint the issue before making any rule changes.

**Do not use `wg-quick down wg0` during active incident response** unless the RPi5 itself is the threat. Bringing the interface down does not remove a peer — use `wg set wg0 peer <PUBKEY> remove` for immediate peer revocation without affecting other peers.

**`lbu commit -d` is required after every config change.** Alpine diskless loses all uncommitted changes on reboot. Any maintenance procedure that modifies files under `/etc/` must be followed by `lbu commit -d`. End every session on the RPi5 with `lbu status` and commit if anything changed.

**Key material never leaves the device unencrypted.** When transferring PSKs to peer operators during rotation, use an encrypted channel (Signal, age-encrypted file, or GPG). Never send key material over email, SMS, or unencrypted chat.

**Backups contain key material.** The `.apkovl.tar.gz` backup archive includes WireGuard private keys and PSKs. Store it encrypted, access-controlled, and off-network. Treat it with the same sensitivity as the keys themselves.

**NTP is a security dependency.** WireGuard's handshake mechanism rejects messages with a timestamp drift greater than 3 minutes. If the RPi5's clock drifts — for example after a long power outage without NTP resync — all peers will fail to connect with no obvious error message on the client side. Ensure NTP is enabled at boot (`rc-update add ntpd`) and that pfSense allows outbound NTP from the DMZ.

**After any compromise, assume all secrets on the affected device are leaked.** This includes WireGuard private keys, PSKs, SSH host keys, and any credentials stored in config files. The sequence is: revoke access first, then rotate credentials, then restore service. Never rotate credentials before revoking access.

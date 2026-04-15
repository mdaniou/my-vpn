# SSH Hardening and MGMT VLAN Access

## Overview

This document hardens SSH access to the RPi5 and pfSense, restricting it to the MGMT VLAN only. This provides two independent enforcement layers: sshd_config limits SSH to specific users and authentication methods, while pfSense firewall rules enforce the MGMT-only source IP restriction at the network layer.

The threat model this addresses: if the RPi5 is compromised via a WireGuard exploit or a vulnerability in another service, the attacker cannot use SSH to maintain persistent access from their WireGuard tunnel IP or from the LAN — only a device physically on the MGMT VLAN can SSH in.

## Prerequisites

- `07-alpine-hardening-and-network-config.md` completed: SSH basic config in place, admin user created
- `05-pfsense-dmz-to-lan-firewall-rules.md` completed: pfSense DMZ rules framework in place
- Admin workstation on MGMT VLAN with an SSH keypair generated

## Security Objectives

- SSH access to RPi5 restricted to MGMT VLAN (`<MGMT_SUBNET>`) at both the application layer (sshd) and the network layer (pfSense)
- Key-only authentication — passwords disabled entirely after keys are verified
- No root SSH login
- sshd binds to the DMZ interface IP only (not all interfaces)
- pfSense admin UI accessible from MGMT VLAN only
- Two-layer enforcement: a sshd misconfiguration cannot be exploited unless pfSense is also misconfigured

## Steps

### 1. Generate SSH Keypair on Admin Workstation

Generate the keypair on the admin workstation (the single hardened device on MGMT VLAN). The private key never leaves this machine.

```bash
# On admin workstation (MGMT VLAN device)
# Use Ed25519 — modern, small key, fast, and resistant to implementation bugs
ssh-keygen -t ed25519 -C "admin@vpn-gw-$(date +%Y)" -f ~/.ssh/vpn_gw

# This creates:
# ~/.ssh/vpn_gw       — private key (protect this)
# ~/.ssh/vpn_gw.pub   — public key (safe to share)

# Set strict permissions
chmod 600 ~/.ssh/vpn_gw
chmod 644 ~/.ssh/vpn_gw.pub

# View the public key — you will install this on RPi5
cat ~/.ssh/vpn_gw.pub
# Record as <ADMIN_PUBLIC_KEY>
```

Use a passphrase on the private key for additional protection against theft of the admin workstation.

### 2. Install SSH Public Key on RPi5

While you still have console access (or before password auth is disabled), install the public key.

```bash
# On RPi5 (as root or using su)
# If admin user doesn't exist yet, create them
adduser -D <ADMIN_USER>     # -D: no password set
passwd -l <ADMIN_USER>      # Lock the password (key-only)

# Install the public key
mkdir -p /home/<ADMIN_USER>/.ssh
chmod 700 /home/<ADMIN_USER>/.ssh
chown <ADMIN_USER>:<ADMIN_USER> /home/<ADMIN_USER>/.ssh

# Paste the public key content from cat ~/.ssh/vpn_gw.pub on your workstation
cat >> /home/<ADMIN_USER>/.ssh/authorized_keys << 'EOF'
ssh-ed25519 AAAA...<YOUR_PUBLIC_KEY_CONTENT_HERE> admin@vpn-gw-2025
EOF

chmod 600 /home/<ADMIN_USER>/.ssh/authorized_keys
chown <ADMIN_USER>:<ADMIN_USER> /home/<ADMIN_USER>/.ssh/authorized_keys
```

### 3. Harden sshd_config on RPi5

Write the full hardened SSH configuration:

```bash
cat > /etc/ssh/sshd_config << 'EOF'
# /etc/ssh/sshd_config — WireGuard gateway RPi5
# All defaults are overridden explicitly.

# -------------------------------------------------------
# Binding
# -------------------------------------------------------
# Bind only to the DMZ interface IP, not to all interfaces.
# This means sshd is not accessible on:
# - wg0 (WireGuard tunnel interface) — prevents tunnel-IP SSH attempts
# - lo (loopback)
# If a future vulnerability re-enables 0.0.0.0, pfSense rules still protect.
ListenAddress <DMZ_RPi5_IP>
Port 22

# -------------------------------------------------------
# Authentication
# -------------------------------------------------------
# Disable root login — use a named admin user with sudo if needed
PermitRootLogin no

# Disable all password-based authentication
# After keys are installed and verified, this removes the password attack surface
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no

# Disable PAM (not needed with key-only auth on Alpine)
UsePAM no

# Enable public key authentication only
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Accept only modern key algorithms
# Removes DSA, old RSA, and weak ECDSA curves
PubkeyAcceptedKeyTypes ssh-ed25519,ecdsa-sha2-nistp256,ecdsa-sha2-nistp384,ecdsa-sha2-nistp521,rsa-sha2-256,rsa-sha2-512

# Empty passwords are never permitted, regardless of PAM config
PermitEmptyPasswords no

# -------------------------------------------------------
# Access control
# -------------------------------------------------------
# Only this specific user may log in via SSH
# Any other user with a valid key is still rejected
AllowUsers <ADMIN_USER>

# -------------------------------------------------------
# Tunneling and forwarding
# -------------------------------------------------------
# Disable all forwarding — this is a WireGuard gateway, not a jump host
AllowTcpForwarding no
X11Forwarding no
AllowAgentForwarding no
PermitTunnel no
GatewayPorts no

# -------------------------------------------------------
# Timeouts and limits
# -------------------------------------------------------
# Disconnect idle sessions after 10 minutes (300s × 2 = 600s max)
ClientAliveInterval 300
ClientAliveCountMax 2

# How long to wait for authentication before disconnecting unauthenticated connections
# Short value reduces the window for brute-force timing attacks
LoginGraceTime 30

# Limit failed authentication attempts per connection
MaxAuthTries 3

# Limit concurrent pre-auth connections (prevents connection flood)
# Format: start:rate:full — starts throttling at 3, 50% rate, drops at 10
MaxStartups 3:50:10

# -------------------------------------------------------
# Crypto hardening
# -------------------------------------------------------
# Restrict to modern, vetted algorithms only
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes128-gcm@openssh.com
MACs hmac-sha2-256-etm@openssh.com,hmac-sha2-512-etm@openssh.com,umac-128-etm@openssh.com

# -------------------------------------------------------
# Logging
# -------------------------------------------------------
# INFO logs successful logins and failures; VERBOSE adds key fingerprints
LogLevel INFO
SyslogFacility AUTH

# -------------------------------------------------------
# Misc
# -------------------------------------------------------
Protocol 2
PrintMotd no
EOF
```

Verify the config syntax before restarting:

```bash
sshd -t
# Expected: no output (no errors)
# If errors appear, fix them before restarting sshd

rc-service sshd restart
```

### 4. Test Key Auth Before Locking Down

**Do not proceed until you have successfully logged in with the SSH key.** If you lock down sshd and key auth doesn't work, you will need physical console access to recover.

```bash
# From admin workstation on MGMT VLAN
ssh -i ~/.ssh/vpn_gw -v <ADMIN_USER>@<DMZ_RPi5_IP>
# Expected: successful login (no password prompt)
# The -v flag shows verbose auth debug info if something is wrong
```

If it fails, check `/var/log/auth.log` on RPi5 for the rejection reason and fix it before continuing.

### 5. Add ~/.ssh/config on Admin Workstation for Convenience

```
# ~/.ssh/config on admin workstation
Host vpn-gw
    HostName <DMZ_RPi5_IP>
    User <ADMIN_USER>
    IdentityFile ~/.ssh/vpn_gw
    IdentitiesOnly yes
    Port 22
```

Now connect with just: `ssh vpn-gw`

### 6. Add pfSense Firewall Rules for SSH Enforcement

These rules enforce the MGMT-only SSH restriction at the network layer, independent of sshd_config.

Navigate to **Firewall > Rules > DMZ** in pfSense.

Add a block rule at the top (highest priority) to block SSH from WireGuard tunnel IPs:

| Field | Value | Reason |
|-------|-------|--------|
| Action | Block | Silent drop |
| Interface | DMZ | |
| Protocol | TCP | |
| Source | `WG_TUNNEL` (10.0.0.0/24) | Tunnel IPs cannot SSH to RPi5 |
| Destination | `<DMZ_RPi5_IP>` | |
| Destination port | 22 | |
| Log | checked | |
| Description | `Block tunnel IPs → RPi5 SSH` | |

Note: This rule also prevents a compromised WireGuard peer from attempting to SSH to the RPi5 via the tunnel.

Navigate to **Firewall > Rules > MGMT** in pfSense.

Add an allow rule for SSH from MGMT to DMZ:

| Field | Value |
|-------|-------|
| Action | Pass |
| Interface | MGMT |
| Protocol | TCP |
| Source | `<MGMT_SUBNET>` |
| Destination | `<DMZ_RPi5_IP>` |
| Destination port | 22 |
| Description | `MGMT → RPi5 SSH` |

Any traffic not explicitly allowed in MGMT rules is already blocked by pfSense's default deny.

### 7. Harden pfSense Admin UI Access

Navigate to **System > Advanced > Admin Access** in pfSense.

| Setting | Value | Reason |
|---------|-------|--------|
| Protocol | `HTTPS` | Never HTTP |
| TCP port | `443` (or custom like `8443`) | |
| webConfigurator login protection | enabled | Rate-limit brute force |
| Max login attempts | `3` | |
| Anti-lockout rule | `enabled` (default) | Protects LAN access to UI |
| SSH | `enabled` | |
| SSH port | `22` | |
| Authentication method | `Public Key Only` | Disable password auth |

Navigate to **System > User Manager**. For the admin user:
- Add your SSH public key under **Authorized SSH Keys**
- Set a strong password as backup (required if SSH key fails)

Navigate to **Firewall > Rules > MGMT**. Ensure the pfSense admin UI (port 443) is accessible:

| Action | Source | Destination | Port | Description |
|--------|--------|-------------|------|-------------|
| Pass | `<MGMT_SUBNET>` | `This Firewall` | 443/TCP | MGMT → pfSense UI |
| Pass | `<MGMT_SUBNET>` | `This Firewall` | 22/TCP | MGMT → pfSense SSH |

### 8. Commit on RPi5

```bash
lbu commit -d
```

## Verification

```bash
# sshd listening only on DMZ IP
ss -tlnp | grep ':22'
# Expected: *:22 NOT shown; only <DMZ_RPi5_IP>:22

# SSH from MGMT VLAN succeeds
ssh vpn-gw
# Expected: shell prompt, no password

# SSH from wrong subnet (test from a LAN device)
ssh <ADMIN_USER>@<DMZ_RPi5_IP>
# Expected: connection refused or timeout (pfSense blocks LAN→DMZ:22)
# If LAN→DMZ:22 is not explicitly blocked, the default DMZ deny handles it

# Verify pfSense logs block entry when SSH is attempted from tunnel IP
# Connect a WireGuard peer, attempt: ssh <ADMIN_USER>@<DMZ_RPi5_IP> from the tunnel
# Check Status > System Logs > Firewall for block entry matching source=10.0.0.x, dst=<DMZ_RPi5_IP>:22
```

## Troubleshooting

**SSH key auth works but then fails after RPi5 reboot**
- `authorized_keys` file was not committed: `lbu commit -d`
- Verify `/home/<ADMIN_USER>/.ssh/authorized_keys` exists after reboot: `ls -la /home/<ADMIN_USER>/.ssh/`

**sshd -t reports "Invalid algorithm" for KexAlgorithms/Ciphers**
- Alpine's OpenSSH may not support all listed algorithms depending on the version
- Remove the unsupported algorithm from the list; check available algorithms with `ssh -Q kex`, `ssh -Q cipher`, `ssh -Q mac`

**Cannot SSH from MGMT VLAN despite correct key**
- Check pfSense has a MGMT→DMZ:22 allow rule
- Verify the admin workstation is actually on MGMT VLAN (check IP address — it should be in `<MGMT_SUBNET>`)
- Use `ssh -v` for verbose output; check which step fails

**Locked out — no console access**
- Boot from another Alpine SD card in read-only mode
- Mount the existing SD card's data partition
- Edit `authorized_keys` or temporarily re-enable password auth in `sshd_config`
- The `apkovl` archive is a tar.gz — extract it, edit, re-compress, replace

## Security Notes

**Remove the HDMI monitor and USB keyboard after confirming SSH works.** Physical console access bypasses all SSH restrictions. The console should only be used for emergency recovery, not routine administration.

**The `AllowUsers` directive is the last line of defense in sshd.** Even if an attacker obtains a valid private key for a user not in `AllowUsers`, they cannot log in. Combine with `passwd -l` to ensure no user has a usable password.

**pfSense anti-lockout rule**: pfSense has a built-in anti-lockout rule that always permits access to the admin UI from the LAN interface. This cannot be disabled and prevents accidental complete lockout. The MGMT restriction is additive — it restricts who from the LAN (or other interfaces) can reach the UI, but the LAN anti-lockout always provides a fallback.

**Rotate SSH keys annually or when the admin workstation changes.** Generate a new keypair, add the new public key to `authorized_keys`, verify login, remove the old key, commit.

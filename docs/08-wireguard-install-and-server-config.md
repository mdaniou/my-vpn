# WireGuard Install and Server Configuration

## Overview

This document installs WireGuard on the RPi5 and configures it as the VPN server. This includes key generation, server interface configuration, per-peer configuration with PresharedKeys, local iptables rules on the RPi5, and enabling the WireGuard service at boot.

After completing this document, the RPi5 will accept encrypted WireGuard connections on UDP 51820 and forward authenticated peer traffic toward the LAN (subject to pfSense DMZ→LAN rules).

## Prerequisites

- `07-alpine-hardening-and-network-config.md` completed: Alpine hardened, IP forwarding enabled, SSH working
- pfSense WAN port forward (doc 04) and DMZ→LAN rules (doc 05) already in place
- Client public keys available (if you pre-generate keys on each client device — see Security Notes for the key generation strategy)

## Security Objectives

- Server private key generated locally on the RPi5, never transmitted over the network
- PresharedKey for every peer adds a symmetric encryption layer on top of WireGuard's Curve25519 key exchange (post-quantum resistance)
- Server-side AllowedIPs set to each peer's `/32` tunnel IP — the server will not accept spoofed source IPs from peers
- Local iptables rules on RPi5 enforce a second layer of forwarding policy (defense in depth beyond pfSense)
- `SaveConfig = false` prevents WireGuard from overwriting the config file with transient runtime state

## Steps

### 1. Verify WireGuard Module Is Available

Alpine Linux ≥3.17 includes the WireGuard kernel module built-in. `wireguard-tools` (installed in doc 06) provides the userspace commands.

```bash
# Confirm WireGuard kernel support
modprobe wireguard 2>&1 || echo "Already built-in or loaded"

# Verify tools are available
wg --version
wg-quick --version
```

If `wg` is not found, run `apk add wireguard-tools` and `lbu commit -d`.

### 2. Create the WireGuard Config Directory

```bash
# Create directory with strict permissions — private keys live here
mkdir -p /etc/wireguard
chmod 700 /etc/wireguard
```

`chmod 700` ensures only root can list or read files in this directory. This protects private keys and PSKs from any non-root process that might somehow be running on the system.

### 3. Generate Server Key Pair

```bash
# Generate private key, derive and store public key simultaneously
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key

# Lock down permissions immediately after generation
chmod 600 /etc/wireguard/server_private.key
chmod 644 /etc/wireguard/server_public.key  # public key can be readable

# Display the public key — you will need to share this with clients
cat /etc/wireguard/server_public.key
# Output: a 44-character base64 string, e.g. ABC123.../+xyz=
# Record this as <SERVER_PUBLIC_KEY>
```

Key generation is done on the RPi5 because:
1. The private key never leaves the machine where it was generated
2. The RPi5's `/dev/random` entropy source is sufficient (Alpine uses the kernel's CSPRNG)
3. Generating on a laptop and copying would require secure transmission of a secret — unnecessary risk

### 4. Generate Per-Peer PresharedKeys

Each peer gets a unique PSK. A PSK is a symmetric secret shared between the server and one specific peer. It adds a symmetric layer of encryption on top of WireGuard's asymmetric (Curve25519) handshake, providing defense against future quantum computing attacks that could break Curve25519.

```bash
# Generate PSK for laptop peer
wg genpsk > /etc/wireguard/psk_laptop.key
chmod 600 /etc/wireguard/psk_laptop.key

# Generate PSK for phone peer
wg genpsk > /etc/wireguard/psk_phone.key
chmod 600 /etc/wireguard/psk_phone.key

# View PSK content (you will need to distribute this to the peer — see doc 10)
cat /etc/wireguard/psk_laptop.key
# Output: a 44-character base64 string
# Record as <PSK_LAPTOP>
```

PSKs are stored in separate files (not inline in `wg0.conf`) so that the config file can be committed via `lbu` while the PSK files remain excluded (see the `lbu` exclusion set up in doc 06).

### 5. Obtain Client Public Keys

Each client must generate its own private/public keypair (see doc 10). You need the **public key** from each client to configure the server. The public key is not secret.

For the laptop:
```bash
# On the laptop (Linux/macOS), generate a keypair:
wg genkey | tee ~/.wireguard/laptop_private.key | wg pubkey > ~/.wireguard/laptop_public.key
cat ~/.wireguard/laptop_public.key
# Record as <LAPTOP_PUBLIC_KEY>
```

For the phone: generate from the WireGuard Android/iOS app and copy the public key.

### 6. Write the Server Configuration

```bash
cat > /etc/wireguard/wg0.conf << 'EOF'
[Interface]
# WireGuard tunnel interface address — the server's IP within the VPN tunnel
Address = 10.0.0.1/24

# Listen on all interfaces (0.0.0.0) so the DMZ ethernet interface receives connections
# The pfSense NAT forwards inbound UDP 51820 to <DMZ_RPi5_IP>, which arrives on eth0
ListenPort = 51820

# Server private key — read from file at runtime; do not inline if you want to exclude
# from lbu (see Security Notes)
PrivateKey = <SERVER_PRIVATE_KEY_CONTENT>
# Substitute the actual key: $(cat /etc/wireguard/server_private.key)

# Prevent wg-quick from writing runtime peer state back to this file.
# Without this, wg-quick would overwrite wg0.conf with runtime data (last handshake times,
# transfer bytes) on wg-quick down, making the file dirty and complicating lbu commits.
SaveConfig = false

# PostUp: iptables rules applied when the interface comes up
# These enforce local forwarding policy on the RPi5 itself.
# pfSense also filters this traffic — these rules are defense in depth.
PostUp = iptables -A FORWARD -i wg0 -o eth0 -j ACCEPT
PostUp = iptables -A FORWARD -i eth0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
PostUp = iptables -A INPUT -i eth0 -p udp --dport 51820 -j ACCEPT

# PostDown: remove iptables rules when the interface goes down
# Avoids duplicate rules if wg-quick is restarted
PostDown = iptables -D FORWARD -i wg0 -o eth0 -j ACCEPT
PostDown = iptables -D FORWARD -i eth0 -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -D INPUT -i eth0 -p udp --dport 51820 -j ACCEPT

# ============================================================
# Peer: Laptop
# ============================================================
[Peer]
# Laptop's WireGuard public key (generated on the laptop)
PublicKey = <LAPTOP_PUBLIC_KEY>

# PSK for this peer — symmetric layer, unique to this peer
PresharedKey = <PSK_LAPTOP_CONTENT>
# Substitute: $(cat /etc/wireguard/psk_laptop.key)

# AllowedIPs = the tunnel IP address assigned to this peer
# /32 means: only packets sourced from exactly 10.0.0.2 are accepted from this peer.
# Using the full /24 here would allow this peer to spoof other peers' IPs.
AllowedIPs = 10.0.0.2/32

# No Endpoint here — the server accepts connections from any address.
# The client's Endpoint points to the server; the server learns the client's
# address from the first authenticated handshake packet.

# ============================================================
# Peer: Phone
# ============================================================
[Peer]
PublicKey = <PHONE_PUBLIC_KEY>
PresharedKey = <PSK_PHONE_CONTENT>
# Substitute: $(cat /etc/wireguard/psk_phone.key)
AllowedIPs = 10.0.0.3/32
EOF

chmod 600 /etc/wireguard/wg0.conf
```

**Note on inlining keys**: The config above requires inlining the actual key material. This is a trade-off — `wg0.conf` can be read by any future vulnerability in a root process. An alternative is to use `PostUp = wg set wg0 private-key /etc/wireguard/server_private.key` and keep the `PrivateKey` line pointing to the file path. Either approach is acceptable; see Security Notes.

To substitute key values:
```bash
# Read actual key content and insert
SERVER_PRIVKEY=$(cat /etc/wireguard/server_private.key)
PSK_LAPTOP=$(cat /etc/wireguard/psk_laptop.key)
PSK_PHONE=$(cat /etc/wireguard/psk_phone.key)

# Edit wg0.conf, replacing placeholders with actual values
sed -i "s|<SERVER_PRIVATE_KEY_CONTENT>|${SERVER_PRIVKEY}|" /etc/wireguard/wg0.conf
sed -i "s|<PSK_LAPTOP_CONTENT>|${PSK_LAPTOP}|" /etc/wireguard/wg0.conf
sed -i "s|<PSK_PHONE_CONTENT>|${PSK_PHONE}|" /etc/wireguard/wg0.conf
# Substitute <LAPTOP_PUBLIC_KEY> and <PHONE_PUBLIC_KEY> manually
```

### 7. Configure the Default iptables FORWARD Policy

Before bringing up the interface, set the default FORWARD chain to DROP. This ensures that without the explicit PostUp rules, no forwarding occurs:

```bash
# Set default policy: drop all forwarded packets unless explicitly allowed
iptables -P FORWARD DROP

# Also set INPUT and OUTPUT policies
iptables -P INPUT DROP
iptables -P OUTPUT ACCEPT  # Allow all outgoing from RPi5 itself

# Allow established/related inbound (for RPi5's own connections, e.g. NTP, DNS)
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow SSH from MGMT VLAN only
iptables -A INPUT -i eth0 -s <MGMT_SUBNET> -p tcp --dport 22 -j ACCEPT

# Allow loopback
iptables -A INPUT -i lo -j ACCEPT
```

Persist these rules so they apply before WireGuard comes up. On Alpine, use the `iptables` service:

```bash
# Save current iptables rules
iptables-save > /etc/iptables/rules-save

# Enable the iptables service to restore rules at boot
rc-update add iptables default
rc-service iptables save
```

### 8. Start WireGuard

```bash
# Bring up the WireGuard interface
wg-quick up wg0

# Verify the interface is up
wg show wg0
```

Expected output from `wg show wg0`:
```
interface: wg0
  public key: <SERVER_PUBLIC_KEY>
  private key: (hidden)
  listening port: 51820

peer: <LAPTOP_PUBLIC_KEY>
  preshared key: (hidden)
  allowed ips: 10.0.0.2/32
  transfer: 0 B received, 0 B sent

peer: <PHONE_PUBLIC_KEY>
  preshared key: (hidden)
  allowed ips: 10.0.0.3/32
  transfer: 0 B received, 0 B sent
```

No "latest handshake" is shown until a client connects — this is expected.

### 9. Enable WireGuard at Boot

```bash
# Alpine uses the wg-quick init script
# Enable wg0 to start automatically in the default runlevel
rc-update add wg-quick.wg0 default

# Verify
rc-update show default | grep wg
# Expected: wg-quick.wg0
```

### 10. Commit All Changes

```bash
lbu commit -d
```

Check what was committed — verify that private key files are excluded:

```bash
lbu list | grep wireguard
# wg0.conf should appear
# server_private.key, psk_*.key should NOT appear (they are in lbu exclude)
```

If private keys appear in `lbu list`, verify `/etc/lbu/exclude` contains the patterns set in doc 06.

## Verification

```bash
# Interface is up and listening
ip addr show wg0
# Expected: inet 10.0.0.1/24

ss -ulnp | grep 51820
# Expected: udp UNCONN ... *:51820 ... ("wg-quick")

# iptables FORWARD rules are in place
iptables -L FORWARD -v -n
# Expected: rules for wg0 → eth0 ACCEPT and eth0 → wg0 ESTABLISHED,RELATED ACCEPT

# Verify WireGuard peers are configured
wg show wg0 peers
# Expected: both peer public keys listed

# Test from a connected client (after client config — doc 10)
# On RPi5, watch for handshake:
watch wg show wg0
# After client connects: "latest handshake: X seconds ago" appears under each connected peer
```

## Troubleshooting

**`wg-quick up wg0` fails with "RTNETLINK answers: Operation not permitted"**
- WireGuard module not loaded: `modprobe wireguard`
- Alpine kernel too old: verify with `uname -r`, requires ≥5.6 (Alpine 3.17+)

**`wg show` shows peer but no handshake after client connects**
- Key mismatch: verify the public key in `wg0.conf` matches the client's actual public key
- PSK mismatch: PSK must be identical on both server and client
- Client Endpoint points to wrong IP or port: verify `<WAN_STATIC_IP>:51820`
- pfSense NAT not forwarding: check doc 04 verification steps

**Packets arrive at RPi5 but are not forwarded to LAN**
- Check IP forwarding: `sysctl net.ipv4.ip_forward` — must be `1`
- Check FORWARD chain: `iptables -L FORWARD -v -n` — the `wg0 → eth0` rule must be present
- Check pfSense DMZ→LAN rules (doc 05) — the peer's tunnel IP must have an allow rule

**wg-quick.wg0 service fails to start at boot**
- Check `/var/log/messages` for the error
- Common cause: iptables rules applied before wg0 comes up, leaving stale rules — check PostUp syntax

## Security Notes

**Private key in wg0.conf**: If `wg0.conf` is committed via lbu (which it should be, excluding the key files), the private key is in `/etc/wireguard/wg0.conf` on the SD card. Set the lbu exclude and use a separate file reference if you want to keep the private key off the SD card entirely:
```
PrivateKey = (leave blank)
PostUp = wg set wg0 private-key /etc/wireguard/server_private.key
```
This keeps the private key only in RAM after boot. The trade-off: the key must be re-entered or restored from a secure backup after every reboot. For a home lab with infrequent reboots, this may be acceptable.

**AllowedIPs /32 per peer**: Never use `0.0.0.0/0` or the entire `10.0.0.0/24` as AllowedIPs for a peer on the server side. AllowedIPs tells WireGuard which source IP addresses are legitimate for that peer. A `/32` means only that peer's assigned IP is accepted — another peer cannot spoof that IP.

**PersistentKeepalive on server**: Do not add `PersistentKeepalive` to server-side peer entries. Keepalives are only needed on the client side (to maintain NAT table entries at the client's ISP). The server does not need to initiate contact.

**Key rotation**: Schedule PSK rotation every 90 days and private key rotation annually. See `14-maintenance-and-troubleshooting.md` for the rotation procedure.

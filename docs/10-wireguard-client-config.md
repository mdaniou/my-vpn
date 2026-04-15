# WireGuard Client Configuration

## Overview

This document covers WireGuard client configuration for the two initial peers: a Linux laptop and a mobile phone (Android/iOS). Both are configured as split-tunnel clients — only homelab traffic routes through the VPN, while general internet traffic continues to use the local ISP directly.

## Prerequisites

- `08-wireguard-install-and-server-config.md` completed: WireGuard server running on RPi5
- `09-wireguard-peer-management.md` reviewed: understand how to register a new peer on the server
- Server public key available: `<SERVER_PUBLIC_KEY>`
- PSK for each peer generated on the server and transmitted securely: `<PSK_LAPTOP>`, `<PSK_PHONE>`
- Tunnel IPs assigned: laptop = `10.0.0.2`, phone = `10.0.0.3`

## Security Objectives

- Client private keys are generated on the client device — they never exist on the server or transit the network
- Split tunnel: internet traffic never routes through the VPN (reduces exposure, preserves WAN performance)
- DNS queries for homelab resources resolve via Pi-hole through the tunnel — no DNS leaks to public resolvers when connected
- PresharedKey provides post-quantum protection on every connection
- Config file permissions are locked to owner-only (600)

## Steps

### 1. Laptop — Generate Client Keypair (Linux/macOS)

Key generation on the client device ensures the private key is never transmitted.

```bash
# Create a directory for WireGuard keys — keep separate from /etc/wireguard
# which is the server location; on a client, keys typically live in the config dir
mkdir -p ~/.wireguard
chmod 700 ~/.wireguard

# Generate keypair
wg genkey | tee ~/.wireguard/laptop_private.key | wg pubkey > ~/.wireguard/laptop_public.key

# Lock private key permissions immediately
chmod 600 ~/.wireguard/laptop_private.key
chmod 644 ~/.wireguard/laptop_public.key

# Display the public key — send this to the RPi5 operator to register as a peer
cat ~/.wireguard/laptop_public.key
# Record as <LAPTOP_PUBLIC_KEY>
```

Register this public key on the server per the procedure in doc 09 (Section 2) before writing the client config — you'll need the PSK that the server generates in return.

### 2. Laptop — Write the Client Config (Linux)

```bash
# On Linux, wg-quick reads from /etc/wireguard/wg0.conf by default
# Create the config file with strict permissions
sudo install -m 600 /dev/null /etc/wireguard/wg0.conf
sudo tee /etc/wireguard/wg0.conf << 'EOF'
[Interface]
# This client's tunnel IP address — must match AllowedIPs on server for this peer
Address = 10.0.0.2/24

# Private key for this client — generated locally, never shared
PrivateKey = <LAPTOP_PRIVATE_KEY_CONTENT>

# Route DNS queries through Pi-hole when connected to the VPN
# This prevents homelab hostnames from leaking to public DNS resolvers,
# and ensures .local / homelab domains resolve correctly.
# Only takes effect when the tunnel is up (wg-quick manages resolv.conf).
DNS = <PIHOLE_IP>

# Optional: MTU tuning
# WireGuard adds 60-byte overhead. Default MTU of 1420 is conservative and correct
# for most ISP connections. If you see fragmentation issues, try lower values.
# MTU = 1420

[Peer]
# Server's public key — the RPi5 WireGuard gateway
PublicKey = <SERVER_PUBLIC_KEY>

# PresharedKey — symmetric layer, unique to this peer
# Generated on the server and transmitted to you via secure channel
PresharedKey = <PSK_LAPTOP>

# Server endpoint — the public WAN IP and WireGuard port
Endpoint = <WAN_STATIC_IP>:51820

# Split tunnel: route only homelab subnets through the VPN
# - 10.0.0.0/24: reach other VPN peers and the server itself
# - <LAN_SUBNET>: reach homelab devices (Proxmox, Pi-hole, Mac Minis, etc.)
# NOT 0.0.0.0/0: internet traffic goes via local ISP, not through the VPN
AllowedIPs = 10.0.0.0/24, <LAN_SUBNET>

# Keepalive: send a packet every 25 seconds to maintain the NAT table entry
# at your ISP. Without this, a NAT mapping that times out (typically 30–120s for UDP)
# causes a re-handshake delay on the next packet.
# 25 seconds is below most ISP NAT UDP timeout thresholds.
PersistentKeepalive = 25
EOF
```

### 3. Laptop — Connect and Disconnect

```bash
# Bring up the VPN tunnel
sudo wg-quick up wg0

# Check tunnel status
wg show wg0
# Expected after server handshake:
# latest handshake: X seconds ago
# transfer: X B received, X B sent

# Verify tunnel IP is assigned
ip addr show wg0
# Expected: inet 10.0.0.2/24

# Take down the tunnel
sudo wg-quick down wg0
```

### 4. Laptop — Optional: Enable Autostart at Boot

Only enable autostart if the laptop is a desktop or server that should always be connected. For a laptop that travels, on-demand connection is more appropriate.

```bash
# systemd (most Linux distributions)
sudo systemctl enable wg-quick@wg0

# To start/stop:
sudo systemctl start wg-quick@wg0
sudo systemctl stop wg-quick@wg0
```

Trade-off: autostart means the VPN tries to connect from any network (including coffee shops, hotel WiFi). This is safe but may delay connectivity if `<WAN_STATIC_IP>` is unreachable. On-demand gives you more control.

### 5. Laptop — macOS Client

Install WireGuard via Homebrew or the Mac App Store:

```bash
# Homebrew approach (CLI)
brew install wireguard-tools
```

Config file location on macOS: `/usr/local/etc/wireguard/wg0.conf` (Homebrew) or `~/Library/wireguard/wg0.conf` (App Store version uses its own directory).

The config content is identical to the Linux version in Step 2. After creating the file:

```bash
# Using Homebrew wireguard-tools:
sudo wg-quick up wg0
sudo wg-quick down wg0
```

Using the macOS WireGuard App (App Store): import the config file via the app's "Import Tunnel(s) from File" option. The app provides a menu bar toggle for quick connect/disconnect.

### 6. Phone — Android/iOS Configuration

The WireGuard mobile apps can import config via QR code or file. The most secure method is QR code generated directly on the server, displayed once, and discarded.

#### Generate the phone config file on the RPi5:

```bash
# On RPi5, create the phone client config temporarily
cat > /tmp/phone_client.conf << EOF
[Interface]
PrivateKey = <PHONE_PRIVATE_KEY>
Address = 10.0.0.3/24
DNS = <PIHOLE_IP>

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
PresharedKey = $(cat /etc/wireguard/psk_phone.key)
Endpoint = <WAN_STATIC_IP>:51820
AllowedIPs = 10.0.0.0/24, <LAN_SUBNET>
PersistentKeepalive = 25
EOF
```

Note: `<PHONE_PRIVATE_KEY>` must be the private key generated on the phone (from the WireGuard app's "Add Tunnel > Create from Scratch" → "Generate keypair"). Copy the private key out of the app and paste it here.

Alternatively, generate the keypair for the phone on the RPi5 (see Security Notes for trade-offs):
```bash
wg genkey | tee /tmp/phone_private.key | wg pubkey > /tmp/phone_public.key
# Use the private key in phone_client.conf, register the public key on the server
```

#### Generate QR code for phone import:

```bash
# Install qrencode if not present
apk add qrencode

# Display QR code in terminal — scan with the WireGuard app
qrencode -t ansiutf8 < /tmp/phone_client.conf

# After scanning, immediately delete the temp files
rm /tmp/phone_client.conf /tmp/phone_private.key 2>/dev/null
# The QR code is no longer needed — the config is now in the phone app
```

On the phone, open the WireGuard app and use "Add Tunnel > Scan QR Code". Point the camera at the terminal. The tunnel is imported immediately.

#### Security precautions for QR code:
- Do this from a private terminal session — no screen recording, no one looking over your shoulder
- The QR code contains the private key in plaintext — do not screenshot it
- Delete `/tmp/phone_client.conf` immediately after scanning
- If you accidentally captured the QR code, regenerate the phone's keypair and re-register

### 7. Verify DNS Is Routing Through Pi-hole

When connected to the VPN:

```bash
# On laptop — check which DNS server is active
# (After wg-quick up, DNS should be set to <PIHOLE_IP>)
cat /etc/resolv.conf | grep nameserver
# Expected: nameserver <PIHOLE_IP>

# Verify DNS queries reach Pi-hole
dig @<PIHOLE_IP> google.com
# Expected: valid answer (Pi-hole forwards to upstream resolver)

# If Pi-hole has a block list, test that blocking works
dig @<PIHOLE_IP> doubleclick.net
# Expected: 0.0.0.0 (blocked by Pi-hole)
```

DNS leak test (from a web browser, use dnsleaktest.com or similar):
- While connected to VPN: the DNS servers shown should be your ISP's upstream resolvers (as configured in Pi-hole), NOT your local ISP's default resolvers and NOT Google/Cloudflare.
- If you see Google 8.8.8.8 or Cloudflare 1.1.1.1 as the resolver, Pi-hole is not being used and DNS is leaking.

### 8. Verify Split Tunnel Is Working

```bash
# While connected to VPN
# Check your public IP — should be your local ISP's IP, NOT <WAN_STATIC_IP>
curl https://ifconfig.me
# Expected: your local ISP-assigned IP (same as when VPN is disconnected)
# If it returns <WAN_STATIC_IP>, you have a full tunnel — check AllowedIPs config

# Verify homelab subnet routes through tunnel
ip route show | grep <LAN_SUBNET>
# Expected: <LAN_SUBNET> dev wg0 ...
# This means LAN-bound traffic goes via the tunnel

# Verify internet routes do NOT go through tunnel
ip route show | grep "default"
# Expected: default via <LOCAL_GATEWAY> dev <LOCAL_INTERFACE>
# The default route should still point to your local ISP gateway
```

## Verification

| Test | Command | Expected Result |
|------|---------|-----------------|
| Tunnel interface up | `ip addr show wg0` | `inet 10.0.0.2/24` |
| Server handshake | `wg show wg0` | `latest handshake: X seconds ago` |
| Ping VPN server | `ping 10.0.0.1` | Replies received |
| Reach LAN device | `ping <PROXMOX_IP>` | Replies received |
| Internet via local ISP | `curl ifconfig.me` | Local ISP IP, not `<WAN_STATIC_IP>` |
| DNS via Pi-hole | `dig @<PIHOLE_IP> example.com` | Resolved answer |
| No IPv6 tunnel traffic | `ping6 ::1` or `ip -6 route` | No IPv6 routes via wg0 |

## Troubleshooting

**Handshake fails immediately after config is set up**
- Key mismatch: verify `PublicKey` in client config matches `server_public.key` content
- PSK mismatch: verify the PSK was copied correctly (no trailing newline or whitespace)
- Clock skew: WireGuard uses timestamps in the handshake to prevent replay attacks. If the client clock is more than a few minutes off the server clock, handshakes fail. Sync time: `sudo ntpdate pool.ntp.org` or enable NTP.

**Handshake succeeds but cannot reach LAN**
- Check `AllowedIPs` includes `<LAN_SUBNET>` (not just `10.0.0.0/24`)
- Check pfSense DMZ→LAN rules allow this peer's tunnel IP (doc 05)
- Verify IP forwarding on RPi5: `sysctl net.ipv4.ip_forward` = 1

**DNS not working via Pi-hole**
- Verify `DNS = <PIHOLE_IP>` is in the `[Interface]` section of the client config
- Verify `wg-quick` is responsible for DNS management (it rewrites `/etc/resolv.conf` on up)
- On macOS: DNS changes may require a few seconds to propagate; try `sudo dscacheutil -flushcache`
- Check pfSense: does the peer's tunnel IP have a DMZ→LAN rule allowing TCP/UDP port 53 to `<PIHOLE_IP>`?

**wg-quick fails: "unable to modify interface" or "RTNETLINK"**
- On Linux, ensure `wireguard` kernel module is loaded: `sudo modprobe wireguard`
- Verify you are running as root or with `sudo`

## Security Notes

**Phone keypair generation trade-off**: If you generate the phone's keypair on the RPi5 for convenience, the phone's private key exists (briefly) on the server. This is a weaker security posture than generating it on the phone. If the RPi5 is ever compromised after this point, the phone's private key may have been exposed. For maximum security, generate on the device; for convenience (especially for a home lab), generating on the server and loading via QR is acceptable if done carefully.

**Never add `0.0.0.0/0` to `AllowedIPs` unless you intend a full tunnel.** A full tunnel routes all traffic (including internet browsing) through the RPi5 and out to pfSense's WAN. This increases latency, causes pfSense WAN traffic from your phone/laptop to appear as your home IP, and is unnecessary for homelab access. It also violates the split-tunnel security principle for this setup.

**Protect the client config file.** The `wg0.conf` on the laptop contains the private key in plaintext. Ensure full-disk encryption is enabled on the laptop (FileVault on macOS, LUKS on Linux). At minimum, `chmod 600 /etc/wireguard/wg0.conf`.

**The WireGuard mobile app stores keys in the platform keychain.** On iOS and Android, private keys are stored in the OS-protected keychain/keystore. Revoking the peer on the server (doc 09) is sufficient to revoke access even if the device is lost — the private key on the device cannot generate valid traffic once the server removes the peer entry.

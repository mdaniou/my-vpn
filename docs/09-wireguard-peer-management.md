# WireGuard Peer Management

## Overview

This document covers the operational workflows for managing WireGuard peers: adding new peers, revoking existing peers, rotating keys, and maintaining a peer inventory. Each operation requires coordinated changes on both the RPi5 server and pfSense.

The guiding principle is that adding a peer is a two-step process: WireGuard configuration (what the VPN allows) and pfSense rules (what the network allows). A peer with a WireGuard entry but no pfSense rules has tunnel connectivity but cannot reach any LAN resource — this is intentional.

## Prerequisites

- `08-wireguard-install-and-server-config.md` completed: WireGuard server running with initial peers
- pfSense DMZ→LAN rules in place (doc 05)
- SSH access to RPi5 from MGMT VLAN

## Security Objectives

- Each peer has a unique tunnel IP, unique keypair, and unique PSK — compromise of one peer does not expose others
- Peer revocation takes effect immediately without restarting WireGuard (no connection interruption for other peers)
- pfSense rules are updated atomically with WireGuard changes — no window where a peer has access it shouldn't
- Key rotation is documented and scheduled to minimize the impact of undetected key compromise

## Steps

### 1. Peer Inventory Table

Maintain this table in your secure notes (not committed to git — it contains partial key information):

| Peer Name | Tunnel IP | Public Key (last 8 chars) | PSK File | Added | PSK Last Rotated | Key Last Rotated | pfSense Rules |
|-----------|-----------|---------------------------|----------|-------|-----------------|-----------------|---------------|
| laptop    | 10.0.0.2  | `...abc12345`             | `psk_laptop.key` | 2025-01-01 | 2025-01-01 | 2025-01-01 | Rules 3–5 in DMZ tab |
| phone     | 10.0.0.3  | `...xyz98765`             | `psk_phone.key`  | 2025-01-01 | 2025-01-01 | 2025-01-01 | Rule 6 in DMZ tab |

Assign tunnel IPs sequentially starting from `10.0.0.2`. Never reuse a tunnel IP until you are certain the previous peer using it is fully revoked and all pfSense state table entries for that IP have expired.

### 2. Adding a New Peer

This procedure adds a peer named `<NEW_PEER>` with tunnel IP `<NEW_PEER_IP>` (e.g. 10.0.0.4).

#### Step A: Generate keys on the server

```bash
# On RPi5
# Generate the PSK for this peer (server-side, then distribute to peer)
wg genpsk > /etc/wireguard/psk_<NEW_PEER>.key
chmod 600 /etc/wireguard/psk_<NEW_PEER>.key

# Read PSK content — you will need to provide this to the peer's config
cat /etc/wireguard/psk_<NEW_PEER>.key
# Record as <PSK_NEW_PEER>
```

The new peer generates their own private/public keypair on their device (see doc 10). You need their public key: `<NEW_PEER_PUBLIC_KEY>`.

#### Step B: Add the peer to wg0.conf

```bash
# Append the new peer block to wg0.conf
cat >> /etc/wireguard/wg0.conf << EOF

# ============================================================
# Peer: <NEW_PEER>
# ============================================================
[Peer]
PublicKey = <NEW_PEER_PUBLIC_KEY>
PresharedKey = $(cat /etc/wireguard/psk_<NEW_PEER>.key)
AllowedIPs = <NEW_PEER_IP>/32
EOF
```

#### Step C: Add the peer to the running WireGuard instance (no restart needed)

```bash
# Hot-add the peer without disrupting existing connections
wg set wg0 peer <NEW_PEER_PUBLIC_KEY> \
  preshared-key /etc/wireguard/psk_<NEW_PEER>.key \
  allowed-ips <NEW_PEER_IP>/32

# Verify the peer appears
wg show wg0 peers
# Expected: <NEW_PEER_PUBLIC_KEY> is now in the list
```

#### Step D: Add pfSense firewall rules

In pfSense **Firewall > Aliases > IP**, add a new host alias:
- Name: `WG_<NEW_PEER>` (e.g. `WG_TABLET`)
- Type: Host
- IP: `<NEW_PEER_IP>`

Then in **Firewall > Rules > DMZ**, add allow rules for this peer with minimum required access (above the default deny rule). Example: if the new peer only needs Pi-hole DNS:

| Action | Source | Destination | Port | Description |
|--------|--------|-------------|------|-------------|
| Pass | `WG_<NEW_PEER>` | `<PIHOLE_IP>` | 53/TCP+UDP | New peer → DNS |

Click **Save**, then **Apply Changes**.

#### Step E: Persist on RPi5

```bash
lbu commit -d

# Verify psk file is excluded from commit
lbu list | grep psk_<NEW_PEER>
# Should NOT appear (excluded via /etc/lbu/exclude pattern)
```

#### Step F: Distribute config to the client

Provide the client with:
1. Server public key: `cat /etc/wireguard/server_public.key`
2. PSK: `cat /etc/wireguard/psk_<NEW_PEER>.key`
3. Assigned tunnel IP: `<NEW_PEER_IP>/24`
4. Server endpoint: `<WAN_STATIC_IP>:51820`
5. AllowedIPs (split tunnel): `10.0.0.0/24, <LAN_SUBNET>`

See doc 10 for the client config file format. Transmit the PSK and server public key via a secure channel (e.g. Signal, encrypted email, or in person).

### 3. Revoking a Peer

Revocation takes effect immediately — no WireGuard restart required.

#### Step A: Remove peer from running WireGuard instance

```bash
# Immediate revocation — takes effect within milliseconds
wg set wg0 peer <PEER_PUBLIC_KEY> remove

# Verify peer is gone
wg show wg0 peers
# Expected: <PEER_PUBLIC_KEY> no longer appears
```

This step alone prevents the peer from establishing new handshakes or sending authenticated traffic. Existing sessions (if any) will time out within the handshake expiry window (≈3 minutes by default).

#### Step B: Remove peer from wg0.conf

```bash
# Edit /etc/wireguard/wg0.conf
# Remove the entire [Peer] block for this peer:
# [Peer]
# PublicKey = <PEER_PUBLIC_KEY>
# PresharedKey = ...
# AllowedIPs = <PEER_IP>/32
vi /etc/wireguard/wg0.conf
```

#### Step C: Remove pfSense firewall rules

In pfSense **Firewall > Rules > DMZ**:
- Delete all rules with source `WG_<PEER>` alias
- Delete the `WG_<PEER>` alias from **Firewall > Aliases**
- Click **Apply Changes**

This ensures that even if the peer somehow retains old keys (e.g. exported config), the pfSense rules block their tunnel IP from reaching any LAN resource.

#### Step D: Delete PSK file

```bash
rm /etc/wireguard/psk_<PEER>.key
```

#### Step E: Persist

```bash
lbu commit -d
```

#### Step F: Clear pfSense state table

Navigate to **Diagnostics > States** in pfSense. Filter by source IP `<PEER_TUNNEL_IP>` and delete any matching states. This immediately terminates any active connections from that peer's IP.

### 4. Rotating a Peer's PSK (Every 90 Days)

PSK rotation does not require the peer to reconnect immediately. Plan a brief maintenance window (< 5 seconds of dropped connectivity).

```bash
# On RPi5
# Generate new PSK
wg genpsk > /etc/wireguard/psk_<PEER>_new.key
chmod 600 /etc/wireguard/psk_<PEER>_new.key
NEW_PSK=$(cat /etc/wireguard/psk_<PEER>_new.key)

# Update the running WireGuard instance
wg set wg0 peer <PEER_PUBLIC_KEY> preshared-key /etc/wireguard/psk_<PEER>_new.key

# Update wg0.conf with the new PSK value
# Replace the old PresharedKey line in the peer block
# (Use vi or sed — be precise to update only the correct peer's block)

# Replace old PSK file
mv /etc/wireguard/psk_<PEER>_new.key /etc/wireguard/psk_<PEER>.key

# Distribute new PSK to peer (via secure channel)
cat /etc/wireguard/psk_<PEER>.key

lbu commit -d
```

The peer must update their config with the new PSK before the next handshake (25-second window if PersistentKeepalive is 25). Coordinate with the peer.

### 5. Rotating a Peer's Keypair (Annual or After Compromise)

Full keypair rotation requires both a server-side and client-side config update. This causes a brief disconnection.

#### On the client device:
```bash
# Generate new keypair
wg genkey | tee ~/new_private.key | wg pubkey > ~/new_public.key
# Record <NEW_PEER_PUBLIC_KEY>
```

#### On RPi5:
```bash
# Step 1: Add new peer entry alongside old one (allows overlap period)
wg set wg0 peer <NEW_PEER_PUBLIC_KEY> \
  preshared-key /etc/wireguard/psk_<PEER>.key \
  allowed-ips <PEER_IP>/32

# Step 2: Update client config with new private key (doc 10)
# Step 3: Client reconnects with new key
# Step 4: Remove old peer
wg set wg0 peer <OLD_PEER_PUBLIC_KEY> remove

# Step 5: Update wg0.conf — replace old PublicKey with new
vi /etc/wireguard/wg0.conf

lbu commit -d
```

### 6. Emergency: Immediate Full Lockout

If a device is stolen or a key compromise is suspected:

```bash
# Immediately remove the peer from the running instance
wg set wg0 peer <COMPROMISED_PUBLIC_KEY> remove

# Then follow the full revocation procedure (Section 3)
# Prioritize: Steps A and C (WireGuard removal + pfSense rules) before anything else
```

The entire revocation via `wg set ... remove` takes effect in under a second. There is no need to restart WireGuard or the RPi5.

If you suspect the server private key itself is compromised (not just a peer key), you must regenerate the server keypair, update all peer configs with the new server public key, and re-establish all connections. This is a full re-deployment — see doc 14.

## Verification

After adding a peer:
```bash
# Peer appears in WireGuard
wg show wg0 | grep -A4 <NEW_PEER_PUBLIC_KEY>

# pfSense rule exists
# Check Firewall > Rules > DMZ in pfSense UI
```

After revoking a peer:
```bash
# Peer absent from WireGuard
wg show wg0 peers | grep <REVOKED_PUBLIC_KEY>
# Expected: no output

# From a device on the former peer's IP — attempt connection
# Expected: handshake fails, no response from server
```

## Troubleshooting

**Hot-add peer does not persist after reboot**
- `wg set` modifies the running instance only. You must also update `/etc/wireguard/wg0.conf` and run `lbu commit -d`.

**Peer shows handshake but cannot reach LAN**
- pfSense rules for the new peer's tunnel IP are missing or incorrect.
- Check the source IP in pfSense logs — confirm it is `<PEER_IP>` (tunnel IP), not `<DMZ_RPi5_IP>`.

**After PSK rotation, peer cannot reconnect**
- The client config still has the old PSK. Verify the client updated their config with the new PSK value.
- WireGuard PSK mismatch causes silent handshake failure — no error is logged on either side.

## Security Notes

**Never reuse tunnel IPs.** If peer `10.0.0.2` is revoked and you later add a new peer with `10.0.0.2`, old pfSense state table entries might briefly allow the new peer access via the old peer's still-active states. Wait for state table expiry (configurable in pfSense, default several minutes) before reusing an IP, or clear states manually.

**PSK compromise affects only one peer.** Each peer has its own PSK. If one PSK is exposed, rotate only that peer's PSK — other peers are unaffected. This is why per-peer PSKs are mandatory rather than a single shared PSK.

**Distributing PSKs and server public key**: Never send these over unencrypted channels (plain email, SMS, Slack). Use Signal, a password manager's secure sharing feature, or physical delivery.

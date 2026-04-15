# Verification and Testing

## Overview

This document is a comprehensive end-to-end test plan for the WireGuard VPN infrastructure. It validates every security control across all layers: pfSense firewall rules, WireGuard server, RPi5 host hardening, client split tunnel behavior, IPv6 suppression, and peer isolation.

Run these tests in order after completing the full deployment (docs 01–12). Re-run the full suite after any configuration change, key rotation, or system update.

## Prerequisites

- Docs 01–12 fully completed and applied
- Admin workstation connected to MGMT VLAN
- Laptop client configured with `wg0` profile (doc 10)
- Phone client configured (doc 10)
- SSH access to RPi5 from MGMT VLAN
- SSH access to pfSense from MGMT VLAN
- External network or hotspot available for WAN-side tests
- Tools available on laptop: `wg`, `wg-quick`, `ping`, `traceroute`, `curl`, `dig`, `nmap`, `ip`, `ss`, `nc`

## Security Objectives

- Confirm that only UDP 51820 is reachable from WAN — all other ports are silently dropped
- Confirm that VPN peers can only reach what pfSense DMZ→LAN rules explicitly permit
- Confirm that SSH to RPi5 is only possible from MGMT VLAN
- Confirm that IPv6 is fully disabled on all relevant devices
- Confirm that split tunnel is working — internet traffic does not traverse the VPN
- Confirm that WireGuard peers cannot reach each other laterally
- Confirm that all hardening survives a reboot (Alpine diskless persistence)

## Steps

### 1. WireGuard Server Health (on RPi5)

SSH to RPi5 from MGMT VLAN before running tunnel tests.

---

**1.1 — WireGuard interface is up with correct address**

What to test: `wg0` interface exists and holds the correct server IP.

```bash
ip addr show wg0
```

Expected result:
```
3: wg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 ...
    inet 10.0.0.1/24 scope global wg0
```

Failure indicates: WireGuard interface did not start. Check `rc-service wg0 status` and review `/etc/wireguard/wg0.conf`.

---

**1.2 — WireGuard shows all configured peers**

What to test: `wg show` reflects all peers with correct public keys and allowed IPs.

```bash
wg show wg0
```

Expected result:
```
interface: wg0
  public key: <SERVER_PUBKEY>
  private key: (hidden)
  listening port: 51820

peer: <LAPTOP_PUBKEY>
  preshared key: (hidden)
  allowed ips: 10.0.0.2/32

peer: <PHONE_PUBKEY>
  preshared key: (hidden)
  allowed ips: 10.0.0.3/32
```

Every configured peer must appear. Each peer must have a preshared key shown as `(hidden)`. Absence of the `preshared key` line means no PSK is set — this violates the security model.

Failure indicates: Missing peer means the peer block is absent or malformed in `/etc/wireguard/wg0.conf`. Missing preshared key means PSK was not configured — see doc 09.

---

**1.3 — WireGuard is listening on the correct port**

What to test: The WireGuard daemon is bound to UDP 51820 with no IPv6 socket.

```bash
ss -ulnp | grep 51820
```

Expected result:
```
UNCONN  0  0  0.0.0.0:51820  0.0.0.0:*  users:(("wireguard",pid=<PID>,fd=<N>))
```

The socket must be bound to `0.0.0.0:51820`. It must NOT show an IPv6 address (`::`).

Failure indicates: WireGuard is not running, or bound to a wrong port. Check `ListenPort` in `/etc/wireguard/wg0.conf` and restart with `rc-service wg0 restart`.

---

**1.4 — IPv6 is disabled on RPi5**

What to test: No IPv6 addresses exist on active interfaces and sysctl disables IPv6 system-wide.

```bash
ip -6 addr show
sysctl net.ipv6.conf.all.disable_ipv6
sysctl net.ipv6.conf.default.disable_ipv6
```

Expected result:

`ip -6 addr show` returns only loopback:
```
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 state UNKNOWN
    inet6 ::1/128 scope host
```

No `eth0` or `wg0` line should appear.

Both sysctl values:
```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
```

Failure indicates: IPv6 hardening in `/etc/sysctl.d/99-hardening.conf` was not applied or `sysctl -p` was not run. See doc 12.

---

### 2. pfSense WAN Firewall

Run these tests from a machine on the **external/WAN side** — either via a mobile hotspot or from a remote host. Do not run from LAN; the result will be wrong due to NAT.

---

**2.1 — Only UDP 51820 is open on WAN**

What to test: All TCP ports and all UDP ports except 51820 are blocked at the WAN interface.

```bash
# TCP scan — all ports should be filtered
nmap -sS -p 1-65535 --open <WAN_STATIC_IP>

# UDP scan on common ports
nmap -sU -p 53,500,1194,4500,51820 <WAN_STATIC_IP>
```

Expected result:

TCP scan: zero open ports. All ports show `filtered`.

UDP scan: port 51820 shows `open|filtered` (normal for UDP with no response). All other ports show `filtered` or `closed`.

Failure indicates:
- Open TCP ports: A pfSense service is exposed to WAN. Audit WAN rules and disable WAN management access under **System → Advanced → Admin Access**.
- UDP ports other than 51820 visible: Unintended WAN rules exist. Review **Firewall → Rules → WAN**.

---

**2.2 — pfSense firewall logs show blocked WAN traffic**

What to test: pfSense is actively logging traffic that does not match the UDP 51820 rule.

From external host, attempt a blocked connection:
```bash
nc -zv -w 3 <WAN_STATIC_IP> 22
```

Expected result: Connection times out.

On pfSense: **Status → System Logs → Firewall**, filter by interface `WAN`. A block entry for the source IP must appear.

Failure indicates: The default deny rule is not logging. In pfSense, the implicit deny does not log by default — add an explicit block rule at the bottom of the WAN ruleset with logging enabled.

---

### 3. VPN Connectivity (from Laptop Client)

Run from the laptop connected to an external network (hotspot or remote). The laptop must not be on the same LAN as the VPN server.

---

**3.1 — Tunnel establishes successfully**

What to test: `wg-quick up` completes without errors and creates the interface.

```bash
sudo wg-quick up wg0
ip addr show wg0
```

Expected result during bring-up:
```
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.0.0.2/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] ip -4 route add <LAN_SUBNET> dev wg0
[#] ip -4 route add 10.0.0.0/24 dev wg0
```

Interface shows:
```
inet 10.0.0.2/24 scope global wg0
```

Failure indicates: Key mismatch, unreachable endpoint, or wrong port. Check `wg show wg0` for a `latest handshake` timestamp — absence means no handshake occurred. Verify `<WAN_STATIC_IP>:51820` is reachable and the pfSense port forward is active.

---

**3.2 — Ping the WireGuard server IP**

What to test: ICMP to RPi5's tunnel IP succeeds.

```bash
ping -c 4 10.0.0.1
```

Expected result:
```
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=<RTT> ms
4 packets transmitted, 4 received, 0% packet loss
```

Failure indicates: Tunnel is up but RPi5 is not responding to ICMP. Check RPi5 iptables INPUT rules — ICMP must be accepted on `wg0`. Verify `wg show wg0` on RPi5 shows a recent handshake for the laptop peer.

---

**3.3 — Reach a LAN device through the VPN**

What to test: A device on `<LAN_SUBNET>` is reachable from the laptop via the tunnel.

```bash
ping -c 4 <LAN_DEVICE_IP>
traceroute <LAN_DEVICE_IP>
```

Expected result for ping:
```
64 bytes from <LAN_DEVICE_IP>: icmp_seq=1 ttl=<N> time=<RTT> ms
```

Expected result for traceroute — first hop must be the WireGuard server:
```
 1  10.0.0.1  (<RTT> ms)
 2  <LAN_DEVICE_IP>  (<RTT> ms)
```

If `10.0.0.1` is not the first hop, traffic is not traversing the tunnel correctly.

Failure indicates:
- No response: pfSense DMZ→LAN rules may not permit ICMP to this device. Check **Firewall → Rules → DMZ** on pfSense.
- First hop is not `10.0.0.1`: Client routing is wrong. Verify `AllowedIPs` in `/etc/wireguard/wg0.conf` includes `<LAN_SUBNET>`.

---

### 4. Split Tunnel Verification

Run while the VPN is connected from the external network.

---

**4.1 — Internet traffic does NOT route through VPN**

What to test: Your public IP as seen by external services is your local ISP IP, not `<WAN_STATIC_IP>`.

```bash
curl -4 https://ifconfig.me
```

Expected result: The returned IP is your current ISP's IP (hotspot or remote network), NOT `<WAN_STATIC_IP>`.

Failure indicates: `AllowedIPs` in the client config includes `0.0.0.0/0`, which tunnels all traffic. Split tunnel config must list only `<LAN_SUBNET>, 10.0.0.0/24`. See doc 10.

---

**4.2 — Routing table shows correct split tunnel entries**

What to test: Only VPN subnets route through `wg0`; the default route still points to the local gateway.

```bash
ip route show
```

Expected result — the following routes must be present:
```
10.0.0.0/24 dev wg0 scope link
<LAN_SUBNET> dev wg0 scope link
default via <LOCAL_GW_IP> dev <LOCAL_IFACE>
```

The default route must NOT point to `wg0`.

Failure indicates: If `default via` routes through `wg0`, all traffic is tunneled. Fix `AllowedIPs` in the client config. If `<LAN_SUBNET>` is absent, the client cannot reach LAN — add it to `AllowedIPs`.

---

### 5. DNS Leak Test

Run while the VPN is connected.

---

**5.1 — DNS resolves via Pi-hole, not a public resolver**

What to test: DNS queries use `<PIHOLE_IP>` and do not leak to external resolvers.

```bash
dig @<PIHOLE_IP> example.com
```

Expected result:
```
;; SERVER: <PIHOLE_IP>#53(<PIHOLE_IP>)
;; ANSWER SECTION:
example.com.  <TTL>  IN  A  93.184.216.34
```

The `SERVER` line must show `<PIHOLE_IP>`, not `8.8.8.8`, `1.1.1.1`, or any other public resolver.

Failure indicates: The client `DNS` line in `/etc/wireguard/wg0.conf` is not set to `<PIHOLE_IP>`, or `wg-quick` is not applying it. Verify with `resolvectl status` or `cat /etc/resolv.conf` while tunnel is up. See doc 10.

---

**5.2 — No DNS queries escape to the internet**

What to test: Internal hostnames resolve correctly via Pi-hole, not an external resolver.

```bash
dig @<PIHOLE_IP> <INTERNAL_HOSTNAME>.<LAN_DOMAIN>
```

Expected result: Resolves to an internal IP. No `SERVFAIL` from an external resolver.

Failure indicates: Pi-hole does not have local DNS entries configured, or the client is not using `<PIHOLE_IP>` as its resolver.

---

### 6. DMZ → LAN Firewall Enforcement

Run from RPi5 (logged in via SSH from MGMT VLAN). RPi5 is physically in the DMZ, so any connection attempt from RPi5 to LAN is subject to pfSense DMZ→LAN rules.

---

**6.1 — Access to a disallowed LAN port is blocked**

What to test: From DMZ (RPi5), a connection attempt to a LAN device on a port NOT listed in pfSense DMZ→LAN rules is blocked.

```bash
# Replace with a port not in your DMZ→LAN ruleset
nc -zv -w 3 <LAN_DEVICE_IP> <BLOCKED_PORT>
```

Expected result:
```
nc: connect to <LAN_DEVICE_IP> port <BLOCKED_PORT> (tcp) failed: Connection timed out
```

On pfSense: **Status → System Logs → Firewall**, filter by source `<DMZ_RPi5_IP>` — a block entry must appear.

Failure indicates: pfSense DMZ→LAN rules are too permissive or a catch-all allow rule exists. Review **Firewall → Rules → DMZ** and remove any `any`/`any` rules.

---

**6.2 — MGMT subnet is unreachable from DMZ**

What to test: RPi5 cannot reach the MGMT VLAN.

```bash
ping -c 3 -W 2 <MGMT_SUBNET_GATEWAY>
nc -zv -w 3 <MGMT_DEVICE_IP> 22
```

Expected result: Both commands time out with no response.

Failure indicates: pfSense has a rule permitting DMZ→MGMT traffic. This must not exist. MGMT must be reachable only from the MGMT VLAN itself.

---

**6.3 — VPN peer cannot reach LAN beyond what pfSense rules allow**

What to test: From laptop (10.0.0.2, tunnel connected), a connection to a LAN device on a blocked port is rejected.

```bash
nc -zv -w 3 <LAN_DEVICE_IP> <BLOCKED_PORT>
```

Expected result: Connection times out. pfSense firewall log shows a block entry with source `<DMZ_RPi5_IP>` (WireGuard traffic exits RPi5 via its DMZ IP before pfSense enforces rules).

Failure indicates: pfSense DMZ→LAN rules are too broad. All VPN peer traffic exits RPi5 via its DMZ interface IP, so pfSense evaluates it against DMZ rules. Ensure rules restrict by port and destination, not just source.

---

### 7. SSH Access Restriction

---

**7.1 — SSH from MGMT VLAN succeeds**

What to test: The admin workstation on MGMT VLAN can SSH to RPi5.

From admin workstation (MGMT VLAN):
```bash
ssh -i ~/.ssh/<ADMIN_KEY> <ADMIN_USER>@<DMZ_RPi5_IP>
```

Expected result: Successful authentication and shell access.

Failure indicates: RPi5 iptables INPUT rule is blocking the MGMT VLAN source range, or the SSH key is wrong. Run `iptables -L INPUT -n -v` on RPi5 to inspect the SSH allow rule source CIDR.

---

**7.2 — SSH from WireGuard tunnel IP is blocked**

What to test: A VPN-connected client (10.0.0.2) cannot SSH to RPi5.

From laptop while VPN is connected:
```bash
ssh -o ConnectTimeout=5 -i ~/.ssh/<ADMIN_KEY> <ADMIN_USER>@10.0.0.1
ssh -o ConnectTimeout=5 -i ~/.ssh/<ADMIN_KEY> <ADMIN_USER>@<DMZ_RPi5_IP>
```

Expected result: Both attempts time out or return `Connection refused`.

Failure indicates: RPi5 iptables INPUT chain permits SSH from `10.0.0.0/24` or from all sources. The SSH accept rule must restrict source to `<MGMT_SUBNET>` only. See doc 11.

---

**7.3 — SSH from LAN is blocked**

What to test: A device on LAN cannot SSH to RPi5 directly.

From a LAN device:
```bash
ssh -o ConnectTimeout=5 <ADMIN_USER>@<DMZ_RPi5_IP>
```

Expected result: Connection times out. pfSense blocks LAN→DMZ TCP 22, and RPi5 iptables rejects it even if a packet somehow arrived.

Failure indicates: pfSense has a LAN→DMZ rule permitting TCP 22, or RPi5 iptables is missing the source restriction. Both layers must block this independently.

---

### 8. IPv6 Leak Test

Run on all relevant devices.

---

**8.1 — RPi5 has no active IPv6 addresses**

Covered in step 1.4. Repeat this check after reboot (step 10) to confirm persistence.

---

**8.2 — Laptop WireGuard interface has no IPv6**

What to test: `wg0` has no IPv6 address while tunnel is up.

```bash
ip -6 addr show wg0
```

Expected result: No output.

Failure indicates: The client config has an IPv6 address in the `Address =` line. Remove it. The tunnel must use only `10.0.0.2/24` (IPv4).

---

**8.3 — IPv6 connectivity is non-functional**

What to test: No outbound IPv6 traffic is possible from the laptop.

```bash
curl -6 --max-time 5 https://ifconfig.me
```

Expected result:
```
curl: (28) Operation timed out after 5000 milliseconds
```
or
```
curl: (7) Couldn't connect to server
```

Failure indicates: IPv6 is active and routing traffic outside the tunnel. Disable IPv6 at the OS level or ensure no IPv6 default route exists on the machine.

---

### 9. WireGuard Peer Isolation

---

**9.1 — Laptop peer cannot reach phone peer**

What to test: From laptop (10.0.0.2), connections to phone (10.0.0.3) are blocked. Peers must not communicate laterally through the server.

From laptop with VPN connected:
```bash
ping -c 3 -W 2 10.0.0.3
```

Expected result:
```
PING 10.0.0.3 (10.0.0.3) 56(84) bytes of data.
--- 10.0.0.3 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss
```

On RPi5, verify no broad FORWARD rule exists:
```bash
iptables -L FORWARD -n -v
```

The FORWARD chain should only permit forwarding from `wg0` destined for `<LAN_SUBNET>` via the DMZ interface. There must be no rule forwarding between peer IPs within `10.0.0.0/24`.

Failure indicates: RPi5 iptables FORWARD chain has an overly broad rule allowing all `wg0` forwarding. Restrict FORWARD rules to `wg0 → eth0` with destination `<LAN_SUBNET>` only. See doc 08.

---

### 10. Diskless Persistence Test (Alpine RPi5)

---

**10.1 — Reboot RPi5 and verify automatic recovery**

What to test: After a cold reboot, all services and security settings come back without manual intervention.

From MGMT VLAN SSH session:
```bash
sudo reboot
```

Wait 60–90 seconds, then reconnect:
```bash
ssh -i ~/.ssh/<ADMIN_KEY> <ADMIN_USER>@<DMZ_RPi5_IP>
```

Once logged in, run the following checks sequentially.

**WireGuard is running:**
```bash
wg show wg0
rc-service wg0 status
```
Expected: Interface up, all peers listed, status shows `started`.

**sysctl hardening is applied:**
```bash
sysctl net.ipv6.conf.all.disable_ipv6
sysctl net.ipv4.conf.all.forwarding
sysctl net.ipv4.conf.all.rp_filter
```
Expected values: `disable_ipv6 = 1`, `forwarding = 1`, `rp_filter = 1`.

**iptables rules are loaded:**
```bash
iptables -L INPUT -n --line-numbers
iptables -L FORWARD -n --line-numbers
iptables -L OUTPUT -n --line-numbers
iptables -L -n | grep "policy DROP"
```
Expected: All rules from doc 08 are present. Default policy is `DROP` for INPUT and FORWARD.

**SSH daemon is running:**
```bash
rc-service sshd status
```
Expected: `sshd | status: started`.

Failure indicates:
- WireGuard not started: The `/etc/local.d/wireguard.start` script or `rc-update add` registration is missing. See doc 08.
- sysctl not applied: `/etc/sysctl.d/99-hardening.conf` exists but is not applied at boot. Ensure the sysctl init script is enabled: `rc-update add sysctl boot`.
- iptables rules missing: The restore script in `/etc/local.d/` was not created or not marked executable (`chmod +x`). See doc 08.
- If any of these fail, `lbu commit -d` was likely not run after applying changes. Reapply, then run `lbu commit -d` and retest.

---

**10.2 — VPN reconnects after server reboot**

What to test: After RPi5 reboots, the laptop can reconnect without any manual intervention on the server side.

From laptop:
```bash
sudo wg-quick down wg0
sleep 5
sudo wg-quick up wg0
ping -c 4 10.0.0.1
ping -c 4 <LAN_DEVICE_IP>
```

Expected result: Both pings succeed within 30 seconds of bringing the tunnel up.

Failure indicates: WireGuard did not restart automatically on RPi5 after reboot. Review Alpine autostart configuration from doc 08.

---

## Verification

After completing all steps above, confirm the following checklist. Every item must pass before the deployment is considered complete.

```
[ ] 1.1  ip addr show wg0: inet 10.0.0.1/24 present
[ ] 1.2  wg show wg0: all peers listed, preshared key shown as (hidden) for each
[ ] 1.3  ss -ulnp: WireGuard listening on UDP 0.0.0.0:51820, no IPv6 socket
[ ] 1.4  ip -6 addr: no global IPv6 on RPi5; sysctl disable_ipv6 = 1
[ ] 2.1  nmap from WAN: zero open TCP ports; UDP 51820 open|filtered only
[ ] 2.2  pfSense logs: blocked entry appears for test connection to port 22
[ ] 3.1  wg-quick up: completes without errors, wg0 created
[ ] 3.2  ping 10.0.0.1 from laptop: 0% packet loss
[ ] 3.3  ping <LAN_DEVICE_IP>: succeeds; traceroute first hop is 10.0.0.1
[ ] 4.1  curl ifconfig.me over VPN: returns ISP IP, not <WAN_STATIC_IP>
[ ] 4.2  ip route: <LAN_SUBNET> and 10.0.0.0/24 via wg0; default via local ISP
[ ] 5.1  dig @<PIHOLE_IP>: SERVER line shows <PIHOLE_IP>, not public resolver
[ ] 5.2  internal hostname resolves to internal IP via Pi-hole
[ ] 6.1  nc to blocked LAN port from RPi5: times out; pfSense logs block entry
[ ] 6.2  MGMT subnet unreachable from RPi5 (ping and nc both time out)
[ ] 6.3  nc to blocked port from VPN-connected laptop: times out
[ ] 7.1  SSH from MGMT VLAN admin workstation to RPi5: succeeds
[ ] 7.2  SSH from 10.0.0.2 (laptop VPN IP) to RPi5: blocked
[ ] 7.3  SSH from LAN device to RPi5: blocked
[ ] 8.2  ip -6 addr show wg0: no output
[ ] 8.3  curl -6 ifconfig.me: times out or connection refused
[ ] 9.1  ping 10.0.0.3 from 10.0.0.2: 100% packet loss
[ ] 10.1 Post-reboot: wg0 up, iptables DROP policy loaded, sysctl applied
[ ] 10.2 Post-reboot: VPN reconnects from laptop without manual server intervention
```

## Troubleshooting

**Tunnel establishes but no traffic flows to LAN**

Run on RPi5:
```bash
wg show wg0              # Verify latest handshake is recent
iptables -L FORWARD -n -v  # Check packet counters on FORWARD rules
tcpdump -i wg0 icmp      # Confirm packets enter the tunnel interface
tcpdump -i eth0 icmp     # Confirm packets exit toward LAN
```

If packets arrive on `wg0` but do not exit `eth0`, the FORWARD chain is dropping them. Check that `net.ipv4.ip_forward = 1` and that the FORWARD rule permits `wg0 → eth0` for `<LAN_SUBNET>`.

---

**pfSense blocks legitimate VPN traffic**

Navigate to **Status → System Logs → Firewall** and filter by source `<DMZ_RPi5_IP>`. Each block entry shows which rule matched (by rule ID). Cross-reference with **Firewall → Rules → DMZ** using that rule ID to identify and correct the mismatch.

---

**Configuration changes lost after reboot on Alpine**

```bash
lbu status    # Show uncommitted changes
lbu diff      # Show exactly what differs from the committed state
lbu commit -d # Write committed state to SD card
```

After any configuration change on Alpine diskless, `lbu commit -d` is mandatory. Changes not committed are silently discarded on next boot.

---

**SSH blocked from MGMT VLAN unexpectedly**

If console access is available on RPi5:
```bash
iptables -L INPUT -n -v --line-numbers
```

Locate the SSH accept rule. Verify the source CIDR matches `<MGMT_SUBNET>`. If the rule is absent or has the wrong source, re-apply the iptables rules from doc 08, then run `lbu commit -d`.

---

**WireGuard handshake never completes**

Diagnose in order:

1. pfSense NAT port forward: **Firewall → NAT → Port Forward** — UDP 51820 must forward to `<DMZ_RPi5_IP>:51820`.
2. pfSense WAN rule: **Firewall → Rules → WAN** — UDP 51820 must have a pass rule linked to the NAT entry.
3. RPi5 iptables INPUT: UDP 51820 must be accepted on the DMZ-facing interface.
4. Client endpoint: `/etc/wireguard/wg0.conf` `Endpoint` must be `<WAN_STATIC_IP>:51820`.
5. Key mismatch: each peer's `PublicKey` on the server must match `wg pubkey < <peer_private.key>` on the client. If in doubt, regenerate and redistribute keys (doc 09).

---

**`curl ifconfig.me` returns `<WAN_STATIC_IP>` (full tunnel active)**

The client is routing all traffic through the VPN. Open `/etc/wireguard/wg0.conf` on the laptop and verify:

```ini
[Peer]
AllowedIPs = 10.0.0.0/24, <LAN_SUBNET>
```

`0.0.0.0/0` must not appear in `AllowedIPs`. Bring the tunnel down, correct the config, and bring it up again.

---

## Security Notes

- **Never run the nmap WAN scan from inside LAN or DMZ.** NAT will make all ports appear filtered regardless of actual firewall rules. Use a mobile hotspot or a remote host outside your network.

- **Do not relax iptables or pfSense rules to fix a failing test.** Diagnose the root cause. Every rule relaxation is a permanent reduction in the security posture.

- **The split tunnel test (`curl ifconfig.me`) only validates the default route.** It does not prove all internet traffic is protected. Always verify `ip route show` directly.

- **Peer isolation (step 9) depends entirely on RPi5 FORWARD rules.** If new peers are added later, verify that no FORWARD rule inadvertently permits peer-to-peer traffic within `10.0.0.0/24`.

- **After any key rotation**, re-run steps 1.2 (wg show), 3.1–3.3 (connectivity), and 10.2 (post-reboot reconnect) at minimum to confirm the new keys are deployed correctly on all devices.

- **`lbu commit -d` is the final step of every maintenance operation on Alpine diskless.** Any change not committed is silently lost on reboot. Treat it as equivalent to saving a file.

- **Do not expose the pfSense management UI to WAN.** Verify this is disabled under **System → Advanced → Admin Access**. The anti-lockout rule applies only to LAN; it does not protect WAN exposure.

- **All 22 checklist items in the Verification section must pass.** Partial completion is not acceptable for a security-focused deployment. Document any item that cannot pass, the reason, and the accepted risk before considering the deployment live.

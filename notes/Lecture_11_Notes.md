# Lecture 11: NAT, DHCP and Local Network Communication
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Explain ARP and trace a full ARP resolution
- Describe NAT/PAT with a state table walkthrough
- Trace the DHCP DORA process
- Understand what a home router actually does internally
- Trace a complete local-to-internet packet journey

---

## 1. ARP — Address Resolution Protocol

### The Problem

IP routes to IP addresses. Ethernet requires MAC addresses. **ARP** bridges this gap — it maps an IP address to a MAC address on the local network.

### ARP Resolution Process

```
Device wants to send to IP 192.168.1.1 (router):

Step 1: Check ARP cache
  "Do I already know the MAC for 192.168.1.1?"
  YES → use cached MAC → skip to step 5
  NO  → proceed

Step 2: ARP Request (Broadcast)
  Ethernet frame:
    Dst MAC: FF:FF:FF:FF:FF:FF  ← Broadcast to all
    Src MAC: AA:BB:CC:DD:EE:FF  ← Sender's MAC

  ARP payload:
    "WHO HAS 192.168.1.1? TELL 192.168.1.5"

  Every device on LAN receives this!

Step 3: ARP Reply (Unicast)
  Only 192.168.1.1 responds:
  "192.168.1.1 IS AT 11:22:33:44:55:66"
  Sent directly back to AA:BB:CC:DD:EE:FF

Step 4: Update ARP Cache
  Store: 192.168.1.1 → 11:22:33:44:55:66

Step 5: Build Ethernet Frame
  Use 11:22:33:44:55:66 as destination MAC
```

### ARP Commands

```bash
# View ARP cache
arp -a
ip neigh show      # Linux preferred

# Example output:
# 192.168.1.1 dev eth0 lladdr 11:22:33:44:55:66 REACHABLE
# 192.168.1.10 dev eth0 lladdr aa:bb:cc:dd:ee:ff STALE

# Flush ARP cache (Linux, root required)
ip neigh flush all
```

### ARP Cache States

| State | Meaning |
|-------|---------|
| REACHABLE | Entry confirmed recently (~30s default) |
| STALE | Not used recently; will re-verify on next use |
| DELAY | Just turned stale; probe pending |
| PROBE | Sending unicast probe to re-verify |
| FAILED | No response to probes; entry deleted |

### Special ARP Types

**Gratuitous ARP:**
A device ARPs for its own IP address.
- Purpose 1: Detect IP conflicts (if anyone replies, conflict exists)
- Purpose 2: Force other devices to update their ARP caches (used in failover)

**Proxy ARP:**
A router responds to ARP requests on behalf of another device. Allows devices on different subnets to communicate without knowing a router exists (mostly legacy).

---

## 2. NAT — Network Address Translation

### Purpose

NAT allows multiple devices with private IPs to share a single public IP address.

### PAT (Port Address Translation) — State Table

**Setup:**
- LAN: `192.168.1.0/24`
- Public IP: `203.0.113.5` (router's WAN interface)

**NAT State Table Example:**

| Private IP:Port | Public IP:Port | Destination | Protocol |
|----------------|----------------|-------------|---------|
| 192.168.1.5:52341 | 203.0.113.5:10001 | 142.250.195.46:80 | TCP |
| 192.168.1.8:34100 | 203.0.113.5:10002 | 142.250.195.46:80 | TCP |
| 192.168.1.12:45200 | 203.0.113.5:10003 | 54.230.45.100:443 | TCP |

### Outbound (SNAT) Flow

```
Laptop sends:  SrcIP:192.168.1.5:52341 → Dst:142.250.195.46:80
Router:        SrcIP:203.0.113.5:10001 → Dst:142.250.195.46:80
               Creates NAT table entry
```

### Inbound (Reply) Flow

```
Google replies:  Src:142.250.195.46:80 → Dst:203.0.113.5:10001
Router:          Look up port 10001 in NAT table → 192.168.1.5:52341
                 Rewrite Dst: 192.168.1.5:52341
                 Forward to laptop on LAN
```

### Port Forwarding (DNAT)

Port forwarding allows inbound connections to reach a device on the private network:

```
Router rule: TCP port 8080 → 192.168.1.10:8080

External client connects to: 203.0.113.5:8080
Router translates:           → 192.168.1.10:8080 (DNAT)
```

**Common uses:** Game server, web server, camera stream, SSH access to home machine.

### NAT Security Behavior

By default, NAT **blocks all unsolicited inbound traffic** — no entry in the state table = packet dropped. This makes NAT an accidental firewall.

### NAT Limitations

| Problem | Cause | Solution |
|---------|-------|---------|
| P2P broken | Peers can't initiate inbound | STUN/TURN/ICE |
| VoIP issues | SIP protocol embeds IPs in payload | Application Layer Gateway |
| FTP active mode | FTP embeds IP in payload | Passive FTP mode / ALG |
| Gaming | Direct player-to-player connections needed | STUN, relay servers |

### NAT Traversal

| Technique | What it does |
|-----------|-------------|
| **STUN** | Public server tells you your public IP:port; peers attempt direct connection |
| **TURN** | Relay server proxies all traffic (fallback when STUN fails) |
| **ICE** | Tries STUN first, then TURN; used in WebRTC (video calls) |

---

## 3. DHCP — Dynamic Host Configuration Protocol

### What DHCP Provides

| Configuration | Value |
|-------------|-------|
| IP Address | e.g., 192.168.1.5 |
| Subnet Mask | e.g., 255.255.255.0 |
| Default Gateway | e.g., 192.168.1.1 |
| DNS Server(s) | e.g., 8.8.8.8, 8.8.4.4 |
| Lease Duration | e.g., 86400 seconds (24 hours) |

### DHCP DORA Process

```
CLIENT (no IP)              DHCP SERVER (usually router, port 67)

    │──── DISCOVER ─────────────────────────────────▶│
    │  Src: 0.0.0.0:68                               │
    │  Dst: 255.255.255.255:67 (broadcast)           │
    │  "I need an IP! (My MAC is AA:BB:CC:DD:EE:FF)" │

    │◀─── OFFER ─────────────────────────────────────│
    │  "Here's an offer:                             │
    │   IP: 192.168.1.5, Mask: /24                  │
    │   GW: 192.168.1.1, DNS: 8.8.8.8              │
    │   Lease: 86400s"                              │

    │──── REQUEST ──────────────────────────────────▶│
    │  (still broadcast — tells other DHCP servers   │
    │   that their offers are declined)              │
    │  "I'd like 192.168.1.5 please"                │

    │◀─── ACKNOWLEDGE ───────────────────────────────│
    │  "Confirmed: 192.168.1.5 is yours for 86400s" │
```

**Why UDP?** Client has no IP yet → can't use TCP. Also uses broadcasts → TCP is unicast only.

**Ports:** DHCP server listens on UDP **67**, clients on UDP **68**.

### DHCP Lease Renewal

| Time | Event |
|------|-------|
| T1 = 50% of lease | Client sends unicast REQUEST to original server → try to renew |
| T2 = 87.5% of lease | If T1 failed, client broadcasts REQUEST → any server can renew |
| T3 = 100% (expiry) | IP is lost; device must start DORA again |

**Example with 24h lease:**
- T1 = 12h → try to renew with original server
- T2 = 21h → broadcast for any DHCP server
- T3 = 24h → IP expires, device loses connectivity

### DHCP Relay Agents

On large networks with multiple subnets, each subnet can't have its own DHCP server. A **DHCP relay agent** on the router forwards broadcast DHCP messages to a central DHCP server (unicast), which responds with configuration for that subnet.

---

## 4. What a Home Router Actually Does

A home "router" is actually 4+ devices in one:

| Function | What it does |
|---------|-------------|
| **Router (L3)** | Routes packets between LAN and WAN; NAT |
| **DHCP Server** | Assigns private IPs to devices on LAN |
| **DNS Forwarder** | Relays DNS queries to ISP's or Google's DNS |
| **L2 Switch** | Connects wired devices on the LAN |
| **Wi-Fi Access Point** | Wireless connectivity (802.11) |
| **Firewall** | Blocks unsolicited inbound traffic (partly via NAT) |

### Router Boot Sequence

1. WAN port sends DHCP DISCOVER to ISP
2. ISP's DHCP server assigns router a public IP, mask, gateway, DNS
3. Router's own DHCP server starts — allocates private IP pool
4. Router is ready to serve LAN clients

### Device Join Sequence (New Laptop Connects to Wi-Fi)

```
1. 802.11 Association: laptop negotiates with AP (Wi-Fi setup, not IP layer)
2. DHCP DORA: laptop gets 192.168.1.5, mask /24, GW 192.168.1.1, DNS 192.168.1.1
3. Gratuitous ARP: laptop ARPs for 192.168.1.5 (conflict check)
4. ARP for gateway: laptop ARPs for 192.168.1.1 to cache router's MAC
5. Ready: laptop can now communicate on LAN and internet
```

---

## 5. Complete Local-to-Internet Journey

**Laptop (192.168.1.5) requests 8.8.8.8:**

```
Step 1: Routing decision
  "Is 8.8.8.8 in 192.168.1.0/24?" → No
  "Send to default gateway: 192.168.1.1"

Step 2: ARP for gateway's MAC
  (if not cached) ARP broadcast → gets router's MAC 11:22:33:44:55:66

Step 3: Frame sent to router
  Ethernet: Dst=11:22:33:44:55:66 (router), Src=AA:BB:CC:DD:EE:FF (laptop)
  IP:       Src=192.168.1.5, Dst=8.8.8.8

Step 4: Router receives on LAN interface
  Strips Ethernet frame
  IP: Src=192.168.1.5, Dst=8.8.8.8
  → LPM lookup → default route → WAN interface

Step 5: NAT (SNAT)
  Src: 192.168.1.5:52341 → 203.0.113.5:10001
  Creates NAT table entry

Step 6: Router ARPs for ISP router's MAC on WAN link (if needed)
  New Ethernet frame: Dst=ISP router MAC, Src=Router WAN MAC
  IP: Src=203.0.113.5, Dst=8.8.8.8

Step 7: Multiple routers across internet forward toward 8.8.8.8

Step 8: 8.8.8.8 receives, replies: Src=8.8.8.8, Dst=203.0.113.5:10001

Step 9: Router receives reply on WAN
  NAT lookup: port 10001 → 192.168.1.5:52341
  Rewrites Dst to 192.168.1.5:52341
  ARP for laptop's MAC → forwards on LAN

Step 10: Laptop receives reply
  OS demux: port 52341 → browser connection
  Data delivered to application
```

---

## 📌 Key Takeaways

1. **ARP**: broadcast "who has IP X?" → unicast reply; cache the result (~30-60s TTL)
2. **Gratuitous ARP**: ARP for own IP → conflict detection and cache update
3. **NAT/PAT**: many private IPs share one public IP via port mapping table
4. **Port forwarding (DNAT)**: specific public port maps to specific private host:port
5. **DHCP DORA**: Discover → Offer → Request → Acknowledge; uses UDP 67/68
6. **Lease renewal**: at T1 (50%), T2 (87.5%) of lease duration
7. **Home router** = Router + DHCP Server + DNS forwarder + L2 Switch + Wi-Fi AP

---

## 🧠 Quick Self-Check Questions

1. Why does ARP use a broadcast for the request but a unicast for the reply?
2. What happens if two devices on the same LAN have the same IP address?
3. In the NAT state table, what uniquely identifies each connection?
4. Why can't an external device initiate a connection to 192.168.1.5 without port forwarding?
5. In DHCP DORA, why is REQUEST still sent as a broadcast even though the client received an offer?
6. Your DHCP lease is 12 hours. At what time does T1 occur? What happens if the server doesn't respond?
7. Trace exactly what happens when you first connect your phone to a new Wi-Fi network (from Wi-Fi association to browser working).
8. Why does DHCP use UDP and not TCP?

---

*Lecture 11 of 13 — Computer Networks, Term 5, SST*

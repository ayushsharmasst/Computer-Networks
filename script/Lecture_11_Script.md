# Lecture 11: NAT, DHCP and Local Network Communication
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Explain ARP and trace an ARP resolution
- Describe NAT/PAT in detail with state table
- Explain the DHCP lease process
- Understand how a home router orchestrates all of this
- Trace a complete local-to-internet communication

**Duration:** ~90 minutes

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Ask the class]**

> "You just bought a new laptop. You take it home, open it for the first time, and connect to your Wi-Fi. Within 10 seconds you're browsing the internet. Nothing was pre-configured on this laptop. How did it get an IP address? How did it know your router's address? How did it start routing traffic?"

> "In the next 90 minutes, you'll understand every step of what just happened — from that first moment your laptop's Wi-Fi card joined the network to successfully accessing a web server on the internet."

---

## SECTION 1: ARP — Address Resolution Protocol (20 minutes)

### The Problem

> "IP packets need to get from your laptop to your router. Your laptop knows the router's IP: 192.168.1.1. But Ethernet needs MAC addresses. How does your laptop find the router's MAC address?"

**Answer: ARP**

### How ARP Works

**[INSTRUCTOR: Draw LAN with 3 devices + router]**

**Scenario:** Your laptop (192.168.1.5) wants to send a packet to 192.168.1.1 (router).

```
Step 1 — Check ARP cache:
  "Do I already know the MAC for 192.168.1.1?"
  → Not in cache → proceed

Step 2 — ARP Request (Broadcast):
  Source: MAC=AA:BB:CC:DD:EE:FF, IP=192.168.1.5
  Destination: MAC=FF:FF:FF:FF:FF:FF (BROADCAST), IP=?
  
  Message: "WHO HAS 192.168.1.1? TELL 192.168.1.5"
  
  Every device on LAN receives this!
  → Only 192.168.1.1 responds

Step 3 — ARP Reply (Unicast):
  Router at 192.168.1.1 replies directly to AA:BB:CC:DD:EE:FF
  Message: "192.168.1.1 IS AT 11:22:33:44:55:66"

Step 4 — Cache:
  Laptop stores: 192.168.1.1 → 11:22:33:44:55:66 in ARP table
  Now builds Ethernet frame with dst MAC=11:22:33:44:55:66
```

### ARP Commands

```bash
# View ARP cache (Linux/Mac)
arp -a
# or
ip neigh show

# Windows
arp -a

# Flush ARP cache (Linux, requires root)
ip neigh flush all

# Sample output:
# 192.168.1.1 dev eth0 lladdr 11:22:33:44:55:66 REACHABLE
# 192.168.1.10 dev eth0 lladdr aa:bb:cc:dd:ee:ff STALE
```

**[INSTRUCTOR: Run this live if possible. Show REACHABLE and STALE entries.]**

### ARP Caching and Expiry

> "ARP entries don't last forever. The default ARP cache timeout on Linux is ~30-60 seconds for REACHABLE entries. After timeout, the entry becomes STALE and is revalidated on next use. This prevents stale entries from causing issues when a device changes its IP."

### Proxy ARP

> "In some scenarios, a router answers ARP requests on behalf of another device. For example, if hosts on different subnets need to communicate without knowing about the router, the router can respond to ARPs with its own MAC. This is called **Proxy ARP** — mostly legacy, but worth knowing."

### Gratuitous ARP

> "A **Gratuitous ARP** is when a device ARPs for its OWN IP address. This:
> 1. Checks for IP conflicts (if anyone replies, there's a conflict)
> 2. Updates ARP caches of other devices (e.g., after a failover, a new NIC takes over an IP)"

---

## SECTION 2: NAT and PAT in Detail (20 minutes)

### Quick Recap from Lecture 4

> "NAT translates private IPs to public IPs. PAT (Port Address Translation) allows MANY private IPs to share ONE public IP using port numbers to distinguish sessions."

### NAT State Table — Deep Dive

**Setup:**
```
LAN: 192.168.1.0/24
Public IP: 203.0.113.5 (router's WAN interface)
```

**Multiple devices browsing simultaneously:**

| Device | Private IP:Port | ↔ | Public IP:Port | Destination |
|--------|----------------|---|----------------|-------------|
| Laptop | 192.168.1.5:52341 | ↔ | 203.0.113.5:10001 | 142.250.195.46:80 |
| Phone | 192.168.1.8:34100 | ↔ | 203.0.113.5:10002 | 142.250.195.46:80 |
| TV | 192.168.1.12:45200 | ↔ | 203.0.113.5:10003 | 54.230.45.100:443 |

**[INSTRUCTOR: Draw the table on board. Show that all three devices communicate simultaneously using the SAME public IP but DIFFERENT translated ports.]**

### NAT State Table Lifecycle

**When a connection is established:**
```
Laptop SYN → router
Router: allocates port 10001, creates entry:
{proto:TCP, private:192.168.1.5:52341, public:203.0.113.5:10001, dst:142.250.195.46:80, state:SYN_SENT}
```

**When connection closes:**
```
TCP FIN received → entry marked for deletion
Timeout: 30-120 seconds → entry removed from table
```

**For UDP:**
```
No connection tracking possible → entry times out after 30-60 seconds of inactivity
```

### NAT and Port Forwarding

> "NAT blocks all inbound connections by default — the translation table has no entry for unsolicited inbound traffic. This is actually a security benefit: your home network is accidentally firewalled."

**Enabling inbound connections (port forwarding):**
```
Router rule: tcp:8080 → 192.168.1.10:8080

External: 203.0.113.5:8080 → router
Router: lookup rule → DNAT to 192.168.1.10:8080
192.168.1.10 receives the connection
```

**[INSTRUCTOR: Ask — "What are some cases where NAT breaks things?"  
Expected answers: VoIP (SIP has IP in payload), FTP (active mode embeds IP), peer-to-peer, gaming (direct connections between players)]**

### NAT Traversal Techniques

> "Because NAT blocks unsolicited inbound connections, applications that need peer-to-peer connectivity (VoIP, WebRTC, gaming) must work around NAT."

**STUN (Session Traversal Utilities for NAT):**
> "A public STUN server tells each peer its public IP:port as seen from outside. Peers then try to connect to each other's public endpoints."

**TURN (Traversal Using Relays around NAT):**
> "When STUN fails (symmetric NAT), a TURN relay server is used — both peers connect to a common server that relays traffic."

**ICE (Interactive Connectivity Establishment):**
> "WebRTC uses ICE which tries STUN first, falls back to TURN. This is what makes video calls work even through NAT."

---

## SECTION 3: DHCP — Dynamic Host Configuration Protocol (15 minutes)

### What DHCP Does

> "DHCP automatically assigns IP addresses (and other network config) to devices joining a network. Instead of manually configuring IP, mask, gateway, and DNS on every device, DHCP automates it."

**DHCP provides:**
- IP address
- Subnet mask
- Default gateway
- DNS server(s)
- Lease duration

### The DHCP DORA Process

> "DORA = Discover, Offer, Request, Acknowledge — the four-message exchange."

**[INSTRUCTOR: Draw timeline with client and DHCP server (usually the router)]**

```
NEW DEVICE (no IP yet)                    DHCP SERVER (192.168.1.1)
       │                                         │
       │────DISCOVER──────────────────────────▶ │
       │  Src: 0.0.0.0:68                        │
       │  Dst: 255.255.255.255:67 (BROADCAST)    │
       │  "I need an IP! Anyone out there?"      │
       │  (Client MAC: AA:BB:CC:DD:EE:FF)        │
       │                                         │
       │◀───OFFER────────────────────────────── │
       │  Src: 192.168.1.1:67                   │
       │  Dst: 255.255.255.255:68               │
       │  "I offer: IP=192.168.1.5              │
       │   Mask: 255.255.255.0                  │
       │   GW: 192.168.1.1                      │
       │   DNS: 8.8.8.8, 8.8.4.4               │
       │   Lease: 86400 seconds"               │
       │                                         │
       │────REQUEST──────────────────────────▶ │
       │  "I'd like 192.168.1.5 please"         │
       │  (still broadcast — other servers may  │
       │   have made offers, tells them to stop) │
       │                                         │
       │◀───ACKNOWLEDGE──────────────────────── │
       │  "Confirmed: 192.168.1.5 is yours      │
       │   for 86400 seconds"                   │
```

**[INSTRUCTOR: Ask — "Why is DISCOVER a broadcast? The client has no IP yet, so it can't unicast."  
Follow up: "Why is OFFER also a broadcast in some implementations? The client doesn't have an IP to unicast TO yet."  
Answer: It sends to 255.255.255.255 but with the client's MAC as destination in the Ethernet frame, or uses the broadcast MAC.]**

### Why UDP?

> "DHCP uses UDP (port 67 server, 68 client). TCP can't be used because:
> 1. The client has no IP address yet — TCP needs an IP to establish a connection
> 2. DHCP uses broadcasts — TCP is unicast-only"

### DHCP Lease Renewal

> "Leases have a duration. The client tries to renew at 50% of the lease time:"

```
Lease: 86400 seconds (24 hours)
T1 = 43200s (12h): Client sends unicast REQUEST to same server → Renew
T2 = 75600s (21h): If T1 renewal failed, client broadcasts REQUEST → Any server
T3 = 86400s (24h): Lease expires → Client must get new IP or lose connectivity
```

### DHCP Options

> "DHCP can provide more than just IP/mask/gateway. Through 'options,' it can provide:"

| Option Code | Data |
|------------|------|
| 1 | Subnet Mask |
| 3 | Default Gateway |
| 6 | DNS Servers |
| 12 | Hostname |
| 15 | Domain Name |
| 28 | Broadcast Address |
| 42 | NTP Servers |
| 51 | Lease Time |

---

## SECTION 4: Your Home Router — What It Actually Does (12 minutes)

> "A home router is NOT just a router. It's 4+ devices in one box:"

**[INSTRUCTOR: Draw the home router internals]**

```
                        ┌─────────────────────────────────────────────┐
INTERNET ───────────────│ WAN Interface (Public IP from ISP via DHCP)  │
                        │                                             │
                        │  ┌─────────────┐  ┌──────────────────────┐ │
                        │  │   ROUTER    │  │    DHCP SERVER       │ │
                        │  │  (L3 NAT)   │  │  (assigns 192.168.x) │ │
                        │  └─────────────┘  └──────────────────────┘ │
                        │  ┌─────────────┐  ┌──────────────────────┐ │
                        │  │  SWITCH (L2)│  │   DNS RELAY/FORWARDER│ │
                        │  │ (4-8 ports) │  │  (forwards to 8.8.8.8│ │
                        │  └─────────────┘  └──────────────────────┘ │
                        │  ┌─────────────┐                            │
                        │  │  Wi-Fi AP   │                            │
                        │  │  (802.11)   │                            │
                        │  └─────────────┘                            │
                        │ LAN Interface (192.168.1.1)                 │
                        └─────────────────────────────────────────────┘
                               │
                     ┌─────────┼──────────┐
                 Laptop     Phone      Smart TV
              192.168.1.5  192.168.1.8  192.168.1.12
```

### Router Boot + Client Join — Full Sequence

**When router starts:**
1. WAN port sends DHCP DISCOVER to ISP
2. ISP's DHCP server gives router a public IP, mask, gateway, DNS
3. Router's DHCP server starts, ready to assign private IPs

**When your laptop joins:**
1. **Wi-Fi association**: 802.11 negotiation (not IP-layer)
2. **DHCP**: Discover → Offer (192.168.1.5) → Request → ACK
3. **ARP**: Laptop ARPs for gateway (192.168.1.1) to verify it exists
4. **DNS**: Laptop's DNS is set to 192.168.1.1 (router acts as DNS forwarder)

---

## SECTION 5: Complete Local-to-Internet Trace (12 minutes)

**[INSTRUCTOR: Walk through this scenario: Your laptop (192.168.1.5) accesses 8.8.8.8]**

```
Step 1 — Laptop needs to send to 8.8.8.8
  "Is 8.8.8.8 in 192.168.1.0/24?"
  No → send to default gateway: 192.168.1.1

Step 2 — Laptop → Router (ARP if needed)
  ARP for 192.168.1.1's MAC (if not cached)
  Ethernet frame: Dst MAC=router, Src MAC=laptop
  IP packet: Src=192.168.1.5, Dst=8.8.8.8
  
Step 3 — Router receives packet on LAN interface
  Looks up routing table: 8.8.8.8 → default route → WAN interface (203.0.113.5)
  
Step 4 — NAT translation (PAT/SNAT)
  Source translated: 192.168.1.5:52341 → 203.0.113.5:10001
  NAT table entry created
  
Step 5 — Router → ISP (ARP for ISP router's MAC on WAN link)
  New Ethernet frame: Dst=ISP router MAC, Src=Router's WAN MAC
  IP packet: Src=203.0.113.5, Dst=8.8.8.8

Step 6 — ISP routes packet through internet toward 8.8.8.8
  Multiple hops, routing tables, BGP...

Step 7 — 8.8.8.8 receives packet
  Src=203.0.113.5, Dst=8.8.8.8
  Sends reply: Src=8.8.8.8, Dst=203.0.113.5

Step 8 — Router receives reply on WAN interface
  Dst=203.0.113.5:10001 → lookup NAT table → 192.168.1.5:52341
  Rewrite: Dst=192.168.1.5:52341
  ARP for 192.168.1.5's MAC (may be cached)
  Forward on LAN interface

Step 9 — Laptop receives reply
  "This is for port 52341 → my browser's connection to 8.8.8.8"
  Data delivered to application
```

---

## SUMMARY (5 minutes)

```
✅ ARP: broadcast "who has IP X?" → unicast reply with MAC
    - ARP cache stores IP→MAC mappings (timeout ~30-60s)
    - Gratuitous ARP: ARP for your own IP (conflict detection / cache update)

✅ NAT (PAT/Masquerade): many private IPs → one public IP
    - State table: (private IP:port) ↔ (public IP:port) ↔ (dst)
    - Inbound blocked by default (accidental firewall)
    - Port forwarding = DNAT: specific public port → private IP:port
    - NAT breaks P2P → solved by STUN/TURN/ICE

✅ DHCP DORA: Discover → Offer → Request → Acknowledge
    - All messages over UDP (67/68)
    - Provides: IP, mask, gateway, DNS, lease time
    - Renewal at 50% of lease time (T1), then 87.5% (T2)

✅ Home router = Router + DHCP Server + DNS forwarder + L2 Switch + Wi-Fi AP
```

---

*End of Lecture 11 Script*

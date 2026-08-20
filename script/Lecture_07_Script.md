# Lecture 7: Routing and Forwarding I
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Understand the difference between routing and forwarding
- Read and interpret a routing table
- Explain Longest Prefix Match
- Configure and understand static routes and default routes
- Trace how a packet is forwarded hop-by-hop

**Duration:** ~90 minutes

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Ask the class]**

> "You're in Delhi. You want to get to 42 Koramangala, Bangalore. You don't know Bangalore at all. What do you do?"

Expected answers: Take a flight, then a cab from airport, ask someone, use Google Maps...

> "That's exactly how packet routing works. A packet from your laptop to Google doesn't know the full path. It just knows its destination IP. At each router, it gets a new set of directions — 'go this way' — until it reaches its destination. Each router only needs to know the NEXT step, not the full path. This is called **hop-by-hop routing**."

---

## SECTION 1: Routing vs. Forwarding (8 minutes)

> "These two words are often used interchangeably but they mean different things:"

**Write on board:**

```
ROUTING:   The process of deciding WHERE to send a packet
           → Building and maintaining the routing table
           → Run by routing protocols (OSPF, BGP)
           → Happens in the control plane

FORWARDING: The act of actually sending the packet out an interface
           → Looking up the routing table and moving the packet
           → Happens for every single packet, millions per second
           → Happens in the data plane (hardware, ASICs)
```

> "Routing is the planning. Forwarding is the execution. A router does both."

**Analogy:**
> "Think of a post office. The routing manager studies all delivery routes and creates a manual — 'letters for Bangalore go via Mumbai sorting center.' That's routing. The clerk at the counter takes your letter, looks up the manual, and puts it on the right truck. That's forwarding."

---

## SECTION 2: The Router (10 minutes)

**[INSTRUCTOR: Draw a router with multiple interfaces on the board]**

```
                    ┌─────────────────────────┐
Interface eth0 ────→│  192.168.1.1/24         │
  (LAN)             │                         │
                    │       R O U T E R       │
Interface eth1 ────→│  10.0.0.1/30            │
  (WAN to ISP)      │                         │
Interface eth2 ────→│  10.0.1.1/30            │
  (WAN to datacenter│                         │
                    └─────────────────────────┘
```

> "A router is a device with multiple network interfaces, each on a different network. Its job is to receive packets on one interface and forward them out another interface, moving them closer to their destination."

**What a router contains:**
1. **Routing Table:** A database of known networks and how to reach them
2. **Routing Algorithm:** How it learns and updates routes (OSPF, BGP, etc.)
3. **Forwarding Engine:** Hardware/software that does the actual packet lookup and dispatch

**Key difference from a switch:**
> "A switch operates at Layer 2 — it forwards Ethernet frames using MAC addresses within a LAN. A router operates at Layer 3 — it forwards IP packets between different networks. A switch doesn't look at IP. A router doesn't care about MACs."

---

## SECTION 3: The Routing Table (15 minutes)

### Structure of a Routing Table

> "The routing table is the router's decision database. Each entry says: 'For packets going to THIS network, send them THIS way.'"

**Fields in a routing table entry:**

| Field | Description | Example |
|-------|-------------|---------|
| Destination Network | Network prefix (CIDR) | 192.168.1.0/24 |
| Next Hop | IP of next router OR "directly connected" | 10.0.0.2 |
| Interface | Which interface to send it out | eth1 |
| Metric | Cost of this route (lower = preferred) | 1 |
| Source | How this route was learned | OSPF, static, connected |

**Viewing a routing table:**

```bash
# Linux
ip route show

# Output example:
default via 192.168.1.1 dev eth0
10.0.0.0/24 dev eth1 proto kernel scope link src 10.0.0.1
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.5
172.16.0.0/16 via 10.0.0.2 dev eth1 metric 100
```

**[INSTRUCTOR: Parse each line for students]**

- `default via 192.168.1.1 dev eth0` → catch-all, go to 192.168.1.1 via eth0
- `10.0.0.0/24 dev eth1 ... src 10.0.0.1` → directly connected network, I AM the router for this
- `172.16.0.0/16 via 10.0.0.2` → to reach 172.16.x.x, forward to 10.0.0.2

### Building an Example Routing Table

**Scenario:** Router R1 at a company:
```
eth0: 192.168.10.1/24 (Engineering LAN)
eth1: 192.168.20.1/24 (Marketing LAN)
eth2: 10.1.1.1/30 (Link to ISP router R2)

Company was given public IP block 203.0.113.0/28 by ISP
Internet is accessible via R2 at 10.1.1.2
```

**R1's routing table:**
```
Destination       Next Hop    Interface  How Learned
192.168.10.0/24   directly    eth0       connected
192.168.20.0/24   directly    eth1       connected
10.1.1.0/30       directly    eth2       connected
0.0.0.0/0         10.1.1.2    eth2       static (default)
```

---

## SECTION 4: Longest Prefix Match (15 minutes)

### The Problem — Multiple Matching Routes

> "What if multiple entries in the routing table match the destination IP? Which one does the router use?"

**Scenario:**
```
Routing Table:
192.168.0.0/16   →  via 10.0.0.1  (interface A)
192.168.1.0/24   →  via 10.0.0.2  (interface B)
192.168.1.128/25 →  via 10.0.0.3  (interface C)
0.0.0.0/0        →  via 10.0.0.4  (interface D — default)

Packet arrives destined for: 192.168.1.200
```

**Which route matches?**
- 192.168.0.0/16 → matches (192.168.1.200 is in 192.168.0.0/16)
- 192.168.1.0/24 → matches (192.168.1.200 is in 192.168.1.0/24)
- 192.168.1.128/25 → matches! (200 = 11001000 → same top 25 bits as .128 = 10000000... wait let's check)
- 0.0.0.0/0 → always matches (default)

**Check 192.168.1.128/25:**
```
.128 = 10000000
.200 = 11001000

/25 means compare first 25 bits:
192.168.1.128 in binary: ...00000001.1_0000000
192.168.1.200 in binary: ...00000001.1_1001000
                                      ↑ 25th bit
Match on first 25 bits? 
192.168.1.200: last octet = 11001000, first bit = 1 → same!
YES, .200 is in the range 192.168.1.128 to 192.168.1.255 ✓
```

**Longest Prefix Match Rule:**
> "Among ALL matching routes, use the one with the LONGEST prefix (most specific). More bits matched = more specific route = use this one."

**Answer:** Router uses `192.168.1.128/25` (25-bit prefix — most specific match)

**[INSTRUCTOR: Walk through this slowly. It's a key concept for the exam.]**

### Why Longest Prefix Match?

> "Longest prefix match allows the routing table to be hierarchical. Your ISP might have a route for 10.0.0.0/8. Inside their network, they have more specific routes like 10.45.0.0/16 for a specific customer. Traffic to 10.45.x.x follows the /16 route (specific), while traffic to all other 10.x.x.x follows the /8 (general). Both coexist — more specific wins."

### Default Route — 0.0.0.0/0

> "The default route `0.0.0.0/0` matches EVERY destination IP (0 bits need to match). It's the fallback — 'if nothing else matches, send it here.'"

> "On your home router, the default route points to your ISP. On the ISP's backbone router, the default route might point to a Tier-1 ISP. Deep in the internet's core, Tier-1 routers have full routing tables (hundreds of thousands of routes) and no default — they know routes to every corner of the internet."

---

## SECTION 5: Static Routing (10 minutes)

### What is Static Routing?

> "A **static route** is manually configured by a network administrator. The router doesn't learn it automatically — you type it in."

**Command syntax:**

```bash
# Linux (ip route command)
ip route add 192.168.20.0/24 via 10.0.0.1
ip route add default via 10.0.0.1         # Default route

# Cisco IOS
ip route 192.168.20.0 255.255.255.0 10.0.0.1
ip route 0.0.0.0 0.0.0.0 10.0.0.1        # Default route
```

### When to Use Static Routing

**Appropriate for:**
- Small networks with simple topology
- Stub networks (one path in, one path out)
- Security-critical routes (ensure traffic never goes unexpected places)
- Default routes on edge routers

**Problems with static routing at scale:**
- Manual configuration → human error
- No automatic failover: if the next-hop router goes down, static route keeps forwarding (and packets get dropped)
- Doesn't scale: a large network would need thousands of manual entries

**[INSTRUCTOR: Demo — add a static route on Linux VM if available]**

```bash
# Add temporary static route
ip route add 10.20.0.0/16 via 192.168.1.1

# Verify
ip route show | grep 10.20

# Delete
ip route del 10.20.0.0/16 via 192.168.1.1
```

### Floating Static Routes

> "A **floating static route** has a high metric (low priority) and acts as a backup. If the primary dynamic route disappears, the floating static kicks in."

```bash
# Primary: OSPF learns route to 10.5.0.0/24 with metric 10
# Backup: manual static route with high metric
ip route add 10.5.0.0/24 via 10.0.1.1 metric 200
# This only gets used if OSPF route disappears
```

---

## SECTION 6: Hop-by-Hop Routing in Practice (12 minutes)

### Tracing a Packet's Journey

**Scenario:** Your laptop at 192.168.1.5 is accessing a server at 10.50.20.100.

**Network topology:**
```
Laptop(192.168.1.5) → R1(192.168.1.1) → R2(10.0.0.2) → R3(10.0.1.2) → Server(10.50.20.100)
                         (eth0)    (eth1)            (eth0)         (eth0)
```

**R1's Routing Table:**
```
192.168.1.0/24   directly connected (eth0)
10.0.0.0/30      directly connected (eth1)
10.50.0.0/16     via 10.0.0.2 (eth1)
0.0.0.0/0        via 10.0.0.2 (eth1)
```

**R2's Routing Table:**
```
10.0.0.0/30      directly connected (eth0)
10.0.1.0/30      directly connected (eth1)
10.50.0.0/16     via 10.0.1.2 (eth1)
```

**R3's Routing Table:**
```
10.0.1.0/30      directly connected (eth0)
10.50.20.0/24    directly connected (eth1)
```

**Hop-by-hop trace:**

```
Step 1: Laptop sends packet
  SrcIP: 192.168.1.5, DstIP: 10.50.20.100
  10.50.20.100 not in 192.168.1.0/24 → send to default gateway 192.168.1.1

Step 2: R1 receives packet on eth0
  Lookup 10.50.20.100 in routing table:
  → Matches 10.50.0.0/16 via 10.0.0.2 (eth1)
  Forward out eth1 to 10.0.0.2

Step 3: R2 receives packet on eth0
  Lookup 10.50.20.100:
  → Matches 10.50.0.0/16 via 10.0.1.2 (eth1)
  Forward out eth1 to 10.0.1.2

Step 4: R3 receives packet on eth0
  Lookup 10.50.20.100:
  → Matches 10.50.20.0/24 — directly connected (eth1)
  ARP for 10.50.20.100's MAC address
  Forward directly to server

Step 5: Server receives packet
  Sends response back: SrcIP 10.50.20.100 → DstIP 192.168.1.5
  (Return path is calculated similarly)
```

**[INSTRUCTOR: Walk through this on the board with arrows. Point out that at each step, only the NEXT hop matters. The router doesn't know the full path.]**

### What Changes at Each Hop?

```
Field           Changes at each router?
IP Source       ❌ No (stays 192.168.1.5 unless NAT)
IP Destination  ❌ No (stays 10.50.20.100)
TTL             ✅ Yes — decremented by 1 at each router
Ethernet Src MAC ✅ Yes — new router's interface MAC
Ethernet Dst MAC ✅ Yes — next router's interface MAC
```

**TTL — Time to Live:**
> "Each IP packet has a TTL field (usually starts at 64 or 128). Every router decrements it by 1. If it reaches 0, the router discards the packet and sends an ICMP 'Time Exceeded' message back. This prevents packets from looping forever."

---

## SECTION 7: Packet Forwarding — Real-World Numbers (5 minutes)

> "Modern router ASICs can forward hundreds of millions to billions of packets per second. How? Hardware forwarding with TCAM (Ternary Content-Addressable Memory) — specialized memory that can search all routing table entries simultaneously in O(1) time."

> "This is why routing tables can have 900,000+ entries (the full internet routing table) and still do Longest Prefix Match in nanoseconds."

---

## SUMMARY (5 minutes)

```
✅ Routing = deciding where packets go (control plane)
✅ Forwarding = actually sending packets (data plane)
✅ Routing table: destination network → next hop + interface
✅ Longest Prefix Match: most specific match wins
✅ Default route 0.0.0.0/0: matches everything (fallback)
✅ Static routes: manually configured, good for small/stub networks
✅ Hop-by-hop: each router only knows the next step
✅ TTL: decremented at each hop, prevents infinite loops
✅ MAC changes at each hop; IP stays the same (absent NAT)
```

---

## 🔗 Preview

> "Today we covered the mechanics of how a router makes decisions. Next lecture, we'll see how routers LEARN their routes dynamically — distance vector protocols like RIP, link-state protocols like OSPF, and the internet's inter-domain routing protocol BGP."

---

*End of Lecture 7 Script*

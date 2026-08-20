# Lecture 7: Routing and Forwarding I
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Distinguish routing from forwarding
- Read and interpret routing tables
- Apply Longest Prefix Match
- Explain static routing and default routes
- Trace hop-by-hop packet forwarding

---

## 1. Routing vs. Forwarding

| Concept | Definition | When it happens | Plane |
|---------|-----------|----------------|-------|
| **Routing** | Deciding WHERE to send packets; building the routing table | Periodically (when topology changes) | Control plane |
| **Forwarding** | Actually moving packets out the correct interface | For every single packet | Data plane |

- **Routing** is slow and intelligent (algorithms, protocols)
- **Forwarding** is fast and mechanical (table lookup, hardware ASICs)

---

## 2. What is a Router?

A **router** is a networking device with:
- **Multiple network interfaces**, each connected to a different network
- A **routing table** that maps destinations to next hops
- A **routing engine** to learn and maintain routes
- A **forwarding engine** to process packets quickly

```
               Interface eth0 — 192.168.1.1/24 (LAN)
               │
    [ROUTER]───┤
               │
               Interface eth1 — 10.0.0.1/30 (WAN to ISP)
```

**Router vs Switch:**
- Switch: Layer 2, uses MAC addresses, within a LAN
- Router: Layer 3, uses IP addresses, connects different networks

---

## 3. Routing Table

The routing table is the router's lookup database.

### Routing Table Fields

| Field | Description |
|-------|-------------|
| **Destination Network** | The IP prefix this entry covers |
| **Next Hop** | IP of the next router to send the packet to |
| **Interface** | Which interface to send it out |
| **Metric** | Cost of this route (lower = more preferred) |
| **Source** | How the route was learned (connected, static, OSPF) |

### Viewing Routing Tables

```bash
# Linux
ip route show

# Sample output:
default via 192.168.1.1 dev eth0
10.0.0.0/30 dev eth1 proto kernel scope link src 10.0.0.1
192.168.1.0/24 dev eth0 proto kernel scope link src 192.168.1.5
172.16.0.0/16 via 10.0.0.2 dev eth1 metric 100
```

**Reading the output:**
- `default via 192.168.1.1` → default route (catch-all), next hop = 192.168.1.1
- `10.0.0.0/30 dev eth1 ... src 10.0.0.1` → directly connected network
- `172.16.0.0/16 via 10.0.0.2` → reach 172.16.x.x by forwarding to 10.0.0.2

### Example Routing Table

**Router R1 with 3 interfaces:**
```
eth0: 192.168.10.1/24 (Engineering LAN)
eth1: 192.168.20.1/24 (Marketing LAN)
eth2: 10.1.1.1/30 (WAN to ISP)
```

| Destination | Next Hop | Interface | Source |
|------------|---------|-----------|--------|
| 192.168.10.0/24 | directly connected | eth0 | connected |
| 192.168.20.0/24 | directly connected | eth1 | connected |
| 10.1.1.0/30 | directly connected | eth2 | connected |
| 0.0.0.0/0 | 10.1.1.2 | eth2 | static |

---

## 4. Longest Prefix Match (LPM)

When a packet's destination IP matches **multiple routing table entries**, the router uses the entry with the **longest (most specific) prefix**.

### Rule: Most bits matched = Most specific = Wins

**Example:**
```
Routing Table:
192.168.0.0/16    →  via 10.0.0.1  (interface A)
192.168.1.0/24    →  via 10.0.0.2  (interface B)
192.168.1.128/25  →  via 10.0.0.3  (interface C)
0.0.0.0/0         →  via 10.0.0.4  (interface D — default)

Incoming packet: destination = 192.168.1.200
```

**Matching entries:**
- `/16` matches (192.168.1.200 is in 192.168.0.0/16) ← 16 bits
- `/24` matches (192.168.1.200 is in 192.168.1.0/24) ← 24 bits
- `/25` matches (192.168.1.200 is in 192.168.1.128-255) ← 25 bits ✓ WINNER
- `/0` matches (everything matches default route) ← 0 bits

**Decision:** Use `/25` route → forward to 10.0.0.3

### Why LPM Matters

LPM allows hierarchical routing:
- ISP has a route for `10.0.0.0/8`
- Specific customer has a more specific route `10.45.0.0/16`
- Traffic to `10.45.x.x` uses the `/16` (customer-specific)
- All other `10.x.x.x` uses the `/8` (general ISP route)
- Both coexist; more specific always wins

---

## 5. Default Route — 0.0.0.0/0

The **default route** matches every destination IP (0 bits need to match). It's the fallback when no other route matches.

```
Route: 0.0.0.0/0 via 10.1.1.2
Meaning: "If nothing else in the routing table matches, send to 10.1.1.2"
```

**Placement in the internet hierarchy:**
```
Home device → Home router (default → ISP)
ISP router → Tier-2 ISP (default → Tier-1 ISP)
Tier-1 core router → NO default! Has full routing table (900K+ routes)
```

---

## 6. Static Routing

A **static route** is manually configured by an administrator — not learned automatically.

### Configuration

```bash
# Linux
ip route add 192.168.20.0/24 via 10.0.0.1       # Specific route
ip route add default via 10.0.0.1                 # Default route
ip route del 192.168.20.0/24 via 10.0.0.1         # Delete route

# Verify
ip route show
```

### When to Use Static Routing

**Good for:**
- Small, simple networks
- Stub networks (single in/out path)
- Specific security-critical routes
- Default gateway configuration

**Problems at scale:**
- Manual → error-prone
- No automatic failover (if next-hop dies, static route still forwards → packets dropped)
- Doesn't scale: large networks need too many entries

### Floating Static Routes (Backup Routes)

```bash
# Primary route learned via OSPF (metric 10)
# Backup static with high metric:
ip route add 10.5.0.0/24 via 10.0.1.1 metric 200
# Only used if OSPF route disappears
```

---

## 7. Hop-by-Hop Routing

Packets travel hop-by-hop — each router only knows the **next step**, not the full path.

### Packet Forwarding Decision at Each Router

```
1. Extract destination IP from packet
2. Search routing table for best match (Longest Prefix Match)
3. If match found: forward packet to next hop via specified interface
4. If no match (and no default): drop packet, send ICMP "Destination Unreachable"
```

### Complete Hop-by-Hop Example

**Scenario:** Laptop (192.168.1.5) → Server (10.50.20.100)

**Topology:**
```
Laptop → R1 → R2 → R3 → Server
192.168.1.5  192.168.1.1/10.0.0.1  10.0.0.2/10.0.1.1  10.0.1.2/10.50.20.1  10.50.20.100
```

**At each hop:**

| Step | Location | Action |
|------|----------|--------|
| 1 | Laptop | DstIP 10.50.20.100 not in 192.168.1.0/24 → send to gateway 192.168.1.1 |
| 2 | R1 | LPM: 10.50.0.0/16 → next hop 10.0.0.2 via eth1 |
| 3 | R2 | LPM: 10.50.0.0/16 → next hop 10.0.1.2 via eth1 |
| 4 | R3 | LPM: 10.50.20.0/24 directly connected → ARP for 10.50.20.100, forward directly |
| 5 | Server | Packet received! |

---

## 8. What Changes at Each Hop?

| Header Field | Changes at each router? | Reason |
|-------------|------------------------|--------|
| IP Source Address | ❌ No (unless NAT) | End-to-end addressing |
| IP Destination Address | ❌ No (unless NAT) | End-to-end addressing |
| **TTL (Time To Live)** | ✅ Yes — decremented by 1 | Prevents infinite loops |
| **Ethernet Source MAC** | ✅ Yes — router's outgoing interface | New L2 frame each hop |
| **Ethernet Destination MAC** | ✅ Yes — next router's incoming interface | Local delivery |

### TTL (Time To Live)

- Starts at 64 (Linux/Mac) or 128 (Windows)
- Decremented by 1 at **each router**
- If TTL reaches 0: packet is dropped, router sends **ICMP Time Exceeded** to source
- **Purpose:** Prevents packets from looping forever in routing loops

```
traceroute uses TTL:
- Sends packet with TTL=1 → first router drops it, replies with ICMP Time Exceeded
- TTL=2 → second router replies
- TTL=3 → third router replies
- ... until destination is reached
```

---

## 📌 Key Takeaways

1. **Routing** = deciding routes (control plane); **Forwarding** = moving packets (data plane)
2. **Routing table** entries: destination network → next hop + interface
3. **Longest Prefix Match**: most specific route wins when multiple entries match
4. **Default route** `0.0.0.0/0`: matches everything; gateway of last resort
5. **Static routes**: manually configured; simple but no failover
6. **Hop-by-hop**: each router knows only the next step; IP addresses stay the same
7. **TTL**: decremented at each hop; prevents infinite loops

---

## 🧠 Quick Self-Check Questions

1. Given routing table entries `/8`, `/16`, `/24`, and `/0` all matching 10.45.20.5, which entry is selected?
2. What happens to a packet when its TTL reaches 0?
3. Write the Linux command to add a static route to 172.16.0.0/16 via 10.0.0.1.
4. Your routing table has no default route and no matching entry for a destination. What happens?
5. A packet travels through 5 routers. How many times does the Ethernet frame change?
6. Why can't a home router have just one routing table entry (the default route) and still work?
7. What does the `metric` field in a routing table entry represent?

---

*Lecture 7 of 13 — Computer Networks, Term 5, SST*

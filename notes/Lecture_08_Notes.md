# Lecture 8: Routing and Forwarding II
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Explain distance vector routing and trace RIP convergence
- Describe the count-to-infinity problem and its solutions
- Explain link state routing and OSPF operation
- Understand BGP at a conceptual level
- Define convergence and routing loops

---

## 1. Distance Vector Routing

### Core Principle
Each router:
1. Knows costs to **directly connected** neighbors
2. Periodically shares its **entire routing table** with those neighbors
3. Updates routes using: `dist[dest] = min(dist[dest], cost_to_neighbor + neighbor_dist[dest])`

This is a distributed implementation of **Bellman-Ford**.

### Convergence Example (4 routers: A-B-C-D in a line, cost=1 each)

**Initial state (each router only knows direct links):**
```
A: {A:0, B:1, C:∞, D:∞}    B: {A:1, B:0, C:1, D:∞}
C: {A:∞, B:1, C:0, D:1}    D: {A:∞, B:∞, C:1, D:0}
```

**After Round 1 (share with neighbors):**
```
A: {A:0, B:1, C:2, D:∞}    B: {A:1, B:0, C:1, D:2}
C: {A:2, B:1, C:0, D:1}    D: {A:∞, B:2, C:1, D:0}
```

**After Round 2 (converged):**
```
A: {A:0, B:1, C:2, D:3}    B: {A:1, B:0, C:1, D:2}
C: {A:2, B:1, C:0, D:1}    D: {A:3, B:2, C:1, D:0}
```

---

## 2. RIP — Routing Information Protocol

A practical implementation of distance vector routing.

| Property | Value |
|---------|-------|
| Metric | Hop count |
| Maximum hops | 15 (16 = unreachable) |
| Update interval | Every 30 seconds |
| Algorithm | Bellman-Ford (distributed) |
| Scale | Small networks |

| Feature | RIPv1 | RIPv2 |
|---------|-------|-------|
| CIDR support | ❌ | ✅ |
| Authentication | ❌ | ✅ |
| Subnet masks in updates | ❌ | ✅ |

---

## 3. Count-to-Infinity Problem

### Scenario

Network: `A ——1—— B ——1—— C`

**Normal state:** A knows C=2 (via B), B knows C=1 (direct)

**Link B-C fails:**

What should happen: B detects failure, sets C=∞, tells A → A sets C=∞ → Done.

**What can happen (without fixes):**
```
B detects failure: C=∞
Before B notifies A, A sends its old table: {A:0, B:1, C:2}
B thinks: "A can reach C in 2 hops! So I can reach C via A in 3 hops"
B: C=3

B tells A: C=3
A thinks: "B says C=3, via B: C=4"
A: C=4

Continues until C=16 (unreachable in RIP) → Slow convergence!
```

### Solutions

**1. Split Horizon**

Rule: **Never advertise a route back to the neighbor you learned it from.**

In the A-B-C example: A's routing table shows C=2, next hop=B. This means A learned about C *through* B. With split horizon, when A sends its periodic update to B, it omits C entirely from that update.

```
A's update TO B (with split horizon):
  "I know: A=0, B=1"      ← C is silently omitted

Without split horizon:
  "I know: A=0, B=1, C=2" ← B thinks A has an alternative path to C
```

When B-C fails and B sets C=∞, B never hears from A that "A can reach C." So B correctly keeps C=∞ and the loop never starts.

**Why this works:** A's path to C *requires going through B*. If B can't reach C, A can't reach C either. There is no alternative path — so A should never claim to B that it has one.

**2. Poison Reverse**

Instead of silently omitting C, A actively tells B: "C = ∞" (even though A's own table says C=2 via B).

```
A's update TO B (with poison reverse):
  "A=0, B=1, C=∞"    ← explicitly poisons the route back to B
```

This is more aggressive than split horizon — B already knows "A can't reach C independently" before any failure even occurs, so convergence is instantaneous.

**3. Route Poisoning (Triggered Updates)**

When B detects the B-C link failure, instead of waiting for the 30-second periodic timer, B *immediately* sends an update to all neighbors: "C = 16 (unreachable)." This dramatically shrinks the time window during which count-to-infinity can occur.

**4. Holddown Timer**

After a route is poisoned as unreachable, the router ignores any new advertisements claiming the destination is reachable, for a hold period (e.g., 180 seconds). This prevents a stale update from a slow router from re-introducing a bad route during convergence.

---

## 4. Link State Routing — OSPF

### Core Principle
Each router:
1. Discovers neighbors (via **Hello** packets)
2. Creates a **Link State Advertisement (LSA)** describing its links
3. **Floods** LSAs to ALL routers in the network
4. Builds a complete **Link State Database (LSDB)**
5. Runs **Dijkstra (SPF)** on the LSDB
6. Installs results in routing table

### OSPF Operation Steps

```
Step 1 — Neighbor Discovery:
  Send Hello packets every 10 seconds
  Agree on: Hello interval, area ID, authentication
  → Become OSPF neighbors

Step 2 — Adjacency Formation:
  Exchange DBD (Database Description) packets
  Request missing LSAs with LSR (Link State Request)
  Receive LSAs via LSU (Link State Update)
  → Full adjacency formed

Step 3 — LSA Flooding:
  Each router: "I am Router R1. Link to R2 cost=10, link to R3 cost=5"
  LSA flooded to ALL routers in area
  Each router stores all LSAs in LSDB

Step 4 — SPF Calculation:
  Run Dijkstra on LSDB
  Find shortest path to every destination
  
Step 5 — Route Installation:
  Best paths installed in routing table
```

### OSPF Cost Calculation

```
Cost = Reference Bandwidth / Interface Bandwidth
Reference Bandwidth = 100 Mbps (default)

100 Mbps Ethernet:  Cost = 100/100 = 1
10 Mbps Ethernet:   Cost = 100/10  = 10
1 Mbps serial link: Cost = 100/1   = 100
```

Lower cost = preferred path.

### OSPF Areas

Large networks use multiple areas to limit LSA flooding:

```
    [Area 1]            [Area 0 — Backbone]            [Area 2]
    R4 — R5 ——ABR—— R1 — R2 — R3 ——ABR—— R6 — R7
```

- All areas must connect to Area 0
- LSAs flooded only within an area
- ABR (Area Border Router): sits at boundary, summarizes routes

---

## 5. BGP — Border Gateway Protocol

### What is BGP?

- **EGP (Exterior Gateway Protocol)**: handles routing **between** Autonomous Systems
- Used on every internet core router
- Policy-driven: business relationships dictate routes, not just path length

### Autonomous System (AS)

A network under one administrative control, with a unique **ASN (AS Number)**.

**Examples:**
- Google: AS15169, Cloudflare: AS13335, Jio: AS55836, BSNL: AS9829

### BGP — Path Vector Protocol

BGP advertises the **full AS path** to reach a prefix:

```
Google advertises: 8.8.8.0/24, AS_PATH: [15169]
Tata receives & re-advertises: 8.8.8.0/24, AS_PATH: [6453, 15169]
Airtel receives & re-advertises: 8.8.8.0/24, AS_PATH: [9498, 6453, 15169]
```

**Loop prevention:** If a router sees its own ASN in the AS_PATH → discard the route.

### BGP Route Selection (simplified)

When multiple paths exist to same prefix, BGP prefers (in order):
1. Higher **Local Preference** (set within your own AS — policy)
2. Shorter **AS Path** (fewer ASes traversed)
3. Lower **MED** (Multi-Exit Discriminator, hint from neighbor)
4. **eBGP > iBGP** (external routes preferred over internal)
5. Lower **IGP metric** to next hop

### iBGP vs eBGP

| Type | Between | Use |
|------|---------|-----|
| **eBGP** | Different ASes | Exchange routes across internet |
| **iBGP** | Same AS | Distribute external routes internally |

### BGP Hijacking

**What it is:** An AS maliciously announces a more specific prefix (e.g., /24 instead of /16) for someone else's IP space.

**Why it works:** BGP routers trust their peers; no authentication by default.

**Example:** Pakistan Telecom announcing YouTube's /24 → routers worldwide chose the more specific route → YouTube traffic sent to Pakistan.

**Modern defense:** RPKI (Resource Public Key Infrastructure) — cryptographic signing of route announcements.

---

## 6. Convergence

**Convergence** = state when all routers have consistent, accurate routing tables.

**Convergence time** = time to stabilize after a topology change.

| Protocol | Convergence Time | Why |
|----------|----------------|-----|
| RIP | 30 sec – minutes | 30s update intervals |
| OSPF | 1–10 seconds | Triggered, flooded updates |
| BGP | Minutes – hours | Policy processing, dampening |

**OSPF Convergence Process:**
1. Link failure detected via dead Hello timer (~40 seconds default, can be tuned to < 1s)
2. Originating router generates new LSA immediately
3. LSA flooded to entire area
4. All routers run SPF
5. New routes installed

---

## 7. Routing Loops

**Routing loop** = two or more routers send packets for a destination to each other in a cycle.

```
A's table: 10.1.0.0/24 → via B
B's table: 10.1.0.0/24 → via A   ← LOOP

Packet to 10.1.0.0 → A sends to B → B sends to A → A sends to B → ...
TTL reaches 0 → packet dropped, ICMP Time Exceeded sent to source
```

### Loop Prevention Techniques

| Technique | Protocol | How |
|-----------|---------|-----|
| **TTL** | IP (universal) | Packets die after N hops |
| **Split Horizon** | RIP | Don't advertise route back to source |
| **Poison Reverse** | RIP | Advertise ∞ back to source |
| **AS Path** | BGP | Contains own ASN → discard |
| **Full topology** | OSPF | Complete map → consistent, loop-free |
| **STP** | Ethernet | Disable redundant links |

---

## Protocol Comparison Summary

| Feature | RIP | OSPF | BGP |
|---------|-----|------|-----|
| Type | Distance Vector | Link State | Path Vector |
| Algorithm | Bellman-Ford | Dijkstra (SPF) | Policy-based |
| Metric | Hop count | Cost (bandwidth) | Policy attributes |
| Scope | Within AS (IGP) | Within AS (IGP) | Between ASes (EGP) |
| Max size | ~15 hops | Large (with areas) | Internet-scale |
| Convergence | Slow | Fast | Slow |
| Loop prevention | Split horizon | Full topology | AS Path |
| Standard | RFC 2453 | RFC 2328 | RFC 4271 |

---

## 📌 Key Takeaways

1. **Distance Vector (RIP)**: share routing table; distributed Bellman-Ford; count-to-infinity is the main problem
2. **Link State (OSPF)**: flood LSAs; all routers run Dijkstra; no count-to-infinity; fast convergence
3. **BGP**: path-vector; between ASes; AS_PATH for loop prevention; policy-driven
4. **Convergence**: time for all routers to agree; OSPF converges fast, RIP slow
5. **Routing loops**: detected by TTL=0; prevented by split horizon, AS path, OSPF topology

---

## 🧠 Quick Self-Check Questions

1. In RIP, what happens when a destination becomes unreachable (beyond 15 hops)?
2. Explain why split horizon prevents count-to-infinity.
3. In OSPF, what is the LSDB and which algorithm is run on it?
4. What is an Autonomous System? Give two examples.
5. Why is BGP called a path-vector protocol rather than distance-vector?
6. Your network has two paths to a destination. One is 3 hops (via a 10 Mbps link), one is 5 hops (via a 1 Gbps link). Which does RIP prefer? Which does OSPF prefer?
7. How does BGP prevent routing loops between ASes?

---

*Lecture 8 of 13 — Computer Networks, Term 5, SST*

# Lecture 8: Routing and Forwarding II
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Explain Distance Vector routing and trace how routes are exchanged
- Explain Link State routing and how OSPF works
- Understand BGP at a high level
- Define routing convergence and explain routing loops
- Compare routing protocols by their characteristics

**Duration:** ~90 minutes

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Tell this story]**

> "In 1997, a Pakistani ISP called PCCW accidentally announced that it was the best path to reach YouTube — basically to the entire internet. Every router on the internet believed it. For about 2 hours, millions of people trying to reach YouTube were having their traffic sent to Pakistan instead, where it got dropped. This is called a **BGP hijacking**."

> "In 2010, China Telecom announced routes for 15% of the entire internet for 18 minutes — traffic from US military sites, companies, and ISPs briefly flowed through China."

> "How is it possible that one router's announcement can break the internet? Because routing protocols TRUST what their neighbors tell them. Today we're going to understand exactly how that trust works — and why it can be exploited."

---

## SECTION 1: Distance Vector Routing (20 minutes)

### The Core Idea

> "In distance vector routing, each router tells its neighbors: 'Here's my routing table — here's how far I am from every destination I know.'"

> "Each router's view is like looking at directions with only the distances — 'Mumbai is 5 hops,' 'Bangalore is 3 hops' — without knowing the exact path. You only know the distance (hence 'distance vector') and which neighbor to use."

### Bellman-Ford in Action — Distributed

**[INSTRUCTOR: Draw 4 routers: A-B-C-D in a line]**

```
A ——1—— B ——1—— C ——1—— D
```

**Initial state** (each router only knows its directly connected links):

```
Router A's table:     Router B's table:
A: 0                  A: 1
B: 1                  B: 0
C: ∞                  C: 1
D: ∞                  D: ∞

Router C's table:     Router D's table:
A: ∞                  A: ∞
B: 1                  B: ∞
C: 0                  C: 1
D: 1                  D: 0
```

**Round 1 — Routers share tables with direct neighbors:**

```
B learns from A: A=0, B=1, C=∞, D=∞ → A is 1 hop away, so via A: A=1
B learns from C: A=∞, B=1, C=0, D=1 → C is 1 hop, so via C: C=1, D=2
B updates: {A:1, B:0, C:1, D:2}

A learns from B: {A:1, B:0, C:1, D:2} → via B: B=1, C=2, D=3
A updates: {A:0, B:1, C:2, D:3}
```

**Round 2 — Share again:**

```
Everything converges. Each router now knows distance to every other.
A: {A:0, B:1, C:2, D:3}
B: {A:1, B:0, C:1, D:2}
C: {A:2, B:1, C:0, D:1}
D: {A:3, B:2, C:1, D:0}
```

> "This is the essence of RIP. Routes propagate outward through the network like ripples in a pond."

### RIP — Routing Information Protocol

**Properties:**
- Hop count as metric (max 15; 16 = unreachable)
- Sends full routing table every 30 seconds
- Very simple to configure; good for small networks
- **Problems:** Slow convergence, count-to-infinity

**RIP version differences:**
| Feature | RIPv1 | RIPv2 |
|---------|-------|-------|
| Classless (CIDR) | ❌ | ✅ |
| Authentication | ❌ | ✅ |
| Subnet mask in updates | ❌ | ✅ |

### Count-to-Infinity Problem

> "Here's the famous problem with distance vector."

**[INSTRUCTOR: Draw: A ——1—— B ——1—— C]**

```
Normal state:
A knows: B=1, C=2 (via B)
B knows: A=1, C=1
C knows: A=2, B=1

Now: link B-C fails.
```

**What should happen:**
```
C: now unreachable via B
B: updates C = ∞
A: updates C = ∞
Done.
```

**What actually happens (without fix):**

```
B detects failure: C=∞ in B's table

Before B can tell A, A sends its table to B:
"A knows C=2" → B thinks: "A can reach C in 2 hops!
So I can reach C via A in 2+1=3 hops"
B updates: C=3

B tells A: C=3
A thinks: "B says C=3, I can reach C via B in 3+1=4"
A updates: C=4

A tells B: C=4
B thinks: C=5 via A
...

This ping-ponging continues until it hits 16 (unreachable in RIP)
```

**Fixes:**
1. **Split Horizon:** Don't advertise a route BACK to the neighbor you learned it from
   - B learned C via C directly, so don't tell A that B can reach C (B doesn't know C via A)
   - Actually: B learned C directly, not from A, so B does tell A C=1 normally
   - But: A learned C=2 via B, so A should NOT tell B that A can reach C → fixes the problem!
   
2. **Poison Reverse:** Instead of not advertising, send infinity
   - A sends to B: "C = ∞" (even though A thinks C=2 via B)
   - This explicitly tells B "I cannot reach C independently of you"

3. **Route Poisoning:** When a route fails, immediately propagate infinity to all neighbors

---

## SECTION 2: Link State Routing and OSPF (20 minutes)

### The Link State Approach

> "Instead of sharing routing tables, link state routers share their local knowledge: 'Here are my neighbors and the cost to reach each of them.' This is called a **Link State Advertisement (LSA)**."

> "Every router FLOODS its LSA to the entire network. Eventually, every router has a complete map — they all see the same topology. Then each independently runs Dijkstra."

### How OSPF Works — Step by Step

**Step 1 — Hello Protocol:**
> "When OSPF routers are first connected, they send 'Hello' packets to discover neighbors. If two routers agree on parameters (hello interval, area ID, etc.), they become **OSPF neighbors**."

**Step 2 — Form Adjacency:**
> "After neighbor discovery, OSPF establishes **adjacencies** — full two-way relationships where routers will exchange LSAs."

**Step 3 — Flood LSAs:**
> "Each router creates an LSA describing its links: 'I am Router A. I have links to Router B (cost 5) and Router C (cost 3).' This LSA is flooded to ALL routers in the area."

**Step 4 — Build LSDB:**
> "Every router stores all received LSAs in its **Link State Database (LSDB)**. The LSDB is the complete topology map."

**Step 5 — Run Dijkstra:**
> "Using the LSDB, each router runs SPF (Shortest Path First = Dijkstra) to calculate the best path to every destination."

**Step 6 — Install Routes:**
> "The results of SPF are installed in the routing table."

### OSPF Areas

> "OSPF has a concept of **areas** to scale to large networks. The backbone is Area 0 (Area 0 connects all areas)."

```
        [Area 1]
         R4 — R5
          \
           R2 — [Area 0] — R3
          /
         R1
        [Area 2]
```

> "LSAs are only flooded within an area. Area Border Routers (ABRs) summarize routes between areas. This reduces the size of the LSDB each router must maintain."

### OSPF vs RIP — Quick Comparison

| Feature | RIP | OSPF |
|---------|-----|------|
| Algorithm | Distance Vector / Bellman-Ford | Link State / Dijkstra |
| Metric | Hop count (max 15) | Cost (bandwidth-based) |
| Convergence | Slow (30s updates) | Fast (triggered updates) |
| Topology knowledge | Partial | Full |
| Scale | Small networks | Enterprise / ISP |
| Count-to-infinity | Yes | No |
| VLSM support | RIPv2 only | Yes |

---

## SECTION 3: BGP — Border Gateway Protocol (15 minutes)

### What is BGP?

> "BGP is the routing protocol that makes the internet work — it's how different ISPs and organizations exchange routing information with each other."

> "BGP is an EGP — Exterior Gateway Protocol. While OSPF handles routing WITHIN an organization's network (AS), BGP handles routing BETWEEN organizations (ASes)."

**[INSTRUCTOR: Draw the internet as a set of bubbles (ASes) connected by BGP]**

```
[Jio AS 55836] ——BGP—— [BSNL AS 9829] ——BGP—— [Google AS 15169]
       |
      BGP
       |
[Airtel AS 9498] ——BGP—— [Tata AS 6453]
```

### ASes — Autonomous Systems

> "An **Autonomous System (AS)** is a network under one administrative domain. Your college, your ISP, Amazon AWS, Google — each is an AS with a unique **AS Number (ASN)** assigned by IANA."

**ASN examples:**
- Google: AS15169
- Cloudflare: AS13335
- Jio: AS55836
- BSNL: AS9829

**[Students can verify: look up their ISP's ASN at bgp.he.net]**

### How BGP Works — Path Vector Protocol

> "BGP is a **path vector** protocol. Instead of advertising just a distance, BGP advertises the FULL AS PATH to a destination."

**Example BGP announcement:**
```
"I can reach 8.8.8.0/24 via path: [AS15169]"
(Google announces its own prefix with just its own AS number)

When Tata receives this, it adds its own AS:
"I can reach 8.8.8.0/24 via path: [AS6453, AS15169]"

When Airtel receives this from Tata:
"I can reach 8.8.8.0/24 via path: [AS9498, AS6453, AS15169]"
```

**Loop prevention:** If a router sees its OWN AS number in the AS path → discard! (Loop detected)

### BGP Decision Process (Simplified)

> "When BGP receives multiple routes to the same prefix, it selects the BEST one based on these attributes (in order):"

1. **Local Preference** — how much does your own AS prefer this route? (higher = better)
2. **AS Path length** — shorter is better
3. **Origin** — IGP > EGP > Incomplete
4. **MED (Multi-Exit Discriminator)** — preference between paths to same neighbor
5. **eBGP over iBGP** — prefer routes from external BGP neighbors
6. **IGP metric to next hop** — closest exit point

> "This is important: BGP is NOT just about shortest path. It's heavily influenced by BUSINESS RELATIONSHIPS between ISPs. A Tier-2 ISP might prefer to send traffic through a paid transit provider vs. a free peering partner based on agreements, not hop count."

### Why BGP Hijacking Happens

> "BGP trusts what its neighbors announce. If ISP A announces '8.8.8.0/24 is reachable via me,' all BGP peers will believe it and potentially route traffic that way — even if A has no legitimate connection to Google's 8.8.8.0/24."

> "BGP Security (BGPsec) and RPKI (Resource Public Key Infrastructure) are modern solutions that digitally sign route announcements. Still being adopted worldwide."

### iBGP vs eBGP

| Type | Full Name | Used Between |
|------|-----------|-------------|
| eBGP | External BGP | Routers in DIFFERENT ASes |
| iBGP | Internal BGP | Routers in the SAME AS |

> "Inside a large ISP, all their routers need to know about external routes. They use iBGP to share BGP routes internally (OSPF handles internal topology separately)."

---

## SECTION 4: Convergence (10 minutes)

### What is Convergence?

> "A network has **converged** when all routers have consistent, accurate routing tables. During convergence, some packets may be misrouted or dropped."

> "**Convergence time** is how long it takes the network to stabilize after a topology change (link failure, new router added, etc.)."

### Convergence by Protocol

| Protocol | Typical Convergence | Reason |
|----------|---------------------|--------|
| RIP | 30 seconds to minutes | 30s update interval |
| OSPF | 1-10 seconds | Triggered updates, faster |
| BGP | Minutes to hours | Policies, route dampening |

**OSPF convergence process:**
1. Link failure detected (usually within 1-10 seconds via Hello timer)
2. Router generates new LSA, floods it to ALL routers in area
3. All routers receive LSA and re-run SPF
4. New routes installed
5. Converged!

### Routing Loops

> "A **routing loop** occurs when packets circulate between two or more routers, never reaching their destination."

**Example:**
```
Router A's table: to reach 10.1.0.0 → send to B
Router B's table: to reach 10.1.0.0 → send to A  ← LOOP!

Packet from outside: destination 10.1.0.0
→ A sends to B → B sends back to A → A sends to B → ...
TTL eventually reaches 0 → packet dropped
```

**How detected:** TTL = 0 → ICMP Time Exceeded sent back to source.

**Prevention mechanisms:**
- **TTL:** Kills looping packets before infinite loop
- **Split Horizon / Poison Reverse:** Prevents count-to-infinity in distance vector
- **AS Path in BGP:** Prevents inter-domain loops
- **OSPF full topology:** Can't create loops because everyone has same map

---

## SUMMARY (5 minutes)

```
✅ Distance Vector (RIP): share routing table with neighbors; Bellman-Ford
   - Simple, slow convergence, count-to-infinity problem
   - Fix: Split Horizon, Poison Reverse, Route Poisoning

✅ Link State (OSPF): share LSAs with ALL; Dijkstra
   - Full topology, fast convergence, scalable with areas
   - Flood LSAs → Build LSDB → Run SPF → Install routes

✅ BGP: path-vector EGP for inter-AS routing
   - Advertises full AS PATH (loop prevention)
   - Policy-driven (business relationships, not just shortest path)
   - BGP hijacking: router maliciously announces others' prefixes

✅ Convergence: time for all routers to reach consistent state
✅ Routing loops: packets circulate forever; TTL kills them
```

---

*End of Lecture 8 Script*

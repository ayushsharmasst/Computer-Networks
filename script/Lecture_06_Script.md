# Lecture 6: Graph Algorithms for Networking II
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Apply Bellman-Ford algorithm and understand its role in routing
- Understand Minimum Spanning Trees conceptually
- Connect graph algorithms to real routing protocol design decisions

**Duration:** ~90 minutes

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: On the board, add a negative edge to last lecture's graph]**

> "Last lecture we used Dijkstra to find shortest paths. But Dijkstra has a constraint — it only works with non-negative edge weights."

> "What does a 'negative cost' link mean in networking? Imagine two ISPs — ISP A and ISP B. ISP B actually PAYS ISP A to route traffic through it (traffic-pumping scam, or peering arrangements). From ISP A's perspective, the cost of using that link is negative — they gain money. Or think of it as 'preference': if a link has cost -2, it means routers prefer it above others."

> "Dijkstra breaks with negative edges. So how does the internet handle real-world cost assignments? Enter Bellman-Ford."

---

## SECTION 1: Why Dijkstra Fails on Negative Edges (8 minutes)

**[INSTRUCTOR: Draw this tiny graph]**

```
     A ——2——→ B
      \      ↑
       4    -3
        \  /
         C
```

**Correct shortest path A → B:**
- A → B: cost 2
- A → C → B: cost 4 + (-3) = 1 ← cheaper!

**Dijkstra's trace:**
```
Init: dist = {A:0, B:∞, C:∞}, PQ = [(0,A)]

Step 1: Pop A (cost 0)
  → B: cost 2 → dist[B]=2
  → C: cost 4 → dist[C]=4
  PQ = [(2,B), (4,C)]

Step 2: Pop B (cost 2) — mark B as FINAL
  B is done. dist[B] = 2 ← WRONG!

Step 3: Pop C (cost 4)
  → B: new cost = 4-3 = 1 < 2, but B is already finalized!
  Dijkstra ignores this update — incorrect result
```

**Say:** 
> "Dijkstra's greedy assumption — 'once finalized, a node has its shortest path' — breaks when negative edges can create cheaper paths through later-explored nodes."

---

## SECTION 2: Bellman-Ford Algorithm (20 minutes)

### Core Idea

> "Instead of being greedy, Bellman-Ford is exhaustive. It relaxes ALL edges, V-1 times. Each iteration can potentially update the shortest path to any node."

> "Why V-1 times? Because the shortest path in a graph with V nodes can have at most V-1 edges (before it would have to revisit a node, creating a cycle)."

### Algorithm

```
Bellman-Ford(graph, source):
    dist = {node: ∞ for all nodes}
    dist[source] = 0
    prev = {node: None for all nodes}
    
    // Relax all edges V-1 times
    for i in range(V - 1):
        for each edge (u, v, weight) in all_edges:
            if dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                prev[v] = u
    
    // Check for negative cycles (V-th pass)
    for each edge (u, v, weight) in all_edges:
        if dist[u] + weight < dist[v]:
            // A negative cycle exists!
            return "Negative cycle detected"
    
    return dist, prev
```

### Python Implementation

```python
def bellman_ford(edges, vertices, source):
    """
    edges: list of (u, v, weight)
    vertices: list of all node names
    source: starting node
    """
    dist = {v: float('inf') for v in vertices}
    dist[source] = 0
    prev = {v: None for v in vertices}
    
    # Relax V-1 times
    for _ in range(len(vertices) - 1):
        for u, v, w in edges:
            if dist[u] != float('inf') and dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                prev[v] = u
    
    # Detect negative cycles
    for u, v, w in edges:
        if dist[u] != float('inf') and dist[u] + w < dist[v]:
            raise ValueError("Graph contains a negative weight cycle")
    
    return dist, prev
```

### Live Trace — The Negative Edge Problem

```
Graph edges:
(A, B, 2), (A, C, 4), (C, B, -3)

vertices = [A, B, C]
source = A

Initial: dist = {A:0, B:∞, C:∞}

Iteration 1 (relax all edges):
  Edge (A,B,2): dist[A]+2=2 < ∞ → dist[B]=2
  Edge (A,C,4): dist[A]+4=4 < ∞ → dist[C]=4
  Edge (C,B,-3): dist[C]+(-3)=1 < 2 → dist[B]=1 ✓

After Iteration 1: {A:0, B:1, C:4}

Iteration 2 (relax all edges again):
  Edge (A,B,2): 0+2=2 > 1 → no update
  Edge (A,C,4): 0+4=4 = 4 → no update
  Edge (C,B,-3): 4+(-3)=1 = 1 → no update

Stable → DONE
Final: {A:0, B:1, C:4}
Path A→B: A→C→B (cost 1) ✓ Correct!
```

**[INSTRUCTOR: Point out Bellman-Ford correctly finds cost 1, while Dijkstra got 2]**

### Negative Cycle Detection

> "What happens if there's a negative cycle — a loop where you can keep reducing the path cost by going around it?"

**Example:**
```
A → B (cost 1) → C (cost -3) → A (cost 1)
Loop cost: 1 + (-3) + 1 = -1 per cycle
Each loop makes cost more negative — infinite loop!
```

> "Bellman-Ford detects this: if on the V-th pass ANY edge can still be relaxed, a negative cycle exists."

**Network meaning:**
> "Negative cycles in routing = routing loops. Packets would ping-pong between routers forever, each claiming it has a better path. BGP and RIP have loop prevention mechanisms. We'll see this in Lecture 8."

### Bellman-Ford vs Dijkstra

| Feature | Dijkstra | Bellman-Ford |
|---------|----------|-------------|
| Handles negative edges | ❌ No | ✅ Yes |
| Handles negative cycles | ❌ No | ✅ Detects them |
| Time complexity | O((V+E) log V) | O(V·E) |
| Approach | Greedy | Dynamic programming / relaxation |
| Network use | OSPF | RIP, BGP (conceptually) |

**Practical note:**
> "The internet doesn't literally have negative edge weights — but Bellman-Ford's distributed nature (where each router tells its neighbors its distances) maps perfectly to how RIP and BGP work in the real world. Each router is doing a form of distributed relaxation."

---

## SECTION 3: Minimum Spanning Tree — Conceptual (15 minutes)

### What is a Spanning Tree?

> "A **spanning tree** of a graph is a subgraph that:
> - Connects all vertices (spans the whole graph)
> - Has no cycles
> - Has exactly V-1 edges (minimum needed to connect V nodes)"

**[INSTRUCTOR: Draw the Mumbai graph and circle a spanning tree]**

### Minimum Spanning Tree (MST)

> "An **MST** is the spanning tree where the **total edge weight is minimized**. Among all possible trees that connect all nodes, find the one with the lowest total cost."

**Real network use case:**
> "You're laying fiber optic cables between 10 cities. Each possible cable route has a cost (distance, terrain). What's the minimum cost cable layout that keeps all cities connected? → MST problem."

### MST Algorithms (Conceptual)

**Prim's Algorithm** (similar to Dijkstra):
- Start from any node
- Greedily add the cheapest edge that connects a new node to the current tree
- Repeat until all nodes are included

**Kruskal's Algorithm**:
- Sort all edges by weight
- Add edges one by one (cheapest first) if they don't create a cycle
- Use Union-Find to check for cycles efficiently

### MST Trace — Mumbai Network

**Sorted edges:**
```
Hyderabad-Bangalore: 2
Delhi-Hyderabad:    2
Mumbai-Hyderabad:   3
Mumbai-Delhi:       5
Chennai-Bangalore:  4
Delhi-Chennai:      8
```

**Kruskal's (add cheapest, no cycle):**
```
1. Add (Hyderabad-Bangalore, 2) → OK
2. Add (Delhi-Hyderabad, 2) → OK (connects Delhi)
3. Add (Mumbai-Hyderabad, 3) → OK (connects Mumbai)
4. Add (Chennai-Bangalore, 4) → OK (connects Chennai)
5. Check (Mumbai-Delhi, 5) → would create cycle → SKIP
6. Check (Delhi-Chennai, 8) → would create cycle → SKIP

MST = {H-B:2, D-H:2, M-H:3, C-B:4} = total cost 11
```

### MST in Networking — Spanning Tree Protocol (STP)

> "In Ethernet networks, if you have multiple switches and redundant links, you get loops — a broadcast frame would loop forever. STP (Spanning Tree Protocol, IEEE 802.1D) runs an MST-like algorithm to identify and DISABLE redundant links, creating a loop-free tree."

```
Switch A — Switch B — Switch C
    \__________________/
    (redundant link)

STP: Calculates a spanning tree and blocks one link:
Switch A — Switch B — Switch C
    (blocked: A directly to C)
```

> "If a link fails, STP recalculates and unblocks another link. This gives you redundancy without loops."

**[INSTRUCTOR: Draw 3 switches with redundant connections and show which link STP would block]**

---

## SECTION 4: Why Routing Uses Graphs (10 minutes)

### The Routing Problem Formalized

> "Routing is, fundamentally, a shortest path problem on a graph:"

```
Given:
  - Graph G = (Routers, Links)
  - Edge weights = link costs (latency, bandwidth, etc.)
  - Source = your machine
  - Destination = Google's server

Find:
  - Minimum cost path from Source to Destination
```

### Two Families of Routing Protocols

**Distance Vector (Bellman-Ford based):**
- Each router tells its NEIGHBORS its distance to every destination
- Neighbors update their own tables
- After many rounds, converges to correct routes
- Used by: RIP (older), BGP (in a modified form)

**Link State (Dijkstra based):**
- Each router FLOODS its link information to ALL routers
- Every router builds a complete map of the network
- Every router runs Dijkstra independently
- Used by: OSPF, IS-IS

```
Distance Vector:
Router A tells Router B: "I can reach Google in 3 hops"
Router B tells Router C: "I can reach Google in 4 hops (via A)"

Link State:
Router A tells ALL routers: "My link to B costs 2, my link to C costs 5"
Everyone builds the same complete graph and runs Dijkstra
```

**[INSTRUCTOR: Draw both scenarios on board to visualize the difference]**

### Why Link State Scales Better

> "Distance vector has a serious problem: **count-to-infinity**. If Router A goes down, its neighbor B still thinks it can reach A's destinations through C, and C thinks it can reach them through B. They keep incrementing the hop count until it hits the maximum (15 in RIP). Meanwhile, packets loop."

> "Link state avoids this because every router has the FULL picture. If any router detects a link failure, it floods the info to everyone, and everyone re-runs Dijkstra with the correct topology."

---

## SECTION 5: Connecting Everything — The Internet's Routing Architecture (5 minutes)

> "The internet is divided into **Autonomous Systems (AS)** — basically, each ISP or large organization controls their own AS."

**Inside an AS:** OSPF (link-state, Dijkstra) — full topology knowledge, fast convergence

**Between ASes:** BGP — distance-vector like, but with path information to prevent loops; doesn't use simple cost metrics but policy and business relationships

```
Your ISP (AS 1234) → BSNL (AS 9829) → Tata (AS 6453) → Google (AS 15169)
       [OSPF inside]     [BGP between]      [BGP]       [OSPF inside]
```

---

## SUMMARY (5 minutes)

```
✅ Dijkstra fails on negative edges (greedy assumption breaks)
✅ Bellman-Ford: relax ALL edges V-1 times → handles negative edges
✅ Bellman-Ford negative cycle detection: V-th pass still relaxes → cycle
✅ Bellman-Ford: O(V·E) — slower but more correct than Dijkstra
✅ MST: minimum cost tree connecting all nodes
    - Kruskal: sort edges, add cheapest without cycle
    - Prim: grow tree greedily from one node
    - STP in Ethernet uses MST to prevent L2 loops
✅ Routing families:
    - Distance Vector (Bellman-Ford) → RIP, BGP
    - Link State (Dijkstra) → OSPF, IS-IS
```

---

## 📝 Exercise

1. Run Bellman-Ford on this graph from A:
   - Edges: A→B(1), B→C(-2), C→D(3), A→D(10)
   - What are final shortest distances?

2. In the Mumbai network, what's the MST? Draw it and calculate total cost.

3. Why does RIP have a maximum hop count of 15? What problem would occur without it?

---

*End of Lecture 6 Script*

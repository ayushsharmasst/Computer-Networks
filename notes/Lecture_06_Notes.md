# Lecture 6: Graph Algorithms for Networking II
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Understand why Dijkstra fails on negative-weight edges
- Implement and trace Bellman-Ford algorithm
- Understand Minimum Spanning Trees and their network applications
- Connect graph algorithms to routing protocol design (Distance Vector vs. Link State)

---

## 1. Why Dijkstra Fails on Negative Edges

Dijkstra's greedy approach assumes: **once a node is finalized, its shortest distance is correct**.

This assumption breaks when negative edges exist — a longer path might later become cheaper.

**Counterexample:**
```
     A ——2——→ B
      \      ↑
       4    -3
        \  /
         C

Shortest path A → B: A → C → B (cost 4 + (-3) = 1)
Dijkstra finds: A → B (cost 2) — WRONG
```

**Root cause:** Dijkstra finalizes B (cost 2) before exploring C → B (cost 1).

---

## 2. Bellman-Ford Algorithm

### Core Idea
Relax all edges **V − 1** times. Each iteration propagates shortest paths one more hop further.

**Why V−1?** Any shortest path without a negative cycle visits at most V−1 edges (V nodes).

### Algorithm

```python
def bellman_ford(edges, vertices, source):
    """
    edges: list of (u, v, weight) tuples
    vertices: list of all node names
    source: starting node
    """
    dist = {v: float('inf') for v in vertices}
    dist[source] = 0
    prev = {v: None for v in vertices}

    # Step 1: Relax all edges V-1 times
    for _ in range(len(vertices) - 1):
        for u, v, w in edges:
            if dist[u] != float('inf') and dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                prev[v] = u

    # Step 2: Detect negative cycles (V-th pass)
    for u, v, w in edges:
        if dist[u] != float('inf') and dist[u] + w < dist[v]:
            raise ValueError("Negative weight cycle detected!")

    return dist, prev
```

### Bellman-Ford Trace

**Graph:** Edges: (A,B,2), (A,C,4), (C,B,-3)  
**Source:** A

```
Initial: dist = {A:0, B:∞, C:∞}

Iteration 1:
  Process (A,B,2): 0+2=2 < ∞ → dist[B] = 2
  Process (A,C,4): 0+4=4 < ∞ → dist[C] = 4
  Process (C,B,-3): 4+(-3)=1 < 2 → dist[B] = 1

After Iteration 1: {A:0, B:1, C:4}

Iteration 2:
  (A,B,2): 0+2=2 > 1 → no update
  (A,C,4): 0+4=4 = 4 → no update
  (C,B,-3): 4+(-3)=1 = 1 → no update

Converged!
Final: {A:0, B:1, C:4}
Path A→B: A → C → B (cost 1) ✓
```

### Negative Cycle Detection

If on the **V-th** pass any edge can still be relaxed → **negative cycle exists**.

**Negative cycle example:**
```
A → B (1) → C (-3) → A (1)
Cycle cost = 1 + (-3) + 1 = -1 per round
→ Path cost decreases indefinitely → meaningless "shortest path"
```

**In networking:** A negative cycle = a **routing loop** where packets circulate forever.

---

## 3. Dijkstra vs. Bellman-Ford

| Feature | Dijkstra | Bellman-Ford |
|---------|----------|-------------|
| Negative edges | ❌ Fails | ✅ Handles |
| Negative cycles | ❌ Cannot detect | ✅ Detects |
| Time complexity | O((V+E) log V) | O(V × E) |
| Strategy | Greedy | Dynamic programming |
| Network protocol | OSPF | RIP, BGP (conceptually) |
| When to use | Non-negative weights, performance critical | Negative weights, loop detection needed |

---

## 4. Minimum Spanning Tree (MST)

### Definition

A **Spanning Tree** of a graph:
- Connects **all vertices**
- Has **no cycles**
- Contains exactly **V−1 edges**

A **Minimum Spanning Tree** is the spanning tree with the **lowest total edge weight**.

### MST Algorithms

#### Kruskal's Algorithm
1. Sort all edges by weight (ascending)
2. For each edge: add it if it doesn't create a cycle (use Union-Find)
3. Stop when V−1 edges are added

```python
def kruskal(edges, vertices):
    # Sort edges by weight
    edges.sort(key=lambda x: x[2])
    
    parent = {v: v for v in vertices}
    
    def find(x):
        if parent[x] != x:
            parent[x] = find(parent[x])
        return parent[x]
    
    def union(x, y):
        px, py = find(x), find(y)
        if px == py:
            return False  # Cycle!
        parent[px] = py
        return True
    
    mst = []
    for u, v, w in edges:
        if union(u, v):
            mst.append((u, v, w))
    
    return mst
```

#### Prim's Algorithm
1. Start from any node
2. Greedily add the minimum-weight edge that connects a new node to the current tree
3. Repeat until all nodes are in the tree

### MST Trace — Mumbai Network

**Edges sorted by weight:**
```
(Hyderabad-Bangalore, 2)
(Delhi-Hyderabad, 2)
(Mumbai-Hyderabad, 3)
(Chennai-Bangalore, 4)
(Mumbai-Delhi, 5)
(Delhi-Chennai, 8)
```

**Kruskal's steps:**
```
1. H-B (2): Add → Tree: {H, B}
2. D-H (2): Add → Tree: {H, B, D}
3. M-H (3): Add → Tree: {H, B, D, M}
4. C-B (4): Add → Tree: {H, B, D, M, C} — all connected!

MST Edges: {H-B:2, D-H:2, M-H:3, C-B:4}
Total cost: 2+2+3+4 = 11
```

### MST in Networking — Spanning Tree Protocol (STP)

**Problem:** Ethernet switches with redundant links create loops — broadcast frames circulate forever.

**Solution:** IEEE 802.1D Spanning Tree Protocol (STP)
- Routers run a distributed MST algorithm
- Redundant links are **logically blocked**
- Result: a loop-free tree topology

```
Physical (with redundant links):       Logical (STP blocks one link):
  SW-A ——— SW-B ——— SW-C                SW-A ——— SW-B ——— SW-C
     \________________/                   (link A-C blocked)
```

**Failover:** If SW-A's link to SW-B fails, STP detects it and unblocks the A-C link.

---

## 5. Routing Protocol Families

### Distance Vector Routing (Bellman-Ford based)

**How it works:**
- Each router knows costs to its **direct neighbors**
- Periodically broadcasts its **entire routing table** to neighbors
- Neighbors update their tables based on received info
- Converges over many rounds

```
Router A's table:      Router B gets A's table:
Destination | Cost     B's view: "To reach C, go via A → A says cost 3"
A           | 0        So B→C via A = B→A cost + A→C cost
B           | 2
C           | 3
```

**Problem — Count to Infinity:**
If a route breaks, routers keep incrementing cost until they reach maximum (15 in RIP), causing slow convergence.

**Protocols:** RIP (max 15 hops), BGP (uses path vector to avoid loops)

### Link State Routing (Dijkstra based)

**How it works:**
- Each router floods its **local link state** (neighbors + costs) to ALL routers
- Every router builds a **complete topology map**
- Every router independently runs **Dijkstra** to compute shortest paths

```
All routers know:
Mumbai ——3——→ Hyderabad ——2——→ Bangalore
        ——5——→ Delhi ——2——→ Hyderabad...

Each runs Dijkstra → same result → consistent routing
```

**Advantage:** No count-to-infinity; fast convergence when topology changes.

**Protocols:** OSPF (within ISPs), IS-IS (large ISPs, data centers)

### Comparison

| Feature | Distance Vector | Link State |
|---------|---------------|-----------|
| Information shared | Routing table (distances) | Link state (direct neighbors) |
| Algorithm basis | Bellman-Ford | Dijkstra |
| Topology knowledge | Partial (local view) | Full (global view) |
| Convergence speed | Slow | Fast |
| Bandwidth use | Low (periodic updates) | High (initial flooding) |
| Loop risk | Count-to-infinity | None (full topology) |
| Examples | RIP, BGP | OSPF, IS-IS |

---

## 6. Internet Routing Architecture

```
 [Your Device]
      ↓ OSPF/IS-IS (inside your ISP's AS)
 [Your ISP — AS 1234]
      ↓ BGP (between ASes)
 [National ISP — AS 9829]
      ↓ BGP
 [Global ISP — AS 6453]
      ↓ BGP
 [Google — AS 15169]
```

- **Within an AS (Autonomous System):** OSPF or IS-IS (link-state)
- **Between ASes:** BGP (path-vector — shares AS path to prevent loops)

---

## 📌 Key Takeaways

1. **Dijkstra breaks** with negative edges — greedy finalization is incorrect
2. **Bellman-Ford** relaxes all edges V−1 times — handles negative edges, detects negative cycles
3. **MST** = minimum weight tree connecting all nodes; V−1 edges, no cycles
4. **STP** uses MST to prevent L2 loops in Ethernet switch networks
5. **Distance Vector** (Bellman-Ford) → RIP — simple, slow convergence
6. **Link State** (Dijkstra) → OSPF — complete topology, fast convergence
7. **BGP** = path-vector (extension of distance vector) between internet's ASes

---

## 🧠 Quick Self-Check Questions

1. Trace Bellman-Ford on: A→B(3), B→C(-5), A→C(2) from A. What are final distances?
2. Does a negative edge weight necessarily mean a negative cycle? Explain.
3. How many edges does an MST of a 10-node graph have?
4. Why does STP need to block links? What would happen without it?
5. What's the maximum hop count in RIP? Why was this limit chosen?
6. In link-state routing, what does a router flood when a link changes?
7. Why is BGP called a "path-vector" protocol rather than distance-vector?

---

*Lecture 6 of 13 — Computer Networks, Term 5, SST*

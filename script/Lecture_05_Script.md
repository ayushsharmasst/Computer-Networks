# Lecture 5: Graph Algorithms for Networking
## Instructor Script — SST Computer Networks (Term 5)

---

### 🎯 Learning Objectives
By the end of this session, students will:
- Model a computer network as a graph
- Implement BFS and DFS in a network context
- Apply Dijkstra's algorithm to find shortest/least-cost paths
- Understand why routers use these algorithms

**Duration:** ~90 minutes  
**Teaching Style:** Code + whiteboard; algorithm-first, network application second

---

## 📦 OPENING HOOK (5 minutes)

**[INSTRUCTOR: Draw this on board — 5 nodes representing cities, connected by edges with numbers (bandwidth/cost)]**

```
        Mumbai
       /  5   \  3
      /         \
  Delhi --- 2 --- Hyderabad
      \         /
       \ 8    / 2
        \   /
       Chennai --- 4 --- Bangalore
```

> "A packet leaves Mumbai. It needs to reach Bangalore. There are multiple paths:
> - Mumbai → Hyderabad → Bangalore: cost 3 + 2 = 5
> - Mumbai → Delhi → Hyderabad → Bangalore: cost 5 + 2 + 2 = 9
> - Mumbai → Delhi → Chennai → Bangalore: cost 5 + 8 + 4 = 17
>
> Which path should the packet take? How does the router know? That's a shortest-path problem — and routers solve it using graph algorithms."

---

## SECTION 1: Networks as Graphs (10 minutes)

### Graph Fundamentals

> "A **graph** is a collection of **vertices** (also called nodes) connected by **edges**."

**In networking:**
- **Vertex** = Router / network node
- **Edge** = Network link between two routers
- **Edge weight** = Cost of traversing that link (latency, bandwidth, hop count)

### Graph Types Relevant to Networking

**Undirected Graph:** Link works both ways (most physical links)
```
Router A ——— Router B   (A can reach B, B can reach A)
```

**Directed Graph:** Traffic can go one-way (asymmetric paths, satellite links)
```
Router A ——→ Router B   (A can reach B, but B might route differently back)
```

**Weighted Graph:** Each edge has a cost
```
Router A ——3——→ Router B ——1——→ Router C
              ↗ 10
        Router D
```

### Graph Representation in Code

**Adjacency List** (preferred for sparse networks):
```python
graph = {
    'Mumbai':    [('Delhi', 5), ('Hyderabad', 3)],
    'Delhi':     [('Mumbai', 5), ('Hyderabad', 2), ('Chennai', 8)],
    'Hyderabad': [('Mumbai', 3), ('Delhi', 2), ('Bangalore', 2)],
    'Chennai':   [('Delhi', 8), ('Bangalore', 4)],
    'Bangalore': [('Hyderabad', 2), ('Chennai', 4)]
}
```

**Adjacency Matrix** (for dense networks or quick lookup):
```python
#           M   D   H   C   B
#  Mumbai  [0,  5,  3,  ∞,  ∞]
#  Delhi   [5,  0,  2,  8,  ∞]
#  Hyderabad[3, 2,  0,  ∞,  2]
#  Chennai [∞,  8,  ∞,  0,  4]
#  Bangalore[∞, ∞,  2,  4,  0]
```

**[INSTRUCTOR: Ask — "Which representation would you choose for the internet, which has millions of routers but each router connects to only a few others?"  
Answer: Adjacency list — sparse graph, saves memory]**

---

## SECTION 2: BFS — Breadth-First Search (15 minutes)

### What BFS Does

> "BFS explores a graph layer by layer — first all nodes 1 hop away, then 2 hops, then 3 hops. It finds the **shortest path in terms of hops** (not cost)."

**Network application:** RIP (Routing Information Protocol) uses a hop-count metric. BFS is conceptually how you find minimum-hop paths.

### BFS Algorithm

```
BFS(graph, start):
    queue = [start]
    visited = {start}
    while queue is not empty:
        node = queue.pop_front()
        for each neighbor of node:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
```

### Live Trace — BFS from Mumbai

```
Network:
Mumbai → {Delhi(5), Hyderabad(3)}
Delhi → {Mumbai(5), Hyderabad(2), Chennai(8)}
Hyderabad → {Mumbai(3), Delhi(2), Bangalore(2)}
Chennai → {Delhi(8), Bangalore(4)}
Bangalore → {Hyderabad(2), Chennai(4)}

BFS from Mumbai:
Queue: [Mumbai]
Visit: Mumbai → add Delhi, Hyderabad → Queue: [Delhi, Hyderabad]
Visit: Delhi → add Chennai (Hyderabad already visited) → Queue: [Hyderabad, Chennai]
Visit: Hyderabad → add Bangalore → Queue: [Chennai, Bangalore]
Visit: Chennai → (Bangalore already queued) → Queue: [Bangalore]
Visit: Bangalore → done!

Order: Mumbai → Delhi → Hyderabad → Chennai → Bangalore
Hops to Bangalore = 2 (via Hyderabad)
```

**[INSTRUCTOR: Do this trace on the board with the graph drawn, showing the layers]**

### BFS with Path Tracking

```python
from collections import deque

def bfs_path(graph, start, end):
    queue = deque([(start, [start])])
    visited = {start}
    
    while queue:
        node, path = queue.popleft()
        if node == end:
            return path
        for neighbor, weight in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append((neighbor, path + [neighbor]))
    return None

path = bfs_path(graph, 'Mumbai', 'Bangalore')
print(path)  # ['Mumbai', 'Hyderabad', 'Bangalore'] — 2 hops
```

**[INSTRUCTOR: Walk through this code. Emphasize the use of deque (O(1) pop from front) vs list (O(n) for pop(0))]**

### When Does BFS Give Wrong Answer?

> "BFS finds the MINIMUM HOP path, not the minimum COST path. In our graph:
> - BFS finds: Mumbai → Hyderabad → Bangalore (2 hops, cost 5)
> - But Mumbai → Delhi → Hyderabad → Bangalore is 3 hops, cost 9 — worse
> 
> Here BFS happens to find the cheapest too. But what if the cheap link had 3 hops?"

**Create a counterexample:**
```
A ——1——→ B ——1——→ C ——1——→ D
 \                          ↗
  \———————————100——————————/

BFS: A → B → C → D (3 hops)
But cost = 1+1+1 = 3 — correct here

Now: A → D directly (1 hop, cost 100)
BFS picks: A → D (1 hop!)
But minimum COST is A → B → C → D (cost 3, 3 hops)
```

> "For cost-based routing (like OSPF), we need Dijkstra."

---

## SECTION 3: DFS — Depth-First Search (10 minutes)

### What DFS Does

> "DFS goes as deep as possible down one path before backtracking. It finds A path to the destination, but not necessarily the shortest."

**Network use cases:**
- Network topology exploration
- Cycle detection in routing (detecting routing loops)
- Spanning tree construction

### DFS Algorithm

```
DFS(graph, start, visited=set()):
    visited.add(start)
    for each neighbor of start:
        if neighbor not in visited:
            DFS(graph, neighbor, visited)
```

### DFS Trace from Mumbai

```
DFS from Mumbai (alphabetical order for ties):
Start → Mumbai (visited)
  → Delhi (visited)
    → Hyderabad (visited)
      → Bangalore (visited)
        → Chennai (visited)

Order: Mumbai → Delhi → Hyderabad → Bangalore → Chennai
```

**Compare to BFS order:** Mumbai → Delhi → Hyderabad → Chennai → Bangalore

**[INSTRUCTOR: Emphasize — DFS goes deep first, BFS explores wide first. Both visit all nodes, different order.]**

### DFS for Cycle Detection

```python
def has_cycle(graph, node, visited, rec_stack):
    visited.add(node)
    rec_stack.add(node)
    
    for neighbor, _ in graph[node]:
        if neighbor not in visited:
            if has_cycle(graph, neighbor, visited, rec_stack):
                return True
        elif neighbor in rec_stack:
            return True   # Back edge = cycle found!
    
    rec_stack.remove(node)
    return False
```

> "Routers use cycle detection to identify routing loops — situations where packets bounce between routers forever. We'll see this in Lecture 8 with distance-vector routing."

---

## SECTION 4: Dijkstra's Algorithm (20 minutes)

### Why Dijkstra?

> "Dijkstra's algorithm finds the **minimum cost path** from a source to all other nodes. This is what OSPF (Open Shortest Path First) is based on."

### Core Idea — Greedy + Priority Queue

> "At each step, expand the cheapest unvisited node. Maintain a running cost to reach each node."

### The Algorithm

```
Dijkstra(graph, source):
    dist = {node: ∞ for all nodes}
    dist[source] = 0
    pq = [(0, source)]    ← priority queue: (cost, node)
    visited = set()
    
    while pq is not empty:
        cost, node = pq.pop_min()
        if node in visited: continue
        visited.add(node)
        
        for neighbor, edge_weight in graph[node]:
            new_cost = cost + edge_weight
            if new_cost < dist[neighbor]:
                dist[neighbor] = new_cost
                pq.push((new_cost, neighbor))
    
    return dist
```

### Live Trace — Dijkstra from Mumbai

```
Graph:
Mumbai-Delhi:5, Mumbai-Hyderabad:3, Delhi-Hyderabad:2, 
Delhi-Chennai:8, Hyderabad-Bangalore:2, Chennai-Bangalore:4

Initial: dist = {Mumbai:0, Delhi:∞, Hyderabad:∞, Chennai:∞, Bangalore:∞}
PQ: [(0, Mumbai)]

Step 1: Pop (0, Mumbai)
  → Delhi: 0+5=5 < ∞ → dist[Delhi]=5, push (5, Delhi)
  → Hyderabad: 0+3=3 < ∞ → dist[Hyderabad]=3, push (3, Hyderabad)
dist: {Mumbai:0, Delhi:5, Hyderabad:3, Chennai:∞, Bangalore:∞}

Step 2: Pop (3, Hyderabad)
  → Mumbai: 3+3=6 > 0 → skip
  → Delhi: 3+2=5 = 5 → equal, no update
  → Bangalore: 3+2=5 < ∞ → dist[Bangalore]=5, push (5, Bangalore)
dist: {Mumbai:0, Delhi:5, Hyderabad:3, Chennai:∞, Bangalore:5}

Step 3: Pop (5, Delhi)
  → Mumbai: 5+5=10 > 0 → skip
  → Hyderabad: 5+2=7 > 3 → skip
  → Chennai: 5+8=13 < ∞ → dist[Chennai]=13, push (13, Chennai)
dist: {Mumbai:0, Delhi:5, Hyderabad:3, Chennai:13, Bangalore:5}

Step 4: Pop (5, Bangalore)
  → Hyderabad: 5+2=7 > 3 → skip
  → Chennai: 5+4=9 < 13 → dist[Chennai]=9, push (9, Chennai)

Step 5: Pop (9, Chennai)
  → All neighbors already finalized

FINAL: dist = {Mumbai:0, Delhi:5, Hyderabad:3, Chennai:9, Bangalore:5}
```

**[INSTRUCTOR: Do this trace on board. Color-code the priority queue state at each step.]**

### Python Implementation

```python
import heapq

def dijkstra(graph, source):
    dist = {node: float('inf') for node in graph}
    dist[source] = 0
    prev = {node: None for node in graph}
    pq = [(0, source)]
    
    while pq:
        cost, node = heapq.heappop(pq)
        
        if cost > dist[node]:   # Already found a better path
            continue
            
        for neighbor, weight in graph[node]:
            new_cost = cost + weight
            if new_cost < dist[neighbor]:
                dist[neighbor] = new_cost
                prev[neighbor] = node
                heapq.heappush(pq, (new_cost, neighbor))
    
    return dist, prev

def reconstruct_path(prev, source, target):
    path = []
    node = target
    while node is not None:
        path.append(node)
        node = prev[node]
    return list(reversed(path))

graph = {
    'Mumbai':    [('Delhi', 5), ('Hyderabad', 3)],
    'Delhi':     [('Mumbai', 5), ('Hyderabad', 2), ('Chennai', 8)],
    'Hyderabad': [('Mumbai', 3), ('Delhi', 2), ('Bangalore', 2)],
    'Chennai':   [('Delhi', 8), ('Bangalore', 4)],
    'Bangalore': [('Hyderabad', 2), ('Chennai', 4)]
}

dist, prev = dijkstra(graph, 'Mumbai')
print("Shortest distances from Mumbai:", dist)
path = reconstruct_path(prev, 'Mumbai', 'Bangalore')
print("Path to Bangalore:", path)
# Output: ['Mumbai', 'Hyderabad', 'Bangalore']
```

**[INSTRUCTOR: Run this in class or walk through line by line. Emphasize heapq module.]**

### Time Complexity

| Implementation | Time |
|---------------|------|
| Simple (array for priority) | O(V²) |
| Binary heap (heapq) | O((V + E) log V) |
| Fibonacci heap | O(E + V log V) |

> "Real routing protocols use optimized versions. OSPF uses Dijkstra with E log V complexity."

---

## SECTION 5: Graph Algorithms → Routing Protocols (5 minutes)

> "Here's the connection to networking:"

| Algorithm | Routing Protocol | Layer |
|-----------|----------------|-------|
| BFS / hop count | RIP (Routing Information Protocol) | L3 |
| Dijkstra | OSPF (Open Shortest Path First) | L3 |
| Bellman-Ford | RIP, BGP path vector | L3 |
| Minimum Spanning Tree | Some redundancy protocols | L2 (STP) |

**Preview:**
> "In Lecture 8, we'll see Bellman-Ford (which works even with negative weights but is slower) used in distance-vector routing. OSPF is link-state routing — every router runs Dijkstra on a complete map of the network."

---

## SUMMARY (5 minutes)

```
✅ Networks = Weighted graphs (routers = nodes, links = edges)
✅ Adjacency list is preferred for sparse networks
✅ BFS: finds minimum-HOP path, uses a queue, O(V+E)
✅ DFS: explores deep first, uses recursion/stack, O(V+E)
    - used for cycle detection, topology exploration
✅ Dijkstra: finds minimum-COST path, uses priority queue
    - Works on non-negative weights only
    - O((V+E) log V) with binary heap
✅ Real protocols: BFS → RIP, Dijkstra → OSPF
```

---

## 📝 Coding Exercise (Homework)

1. Implement BFS that returns the shortest hop-count path between two routers.
2. Modify Dijkstra to also return the actual path (not just the distance).
3. Add a router `Pune` connected to `Mumbai (2)` and `Hyderabad (4)`. Re-run Dijkstra from Mumbai to Bangalore — does the path change?

---

*End of Lecture 5 Script*

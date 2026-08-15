# Lecture 5: Graph Algorithms for Networking
## Student Notes — SST Computer Networks (Term 5)

---

## 🎯 Learning Objectives
- Model a network as a weighted graph
- Implement and trace BFS for hop-count shortest path
- Implement and trace DFS for topology exploration and cycle detection
- Apply Dijkstra's algorithm for least-cost path routing

---

## 1. Networks as Graphs

A **graph** G = (V, E) where:
- **V (Vertices)** = Routers / Network nodes
- **E (Edges)** = Network links between nodes
- **Edge weight** = Link cost (latency, bandwidth, hop cost)

### Example Network Graph

```
        Mumbai
       /  5   \  3
      /         \
  Delhi --- 2 --- Hyderabad
      \               \
       \ 8              \ 2
        \                \
       Chennai --- 4 --- Bangalore
```

### Graph Representations

**Adjacency List** (used when graph is sparse — few connections per node):

```python
graph = {
    'Mumbai':    [('Delhi', 5), ('Hyderabad', 3)],
    'Delhi':     [('Mumbai', 5), ('Hyderabad', 2), ('Chennai', 8)],
    'Hyderabad': [('Mumbai', 3), ('Delhi', 2), ('Bangalore', 2)],
    'Chennai':   [('Delhi', 8), ('Bangalore', 4)],
    'Bangalore': [('Hyderabad', 2), ('Chennai', 4)]
}
```

**Adjacency Matrix** (for dense graphs or fast edge lookup):

```
         M   D   H   C   B
Mumbai  [0,  5,  3,  ∞,  ∞]
Delhi   [5,  0,  2,  8,  ∞]
Hyderabad[3, 2,  0,  ∞,  2]
Chennai [∞,  8,  ∞,  0,  4]
Bangalore[∞, ∞,  2,  4,  0]
```

> **Preferred for real networks:** Adjacency list — the internet is sparse (each router connects to a few others).

---

## 2. BFS — Breadth-First Search

### What BFS Finds
**Minimum hop-count path** from source to all reachable nodes.

### Key Properties
- Uses a **queue** (FIFO)
- Explores nodes layer by layer (1 hop, then 2 hops, then 3 hops...)
- Guarantees shortest path in terms of **number of edges**

### Algorithm

```python
from collections import deque

def bfs(graph, start):
    visited = {start}
    queue = deque([start])
    order = []
    
    while queue:
        node = queue.popleft()
        order.append(node)
        for neighbor, weight in graph[node]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return order
```

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
```

### BFS Trace — Mumbai to Bangalore

```
Initial:  Queue = [Mumbai], Visited = {Mumbai}

Step 1: Visit Mumbai → add Delhi, Hyderabad
        Queue = [Delhi, Hyderabad]

Step 2: Visit Delhi → add Chennai (Hyderabad already visited)
        Queue = [Hyderabad, Chennai]

Step 3: Visit Hyderabad → add Bangalore
        Queue = [Chennai, Bangalore]

Step 4: Visit Chennai → (Bangalore already queued)
        Queue = [Bangalore]

Step 5: Visit Bangalore → FOUND!
        Path: Mumbai → Hyderabad → Bangalore (2 hops)
```

### Limitation of BFS

BFS minimizes **hops**, not **cost**. If a 1-hop path costs 100 and a 3-hop path costs 3, BFS picks the 1-hop path (more expensive). For cost-based routing, use **Dijkstra**.

---

## 3. DFS — Depth-First Search

### What DFS Does
Explores as **deep as possible** along one branch before backtracking. Finds a path (not necessarily shortest).

### Key Properties
- Uses a **stack** (recursion or explicit stack)
- O(V + E) time complexity
- Used for: topology discovery, cycle detection, spanning tree construction

### Algorithm (Recursive)

```python
def dfs(graph, node, visited=None):
    if visited is None:
        visited = set()
    visited.add(node)
    print(node)
    for neighbor, weight in graph[node]:
        if neighbor not in visited:
            dfs(graph, neighbor, visited)
```

### Algorithm (Iterative)

```python
def dfs_iterative(graph, start):
    stack = [start]
    visited = set()
    order = []
    
    while stack:
        node = stack.pop()
        if node not in visited:
            visited.add(node)
            order.append(node)
            for neighbor, weight in graph[node]:
                if neighbor not in visited:
                    stack.append(neighbor)
    return order
```

### DFS Trace — From Mumbai

```
Stack: [Mumbai]
Visit Mumbai → push Delhi, Hyderabad → Stack: [Delhi, Hyderabad]
Pop Hyderabad → visit → push Bangalore, Delhi(skip) → Stack: [Delhi, Bangalore]
Pop Bangalore → visit → push Chennai → Stack: [Delhi, Chennai]
Pop Chennai → visit → push Delhi(skip) → Stack: [Delhi]
Pop Delhi → visit → all neighbors visited
Done!
Order: Mumbai → Hyderabad → Bangalore → Chennai → Delhi
```

### Cycle Detection with DFS

```python
def has_cycle(graph, node, visited, rec_stack):
    visited.add(node)
    rec_stack.add(node)
    
    for neighbor, _ in graph[node]:
        if neighbor not in visited:
            if has_cycle(graph, neighbor, visited, rec_stack):
                return True
        elif neighbor in rec_stack:
            return True  # Back edge = cycle
    
    rec_stack.discard(node)
    return False
```

**Network use:** Detect routing loops — packets bouncing between routers forever.

---

## 4. Dijkstra's Algorithm

### What Dijkstra Finds
**Minimum cost path** from a source to all other nodes.

### Key Properties
- Uses a **min-priority queue** (min-heap)
- Greedy: always expands the cheapest unfinished node
- Works on **non-negative edge weights** only
- Time complexity: O((V + E) log V) with binary heap

### Algorithm

```python
import heapq

def dijkstra(graph, source):
    # Initialize all distances to infinity
    dist = {node: float('inf') for node in graph}
    dist[source] = 0
    prev = {node: None for node in graph}
    
    pq = [(0, source)]  # (cost, node)
    
    while pq:
        cost, node = heapq.heappop(pq)
        
        # Skip if we've already found a better path
        if cost > dist[node]:
            continue
        
        for neighbor, weight in graph[node]:
            new_cost = cost + weight
            if new_cost < dist[neighbor]:
                dist[neighbor] = new_cost
                prev[neighbor] = node
                heapq.heappush(pq, (new_cost, neighbor))
    
    return dist, prev

def reconstruct_path(prev, target):
    path = []
    node = target
    while node is not None:
        path.append(node)
        node = prev[node]
    return list(reversed(path))
```

### Dijkstra Trace — Mumbai to All Nodes

| Step | Expanded | dist[Mumbai] | dist[Delhi] | dist[Hyderabad] | dist[Chennai] | dist[Bangalore] |
|------|----------|-------------|------------|----------------|--------------|----------------|
| Init | — | 0 | ∞ | ∞ | ∞ | ∞ |
| 1 | Mumbai | 0 | 5 | 3 | ∞ | ∞ |
| 2 | Hyderabad | 0 | 5 | 3 | ∞ | 5 |
| 3 | Delhi | 0 | 5 | 3 | 13 | 5 |
| 4 | Bangalore | 0 | 5 | 3 | 9 | 5 |
| 5 | Chennai | 0 | 5 | 3 | 9 | 5 |

**Final shortest paths from Mumbai:**
```
Mumbai → Delhi:     cost 5   (direct)
Mumbai → Hyderabad: cost 3   (direct)
Mumbai → Bangalore: cost 5   (Mumbai → Hyderabad → Bangalore)
Mumbai → Chennai:   cost 9   (Mumbai → Hyderabad → Bangalore → Chennai)
```

---

## 5. BFS vs DFS vs Dijkstra Comparison

| Feature | BFS | DFS | Dijkstra |
|---------|-----|-----|---------|
| Data Structure | Queue (FIFO) | Stack / Recursion | Min-heap |
| What it finds | Shortest hops | A path (any) | Minimum cost |
| Time complexity | O(V + E) | O(V + E) | O((V+E) log V) |
| Works with weights? | No (ignores) | No (ignores) | Yes |
| Negative weights? | N/A | N/A | No |
| Network use | RIP (hop count) | Topology/loop detection | OSPF (link cost) |

---

## 6. Graph Algorithms → Network Protocols

| Algorithm | Real Protocol | Description |
|-----------|-------------|-------------|
| BFS (hop count) | **RIP** | Routing Information Protocol — metric is hop count |
| Dijkstra | **OSPF** | Open Shortest Path First — each router runs Dijkstra |
| Bellman-Ford | **RIP, BGP** | Handles distributed updates; next lecture |
| MST algorithms | **STP** | Spanning Tree Protocol — prevents L2 loops |

---

## 📌 Key Takeaways

1. **Networks are graphs**: routers = nodes, links = edges (weighted)
2. **BFS** = minimum hops; uses queue; O(V+E)
3. **DFS** = explore deep first; used for cycle detection; O(V+E)
4. **Dijkstra** = minimum cost; uses min-heap; O((V+E) log V); requires non-negative weights
5. BFS ↔ RIP; Dijkstra ↔ OSPF — real protocols are based on these algorithms

---

## 🧠 Quick Self-Check Questions

1. What data structure distinguishes BFS from DFS?
2. Why can't Dijkstra handle negative edge weights?
3. In the Mumbai network, what's the minimum-cost path from Delhi to Bangalore?
4. Which algorithm would you use to find ALL paths between two routers (for redundancy analysis)?
5. Why is adjacency list preferred over adjacency matrix for internet-scale routing?
6. In Dijkstra, why do we check `if cost > dist[node]: continue`?
7. Trace BFS from Chennai on the example graph. What's the order of visits?

---

*Lecture 5 of 13 — Computer Networks, Term 5, SST*

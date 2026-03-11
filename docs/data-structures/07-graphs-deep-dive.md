# Graphs Deep Dive

## Table of Contents
- [Graph Fundamentals](#graph-fundamentals)
- [Graph Representations](#graph-representations)
- [Graph Traversals](#graph-traversals)
- [Shortest Path Algorithms](#shortest-path-algorithms)
- [Minimum Spanning Tree](#minimum-spanning-tree)
- [Topological Sort](#topological-sort)
- [Cycle Detection](#cycle-detection)
- [Connected Components](#connected-components)
- [Bipartite Check](#bipartite-check)
- [Performance Characteristics](#performance-characteristics)
- [Common Patterns](#common-patterns)
- [Interview Problems](#interview-problems)

---

## Graph Fundamentals

### Terminology
```
Undirected Graph:          Directed Graph (Digraph):
  1 --- 2                    1 → 2
  |     |                    ↑   ↓
  3 --- 4                    3 ← 4

- Vertex/Node: 1, 2, 3, 4
- Edge: connection between vertices
- Degree: number of edges at a vertex
- Path: sequence of vertices connected by edges
- Cycle: path that starts and ends at same vertex
- Connected: path exists between every pair of vertices
- Weighted: edges have associated costs/weights
```

### Graph Types
```
Directed Acyclic Graph (DAG):   Weighted Graph:
  A → B → D                      A --5-- B
  ↓   ↓                          |       |
  C → E                          3       2
                                 |       |
No cycles                        C --4-- D
Used for: dependencies,
topological sort
```

---

## Graph Representations

### Adjacency List
```go
// Most common — efficient for sparse graphs
type Graph struct {
    vertices int
    adjList  map[int][]int
}

func NewGraph(v int) *Graph {
    return &Graph{
        vertices: v,
        adjList:  make(map[int][]int),
    }
}

func (g *Graph) AddEdge(u, v int) {
    g.adjList[u] = append(g.adjList[u], v)
    g.adjList[v] = append(g.adjList[v], u)  // Remove for directed
}

// Space: O(V + E)
// Add edge: O(1)
// Check edge: O(degree)
// Get neighbors: O(degree)
```

### Adjacency Matrix
```go
// Efficient for dense graphs or frequent edge lookups
type MatrixGraph struct {
    vertices int
    matrix   [][]int
}

func NewMatrixGraph(v int) *MatrixGraph {
    matrix := make([][]int, v)
    for i := range matrix {
        matrix[i] = make([]int, v)
    }
    return &MatrixGraph{vertices: v, matrix: matrix}
}

func (g *MatrixGraph) AddEdge(u, v, weight int) {
    g.matrix[u][v] = weight
    g.matrix[v][u] = weight  // Remove for directed
}

func (g *MatrixGraph) HasEdge(u, v int) bool {
    return g.matrix[u][v] != 0
}

// Space: O(V²)
// Add edge: O(1)
// Check edge: O(1)
// Get neighbors: O(V)
```

### Edge List
```go
type Edge struct {
    From, To, Weight int
}

type EdgeGraph struct {
    vertices int
    edges    []Edge
}

func (g *EdgeGraph) AddEdge(from, to, weight int) {
    g.edges = append(g.edges, Edge{from, to, weight})
}

// Space: O(E)
// Used for: Kruskal's MST, Bellman-Ford
```

### Comparison
```
              Adjacency List   Adjacency Matrix   Edge List
Space:        O(V + E)         O(V²)              O(E)
Add Edge:     O(1)             O(1)               O(1)
Check Edge:   O(degree)        O(1)               O(E)
Neighbors:    O(degree)        O(V)               O(E)
Best for:     Sparse graphs    Dense graphs       MST algorithms
```

---

## Graph Traversals

### Depth-First Search (DFS)
```go
// Recursive DFS
func (g *Graph) DFS(start int) []int {
    visited := make(map[int]bool)
    result := []int{}

    var dfs func(v int)
    dfs = func(v int) {
        visited[v] = true
        result = append(result, v)
        for _, neighbor := range g.adjList[v] {
            if !visited[neighbor] {
                dfs(neighbor)
            }
        }
    }

    dfs(start)
    return result
}

// Iterative DFS
func (g *Graph) DFSIterative(start int) []int {
    visited := make(map[int]bool)
    result := []int{}
    stack := []int{start}

    for len(stack) > 0 {
        v := stack[len(stack)-1]
        stack = stack[:len(stack)-1]

        if visited[v] {
            continue
        }
        visited[v] = true
        result = append(result, v)

        for _, neighbor := range g.adjList[v] {
            if !visited[neighbor] {
                stack = append(stack, neighbor)
            }
        }
    }
    return result
}

// Time: O(V + E), Space: O(V)
```

### Breadth-First Search (BFS)
```go
func (g *Graph) BFS(start int) []int {
    visited := make(map[int]bool)
    result := []int{}
    queue := []int{start}
    visited[start] = true

    for len(queue) > 0 {
        v := queue[0]
        queue = queue[1:]
        result = append(result, v)

        for _, neighbor := range g.adjList[v] {
            if !visited[neighbor] {
                visited[neighbor] = true
                queue = append(queue, neighbor)
            }
        }
    }
    return result
}

// Time: O(V + E), Space: O(V)
// BFS gives shortest path in unweighted graphs
```

### DFS vs BFS
```
DFS:                        BFS:
- Uses stack (recursion)    - Uses queue
- Goes deep first           - Goes wide first
- Good for: cycle detection,- Good for: shortest path,
  topological sort,           level-order, connected
  path finding               components
- Space: O(h) where h=height- Space: O(w) where w=width
```

---

## Shortest Path Algorithms

### Dijkstra's Algorithm (Non-negative weights)
```go
import "container/heap"

type dijkstraItem struct {
    node, dist int
}
type dijkstraHeap []dijkstraItem

func (h dijkstraHeap) Len() int            { return len(h) }
func (h dijkstraHeap) Less(i, j int) bool  { return h[i].dist < h[j].dist }
func (h dijkstraHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *dijkstraHeap) Push(x interface{}) { *h = append(*h, x.(dijkstraItem)) }
func (h *dijkstraHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func dijkstra(graph map[int][][2]int, start, n int) []int {
    dist := make([]int, n)
    for i := range dist {
        dist[i] = math.MaxInt32
    }
    dist[start] = 0

    h := &dijkstraHeap{{start, 0}}
    heap.Init(h)

    for h.Len() > 0 {
        item := heap.Pop(h).(dijkstraItem)
        u, d := item.node, item.dist

        if d > dist[u] {
            continue  // Outdated entry
        }

        for _, edge := range graph[u] {
            v, w := edge[0], edge[1]
            if dist[u]+w < dist[v] {
                dist[v] = dist[u] + w
                heap.Push(h, dijkstraItem{v, dist[v]})
            }
        }
    }
    return dist
}

// Time: O((V + E) log V), Space: O(V)
```

### Bellman-Ford (Handles negative weights)
```go
func bellmanFord(edges []Edge, n, start int) ([]int, bool) {
    dist := make([]int, n)
    for i := range dist {
        dist[i] = math.MaxInt32
    }
    dist[start] = 0

    // Relax all edges V-1 times
    for i := 0; i < n-1; i++ {
        for _, e := range edges {
            if dist[e.From] != math.MaxInt32 &&
                dist[e.From]+e.Weight < dist[e.To] {
                dist[e.To] = dist[e.From] + e.Weight
            }
        }
    }

    // Check for negative cycles
    for _, e := range edges {
        if dist[e.From] != math.MaxInt32 &&
            dist[e.From]+e.Weight < dist[e.To] {
            return nil, true  // Negative cycle exists
        }
    }

    return dist, false
}

// Time: O(V * E), Space: O(V)
```

### Floyd-Warshall (All pairs shortest path)
```go
func floydWarshall(graph [][]int) [][]int {
    n := len(graph)
    dist := make([][]int, n)
    for i := range dist {
        dist[i] = make([]int, n)
        copy(dist[i], graph[i])
    }

    for k := 0; k < n; k++ {
        for i := 0; i < n; i++ {
            for j := 0; j < n; j++ {
                if dist[i][k] != math.MaxInt32 &&
                    dist[k][j] != math.MaxInt32 &&
                    dist[i][k]+dist[k][j] < dist[i][j] {
                    dist[i][j] = dist[i][k] + dist[k][j]
                }
            }
        }
    }
    return dist
}

// Time: O(V³), Space: O(V²)
// Use when: small graph, need all-pairs shortest paths
```

---

## Minimum Spanning Tree

### Kruskal's Algorithm
```go
func kruskal(n int, edges []Edge) []Edge {
    // Sort edges by weight
    sort.Slice(edges, func(i, j int) bool {
        return edges[i].Weight < edges[j].Weight
    })

    uf := NewUnionFind(n)
    mst := []Edge{}

    for _, e := range edges {
        if uf.Find(e.From) != uf.Find(e.To) {
            uf.Union(e.From, e.To)
            mst = append(mst, e)
            if len(mst) == n-1 {
                break
            }
        }
    }
    return mst
}

// Time: O(E log E), Space: O(V)
```

### Prim's Algorithm
```go
func prim(graph map[int][][2]int, n int) int {
    visited := make([]bool, n)
    h := &dijkstraHeap{{0, 0}}  // {node, weight}
    heap.Init(h)
    totalWeight := 0

    for h.Len() > 0 {
        item := heap.Pop(h).(dijkstraItem)
        u, w := item.node, item.dist

        if visited[u] {
            continue
        }
        visited[u] = true
        totalWeight += w

        for _, edge := range graph[u] {
            v, ew := edge[0], edge[1]
            if !visited[v] {
                heap.Push(h, dijkstraItem{v, ew})
            }
        }
    }
    return totalWeight
}

// Time: O((V + E) log V), Space: O(V)
```

---

## Topological Sort

### Kahn's Algorithm (BFS-based)
```go
func topoSortBFS(n int, edges [][2]int) ([]int, bool) {
    inDegree := make([]int, n)
    adj := make([][]int, n)

    for _, e := range edges {
        adj[e[0]] = append(adj[e[0]], e[1])
        inDegree[e[1]]++
    }

    queue := []int{}
    for i := 0; i < n; i++ {
        if inDegree[i] == 0 {
            queue = append(queue, i)
        }
    }

    order := []int{}
    for len(queue) > 0 {
        u := queue[0]
        queue = queue[1:]
        order = append(order, u)

        for _, v := range adj[u] {
            inDegree[v]--
            if inDegree[v] == 0 {
                queue = append(queue, v)
            }
        }
    }

    if len(order) != n {
        return nil, false  // Cycle detected
    }
    return order, true
}

// Time: O(V + E), Space: O(V)
```

### DFS-based Topological Sort
```go
func topoSortDFS(n int, adj [][]int) ([]int, bool) {
    // 0=unvisited, 1=in-progress, 2=done
    state := make([]int, n)
    order := []int{}
    hasCycle := false

    var dfs func(u int)
    dfs = func(u int) {
        if hasCycle {
            return
        }
        state[u] = 1  // In progress
        for _, v := range adj[u] {
            if state[v] == 1 {
                hasCycle = true
                return
            }
            if state[v] == 0 {
                dfs(v)
            }
        }
        state[u] = 2
        order = append([]int{u}, order...)  // Prepend
    }

    for i := 0; i < n; i++ {
        if state[i] == 0 {
            dfs(i)
        }
    }

    if hasCycle {
        return nil, false
    }
    return order, true
}
```

---

## Cycle Detection

### Undirected Graph (DFS)
```go
func hasCycleUndirected(n int, adj [][]int) bool {
    visited := make([]bool, n)

    var dfs func(u, parent int) bool
    dfs = func(u, parent int) bool {
        visited[u] = true
        for _, v := range adj[u] {
            if !visited[v] {
                if dfs(v, u) {
                    return true
                }
            } else if v != parent {
                return true  // Back edge = cycle
            }
        }
        return false
    }

    for i := 0; i < n; i++ {
        if !visited[i] {
            if dfs(i, -1) {
                return true
            }
        }
    }
    return false
}
```

### Directed Graph (DFS with colors)
```go
func hasCycleDirected(n int, adj [][]int) bool {
    // 0=white, 1=gray(in-stack), 2=black(done)
    color := make([]int, n)

    var dfs func(u int) bool
    dfs = func(u int) bool {
        color[u] = 1
        for _, v := range adj[u] {
            if color[v] == 1 {
                return true  // Back edge
            }
            if color[v] == 0 && dfs(v) {
                return true
            }
        }
        color[u] = 2
        return false
    }

    for i := 0; i < n; i++ {
        if color[i] == 0 && dfs(i) {
            return true
        }
    }
    return false
}
```

---

## Connected Components

### Undirected Graph
```go
func countComponents(n int, edges [][2]int) int {
    adj := make([][]int, n)
    for _, e := range edges {
        adj[e[0]] = append(adj[e[0]], e[1])
        adj[e[1]] = append(adj[e[1]], e[0])
    }

    visited := make([]bool, n)
    count := 0

    var dfs func(u int)
    dfs = func(u int) {
        visited[u] = true
        for _, v := range adj[u] {
            if !visited[v] {
                dfs(v)
            }
        }
    }

    for i := 0; i < n; i++ {
        if !visited[i] {
            dfs(i)
            count++
        }
    }
    return count
}
```

### Strongly Connected Components (Kosaraju's)
```go
func scc(n int, adj [][]int) [][]int {
    // Step 1: DFS on original graph, record finish order
    visited := make([]bool, n)
    order := []int{}

    var dfs1 func(u int)
    dfs1 = func(u int) {
        visited[u] = true
        for _, v := range adj[u] {
            if !visited[v] {
                dfs1(v)
            }
        }
        order = append(order, u)
    }

    for i := 0; i < n; i++ {
        if !visited[i] {
            dfs1(i)
        }
    }

    // Step 2: Build reverse graph
    radj := make([][]int, n)
    for u := 0; u < n; u++ {
        for _, v := range adj[u] {
            radj[v] = append(radj[v], u)
        }
    }

    // Step 3: DFS on reverse graph in reverse finish order
    visited = make([]bool, n)
    components := [][]int{}

    var dfs2 func(u int, comp *[]int)
    dfs2 = func(u int, comp *[]int) {
        visited[u] = true
        *comp = append(*comp, u)
        for _, v := range radj[u] {
            if !visited[v] {
                dfs2(v, comp)
            }
        }
    }

    for i := len(order) - 1; i >= 0; i-- {
        u := order[i]
        if !visited[u] {
            comp := []int{}
            dfs2(u, &comp)
            components = append(components, comp)
        }
    }
    return components
}
```

---

## Bipartite Check

```go
func isBipartite(n int, adj [][]int) bool {
    color := make([]int, n)
    for i := range color {
        color[i] = -1
    }

    for i := 0; i < n; i++ {
        if color[i] != -1 {
            continue
        }
        queue := []int{i}
        color[i] = 0

        for len(queue) > 0 {
            u := queue[0]
            queue = queue[1:]
            for _, v := range adj[u] {
                if color[v] == -1 {
                    color[v] = 1 - color[u]
                    queue = append(queue, v)
                } else if color[v] == color[u] {
                    return false  // Same color = not bipartite
                }
            }
        }
    }
    return true
}

// A graph is bipartite if it can be 2-colored
// Equivalent to: contains no odd-length cycles
```

---

## Performance Characteristics

### Algorithm Comparison
| Algorithm | Time | Space | Use Case |
|-----------|------|-------|----------|
| DFS | O(V+E) | O(V) | Cycle detection, paths |
| BFS | O(V+E) | O(V) | Shortest path (unweighted) |
| Dijkstra | O((V+E)logV) | O(V) | Shortest path (non-negative) |
| Bellman-Ford | O(VE) | O(V) | Shortest path (negative weights) |
| Floyd-Warshall | O(V³) | O(V²) | All-pairs shortest path |
| Kruskal | O(E logE) | O(V) | MST |
| Prim | O((V+E)logV) | O(V) | MST |
| Topo Sort | O(V+E) | O(V) | DAG ordering |

---

## Common Patterns

### Number of Islands
```go
func numIslands(grid [][]byte) int {
    if len(grid) == 0 {
        return 0
    }
    rows, cols := len(grid), len(grid[0])
    count := 0

    var dfs func(r, c int)
    dfs = func(r, c int) {
        if r < 0 || r >= rows || c < 0 || c >= cols || grid[r][c] == '0' {
            return
        }
        grid[r][c] = '0'  // Mark visited
        dfs(r+1, c)
        dfs(r-1, c)
        dfs(r, c+1)
        dfs(r, c-1)
    }

    for r := 0; r < rows; r++ {
        for c := 0; c < cols; c++ {
            if grid[r][c] == '1' {
                dfs(r, c)
                count++
            }
        }
    }
    return count
}
```

### Shortest Path in Grid (BFS)
```go
func shortestPath(grid [][]int, start, end [2]int) int {
    rows, cols := len(grid), len(grid[0])
    dirs := [][2]int{{0, 1}, {0, -1}, {1, 0}, {-1, 0}}
    visited := make([][]bool, rows)
    for i := range visited {
        visited[i] = make([]bool, cols)
    }

    type pos struct{ r, c, dist int }
    queue := []pos{{start[0], start[1], 0}}
    visited[start[0]][start[1]] = true

    for len(queue) > 0 {
        cur := queue[0]
        queue = queue[1:]

        if cur.r == end[0] && cur.c == end[1] {
            return cur.dist
        }

        for _, d := range dirs {
            nr, nc := cur.r+d[0], cur.c+d[1]
            if nr >= 0 && nr < rows && nc >= 0 && nc < cols &&
                !visited[nr][nc] && grid[nr][nc] == 0 {
                visited[nr][nc] = true
                queue = append(queue, pos{nr, nc, cur.dist + 1})
            }
        }
    }
    return -1
}
```

---

## Interview Problems

### 1. Course Schedule (Cycle Detection in DAG)
```go
func canFinish(numCourses int, prerequisites [][2]int) bool {
    adj := make([][]int, numCourses)
    for _, p := range prerequisites {
        adj[p[1]] = append(adj[p[1]], p[0])
    }
    // 0=unvisited, 1=visiting, 2=visited
    state := make([]int, numCourses)

    var dfs func(u int) bool
    dfs = func(u int) bool {
        if state[u] == 1 { return false }  // Cycle
        if state[u] == 2 { return true }   // Already processed
        state[u] = 1
        for _, v := range adj[u] {
            if !dfs(v) { return false }
        }
        state[u] = 2
        return true
    }

    for i := 0; i < numCourses; i++ {
        if !dfs(i) { return false }
    }
    return true
}
```

### 2. Clone Graph
```go
func cloneGraph(node *Node) *Node {
    if node == nil {
        return nil
    }
    visited := make(map[*Node]*Node)

    var dfs func(n *Node) *Node
    dfs = func(n *Node) *Node {
        if clone, ok := visited[n]; ok {
            return clone
        }
        clone := &Node{Val: n.Val}
        visited[n] = clone
        for _, neighbor := range n.Neighbors {
            clone.Neighbors = append(clone.Neighbors, dfs(neighbor))
        }
        return clone
    }
    return dfs(node)
}
```

### 3. Word Ladder (BFS)
```go
func ladderLength(beginWord, endWord string, wordList []string) int {
    wordSet := make(map[string]bool)
    for _, w := range wordList {
        wordSet[w] = true
    }
    if !wordSet[endWord] {
        return 0
    }

    queue := []string{beginWord}
    visited := map[string]bool{beginWord: true}
    steps := 1

    for len(queue) > 0 {
        size := len(queue)
        for i := 0; i < size; i++ {
            word := queue[i]
            for j := 0; j < len(word); j++ {
                for c := 'a'; c <= 'z'; c++ {
                    next := word[:j] + string(c) + word[j+1:]
                    if next == endWord {
                        return steps + 1
                    }
                    if wordSet[next] && !visited[next] {
                        visited[next] = true
                        queue = append(queue, next)
                    }
                }
            }
        }
        queue = queue[size:]
        steps++
    }
    return 0
}
```

---

## Summary

### Key Concepts
1. Adjacency list for sparse graphs, matrix for dense
2. DFS: stack-based, good for cycle detection and paths
3. BFS: queue-based, good for shortest path (unweighted)
4. Dijkstra: shortest path with non-negative weights
5. Topological sort: only for DAGs

### When to Use
| Problem | Algorithm |
|---------|-----------|
| Shortest path (unweighted) | BFS |
| Shortest path (weighted, non-negative) | Dijkstra |
| Shortest path (negative weights) | Bellman-Ford |
| All-pairs shortest path | Floyd-Warshall |
| Minimum spanning tree | Kruskal / Prim |
| Topological ordering | Kahn's / DFS |
| Cycle detection | DFS with colors |
| Connected components | DFS / Union-Find |

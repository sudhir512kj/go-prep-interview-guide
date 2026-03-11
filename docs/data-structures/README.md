# Data Structures Deep Dive

Comprehensive in-depth documentation for all data structures in Golang, covering internal implementations, memory layouts, performance analysis, common patterns, and interview problems.

## Table of Contents

1. [Arrays & Slices](#1-arrays--slices)
2. [Maps](#2-maps)
3. [Linked Lists](#3-linked-lists)
4. [Stacks & Queues](#4-stacks--queues)
5. [Trees](#5-trees)
6. [Heaps](#6-heaps)
7. [Graphs](#7-graphs)
8. [Tries](#8-tries)
9. [Union-Find, Segment Trees & Bloom Filters](#9-union-find-segment-trees--bloom-filters)

---

## 1. Arrays & Slices
**File**: [01-arrays-slices-deep-dive.md](./01-arrays-slices-deep-dive.md)

### Topics Covered
- Array fundamentals and memory layout
- Slice header structure (pointer, len, cap)
- Capacity growth strategy (doubles until 256, then 1.25x)
- Slice operations: append, insert, delete, filter
- Full slice expression `s[low:high:max]`
- Copy vs append performance
- Multi-dimensional arrays and slices
- Memory leaks from slice aliasing
- Common pitfalls: aliasing, loop variables, comparison

### Key Concepts
```go
// Slice header — 3 fields
type slice struct {
    array unsafe.Pointer  // Points to backing array
    len   int
    cap   int
}

// Growth: doubles until 256, then ~1.25x
s := make([]int, 0, 1000)  // Pre-allocate when size known
```

---

## 2. Maps
**File**: [02-maps-deep-dive.md](./02-maps-deep-dive.md)

### Topics Covered
- Hash table internals (hmap, bmap structures)
- Bucket structure (8 key-value pairs per bucket)
- Hash function and tophash optimization
- Collision resolution via overflow buckets
- Growth triggers (load factor > 6.5)
- Incremental rehashing / evacuation
- Iteration randomization
- Concurrent access and sync.Map
- Common pitfalls: nil map, reference semantics

### Key Concepts
```go
// Each bucket holds 8 key-value pairs
// Load factor ~6.5 before growth
// Iteration order is intentionally random
m := make(map[string]int, 1000)  // Pre-allocate
set := make(map[string]struct{}) // Zero-size value for sets
```

---

## 3. Linked Lists
**File**: [03-linked-lists-deep-dive.md](./03-linked-lists-deep-dive.md)

### Topics Covered
- Singly, doubly, and circular linked lists
- Memory layout vs arrays (non-contiguous)
- All CRUD operations with complexity
- Two-pointer techniques (fast/slow)
- Floyd's cycle detection algorithm
- Reversal: iterative, recursive, in groups, between positions
- Merge operations: two sorted lists, K sorted lists, merge sort
- Interview problems: palindrome, intersection, add two numbers

### Key Concepts
```go
// Fast/slow pointer — find middle, detect cycle
slow, fast := head, head
for fast != nil && fast.Next != nil {
    slow = slow.Next
    fast = fast.Next.Next
}
// slow is at middle when fast reaches end
```

---

## 4. Stacks & Queues
**File**: [04-stacks-queues-deep-dive.md](./04-stacks-queues-deep-dive.md)

### Topics Covered
- Stack (LIFO) and Queue (FIFO) fundamentals
- Slice-based and linked-list-based implementations
- Generic stack using Go generics
- Circular queue with wrap-around
- Deque (double-ended queue)
- Priority queue using container/heap
- Monotonic stack: next greater/smaller element
- Monotonic queue: sliding window maximum
- Interview problems: balanced parentheses, min stack, rain water

### Key Concepts
```go
// Monotonic stack — next greater element
for len(stack) > 0 && nums[stack[top]] < nums[i] {
    result[stack[top]] = nums[i]
    stack = stack[:top]
}

// Min stack — track minimum in O(1)
type MinStack struct {
    stack    []int
    minStack []int
}
```

---

## 5. Trees
**File**: [05-trees-deep-dive.md](./05-trees-deep-dive.md)

### Topics Covered
- Binary tree types: full, complete, perfect
- BST: insert, search, delete (3 cases)
- AVL tree: balance factor, 4 rotation cases (LL, RR, LR, RL)
- Red-Black tree properties and comparison with AVL
- All traversals: inorder, preorder, postorder (recursive + iterative)
- Level order and zigzag traversal
- BST operations: kth smallest, validate, LCA
- Tree properties: height, diameter, balance check
- Interview problems: max path sum, right side view, serialize/deserialize

### Key Concepts
```go
// BST inorder = sorted output
// AVL: balance factor = height(left) - height(right), must be -1,0,1
// Delete: replace with inorder successor (min of right subtree)

// LCA in binary tree
func lca(root, p, q *TreeNode) *TreeNode {
    if root == nil || root == p || root == q { return root }
    left := lca(root.Left, p, q)
    right := lca(root.Right, p, q)
    if left != nil && right != nil { return root }
    if left != nil { return left }
    return right
}
```

---

## 6. Heaps
**File**: [06-heaps-deep-dive.md](./06-heaps-deep-dive.md)

### Topics Covered
- Min-heap and max-heap properties
- Array representation (no pointers needed)
- Index relationships: parent=(i-1)/2, left=2i+1, right=2i+2
- Sift up (after insert) and sift down (after delete)
- Build heap in O(n) using bottom-up heapify
- Go's container/heap interface (5 methods)
- Heap sort: O(n log n), O(1) space
- Priority queue with custom structs
- Running median using two heaps
- Interview problems: K largest, merge K sorted, task scheduler

### Key Concepts
```go
// Go heap interface
type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{}  { /* remove last */ }

// Build heap: O(n) — not O(n log n)
heap.Init(h)
```

---

## 7. Graphs
**File**: [07-graphs-deep-dive.md](./07-graphs-deep-dive.md)

### Topics Covered
- Adjacency list vs adjacency matrix vs edge list
- DFS: recursive and iterative
- BFS: shortest path in unweighted graphs
- Dijkstra's algorithm (non-negative weights)
- Bellman-Ford (negative weights, cycle detection)
- Floyd-Warshall (all-pairs shortest path)
- Kruskal's and Prim's MST algorithms
- Topological sort: Kahn's (BFS) and DFS-based
- Cycle detection: undirected and directed graphs
- Connected components and Kosaraju's SCC
- Bipartite check
- Interview problems: number of islands, word ladder, course schedule

### Key Concepts
```go
// Dijkstra with min-heap
// Topological sort with in-degree (Kahn's)
// Cycle in directed graph: 3-color DFS (white/gray/black)
// SCC: DFS on original + DFS on reversed graph
```

---

## 8. Tries
**File**: [08-tries-deep-dive.md](./08-tries-deep-dive.md)

### Topics Covered
- Trie (prefix tree) fundamentals
- Array-based node [26]*TrieNode vs map-based
- Insert, search, startsWith, delete operations
- Get all words with prefix, count words with prefix
- Longest common prefix
- Compressed trie (Radix tree) — memory efficient
- Ternary Search Tree — space efficient
- Binary trie for XOR maximization
- IP routing with longest prefix match
- Interview problems: word search II, replace words, max XOR

### Key Concepts
```go
// O(m) operations independent of number of words
// Prefix search: O(p) vs O(n*m) for hash map
type TrieNode struct {
    children [26]*TrieNode
    isEnd    bool
}
// Map-based for large/unicode alphabets
type TrieNodeMap struct {
    children map[rune]*TrieNodeMap
    isEnd    bool
}
```

---

## 9. Union-Find, Segment Trees & Bloom Filters
**File**: [09-union-find-segment-tree-bloom-filter.md](./09-union-find-segment-tree-bloom-filter.md)

### Union-Find Topics
- Disjoint set union fundamentals
- Path compression: O(log n) → O(α(n))
- Union by rank and union by size
- Applications: Kruskal's MST, cycle detection, accounts merge

### Segment Tree Topics
- Range queries (sum, min, max) in O(log n)
- Point updates in O(log n)
- Build in O(n) using bottom-up construction
- Lazy propagation for range updates in O(log n)

### Bloom Filter Topics
- Probabilistic membership testing
- k hash functions on m-bit array
- Optimal m and k formulas
- False positive rate analysis
- Applications: web cache, spam filter, duplicate URL detection

### Key Concepts
```go
// Union-Find with both optimizations: O(α(n)) ≈ O(1)
func (uf *UnionFind) Find(x int) int {
    if uf.parent[x] != x {
        uf.parent[x] = uf.Find(uf.parent[x])  // Path compression
    }
    return uf.parent[x]
}

// Segment tree: 4*n array, O(log n) query and update
// Bloom filter: ~1.2KB for 1000 items at 1% false positive rate
```

---

## Complexity Quick Reference

### Time Complexity Summary
| Structure | Access | Search | Insert | Delete | Notes |
|-----------|--------|--------|--------|--------|-------|
| Array | O(1) | O(n) | O(n) | O(n) | Random access |
| Slice | O(1) | O(n) | O(1)* | O(n) | *amortized append |
| Map | O(1) | O(1) | O(1) | O(1) | Average case |
| Linked List | O(n) | O(n) | O(1)† | O(1)† | †at known position |
| Stack | O(n) | O(n) | O(1) | O(1) | LIFO |
| Queue | O(n) | O(n) | O(1) | O(1) | FIFO |
| BST | O(h) | O(h) | O(h) | O(h) | h=height |
| AVL/RB Tree | O(log n) | O(log n) | O(log n) | O(log n) | Balanced |
| Heap | O(1)‡ | O(n) | O(log n) | O(log n) | ‡peek only |
| Trie | O(m) | O(m) | O(m) | O(m) | m=key length |
| Union-Find | - | O(α(n)) | O(α(n)) | - | ≈O(1) |
| Segment Tree | O(log n) | O(log n) | O(log n) | O(log n) | Range ops |
| Bloom Filter | - | O(k) | O(k) | ❌ | Probabilistic |

### Space Complexity Summary
| Structure | Space | Notes |
|-----------|-------|-------|
| Array/Slice | O(n) | May have unused capacity |
| Map | O(n) | ~48 bytes overhead + buckets |
| Linked List | O(n) | Extra pointer per node |
| Tree | O(n) | O(h) stack for recursion |
| Heap | O(n) | Array-based, no pointers |
| Trie | O(ALPHABET * m * n) | Can be large |
| Segment Tree | O(n) | 4n array |
| Bloom Filter | O(m) | m << n |

---

## Choosing the Right Data Structure

```
Need fast lookup by key?
  → Map (O(1) average)

Need ordered data with fast insert/delete?
  → BST / AVL / Red-Black Tree (O(log n))

Need min/max repeatedly?
  → Heap (O(1) peek, O(log n) insert/delete)

Need prefix matching / autocomplete?
  → Trie (O(m) per operation)

Need dynamic connectivity?
  → Union-Find (O(α(n)) ≈ O(1))

Need range queries with updates?
  → Segment Tree (O(log n))

Need space-efficient membership test?
  → Bloom Filter (probabilistic)

Need LIFO ordering?
  → Stack

Need FIFO ordering?
  → Queue

Need sequential access with frequent head insert/delete?
  → Linked List

Need random access?
  → Array / Slice
```

---

## Learning Path

### Beginner
1. [Arrays & Slices](./01-arrays-slices-deep-dive.md) — foundation of Go
2. [Maps](./02-maps-deep-dive.md) — most used data structure
3. [Stacks & Queues](./04-stacks-queues-deep-dive.md) — fundamental patterns

### Intermediate
4. [Linked Lists](./03-linked-lists-deep-dive.md) — pointer manipulation
5. [Trees](./05-trees-deep-dive.md) — hierarchical data
6. [Heaps](./06-heaps-deep-dive.md) — priority-based processing

### Advanced
7. [Graphs](./07-graphs-deep-dive.md) — complex relationships
8. [Tries](./08-tries-deep-dive.md) — string operations
9. [Union-Find, Segment Trees & Bloom Filters](./09-union-find-segment-tree-bloom-filter.md) — specialized structures

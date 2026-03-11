# Union-Find, Segment Trees & Bloom Filters Deep Dive

## Table of Contents
- [Union-Find (Disjoint Set Union)](#union-find-disjoint-set-union)
  - [Fundamentals](#union-find-fundamentals)
  - [Internal Implementation](#union-find-internal-implementation)
  - [Optimizations](#optimizations)
  - [Applications](#union-find-applications)
- [Segment Tree](#segment-tree)
  - [Fundamentals](#segment-tree-fundamentals)
  - [Internal Implementation](#segment-tree-internal-implementation)
  - [Range Queries](#range-queries)
  - [Lazy Propagation](#lazy-propagation)
  - [Applications](#segment-tree-applications)
- [Bloom Filter](#bloom-filter)
  - [Fundamentals](#bloom-filter-fundamentals)
  - [Internal Implementation](#bloom-filter-internal-implementation)
  - [False Positive Rate](#false-positive-rate)
  - [Applications](#bloom-filter-applications)
- [Performance Comparison](#performance-comparison)

---

# Union-Find (Disjoint Set Union)

## Union-Find Fundamentals

### Definition
Union-Find (DSU) maintains a collection of disjoint sets and supports two operations:
- **Find**: Determine which set an element belongs to
- **Union**: Merge two sets together

```
Initial: {0} {1} {2} {3} {4}

Union(0,1): {0,1} {2} {3} {4}
Union(2,3): {0,1} {2,3} {4}
Union(1,3): {0,1,2,3} {4}

Find(0) == Find(2)? → Yes (same set)
Find(0) == Find(4)? → No (different sets)
```

### Use Cases
- Kruskal's MST algorithm
- Detecting cycles in undirected graphs
- Connected components
- Network connectivity
- Percolation problems

---

## Union-Find Internal Implementation

### Basic Implementation
```go
type UnionFind struct {
    parent []int
    rank   []int
    size   []int
    count  int  // Number of components
}

func NewUnionFind(n int) *UnionFind {
    parent := make([]int, n)
    rank := make([]int, n)
    size := make([]int, n)
    for i := range parent {
        parent[i] = i  // Each element is its own parent
        size[i] = 1
    }
    return &UnionFind{parent: parent, rank: rank, size: size, count: n}
}
```

### Memory Layout
```
n=5, initial state:
Index:  0  1  2  3  4
parent: 0  1  2  3  4  ← Each is its own root
rank:   0  0  0  0  0
size:   1  1  1  1  1

After Union(0,1), Union(2,3), Union(1,3):
Index:  0  1  2  3  4
parent: 0  0  2  0  4  ← 1,2,3 point to 0
rank:   1  0  0  0  0
size:   4  1  1  1  1
```

---

## Optimizations

### 1. Path Compression
```go
// Without path compression: O(n) worst case
func (uf *UnionFind) FindNaive(x int) int {
    for uf.parent[x] != x {
        x = uf.parent[x]
    }
    return x
}

// With path compression: O(α(n)) amortized
func (uf *UnionFind) Find(x int) int {
    if uf.parent[x] != x {
        uf.parent[x] = uf.Find(uf.parent[x])  // Path compression
    }
    return uf.parent[x]
}

// Path compression visualization:
// Before:  0 ← 1 ← 2 ← 3 ← 4
// Find(4): 0 ← 1 ← 2 ← 3 ← 4  (traverses chain)
// After:   0 ← 1, 0 ← 2, 0 ← 3, 0 ← 4  (all point to root)
```

### 2. Union by Rank
```go
func (uf *UnionFind) UnionByRank(x, y int) bool {
    rootX, rootY := uf.Find(x), uf.Find(y)
    if rootX == rootY {
        return false  // Already in same set
    }

    // Attach smaller rank tree under larger rank tree
    if uf.rank[rootX] < uf.rank[rootY] {
        uf.parent[rootX] = rootY
    } else if uf.rank[rootX] > uf.rank[rootY] {
        uf.parent[rootY] = rootX
    } else {
        uf.parent[rootY] = rootX
        uf.rank[rootX]++
    }
    uf.count--
    return true
}
```

### 3. Union by Size
```go
func (uf *UnionFind) Union(x, y int) bool {
    rootX, rootY := uf.Find(x), uf.Find(y)
    if rootX == rootY {
        return false
    }

    // Attach smaller tree under larger tree
    if uf.size[rootX] < uf.size[rootY] {
        uf.parent[rootX] = rootY
        uf.size[rootY] += uf.size[rootX]
    } else {
        uf.parent[rootY] = rootX
        uf.size[rootX] += uf.size[rootY]
    }
    uf.count--
    return true
}

func (uf *UnionFind) Connected(x, y int) bool {
    return uf.Find(x) == uf.Find(y)
}

func (uf *UnionFind) ComponentSize(x int) int {
    return uf.size[uf.Find(x)]
}

func (uf *UnionFind) Count() int {
    return uf.count
}
```

### Complexity with Optimizations
```
Without optimizations:  O(n) per operation
With path compression:  O(log n) amortized
With union by rank:     O(log n) amortized
With both:              O(α(n)) ≈ O(1) amortized
                        α = inverse Ackermann function
                        α(n) ≤ 4 for all practical n
```

---

## Union-Find Applications

### Kruskal's MST
```go
func kruskalMST(n int, edges [][]int) int {
    sort.Slice(edges, func(i, j int) bool {
        return edges[i][2] < edges[j][2]
    })

    uf := NewUnionFind(n)
    totalWeight := 0

    for _, e := range edges {
        if uf.Union(e[0], e[1]) {
            totalWeight += e[2]
        }
    }
    return totalWeight
}
```

### Number of Connected Components
```go
func countComponents(n int, edges [][2]int) int {
    uf := NewUnionFind(n)
    for _, e := range edges {
        uf.Union(e[0], e[1])
    }
    return uf.Count()
}
```

### Detect Cycle in Undirected Graph
```go
func hasCycle(n int, edges [][2]int) bool {
    uf := NewUnionFind(n)
    for _, e := range edges {
        if !uf.Union(e[0], e[1]) {
            return true  // Already connected = cycle
        }
    }
    return false
}
```

### Accounts Merge
```go
func accountsMerge(accounts [][]string) [][]string {
    emailToID := make(map[string]int)
    emailToName := make(map[string]string)
    id := 0

    for _, account := range accounts {
        name := account[0]
        for _, email := range account[1:] {
            if _, exists := emailToID[email]; !exists {
                emailToID[email] = id
                id++
            }
            emailToName[email] = name
        }
    }

    uf := NewUnionFind(id)
    for _, account := range accounts {
        firstID := emailToID[account[1]]
        for _, email := range account[2:] {
            uf.Union(firstID, emailToID[email])
        }
    }

    groups := make(map[int][]string)
    for email, eid := range emailToID {
        root := uf.Find(eid)
        groups[root] = append(groups[root], email)
    }

    result := [][]string{}
    for root, emails := range groups {
        sort.Strings(emails)
        name := emailToName[emails[0]]
        _ = root
        result = append(result, append([]string{name}, emails...))
    }
    return result
}
```

---

# Segment Tree

## Segment Tree Fundamentals

### Definition
A segment tree is a binary tree where each node stores aggregate information (sum, min, max) for a range of array elements.

```
Array: [1, 3, 5, 7, 9, 11]

Segment Tree (sum):
              36 [0,5]
            /          \
        9 [0,2]        27 [3,5]
        /    \          /    \
    4 [0,1] 5[2,2]  16[3,4] 11[5,5]
    /    \          /    \
 1[0,0] 3[1,1]  7[3,3] 9[4,4]
```

### Key Properties
- Leaf nodes represent individual elements
- Internal nodes represent ranges
- Height = O(log n)
- Supports range queries and point updates in O(log n)

---

## Segment Tree Internal Implementation

### Array-based Segment Tree
```go
type SegmentTree struct {
    tree []int
    n    int
}

func NewSegmentTree(arr []int) *SegmentTree {
    n := len(arr)
    tree := make([]int, 4*n)  // 4n is safe upper bound
    st := &SegmentTree{tree: tree, n: n}
    st.build(arr, 0, 0, n-1)
    return st
}

func (st *SegmentTree) build(arr []int, node, start, end int) {
    if start == end {
        st.tree[node] = arr[start]
        return
    }
    mid := (start + end) / 2
    st.build(arr, 2*node+1, start, mid)
    st.build(arr, 2*node+2, mid+1, end)
    st.tree[node] = st.tree[2*node+1] + st.tree[2*node+2]  // Sum
}

// Node index relationships (0-indexed):
// Left child:  2*i + 1
// Right child: 2*i + 2
// Parent:      (i-1) / 2
```

### Memory Layout
```
Array: [1, 3, 5, 7]
Tree array (4*n = 16 slots):

Index: 0   1   2   3   4   5   6
Value: 16  4   12  1   3   5   7
       ↑   ↑   ↑   ↑   ↑   ↑   ↑
      root [0,1][2,3][0][1][2][3]
```

---

## Range Queries

### Range Sum Query
```go
func (st *SegmentTree) Query(node, start, end, l, r int) int {
    if r < start || end < l {
        return 0  // Out of range
    }
    if l <= start && end <= r {
        return st.tree[node]  // Fully within range
    }
    mid := (start + end) / 2
    leftSum := st.Query(2*node+1, start, mid, l, r)
    rightSum := st.Query(2*node+2, mid+1, end, l, r)
    return leftSum + rightSum
}

func (st *SegmentTree) RangeSum(l, r int) int {
    return st.Query(0, 0, st.n-1, l, r)
}

// Time: O(log n)
```

### Point Update
```go
func (st *SegmentTree) Update(node, start, end, idx, val int) {
    if start == end {
        st.tree[node] = val
        return
    }
    mid := (start + end) / 2
    if idx <= mid {
        st.Update(2*node+1, start, mid, idx, val)
    } else {
        st.Update(2*node+2, mid+1, end, idx, val)
    }
    st.tree[node] = st.tree[2*node+1] + st.tree[2*node+2]
}

func (st *SegmentTree) PointUpdate(idx, val int) {
    st.Update(0, 0, st.n-1, idx, val)
}

// Time: O(log n)
```

### Range Min Query
```go
type MinSegTree struct {
    tree []int
    n    int
}

func (st *MinSegTree) build(arr []int, node, start, end int) {
    if start == end {
        st.tree[node] = arr[start]
        return
    }
    mid := (start + end) / 2
    st.build(arr, 2*node+1, start, mid)
    st.build(arr, 2*node+2, mid+1, end)
    st.tree[node] = min(st.tree[2*node+1], st.tree[2*node+2])
}

func (st *MinSegTree) Query(node, start, end, l, r int) int {
    if r < start || end < l {
        return math.MaxInt32
    }
    if l <= start && end <= r {
        return st.tree[node]
    }
    mid := (start + end) / 2
    return min(
        st.Query(2*node+1, start, mid, l, r),
        st.Query(2*node+2, mid+1, end, l, r),
    )
}
```

---

## Lazy Propagation

### Definition
Defers range updates to avoid O(n) updates. Stores pending updates in lazy array.

```go
type LazySegTree struct {
    tree []int
    lazy []int
    n    int
}

func NewLazySegTree(arr []int) *LazySegTree {
    n := len(arr)
    st := &LazySegTree{
        tree: make([]int, 4*n),
        lazy: make([]int, 4*n),
        n:    n,
    }
    st.build(arr, 0, 0, n-1)
    return st
}

func (st *LazySegTree) build(arr []int, node, start, end int) {
    if start == end {
        st.tree[node] = arr[start]
        return
    }
    mid := (start + end) / 2
    st.build(arr, 2*node+1, start, mid)
    st.build(arr, 2*node+2, mid+1, end)
    st.tree[node] = st.tree[2*node+1] + st.tree[2*node+2]
}

func (st *LazySegTree) propagate(node, start, end int) {
    if st.lazy[node] != 0 {
        mid := (start + end) / 2
        st.tree[2*node+1] += st.lazy[node] * (mid - start + 1)
        st.tree[2*node+2] += st.lazy[node] * (end - mid)
        st.lazy[2*node+1] += st.lazy[node]
        st.lazy[2*node+2] += st.lazy[node]
        st.lazy[node] = 0
    }
}

// Range Update: add val to all elements in [l, r]
func (st *LazySegTree) RangeUpdate(node, start, end, l, r, val int) {
    if r < start || end < l {
        return
    }
    if l <= start && end <= r {
        st.tree[node] += val * (end - start + 1)
        st.lazy[node] += val
        return
    }
    st.propagate(node, start, end)
    mid := (start + end) / 2
    st.RangeUpdate(2*node+1, start, mid, l, r, val)
    st.RangeUpdate(2*node+2, mid+1, end, l, r, val)
    st.tree[node] = st.tree[2*node+1] + st.tree[2*node+2]
}

func (st *LazySegTree) Query(node, start, end, l, r int) int {
    if r < start || end < l {
        return 0
    }
    if l <= start && end <= r {
        return st.tree[node]
    }
    st.propagate(node, start, end)
    mid := (start + end) / 2
    return st.Query(2*node+1, start, mid, l, r) +
        st.Query(2*node+2, mid+1, end, l, r)
}

// Time: O(log n) for both range update and range query
```

---

## Segment Tree Applications

### Range Sum with Updates
```go
// Classic use case
st := NewSegmentTree([]int{1, 3, 5, 7, 9, 11})
sum := st.RangeSum(1, 3)    // Sum of [3,5,7] = 15
st.PointUpdate(1, 10)        // arr[1] = 10
sum = st.RangeSum(1, 3)     // Sum of [10,5,7] = 22
```

### Count of Range Sum
```go
// Count subarrays with sum in [lower, upper]
// Uses merge sort tree or segment tree on prefix sums
```

### Interval Scheduling
```go
// Find maximum non-overlapping intervals
// Segment tree on time axis
```

---

# Bloom Filter

## Bloom Filter Fundamentals

### Definition
A probabilistic data structure that tests whether an element is a member of a set. May have false positives but never false negatives.

```
"Definitely not in set" — always correct
"Probably in set"       — may be wrong (false positive)

Use case: Quick membership test before expensive lookup
```

### How It Works
```
Bit array of m bits, k hash functions

Insert "hello":
hash1("hello") = 2 → set bit 2
hash2("hello") = 5 → set bit 5
hash3("hello") = 9 → set bit 9

Bit array: [0,0,1,0,0,1,0,0,0,1,0,0,0,0,0,0]
                ↑       ↑           ↑

Query "hello":
hash1("hello") = 2 → bit 2 is 1 ✓
hash2("hello") = 5 → bit 5 is 1 ✓
hash3("hello") = 9 → bit 9 is 1 ✓
→ "Probably in set"

Query "world":
hash1("world") = 2 → bit 2 is 1 ✓
hash2("world") = 7 → bit 7 is 0 ✗
→ "Definitely not in set"
```

---

## Bloom Filter Internal Implementation

```go
import (
    "hash/fnv"
    "math"
)

type BloomFilter struct {
    bits    []bool
    m       uint  // Number of bits
    k       uint  // Number of hash functions
    count   uint  // Number of inserted elements
}

func NewBloomFilter(expectedItems uint, falsePositiveRate float64) *BloomFilter {
    m := optimalM(expectedItems, falsePositiveRate)
    k := optimalK(m, expectedItems)
    return &BloomFilter{
        bits: make([]bool, m),
        m:    m,
        k:    k,
    }
}

// Optimal number of bits: m = -n*ln(p) / (ln(2))²
func optimalM(n uint, p float64) uint {
    return uint(math.Ceil(-float64(n) * math.Log(p) / (math.Log(2) * math.Log(2))))
}

// Optimal number of hash functions: k = (m/n) * ln(2)
func optimalK(m, n uint) uint {
    return uint(math.Round(float64(m) / float64(n) * math.Log(2)))
}

func (bf *BloomFilter) hashes(data []byte) []uint {
    h1 := fnv.New64()
    h1.Write(data)
    v1 := h1.Sum64()

    h2 := fnv.New64a()
    h2.Write(data)
    v2 := h2.Sum64()

    // Double hashing: h_i(x) = h1(x) + i*h2(x)
    positions := make([]uint, bf.k)
    for i := uint(0); i < bf.k; i++ {
        positions[i] = uint((v1 + uint64(i)*v2) % uint64(bf.m))
    }
    return positions
}

func (bf *BloomFilter) Add(item string) {
    for _, pos := range bf.hashes([]byte(item)) {
        bf.bits[pos] = true
    }
    bf.count++
}

func (bf *BloomFilter) Contains(item string) bool {
    for _, pos := range bf.hashes([]byte(item)) {
        if !bf.bits[pos] {
            return false  // Definitely not in set
        }
    }
    return true  // Probably in set
}

func (bf *BloomFilter) FalsePositiveRate() float64 {
    // (1 - e^(-k*n/m))^k
    exponent := -float64(bf.k) * float64(bf.count) / float64(bf.m)
    return math.Pow(1-math.Exp(exponent), float64(bf.k))
}
```

---

## False Positive Rate

### Mathematical Analysis
```
m = number of bits
n = number of inserted elements
k = number of hash functions

Probability of false positive:
p ≈ (1 - e^(-kn/m))^k

Optimal k = (m/n) * ln(2) ≈ 0.693 * (m/n)

Example:
n = 1000 items
p = 1% false positive rate
m = -1000 * ln(0.01) / (ln(2))² ≈ 9585 bits ≈ 1.2 KB
k = 9585/1000 * ln(2) ≈ 7 hash functions
```

### Space Efficiency
```
Bloom filter vs Hash Set:
1 million items, 1% false positive:
- Bloom filter: ~1.2 MB
- Hash set (string keys): ~50-100 MB

~50-100x more space efficient!
```

---

## Bloom Filter Applications

### Web Cache (Avoid Disk Lookup)
```go
type WebCache struct {
    bloom *BloomFilter
    cache map[string]string
    db    Database
}

func (wc *WebCache) Get(url string) string {
    // Quick check: definitely not cached?
    if !wc.bloom.Contains(url) {
        return wc.db.Fetch(url)  // Go to DB
    }
    // Probably cached — check actual cache
    if val, ok := wc.cache[url]; ok {
        return val
    }
    // False positive — fetch from DB
    return wc.db.Fetch(url)
}

func (wc *WebCache) Set(url, content string) {
    wc.bloom.Add(url)
    wc.cache[url] = content
}
```

### Duplicate URL Detection (Web Crawler)
```go
type Crawler struct {
    visited *BloomFilter
    queue   []string
}

func (c *Crawler) ShouldVisit(url string) bool {
    if c.visited.Contains(url) {
        return false  // Probably already visited
    }
    c.visited.Add(url)
    return true
}
```

### Spam Filter
```go
type SpamFilter struct {
    spamWords *BloomFilter
}

func (sf *SpamFilter) IsSpam(email string) bool {
    words := strings.Fields(email)
    spamCount := 0
    for _, word := range words {
        if sf.spamWords.Contains(strings.ToLower(word)) {
            spamCount++
        }
    }
    return spamCount > len(words)/3
}
```

### Password Breach Check
```go
// Check if password appears in breach database
// Without storing all breached passwords
type BreachChecker struct {
    bloom *BloomFilter
}

func (bc *BreachChecker) IsBreached(password string) bool {
    return bc.bloom.Contains(password)
    // false = definitely safe
    // true  = probably breached (verify with full DB)
}
```

---

## Performance Comparison

### Union-Find
| Operation | Naive | Path Compression | Both Optimizations |
|-----------|-------|-----------------|-------------------|
| Find | O(n) | O(log n) | O(α(n)) ≈ O(1) |
| Union | O(n) | O(log n) | O(α(n)) ≈ O(1) |
| Space | O(n) | O(n) | O(n) |

### Segment Tree
| Operation | Naive Array | Segment Tree | Lazy Segment Tree |
|-----------|-------------|--------------|-------------------|
| Point Update | O(1) | O(log n) | O(log n) |
| Range Query | O(n) | O(log n) | O(log n) |
| Range Update | O(n) | O(n log n) | O(log n) |
| Space | O(n) | O(n) | O(n) |

### Bloom Filter
| Operation | Hash Set | Bloom Filter |
|-----------|----------|--------------|
| Insert | O(1) | O(k) |
| Lookup | O(1) | O(k) |
| Delete | O(1) | ❌ Not supported* |
| Space | O(n) | O(m) << O(n) |
| False positives | None | Yes (tunable) |
| False negatives | None | None |

*Counting Bloom Filter supports deletion

---

## Summary

### Union-Find
- Maintains disjoint sets with near-O(1) operations
- Path compression + union by rank = O(α(n)) ≈ O(1)
- Essential for: Kruskal's MST, cycle detection, connected components

### Segment Tree
- Range queries and updates in O(log n)
- Lazy propagation for range updates
- Use when: frequent range queries with updates

### Bloom Filter
- Probabilistic membership test
- Space-efficient (50-100x vs hash set)
- No false negatives, tunable false positive rate
- Use when: quick pre-filter before expensive lookup

### When to Use Each
| Problem | Data Structure |
|---------|---------------|
| Dynamic connectivity | Union-Find |
| Cycle detection | Union-Find |
| Range sum/min/max queries | Segment Tree |
| Range updates + queries | Lazy Segment Tree |
| Membership test (space-critical) | Bloom Filter |
| Duplicate detection at scale | Bloom Filter |
| Cache pre-check | Bloom Filter |

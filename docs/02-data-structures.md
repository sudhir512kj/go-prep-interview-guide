# Data Structures in Golang

## Table of Contents
- [Introduction](#introduction)
  - [Complexity Analysis](#complexity-analysis)
- [Arrays and Slices](#arrays-and-slices)
  - [Memory Layout](#memory-layout)
  - [Array Operations](#array-operations)
  - [Slice Operations](#slice-operations)
- [Maps](#maps)
  - [Hash Map Implementation](#hash-map-implementation)
- [Linked List](#linked-list)
  - [Theory](#theory)
  - [Singly Linked List](#singly-linked-list)
  - [Doubly Linked List](#doubly-linked-list)
- [Stack](#stack)
  - [Theory](#theory-1)
  - [Implementation](#implementation)
- [Queue](#queue)
  - [Theory](#theory-2)
  - [Implementation](#implementation-1)
  - [Circular Queue](#circular-queue)
- [Binary Tree](#binary-tree)
  - [Theory](#theory-3)
  - [Implementation](#implementation-2)
  - [Tree Traversals](#tree-traversals)
  - [Tree Operations](#tree-operations)
- [Heap (Priority Queue)](#heap-priority-queue)
  - [Theory](#theory-4)
  - [Implementation](#implementation-3)
- [Graph](#graph)
  - [Theory](#theory-5)
  - [Implementation](#implementation-4)
  - [Graph Algorithms](#graph-algorithms)
- [Trie (Prefix Tree)](#trie-prefix-tree)
  - [Theory](#theory-6)
  - [Implementation](#implementation-5)
  - [Trie Operations](#trie-operations)
- [Union-Find (Disjoint Set)](#union-find-disjoint-set)
  - [Theory](#theory-7)
  - [Implementation](#implementation-6)
- [Segment Tree](#segment-tree)
  - [Theory](#theory-8)
  - [Implementation](#implementation-7)
- [Bloom Filter](#bloom-filter)
  - [Theory](#theory-9)
  - [Implementation](#implementation-8)

---

## Introduction

Data structures are fundamental to efficient programming. Go provides built-in support for arrays, slices, and maps, while other structures can be implemented using these primitives and structs.

### Complexity Analysis

Understanding time and space complexity is crucial:
- **O(1)**: Constant time - operation takes same time regardless of input size
- **O(log n)**: Logarithmic - time grows logarithmically with input (e.g., binary search)
- **O(n)**: Linear - time grows linearly with input
- **O(n log n)**: Linearithmic - efficient sorting algorithms
- **O(n²)**: Quadratic - nested loops over input
- **O(2ⁿ)**: Exponential - recursive algorithms without memoization

## Arrays and Slices

### Memory Layout

**Arrays:**
- Contiguous memory block
- Fixed size known at compile time
- Stack allocated (if small enough)
- Value semantics (copied on assignment)

**Slices:**
- Three-word structure: pointer, length, capacity
- Points to underlying array
- Heap allocated (underlying array)
- Reference semantics (shares underlying array)

### Array Operations
```go
// Fixed size array
arr := [5]int{1, 2, 3, 4, 5}

// Length
len(arr) // 5

// Iterate
for i, v := range arr {
    fmt.Println(i, v)
}
```

### Slice Operations
```go
// Create slice
s := []int{1, 2, 3}
s = append(s, 4, 5)

// Copy slice
dest := make([]int, len(s))
copy(dest, s)

// Slice tricks
// Remove element at index i
s = append(s[:i], s[i+1:]...)

// Insert at index i
s = append(s[:i], append([]int{x}, s[i:]...)...)

// Reverse
for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
    s[i], s[j] = s[j], s[i]
}
```

## Maps

### Hash Map Implementation
```go
type HashMap struct {
    data map[string]interface{}
}

func NewHashMap() *HashMap {
    return &HashMap{data: make(map[string]interface{})}
}

func (h *HashMap) Put(key string, value interface{}) {
    h.data[key] = value
}

func (h *HashMap) Get(key string) (interface{}, bool) {
    val, exists := h.data[key]
    return val, exists
}

func (h *HashMap) Delete(key string) {
    delete(h.data, key)
}
```

## Linked List

### Theory

A linked list is a linear data structure where elements are stored in nodes. Each node contains data and a reference (link) to the next node.

**Advantages:**
- Dynamic size
- Efficient insertion/deletion at beginning: O(1)
- No memory waste (allocates as needed)

**Disadvantages:**
- No random access: O(n) to access element
- Extra memory for pointers
- Poor cache locality

**When to Use:**
- Frequent insertions/deletions at beginning
- Unknown or highly variable size
- Implementing stacks, queues, or other ADTs

### Singly Linked List
```go
type Node struct {
    Value int
    Next  *Node
}

type LinkedList struct {
    Head *Node
}

func (l *LinkedList) Insert(value int) {
    newNode := &Node{Value: value}
    if l.Head == nil {
        l.Head = newNode
        return
    }
    current := l.Head
    for current.Next != nil {
        current = current.Next
    }
    current.Next = newNode
}

func (l *LinkedList) Delete(value int) {
    if l.Head == nil {
        return
    }
    if l.Head.Value == value {
        l.Head = l.Head.Next
        return
    }
    current := l.Head
    for current.Next != nil {
        if current.Next.Value == value {
            current.Next = current.Next.Next
            return
        }
        current = current.Next
    }
}

func (l *LinkedList) Search(value int) *Node {
    current := l.Head
    for current != nil {
        if current.Value == value {
            return current
        }
        current = current.Next
    }
    return nil
}

func (l *LinkedList) Reverse() {
    var prev *Node
    current := l.Head
    for current != nil {
        next := current.Next
        current.Next = prev
        prev = current
        current = next
    }
    l.Head = prev
}
```

### Doubly Linked List

**Advantages over Singly Linked:**
- Can traverse backwards
- Easier deletion (don't need previous node)
- Can delete node given only pointer to it

**Disadvantages:**
- Extra memory for prev pointer
- More complex insertion/deletion logic
```go
type DNode struct {
    Value int
    Next  *DNode
    Prev  *DNode
}

type DoublyLinkedList struct {
    Head *DNode
    Tail *DNode
}

func (d *DoublyLinkedList) InsertAtEnd(value int) {
    newNode := &DNode{Value: value}
    if d.Head == nil {
        d.Head = newNode
        d.Tail = newNode
        return
    }
    d.Tail.Next = newNode
    newNode.Prev = d.Tail
    d.Tail = newNode
}
```

## Stack

### Theory

Stack is a Last-In-First-Out (LIFO) data structure. Think of a stack of plates - you add and remove from the top.

**Operations:**
- Push: O(1) - add element to top
- Pop: O(1) - remove element from top
- Peek: O(1) - view top element

**Applications:**
- Function call stack
- Expression evaluation
- Backtracking algorithms
- Undo mechanisms
- Syntax parsing

```go
type Stack struct {
    items []int
}

func (s *Stack) Push(item int) {
    s.items = append(s.items, item)
}

func (s *Stack) Pop() (int, bool) {
    if len(s.items) == 0 {
        return 0, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

func (s *Stack) Peek() (int, bool) {
    if len(s.items) == 0 {
        return 0, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack) IsEmpty() bool {
    return len(s.items) == 0
}

func (s *Stack) Size() int {
    return len(s.items)
}
```

## Queue

### Theory

Queue is a First-In-First-Out (FIFO) data structure. Think of a line at a store - first person in line is served first.

**Operations:**
- Enqueue: O(1) - add element to rear
- Dequeue: O(1) - remove element from front
- Front: O(1) - view front element

**Applications:**
- Task scheduling
- Breadth-first search
- Request handling
- Print queue
- Message queues

**Note:** The simple slice-based implementation above has O(n) dequeue due to slice reslicing. For better performance, use a circular buffer or linked list.

```go
type Queue struct {
    items []int
}

func (q *Queue) Enqueue(item int) {
    q.items = append(q.items, item)
}

func (q *Queue) Dequeue() (int, bool) {
    if len(q.items) == 0 {
        return 0, false
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item, true
}

// Circular Queue (more efficient)
type CircularQueue struct {
    items []int
    front int
    rear  int
    size  int
    cap   int
}

func NewCircularQueue(capacity int) *CircularQueue {
    return &CircularQueue{
        items: make([]int, capacity),
        cap:   capacity,
    }
}

func (q *CircularQueue) Enqueue(item int) bool {
    if q.size == q.cap {
        return false // Queue full
    }
    q.items[q.rear] = item
    q.rear = (q.rear + 1) % q.cap
    q.size++
    return true
}

func (q *CircularQueue) Dequeue() (int, bool) {
    if q.size == 0 {
        return 0, false
    }
    item := q.items[q.front]
    q.front = (q.front + 1) % q.cap
    q.size--
    return item, true
}

func (q *Queue) Front() (int, bool) {
    if len(q.items) == 0 {
        return 0, false
    }
    return q.items[0], true
}

func (q *Queue) IsEmpty() bool {
    return len(q.items) == 0
}

func (q *Queue) Size() int {
    return len(q.items)
}
```

## Binary Tree

### Theory

A binary tree is a hierarchical data structure where each node has at most two children (left and right).

**Types:**
1. **Binary Search Tree (BST)**: Left child < parent < right child
2. **Complete Binary Tree**: All levels filled except possibly last, filled left to right
3. **Full Binary Tree**: Every node has 0 or 2 children
4. **Perfect Binary Tree**: All internal nodes have 2 children, all leaves at same level
5. **Balanced Binary Tree**: Height difference between left and right subtrees ≤ 1

**BST Properties:**
- Search: O(log n) average, O(n) worst
- Insert: O(log n) average, O(n) worst
- Delete: O(log n) average, O(n) worst
- Inorder traversal gives sorted sequence

**Traversal Methods:**
1. **Inorder (Left-Root-Right)**: Gives sorted order for BST
2. **Preorder (Root-Left-Right)**: Used for copying tree
3. **Postorder (Left-Right-Root)**: Used for deleting tree
4. **Level-order**: BFS traversal

```go
type TreeNode struct {
    Value int
    Left  *TreeNode
    Right *TreeNode
}

type BinaryTree struct {
    Root *TreeNode
}

// Insert
func (t *BinaryTree) Insert(value int) {
    t.Root = insertNode(t.Root, value)
}

func insertNode(node *TreeNode, value int) *TreeNode {
    if node == nil {
        return &TreeNode{Value: value}
    }
    if value < node.Value {
        node.Left = insertNode(node.Left, value)
    } else {
        node.Right = insertNode(node.Right, value)
    }
    return node
}

// Search
func (t *BinaryTree) Search(value int) bool {
    return searchNode(t.Root, value)
}

func searchNode(node *TreeNode, value int) bool {
    if node == nil {
        return false
    }
    if node.Value == value {
        return true
    }
    if value < node.Value {
        return searchNode(node.Left, value)
    }
    return searchNode(node.Right, value)
}

// Inorder Traversal
func (t *BinaryTree) InorderTraversal() []int {
    result := []int{}
    inorder(t.Root, &result)
    return result
}

func inorder(node *TreeNode, result *[]int) {
    if node == nil {
        return
    }
    inorder(node.Left, result)
    *result = append(*result, node.Value)
    inorder(node.Right, result)
}

// Preorder Traversal
func preorder(node *TreeNode, result *[]int) {
    if node == nil {
        return
    }
    *result = append(*result, node.Value)
    preorder(node.Left, result)
    preorder(node.Right, result)
}

// Postorder Traversal
func postorder(node *TreeNode, result *[]int) {
    if node == nil {
        return
    }
    postorder(node.Left, result)
    postorder(node.Right, result)
    *result = append(*result, node.Value)
}

// Height of tree
func (t *BinaryTree) Height() int {
    return height(t.Root)
}

func height(node *TreeNode) int {
    if node == nil {
        return 0
    }
    leftHeight := height(node.Left)
    rightHeight := height(node.Right)
    if leftHeight > rightHeight {
        return leftHeight + 1
    }
    return rightHeight + 1
}

// Check if tree is balanced
func (t *BinaryTree) IsBalanced() bool {
    _, balanced := checkBalance(t.Root)
    return balanced
}

func checkBalance(node *TreeNode) (int, bool) {
    if node == nil {
        return 0, true
    }
    
    leftHeight, leftBalanced := checkBalance(node.Left)
    if !leftBalanced {
        return 0, false
    }
    
    rightHeight, rightBalanced := checkBalance(node.Right)
    if !rightBalanced {
        return 0, false
    }
    
    diff := leftHeight - rightHeight
    if diff < 0 {
        diff = -diff
    }
    
    if diff > 1 {
        return 0, false
    }
    
    height := leftHeight
    if rightHeight > leftHeight {
        height = rightHeight
    }
    return height + 1, true
}

// Level Order Traversal
func (t *BinaryTree) LevelOrder() [][]int {
    result := [][]int{}
    if t.Root == nil {
        return result
    }
    queue := []*TreeNode{t.Root}
    for len(queue) > 0 {
        levelSize := len(queue)
        level := []int{}
        for i := 0; i < levelSize; i++ {
            node := queue[0]
            queue = queue[1:]
            level = append(level, node.Value)
            if node.Left != nil {
                queue = append(queue, node.Left)
            }
            if node.Right != nil {
                queue = append(queue, node.Right)
            }
        }
        result = append(result, level)
    }
    return result
}
```

## Heap (Priority Queue)

### Theory

A heap is a complete binary tree that satisfies the heap property:
- **Min Heap**: Parent ≤ children (smallest element at root)
- **Max Heap**: Parent ≥ children (largest element at root)

**Properties:**
- Insert: O(log n)
- Extract min/max: O(log n)
- Peek min/max: O(1)
- Build heap: O(n)

**Array Representation:**
For node at index i:
- Left child: 2i + 1
- Right child: 2i + 2
- Parent: (i - 1) / 2

**Applications:**
- Priority queues
- Heap sort
- Finding k largest/smallest elements
- Median maintenance
- Dijkstra's shortest path

```go
type MinHeap struct {
    items []int
}

func (h *MinHeap) Push(item int) {
    h.items = append(h.items, item)
    h.heapifyUp(len(h.items) - 1)
}

func (h *MinHeap) Pop() (int, bool) {
    if len(h.items) == 0 {
        return 0, false
    }
    item := h.items[0]
    lastIdx := len(h.items) - 1
    h.items[0] = h.items[lastIdx]
    h.items = h.items[:lastIdx]
    h.heapifyDown(0)
    return item, true
}

func (h *MinHeap) heapifyUp(index int) {
    for index > 0 {
        parent := (index - 1) / 2
        if h.items[parent] <= h.items[index] {
            break
        }
        h.items[parent], h.items[index] = h.items[index], h.items[parent]
        index = parent
    }
}

func (h *MinHeap) Peek() (int, bool) {
    if len(h.items) == 0 {
        return 0, false
    }
    return h.items[0], true
}

func (h *MinHeap) Size() int {
    return len(h.items)
}

func (h *MinHeap) heapifyDown(index int) {
    for {
        left := 2*index + 1
        right := 2*index + 2
        smallest := index
        
        if left < len(h.items) && h.items[left] < h.items[smallest] {
            smallest = left
        }
        if right < len(h.items) && h.items[right] < h.items[smallest] {
            smallest = right
        }
        if smallest == index {
            break
        }
        h.items[index], h.items[smallest] = h.items[smallest], h.items[index]
        index = smallest
    }
}
```

## Graph

### Theory

A graph is a collection of nodes (vertices) connected by edges.

**Types:**
1. **Directed vs Undirected**: Edges have direction or not
2. **Weighted vs Unweighted**: Edges have weights or not
3. **Cyclic vs Acyclic**: Contains cycles or not
4. **Connected vs Disconnected**: All vertices reachable or not

**Representations:**
1. **Adjacency List**: Map of vertex to list of neighbors (space efficient for sparse graphs)
2. **Adjacency Matrix**: 2D array (space efficient for dense graphs, O(1) edge lookup)
3. **Edge List**: List of all edges

**Common Algorithms:**
- BFS: Shortest path in unweighted graph, level-order traversal
- DFS: Cycle detection, topological sort, connected components
- Dijkstra: Shortest path in weighted graph
- Bellman-Ford: Shortest path with negative weights
- Floyd-Warshall: All-pairs shortest path
- Kruskal/Prim: Minimum spanning tree

```go
type Graph struct {
    vertices map[int][]int
}

func NewGraph() *Graph {
    return &Graph{vertices: make(map[int][]int)}
}

func (g *Graph) AddEdge(from, to int) {
    g.vertices[from] = append(g.vertices[from], to)
}

// BFS
func (g *Graph) BFS(start int) []int {
    visited := make(map[int]bool)
    result := []int{}
    queue := []int{start}
    visited[start] = true
    
    for len(queue) > 0 {
        vertex := queue[0]
        queue = queue[1:]
        result = append(result, vertex)
        
        for _, neighbor := range g.vertices[vertex] {
            if !visited[neighbor] {
                visited[neighbor] = true
                queue = append(queue, neighbor)
            }
        }
    }
    return result
}

// DFS
func (g *Graph) DFS(start int) []int {
    visited := make(map[int]bool)
    result := []int{}
    g.dfsHelper(start, visited, &result)
    return result
}

// Detect cycle in directed graph
func (g *Graph) HasCycle() bool {
    visited := make(map[int]bool)
    recStack := make(map[int]bool)
    
    for vertex := range g.vertices {
        if !visited[vertex] {
            if g.hasCycleUtil(vertex, visited, recStack) {
                return true
            }
        }
    }
    return false
}

func (g *Graph) hasCycleUtil(vertex int, visited, recStack map[int]bool) bool {
    visited[vertex] = true
    recStack[vertex] = true
    
    for _, neighbor := range g.vertices[vertex] {
        if !visited[neighbor] {
            if g.hasCycleUtil(neighbor, visited, recStack) {
                return true
            }
        } else if recStack[neighbor] {
            return true
        }
    }
    
    recStack[vertex] = false
    return false
}

// Topological Sort (for DAG)
func (g *Graph) TopologicalSort() []int {
    visited := make(map[int]bool)
    stack := []int{}
    
    for vertex := range g.vertices {
        if !visited[vertex] {
            g.topologicalSortUtil(vertex, visited, &stack)
        }
    }
    
    // Reverse stack
    for i, j := 0, len(stack)-1; i < j; i, j = i+1, j-1 {
        stack[i], stack[j] = stack[j], stack[i]
    }
    return stack
}

func (g *Graph) topologicalSortUtil(vertex int, visited map[int]bool, stack *[]int) {
    visited[vertex] = true
    
    for _, neighbor := range g.vertices[vertex] {
        if !visited[neighbor] {
            g.topologicalSortUtil(neighbor, visited, stack)
        }
    }
    
    *stack = append(*stack, vertex)
}

func (g *Graph) dfsHelper(vertex int, visited map[int]bool, result *[]int) {
    visited[vertex] = true
    *result = append(*result, vertex)
    
    for _, neighbor := range g.vertices[vertex] {
        if !visited[neighbor] {
            g.dfsHelper(neighbor, visited, result)
        }
    }
}
```

## Trie (Prefix Tree)

### Theory

A trie is a tree-like data structure for storing strings, where each node represents a character.

**Properties:**
- Insert: O(m) where m is string length
- Search: O(m)
- Space: O(ALPHABET_SIZE * m * n) where n is number of strings

**Advantages:**
- Fast prefix searches
- Alphabetical ordering
- No hash collisions

**Applications:**
- Autocomplete
- Spell checker
- IP routing (longest prefix matching)
- Dictionary implementation
- Pattern matching

```go
type TrieNode struct {
    children map[rune]*TrieNode
    isEnd    bool
}

type Trie struct {
    root *TrieNode
}

func NewTrie() *Trie {
    return &Trie{root: &TrieNode{children: make(map[rune]*TrieNode)}}
}

func (t *Trie) Insert(word string) {
    node := t.root
    for _, ch := range word {
        if _, exists := node.children[ch]; !exists {
            node.children[ch] = &TrieNode{children: make(map[rune]*TrieNode)}
        }
        node = node.children[ch]
    }
    node.isEnd = true
}

func (t *Trie) Search(word string) bool {
    node := t.root
    for _, ch := range word {
        if _, exists := node.children[ch]; !exists {
            return false
        }
        node = node.children[ch]
    }
    return node.isEnd
}



// Delete word from trie
func (t *Trie) Delete(word string) bool {
    return deleteHelper(t.root, []rune(word), 0)
}

func deleteHelper(node *TrieNode, word []rune, index int) bool {
    if index == len(word) {
        if !node.isEnd {
            return false // Word doesn't exist
        }
        node.isEnd = false
        return len(node.children) == 0 // Return true if node has no children
    }
    
    ch := word[index]
    childNode, exists := node.children[ch]
    if !exists {
        return false
    }
    
    shouldDeleteChild := deleteHelper(childNode, word, index+1)
    
    if shouldDeleteChild {
        delete(node.children, ch)
        return len(node.children) == 0 && !node.isEnd
    }
    
    return false
}

// Find all words with given prefix
func (t *Trie) WordsWithPrefix(prefix string) []string {
    node := t.root
    for _, ch := range prefix {
        if _, exists := node.children[ch]; !exists {
            return []string{}
        }
        node = node.children[ch]
    }
    
    words := []string{}
    collectWords(node, prefix, &words)
    return words
}

func collectWords(node *TrieNode, prefix string, words *[]string) {
    if node.isEnd {
        *words = append(*words, prefix)
    }
    
    for ch, child := range node.children {
        collectWords(child, prefix+string(ch), words)
    }
}

## Union-Find (Disjoint Set)

### Theory

Union-Find is a data structure that tracks elements partitioned into disjoint sets.

**Operations:**
- Find: O(α(n)) - find set representative (nearly constant with path compression)
- Union: O(α(n)) - merge two sets

**Applications:**
- Kruskal's MST algorithm
- Cycle detection in undirected graphs
- Network connectivity
- Image processing (connected components)

```go
type UnionFind struct {
    parent []int
    rank   []int
}

func NewUnionFind(size int) *UnionFind {
    uf := &UnionFind{
        parent: make([]int, size),
        rank:   make([]int, size),
    }
    for i := range uf.parent {
        uf.parent[i] = i
    }
    return uf
}

// Find with path compression
func (uf *UnionFind) Find(x int) int {
    if uf.parent[x] != x {
        uf.parent[x] = uf.Find(uf.parent[x]) // Path compression
    }
    return uf.parent[x]
}

// Union by rank
func (uf *UnionFind) Union(x, y int) {
    rootX := uf.Find(x)
    rootY := uf.Find(y)
    
    if rootX == rootY {
        return
    }
    
    // Attach smaller rank tree under root of higher rank tree
    if uf.rank[rootX] < uf.rank[rootY] {
        uf.parent[rootX] = rootY
    } else if uf.rank[rootX] > uf.rank[rootY] {
        uf.parent[rootY] = rootX
    } else {
        uf.parent[rootY] = rootX
        uf.rank[rootX]++
    }
}

func (uf *UnionFind) Connected(x, y int) bool {
    return uf.Find(x) == uf.Find(y)
}
```

## Segment Tree

### Theory

Segment tree is used for range queries and updates on arrays.

**Operations:**
- Build: O(n)
- Query: O(log n)
- Update: O(log n)

**Applications:**
- Range sum/min/max queries
- Range updates
- Finding GCD/LCM in range

```go
type SegmentTree struct {
    tree []int
    n    int
}

func NewSegmentTree(arr []int) *SegmentTree {
    n := len(arr)
    st := &SegmentTree{
        tree: make([]int, 4*n),
        n:    n,
    }
    st.build(arr, 0, 0, n-1)
    return st
}

func (st *SegmentTree) build(arr []int, node, start, end int) {
    if start == end {
        st.tree[node] = arr[start]
        return
    }
    
    mid := (start + end) / 2
    leftChild := 2*node + 1
    rightChild := 2*node + 2
    
    st.build(arr, leftChild, start, mid)
    st.build(arr, rightChild, mid+1, end)
    
    st.tree[node] = st.tree[leftChild] + st.tree[rightChild]
}

func (st *SegmentTree) Query(l, r int) int {
    return st.query(0, 0, st.n-1, l, r)
}

func (st *SegmentTree) query(node, start, end, l, r int) int {
    if r < start || l > end {
        return 0 // Out of range
    }
    
    if l <= start && end <= r {
        return st.tree[node] // Completely in range
    }
    
    mid := (start + end) / 2
    leftSum := st.query(2*node+1, start, mid, l, r)
    rightSum := st.query(2*node+2, mid+1, end, l, r)
    
    return leftSum + rightSum
}

func (st *SegmentTree) Update(index, value int) {
    st.update(0, 0, st.n-1, index, value)
}

func (st *SegmentTree) update(node, start, end, index, value int) {
    if start == end {
        st.tree[node] = value
        return
    }
    
    mid := (start + end) / 2
    if index <= mid {
        st.update(2*node+1, start, mid, index, value)
    } else {
        st.update(2*node+2, mid+1, end, index, value)
    }
    
    st.tree[node] = st.tree[2*node+1] + st.tree[2*node+2]
}
```

## Bloom Filter

### Theory

A space-efficient probabilistic data structure for testing set membership.

**Properties:**
- Can have false positives (says element exists when it doesn't)
- Never has false negatives (if it says no, element definitely doesn't exist)
- Space efficient compared to hash tables

**Applications:**
- Cache filtering
- Spell checkers
- Database query optimization
- Network routers

```go
type BloomFilter struct {
    bits []bool
    size int
    hash int // Number of hash functions
}

func NewBloomFilter(size, hashCount int) *BloomFilter {
    return &BloomFilter{
        bits: make([]bool, size),
        size: size,
        hash: hashCount,
    }
}

func (bf *BloomFilter) Add(item string) {
    for i := 0; i < bf.hash; i++ {
        index := bf.hashFunc(item, i)
        bf.bits[index] = true
    }
}

func (bf *BloomFilter) Contains(item string) bool {
    for i := 0; i < bf.hash; i++ {
        index := bf.hashFunc(item, i)
        if !bf.bits[index] {
            return false
        }
    }
    return true // Might be false positive
}

func (bf *BloomFilter) hashFunc(item string, seed int) int {
    h := 0
    for i, ch := range item {
        h = (h*31 + int(ch) + seed*i) % bf.size
    }
    if h < 0 {
        h += bf.size
    }
    return h
}
    node := t.root
    for _, ch := range prefix {
        if _, exists := node.children[ch]; !exists {
            return false
        }
        node = node.children[ch]
    }
    return true
}
```

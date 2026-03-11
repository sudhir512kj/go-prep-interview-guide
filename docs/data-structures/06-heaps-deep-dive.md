# Heaps Deep Dive

## Table of Contents
- [Heap Fundamentals](#heap-fundamentals)
- [Internal Implementation](#internal-implementation)
- [Heap Operations](#heap-operations)
- [Min-Heap Implementation](#min-heap-implementation)
- [Max-Heap Implementation](#max-heap-implementation)
- [Go container/heap Interface](#go-containerheap-interface)
- [Heapify](#heapify)
- [Heap Sort](#heap-sort)
- [Priority Queue Patterns](#priority-queue-patterns)
- [Performance Characteristics](#performance-characteristics)
- [Common Patterns](#common-patterns)
- [Interview Problems](#interview-problems)

---

## Heap Fundamentals

### Definition
A heap is a complete binary tree satisfying the heap property:
- **Min-Heap**: Parent ≤ Children (root is minimum)
- **Max-Heap**: Parent ≥ Children (root is maximum)

```
Min-Heap:           Max-Heap:
      1                   9
    /   \               /   \
   3     2             7     8
  / \   / \           / \   / \
 7   4 5   6         2   4 3   5

Root = minimum      Root = maximum
```

### Key Properties
- Complete binary tree (all levels filled except possibly last, filled left to right)
- Stored efficiently as an array
- Root always contains min (min-heap) or max (max-heap)
- Height = O(log n)

---

## Internal Implementation

### Array Representation
```
Heap stored as array — no pointers needed!

Min-Heap:
        1
      /   \
     3     2
    / \   / \
   7   4 5   6

Array: [1, 3, 2, 7, 4, 5, 6]
Index:  0  1  2  3  4  5  6

Index relationships:
- Parent of i:      (i-1) / 2
- Left child of i:  2*i + 1
- Right child of i: 2*i + 2
```

### Index Relationships
```go
func parent(i int) int     { return (i - 1) / 2 }
func leftChild(i int) int  { return 2*i + 1 }
func rightChild(i int) int { return 2*i + 2 }

// Example: i=1 (value 3)
// parent(1) = 0 → value 1 ✓
// leftChild(1) = 3 → value 7 ✓
// rightChild(1) = 4 → value 4 ✓
```

### Memory Layout
```
Array: [1][3][2][7][4][5][6]
        0   1  2  3  4  5  6

Level 0 (root):    index 0         → 1 element
Level 1:           index 1-2       → 2 elements
Level 2:           index 3-6       → 4 elements
Level k:           index 2^k-1 to 2^(k+1)-2
```

---

## Heap Operations

### Sift Up (Bubble Up)
Used after inserting a new element at the end.

```go
func siftUp(heap []int, i int) {
    for i > 0 {
        p := (i - 1) / 2
        if heap[p] <= heap[i] {  // Min-heap: parent ≤ child
            break
        }
        heap[p], heap[i] = heap[i], heap[p]
        i = p
    }
}

// Visualization: Insert 1 into [2,3,5,7,4,6]
// Array: [2,3,5,7,4,6,1]  ← 1 added at end
//         sift up:
// Compare 1 with parent 5 (index 2): 1 < 5, swap
// Array: [2,3,1,7,4,6,5]
// Compare 1 with parent 2 (index 0): 1 < 2, swap
// Array: [1,3,2,7,4,6,5]  ← done
```

### Sift Down (Bubble Down)
Used after removing the root (replacing with last element).

```go
func siftDown(heap []int, i, n int) {
    for {
        smallest := i
        l, r := 2*i+1, 2*i+2

        if l < n && heap[l] < heap[smallest] {
            smallest = l
        }
        if r < n && heap[r] < heap[smallest] {
            smallest = r
        }
        if smallest == i {
            break
        }
        heap[i], heap[smallest] = heap[smallest], heap[i]
        i = smallest
    }
}

// Visualization: Remove root from [1,3,2,7,4,5,6]
// Step 1: Replace root with last element
// Array: [6,3,2,7,4,5]
// Step 2: Sift down 6
// Compare 6 with children 3,2: min is 2, swap
// Array: [2,3,6,7,4,5]
// Compare 6 with children 5: 5 < 6, swap
// Array: [2,3,5,7,4,6]  ← done
```

---

## Min-Heap Implementation

```go
type MinHeap struct {
    data []int
}

func (h *MinHeap) Len() int  { return len(h.data) }
func (h *MinHeap) IsEmpty() bool { return len(h.data) == 0 }

func (h *MinHeap) Push(val int) {
    h.data = append(h.data, val)
    h.siftUp(len(h.data) - 1)
}

func (h *MinHeap) Pop() (int, error) {
    if h.IsEmpty() {
        return 0, errors.New("heap is empty")
    }
    min := h.data[0]
    n := len(h.data) - 1
    h.data[0] = h.data[n]
    h.data = h.data[:n]
    if !h.IsEmpty() {
        h.siftDown(0)
    }
    return min, nil
}

func (h *MinHeap) Peek() (int, error) {
    if h.IsEmpty() {
        return 0, errors.New("heap is empty")
    }
    return h.data[0], nil
}

func (h *MinHeap) siftUp(i int) {
    for i > 0 {
        p := (i - 1) / 2
        if h.data[p] <= h.data[i] {
            break
        }
        h.data[p], h.data[i] = h.data[i], h.data[p]
        i = p
    }
}

func (h *MinHeap) siftDown(i int) {
    n := len(h.data)
    for {
        smallest := i
        l, r := 2*i+1, 2*i+2
        if l < n && h.data[l] < h.data[smallest] {
            smallest = l
        }
        if r < n && h.data[r] < h.data[smallest] {
            smallest = r
        }
        if smallest == i {
            break
        }
        h.data[i], h.data[smallest] = h.data[smallest], h.data[i]
        i = smallest
    }
}
```

---

## Max-Heap Implementation

```go
type MaxHeap struct {
    data []int
}

func (h *MaxHeap) Push(val int) {
    h.data = append(h.data, val)
    h.siftUp(len(h.data) - 1)
}

func (h *MaxHeap) Pop() (int, error) {
    if len(h.data) == 0 {
        return 0, errors.New("heap is empty")
    }
    max := h.data[0]
    n := len(h.data) - 1
    h.data[0] = h.data[n]
    h.data = h.data[:n]
    if len(h.data) > 0 {
        h.siftDown(0)
    }
    return max, nil
}

func (h *MaxHeap) siftUp(i int) {
    for i > 0 {
        p := (i - 1) / 2
        if h.data[p] >= h.data[i] {  // Max-heap: parent ≥ child
            break
        }
        h.data[p], h.data[i] = h.data[i], h.data[p]
        i = p
    }
}

func (h *MaxHeap) siftDown(i int) {
    n := len(h.data)
    for {
        largest := i
        l, r := 2*i+1, 2*i+2
        if l < n && h.data[l] > h.data[largest] {
            largest = l
        }
        if r < n && h.data[r] > h.data[largest] {
            largest = r
        }
        if largest == i {
            break
        }
        h.data[i], h.data[largest] = h.data[largest], h.data[i]
        i = largest
    }
}
```

---

## Go container/heap Interface

```go
import "container/heap"

// Must implement heap.Interface:
// Len() int
// Less(i, j int) bool
// Swap(i, j int)
// Push(x interface{})
// Pop() interface{}

// Min-Heap of ints
type IntMinHeap []int

func (h IntMinHeap) Len() int           { return len(h) }
func (h IntMinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h IntMinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *IntMinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *IntMinHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

// Usage
h := &IntMinHeap{5, 3, 1, 4, 2}
heap.Init(h)                    // Build heap in O(n)
heap.Push(h, 0)                 // Push O(log n)
min := heap.Pop(h).(int)        // Pop O(log n)
fmt.Println((*h)[0])            // Peek O(1)
```

### Custom Struct Priority Queue
```go
type Task struct {
    Name     string
    Priority int
}

type TaskQueue []*Task

func (tq TaskQueue) Len() int            { return len(tq) }
func (tq TaskQueue) Less(i, j int) bool  { return tq[i].Priority > tq[j].Priority } // Max priority first
func (tq TaskQueue) Swap(i, j int)       { tq[i], tq[j] = tq[j], tq[i] }
func (tq *TaskQueue) Push(x interface{}) { *tq = append(*tq, x.(*Task)) }
func (tq *TaskQueue) Pop() interface{} {
    old := *tq
    n := len(old)
    task := old[n-1]
    *tq = old[:n-1]
    return task
}

// Usage
pq := &TaskQueue{}
heap.Init(pq)
heap.Push(pq, &Task{"low", 1})
heap.Push(pq, &Task{"high", 10})
heap.Push(pq, &Task{"medium", 5})
top := heap.Pop(pq).(*Task)  // "high" with priority 10
```

---

## Heapify

### Build Heap from Array — O(n)
```go
// Naive: Push each element — O(n log n)
// Optimal: Heapify from bottom up — O(n)

func buildHeap(arr []int) {
    n := len(arr)
    // Start from last non-leaf node
    for i := n/2 - 1; i >= 0; i-- {
        siftDown(arr, i, n)
    }
}

// Why O(n)?
// Nodes at height h: n/2^(h+1)
// Work per node at height h: O(h)
// Total: Σ n/2^(h+1) * h = O(n)
```

### Heapify Visualization
```
Input: [4, 10, 3, 5, 1]

Step 1: Start at index 1 (last non-leaf)
[4, 10, 3, 5, 1]
     ↑ siftDown(1): children are 5,1 → no swap needed

Step 2: siftDown(0)
[4, 10, 3, 5, 1]
 ↑ children are 10,3 → swap with 3
[3, 10, 4, 5, 1]
         ↑ siftDown continues: no children smaller

Result: [1, 4, 3, 5, 10]  ← valid min-heap
```

---

## Heap Sort

```go
func heapSort(arr []int) {
    n := len(arr)

    // Build max-heap
    for i := n/2 - 1; i >= 0; i-- {
        maxSiftDown(arr, i, n)
    }

    // Extract elements one by one
    for i := n - 1; i > 0; i-- {
        arr[0], arr[i] = arr[i], arr[0]  // Move max to end
        maxSiftDown(arr, 0, i)            // Restore heap
    }
}

func maxSiftDown(arr []int, i, n int) {
    for {
        largest := i
        l, r := 2*i+1, 2*i+2
        if l < n && arr[l] > arr[largest] {
            largest = l
        }
        if r < n && arr[r] > arr[largest] {
            largest = r
        }
        if largest == i {
            break
        }
        arr[i], arr[largest] = arr[largest], arr[i]
        i = largest
    }
}

// Time: O(n log n), Space: O(1) — in-place
```

---

## Priority Queue Patterns

### K Largest Elements
```go
func kLargest(nums []int, k int) []int {
    h := &IntMinHeap{}
    heap.Init(h)

    for _, num := range nums {
        heap.Push(h, num)
        if h.Len() > k {
            heap.Pop(h)  // Remove smallest
        }
    }

    result := make([]int, k)
    for i := k - 1; i >= 0; i-- {
        result[i] = heap.Pop(h).(int)
    }
    return result
}
// Time: O(n log k), Space: O(k)
```

### K Smallest Elements
```go
func kSmallest(nums []int, k int) []int {
    h := &IntMaxHeap{}  // Max-heap of size k
    heap.Init(h)

    for _, num := range nums {
        heap.Push(h, num)
        if h.Len() > k {
            heap.Pop(h)  // Remove largest
        }
    }

    result := make([]int, k)
    for i := k - 1; i >= 0; i-- {
        result[i] = heap.Pop(h).(int)
    }
    return result
}
```

### Kth Largest Element
```go
func findKthLargest(nums []int, k int) int {
    h := &IntMinHeap{}
    heap.Init(h)

    for _, num := range nums {
        heap.Push(h, num)
        if h.Len() > k {
            heap.Pop(h)
        }
    }

    return (*h)[0]  // Root of min-heap = kth largest
}
// Time: O(n log k)
```

### Merge K Sorted Arrays
```go
type heapItem struct {
    val, row, col int
}

type mergeHeap []heapItem

func (h mergeHeap) Len() int            { return len(h) }
func (h mergeHeap) Less(i, j int) bool  { return h[i].val < h[j].val }
func (h mergeHeap) Swap(i, j int)       { h[i], h[j] = h[j], h[i] }
func (h *mergeHeap) Push(x interface{}) { *h = append(*h, x.(heapItem)) }
func (h *mergeHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

func mergeKArrays(arrays [][]int) []int {
    h := &mergeHeap{}
    heap.Init(h)

    for i, arr := range arrays {
        if len(arr) > 0 {
            heap.Push(h, heapItem{arr[0], i, 0})
        }
    }

    result := []int{}
    for h.Len() > 0 {
        item := heap.Pop(h).(heapItem)
        result = append(result, item.val)
        if item.col+1 < len(arrays[item.row]) {
            heap.Push(h, heapItem{
                arrays[item.row][item.col+1],
                item.row,
                item.col + 1,
            })
        }
    }
    return result
}
```

---

## Performance Characteristics

### Time Complexity
| Operation | Time |
|-----------|------|
| Push | O(log n) |
| Pop | O(log n) |
| Peek | O(1) |
| Build heap | O(n) |
| Heap sort | O(n log n) |
| Search | O(n) |

### Space Complexity
- O(n) for storing n elements
- O(1) extra space for operations (in-place)

### Comparison with Other Structures
| Structure | Insert | Delete Min | Peek Min |
|-----------|--------|------------|----------|
| Sorted Array | O(n) | O(1) | O(1) |
| Unsorted Array | O(1) | O(n) | O(n) |
| BST | O(log n) | O(log n) | O(log n) |
| Min-Heap | O(log n) | O(log n) | O(1) |

---

## Common Patterns

### Running Median (Two Heaps)
```go
type MedianFinder struct {
    lower *IntMaxHeap  // Max-heap for lower half
    upper *IntMinHeap  // Min-heap for upper half
}

func (mf *MedianFinder) AddNum(num int) {
    heap.Push(mf.lower, num)

    // Balance: lower max ≤ upper min
    if mf.upper.Len() > 0 && (*mf.lower)[0] > (*mf.upper)[0] {
        heap.Push(mf.upper, heap.Pop(mf.lower))
    }

    // Balance sizes: lower can have at most 1 more than upper
    if mf.lower.Len() > mf.upper.Len()+1 {
        heap.Push(mf.upper, heap.Pop(mf.lower))
    } else if mf.upper.Len() > mf.lower.Len() {
        heap.Push(mf.lower, heap.Pop(mf.upper))
    }
}

func (mf *MedianFinder) FindMedian() float64 {
    if mf.lower.Len() > mf.upper.Len() {
        return float64((*mf.lower)[0])
    }
    return float64((*mf.lower)[0]+(*mf.upper)[0]) / 2.0
}
```

### Task Scheduler
```go
func leastInterval(tasks []byte, n int) int {
    freq := make([]int, 26)
    for _, t := range tasks {
        freq[t-'A']++
    }

    h := &IntMaxHeap{}
    for _, f := range freq {
        if f > 0 {
            heap.Push(h, f)
        }
    }
    heap.Init(h)

    time := 0
    for h.Len() > 0 {
        cycle := n + 1
        temp := []int{}

        for cycle > 0 && h.Len() > 0 {
            f := heap.Pop(h).(int)
            if f > 1 {
                temp = append(temp, f-1)
            }
            cycle--
            time++
        }

        for _, f := range temp {
            heap.Push(h, f)
        }

        if h.Len() > 0 {
            time += cycle  // Idle time
        }
    }
    return time
}
```

---

## Interview Problems

### 1. Top K Frequent Elements
```go
func topKFrequent(nums []int, k int) []int {
    freq := make(map[int]int)
    for _, n := range nums {
        freq[n]++
    }

    type pair struct{ val, cnt int }
    h := &([]pair{})
    // Min-heap by frequency
    // ... (implement heap.Interface)

    for val, cnt := range freq {
        heap.Push(h, pair{val, cnt})
        if len(*h) > k {
            heap.Pop(h)
        }
    }

    result := make([]int, k)
    for i := k - 1; i >= 0; i-- {
        result[i] = heap.Pop(h).(pair).val
    }
    return result
}
```

### 2. Reorganize String
```go
func reorganizeString(s string) string {
    freq := make([]int, 26)
    for _, c := range s {
        freq[c-'a']++
    }

    h := &IntMaxHeap{}
    for i, f := range freq {
        if f > 0 {
            heap.Push(h, i*100+f)  // Encode char and freq
        }
    }
    heap.Init(h)

    result := []byte{}
    for h.Len() >= 2 {
        first := heap.Pop(h).(int)
        second := heap.Pop(h).(int)
        result = append(result, byte(first/100+'a'))
        result = append(result, byte(second/100+'a'))
        if first%100-1 > 0 {
            heap.Push(h, first-1)
        }
        if second%100-1 > 0 {
            heap.Push(h, second-1)
        }
    }

    if h.Len() > 0 {
        last := heap.Pop(h).(int)
        if last%100 > 1 {
            return ""
        }
        result = append(result, byte(last/100+'a'))
    }
    return string(result)
}
```

### 3. Find Median from Data Stream
```
Use two heaps approach (see Running Median above)
- Lower half: max-heap
- Upper half: min-heap
- Keep sizes balanced
- Median = top of larger heap or average of both tops
```

---

## Summary

### Key Concepts
1. Complete binary tree stored as array
2. Heap property: parent ≤ children (min) or parent ≥ children (max)
3. Root always contains min/max — O(1) peek
4. Sift up after insert, sift down after delete
5. Build heap in O(n) using bottom-up heapify

### When to Use
- Need repeated access to min/max element
- Priority-based processing
- K largest/smallest elements
- Merge K sorted sequences
- Running median (two heaps)

### Go-specific
- Use `container/heap` package
- Implement `heap.Interface` (5 methods)
- `heap.Init` for O(n) build
- `heap.Push` / `heap.Pop` for O(log n) operations

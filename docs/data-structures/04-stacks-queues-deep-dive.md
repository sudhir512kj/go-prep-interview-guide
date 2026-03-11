# Stacks and Queues Deep Dive

## Table of Contents
- [Stack Fundamentals](#stack-fundamentals)
- [Stack Internal Implementation](#stack-internal-implementation)
- [Stack Operations](#stack-operations)
- [Stack Using Slice](#stack-using-slice)
- [Stack Using Linked List](#stack-using-linked-list)
- [Queue Fundamentals](#queue-fundamentals)
- [Queue Internal Implementation](#queue-internal-implementation)
- [Queue Using Slice](#queue-using-slice)
- [Queue Using Linked List](#queue-using-linked-list)
- [Circular Queue](#circular-queue)
- [Deque (Double-Ended Queue)](#deque-double-ended-queue)
- [Priority Queue](#priority-queue)
- [Monotonic Stack](#monotonic-stack)
- [Monotonic Queue](#monotonic-queue)
- [Performance Characteristics](#performance-characteristics)
- [Common Patterns](#common-patterns)
- [Interview Problems](#interview-problems)

---

## Stack Fundamentals

### Definition
A stack is a LIFO (Last In, First Out) data structure. The last element inserted is the first to be removed.

```
Push → [5] [4] [3] [2] [1] ← Pop
        Top                  Bottom
```

### Real-world Analogies
- Stack of plates
- Browser back button history
- Undo/redo operations
- Function call stack

### Core Operations
- **Push**: Add element to top — O(1)
- **Pop**: Remove element from top — O(1)
- **Peek/Top**: View top element without removing — O(1)
- **IsEmpty**: Check if stack is empty — O(1)
- **Size**: Number of elements — O(1)

---

## Stack Internal Implementation

### Memory Model
```
Stack state after pushing 1, 2, 3, 4, 5:

Index:  0    1    2    3    4
       [1]  [2]  [3]  [4]  [5]
                              ↑
                             top (index 4)

Push 6:
       [1]  [2]  [3]  [4]  [5]  [6]
                                  ↑
                                 top (index 5)

Pop:
       [1]  [2]  [3]  [4]  [5]
                              ↑
                             top (index 4), returns 6
```

### Call Stack Visualization
```
func main() {
    a()
}
func a() { b() }
func b() { c() }
func c() { /* base */ }

Call Stack:
┌──────────┐ ← top
│  c()     │
├──────────┤
│  b()     │
├──────────┤
│  a()     │
├──────────┤
│  main()  │
└──────────┘ ← bottom
```

---

## Stack Operations

### Generic Stack Implementation
```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, error) {
    var zero T
    if s.IsEmpty() {
        return zero, errors.New("stack is empty")
    }
    top := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return top, nil
}

func (s *Stack[T]) Peek() (T, error) {
    var zero T
    if s.IsEmpty() {
        return zero, errors.New("stack is empty")
    }
    return s.items[len(s.items)-1], nil
}

func (s *Stack[T]) IsEmpty() bool {
    return len(s.items) == 0
}

func (s *Stack[T]) Size() int {
    return len(s.items)
}
```

---

## Stack Using Slice

### Simple Int Stack
```go
type IntStack struct {
    items []int
}

func (s *IntStack) Push(v int)        { s.items = append(s.items, v) }
func (s *IntStack) Pop() int          { n := len(s.items) - 1; v := s.items[n]; s.items = s.items[:n]; return v }
func (s *IntStack) Peek() int         { return s.items[len(s.items)-1] }
func (s *IntStack) IsEmpty() bool     { return len(s.items) == 0 }
func (s *IntStack) Size() int         { return len(s.items) }
```

### Thread-Safe Stack
```go
type SafeStack[T any] struct {
    mu    sync.Mutex
    items []T
}

func (s *SafeStack[T]) Push(item T) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.items = append(s.items, item)
}

func (s *SafeStack[T]) Pop() (T, bool) {
    s.mu.Lock()
    defer s.mu.Unlock()
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    n := len(s.items) - 1
    item := s.items[n]
    s.items = s.items[:n]
    return item, true
}
```

---

## Stack Using Linked List

```go
type stackNode struct {
    val  int
    next *stackNode
}

type LinkedStack struct {
    top  *stackNode
    size int
}

func (s *LinkedStack) Push(val int) {
    s.top = &stackNode{val: val, next: s.top}
    s.size++
}

func (s *LinkedStack) Pop() (int, error) {
    if s.top == nil {
        return 0, errors.New("stack is empty")
    }
    val := s.top.val
    s.top = s.top.next
    s.size--
    return val, nil
}

func (s *LinkedStack) Peek() (int, error) {
    if s.top == nil {
        return 0, errors.New("stack is empty")
    }
    return s.top.val, nil
}

func (s *LinkedStack) IsEmpty() bool { return s.top == nil }
func (s *LinkedStack) Size() int     { return s.size }
```

---

## Queue Fundamentals

### Definition
A queue is a FIFO (First In, First Out) data structure. The first element inserted is the first to be removed.

```
Enqueue → [1] [2] [3] [4] [5] → Dequeue
           Front              Back
```

### Real-world Analogies
- Line at a ticket counter
- Print job queue
- CPU task scheduling
- BFS traversal

### Core Operations
- **Enqueue**: Add element to back — O(1)
- **Dequeue**: Remove element from front — O(1)
- **Front/Peek**: View front element — O(1)
- **IsEmpty**: Check if queue is empty — O(1)
- **Size**: Number of elements — O(1)

---

## Queue Internal Implementation

### Naive Slice Queue (Inefficient)
```go
// Problem: Dequeue shifts all elements O(n)
type NaiveQueue struct {
    items []int
}

func (q *NaiveQueue) Enqueue(v int) {
    q.items = append(q.items, v)  // O(1)
}

func (q *NaiveQueue) Dequeue() int {
    v := q.items[0]
    q.items = q.items[1:]  // O(n) - shifts all elements!
    return v
}
```

### Memory Waste Problem
```
After many enqueue/dequeue operations:

items: [_][_][_][_][_][3][4][5]
                        ↑
                       front (wasted space at beginning)
```

---

## Queue Using Slice

### Efficient Queue with Head Index
```go
type Queue struct {
    items []int
    head  int
}

func (q *Queue) Enqueue(v int) {
    q.items = append(q.items, v)
}

func (q *Queue) Dequeue() (int, error) {
    if q.IsEmpty() {
        return 0, errors.New("queue is empty")
    }
    v := q.items[q.head]
    q.head++
    // Compact when head reaches half
    if q.head > len(q.items)/2 {
        q.items = q.items[q.head:]
        q.head = 0
    }
    return v, nil
}

func (q *Queue) Peek() (int, error) {
    if q.IsEmpty() {
        return 0, errors.New("queue is empty")
    }
    return q.items[q.head], nil
}

func (q *Queue) IsEmpty() bool { return q.head >= len(q.items) }
func (q *Queue) Size() int     { return len(q.items) - q.head }
```

---

## Queue Using Linked List

```go
type queueNode struct {
    val  int
    next *queueNode
}

type LinkedQueue struct {
    front *queueNode
    back  *queueNode
    size  int
}

func (q *LinkedQueue) Enqueue(val int) {
    node := &queueNode{val: val}
    if q.back == nil {
        q.front = node
        q.back = node
    } else {
        q.back.next = node
        q.back = node
    }
    q.size++
}

func (q *LinkedQueue) Dequeue() (int, error) {
    if q.front == nil {
        return 0, errors.New("queue is empty")
    }
    val := q.front.val
    q.front = q.front.next
    if q.front == nil {
        q.back = nil
    }
    q.size--
    return val, nil
}

func (q *LinkedQueue) Peek() (int, error) {
    if q.front == nil {
        return 0, errors.New("queue is empty")
    }
    return q.front.val, nil
}

func (q *LinkedQueue) IsEmpty() bool { return q.front == nil }
func (q *LinkedQueue) Size() int     { return q.size }
```

---

## Circular Queue

### Fixed-size Circular Buffer
```go
type CircularQueue struct {
    items    []int
    front    int
    back     int
    size     int
    capacity int
}

func NewCircularQueue(capacity int) *CircularQueue {
    return &CircularQueue{
        items:    make([]int, capacity),
        capacity: capacity,
    }
}

func (q *CircularQueue) Enqueue(val int) error {
    if q.IsFull() {
        return errors.New("queue is full")
    }
    q.items[q.back] = val
    q.back = (q.back + 1) % q.capacity
    q.size++
    return nil
}

func (q *CircularQueue) Dequeue() (int, error) {
    if q.IsEmpty() {
        return 0, errors.New("queue is empty")
    }
    val := q.items[q.front]
    q.front = (q.front + 1) % q.capacity
    q.size--
    return val, nil
}

func (q *CircularQueue) IsFull() bool  { return q.size == q.capacity }
func (q *CircularQueue) IsEmpty() bool { return q.size == 0 }
func (q *CircularQueue) Size() int     { return q.size }
```

### Circular Queue Visualization
```
Capacity = 5, after Enqueue(1,2,3) and Dequeue():

Before Dequeue:
index:  0    1    2    3    4
       [1]  [2]  [3]  [ ]  [ ]
        ↑              ↑
       front          back

After Dequeue (removes 1):
index:  0    1    2    3    4
       [ ]  [2]  [3]  [ ]  [ ]
             ↑         ↑
            front     back

After Enqueue(4,5,6) — wraps around:
index:  0    1    2    3    4
       [6]  [2]  [3]  [4]  [5]
        ↑    ↑
       back front
```

---

## Deque (Double-Ended Queue)

```go
type Deque struct {
    items []int
}

func (d *Deque) PushFront(val int) {
    d.items = append([]int{val}, d.items...)
}

func (d *Deque) PushBack(val int) {
    d.items = append(d.items, val)
}

func (d *Deque) PopFront() (int, error) {
    if d.IsEmpty() {
        return 0, errors.New("deque is empty")
    }
    val := d.items[0]
    d.items = d.items[1:]
    return val, nil
}

func (d *Deque) PopBack() (int, error) {
    if d.IsEmpty() {
        return 0, errors.New("deque is empty")
    }
    n := len(d.items) - 1
    val := d.items[n]
    d.items = d.items[:n]
    return val, nil
}

func (d *Deque) PeekFront() (int, error) {
    if d.IsEmpty() {
        return 0, errors.New("deque is empty")
    }
    return d.items[0], nil
}

func (d *Deque) PeekBack() (int, error) {
    if d.IsEmpty() {
        return 0, errors.New("deque is empty")
    }
    return d.items[len(d.items)-1], nil
}

func (d *Deque) IsEmpty() bool { return len(d.items) == 0 }
func (d *Deque) Size() int     { return len(d.items) }
```

### Efficient Deque Using Doubly Linked List
```go
type dequeNode struct {
    val        int
    prev, next *dequeNode
}

type LinkedDeque struct {
    front, back *dequeNode
    size        int
}

func (d *LinkedDeque) PushFront(val int) {
    node := &dequeNode{val: val, next: d.front}
    if d.front != nil {
        d.front.prev = node
    } else {
        d.back = node
    }
    d.front = node
    d.size++
}

func (d *LinkedDeque) PushBack(val int) {
    node := &dequeNode{val: val, prev: d.back}
    if d.back != nil {
        d.back.next = node
    } else {
        d.front = node
    }
    d.back = node
    d.size++
}

func (d *LinkedDeque) PopFront() (int, error) {
    if d.front == nil {
        return 0, errors.New("deque is empty")
    }
    val := d.front.val
    d.front = d.front.next
    if d.front != nil {
        d.front.prev = nil
    } else {
        d.back = nil
    }
    d.size--
    return val, nil
}

func (d *LinkedDeque) PopBack() (int, error) {
    if d.back == nil {
        return 0, errors.New("deque is empty")
    }
    val := d.back.val
    d.back = d.back.prev
    if d.back != nil {
        d.back.next = nil
    } else {
        d.front = nil
    }
    d.size--
    return val, nil
}
```

---

## Priority Queue

### Using container/heap
```go
import "container/heap"

// Min-Heap
type MinHeap []int

func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }

func (h *MinHeap) Push(x interface{}) {
    *h = append(*h, x.(int))
}

func (h *MinHeap) Pop() interface{} {
    old := *h
    n := len(old)
    x := old[n-1]
    *h = old[:n-1]
    return x
}

// Usage
pq := &MinHeap{3, 1, 4, 1, 5, 9}
heap.Init(pq)
heap.Push(pq, 2)
min := heap.Pop(pq).(int)  // 1
```

### Custom Priority Queue
```go
type Item struct {
    value    string
    priority int
    index    int
}

type PriorityQueue []*Item

func (pq PriorityQueue) Len() int { return len(pq) }
func (pq PriorityQueue) Less(i, j int) bool {
    return pq[i].priority > pq[j].priority  // Max-heap
}
func (pq PriorityQueue) Swap(i, j int) {
    pq[i], pq[j] = pq[j], pq[i]
    pq[i].index = i
    pq[j].index = j
}

func (pq *PriorityQueue) Push(x interface{}) {
    n := len(*pq)
    item := x.(*Item)
    item.index = n
    *pq = append(*pq, item)
}

func (pq *PriorityQueue) Pop() interface{} {
    old := *pq
    n := len(old)
    item := old[n-1]
    old[n-1] = nil
    item.index = -1
    *pq = old[:n-1]
    return item
}

func (pq *PriorityQueue) Update(item *Item, value string, priority int) {
    item.value = value
    item.priority = priority
    heap.Fix(pq, item.index)
}
```

---

## Monotonic Stack

### Definition
A stack that maintains elements in monotonically increasing or decreasing order.

### Monotonic Increasing Stack
```go
// Maintains increasing order from bottom to top
func nextGreaterElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }
    
    stack := []int{}  // Stores indices
    
    for i := 0; i < n; i++ {
        // Pop elements smaller than current
        for len(stack) > 0 && nums[stack[len(stack)-1]] < nums[i] {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = nums[i]
        }
        stack = append(stack, i)
    }
    
    return result
}

// Example: [2, 1, 2, 4, 3]
// Result:  [4, 2, 4,-1,-1]
```

### Monotonic Decreasing Stack
```go
// Maintains decreasing order from bottom to top
func nextSmallerElement(nums []int) []int {
    n := len(nums)
    result := make([]int, n)
    for i := range result {
        result[i] = -1
    }
    
    stack := []int{}
    
    for i := 0; i < n; i++ {
        for len(stack) > 0 && nums[stack[len(stack)-1]] > nums[i] {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = nums[i]
        }
        stack = append(stack, i)
    }
    
    return result
}
```

### Largest Rectangle in Histogram
```go
func largestRectangleArea(heights []int) int {
    stack := []int{}
    maxArea := 0
    heights = append(heights, 0)  // Sentinel
    
    for i, h := range heights {
        for len(stack) > 0 && heights[stack[len(stack)-1]] > h {
            height := heights[stack[len(stack)-1]]
            stack = stack[:len(stack)-1]
            
            width := i
            if len(stack) > 0 {
                width = i - stack[len(stack)-1] - 1
            }
            maxArea = max(maxArea, height*width)
        }
        stack = append(stack, i)
    }
    
    return maxArea
}
```

---

## Monotonic Queue

### Sliding Window Maximum
```go
func maxSlidingWindow(nums []int, k int) []int {
    deque := []int{}  // Stores indices, decreasing values
    result := []int{}
    
    for i, num := range nums {
        // Remove elements outside window
        for len(deque) > 0 && deque[0] < i-k+1 {
            deque = deque[1:]
        }
        
        // Remove smaller elements from back
        for len(deque) > 0 && nums[deque[len(deque)-1]] < num {
            deque = deque[:len(deque)-1]
        }
        
        deque = append(deque, i)
        
        // Add to result when window is full
        if i >= k-1 {
            result = append(result, nums[deque[0]])
        }
    }
    
    return result
}

// Example: nums=[1,3,-1,-3,5,3,6,7], k=3
// Result: [3,3,5,5,6,7]
```

---

## Performance Characteristics

### Time Complexity
| Operation | Stack (Slice) | Stack (List) | Queue (List) | Circular Queue |
|-----------|--------------|--------------|--------------|----------------|
| Push/Enqueue | O(1) amortized | O(1) | O(1) | O(1) |
| Pop/Dequeue | O(1) | O(1) | O(1) | O(1) |
| Peek | O(1) | O(1) | O(1) | O(1) |
| Search | O(n) | O(n) | O(n) | O(n) |
| Size | O(1) | O(1) | O(1) | O(1) |

### Space Complexity
- Stack/Queue with slice: O(n)
- Stack/Queue with linked list: O(n) + pointer overhead
- Circular queue: O(capacity) — fixed

---

## Common Patterns

### Balanced Parentheses
```go
func isValid(s string) bool {
    stack := []rune{}
    pairs := map[rune]rune{')': '(', '}': '{', ']': '['}
    
    for _, ch := range s {
        if ch == '(' || ch == '{' || ch == '[' {
            stack = append(stack, ch)
        } else {
            if len(stack) == 0 || stack[len(stack)-1] != pairs[ch] {
                return false
            }
            stack = stack[:len(stack)-1]
        }
    }
    
    return len(stack) == 0
}
```

### Evaluate Postfix Expression
```go
func evalRPN(tokens []string) int {
    stack := []int{}
    
    for _, token := range tokens {
        switch token {
        case "+", "-", "*", "/":
            b, a := stack[len(stack)-1], stack[len(stack)-2]
            stack = stack[:len(stack)-2]
            switch token {
            case "+": stack = append(stack, a+b)
            case "-": stack = append(stack, a-b)
            case "*": stack = append(stack, a*b)
            case "/": stack = append(stack, a/b)
            }
        default:
            n, _ := strconv.Atoi(token)
            stack = append(stack, n)
        }
    }
    
    return stack[0]
}
```

### BFS Using Queue
```go
func bfs(root *TreeNode) []int {
    if root == nil {
        return nil
    }
    
    result := []int{}
    queue := []*TreeNode{root}
    
    for len(queue) > 0 {
        node := queue[0]
        queue = queue[1:]
        result = append(result, node.Val)
        
        if node.Left != nil {
            queue = append(queue, node.Left)
        }
        if node.Right != nil {
            queue = append(queue, node.Right)
        }
    }
    
    return result
}
```

### Min Stack
```go
type MinStack struct {
    stack    []int
    minStack []int
}

func (s *MinStack) Push(val int) {
    s.stack = append(s.stack, val)
    if len(s.minStack) == 0 || val <= s.minStack[len(s.minStack)-1] {
        s.minStack = append(s.minStack, val)
    }
}

func (s *MinStack) Pop() {
    top := s.stack[len(s.stack)-1]
    s.stack = s.stack[:len(s.stack)-1]
    if top == s.minStack[len(s.minStack)-1] {
        s.minStack = s.minStack[:len(s.minStack)-1]
    }
}

func (s *MinStack) Top() int    { return s.stack[len(s.stack)-1] }
func (s *MinStack) GetMin() int { return s.minStack[len(s.minStack)-1] }
```

---

## Interview Problems

### 1. Daily Temperatures
```go
// Find days until warmer temperature
func dailyTemperatures(temps []int) []int {
    result := make([]int, len(temps))
    stack := []int{}  // Indices
    
    for i, t := range temps {
        for len(stack) > 0 && temps[stack[len(stack)-1]] < t {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = i - idx
        }
        stack = append(stack, i)
    }
    
    return result
}
```

### 2. Implement Queue Using Two Stacks
```go
type MyQueue struct {
    inbox  []int
    outbox []int
}

func (q *MyQueue) Push(x int) {
    q.inbox = append(q.inbox, x)
}

func (q *MyQueue) transfer() {
    if len(q.outbox) == 0 {
        for len(q.inbox) > 0 {
            n := len(q.inbox) - 1
            q.outbox = append(q.outbox, q.inbox[n])
            q.inbox = q.inbox[:n]
        }
    }
}

func (q *MyQueue) Pop() int {
    q.transfer()
    n := len(q.outbox) - 1
    val := q.outbox[n]
    q.outbox = q.outbox[:n]
    return val
}

func (q *MyQueue) Peek() int {
    q.transfer()
    return q.outbox[len(q.outbox)-1]
}

func (q *MyQueue) Empty() bool {
    return len(q.inbox) == 0 && len(q.outbox) == 0
}
```

### 3. Implement Stack Using Two Queues
```go
type MyStack struct {
    q1, q2 []int
}

func (s *MyStack) Push(x int) {
    s.q2 = append(s.q2, x)
    for len(s.q1) > 0 {
        s.q2 = append(s.q2, s.q1[0])
        s.q1 = s.q1[1:]
    }
    s.q1, s.q2 = s.q2, s.q1
}

func (s *MyStack) Pop() int {
    val := s.q1[0]
    s.q1 = s.q1[1:]
    return val
}

func (s *MyStack) Top() int   { return s.q1[0] }
func (s *MyStack) Empty() bool { return len(s.q1) == 0 }
```

### 4. Trapping Rain Water
```go
func trap(height []int) int {
    stack := []int{}
    water := 0
    
    for i, h := range height {
        for len(stack) > 0 && height[stack[len(stack)-1]] < h {
            bottom := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            
            if len(stack) == 0 {
                break
            }
            
            left := stack[len(stack)-1]
            width := i - left - 1
            boundedHeight := min(height[left], h) - height[bottom]
            water += width * boundedHeight
        }
        stack = append(stack, i)
    }
    
    return water
}
```

---

## Summary

### Stack
- LIFO — last in, first out
- Use for: DFS, backtracking, expression evaluation, undo operations
- Key operations: Push O(1), Pop O(1), Peek O(1)

### Queue
- FIFO — first in, first out
- Use for: BFS, task scheduling, buffering
- Key operations: Enqueue O(1), Dequeue O(1), Peek O(1)

### Monotonic Stack/Queue
- Maintain sorted order for range queries
- Next greater/smaller element problems
- Sliding window maximum/minimum

### When to Use
| Problem Type | Data Structure |
|---|---|
| DFS / backtracking | Stack |
| BFS / level order | Queue |
| Undo/redo | Stack |
| Task scheduling | Priority Queue |
| Sliding window max | Monotonic Queue |
| Next greater element | Monotonic Stack |

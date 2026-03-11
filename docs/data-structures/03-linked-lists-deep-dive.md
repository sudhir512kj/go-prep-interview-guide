# Linked Lists Deep Dive

## Table of Contents
- [Linked List Fundamentals](#linked-list-fundamentals)
- [Singly Linked List](#singly-linked-list)
- [Doubly Linked List](#doubly-linked-list)
- [Circular Linked List](#circular-linked-list)
- [Memory Layout](#memory-layout)
- [Common Operations](#common-operations)
- [Two Pointer Techniques](#two-pointer-techniques)
- [Cycle Detection](#cycle-detection)
- [Reversal Techniques](#reversal-techniques)
- [Merge Operations](#merge-operations)
- [Performance Characteristics](#performance-characteristics)
- [Common Patterns](#common-patterns)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)
- [Interview Problems](#interview-problems)

---

## Linked List Fundamentals

### Definition
A linked list is a linear data structure where elements are stored in nodes, each pointing to the next node.

### Advantages
- Dynamic size
- Efficient insertion/deletion at beginning (O(1))
- No memory waste from unused capacity
- Easy to implement stacks and queues

### Disadvantages
- No random access (O(n) to access element)
- Extra memory for pointers
- Poor cache locality
- Cannot use binary search

---

## Singly Linked List

### Node Structure
```go
type Node struct {
    Data int
    Next *Node
}

type LinkedList struct {
    Head *Node
    Size int
}
```

### Basic Operations

#### Insert at Beginning
```go
func (ll *LinkedList) InsertAtBeginning(data int) {
    newNode := &Node{Data: data, Next: ll.Head}
    ll.Head = newNode
    ll.Size++
}

// Time: O(1), Space: O(1)
```

#### Insert at End
```go
func (ll *LinkedList) InsertAtEnd(data int) {
    newNode := &Node{Data: data}
    
    if ll.Head == nil {
        ll.Head = newNode
        ll.Size++
        return
    }
    
    current := ll.Head
    for current.Next != nil {
        current = current.Next
    }
    current.Next = newNode
    ll.Size++
}

// Time: O(n), Space: O(1)
```

#### Insert at Position
```go
func (ll *LinkedList) InsertAt(position, data int) error {
    if position < 0 || position > ll.Size {
        return errors.New("invalid position")
    }
    
    if position == 0 {
        ll.InsertAtBeginning(data)
        return nil
    }
    
    newNode := &Node{Data: data}
    current := ll.Head
    
    for i := 0; i < position-1; i++ {
        current = current.Next
    }
    
    newNode.Next = current.Next
    current.Next = newNode
    ll.Size++
    return nil
}

// Time: O(n), Space: O(1)
```

#### Delete at Beginning
```go
func (ll *LinkedList) DeleteAtBeginning() error {
    if ll.Head == nil {
        return errors.New("list is empty")
    }
    
    ll.Head = ll.Head.Next
    ll.Size--
    return nil
}

// Time: O(1), Space: O(1)
```

#### Delete at End
```go
func (ll *LinkedList) DeleteAtEnd() error {
    if ll.Head == nil {
        return errors.New("list is empty")
    }
    
    if ll.Head.Next == nil {
        ll.Head = nil
        ll.Size--
        return nil
    }
    
    current := ll.Head
    for current.Next.Next != nil {
        current = current.Next
    }
    current.Next = nil
    ll.Size--
    return nil
}

// Time: O(n), Space: O(1)
```

#### Delete by Value
```go
func (ll *LinkedList) DeleteByValue(data int) error {
    if ll.Head == nil {
        return errors.New("list is empty")
    }
    
    // If head needs to be deleted
    if ll.Head.Data == data {
        ll.Head = ll.Head.Next
        ll.Size--
        return nil
    }
    
    current := ll.Head
    for current.Next != nil {
        if current.Next.Data == data {
            current.Next = current.Next.Next
            ll.Size--
            return nil
        }
        current = current.Next
    }
    
    return errors.New("value not found")
}

// Time: O(n), Space: O(1)
```

#### Search
```go
func (ll *LinkedList) Search(data int) *Node {
    current := ll.Head
    for current != nil {
        if current.Data == data {
            return current
        }
        current = current.Next
    }
    return nil
}

// Time: O(n), Space: O(1)
```

#### Display
```go
func (ll *LinkedList) Display() {
    current := ll.Head
    for current != nil {
        fmt.Printf("%d -> ", current.Data)
        current = current.Next
    }
    fmt.Println("nil")
}
```

---

## Doubly Linked List

### Node Structure
```go
type DNode struct {
    Data int
    Next *DNode
    Prev *DNode
}

type DoublyLinkedList struct {
    Head *DNode
    Tail *DNode
    Size int
}
```

### Basic Operations

#### Insert at Beginning
```go
func (dll *DoublyLinkedList) InsertAtBeginning(data int) {
    newNode := &DNode{Data: data}
    
    if dll.Head == nil {
        dll.Head = newNode
        dll.Tail = newNode
    } else {
        newNode.Next = dll.Head
        dll.Head.Prev = newNode
        dll.Head = newNode
    }
    dll.Size++
}

// Time: O(1), Space: O(1)
```

#### Insert at End
```go
func (dll *DoublyLinkedList) InsertAtEnd(data int) {
    newNode := &DNode{Data: data}
    
    if dll.Tail == nil {
        dll.Head = newNode
        dll.Tail = newNode
    } else {
        newNode.Prev = dll.Tail
        dll.Tail.Next = newNode
        dll.Tail = newNode
    }
    dll.Size++
}

// Time: O(1), Space: O(1)
```

#### Delete at Beginning
```go
func (dll *DoublyLinkedList) DeleteAtBeginning() error {
    if dll.Head == nil {
        return errors.New("list is empty")
    }
    
    if dll.Head == dll.Tail {
        dll.Head = nil
        dll.Tail = nil
    } else {
        dll.Head = dll.Head.Next
        dll.Head.Prev = nil
    }
    dll.Size--
    return nil
}

// Time: O(1), Space: O(1)
```

#### Delete at End
```go
func (dll *DoublyLinkedList) DeleteAtEnd() error {
    if dll.Tail == nil {
        return errors.New("list is empty")
    }
    
    if dll.Head == dll.Tail {
        dll.Head = nil
        dll.Tail = nil
    } else {
        dll.Tail = dll.Tail.Prev
        dll.Tail.Next = nil
    }
    dll.Size--
    return nil
}

// Time: O(1), Space: O(1)
```

#### Reverse Traversal
```go
func (dll *DoublyLinkedList) DisplayReverse() {
    current := dll.Tail
    for current != nil {
        fmt.Printf("%d -> ", current.Data)
        current = current.Prev
    }
    fmt.Println("nil")
}
```

---

## Circular Linked List

### Singly Circular
```go
type CircularList struct {
    Head *Node
    Size int
}

func (cl *CircularList) InsertAtEnd(data int) {
    newNode := &Node{Data: data}
    
    if cl.Head == nil {
        cl.Head = newNode
        newNode.Next = newNode  // Points to itself
    } else {
        current := cl.Head
        for current.Next != cl.Head {
            current = current.Next
        }
        current.Next = newNode
        newNode.Next = cl.Head
    }
    cl.Size++
}

func (cl *CircularList) Display() {
    if cl.Head == nil {
        return
    }
    
    current := cl.Head
    for {
        fmt.Printf("%d -> ", current.Data)
        current = current.Next
        if current == cl.Head {
            break
        }
    }
    fmt.Println("(back to head)")
}
```

### Doubly Circular
```go
type CircularDNode struct {
    Data int
    Next *CircularDNode
    Prev *CircularDNode
}

type DoublyCircularList struct {
    Head *CircularDNode
    Size int
}

func (dcl *DoublyCircularList) InsertAtEnd(data int) {
    newNode := &CircularDNode{Data: data}
    
    if dcl.Head == nil {
        dcl.Head = newNode
        newNode.Next = newNode
        newNode.Prev = newNode
    } else {
        tail := dcl.Head.Prev
        tail.Next = newNode
        newNode.Prev = tail
        newNode.Next = dcl.Head
        dcl.Head.Prev = newNode
    }
    dcl.Size++
}
```

---

## Memory Layout

### Singly Linked List
```
Head → [Data|Next•] → [Data|Next•] → [Data|Next•] → nil
        Node 1         Node 2         Node 3

Memory (non-contiguous):
Address 0x1000: [10 | 0x2000]
Address 0x2000: [20 | 0x3000]
Address 0x3000: [30 | nil   ]
```

### Doubly Linked List
```
Head → [Prev|Data|Next] ⇄ [Prev|Data|Next] ⇄ [Prev|Data|Next] ← Tail
       nil  10   •          •    20   •          •    30   nil

Memory:
Address 0x1000: [nil   | 10 | 0x2000]
Address 0x2000: [0x1000| 20 | 0x3000]
Address 0x3000: [0x2000| 30 | nil   ]
```

### Array vs Linked List Memory
```
Array: Contiguous memory
[10][20][30][40][50]
 ↑
Base address, all elements adjacent

Linked List: Scattered memory
[10|•]     [20|•]     [30|•]
  ↓          ↓          ↓
Random addresses, connected by pointers
```

---

## Common Operations

### Get Length
```go
func (ll *LinkedList) Length() int {
    count := 0
    current := ll.Head
    for current != nil {
        count++
        current = current.Next
    }
    return count
}

// Time: O(n), Space: O(1)
// Better: Maintain size field (O(1))
```

### Get Nth Node
```go
func (ll *LinkedList) GetNth(n int) (*Node, error) {
    if n < 0 {
        return nil, errors.New("invalid index")
    }
    
    current := ll.Head
    for i := 0; i < n && current != nil; i++ {
        current = current.Next
    }
    
    if current == nil {
        return nil, errors.New("index out of bounds")
    }
    return current, nil
}

// Time: O(n), Space: O(1)
```

### Get Middle Node
```go
func (ll *LinkedList) GetMiddle() *Node {
    if ll.Head == nil {
        return nil
    }
    
    slow := ll.Head
    fast := ll.Head
    
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    
    return slow
}

// Time: O(n), Space: O(1)
// Fast pointer moves 2x, slow pointer at middle when fast reaches end
```

### Get Nth from End
```go
func (ll *LinkedList) GetNthFromEnd(n int) (*Node, error) {
    if n <= 0 {
        return nil, errors.New("invalid n")
    }
    
    first := ll.Head
    second := ll.Head
    
    // Move first pointer n steps ahead
    for i := 0; i < n; i++ {
        if first == nil {
            return nil, errors.New("n is larger than list size")
        }
        first = first.Next
    }
    
    // Move both pointers until first reaches end
    for first != nil {
        first = first.Next
        second = second.Next
    }
    
    return second, nil
}

// Time: O(n), Space: O(1)
```

---

## Two Pointer Techniques

### Fast and Slow Pointers
```go
// Find middle element
func findMiddle(head *Node) *Node {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    return slow
}

// Check if length is even or odd
func isLengthEven(head *Node) bool {
    fast := head
    for fast != nil && fast.Next != nil {
        fast = fast.Next.Next
    }
    return fast == nil  // Even if fast is nil
}
```

### Two Pointers with Gap
```go
// Remove Nth node from end
func removeNthFromEnd(head *Node, n int) *Node {
    dummy := &Node{Next: head}
    first, second := dummy, dummy
    
    // Move first n+1 steps ahead
    for i := 0; i <= n; i++ {
        first = first.Next
    }
    
    // Move both until first reaches end
    for first != nil {
        first = first.Next
        second = second.Next
    }
    
    // Remove node
    second.Next = second.Next.Next
    return dummy.Next
}
```

---

## Cycle Detection

### Floyd's Cycle Detection (Tortoise and Hare)
```go
func hasCycle(head *Node) bool {
    if head == nil {
        return false
    }
    
    slow := head
    fast := head
    
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        
        if slow == fast {
            return true
        }
    }
    
    return false
}

// Time: O(n), Space: O(1)
```

### Find Cycle Start
```go
func detectCycle(head *Node) *Node {
    if head == nil {
        return nil
    }
    
    // Detect cycle
    slow, fast := head, head
    hasCycle := false
    
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        
        if slow == fast {
            hasCycle = true
            break
        }
    }
    
    if !hasCycle {
        return nil
    }
    
    // Find cycle start
    slow = head
    for slow != fast {
        slow = slow.Next
        fast = fast.Next
    }
    
    return slow
}

// Time: O(n), Space: O(1)
```

### Cycle Length
```go
func cycleLength(head *Node) int {
    slow, fast := head, head
    
    // Detect cycle
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        
        if slow == fast {
            // Count cycle length
            count := 1
            current := slow.Next
            for current != slow {
                count++
                current = current.Next
            }
            return count
        }
    }
    
    return 0  // No cycle
}
```

---

## Reversal Techniques

### Iterative Reversal
```go
func reverse(head *Node) *Node {
    var prev *Node
    current := head
    
    for current != nil {
        next := current.Next
        current.Next = prev
        prev = current
        current = next
    }
    
    return prev
}

// Time: O(n), Space: O(1)

// Visualization:
// Before: 1 → 2 → 3 → nil
// After:  nil ← 1 ← 2 ← 3
```

### Recursive Reversal
```go
func reverseRecursive(head *Node) *Node {
    if head == nil || head.Next == nil {
        return head
    }
    
    newHead := reverseRecursive(head.Next)
    head.Next.Next = head
    head.Next = nil
    
    return newHead
}

// Time: O(n), Space: O(n) for recursion stack
```

### Reverse in Groups
```go
func reverseInGroups(head *Node, k int) *Node {
    current := head
    var prev, next *Node
    count := 0
    
    // Reverse first k nodes
    for current != nil && count < k {
        next = current.Next
        current.Next = prev
        prev = current
        current = next
        count++
    }
    
    // Recursively reverse remaining list
    if next != nil {
        head.Next = reverseInGroups(next, k)
    }
    
    return prev
}

// Example: k=3
// Input:  1 → 2 → 3 → 4 → 5 → 6 → 7
// Output: 3 → 2 → 1 → 6 → 5 → 4 → 7
```

### Reverse Between Positions
```go
func reverseBetween(head *Node, left, right int) *Node {
    if head == nil || left == right {
        return head
    }
    
    dummy := &Node{Next: head}
    prev := dummy
    
    // Move to position left-1
    for i := 0; i < left-1; i++ {
        prev = prev.Next
    }
    
    // Reverse from left to right
    current := prev.Next
    for i := 0; i < right-left; i++ {
        next := current.Next
        current.Next = next.Next
        next.Next = prev.Next
        prev.Next = next
    }
    
    return dummy.Next
}

// Example: left=2, right=4
// Input:  1 → 2 → 3 → 4 → 5
// Output: 1 → 4 → 3 → 2 → 5
```

---

## Merge Operations

### Merge Two Sorted Lists
```go
func mergeTwoLists(l1, l2 *Node) *Node {
    dummy := &Node{}
    current := dummy
    
    for l1 != nil && l2 != nil {
        if l1.Data < l2.Data {
            current.Next = l1
            l1 = l1.Next
        } else {
            current.Next = l2
            l2 = l2.Next
        }
        current = current.Next
    }
    
    // Attach remaining nodes
    if l1 != nil {
        current.Next = l1
    } else {
        current.Next = l2
    }
    
    return dummy.Next
}

// Time: O(n+m), Space: O(1)
```

### Merge K Sorted Lists
```go
func mergeKLists(lists []*Node) *Node {
    if len(lists) == 0 {
        return nil
    }
    
    for len(lists) > 1 {
        merged := []*Node{}
        
        for i := 0; i < len(lists); i += 2 {
            l1 := lists[i]
            var l2 *Node
            if i+1 < len(lists) {
                l2 = lists[i+1]
            }
            merged = append(merged, mergeTwoLists(l1, l2))
        }
        
        lists = merged
    }
    
    return lists[0]
}

// Time: O(N log k) where N is total nodes, k is number of lists
// Space: O(1)
```

### Merge Sort on Linked List
```go
func mergeSort(head *Node) *Node {
    if head == nil || head.Next == nil {
        return head
    }
    
    // Find middle
    middle := getMiddle(head)
    nextToMiddle := middle.Next
    middle.Next = nil
    
    // Sort both halves
    left := mergeSort(head)
    right := mergeSort(nextToMiddle)
    
    // Merge sorted halves
    return mergeTwoLists(left, right)
}

func getMiddle(head *Node) *Node {
    if head == nil {
        return head
    }
    
    slow, fast := head, head.Next
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    
    return slow
}

// Time: O(n log n), Space: O(log n) for recursion
```

---

## Performance Characteristics

### Time Complexity
| Operation | Singly | Doubly | Array |
|-----------|--------|--------|-------|
| Access by index | O(n) | O(n) | O(1) |
| Search | O(n) | O(n) | O(n) |
| Insert at beginning | O(1) | O(1) | O(n) |
| Insert at end | O(n) | O(1)* | O(1)* |
| Insert at position | O(n) | O(n) | O(n) |
| Delete at beginning | O(1) | O(1) | O(n) |
| Delete at end | O(n) | O(1)* | O(1)* |
| Delete at position | O(n) | O(n) | O(n) |

*With tail pointer

### Space Complexity
- Singly: O(n) - one pointer per node
- Doubly: O(n) - two pointers per node
- Circular: O(n) - same as singly/doubly

### Cache Performance
```go
// Array: Better cache locality
arr := []int{1, 2, 3, 4, 5}
// Elements stored contiguously
// CPU can prefetch next elements

// Linked List: Poor cache locality
// Nodes scattered in memory
// Each access may cause cache miss
```

---

## Common Patterns

### Dummy Head Node
```go
// Simplifies edge cases
func removeElements(head *Node, val int) *Node {
    dummy := &Node{Next: head}
    current := dummy
    
    for current.Next != nil {
        if current.Next.Data == val {
            current.Next = current.Next.Next
        } else {
            current = current.Next
        }
    }
    
    return dummy.Next
}
```

### Runner Technique
```go
// Weave two lists
func weave(head *Node) *Node {
    if head == nil || head.Next == nil {
        return head
    }
    
    // Split into two halves
    slow, fast := head, head
    for fast.Next != nil && fast.Next.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    
    second := slow.Next
    slow.Next = nil
    
    // Reverse second half
    second = reverse(second)
    
    // Merge alternately
    first := head
    for second != nil {
        temp1, temp2 := first.Next, second.Next
        first.Next = second
        second.Next = temp1
        first = temp1
        second = temp2
    }
    
    return head
}
```

### Recursion for Traversal
```go
func printReverse(head *Node) {
    if head == nil {
        return
    }
    printReverse(head.Next)
    fmt.Printf("%d ", head.Data)
}
```

---

## Best Practices

### 1. Use Dummy Head for Edge Cases
```go
// Simplifies operations at head
dummy := &Node{Next: head}
// Work with dummy.Next
return dummy.Next
```

### 2. Check for Nil
```go
func safeOperation(head *Node) {
    if head == nil {
        return
    }
    // Proceed with operation
}
```

### 3. Maintain Size Field
```go
type LinkedList struct {
    Head *Node
    Size int  // Track size for O(1) length
}
```

### 4. Use Two Pointers
```go
// For many problems: middle, cycle, nth from end
slow, fast := head, head
```

### 5. Draw Diagrams
```go
// Visualize pointer changes
// Before: A → B → C
// After:  A ← B ← C
```

---

## Common Pitfalls

### 1. Losing Reference to Head
```go
// Wrong
func wrong(head *Node) {
    head = head.Next  // Loses original head
}

// Correct
func correct(head *Node) *Node {
    newHead := head.Next
    return newHead
}
```

### 2. Not Handling Empty List
```go
// Wrong
func wrong(head *Node) {
    head.Next = nil  // Panic if head is nil
}

// Correct
func correct(head *Node) {
    if head == nil {
        return
    }
    head.Next = nil
}
```

### 3. Infinite Loop in Circular List
```go
// Wrong
func wrong(head *Node) {
    current := head
    for current.Next != nil {  // Never nil in circular
        current = current.Next
    }
}

// Correct
func correct(head *Node) {
    if head == nil {
        return
    }
    current := head
    for current.Next != head {
        current = current.Next
    }
}
```

### 4. Memory Leak in Deletion
```go
// In Go, GC handles this, but be aware
func delete(head *Node) *Node {
    temp := head
    head = head.Next
    // temp will be garbage collected
    return head
}
```

---

## Interview Problems

### 1. Palindrome Check
```go
func isPalindrome(head *Node) bool {
    // Find middle
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
    }
    
    // Reverse second half
    second := reverse(slow)
    first := head
    
    // Compare
    for second != nil {
        if first.Data != second.Data {
            return false
        }
        first = first.Next
        second = second.Next
    }
    
    return true
}
```

### 2. Intersection Point
```go
func getIntersection(head1, head2 *Node) *Node {
    if head1 == nil || head2 == nil {
        return nil
    }
    
    p1, p2 := head1, head2
    
    for p1 != p2 {
        if p1 == nil {
            p1 = head2
        } else {
            p1 = p1.Next
        }
        
        if p2 == nil {
            p2 = head1
        } else {
            p2 = p2.Next
        }
    }
    
    return p1
}
```

### 3. Add Two Numbers
```go
func addTwoNumbers(l1, l2 *Node) *Node {
    dummy := &Node{}
    current := dummy
    carry := 0
    
    for l1 != nil || l2 != nil || carry > 0 {
        sum := carry
        
        if l1 != nil {
            sum += l1.Data
            l1 = l1.Next
        }
        
        if l2 != nil {
            sum += l2.Data
            l2 = l2.Next
        }
        
        carry = sum / 10
        current.Next = &Node{Data: sum % 10}
        current = current.Next
    }
    
    return dummy.Next
}
```

### 4. Flatten Multilevel List
```go
type MultiNode struct {
    Data  int
    Next  *MultiNode
    Child *MultiNode
}

func flatten(head *MultiNode) *MultiNode {
    if head == nil {
        return nil
    }
    
    current := head
    for current != nil {
        if current.Child != nil {
            // Find tail of child list
            tail := current.Child
            for tail.Next != nil {
                tail = tail.Next
            }
            
            // Insert child list
            tail.Next = current.Next
            current.Next = current.Child
            current.Child = nil
        }
        current = current.Next
    }
    
    return head
}
```

---

## Summary

### Key Concepts
1. **Dynamic Size**: Grows/shrinks as needed
2. **Pointer-based**: Nodes connected via pointers
3. **No Random Access**: Must traverse from head
4. **Efficient Insert/Delete**: At known positions

### When to Use
- Frequent insertions/deletions at beginning
- Unknown or changing size
- Don't need random access
- Implementing stacks, queues, graphs

### When Not to Use
- Need random access (use array/slice)
- Memory is constrained (extra pointer overhead)
- Cache performance critical
- Need binary search

### Common Techniques
1. Two pointers (fast/slow)
2. Dummy head node
3. Recursion for reversal
4. Floyd's cycle detection
5. Merge operations

### Time Complexity Summary
- Access: O(n)
- Search: O(n)
- Insert at head: O(1)
- Insert at tail: O(n) or O(1) with tail pointer
- Delete at head: O(1)
- Delete at tail: O(n) or O(1) with tail pointer (doubly)

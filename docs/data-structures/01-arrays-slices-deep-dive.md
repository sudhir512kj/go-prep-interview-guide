# Arrays and Slices Deep Dive

## Table of Contents
- [Arrays Fundamentals](#arrays-fundamentals)
- [Arrays Internal Implementation](#arrays-internal-implementation)
- [Slices Fundamentals](#slices-fundamentals)
- [Slices Internal Implementation](#slices-internal-implementation)
- [Slice Header Structure](#slice-header-structure)
- [Memory Layout](#memory-layout)
- [Capacity and Growth](#capacity-and-growth)
- [Slice Operations](#slice-operations)
- [Slicing Operations](#slicing-operations)
- [Copy vs Append](#copy-vs-append)
- [Multi-dimensional Arrays and Slices](#multi-dimensional-arrays-and-slices)
- [Performance Characteristics](#performance-characteristics)
- [Common Patterns](#common-patterns)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)
- [Memory Leaks](#memory-leaks)
- [Benchmarking](#benchmarking)

---

## Arrays Fundamentals

### Definition
Arrays are fixed-size, contiguous memory blocks holding elements of the same type.

```go
// Array declaration
var arr1 [5]int                    // [0 0 0 0 0]
arr2 := [5]int{1, 2, 3, 4, 5}     // [1 2 3 4 5]
arr3 := [...]int{1, 2, 3}         // Compiler infers size: [3]int
arr4 := [5]int{1: 10, 3: 30}      // [0 10 0 30 0] - indexed initialization
```

### Key Characteristics
- **Fixed Size**: Size is part of the type
- **Value Type**: Arrays are copied when assigned or passed
- **Zero Value**: Elements initialized to zero value of type
- **Comparable**: Arrays of same type and size can be compared with ==

```go
// Arrays are value types
arr1 := [3]int{1, 2, 3}
arr2 := arr1              // Copy, not reference
arr2[0] = 100
fmt.Println(arr1)         // [1 2 3] - unchanged
fmt.Println(arr2)         // [100 2 3]

// Arrays are comparable
a := [3]int{1, 2, 3}
b := [3]int{1, 2, 3}
fmt.Println(a == b)       // true
```

---

## Arrays Internal Implementation

### Memory Layout
Arrays are stored as contiguous blocks in memory.

```
Array: [5]int{10, 20, 30, 40, 50}

Memory Layout:
┌────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │ 50 │
└────┴────┴────┴────┴────┘
 ↑
 Base Address
```

### Size Calculation
```go
// Array size = element_size * length
var arr [100]int64
// Size = 8 bytes * 100 = 800 bytes

var arr2 [1000]byte
// Size = 1 byte * 1000 = 1000 bytes
```

### Stack vs Heap Allocation
```go
func stackArray() {
    arr := [10]int{1, 2, 3}  // Small array - stack allocated
    _ = arr
}

func heapArray() *[10000]int {
    arr := [10000]int{}      // Large array - heap allocated
    return &arr              // Escapes to heap
}
```

---

## Slices Fundamentals

### Definition
Slices are dynamic, flexible views into arrays with variable length.

```go
// Slice declaration
var s1 []int                      // nil slice
s2 := []int{}                     // Empty slice (not nil)
s3 := []int{1, 2, 3}             // Slice literal
s4 := make([]int, 5)             // Length 5, capacity 5
s5 := make([]int, 5, 10)         // Length 5, capacity 10
```

### Nil vs Empty Slice
```go
var nilSlice []int               // nil slice
emptySlice := []int{}            // empty slice

fmt.Println(nilSlice == nil)     // true
fmt.Println(emptySlice == nil)   // false
fmt.Println(len(nilSlice))       // 0
fmt.Println(len(emptySlice))     // 0
```

---

## Slices Internal Implementation

### Slice Header Structure
A slice is a descriptor containing three fields:

```go
// Runtime representation (runtime/slice.go)
type slice struct {
    array unsafe.Pointer  // Pointer to underlying array
    len   int            // Current length
    cap   int            // Capacity
}
```

### Visual Representation
```
Slice: s := []int{1, 2, 3, 4, 5}

Slice Header:
┌─────────┬─────┬─────┐
│ pointer │ len │ cap │
│    •────┼──5──┼──5──┤
└────┼────┴─────┴─────┘
     │
     ↓
Underlying Array:
┌────┬────┬────┬────┬────┐
│ 1  │ 2  │ 3  │ 4  │ 5  │
└────┴────┴────┴────┴────┘
```

### Creating Slices from Arrays
```go
arr := [5]int{1, 2, 3, 4, 5}
s1 := arr[:]        // Slice of entire array
s2 := arr[1:4]      // Slice from index 1 to 3
s3 := arr[:3]       // Slice from start to index 2
s4 := arr[2:]       // Slice from index 2 to end

// All slices share the same underlying array
s1[0] = 100
fmt.Println(arr)    // [100 2 3 4 5]
```

---

## Memory Layout

### Slice Memory Model
```go
s := make([]int, 3, 5)
s[0], s[1], s[2] = 10, 20, 30

Memory Layout:
Slice Header (24 bytes on 64-bit):
┌──────────────┬─────┬─────┐
│   pointer    │ len │ cap │
│      •───────┼──3──┼──5──┤
└──────┼───────┴─────┴─────┘
       │
       ↓
Underlying Array (heap):
┌────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 0  │ 0  │
└────┴────┴────┴────┴────┘
 ←──── len ────→
 ←──────── cap ────────→
```

### Shared Underlying Array
```go
s1 := []int{1, 2, 3, 4, 5}
s2 := s1[1:4]  // [2 3 4]

Visualization:
s1: ┌─────────┬─────┬─────┐
    │ pointer │ len │ cap │
    │    •────┼──5──┼──5──┤
    └────┼────┴─────┴─────┘
         │
         ↓
Array:  ┌────┬────┬────┬────┬────┐
        │ 1  │ 2  │ 3  │ 4  │ 5  │
        └────┴────┴────┴────┴────┘
              ↑
              │
s2: ┌─────────┼─────┬─────┐
    │ pointer │ len │ cap │
    │    •────┼──3──┼──4──┤
    └─────────┴─────┴─────┘
```

---

## Capacity and Growth

### Growth Strategy
When appending exceeds capacity, Go allocates a new array with increased capacity.

```go
// Growth algorithm (simplified from runtime/slice.go)
func growslice(oldCap, newLen int) int {
    newCap := oldCap
    doubleCap := newCap + newCap
    
    if newLen > doubleCap {
        newCap = newLen
    } else {
        const threshold = 256
        if oldCap < threshold {
            newCap = doubleCap  // Double for small slices
        } else {
            // Grow by 1.25x for larger slices
            for newCap < newLen {
                newCap += (newCap + 3*threshold) / 4
            }
        }
    }
    return newCap
}
```

### Growth Example
```go
s := make([]int, 0)
capacities := []int{}

for i := 0; i < 20; i++ {
    s = append(s, i)
    capacities = append(capacities, cap(s))
}

fmt.Println(capacities)
// Output: [1 2 4 4 8 8 8 8 16 16 16 16 16 16 16 16 32 32 32 32]
// Pattern: 1 → 2 → 4 → 8 → 16 → 32 (doubles until 256)
```

### Reallocation Visualization
```go
s := make([]int, 2, 2)  // len=2, cap=2
s = append(s, 3)        // Triggers reallocation

Before append:
s: [pointer•] len=2 cap=2
         ↓
    [elem0][elem1]

After append:
s: [pointer•] len=3 cap=4
         ↓
    [elem0][elem1][3][unused]
    
Old array is garbage collected
```

---

## Slice Operations

### Append Operation
```go
// Single element
s := []int{1, 2, 3}
s = append(s, 4)              // [1 2 3 4]

// Multiple elements
s = append(s, 5, 6, 7)        // [1 2 3 4 5 6 7]

// Append another slice
s2 := []int{8, 9}
s = append(s, s2...)          // [1 2 3 4 5 6 7 8 9]

// Append to nil slice
var s3 []int
s3 = append(s3, 1)            // [1] - works fine
```

### Insert Operation
```go
func insert(slice []int, index, value int) []int {
    // Grow slice by 1
    slice = append(slice, 0)
    // Shift elements right
    copy(slice[index+1:], slice[index:])
    // Insert value
    slice[index] = value
    return slice
}

s := []int{1, 2, 4, 5}
s = insert(s, 2, 3)           // [1 2 3 4 5]
```

### Delete Operation
```go
// Delete element at index
func delete(slice []int, index int) []int {
    return append(slice[:index], slice[index+1:]...)
}

s := []int{1, 2, 3, 4, 5}
s = delete(s, 2)              // [1 2 4 5]

// Delete range [i:j]
func deleteRange(slice []int, i, j int) []int {
    return append(slice[:i], slice[j:]...)
}

s = []int{1, 2, 3, 4, 5}
s = deleteRange(s, 1, 3)      // [1 4 5]
```

### Filter Operation
```go
func filter(slice []int, fn func(int) bool) []int {
    result := slice[:0]  // Reuse backing array
    for _, v := range slice {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}

s := []int{1, 2, 3, 4, 5, 6}
even := filter(s, func(n int) bool { return n%2 == 0 })
fmt.Println(even)             // [2 4 6]
```

---

## Slicing Operations

### Slice Expressions
```go
s := []int{0, 1, 2, 3, 4, 5}

// Basic slicing: s[low:high]
s1 := s[1:4]      // [1 2 3] - len=3, cap=5
s2 := s[:3]       // [0 1 2] - len=3, cap=6
s3 := s[2:]       // [2 3 4 5] - len=4, cap=4
s4 := s[:]        // [0 1 2 3 4 5] - full slice

// Full slice expression: s[low:high:max]
s5 := s[1:3:4]    // [1 2] - len=2, cap=3
// Limits capacity to prevent affecting original
```

### Full Slice Expression Benefits
```go
// Without full slice expression
s1 := []int{1, 2, 3, 4, 5}
s2 := s1[0:2]              // len=2, cap=5
s2 = append(s2, 100)       // Modifies s1!
fmt.Println(s1)            // [1 2 100 4 5]

// With full slice expression
s1 = []int{1, 2, 3, 4, 5}
s2 = s1[0:2:2]             // len=2, cap=2
s2 = append(s2, 100)       // New allocation
fmt.Println(s1)            // [1 2 3 4 5] - unchanged
```

---

## Copy vs Append

### Copy Function
```go
// copy(dst, src) copies min(len(dst), len(src)) elements
src := []int{1, 2, 3, 4, 5}
dst := make([]int, 3)
n := copy(dst, src)
fmt.Println(dst, n)        // [1 2 3] 3

// Overlapping slices
s := []int{1, 2, 3, 4, 5}
copy(s[1:], s[:4])         // Shift left
fmt.Println(s)             // [1 1 2 3 4]

// Copy returns number of elements copied
dst = make([]int, 10)
n = copy(dst, src)
fmt.Println(n)             // 5 (limited by src length)
```

### Copy vs Append Performance
```go
// Append - may allocate
s1 := make([]int, 0, 5)
for i := 0; i < 5; i++ {
    s1 = append(s1, i)
}

// Copy - pre-allocated
s2 := make([]int, 5)
for i := 0; i < 5; i++ {
    s2[i] = i
}
// Copy is faster when size is known
```

---

## Multi-dimensional Arrays and Slices

### 2D Arrays
```go
// Fixed-size 2D array
var matrix [3][4]int
matrix[0][0] = 1

// Initialized 2D array
matrix2 := [2][3]int{
    {1, 2, 3},
    {4, 5, 6},
}
```

### 2D Slices
```go
// Method 1: Slice of slices (jagged)
rows := 3
cols := 4
matrix := make([][]int, rows)
for i := range matrix {
    matrix[i] = make([]int, cols)
}

// Method 2: Single allocation (better performance)
func make2D(rows, cols int) [][]int {
    // Allocate single backing array
    data := make([]int, rows*cols)
    // Create slice of slices
    matrix := make([][]int, rows)
    for i := range matrix {
        matrix[i] = data[i*cols : (i+1)*cols]
    }
    return matrix
}

m := make2D(3, 4)
m[1][2] = 42
```

### 3D Slices
```go
func make3D(x, y, z int) [][][]int {
    data := make([]int, x*y*z)
    matrix := make([][][]int, x)
    for i := range matrix {
        matrix[i] = make([][]int, y)
        for j := range matrix[i] {
            start := (i*y+j)*z
            matrix[i][j] = data[start : start+z]
        }
    }
    return matrix
}
```

---

## Performance Characteristics

### Time Complexity
| Operation | Array | Slice |
|-----------|-------|-------|
| Access by index | O(1) | O(1) |
| Update by index | O(1) | O(1) |
| Append | N/A | O(1) amortized |
| Insert at index | N/A | O(n) |
| Delete at index | N/A | O(n) |
| Copy | O(n) | O(n) |
| Slice operation | N/A | O(1) |

### Space Complexity
- Array: O(n) - fixed size
- Slice: O(n) - dynamic, may have unused capacity

### Amortized Analysis of Append
```
Appends: 1, 2, 3, 4, 5, 6, 7, 8, 9...
Copies:  0, 1, 2, 0, 4, 0, 0, 0, 8...

Total copies for n appends: n/2 + n/4 + n/8 + ... ≈ n
Average per append: n/n = O(1) amortized
```

---

## Common Patterns

### Pre-allocation
```go
// Bad: Multiple reallocations
s := []int{}
for i := 0; i < 1000; i++ {
    s = append(s, i)
}

// Good: Single allocation
s := make([]int, 0, 1000)
for i := 0; i < 1000; i++ {
    s = append(s, i)
}
```

### Filtering In-Place
```go
func filterInPlace(s []int, keep func(int) bool) []int {
    n := 0
    for _, v := range s {
        if keep(v) {
            s[n] = v
            n++
        }
    }
    return s[:n]
}
```

### Reversing
```go
func reverse(s []int) {
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        s[i], s[j] = s[j], s[i]
    }
}
```

### Rotating
```go
func rotate(s []int, k int) []int {
    k = k % len(s)
    reverse(s[:k])
    reverse(s[k:])
    reverse(s)
    return s
}
```

### Deduplication
```go
func deduplicate(s []int) []int {
    if len(s) == 0 {
        return s
    }
    j := 0
    for i := 1; i < len(s); i++ {
        if s[i] != s[j] {
            j++
            s[j] = s[i]
        }
    }
    return s[:j+1]
}
```

---

## Best Practices

### 1. Pre-allocate When Size is Known
```go
// Bad
var result []int
for i := 0; i < 1000; i++ {
    result = append(result, i)
}

// Good
result := make([]int, 0, 1000)
for i := 0; i < 1000; i++ {
    result = append(result, i)
}
```

### 2. Use Full Slice Expression for Safety
```go
// Prevent unintended sharing
func process(data []int) []int {
    // Limit capacity to prevent affecting original
    return data[0:len(data):len(data)]
}
```

### 3. Clear Slice References
```go
// Clear references to prevent memory leaks
func clearSlice(s []*BigStruct) []*BigStruct {
    for i := range s {
        s[i] = nil  // Allow GC to collect
    }
    return s[:0]
}
```

### 4. Use Copy for Independent Slices
```go
// Create independent copy
original := []int{1, 2, 3}
copied := make([]int, len(original))
copy(copied, original)
```

### 5. Check Nil Before Operations
```go
func safeAppend(s []int, v int) []int {
    // Works with nil slice
    return append(s, v)
}

func safeLen(s []int) int {
    // len(nil) returns 0
    return len(s)
}
```

---

## Common Pitfalls

### 1. Slice Aliasing
```go
// Problem: Shared backing array
s1 := []int{1, 2, 3, 4, 5}
s2 := s1[1:3]
s2[0] = 100
fmt.Println(s1)  // [1 100 3 4 5] - unexpected!

// Solution: Copy to new slice
s2 := make([]int, 2)
copy(s2, s1[1:3])
```

### 2. Append to Slice in Loop
```go
// Problem: Unexpected behavior
s := []int{1, 2, 3}
for _, v := range s {
    s = append(s, v)  // Infinite loop potential
}

// Solution: Use index or copy
original := make([]int, len(s))
copy(original, s)
for _, v := range original {
    s = append(s, v)
}
```

### 3. Slice of Pointers to Loop Variables
```go
// Problem: All pointers point to same variable
var ptrs []*int
for i := 0; i < 3; i++ {
    ptrs = append(ptrs, &i)
}
// All pointers have value 3

// Solution: Create new variable
for i := 0; i < 3; i++ {
    i := i  // Create new variable
    ptrs = append(ptrs, &i)
}
```

### 4. Comparing Slices
```go
// Problem: Slices are not comparable
s1 := []int{1, 2, 3}
s2 := []int{1, 2, 3}
// fmt.Println(s1 == s2)  // Compile error

// Solution: Use reflect.DeepEqual or manual comparison
equal := reflect.DeepEqual(s1, s2)

// Or manual
func equal(a, b []int) bool {
    if len(a) != len(b) {
        return false
    }
    for i := range a {
        if a[i] != b[i] {
            return false
        }
    }
    return true
}
```

---

## Memory Leaks

### Problem: Retaining Large Array
```go
// Problem: Keeping reference to large array
func findDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)  // Large file
    digits := make([]byte, 0)
    for _, c := range b {
        if unicode.IsDigit(rune(c)) {
            digits = append(digits, c)
        }
    }
    return digits  // Still references large array
}

// Solution: Copy to new slice
func findDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    digits := make([]byte, 0)
    for _, c := range b {
        if unicode.IsDigit(rune(c)) {
            digits = append(digits, c)
        }
    }
    // Create independent copy
    result := make([]byte, len(digits))
    copy(result, digits)
    return result
}
```

### Problem: Slice of Pointers
```go
// Problem: Pointers prevent GC
type Item struct {
    data [1024]byte
}

cache := make([]*Item, 1000)
// Even if we clear the slice, pointers remain
cache = cache[:0]  // Items not collected!

// Solution: Nil out pointers
for i := range cache {
    cache[i] = nil
}
cache = cache[:0]
```

---

## Benchmarking

### Append Performance
```go
func BenchmarkAppendNoPrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := []int{}
        for j := 0; j < 1000; j++ {
            s = append(s, j)
        }
    }
}

func BenchmarkAppendPrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        s := make([]int, 0, 1000)
        for j := 0; j < 1000; j++ {
            s = append(s, j)
        }
    }
}
// Preallocated is ~10x faster
```

### Copy vs Append
```go
func BenchmarkCopy(b *testing.B) {
    src := make([]int, 1000)
    for i := 0; i < b.N; i++ {
        dst := make([]int, 1000)
        copy(dst, src)
    }
}

func BenchmarkAppend(b *testing.B) {
    src := make([]int, 1000)
    for i := 0; i < b.N; i++ {
        dst := append([]int{}, src...)
    }
}
// Copy is faster for known sizes
```

---

## Summary

### Arrays
- Fixed size, value type
- Stack allocated (if small)
- Comparable with ==
- Use when size is known and fixed

### Slices
- Dynamic size, reference type (header)
- Flexible and efficient
- Growth strategy: double until 256, then 1.25x
- Pre-allocate when size is known
- Be aware of shared backing arrays
- Use full slice expression for safety

### Key Takeaways
1. Understand slice header structure (pointer, len, cap)
2. Pre-allocate slices when size is known
3. Be careful with slice aliasing
4. Use copy for independent slices
5. Clear references to prevent memory leaks
6. Slices are not comparable (use reflect.DeepEqual)
7. Append is O(1) amortized
8. Use full slice expression to limit capacity

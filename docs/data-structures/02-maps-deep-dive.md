# Maps Deep Dive

## Table of Contents
- [Maps Fundamentals](#maps-fundamentals)
- [Internal Implementation](#internal-implementation)
- [Hash Function](#hash-function)
- [Collision Resolution](#collision-resolution)
- [Bucket Structure](#bucket-structure)
- [Growth and Rehashing](#growth-and-rehashing)
- [Map Operations](#map-operations)
- [Iteration Order](#iteration-order)
- [Concurrent Access](#concurrent-access)
- [Performance Characteristics](#performance-characteristics)
- [Common Patterns](#common-patterns)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)
- [sync.Map](#syncmap)
- [Benchmarking](#benchmarking)

---

## Maps Fundamentals

### Definition
Maps are unordered collections of key-value pairs with O(1) average lookup time.

```go
// Map declaration
var m1 map[string]int              // nil map (read-only)
m2 := map[string]int{}             // Empty map
m3 := make(map[string]int)         // Empty map
m4 := make(map[string]int, 100)    // With capacity hint

// Map literal
m5 := map[string]int{
    "alice": 25,
    "bob":   30,
    "carol": 28,
}
```

### Key Requirements
Keys must be comparable (support == and !=):
- ✅ Valid: int, string, bool, pointer, channel, interface, struct/array with comparable fields
- ❌ Invalid: slice, map, function

```go
// Valid key types
m1 := make(map[int]string)
m2 := make(map[string]int)
m3 := make(map[struct{x, y int}]bool)

// Invalid key types
// m4 := make(map[[]int]string)     // Error: slice
// m5 := make(map[map[string]int]bool) // Error: map
```

### Zero Value
```go
var m map[string]int               // nil map
fmt.Println(m == nil)              // true
fmt.Println(len(m))                // 0
fmt.Println(m["key"])              // 0 (zero value)
// m["key"] = 1                    // panic: assignment to nil map

// Must initialize before writing
m = make(map[string]int)
m["key"] = 1                       // OK
```

---

## Internal Implementation

### Hash Table Structure
Go maps use hash tables with separate chaining via buckets.

```go
// Simplified runtime representation (runtime/map.go)
type hmap struct {
    count     int    // Number of elements
    flags     uint8  // State flags
    B         uint8  // log_2 of number of buckets
    noverflow uint16 // Approximate overflow bucket count
    hash0     uint32 // Hash seed
    buckets   unsafe.Pointer // Array of 2^B buckets
    oldbuckets unsafe.Pointer // Previous buckets during growth
    nevacuate uintptr // Progress counter for evacuation
}
```

### Bucket Structure
Each bucket holds up to 8 key-value pairs.

```go
// Bucket structure (simplified)
type bmap struct {
    tophash [8]uint8  // High 8 bits of hash for each key
    keys    [8]keytype
    values  [8]valuetype
    overflow *bmap    // Pointer to overflow bucket
}
```

### Visual Representation
```
Map: m := map[string]int{"a": 1, "b": 2, "c": 3}

hmap:
┌──────────┬────────┬───┬──────────┬──────────┐
│ count: 3 │ B: 0   │...│ buckets  │ hash0    │
└──────────┴────────┴───┴────┬─────┴──────────┘
                              │
                              ↓
Buckets Array (2^B = 1 bucket):
┌─────────────────────────────────────┐
│ Bucket 0                            │
│ ┌──────────────────────────────┐   │
│ │ tophash: [h1|h2|h3|0|0|0|0|0]│   │
│ │ keys:    [a |b |c |·|·|·|·|·]│   │
│ │ values:  [1 |2 |3 |·|·|·|·|·]│   │
│ │ overflow: nil                 │   │
│ └──────────────────────────────┘   │
└─────────────────────────────────────┘
```

---

## Hash Function

### Hash Computation
Go uses different hash functions based on key type.

```go
// Hash function selection (runtime/alg.go)
// - String: AES-based hash
// - Integer: identity or multiplication
// - Float: bit pattern hash
// - Complex: combine real and imaginary parts

// Hash includes random seed (hash0) for security
hash := hashFunction(key, hash0)
```

### Bucket Selection
```go
// Determine bucket index
bucketIndex := hash & (2^B - 1)  // Use low B bits

// Example: B=3 (8 buckets)
// hash = 0b10110101
// mask = 0b00000111 (2^3 - 1)
// index = 0b00000101 = 5
```

### Top Hash
```go
// Store high 8 bits for quick comparison
tophash := hash >> (64 - 8)  // Top 8 bits

// Used to quickly reject non-matching keys
// before doing full key comparison
```

---

## Collision Resolution

### Separate Chaining with Buckets
When multiple keys hash to same bucket:

```go
// 1. Check tophash array for matching hash
// 2. If match found, compare full key
// 3. If bucket full, use overflow bucket

Bucket (8 slots):
┌──────────────────────────────┐
│ tophash: [h1|h2|h3|h4|h5|h6|h7|h8] │
│ keys:    [k1|k2|k3|k4|k5|k6|k7|k8] │
│ values:  [v1|v2|v3|v4|v5|v6|v7|v8] │
│ overflow: •──────────────────┐     │
└──────────────────────────────┘     │
                                     ↓
                    Overflow Bucket:
                    ┌──────────────────────────────┐
                    │ tophash: [h9|h10|0|0|0|0|0|0]│
                    │ keys:    [k9|k10|·|·|·|·|·|·]│
                    │ values:  [v9|v10|·|·|·|·|·|·]│
                    │ overflow: nil                │
                    └──────────────────────────────┘
```

### Lookup Algorithm
```go
func mapaccess(m *hmap, key keytype) valuetype {
    hash := hashFunction(key, m.hash0)
    bucketIndex := hash & (1<<m.B - 1)
    tophash := uint8(hash >> 56)
    
    bucket := (*bmap)(add(m.buckets, bucketIndex*bucketSize))
    
    for bucket != nil {
        for i := 0; i < 8; i++ {
            if bucket.tophash[i] != tophash {
                continue
            }
            k := bucket.keys[i]
            if key == k {
                return bucket.values[i]
            }
        }
        bucket = bucket.overflow
    }
    return zeroValue
}
```

---

## Bucket Structure

### Memory Layout
Keys and values stored separately for better cache locality.

```go
// Bucket layout in memory
type bmap struct {
    tophash [8]uint8
    // Followed by 8 keys
    // Followed by 8 values
    // Followed by overflow pointer
}

Memory Layout:
┌────────────────────────┐
│ tophash[0..7]          │ 8 bytes
├────────────────────────┤
│ key[0]                 │
│ key[1]                 │
│ ...                    │
│ key[7]                 │
├────────────────────────┤
│ value[0]               │
│ value[1]               │
│ ...                    │
│ value[7]               │
├────────────────────────┤
│ overflow pointer       │ 8 bytes
└────────────────────────┘
```

### Why Separate Keys and Values?
```go
// Better cache locality when scanning keys
// Keys are checked more often than values

// Bad layout (alternating):
[k1][v1][k2][v2][k3][v3]...
// Wastes cache lines when only checking keys

// Good layout (grouped):
[k1][k2][k3][k4][k5][k6][k7][k8][v1][v2]...
// All keys fit in fewer cache lines
```

---

## Growth and Rehashing

### Growth Triggers
Map grows when:
1. Load factor > 6.5 (too many elements per bucket)
2. Too many overflow buckets

```go
// Growth conditions (runtime/map.go)
const (
    loadFactorNum = 13
    loadFactorDen = 2  // 6.5 = 13/2
)

// Trigger growth if:
// count > bucketCount * loadFactorNum / loadFactorDen
// OR too many overflow buckets
```

### Growth Types

#### 1. Doubling Growth (overLoadFactor)
```go
// When load factor exceeded
// Double number of buckets: 2^B → 2^(B+1)

Before (B=2, 4 buckets):
┌────┬────┬────┬────┐
│ B0 │ B1 │ B2 │ B3 │
└────┴────┴────┴────┘

After (B=3, 8 buckets):
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ B0 │ B1 │ B2 │ B3 │ B4 │ B5 │ B6 │ B7 │
└────┴────┴────┴────┴────┴────┴────┴────┘
```

#### 2. Same-size Growth (tooManyOverflowBuckets)
```go
// When too many overflow buckets but low load
// Compact data to eliminate overflow buckets
// Number of buckets stays same

Before:
Bucket → Overflow → Overflow
[k1|k2|k3|k4|k5|k6|k7|k8] → [k9|·|·|·|·|·|·|·] → [k10|·|·|·|·|·|·|·]

After:
Bucket
[k1|k2|k3|k4|k5|k6|k7|k8|k9|k10|·|·|·|·|·|·]
```

### Incremental Evacuation
```go
// Growth happens incrementally during map operations
// Not all at once (avoids long pauses)

type hmap struct {
    oldbuckets unsafe.Pointer  // Old bucket array
    nevacuate  uintptr         // Progress counter
}

// Each map operation evacuates 1-2 buckets
// Until all old buckets are evacuated
```

### Evacuation Process
```go
// Evacuate old bucket to new buckets
func evacuate(m *hmap, oldbucket int) {
    oldBucket := (*bmap)(add(m.oldbuckets, oldbucket*bucketSize))
    
    // For each key-value in old bucket
    for each key-value pair {
        hash := hashFunction(key, m.hash0)
        
        // Determine new bucket
        if growing by doubling {
            // Use one more bit of hash
            newBucketIndex := hash & (1<<m.B - 1)
        } else {
            // Same bucket index
            newBucketIndex := oldbucket
        }
        
        // Insert into new bucket
        insertIntoNewBucket(newBucketIndex, key, value)
    }
}
```

---

## Map Operations

### Insert/Update
```go
m := make(map[string]int)

// Insert
m["alice"] = 25

// Update
m["alice"] = 26

// Insert or update
m["bob"] = 30
```

### Lookup
```go
// Simple lookup
value := m["alice"]  // Returns zero value if not found

// Check existence
value, ok := m["alice"]
if ok {
    fmt.Println("Found:", value)
} else {
    fmt.Println("Not found")
}

// Idiomatic check
if value, ok := m["alice"]; ok {
    fmt.Println("Found:", value)
}
```

### Delete
```go
// Delete key
delete(m, "alice")

// Delete non-existent key (no-op)
delete(m, "nonexistent")  // Safe, no panic

// Delete from nil map (no-op)
var m map[string]int
delete(m, "key")  // Safe, no panic
```

### Length
```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
fmt.Println(len(m))  // 3

// Length of nil map
var m2 map[string]int
fmt.Println(len(m2))  // 0
```

### Clear (Go 1.21+)
```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
clear(m)  // Removes all elements
fmt.Println(len(m))  // 0
```

---

## Iteration Order

### Non-deterministic Order
```go
m := map[string]int{"a": 1, "b": 2, "c": 3}

// Order is random and changes between runs
for k, v := range m {
    fmt.Println(k, v)
}
// Output varies: might be a,b,c or c,a,b or b,c,a
```

### Why Random Order?
```go
// 1. Hash table has no inherent order
// 2. Intentionally randomized to prevent reliance on order
// 3. Random starting point in iteration

// Runtime adds randomness (runtime/map.go)
func mapiterinit(m *hmap, it *hiter) {
    r := uintptr(fastrand())
    it.startBucket = r & (1<<m.B - 1)
    it.offset = uint8(r >> m.B)
}
```

### Sorted Iteration
```go
m := map[string]int{"c": 3, "a": 1, "b": 2}

// Extract and sort keys
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)

// Iterate in sorted order
for _, k := range keys {
    fmt.Println(k, m[k])
}
// Output: a 1, b 2, c 3
```

---

## Concurrent Access

### Not Thread-Safe
```go
// Concurrent reads and writes cause panic
m := make(map[string]int)

// This will panic with "concurrent map read and map write"
go func() {
    for {
        m["key"] = 1  // Write
    }
}()

go func() {
    for {
        _ = m["key"]  // Read
    }
}()
```

### Detection Mechanism
```go
// Runtime detects concurrent access
type hmap struct {
    flags uint8  // Includes write flag
}

const (
    hashWriting = 4  // Set during write operations
)

// On write: set flag
// On read/write: check flag, panic if set
```

### Solutions

#### 1. Mutex Protection
```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (sm *SafeMap) Get(key string) (int, bool) {
    sm.mu.RLock()
    defer sm.mu.RUnlock()
    val, ok := sm.m[key]
    return val, ok
}

func (sm *SafeMap) Set(key string, value int) {
    sm.mu.Lock()
    defer sm.mu.Unlock()
    sm.m[key] = value
}
```

#### 2. sync.Map
```go
var m sync.Map

// Store
m.Store("key", 42)

// Load
value, ok := m.Load("key")

// LoadOrStore
actual, loaded := m.LoadOrStore("key", 42)

// Delete
m.Delete("key")

// Range
m.Range(func(key, value interface{}) bool {
    fmt.Println(key, value)
    return true  // Continue iteration
})
```

#### 3. Channel-based
```go
type MapOp struct {
    op    string
    key   string
    value int
    resp  chan int
}

func mapManager(ops chan MapOp) {
    m := make(map[string]int)
    for op := range ops {
        switch op.op {
        case "get":
            op.resp <- m[op.key]
        case "set":
            m[op.key] = op.value
        }
    }
}
```

---

## Performance Characteristics

### Time Complexity
| Operation | Average | Worst Case |
|-----------|---------|------------|
| Insert | O(1) | O(n) |
| Lookup | O(1) | O(n) |
| Delete | O(1) | O(n) |
| Iteration | O(n) | O(n) |

### Space Complexity
- O(n) where n is number of elements
- Additional overhead: ~48 bytes per map + bucket overhead
- Load factor kept around 6.5 elements per bucket

### Load Factor Impact
```go
// Low load factor: Fast but wastes memory
// High load factor: Memory efficient but slower

// Go's choice: 6.5 (good balance)
// Average 6.5 elements per bucket
// Max 8 elements per bucket before overflow
```

---

## Common Patterns

### Set Implementation
```go
// Use map[T]bool or map[T]struct{}
set := make(map[string]struct{})

// Add
set["item"] = struct{}{}

// Check membership
if _, exists := set["item"]; exists {
    fmt.Println("Found")
}

// Delete
delete(set, "item")

// Size
size := len(set)
```

### Counting Occurrences
```go
func countWords(words []string) map[string]int {
    counts := make(map[string]int)
    for _, word := range words {
        counts[word]++
    }
    return counts
}
```

### Grouping
```go
func groupByLength(words []string) map[int][]string {
    groups := make(map[int][]string)
    for _, word := range words {
        length := len(word)
        groups[length] = append(groups[length], word)
    }
    return groups
}
```

### Memoization
```go
var cache = make(map[int]int)

func fibonacci(n int) int {
    if n <= 1 {
        return n
    }
    if val, ok := cache[n]; ok {
        return val
    }
    result := fibonacci(n-1) + fibonacci(n-2)
    cache[n] = result
    return result
}
```

### Default Values
```go
// Method 1: Check and initialize
if _, ok := m[key]; !ok {
    m[key] = defaultValue
}

// Method 2: Use zero value
m[key] = m[key] + 1  // Works for int (zero value is 0)

// Method 3: Helper function
func getOrDefault(m map[string]int, key string, def int) int {
    if val, ok := m[key]; ok {
        return val
    }
    return def
}
```

---

## Best Practices

### 1. Pre-allocate with Size Hint
```go
// Bad: Multiple reallocations
m := make(map[string]int)
for i := 0; i < 10000; i++ {
    m[fmt.Sprintf("key%d", i)] = i
}

// Good: Single allocation
m := make(map[string]int, 10000)
for i := 0; i < 10000; i++ {
    m[fmt.Sprintf("key%d", i)] = i
}
```

### 2. Check Existence Before Use
```go
// Bad: Assumes key exists
value := m["key"]
result := value * 2  // Wrong if key doesn't exist

// Good: Check existence
if value, ok := m["key"]; ok {
    result := value * 2
}
```

### 3. Use struct{} for Sets
```go
// Bad: Wastes memory
set := make(map[string]bool)

// Good: Zero-size value
set := make(map[string]struct{})
// struct{} has size 0
```

### 4. Don't Rely on Iteration Order
```go
// Bad: Assumes order
keys := []string{}
for k := range m {
    keys = append(keys, k)
}
// keys order is random

// Good: Sort if order matters
sort.Strings(keys)
```

### 5. Protect Concurrent Access
```go
// Bad: Concurrent access
var m = make(map[string]int)
go func() { m["key"] = 1 }()
go func() { _ = m["key"] }()

// Good: Use mutex or sync.Map
var mu sync.RWMutex
var m = make(map[string]int)
```

---

## Common Pitfalls

### 1. Nil Map Assignment
```go
// Problem: Assignment to nil map panics
var m map[string]int
m["key"] = 1  // panic: assignment to entry in nil map

// Solution: Initialize map
m = make(map[string]int)
m["key"] = 1  // OK
```

### 2. Map is Not Copied
```go
// Problem: Maps are reference types
m1 := map[string]int{"a": 1}
m2 := m1
m2["a"] = 2
fmt.Println(m1["a"])  // 2 - m1 is modified!

// Solution: Deep copy
m2 := make(map[string]int, len(m1))
for k, v := range m1 {
    m2[k] = v
}
```

### 3. Modifying Map During Iteration
```go
// Problem: Undefined behavior
m := map[string]int{"a": 1, "b": 2, "c": 3}
for k := range m {
    if k == "b" {
        delete(m, k)  // Allowed but may skip elements
    }
}

// Solution: Collect keys first
toDelete := []string{}
for k := range m {
    if shouldDelete(k) {
        toDelete = append(toDelete, k)
    }
}
for _, k := range toDelete {
    delete(m, k)
}
```

### 4. Map Keys Must Be Comparable
```go
// Problem: Slice as key
// m := make(map[[]int]string)  // Compile error

// Solution: Convert to comparable type
m := make(map[string]string)
key := fmt.Sprintf("%v", []int{1, 2, 3})
m[key] = "value"
```

### 5. Concurrent Map Access
```go
// Problem: Data race
m := make(map[string]int)
go func() { m["key"] = 1 }()
go func() { _ = m["key"] }()  // panic or data race

// Solution: Use sync.Map or mutex
var sm sync.Map
go func() { sm.Store("key", 1) }()
go func() { sm.Load("key") }()
```

---

## sync.Map

### When to Use
- Multiple goroutines read, write, and overwrite entries
- Disjoint sets of keys (different goroutines use different keys)
- Read-heavy workloads

```go
var m sync.Map

// Store
m.Store("key", 42)

// Load
value, ok := m.Load("key")
if ok {
    fmt.Println(value.(int))
}

// LoadOrStore
actual, loaded := m.LoadOrStore("key", 42)
if loaded {
    fmt.Println("Key existed:", actual)
} else {
    fmt.Println("Key stored:", actual)
}

// LoadAndDelete
value, loaded := m.LoadAndDelete("key")

// Delete
m.Delete("key")

// Range
m.Range(func(key, value interface{}) bool {
    fmt.Println(key, value)
    return true  // Continue iteration
})
```

### Internal Structure
```go
// Simplified sync.Map structure
type Map struct {
    mu     Mutex
    read   atomic.Value  // readOnly map (lock-free reads)
    dirty  map[any]*entry  // Requires lock
    misses int           // Counter for promoting dirty to read
}

// Two-tier structure:
// 1. read: Lock-free for reads
// 2. dirty: Protected by mutex for writes
```

### Performance Characteristics
```go
// sync.Map vs map + RWMutex

// sync.Map better when:
// - Read-heavy (90%+ reads)
// - Disjoint key sets
// - Stable key set

// map + RWMutex better when:
// - Write-heavy
// - Overlapping key sets
// - Known key set
```

---

## Benchmarking

### Map vs Slice for Small Collections
```go
func BenchmarkMapLookup(b *testing.B) {
    m := make(map[int]int, 100)
    for i := 0; i < 100; i++ {
        m[i] = i
    }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        _ = m[50]
    }
}

func BenchmarkSliceLookup(b *testing.B) {
    s := make([]int, 100)
    for i := 0; i < 100; i++ {
        s[i] = i
    }
    b.ResetTimer()
    for i := 0; i < b.N; i++ {
        for j := range s {
            if s[j] == 50 {
                break
            }
        }
    }
}
// For n < 10-20, slice can be faster due to cache locality
```

### Pre-allocation Impact
```go
func BenchmarkMapNoPrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        m := make(map[int]int)
        for j := 0; j < 1000; j++ {
            m[j] = j
        }
    }
}

func BenchmarkMapPrealloc(b *testing.B) {
    for i := 0; i < b.N; i++ {
        m := make(map[int]int, 1000)
        for j := 0; j < 1000; j++ {
            m[j] = j
        }
    }
}
// Pre-allocation is ~30% faster
```

### sync.Map vs Mutex Map
```go
func BenchmarkSyncMap(b *testing.B) {
    var m sync.Map
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            m.Store(1, 1)
            m.Load(1)
        }
    })
}

func BenchmarkMutexMap(b *testing.B) {
    var mu sync.RWMutex
    m := make(map[int]int)
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            mu.Lock()
            m[1] = 1
            mu.Unlock()
            mu.RLock()
            _ = m[1]
            mu.RUnlock()
        }
    })
}
// sync.Map faster for read-heavy workloads
```

---

## Summary

### Key Concepts
1. **Hash Table**: Maps use hash tables with separate chaining
2. **Buckets**: Each bucket holds 8 key-value pairs
3. **Growth**: Incremental rehashing when load factor > 6.5
4. **Not Thread-Safe**: Use mutex or sync.Map for concurrent access
5. **Random Iteration**: Order is intentionally randomized

### Performance
- Average O(1) for insert, lookup, delete
- Load factor ~6.5 for good balance
- Pre-allocate when size is known
- Use struct{} for sets (zero size)

### Best Practices
1. Initialize before use (avoid nil map)
2. Check existence with comma-ok idiom
3. Pre-allocate with size hint
4. Don't rely on iteration order
5. Protect concurrent access
6. Use sync.Map for read-heavy concurrent workloads

### Common Mistakes
1. Assignment to nil map (panic)
2. Assuming maps are copied (they're not)
3. Using non-comparable keys
4. Concurrent access without synchronization
5. Relying on iteration order

# Pointers Deep Dive

## Table of Contents
- [Pointer Fundamentals](#pointer-fundamentals)
- [Internal Memory Model](#internal-memory-model)
- [Pointer Operations](#pointer-operations)
- [Stack vs Heap Allocation](#stack-vs-heap-allocation)
- [Escape Analysis](#escape-analysis)
- [Pointer Receivers vs Value Receivers](#pointer-receivers-vs-value-receivers)
- [Unsafe Pointers](#unsafe-pointers)
- [Pointer Patterns](#pointer-patterns)
- [Common Pitfalls](#common-pitfalls)
- [Performance Implications](#performance-implications)

---

## Pointer Fundamentals

### Definition
A pointer holds the memory address of a value. In Go, pointers are typed — a `*int` can only point to an `int`.

```go
var x int = 42
var p *int = &x   // p holds the address of x

fmt.Println(x)    // 42       — value
fmt.Println(&x)   // 0xc0000b4008 — address of x
fmt.Println(p)    // 0xc0000b4008 — same address
fmt.Println(*p)   // 42       — dereference: value at address
```

### Zero Value
```go
var p *int        // nil — zero value for any pointer type
fmt.Println(p)    // <nil>
fmt.Println(p == nil) // true

// Dereferencing nil panics
// *p = 1  // panic: runtime error: invalid memory address or nil pointer dereference
```

### new() vs &
```go
// new(T) allocates zeroed T, returns *T
p1 := new(int)       // *int pointing to 0
fmt.Println(*p1)     // 0

// & takes address of existing variable
x := 42
p2 := &x             // *int pointing to 42
fmt.Println(*p2)     // 42

// & on composite literal — most common
p3 := &struct{ x int }{x: 10}
p4 := &[]int{1, 2, 3}
p5 := &map[string]int{"a": 1}
```

---

## Internal Memory Model

### Pointer Size
```go
// Pointer size = word size of the machine
// 64-bit system: 8 bytes
// 32-bit system: 4 bytes

import "unsafe"
fmt.Println(unsafe.Sizeof((*int)(nil)))    // 8 (on 64-bit)
fmt.Println(unsafe.Sizeof((*string)(nil))) // 8 (same — all pointers same size)
```

### Memory Layout
```
x := 42
p := &x

Stack:
┌──────────────────────────────┐
│ x: 42          addr: 0x1000  │
│ p: 0x1000      addr: 0x1008  │
└──────────────────────────────┘

p stores the address 0x1000
*p reads the value at address 0x1000 → 42
```

### Pointer to Pointer
```go
x := 42
p := &x    // *int
pp := &p   // **int

fmt.Println(**pp)  // 42
**pp = 100
fmt.Println(x)     // 100

Memory:
x:  42       at 0x1000
p:  0x1000   at 0x1008
pp: 0x1008   at 0x1010
```

---

## Pointer Operations

### Dereferencing
```go
x := 10
p := &x

// Read through pointer
val := *p       // val = 10

// Write through pointer
*p = 20
fmt.Println(x)  // 20 — x is modified
```

### Pointer Arithmetic (Not Allowed in Go)
```go
// Go does NOT allow pointer arithmetic (unlike C)
p := &x
// p++       // Compile error
// p + 1     // Compile error

// Use unsafe.Pointer + uintptr for arithmetic (dangerous)
import "unsafe"
p2 := unsafe.Pointer(uintptr(unsafe.Pointer(p)) + unsafe.Sizeof(x))
// Only valid if you know the memory layout
```

### Comparing Pointers
```go
x, y := 10, 10
p1 := &x
p2 := &x
p3 := &y

fmt.Println(p1 == p2)  // true  — same address
fmt.Println(p1 == p3)  // false — different addresses
fmt.Println(*p1 == *p3) // true  — same value
```

### Passing Pointers to Functions
```go
// Value semantics — copy is made
func doubleValue(n int) {
    n *= 2  // Only modifies local copy
}

// Pointer semantics — original is modified
func doublePointer(n *int) {
    *n *= 2  // Modifies original
}

x := 5
doubleValue(x)
fmt.Println(x)   // 5 — unchanged

doublePointer(&x)
fmt.Println(x)   // 10 — modified
```

---

## Stack vs Heap Allocation

### Stack Allocation
```go
// Small, short-lived variables go on the stack
func stackAlloc() {
    x := 42          // Stack allocated
    arr := [10]int{} // Stack allocated (small, fixed size)
    _ = x
    _ = arr
}
// x and arr are freed when stackAlloc returns
```

### Heap Allocation
```go
// Variables that escape to heap are heap allocated
func heapAlloc() *int {
    x := 42
    return &x   // x escapes to heap — must outlive the function
}

p := heapAlloc()
fmt.Println(*p)  // 42 — still valid after function returns
```

### What Causes Heap Allocation?
```go
// 1. Returning pointer to local variable
func f1() *int {
    x := 1
    return &x  // x escapes
}

// 2. Storing in interface
func f2() interface{} {
    x := 1
    return x   // x may escape (depends on size)
}

// 3. Sending to channel
func f3(ch chan *int) {
    x := 1
    ch <- &x   // x escapes
}

// 4. Closure captures variable
func f4() func() int {
    x := 1
    return func() int { return x }  // x escapes
}

// 5. Large objects (compiler decides)
func f5() {
    arr := [1000000]int{}  // May escape to heap
    _ = arr
}
```

---

## Escape Analysis

### What is Escape Analysis?
The Go compiler determines at compile time whether a variable can be allocated on the stack or must be allocated on the heap.

```bash
# View escape analysis decisions
go build -gcflags="-m" ./...
go build -gcflags="-m -m" ./...  # More verbose
```

### Example Output
```go
package main

func createInt() *int {
    x := 42      // "x escapes to heap"
    return &x
}

func noEscape() int {
    x := 42      // stays on stack
    return x
}
```

```bash
$ go build -gcflags="-m" main.go
./main.go:4:2: moved to heap: x
```

### Escape Analysis Rules
```go
// Does NOT escape (stack):
func f() {
    x := 42
    fmt.Println(x)   // x stays on stack
}

// DOES escape (heap):
func g() *int {
    x := 42
    return &x        // x must outlive g()
}

// Interface boxing may cause escape:
var i interface{} = 42  // 42 may escape to heap

// Slice backing array may escape:
s := make([]int, 0, 1000)  // backing array on heap
```

### Checking Escape Analysis
```go
// go build -gcflags="-m=2" shows detailed escape analysis
// Look for:
// "moved to heap: x"     — variable escapes
// "x does not escape"    — stays on stack
// "inlining call to f"   — function inlined (no stack frame)
```

---

## Pointer Receivers vs Value Receivers

### Value Receiver — Copy
```go
type Counter struct {
    count int
}

// Value receiver — works on a copy
func (c Counter) Value() int {
    return c.count
}

func (c Counter) Increment() {
    c.count++  // Modifies copy, NOT original
}

c := Counter{count: 0}
c.Increment()
fmt.Println(c.count)  // 0 — unchanged!
```

### Pointer Receiver — Original
```go
// Pointer receiver — works on original
func (c *Counter) Increment() {
    c.count++  // Modifies original
}

func (c *Counter) Reset() {
    c.count = 0
}

c := Counter{count: 0}
c.Increment()
fmt.Println(c.count)  // 1 — modified!
```

### When to Use Each
```go
// Use pointer receiver when:
// 1. Method modifies the receiver
func (c *Counter) Increment() { c.count++ }

// 2. Receiver is large (avoid copying)
type LargeStruct struct {
    data [1024]byte
}
func (l *LargeStruct) Process() { /* ... */ }

// 3. Consistency — if any method uses pointer, use all pointer
type File struct { /* ... */ }
func (f *File) Read() []byte  { /* ... */ }
func (f *File) Write([]byte)  { /* ... */ }
func (f *File) Close() error  { /* ... */ }

// Use value receiver when:
// 1. Method doesn't modify receiver
func (c Counter) Value() int { return c.count }

// 2. Receiver is small and copying is cheap
type Point struct { x, y float64 }
func (p Point) Distance() float64 { return math.Sqrt(p.x*p.x + p.y*p.y) }

// 3. Receiver is a basic type (int, string, etc.)
type Celsius float64
func (c Celsius) ToFahrenheit() float64 { return float64(c)*9/5 + 32 }
```

### Method Set Rules
```go
// Value T has method set: value receiver methods only
// Pointer *T has method set: both value and pointer receiver methods

type T struct{}
func (t T) ValueMethod()   {}
func (t *T) PtrMethod()    {}

var v T
v.ValueMethod()   // OK
v.PtrMethod()     // OK — Go auto-takes address: (&v).PtrMethod()

var p *T = &T{}
p.ValueMethod()   // OK — Go auto-dereferences: (*p).ValueMethod()
p.PtrMethod()     // OK

// Interface satisfaction:
type I interface {
    PtrMethod()
}
var _ I = &T{}  // OK — *T implements I
// var _ I = T{}  // ERROR — T does not implement I (PtrMethod has pointer receiver)
```

---

## Unsafe Pointers

### unsafe.Pointer
```go
import "unsafe"

// unsafe.Pointer can hold address of any type
// Bridges between typed pointers and uintptr

x := 42
p := unsafe.Pointer(&x)  // Convert *int to unsafe.Pointer

// Convert back to typed pointer
p2 := (*int)(p)
fmt.Println(*p2)  // 42
```

### uintptr for Arithmetic
```go
type Point struct {
    X, Y int
}

p := &Point{X: 1, Y: 2}

// Access Y field via pointer arithmetic
yPtr := (*int)(unsafe.Pointer(
    uintptr(unsafe.Pointer(p)) + unsafe.Offsetof(p.Y),
))
fmt.Println(*yPtr)  // 2

// DANGER: uintptr is NOT a pointer — GC can move objects
// Never store uintptr across GC points
```

### unsafe.Sizeof, Alignof, Offsetof
```go
type Example struct {
    A bool    // 1 byte
    B int32   // 4 bytes
    C int64   // 8 bytes
}

fmt.Println(unsafe.Sizeof(Example{}))      // 16 (with padding)
fmt.Println(unsafe.Alignof(Example{}))     // 8
fmt.Println(unsafe.Offsetof(Example{}.A))  // 0
fmt.Println(unsafe.Offsetof(Example{}.B))  // 4 (3 bytes padding after A)
fmt.Println(unsafe.Offsetof(Example{}.C))  // 8
```

### String ↔ []byte Without Copy
```go
// Zero-copy conversion using unsafe (use with caution)
func stringToBytes(s string) []byte {
    return unsafe.Slice(unsafe.StringData(s), len(s))
}

func bytesToString(b []byte) string {
    return unsafe.String(unsafe.SliceData(b), len(b))
}
// Only safe if you don't modify the []byte after conversion
```

---

## Pointer Patterns

### Optional Values
```go
// Use pointer to represent optional/nullable values
type Config struct {
    Timeout *int    // nil means "use default"
    Debug   *bool   // nil means "not set"
}

timeout := 30
cfg := Config{Timeout: &timeout}

if cfg.Timeout != nil {
    fmt.Println("Timeout:", *cfg.Timeout)
} else {
    fmt.Println("Using default timeout")
}
```

### In-place Modification
```go
func normalize(s *string) {
    *s = strings.TrimSpace(strings.ToLower(*s))
}

name := "  Alice  "
normalize(&name)
fmt.Println(name)  // "alice"
```

### Linked Structures
```go
type Node struct {
    Val  int
    Next *Node  // Pointer enables recursive structure
}

head := &Node{Val: 1, Next: &Node{Val: 2, Next: &Node{Val: 3}}}
```

### Functional Options Pattern
```go
type Server struct {
    host    string
    port    int
    timeout *time.Duration
}

type Option func(*Server)

func WithTimeout(d time.Duration) Option {
    return func(s *Server) {
        s.timeout = &d
    }
}

func NewServer(host string, port int, opts ...Option) *Server {
    s := &Server{host: host, port: port}
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

---

## Common Pitfalls

### 1. Returning Pointer to Loop Variable
```go
// Problem: all pointers point to same variable
ptrs := make([]*int, 3)
for i := 0; i < 3; i++ {
    ptrs[i] = &i  // All point to same i!
}
// After loop: i = 3
fmt.Println(*ptrs[0], *ptrs[1], *ptrs[2])  // 3 3 3

// Fix: create new variable each iteration
for i := 0; i < 3; i++ {
    i := i          // Shadow with new variable
    ptrs[i] = &i
}
fmt.Println(*ptrs[0], *ptrs[1], *ptrs[2])  // 0 1 2
```

### 2. Nil Pointer Dereference
```go
var p *int
fmt.Println(*p)  // panic: nil pointer dereference

// Always check before dereferencing
if p != nil {
    fmt.Println(*p)
}
```

### 3. Pointer to Interface (Usually Wrong)
```go
type Animal interface { Speak() string }
type Dog struct{}
func (d Dog) Speak() string { return "Woof" }

// Wrong: pointer to interface
var a *Animal  // Almost never what you want

// Right: interface holding pointer
var a Animal = &Dog{}
```

### 4. Modifying Slice Through Pointer
```go
// Pointer to slice vs pointer to element
s := []int{1, 2, 3}
p := &s[0]   // Pointer to first element

s = append(s, 4, 5, 6)  // May reallocate!
*p = 100                  // May write to old backing array!

// Safe: use index, not pointer to element
idx := 0
s[idx] = 100  // Always correct
```

---

## Performance Implications

### Pointer vs Value for Function Arguments
```go
type SmallStruct struct{ x, y int }      // 16 bytes
type LargeStruct struct{ data [1024]byte } // 1024 bytes

// Small struct: value is faster (no indirection)
func processSmall(s SmallStruct) {}      // Copy 16 bytes — fast
func processSmallPtr(s *SmallStruct) {}  // Pass 8 bytes + indirection

// Large struct: pointer is faster (avoid large copy)
func processLarge(s LargeStruct) {}      // Copy 1024 bytes — slow
func processLargePtr(s *LargeStruct) {}  // Pass 8 bytes — fast

// Rule of thumb: use pointer for structs > ~64 bytes
```

### Pointer and GC Pressure
```go
// More pointers = more GC work
// GC must scan all pointers to find live objects

// Prefer value types in hot paths
type Node struct {
    Val  int
    Next *Node  // Each pointer adds GC scanning overhead
}

// For performance-critical code, consider:
// - Flat arrays instead of pointer-linked structures
// - sync.Pool to reuse allocations
// - Avoid unnecessary heap allocations
```

### Benchmarking
```go
func BenchmarkValueArg(b *testing.B) {
    s := LargeStruct{}
    for i := 0; i < b.N; i++ {
        processLarge(s)   // Copies 1024 bytes each call
    }
}

func BenchmarkPointerArg(b *testing.B) {
    s := &LargeStruct{}
    for i := 0; i < b.N; i++ {
        processLargePtr(s)  // Passes 8-byte pointer
    }
}
// Pointer version is ~10-50x faster for large structs
```

---

## Summary

### Key Concepts
1. Pointer holds a memory address; `*p` dereferences it
2. `&x` takes the address of `x`; `new(T)` allocates zeroed T
3. Go has no pointer arithmetic (use `unsafe` only when necessary)
4. Escape analysis decides stack vs heap allocation
5. Pointer receivers modify the original; value receivers work on a copy
6. `*T` satisfies interfaces with pointer receivers; `T` does not

### Decision Guide
```
Modify receiver?          → Pointer receiver
Large struct (> ~64B)?    → Pointer receiver/argument
Optional/nullable value?  → *T (pointer)
Recursive data structure? → *T (pointer)
Small, immutable value?   → Value receiver/argument
Interface satisfaction?   → Check method set rules
```

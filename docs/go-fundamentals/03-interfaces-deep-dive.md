# Interfaces Deep Dive

## Table of Contents
- [Interface Fundamentals](#interface-fundamentals)
- [Internal Implementation](#internal-implementation)
- [Interface Satisfaction](#interface-satisfaction)
- [Empty Interface](#empty-interface)
- [Type Assertions](#type-assertions)
- [Type Switches](#type-switches)
- [Interface Composition](#interface-composition)
- [Interface vs Concrete Types](#interface-vs-concrete-types)
- [Common Standard Interfaces](#common-standard-interfaces)
- [Interface Design Principles](#interface-design-principles)
- [Common Patterns](#common-patterns)
- [Common Pitfalls](#common-pitfalls)
- [Performance Implications](#performance-implications)

---

## Interface Fundamentals

### Definition
An interface defines a set of method signatures. Any type that implements all methods satisfies the interface — implicitly, with no `implements` keyword.

```go
type Animal interface {
    Speak() string
    Move() string
}

type Dog struct{ Name string }
func (d Dog) Speak() string { return "Woof!" }
func (d Dog) Move() string  { return "runs" }

type Bird struct{ Name string }
func (b Bird) Speak() string { return "Tweet!" }
func (b Bird) Move() string  { return "flies" }

// Both Dog and Bird satisfy Animal — implicitly
var a Animal = Dog{Name: "Rex"}
fmt.Println(a.Speak())  // Woof!

a = Bird{Name: "Tweety"}
fmt.Println(a.Speak())  // Tweet!
```

### Implicit Satisfaction
```go
// No "implements" keyword needed
// If a type has the methods, it satisfies the interface

type Stringer interface {
    String() string
}

type Point struct{ X, Y int }

// Point satisfies Stringer by having String() method
func (p Point) String() string {
    return fmt.Sprintf("(%d, %d)", p.X, p.Y)
}

var s Stringer = Point{1, 2}
fmt.Println(s.String())  // (1, 2)
```

---

## Internal Implementation

### iface and eface
Go represents interfaces as two-word structs internally.

```go
// Non-empty interface (runtime/iface.go)
type iface struct {
    tab  *itab          // Type information + method table
    data unsafe.Pointer // Pointer to the actual value
}

// Empty interface (runtime/iface.go)
type eface struct {
    _type *_type        // Type information
    data  unsafe.Pointer // Pointer to the actual value
}
```

### itab Structure
```go
// itab holds type info and method pointers
type itab struct {
    inter *interfacetype  // Interface type descriptor
    _type *_type          // Concrete type descriptor
    hash  uint32          // Copy of _type.hash for type switches
    _     [4]byte
    fun   [1]uintptr      // Method table (variable length)
}
```

### Memory Layout
```
Interface value: var a Animal = Dog{Name: "Rex"}

iface:
┌──────────────────┬──────────────────┐
│   tab (*itab)    │   data (ptr)     │
│   ┌──────────┐   │   ┌──────────┐   │
│   │ inter    │   │   │ Dog{     │   │
│   │ _type    │   │   │  Name:   │   │
│   │ hash     │   │   │  "Rex"   │   │
│   │ fun[0]   │   │   │ }        │   │
│   │ fun[1]   │   │   └──────────┘   │
│   └──────────┘   │                  │
└──────────────────┴──────────────────┘
```

### Small Value Optimization
```go
// Small values (≤ pointer size) stored directly in data field
// Larger values stored on heap, data holds pointer

var i interface{} = 42      // 42 stored directly in data
var j interface{} = "hello" // string header stored in data
var k interface{} = [100]int{} // array on heap, data = pointer
```

### Interface Value Equality
```go
// Two interface values are equal if:
// 1. Both nil
// 2. Same dynamic type AND same dynamic value

var a, b Animal
fmt.Println(a == b)  // true — both nil

a = Dog{Name: "Rex"}
b = Dog{Name: "Rex"}
fmt.Println(a == b)  // true — same type, same value

b = Dog{Name: "Buddy"}
fmt.Println(a == b)  // false — same type, different value

b = Bird{Name: "Rex"}
fmt.Println(a == b)  // false — different type
```

---

## Interface Satisfaction

### Compile-time Check
```go
// Explicit compile-time interface satisfaction check
type MyWriter struct{}
func (w *MyWriter) Write(p []byte) (int, error) { return len(p), nil }

// This line causes compile error if MyWriter doesn't satisfy io.Writer
var _ io.Writer = (*MyWriter)(nil)
// or
var _ io.Writer = &MyWriter{}
```

### Value vs Pointer Receiver
```go
type Counter struct{ n int }

func (c *Counter) Increment() { c.n++ }  // Pointer receiver
func (c Counter) Value() int  { return c.n }  // Value receiver

type Incrementer interface { Increment() }
type Valuer interface     { Value() int }

var c Counter

// *Counter satisfies both interfaces
var _ Incrementer = &c  // OK
var _ Valuer      = &c  // OK

// Counter satisfies only Valuer (no pointer receiver methods)
var _ Valuer      = c   // OK
// var _ Incrementer = c  // ERROR: Counter does not implement Incrementer
```

### Method Set Rules
```
Type T:
  - Value receiver methods of T

Type *T:
  - Value receiver methods of T
  - Pointer receiver methods of T

Interface variable holding T:
  - Can only call value receiver methods

Interface variable holding *T:
  - Can call both value and pointer receiver methods
```

---

## Empty Interface

### any / interface{}
```go
// interface{} (or 'any' since Go 1.18) accepts any value
var x interface{}
x = 42
x = "hello"
x = []int{1, 2, 3}
x = nil

// any is an alias for interface{}
var y any = 42
```

### Type Information is Preserved
```go
var x interface{} = 42
fmt.Printf("%T\n", x)  // int — type is preserved
fmt.Printf("%v\n", x)  // 42  — value is preserved
```

### Passing Any Type
```go
func printAnything(v interface{}) {
    fmt.Printf("Type: %T, Value: %v\n", v, v)
}

printAnything(42)           // Type: int, Value: 42
printAnything("hello")      // Type: string, Value: hello
printAnything([]int{1,2,3}) // Type: []int, Value: [1 2 3]
```

---

## Type Assertions

### Syntax
```go
// x.(T) — asserts x holds value of type T
var i interface{} = "hello"

s := i.(string)         // s = "hello"
fmt.Println(s, len(s))  // hello 5

// Panics if wrong type
// n := i.(int)  // panic: interface conversion: interface {} is string, not int
```

### Safe Type Assertion (Comma-ok)
```go
var i interface{} = "hello"

s, ok := i.(string)
fmt.Println(s, ok)  // hello true

n, ok := i.(int)
fmt.Println(n, ok)  // 0 false — no panic
```

### Asserting to Interface
```go
type Stringer interface { String() string }
type Writer  interface { Write([]byte) (int, error) }

var i interface{} = os.Stdout  // *os.File

// Assert to interface type
if w, ok := i.(Writer); ok {
    w.Write([]byte("hello"))
}

if s, ok := i.(Stringer); ok {
    fmt.Println(s.String())
}
```

---

## Type Switches

### Basic Type Switch
```go
func describe(i interface{}) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case string:
        return fmt.Sprintf("string: %q (len=%d)", v, len(v))
    case bool:
        return fmt.Sprintf("bool: %t", v)
    case []int:
        return fmt.Sprintf("[]int with %d elements", len(v))
    case nil:
        return "nil"
    default:
        return fmt.Sprintf("unknown type: %T", v)
    }
}

fmt.Println(describe(42))          // int: 42
fmt.Println(describe("hello"))     // string: "hello" (len=5)
fmt.Println(describe(nil))         // nil
```

### Multiple Types per Case
```go
func isNumber(i interface{}) bool {
    switch i.(type) {
    case int, int8, int16, int32, int64,
        uint, uint8, uint16, uint32, uint64,
        float32, float64:
        return true
    default:
        return false
    }
}
```

### Type Switch with Interface
```go
type Stringer interface { String() string }
type Sizer   interface { Size() int }

func process(i interface{}) {
    switch v := i.(type) {
    case Stringer:
        fmt.Println("Stringer:", v.String())
    case Sizer:
        fmt.Println("Sizer:", v.Size())
    default:
        fmt.Printf("Unknown: %T\n", v)
    }
}
```

---

## Interface Composition

### Embedding Interfaces
```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

// Compose from smaller interfaces
type ReadWriter interface {
    Reader
    Writer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// Standard library examples:
// io.Reader, io.Writer, io.Closer
// io.ReadWriter, io.ReadCloser, io.WriteCloser
// io.ReadWriteCloser
```

### Building Interfaces Incrementally
```go
// Start small, compose as needed
type Opener interface {
    Open() error
}

type Processor interface {
    Process(data []byte) error
}

type Pipeline interface {
    Opener
    Processor
    Closer
}
```

---

## Interface vs Concrete Types

### When to Use Interface
```go
// 1. Multiple implementations expected
type Storage interface {
    Get(key string) ([]byte, error)
    Set(key string, val []byte) error
}

type MemoryStorage struct{ data map[string][]byte }
type RedisStorage  struct{ client *redis.Client }
type FileStorage   struct{ dir string }

// All satisfy Storage — swap implementations easily

// 2. Testing / mocking
type EmailSender interface {
    Send(to, subject, body string) error
}

type MockEmailSender struct {
    SentEmails []string
}
func (m *MockEmailSender) Send(to, subject, body string) error {
    m.SentEmails = append(m.SentEmails, to)
    return nil
}

// 3. Decoupling packages
// Accept interfaces, return concrete types (Rob Pike's advice)
func NewProcessor(r io.Reader) *Processor { /* ... */ }
```

### When NOT to Use Interface
```go
// Don't create interface for single implementation
// (premature abstraction)

// Bad: unnecessary interface
type UserServiceInterface interface {
    GetUser(id int) (*User, error)
    CreateUser(u *User) error
}
type UserService struct{ db *sql.DB }
// Only one implementation — just use *UserService directly

// Good: use concrete type, add interface when needed
type UserService struct{ db *sql.DB }
func (s *UserService) GetUser(id int) (*User, error) { /* ... */ }
```

---

## Common Standard Interfaces

### io.Reader / io.Writer
```go
// io.Reader — anything that can be read from
type Reader interface {
    Read(p []byte) (n int, err error)
}

// io.Writer — anything that can be written to
type Writer interface {
    Write(p []byte) (n int, err error)
}

// Implementations: os.File, bytes.Buffer, strings.Reader,
//                  net.Conn, http.ResponseWriter, gzip.Writer, ...

func copyData(dst io.Writer, src io.Reader) error {
    _, err := io.Copy(dst, src)
    return err
}
// Works with any combination of Reader/Writer implementations
```

### fmt.Stringer
```go
type Stringer interface {
    String() string
}

type Color int
const (Red Color = iota; Green; Blue)

func (c Color) String() string {
    switch c {
    case Red:   return "Red"
    case Green: return "Green"
    case Blue:  return "Blue"
    default:    return "Unknown"
    }
}

fmt.Println(Red)    // Red — uses String() automatically
fmt.Println(Green)  // Green
```

### error
```go
// The built-in error interface
type error interface {
    Error() string
}

// Custom error type
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: %s — %s", e.Field, e.Message)
}

func validate(name string) error {
    if name == "" {
        return &ValidationError{Field: "name", Message: "cannot be empty"}
    }
    return nil
}
```

### sort.Interface
```go
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}

type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }

people := []Person{{"Alice", 30}, {"Bob", 25}, {"Carol", 35}}
sort.Sort(ByAge(people))
// [{Bob 25} {Alice 30} {Carol 35}]

// Simpler with sort.Slice (Go 1.8+)
sort.Slice(people, func(i, j int) bool {
    return people[i].Age < people[j].Age
})
```

### http.Handler
```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// HandlerFunc adapts a function to http.Handler
type HandlerFunc func(ResponseWriter, *Request)
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) { f(w, r) }

// Usage
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "Hello, World!")
})
```

---

## Interface Design Principles

### Accept Interfaces, Return Concrete Types
```go
// Good: accept interface (flexible input)
func Save(w io.Writer, data []byte) error {
    _, err := w.Write(data)
    return err
}

// Good: return concrete type (caller knows what they get)
func NewBuffer() *bytes.Buffer {
    return &bytes.Buffer{}
}

// Bad: return interface (hides useful methods)
func NewBuffer() io.Writer {  // Caller can't use Buffer-specific methods
    return &bytes.Buffer{}
}
```

### Keep Interfaces Small
```go
// Good: small, focused interfaces
type Saver interface   { Save() error }
type Loader interface  { Load() error }
type Deleter interface { Delete() error }

// Compose when needed
type Repository interface {
    Saver
    Loader
    Deleter
}

// Bad: large interface (hard to implement, hard to mock)
type GodInterface interface {
    Save() error
    Load() error
    Delete() error
    List() ([]Item, error)
    Count() int
    Exists(id string) bool
    // ... 20 more methods
}
```

### Define Interfaces at Point of Use
```go
// Bad: define interface in the package that implements it
// package storage
type Storage interface { Get(key string) []byte }
type RedisStorage struct{}
func (r *RedisStorage) Get(key string) []byte { /* ... */ }

// Good: define interface where it's consumed
// package service
type cache interface { Get(key string) []byte }  // Defined here
type Service struct { cache cache }
// Now any type with Get() works — including RedisStorage
```

---

## Common Patterns

### Middleware / Decorator
```go
type Handler interface {
    Handle(req Request) Response
}

// Logging middleware
type LoggingHandler struct {
    next   Handler
    logger *log.Logger
}

func (h *LoggingHandler) Handle(req Request) Response {
    h.logger.Printf("Request: %v", req)
    resp := h.next.Handle(req)
    h.logger.Printf("Response: %v", resp)
    return resp
}

// Chain middlewares
handler := &LoggingHandler{
    next: &AuthHandler{
        next: &RealHandler{},
    },
}
```

### Strategy Pattern
```go
type SortStrategy interface {
    Sort(data []int)
}

type BubbleSort struct{}
func (b BubbleSort) Sort(data []int) { /* bubble sort */ }

type QuickSort struct{}
func (q QuickSort) Sort(data []int) { /* quick sort */ }

type Sorter struct {
    strategy SortStrategy
}

func (s *Sorter) SetStrategy(strategy SortStrategy) {
    s.strategy = strategy
}

func (s *Sorter) Sort(data []int) {
    s.strategy.Sort(data)
}
```

### Nil Interface Check
```go
// Checking if interface is nil
func process(r io.Reader) {
    if r == nil {
        return
    }
    // ...
}

// WARNING: typed nil is NOT nil interface
type MyReader struct{}
func (r *MyReader) Read(p []byte) (int, error) { return 0, nil }

var r *MyReader = nil
var i io.Reader = r

fmt.Println(r == nil)  // true
fmt.Println(i == nil)  // false! — interface has type info, data is nil
```

---

## Common Pitfalls

### 1. Nil Interface vs Nil Pointer in Interface
```go
// The most common Go gotcha
func getError() error {
    var err *MyError = nil
    return err  // Returns non-nil interface holding nil pointer!
}

err := getError()
fmt.Println(err == nil)  // false — SURPRISE!

// Fix: return untyped nil
func getError() error {
    var err *MyError = nil
    if err == nil {
        return nil  // Return untyped nil
    }
    return err
}
```

### 2. Interface Comparison Panic
```go
// Comparing interfaces with non-comparable dynamic types panics
var a, b interface{}
a = []int{1, 2, 3}
b = []int{1, 2, 3}
// fmt.Println(a == b)  // panic: runtime error: comparing uncomparable type []int

// Safe comparison
fmt.Println(reflect.DeepEqual(a, b))  // true
```

### 3. Pointer to Interface
```go
// Almost never correct
var w *io.Writer  // Pointer to interface — usually wrong

// What you want:
var w io.Writer = os.Stdout  // Interface holding pointer

// Pointer to interface is only useful in rare cases
// (e.g., passing interface to be set by callee)
```

### 4. Storing Non-Pointer in Interface for Mutation
```go
type Counter struct{ n int }
func (c *Counter) Inc() { c.n++ }

var i interface{} = Counter{}  // Stores value copy
// Can't call Inc() — Counter doesn't have Inc(), *Counter does
// And even if you extract it, you'd modify a copy

// Fix: store pointer
var i interface{} = &Counter{}
i.(*Counter).Inc()
```

---

## Performance Implications

### Interface Call Overhead
```go
// Direct call: ~1ns
c := &Counter{}
c.Increment()

// Interface call: ~2-5ns (indirect dispatch via itab)
var i Incrementer = &Counter{}
i.Increment()

// The overhead comes from:
// 1. Loading itab pointer
// 2. Loading method pointer from itab
// 3. Indirect call through method pointer
```

### Interface Boxing Allocations
```go
// Storing value in interface may allocate
var i interface{} = 42        // No alloc — int fits in pointer
var j interface{} = "hello"   // No alloc — string header fits
var k interface{} = [100]int{} // Alloc — too large for pointer

// Benchmark to check
func BenchmarkBoxing(b *testing.B) {
    var x interface{}
    for i := 0; i < b.N; i++ {
        x = [100]int{}  // Allocates each iteration
    }
    _ = x
}
```

### Inlining and Devirtualization
```go
// Go compiler can devirtualize interface calls when type is known
// at compile time (monomorphization)

// This may be inlined/devirtualized:
func process(w io.Writer) {
    w.Write([]byte("hello"))
}
process(os.Stdout)  // Compiler may know it's *os.File

// Use -gcflags="-m" to see inlining decisions
```

---

## Summary

### Key Concepts
1. Interfaces are satisfied implicitly — no `implements` keyword
2. Interface value = (type pointer, data pointer) — two words
3. Nil interface ≠ interface holding nil pointer (common gotcha)
4. `*T` satisfies interfaces with pointer receivers; `T` does not
5. Type assertions extract concrete value; type switches dispatch on type
6. Keep interfaces small and define them at point of use

### Decision Guide
```
Need polymorphism?                    → interface
Need to mock for testing?             → interface
Multiple implementations expected?   → interface
Single implementation, no testing?   → concrete type
Need to check type at runtime?        → type assertion / type switch
Need to compose behaviors?            → interface embedding
Accepting any type?                   → any / interface{}
```

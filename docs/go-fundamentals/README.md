# Go Fundamentals Deep Dive

Comprehensive in-depth documentation for core Go language features — pointers, structs, interfaces, functions/closures, types/generics, and error handling — covering internal implementations, memory models, patterns, and common pitfalls.

## Table of Contents

1. [Pointers](#1-pointers)
2. [Structs](#2-structs)
3. [Interfaces](#3-interfaces)
4. [Functions and Closures](#4-functions-and-closures)
5. [Types and Generics](#5-types-and-generics)
6. [Error Handling](#6-error-handling)

---

## 1. Pointers
**File**: [01-pointers-deep-dive.md](./01-pointers-deep-dive.md)

### Topics Covered
- Pointer declaration, dereferencing, `&` and `new()`
- Pointer size (8 bytes on 64-bit), pointer-to-pointer
- Stack vs heap allocation — what causes heap escape
- Escape analysis with `go build -gcflags="-m"`
- Pointer receivers vs value receivers — method set rules
- `unsafe.Pointer` and `uintptr` for arithmetic
- `unsafe.Sizeof`, `Alignof`, `Offsetof`
- Zero-copy string ↔ []byte conversion
- Common pitfalls: loop variable capture, nil dereference, pointer to interface

### Key Concepts
```go
// Escape analysis — variable moves to heap when it outlives its function
func heapAlloc() *int {
    x := 42
    return &x   // x escapes to heap
}

// Pointer receiver — modifies original
func (c *Counter) Increment() { c.count++ }

// Value receiver — works on copy
func (c Counter) Value() int { return c.count }

// *T satisfies interfaces with pointer receivers; T does not
var _ Incrementer = &Counter{}  // OK
// var _ Incrementer = Counter{} // ERROR
```

---

## 2. Structs
**File**: [02-structs-deep-dive.md](./02-structs-deep-dive.md)

### Topics Covered
- Struct as value type — assignment copies all fields
- Contiguous memory layout and field offsets
- Struct padding and alignment rules
- Field ordering to minimize padding (largest → smallest)
- Empty struct `struct{}` — zero size, uses as set/signal
- Initialization: named fields, `new()`, constructor functions
- Functional options pattern
- Methods: value vs pointer receivers, method expressions
- Embedding: composition, promoted fields/methods, shadowing
- Embedding interfaces in structs (for wrapping/mocking)
- Anonymous structs for test tables and one-off types
- Struct tags: json, db, yaml, xml, validate
- Struct comparison — comparable vs non-comparable
- Patterns: builder, immutable, copy-on-write
- Common pitfalls: copying mutex, nil pointer, unexported fields + JSON

### Key Concepts
```go
// Padding: order fields largest → smallest
type Good struct {
    B int64   // 8 bytes at offset 0
    A bool    // 1 byte  at offset 8
    C bool    // 1 byte  at offset 9
}             // 16 bytes total (vs 24 for bad ordering)

// Embedding — composition, not inheritance
type Dog struct {
    Animal        // Promoted fields and methods
    Breed string
}
d.Name   // Promoted from Animal
d.Speak() // Dog's method shadows Animal's

// Functional options
func NewServer(host string, opts ...Option) *Server
```

---

## 3. Interfaces
**File**: [03-interfaces-deep-dive.md](./03-interfaces-deep-dive.md)

### Topics Covered
- Implicit satisfaction — no `implements` keyword
- Internal representation: `iface` (tab + data) and `eface`
- `itab` structure: interface type, concrete type, method table
- Small value optimization in interface data field
- Interface value equality rules
- Compile-time satisfaction check: `var _ io.Writer = (*T)(nil)`
- Value vs pointer receiver and method set rules
- `any` / `interface{}` — empty interface internals
- Type assertions: panicking and comma-ok forms
- Type switches: single type, multiple types, interface types
- Interface composition via embedding
- Accept interfaces, return concrete types (Rob Pike)
- Keep interfaces small — single responsibility
- Define interfaces at point of use (consumer, not producer)
- Common standard interfaces: io.Reader/Writer, fmt.Stringer, error, sort.Interface, http.Handler
- Patterns: middleware/decorator, strategy
- Pitfalls: nil interface vs nil pointer in interface, pointer to interface, interface comparison panic

### Key Concepts
```go
// iface — two words: type info + data pointer
// Nil interface ≠ interface holding nil pointer (most common gotcha)
func getError() error {
    var err *MyError = nil
    return nil  // Must return untyped nil, not typed nil
}

// Type assertion
s, ok := i.(string)

// Type switch
switch v := i.(type) {
case int:    fmt.Println("int:", v)
case string: fmt.Println("string:", v)
}

// Interface composition
type ReadWriter interface { Reader; Writer }
```

---

## 4. Functions and Closures
**File**: [04-functions-closures-deep-dive.md](./04-functions-closures-deep-dive.md)

### Topics Covered
- Function declaration, signatures, named return values
- Stack frames and goroutine growable stacks
- Register-based calling convention (Go 1.17+)
- Multiple return values — idiomatic `(value, error)` pattern
- Named returns with defer for cleanup/transaction patterns
- Variadic functions — `...T` syntax, spreading slices, no allocation on spread
- First-class functions: assign, pass, return
- Higher-order functions: map, filter, reduce
- Closures: capture by reference, mutable state, shared variables
- Closure internals: heap allocation of captured variables, escape analysis
- Defer: LIFO order, argument evaluation timing, cleanup patterns
- Defer in loops — common mistake and fix
- Panic: any value, stack unwinding, deferred functions run
- Recover: only in defer, converting panic to error
- Recursion: tail calls not optimized, memoization pattern
- Patterns: middleware chain, option functions, retry, once

### Key Concepts
```go
// Closures capture by reference
x := 10
fn := func() { fmt.Println(x) }
x = 20
fn()  // Prints 20, not 10

// Defer: LIFO, args evaluated immediately
defer fmt.Println(x)  // Captures x=20 now

// Loop variable capture — classic bug
for i := 0; i < 3; i++ {
    i := i  // Shadow to capture current value
    go func() { fmt.Println(i) }()
}

// Recover only works in defer
defer func() {
    if r := recover(); r != nil { /* handle */ }
}()
```

---

## 5. Types and Generics
**File**: [05-types-generics-deep-dive.md](./05-types-generics-deep-dive.md)

### Topics Covered
- Built-in types: sizes, zero values
- Named types — distinct from underlying type, can have methods
- Type aliases (`=`) — same type, just another name
- Explicit type conversions — no implicit conversions in Go
- Type assertions and reflection basics
- Generics (Go 1.18+): type parameters `[T constraint]`
- Type constraints: `any`, `comparable`, union types `int | float64`
- Tilde `~T` — includes all types with underlying type T
- `golang.org/x/exp/constraints`: Integer, Float, Ordered, Signed
- Generic functions: Map, Filter, Reduce, Keys, Values, Contains
- Generic types: Stack[T], Set[T comparable], Pair[A,B], Result[T], Cache[K,V]
- Type inference — automatic from arguments, explicit for return-only params
- `comparable` vs `constraints.Ordered`
- Common generic patterns: Optional[T], generic cache, generic result type
- Reflection: TypeOf, ValueOf, struct inspection, dynamic method calls
- Pitfalls: operators on unconstrained T, type param as interface, reflect on unexported fields

### Key Concepts
```go
// Named type — distinct, can have methods
type Celsius float64
func (c Celsius) ToFahrenheit() Fahrenheit { return Fahrenheit(c*9/5 + 32) }

// Generic function
func Map[T, U any](s []T, fn func(T) U) []U { /* ... */ }

// Generic type
type Stack[T any] struct { items []T }
func (s *Stack[T]) Push(item T) { s.items = append(s.items, item) }

// Tilde constraint — includes named types
type Float interface { ~float32 | ~float64 }
func double[T Float](v T) T { return v * 2 }
double(Celsius(100))  // Works — Celsius has underlying float64
```

---

## 6. Error Handling
**File**: [06-error-handling-deep-dive.md](./06-error-handling-deep-dive.md)

### Topics Covered
- `error` interface internals — `iface` with `errorString`
- `errors.New` returns pointer — each call creates unique error
- `fmt.Errorf` with `%w` — wraps error for chain traversal
- `errors.Unwrap` — one level unwrap
- Error chain visualization
- Custom error types: struct-based, with context, HTTP errors
- `errors.Is` — identity check through chain, custom `Is()` method
- `errors.As` — type extraction through chain, custom `As()` method
- Sentinel errors — package-level variables for comparison
- Error handling patterns: early return, accumulation, `errors.Join` (Go 1.20+)
- Retry with error classification
- Panic internals — goroutine panic info, stack unwinding
- Recover internals — only in defer, converting panic to error
- Concurrent error handling: `errgroup`, error channels
- Pitfalls: ignoring errors, typed nil in interface, wrapping nil, losing context
- Best practices: add context, sentinel errors, log once at top

### Key Concepts
```go
// Wrap with context
return fmt.Errorf("getUserByEmail(%q): %w", email, err)

// errors.Is traverses chain
errors.Is(wrappedErr, ErrNotFound)  // true even if wrapped

// errors.As extracts type from chain
var ve *ValidationError
errors.As(err, &ve)  // ve now holds the extracted error

// Typed nil gotcha
func getError() error {
    var err *MyError = nil
    return nil  // NOT return err — typed nil is non-nil interface!
}

// errors.Join (Go 1.20+)
combined := errors.Join(err1, err2, err3)
```

---

## Quick Reference

### Pointer Rules
```
&x          → address of x
*p          → value at address p
new(T)      → *T pointing to zeroed T
T method    → value receiver (copy)
*T method   → pointer receiver (original)
*T          → satisfies interfaces with pointer receivers
T           → only satisfies interfaces with value receivers
```

### Struct Memory
```
Fields stored contiguously in declaration order
Padding added to satisfy alignment requirements
Order largest → smallest to minimize padding
struct{} has zero size — use for sets and signals
```

### Interface Rules
```
Satisfied implicitly — no implements keyword
iface = (type pointer, data pointer)
nil interface ≠ interface holding nil pointer
Type assertion: v, ok := i.(T)
Type switch:    switch v := i.(type) { case T: }
```

### Error Handling
```
errors.New("msg")           → unique error value
fmt.Errorf("ctx: %w", err) → wrapped error
errors.Is(err, target)      → identity check through chain
errors.As(err, &target)     → type extraction through chain
errors.Unwrap(err)          → one level unwrap
errors.Join(e1, e2)         → combine errors (Go 1.20+)
```

### Generics
```
[T any]              → unconstrained type parameter
[T comparable]       → supports == and !=
[T constraints.Ordered] → supports <, >, <=, >=
[T int | float64]    → union constraint
[T ~int]             → T or any type with underlying int
```

---

## Learning Path

### Beginner
1. [Pointers](./01-pointers-deep-dive.md) — understand memory and references
2. [Structs](./02-structs-deep-dive.md) — group data, add behavior
3. [Error Handling](./06-error-handling-deep-dive.md) — idiomatic Go errors

### Intermediate
4. [Interfaces](./03-interfaces-deep-dive.md) — polymorphism and abstraction
5. [Functions and Closures](./04-functions-closures-deep-dive.md) — functional patterns

### Advanced
6. [Types and Generics](./05-types-generics-deep-dive.md) — type system and reusable code

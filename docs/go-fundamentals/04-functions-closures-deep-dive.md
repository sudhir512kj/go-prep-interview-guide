# Functions and Closures Deep Dive

## Table of Contents
- [Function Fundamentals](#function-fundamentals)
- [Internal Implementation](#internal-implementation)
- [Multiple Return Values](#multiple-return-values)
- [Variadic Functions](#variadic-functions)
- [First-Class Functions](#first-class-functions)
- [Closures](#closures)
- [Closure Internals](#closure-internals)
- [Defer](#defer)
- [Panic and Recover](#panic-and-recover)
- [Recursion](#recursion)
- [Function Types and Patterns](#function-types-and-patterns)
- [Common Pitfalls](#common-pitfalls)
- [Performance Implications](#performance-implications)

---

## Function Fundamentals

### Declaration
```go
// Basic function
func add(a, b int) int {
    return a + b
}

// Multiple parameters of same type
func add(a, b, c int) int {
    return a + b + c
}

// Named return values
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = errors.New("division by zero")
        return  // Naked return — returns named values
    }
    result = a / b
    return
}

// No return value
func greet(name string) {
    fmt.Println("Hello,", name)
}
```

### Function Signature
```go
// Function type: func(paramTypes) returnTypes
type MathFunc func(int, int) int

var add MathFunc = func(a, b int) int { return a + b }
var mul MathFunc = func(a, b int) int { return a * b }
```

---

## Internal Implementation

### Stack Frame
```go
// Each function call creates a stack frame
func main() {
    x := add(3, 4)
    _ = x
}

func add(a, b int) int {
    return a + b
}

// Call stack:
// ┌──────────────────┐ ← top
// │ add frame        │
// │   a = 3          │
// │   b = 4          │
// │   return addr    │
// ├──────────────────┤
// │ main frame       │
// │   x = ?          │
// └──────────────────┘ ← bottom
```

### Goroutine Stack
```go
// Go uses growable stacks (not fixed-size like C)
// Initial stack: 2KB (Go 1.4+) or 8KB (older)
// Grows as needed up to 1GB default max
// Shrinks when no longer needed

// Stack growth mechanism:
// 1. Compiler inserts stack check at function entry
// 2. If stack too small, allocate larger stack
// 3. Copy all frames to new stack
// 4. Update all pointers
```

### Function Call Convention
```go
// Arguments passed on stack (or registers in Go 1.17+)
// Return values also on stack/registers
// Go 1.17+ uses register-based calling convention for performance

// Before Go 1.17: all args/returns on stack
// Go 1.17+: up to 9 integer args in registers (RAX, RBX, RCX, RDI, RSI, R8, R9, R10, R11)
//           up to 15 float args in registers (X0-X14)
```

---

## Multiple Return Values

### Basic Multiple Returns
```go
func minMax(arr []int) (int, int) {
    min, max := arr[0], arr[0]
    for _, v := range arr[1:] {
        if v < min { min = v }
        if v > max { max = v }
    }
    return min, max
}

min, max := minMax([]int{3, 1, 4, 1, 5, 9})
fmt.Println(min, max)  // 1 9
```

### Error as Second Return
```go
// Idiomatic Go: return (value, error)
func parseInt(s string) (int, error) {
    n, err := strconv.Atoi(s)
    if err != nil {
        return 0, fmt.Errorf("parseInt: %w", err)
    }
    return n, nil
}

n, err := parseInt("42")
if err != nil {
    log.Fatal(err)
}
fmt.Println(n)  // 42
```

### Named Return Values
```go
// Named returns: useful for documentation and defer
func readFile(path string) (content []byte, err error) {
    f, err := os.Open(path)
    if err != nil {
        return  // Returns content=nil, err=<open error>
    }
    defer f.Close()

    content, err = io.ReadAll(f)
    return  // Returns content=<data>, err=nil (or read error)
}

// Named returns with defer for cleanup
func transaction(db *sql.DB) (err error) {
    tx, err := db.Begin()
    if err != nil {
        return
    }
    defer func() {
        if err != nil {
            tx.Rollback()
        } else {
            err = tx.Commit()
        }
    }()
    // ... do work
    return
}
```

---

## Variadic Functions

### Definition
```go
// Variadic: accepts zero or more arguments of a type
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

fmt.Println(sum())           // 0
fmt.Println(sum(1, 2, 3))    // 6
fmt.Println(sum(1, 2, 3, 4)) // 10
```

### Spreading a Slice
```go
nums := []int{1, 2, 3, 4, 5}
fmt.Println(sum(nums...))  // 15 — spread slice into variadic
```

### Variadic with Other Parameters
```go
// Variadic must be last parameter
func printf(format string, args ...interface{}) {
    fmt.Printf(format, args...)
}

// Passing variadic to another variadic
func logf(level string, format string, args ...interface{}) {
    fmt.Printf("[%s] "+format+"\n", append([]interface{}{level}, args...)...)
}
```

### Variadic Internals
```go
// Variadic args are passed as a slice
func f(args ...int) {
    // args is []int
    fmt.Printf("type: %T, len: %d\n", args, len(args))
}

f(1, 2, 3)  // type: []int, len: 3

// No allocation if called with spread slice
s := []int{1, 2, 3}
f(s...)  // No new slice allocated — s is passed directly
```

---

## First-Class Functions

### Functions as Values
```go
// Assign to variable
add := func(a, b int) int { return a + b }
fmt.Println(add(3, 4))  // 7

// Pass as argument
func apply(fn func(int, int) int, a, b int) int {
    return fn(a, b)
}
fmt.Println(apply(add, 3, 4))  // 7

// Return from function
func multiplier(factor int) func(int) int {
    return func(n int) int {
        return n * factor
    }
}
double := multiplier(2)
triple := multiplier(3)
fmt.Println(double(5))  // 10
fmt.Println(triple(5))  // 15
```

### Function Slice
```go
type Transform func(int) int

transforms := []Transform{
    func(n int) int { return n * 2 },
    func(n int) int { return n + 10 },
    func(n int) int { return n * n },
}

n := 3
for _, t := range transforms {
    n = t(n)
}
fmt.Println(n)  // ((3*2)+10)^2 = 256
```

### Higher-Order Functions
```go
// Map
func mapSlice(s []int, fn func(int) int) []int {
    result := make([]int, len(s))
    for i, v := range s {
        result[i] = fn(v)
    }
    return result
}

// Filter
func filter(s []int, fn func(int) bool) []int {
    result := []int{}
    for _, v := range s {
        if fn(v) {
            result = append(result, v)
        }
    }
    return result
}

// Reduce
func reduce(s []int, init int, fn func(int, int) int) int {
    acc := init
    for _, v := range s {
        acc = fn(acc, v)
    }
    return acc
}

nums := []int{1, 2, 3, 4, 5}
doubled := mapSlice(nums, func(n int) int { return n * 2 })
evens   := filter(nums, func(n int) bool { return n%2 == 0 })
sum     := reduce(nums, 0, func(a, b int) int { return a + b })
```

---

## Closures

### Definition
A closure is a function that captures variables from its surrounding scope.

```go
func counter() func() int {
    count := 0  // Captured variable
    return func() int {
        count++
        return count
    }
}

c1 := counter()
c2 := counter()

fmt.Println(c1())  // 1
fmt.Println(c1())  // 2
fmt.Println(c1())  // 3
fmt.Println(c2())  // 1 — independent counter
fmt.Println(c2())  // 2
```

### Capturing by Reference
```go
// Closures capture variables by reference, not by value
x := 10
fn := func() {
    fmt.Println(x)  // Captures x by reference
}
x = 20
fn()  // Prints 20, not 10!
```

### Mutable State via Closure
```go
func makeAdder(base int) (func(int) int, func(int)) {
    add := func(n int) int {
        base += n
        return base
    }
    reset := func(n int) {
        base = n
    }
    return add, reset
}

add, reset := makeAdder(10)
fmt.Println(add(5))   // 15
fmt.Println(add(3))   // 18
reset(0)
fmt.Println(add(1))   // 1
```

---

## Closure Internals

### How Closures Work
```go
// When a closure captures a variable, Go allocates it on the heap
// (escape analysis: variable escapes to heap)

func counter() func() int {
    count := 0  // Allocated on heap because it escapes
    return func() int {
        count++
        return count
    }
}

// The returned function holds a pointer to count
// Closure struct (conceptually):
// type closure struct {
//     fn    func()
//     count *int  // Pointer to captured variable
// }
```

### Escape Analysis for Closures
```bash
$ go build -gcflags="-m" main.go
./main.go:3:2: moved to heap: count
# count escapes because the closure outlives counter()
```

### Shared Captured Variables
```go
// Multiple closures sharing the same variable
funcs := make([]func(), 3)
x := 0
for i := 0; i < 3; i++ {
    funcs[i] = func() {
        x++
        fmt.Println(x)
    }
}
funcs[0]()  // 1
funcs[1]()  // 2
funcs[2]()  // 3
// All closures share the same x
```

---

## Defer

### Basic Defer
```go
// Deferred calls execute when surrounding function returns
// LIFO order (last deferred = first executed)
func example() {
    defer fmt.Println("third")
    defer fmt.Println("second")
    defer fmt.Println("first")
    fmt.Println("body")
}
// Output:
// body
// first
// second
// third
```

### Defer Internals
```go
// Defer arguments are evaluated immediately
x := 10
defer fmt.Println(x)  // Captures x=10 now
x = 20
// Prints 10, not 20

// But pointer/reference captures current value
defer fmt.Println(&x)  // Prints address
// *(&x) at call time = 20
```

### Defer for Cleanup
```go
func readFile(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()  // Always closes, even on error

    return io.ReadAll(f)
}

func withLock(mu *sync.Mutex) {
    mu.Lock()
    defer mu.Unlock()  // Always unlocks
    // ... critical section
}
```

### Defer with Named Returns
```go
func divide(a, b float64) (result float64, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered: %v", r)
        }
    }()

    if b == 0 {
        panic("division by zero")
    }
    result = a / b
    return
}
```

### Defer in Loop (Common Mistake)
```go
// Wrong: defers accumulate, all run at function end
func processFiles(files []string) {
    for _, f := range files {
        file, _ := os.Open(f)
        defer file.Close()  // All deferred until processFiles returns!
    }
}

// Fix: use anonymous function or helper
func processFiles(files []string) {
    for _, f := range files {
        func() {
            file, _ := os.Open(f)
            defer file.Close()  // Runs when anonymous func returns
            // process file
        }()
    }
}
```

---

## Panic and Recover

### Panic
```go
// panic stops normal execution, unwinds stack, runs deferred functions
func mustPositive(n int) int {
    if n <= 0 {
        panic(fmt.Sprintf("expected positive, got %d", n))
    }
    return n
}

// Panic with any value
panic("something went wrong")
panic(42)
panic(errors.New("error"))
```

### Recover
```go
// recover() stops panic propagation — only works inside defer
func safeDiv(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()
    return a / b, nil  // Panics if b == 0
}

result, err := safeDiv(10, 0)
fmt.Println(result, err)  // 0, recovered from panic: runtime error: integer divide by zero
```

### Panic Propagation
```go
// Panic propagates up the call stack
// Each frame's deferred functions run
// If no recover(), program crashes with stack trace

func c() { panic("panic in c") }
func b() { defer fmt.Println("defer in b"); c() }
func a() { defer fmt.Println("defer in a"); b() }

// a() → b() → c() → panic
// Defers run: "defer in b", "defer in a"
// Then program crashes
```

### When to Use Panic
```go
// Use panic for:
// 1. Programmer errors (impossible states)
func mustCompile(pattern string) *regexp.Regexp {
    re, err := regexp.Compile(pattern)
    if err != nil {
        panic(err)  // Pattern is hardcoded — error is programmer mistake
    }
    return re
}

// 2. Initialization failures
func init() {
    if err := db.Ping(); err != nil {
        panic("cannot connect to database: " + err.Error())
    }
}

// Do NOT use panic for:
// - Expected errors (use error return)
// - User input errors
// - Network/IO errors
```

---

## Recursion

### Basic Recursion
```go
func factorial(n int) int {
    if n <= 1 {
        return 1
    }
    return n * factorial(n-1)
}

// Stack frames for factorial(4):
// factorial(4) → 4 * factorial(3)
//   factorial(3) → 3 * factorial(2)
//     factorial(2) → 2 * factorial(1)
//       factorial(1) → 1
```

### Tail Recursion
```go
// Go does NOT optimize tail calls (no TCO)
// Deep recursion can cause stack overflow

// Tail recursive version (not optimized in Go)
func factTail(n, acc int) int {
    if n <= 1 {
        return acc
    }
    return factTail(n-1, n*acc)
}

// Convert to iteration for deep recursion
func factIter(n int) int {
    result := 1
    for i := 2; i <= n; i++ {
        result *= i
    }
    return result
}
```

### Mutual Recursion
```go
func isEven(n int) bool {
    if n == 0 { return true }
    return isOdd(n - 1)
}

func isOdd(n int) bool {
    if n == 0 { return false }
    return isEven(n - 1)
}
```

### Memoization
```go
func memoize(fn func(int) int) func(int) int {
    cache := make(map[int]int)
    return func(n int) int {
        if v, ok := cache[n]; ok {
            return v
        }
        result := fn(n)
        cache[n] = result
        return result
    }
}

var fib func(int) int
fib = memoize(func(n int) int {
    if n <= 1 { return n }
    return fib(n-1) + fib(n-2)
})
```

---

## Function Types and Patterns

### Middleware Pattern
```go
type HandlerFunc func(ctx context.Context) error

type Middleware func(HandlerFunc) HandlerFunc

func Chain(h HandlerFunc, middlewares ...Middleware) HandlerFunc {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

func Logging(next HandlerFunc) HandlerFunc {
    return func(ctx context.Context) error {
        log.Println("before")
        err := next(ctx)
        log.Println("after")
        return err
    }
}

func Auth(next HandlerFunc) HandlerFunc {
    return func(ctx context.Context) error {
        // check auth
        return next(ctx)
    }
}

handler := Chain(myHandler, Logging, Auth)
```

### Option Functions
```go
type Option func(*Config)

func WithHost(host string) Option {
    return func(c *Config) { c.host = host }
}

func WithPort(port int) Option {
    return func(c *Config) { c.port = port }
}

func NewConfig(opts ...Option) *Config {
    c := &Config{host: "localhost", port: 8080}
    for _, opt := range opts {
        opt(c)
    }
    return c
}
```

### Once Function
```go
// Execute function only once
var once sync.Once
var instance *Singleton

func getInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}
```

### Retry Pattern
```go
func retry(attempts int, delay time.Duration, fn func() error) error {
    for i := 0; i < attempts; i++ {
        if err := fn(); err == nil {
            return nil
        } else if i < attempts-1 {
            time.Sleep(delay)
        }
    }
    return fmt.Errorf("failed after %d attempts", attempts)
}

err := retry(3, time.Second, func() error {
    return callExternalAPI()
})
```

---

## Common Pitfalls

### 1. Closure Captures Loop Variable
```go
// Problem: all goroutines share same loop variable
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i)  // All print 3!
    }()
}

// Fix 1: pass as argument
for i := 0; i < 3; i++ {
    go func(i int) {
        fmt.Println(i)  // 0, 1, 2
    }(i)
}

// Fix 2: shadow variable (Go 1.22+ fixes this automatically)
for i := 0; i < 3; i++ {
    i := i  // New variable per iteration
    go func() {
        fmt.Println(i)
    }()
}
```

### 2. Defer in Loop
```go
// Problem: defers accumulate until function returns
for _, f := range files {
    r, _ := os.Open(f)
    defer r.Close()  // All deferred until outer function returns!
}

// Fix: wrap in function
for _, f := range files {
    processFile(f)
}

func processFile(path string) {
    r, _ := os.Open(path)
    defer r.Close()  // Runs when processFile returns
}
```

### 3. Modifying Slice in Closure
```go
s := []int{1, 2, 3}
fn := func() {
    s = append(s, 4)  // May or may not affect outer s
}
fn()
// s may be [1,2,3,4] or original [1,2,3] depending on reallocation
```

### 4. Recover Outside Defer
```go
// recover() only works inside a deferred function
func wrong() {
    r := recover()  // Always returns nil — not in defer
    fmt.Println(r)
}

func correct() {
    defer func() {
        r := recover()  // Works — inside defer
        fmt.Println(r)
    }()
    panic("oops")
}
```

---

## Performance Implications

### Function Call Cost
```go
// Direct call: ~1ns
// Interface call: ~2-5ns
// Closure call: ~2-3ns (extra pointer indirection)
// Reflect call: ~100ns+

// Inlining eliminates call overhead
// go build -gcflags="-m" shows inlining decisions
```

### Closure Allocation
```go
// Closures that capture variables allocate on heap
func makeCounter() func() int {
    n := 0          // Heap allocated (escapes)
    return func() int {
        n++
        return n
    }
}

// Closures that don't capture are just function pointers
fn := func(x int) int { return x * 2 }  // No allocation
```

### Defer Overhead
```go
// Defer has overhead (~25ns per defer in older Go)
// Go 1.14+ optimized open-coded defers (~0ns for simple cases)

// Benchmark
func BenchmarkDefer(b *testing.B) {
    for i := 0; i < b.N; i++ {
        func() {
            defer func() {}()
        }()
    }
}
```

---

## Summary

### Key Concepts
1. Functions are first-class values — assign, pass, return
2. Closures capture variables by reference (not value)
3. Defer runs LIFO when function returns; arguments evaluated immediately
4. Panic unwinds stack; recover() in defer stops propagation
5. Go does NOT optimize tail calls — use iteration for deep recursion
6. Variadic args are a slice; spread with `...`

### Decision Guide
```
Need to modify caller's variable?     → pointer parameter
Need optional configuration?          → variadic options pattern
Need cleanup regardless of error?     → defer
Need to handle panic gracefully?      → defer + recover
Need stateful function?               → closure
Need to share behavior?               → function type / interface
Deep recursion?                       → convert to iteration
```

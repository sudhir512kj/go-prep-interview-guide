# Golang Basics

## Introduction to Go

Go (Golang) is a statically typed, compiled programming language designed at Google by Robert Griesemer, Rob Pike, and Ken Thompson. First released in 2009, Go was created to address shortcomings in other languages while maintaining their strengths.

### Design Philosophy
- **Simplicity**: Go has a small, orthogonal feature set
- **Efficiency**: Fast compilation and execution
- **Safety**: Strong typing with memory safety
- **Concurrency**: First-class support for concurrent programming
- **Pragmatism**: Designed for real-world software engineering

### Why Go?
1. **Fast Compilation**: Go compiles to native machine code quickly
2. **Garbage Collection**: Automatic memory management with low latency
3. **Built-in Concurrency**: Goroutines and channels make concurrent programming easy
4. **Standard Library**: Rich, well-designed standard library
5. **Cross-Platform**: Easy cross-compilation for different platforms
6. **Tooling**: Excellent built-in tools (gofmt, go test, go vet, etc.)

## Variables and Data Types

### Understanding Variables

In Go, variables are explicitly declared and used by the compiler to check type-correctness of function calls. Go is statically typed, meaning variable types are known at compile time.

### Variable Declaration
```go
// Explicit type
var name string = "John"

// Type inference
var age = 30

// Short declaration (inside functions only)
city := "New York"

// Multiple variables
var x, y int = 1, 2
```

### Basic Data Types
- **bool**: true/false
- **string**: UTF-8 encoded text
- **int, int8, int16, int32, int64**: signed integers
- **uint, uint8, uint16, uint32, uint64**: unsigned integers
- **byte**: alias for uint8
- **rune**: alias for int32 (Unicode code point)
- **float32, float64**: floating-point numbers
- **complex64, complex128**: complex numbers

### Zero Values

Go initializes all variables to their "zero value" if no explicit initialization is provided. This eliminates undefined behavior and makes code more predictable.

```go
var i int       // 0
var f float64   // 0.0
var b bool      // false
var s string    // "" (empty string)
var p *int      // nil
var slice []int // nil
var m map[string]int // nil
var ch chan int // nil
var fn func()   // nil
```

**Why Zero Values Matter:**
- Eliminates uninitialized variable bugs
- Makes code behavior predictable
- Simplifies initialization logic
- Zero values are often useful defaults (e.g., empty string, false, 0)

### Type System Deep Dive

**Numeric Types:**
```go
// Signed integers
int8   // -128 to 127
int16  // -32768 to 32767
int32  // -2147483648 to 2147483647
int64  // -9223372036854775808 to 9223372036854775807
int    // Platform dependent (32 or 64 bit)

// Unsigned integers
uint8  // 0 to 255
uint16 // 0 to 65535
uint32 // 0 to 4294967295
uint64 // 0 to 18446744073709551615
uint   // Platform dependent

// Floating point
float32 // IEEE-754 32-bit floating-point
float64 // IEEE-754 64-bit floating-point

// Complex numbers
complex64  // complex numbers with float32 real and imaginary parts
complex128 // complex numbers with float64 real and imaginary parts
```

**Type Aliases:**
```go
byte = uint8  // Used for raw data
rune = int32  // Used for Unicode code points
```

### Type Conversion

Go requires explicit type conversion - there are no implicit conversions.

```go
var i int = 42
var f float64 = float64(i)  // Explicit conversion required
var u uint = uint(f)

// String conversions
str := string(65)           // "A" (converts rune to string)
num := strconv.Atoi("123") // String to int
str2 := strconv.Itoa(42)   // Int to string
```

## Control Structures

### Philosophy

Go has a minimal set of control structures, emphasizing simplicity and readability. There's no while loop, no do-while, and no ternary operator - everything is done with `for` and `if`.

### If-Else
```go
if x > 0 {
    fmt.Println("Positive")
} else if x < 0 {
    fmt.Println("Negative")
} else {
    fmt.Println("Zero")
}

// If with initialization
if val := compute(); val > 0 {
    fmt.Println(val)
}
```

### Switch
```go
switch day {
case "Monday":
    fmt.Println("Start of week")
case "Friday":
    fmt.Println("End of week")
default:
    fmt.Println("Midweek")
}

// Switch without condition (like if-else chain)
switch {
case x < 0:
    fmt.Println("Negative")
case x > 0:
    fmt.Println("Positive")
}
```

### Loops

Go has only one looping construct: `for`. It's versatile enough to handle all looping scenarios.

```go
// Traditional for loop
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// While-style loop (no while keyword in Go)
for x < 100 {
    x++
}

// Infinite loop
for {
    // break to exit
    if condition {
        break
    }
}

// Range loop - iterates over slices, arrays, maps, strings, channels
for index, value := range slice {
    fmt.Println(index, value)
}

// Ignore index with blank identifier
for _, value := range slice {
    fmt.Println(value)
}

// Only index
for index := range slice {
    fmt.Println(index)
}

// Range over string (iterates runes, not bytes)
for i, ch := range "Hello, 世界" {
    fmt.Printf("%d: %c\n", i, ch)
}

// Range over map (order is random)
for key, value := range myMap {
    fmt.Println(key, value)
}

// Range over channel (receives until closed)
for value := range channel {
    fmt.Println(value)
}
```

**Loop Control:**
```go
// break: exit loop
for i := 0; i < 10; i++ {
    if i == 5 {
        break
    }
}

// continue: skip to next iteration
for i := 0; i < 10; i++ {
    if i%2 == 0 {
        continue
    }
    fmt.Println(i) // Only odd numbers
}

// Labels for nested loops
Outer:
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if i*j > 4 {
            break Outer // Break outer loop
        }
    }
}
```

## Functions

### Function Fundamentals

Functions are first-class citizens in Go. They can be assigned to variables, passed as arguments, and returned from other functions.

### Basic Function
```go
func add(a int, b int) int {
    return a + b
}

// Same type parameters
func multiply(a, b int) int {
    return a * b
}
```

### Multiple Return Values
```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, fmt.Errorf("division by zero")
    }
    return a / b, nil
}
```

### Named Return Values
```go
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return // naked return
}
```

### Variadic Functions
```go
func sum(nums ...int) int {
    total := 0
    for _, num := range nums {
        total += num
    }
    return total
}
```

### Closures

A closure is a function value that references variables from outside its body. The function may access and assign to the referenced variables.

```go
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

// Usage
c1 := counter()
fmt.Println(c1()) // 1
fmt.Println(c1()) // 2

c2 := counter()   // New closure with its own count
fmt.Println(c2()) // 1
```

**Practical Closure Example:**
```go
// Fibonacci generator using closure
func fibonacci() func() int {
    a, b := 0, 1
    return func() int {
        a, b = b, a+b
        return a
    }
}

f := fibonacci()
for i := 0; i < 10; i++ {
    fmt.Println(f())
}
```

### Higher-Order Functions

```go
// Function that takes a function as parameter
func apply(nums []int, fn func(int) int) []int {
    result := make([]int, len(nums))
    for i, v := range nums {
        result[i] = fn(v)
    }
    return result
}

// Usage
nums := []int{1, 2, 3, 4, 5}
squared := apply(nums, func(n int) int {
    return n * n
})
```

### Method Values and Expressions

```go
type Person struct {
    Name string
}

func (p Person) Greet() {
    fmt.Println("Hello, I'm", p.Name)
}

// Method value (bound to instance)
p := Person{Name: "Alice"}
greet := p.Greet
greet() // "Hello, I'm Alice"

// Method expression (needs instance as first argument)
greetFunc := Person.Greet
greetFunc(p) // "Hello, I'm Alice"
```
```

## Pointers

### Understanding Pointers

A pointer holds the memory address of a value. Go has pointers but no pointer arithmetic (unlike C/C++).

**Why Pointers?**
1. Modify values in functions
2. Avoid copying large structures
3. Share data between different parts of program
4. Represent optional values (nil pointer)

```go
// Pointer declaration
var p *int

// Address operator
x := 42
p = &x

// Dereference operator
fmt.Println(*p) // 42
*p = 21
fmt.Println(x)  // 21

// Pointers in functions
func increment(n *int) {
    *n++
}

x := 5
increment(&x)
fmt.Println(x) // 6
```

### Pointer vs Value Semantics

```go
// Value semantics: copy is made
func modifyValue(p Person) {
    p.Name = "Changed" // Modifies copy, not original
}

// Pointer semantics: original is modified
func modifyPointer(p *Person) {
    p.Name = "Changed" // Modifies original
}

person := Person{Name: "Alice"}
modifyValue(person)
fmt.Println(person.Name) // "Alice" (unchanged)

modifyPointer(&person)
fmt.Println(person.Name) // "Changed"
```

### Pointer Safety

```go
// Go prevents common pointer errors:

// 1. No pointer arithmetic
var p *int
// p++ // Compile error

// 2. Automatic dereferencing for struct fields
type Point struct { X, Y int }
p := &Point{1, 2}
fmt.Println(p.X) // Automatically dereferences, no need for p->X

// 3. Nil pointer checks
var ptr *int
if ptr != nil {
    fmt.Println(*ptr)
}
```

### new() Function

```go
// new() allocates memory and returns pointer to zero value
p := new(int)
fmt.Println(*p) // 0

*p = 42
fmt.Println(*p) // 42
```

## Structs

### Understanding Structs

Structs are typed collections of fields. They're used to group data together to form records. Go doesn't have classes - structs with methods provide similar functionality.

### Definition and Usage
```go
type Person struct {
    Name string
    Age  int
}

// Create struct
p1 := Person{Name: "Alice", Age: 30}
p2 := Person{"Bob", 25}

// Access fields
fmt.Println(p1.Name)
p1.Age = 31
```

### Struct Methods

Methods are functions with a receiver argument. The receiver appears between `func` and the method name.

```go
// Value receiver: operates on a copy
func (p Person) Greet() string {
    return "Hello, " + p.Name
}

// Pointer receiver: can modify the original
func (p *Person) Birthday() {
    p.Age++
}

// When to use pointer receivers:
// 1. Method needs to modify the receiver
// 2. Receiver is a large struct (avoid copying)
// 3. Consistency: if some methods need pointer receivers, use them for all
```

**Method Sets:**
```go
type Counter struct {
    count int
}

func (c *Counter) Increment() { c.count++ }
func (c Counter) Value() int  { return c.count }

// Pointer receiver methods can be called on values (Go auto-converts)
c := Counter{}
c.Increment() // Go converts to (&c).Increment()

// Value receiver methods can be called on pointers (Go auto-dereferences)
p := &Counter{}
val := p.Value() // Go converts to (*p).Value()
```

### Struct Tags

Struct tags provide metadata about fields, commonly used for serialization.

```go
type User struct {
    ID        int    `json:"id" db:"user_id"`
    Name      string `json:"name" validate:"required"`
    Email     string `json:"email,omitempty" validate:"email"`
    Password  string `json:"-"` // Never serialize
    CreatedAt time.Time `json:"created_at" db:"created_at"`
}

// Accessing tags using reflection
field, _ := reflect.TypeOf(User{}).FieldByName("Name")
tag := field.Tag.Get("json") // "name"
```

### Anonymous Structs

```go
// Useful for one-off data structures
person := struct {
    Name string
    Age  int
}{
    Name: "Alice",
    Age:  30,
}

// Common in table-driven tests
tests := []struct {
    input    int
    expected int
}{
    {1, 2},
    {2, 4},
    {3, 6},
}
```

### Embedded Structs
```go
type Address struct {
    City    string
    Country string
}

type Employee struct {
    Person
    Address
    Salary int
}

e := Employee{
    Person:  Person{Name: "John", Age: 30},
    Address: Address{City: "NYC", Country: "USA"},
    Salary:  50000,
}
```

## Interfaces

### Interface Philosophy

Interfaces in Go are implicit - a type implements an interface by implementing its methods. No explicit declaration is needed. This is called "structural typing" or "duck typing".

**Key Principles:**
1. Interfaces define behavior, not data
2. Small interfaces are better (often just 1-2 methods)
3. Accept interfaces, return concrete types
4. Interfaces enable polymorphism and dependency injection

### Definition
```go
type Shape interface {
    Area() float64
    Perimeter() float64
}

type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}
```

### Empty Interface

The empty interface `interface{}` can hold values of any type. Every type implements the empty interface.

```go
func describe(i interface{}) {
    fmt.Printf("Type: %T, Value: %v\n", i, i)
}

// Can accept any type
describe(42)
describe("hello")
describe([]int{1, 2, 3})

// Common uses:
// 1. Generic containers
var values []interface{}
values = append(values, 1, "two", 3.0)

// 2. JSON unmarshaling
var data interface{}
json.Unmarshal(jsonBytes, &data)

// Note: Go 1.18+ has generics, reducing need for interface{}
```

### Interface Values

An interface value holds a concrete value and its type.

```go
var w io.Writer
// w is nil (both type and value are nil)

w = os.Stdout
// w now holds (*os.File, pointer to os.Stdout)

w = new(bytes.Buffer)
// w now holds (*bytes.Buffer, pointer to buffer)

// Interface value structure:
// [type pointer | value pointer]
```

**Nil Interface vs Nil Value:**
```go
var w io.Writer // nil interface (type=nil, value=nil)

var buf *bytes.Buffer // nil pointer
w = buf // non-nil interface (type=*bytes.Buffer, value=nil)

// This is a common gotcha:
if w != nil {
    // This executes! w is not nil (has type)
    // But calling methods may panic if value is nil
}
```

### Interface Composition

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

// Composed interface
type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}

// Any type implementing Read, Write, and Close
// automatically implements ReadWriteCloser
```

### Type Assertion
```go
var i interface{} = "hello"

s := i.(string)
fmt.Println(s)

s, ok := i.(string)
if ok {
    fmt.Println(s)
}
```

### Type Switch
```go
func do(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Int: %v\n", v)
    case string:
        fmt.Printf("String: %v\n", v)
    default:
        fmt.Printf("Unknown type\n")
    }
}
```

## Arrays vs Slices

### Understanding the Difference

**Arrays:**
- Fixed size (part of the type)
- Value type (copied when assigned or passed)
- Rarely used directly in Go

**Slices:**
- Dynamic size
- Reference type (header points to underlying array)
- Most common collection type in Go

### Arrays (Fixed Size)
```go
var arr [5]int
arr[0] = 1

arr2 := [3]string{"a", "b", "c"}
```

### Slices (Dynamic)

**Slice Internals:**
A slice is a descriptor containing:
1. Pointer to underlying array
2. Length (number of elements)
3. Capacity (size of underlying array)

```go
slice := []int{1, 2, 3}
slice = append(slice, 4)

// Make slice
s := make([]int, 5)      // len=5, cap=5, all zeros
s := make([]int, 0, 5)   // len=0, cap=5, empty but pre-allocated

// Slice operations
s[1:3]  // elements at index 1 and 2
s[:3]   // first 3 elements (0, 1, 2)
s[2:]   // from index 2 to end
s[:]    // entire slice (creates new slice header, same underlying array)

// Full slice expression (controls capacity)
a := []int{1, 2, 3, 4, 5}
b := a[1:3:4] // [2, 3] with cap=3 (4-1)
```

**Slice Growth:**
```go
s := make([]int, 0, 4)
fmt.Printf("len=%d cap=%d\n", len(s), cap(s)) // len=0 cap=4

for i := 0; i < 10; i++ {
    s = append(s, i)
    fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
}
// When capacity is exceeded, Go allocates new array (typically 2x size)
// and copies elements
```

**Slice Gotchas:**
```go
// 1. Slices share underlying array
a := []int{1, 2, 3, 4, 5}
b := a[1:3] // [2, 3]
b[0] = 99
fmt.Println(a) // [1, 99, 3, 4, 5] - a is modified!

// 2. Append may or may not modify original
a := []int{1, 2, 3}
b := a[:2]        // [1, 2] - shares array with a
b = append(b, 99) // Modifies a[2] if cap allows

// 3. Safe copying
a := []int{1, 2, 3}
b := make([]int, len(a))
copy(b, a) // Now b is independent
```

## Maps

### Understanding Maps

Maps are Go's built-in hash table implementation. They provide O(1) average-case lookup, insert, and delete operations.

**Map Characteristics:**
- Reference type (like slices)
- Unordered (iteration order is random)
- Not safe for concurrent use (need sync.Map or mutex)
- Keys must be comparable (can use == operator)

```go
// Create map
m := make(map[string]int)
m["key"] = 42

// Map literal
m := map[string]int{
    "foo": 1,
    "bar": 2,
}

// Check existence
value, exists := m["key"]

// Delete
delete(m, "key")

// Iterate (order is randomized)
for key, value := range m {
    fmt.Println(key, value)
}

// Iterate keys only
for key := range m {
    fmt.Println(key)
}
```

### Map Internals

```go
// Maps are implemented as hash tables with buckets
// Each bucket holds up to 8 key-value pairs

// Zero value is nil
var m map[string]int // nil map
// m["key"] = 1      // panic: assignment to entry in nil map

// Must initialize with make or literal
m = make(map[string]int)
m["key"] = 1 // OK

// Pre-allocate for performance
m := make(map[string]int, 1000) // Hint: expect 1000 elements
```

### Map as Set

```go
// Go doesn't have a built-in set type
// Use map[T]bool or map[T]struct{}

// Using bool
set := make(map[string]bool)
set["apple"] = true
set["banana"] = true

if set["apple"] {
    fmt.Println("apple exists")
}

// Using struct{} (more memory efficient)
set := make(map[string]struct{})
set["apple"] = struct{}{}
set["banana"] = struct{}{}

if _, exists := set["apple"]; exists {
    fmt.Println("apple exists")
}
```

### Concurrent Map Access

```go
// Maps are not safe for concurrent use
// Option 1: Use mutex
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

// Option 2: Use sync.Map (for specific use cases)
var m sync.Map
m.Store("key", "value")
val, ok := m.Load("key")
m.Delete("key")
```

## Defer, Panic, Recover

### Understanding Defer

Defer schedules a function call to be executed after the surrounding function returns. Deferred calls are executed in LIFO (Last In, First Out) order.

**Common Uses:**
1. Resource cleanup (close files, unlock mutexes)
2. Logging function exit
3. Recovering from panics
4. Modifying return values

### Defer
```go
func example() {
    defer fmt.Println("world")
    fmt.Println("hello")
}
// Output: hello world

// Multiple defers (LIFO)
defer fmt.Println("1")
defer fmt.Println("2")
defer fmt.Println("3")
// Output: 3 2 1
```

### Panic and Recover

**Panic:**
- Stops normal execution of current goroutine
- Runs deferred functions
- Returns to caller (which panics)
- Continues up the stack until program crashes or panic is recovered

**When to Panic:**
- Unrecoverable errors (programmer mistakes)
- Initialization failures
- Never in library code (return errors instead)

```go
func mayPanic() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered:", r)
            // Can also inspect stack trace
            debug.PrintStack()
        }
    }()
    panic("something went wrong")
}

// Panic with custom type
type MyError struct {
    Msg string
}

func riskyOperation() {
    defer func() {
        if r := recover(); r != nil {
            if err, ok := r.(MyError); ok {
                fmt.Println("Custom error:", err.Msg)
            }
        }
    }()
    panic(MyError{Msg: "custom panic"})
}
```

### Defer Evaluation Rules

```go
// 1. Deferred function arguments are evaluated immediately
func example1() {
    x := 1
    defer fmt.Println(x) // Prints 1 (not 2)
    x = 2
}

// 2. Deferred functions can modify named return values
func example2() (result int) {
    defer func() {
        result++ // Modifies return value
    }()
    return 1 // Returns 2
}

// 3. Deferred functions execute in LIFO order
func example3() {
    defer fmt.Println("first")
    defer fmt.Println("second")
    defer fmt.Println("third")
    // Output: third, second, first
}

// 4. Defer in loops
func example4() {
    for i := 0; i < 3; i++ {
        defer fmt.Println(i)
    }
    // Output: 2, 1, 0
}
```

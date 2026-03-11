# Structs Deep Dive

## Table of Contents
- [Struct Fundamentals](#struct-fundamentals)
- [Internal Memory Layout](#internal-memory-layout)
- [Struct Padding and Alignment](#struct-padding-and-alignment)
- [Struct Initialization](#struct-initialization)
- [Methods on Structs](#methods-on-structs)
- [Embedding](#embedding)
- [Anonymous Structs](#anonymous-structs)
- [Struct Tags](#struct-tags)
- [Struct Comparison](#struct-comparison)
- [Struct Patterns](#struct-patterns)
- [Common Pitfalls](#common-pitfalls)
- [Performance Optimization](#performance-optimization)

---

## Struct Fundamentals

### Definition
A struct is a composite type that groups fields of different types under a single name.

```go
type Person struct {
    Name string
    Age  int
    Email string
}
```

### Zero Value
Every struct field is initialized to its zero value.

```go
var p Person
fmt.Println(p.Name)   // ""
fmt.Println(p.Age)    // 0
fmt.Println(p.Email)  // ""
```

### Value Type
Structs are value types — assignment copies all fields.

```go
p1 := Person{Name: "Alice", Age: 30}
p2 := p1          // Full copy
p2.Name = "Bob"
fmt.Println(p1.Name)  // "Alice" — unchanged
fmt.Println(p2.Name)  // "Bob"
```

---

## Internal Memory Layout

### Contiguous Memory
Struct fields are stored contiguously in memory in declaration order.

```go
type Point struct {
    X float64  // 8 bytes at offset 0
    Y float64  // 8 bytes at offset 8
}
// Total: 16 bytes, no padding needed (both 8-byte aligned)

Memory:
┌──────────────────┬──────────────────┐
│   X (8 bytes)    │   Y (8 bytes)    │
│   offset: 0      │   offset: 8      │
└──────────────────┴──────────────────┘
```

### Checking Layout
```go
import "unsafe"

type Point struct {
    X float64
    Y float64
}

fmt.Println(unsafe.Sizeof(Point{}))         // 16
fmt.Println(unsafe.Offsetof(Point{}.X))     // 0
fmt.Println(unsafe.Offsetof(Point{}.Y))     // 8
fmt.Println(unsafe.Alignof(Point{}))        // 8
```

---

## Struct Padding and Alignment

### Alignment Rules
Each field must be aligned to its own size (or the platform's max alignment, whichever is smaller).

```
bool:    1-byte aligned
int8:    1-byte aligned
int16:   2-byte aligned
int32:   4-byte aligned
int64:   8-byte aligned
float64: 8-byte aligned
pointer: 8-byte aligned (64-bit)
```

### Padding Example
```go
type Bad struct {
    A bool    // 1 byte  at offset 0
              // 7 bytes padding
    B int64   // 8 bytes at offset 8
    C bool    // 1 byte  at offset 16
              // 7 bytes padding
}
// Total: 24 bytes (8 bytes wasted!)

type Good struct {
    B int64   // 8 bytes at offset 0
    A bool    // 1 byte  at offset 8
    C bool    // 1 byte  at offset 9
              // 6 bytes padding (to align struct size to 8)
}
// Total: 16 bytes (8 bytes saved!)
```

### Visualizing Padding
```
Bad struct layout:
┌──┬───────────────┬──────────────────┬──┬───────────────┐
│A │   padding(7)  │       B(8)       │C │   padding(7)  │
└──┴───────────────┴──────────────────┴──┴───────────────┘
 0  1             8                  16 17              24

Good struct layout:
┌──────────────────┬──┬──┬──────────────────────────────┐
│       B(8)       │A │C │         padding(6)            │
└──────────────────┴──┴──┴──────────────────────────────┘
 0                8  9  10                              16
```

### Field Ordering for Minimal Padding
```go
// Rule: order fields largest to smallest
type Optimized struct {
    F float64  // 8 bytes — largest first
    I int32    // 4 bytes
    J int32    // 4 bytes
    B bool     // 1 byte
    C bool     // 1 byte
               // 6 bytes padding
}
// Total: 24 bytes

// Worst case ordering:
type Unoptimized struct {
    B bool     // 1 byte + 7 padding
    F float64  // 8 bytes
    C bool     // 1 byte + 3 padding
    I int32    // 4 bytes
}
// Total: 24 bytes (same here, but can be worse)
```

### Empty Struct
```go
type Empty struct{}

fmt.Println(unsafe.Sizeof(Empty{}))  // 0 — zero size!

// Uses:
// 1. Set implementation
set := make(map[string]struct{})
set["key"] = struct{}{}

// 2. Signal channels
done := make(chan struct{})
done <- struct{}{}

// 3. Method-only types
type Handler struct{}
func (h Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {}
```

---

## Struct Initialization

### Named Fields (Preferred)
```go
p := Person{
    Name:  "Alice",
    Age:   30,
    Email: "alice@example.com",
}
// Unspecified fields get zero value
p2 := Person{Name: "Bob"}  // Age=0, Email=""
```

### Positional (Fragile — Avoid)
```go
// Order must match declaration — breaks if fields are added/reordered
p := Person{"Alice", 30, "alice@example.com"}
```

### new() — Returns Pointer
```go
p := new(Person)       // *Person, all fields zeroed
p.Name = "Alice"
p.Age = 30
```

### Constructor Functions
```go
// Idiomatic Go: NewXxx function
func NewPerson(name string, age int) *Person {
    if age < 0 {
        panic("age cannot be negative")
    }
    return &Person{
        Name: name,
        Age:  age,
    }
}

p := NewPerson("Alice", 30)
```

### Functional Options Constructor
```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
    maxConn int
}

type ServerOption func(*Server)

func WithPort(port int) ServerOption {
    return func(s *Server) { s.port = port }
}

func WithTimeout(d time.Duration) ServerOption {
    return func(s *Server) { s.timeout = d }
}

func NewServer(host string, opts ...ServerOption) *Server {
    s := &Server{
        host:    host,
        port:    8080,           // defaults
        timeout: 30 * time.Second,
        maxConn: 100,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage
s := NewServer("localhost",
    WithPort(9090),
    WithTimeout(60*time.Second),
)
```

---

## Methods on Structs

### Value Receiver
```go
type Rectangle struct {
    Width, Height float64
}

// Value receiver — read-only, works on copy
func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.Width + r.Height)
}
```

### Pointer Receiver
```go
// Pointer receiver — can modify struct
func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}

r := Rectangle{Width: 10, Height: 5}
r.Scale(2)
fmt.Println(r.Area())  // 200
```

### Method Expressions and Values
```go
// Method value — bound to specific receiver
r := Rectangle{Width: 10, Height: 5}
areaFn := r.Area          // func() float64
fmt.Println(areaFn())     // 50

// Method expression — receiver passed as first argument
areaExpr := Rectangle.Area          // func(Rectangle) float64
fmt.Println(areaExpr(r))            // 50

scaleFn := (*Rectangle).Scale       // func(*Rectangle, float64)
scaleFn(&r, 2)
```

### Stringer Interface
```go
func (p Person) String() string {
    return fmt.Sprintf("%s (age %d)", p.Name, p.Age)
}

p := Person{Name: "Alice", Age: 30}
fmt.Println(p)  // Alice (age 30)
```

---

## Embedding

### Struct Embedding (Composition)
```go
type Animal struct {
    Name string
}

func (a Animal) Speak() string {
    return a.Name + " makes a sound"
}

type Dog struct {
    Animal        // Embedded — not a named field
    Breed string
}

func (d Dog) Speak() string {
    return d.Name + " barks"  // Promoted field
}

d := Dog{
    Animal: Animal{Name: "Rex"},
    Breed:  "Labrador",
}

fmt.Println(d.Name)         // "Rex" — promoted field
fmt.Println(d.Animal.Name)  // "Rex" — explicit access
fmt.Println(d.Speak())      // "Rex barks" — Dog's method
fmt.Println(d.Animal.Speak()) // "Rex makes a sound" — Animal's method
```

### Embedding Interfaces
```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Embed interfaces to compose new interfaces
type ReadWriter interface {
    Reader
    Writer
}

// Embed interface in struct — useful for mocking/wrapping
type LoggingReader struct {
    Reader        // Embedded interface
    logger *log.Logger
}

func (lr *LoggingReader) Read(p []byte) (int, error) {
    n, err := lr.Reader.Read(p)  // Delegate to embedded
    lr.logger.Printf("Read %d bytes", n)
    return n, err
}
```

### Embedding vs Inheritance
```go
// Go has NO inheritance — embedding is composition
// Promoted methods are syntactic sugar, not inheritance

type Base struct{ ID int }
func (b Base) Describe() string { return fmt.Sprintf("ID: %d", b.ID) }

type Derived struct {
    Base
    Name string
}

d := Derived{Base: Base{ID: 1}, Name: "test"}
d.Describe()        // Calls Base.Describe() — promoted
d.Base.Describe()   // Explicit — same result

// Derived does NOT "is-a" Base
// var b Base = d  // Compile error
var b Base = d.Base  // Must extract explicitly
```

### Embedding Pointer
```go
type Logger struct {
    prefix string
}

func (l *Logger) Log(msg string) {
    fmt.Printf("[%s] %s\n", l.prefix, msg)
}

type Service struct {
    *Logger        // Embedded pointer — shared logger
    name string
}

logger := &Logger{prefix: "INFO"}
s := Service{Logger: logger, name: "auth"}
s.Log("started")  // [INFO] started
```

### Field Promotion and Shadowing
```go
type A struct{ X int }
type B struct{ X int }

type C struct {
    A
    B
}

c := C{A: A{X: 1}, B: B{X: 2}}
// c.X  // Ambiguous — compile error
c.A.X   // 1 — explicit
c.B.X   // 2 — explicit

// Shadowing: C's own field takes priority
type D struct {
    A
    X int  // Shadows A.X
}

d := D{A: A{X: 1}, X: 99}
fmt.Println(d.X)    // 99 — D's own field
fmt.Println(d.A.X)  // 1  — explicit
```

---

## Anonymous Structs

### Inline Definition
```go
// One-off struct without a type name
point := struct {
    X, Y int
}{X: 10, Y: 20}

fmt.Println(point.X)  // 10
```

### In Function Returns
```go
func getConfig() struct {
    Host string
    Port int
} {
    return struct {
        Host string
        Port int
    }{Host: "localhost", Port: 8080}
}
```

### In Test Tables
```go
tests := []struct {
    input    string
    expected int
}{
    {"hello", 5},
    {"world", 5},
    {"go", 2},
}

for _, tt := range tests {
    got := len(tt.input)
    if got != tt.expected {
        t.Errorf("len(%q) = %d, want %d", tt.input, got, tt.expected)
    }
}
```

### JSON Decoding
```go
var result struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}
json.Unmarshal(data, &result)
```

---

## Struct Tags

### Syntax
```go
type User struct {
    ID       int    `json:"id" db:"user_id"`
    Name     string `json:"name" validate:"required,min=2"`
    Email    string `json:"email,omitempty" validate:"email"`
    Password string `json:"-"`  // Excluded from JSON
}
```

### Reading Tags via Reflection
```go
import "reflect"

type User struct {
    Name string `json:"name" validate:"required"`
}

t := reflect.TypeOf(User{})
field, _ := t.FieldByName("Name")

jsonTag := field.Tag.Get("json")       // "name"
validateTag := field.Tag.Get("validate") // "required"
```

### Common Tag Packages
```go
// encoding/json
type T struct {
    Field string `json:"field_name"`           // rename
    Opt   string `json:"opt,omitempty"`        // omit if zero
    Skip  string `json:"-"`                    // always skip
    Str   int    `json:",string"`              // encode as string
}

// database/sql (via sqlx, gorm, etc.)
type T struct {
    ID   int    `db:"id"`
    Name string `db:"name"`
}

// encoding/xml
type T struct {
    XMLName xml.Name `xml:"person"`
    Name    string   `xml:"name,attr"`
    Age     int      `xml:"age"`
}

// gopkg.in/yaml.v3
type T struct {
    Name string `yaml:"name"`
    Age  int    `yaml:"age,omitempty"`
}
```

---

## Struct Comparison

### Comparable Structs
```go
// Structs are comparable if all fields are comparable
type Point struct{ X, Y int }

p1 := Point{1, 2}
p2 := Point{1, 2}
p3 := Point{3, 4}

fmt.Println(p1 == p2)  // true
fmt.Println(p1 == p3)  // false
fmt.Println(p1 != p3)  // true
```

### Non-comparable Structs
```go
// Structs with slice, map, or function fields are NOT comparable
type Bad struct {
    Data []int  // slice — not comparable
}

b1 := Bad{Data: []int{1, 2}}
b2 := Bad{Data: []int{1, 2}}
// b1 == b2  // Compile error: struct containing []int cannot be compared

// Use reflect.DeepEqual for deep comparison
fmt.Println(reflect.DeepEqual(b1, b2))  // true
```

### Using as Map Keys
```go
// Comparable structs can be map keys
type Point struct{ X, Y int }
m := map[Point]string{
    {0, 0}: "origin",
    {1, 0}: "right",
}
fmt.Println(m[Point{0, 0}])  // "origin"
```

---

## Struct Patterns

### Builder Pattern
```go
type QueryBuilder struct {
    table      string
    conditions []string
    limit      int
    offset     int
}

func (qb *QueryBuilder) From(table string) *QueryBuilder {
    qb.table = table
    return qb
}

func (qb *QueryBuilder) Where(cond string) *QueryBuilder {
    qb.conditions = append(qb.conditions, cond)
    return qb
}

func (qb *QueryBuilder) Limit(n int) *QueryBuilder {
    qb.limit = n
    return qb
}

func (qb *QueryBuilder) Build() string {
    q := "SELECT * FROM " + qb.table
    if len(qb.conditions) > 0 {
        q += " WHERE " + strings.Join(qb.conditions, " AND ")
    }
    if qb.limit > 0 {
        q += fmt.Sprintf(" LIMIT %d", qb.limit)
    }
    return q
}

// Usage
query := (&QueryBuilder{}).
    From("users").
    Where("age > 18").
    Where("active = true").
    Limit(10).
    Build()
```

### Immutable Struct
```go
// Unexported fields + constructor + getters = immutable
type Money struct {
    amount   int64  // cents
    currency string
}

func NewMoney(amount int64, currency string) Money {
    return Money{amount: amount, currency: currency}
}

func (m Money) Amount() int64    { return m.amount }
func (m Money) Currency() string { return m.currency }

func (m Money) Add(other Money) (Money, error) {
    if m.currency != other.currency {
        return Money{}, errors.New("currency mismatch")
    }
    return Money{amount: m.amount + other.amount, currency: m.currency}, nil
}
```

### Copy-on-Write
```go
type Config struct {
    host    string
    port    int
    timeout time.Duration
}

func (c Config) WithHost(host string) Config {
    c.host = host  // Modifies copy
    return c       // Returns new Config
}

func (c Config) WithPort(port int) Config {
    c.port = port
    return c
}

// Usage — original unchanged
base := Config{host: "localhost", port: 8080}
prod := base.WithHost("prod.example.com").WithPort(443)
```

---

## Common Pitfalls

### 1. Copying Mutex
```go
// Problem: copying a struct with a mutex
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

c1 := SafeCounter{}
c2 := c1  // Copies mutex — WRONG! Mutex must not be copied after first use

// Fix: use pointer
c2 := &c1  // Share the same mutex
// Or: always pass *SafeCounter
```

### 2. Nil Pointer to Struct
```go
type Node struct {
    Val  int
    Next *Node
}

var n *Node
fmt.Println(n.Val)  // panic: nil pointer dereference

// Fix: check nil
if n != nil {
    fmt.Println(n.Val)
}
```

### 3. Struct Embedding Confusion
```go
type Logger struct{}
func (l Logger) Log(msg string) { fmt.Println(msg) }

type Service struct {
    Logger
}

s := Service{}
s.Log("hello")         // OK — promoted method
s.Logger.Log("hello")  // Also OK — explicit

// But: Service does NOT implement an interface requiring Log
// unless the interface is satisfied by the promoted method
type Loggable interface { Log(string) }
var _ Loggable = Service{}  // OK — promoted method satisfies interface
```

### 4. Unexported Fields and JSON
```go
type secret struct {
    password string  // unexported — JSON ignores it
}

s := secret{password: "abc123"}
data, _ := json.Marshal(s)
fmt.Println(string(data))  // {} — empty!

// Fix: export fields or implement json.Marshaler
type Secret struct {
    Password string `json:"password"`
}
```

---

## Performance Optimization

### Field Ordering for Cache Efficiency
```go
// Hot fields together — better cache locality
type HotCold struct {
    // Hot fields (frequently accessed)
    Count    int64
    LastSeen time.Time
    // Cold fields (rarely accessed)
    Name        string
    Description string
    Tags        []string
}
```

### Avoid Large Value Copies
```go
// Bad: copies entire struct on each call
func processLarge(s LargeStruct) { /* ... */ }

// Good: pass pointer
func processLarge(s *LargeStruct) { /* ... */ }

// Good: return pointer from constructor
func NewLargeStruct() *LargeStruct {
    return &LargeStruct{ /* ... */ }
}
```

### Struct Size and Alignment
```go
// Use fieldalignment tool to optimize struct layout
// go install golang.org/x/tools/go/analysis/passes/fieldalignment/cmd/fieldalignment@latest
// fieldalignment -fix ./...

// Before:
type Before struct {
    A bool    // 1 + 7 padding
    B int64   // 8
    C bool    // 1 + 7 padding
}  // 24 bytes

// After:
type After struct {
    B int64   // 8
    A bool    // 1
    C bool    // 1 + 6 padding
}  // 16 bytes
```

---

## Summary

### Key Concepts
1. Structs are value types — assignment copies all fields
2. Fields stored contiguously; padding added for alignment
3. Order fields largest-to-smallest to minimize padding
4. Embedding provides composition (not inheritance)
5. Promoted fields/methods accessible directly on outer struct
6. Struct tags enable metadata for encoding, validation, ORM

### Decision Guide
```
Need to group related data?           → struct
Need methods that modify state?       → pointer receiver
Need immutable value object?          → value receiver + unexported fields
Need optional configuration?          → functional options pattern
Need to compose behavior?             → embedding
Need zero-allocation set/signal?      → struct{}
Need to minimize memory?              → order fields largest-to-smallest
```

# Types and Generics Deep Dive

## Table of Contents
- [Type System Fundamentals](#type-system-fundamentals)
- [Named Types](#named-types)
- [Type Aliases](#type-aliases)
- [Type Conversions](#type-conversions)
- [Type Assertions and Reflection](#type-assertions-and-reflection)
- [Generics (Go 1.18+)](#generics-go-118)
- [Type Constraints](#type-constraints)
- [Generic Functions](#generic-functions)
- [Generic Types](#generic-types)
- [Type Inference](#type-inference)
- [Comparable and Ordered](#comparable-and-ordered)
- [Common Generic Patterns](#common-generic-patterns)
- [Reflection](#reflection)
- [Common Pitfalls](#common-pitfalls)

---

## Type System Fundamentals

### Built-in Types
```go
// Boolean
bool

// Integer
int, int8, int16, int32, int64
uint, uint8, uint16, uint32, uint64, uintptr

// Float
float32, float64

// Complex
complex64, complex128

// String
string

// Byte and Rune
byte   // alias for uint8
rune   // alias for int32 (Unicode code point)

// Architecture-dependent
int    // 32 or 64 bit depending on platform
uint   // 32 or 64 bit
uintptr // large enough to hold pointer value
```

### Type Sizes
```go
import "unsafe"

fmt.Println(unsafe.Sizeof(bool(false)))    // 1
fmt.Println(unsafe.Sizeof(int8(0)))        // 1
fmt.Println(unsafe.Sizeof(int16(0)))       // 2
fmt.Println(unsafe.Sizeof(int32(0)))       // 4
fmt.Println(unsafe.Sizeof(int64(0)))       // 8
fmt.Println(unsafe.Sizeof(float32(0)))     // 4
fmt.Println(unsafe.Sizeof(float64(0)))     // 8
fmt.Println(unsafe.Sizeof(string("")))     // 16 (ptr + len)
fmt.Println(unsafe.Sizeof([]int{}))        // 24 (ptr + len + cap)
fmt.Println(unsafe.Sizeof(map[int]int{}))  // 8 (pointer to hmap)
```

### Zero Values
```go
var b bool       // false
var i int        // 0
var f float64    // 0.0
var s string     // ""
var p *int       // nil
var sl []int     // nil
var m map[string]int // nil
var fn func()    // nil
var ch chan int   // nil
var iface interface{} // nil
```

---

## Named Types

### Defining Named Types
```go
// Named type based on existing type
type Celsius float64
type Fahrenheit float64
type Meters float64
type UserID int64

// Named types are distinct — no implicit conversion
var c Celsius = 100
var f Fahrenheit = 212
// c = f  // Compile error: cannot use f (type Fahrenheit) as type Celsius
```

### Methods on Named Types
```go
type Celsius float64

func (c Celsius) ToFahrenheit() Fahrenheit {
    return Fahrenheit(c*9/5 + 32)
}

func (c Celsius) String() string {
    return fmt.Sprintf("%.2f°C", float64(c))
}

c := Celsius(100)
fmt.Println(c)                // 100.00°C
fmt.Println(c.ToFahrenheit()) // 212
```

### Named Slice/Map Types
```go
type StringSlice []string

func (s StringSlice) Contains(target string) bool {
    for _, v := range s {
        if v == target {
            return true
        }
    }
    return false
}

func (s StringSlice) Join(sep string) string {
    return strings.Join(s, sep)
}

words := StringSlice{"hello", "world", "go"}
fmt.Println(words.Contains("go"))   // true
fmt.Println(words.Join(", "))       // hello, world, go
```

### Named Function Types
```go
type Predicate func(int) bool
type Transform func(int) int
type Handler func(http.ResponseWriter, *http.Request)

// Methods on function types
type Middleware func(http.Handler) http.Handler

func (m Middleware) Then(next http.Handler) http.Handler {
    return m(next)
}
```

---

## Type Aliases

### Alias vs Named Type
```go
// Type alias — same type, just another name
type MyString = string  // Alias: MyString IS string

// Named type — new distinct type
type MyType string      // Named: MyType is based on string but distinct

var s string = "hello"
var ms MyString = s     // OK — same type
// var mt MyType = s    // Error — different types (need explicit conversion)
var mt MyType = MyType(s) // OK — explicit conversion
```

### When to Use Aliases
```go
// 1. Gradual code migration
// Old package
package oldpkg
type Config struct{ /* ... */ }

// New package — alias for backward compatibility
package newpkg
import "oldpkg"
type Config = oldpkg.Config  // Alias — same type

// 2. Shorter names for long types
type Handler = func(http.ResponseWriter, *http.Request)

// 3. Interface satisfaction across packages
// byte = uint8, rune = int32 are built-in aliases
```

---

## Type Conversions

### Explicit Conversions
```go
// Go requires explicit type conversions (no implicit)
var i int = 42
var f float64 = float64(i)  // Explicit
var u uint = uint(f)

// String conversions
s := string(65)          // "A" — rune to string
b := []byte("hello")     // string to []byte
r := []rune("hello")     // string to []rune
s2 := string(b)          // []byte to string

// Numeric string conversions (use strconv)
n, _ := strconv.Atoi("42")          // string to int
s3 := strconv.Itoa(42)              // int to string
f2, _ := strconv.ParseFloat("3.14", 64)
s4 := strconv.FormatFloat(3.14, 'f', 2, 64)
```

### Conversion Rules
```go
// Allowed conversions:
// - Between numeric types (may lose precision)
// - Between string and []byte / []rune
// - Between named types with same underlying type
// - Pointer to unsafe.Pointer and back

// NOT allowed:
// - Between incompatible types
// - Implicit conversions (always explicit in Go)

type Meters float64
type Feet float64

m := Meters(1.0)
// f := Feet(m)  // Error — different named types
f := Feet(float64(m) * 3.28084)  // Must go through underlying type
```

---

## Type Assertions and Reflection

### Type Assertion
```go
var i interface{} = "hello"

// Panics if wrong type
s := i.(string)

// Safe — no panic
s, ok := i.(string)
if ok {
    fmt.Println(s)
}
```

### reflect.TypeOf and reflect.ValueOf
```go
import "reflect"

x := 42
t := reflect.TypeOf(x)
v := reflect.ValueOf(x)

fmt.Println(t)          // int
fmt.Println(t.Kind())   // int
fmt.Println(v)          // 42
fmt.Println(v.Int())    // 42

// Struct reflection
type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}

p := Person{"Alice", 30}
t = reflect.TypeOf(p)
v = reflect.ValueOf(p)

for i := 0; i < t.NumField(); i++ {
    field := t.Field(i)
    value := v.Field(i)
    fmt.Printf("%s (%s) = %v, tag: %s\n",
        field.Name, field.Type, value, field.Tag.Get("json"))
}
// Name (string) = Alice, tag: name
// Age (int) = 30, tag: age
```

---

## Generics (Go 1.18+)

### Why Generics?
```go
// Before generics: duplicate code or interface{}
func sumInts(s []int) int {
    total := 0
    for _, v := range s { total += v }
    return total
}

func sumFloat64s(s []float64) float64 {
    total := 0.0
    for _, v := range s { total += v }
    return total
}

// With generics: one function for all numeric types
func sum[T int | float64](s []T) T {
    var total T
    for _, v := range s { total += v }
    return total
}

fmt.Println(sum([]int{1, 2, 3}))          // 6
fmt.Println(sum([]float64{1.1, 2.2, 3.3})) // 6.6
```

### Generic Syntax
```go
// Type parameter list: [T constraint]
func FunctionName[T TypeConstraint](params) returnType {
    // T is a type parameter
}

// Multiple type parameters
func Map[K comparable, V any](m map[K]V, fn func(V) V) map[K]V {
    result := make(map[K]V, len(m))
    for k, v := range m {
        result[k] = fn(v)
    }
    return result
}
```

---

## Type Constraints

### Built-in Constraints
```go
import "golang.org/x/exp/constraints"

// any — no constraint (same as interface{})
func Print[T any](v T) { fmt.Println(v) }

// comparable — supports == and !=
func Contains[T comparable](s []T, v T) bool {
    for _, item := range s {
        if item == v { return true }
    }
    return false
}
```

### Union Constraints
```go
// Inline union
func add[T int | float64 | string](a, b T) T {
    return a + b
}

// Named constraint interface
type Number interface {
    int | int8 | int16 | int32 | int64 |
    uint | uint8 | uint16 | uint32 | uint64 |
    float32 | float64
}

func sum[T Number](s []T) T {
    var total T
    for _, v := range s { total += v }
    return total
}
```

### Tilde (~) for Underlying Types
```go
// ~ means "any type whose underlying type is T"
type Celsius float64
type Fahrenheit float64

type Float interface {
    ~float32 | ~float64  // Includes named types based on float32/float64
}

func double[T Float](v T) T {
    return v * 2
}

double(3.14)           // float64
double(Celsius(100))   // Celsius — works because ~float64
double(Fahrenheit(32)) // Fahrenheit — works because ~float64
```

### Constraint with Methods
```go
// Constraint requiring both type set and methods
type Stringer interface {
    String() string
}

type StringerNumber interface {
    Number
    String() string
}

// Constraint combining type set and method
type Printable interface {
    ~int | ~string
    String() string
}
```

### golang.org/x/exp/constraints Package
```go
import "golang.org/x/exp/constraints"

// constraints.Integer — all integer types
// constraints.Float   — all float types
// constraints.Ordered — all types supporting <, >, <=, >=
// constraints.Signed  — all signed integer types
// constraints.Unsigned — all unsigned integer types
// constraints.Complex — all complex types

func Min[T constraints.Ordered](a, b T) T {
    if a < b { return a }
    return b
}

func Max[T constraints.Ordered](a, b T) T {
    if a > b { return a }
    return b
}
```

---

## Generic Functions

### Slice Operations
```go
// Map
func Map[T, U any](s []T, fn func(T) U) []U {
    result := make([]U, len(s))
    for i, v := range s {
        result[i] = fn(v)
    }
    return result
}

// Filter
func Filter[T any](s []T, fn func(T) bool) []T {
    result := []T{}
    for _, v := range s {
        if fn(v) { result = append(result, v) }
    }
    return result
}

// Reduce
func Reduce[T, U any](s []T, init U, fn func(U, T) U) U {
    acc := init
    for _, v := range s {
        acc = fn(acc, v)
    }
    return acc
}

// Usage
nums := []int{1, 2, 3, 4, 5}
doubled := Map(nums, func(n int) int { return n * 2 })
evens   := Filter(nums, func(n int) bool { return n%2 == 0 })
sum     := Reduce(nums, 0, func(acc, n int) int { return acc + n })
strs    := Map(nums, strconv.Itoa)  // []string{"1","2","3","4","5"}
```

### Generic Keys/Values
```go
func Keys[K comparable, V any](m map[K]V) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}

func Values[K comparable, V any](m map[K]V) []V {
    vals := make([]V, 0, len(m))
    for _, v := range m {
        vals = append(vals, v)
    }
    return vals
}

func Contains[T comparable](s []T, v T) bool {
    for _, item := range s {
        if item == v { return true }
    }
    return false
}
```

---

## Generic Types

### Generic Stack
```go
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    n := len(s.items) - 1
    item := s.items[n]
    s.items = s.items[:n]
    return item, true
}

func (s *Stack[T]) Peek() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    return s.items[len(s.items)-1], true
}

func (s *Stack[T]) Len() int { return len(s.items) }

// Usage
s := Stack[int]{}
s.Push(1); s.Push(2); s.Push(3)
v, _ := s.Pop()  // 3

ss := Stack[string]{}
ss.Push("hello")
```

### Generic Pair / Result
```go
type Pair[A, B any] struct {
    First  A
    Second B
}

func NewPair[A, B any](a A, b B) Pair[A, B] {
    return Pair[A, B]{First: a, Second: b}
}

p := NewPair("hello", 42)
fmt.Println(p.First, p.Second)  // hello 42

// Result type
type Result[T any] struct {
    Value T
    Err   error
}

func (r Result[T]) Unwrap() T {
    if r.Err != nil { panic(r.Err) }
    return r.Value
}

func divide[T constraints.Float](a, b T) Result[T] {
    if b == 0 {
        return Result[T]{Err: errors.New("division by zero")}
    }
    return Result[T]{Value: a / b}
}
```

### Generic Set
```go
type Set[T comparable] struct {
    items map[T]struct{}
}

func NewSet[T comparable]() *Set[T] {
    return &Set[T]{items: make(map[T]struct{})}
}

func (s *Set[T]) Add(v T)           { s.items[v] = struct{}{} }
func (s *Set[T]) Remove(v T)        { delete(s.items, v) }
func (s *Set[T]) Contains(v T) bool { _, ok := s.items[v]; return ok }
func (s *Set[T]) Len() int          { return len(s.items) }

func (s *Set[T]) Union(other *Set[T]) *Set[T] {
    result := NewSet[T]()
    for k := range s.items    { result.Add(k) }
    for k := range other.items { result.Add(k) }
    return result
}

func (s *Set[T]) Intersection(other *Set[T]) *Set[T] {
    result := NewSet[T]()
    for k := range s.items {
        if other.Contains(k) { result.Add(k) }
    }
    return result
}
```

---

## Type Inference

### Automatic Type Inference
```go
// Go infers type parameters from arguments
func identity[T any](v T) T { return v }

identity(42)       // T inferred as int
identity("hello")  // T inferred as string
identity(3.14)     // T inferred as float64

// Explicit when inference fails
identity[int](42)  // Explicit
```

### Inference Limitations
```go
// Cannot infer return-only type parameters
func zero[T any]() T {
    var v T
    return v
}

// Must specify explicitly
zero[int]()     // OK
zero[string]()  // OK
// zero()       // Error: cannot infer T
```

---

## Comparable and Ordered

### comparable Constraint
```go
// comparable: supports == and !=
// Includes: bool, int, float, string, pointer, channel, array, struct (with comparable fields)
// Excludes: slice, map, function

func Equal[T comparable](a, b T) bool {
    return a == b
}

Equal(1, 1)         // true
Equal("a", "b")     // false
// Equal([]int{}, []int{})  // Error: []int is not comparable
```

### Ordered Constraint
```go
// From golang.org/x/exp/constraints or define inline
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 | ~uintptr |
    ~float32 | ~float64 |
    ~string
}

func Clamp[T Ordered](v, min, max T) T {
    if v < min { return min }
    if v > max { return max }
    return v
}

Clamp(5, 1, 10)    // 5
Clamp(-1, 1, 10)   // 1
Clamp(15, 1, 10)   // 10
```

---

## Common Generic Patterns

### Optional / Maybe
```go
type Optional[T any] struct {
    value *T
}

func Some[T any](v T) Optional[T] {
    return Optional[T]{value: &v}
}

func None[T any]() Optional[T] {
    return Optional[T]{}
}

func (o Optional[T]) IsPresent() bool { return o.value != nil }

func (o Optional[T]) Get() (T, bool) {
    if o.value == nil {
        var zero T
        return zero, false
    }
    return *o.value, true
}

func (o Optional[T]) OrElse(def T) T {
    if o.value == nil { return def }
    return *o.value
}
```

### Generic Cache
```go
type Cache[K comparable, V any] struct {
    mu    sync.RWMutex
    items map[K]V
}

func NewCache[K comparable, V any]() *Cache[K, V] {
    return &Cache[K, V]{items: make(map[K]V)}
}

func (c *Cache[K, V]) Set(key K, val V) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = val
}

func (c *Cache[K, V]) Get(key K) (V, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    v, ok := c.items[key]
    return v, ok
}

func (c *Cache[K, V]) GetOrSet(key K, fn func() V) V {
    if v, ok := c.Get(key); ok {
        return v
    }
    v := fn()
    c.Set(key, v)
    return v
}
```

---

## Reflection

### When to Use Reflection
```go
// Use reflection when:
// 1. Type is unknown at compile time
// 2. Working with arbitrary structs (JSON, ORM, etc.)
// 3. Building generic frameworks

// Avoid reflection when:
// 1. Type is known — use generics or interfaces
// 2. Performance is critical (~100x slower than direct calls)
```

### Struct Inspection
```go
func inspectStruct(v interface{}) {
    t := reflect.TypeOf(v)
    val := reflect.ValueOf(v)

    if t.Kind() == reflect.Ptr {
        t = t.Elem()
        val = val.Elem()
    }

    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        value := val.Field(i)
        fmt.Printf("Field: %s, Type: %s, Value: %v\n",
            field.Name, field.Type, value)
    }
}
```

### Dynamic Method Calls
```go
func callMethod(obj interface{}, method string, args ...interface{}) []reflect.Value {
    v := reflect.ValueOf(obj)
    m := v.MethodByName(method)
    if !m.IsValid() {
        panic("method not found: " + method)
    }

    in := make([]reflect.Value, len(args))
    for i, arg := range args {
        in[i] = reflect.ValueOf(arg)
    }
    return m.Call(in)
}
```

---

## Common Pitfalls

### 1. Generics Don't Support Operator Overloading
```go
// Cannot use + on arbitrary T
func add[T any](a, b T) T {
    // return a + b  // Error: operator + not defined on T
}

// Must constrain to types that support +
func add[T Number](a, b T) T {
    return a + b  // OK — Number constraint includes +
}
```

### 2. Type Parameter Cannot Be Used as Interface
```go
// Cannot use type parameter as interface directly
func wrong[T any](v T) {
    // v.String()  // Error: T has no method String
}

// Fix: add constraint
func correct[T fmt.Stringer](v T) {
    v.String()  // OK
}
```

### 3. Reflection Panics on Unexported Fields
```go
type secret struct{ password string }

s := secret{password: "abc"}
v := reflect.ValueOf(s)
f := v.FieldByName("password")
// f.String()  // panic: reflect.Value.String on unexported field
```

### 4. Named Type vs Underlying Type in Generics
```go
type MyInt int

func double[T ~int](v T) T { return v * 2 }

double(42)        // OK — int
double(MyInt(5))  // OK — ~int includes MyInt
// double(int32(5)) // Error — int32 is not ~int
```

---

## Summary

### Key Concepts
1. Named types are distinct from their underlying type — explicit conversion required
2. Type aliases (`=`) are the same type; named types are new types
3. Generics use type parameters `[T constraint]` for type-safe reuse
4. `~T` in constraints includes all types with underlying type T
5. `comparable` for equality; `constraints.Ordered` for ordering
6. Type inference works for function arguments; explicit for return-only params

### Decision Guide
```
Same logic for multiple types?        → Generics
Unknown type at runtime?              → Reflection or interface{}
Need type-safe container?             → Generic type (Stack[T], Set[T])
Need to add methods to existing type? → Named type
Need backward-compatible rename?      → Type alias
Need to constrain to numeric types?   → Number constraint
Need to constrain to comparable?      → comparable
```

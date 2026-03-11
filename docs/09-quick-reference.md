# Golang Quick Reference Guide

## Language Fundamentals

### Variable Declaration
```go
var x int = 10          // Explicit type
var y = 20              // Type inference
z := 30                 // Short declaration (functions only)
const PI = 3.14         // Constant
```

### Data Types
- **Basic**: int, float64, string, bool, byte, rune
- **Composite**: array, slice, map, struct, pointer, function, interface, channel
- **Zero values**: 0, 0.0, "", false, nil

### Control Flow
```go
// If
if x > 0 {
} else if x < 0 {
} else {
}

// Switch
switch x {
case 1:
case 2:
default:
}

// For (only loop in Go)
for i := 0; i < 10; i++ {}
for condition {}
for {}
for i, v := range collection {}
```

## Data Structures Cheat Sheet

| Structure | Access | Insert | Delete | Search | Use Case |
|-----------|--------|--------|--------|--------|----------|
| Array | O(1) | N/A | N/A | O(n) | Fixed size |
| Slice | O(1) | O(1)* | O(n) | O(n) | Dynamic array |
| Map | O(1) | O(1) | O(1) | O(1) | Key-value |
| Linked List | O(n) | O(1) | O(1) | O(n) | Insert/delete |
| Stack | O(1) | O(1) | O(1) | O(n) | LIFO |
| Queue | O(1) | O(1) | O(1) | O(n) | FIFO |
| Binary Tree | O(log n) | O(log n) | O(log n) | O(log n) | Hierarchical |
| Heap | O(1) | O(log n) | O(log n) | O(n) | Priority queue |

*Amortized

## Algorithm Complexity

### Sorting
- Bubble Sort: O(n²)
- Selection Sort: O(n²)
- Insertion Sort: O(n²)
- Merge Sort: O(n log n)
- Quick Sort: O(n log n) avg, O(n²) worst
- Heap Sort: O(n log n)

### Searching
- Linear Search: O(n)
- Binary Search: O(log n) - requires sorted array

### Common Patterns
- Two Pointers: O(n)
- Sliding Window: O(n)
- Binary Search: O(log n)
- DFS/BFS: O(V + E)
- Dynamic Programming: Problem-specific

## Concurrency Patterns

### Goroutines
```go
go func() {
    // Concurrent execution
}()
```

### Channels
```go
ch := make(chan int)        // Unbuffered
ch := make(chan int, 10)    // Buffered
ch <- value                 // Send
value := <-ch               // Receive
close(ch)                   // Close
```

### Select
```go
select {
case v := <-ch1:
case ch2 <- v:
case <-time.After(timeout):
default:
}
```

### Synchronization
```go
var wg sync.WaitGroup       // Wait for goroutines
var mu sync.Mutex           // Mutual exclusion
var rwmu sync.RWMutex       // Read-write lock
var once sync.Once          // Execute once
atomic.AddInt64(&counter, 1) // Atomic operations
```

### Context
```go
ctx, cancel := context.WithCancel(context.Background())
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
ctx, cancel := context.WithDeadline(ctx, time.Now().Add(5*time.Second))
ctx = context.WithValue(ctx, key, value)
```

## Error Handling

### Basic
```go
if err != nil {
    return err
}
```

### Wrapping
```go
return fmt.Errorf("operation failed: %w", err)
```

### Checking
```go
if errors.Is(err, ErrNotFound) {}
if errors.As(err, &customErr) {}
```

### Custom Errors
```go
type MyError struct {
    Msg string
}

func (e *MyError) Error() string {
    return e.Msg
}
```

## Testing

### Unit Test
```go
func TestAdd(t *testing.T) {
    result := Add(2, 3)
    if result != 5 {
        t.Errorf("got %d, want 5", result)
    }
}
```

### Table-Driven
```go
tests := []struct {
    name string
    input int
    want int
}{
    {"case1", 1, 2},
    {"case2", 2, 4},
}

for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got := Double(tt.input)
        if got != tt.want {
            t.Errorf("got %d, want %d", got, tt.want)
        }
    })
}
```

### Benchmark
```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}
```

## Common Idioms

### Error Handling
```go
if err := doSomething(); err != nil {
    return err
}
```

### Defer for Cleanup
```go
f, err := os.Open("file.txt")
if err != nil {
    return err
}
defer f.Close()
```

### Interface Satisfaction
```go
var _ io.Reader = (*MyReader)(nil) // Compile-time check
```

### Functional Options
```go
type Option func(*Config)

func WithTimeout(d time.Duration) Option {
    return func(c *Config) {
        c.Timeout = d
    }
}

func New(opts ...Option) *Config {
    c := &Config{}
    for _, opt := range opts {
        opt(c)
    }
    return c
}
```

## Performance Tips

1. **Preallocate slices**: `make([]int, 0, capacity)`
2. **Use strings.Builder**: For string concatenation
3. **Avoid unnecessary allocations**: Reuse objects with sync.Pool
4. **Use pointers for large structs**: Avoid copying
5. **Profile before optimizing**: Use pprof
6. **Benchmark changes**: Use go test -bench
7. **Use buffered channels**: When appropriate
8. **Avoid goroutine leaks**: Always provide exit mechanism

## Common Pitfalls

### 1. Loop Variable Capture
```go
// Wrong
for _, v := range values {
    go func() {
        fmt.Println(v) // Captures loop variable
    }()
}

// Correct
for _, v := range values {
    v := v // Create new variable
    go func() {
        fmt.Println(v)
    }()
}
```

### 2. Nil Interface
```go
var p *MyType = nil
var i interface{} = p
fmt.Println(i == nil) // false! Interface is not nil
```

### 3. Slice Append
```go
a := []int{1, 2, 3}
b := a[0:2]
b = append(b, 4) // May modify a!
```

### 4. Map Concurrent Access
```go
// Wrong - race condition
m := make(map[string]int)
go func() { m["key"] = 1 }()
go func() { m["key"] = 2 }()

// Correct - use mutex or sync.Map
```

### 5. Defer in Loop
```go
// Wrong - defers accumulate
for _, file := range files {
    f, _ := os.Open(file)
    defer f.Close() // Defers until function returns
}

// Correct - use function
for _, file := range files {
    func() {
        f, _ := os.Open(file)
        defer f.Close()
    }()
}
```

## Go Commands

```bash
# Build
go build                    # Build current package
go build -o output          # Build with custom name
go build -race              # Build with race detector

# Run
go run main.go              # Compile and run
go run -race main.go        # Run with race detector

# Test
go test                     # Run tests
go test -v                  # Verbose output
go test -cover              # With coverage
go test -race               # With race detector
go test -bench=.            # Run benchmarks
go test -cpuprofile=cpu.prof # CPU profiling

# Format
go fmt ./...                # Format all files
gofmt -w file.go            # Format specific file

# Lint
go vet ./...                # Check for common mistakes
golint ./...                # Style checker

# Modules
go mod init                 # Initialize module
go mod tidy                 # Clean up dependencies
go mod vendor               # Vendor dependencies
go get package@version      # Add/update dependency

# Tools
go doc package              # View documentation
go tool pprof profile       # Profile analysis
go tool cover -html=c.out   # View coverage
```

## Best Practices Summary

1. **Code Organization**: Small packages, clear names
2. **Error Handling**: Always check errors, wrap with context
3. **Concurrency**: Use channels for communication, mutexes for state
4. **Testing**: Write tests first, use table-driven tests
5. **Performance**: Profile before optimizing
6. **Documentation**: Comment exported identifiers
7. **Simplicity**: Prefer simple solutions
8. **Interfaces**: Small, focused interfaces
9. **Naming**: Clear, descriptive names
10. **Idioms**: Follow Go conventions

## Interview Preparation Checklist

### Fundamentals
- [ ] Variables, types, and zero values
- [ ] Control structures (if, for, switch)
- [ ] Functions and methods
- [ ] Pointers and values
- [ ] Structs and interfaces
- [ ] Arrays, slices, and maps

### Concurrency
- [ ] Goroutines and channels
- [ ] Select statement
- [ ] Mutexes and synchronization
- [ ] Context package
- [ ] Common patterns (worker pool, pipeline)

### Data Structures
- [ ] Implement common structures
- [ ] Understand time/space complexity
- [ ] Know when to use each

### Algorithms
- [ ] Sorting and searching
- [ ] Recursion and DP
- [ ] Two pointers and sliding window
- [ ] Graph algorithms (BFS, DFS)

### Best Practices
- [ ] Error handling patterns
- [ ] Testing strategies
- [ ] Performance optimization
- [ ] Code organization
- [ ] Common pitfalls

### System Design
- [ ] Scalability concepts
- [ ] Caching strategies
- [ ] Rate limiting
- [ ] Load balancing
- [ ] Database design

## Resources

### Official
- [Go Documentation](https://go.dev/doc/)
- [Effective Go](https://go.dev/doc/effective_go)
- [Go Blog](https://go.dev/blog/)

### Books
- "The Go Programming Language" by Donovan & Kernighan
- "Concurrency in Go" by Katherine Cox-Buday
- "Go in Action" by William Kennedy

### Practice
- [Go by Example](https://gobyexample.com/)
- [LeetCode](https://leetcode.com/)
- [HackerRank](https://www.hackerrank.com/)
- [Exercism](https://exercism.org/tracks/go)

# Goroutines - Deep Dive

## Table of Contents
- [Introduction](#introduction)
- [What is a Goroutine?](#what-is-a-goroutine)
- [Goroutine vs Thread](#goroutine-vs-thread)
- [How Goroutines Work](#how-goroutines-work)
- [Goroutine Lifecycle](#goroutine-lifecycle)
- [Creating Goroutines](#creating-goroutines)
- [Goroutine Scheduling](#goroutine-scheduling)
- [Stack Management](#stack-management)
- [Goroutine States](#goroutine-states)
- [Common Patterns](#common-patterns)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)
- [Performance Considerations](#performance-considerations)
- [Debugging Goroutines](#debugging-goroutines)

---

## Introduction

Goroutines are the fundamental unit of concurrency in Go. They are lightweight, independently executing functions that run concurrently with other goroutines in the same address space.

### Key Characteristics
- **Lightweight**: Start with only 2KB of stack space
- **Cheap**: Can create millions of goroutines
- **Managed**: Scheduled by Go runtime, not OS
- **Concurrent**: Run independently but share memory
- **Cooperative**: Yield control at specific points

## What is a Goroutine?

A goroutine is a function executing concurrently with other goroutines in the same address space.

### Anatomy of a Goroutine

```go
// Regular function call (synchronous)
doWork()

// Goroutine (asynchronous)
go doWork()
```

**Components:**
1. **Function**: The code to execute
2. **Stack**: Memory for local variables (starts at 2KB)
3. **Program Counter**: Current execution point
4. **Goroutine ID**: Internal identifier
5. **Status**: Current state (running, waiting, etc.)

## Goroutine vs Thread

### Comparison Table

| Feature | Goroutine | OS Thread |
|---------|-----------|-----------|
| **Creation Cost** | ~2KB stack | 1-2MB stack |
| **Switching Cost** | ~200ns | ~1-2μs |
| **Scheduling** | Go runtime (M:N) | OS kernel |
| **Number** | Millions possible | Thousands max |
| **Memory** | Growable stack | Fixed stack |
| **Management** | Automatic | Manual |

### Why Goroutines are Better

**1. Memory Efficiency**
```go
// Can create millions of goroutines
for i := 0; i < 1000000; i++ {
    go func(n int) {
        time.Sleep(time.Hour)
    }(i)
}
// Total memory: ~2GB (vs 1-2TB for threads)
```

**2. Fast Context Switching**
```go
// Goroutine switching is ~10x faster than thread switching
```

**3. Automatic Stack Management**
```go
func recursive(n int) {
    if n == 0 {
        return
    }
    var buffer [1024]byte
    recursive(n - 1)
}

go recursive(10000) // Stack grows automatically
```

## How Goroutines Work

### The Go Scheduler (GMP Model)

Go uses an M:N scheduler that multiplexes M goroutines onto N OS threads.

**Components:**
- **G (Goroutine)**: The goroutine itself
- **M (Machine)**: OS thread
- **P (Processor)**: Context for scheduling (GOMAXPROCS)

### Scheduling Algorithm

**1. Work Stealing**
```go
// When P's local queue is empty, it steals from other Ps
P1 (empty) steals from P2 (full)
```

**2. Hand-off**
```go
// When M blocks (syscall), P is handed to another M
M1 blocks on syscall
P detaches from M1
P attaches to M2
```

**3. Preemption**
```go
// Goroutines are preempted at function calls
func cpuBound() {
    for i := 0; i < 1000000000; i++ {
        doSomething() // Preemption check here
    }
}
```

### Setting GOMAXPROCS

```go
import "runtime"

// Get current value
n := runtime.GOMAXPROCS(0)

// Set to number of CPUs
runtime.GOMAXPROCS(runtime.NumCPU())

// Set to specific value
runtime.GOMAXPROCS(4)
```

## Goroutine Lifecycle

### States

```
Created → Runnable → Running → Waiting → Dead
```

### State Transitions

**1. Created → Runnable**
```go
go func() {
    // Goroutine created and placed in run queue
}()
```

**2. Running → Waiting**
```go
ch <- value        // Channel send
value := <-ch      // Channel receive
time.Sleep(d)      // Timer
mu.Lock()          // Mutex
```

**3. Waiting → Runnable**
```go
// Goroutine unblocked when operation completes
```

**4. Running → Dead**
```go
func worker() {
    return // Goroutine terminates
}
```

## Creating Goroutines

### Basic Creation

```go
// Anonymous function
go func() {
    fmt.Println("Hello from goroutine")
}()

// Named function
func sayHello() {
    fmt.Println("Hello")
}
go sayHello()

// Method
type Worker struct{}
func (w Worker) DoWork() {
    fmt.Println("Working")
}
w := Worker{}
go w.DoWork()
```

### Passing Arguments

```go
// Correct: Pass by value
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}

// Wrong: Closure captures loop variable
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // All see final value
    }()
}
```

### Goroutine with Return Values

```go
func compute(n int) <-chan int {
    result := make(chan int)
    go func() {
        time.Sleep(time.Second)
        result <- n * n
    }()
    return result
}

resultCh := compute(5)
result := <-resultCh
fmt.Println(result) // 25
```

## Stack Management

### Dynamic Stack Growth

```go
// Stack starts at 2KB
func recursive(n int) {
    if n == 0 {
        return
    }
    var buffer [1024]byte
    recursive(n - 1)
}

// Stack grows: 2KB → 4KB → 8KB → 16KB
```

### Stack Shrinking

```go
// Stack shrinks during garbage collection
// If using < 25% of stack, shrink by half
```

### Stack Overflow

```go
func infiniteRecursion() {
    infiniteRecursion()
}
// panic: runtime: goroutine stack exceeds limit
```

## Common Patterns

### Worker Pool

```go
func workerPool(jobs <-chan int, results chan<- int, workers int) {
    var wg sync.WaitGroup
    
    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range jobs {
                results <- process(job)
            }
        }(i)
    }
    
    wg.Wait()
    close(results)
}
```

### Fan-Out/Fan-In

```go
func fanOut(input <-chan int, n int) []<-chan int {
    channels := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        channels[i] = worker(input)
    }
    return channels
}

func fanIn(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                out <- v
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}
```

### Pipeline

```go
func pipeline() {
    gen := func() <-chan int {
        out := make(chan int)
        go func() {
            defer close(out)
            for i := 0; i < 10; i++ {
                out <- i
            }
        }()
        return out
    }
    
    square := func(in <-chan int) <-chan int {
        out := make(chan int)
        go func() {
            defer close(out)
            for n := range in {
                out <- n * n
            }
        }()
        return out
    }
    
    numbers := gen()
    squares := square(numbers)
    
    for s := range squares {
        fmt.Println(s)
    }
}
```

## Best Practices

### 1. Always Provide Exit Mechanism

```go
// Good: Context for cancellation
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // Do work
        }
    }
}
```

### 2. Use WaitGroup for Multiple Goroutines

```go
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        process(n)
    }(i)
}
wg.Wait()
```

### 3. Limit Goroutine Creation

```go
// Good: Worker pool
jobs := make(chan Job, 100)
for i := 0; i < 10; i++ {
    go worker(jobs)
}

// Bad: Unlimited goroutines
for _, job := range jobs {
    go process(job)
}
```

## Common Pitfalls

### 1. Goroutine Leaks

```go
// Leak
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch // Blocks forever
        fmt.Println(val)
    }()
}

// Fix
func noLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case val := <-ch:
            fmt.Println(val)
        case <-ctx.Done():
            return
        }
    }()
}
```

### 2. Race Conditions

```go
// Race
var counter int
for i := 0; i < 1000; i++ {
    go func() {
        counter++
    }()
}

// Fix
var counter int64
for i := 0; i < 1000; i++ {
    go func() {
        atomic.AddInt64(&counter, 1)
    }()
}
```

### 3. Closure Variable Capture

```go
// Wrong
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i)
    }()
}

// Correct
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}
```

## Performance Considerations

### Goroutine Creation Cost

```go
// ~1-2μs per goroutine creation
// ~2KB initial memory per goroutine
```

### Context Switching Cost

```go
// Goroutine switching: ~200ns
// Thread switching: ~1-2μs
```

### Memory Overhead

```go
// 1 million goroutines: ~2GB memory
// 1 million threads: 1-2TB memory
```

## Debugging Goroutines

### Runtime Information

```go
import "runtime"

// Number of goroutines
n := runtime.NumGoroutine()

// Stack trace
buf := make([]byte, 1<<16)
runtime.Stack(buf, true)
```

### Using pprof

```go
import _ "net/http/pprof"

func main() {
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    
    // http://localhost:6060/debug/pprof/goroutine
}
```

### Race Detector

```bash
go build -race
go run -race main.go
go test -race
```

---

**Summary**: Goroutines are lightweight, efficient units of concurrency in Go. Understanding their lifecycle, scheduling, and proper usage patterns is crucial for writing efficient concurrent programs.

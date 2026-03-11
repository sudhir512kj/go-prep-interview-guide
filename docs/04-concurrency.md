# Concurrency in Golang

## Introduction to Concurrency

Concurrency is about dealing with multiple things at once. It's the composition of independently executing processes.

### Concurrency vs Parallelism

**Concurrency**: Structure of program - dealing with multiple things at once
**Parallelism**: Execution of program - doing multiple things at once

- Concurrency is about **structure** (how code is organized)
- Parallelism is about **execution** (how code runs)
- Concurrent program may or may not run in parallel
- Parallelism requires multiple CPU cores

**Example:**
- One person juggling multiple tasks (concurrency)
- Multiple people each doing one task (parallelism)

### Why Go's Concurrency Model is Special

1. **Goroutines**: Lightweight threads (2KB initial stack vs 1-2MB for OS threads)
2. **Channels**: Built-in communication mechanism
3. **Select**: Multiplexing channel operations
4. **Simple syntax**: `go` keyword to spawn goroutine
5. **CSP Model**: Communicating Sequential Processes

**Go Proverb**: "Don't communicate by sharing memory; share memory by communicating"

## Goroutines

### Understanding Goroutines

A goroutine is a lightweight thread managed by the Go runtime. Goroutines run in the same address space, so access to shared memory must be synchronized.

**Characteristics:**
- Very cheap: Start with 2KB stack (grows/shrinks dynamically)
- Multiplexed onto OS threads by Go scheduler
- Can have millions of goroutines
- Scheduled cooperatively (not preemptively like OS threads)

**Goroutine Lifecycle:**
1. Created with `go` keyword
2. Scheduled by Go runtime
3. Runs until completion or blocks
4. Garbage collected when done

### Basic Goroutine

```go
func main() {
    go sayHello()
    time.Sleep(time.Second) // Wait for goroutine
}

func sayHello() {
    fmt.Println("Hello from goroutine")
}
```

**Important**: Main function doesn't wait for goroutines. If main exits, all goroutines are terminated.

### Anonymous Goroutine

```go
func main() {
    go func() {
        fmt.Println("Anonymous goroutine")
    }()
    time.Sleep(time.Second)
}
```

### Multiple Goroutines

```go
func main() {
    for i := 0; i < 5; i++ {
        go func(n int) {
            fmt.Println("Goroutine", n)
        }(i) // Pass i as argument to avoid closure issue
    }
    time.Sleep(time.Second)
}
```

**Common Mistake - Closure Issue:**
```go
// Wrong: All goroutines may print same value
for i := 0; i < 5; i++ {
    go func() {
        fmt.Println(i) // Captures loop variable
    }()
}

// Correct: Pass value as parameter
for i := 0; i < 5; i++ {
    go func(n int) {
        fmt.Println(n)
    }(i)
}
```

### Goroutine Scheduling

Go uses M:N scheduling - M goroutines on N OS threads.

**Components:**
- **G (Goroutine)**: Goroutine
- **M (Machine)**: OS thread
- **P (Processor)**: Context for scheduling (GOMAXPROCS)

**Scheduler Behavior:**
- Work stealing: Idle P steals work from other Ps
- Preemption: Long-running goroutines can be preempted
- System calls: M blocks, P finds another M

```go
// Set number of OS threads
runtime.GOMAXPROCS(4) // Use 4 cores

// Get number of goroutines
fmt.Println(runtime.NumGoroutine())
```

## Channels

### Understanding Channels

Channels are typed conduits for communication between goroutines. They provide synchronization and data transfer.

**Philosophy**: "Don't communicate by sharing memory; share memory by communicating"

**Channel Operations:**
- Send: `ch <- value`
- Receive: `value := <-ch`
- Close: `close(ch)`

**Channel States:**
- nil channel: Blocks forever on send/receive
- Open channel: Normal operation
- Closed channel: Receive returns zero value, send panics

### Unbuffered Channel

Unbuffered channels provide synchronization - sender blocks until receiver is ready.

```go
func main() {
    ch := make(chan int) // Unbuffered
    
    go func() {
        ch <- 42 // Blocks until received
    }()
    
    value := <-ch // Blocks until sent
    fmt.Println(value)
}
```

**Use Cases:**
- Synchronization between goroutines
- Guaranteed delivery
- Handoff semantics

### Buffered Channel

Buffered channels have capacity - sender blocks only when buffer is full.

```go
func main() {
    ch := make(chan int, 2) // Buffer size 2
    ch <- 1 // Doesn't block
    ch <- 2 // Doesn't block
    // ch <- 3 // Would block (buffer full)
    
    fmt.Println(<-ch) // 1
    fmt.Println(<-ch) // 2
}
```

**Use Cases:**
- Decoupling sender and receiver
- Rate limiting
- Buffering work items

**Choosing Buffer Size:**
- 0 (unbuffered): Synchronization required
- 1: Latest value semantics
- n: Known capacity (e.g., worker pool size)
- Large: Avoid blocking (use with caution)

### Channel Direction

Restrict channel operations for type safety.

```go
// Send-only channel
func send(ch chan<- int) {
    ch <- 42
    // value := <-ch // Compile error
}

// Receive-only channel
func receive(ch <-chan int) {
    value := <-ch
    // ch <- 42 // Compile error
}

func main() {
    ch := make(chan int)
    go send(ch)
    receive(ch)
}
```

**Benefits:**
- Type safety
- Clear intent
- Prevents misuse

### Closing Channels

Only sender should close channel. Closing indicates no more values will be sent.

```go
func main() {
    ch := make(chan int, 3)
    ch <- 1
    ch <- 2
    ch <- 3
    close(ch)
    
    // Range over closed channel
    for value := range ch {
        fmt.Println(value)
    }
    // Loop exits when channel is closed and drained
}
```

**Rules:**
- Closing nil channel: panic
- Closing closed channel: panic
- Sending to closed channel: panic
- Receiving from closed channel: Returns zero value

### Check if Channel is Closed

```go
func main() {
    ch := make(chan int, 1)
    ch <- 42
    close(ch)
    
    value, ok := <-ch
    if ok {
        fmt.Println("Received:", value) // 42
    }
    
    value, ok = <-ch
    if !ok {
        fmt.Println("Channel closed") // Zero value, false
    }
}
```

### Channel Patterns

**Pipeline Pattern:**
```go
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func main() {
    nums := generator(1, 2, 3, 4)
    squares := square(nums)
    
    for sq := range squares {
        fmt.Println(sq)
    }
}
```

## Select Statement

### Understanding Select

Select lets a goroutine wait on multiple channel operations. It blocks until one case can proceed.

**Characteristics:**
- Chooses randomly if multiple cases ready
- Default case makes it non-blocking
- nil channel cases are ignored

### Basic Select

```go
func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)
    
    go func() {
        time.Sleep(time.Second)
        ch1 <- "one"
    }()
    
    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "two"
    }()
    
    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-ch1:
            fmt.Println("Received", msg1)
        case msg2 := <-ch2:
            fmt.Println("Received", msg2)
        }
    }
}
```

### Select with Default

Non-blocking channel operations.

```go
func main() {
    ch := make(chan int)
    
    select {
    case value := <-ch:
        fmt.Println(value)
    default:
        fmt.Println("No value received")
    }
}
```

**Use Cases:**
- Non-blocking sends/receives
- Polling channels
- Implementing timeouts

### Select with Timeout

```go
func main() {
    ch := make(chan string)
    
    go func() {
        time.Sleep(2 * time.Second)
        ch <- "result"
    }()
    
    select {
    case res := <-ch:
        fmt.Println(res)
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout")
    }
}
```

### Select Patterns

**Fan-in (Merge channels):**
```go
func fanIn(ch1, ch2 <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case v, ok := <-ch1:
                if !ok {
                    ch1 = nil // Disable this case
                    continue
                }
                out <- v
            case v, ok := <-ch2:
                if !ok {
                    ch2 = nil
                    continue
                }
                out <- v
            }
            if ch1 == nil && ch2 == nil {
                break
            }
        }
    }()
    return out
}
```

**Quit Channel:**
```go
func worker(quit <-chan bool) {
    for {
        select {
        case <-quit:
            fmt.Println("Quitting")
            return
        default:
            // Do work
            fmt.Println("Working...")
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    quit := make(chan bool)
    go worker(quit)
    
    time.Sleep(2 * time.Second)
    quit <- true
    time.Sleep(time.Second)
}
```

## WaitGroup

### Understanding WaitGroup

WaitGroup waits for a collection of goroutines to finish. It's a counter-based synchronization primitive.

**Methods:**
- `Add(n)`: Increment counter by n
- `Done()`: Decrement counter by 1
- `Wait()`: Block until counter is 0

**Rules:**
- Add before starting goroutine
- Done in goroutine (usually with defer)
- Wait in main goroutine

### Basic WaitGroup

```go
func main() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            fmt.Println("Goroutine", n)
        }(i)
    }
    
    wg.Wait()
    fmt.Println("All goroutines completed")
}
```

### WaitGroup with Worker Pool

```go
func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        time.Sleep(time.Second)
        results <- job * 2
    }
}

func main() {
    jobs := make(chan int, 10)
    results := make(chan int, 10)
    var wg sync.WaitGroup
    
    // Start workers
    for i := 1; i <= 3; i++ {
        wg.Add(1)
        go worker(i, jobs, results, &wg)
    }
    
    // Send jobs
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs)
    
    // Wait for workers
    go func() {
        wg.Wait()
        close(results)
    }()
    
    // Collect results
    for result := range results {
        fmt.Println("Result:", result)
    }
}
```

## Mutex

### Understanding Mutex

Mutex (mutual exclusion) provides exclusive access to shared resources. Use when channels are not suitable.

**When to use Mutex:**
- Protecting shared state
- Short critical sections
- Caching
- Counters

**When to use Channels:**
- Passing ownership of data
- Distributing work
- Communicating async results

### Basic Mutex

```go
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}

func main() {
    counter := &Counter{}
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter.Increment()
        }()
    }
    
    wg.Wait()
    fmt.Println("Final count:", counter.Value())
}
```

**Best Practices:**
- Keep critical sections small
- Use defer to unlock
- Don't call exported methods while holding lock
- Avoid nested locks (deadlock risk)

### RWMutex

RWMutex allows multiple readers OR one writer.

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

func (c *Cache) Get(key string) (string, bool) {
    c.mu.RLock() // Multiple readers allowed
    defer c.mu.RUnlock()
    value, exists := c.data[key]
    return value, exists
}

func (c *Cache) Set(key, value string) {
    c.mu.Lock() // Exclusive access
    defer c.mu.Unlock()
    c.data[key] = value
}

func main() {
    cache := &Cache{data: make(map[string]string)}
    var wg sync.WaitGroup
    
    // Multiple readers
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            cache.Get("key")
        }(i)
    }
    
    // One writer
    wg.Add(1)
    go func() {
        defer wg.Done()
        cache.Set("key", "value")
    }()
    
    wg.Wait()
}
```

**When to use RWMutex:**
- Read-heavy workloads
- Expensive read operations
- Many concurrent readers

## Once

### Understanding Once

Once ensures a function is executed only once, even with multiple goroutines.

**Use Cases:**
- Singleton initialization
- One-time setup
- Lazy initialization

```go
var once sync.Once
var instance *Singleton

type Singleton struct {
    data string
}

func GetInstance() *Singleton {
    once.Do(func() {
        fmt.Println("Initializing singleton")
        instance = &Singleton{data: "initialized"}
    })
    return instance
}

func main() {
    var wg sync.WaitGroup
    
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            s := GetInstance()
            fmt.Println(s.data)
        }()
    }
    
    wg.Wait()
    // "Initializing singleton" printed only once
}
```

## Context

### Understanding Context

Context carries deadlines, cancellation signals, and request-scoped values across API boundaries and goroutines.

**Use Cases:**
- Request cancellation
- Timeouts
- Deadlines
- Request-scoped values (user ID, trace ID)

**Context Rules:**
- Pass context as first parameter
- Don't store contexts in structs
- Pass nil only if unsure (use context.TODO())
- Context values for request-scoped data only

### Context with Cancel

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    go func() {
        for {
            select {
            case <-ctx.Done():
                fmt.Println("Goroutine cancelled:", ctx.Err())
                return
            default:
                fmt.Println("Working...")
                time.Sleep(500 * time.Millisecond)
            }
        }
    }()
    
    time.Sleep(2 * time.Second)
    cancel() // Cancel context
    time.Sleep(time.Second)
}
```

### Context with Timeout

```go
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel() // Always call cancel to release resources
    
    select {
    case <-time.After(3 * time.Second):
        fmt.Println("Operation completed")
    case <-ctx.Done():
        fmt.Println("Timeout:", ctx.Err())
    }
}
```

### Context with Deadline

```go
func main() {
    deadline := time.Now().Add(2 * time.Second)
    ctx, cancel := context.WithDeadline(context.Background(), deadline)
    defer cancel()
    
    select {
    case <-time.After(3 * time.Second):
        fmt.Println("Operation completed")
    case <-ctx.Done():
        fmt.Println("Deadline exceeded:", ctx.Err())
    }
}
```

### Context with Value

```go
type key string

const userIDKey key = "userID"

func main() {
    ctx := context.WithValue(context.Background(), userIDKey, 123)
    processRequest(ctx)
}

func processRequest(ctx context.Context) {
    userID := ctx.Value(userIDKey)
    fmt.Println("Processing request for user:", userID)
    
    // Pass context to other functions
    doWork(ctx)
}

func doWork(ctx context.Context) {
    userID := ctx.Value(userIDKey)
    fmt.Println("Doing work for user:", userID)
}
```

**Context Value Best Practices:**
- Use for request-scoped data only
- Use typed keys (not string)
- Don't use for optional parameters
- Document what values are expected

## Atomic Operations

### Understanding Atomic Operations

Atomic operations provide lock-free synchronization for simple operations.

**When to use:**
- Simple counters
- Flags
- Performance-critical code
- Lock-free algorithms

**Supported Types:**
- int32, int64
- uint32, uint64
- uintptr
- unsafe.Pointer

```go
import "sync/atomic"

func main() {
    var counter int64
    var wg sync.WaitGroup
    
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            atomic.AddInt64(&counter, 1)
        }()
    }
    
    wg.Wait()
    fmt.Println("Counter:", atomic.LoadInt64(&counter))
}
```

**Atomic Operations:**
```go
var value int64

// Add
atomic.AddInt64(&value, 1)

// Load
v := atomic.LoadInt64(&value)

// Store
atomic.StoreInt64(&value, 42)

// Swap
old := atomic.SwapInt64(&value, 100)

// Compare and Swap
swapped := atomic.CompareAndSwapInt64(&value, 100, 200)
```

## Common Patterns

### Fan-Out, Fan-In

Distribute work to multiple goroutines (fan-out), then combine results (fan-in).

```go
func producer(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func merge(channels ...<-chan int) <-chan int {
    out := make(chan int)
    var wg sync.WaitGroup
    
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for n := range c {
                out <- n
            }
        }(ch)
    }
    
    go func() {
        wg.Wait()
        close(out)
    }()
    
    return out
}

func main() {
    in := producer(1, 2, 3, 4, 5, 6, 7, 8)
    
    // Fan-out: distribute to multiple workers
    c1 := square(in)
    c2 := square(in)
    c3 := square(in)
    
    // Fan-in: merge results
    for n := range merge(c1, c2, c3) {
        fmt.Println(n)
    }
}
```

### Pipeline

Chain stages where each stage transforms data.

```go
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

func square(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            out <- n * n
        }
    }()
    return out
}

func filter(in <-chan int, predicate func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in {
            if predicate(n) {
                out <- n
            }
        }
    }()
    return out
}

func main() {
    nums := generator(1, 2, 3, 4, 5)
    squared := square(nums)
    filtered := filter(squared, func(n int) bool { return n > 10 })
    
    for n := range filtered {
        fmt.Println(n)
    }
}
```

### Worker Pool

Fixed number of workers processing jobs from a queue.

```go
type Job struct {
    ID     int
    Data   string
}

type Result struct {
    Job    Job
    Output string
}

func worker(id int, jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job.ID)
        time.Sleep(time.Second)
        results <- Result{
            Job:    job,
            Output: fmt.Sprintf("Processed: %s", job.Data),
        }
    }
}

func main() {
    numWorkers := 3
    numJobs := 5
    
    jobs := make(chan Job, numJobs)
    results := make(chan Result, numJobs)
    
    // Start workers
    for w := 1; w <= numWorkers; w++ {
        go worker(w, jobs, results)
    }
    
    // Send jobs
    for j := 1; j <= numJobs; j++ {
        jobs <- Job{ID: j, Data: fmt.Sprintf("data-%d", j)}
    }
    close(jobs)
    
    // Collect results
    for a := 1; a <= numJobs; a++ {
        result := <-results
        fmt.Println(result.Output)
    }
}
```

### Rate Limiting

Control rate of operations.

```go
func main() {
    requests := make(chan int, 5)
    for i := 1; i <= 5; i++ {
        requests <- i
    }
    close(requests)
    
    // Rate limiter: 1 request per 200ms
    limiter := time.Tick(200 * time.Millisecond)
    
    for req := range requests {
        <-limiter // Wait for rate limiter
        fmt.Println("Request", req, time.Now())
    }
}

// Bursty rate limiter
func burstRateLimiter() {
    requests := make(chan int, 5)
    for i := 1; i <= 5; i++ {
        requests <- i
    }
    close(requests)
    
    // Allow bursts of 3
    burstyLimiter := make(chan time.Time, 3)
    for i := 0; i < 3; i++ {
        burstyLimiter <- time.Now()
    }
    
    // Refill 1 per 200ms
    go func() {
        for t := range time.Tick(200 * time.Millisecond) {
            burstyLimiter <- t
        }
    }()
    
    for req := range requests {
        <-burstyLimiter
        fmt.Println("Request", req, time.Now())
    }
}
```

### Timeout Pattern

```go
func doWork(done chan bool) {
    time.Sleep(2 * time.Second)
    done <- true
}

func main() {
    done := make(chan bool)
    go doWork(done)
    
    select {
    case <-done:
        fmt.Println("Work completed")
    case <-time.After(1 * time.Second):
        fmt.Println("Timeout")
    }
}
```

## Concurrency Pitfalls

### Race Conditions

```go
// Race condition
var counter int

func increment() {
    counter++ // Not atomic: read, increment, write
}

// Fix with mutex
var (
    counter int
    mu      sync.Mutex
)

func increment() {
    mu.Lock()
    counter++
    mu.Unlock()
}

// Or use atomic
var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)
}
```

### Goroutine Leaks

```go
// Leak: goroutine never exits
func leak() {
    ch := make(chan int)
    go func() {
        val := <-ch // Blocks forever
        fmt.Println(val)
    }()
    // ch never receives value
}

// Fix: use context for cancellation
func noLeak() {
    ch := make(chan int)
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()
    
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

### Deadlock

```go
// Deadlock: all goroutines blocked
func deadlock() {
    ch := make(chan int)
    ch <- 1 // Blocks forever (no receiver)
    fmt.Println(<-ch)
}

// Fix: use goroutine or buffered channel
func noDeadlock() {
    ch := make(chan int, 1) // Buffered
    ch <- 1
    fmt.Println(<-ch)
}
```

## Best Practices

1. **Start goroutines when you know they'll stop**
2. **Use channels for ownership transfer**
3. **Use mutexes for protecting state**
4. **Keep critical sections small**
5. **Use context for cancellation**
6. **Avoid goroutine leaks**
7. **Test with race detector**: `go test -race`
8. **Profile concurrent code**: `go tool pprof`
9. **Document synchronization requirements**
10. **Prefer simple synchronization**

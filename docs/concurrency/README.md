# Go Concurrency - Complete Reference

## Table of Contents

### Core Concepts
1. [Goroutines Deep Dive](./01-goroutines-deep-dive.md) ✅
2. [Channels Deep Dive](./02-channels-deep-dive.md) ✅
3. [Select Statement](./03-select-statement.md)
4. [Sync Package](./04-sync-package.md)
5. [Context Package](./05-context-package.md)
6. [Atomic Operations](./06-atomic-operations.md)

### Synchronization Primitives
7. [Mutex and RWMutex](./07-mutex-rwmutex.md)
8. [WaitGroup](./08-waitgroup.md)
9. [Once](./09-once.md)
10. [Cond](./10-cond.md)
11. [Pool](./11-pool.md)
12. [Map (sync.Map)](./12-sync-map.md)

### Advanced Topics
13. [Memory Model](./13-memory-model.md)
14. [Happens-Before](./14-happens-before.md)
15. [Race Detector](./15-race-detector.md)
16. [Scheduler Internals](./16-scheduler-internals.md)
17. [Channel Internals](./17-channel-internals.md)
18. [Goroutine Stack](./18-goroutine-stack.md)

### Patterns
19. [Pipeline Pattern](./19-pipeline-pattern.md)
20. [Fan-Out/Fan-In](./20-fan-out-fan-in.md)
21. [Worker Pool](./21-worker-pool.md)
22. [Rate Limiting](./22-rate-limiting.md)
23. [Circuit Breaker](./23-circuit-breaker.md)
24. [Semaphore](./24-semaphore.md)
25. [Barrier](./25-barrier.md)

### Common Problems
26. [Deadlock](./26-deadlock.md)
27. [Race Conditions](./27-race-conditions.md)
28. [Goroutine Leaks](./28-goroutine-leaks.md)
29. [Channel Misuse](./29-channel-misuse.md)
30. [Context Cancellation](./30-context-cancellation.md)

### Best Practices
31. [Concurrency Best Practices](./31-best-practices.md)
32. [Error Handling in Concurrent Code](./32-error-handling.md)
33. [Testing Concurrent Code](./33-testing-concurrent.md)
34. [Debugging Concurrent Programs](./34-debugging.md)
35. [Performance Tuning](./35-performance-tuning.md)

---

## Quick Reference

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

### Sync Primitives
```go
var mu sync.Mutex
var wg sync.WaitGroup
var once sync.Once
var pool sync.Pool
var m sync.Map
```

### Context
```go
ctx, cancel := context.WithCancel(context.Background())
ctx, cancel := context.WithTimeout(ctx, time.Second)
ctx, cancel := context.WithDeadline(ctx, deadline)
ctx = context.WithValue(ctx, key, value)
```

### Atomic
```go
atomic.AddInt64(&counter, 1)
atomic.LoadInt64(&counter)
atomic.StoreInt64(&counter, value)
atomic.CompareAndSwapInt64(&counter, old, new)
```

## Terminology

### A-C
- **Atomic Operations**: Operations that complete without interruption
- **Blocking**: Operation waits until condition met
- **Buffered Channel**: Channel with capacity > 0
- **Channel**: Communication mechanism between goroutines
- **Concurrency**: Composition of independently executing processes
- **Context**: Carries deadlines, cancellation signals, and values
- **Critical Section**: Code that accesses shared resources

### D-G
- **Deadlock**: All goroutines blocked waiting for each other
- **Fan-In**: Multiplexing multiple channels into one
- **Fan-Out**: Distributing work to multiple goroutines
- **Goroutine**: Lightweight thread managed by Go runtime
- **GOMAXPROCS**: Number of OS threads for goroutines

### H-M
- **Happens-Before**: Memory ordering guarantee
- **Livelock**: Goroutines actively changing state but making no progress
- **Memory Model**: Rules for memory visibility across goroutines
- **Mutex**: Mutual exclusion lock
- **M:N Scheduling**: M goroutines on N OS threads

### P-S
- **Parallelism**: Simultaneous execution on multiple CPUs
- **Pipeline**: Series of stages connected by channels
- **Preemption**: Forcibly stopping goroutine execution
- **Race Condition**: Outcome depends on timing of events
- **RWMutex**: Read-write mutex allowing multiple readers
- **Scheduler**: Go runtime component managing goroutines
- **Select**: Multiplexes channel operations
- **Semaphore**: Limits concurrent access to resource
- **Starvation**: Goroutine unable to gain access to resource
- **Synchronization**: Coordinating goroutine execution

### T-Z
- **Unbuffered Channel**: Channel with zero capacity
- **WaitGroup**: Waits for collection of goroutines
- **Work Stealing**: Idle processor steals work from busy processor

## Common Patterns Quick Reference

### Worker Pool
```go
jobs := make(chan Job, 100)
results := make(chan Result, 100)

for i := 0; i < numWorkers; i++ {
    go worker(jobs, results)
}
```

### Pipeline
```go
gen := generator()
squared := square(gen)
filtered := filter(squared)
```

### Fan-Out/Fan-In
```go
channels := fanOut(input, numWorkers)
output := fanIn(channels...)
```

### Rate Limiting
```go
limiter := time.Tick(time.Second / rate)
for range limiter {
    // Do work
}
```

### Timeout
```go
select {
case result := <-ch:
    return result
case <-time.After(timeout):
    return ErrTimeout
}
```

### Context Cancellation
```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

select {
case <-ctx.Done():
    return ctx.Err()
case result := <-ch:
    return result
}
```

## Learning Path

### Beginner
1. Goroutines basics
2. Channels (unbuffered/buffered)
3. WaitGroup
4. Basic patterns (worker pool)

### Intermediate
5. Select statement
6. Context package
7. Mutex and RWMutex
8. Common patterns (pipeline, fan-out/fan-in)

### Advanced
9. Memory model
10. Scheduler internals
11. Atomic operations
12. Advanced patterns (circuit breaker, semaphore)
13. Performance tuning
14. Debugging and testing

## Resources

### Official Documentation
- [Effective Go - Concurrency](https://go.dev/doc/effective_go#concurrency)
- [Go Memory Model](https://go.dev/ref/mem)
- [Go Blog - Concurrency](https://go.dev/blog/pipelines)

### Books
- "Concurrency in Go" by Katherine Cox-Buday
- "The Go Programming Language" by Donovan & Kernighan

### Tools
- Race Detector: `go run -race`
- pprof: Goroutine profiling
- Trace: Execution tracer

---

**Note**: Documents marked with ✅ are complete. Others are planned for future updates.

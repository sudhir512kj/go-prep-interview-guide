# Channels - Deep Dive

## Table of Contents
- [Introduction](#introduction)
- [What are Channels?](#what-are-channels)
- [Channel Types](#channel-types)
- [Channel Operations](#channel-operations)
- [Channel Internals](#channel-internals)
- [Buffered vs Unbuffered](#buffered-vs-unbuffered)
- [Channel Direction](#channel-direction)
- [Closing Channels](#closing-channels)
- [Select Statement](#select-statement)
- [Channel Patterns](#channel-patterns)
- [Best Practices](#best-practices)
- [Common Pitfalls](#common-pitfalls)
- [Performance](#performance)

---

## Introduction

Channels are Go's way of implementing communication between goroutines. They provide a type-safe way to send and receive values between concurrent goroutines.

**Philosophy**: "Don't communicate by sharing memory; share memory by communicating"

## What are Channels?

A channel is a typed conduit through which you can send and receive values with the channel operator `<-`.

```go
ch := make(chan int)    // Create channel
ch <- 42                // Send to channel
value := <-ch           // Receive from channel
```

### Channel Characteristics

- **Type-safe**: Can only send/receive specific type
- **Thread-safe**: Safe for concurrent access
- **FIFO**: First-in, first-out ordering
- **Blocking**: Operations block until ready
- **Synchronization**: Provides happens-before guarantees

## Channel Types

### 1. Unbuffered Channels

```go
ch := make(chan int)

// Sender blocks until receiver ready
go func() {
    ch <- 42  // Blocks here
}()

value := <-ch  // Unblocks sender
```

**Characteristics:**
- Zero capacity
- Synchronous communication
- Sender and receiver must be ready simultaneously
- Provides strong synchronization

### 2. Buffered Channels

```go
ch := make(chan int, 3)  // Buffer size 3

ch <- 1  // Doesn't block
ch <- 2  // Doesn't block
ch <- 3  // Doesn't block
ch <- 4  // Blocks (buffer full)
```

**Characteristics:**
- Has capacity
- Asynchronous up to buffer size
- Sender blocks only when buffer full
- Receiver blocks only when buffer empty

### 3. Nil Channels

```go
var ch chan int  // nil channel

ch <- 42   // Blocks forever
<-ch       // Blocks forever
close(ch)  // Panics
```

**Use Cases:**
- Disabling select cases
- Conditional channel operations

## Channel Operations

### Send Operation

```go
ch <- value

// Blocks until:
// - Receiver is ready (unbuffered)
// - Buffer has space (buffered)
// - Channel is closed (panics)
```

### Receive Operation

```go
value := <-ch

// Blocks until:
// - Sender sends value
// - Channel is closed (returns zero value)

// Check if channel is closed
value, ok := <-ch
if !ok {
    // Channel closed
}
```

### Close Operation

```go
close(ch)

// Effects:
// - No more sends allowed (panic if attempted)
// - Receives return zero value and false
// - Closing closed channel panics
// - Closing nil channel panics
```

## Channel Internals

### Channel Structure

```go
type hchan struct {
    qcount   uint           // Total data in queue
    dataqsiz uint           // Size of circular queue
    buf      unsafe.Pointer // Points to array of dataqsiz elements
    elemsize uint16         // Size of element
    closed   uint32         // Channel closed flag
    elemtype *_type         // Element type
    sendx    uint           // Send index
    recvx    uint           // Receive index
    recvq    waitq          // List of recv waiters
    sendq    waitq          // List of send waiters
    lock     mutex          // Protects all fields
}
```

### How Channels Work

**Unbuffered Channel:**
```
Sender goroutine          Receiver goroutine
      │                          │
      ├──────── value ──────────>│
      │                          │
   (blocks)                  (blocks)
      │                          │
      └────── handshake ─────────┘
```

**Buffered Channel:**
```
Sender                Buffer              Receiver
  │                    ┌───┐                 │
  ├─── value ────────> │ 1 │                 │
  │                    ├───┤                 │
  ├─── value ────────> │ 2 │                 │
  │                    ├───┤                 │
  ├─── value ────────> │ 3 │ ────> value ───┤
  │                    └───┘                 │
```

### Memory Layout

```go
// Unbuffered channel
ch := make(chan int)
// Memory: ~96 bytes (hchan structure)

// Buffered channel
ch := make(chan int, 100)
// Memory: ~96 bytes + (100 * sizeof(int))
```

## Buffered vs Unbuffered

### Unbuffered Channels

**Advantages:**
- Strong synchronization
- No memory overhead for buffer
- Guaranteed delivery

**Disadvantages:**
- Can cause deadlocks easily
- Requires both parties ready
- Lower throughput

**Use Cases:**
```go
// Synchronization point
done := make(chan bool)
go func() {
    doWork()
    done <- true
}()
<-done  // Wait for completion

// Request-response
request := make(chan Request)
response := make(chan Response)
```

### Buffered Channels

**Advantages:**
- Decouples sender and receiver
- Higher throughput
- Reduces blocking

**Disadvantages:**
- Memory overhead
- Can hide synchronization issues
- Harder to reason about

**Use Cases:**
```go
// Rate limiting
limiter := make(chan struct{}, 10)

// Work queue
jobs := make(chan Job, 100)

// Batch processing
batch := make(chan Item, 50)
```

### Choosing Buffer Size

```go
// No buffer: Synchronization
ch := make(chan int)

// Buffer = 1: Latest value semantics
ch := make(chan int, 1)

// Buffer = N: Known capacity
ch := make(chan int, numWorkers)

// Large buffer: Avoid blocking (use with caution)
ch := make(chan int, 10000)
```

## Channel Direction

### Send-Only Channels

```go
func producer(ch chan<- int) {
    ch <- 42
    // <-ch  // Compile error
}
```

### Receive-Only Channels

```go
func consumer(ch <-chan int) {
    value := <-ch
    // ch <- 42  // Compile error
}
```

### Bidirectional Channels

```go
func worker(ch chan int) {
    ch <- 42
    value := <-ch
}
```

### Conversion

```go
ch := make(chan int)

// Implicit conversion
producer(ch)  // chan int → chan<- int
consumer(ch)  // chan int → <-chan int

// Cannot convert back
var sendOnly chan<- int = ch
// var bidir chan int = sendOnly  // Compile error
```

## Closing Channels

### When to Close

```go
// Producer closes channel
func producer(ch chan<- int) {
    defer close(ch)
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

// Consumer ranges over channel
func consumer(ch <-chan int) {
    for value := range ch {
        fmt.Println(value)
    }
}
```

### Checking if Closed

```go
value, ok := <-ch
if !ok {
    // Channel closed
}

// Or use range
for value := range ch {
    // Exits when channel closed
}
```

### Multiple Senders Problem

```go
// Problem: Multiple senders, who closes?
// Solution: Use sync.WaitGroup

func multipleSenders(ch chan int) {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1)
        go func(n int) {
            defer wg.Done()
            ch <- n
        }(i)
    }
    
    go func() {
        wg.Wait()
        close(ch)  // Close after all senders done
    }()
}
```

## Select Statement

### Basic Select

```go
select {
case v := <-ch1:
    fmt.Println("Received from ch1:", v)
case v := <-ch2:
    fmt.Println("Received from ch2:", v)
case ch3 <- value:
    fmt.Println("Sent to ch3")
}
```

### Select with Default

```go
select {
case v := <-ch:
    fmt.Println(v)
default:
    fmt.Println("No value ready")
}
```

### Select with Timeout

```go
select {
case v := <-ch:
    fmt.Println(v)
case <-time.After(time.Second):
    fmt.Println("Timeout")
}
```

### Select Behavior

**Random Selection:**
```go
// If multiple cases ready, one chosen randomly
ch1 := make(chan int, 1)
ch2 := make(chan int, 1)
ch1 <- 1
ch2 <- 2

select {
case v := <-ch1:
    fmt.Println("ch1:", v)
case v := <-ch2:
    fmt.Println("ch2:", v)
}
// Randomly prints either "ch1: 1" or "ch2: 2"
```

**Nil Channel in Select:**
```go
var ch1 chan int  // nil
ch2 := make(chan int)

select {
case <-ch1:  // Never selected (nil channel)
    fmt.Println("ch1")
case <-ch2:
    fmt.Println("ch2")
}
```

## Channel Patterns

### 1. Generator Pattern

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
```

### 2. Pipeline Pattern

```go
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
```

### 3. Fan-Out Pattern

```go
func fanOut(in <-chan int, n int) []<-chan int {
    channels := make([]<-chan int, n)
    for i := 0; i < n; i++ {
        channels[i] = worker(in)
    }
    return channels
}
```

### 4. Fan-In Pattern

```go
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

### 5. Or-Done Pattern

```go
func orDone(done <-chan struct{}, ch <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case <-done:
                return
            case v, ok := <-ch:
                if !ok {
                    return
                }
                select {
                case out <- v:
                case <-done:
                    return
                }
            }
        }
    }()
    return out
}
```

### 6. Tee Pattern

```go
func tee(in <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    
    go func() {
        defer close(out1)
        defer close(out2)
        for v := range in {
            out1, out2 := out1, out2
            for i := 0; i < 2; i++ {
                select {
                case out1 <- v:
                    out1 = nil
                case out2 <- v:
                    out2 = nil
                }
            }
        }
    }()
    
    return out1, out2
}
```

### 7. Bridge Pattern

```go
func bridge(channels <-chan <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for ch := range channels {
            for v := range ch {
                out <- v
            }
        }
    }()
    
    return out
}
```

## Best Practices

### 1. Close Channels from Sender

```go
// Good
func producer(ch chan<- int) {
    defer close(ch)
    for i := 0; i < 10; i++ {
        ch <- i
    }
}

// Bad: Receiver closes
func consumer(ch <-chan int) {
    for v := range ch {
        fmt.Println(v)
    }
    // close(ch)  // Compile error (receive-only)
}
```

### 2. Use Buffered Channels for Known Capacity

```go
// Good: Buffer matches capacity
results := make(chan Result, numWorkers)

// Bad: Unbuffered with many senders
results := make(chan Result)
```

### 3. Use Select for Timeouts

```go
select {
case result := <-ch:
    return result
case <-time.After(timeout):
    return ErrTimeout
}
```

### 4. Use Context for Cancellation

```go
func worker(ctx context.Context, ch <-chan int) {
    for {
        select {
        case <-ctx.Done():
            return
        case v := <-ch:
            process(v)
        }
    }
}
```

## Common Pitfalls

### 1. Sending on Closed Channel

```go
ch := make(chan int)
close(ch)
ch <- 42  // panic: send on closed channel
```

### 2. Closing Closed Channel

```go
ch := make(chan int)
close(ch)
close(ch)  // panic: close of closed channel
```

### 3. Deadlock

```go
// Deadlock: No receiver
ch := make(chan int)
ch <- 42  // Blocks forever

// Fix: Use goroutine or buffered channel
go func() { ch <- 42 }()
// or
ch := make(chan int, 1)
ch <- 42
```

### 4. Goroutine Leak

```go
// Leak: Goroutine waits forever
func leak() {
    ch := make(chan int)
    go func() {
        <-ch  // Blocks forever
    }()
}

// Fix: Use context
func noLeak(ctx context.Context) {
    ch := make(chan int)
    go func() {
        select {
        case <-ch:
        case <-ctx.Done():
            return
        }
    }()
}
```

### 5. Range on Nil Channel

```go
var ch chan int
for v := range ch {  // Blocks forever
    fmt.Println(v)
}
```

## Performance

### Channel Operation Costs

```go
// Send/Receive: ~100-200ns
// Select with 2 cases: ~200-300ns
// Select with 10 cases: ~500-600ns
```

### Memory Overhead

```go
// Unbuffered: ~96 bytes
// Buffered: ~96 bytes + (capacity * element_size)

ch := make(chan int, 1000)
// Memory: 96 + (1000 * 8) = 8096 bytes
```

### Optimization Tips

```go
// 1. Use buffered channels to reduce blocking
ch := make(chan int, 100)

// 2. Reuse channels when possible
var chPool = sync.Pool{
    New: func() interface{} {
        return make(chan int, 10)
    },
}

// 3. Close channels to free resources
defer close(ch)

// 4. Use select default for non-blocking
select {
case ch <- value:
default:
    // Don't block
}
```

---

**Summary**: Channels are Go's primary mechanism for goroutine communication. Understanding their behavior, patterns, and best practices is essential for writing correct concurrent programs.

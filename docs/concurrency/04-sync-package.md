# Sync Package - Deep Dive

## Table of Contents
- [Introduction](#introduction)
- [Mutex](#mutex)
- [RWMutex](#rwmutex)
- [WaitGroup](#waitgroup)
- [Once](#once)
- [Cond](#cond)
- [Pool](#pool)
- [Map](#map)
- [Best Practices](#best-practices)
- [Performance](#performance)

---

## Introduction

The `sync` package provides basic synchronization primitives such as mutual exclusion locks. These are intended for low-level library routines. Higher-level synchronization is better done via channels and communication.

```go
import "sync"
```

## Mutex

### What is Mutex?

Mutex (mutual exclusion) provides exclusive access to shared resources.

```go
type Mutex struct {
    // contains filtered or unexported fields
}
```

### Basic Usage

```go
var (
    counter int
    mu      sync.Mutex
)

func increment() {
    mu.Lock()
    defer mu.Unlock()
    counter++
}
```

### Methods

```go
func (m *Mutex) Lock()      // Acquire lock
func (m *Mutex) Unlock()    // Release lock
func (m *Mutex) TryLock() bool  // Try to acquire (Go 1.18+)
```

### Mutex States

```
Unlocked → Locked → Unlocked
```

### Internal Implementation

```go
type Mutex struct {
    state int32   // Lock state
    sema  uint32  // Semaphore for blocking
}

// State bits:
// bit 0: locked (1) or unlocked (0)
// bit 1: woken
// bit 2: starving mode
// bits 3+: waiter count
```

### Mutex Modes

**Normal Mode:**
- Waiters queue in FIFO
- Woken goroutine competes with new arrivals
- New arrivals have advantage (already on CPU)

**Starvation Mode:**
- Activated if waiter waits > 1ms
- Lock handed directly to front of queue
- Prevents tail latency

### Example: Thread-Safe Counter

```go
type Counter struct {
    mu    sync.Mutex
    value int
}

func (c *Counter) Inc() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

func (c *Counter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.value
}
```

### Common Mistakes

```go
// Wrong: Copying mutex
func (c Counter) Inc() {  // Value receiver copies mutex
    c.mu.Lock()
    defer c.mu.Unlock()
    c.value++
}

// Wrong: Not unlocking
func bad() {
    mu.Lock()
    if condition {
        return  // Forgot to unlock!
    }
    mu.Unlock()
}

// Wrong: Unlocking unlocked mutex
mu.Unlock()  // Panic if not locked
```

## RWMutex

### What is RWMutex?

RWMutex allows multiple readers OR one writer.

```go
type RWMutex struct {
    // contains filtered or unexported fields
}
```

### Basic Usage

```go
var (
    data map[string]int
    mu   sync.RWMutex
)

func read(key string) int {
    mu.RLock()
    defer mu.RUnlock()
    return data[key]
}

func write(key string, value int) {
    mu.Lock()
    defer mu.Unlock()
    data[key] = value
}
```

### Methods

```go
func (rw *RWMutex) Lock()        // Write lock
func (rw *RWMutex) Unlock()      // Write unlock
func (rw *RWMutex) RLock()       // Read lock
func (rw *RWMutex) RUnlock()     // Read unlock
func (rw *RWMutex) TryLock() bool    // Try write lock
func (rw *RWMutex) TryRLock() bool   // Try read lock
```

### RWMutex States

```
No locks → Multiple readers → No locks
No locks → One writer → No locks
```

### When to Use RWMutex

```go
// Use RWMutex when:
// - Read operations >> Write operations
// - Read operations are expensive
// - Multiple concurrent readers needed

// Example: Cache
type Cache struct {
    mu   sync.RWMutex
    data map[string]interface{}
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    val, ok := c.data[key]
    return val, ok
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.data[key] = value
}
```

### Performance Comparison

```go
// Benchmark results (approximate):
// Mutex read:    100ns
// RWMutex read:  50ns (with no writers)
// RWMutex write: 150ns
```

## WaitGroup

### What is WaitGroup?

WaitGroup waits for a collection of goroutines to finish.

```go
type WaitGroup struct {
    // contains filtered or unexported fields
}
```

### Basic Usage

```go
var wg sync.WaitGroup

for i := 0; i < 5; i++ {
    wg.Add(1)
    go func(n int) {
        defer wg.Done()
        process(n)
    }(i)
}

wg.Wait()  // Wait for all goroutines
```

### Methods

```go
func (wg *WaitGroup) Add(delta int)  // Add to counter
func (wg *WaitGroup) Done()          // Decrement counter
func (wg *WaitGroup) Wait()          // Block until counter is 0
```

### Internal Implementation

```go
type WaitGroup struct {
    noCopy noCopy
    state1 [3]uint32  // state and semaphore
}

// state1[0]: counter (high 32 bits) + waiter count (low 32 bits)
// state1[1]: semaphore
```

### Best Practices

```go
// Good: Add before goroutine
wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()

// Bad: Add inside goroutine (race condition)
go func() {
    wg.Add(1)  // May execute after Wait()
    defer wg.Done()
    work()
}()

// Good: Add in batch
wg.Add(n)
for i := 0; i < n; i++ {
    go worker()
}

// Bad: Add one at a time in loop
for i := 0; i < n; i++ {
    wg.Add(1)
    go worker()
}
```

### Example: Parallel Processing

```go
func processItems(items []Item) []Result {
    results := make([]Result, len(items))
    var wg sync.WaitGroup
    
    for i, item := range items {
        wg.Add(1)
        go func(idx int, it Item) {
            defer wg.Done()
            results[idx] = process(it)
        }(i, item)
    }
    
    wg.Wait()
    return results
}
```

## Once

### What is Once?

Once ensures a function is executed only once, even with multiple goroutines.

```go
type Once struct {
    // contains filtered or unexported fields
}
```

### Basic Usage

```go
var once sync.Once

func initialize() {
    fmt.Println("Initializing...")
}

func main() {
    for i := 0; i < 10; i++ {
        go func() {
            once.Do(initialize)  // Only executes once
        }()
    }
}
```

### Methods

```go
func (o *Once) Do(f func())  // Execute f only once
```

### Internal Implementation

```go
type Once struct {
    done uint32  // Atomic flag
    m    Mutex   // Mutex for first call
}

func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}
```

### Use Cases

**1. Singleton Pattern**
```go
var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{}
    })
    return instance
}
```

**2. One-Time Initialization**
```go
var (
    config *Config
    once   sync.Once
)

func loadConfig() {
    once.Do(func() {
        config = parseConfigFile()
    })
}
```

**3. Lazy Initialization**
```go
type Resource struct {
    once sync.Once
    conn *Connection
}

func (r *Resource) GetConnection() *Connection {
    r.once.Do(func() {
        r.conn = openConnection()
    })
    return r.conn
}
```

## Cond

### What is Cond?

Cond implements a condition variable for waiting/signaling between goroutines.

```go
type Cond struct {
    L Locker  // Held during Wait
    // contains filtered or unexported fields
}
```

### Basic Usage

```go
var (
    mu    sync.Mutex
    cond  = sync.NewCond(&mu)
    ready bool
)

// Waiter
go func() {
    cond.L.Lock()
    for !ready {
        cond.Wait()  // Releases lock, waits, reacquires lock
    }
    // Do work
    cond.L.Unlock()
}()

// Signaler
cond.L.Lock()
ready = true
cond.Signal()  // or cond.Broadcast()
cond.L.Unlock()
```

### Methods

```go
func NewCond(l Locker) *Cond
func (c *Cond) Wait()       // Wait for signal
func (c *Cond) Signal()     // Wake one waiter
func (c *Cond) Broadcast()  // Wake all waiters
```

### Example: Producer-Consumer

```go
type Queue struct {
    mu    sync.Mutex
    cond  *sync.Cond
    items []int
}

func NewQueue() *Queue {
    q := &Queue{}
    q.cond = sync.NewCond(&q.mu)
    return q
}

func (q *Queue) Enqueue(item int) {
    q.mu.Lock()
    defer q.mu.Unlock()
    q.items = append(q.items, item)
    q.cond.Signal()
}

func (q *Queue) Dequeue() int {
    q.mu.Lock()
    defer q.mu.Unlock()
    for len(q.items) == 0 {
        q.cond.Wait()
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item
}
```

## Pool

### What is Pool?

Pool is a set of temporary objects that can be individually saved and retrieved.

```go
type Pool struct {
    New func() interface{}  // Function to create new objects
    // contains filtered or unexported fields
}
```

### Basic Usage

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func process() {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer bufferPool.Put(buf)
    buf.Reset()
    
    // Use buffer
}
```

### Methods

```go
func (p *Pool) Get() interface{}      // Get from pool
func (p *Pool) Put(x interface{})     // Return to pool
```

### When to Use Pool

```go
// Use Pool for:
// - Frequently allocated/deallocated objects
// - Expensive to create objects
// - Reducing GC pressure

// Example: HTTP request buffers
var requestPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}
```

### Important Notes

```go
// Pool is cleared on GC
// Don't rely on objects staying in pool
// Don't pool objects with important state
// Pool is safe for concurrent use
```

## Map

### What is sync.Map?

sync.Map is a concurrent map safe for multiple goroutines.

```go
type Map struct {
    // contains filtered or unexported fields
}
```

### Basic Usage

```go
var m sync.Map

// Store
m.Store("key", "value")

// Load
value, ok := m.Load("key")

// Delete
m.Delete("key")

// LoadOrStore
actual, loaded := m.LoadOrStore("key", "value")

// Range
m.Range(func(key, value interface{}) bool {
    fmt.Println(key, value)
    return true  // continue iteration
})
```

### Methods

```go
func (m *Map) Store(key, value interface{})
func (m *Map) Load(key interface{}) (value interface{}, ok bool)
func (m *Map) LoadOrStore(key, value interface{}) (actual interface{}, loaded bool)
func (m *Map) LoadAndDelete(key interface{}) (value interface{}, loaded bool)
func (m *Map) Delete(key interface{})
func (m *Map) Range(f func(key, value interface{}) bool)
```

### When to Use sync.Map

```go
// Use sync.Map when:
// 1. Entry written once, read many times
// 2. Multiple goroutines read/write disjoint keys
// 3. Heavy contention on map[interface{}]interface{} with Mutex

// Don't use when:
// - Keys are of known type (use map with Mutex)
// - Frequent updates to same keys
```

### Performance Comparison

```go
// sync.Map vs map+Mutex:
// Read-heavy: sync.Map faster
// Write-heavy: map+Mutex faster
// Mixed: Depends on access pattern
```

## Best Practices

### 1. Use defer with Unlock

```go
mu.Lock()
defer mu.Unlock()
// Safe even with panic or early return
```

### 2. Keep Critical Sections Small

```go
// Good
mu.Lock()
counter++
mu.Unlock()

// Bad
mu.Lock()
defer mu.Unlock()
expensiveOperation()  // Don't hold lock during expensive ops
```

### 3. Avoid Nested Locks

```go
// Bad: Deadlock risk
mu1.Lock()
mu2.Lock()
// ...
mu2.Unlock()
mu1.Unlock()

// Good: Always acquire in same order
```

### 4. Use RWMutex for Read-Heavy Workloads

```go
// If reads >> writes, use RWMutex
var mu sync.RWMutex

func read() {
    mu.RLock()
    defer mu.RUnlock()
    // Read operation
}
```

### 5. Don't Copy Sync Types

```go
// Wrong
func bad(mu sync.Mutex) {  // Copies mutex
    mu.Lock()
    defer mu.Unlock()
}

// Correct
func good(mu *sync.Mutex) {
    mu.Lock()
    defer mu.Unlock()
}
```

## Performance

### Lock Contention

```go
// High contention: Many goroutines competing for lock
// Solutions:
// 1. Reduce critical section size
// 2. Use RWMutex for reads
// 3. Shard locks
// 4. Use lock-free algorithms (atomic)
```

### Sharded Locks

```go
type ShardedMap struct {
    shards []*MapShard
    count  int
}

type MapShard struct {
    mu   sync.RWMutex
    data map[string]interface{}
}

func (sm *ShardedMap) getShard(key string) *MapShard {
    hash := fnv.New32()
    hash.Write([]byte(key))
    return sm.shards[hash.Sum32()%uint32(sm.count)]
}
```

---

**Summary**: The sync package provides essential synchronization primitives. Choose the right primitive for your use case and follow best practices to avoid common pitfalls.

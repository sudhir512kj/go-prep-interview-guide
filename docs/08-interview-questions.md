# Common Golang Interview Questions

## Basic Concepts

### 1. What is Go? What are its key features?
Go is a statically typed, compiled programming language designed at Google. Key features:
- Fast compilation
- Built-in concurrency (goroutines, channels)
- Garbage collection
- Simple syntax
- Strong standard library
- Cross-platform support

### 2. What is the difference between `var` and `:=`?
```go
// var: explicit declaration, can be used at package level
var name string = "John"
var age = 30

// := short declaration, only inside functions
func main() {
    city := "NYC"  // type inferred
}
```

### 3. What are goroutines?
Lightweight threads managed by Go runtime. Much cheaper than OS threads.
```go
go func() {
    fmt.Println("Running in goroutine")
}()
```

### 4. What is a channel?
Channels are typed conduits for communication between goroutines.
```go
ch := make(chan int)
go func() { ch <- 42 }()
value := <-ch
```

### 5. What is the difference between buffered and unbuffered channels?
```go
// Unbuffered: blocks until receiver is ready
ch1 := make(chan int)

// Buffered: blocks only when buffer is full
ch2 := make(chan int, 5)
```

## Intermediate Concepts

### 6. Explain defer, panic, and recover
```go
// defer: executes after function returns
defer fmt.Println("cleanup")

// panic: stops normal execution
panic("something went wrong")

// recover: regains control after panic
defer func() {
    if r := recover(); r != nil {
        fmt.Println("Recovered:", r)
    }
}()
```

### 7. What is the difference between array and slice?
```go
// Array: fixed size
arr := [5]int{1, 2, 3, 4, 5}

// Slice: dynamic size
slice := []int{1, 2, 3}
slice = append(slice, 4)
```

### 8. How does Go handle memory management?
Go uses garbage collection. The GC automatically frees memory that's no longer referenced.

### 9. What is an interface in Go?
A collection of method signatures. Any type that implements all methods satisfies the interface.
```go
type Writer interface {
    Write([]byte) (int, error)
}
```

### 10. What is the empty interface?
```go
var i interface{} // Can hold any type
i = 42
i = "hello"
i = struct{}{}
```

## Advanced Concepts

### 11. Explain the select statement
```go
select {
case msg := <-ch1:
    fmt.Println(msg)
case msg := <-ch2:
    fmt.Println(msg)
case <-time.After(1 * time.Second):
    fmt.Println("timeout")
default:
    fmt.Println("no message")
}
```

### 12. What is a mutex? When would you use it?
Mutex provides mutual exclusion for shared resources.
```go
var mu sync.Mutex
mu.Lock()
// Critical section
mu.Unlock()
```

### 13. What is the difference between Mutex and RWMutex?
```go
// Mutex: exclusive lock
var mu sync.Mutex

// RWMutex: multiple readers OR one writer
var rwmu sync.RWMutex
rwmu.RLock()   // Read lock
rwmu.RUnlock()
rwmu.Lock()    // Write lock
rwmu.Unlock()
```

### 14. What is a WaitGroup?
Waits for a collection of goroutines to finish.
```go
var wg sync.WaitGroup
wg.Add(1)
go func() {
    defer wg.Done()
    // Work
}()
wg.Wait()
```

### 15. Explain context package
Context carries deadlines, cancellation signals, and request-scoped values.
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

select {
case <-ctx.Done():
    return ctx.Err()
}
```

## Coding Problems

### 16. Reverse a string
```go
func reverseString(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}
```

### 17. Check if a string is a palindrome
```go
func isPalindrome(s string) bool {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        if runes[i] != runes[j] {
            return false
        }
    }
    return true
}
```

### 18. Find duplicate in array
```go
func findDuplicate(nums []int) int {
    seen := make(map[int]bool)
    for _, num := range nums {
        if seen[num] {
            return num
        }
        seen[num] = true
    }
    return -1
}
```

### 19. Implement a stack
```go
type Stack struct {
    items []int
}

func (s *Stack) Push(item int) {
    s.items = append(s.items, item)
}

func (s *Stack) Pop() (int, bool) {
    if len(s.items) == 0 {
        return 0, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}
```

### 20. Implement a queue
```go
type Queue struct {
    items []int
}

func (q *Queue) Enqueue(item int) {
    q.items = append(q.items, item)
}

func (q *Queue) Dequeue() (int, bool) {
    if len(q.items) == 0 {
        return 0, false
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item, true
}
```

### 21. Merge two sorted arrays
```go
func mergeSorted(arr1, arr2 []int) []int {
    result := make([]int, 0, len(arr1)+len(arr2))
    i, j := 0, 0
    
    for i < len(arr1) && j < len(arr2) {
        if arr1[i] <= arr2[j] {
            result = append(result, arr1[i])
            i++
        } else {
            result = append(result, arr2[j])
            j++
        }
    }
    
    result = append(result, arr1[i:]...)
    result = append(result, arr2[j:]...)
    return result
}
```

### 22. Find missing number in array
```go
func findMissing(nums []int) int {
    n := len(nums)
    expectedSum := n * (n + 1) / 2
    actualSum := 0
    for _, num := range nums {
        actualSum += num
    }
    return expectedSum - actualSum
}
```

### 23. Two Sum problem
```go
func twoSum(nums []int, target int) []int {
    seen := make(map[int]int)
    for i, num := range nums {
        complement := target - num
        if j, exists := seen[complement]; exists {
            return []int{j, i}
        }
        seen[num] = i
    }
    return nil
}
```

### 24. Valid Parentheses
```go
func isValid(s string) bool {
    stack := []rune{}
    pairs := map[rune]rune{')': '(', '}': '{', ']': '['}
    
    for _, ch := range s {
        if ch == '(' || ch == '{' || ch == '[' {
            stack = append(stack, ch)
        } else {
            if len(stack) == 0 || stack[len(stack)-1] != pairs[ch] {
                return false
            }
            stack = stack[:len(stack)-1]
        }
    }
    return len(stack) == 0
}
```

### 25. Longest substring without repeating characters
```go
func lengthOfLongestSubstring(s string) int {
    charMap := make(map[byte]int)
    left, maxLen := 0, 0
    
    for right := 0; right < len(s); right++ {
        if idx, exists := charMap[s[right]]; exists && idx >= left {
            left = idx + 1
        }
        charMap[s[right]] = right
        if right-left+1 > maxLen {
            maxLen = right - left + 1
        }
    }
    return maxLen
}
```

## Scenario-Based Questions

### 26. How would you implement a rate limiter?
```go
type RateLimiter struct {
    tokens chan struct{}
}

func NewRateLimiter(rate int, interval time.Duration) *RateLimiter {
    rl := &RateLimiter{
        tokens: make(chan struct{}, rate),
    }
    
    // Fill bucket
    for i := 0; i < rate; i++ {
        rl.tokens <- struct{}{}
    }
    
    // Refill tokens
    go func() {
        ticker := time.NewTicker(interval)
        for range ticker.C {
            select {
            case rl.tokens <- struct{}{}:
            default:
            }
        }
    }()
    
    return rl
}

func (rl *RateLimiter) Allow() bool {
    select {
    case <-rl.tokens:
        return true
    default:
        return false
    }
}
```

### 27. How would you implement a worker pool?
```go
func workerPool(numWorkers int, jobs <-chan Job, results chan<- Result) {
    var wg sync.WaitGroup
    
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for job := range jobs {
                results <- processJob(job)
            }
        }(i)
    }
    
    wg.Wait()
    close(results)
}
```

### 28. How would you implement a cache with expiration?
```go
type CacheItem struct {
    Value      interface{}
    Expiration time.Time
}

type Cache struct {
    mu    sync.RWMutex
    items map[string]CacheItem
}

func (c *Cache) Set(key string, value interface{}, ttl time.Duration) {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.items[key] = CacheItem{
        Value:      value,
        Expiration: time.Now().Add(ttl),
    }
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    item, exists := c.items[key]
    if !exists || time.Now().After(item.Expiration) {
        return nil, false
    }
    return item.Value, true
}
```

### 29. How would you handle graceful shutdown?
```go
func main() {
    server := &http.Server{Addr: ":8080"}
    
    go func() {
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()
    
    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    // Graceful shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := server.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }
}
```

### 30. How would you implement a singleton pattern?
```go
type Singleton struct {
    data string
}

var (
    instance *Singleton
    once     sync.Once
)

func GetInstance() *Singleton {
    once.Do(func() {
        instance = &Singleton{data: "initialized"}
    })
    return instance
}
```

## Conceptual Questions

### 31. What is the difference between concurrency and parallelism?
- **Concurrency**: Dealing with multiple things at once (composition of independently executing processes)
- **Parallelism**: Doing multiple things at once (simultaneous execution)

### 32. What is a race condition? How do you prevent it?
Race condition occurs when multiple goroutines access shared data concurrently and at least one modifies it.

Prevention:
- Use mutexes
- Use channels
- Use atomic operations

### 33. What is the difference between `make` and `new`?
```go
// new: allocates memory, returns pointer to zero value
p := new(int) // *int, value is 0

// make: initializes slice, map, or channel
s := make([]int, 5)    // slice
m := make(map[string]int) // map
ch := make(chan int)   // channel
```

### 34. What is method receiver? Difference between value and pointer receiver?
```go
// Value receiver: operates on copy
func (p Person) GetName() string {
    return p.Name
}

// Pointer receiver: operates on original, can modify
func (p *Person) SetName(name string) {
    p.Name = name
}
```

### 35. What is type assertion?
```go
var i interface{} = "hello"

// Type assertion
s := i.(string)

// Safe type assertion
s, ok := i.(string)
if ok {
    fmt.Println(s)
}
```

### 36. What is type switch?
```go
func describe(i interface{}) {
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

### 37. What are Go modules?
Go modules are the dependency management system in Go. They track and manage project dependencies.

```bash
go mod init github.com/user/project
go get github.com/pkg/errors
go mod tidy
```

### 38. What is the init function?
```go
func init() {
    // Runs automatically before main
    // Used for initialization
}
```

### 39. What is a closure?
A function that references variables from outside its body.
```go
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}
```

### 40. What is the difference between `len()` and `cap()` for slices?
```go
s := make([]int, 3, 5)
len(s) // 3 - number of elements
cap(s) // 5 - capacity of underlying array
```


## Deep Dive Questions

### 41. Explain Go's memory model
Go's memory model specifies conditions under which reads of a variable in one goroutine can be guaranteed to observe values produced by writes to the same variable in a different goroutine.

**Happens-Before Relationship:**
- Initialization happens before main.main starts
- Goroutine creation happens before goroutine execution
- Channel send happens before corresponding receive completes
- Channel close happens before receive returns zero value
- Mutex unlock happens before next lock

**Example:**
```go
var a string
var done = make(chan bool)

func setup() {
    a = "hello, world"
    done <- true
}

func main() {
    go setup()
    <-done
    print(a) // Guaranteed to print "hello, world"
}
```

### 42. What is escape analysis?
Escape analysis determines whether a variable can be allocated on the stack or must be allocated on the heap.

**Stack Allocation:**
- Faster (no GC overhead)
- Automatic cleanup
- Limited size

**Heap Allocation:**
- Slower (GC overhead)
- Manual cleanup (GC)
- Unlimited size

```go
// Escapes to heap (returned pointer)
func createUser() *User {
    u := User{Name: "John"}
    return &u // u escapes to heap
}

// Stays on stack
func processUser() {
    u := User{Name: "John"}
    fmt.Println(u.Name) // u stays on stack
}

// Check with: go build -gcflags='-m'
```

### 43. Explain Go's garbage collector
Go uses a concurrent, tri-color mark-and-sweep garbage collector.

**Phases:**
1. **Mark Setup (STW)**: Stop the world, enable write barrier
2. **Marking (Concurrent)**: Mark reachable objects
3. **Mark Termination (STW)**: Disable write barrier
4. **Sweeping (Concurrent)**: Reclaim unmarked objects

**Tuning:**
```go
// Set GC percentage (default 100)
debug.SetGCPercent(50) // Run GC more frequently

// Force GC
runtime.GC()

// Get GC stats
var stats runtime.MemStats
runtime.ReadMemStats(&stats)
fmt.Printf("Alloc: %v MB\n", stats.Alloc/1024/1024)
fmt.Printf("NumGC: %v\n", stats.NumGC)
```

### 44. What are build tags?
Build tags control which files are included in compilation.

```go
// +build linux darwin
// +build !windows

package mypackage

// This file is included only on Linux or Darwin, but not Windows
```

**Usage:**
```bash
go build -tags=integration
go test -tags="integration,e2e"
```

### 45. Explain Go modules and versioning
Go modules are Go's dependency management system using semantic versioning.

**go.mod file:**
```
module github.com/user/project

go 1.21

require (
    github.com/pkg/errors v0.9.1
    github.com/stretchr/testify v1.8.4
)

replace github.com/old/package => github.com/new/package v1.0.0
```

**Semantic Versioning:**
- v1.2.3: Major.Minor.Patch
- v0.x.x: Pre-1.0, no compatibility guarantees
- v2+: Major version in import path

```go
import "github.com/user/project/v2"
```

## Advanced Coding Problems

### 46. Implement LRU Cache

```go
type LRUCache struct {
    capacity int
    cache    map[int]*Node
    head     *Node
    tail     *Node
}

type Node struct {
    key   int
    value int
    prev  *Node
    next  *Node
}

func Constructor(capacity int) LRUCache {
    head := &Node{}
    tail := &Node{}
    head.next = tail
    tail.prev = head
    
    return LRUCache{
        capacity: capacity,
        cache:    make(map[int]*Node),
        head:     head,
        tail:     tail,
    }
}

func (lru *LRUCache) Get(key int) int {
    if node, exists := lru.cache[key]; exists {
        lru.moveToFront(node)
        return node.value
    }
    return -1
}

func (lru *LRUCache) Put(key, value int) {
    if node, exists := lru.cache[key]; exists {
        node.value = value
        lru.moveToFront(node)
        return
    }
    
    node := &Node{key: key, value: value}
    lru.cache[key] = node
    lru.addToFront(node)
    
    if len(lru.cache) > lru.capacity {
        removed := lru.removeLast()
        delete(lru.cache, removed.key)
    }
}

func (lru *LRUCache) moveToFront(node *Node) {
    lru.removeNode(node)
    lru.addToFront(node)
}

func (lru *LRUCache) addToFront(node *Node) {
    node.next = lru.head.next
    node.prev = lru.head
    lru.head.next.prev = node
    lru.head.next = node
}

func (lru *LRUCache) removeNode(node *Node) {
    node.prev.next = node.next
    node.next.prev = node.prev
}

func (lru *LRUCache) removeLast() *Node {
    node := lru.tail.prev
    lru.removeNode(node)
    return node
}
```

### 47. Implement Thread-Safe Counter with Timeout

```go
type Counter struct {
    mu      sync.RWMutex
    value   int
    timeout time.Duration
    timer   *time.Timer
}

func NewCounter(timeout time.Duration) *Counter {
    return &Counter{
        timeout: timeout,
    }
}

func (c *Counter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    c.value++
    
    if c.timer != nil {
        c.timer.Stop()
    }
    
    c.timer = time.AfterFunc(c.timeout, func() {
        c.mu.Lock()
        c.value = 0
        c.mu.Unlock()
    })
}

func (c *Counter) Value() int {
    c.mu.RLock()
    defer c.mu.RUnlock()
    return c.value
}
```

### 48. Implement Concurrent Map with Sharding

```go
type ConcurrentMap struct {
    shards []*MapShard
    count  int
}

type MapShard struct {
    mu   sync.RWMutex
    data map[string]interface{}
}

func NewConcurrentMap(shardCount int) *ConcurrentMap {
    cm := &ConcurrentMap{
        shards: make([]*MapShard, shardCount),
        count:  shardCount,
    }
    
    for i := 0; i < shardCount; i++ {
        cm.shards[i] = &MapShard{
            data: make(map[string]interface{}),
        }
    }
    
    return cm
}

func (cm *ConcurrentMap) getShard(key string) *MapShard {
    hash := fnv.New32()
    hash.Write([]byte(key))
    return cm.shards[hash.Sum32()%uint32(cm.count)]
}

func (cm *ConcurrentMap) Set(key string, value interface{}) {
    shard := cm.getShard(key)
    shard.mu.Lock()
    defer shard.mu.Unlock()
    shard.data[key] = value
}

func (cm *ConcurrentMap) Get(key string) (interface{}, bool) {
    shard := cm.getShard(key)
    shard.mu.RLock()
    defer shard.mu.RUnlock()
    val, ok := shard.data[key]
    return val, ok
}

func (cm *ConcurrentMap) Delete(key string) {
    shard := cm.getShard(key)
    shard.mu.Lock()
    defer shard.mu.Unlock()
    delete(shard.data, key)
}
```

### 49. Implement Semaphore

```go
type Semaphore struct {
    permits chan struct{}
}

func NewSemaphore(maxPermits int) *Semaphore {
    return &Semaphore{
        permits: make(chan struct{}, maxPermits),
    }
}

func (s *Semaphore) Acquire() {
    s.permits <- struct{}{}
}

func (s *Semaphore) TryAcquire() bool {
    select {
    case s.permits <- struct{}{}:
        return true
    default:
        return false
    }
}

func (s *Semaphore) AcquireWithTimeout(timeout time.Duration) bool {
    select {
    case s.permits <- struct{}{}:
        return true
    case <-time.After(timeout):
        return false
    }
}

func (s *Semaphore) Release() {
    <-s.permits
}

// Usage
func main() {
    sem := NewSemaphore(3) // Max 3 concurrent operations
    var wg sync.WaitGroup
    
    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            
            sem.Acquire()
            defer sem.Release()
            
            fmt.Printf("Worker %d working\n", id)
            time.Sleep(time.Second)
        }(i)
    }
    
    wg.Wait()
}
```

### 50. Implement Circuit Breaker

```go
type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

type CircuitBreaker struct {
    mu            sync.Mutex
    state         State
    failures      int
    successes     int
    maxFailures   int
    timeout       time.Duration
    lastFailTime  time.Time
}

func NewCircuitBreaker(maxFailures int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:       StateClosed,
        maxFailures: maxFailures,
        timeout:     timeout,
    }
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    cb.mu.Lock()
    
    // Check if we should transition from Open to HalfOpen
    if cb.state == StateOpen {
        if time.Since(cb.lastFailTime) > cb.timeout {
            cb.state = StateHalfOpen
            cb.successes = 0
        } else {
            cb.mu.Unlock()
            return errors.New("circuit breaker is open")
        }
    }
    
    cb.mu.Unlock()
    
    // Execute function
    err := fn()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failures++
        cb.lastFailTime = time.Now()
        
        if cb.failures >= cb.maxFailures {
            cb.state = StateOpen
        }
        
        return err
    }
    
    // Success
    if cb.state == StateHalfOpen {
        cb.successes++
        if cb.successes >= 2 {
            cb.state = StateClosed
            cb.failures = 0
        }
    } else {
        cb.failures = 0
    }
    
    return nil
}

func (cb *CircuitBreaker) State() State {
    cb.mu.Lock()
    defer cb.mu.Unlock()
    return cb.state
}
```

## System Design Questions

### 51. Design a URL Shortener

**Requirements:**
- Shorten long URLs
- Redirect short URLs to original
- Track click statistics
- Handle high traffic

**Solution:**
```go
type URLShortener struct {
    mu      sync.RWMutex
    urlMap  map[string]string // short -> long
    counter uint64
}

func NewURLShortener() *URLShortener {
    return &URLShortener{
        urlMap: make(map[string]string),
    }
}

func (us *URLShortener) Shorten(longURL string) string {
    us.mu.Lock()
    defer us.mu.Unlock()
    
    // Generate short code
    id := atomic.AddUint64(&us.counter, 1)
    shortCode := us.encode(id)
    
    us.urlMap[shortCode] = longURL
    return shortCode
}

func (us *URLShortener) Expand(shortCode string) (string, bool) {
    us.mu.RLock()
    defer us.mu.RUnlock()
    
    longURL, exists := us.urlMap[shortCode]
    return longURL, exists
}

func (us *URLShortener) encode(num uint64) string {
    const alphabet = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
    base := uint64(len(alphabet))
    
    if num == 0 {
        return string(alphabet[0])
    }
    
    var result []byte
    for num > 0 {
        result = append([]byte{alphabet[num%base]}, result...)
        num /= base
    }
    
    return string(result)
}
```

### 52. Design a Rate Limiter

**Token Bucket Algorithm:**
```go
type TokenBucket struct {
    capacity  int
    tokens    int
    rate      time.Duration
    lastRefill time.Time
    mu        sync.Mutex
}

func NewTokenBucket(capacity int, rate time.Duration) *TokenBucket {
    return &TokenBucket{
        capacity:   capacity,
        tokens:     capacity,
        rate:       rate,
        lastRefill: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    tb.refill()
    
    if tb.tokens > 0 {
        tb.tokens--
        return true
    }
    
    return false
}

func (tb *TokenBucket) refill() {
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill)
    tokensToAdd := int(elapsed / tb.rate)
    
    if tokensToAdd > 0 {
        tb.tokens = min(tb.capacity, tb.tokens+tokensToAdd)
        tb.lastRefill = now
    }
}

func min(a, b int) int {
    if a < b {
        return a
    }
    return b
}
```

## Tricky Questions

### 53. What's the output?

```go
func main() {
    var wg sync.WaitGroup
    for i := 0; i < 3; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            fmt.Println(i)
        }()
    }
    wg.Wait()
}
```

**Answer**: Likely prints "3 3 3" (or some combination) due to closure capturing loop variable. Fix by passing `i` as parameter.

### 54. What's wrong with this code?

```go
type MyStruct struct {
    data map[string]int
}

func (m MyStruct) Set(key string, value int) {
    m.data[key] = value
}
```

**Answer**: Value receiver doesn't modify original. Should use pointer receiver `(m *MyStruct)`.

### 55. What's the difference?

```go
// Version 1
var m map[string]int
m["key"] = 1 // panic

// Version 2
m := make(map[string]int)
m["key"] = 1 // OK
```

**Answer**: Version 1 has nil map. Must initialize with `make()` or literal.

### 56. Explain the output

```go
func main() {
    ch := make(chan int, 1)
    ch <- 1
    close(ch)
    fmt.Println(<-ch)
    fmt.Println(<-ch)
}
```

**Answer**: Prints "1" then "0". Closed channel returns zero value for subsequent receives.

### 57. What happens here?

```go
func main() {
    var i interface{} = (*int)(nil)
    fmt.Println(i == nil)
}
```

**Answer**: Prints "false". Interface with nil value is not nil interface. Interface is nil only if both type and value are nil.

## Performance Questions

### 58. Which is faster and why?

```go
// Option A
for i := 0; i < len(slice); i++ {
    process(slice[i])
}

// Option B
for _, v := range slice {
    process(v)
}
```

**Answer**: Both similar performance. Range is slightly cleaner and avoids bounds checking. For large structs, range copies values, so use index or pointers.

### 59. Optimize this code

```go
func contains(slice []int, target int) bool {
    for _, v := range slice {
        if v == target {
            return true
        }
    }
    return false
}
```

**Optimization**: If slice is sorted, use binary search O(log n) instead of linear O(n). If frequent lookups, use map O(1).

### 60. Memory leak?

```go
func leak() {
    ticker := time.NewTicker(time.Second)
    for {
        select {
        case <-ticker.C:
            // Do something
        }
    }
}
```

**Answer**: Yes! Ticker is never stopped. Should call `ticker.Stop()` or use `defer ticker.Stop()`.

# Context Package - Deep Dive

## Table of Contents
- [Introduction](#introduction)
- [What is Context?](#what-is-context)
- [Context Interface](#context-interface)
- [Context Types](#context-types)
- [Creating Contexts](#creating-contexts)
- [Context Values](#context-values)
- [Cancellation](#cancellation)
- [Timeouts and Deadlines](#timeouts-and-deadlines)
- [Best Practices](#best-practices)
- [Common Patterns](#common-patterns)
- [Common Mistakes](#common-mistakes)

---

## Introduction

The `context` package provides a way to carry deadlines, cancellation signals, and request-scoped values across API boundaries and between goroutines.

```go
import "context"
```

**Key Purposes:**
1. **Cancellation**: Stop operations when no longer needed
2. **Deadlines**: Set time limits for operations
3. **Values**: Pass request-scoped data

## What is Context?

Context is an interface that carries deadlines, cancellation signals, and request-scoped values across API boundaries.

**Philosophy**: "Context should flow through your program"

### Why Context?

```go
// Without context: No way to cancel
func fetchData(url string) ([]byte, error) {
    resp, err := http.Get(url)
    // What if user cancels request?
    // What if it takes too long?
}

// With context: Cancellable and time-limited
func fetchData(ctx context.Context, url string) ([]byte, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    // Automatically cancelled if context cancelled
}
```

## Context Interface

```go
type Context interface {
    // Deadline returns the time when work should be cancelled
    Deadline() (deadline time.Time, ok bool)
    
    // Done returns a channel that's closed when work should be cancelled
    Done() <-chan struct{}
    
    // Err returns why this context was cancelled
    Err() error
    
    // Value returns the value associated with key
    Value(key interface{}) interface{}
}
```

### Context Errors

```go
var Canceled = errors.New("context canceled")
var DeadlineExceeded = errors.New("context deadline exceeded")
```

## Context Types

### 1. Background Context

```go
ctx := context.Background()

// Use for:
// - Main function
// - Initialization
// - Tests
// - Top-level requests
```

### 2. TODO Context

```go
ctx := context.TODO()

// Use when:
// - Unclear which context to use
// - Refactoring code
// - Placeholder during development
```

### 3. Derived Contexts

```go
// All derived contexts created from parent
ctx := context.Background()
childCtx, cancel := context.WithCancel(ctx)
```

## Creating Contexts

### WithCancel

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()  // Always call cancel

go func() {
    select {
    case <-ctx.Done():
        fmt.Println("Cancelled:", ctx.Err())
        return
    case <-time.After(time.Second):
        fmt.Println("Work done")
    }
}()

// Cancel after 500ms
time.Sleep(500 * time.Millisecond)
cancel()
```

**When to Use:**
- Manual cancellation needed
- Parent-child goroutine relationships
- Cleanup on exit

### WithTimeout

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

select {
case <-time.After(3 * time.Second):
    fmt.Println("Work done")
case <-ctx.Done():
    fmt.Println("Timeout:", ctx.Err())
}
```

**When to Use:**
- Operations with time limits
- Network requests
- Database queries
- External API calls

### WithDeadline

```go
deadline := time.Now().Add(2 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

select {
case <-time.After(3 * time.Second):
    fmt.Println("Work done")
case <-ctx.Done():
    fmt.Println("Deadline exceeded:", ctx.Err())
}
```

**When to Use:**
- Specific deadline time
- Coordinating multiple operations
- Request deadlines

### WithValue

```go
type key string

const userIDKey key = "userID"

ctx := context.WithValue(context.Background(), userIDKey, 12345)

// Retrieve value
if userID, ok := ctx.Value(userIDKey).(int); ok {
    fmt.Println("User ID:", userID)
}
```

**When to Use:**
- Request-scoped data
- User authentication
- Request IDs
- Trace IDs

## Context Values

### Storing Values

```go
type contextKey string

const (
    requestIDKey contextKey = "requestID"
    userKey      contextKey = "user"
)

func WithRequestID(ctx context.Context, id string) context.Context {
    return context.WithValue(ctx, requestIDKey, id)
}

func GetRequestID(ctx context.Context) string {
    if id, ok := ctx.Value(requestIDKey).(string); ok {
        return id
    }
    return ""
}
```

### Best Practices for Values

```go
// Good: Use custom type for keys
type key int

const userKey key = 0

// Bad: Use string keys (collision risk)
ctx = context.WithValue(ctx, "user", user)

// Good: Provide accessor functions
func WithUser(ctx context.Context, u *User) context.Context {
    return context.WithValue(ctx, userKey, u)
}

func GetUser(ctx context.Context) (*User, bool) {
    u, ok := ctx.Value(userKey).(*User)
    return u, ok
}
```

### What NOT to Store

```go
// Don't store:
// - Optional parameters (use function arguments)
// - Large data structures
// - Mutable data
// - Business logic data

// Do store:
// - Request IDs
// - User authentication
// - Trace information
// - Deadlines/cancellation
```

## Cancellation

### Manual Cancellation

```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    <-time.After(time.Second)
    cancel()  // Cancel after 1 second
}()

select {
case <-ctx.Done():
    fmt.Println("Cancelled")
}
```

### Propagating Cancellation

```go
func parent(ctx context.Context) {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    
    go child(ctx)  // Child inherits cancellation
    
    // When parent cancels, child is also cancelled
}

func child(ctx context.Context) {
    select {
    case <-ctx.Done():
        fmt.Println("Child cancelled")
        return
    }
}
```

### Checking Cancellation

```go
func worker(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // Do work
            if err := doWork(); err != nil {
                return err
            }
        }
    }
}
```

## Timeouts and Deadlines

### HTTP Request with Timeout

```go
func fetchWithTimeout(url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return nil, err
    }
    defer resp.Body.Close()
    
    return io.ReadAll(resp.Body)
}
```

### Database Query with Timeout

```go
func queryWithTimeout(db *sql.DB, query string) ([]Row, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()
    
    rows, err := db.QueryContext(ctx, query)
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    // Process rows
    return processRows(rows)
}
```

### Cascading Timeouts

```go
func handler(w http.ResponseWriter, r *http.Request) {
    // Parent timeout: 10 seconds
    ctx, cancel := context.WithTimeout(r.Context(), 10*time.Second)
    defer cancel()
    
    // Child timeout: 5 seconds (shorter than parent)
    dbCtx, dbCancel := context.WithTimeout(ctx, 5*time.Second)
    defer dbCancel()
    
    data, err := queryDatabase(dbCtx)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(data)
}
```

## Best Practices

### 1. Pass Context as First Parameter

```go
// Good
func DoWork(ctx context.Context, arg string) error

// Bad
func DoWork(arg string, ctx context.Context) error
```

### 2. Don't Store Context in Structs

```go
// Bad
type Worker struct {
    ctx context.Context
}

// Good
type Worker struct {
    // other fields
}

func (w *Worker) DoWork(ctx context.Context) error
```

### 3. Always Call Cancel

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()  // Always defer cancel

// Even if not explicitly cancelling
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()  // Releases resources
```

### 4. Don't Pass Nil Context

```go
// Bad
DoWork(nil, "arg")

// Good
DoWork(context.Background(), "arg")

// Or if unsure
DoWork(context.TODO(), "arg")
```

### 5. Context Values for Request-Scoped Only

```go
// Good: Request-scoped data
ctx = context.WithValue(ctx, requestIDKey, uuid.New())

// Bad: Optional parameters
ctx = context.WithValue(ctx, "timeout", 5*time.Second)
// Use function parameter instead
```

## Common Patterns

### 1. HTTP Server with Context

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    
    // Add request ID
    requestID := uuid.New().String()
    ctx = context.WithValue(ctx, requestIDKey, requestID)
    
    // Set timeout
    ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
    defer cancel()
    
    // Process request
    result, err := processRequest(ctx, r)
    if err != nil {
        if err == context.DeadlineExceeded {
            http.Error(w, "Request timeout", http.StatusRequestTimeout)
            return
        }
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(result)
}
```

### 2. Worker Pool with Context

```go
func workerPool(ctx context.Context, jobs <-chan Job) {
    var wg sync.WaitGroup
    
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-jobs:
                    if !ok {
                        return
                    }
                    processJob(ctx, job)
                }
            }
        }(i)
    }
    
    wg.Wait()
}
```

### 3. Graceful Shutdown

```go
func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    // Start server
    srv := &http.Server{Addr: ":8080"}
    go func() {
        if err := srv.ListenAndServe(); err != nil {
            log.Println(err)
        }
    }()
    
    // Wait for interrupt
    sigChan := make(chan os.Signal, 1)
    signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)
    <-sigChan
    
    // Cancel context
    cancel()
    
    // Shutdown server
    shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer shutdownCancel()
    
    if err := srv.Shutdown(shutdownCtx); err != nil {
        log.Fatal(err)
    }
}
```

### 4. Pipeline with Context

```go
func pipeline(ctx context.Context) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for i := 0; i < 100; i++ {
            select {
            case <-ctx.Done():
                return
            case out <- i:
            }
        }
    }()
    
    return out
}
```

## Common Mistakes

### 1. Not Checking Context

```go
// Bad: Ignoring context
func worker(ctx context.Context) {
    for {
        doWork()  // Never checks ctx.Done()
    }
}

// Good: Check context regularly
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            doWork()
        }
    }
}
```

### 2. Forgetting to Cancel

```go
// Bad: Resource leak
func bad() {
    ctx, _ := context.WithCancel(context.Background())
    // Forgot to call cancel
}

// Good: Always defer cancel
func good() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
}
```

### 3. Using Context for Optional Parameters

```go
// Bad
ctx = context.WithValue(ctx, "retries", 3)

// Good: Use function parameter
func DoWork(ctx context.Context, retries int) error
```

### 4. Storing Context in Struct

```go
// Bad
type Server struct {
    ctx context.Context
}

// Good
type Server struct {
    // other fields
}

func (s *Server) Handle(ctx context.Context) error
```

### 5. Creating Context in Loop

```go
// Bad: Creates many contexts
for i := 0; i < 1000; i++ {
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()  // Defers accumulate!
    doWork(ctx)
}

// Good: Reuse context or use function
for i := 0; i < 1000; i++ {
    func() {
        ctx, cancel := context.WithTimeout(context.Background(), time.Second)
        defer cancel()
        doWork(ctx)
    }()
}
```

---

**Summary**: Context is essential for managing cancellation, timeouts, and request-scoped values in Go. Always pass context as the first parameter, check for cancellation, and follow best practices to write robust concurrent code.

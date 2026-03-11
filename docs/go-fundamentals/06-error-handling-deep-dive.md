# Error Handling Deep Dive

## Table of Contents
- [Error Fundamentals](#error-fundamentals)
- [Error Interface Internals](#error-interface-internals)
- [Creating Errors](#creating-errors)
- [Custom Error Types](#custom-error-types)
- [Error Wrapping (Go 1.13+)](#error-wrapping-go-113)
- [errors.Is and errors.As](#errorsis-and-errorsas)
- [Sentinel Errors](#sentinel-errors)
- [Error Handling Patterns](#error-handling-patterns)
- [Panic and Recover](#panic-and-recover)
- [Concurrent Error Handling](#concurrent-error-handling)
- [Common Pitfalls](#common-pitfalls)
- [Best Practices](#best-practices)

---

## Error Fundamentals

### The error Interface
```go
// Built-in error interface — just one method
type error interface {
    Error() string
}

// nil error means success
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

result, err := divide(10, 2)
if err != nil {
    log.Fatal(err)
}
fmt.Println(result)  // 5
```

### Error Handling Philosophy
```go
// Go philosophy: errors are values, handle them explicitly
// No exceptions — errors are returned, not thrown

// Idiomatic pattern
result, err := someOperation()
if err != nil {
    return fmt.Errorf("someOperation failed: %w", err)
}
// Use result
```

---

## Error Interface Internals

### How error Works
```go
// error is an interface — two-word struct internally
// (type pointer, value pointer)

var err error = errors.New("something went wrong")

// Internally:
// iface{
//   tab:  *itab for *errors.errorString implementing error
//   data: pointer to errorString{"something went wrong"}
// }
```

### errorString (errors.New)
```go
// From standard library (errors/errors.go)
package errors

type errorString struct {
    s string
}

func (e *errorString) Error() string {
    return e.s
}

func New(text string) error {
    return &errorString{text}  // Returns pointer — each call creates unique error
}

// Why pointer? So each error is unique:
err1 := errors.New("EOF")
err2 := errors.New("EOF")
fmt.Println(err1 == err2)  // false — different pointers
```

### fmt.Errorf
```go
// fmt.Errorf creates formatted error
err := fmt.Errorf("user %d not found", userID)

// With %w — wraps an error (Go 1.13+)
originalErr := errors.New("connection refused")
wrappedErr := fmt.Errorf("database query failed: %w", originalErr)

// wrappedErr.Error() = "database query failed: connection refused"
// errors.Unwrap(wrappedErr) = originalErr
```

---

## Creating Errors

### errors.New
```go
import "errors"

var ErrNotFound = errors.New("not found")
var ErrPermission = errors.New("permission denied")

func findUser(id int) (*User, error) {
    if id <= 0 {
        return nil, ErrNotFound
    }
    // ...
}
```

### fmt.Errorf
```go
func findUser(id int) (*User, error) {
    user, err := db.Query(id)
    if err != nil {
        return nil, fmt.Errorf("findUser(%d): %w", id, err)
    }
    return user, nil
}
```

### errors.New vs fmt.Errorf
```go
// errors.New: static message, comparable with ==
var ErrTimeout = errors.New("timeout")

// fmt.Errorf: dynamic message, not comparable
err := fmt.Errorf("timeout after %v", duration)

// fmt.Errorf with %w: wraps for errors.Is/As
err := fmt.Errorf("operation failed: %w", ErrTimeout)
errors.Is(err, ErrTimeout)  // true
```

---

## Custom Error Types

### Struct-based Error
```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on field %q: %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "must be non-negative"}
    }
    if age > 150 {
        return &ValidationError{Field: "age", Message: "unrealistic value"}
    }
    return nil
}

err := validateAge(-1)
if ve, ok := err.(*ValidationError); ok {
    fmt.Println("Field:", ve.Field)    // age
    fmt.Println("Message:", ve.Message) // must be non-negative
}
```

### Error with Context
```go
type DBError struct {
    Op      string  // Operation: "query", "insert", etc.
    Table   string
    Err     error   // Underlying error
}

func (e *DBError) Error() string {
    return fmt.Sprintf("db %s on %s: %v", e.Op, e.Table, e.Err)
}

func (e *DBError) Unwrap() error {
    return e.Err  // Enables errors.Is/As to unwrap
}

func queryUser(id int) (*User, error) {
    row, err := db.QueryRow("SELECT * FROM users WHERE id = ?", id)
    if err != nil {
        return nil, &DBError{Op: "query", Table: "users", Err: err}
    }
    // ...
}
```

### HTTP Error Type
```go
type HTTPError struct {
    Code    int
    Message string
}

func (e *HTTPError) Error() string {
    return fmt.Sprintf("HTTP %d: %s", e.Code, e.Message)
}

var (
    ErrNotFound   = &HTTPError{Code: 404, Message: "not found"}
    ErrForbidden  = &HTTPError{Code: 403, Message: "forbidden"}
    ErrBadRequest = &HTTPError{Code: 400, Message: "bad request"}
)
```

---

## Error Wrapping (Go 1.13+)

### Wrapping with %w
```go
// %w wraps the error — preserves the chain
func processFile(path string) error {
    data, err := os.ReadFile(path)
    if err != nil {
        return fmt.Errorf("processFile: %w", err)
    }
    // ...
    return nil
}

err := processFile("/nonexistent")
fmt.Println(err)
// processFile: open /nonexistent: no such file or directory

// Unwrap the chain
fmt.Println(errors.Unwrap(err))
// open /nonexistent: no such file or directory
```

### errors.Unwrap
```go
// errors.Unwrap returns the wrapped error (one level)
wrapped := fmt.Errorf("outer: %w", fmt.Errorf("inner: %w", io.EOF))

fmt.Println(errors.Unwrap(wrapped))
// inner: EOF

fmt.Println(errors.Unwrap(errors.Unwrap(wrapped)))
// EOF
```

### Error Chain Visualization
```
err = fmt.Errorf("level3: %w",
        fmt.Errorf("level2: %w",
            fmt.Errorf("level1: %w", io.EOF)))

Error chain:
┌─────────────────────────────────────────────────────┐
│ "level3: level2: level1: EOF"                       │
│ Unwrap() →                                          │
│   ┌─────────────────────────────────────────────┐   │
│   │ "level2: level1: EOF"                       │   │
│   │ Unwrap() →                                  │   │
│   │   ┌─────────────────────────────────────┐   │   │
│   │   │ "level1: EOF"                       │   │   │
│   │   │ Unwrap() →                          │   │   │
│   │   │   ┌─────────────────────────────┐   │   │   │
│   │   │   │ io.EOF                      │   │   │   │
│   │   │   └─────────────────────────────┘   │   │   │
│   │   └─────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

---

## errors.Is and errors.As

### errors.Is — Check Error Identity
```go
// errors.Is traverses the error chain
var ErrNotFound = errors.New("not found")

err := fmt.Errorf("database: %w", fmt.Errorf("query: %w", ErrNotFound))

fmt.Println(errors.Is(err, ErrNotFound))  // true — found in chain
fmt.Println(err == ErrNotFound)           // false — not direct equality

// errors.Is uses == by default
// Custom Is() method for special comparison
type NotFoundError struct{ ID int }
func (e *NotFoundError) Error() string { return fmt.Sprintf("id %d not found", e.ID) }
func (e *NotFoundError) Is(target error) bool {
    t, ok := target.(*NotFoundError)
    return ok && (t.ID == 0 || t.ID == e.ID)  // 0 = wildcard
}

err1 := &NotFoundError{ID: 42}
err2 := &NotFoundError{ID: 0}   // Wildcard
fmt.Println(errors.Is(err1, err2))  // true — custom Is() matches
```

### errors.As — Extract Error Type
```go
// errors.As traverses chain and extracts matching type
var ve *ValidationError

err := fmt.Errorf("handler: %w", &ValidationError{Field: "email", Message: "invalid"})

if errors.As(err, &ve) {
    fmt.Println("Field:", ve.Field)    // email
    fmt.Println("Message:", ve.Message) // invalid
}

// errors.As uses type assertion internally
// Custom As() method for special extraction
type MultiError struct{ Errors []error }
func (m *MultiError) Error() string { /* ... */ }
func (m *MultiError) As(target interface{}) bool {
    if t, ok := target.(**MultiError); ok {
        *t = m
        return true
    }
    return false
}
```

### errors.Is vs errors.As
```go
// errors.Is: checks if error IS a specific value/sentinel
errors.Is(err, io.EOF)          // Is this io.EOF?
errors.Is(err, ErrNotFound)     // Is this ErrNotFound?

// errors.As: checks if error IS a specific type and extracts it
var pathErr *os.PathError
errors.As(err, &pathErr)        // Is there a *os.PathError in chain?
// pathErr now holds the extracted error
```

---

## Sentinel Errors

### Definition
Sentinel errors are package-level error variables used for comparison.

```go
// Standard library sentinels
io.EOF
io.ErrUnexpectedEOF
os.ErrNotExist
os.ErrPermission
sql.ErrNoRows
context.Canceled
context.DeadlineExceeded

// Custom sentinels
var (
    ErrNotFound   = errors.New("not found")
    ErrDuplicate  = errors.New("duplicate entry")
    ErrInvalidID  = errors.New("invalid ID")
)
```

### Using Sentinels
```go
func getUser(id int) (*User, error) {
    user, err := db.Find(id)
    if err == sql.ErrNoRows {
        return nil, ErrNotFound  // Translate to domain error
    }
    if err != nil {
        return nil, fmt.Errorf("getUser: %w", err)
    }
    return user, nil
}

// Caller checks with errors.Is
user, err := getUser(42)
if errors.Is(err, ErrNotFound) {
    // Handle not found
} else if err != nil {
    // Handle other errors
}
```

---

## Error Handling Patterns

### Early Return
```go
// Idiomatic: handle error immediately, keep happy path unindented
func processOrder(orderID int) error {
    order, err := getOrder(orderID)
    if err != nil {
        return fmt.Errorf("processOrder: %w", err)
    }

    if err := validateOrder(order); err != nil {
        return fmt.Errorf("processOrder: %w", err)
    }

    if err := chargeCustomer(order); err != nil {
        return fmt.Errorf("processOrder: %w", err)
    }

    return nil
}
```

### Error Accumulation
```go
// Collect multiple errors
type MultiError struct {
    Errors []error
}

func (m *MultiError) Error() string {
    msgs := make([]string, len(m.Errors))
    for i, e := range m.Errors {
        msgs[i] = e.Error()
    }
    return strings.Join(msgs, "; ")
}

func (m *MultiError) Add(err error) {
    if err != nil {
        m.Errors = append(m.Errors, err)
    }
}

func (m *MultiError) Err() error {
    if len(m.Errors) == 0 {
        return nil
    }
    return m
}

// Usage
func validateUser(u User) error {
    var errs MultiError
    if u.Name == "" {
        errs.Add(errors.New("name is required"))
    }
    if u.Age < 0 {
        errs.Add(errors.New("age must be non-negative"))
    }
    if u.Email == "" {
        errs.Add(errors.New("email is required"))
    }
    return errs.Err()
}
```

### errors.Join (Go 1.20+)
```go
// errors.Join combines multiple errors
err1 := errors.New("error 1")
err2 := errors.New("error 2")
err3 := errors.New("error 3")

combined := errors.Join(err1, err2, err3)
fmt.Println(combined)
// error 1
// error 2
// error 3

errors.Is(combined, err1)  // true
errors.Is(combined, err2)  // true
```

### Retry with Error Check
```go
func withRetry(attempts int, fn func() error) error {
    var lastErr error
    for i := 0; i < attempts; i++ {
        if err := fn(); err == nil {
            return nil
        } else {
            lastErr = err
            // Don't retry on non-retryable errors
            if !isRetryable(err) {
                return err
            }
            time.Sleep(time.Duration(i+1) * 100 * time.Millisecond)
        }
    }
    return fmt.Errorf("failed after %d attempts: %w", attempts, lastErr)
}

func isRetryable(err error) bool {
    return errors.Is(err, ErrTemporary) || errors.Is(err, ErrTimeout)
}
```

### Error Middleware
```go
type ErrorHandler func(error) error

func withLogging(next ErrorHandler) ErrorHandler {
    return func(err error) error {
        if err != nil {
            log.Printf("error: %v", err)
        }
        return next(err)
    }
}

func withMetrics(next ErrorHandler) ErrorHandler {
    return func(err error) error {
        if err != nil {
            metrics.Increment("errors")
        }
        return next(err)
    }
}
```

---

## Panic and Recover

### Panic Internals
```go
// panic() stores value in goroutine's panic info
// Unwinds stack, running deferred functions
// If no recover(), prints stack trace and exits

// panic value can be any type
panic("string message")
panic(42)
panic(errors.New("error"))
panic(struct{ code int }{code: 500})
```

### Recover Internals
```go
// recover() returns the panic value
// Only works inside a deferred function
// Returns nil if no panic in progress

func safeCall(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            switch v := r.(type) {
            case error:
                err = v
            case string:
                err = errors.New(v)
            default:
                err = fmt.Errorf("panic: %v", v)
            }
        }
    }()
    fn()
    return nil
}

err := safeCall(func() {
    panic("something went wrong")
})
fmt.Println(err)  // something went wrong
```

### Converting Panic to Error
```go
// Useful for wrapping libraries that panic
func mustNotPanic(fn func() error) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered panic: %v\n%s", r, debug.Stack())
        }
    }()
    return fn()
}
```

---

## Concurrent Error Handling

### errgroup
```go
import "golang.org/x/sync/errgroup"

func fetchAll(urls []string) error {
    g, ctx := errgroup.WithContext(context.Background())

    results := make([]string, len(urls))
    for i, url := range urls {
        i, url := i, url  // Capture loop variables
        g.Go(func() error {
            resp, err := http.Get(url)
            if err != nil {
                return fmt.Errorf("fetch %s: %w", url, err)
            }
            defer resp.Body.Close()
            body, err := io.ReadAll(resp.Body)
            if err != nil {
                return err
            }
            results[i] = string(body)
            return nil
        })
    }

    if err := g.Wait(); err != nil {
        return err  // Returns first error
    }
    return nil
}
```

### Error Channel
```go
func processItems(items []Item) []error {
    errs := make(chan error, len(items))
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)
        go func(item Item) {
            defer wg.Done()
            if err := process(item); err != nil {
                errs <- err
            }
        }(item)
    }

    wg.Wait()
    close(errs)

    var result []error
    for err := range errs {
        result = append(result, err)
    }
    return result
}
```

---

## Common Pitfalls

### 1. Ignoring Errors
```go
// Bad: silently ignoring errors
result, _ := someOperation()

// Good: handle or explicitly acknowledge
result, err := someOperation()
if err != nil {
    return fmt.Errorf("someOperation: %w", err)
}
```

### 2. Typed Nil in Interface
```go
// Classic Go gotcha
func getError() error {
    var err *MyError = nil
    return err  // Non-nil interface holding nil pointer!
}

fmt.Println(getError() == nil)  // false — SURPRISE!

// Fix: return untyped nil
func getError() error {
    var err *MyError = nil
    if err == nil {
        return nil  // Untyped nil
    }
    return err
}
```

### 3. Wrapping Nil Error
```go
// Bad: wrapping nil creates non-nil error
func process() error {
    err := mayFail()
    return fmt.Errorf("process: %w", err)  // If err==nil, returns non-nil!
}

// Good: check before wrapping
func process() error {
    if err := mayFail(); err != nil {
        return fmt.Errorf("process: %w", err)
    }
    return nil
}
```

### 4. Losing Error Context
```go
// Bad: discards original error
func bad() error {
    _, err := db.Query(...)
    if err != nil {
        return errors.New("database error")  // Original error lost!
    }
    return nil
}

// Good: wrap with context
func good() error {
    _, err := db.Query(...)
    if err != nil {
        return fmt.Errorf("database query: %w", err)  // Preserves original
    }
    return nil
}
```

---

## Best Practices

### 1. Add Context When Wrapping
```go
// Include: what operation, what input, why it failed
return fmt.Errorf("getUserByEmail(%q): %w", email, err)
```

### 2. Use Sentinel Errors for Expected Conditions
```go
var ErrNotFound = errors.New("not found")
// Callers can check: errors.Is(err, ErrNotFound)
```

### 3. Use Custom Types for Rich Error Info
```go
type QueryError struct {
    Query string
    Err   error
}
func (e *QueryError) Error() string { return fmt.Sprintf("query %q: %v", e.Query, e.Err) }
func (e *QueryError) Unwrap() error { return e.Err }
```

### 4. Don't Panic for Expected Errors
```go
// Panic: programmer errors, impossible states
// Error return: expected failures, user input, I/O, network
```

### 5. Log at the Top, Wrap at the Bottom
```go
// Low-level: wrap and return
func dbQuery() error {
    return fmt.Errorf("dbQuery: %w", err)
}

// High-level: log once
func handler() {
    if err := dbQuery(); err != nil {
        log.Printf("handler failed: %v", err)
        http.Error(w, "internal error", 500)
    }
}
```

---

## Summary

### Key Concepts
1. `error` is an interface with one method: `Error() string`
2. `errors.New` returns a pointer — each call creates a unique error
3. `fmt.Errorf` with `%w` wraps errors for chain traversal
4. `errors.Is` checks identity through the chain; `errors.As` extracts type
5. Typed nil in interface is NOT nil — common gotcha
6. `recover()` only works inside `defer`

### Error Handling Checklist
```
✓ Return errors, don't panic for expected failures
✓ Wrap errors with context: fmt.Errorf("op: %w", err)
✓ Use sentinel errors for expected conditions
✓ Use custom types for structured error data
✓ Implement Unwrap() for custom wrapped errors
✓ Check errors.Is/As instead of == for wrapped errors
✓ Never return typed nil as error interface
✓ Log once at the top level, wrap at lower levels
```

# Error Handling in Golang

## Basic Error Handling

### Creating Errors
```go
import "errors"

// Using errors.New
err := errors.New("something went wrong")

// Using fmt.Errorf
err := fmt.Errorf("failed to process: %s", filename)
```

### Checking Errors
```go
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 0)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)
}
```

## Custom Errors

### Custom Error Type
```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on field '%s': %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{
            Field:   "age",
            Message: "must be non-negative",
        }
    }
    if age > 150 {
        return &ValidationError{
            Field:   "age",
            Message: "must be less than 150",
        }
    }
    return nil
}
```

### Error with Additional Context
```go
type FileError struct {
    Op   string
    Path string
    Err  error
}

func (e *FileError) Error() string {
    return fmt.Sprintf("%s %s: %v", e.Op, e.Path, e.Err)
}

func (e *FileError) Unwrap() error {
    return e.Err
}

func readFile(path string) error {
    _, err := os.Open(path)
    if err != nil {
        return &FileError{
            Op:   "open",
            Path: path,
            Err:  err,
        }
    }
    return nil
}
```

## Error Wrapping (Go 1.13+)

### Wrapping Errors
```go
func processFile(filename string) error {
    data, err := readFile(filename)
    if err != nil {
        return fmt.Errorf("failed to process file: %w", err)
    }
    // Process data
    return nil
}
```

### Unwrapping Errors
```go
import "errors"

func main() {
    err := processFile("data.txt")
    if err != nil {
        // Unwrap to get the underlying error
        unwrapped := errors.Unwrap(err)
        fmt.Println("Unwrapped:", unwrapped)
    }
}
```

### Error Is
```go
var ErrNotFound = errors.New("not found")

func findUser(id int) error {
    return ErrNotFound
}

func main() {
    err := findUser(123)
    if errors.Is(err, ErrNotFound) {
        fmt.Println("User not found")
    }
}
```

### Error As
```go
func main() {
    err := validateAge(-5)
    
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        fmt.Printf("Validation failed on field: %s\n", validationErr.Field)
    }
}
```

## Panic and Recover

### Panic
```go
func mustConnect(url string) {
    if url == "" {
        panic("URL cannot be empty")
    }
    // Connect logic
}
```

### Recover
```go
func safeExecute(fn func()) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from panic:", r)
        }
    }()
    fn()
}

func main() {
    safeExecute(func() {
        panic("something went wrong")
    })
    fmt.Println("Program continues")
}
```

### Recover in Goroutines
```go
func worker(id int) {
    defer func() {
        if r := recover(); r != nil {
            fmt.Printf("Worker %d recovered from panic: %v\n", id, r)
        }
    }()
    
    if id == 2 {
        panic("worker 2 panicked")
    }
    fmt.Printf("Worker %d completed\n", id)
}

func main() {
    for i := 1; i <= 3; i++ {
        go worker(i)
    }
    time.Sleep(time.Second)
}
```

## Error Handling Patterns

### Multiple Return Values
```go
func parseConfig(filename string) (*Config, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("failed to read config: %w", err)
    }
    
    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("failed to parse config: %w", err)
    }
    
    return &config, nil
}
```

### Error Accumulation
```go
type MultiError struct {
    Errors []error
}

func (m *MultiError) Error() string {
    var msgs []string
    for _, err := range m.Errors {
        msgs = append(msgs, err.Error())
    }
    return strings.Join(msgs, "; ")
}

func (m *MultiError) Add(err error) {
    if err != nil {
        m.Errors = append(m.Errors, err)
    }
}

func validateUser(user *User) error {
    var errs MultiError
    
    if user.Name == "" {
        errs.Add(errors.New("name is required"))
    }
    if user.Email == "" {
        errs.Add(errors.New("email is required"))
    }
    if user.Age < 0 {
        errs.Add(errors.New("age must be non-negative"))
    }
    
    if len(errs.Errors) > 0 {
        return &errs
    }
    return nil
}
```

### Sentinel Errors
```go
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrInvalidInput = errors.New("invalid input")
)

func getUser(id int) (*User, error) {
    if id < 0 {
        return nil, ErrInvalidInput
    }
    // Database lookup
    return nil, ErrNotFound
}

func main() {
    user, err := getUser(-1)
    switch err {
    case nil:
        fmt.Println("User:", user)
    case ErrNotFound:
        fmt.Println("User not found")
    case ErrInvalidInput:
        fmt.Println("Invalid user ID")
    default:
        fmt.Println("Unknown error:", err)
    }
}
```

### Error Context Chain
```go
func loadConfig() error {
    if err := validateEnvironment(); err != nil {
        return fmt.Errorf("environment validation failed: %w", err)
    }
    
    if err := readConfigFile(); err != nil {
        return fmt.Errorf("config file read failed: %w", err)
    }
    
    return nil
}

func validateEnvironment() error {
    if os.Getenv("APP_ENV") == "" {
        return errors.New("APP_ENV not set")
    }
    return nil
}

func readConfigFile() error {
    return errors.New("file not found")
}
```

## Best Practices

### 1. Always Check Errors
```go
// Bad
data, _ := os.ReadFile("file.txt")

// Good
data, err := os.ReadFile("file.txt")
if err != nil {
    return fmt.Errorf("failed to read file: %w", err)
}
```

### 2. Add Context to Errors
```go
// Bad
return err

// Good
return fmt.Errorf("failed to process user %d: %w", userID, err)
```

### 3. Use Custom Errors for Domain Logic
```go
type InsufficientFundsError struct {
    Available float64
    Required  float64
}

func (e *InsufficientFundsError) Error() string {
    return fmt.Sprintf("insufficient funds: have %.2f, need %.2f", 
        e.Available, e.Required)
}

func withdraw(account *Account, amount float64) error {
    if account.Balance < amount {
        return &InsufficientFundsError{
            Available: account.Balance,
            Required:  amount,
        }
    }
    account.Balance -= amount
    return nil
}
```

### 4. Don't Panic in Libraries
```go
// Bad - library code
func MustParse(s string) int {
    n, err := strconv.Atoi(s)
    if err != nil {
        panic(err)
    }
    return n
}

// Good - library code
func Parse(s string) (int, error) {
    n, err := strconv.Atoi(s)
    if err != nil {
        return 0, fmt.Errorf("failed to parse: %w", err)
    }
    return n, nil
}
```

### 5. Use defer for Cleanup
```go
func processFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close()
    
    // Process file
    return nil
}
```

### 6. Error Logging
```go
func handleRequest(w http.ResponseWriter, r *http.Request) {
    if err := processRequest(r); err != nil {
        log.Printf("Error processing request: %v", err)
        http.Error(w, "Internal Server Error", http.StatusInternalServerError)
        return
    }
    w.WriteHeader(http.StatusOK)
}
```

### 7. Retry Logic with Errors
```go
func retryOperation(maxRetries int, operation func() error) error {
    var err error
    for i := 0; i < maxRetries; i++ {
        err = operation()
        if err == nil {
            return nil
        }
        time.Sleep(time.Second * time.Duration(i+1))
    }
    return fmt.Errorf("operation failed after %d retries: %w", maxRetries, err)
}
```


## Error Handling Philosophy in Go

### Why Explicit Error Handling?

Go's approach to error handling is explicit and verbose by design:

1. **Errors are values**: Can be inspected, wrapped, and passed around
2. **No hidden control flow**: No exceptions jumping up the stack
3. **Forces handling**: Can't ignore errors easily
4. **Clear error paths**: Easy to see where errors can occur

### Error vs Exception

**Exceptions (Java, Python):**
- Hidden control flow
- Can be caught far from source
- Performance overhead
- Easy to ignore

**Errors (Go):**
- Explicit return values
- Handled at call site
- No performance overhead
- Hard to ignore

## Advanced Error Patterns

### Error Wrapping Chain

```go
package main

import (
    "errors"
    "fmt"
)

func database() error {
    return errors.New("connection refused")
}

func repository() error {
    err := database()
    if err != nil {
        return fmt.Errorf("repository: %w", err)
    }
    return nil
}

func service() error {
    err := repository()
    if err != nil {
        return fmt.Errorf("service: %w", err)
    }
    return nil
}

func main() {
    err := service()
    if err != nil {
        fmt.Println(err)
        // Output: service: repository: connection refused
        
        // Unwrap to get original error
        for err != nil {
            fmt.Println(err)
            err = errors.Unwrap(err)
        }
    }
}
```

### Structured Errors

```go
type HTTPError struct {
    StatusCode int
    Message    string
    Err        error
}

func (e *HTTPError) Error() string {
    if e.Err != nil {
        return fmt.Sprintf("HTTP %d: %s: %v", e.StatusCode, e.Message, e.Err)
    }
    return fmt.Sprintf("HTTP %d: %s", e.StatusCode, e.Message)
}

func (e *HTTPError) Unwrap() error {
    return e.Err
}

// Usage
func fetchData(url string) error {
    // Simulate HTTP request
    return &HTTPError{
        StatusCode: 404,
        Message:    "Resource not found",
        Err:        errors.New("page does not exist"),
    }
}
```

### Error Groups

```go
type ErrorGroup struct {
    errors []error
}

func (eg *ErrorGroup) Add(err error) {
    if err != nil {
        eg.errors = append(eg.errors, err)
    }
}

func (eg *ErrorGroup) Error() string {
    if len(eg.errors) == 0 {
        return ""
    }
    var msgs []string
    for _, err := range eg.errors {
        msgs = append(msgs, err.Error())
    }
    return fmt.Sprintf("multiple errors: %s", strings.Join(msgs, "; "))
}

func (eg *ErrorGroup) HasErrors() bool {
    return len(eg.errors) > 0
}

// Usage
func validateForm(form *Form) error {
    var eg ErrorGroup
    
    if form.Name == "" {
        eg.Add(errors.New("name is required"))
    }
    if form.Email == "" {
        eg.Add(errors.New("email is required"))
    }
    if form.Age < 0 {
        eg.Add(errors.New("age must be positive"))
    }
    
    if eg.HasErrors() {
        return &eg
    }
    return nil
}
```

## Panic Recovery Patterns

### Recovering from Panics in HTTP Handlers

```go
func recoverMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic recovered: %v", err)
                log.Printf("Stack trace: %s", debug.Stack())
                
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}
```

### Panic in Goroutines

```go
func safeGo(fn func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("Goroutine panic: %v\n%s", r, debug.Stack())
            }
        }()
        fn()
    }()
}

// Usage
func main() {
    safeGo(func() {
        panic("something went wrong")
    })
    
    time.Sleep(time.Second)
    fmt.Println("Main continues")
}
```

## Error Handling in Concurrent Code

### Collecting Errors from Goroutines

```go
func processItems(items []string) error {
    errCh := make(chan error, len(items))
    var wg sync.WaitGroup
    
    for _, item := range items {
        wg.Add(1)
        go func(it string) {
            defer wg.Done()
            if err := process(it); err != nil {
                errCh <- err
            }
        }(item)
    }
    
    wg.Wait()
    close(errCh)
    
    // Collect all errors
    var errors []error
    for err := range errCh {
        errors = append(errors, err)
    }
    
    if len(errors) > 0 {
        return fmt.Errorf("processing failed with %d errors", len(errors))
    }
    return nil
}
```

### Using errgroup Package

```go
import "golang.org/x/sync/errgroup"

func fetchAll(urls []string) error {
    g := new(errgroup.Group)
    
    for _, url := range urls {
        url := url // Capture loop variable
        g.Go(func() error {
            return fetch(url)
        })
    }
    
    // Wait for all goroutines and return first error
    if err := g.Wait(); err != nil {
        return fmt.Errorf("fetch failed: %w", err)
    }
    return nil
}
```

## Testing Error Handling

```go
func TestDivide_DivisionByZero(t *testing.T) {
    _, err := Divide(10, 0)
    
    // Check error exists
    if err == nil {
        t.Fatal("expected error, got nil")
    }
    
    // Check error type
    var divErr *DivisionError
    if !errors.As(err, &divErr) {
        t.Fatalf("expected DivisionError, got %T", err)
    }
    
    // Check error message
    expected := "division by zero"
    if err.Error() != expected {
        t.Errorf("expected %q, got %q", expected, err.Error())
    }
}
```

# Golang Best Practices

## Code Organization

### Project Structure
```
myproject/
├── cmd/
│   └── myapp/
│       └── main.go
├── internal/
│   ├── handler/
│   ├── service/
│   └── repository/
├── pkg/
│   └── utils/
├── api/
├── configs/
├── scripts/
├── test/
├── go.mod
├── go.sum
└── README.md
```

### Package Organization
```go
// Good: Small, focused packages
package user

type User struct {
    ID   int
    Name string
}

type Repository interface {
    GetUser(id int) (*User, error)
    SaveUser(user *User) error
}

// Bad: God package with everything
package models // Contains all models, services, repositories
```

## Naming Conventions

### Variables
```go
// Good
var userCount int
var isActive bool
var maxRetries = 3

// Bad
var uc int
var active bool
var MAX_RETRIES = 3
```

### Functions
```go
// Good: Verb-based names
func GetUser(id int) (*User, error)
func CalculateTotal(items []Item) float64
func IsValid(input string) bool

// Bad
func User(id int) (*User, error)
func Total(items []Item) float64
```

### Interfaces
```go
// Good: -er suffix for single-method interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Good: Descriptive names for multi-method interfaces
type UserRepository interface {
    GetUser(id int) (*User, error)
    SaveUser(user *User) error
    DeleteUser(id int) error
}
```

### Constants
```go
// Good
const (
    MaxConnections = 100
    DefaultTimeout = 30 * time.Second
)

// Bad
const (
    MAX_CONNECTIONS = 100
    default_timeout = 30
)
```

## Error Handling

### Always Check Errors
```go
// Good
data, err := os.ReadFile("file.txt")
if err != nil {
    return fmt.Errorf("failed to read file: %w", err)
}

// Bad
data, _ := os.ReadFile("file.txt")
```

### Wrap Errors with Context
```go
// Good
if err := processUser(user); err != nil {
    return fmt.Errorf("failed to process user %d: %w", user.ID, err)
}

// Bad
if err := processUser(user); err != nil {
    return err
}
```

### Use Custom Errors for Domain Logic
```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}
```

## Concurrency

### Use Channels for Communication
```go
// Good
func worker(jobs <-chan Job, results chan<- Result) {
    for job := range jobs {
        results <- process(job)
    }
}

// Bad: Shared memory with mutex for simple communication
var mu sync.Mutex
var results []Result

func worker(jobs []Job) {
    for _, job := range jobs {
        result := process(job)
        mu.Lock()
        results = append(results, result)
        mu.Unlock()
    }
}
```

### Always Close Channels
```go
// Good
func producer() <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := 0; i < 10; i++ {
            ch <- i
        }
    }()
    return ch
}
```

### Use Context for Cancellation
```go
// Good
func doWork(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()
        default:
            // Do work
        }
    }
}
```

### Avoid Goroutine Leaks
```go
// Good
func search(ctx context.Context, query string) (Result, error) {
    resultCh := make(chan Result, 1)
    
    go func() {
        select {
        case resultCh <- performSearch(query):
        case <-ctx.Done():
        }
    }()
    
    select {
    case result := <-resultCh:
        return result, nil
    case <-ctx.Done():
        return Result{}, ctx.Err()
    }
}
```

## Performance Optimization

### Use Pointers for Large Structs
```go
// Good: Pass pointer for large structs
func ProcessUser(user *User) error {
    // Process user
    return nil
}

// Bad: Pass by value for large structs
func ProcessUser(user User) error {
    // Process user
    return nil
}
```

### Preallocate Slices
```go
// Good
users := make([]User, 0, expectedSize)
for i := 0; i < expectedSize; i++ {
    users = append(users, User{ID: i})
}

// Bad
var users []User
for i := 0; i < 1000; i++ {
    users = append(users, User{ID: i})
}
```

### Use strings.Builder for String Concatenation
```go
// Good
var builder strings.Builder
for _, s := range strings {
    builder.WriteString(s)
}
result := builder.String()

// Bad
var result string
for _, s := range strings {
    result += s
}
```

### Use sync.Pool for Frequently Allocated Objects
```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func processData(data []byte) {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer bufferPool.Put(buf)
    buf.Reset()
    
    // Use buffer
}
```

## Memory Management

### Avoid Memory Leaks
```go
// Good: Clear references
func process() {
    cache := make(map[string]*Data)
    // Use cache
    
    // Clear when done
    for k := range cache {
        delete(cache, k)
    }
}

// Bad: Keep references indefinitely
var globalCache = make(map[string]*Data)

func process() {
    // Add to cache but never remove
    globalCache[key] = data
}
```

### Use defer for Resource Cleanup
```go
// Good
func readFile(filename string) error {
    f, err := os.Open(filename)
    if err != nil {
        return err
    }
    defer f.Close()
    
    // Read file
    return nil
}
```

## Interface Design

### Accept Interfaces, Return Structs
```go
// Good
func ProcessData(r io.Reader) (*Result, error) {
    // Process
    return &Result{}, nil
}

// Bad
func ProcessData(f *os.File) (*Result, error) {
    // Process
    return &Result{}, nil
}
```

### Keep Interfaces Small
```go
// Good
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Bad
type FileHandler interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
    Close() error
    Seek(offset int64, whence int) (int64, error)
    Stat() (FileInfo, error)
}
```

## Testing

### Write Table-Driven Tests
```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 2, 3, 5},
        {"negative", -2, -3, -5},
        {"zero", 0, 0, 0},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("got %d, want %d", result, tt.expected)
            }
        })
    }
}
```

### Use Test Helpers
```go
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if got != want {
        t.Errorf("got %v, want %v", got, want)
    }
}
```

## Common Pitfalls

### 1. Range Loop Variable Capture
```go
// Bad
var wg sync.WaitGroup
for _, v := range values {
    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println(v) // Wrong: captures loop variable
    }()
}

// Good
for _, v := range values {
    wg.Add(1)
    go func(val int) {
        defer wg.Done()
        fmt.Println(val)
    }(v)
}
```

### 2. Slice Append Gotcha
```go
// Bad
func appendValue(slice []int) []int {
    slice = append(slice, 1)
    return slice // May not modify original
}

// Good
func appendValue(slice []int) []int {
    return append(slice, 1)
}
```

### 3. Map Concurrent Access
```go
// Bad
var m = make(map[string]int)

func update(key string, value int) {
    m[key] = value // Race condition
}

// Good
var (
    m  = make(map[string]int)
    mu sync.RWMutex
)

func update(key string, value int) {
    mu.Lock()
    defer mu.Unlock()
    m[key] = value
}
```

### 4. Nil Interface vs Nil Value
```go
// Bad
func returnsError() error {
    var err *MyError = nil
    return err // Returns non-nil interface with nil value
}

// Good
func returnsError() error {
    var err *MyError = nil
    if err != nil {
        return err
    }
    return nil
}
```

### 5. Defer in Loops
```go
// Bad
for _, file := range files {
    f, _ := os.Open(file)
    defer f.Close() // Defers accumulate
}

// Good
for _, file := range files {
    func() {
        f, _ := os.Open(file)
        defer f.Close()
        // Process file
    }()
}
```

## Code Style

### Use gofmt
```bash
# Format all files
gofmt -w .

# Check formatting
gofmt -l .
```

### Use golint
```bash
# Install
go install golang.org/x/lint/golint@latest

# Run
golint ./...
```

### Use go vet
```bash
# Check for common mistakes
go vet ./...
```

### Use staticcheck
```bash
# Install
go install honnef.co/go/tools/cmd/staticcheck@latest

# Run
staticcheck ./...
```

## Documentation

### Package Documentation
```go
// Package user provides user management functionality.
// It includes user creation, authentication, and profile management.
package user
```

### Function Documentation
```go
// GetUser retrieves a user by ID from the database.
// It returns an error if the user is not found or if there's a database error.
func GetUser(id int) (*User, error) {
    // Implementation
}
```

### Example Tests
```go
func ExampleAdd() {
    result := Add(2, 3)
    fmt.Println(result)
    // Output: 5
}
```

## Dependency Management

### Use Go Modules
```bash
# Initialize module
go mod init github.com/username/project

# Add dependency
go get github.com/pkg/errors

# Update dependencies
go get -u ./...

# Tidy dependencies
go mod tidy

# Vendor dependencies
go mod vendor
```

### Pin Dependencies
```go
// go.mod
module github.com/username/project

go 1.21

require (
    github.com/pkg/errors v0.9.1
    github.com/stretchr/testify v1.8.4
)
```


## Advanced Go Patterns

### Functional Options Pattern

```go
type Server struct {
    host    string
    port    int
    timeout time.Duration
    maxConn int
}

type Option func(*Server)

func WithHost(host string) Option {
    return func(s *Server) {
        s.host = host
    }
}

func WithPort(port int) Option {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(timeout time.Duration) Option {
    return func(s *Server) {
        s.timeout = timeout
    }
}

func NewServer(opts ...Option) *Server {
    // Default values
    s := &Server{
        host:    "localhost",
        port:    8080,
        timeout: 30 * time.Second,
        maxConn: 100,
    }
    
    // Apply options
    for _, opt := range opts {
        opt(s)
    }
    
    return s
}

// Usage
server := NewServer(
    WithHost("0.0.0.0"),
    WithPort(9000),
    WithTimeout(60 * time.Second),
)
```

### Builder Pattern

```go
type QueryBuilder struct {
    table   string
    columns []string
    where   []string
    orderBy string
    limit   int
}

func NewQueryBuilder(table string) *QueryBuilder {
    return &QueryBuilder{table: table}
}

func (qb *QueryBuilder) Select(columns ...string) *QueryBuilder {
    qb.columns = columns
    return qb
}

func (qb *QueryBuilder) Where(condition string) *QueryBuilder {
    qb.where = append(qb.where, condition)
    return qb
}

func (qb *QueryBuilder) OrderBy(column string) *QueryBuilder {
    qb.orderBy = column
    return qb
}

func (qb *QueryBuilder) Limit(n int) *QueryBuilder {
    qb.limit = n
    return qb
}

func (qb *QueryBuilder) Build() string {
    query := "SELECT "
    if len(qb.columns) == 0 {
        query += "*"
    } else {
        query += strings.Join(qb.columns, ", ")
    }
    query += " FROM " + qb.table
    
    if len(qb.where) > 0 {
        query += " WHERE " + strings.Join(qb.where, " AND ")
    }
    
    if qb.orderBy != "" {
        query += " ORDER BY " + qb.orderBy
    }
    
    if qb.limit > 0 {
        query += fmt.Sprintf(" LIMIT %d", qb.limit)
    }
    
    return query
}

// Usage
query := NewQueryBuilder("users").
    Select("id", "name", "email").
    Where("age > 18").
    Where("active = true").
    OrderBy("name").
    Limit(10).
    Build()
```

### Strategy Pattern

```go
type PaymentStrategy interface {
    Pay(amount float64) error
}

type CreditCardPayment struct {
    cardNumber string
}

func (c *CreditCardPayment) Pay(amount float64) error {
    fmt.Printf("Paying $%.2f with credit card %s\n", amount, c.cardNumber)
    return nil
}

type PayPalPayment struct {
    email string
}

func (p *PayPalPayment) Pay(amount float64) error {
    fmt.Printf("Paying $%.2f with PayPal %s\n", amount, p.email)
    return nil
}

type PaymentProcessor struct {
    strategy PaymentStrategy
}

func (pp *PaymentProcessor) SetStrategy(strategy PaymentStrategy) {
    pp.strategy = strategy
}

func (pp *PaymentProcessor) ProcessPayment(amount float64) error {
    return pp.strategy.Pay(amount)
}

// Usage
processor := &PaymentProcessor{}

processor.SetStrategy(&CreditCardPayment{cardNumber: "1234-5678"})
processor.ProcessPayment(100.00)

processor.SetStrategy(&PayPalPayment{email: "user@example.com"})
processor.ProcessPayment(50.00)
```

### Observer Pattern

```go
type Observer interface {
    Update(data interface{})
}

type Subject struct {
    observers []Observer
}

func (s *Subject) Attach(observer Observer) {
    s.observers = append(s.observers, observer)
}

func (s *Subject) Notify(data interface{}) {
    for _, observer := range s.observers {
        observer.Update(data)
    }
}

type EmailObserver struct {
    email string
}

func (e *EmailObserver) Update(data interface{}) {
    fmt.Printf("Sending email to %s: %v\n", e.email, data)
}

type SMSObserver struct {
    phone string
}

func (s *SMSObserver) Update(data interface{}) {
    fmt.Printf("Sending SMS to %s: %v\n", s.phone, data)
}

// Usage
subject := &Subject{}
subject.Attach(&EmailObserver{email: "user@example.com"})
subject.Attach(&SMSObserver{phone: "123-456-7890"})

subject.Notify("New order received")
```

## Performance Optimization

### String Concatenation

```go
// Slow: Creates new string each iteration
func slowConcat(strs []string) string {
    result := ""
    for _, s := range strs {
        result += s // O(n²)
    }
    return result
}

// Fast: Uses strings.Builder
func fastConcat(strs []string) string {
    var builder strings.Builder
    builder.Grow(len(strs) * 10) // Pre-allocate
    for _, s := range strs {
        builder.WriteString(s)
    }
    return builder.String()
}
```

### Slice Preallocation

```go
// Slow: Multiple reallocations
func slowAppend() []int {
    var s []int
    for i := 0; i < 10000; i++ {
        s = append(s, i)
    }
    return s
}

// Fast: Single allocation
func fastAppend() []int {
    s := make([]int, 0, 10000)
    for i := 0; i < 10000; i++ {
        s = append(s, i)
    }
    return s
}
```

### Map Preallocation

```go
// Slow
func slowMap() map[int]int {
    m := make(map[int]int)
    for i := 0; i < 10000; i++ {
        m[i] = i * 2
    }
    return m
}

// Fast
func fastMap() map[int]int {
    m := make(map[int]int, 10000)
    for i := 0; i < 10000; i++ {
        m[i] = i * 2
    }
    return m
}
```

### Avoid Unnecessary Allocations

```go
// Slow: Allocates on every call
func slowFormat(name string, age int) string {
    return fmt.Sprintf("Name: %s, Age: %d", name, age)
}

// Fast: Uses strings.Builder
func fastFormat(name string, age int) string {
    var b strings.Builder
    b.WriteString("Name: ")
    b.WriteString(name)
    b.WriteString(", Age: ")
    b.WriteString(strconv.Itoa(age))
    return b.String()
}
```

### Use Sync.Pool for Temporary Objects

```go
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func processData(data []byte) []byte {
    buf := bufferPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufferPool.Put(buf)
    }()
    
    buf.Write(data)
    // Process buffer
    return buf.Bytes()
}
```

## Security Best Practices

### Input Validation

```go
func validateEmail(email string) error {
    if email == "" {
        return errors.New("email is required")
    }
    
    // Simple regex for demonstration
    emailRegex := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    if !emailRegex.MatchString(email) {
        return errors.New("invalid email format")
    }
    
    return nil
}

func sanitizeInput(input string) string {
    // Remove dangerous characters
    return strings.Map(func(r rune) rune {
        if unicode.IsLetter(r) || unicode.IsNumber(r) || unicode.IsSpace(r) {
            return r
        }
        return -1
    }, input)
}
```

### SQL Injection Prevention

```go
// Bad: SQL injection vulnerable
func getUserBad(db *sql.DB, username string) (*User, error) {
    query := fmt.Sprintf("SELECT * FROM users WHERE username = '%s'", username)
    // Vulnerable to: username = "admin' OR '1'='1"
    row := db.QueryRow(query)
    // ...
}

// Good: Use parameterized queries
func getUserGood(db *sql.DB, username string) (*User, error) {
    query := "SELECT * FROM users WHERE username = ?"
    row := db.QueryRow(query, username)
    // ...
}
```

### Secure Password Handling

```go
import "golang.org/x/crypto/bcrypt"

func hashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

func checkPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

### Rate Limiting

```go
type RateLimiter struct {
    requests map[string][]time.Time
    mu       sync.Mutex
    limit    int
    window   time.Duration
}

func NewRateLimiter(limit int, window time.Duration) *RateLimiter {
    return &RateLimiter{
        requests: make(map[string][]time.Time),
        limit:    limit,
        window:   window,
    }
}

func (rl *RateLimiter) Allow(key string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    now := time.Now()
    windowStart := now.Add(-rl.window)
    
    // Clean old requests
    requests := rl.requests[key]
    validRequests := []time.Time{}
    for _, t := range requests {
        if t.After(windowStart) {
            validRequests = append(validRequests, t)
        }
    }
    
    if len(validRequests) >= rl.limit {
        return false
    }
    
    validRequests = append(validRequests, now)
    rl.requests[key] = validRequests
    return true
}
```

## Logging Best Practices

### Structured Logging

```go
import "log/slog"

func setupLogger() *slog.Logger {
    return slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))
}

func processRequest(logger *slog.Logger, userID int, action string) {
    logger.Info("processing request",
        slog.Int("user_id", userID),
        slog.String("action", action),
        slog.Time("timestamp", time.Now()),
    )
}
```

### Context-Aware Logging

```go
type contextKey string

const loggerKey contextKey = "logger"

func WithLogger(ctx context.Context, logger *slog.Logger) context.Context {
    return context.WithValue(ctx, loggerKey, logger)
}

func LoggerFromContext(ctx context.Context) *slog.Logger {
    if logger, ok := ctx.Value(loggerKey).(*slog.Logger); ok {
        return logger
    }
    return slog.Default()
}

func handleRequest(ctx context.Context) {
    logger := LoggerFromContext(ctx)
    logger.Info("handling request")
}
```

## Configuration Management

### Environment-Based Configuration

```go
type Config struct {
    Port        int
    DatabaseURL string
    LogLevel    string
    Debug       bool
}

func LoadConfig() (*Config, error) {
    port, err := strconv.Atoi(getEnv("PORT", "8080"))
    if err != nil {
        return nil, err
    }
    
    return &Config{
        Port:        port,
        DatabaseURL: getEnv("DATABASE_URL", "postgres://localhost/mydb"),
        LogLevel:    getEnv("LOG_LEVEL", "info"),
        Debug:       getEnv("DEBUG", "false") == "true",
    }, nil
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

### Configuration Validation

```go
func (c *Config) Validate() error {
    if c.Port < 1 || c.Port > 65535 {
        return fmt.Errorf("invalid port: %d", c.Port)
    }
    
    if c.DatabaseURL == "" {
        return errors.New("database URL is required")
    }
    
    validLogLevels := map[string]bool{
        "debug": true,
        "info":  true,
        "warn":  true,
        "error": true,
    }
    
    if !validLogLevels[c.LogLevel] {
        return fmt.Errorf("invalid log level: %s", c.LogLevel)
    }
    
    return nil
}
```

## Profiling and Debugging

### CPU Profiling

```go
import "runtime/pprof"

func main() {
    f, err := os.Create("cpu.prof")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()
    
    // Your code here
}

// Analyze with: go tool pprof cpu.prof
```

### Memory Profiling

```go
func main() {
    // Your code here
    
    f, err := os.Create("mem.prof")
    if err != nil {
        log.Fatal(err)
    }
    defer f.Close()
    
    runtime.GC() // Get up-to-date statistics
    if err := pprof.WriteHeapProfile(f); err != nil {
        log.Fatal(err)
    }
}

// Analyze with: go tool pprof mem.prof
```

### HTTP Profiling

```go
import _ "net/http/pprof"

func main() {
    go func() {
        log.Println(http.ListenAndServe("localhost:6060", nil))
    }()
    
    // Your application code
}

// Access profiles at:
// http://localhost:6060/debug/pprof/
// http://localhost:6060/debug/pprof/heap
// http://localhost:6060/debug/pprof/goroutine
```

## Code Review Checklist

### Before Submitting

- [ ] Code follows Go conventions (gofmt, golint)
- [ ] All tests pass
- [ ] No race conditions (go test -race)
- [ ] Error handling is proper
- [ ] Comments explain "why", not "what"
- [ ] No TODO comments without issue references
- [ ] Performance considerations addressed
- [ ] Security implications considered
- [ ] Documentation updated

### During Review

- [ ] Code is readable and maintainable
- [ ] Naming is clear and consistent
- [ ] Functions are small and focused
- [ ] No premature optimization
- [ ] Proper use of interfaces
- [ ] Concurrency is safe
- [ ] Resources are properly cleaned up
- [ ] Edge cases are handled

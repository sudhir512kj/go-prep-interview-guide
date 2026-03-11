# Testing in Golang

## Unit Testing

### Basic Test
```go
// math.go
package math

func Add(a, b int) int {
    return a + b
}

// math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    
    if result != expected {
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}
```

### Table-Driven Tests
```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"mixed numbers", -2, 3, 1},
        {"zeros", 0, 0, 0},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; want %d", 
                    tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

### Subtests
```go
func TestMath(t *testing.T) {
    t.Run("Addition", func(t *testing.T) {
        if Add(2, 3) != 5 {
            t.Error("Addition failed")
        }
    })
    
    t.Run("Subtraction", func(t *testing.T) {
        if Subtract(5, 3) != 2 {
            t.Error("Subtraction failed")
        }
    })
}
```

### Test Helpers
```go
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if got != want {
        t.Errorf("got %v; want %v", got, want)
    }
}

func TestWithHelper(t *testing.T) {
    assertEqual(t, Add(2, 3), 5)
    assertEqual(t, Add(10, 20), 30)
}
```

## Testing with Setup and Teardown

### Test Main
```go
func TestMain(m *testing.M) {
    // Setup
    fmt.Println("Setting up tests")
    
    // Run tests
    code := m.Run()
    
    // Teardown
    fmt.Println("Cleaning up")
    
    os.Exit(code)
}
```

### Setup and Teardown per Test
```go
func setup(t *testing.T) func() {
    t.Log("Setup")
    // Setup code
    
    return func() {
        t.Log("Teardown")
        // Cleanup code
    }
}

func TestWithSetup(t *testing.T) {
    teardown := setup(t)
    defer teardown()
    
    // Test code
}
```

## Mocking

### Interface-Based Mocking
```go
// Interface
type UserRepository interface {
    GetUser(id int) (*User, error)
    SaveUser(user *User) error
}

// Mock implementation
type MockUserRepository struct {
    GetUserFunc  func(id int) (*User, error)
    SaveUserFunc func(user *User) error
}

func (m *MockUserRepository) GetUser(id int) (*User, error) {
    return m.GetUserFunc(id)
}

func (m *MockUserRepository) SaveUser(user *User) error {
    return m.SaveUserFunc(user)
}

// Test
func TestUserService(t *testing.T) {
    mockRepo := &MockUserRepository{
        GetUserFunc: func(id int) (*User, error) {
            return &User{ID: id, Name: "Test User"}, nil
        },
    }
    
    service := NewUserService(mockRepo)
    user, err := service.GetUser(1)
    
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Test User" {
        t.Errorf("got %s; want Test User", user.Name)
    }
}
```

### HTTP Mocking
```go
func TestHTTPClient(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"id": 1, "name": "Test"}`))
    }))
    defer server.Close()
    
    resp, err := http.Get(server.URL)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        t.Errorf("got status %d; want %d", resp.StatusCode, http.StatusOK)
    }
}
```

## Benchmarking

### Basic Benchmark
```go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}
```

### Benchmark with Setup
```go
func BenchmarkComplexOperation(b *testing.B) {
    data := generateTestData(1000)
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        processData(data)
    }
}
```

### Parallel Benchmarks
```go
func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            Add(2, 3)
        }
    })
}
```

### Sub-benchmarks
```go
func BenchmarkMath(b *testing.B) {
    b.Run("Add", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            Add(2, 3)
        }
    })
    
    b.Run("Multiply", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            Multiply(2, 3)
        }
    })
}
```

## Testing with Testify

### Assertions
```go
import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestWithAssert(t *testing.T) {
    result := Add(2, 3)
    assert.Equal(t, 5, result)
    assert.NotNil(t, result)
    assert.True(t, result > 0)
}
```

### Require (Stops on Failure)
```go
import "github.com/stretchr/testify/require"

func TestWithRequire(t *testing.T) {
    user, err := GetUser(1)
    require.NoError(t, err)
    require.NotNil(t, user)
    
    // This won't run if above fails
    assert.Equal(t, "John", user.Name)
}
```

### Mock with Testify
```go
import "github.com/stretchr/testify/mock"

type MockRepository struct {
    mock.Mock
}

func (m *MockRepository) GetUser(id int) (*User, error) {
    args := m.Called(id)
    return args.Get(0).(*User), args.Error(1)
}

func TestWithMock(t *testing.T) {
    mockRepo := new(MockRepository)
    mockRepo.On("GetUser", 1).Return(&User{ID: 1, Name: "Test"}, nil)
    
    service := NewUserService(mockRepo)
    user, err := service.GetUser(1)
    
    assert.NoError(t, err)
    assert.Equal(t, "Test", user.Name)
    mockRepo.AssertExpectations(t)
}
```

## Test Coverage

### Running with Coverage
```bash
# Run tests with coverage
go test -cover

# Generate coverage profile
go test -coverprofile=coverage.out

# View coverage in browser
go tool cover -html=coverage.out

# Coverage by function
go tool cover -func=coverage.out
```

### Coverage Example
```go
// calculator.go
package calculator

func Add(a, b int) int {
    return a + b
}

func Divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// calculator_test.go
func TestAdd(t *testing.T) {
    assert.Equal(t, 5, Add(2, 3))
}

func TestDivide(t *testing.T) {
    result, err := Divide(10, 2)
    assert.NoError(t, err)
    assert.Equal(t, 5, result)
    
    _, err = Divide(10, 0)
    assert.Error(t, err)
}
```

## Integration Testing

### Database Testing
```go
func setupTestDB(t *testing.T) *sql.DB {
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("failed to open db: %v", err)
    }
    
    // Create schema
    _, err = db.Exec(`CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT)`)
    if err != nil {
        t.Fatalf("failed to create table: %v", err)
    }
    
    return db
}

func TestUserRepository(t *testing.T) {
    db := setupTestDB(t)
    defer db.Close()
    
    repo := NewUserRepository(db)
    
    // Test insert
    user := &User{Name: "John"}
    err := repo.SaveUser(user)
    assert.NoError(t, err)
    
    // Test retrieve
    retrieved, err := repo.GetUser(user.ID)
    assert.NoError(t, err)
    assert.Equal(t, "John", retrieved.Name)
}
```

### API Testing
```go
func TestAPI(t *testing.T) {
    router := setupRouter()
    
    t.Run("GET /users", func(t *testing.T) {
        req := httptest.NewRequest("GET", "/users", nil)
        w := httptest.NewRecorder()
        
        router.ServeHTTP(w, req)
        
        assert.Equal(t, http.StatusOK, w.Code)
        
        var users []User
        err := json.Unmarshal(w.Body.Bytes(), &users)
        assert.NoError(t, err)
        assert.NotEmpty(t, users)
    })
    
    t.Run("POST /users", func(t *testing.T) {
        body := `{"name": "John", "email": "john@example.com"}`
        req := httptest.NewRequest("POST", "/users", strings.NewReader(body))
        req.Header.Set("Content-Type", "application/json")
        w := httptest.NewRecorder()
        
        router.ServeHTTP(w, req)
        
        assert.Equal(t, http.StatusCreated, w.Code)
    })
}
```

## Testing Best Practices

### 1. Test Naming Convention
```go
// Format: Test<FunctionName>_<Scenario>_<ExpectedResult>
func TestAdd_PositiveNumbers_ReturnsSum(t *testing.T) {}
func TestDivide_DivisionByZero_ReturnsError(t *testing.T) {}
```

### 2. Arrange-Act-Assert Pattern
```go
func TestUserService_CreateUser(t *testing.T) {
    // Arrange
    mockRepo := &MockUserRepository{}
    service := NewUserService(mockRepo)
    user := &User{Name: "John"}
    
    // Act
    err := service.CreateUser(user)
    
    // Assert
    assert.NoError(t, err)
    assert.NotZero(t, user.ID)
}
```

### 3. Test Isolation
```go
func TestIsolation(t *testing.T) {
    t.Run("Test1", func(t *testing.T) {
        // Independent test
    })
    
    t.Run("Test2", func(t *testing.T) {
        // Independent test
    })
}
```

### 4. Use Test Fixtures
```go
func getTestUser() *User {
    return &User{
        ID:    1,
        Name:  "Test User",
        Email: "test@example.com",
    }
}

func TestWithFixture(t *testing.T) {
    user := getTestUser()
    // Use user in test
}
```

### 5. Test Error Cases
```go
func TestDivide_ErrorCases(t *testing.T) {
    tests := []struct {
        name      string
        a, b      int
        wantError bool
    }{
        {"valid division", 10, 2, false},
        {"division by zero", 10, 0, true},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            _, err := Divide(tt.a, tt.b)
            if tt.wantError {
                assert.Error(t, err)
            } else {
                assert.NoError(t, err)
            }
        })
    }
}
```

## Running Tests

```bash
# Run all tests
go test ./...

# Run tests in current package
go test

# Run specific test
go test -run TestAdd

# Run with verbose output
go test -v

# Run with coverage
go test -cover

# Run benchmarks
go test -bench=.

# Run parallel tests
go test -parallel 4

# Run with race detector
go test -race

# Run with timeout
go test -timeout 30s
```


## Testing Philosophy in Go

### Test-Driven Development (TDD)

**Red-Green-Refactor Cycle:**
1. **Red**: Write failing test
2. **Green**: Write minimal code to pass
3. **Refactor**: Improve code while keeping tests green

### Test Organization

```
project/
├── pkg/
│   ├── calculator/
│   │   ├── calculator.go
│   │   └── calculator_test.go
│   └── user/
│       ├── user.go
│       ├── user_test.go
│       └── testdata/
│           └── users.json
```

## Advanced Testing Patterns

### Test Fixtures

```go
type testFixture struct {
    db     *sql.DB
    server *httptest.Server
}

func setupFixture(t *testing.T) *testFixture {
    t.Helper()
    
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("failed to open db: %v", err)
    }
    
    // Create schema
    _, err = db.Exec(`CREATE TABLE users (id INTEGER, name TEXT)`)
    if err != nil {
        t.Fatalf("failed to create table: %v", err)
    }
    
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    }))
    
    return &testFixture{
        db:     db,
        server: server,
    }
}

func (f *testFixture) teardown() {
    f.db.Close()
    f.server.Close()
}

func TestWithFixture(t *testing.T) {
    fixture := setupFixture(t)
    defer fixture.teardown()
    
    // Use fixture.db and fixture.server
}
```

### Golden Files

```go
var update = flag.Bool("update", false, "update golden files")

func TestOutput(t *testing.T) {
    output := generateOutput()
    
    goldenFile := "testdata/output.golden"
    
    if *update {
        os.WriteFile(goldenFile, []byte(output), 0644)
    }
    
    expected, err := os.ReadFile(goldenFile)
    if err != nil {
        t.Fatal(err)
    }
    
    if output != string(expected) {
        t.Errorf("output mismatch\ngot:\n%s\nwant:\n%s", output, expected)
    }
}

// Run with: go test -update to update golden files
```

### Parallel Tests

```go
func TestParallel(t *testing.T) {
    tests := []struct {
        name string
        input int
    }{
        {"test1", 1},
        {"test2", 2},
        {"test3", 3},
    }
    
    for _, tt := range tests {
        tt := tt // Capture range variable
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel() // Run in parallel
            
            // Test code
            result := process(tt.input)
            if result == 0 {
                t.Error("unexpected result")
            }
        })
    }
}
```

### Fuzzing (Go 1.18+)

```go
func FuzzReverse(f *testing.F) {
    // Seed corpus
    f.Add("hello")
    f.Add("world")
    
    f.Fuzz(func(t *testing.T, input string) {
        reversed := Reverse(input)
        doubleReversed := Reverse(reversed)
        
        if input != doubleReversed {
            t.Errorf("double reverse failed: %q != %q", input, doubleReversed)
        }
    })
}

// Run with: go test -fuzz=Fuzz
```

## Benchmark Patterns

### Comparative Benchmarks

```go
func BenchmarkAppend(b *testing.B) {
    b.Run("Preallocated", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            s := make([]int, 0, 1000)
            for j := 0; j < 1000; j++ {
                s = append(s, j)
            }
        }
    })
    
    b.Run("NotPreallocated", func(b *testing.B) {
        for i := 0; i < b.N; i++ {
            var s []int
            for j := 0; j < 1000; j++ {
                s = append(s, j)
            }
        }
    })
}
```

### Memory Benchmarks

```go
func BenchmarkMemory(b *testing.B) {
    b.ReportAllocs() // Report allocations
    
    for i := 0; i < b.N; i++ {
        data := make([]byte, 1024)
        _ = data
    }
}
```

### Benchmark Analysis

```bash
# Run benchmarks
go test -bench=. -benchmem

# Compare benchmarks
go test -bench=. -benchmem > old.txt
# Make changes
go test -bench=. -benchmem > new.txt
benchstat old.txt new.txt

# CPU profiling
go test -bench=. -cpuprofile=cpu.prof
go tool pprof cpu.prof

# Memory profiling
go test -bench=. -memprofile=mem.prof
go tool pprof mem.prof
```

## Integration Testing

### Testing with Docker

```go
func TestWithDocker(t *testing.T) {
    if testing.Short() {
        t.Skip("skipping integration test")
    }
    
    // Start Docker container
    cmd := exec.Command("docker", "run", "-d", "-p", "5432:5432", "postgres")
    output, err := cmd.CombinedOutput()
    if err != nil {
        t.Fatalf("failed to start container: %v\n%s", err, output)
    }
    
    containerID := strings.TrimSpace(string(output))
    defer func() {
        exec.Command("docker", "stop", containerID).Run()
        exec.Command("docker", "rm", containerID).Run()
    }()
    
    // Wait for container to be ready
    time.Sleep(5 * time.Second)
    
    // Run tests
}
```

### Environment-Based Tests

```go
func TestProduction(t *testing.T) {
    if os.Getenv("ENV") != "production" {
        t.Skip("skipping production test")
    }
    
    // Production-specific tests
}
```

## Test Coverage Analysis

### Coverage Reports

```bash
# Generate coverage
go test -coverprofile=coverage.out

# View in browser
go tool cover -html=coverage.out

# Coverage by function
go tool cover -func=coverage.out

# Coverage for specific packages
go test -cover ./...

# Detailed coverage
go test -covermode=count -coverprofile=coverage.out
```

### Coverage Directives

```go
// +build integration

package mypackage

// This file only included when: go test -tags=integration
```

## Testing Anti-Patterns

### Don't Test Implementation Details

```go
// Bad: Testing internal state
func TestBad(t *testing.T) {
    c := NewCounter()
    c.Increment()
    
    if c.internalValue != 1 { // Testing private field
        t.Error("wrong internal value")
    }
}

// Good: Testing behavior
func TestGood(t *testing.T) {
    c := NewCounter()
    c.Increment()
    
    if c.Value() != 1 { // Testing public API
        t.Error("wrong value")
    }
}
```

### Avoid Test Interdependence

```go
// Bad: Tests depend on each other
var globalUser *User

func TestCreate(t *testing.T) {
    globalUser = CreateUser("test")
}

func TestUpdate(t *testing.T) {
    UpdateUser(globalUser) // Depends on TestCreate
}

// Good: Independent tests
func TestCreate(t *testing.T) {
    user := CreateUser("test")
    if user == nil {
        t.Error("failed to create user")
    }
}

func TestUpdate(t *testing.T) {
    user := CreateUser("test") // Create own test data
    UpdateUser(user)
}
```

## Test Utilities

### Custom Assertions

```go
func assertEqual(t *testing.T, got, want interface{}) {
    t.Helper()
    if !reflect.DeepEqual(got, want) {
        t.Errorf("\ngot:  %+v\nwant: %+v", got, want)
    }
}

func assertNoError(t *testing.T, err error) {
    t.Helper()
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
}

func assertError(t *testing.T, err error, msg string) {
    t.Helper()
    if err == nil {
        t.Fatal("expected error, got nil")
    }
    if !strings.Contains(err.Error(), msg) {
        t.Errorf("error %q does not contain %q", err.Error(), msg)
    }
}
```

### Test Data Builders

```go
type UserBuilder struct {
    user *User
}

func NewUserBuilder() *UserBuilder {
    return &UserBuilder{
        user: &User{
            Name:  "Default Name",
            Email: "default@example.com",
            Age:   30,
        },
    }
}

func (b *UserBuilder) WithName(name string) *UserBuilder {
    b.user.Name = name
    return b
}

func (b *UserBuilder) WithEmail(email string) *UserBuilder {
    b.user.Email = email
    return b
}

func (b *UserBuilder) Build() *User {
    return b.user
}

// Usage
func TestUser(t *testing.T) {
    user := NewUserBuilder().
        WithName("John").
        WithEmail("john@example.com").
        Build()
    
    // Test with user
}
```

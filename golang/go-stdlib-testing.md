# Standard Library & Testing

## Table of Contents

### Standard Library
- [Q1: What are the most commonly used packages in Go's standard library?](#q1)
- [Q2: How do you handle file operations in Go?](#q2)
- [Q3: How do you work with JSON in Go?](#q3)
- [Q4: How do you make HTTP requests in Go?](#q4)
- [Q5: How do you create an HTTP server in Go?](#q5)

### Testing
- [Q6: How do you write unit tests in Go?](#q6)
- [Q7: What are table-driven tests?](#q7)
- [Q8: How do you use test fixtures and helpers?](#q8)
- [Q9: How do you write benchmarks in Go?](#q9)
- [Q10: What are subtests and how do you use them?](#q10)
- [Q11: How do you mock dependencies in Go tests?](#q11)

---

## Standard Library

<a id="q1"></a>
### Q1: What are the most commonly used packages in Go's standard library?
**Answer:**

| Package | Purpose |
|---------|---------|
| `fmt` | Formatted I/O |
| `os` | Operating system functionality |
| `io` | Basic I/O interfaces |
| `strings` | String manipulation |
| `strconv` | String conversions |
| `time` | Time and duration |
| `net/http` | HTTP client and server |
| `encoding/json` | JSON encoding/decoding |
| `context` | Request-scoped values, cancellation |
| `sync` | Synchronization primitives |
| `log` | Logging |
| `errors` | Error handling |
| `path/filepath` | File path operations |
| `regexp` | Regular expressions |
| `sort` | Sorting slices |

```go
// fmt - formatted printing
fmt.Println("Hello")
fmt.Printf("Name: %s, Age: %d\n", name, age)
s := fmt.Sprintf("Value: %v", value)

// strings
strings.Contains("hello", "ell")      // true
strings.Split("a,b,c", ",")           // ["a", "b", "c"]
strings.Join([]string{"a", "b"}, "-") // "a-b"
strings.TrimSpace("  hello  ")        // "hello"
strings.ReplaceAll("foo", "o", "0")   // "f00"

// strconv
i, _ := strconv.Atoi("42")            // string to int
s := strconv.Itoa(42)                 // int to string
f, _ := strconv.ParseFloat("3.14", 64)

// time
now := time.Now()
later := now.Add(24 * time.Hour)
duration := later.Sub(now)
formatted := now.Format("2006-01-02 15:04:05")  // Reference time!
parsed, _ := time.Parse("2006-01-02", "2024-03-15")

// sort
nums := []int{3, 1, 4, 1, 5}
sort.Ints(nums)                       // [1, 1, 3, 4, 5]
sort.Slice(users, func(i, j int) bool {
    return users[i].Age < users[j].Age
})
```

<a id="q2"></a>
### Q2: How do you handle file operations in Go?
**Answer:**

```go
// Reading entire file
data, err := os.ReadFile("file.txt")
if err != nil {
    log.Fatal(err)
}
fmt.Println(string(data))

// Writing entire file
content := []byte("Hello, World!")
err := os.WriteFile("file.txt", content, 0644)

// Open file for reading
file, err := os.Open("file.txt")  // Read-only
if err != nil {
    log.Fatal(err)
}
defer file.Close()

// Open file for writing (create/truncate)
file, err := os.Create("file.txt")
if err != nil {
    log.Fatal(err)
}
defer file.Close()

// Open with flags
file, err := os.OpenFile("file.txt", 
    os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)

// Buffered reading
scanner := bufio.NewScanner(file)
for scanner.Scan() {
    line := scanner.Text()
    fmt.Println(line)
}
if err := scanner.Err(); err != nil {
    log.Fatal(err)
}

// Buffered writing
writer := bufio.NewWriter(file)
writer.WriteString("Hello\n")
writer.Flush()  // Don't forget!

// io.Copy for efficient copying
src, _ := os.Open("source.txt")
dst, _ := os.Create("dest.txt")
defer src.Close()
defer dst.Close()

written, err := io.Copy(dst, src)

// File info
info, err := os.Stat("file.txt")
fmt.Println(info.Name())    // file.txt
fmt.Println(info.Size())    // bytes
fmt.Println(info.IsDir())   // false
fmt.Println(info.ModTime()) // modification time

// Check if file exists
if _, err := os.Stat("file.txt"); os.IsNotExist(err) {
    fmt.Println("File does not exist")
}

// Directory operations
os.Mkdir("newdir", 0755)
os.MkdirAll("path/to/dir", 0755)  // Creates parents
os.Remove("file.txt")
os.RemoveAll("dir")               // Recursive

// Walk directory
filepath.Walk(".", func(path string, info os.FileInfo, err error) error {
    if err != nil {
        return err
    }
    fmt.Println(path)
    return nil
})
```

<a id="q3"></a>
### Q3: How do you work with JSON in Go?
**Answer:**

```go
// Struct for JSON
type User struct {
    ID        int       `json:"id"`
    Name      string    `json:"name"`
    Email     string    `json:"email,omitempty"`
    CreatedAt time.Time `json:"created_at"`
    Password  string    `json:"-"`  // Ignored
}

// Marshal (struct to JSON)
user := User{ID: 1, Name: "Alice", Email: "alice@example.com"}
data, err := json.Marshal(user)
// {"id":1,"name":"Alice","email":"alice@example.com","created_at":"..."}

// MarshalIndent for pretty printing
data, err := json.MarshalIndent(user, "", "  ")

// Unmarshal (JSON to struct)
jsonStr := `{"id":1,"name":"Alice"}`
var user User
err := json.Unmarshal([]byte(jsonStr), &user)

// Decode from io.Reader (streaming)
resp, _ := http.Get("https://api.example.com/user")
defer resp.Body.Close()

var user User
decoder := json.NewDecoder(resp.Body)
err := decoder.Decode(&user)

// Encode to io.Writer
encoder := json.NewEncoder(os.Stdout)
encoder.SetIndent("", "  ")
encoder.Encode(user)

// Dynamic JSON with map
var data map[string]interface{}
json.Unmarshal([]byte(`{"name":"Alice","age":30}`), &data)
name := data["name"].(string)

// JSON with unknown structure
var raw json.RawMessage
json.Unmarshal([]byte(`{"data":[1,2,3]}`), &struct {
    Data *json.RawMessage `json:"data"`
}{&raw})

// Custom JSON marshaling
type Date struct {
    time.Time
}

func (d Date) MarshalJSON() ([]byte, error) {
    return []byte(`"` + d.Format("2006-01-02") + `"`), nil
}

func (d *Date) UnmarshalJSON(data []byte) error {
    t, err := time.Parse(`"2006-01-02"`, string(data))
    if err != nil {
        return err
    }
    d.Time = t
    return nil
}

// Handling null values
type NullString struct {
    String string
    Valid  bool
}

func (ns *NullString) UnmarshalJSON(data []byte) error {
    if string(data) == "null" {
        ns.Valid = false
        return nil
    }
    ns.Valid = true
    return json.Unmarshal(data, &ns.String)
}

// Validate JSON
func isValidJSON(s string) bool {
    var js json.RawMessage
    return json.Unmarshal([]byte(s), &js) == nil
}
```

<a id="q4"></a>
### Q4: How do you make HTTP requests in Go?
**Answer:**

```go
// Simple GET
resp, err := http.Get("https://api.example.com/users")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

body, _ := io.ReadAll(resp.Body)
fmt.Println(string(body))

// POST with JSON body
user := map[string]string{"name": "Alice", "email": "alice@example.com"}
jsonData, _ := json.Marshal(user)

resp, err := http.Post(
    "https://api.example.com/users",
    "application/json",
    bytes.NewBuffer(jsonData),
)

// Custom request with headers
req, err := http.NewRequest("GET", "https://api.example.com/users", nil)
if err != nil {
    log.Fatal(err)
}

req.Header.Set("Authorization", "Bearer token123")
req.Header.Set("Content-Type", "application/json")

client := &http.Client{}
resp, err := client.Do(req)

// Request with context (timeout/cancellation)
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
resp, err := http.DefaultClient.Do(req)
if err != nil {
    if ctx.Err() == context.DeadlineExceeded {
        log.Println("Request timed out")
    }
}

// Custom HTTP client with timeout
client := &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 10,
        IdleConnTimeout:     90 * time.Second,
    },
}

// Parse response into struct
var users []User
err := json.NewDecoder(resp.Body).Decode(&users)

// Check status code
if resp.StatusCode != http.StatusOK {
    log.Printf("Unexpected status: %d", resp.StatusCode)
}

// Form data POST
resp, err := http.PostForm("https://example.com/form", url.Values{
    "username": {"alice"},
    "password": {"secret"},
})

// Multipart form (file upload)
var buf bytes.Buffer
writer := multipart.NewWriter(&buf)
part, _ := writer.CreateFormFile("file", "document.pdf")
io.Copy(part, fileReader)
writer.Close()

req, _ := http.NewRequest("POST", url, &buf)
req.Header.Set("Content-Type", writer.FormDataContentType())
```

<a id="q5"></a>
### Q5: How do you create an HTTP server in Go?
**Answer:**

```go
// Simple server
func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "Hello, World!")
    })
    
    log.Fatal(http.ListenAndServe(":8080", nil))
}

// Handler interface
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}

// Custom handler
type HelloHandler struct{}

func (h *HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello!")
}

http.Handle("/hello", &HelloHandler{})

// JSON response
func usersHandler(w http.ResponseWriter, r *http.Request) {
    users := []User{{ID: 1, Name: "Alice"}}
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

// Route parameters (without framework)
func userHandler(w http.ResponseWriter, r *http.Request) {
    // URL: /users/123
    id := strings.TrimPrefix(r.URL.Path, "/users/")
    
    // Or use http.ServeMux (Go 1.22+)
    // id := r.PathValue("id")
}

// Different HTTP methods
func userHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        // Handle GET
    case http.MethodPost:
        // Handle POST
    case http.MethodPut:
        // Handle PUT
    case http.MethodDelete:
        // Handle DELETE
    default:
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
    }
}

// Read request body
func createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    defer r.Body.Close()
    
    // Process user...
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}

// Query parameters
func searchHandler(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query().Get("q")
    page := r.URL.Query().Get("page")
}

// Custom server with timeouts
server := &http.Server{
    Addr:         ":8080",
    Handler:      handler,
    ReadTimeout:  15 * time.Second,
    WriteTimeout: 15 * time.Second,
    IdleTimeout:  60 * time.Second,
}

log.Fatal(server.ListenAndServe())

// Middleware
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}

http.Handle("/", loggingMiddleware(http.HandlerFunc(homeHandler)))

// Graceful shutdown
server := &http.Server{Addr: ":8080", Handler: handler}

go func() {
    if err := server.ListenAndServe(); err != http.ErrServerClosed {
        log.Fatal(err)
    }
}()

// Wait for interrupt signal
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
server.Shutdown(ctx)
```

---

## Testing

<a id="q6"></a>
### Q6: How do you write unit tests in Go?
**Answer:**

```go
// File: math.go
package math

func Add(a, b int) int {
    return a + b
}

func Divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// File: math_test.go
package math

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5
    
    if result != expected {
        t.Errorf("Add(2, 3) = %d; want %d", result, expected)
    }
}

func TestDivide(t *testing.T) {
    result, err := Divide(10, 2)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if result != 5.0 {
        t.Errorf("Divide(10, 2) = %f; want 5.0", result)
    }
}

func TestDivideByZero(t *testing.T) {
    _, err := Divide(10, 0)
    if err == nil {
        t.Error("expected error for division by zero")
    }
}

// Run tests
// go test
// go test -v              (verbose)
// go test -run TestAdd    (specific test)
// go test ./...           (all packages)
// go test -cover          (coverage)
// go test -coverprofile=coverage.out
// go tool cover -html=coverage.out

// Test assertions with testify (third-party)
import "github.com/stretchr/testify/assert"

func TestAddWithTestify(t *testing.T) {
    assert.Equal(t, 5, Add(2, 3))
    assert.NotEqual(t, 0, Add(1, 1))
    assert.Nil(t, err)
    assert.NotNil(t, result)
    assert.True(t, ok)
    assert.Contains(t, "hello world", "world")
}
```

<a id="q7"></a>
### Q7: What are table-driven tests?
**Answer:**

Table-driven tests run the same test logic with multiple inputs:

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -1, -2, -3},
        {"mixed numbers", -1, 5, 4},
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

// Table test with error cases
func TestDivide(t *testing.T) {
    tests := []struct {
        name      string
        a, b      float64
        want      float64
        wantErr   bool
    }{
        {"normal division", 10, 2, 5, false},
        {"division by zero", 10, 0, 0, true},
        {"negative result", -10, 2, -5, false},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got, err := Divide(tt.a, tt.b)
            
            if (err != nil) != tt.wantErr {
                t.Errorf("Divide() error = %v, wantErr %v", err, tt.wantErr)
                return
            }
            
            if got != tt.want {
                t.Errorf("Divide() = %v, want %v", got, tt.want)
            }
        })
    }
}

// Map-based table tests (for better test naming)
func TestParse(t *testing.T) {
    tests := map[string]struct {
        input string
        want  int
        err   error
    }{
        "valid integer":   {"42", 42, nil},
        "negative number": {"-10", -10, nil},
        "invalid input":   {"abc", 0, strconv.ErrSyntax},
    }
    
    for name, tc := range tests {
        t.Run(name, func(t *testing.T) {
            got, err := strconv.Atoi(tc.input)
            // assertions...
        })
    }
}
```

<a id="q8"></a>
### Q8: How do you use test fixtures and helpers?
**Answer:**

```go
// Test fixtures with TestMain
func TestMain(m *testing.M) {
    // Setup
    db := setupTestDB()
    
    // Run tests
    code := m.Run()
    
    // Teardown
    db.Close()
    
    os.Exit(code)
}

// Setup/teardown per test
func TestDatabase(t *testing.T) {
    // Setup
    db := setupTestDB()
    defer db.Close()  // Teardown
    
    // Test...
}

// Test helper function
func setupTestDB(t *testing.T) *sql.DB {
    t.Helper()  // Marks this as a helper (better error reporting)
    
    db, err := sql.Open("postgres", "...")
    if err != nil {
        t.Fatalf("failed to connect to test db: %v", err)
    }
    
    // Cleanup function (Go 1.14+)
    t.Cleanup(func() {
        db.Close()
    })
    
    return db
}

func TestUser(t *testing.T) {
    db := setupTestDB(t)  // Cleanup is automatic
    // Test...
}

// Test fixtures from files
func loadFixture(t *testing.T, filename string) []byte {
    t.Helper()
    data, err := os.ReadFile(filepath.Join("testdata", filename))
    if err != nil {
        t.Fatalf("failed to load fixture %s: %v", filename, err)
    }
    return data
}

func TestParseConfig(t *testing.T) {
    data := loadFixture(t, "config.json")
    // Test...
}

// Golden files (expected output)
func TestTemplate(t *testing.T) {
    result := renderTemplate(data)
    
    golden := filepath.Join("testdata", t.Name()+".golden")
    
    if *update {  // Flag to update golden files
        os.WriteFile(golden, []byte(result), 0644)
    }
    
    expected, _ := os.ReadFile(golden)
    if result != string(expected) {
        t.Errorf("output mismatch, got:\n%s", result)
    }
}

// Temporary directory
func TestFileOperation(t *testing.T) {
    tmpDir := t.TempDir()  // Automatically cleaned up
    
    filePath := filepath.Join(tmpDir, "test.txt")
    os.WriteFile(filePath, []byte("test"), 0644)
    
    // Test file operations...
}

// Parallel test helper
func runParallel(t *testing.T, tests map[string]func(t *testing.T)) {
    for name, tc := range tests {
        tc := tc  // Capture range variable
        t.Run(name, func(t *testing.T) {
            t.Parallel()
            tc(t)
        })
    }
}
```

<a id="q9"></a>
### Q9: How do you write benchmarks in Go?
**Answer:**

```go
// File: math_test.go
func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}

// Run benchmarks
// go test -bench=.
// go test -bench=BenchmarkAdd
// go test -bench=. -benchmem          (memory stats)
// go test -bench=. -count=5           (run 5 times)
// go test -bench=. -benchtime=5s      (run for 5 seconds)

// Benchmark with setup
func BenchmarkProcess(b *testing.B) {
    data := generateTestData(1000)
    
    b.ResetTimer()  // Don't count setup time
    
    for i := 0; i < b.N; i++ {
        Process(data)
    }
}

// Sub-benchmarks
func BenchmarkSort(b *testing.B) {
    sizes := []int{100, 1000, 10000}
    
    for _, size := range sizes {
        b.Run(fmt.Sprintf("size=%d", size), func(b *testing.B) {
            data := generateData(size)
            b.ResetTimer()
            
            for i := 0; i < b.N; i++ {
                sort.Ints(data)
            }
        })
    }
}

// Memory allocation benchmark
func BenchmarkConcat(b *testing.B) {
    b.ReportAllocs()  // Report allocations
    
    for i := 0; i < b.N; i++ {
        var s string
        for j := 0; j < 100; j++ {
            s += "x"
        }
    }
}

func BenchmarkConcatBuilder(b *testing.B) {
    b.ReportAllocs()
    
    for i := 0; i < b.N; i++ {
        var sb strings.Builder
        for j := 0; j < 100; j++ {
            sb.WriteString("x")
        }
        _ = sb.String()
    }
}

// Output:
// BenchmarkConcat-8         10000    104521 ns/op    53240 B/op    99 allocs/op
// BenchmarkConcatBuilder-8  500000    2341 ns/op      512 B/op     1 allocs/op

// Parallel benchmark
func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            // Operation to benchmark
        }
    })
}

// Compare benchmarks
// go test -bench=. -count=10 > old.txt
// # make changes
// go test -bench=. -count=10 > new.txt
// benchstat old.txt new.txt
```

<a id="q10"></a>
### Q10: What are subtests and how do you use them?
**Answer:**

```go
// Subtests with t.Run
func TestMath(t *testing.T) {
    t.Run("Add", func(t *testing.T) {
        if Add(2, 3) != 5 {
            t.Error("2+3 should be 5")
        }
    })
    
    t.Run("Subtract", func(t *testing.T) {
        if Subtract(5, 3) != 2 {
            t.Error("5-3 should be 2")
        }
    })
}

// Run specific subtest
// go test -run TestMath/Add

// Parallel subtests
func TestParallel(t *testing.T) {
    tests := []string{"test1", "test2", "test3"}
    
    for _, name := range tests {
        name := name  // Capture range variable
        t.Run(name, func(t *testing.T) {
            t.Parallel()  // Run in parallel
            // Test logic...
            time.Sleep(time.Second)
        })
    }
}

// Nested subtests
func TestAPI(t *testing.T) {
    t.Run("Users", func(t *testing.T) {
        t.Run("Create", func(t *testing.T) {
            // Test user creation
        })
        t.Run("Read", func(t *testing.T) {
            // Test user retrieval
        })
    })
    
    t.Run("Products", func(t *testing.T) {
        t.Run("List", func(t *testing.T) {
            // Test product listing
        })
    })
}

// Subtests with shared setup
func TestDatabase(t *testing.T) {
    db := setupTestDB(t)
    
    t.Run("Insert", func(t *testing.T) {
        // Uses db
    })
    
    t.Run("Query", func(t *testing.T) {
        // Uses db
    })
    
    // Cleanup happens after all subtests
}

// Skip subtest conditionally
func TestFeature(t *testing.T) {
    t.Run("RequiresNetwork", func(t *testing.T) {
        if testing.Short() {
            t.Skip("skipping in short mode")
        }
        // Network test...
    })
}

// go test -short  (skips long tests)
```

<a id="q11"></a>
### Q11: How do you mock dependencies in Go tests?
**Answer:**

```go
// Interface-based mocking
type UserRepository interface {
    GetByID(id int) (*User, error)
    Save(user *User) error
}

// Real implementation
type PostgresUserRepo struct {
    db *sql.DB
}

func (r *PostgresUserRepo) GetByID(id int) (*User, error) {
    // Real database query
}

// Mock implementation
type MockUserRepo struct {
    users map[int]*User
    err   error
}

func (m *MockUserRepo) GetByID(id int) (*User, error) {
    if m.err != nil {
        return nil, m.err
    }
    return m.users[id], nil
}

func (m *MockUserRepo) Save(user *User) error {
    if m.err != nil {
        return m.err
    }
    m.users[user.ID] = user
    return nil
}

// Service using the interface
type UserService struct {
    repo UserRepository
}

func (s *UserService) GetUser(id int) (*User, error) {
    return s.repo.GetByID(id)
}

// Test with mock
func TestUserService_GetUser(t *testing.T) {
    mock := &MockUserRepo{
        users: map[int]*User{
            1: {ID: 1, Name: "Alice"},
        },
    }
    
    service := &UserService{repo: mock}
    
    user, err := service.GetUser(1)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if user.Name != "Alice" {
        t.Errorf("expected Alice, got %s", user.Name)
    }
}

// Test error case
func TestUserService_GetUser_Error(t *testing.T) {
    mock := &MockUserRepo{
        err: errors.New("database error"),
    }
    
    service := &UserService{repo: mock}
    
    _, err := service.GetUser(1)
    if err == nil {
        t.Error("expected error")
    }
}

// Using testify/mock
import "github.com/stretchr/testify/mock"

type MockRepo struct {
    mock.Mock
}

func (m *MockRepo) GetByID(id int) (*User, error) {
    args := m.Called(id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func TestWithTestify(t *testing.T) {
    mockRepo := new(MockRepo)
    mockRepo.On("GetByID", 1).Return(&User{Name: "Alice"}, nil)
    
    service := &UserService{repo: mockRepo}
    user, _ := service.GetUser(1)
    
    assert.Equal(t, "Alice", user.Name)
    mockRepo.AssertExpectations(t)
}

// HTTP mock
func TestHTTPClient(t *testing.T) {
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        json.NewEncoder(w).Encode(User{ID: 1, Name: "Alice"})
    }))
    defer server.Close()
    
    // Use server.URL as base URL for client
    client := NewAPIClient(server.URL)
    user, err := client.GetUser(1)
    
    assert.NoError(t, err)
    assert.Equal(t, "Alice", user.Name)
}
```

---

[← Back to Go Index](README.md)

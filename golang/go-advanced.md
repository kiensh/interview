# Advanced Go Topics

## Table of Contents

### Go Modules
- [Q1: What are Go modules and how do they work?](#q1)
- [Q2: How do you manage dependencies with go.mod?](#q2)
- [Q3: How do you handle versioning in Go modules?](#q3)

### Reflection
- [Q4: What is reflection in Go and when should you use it?](#q4)
- [Q5: How do you use the reflect package?](#q5)

### Generics
- [Q6: What are generics in Go and how do they work?](#q6)
- [Q7: What are type constraints?](#q7)
- [Q8: When should you use generics vs interfaces?](#q8)

### Context
- [Q9: What is the context package and why is it important?](#q9)
- [Q10: How do you use context for cancellation and timeouts?](#q10)
- [Q11: What are best practices for using context?](#q11)

---

## Go Modules

<a id="q1"></a>
### Q1: What are Go modules and how do they work?
**Answer:**

Go modules are the standard way to manage dependencies in Go (since Go 1.11, default since Go 1.16):

```bash
# Initialize a new module
go mod init github.com/username/project

# This creates go.mod file
```

```go
// go.mod file
module github.com/username/project

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    github.com/lib/pq v1.10.9
)

require (
    // Indirect dependencies (auto-managed)
    github.com/bytedance/sonic v1.9.1 // indirect
)
```

```bash
# Common commands
go mod tidy          # Add missing, remove unused dependencies
go mod download      # Download dependencies to cache
go mod verify        # Verify dependencies
go mod graph         # Print dependency graph
go mod why <pkg>     # Explain why package is needed
go mod vendor        # Create vendor directory

# Get specific version
go get github.com/pkg/errors@v0.9.1
go get github.com/pkg/errors@latest

# Update dependencies
go get -u ./...      # Update all
go get -u=patch      # Update patch versions only
```

```go
// go.sum file (auto-generated, commit this!)
// Contains cryptographic checksums for verification
github.com/gin-gonic/gin v1.9.1 h1:4idEAncQnU5cB7BeOkPtxjfCSye0AAm1R0RVIqJ+Jmg=
github.com/gin-gonic/gin v1.9.1/go.mod h1:hPrL...
```

<a id="q2"></a>
### Q2: How do you manage dependencies with go.mod?
**Answer:**

```bash
# Add a dependency
go get github.com/sirupsen/logrus

# Add specific version
go get github.com/sirupsen/logrus@v1.9.0

# Add from a branch
go get github.com/user/repo@branch-name

# Add from a commit
go get github.com/user/repo@abc1234

# Remove unused dependencies
go mod tidy

# Replace a dependency (local development)
go mod edit -replace github.com/original/pkg=../local/pkg

# Replace with fork
go mod edit -replace github.com/original/pkg=github.com/fork/pkg@v1.0.0
```

```go
// go.mod with replace directive
module myproject

go 1.21

require github.com/original/pkg v1.0.0

replace github.com/original/pkg => ../local/pkg
// or
replace github.com/original/pkg => github.com/myfork/pkg v1.0.1
```

```bash
# Exclude a version (avoid buggy release)
go mod edit -exclude github.com/pkg@v1.0.0

# Retract versions (for module authors)
# In go.mod:
retract v1.0.0  // Security vulnerability

# Private modules
export GOPRIVATE=github.com/mycompany/*
# or in .gitconfig:
[url "ssh://git@github.com/"]
    insteadOf = https://github.com/

# Workspace mode (Go 1.18+, multiple modules)
go work init ./module1 ./module2
# Creates go.work file
```

<a id="q3"></a>
### Q3: How do you handle versioning in Go modules?
**Answer:**

Go uses Semantic Versioning (semver):

```
v1.2.3
│ │ └── Patch: Bug fixes, no API changes
│ └──── Minor: New features, backward compatible
└────── Major: Breaking changes
```

```go
// Import paths change for major versions >= 2
import "github.com/user/pkg"      // v0.x or v1.x
import "github.com/user/pkg/v2"   // v2.x
import "github.com/user/pkg/v3"   // v3.x

// Module path in go.mod must match
module github.com/user/pkg/v2

// Pre-release versions
go get github.com/user/pkg@v1.0.0-beta.1
go get github.com/user/pkg@v1.0.0-rc.1

// Pseudo-versions (for commits without tags)
// v0.0.0-20210101120000-abcdef123456
// Format: vX.Y.Z-yyyymmddhhmmss-commitHash
```

```bash
# Minimum Version Selection (MVS)
# Go selects the MINIMUM version that satisfies all requirements
# This ensures reproducible builds

# Example: If A requires pkg@v1.2.0 and B requires pkg@v1.3.0
# Go selects v1.3.0 (minimum that satisfies both)

# Check why a version was selected
go mod why -m github.com/some/pkg
go mod graph | grep pkg
```

```go
// Version querying
go list -m all                    // All dependencies
go list -m -versions github.com/pkg  // Available versions
go list -m -u all                 // Check for updates
```

---

## Reflection

<a id="q4"></a>
### Q4: What is reflection in Go and when should you use it?
**Answer:**

Reflection allows examining and manipulating types at runtime:

**When to use:**
- Implementing generic operations (encoding/json, fmt)
- ORM/database mapping
- Dependency injection
- Testing frameworks

**When NOT to use:**
- When static types work (prefer generics in Go 1.18+)
- Performance-critical code
- Simple type assertions suffice

```go
import "reflect"

// Basic reflection
var x float64 = 3.14

t := reflect.TypeOf(x)    // reflect.Type
v := reflect.ValueOf(x)   // reflect.Value

fmt.Println(t)            // float64
fmt.Println(t.Kind())     // float64
fmt.Println(v)            // 3.14
fmt.Println(v.Float())    // 3.14

// Performance impact
// Reflection is 10-100x slower than direct code
// Use sparingly in hot paths

// Reflect vs Generics
// Before Go 1.18 (reflection)
func PrintSlice(s interface{}) {
    v := reflect.ValueOf(s)
    for i := 0; i < v.Len(); i++ {
        fmt.Println(v.Index(i))
    }
}

// Go 1.18+ (generics - preferred)
func PrintSlice[T any](s []T) {
    for _, v := range s {
        fmt.Println(v)
    }
}
```

<a id="q5"></a>
### Q5: How do you use the reflect package?
**Answer:**

```go
// Inspect struct fields
type User struct {
    ID    int    `json:"id" validate:"required"`
    Name  string `json:"name"`
    Email string `json:"email,omitempty"`
}

func inspectStruct(v interface{}) {
    t := reflect.TypeOf(v)
    
    // Handle pointer
    if t.Kind() == reflect.Ptr {
        t = t.Elem()
    }
    
    for i := 0; i < t.NumField(); i++ {
        field := t.Field(i)
        fmt.Printf("Field: %s, Type: %s, Tag: %s\n",
            field.Name, field.Type, field.Tag.Get("json"))
    }
}

// Get and set values
func modifyStruct(v interface{}) {
    val := reflect.ValueOf(v)
    
    // Must be pointer to modify
    if val.Kind() != reflect.Ptr {
        panic("must pass pointer")
    }
    
    val = val.Elem()  // Dereference
    
    nameField := val.FieldByName("Name")
    if nameField.IsValid() && nameField.CanSet() {
        nameField.SetString("Modified")
    }
}

user := User{ID: 1, Name: "Alice"}
modifyStruct(&user)
fmt.Println(user.Name)  // Modified

// Call methods dynamically
type Calculator struct{}

func (c Calculator) Add(a, b int) int { return a + b }

func callMethod(obj interface{}, method string, args ...interface{}) interface{} {
    v := reflect.ValueOf(obj)
    m := v.MethodByName(method)
    
    if !m.IsValid() {
        panic("method not found")
    }
    
    in := make([]reflect.Value, len(args))
    for i, arg := range args {
        in[i] = reflect.ValueOf(arg)
    }
    
    result := m.Call(in)
    if len(result) > 0 {
        return result[0].Interface()
    }
    return nil
}

calc := Calculator{}
result := callMethod(calc, "Add", 2, 3)
fmt.Println(result)  // 5

// Create instances dynamically
func createInstance(t reflect.Type) interface{} {
    return reflect.New(t).Interface()
}

userType := reflect.TypeOf(User{})
newUser := createInstance(userType).(*User)

// Iterate map
func printMap(m interface{}) {
    v := reflect.ValueOf(m)
    for _, key := range v.MapKeys() {
        value := v.MapIndex(key)
        fmt.Printf("%v: %v\n", key, value)
    }
}

// Check if implements interface
var _ fmt.Stringer = (*User)(nil)  // Compile-time check

// Runtime check
stringerType := reflect.TypeOf((*fmt.Stringer)(nil)).Elem()
userType := reflect.TypeOf(&User{})
fmt.Println(userType.Implements(stringerType))  // true/false
```

---

## Generics

<a id="q6"></a>
### Q6: What are generics in Go and how do they work?
**Answer:**

Generics (Go 1.18+) allow writing functions and types that work with any type:

```go
// Generic function
func Min[T constraints.Ordered](a, b T) T {
    if a < b {
        return a
    }
    return b
}

// Usage
Min(3, 5)           // T inferred as int
Min(3.14, 2.71)     // T inferred as float64
Min("a", "b")       // T inferred as string

// Generic type
type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item, true
}

// Usage
intStack := Stack[int]{}
intStack.Push(1)
intStack.Push(2)
v, _ := intStack.Pop()  // 2

stringStack := Stack[string]{}
stringStack.Push("hello")

// Multiple type parameters
func Map[T, U any](slice []T, f func(T) U) []U {
    result := make([]U, len(slice))
    for i, v := range slice {
        result[i] = f(v)
    }
    return result
}

doubled := Map([]int{1, 2, 3}, func(x int) int { return x * 2 })
// [2, 4, 6]

lengths := Map([]string{"a", "bb", "ccc"}, func(s string) int { return len(s) })
// [1, 2, 3]

// Generic map type
type Map[K comparable, V any] struct {
    data map[K]V
}

func NewMap[K comparable, V any]() *Map[K, V] {
    return &Map[K, V]{data: make(map[K]V)}
}
```

<a id="q7"></a>
### Q7: What are type constraints?
**Answer:**

Constraints limit which types can be used with generics:

```go
import "golang.org/x/exp/constraints"

// Built-in constraints
// any       - any type (alias for interface{})
// comparable - types that support == and !=

// constraints package
// constraints.Ordered   - types that support < > <= >=
// constraints.Integer   - all integer types
// constraints.Float     - float32, float64
// constraints.Complex   - complex64, complex128
// constraints.Signed    - signed integers
// constraints.Unsigned  - unsigned integers

// Using Ordered constraint
func Max[T constraints.Ordered](values ...T) T {
    if len(values) == 0 {
        var zero T
        return zero
    }
    max := values[0]
    for _, v := range values[1:] {
        if v > max {
            max = v
        }
    }
    return max
}

// Custom constraint with interface
type Number interface {
    int | int32 | int64 | float32 | float64
}

func Sum[T Number](values []T) T {
    var sum T
    for _, v := range values {
        sum += v
    }
    return sum
}

// Constraint with methods
type Stringer interface {
    String() string
}

func PrintAll[T Stringer](items []T) {
    for _, item := range items {
        fmt.Println(item.String())
    }
}

// Combining constraints
type OrderedStringer interface {
    constraints.Ordered
    fmt.Stringer
}

// Approximation constraint (~)
type MyInt int

type Integer interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64
}

// ~int includes int and any type with int as underlying type
func Double[T Integer](v T) T {
    return v * 2
}

var x MyInt = 5
Double(x)  // Works! MyInt has underlying type int

// Without ~
type StrictInt interface {
    int  // Only exact int type
}

// Constraint with underlying type and method
type Addable interface {
    ~int | ~float64
    Add(other any) any
}
```

<a id="q8"></a>
### Q8: When should you use generics vs interfaces?
**Answer:**

| Use Generics When | Use Interfaces When |
|-------------------|---------------------|
| Same algorithm for different types | Different implementations of same behavior |
| Type safety at compile time | Runtime polymorphism needed |
| Avoid type assertions | Behavior is the contract, not data |
| Data structures (Stack, Queue) | Abstracting implementations (Reader, Writer) |
| Reduce code duplication for types | Dependency injection |

```go
// Generics: Same operation on different types
func Contains[T comparable](slice []T, item T) bool {
    for _, v := range slice {
        if v == item {
            return true
        }
    }
    return false
}

// Works with any comparable type
Contains([]int{1, 2, 3}, 2)           // true
Contains([]string{"a", "b"}, "c")     // false

// Interface: Different implementations
type Storage interface {
    Save(key string, data []byte) error
    Load(key string) ([]byte, error)
}

type FileStorage struct{}
type S3Storage struct{}
type RedisStorage struct{}

// Each implements Storage differently
func (f *FileStorage) Save(key string, data []byte) error { /* file ops */ }
func (s *S3Storage) Save(key string, data []byte) error { /* S3 API */ }
func (r *RedisStorage) Save(key string, data []byte) error { /* Redis */ }

// Use interface for dependency injection
type Service struct {
    storage Storage
}

func NewService(s Storage) *Service {
    return &Service{storage: s}
}

// Generics + Interface combination
type Repository[T any] interface {
    Get(id string) (T, error)
    Save(item T) error
    Delete(id string) error
}

type UserRepo struct{}

func (r *UserRepo) Get(id string) (User, error) { /* ... */ }
func (r *UserRepo) Save(user User) error { /* ... */ }
func (r *UserRepo) Delete(id string) error { /* ... */ }

// Generic function using interface constraint
type Validator interface {
    Validate() error
}

func ValidateAll[T Validator](items []T) error {
    for _, item := range items {
        if err := item.Validate(); err != nil {
            return err
        }
    }
    return nil
}
```

---

## Context

<a id="q9"></a>
### Q9: What is the context package and why is it important?
**Answer:**

Context provides request-scoped values, cancellation signals, and deadlines:

```go
import "context"

// Why context is important:
// 1. Propagate cancellation across goroutines
// 2. Set request deadlines/timeouts
// 3. Pass request-scoped data (user ID, trace ID)

// Creating contexts
ctx := context.Background()  // Root context, never canceled
ctx := context.TODO()        // Placeholder when unsure

// Context with cancel
ctx, cancel := context.WithCancel(context.Background())
defer cancel()  // Always call cancel to release resources

// Context with timeout
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// Context with deadline
deadline := time.Now().Add(10 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()

// Context with value
type contextKey string
const userIDKey contextKey = "userID"

ctx := context.WithValue(context.Background(), userIDKey, "user123")
userID := ctx.Value(userIDKey).(string)

// HTTP request context
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()  // Request context
    
    // Propagate to database, API calls
    user, err := db.GetUser(ctx, userID)
}
```

<a id="q10"></a>
### Q10: How do you use context for cancellation and timeouts?
**Answer:**

```go
// Cancellation
func worker(ctx context.Context) error {
    for {
        select {
        case <-ctx.Done():
            return ctx.Err()  // Canceled or DeadlineExceeded
        default:
            // Do work
            time.Sleep(100 * time.Millisecond)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    
    go func() {
        time.Sleep(2 * time.Second)
        cancel()  // Signal cancellation
    }()
    
    err := worker(ctx)
    fmt.Println(err)  // context canceled
}

// Timeout for HTTP request
func fetchData(ctx context.Context, url string) ([]byte, error) {
    ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()
    
    req, err := http.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return nil, err
    }
    
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        if ctx.Err() == context.DeadlineExceeded {
            return nil, fmt.Errorf("request timed out")
        }
        return nil, err
    }
    defer resp.Body.Close()
    
    return io.ReadAll(resp.Body)
}

// Timeout for database query
func getUser(ctx context.Context, id int) (*User, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    var user User
    err := db.QueryRowContext(ctx, 
        "SELECT * FROM users WHERE id = $1", id).Scan(&user)
    
    if err == context.DeadlineExceeded {
        return nil, fmt.Errorf("database query timed out")
    }
    return &user, err
}

// Multiple goroutines with shared cancellation
func processItems(ctx context.Context, items []Item) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    
    errCh := make(chan error, len(items))
    
    for _, item := range items {
        go func(item Item) {
            err := processItem(ctx, item)
            if err != nil {
                cancel()  // Cancel others on first error
            }
            errCh <- err
        }(item)
    }
    
    // Collect results
    var firstErr error
    for range items {
        if err := <-errCh; err != nil && firstErr == nil {
            firstErr = err
        }
    }
    return firstErr
}

// Check context before expensive operations
func expensiveOperation(ctx context.Context) error {
    // Check early
    if ctx.Err() != nil {
        return ctx.Err()
    }
    
    // Step 1
    doStep1()
    
    // Check between steps
    if ctx.Err() != nil {
        return ctx.Err()
    }
    
    // Step 2
    doStep2()
    
    return nil
}
```

<a id="q11"></a>
### Q11: What are best practices for using context?
**Answer:**

```go
// 1. Pass context as first parameter
func ProcessRequest(ctx context.Context, req *Request) error {
    // Good: ctx is first parameter
}

// 2. Don't store context in structs
// Bad
type Handler struct {
    ctx context.Context  // Don't do this!
}

// Good - pass context to methods
type Handler struct{}

func (h *Handler) Handle(ctx context.Context) error {
    return nil
}

// 3. Use context.Background() at entry points
func main() {
    ctx := context.Background()
    server.Run(ctx)
}

func TestSomething(t *testing.T) {
    ctx := context.Background()
    // ...
}

// 4. Use context.TODO() when unsure
func oldFunction() {
    // TODO: Add context support
    ctx := context.TODO()
    newFunction(ctx)
}

// 5. Don't pass nil context
// Bad
func bad() {
    doSomething(nil)  // Don't do this
}

// Good
func good() {
    doSomething(context.Background())
}

// 6. Context values: use for request-scoped data only
// Good use cases:
// - Request ID, Trace ID
// - User authentication
// - Locale/timezone

// Bad use cases:
// - Database connections (use dependency injection)
// - Configuration (use function parameters)
// - Logger (debatable, but often DI is better)

// 7. Use custom types for context keys
type contextKey string

const (
    requestIDKey contextKey = "requestID"
    userKey      contextKey = "user"
)

// Prevents collisions with other packages
ctx := context.WithValue(ctx, requestIDKey, "abc123")

// 8. Provide accessor functions for values
func GetRequestID(ctx context.Context) string {
    if id, ok := ctx.Value(requestIDKey).(string); ok {
        return id
    }
    return ""
}

func GetUser(ctx context.Context) (*User, bool) {
    user, ok := ctx.Value(userKey).(*User)
    return user, ok
}

// 9. Always call cancel function
ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()  // Always defer cancel!

// 10. Context is immutable - each With* creates new context
ctx := context.Background()
ctx = context.WithValue(ctx, key1, val1)
ctx = context.WithValue(ctx, key2, val2)  // New context
ctx, cancel := context.WithTimeout(ctx, time.Second)  // New context
```

---

[← Back to Go Index](README.md)

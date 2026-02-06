# Types, Interfaces & Error Handling

## Table of Contents

### Structs
- [Q1: How do you define and use structs in Go?](#q1)
- [Q2: What is struct embedding and how does it work?](#q2)
- [Q3: What are struct tags and how are they used?](#q3)

### Interfaces
- [Q4: What are interfaces in Go and how do they work?](#q4)
- [Q5: What is the empty interface (interface{}) and when should you use it?](#q5)
- [Q6: What are type assertions and type switches?](#q6)
- [Q7: How do you check if a type implements an interface?](#q7)

### Error Handling
- [Q8: How does error handling work in Go?](#q8)
- [Q9: How do you create custom error types?](#q9)
- [Q10: What is error wrapping and how do you use it?](#q10)
- [Q11: What are sentinel errors and when should you use them?](#q11)
- [Q12: What is the difference between errors.Is and errors.As?](#q12)

---

## Structs

<a id="q1"></a>
### Q1: How do you define and use structs in Go?
**Answer:**

Structs are typed collections of fields:

```go
// Define a struct
type Person struct {
    Name    string
    Age     int
    Email   string
}

// Create struct instances
// Method 1: Zero value
var p1 Person  // {Name: "", Age: 0, Email: ""}

// Method 2: Struct literal
p2 := Person{
    Name:  "Alice",
    Age:   30,
    Email: "alice@example.com",
}

// Method 3: Positional (not recommended - fragile)
p3 := Person{"Bob", 25, "bob@example.com"}

// Method 4: Using new (returns pointer)
p4 := new(Person)
p4.Name = "Charlie"

// Method 5: Pointer with literal
p5 := &Person{
    Name: "Diana",
    Age:  28,
}

// Access fields
fmt.Println(p2.Name)  // Alice
p2.Age = 31           // Modify field

// Pointer access (auto-dereferenced)
fmt.Println(p5.Name)  // Diana (same as (*p5).Name)

// Anonymous struct
point := struct {
    X, Y int
}{10, 20}

// Struct comparison
p6 := Person{Name: "Alice", Age: 30, Email: "alice@example.com"}
fmt.Println(p2 == p6)  // true (if all fields are comparable)

// Structs with uncomparable fields cannot use ==
type Bad struct {
    Data []int  // Slice is not comparable
}
// b1 == b2  // Compile error
```

<a id="q2"></a>
### Q2: What is struct embedding and how does it work?
**Answer:**

Embedding allows composing structs by including one struct type within another:

```go
// Base struct
type Address struct {
    Street  string
    City    string
    Country string
}

// Embedding (composition, not inheritance)
type Employee struct {
    Name    string
    Address // Embedded - no field name
    Salary  float64
}

func main() {
    emp := Employee{
        Name: "Alice",
        Address: Address{
            Street:  "123 Main St",
            City:    "NYC",
            Country: "USA",
        },
        Salary: 75000,
    }

    // Access embedded fields directly (promoted)
    fmt.Println(emp.City)     // NYC (promoted from Address)
    fmt.Println(emp.Address.City)  // NYC (explicit)
}

// Method promotion
func (a Address) FullAddress() string {
    return fmt.Sprintf("%s, %s, %s", a.Street, a.City, a.Country)
}

// Employee gets FullAddress() method automatically
fmt.Println(emp.FullAddress())

// Embedding interfaces
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// Composed interface
type ReadWriter interface {
    Reader
    Writer
}

// Embedding pointers
type Manager struct {
    *Employee  // Embedded pointer
    Department string
}

mgr := Manager{
    Employee:   &emp,
    Department: "Engineering",
}
fmt.Println(mgr.Name)  // Alice (promoted through pointer)

// Name collision - outer field shadows embedded
type Person struct {
    Name string
    Address
}

type Address struct {
    Name string  // Address has Name too
}

p := Person{Name: "Alice", Address: Address{Name: "Home"}}
fmt.Println(p.Name)          // Alice (Person.Name)
fmt.Println(p.Address.Name)  // Home (explicit access)
```

<a id="q3"></a>
### Q3: What are struct tags and how are they used?
**Answer:**

Struct tags are string metadata attached to struct fields, readable via reflection:

```go
type User struct {
    ID        int    `json:"id" db:"user_id"`
    Name      string `json:"name" db:"user_name" validate:"required"`
    Email     string `json:"email,omitempty" validate:"required,email"`
    Password  string `json:"-" db:"password_hash"`  // "-" means ignore
    CreatedAt time.Time `json:"created_at" db:"created_at"`
}

// JSON serialization
user := User{
    ID:    1,
    Name:  "Alice",
    Email: "alice@example.com",
}

data, _ := json.Marshal(user)
// {"id":1,"name":"Alice","email":"alice@example.com","created_at":"..."}
// Note: Password is excluded due to `json:"-"`

// JSON with omitempty
type Response struct {
    Data  interface{} `json:"data,omitempty"`
    Error string      `json:"error,omitempty"`
}

r := Response{Error: "not found"}
json.Marshal(r)  // {"error":"not found"} - data omitted

// Reading tags via reflection
t := reflect.TypeOf(User{})
field, _ := t.FieldByName("Email")
fmt.Println(field.Tag.Get("json"))     // email,omitempty
fmt.Println(field.Tag.Get("validate")) // required,email

// Common tag conventions
type Example struct {
    // JSON
    Field1 string `json:"field_name"`      // Custom name
    Field2 string `json:"field,omitempty"` // Omit if empty
    Field3 string `json:"-"`               // Always ignore
    
    // Database (GORM, sqlx)
    Field4 int `db:"column_name" gorm:"primaryKey"`
    
    // Validation (go-playground/validator)
    Field5 string `validate:"required,min=3,max=100"`
    Field6 string `validate:"email"`
    Field7 int    `validate:"gte=0,lte=100"`
    
    // Form binding (Gin)
    Field8 string `form:"username" binding:"required"`
}

// Custom tag parsing
func parseTag(tag string) map[string]string {
    result := make(map[string]string)
    for _, part := range strings.Split(tag, ",") {
        kv := strings.SplitN(part, "=", 2)
        if len(kv) == 2 {
            result[kv[0]] = kv[1]
        } else {
            result[kv[0]] = ""
        }
    }
    return result
}
```

---

## Interfaces

<a id="q4"></a>
### Q4: What are interfaces in Go and how do they work?
**Answer:**

Interfaces define behavior through method signatures. Types implement interfaces implicitly:

```go
// Define interface
type Writer interface {
    Write(data []byte) (int, error)
}

type Closer interface {
    Close() error
}

// Composed interface
type WriteCloser interface {
    Writer
    Closer
}

// Implicit implementation - no "implements" keyword
type FileWriter struct {
    filename string
}

func (f *FileWriter) Write(data []byte) (int, error) {
    // Implementation
    return len(data), nil
}

func (f *FileWriter) Close() error {
    return nil
}

// FileWriter implements Writer, Closer, and WriteCloser automatically

// Using interface as parameter
func SaveData(w Writer, data []byte) error {
    _, err := w.Write(data)
    return err
}

// Works with any Writer
fw := &FileWriter{filename: "test.txt"}
SaveData(fw, []byte("hello"))

// Standard library example
var buf bytes.Buffer  // implements io.Writer
fmt.Fprintf(&buf, "Hello %s", "World")

// Interface values have (type, value) pair
var w Writer = &FileWriter{}
fmt.Printf("%T, %v\n", w, w)  // *main.FileWriter, &{...}

// Nil interface vs nil concrete value
var w Writer            // nil interface (type=nil, value=nil)
fmt.Println(w == nil)   // true

var fw *FileWriter = nil
w = fw                   // non-nil interface (type=*FileWriter, value=nil)
fmt.Println(w == nil)    // false! Interface is not nil

// Check for nil value inside interface
if w != nil {
    if v := reflect.ValueOf(w); v.Kind() == reflect.Ptr && v.IsNil() {
        // Handle nil pointer in interface
    }
}
```

<a id="q5"></a>
### Q5: What is the empty interface (interface{}) and when should you use it?
**Answer:**

The empty interface `interface{}` (or `any` in Go 1.18+) has no methods, so all types implement it:

```go
// Empty interface can hold any value
var x interface{}
x = 42
x = "hello"
x = []int{1, 2, 3}
x = struct{ Name string }{"Alice"}

// Common use cases

// 1. Generic containers (before generics)
type Stack struct {
    items []interface{}
}

func (s *Stack) Push(item interface{}) {
    s.items = append(s.items, item)
}

func (s *Stack) Pop() interface{} {
    if len(s.items) == 0 {
        return nil
    }
    item := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return item
}

// 2. JSON with unknown structure
var data interface{}
json.Unmarshal([]byte(`{"name": "Alice", "age": 30}`), &data)
// data is map[string]interface{}

m := data.(map[string]interface{})
fmt.Println(m["name"])  // Alice

// 3. Variadic functions accepting any type
func Printf(format string, args ...interface{}) {
    // fmt.Printf implementation
}

// 4. Reflection-based code
func PrintType(v interface{}) {
    fmt.Printf("Type: %T, Value: %v\n", v, v)
}

// Go 1.18+: Use 'any' alias
var y any = "hello"  // Same as interface{}

// When to avoid interface{}
// - When type is known at compile time
// - When generics can be used (Go 1.18+)
// - Performance-critical code (requires type assertions)

// With generics (preferred in Go 1.18+)
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
```

<a id="q6"></a>
### Q6: What are type assertions and type switches?
**Answer:**

Type assertions extract the concrete type from an interface value:

```go
// Type assertion
var i interface{} = "hello"

// Single-value assertion (panics if wrong type)
s := i.(string)
fmt.Println(s)  // hello

// n := i.(int)  // PANIC: interface conversion error

// Two-value assertion (safe)
s, ok := i.(string)
if ok {
    fmt.Println("String:", s)
}

n, ok := i.(int)
if !ok {
    fmt.Println("Not an int")  // This prints
}

// Type switch
func describe(i interface{}) string {
    switch v := i.(type) {
    case nil:
        return "nil"
    case int:
        return fmt.Sprintf("int: %d", v)
    case string:
        return fmt.Sprintf("string: %s", v)
    case bool:
        return fmt.Sprintf("bool: %t", v)
    case []int:
        return fmt.Sprintf("[]int with %d elements", len(v))
    case fmt.Stringer:
        return fmt.Sprintf("Stringer: %s", v.String())
    default:
        return fmt.Sprintf("unknown type: %T", v)
    }
}

// Multiple types in one case
switch v := i.(type) {
case int, int32, int64:
    fmt.Println("Some integer type")
case string, []byte:
    fmt.Println("String-like type")
}

// Interface type assertion
type Reader interface {
    Read([]byte) (int, error)
}

type ReadCloser interface {
    Reader
    Close() error
}

func process(r Reader) {
    // Check if r also implements Close
    if rc, ok := r.(ReadCloser); ok {
        defer rc.Close()
    }
    // Use r...
}

// Practical example: error type checking
func handleError(err error) {
    switch e := err.(type) {
    case *os.PathError:
        fmt.Println("Path error:", e.Path)
    case *json.SyntaxError:
        fmt.Println("JSON syntax error at:", e.Offset)
    case net.Error:
        if e.Timeout() {
            fmt.Println("Network timeout")
        }
    default:
        fmt.Println("Unknown error:", err)
    }
}
```

<a id="q7"></a>
### Q7: How do you check if a type implements an interface?
**Answer:**

```go
// Compile-time check using blank identifier
type Writer interface {
    Write([]byte) (int, error)
}

type MyWriter struct{}

func (m *MyWriter) Write(data []byte) (int, error) {
    return len(data), nil
}

// This line fails to compile if MyWriter doesn't implement Writer
var _ Writer = (*MyWriter)(nil)
var _ Writer = &MyWriter{}  // Also works

// Common pattern in Go packages
var _ io.Reader = (*MyReader)(nil)
var _ io.Writer = (*MyWriter)(nil)
var _ fmt.Stringer = (*MyType)(nil)

// Runtime check using type assertion
func implementsWriter(v interface{}) bool {
    _, ok := v.(Writer)
    return ok
}

// Using reflect
func implementsInterface(v interface{}, interfaceType reflect.Type) bool {
    return reflect.TypeOf(v).Implements(interfaceType)
}

writerType := reflect.TypeOf((*Writer)(nil)).Elem()
fmt.Println(implementsInterface(&MyWriter{}, writerType))  // true

// Check method existence at runtime
func hasMethod(v interface{}, methodName string) bool {
    t := reflect.TypeOf(v)
    _, found := t.MethodByName(methodName)
    return found
}

// Example: standard library pattern
// io package uses this pattern extensively
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface {
    Close() error
}

type ReadWriter interface {
    Reader
    Writer
}

type ReadCloser interface {
    Reader
    Closer
}

type WriteCloser interface {
    Writer
    Closer
}

type ReadWriteCloser interface {
    Reader
    Writer
    Closer
}
```

---

## Error Handling

<a id="q8"></a>
### Q8: How does error handling work in Go?
**Answer:**

Go uses explicit error returns instead of exceptions:

```go
// Error is an interface
type error interface {
    Error() string
}

// Functions return error as last value
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Calling and checking
result, err := divide(10, 0)
if err != nil {
    log.Fatal(err)  // Handle error
}
fmt.Println(result)

// Common patterns

// Pattern 1: Early return on error
func process(data []byte) error {
    result, err := step1(data)
    if err != nil {
        return err
    }
    
    transformed, err := step2(result)
    if err != nil {
        return err
    }
    
    return step3(transformed)
}

// Pattern 2: Error with context
func readConfig(path string) (*Config, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("reading config %s: %w", path, err)
    }
    
    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("parsing config: %w", err)
    }
    
    return &config, nil
}

// Pattern 3: Defer with error handling
func writeFile(path string, data []byte) (err error) {
    f, err := os.Create(path)
    if err != nil {
        return err
    }
    defer func() {
        closeErr := f.Close()
        if err == nil {
            err = closeErr  // Only set if no prior error
        }
    }()
    
    _, err = f.Write(data)
    return err
}

// Pattern 4: Error accumulation
func validate(user User) error {
    var errs []string
    
    if user.Name == "" {
        errs = append(errs, "name is required")
    }
    if user.Age < 0 {
        errs = append(errs, "age must be positive")
    }
    
    if len(errs) > 0 {
        return errors.New(strings.Join(errs, "; "))
    }
    return nil
}
```

<a id="q9"></a>
### Q9: How do you create custom error types?
**Answer:**

```go
// Simple custom error with errors.New or fmt.Errorf
var ErrNotFound = errors.New("not found")
err := fmt.Errorf("user %d not found", userID)

// Custom error type
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error on %s: %s", e.Field, e.Message)
}

func validateEmail(email string) error {
    if !strings.Contains(email, "@") {
        return &ValidationError{
            Field:   "email",
            Message: "invalid format",
        }
    }
    return nil
}

// Error with additional methods
type HTTPError struct {
    StatusCode int
    Message    string
}

func (e *HTTPError) Error() string {
    return fmt.Sprintf("HTTP %d: %s", e.StatusCode, e.Message)
}

func (e *HTTPError) IsClientError() bool {
    return e.StatusCode >= 400 && e.StatusCode < 500
}

func (e *HTTPError) IsServerError() bool {
    return e.StatusCode >= 500
}

// Using custom error
func fetchUser(id int) (*User, error) {
    resp, err := http.Get(fmt.Sprintf("/users/%d", id))
    if err != nil {
        return nil, err
    }
    
    if resp.StatusCode == 404 {
        return nil, &HTTPError{404, "user not found"}
    }
    if resp.StatusCode != 200 {
        return nil, &HTTPError{resp.StatusCode, "unexpected status"}
    }
    
    // Parse response...
    return user, nil
}

// Check custom error type
user, err := fetchUser(123)
if err != nil {
    var httpErr *HTTPError
    if errors.As(err, &httpErr) {
        if httpErr.StatusCode == 404 {
            // Handle not found
        }
    }
}

// Error with Unwrap (for wrapping)
type OperationError struct {
    Op  string
    Err error
}

func (e *OperationError) Error() string {
    return fmt.Sprintf("%s: %v", e.Op, e.Err)
}

func (e *OperationError) Unwrap() error {
    return e.Err
}
```

<a id="q10"></a>
### Q10: What is error wrapping and how do you use it?
**Answer:**

Error wrapping adds context while preserving the original error:

```go
// Wrap with fmt.Errorf and %w verb (Go 1.13+)
func readFile(path string) ([]byte, error) {
    data, err := os.ReadFile(path)
    if err != nil {
        return nil, fmt.Errorf("reading %s: %w", path, err)
    }
    return data, nil
}

func loadConfig(path string) (*Config, error) {
    data, err := readFile(path)
    if err != nil {
        return nil, fmt.Errorf("loading config: %w", err)
    }
    
    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("parsing config: %w", err)
    }
    return &config, nil
}

// Error chain
// loadConfig -> readFile -> os.ReadFile
// "loading config: reading /path: open /path: no such file"

// Unwrap manually
err := loadConfig("/nonexistent")
fmt.Println(err)
// loading config: reading /nonexistent: open /nonexistent: no such file

unwrapped := errors.Unwrap(err)
fmt.Println(unwrapped)
// reading /nonexistent: open /nonexistent: no such file

// Full unwrap chain
for err != nil {
    fmt.Println(err)
    err = errors.Unwrap(err)
}

// Custom error with Unwrap
type DatabaseError struct {
    Query string
    Err   error
}

func (e *DatabaseError) Error() string {
    return fmt.Sprintf("database error in query %q: %v", e.Query, e.Err)
}

func (e *DatabaseError) Unwrap() error {
    return e.Err
}

// Multiple wrapped errors (Go 1.20+)
err := fmt.Errorf("multiple errors: %w, %w", err1, err2)

// Or use errors.Join
err := errors.Join(err1, err2, err3)
```

<a id="q11"></a>
### Q11: What are sentinel errors and when should you use them?
**Answer:**

Sentinel errors are predefined error values for specific conditions:

```go
// Standard library sentinel errors
var (
    io.EOF
    sql.ErrNoRows
    context.Canceled
    context.DeadlineExceeded
)

// Define your own sentinel errors
var (
    ErrNotFound     = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrInvalidInput = errors.New("invalid input")
)

// Usage
func GetUser(id int) (*User, error) {
    user, found := users[id]
    if !found {
        return nil, ErrNotFound
    }
    return user, nil
}

// Checking sentinel errors
user, err := GetUser(123)
if err != nil {
    if errors.Is(err, ErrNotFound) {
        // Handle not found
        return nil
    }
    // Handle other errors
    return err
}

// Sentinel errors work with wrapping
func GetUserByEmail(email string) (*User, error) {
    user, err := findUser(email)
    if err != nil {
        return nil, fmt.Errorf("GetUserByEmail: %w", err)
    }
    return user, nil
}

err := GetUserByEmail("test@example.com")
if errors.Is(err, ErrNotFound) {  // Still works!
    fmt.Println("User not found")
}

// When to use sentinel errors
// Good:
// - Well-defined, expected conditions
// - Part of public API contract
// - Need to be checked by callers

// Bad:
// - Errors with dynamic content
// - Internal implementation details
// - Errors that need additional context

// Package-level errors (exported)
package user

var (
    ErrNotFound     = errors.New("user: not found")
    ErrDuplicate    = errors.New("user: duplicate")
    ErrInvalidEmail = errors.New("user: invalid email")
)
```

<a id="q12"></a>
### Q12: What is the difference between errors.Is and errors.As?
**Answer:**

| `errors.Is` | `errors.As` |
|-------------|-------------|
| Checks error identity/equality | Checks error type |
| Returns `bool` | Returns `bool`, sets target |
| For sentinel errors | For custom error types |
| Uses `Is()` method if defined | Uses `As()` method if defined |

```go
// errors.Is - check for specific error value
var ErrNotFound = errors.New("not found")

err := fmt.Errorf("user query: %w", ErrNotFound)

if errors.Is(err, ErrNotFound) {
    fmt.Println("Not found!")  // This prints
}

// Works through the error chain
err = fmt.Errorf("outer: %w", fmt.Errorf("inner: %w", ErrNotFound))
errors.Is(err, ErrNotFound)  // true

// Custom Is method
type MyError struct {
    Code int
}

func (e *MyError) Error() string {
    return fmt.Sprintf("error code %d", e.Code)
}

func (e *MyError) Is(target error) bool {
    t, ok := target.(*MyError)
    if !ok {
        return false
    }
    return e.Code == t.Code
}

// errors.As - extract error of specific type
type QueryError struct {
    Query string
    Err   error
}

func (e *QueryError) Error() string {
    return fmt.Sprintf("query %s: %v", e.Query, e.Err)
}

func (e *QueryError) Unwrap() error {
    return e.Err
}

err := &QueryError{Query: "SELECT *", Err: sql.ErrNoRows}
wrappedErr := fmt.Errorf("database: %w", err)

var qe *QueryError
if errors.As(wrappedErr, &qe) {
    fmt.Println("Query was:", qe.Query)  // SELECT *
}

// Check multiple error types
func handleError(err error) {
    var pathErr *os.PathError
    var netErr net.Error
    var queryErr *QueryError
    
    switch {
    case errors.As(err, &pathErr):
        fmt.Println("File operation failed:", pathErr.Path)
    case errors.As(err, &netErr):
        if netErr.Timeout() {
            fmt.Println("Network timeout")
        }
    case errors.As(err, &queryErr):
        fmt.Println("Query failed:", queryErr.Query)
    case errors.Is(err, context.Canceled):
        fmt.Println("Operation canceled")
    default:
        fmt.Println("Unknown error:", err)
    }
}

// Common mistakes
// WRONG: comparing error strings
if err.Error() == "not found" { }  // Fragile!

// WRONG: type assertion without errors.As
if e, ok := err.(*QueryError); ok { }  // Doesn't check wrapped errors

// CORRECT: use errors.Is for sentinel errors
if errors.Is(err, ErrNotFound) { }

// CORRECT: use errors.As for type checking
var qe *QueryError
if errors.As(err, &qe) { }
```

---

[← Back to Go Index](README.md)

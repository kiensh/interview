# Go Basics

## Table of Contents

### Data Types & Variables
- [Q1: What are the basic data types in Go?](#q1)
- [Q2: What is the difference between var and := declaration?](#q2)
- [Q3: What is the zero value concept in Go?](#q3)
- [Q4: What is the difference between constants and variables?](#q4)

### Control Structures
- [Q5: How do loops work in Go?](#q5)
- [Q6: How does the switch statement differ from other languages?](#q6)

### Functions
- [Q7: What are the different ways to define functions in Go?](#q7)
- [Q8: What are variadic functions?](#q8)
- [Q9: What is a closure in Go?](#q9)
- [Q10: What is the difference between call by value and call by reference?](#q10)

### Pointers & Memory
- [Q11: What are pointers in Go and how do you use them?](#q11)
- [Q12: What is the difference between new() and make()?](#q12)
- [Q13: When should you use pointers vs values?](#q13)

### Packages
- [Q14: How do packages and imports work in Go?](#q14)

---

## Data Types & Variables

<a id="q1"></a>
### Q1: What are the basic data types in Go?
**Answer:**

Go has several built-in data types:

| Category | Types | Description |
|----------|-------|-------------|
| **Boolean** | `bool` | `true` or `false` |
| **Numeric** | `int`, `int8`, `int16`, `int32`, `int64` | Signed integers |
| | `uint`, `uint8`, `uint16`, `uint32`, `uint64` | Unsigned integers |
| | `float32`, `float64` | Floating point |
| | `complex64`, `complex128` | Complex numbers |
| **String** | `string` | Immutable sequence of bytes |
| **Byte/Rune** | `byte` (alias for `uint8`), `rune` (alias for `int32`) | Character types |

```go
// Numeric types
var i int = 42
var f float64 = 3.14
var c complex128 = 1 + 2i

// String and characters
var s string = "Hello, 世界"
var b byte = 'A'      // uint8
var r rune = '世'     // int32 (Unicode code point)

// Boolean
var flag bool = true

// Type inference
age := 25           // int
price := 19.99      // float64
name := "Go"        // string
```

**Note:** `int` and `uint` are platform-dependent (32 or 64 bits).

<a id="q2"></a>
### Q2: What is the difference between var and := declaration?
**Answer:**

| `var` declaration | `:=` short declaration |
|-------------------|------------------------|
| Can be used anywhere | Only inside functions |
| Type can be explicit or inferred | Type is always inferred |
| Can declare without initialization | Must initialize |
| Used for package-level variables | Only for local variables |

```go
// var declaration
var name string              // Zero value: ""
var age int = 25             // Explicit type
var price = 19.99            // Type inferred (float64)
var x, y int = 1, 2          // Multiple variables

// Short declaration (:=)
name := "Alice"              // Only inside functions
age, height := 25, 175       // Multiple variables
count := 0                   // Type inferred

// Package level - only var allowed
var GlobalConfig string      // OK
// globalCount := 0          // ERROR: not allowed at package level

// Redeclaration with :=
a, b := 1, 2
b, c := 3, 4   // OK: at least one new variable (c)
// a, b := 5, 6  // ERROR: no new variables
```

<a id="q3"></a>
### Q3: What is the zero value concept in Go?
**Answer:**

In Go, variables declared without explicit initialization get their **zero value**:

| Type | Zero Value |
|------|------------|
| `bool` | `false` |
| Numeric (`int`, `float`, etc.) | `0` |
| `string` | `""` (empty string) |
| Pointers, slices, maps, channels, functions, interfaces | `nil` |
| Arrays | Array of zero values |
| Structs | Struct with all fields set to zero values |

```go
var b bool       // false
var i int        // 0
var f float64    // 0.0
var s string     // ""
var p *int       // nil
var sl []int     // nil (but len=0, cap=0)
var m map[string]int  // nil
var ch chan int  // nil

// Struct zero value
type Person struct {
    Name string
    Age  int
}
var person Person  // {Name: "", Age: 0}

// This is useful for safe defaults
func process(items []int) {
    // nil slice works fine with len(), range
    for _, item := range items {  // Works even if items is nil
        fmt.Println(item)
    }
}
```

<a id="q4"></a>
### Q4: What is the difference between constants and variables?
**Answer:**

| Constants (`const`) | Variables (`var`) |
|---------------------|-------------------|
| Value set at compile time | Value can change at runtime |
| Cannot use `:=` | Can use `var` or `:=` |
| Must be primitive types | Can be any type |
| No address (cannot use `&`) | Has memory address |

```go
// Constants
const Pi = 3.14159
const (
    StatusOK    = 200
    StatusError = 500
)

// Typed vs untyped constants
const typedInt int = 100      // Typed constant
const untypedInt = 100        // Untyped constant (more flexible)

// iota - constant generator
const (
    Sunday = iota    // 0
    Monday           // 1
    Tuesday          // 2
)

const (
    _  = iota             // 0 (ignore)
    KB = 1 << (10 * iota) // 1 << 10 = 1024
    MB                    // 1 << 20
    GB                    // 1 << 30
)

// Untyped constants are flexible
const value = 100
var i int = value       // OK
var f float64 = value   // OK
var b byte = value      // OK

// Cannot do this with variables
var v = 100
// var f2 float64 = v   // ERROR: cannot use v (type int)
```

---

## Control Structures

<a id="q5"></a>
### Q5: How do loops work in Go?
**Answer:**

Go has only one looping construct: `for`. It can be used in multiple ways:

```go
// Traditional for loop
for i := 0; i < 10; i++ {
    fmt.Println(i)
}

// While-style loop
count := 0
for count < 10 {
    count++
}

// Infinite loop
for {
    // Use break to exit
    if condition {
        break
    }
}

// Range over slice
nums := []int{1, 2, 3, 4, 5}
for index, value := range nums {
    fmt.Printf("Index: %d, Value: %d\n", index, value)
}

// Range over map
m := map[string]int{"a": 1, "b": 2}
for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}

// Range over string (iterates over runes)
for i, r := range "Go语言" {
    fmt.Printf("%d: %c\n", i, r)
}
// Output: 0: G, 1: o, 2: 语, 5: 言 (note byte positions)

// Ignore index or value
for _, v := range nums { }  // Ignore index
for i := range nums { }     // Ignore value

// Break and continue with labels
outer:
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if j == 1 {
            continue outer  // Continue outer loop
        }
        if i == 2 {
            break outer     // Break out of both loops
        }
    }
}
```

<a id="q6"></a>
### Q6: How does the switch statement differ from other languages?
**Answer:**

Go's switch has several differences from C/Java:

```go
// No automatic fallthrough (no break needed)
switch day {
case "Monday":
    fmt.Println("Start of week")
case "Friday":
    fmt.Println("TGIF!")
case "Saturday", "Sunday":  // Multiple values
    fmt.Println("Weekend!")
default:
    fmt.Println("Midweek")
}

// Explicit fallthrough
switch num {
case 1:
    fmt.Println("One")
    fallthrough  // Continues to next case
case 2:
    fmt.Println("Two or fell through from One")
}

// Switch with no expression (like if-else chain)
score := 85
switch {
case score >= 90:
    fmt.Println("A")
case score >= 80:
    fmt.Println("B")
case score >= 70:
    fmt.Println("C")
default:
    fmt.Println("F")
}

// Switch with initialization
switch os := runtime.GOOS; os {
case "darwin":
    fmt.Println("macOS")
case "linux":
    fmt.Println("Linux")
default:
    fmt.Println(os)
}

// Type switch
func describe(i interface{}) {
    switch v := i.(type) {
    case int:
        fmt.Printf("Integer: %d\n", v)
    case string:
        fmt.Printf("String: %s\n", v)
    case bool:
        fmt.Printf("Boolean: %t\n", v)
    default:
        fmt.Printf("Unknown type: %T\n", v)
    }
}
```

---

## Functions

<a id="q7"></a>
### Q7: What are the different ways to define functions in Go?
**Answer:**

```go
// Basic function
func greet(name string) string {
    return "Hello, " + name
}

// Multiple return values
func divide(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

// Named return values
func split(sum int) (x, y int) {
    x = sum * 4 / 9
    y = sum - x
    return  // Naked return (returns x, y)
}

// Function as a type
type Operation func(int, int) int

func add(a, b int) int { return a + b }
func sub(a, b int) int { return a - b }

func calculate(op Operation, a, b int) int {
    return op(a, b)
}

// Anonymous function
func main() {
    // Assigned to variable
    double := func(x int) int {
        return x * 2
    }
    fmt.Println(double(5))  // 10

    // Immediately invoked
    result := func(a, b int) int {
        return a + b
    }(3, 4)
    fmt.Println(result)  // 7
}

// Methods (function with receiver)
type Rectangle struct {
    Width, Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}

func (r *Rectangle) Scale(factor float64) {
    r.Width *= factor
    r.Height *= factor
}
```

<a id="q8"></a>
### Q8: What are variadic functions?
**Answer:**

Variadic functions accept a variable number of arguments:

```go
// Variadic function
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

// Usage
sum(1, 2, 3)        // 6
sum(1, 2, 3, 4, 5)  // 15
sum()               // 0

// Passing slice to variadic function
numbers := []int{1, 2, 3, 4}
sum(numbers...)     // Spread operator

// Mixed parameters (variadic must be last)
func printf(format string, args ...interface{}) {
    // format is required, args are optional
}

// Common use: fmt.Println
fmt.Println("a", "b", "c")  // Accepts any number of args

// Variadic with different types using interface{}
func printAll(args ...interface{}) {
    for _, arg := range args {
        fmt.Printf("%v ", arg)
    }
}
printAll(1, "hello", true, 3.14)
```

<a id="q9"></a>
### Q9: What is a closure in Go?
**Answer:**

A closure is a function that references variables from outside its body. The function "closes over" these variables.

```go
// Basic closure
func counter() func() int {
    count := 0  // Enclosed variable
    return func() int {
        count++  // References outer variable
        return count
    }
}

func main() {
    c := counter()
    fmt.Println(c())  // 1
    fmt.Println(c())  // 2
    fmt.Println(c())  // 3

    c2 := counter()   // New closure, new state
    fmt.Println(c2()) // 1
}

// Closure with parameters
func multiplier(factor int) func(int) int {
    return func(x int) int {
        return x * factor  // Closes over 'factor'
    }
}

double := multiplier(2)
triple := multiplier(3)
fmt.Println(double(5))  // 10
fmt.Println(triple(5))  // 15

// Common pitfall: loop variable capture
func main() {
    funcs := make([]func(), 3)
    
    // WRONG: all closures share same 'i'
    for i := 0; i < 3; i++ {
        funcs[i] = func() { fmt.Println(i) }
    }
    for _, f := range funcs {
        f()  // Prints: 3, 3, 3
    }

    // CORRECT: capture loop variable
    for i := 0; i < 3; i++ {
        i := i  // Shadow with new variable
        funcs[i] = func() { fmt.Println(i) }
    }
    // Or pass as parameter
    for i := 0; i < 3; i++ {
        funcs[i] = func(n int) func() {
            return func() { fmt.Println(n) }
        }(i)
    }
}
```

<a id="q10"></a>
### Q10: What is the difference between call by value and call by reference?
**Answer:**

Go is **always pass by value**, but the value can be a pointer:

```go
// Pass by value (copy)
func double(x int) {
    x = x * 2  // Modifies copy, not original
}

func main() {
    n := 10
    double(n)
    fmt.Println(n)  // Still 10
}

// Pass pointer (value of pointer is copied)
func doublePtr(x *int) {
    *x = *x * 2  // Modifies value at pointer address
}

func main() {
    n := 10
    doublePtr(&n)
    fmt.Println(n)  // 20
}

// Slices, maps, channels - reference types
// The header/descriptor is copied, but underlying data is shared
func appendItem(s []int) {
    s[0] = 100      // Modifies original slice's data
    s = append(s, 4) // Creates new slice, doesn't affect original
}

func main() {
    slice := []int{1, 2, 3}
    appendItem(slice)
    fmt.Println(slice)  // [100 2 3] - first element changed
                        // But append didn't affect original
}

// To modify slice itself, use pointer
func appendItemPtr(s *[]int) {
    *s = append(*s, 4)
}

func main() {
    slice := []int{1, 2, 3}
    appendItemPtr(&slice)
    fmt.Println(slice)  // [1 2 3 4]
}
```

| Type | Behavior |
|------|----------|
| Basic types (int, string, bool) | Copied |
| Arrays | Copied (entire array!) |
| Structs | Copied |
| Slices | Header copied, data shared |
| Maps | Reference copied, data shared |
| Channels | Reference copied |
| Pointers | Pointer value copied |

---

## Pointers & Memory

<a id="q11"></a>
### Q11: What are pointers in Go and how do you use them?
**Answer:**

A pointer holds the memory address of a value:

```go
// Declaring pointers
var p *int          // Pointer to int, zero value is nil
x := 42
p = &x              // & gets address of x
fmt.Println(*p)     // * dereferences pointer: 42

// Modifying through pointer
*p = 100
fmt.Println(x)      // 100

// Pointer to struct
type Person struct {
    Name string
    Age  int
}

p := &Person{Name: "Alice", Age: 30}
// These are equivalent:
fmt.Println((*p).Name)  // Explicit dereference
fmt.Println(p.Name)     // Auto-dereference (syntactic sugar)

// No pointer arithmetic in Go (unlike C)
// p++  // ERROR: not allowed

// Nil pointer check
var ptr *int
if ptr != nil {
    fmt.Println(*ptr)  // Safe to dereference
}

// Function returning pointer
func newInt(x int) *int {
    return &x  // Safe! Go uses escape analysis
}

// Common pattern: pointer receiver
func (p *Person) Birthday() {
    p.Age++  // Modifies the original struct
}
```

<a id="q12"></a>
### Q12: What is the difference between new() and make()?
**Answer:**

| `new(T)` | `make(T, args)` |
|----------|-----------------|
| Allocates zeroed memory | Allocates and initializes |
| Returns `*T` (pointer) | Returns `T` (value) |
| Works with any type | Only for slice, map, channel |
| Memory is zeroed | Internal structure initialized |

```go
// new() - returns pointer to zeroed memory
p := new(int)        // *int pointing to 0
fmt.Println(*p)      // 0

s := new([]int)      // *[]int pointing to nil slice
fmt.Println(*s)      // [] (nil slice)
fmt.Println(s == nil) // false (pointer is not nil)

// make() - creates initialized slice, map, or channel
slice := make([]int, 5)       // len=5, cap=5, all zeros
slice2 := make([]int, 0, 10)  // len=0, cap=10

m := make(map[string]int)     // Empty but initialized map
m["key"] = 1                  // OK

ch := make(chan int)          // Unbuffered channel
ch2 := make(chan int, 10)     // Buffered channel

// Comparison
var s1 []int         // nil slice
s2 := make([]int, 0) // Empty but non-nil slice

fmt.Println(s1 == nil)  // true
fmt.Println(s2 == nil)  // false
fmt.Println(len(s1))    // 0 (nil slice has len 0)
fmt.Println(len(s2))    // 0

// Both work with append
s1 = append(s1, 1)  // OK
s2 = append(s2, 1)  // OK

// But JSON encoding differs
json.Marshal(s1)  // null
json.Marshal(s2)  // []
```

<a id="q13"></a>
### Q13: When should you use pointers vs values?
**Answer:**

**Use pointers when:**
- You need to modify the receiver/parameter
- The struct is large (avoid copying)
- Consistency (if some methods need pointer receiver, use for all)
- Working with sync primitives (Mutex, WaitGroup)

**Use values when:**
- The type is small (primitives, small structs)
- The type is immutable (like time.Time)
- The value semantics make sense
- Thread safety is important (copies are safer)

```go
// Small struct - value receiver is fine
type Point struct {
    X, Y int
}

func (p Point) Distance() float64 {
    return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}

// Large struct or needs modification - use pointer
type LargeStruct struct {
    Data [1000]int
    // ... many fields
}

func (l *LargeStruct) Process() {
    // Pointer avoids copying 1000 ints
}

func (l *LargeStruct) Update(val int) {
    l.Data[0] = val  // Can modify
}

// sync types MUST use pointer
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Inc() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// Slices, maps - usually don't need pointer
func process(items []int) {
    // Slice header is small, underlying array is shared
}

// Unless you need to modify the slice itself
func appendAndReturn(items *[]int, val int) {
    *items = append(*items, val)
}
```

---

## Packages

<a id="q14"></a>
### Q14: How do packages and imports work in Go?
**Answer:**

```go
// Package declaration (first line of every Go file)
package mypackage

// Main package is special - entry point
package main

func main() {
    // Program starts here
}

// Import single package
import "fmt"

// Import multiple packages
import (
    "fmt"
    "strings"
    "net/http"
)

// Import with alias
import (
    f "fmt"                    // Alias
    _ "github.com/lib/pq"      // Blank import (side effects only)
    . "strings"                // Dot import (not recommended)
)

f.Println("Hello")  // Using alias
_ = Contains("abc", "a")  // Dot import - no prefix needed

// Exported vs unexported
// Uppercase = exported (public)
// Lowercase = unexported (private to package)
type User struct {
    Name string  // Exported
    age  int     // Unexported
}

func PublicFunc() {}   // Exported
func privateFunc() {}  // Unexported

// init() function - runs before main()
func init() {
    // Initialization code
    // Multiple init() allowed per file
    // Order: imported packages' init() first
}

// Internal packages
// myproject/internal/... can only be imported by myproject

// Package organization
// myproject/
// ├── main.go           (package main)
// ├── go.mod
// ├── handlers/
// │   └── user.go       (package handlers)
// ├── models/
// │   └── user.go       (package models)
// └── internal/
//     └── utils/
//         └── helper.go (package utils)
```

---

[← Back to Go Index](README.md)

# Concurrency in Go

## Table of Contents

### Goroutines
- [Q1: What is a goroutine and how does it differ from a thread?](#q1)
- [Q2: How do you start a goroutine?](#q2)
- [Q3: What happens when the main goroutine exits?](#q3)

### Channels
- [Q4: What are channels and how do they work?](#q4)
- [Q5: What is the difference between buffered and unbuffered channels?](#q5)
- [Q6: How do you close a channel and why is it important?](#q6)
- [Q7: What is a nil channel and when would you use it?](#q7)

### Select Statement
- [Q8: How does the select statement work?](#q8)
- [Q9: How do you implement timeouts with select?](#q9)

### Synchronization
- [Q10: What is sync.Mutex and when should you use it?](#q10)
- [Q11: What is the difference between Mutex and RWMutex?](#q11)
- [Q12: What is sync.WaitGroup and how do you use it?](#q12)
- [Q13: What is sync.Once and when would you use it?](#q13)

### Race Conditions
- [Q14: What is a race condition and how do you detect it?](#q14)

---

## Goroutines

<a id="q1"></a>
### Q1: What is a goroutine and how does it differ from a thread?
**Answer:**

A goroutine is a lightweight thread of execution managed by the Go runtime.

| Goroutine | OS Thread |
|-----------|-----------|
| ~2KB initial stack (grows as needed) | ~1-2MB fixed stack |
| Managed by Go runtime scheduler | Managed by OS |
| M:N scheduling (many goroutines : few threads) | 1:1 mapping |
| Context switch is fast (~200ns) | Context switch is slow (~1-2μs) |
| Can run millions concurrently | Limited by OS (thousands) |
| Cooperative scheduling | Preemptive scheduling |

```go
// Goroutine vs Thread memory comparison
// 1 million goroutines ≈ 2GB memory
// 1 million threads ≈ 1-2TB memory (impossible)

func main() {
    // Start 100,000 goroutines easily
    for i := 0; i < 100000; i++ {
        go func(n int) {
            time.Sleep(time.Second)
        }(i)
    }
    time.Sleep(2 * time.Second)
}
```

**Go Scheduler (GMP Model):**
- **G** (Goroutine): The goroutine itself
- **M** (Machine): OS thread
- **P** (Processor): Logical processor, runs goroutines on M

```
P0 ──── M0 (OS Thread)
  └── G1, G2, G3 (local run queue)

P1 ──── M1 (OS Thread)
  └── G4, G5 (local run queue)

Global Run Queue: G6, G7, G8...
```

<a id="q2"></a>
### Q2: How do you start a goroutine?
**Answer:**

Use the `go` keyword before a function call:

```go
// Start goroutine with named function
func sayHello() {
    fmt.Println("Hello")
}

func main() {
    go sayHello()  // Starts goroutine
    time.Sleep(time.Second)  // Wait (bad practice, use sync)
}

// Start goroutine with anonymous function
go func() {
    fmt.Println("Hello from goroutine")
}()

// Pass arguments to goroutine
go func(msg string) {
    fmt.Println(msg)
}("Hello")

// Common mistake: loop variable capture
for i := 0; i < 3; i++ {
    go func() {
        fmt.Println(i)  // WRONG: may print 3, 3, 3
    }()
}

// Correct: pass as parameter
for i := 0; i < 3; i++ {
    go func(n int) {
        fmt.Println(n)  // Correct: prints 0, 1, 2 (order may vary)
    }(i)
}

// Or shadow the variable
for i := 0; i < 3; i++ {
    i := i  // Shadow loop variable
    go func() {
        fmt.Println(i)
    }()
}
```

<a id="q3"></a>
### Q3: What happens when the main goroutine exits?
**Answer:**

When `main()` returns, the program exits immediately, killing all goroutines:

```go
func main() {
    go func() {
        time.Sleep(time.Second)
        fmt.Println("This may never print!")
    }()
    // main exits immediately, goroutine is killed
}

// Solution 1: WaitGroup
func main() {
    var wg sync.WaitGroup
    wg.Add(1)
    
    go func() {
        defer wg.Done()
        time.Sleep(time.Second)
        fmt.Println("This will print")
    }()
    
    wg.Wait()  // Block until Done() is called
}

// Solution 2: Channel
func main() {
    done := make(chan bool)
    
    go func() {
        time.Sleep(time.Second)
        fmt.Println("This will print")
        done <- true
    }()
    
    <-done  // Block until message received
}

// Solution 3: Context (for cancellation)
func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()
    
    go worker(ctx)
    
    <-ctx.Done()  // Wait for timeout or cancel
}

func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return
        default:
            // Do work
        }
    }
}
```

---

## Channels

<a id="q4"></a>
### Q4: What are channels and how do they work?
**Answer:**

Channels are typed conduits for communication between goroutines:

```go
// Create channel
ch := make(chan int)      // Unbuffered channel
ch := make(chan int, 10)  // Buffered channel (capacity 10)

// Send value (blocks until receiver is ready for unbuffered)
ch <- 42

// Receive value (blocks until sender sends)
value := <-ch

// Check if channel is closed
value, ok := <-ch
if !ok {
    fmt.Println("Channel closed")
}

// Example: producer-consumer
func main() {
    ch := make(chan int)
    
    // Producer
    go func() {
        for i := 0; i < 5; i++ {
            ch <- i
        }
        close(ch)
    }()
    
    // Consumer
    for num := range ch {  // Loops until channel closed
        fmt.Println(num)
    }
}

// Direction-specific channels (for function signatures)
func producer(ch chan<- int) {  // Send-only
    ch <- 42
    // <-ch  // ERROR: cannot receive
}

func consumer(ch <-chan int) {  // Receive-only
    val := <-ch
    // ch <- 1  // ERROR: cannot send
}

func main() {
    ch := make(chan int)  // Bidirectional
    go producer(ch)
    go consumer(ch)
}
```

<a id="q5"></a>
### Q5: What is the difference between buffered and unbuffered channels?
**Answer:**

| Unbuffered | Buffered |
|------------|----------|
| `make(chan T)` | `make(chan T, capacity)` |
| Send blocks until receive | Send blocks only when full |
| Receive blocks until send | Receive blocks only when empty |
| Synchronous communication | Asynchronous (up to capacity) |
| Guarantees message delivery order | FIFO order |

```go
// Unbuffered - synchronization point
ch := make(chan int)

go func() {
    fmt.Println("Sending...")
    ch <- 42  // Blocks until receiver ready
    fmt.Println("Sent!")
}()

time.Sleep(time.Second)
fmt.Println("Receiving...")
<-ch  // Unblocks sender
// Output: Sending... (pause) Receiving... Sent!

// Buffered - asynchronous until full
ch := make(chan int, 2)

ch <- 1  // Doesn't block
ch <- 2  // Doesn't block
// ch <- 3  // Would block (buffer full)

fmt.Println(<-ch)  // 1
fmt.Println(<-ch)  // 2

// Use cases
// Unbuffered: strict synchronization, handoff
// Buffered: rate limiting, batch processing, decoupling

// Rate limiter example
limiter := make(chan struct{}, 3)  // Max 3 concurrent

for _, task := range tasks {
    limiter <- struct{}{}  // Acquire slot
    go func(t Task) {
        defer func() { <-limiter }()  // Release slot
        process(t)
    }(task)
}
```

<a id="q6"></a>
### Q6: How do you close a channel and why is it important?
**Answer:**

```go
// Close a channel
ch := make(chan int)
close(ch)

// Only sender should close
// Closing signals "no more values"
// Receiving from closed channel returns zero value immediately

ch := make(chan int)
close(ch)
val := <-ch         // Returns 0 (zero value)
val, ok := <-ch     // Returns 0, false

// Range over channel - stops when closed
ch := make(chan int)
go func() {
    for i := 0; i < 3; i++ {
        ch <- i
    }
    close(ch)  // Important! Otherwise range blocks forever
}()

for v := range ch {
    fmt.Println(v)  // 0, 1, 2
}

// Panic scenarios
close(ch)
// close(ch)  // PANIC: close of closed channel
// ch <- 1    // PANIC: send on closed channel

// Don't close receiver-side
func consumer(ch <-chan int) {
    // close(ch)  // ERROR: cannot close receive-only channel
}

// Multiple receivers - closing broadcasts to all
ch := make(chan struct{})
for i := 0; i < 3; i++ {
    go func(id int) {
        <-ch  // All goroutines unblock when closed
        fmt.Printf("Worker %d started\n", id)
    }(i)
}
close(ch)  // Broadcast start signal

// Use sync.Once if multiple goroutines might close
var once sync.Once
closeCh := func() { close(ch) }
once.Do(closeCh)  // Safe to call multiple times
```

<a id="q7"></a>
### Q7: What is a nil channel and when would you use it?
**Answer:**

A nil channel blocks forever on send/receive. This is useful for disabling cases in select:

```go
// Nil channel behavior
var ch chan int  // nil
// ch <- 1       // Blocks forever
// <-ch          // Blocks forever
// close(ch)     // PANIC

// Use case: disable select case dynamically
func merge(ch1, ch2 <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        for ch1 != nil || ch2 != nil {
            select {
            case v, ok := <-ch1:
                if !ok {
                    ch1 = nil  // Disable this case
                    continue
                }
                out <- v
            case v, ok := <-ch2:
                if !ok {
                    ch2 = nil  // Disable this case
                    continue
                }
                out <- v
            }
        }
    }()
    
    return out
}

// Another example: conditional channel operations
func process(in <-chan int, enableOutput bool) {
    var out chan<- int
    if enableOutput {
        ch := make(chan int)
        out = ch
        go func() {
            for v := range ch {
                fmt.Println(v)
            }
        }()
    }
    // out is nil if enableOutput is false
    
    for v := range in {
        select {
        case out <- v:  // Skipped if out is nil
        default:
        }
    }
}
```

---

## Select Statement

<a id="q8"></a>
### Q8: How does the select statement work?
**Answer:**

`select` waits on multiple channel operations, proceeding with the first one ready:

```go
// Basic select
select {
case msg := <-ch1:
    fmt.Println("Received from ch1:", msg)
case msg := <-ch2:
    fmt.Println("Received from ch2:", msg)
case ch3 <- 42:
    fmt.Println("Sent to ch3")
}

// Default case (non-blocking)
select {
case msg := <-ch:
    fmt.Println("Received:", msg)
default:
    fmt.Println("No message available")
}

// Random selection when multiple ready
ch1 := make(chan int, 1)
ch2 := make(chan int, 1)
ch1 <- 1
ch2 <- 2

select {
case <-ch1:
    fmt.Println("ch1")  // May or may not be selected
case <-ch2:
    fmt.Println("ch2")  // May or may not be selected
}

// Infinite loop with select
for {
    select {
    case msg := <-messages:
        process(msg)
    case <-quit:
        fmt.Println("Shutting down")
        return
    }
}

// Empty select blocks forever
select {}  // Useful to keep main alive
```

<a id="q9"></a>
### Q9: How do you implement timeouts with select?
**Answer:**

```go
// Timeout for single operation
select {
case result := <-ch:
    fmt.Println("Got result:", result)
case <-time.After(3 * time.Second):
    fmt.Println("Timeout!")
}

// Timeout in a loop (be careful with time.After)
// WRONG: creates new timer each iteration (memory leak)
for {
    select {
    case msg := <-ch:
        process(msg)
    case <-time.After(time.Second):  // New timer each loop!
        fmt.Println("Idle")
    }
}

// CORRECT: reuse timer
timeout := time.NewTimer(time.Second)
for {
    select {
    case msg := <-ch:
        if !timeout.Stop() {
            <-timeout.C
        }
        timeout.Reset(time.Second)
        process(msg)
    case <-timeout.C:
        fmt.Println("Idle")
        timeout.Reset(time.Second)
    }
}

// Deadline (overall timeout)
func fetchWithDeadline(url string) (string, error) {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return "", err  // Returns error on timeout
    }
    defer resp.Body.Close()
    
    body, _ := io.ReadAll(resp.Body)
    return string(body), nil
}

// Ticker for periodic operations
ticker := time.NewTicker(time.Second)
defer ticker.Stop()

for {
    select {
    case <-ticker.C:
        fmt.Println("Tick")
    case <-done:
        return
    }
}
```

---

## Synchronization

<a id="q10"></a>
### Q10: What is sync.Mutex and when should you use it?
**Answer:**

`sync.Mutex` provides mutual exclusion for protecting shared data:

```go
// Basic mutex usage
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()  // Always unlock, even on panic
    c.count++
}

func (c *SafeCounter) Value() int {
    c.mu.Lock()
    defer c.mu.Unlock()
    return c.count
}

// Usage
counter := &SafeCounter{}
var wg sync.WaitGroup

for i := 0; i < 1000; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        counter.Increment()
    }()
}

wg.Wait()
fmt.Println(counter.Value())  // 1000

// Common mistake: copying mutex
type BadCounter struct {
    mu    sync.Mutex
    count int
}

func (c BadCounter) Bad() {  // Value receiver copies mutex!
    c.mu.Lock()  // Different mutex than original
    c.count++
    c.mu.Unlock()
}

// Always use pointer receiver with mutex
func (c *BadCounter) Good() {
    c.mu.Lock()
    c.count++
    c.mu.Unlock()
}

// Embedding mutex
type Cache struct {
    sync.Mutex  // Embedded
    data map[string]string
}

func (c *Cache) Get(key string) string {
    c.Lock()  // Can call directly
    defer c.Unlock()
    return c.data[key]
}
```

<a id="q11"></a>
### Q11: What is the difference between Mutex and RWMutex?
**Answer:**

| Mutex | RWMutex |
|-------|---------|
| One lock type | Read lock + Write lock |
| Exclusive access | Multiple readers OR one writer |
| Good for write-heavy | Good for read-heavy |
| Simpler | Better read performance |

```go
type Cache struct {
    mu   sync.RWMutex
    data map[string]string
}

// Read operation - multiple readers allowed
func (c *Cache) Get(key string) string {
    c.mu.RLock()         // Read lock
    defer c.mu.RUnlock()
    return c.data[key]
}

// Write operation - exclusive access
func (c *Cache) Set(key, value string) {
    c.mu.Lock()          // Write lock
    defer c.mu.Unlock()
    c.data[key] = value
}

// Performance comparison
// 90% reads, 10% writes scenario:
// Mutex: all operations serialize
// RWMutex: reads can happen concurrently

// Caution: RLock holders can starve writers
// If reads are constant, writer may never acquire lock
// Go's RWMutex is writer-preferring to prevent this

// Don't upgrade lock (causes deadlock)
func (c *Cache) Bad(key string) {
    c.mu.RLock()
    if c.data[key] == "" {
        // c.mu.Lock()  // DEADLOCK! Already holding RLock
    }
    c.mu.RUnlock()
}

// Correct pattern
func (c *Cache) GetOrSet(key, value string) string {
    c.mu.RLock()
    v := c.data[key]
    c.mu.RUnlock()
    
    if v == "" {
        c.mu.Lock()
        // Double-check (another goroutine may have set it)
        if c.data[key] == "" {
            c.data[key] = value
        }
        v = c.data[key]
        c.mu.Unlock()
    }
    return v
}
```

<a id="q12"></a>
### Q12: What is sync.WaitGroup and how do you use it?
**Answer:**

`sync.WaitGroup` waits for a collection of goroutines to finish:

```go
func main() {
    var wg sync.WaitGroup
    
    for i := 0; i < 5; i++ {
        wg.Add(1)  // Increment counter before goroutine
        go func(id int) {
            defer wg.Done()  // Decrement when done
            fmt.Printf("Worker %d starting\n", id)
            time.Sleep(time.Second)
            fmt.Printf("Worker %d done\n", id)
        }(i)
    }
    
    wg.Wait()  // Block until counter is zero
    fmt.Println("All workers completed")
}

// Common mistakes

// WRONG: Add inside goroutine (race condition)
for i := 0; i < 5; i++ {
    go func(id int) {
        wg.Add(1)  // May not execute before Wait()
        defer wg.Done()
        // work
    }(i)
}
wg.Wait()  // May return immediately!

// WRONG: Forgetting Done()
wg.Add(1)
go func() {
    // forgot defer wg.Done()
    // work
}()
wg.Wait()  // Blocks forever

// CORRECT: Add before, Done with defer
wg.Add(1)
go func() {
    defer wg.Done()
    // work
}()

// Batch add
wg.Add(5)
for i := 0; i < 5; i++ {
    go func(id int) {
        defer wg.Done()
        // work
    }(i)
}
wg.Wait()

// Cannot copy WaitGroup after first use
func bad(wg sync.WaitGroup) {  // Copies WaitGroup!
    wg.Done()  // Wrong WaitGroup
}

func good(wg *sync.WaitGroup) {  // Use pointer
    wg.Done()
}
```

<a id="q13"></a>
### Q13: What is sync.Once and when would you use it?
**Answer:**

`sync.Once` ensures a function is executed only once, even across goroutines:

```go
// Singleton pattern
var (
    instance *Database
    once     sync.Once
)

func GetDatabase() *Database {
    once.Do(func() {
        instance = &Database{}
        instance.Connect()
    })
    return instance
}

// Multiple goroutines - only one initializes
var wg sync.WaitGroup
for i := 0; i < 10; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        db := GetDatabase()  // Only first call initializes
        db.Query("SELECT 1")
    }()
}
wg.Wait()

// Lazy initialization
type Config struct {
    once     sync.Once
    settings map[string]string
}

func (c *Config) Get(key string) string {
    c.once.Do(func() {
        c.settings = loadConfig()  // Expensive operation
    })
    return c.settings[key]
}

// Once.Do is blocking
// All goroutines wait until the first Do() completes
var once sync.Once
var wg sync.WaitGroup

for i := 0; i < 3; i++ {
    wg.Add(1)
    go func(id int) {
        defer wg.Done()
        once.Do(func() {
            fmt.Println("Initializing...")
            time.Sleep(time.Second)
            fmt.Println("Done initializing")
        })
        fmt.Printf("Goroutine %d proceeding\n", id)
    }(i)
}
wg.Wait()
// Output:
// Initializing...
// Done initializing
// Goroutine 0 proceeding
// Goroutine 1 proceeding
// Goroutine 2 proceeding

// Note: Once.Do doesn't return until function completes
// All callers block until initialization is done
```

---

## Race Conditions

<a id="q14"></a>
### Q14: What is a race condition and how do you detect it?
**Answer:**

A race condition occurs when multiple goroutines access shared data and at least one modifies it:

```go
// Race condition example
var counter int

func main() {
    for i := 0; i < 1000; i++ {
        go func() {
            counter++  // READ-MODIFY-WRITE is not atomic
        }()
    }
    time.Sleep(time.Second)
    fmt.Println(counter)  // Not 1000! Could be any value
}

// Detection: go run -race / go test -race
// $ go run -race main.go
// WARNING: DATA RACE
// Read at 0x... by goroutine 7
// Previous write at 0x... by goroutine 6

// Fix 1: Mutex
var (
    counter int
    mu      sync.Mutex
)

func increment() {
    mu.Lock()
    counter++
    mu.Unlock()
}

// Fix 2: Atomic operations
var counter int64

func increment() {
    atomic.AddInt64(&counter, 1)
}

// Fix 3: Channels
func main() {
    counter := 0
    inc := make(chan struct{})
    done := make(chan struct{})
    
    // Single goroutine owns the counter
    go func() {
        for range inc {
            counter++
        }
        done <- struct{}{}
    }()
    
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            inc <- struct{}{}
        }()
    }
    
    wg.Wait()
    close(inc)
    <-done
    fmt.Println(counter)  // Always 1000
}

// Race detector flags
// -race enables race detector
// Increases memory 5-10x, CPU 2-20x
// Use in testing, not production

// go test -race ./...
// go build -race -o myapp

// Common race patterns to avoid
// 1. Shared variable without sync
// 2. Check-then-act (TOCTOU)
if m[key] == nil {
    m[key] = newValue  // Race: another goroutine may set it
}

// 3. Slice/map concurrent access
// Maps are NOT safe for concurrent use
m := make(map[string]int)
go func() { m["a"] = 1 }()
go func() { _ = m["a"] }()  // Race!

// Use sync.Map for concurrent map access
var m sync.Map
m.Store("key", "value")
v, _ := m.Load("key")
```

---

[← Back to Go Index](README.md)

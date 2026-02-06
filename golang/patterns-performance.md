# Design Patterns & Performance

## Table of Contents

### Design Patterns
- [Q1: What are common creational patterns in Go?](#q1)
- [Q2: What are common structural patterns in Go?](#q2)
- [Q3: What are common behavioral patterns in Go?](#q3)

### Concurrency Patterns
- [Q4: What is the Worker Pool pattern?](#q4)
- [Q5: What is Fan-out/Fan-in?](#q5)
- [Q6: What is the Pipeline pattern?](#q6)
- [Q7: What are other common concurrency patterns?](#q7)

### Performance & Profiling
- [Q8: How do you profile Go applications with pprof?](#q8)
- [Q9: How do you optimize memory usage in Go?](#q9)
- [Q10: How do you optimize CPU performance?](#q10)
- [Q11: What are common performance pitfalls in Go?](#q11)

---

## Design Patterns

<a id="q1"></a>
### Q1: What are common creational patterns in Go?
**Answer:**

```go
// SINGLETON - ensure single instance
type Database struct {
    connection string
}

var (
    instance *Database
    once     sync.Once
)

func GetDatabase() *Database {
    once.Do(func() {
        instance = &Database{connection: "connected"}
    })
    return instance
}

// Usage
db1 := GetDatabase()
db2 := GetDatabase()
// db1 == db2 (same instance)

// FACTORY - create objects without specifying exact class
type Storage interface {
    Save(data []byte) error
}

type FileStorage struct{}
type S3Storage struct{}
type MemoryStorage struct{}

func (f *FileStorage) Save(data []byte) error { return nil }
func (s *S3Storage) Save(data []byte) error { return nil }
func (m *MemoryStorage) Save(data []byte) error { return nil }

func NewStorage(storageType string) (Storage, error) {
    switch storageType {
    case "file":
        return &FileStorage{}, nil
    case "s3":
        return &S3Storage{}, nil
    case "memory":
        return &MemoryStorage{}, nil
    default:
        return nil, fmt.Errorf("unknown storage type: %s", storageType)
    }
}

// BUILDER - construct complex objects step by step
type Server struct {
    Host    string
    Port    int
    TLS     bool
    Timeout time.Duration
}

type ServerBuilder struct {
    server Server
}

func NewServerBuilder() *ServerBuilder {
    return &ServerBuilder{
        server: Server{
            Host:    "localhost",
            Port:    8080,
            Timeout: 30 * time.Second,
        },
    }
}

func (b *ServerBuilder) Host(host string) *ServerBuilder {
    b.server.Host = host
    return b
}

func (b *ServerBuilder) Port(port int) *ServerBuilder {
    b.server.Port = port
    return b
}

func (b *ServerBuilder) WithTLS() *ServerBuilder {
    b.server.TLS = true
    return b
}

func (b *ServerBuilder) Timeout(d time.Duration) *ServerBuilder {
    b.server.Timeout = d
    return b
}

func (b *ServerBuilder) Build() Server {
    return b.server
}

// Usage
server := NewServerBuilder().
    Host("api.example.com").
    Port(443).
    WithTLS().
    Timeout(60 * time.Second).
    Build()

// FUNCTIONAL OPTIONS - alternative to builder
type ServerOption func(*Server)

func WithHost(host string) ServerOption {
    return func(s *Server) { s.Host = host }
}

func WithPort(port int) ServerOption {
    return func(s *Server) { s.Port = port }
}

func WithTLSOption() ServerOption {
    return func(s *Server) { s.TLS = true }
}

func NewServer(opts ...ServerOption) *Server {
    s := &Server{
        Host:    "localhost",
        Port:    8080,
        Timeout: 30 * time.Second,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}

// Usage
server := NewServer(
    WithHost("api.example.com"),
    WithPort(443),
    WithTLSOption(),
)
```

<a id="q2"></a>
### Q2: What are common structural patterns in Go?
**Answer:**

```go
// ADAPTER - make incompatible interfaces work together
type OldPrinter interface {
    PrintLegacy(text string)
}

type NewPrinter interface {
    Print(text string)
}

type LegacyPrinter struct{}

func (l *LegacyPrinter) PrintLegacy(text string) {
    fmt.Println("Legacy: " + text)
}

type PrinterAdapter struct {
    oldPrinter OldPrinter
}

func (a *PrinterAdapter) Print(text string) {
    a.oldPrinter.PrintLegacy(text)
}

// Usage
legacy := &LegacyPrinter{}
adapter := &PrinterAdapter{oldPrinter: legacy}
adapter.Print("Hello")  // Works with NewPrinter interface

// DECORATOR - add behavior without modifying original
type Handler interface {
    Handle(ctx context.Context, req Request) Response
}

type BaseHandler struct{}

func (h *BaseHandler) Handle(ctx context.Context, req Request) Response {
    return Response{Body: "OK"}
}

// Logging decorator
type LoggingHandler struct {
    handler Handler
    logger  *log.Logger
}

func (h *LoggingHandler) Handle(ctx context.Context, req Request) Response {
    h.logger.Printf("Request: %v", req)
    resp := h.handler.Handle(ctx, req)
    h.logger.Printf("Response: %v", resp)
    return resp
}

// Metrics decorator
type MetricsHandler struct {
    handler Handler
}

func (h *MetricsHandler) Handle(ctx context.Context, req Request) Response {
    start := time.Now()
    resp := h.handler.Handle(ctx, req)
    duration := time.Since(start)
    recordMetric("request_duration", duration)
    return resp
}

// Stack decorators
handler := &MetricsHandler{
    handler: &LoggingHandler{
        handler: &BaseHandler{},
        logger:  log.Default(),
    },
}

// FACADE - simplified interface to complex subsystem
type OrderFacade struct {
    inventory *InventoryService
    payment   *PaymentService
    shipping  *ShippingService
    notify    *NotificationService
}

func (f *OrderFacade) PlaceOrder(order Order) error {
    // Check inventory
    if !f.inventory.CheckStock(order.Items) {
        return errors.New("out of stock")
    }
    
    // Process payment
    if err := f.payment.Charge(order.Total); err != nil {
        return err
    }
    
    // Ship order
    trackingID, err := f.shipping.Ship(order)
    if err != nil {
        f.payment.Refund(order.Total)  // Rollback
        return err
    }
    
    // Send notification
    f.notify.SendConfirmation(order.Email, trackingID)
    
    return nil
}

// PROXY - control access to another object
type Database interface {
    Query(query string) ([]Row, error)
}

type RealDatabase struct{}

func (d *RealDatabase) Query(query string) ([]Row, error) {
    // Real database query
    return nil, nil
}

type CachingProxy struct {
    db    Database
    cache map[string][]Row
    mu    sync.RWMutex
}

func (p *CachingProxy) Query(query string) ([]Row, error) {
    p.mu.RLock()
    if cached, ok := p.cache[query]; ok {
        p.mu.RUnlock()
        return cached, nil
    }
    p.mu.RUnlock()
    
    result, err := p.db.Query(query)
    if err != nil {
        return nil, err
    }
    
    p.mu.Lock()
    p.cache[query] = result
    p.mu.Unlock()
    
    return result, nil
}
```

<a id="q3"></a>
### Q3: What are common behavioral patterns in Go?
**Answer:**

```go
// STRATEGY - interchangeable algorithms
type SortStrategy interface {
    Sort([]int) []int
}

type QuickSort struct{}
type MergeSort struct{}
type BubbleSort struct{}

func (q *QuickSort) Sort(data []int) []int { /* ... */ return data }
func (m *MergeSort) Sort(data []int) []int { /* ... */ return data }
func (b *BubbleSort) Sort(data []int) []int { /* ... */ return data }

type Sorter struct {
    strategy SortStrategy
}

func (s *Sorter) SetStrategy(strategy SortStrategy) {
    s.strategy = strategy
}

func (s *Sorter) Sort(data []int) []int {
    return s.strategy.Sort(data)
}

// Usage
sorter := &Sorter{strategy: &QuickSort{}}
sorter.Sort(data)
sorter.SetStrategy(&MergeSort{})  // Change algorithm at runtime

// OBSERVER - notify multiple objects of state changes
type Observer interface {
    Update(event Event)
}

type Subject struct {
    observers []Observer
    mu        sync.RWMutex
}

func (s *Subject) Subscribe(o Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.observers = append(s.observers, o)
}

func (s *Subject) Unsubscribe(o Observer) {
    s.mu.Lock()
    defer s.mu.Unlock()
    for i, obs := range s.observers {
        if obs == o {
            s.observers = append(s.observers[:i], s.observers[i+1:]...)
            break
        }
    }
}

func (s *Subject) Notify(event Event) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    for _, o := range s.observers {
        go o.Update(event)  // Async notification
    }
}

// Channel-based observer (more idiomatic Go)
type EventBus struct {
    subscribers map[string][]chan Event
    mu          sync.RWMutex
}

func (b *EventBus) Subscribe(topic string) <-chan Event {
    ch := make(chan Event, 10)
    b.mu.Lock()
    b.subscribers[topic] = append(b.subscribers[topic], ch)
    b.mu.Unlock()
    return ch
}

func (b *EventBus) Publish(topic string, event Event) {
    b.mu.RLock()
    defer b.mu.RUnlock()
    for _, ch := range b.subscribers[topic] {
        select {
        case ch <- event:
        default:  // Don't block if subscriber is slow
        }
    }
}

// CHAIN OF RESPONSIBILITY
type Middleware func(Handler) Handler

func LoggingMiddleware(next Handler) Handler {
    return HandlerFunc(func(ctx context.Context, req Request) Response {
        log.Printf("Request: %v", req)
        return next.Handle(ctx, req)
    })
}

func AuthMiddleware(next Handler) Handler {
    return HandlerFunc(func(ctx context.Context, req Request) Response {
        if !isAuthorized(req) {
            return Response{Status: 401}
        }
        return next.Handle(ctx, req)
    })
}

// Chain middlewares
func Chain(h Handler, middlewares ...Middleware) Handler {
    for i := len(middlewares) - 1; i >= 0; i-- {
        h = middlewares[i](h)
    }
    return h
}

handler := Chain(baseHandler, LoggingMiddleware, AuthMiddleware)
```

---

## Concurrency Patterns

<a id="q4"></a>
### Q4: What is the Worker Pool pattern?
**Answer:**

Worker pool limits concurrent goroutines processing jobs:

```go
// Simple worker pool
func WorkerPool(numWorkers int, jobs <-chan Job, results chan<- Result) {
    var wg sync.WaitGroup
    
    for i := 0; i < numWorkers; i++ {
        wg.Add(1)
        go func(workerID int) {
            defer wg.Done()
            for job := range jobs {
                result := processJob(job)
                results <- result
            }
        }(i)
    }
    
    wg.Wait()
    close(results)
}

func main() {
    jobs := make(chan Job, 100)
    results := make(chan Result, 100)
    
    // Start workers
    go WorkerPool(5, jobs, results)
    
    // Send jobs
    go func() {
        for _, job := range allJobs {
            jobs <- job
        }
        close(jobs)
    }()
    
    // Collect results
    for result := range results {
        fmt.Println(result)
    }
}

// Worker pool with context and error handling
type WorkerPool struct {
    numWorkers int
    jobs       chan Job
    results    chan Result
    errors     chan error
}

func NewWorkerPool(numWorkers, jobBuffer int) *WorkerPool {
    return &WorkerPool{
        numWorkers: numWorkers,
        jobs:       make(chan Job, jobBuffer),
        results:    make(chan Result, jobBuffer),
        errors:     make(chan error, jobBuffer),
    }
}

func (p *WorkerPool) Start(ctx context.Context) {
    var wg sync.WaitGroup
    
    for i := 0; i < p.numWorkers; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for {
                select {
                case <-ctx.Done():
                    return
                case job, ok := <-p.jobs:
                    if !ok {
                        return
                    }
                    result, err := processJob(job)
                    if err != nil {
                        p.errors <- err
                    } else {
                        p.results <- result
                    }
                }
            }
        }(i)
    }
    
    go func() {
        wg.Wait()
        close(p.results)
        close(p.errors)
    }()
}

func (p *WorkerPool) Submit(job Job) {
    p.jobs <- job
}

func (p *WorkerPool) Close() {
    close(p.jobs)
}

// Semaphore-based rate limiting
type Semaphore struct {
    sem chan struct{}
}

func NewSemaphore(max int) *Semaphore {
    return &Semaphore{sem: make(chan struct{}, max)}
}

func (s *Semaphore) Acquire() {
    s.sem <- struct{}{}
}

func (s *Semaphore) Release() {
    <-s.sem
}

// Usage
sem := NewSemaphore(10)  // Max 10 concurrent

for _, task := range tasks {
    sem.Acquire()
    go func(t Task) {
        defer sem.Release()
        process(t)
    }(task)
}
```

<a id="q5"></a>
### Q5: What is Fan-out/Fan-in?
**Answer:**

Fan-out distributes work; Fan-in collects results:

```go
// Fan-out: start multiple goroutines reading from same channel
func fanOut(input <-chan int, numWorkers int) []<-chan int {
    outputs := make([]<-chan int, numWorkers)
    
    for i := 0; i < numWorkers; i++ {
        outputs[i] = worker(input)
    }
    
    return outputs
}

func worker(input <-chan int) <-chan int {
    output := make(chan int)
    go func() {
        defer close(output)
        for n := range input {
            output <- process(n)
        }
    }()
    return output
}

// Fan-in: merge multiple channels into one
func fanIn(inputs ...<-chan int) <-chan int {
    output := make(chan int)
    var wg sync.WaitGroup
    
    for _, input := range inputs {
        wg.Add(1)
        go func(ch <-chan int) {
            defer wg.Done()
            for n := range ch {
                output <- n
            }
        }(input)
    }
    
    go func() {
        wg.Wait()
        close(output)
    }()
    
    return output
}

// Complete example
func main() {
    // Generate input
    input := make(chan int)
    go func() {
        defer close(input)
        for i := 0; i < 100; i++ {
            input <- i
        }
    }()
    
    // Fan-out to 5 workers
    workers := fanOut(input, 5)
    
    // Fan-in results
    results := fanIn(workers...)
    
    // Consume results
    for result := range results {
        fmt.Println(result)
    }
}

// Fan-out with bounded concurrency
func boundedFanOut[T, R any](
    ctx context.Context,
    input []T,
    maxConcurrency int,
    process func(T) R,
) []R {
    sem := make(chan struct{}, maxConcurrency)
    results := make([]R, len(input))
    var wg sync.WaitGroup
    
    for i, item := range input {
        wg.Add(1)
        go func(idx int, val T) {
            defer wg.Done()
            
            select {
            case <-ctx.Done():
                return
            case sem <- struct{}{}:
                defer func() { <-sem }()
                results[idx] = process(val)
            }
        }(i, item)
    }
    
    wg.Wait()
    return results
}
```

<a id="q6"></a>
### Q6: What is the Pipeline pattern?
**Answer:**

Pipeline connects stages where each stage is a goroutine:

```go
// Pipeline stage type
type Stage func(<-chan int) <-chan int

// Generator stage (source)
func generator(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            out <- n
        }
    }()
    return out
}

// Transform stage
func square(input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range input {
            out <- n * n
        }
    }()
    return out
}

func double(input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range input {
            out <- n * 2
        }
    }()
    return out
}

// Filter stage
func filter(input <-chan int, predicate func(int) bool) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range input {
            if predicate(n) {
                out <- n
            }
        }
    }()
    return out
}

// Compose pipeline
func main() {
    // Create pipeline: generate -> square -> filter -> double
    nums := generator(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
    squared := square(nums)
    filtered := filter(squared, func(n int) bool { return n > 25 })
    doubled := double(filtered)
    
    for result := range doubled {
        fmt.Println(result)
    }
}

// Pipeline with context for cancellation
func generatorCtx(ctx context.Context, nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums {
            select {
            case <-ctx.Done():
                return
            case out <- n:
            }
        }
    }()
    return out
}

func squareCtx(ctx context.Context, input <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case <-ctx.Done():
                return
            case n, ok := <-input:
                if !ok {
                    return
                }
                select {
                case out <- n * n:
                case <-ctx.Done():
                    return
                }
            }
        }
    }()
    return out
}
```

<a id="q7"></a>
### Q7: What are other common concurrency patterns?
**Answer:**

```go
// OR-DONE CHANNEL - wrap channel reads with done check
func orDone(done <-chan struct{}, c <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for {
            select {
            case <-done:
                return
            case v, ok := <-c:
                if !ok {
                    return
                }
                select {
                case out <- v:
                case <-done:
                    return
                }
            }
        }
    }()
    return out
}

// TEE - split one channel into two
func tee(done <-chan struct{}, in <-chan int) (<-chan int, <-chan int) {
    out1 := make(chan int)
    out2 := make(chan int)
    
    go func() {
        defer close(out1)
        defer close(out2)
        
        for val := range orDone(done, in) {
            // Use local variables to avoid blocking
            o1, o2 := out1, out2
            for i := 0; i < 2; i++ {
                select {
                case <-done:
                    return
                case o1 <- val:
                    o1 = nil  // Disable this case after sending
                case o2 <- val:
                    o2 = nil
                }
            }
        }
    }()
    
    return out1, out2
}

// BRIDGE - flatten channel of channels
func bridge(done <-chan struct{}, chanStream <-chan <-chan int) <-chan int {
    out := make(chan int)
    
    go func() {
        defer close(out)
        
        for {
            var stream <-chan int
            select {
            case <-done:
                return
            case maybeStream, ok := <-chanStream:
                if !ok {
                    return
                }
                stream = maybeStream
            }
            
            for val := range orDone(done, stream) {
                select {
                case out <- val:
                case <-done:
                    return
                }
            }
        }
    }()
    
    return out
}

// CIRCUIT BREAKER - prevent cascade failures
type CircuitBreaker struct {
    mu            sync.Mutex
    failures      int
    threshold     int
    state         string  // closed, open, half-open
    lastFailure   time.Time
    resetTimeout  time.Duration
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    cb.mu.Lock()
    
    if cb.state == "open" {
        if time.Since(cb.lastFailure) > cb.resetTimeout {
            cb.state = "half-open"
        } else {
            cb.mu.Unlock()
            return errors.New("circuit breaker is open")
        }
    }
    cb.mu.Unlock()
    
    err := fn()
    
    cb.mu.Lock()
    defer cb.mu.Unlock()
    
    if err != nil {
        cb.failures++
        cb.lastFailure = time.Now()
        if cb.failures >= cb.threshold {
            cb.state = "open"
        }
        return err
    }
    
    cb.failures = 0
    cb.state = "closed"
    return nil
}

// RATE LIMITER - control request rate
type RateLimiter struct {
    ticker *time.Ticker
    tokens chan struct{}
}

func NewRateLimiter(rate int, burst int) *RateLimiter {
    rl := &RateLimiter{
        ticker: time.NewTicker(time.Second / time.Duration(rate)),
        tokens: make(chan struct{}, burst),
    }
    
    // Fill initial burst
    for i := 0; i < burst; i++ {
        rl.tokens <- struct{}{}
    }
    
    // Refill tokens
    go func() {
        for range rl.ticker.C {
            select {
            case rl.tokens <- struct{}{}:
            default:  // Bucket full
            }
        }
    }()
    
    return rl
}

func (rl *RateLimiter) Wait() {
    <-rl.tokens
}

func (rl *RateLimiter) Stop() {
    rl.ticker.Stop()
}
```

---

## Performance & Profiling

<a id="q8"></a>
### Q8: How do you profile Go applications with pprof?
**Answer:**

```go
import (
    "net/http"
    _ "net/http/pprof"  // Import for side effects
    "runtime/pprof"
)

// Method 1: HTTP server (for long-running apps)
func main() {
    // pprof endpoints automatically registered
    go func() {
        http.ListenAndServe("localhost:6060", nil)
    }()
    
    // Your application...
}

// Access profiles:
// http://localhost:6060/debug/pprof/
// http://localhost:6060/debug/pprof/heap
// http://localhost:6060/debug/pprof/goroutine
// http://localhost:6060/debug/pprof/profile?seconds=30

// Method 2: Programmatic profiling
func main() {
    // CPU profile
    cpuFile, _ := os.Create("cpu.prof")
    defer cpuFile.Close()
    pprof.StartCPUProfile(cpuFile)
    defer pprof.StopCPUProfile()
    
    // Your code here...
    
    // Memory profile
    memFile, _ := os.Create("mem.prof")
    defer memFile.Close()
    runtime.GC()  // Get accurate stats
    pprof.WriteHeapProfile(memFile)
}

// Analyze profiles
// go tool pprof cpu.prof
// go tool pprof -http=:8080 cpu.prof  (web UI)

// Common pprof commands
// top           - show top functions
// top10         - top 10 functions
// list funcName - show source with annotations
// web           - generate SVG graph
// svg           - save SVG to file

// Profile specific benchmark
// go test -bench=. -cpuprofile=cpu.prof -memprofile=mem.prof

// Method 3: Benchmark profiling
func BenchmarkFunction(b *testing.B) {
    for i := 0; i < b.N; i++ {
        myFunction()
    }
}

// go test -bench=BenchmarkFunction -cpuprofile=cpu.prof
// go tool pprof cpu.prof

// Tracing
import "runtime/trace"

func main() {
    traceFile, _ := os.Create("trace.out")
    defer traceFile.Close()
    
    trace.Start(traceFile)
    defer trace.Stop()
    
    // Your code...
}

// Analyze trace
// go tool trace trace.out
```

<a id="q9"></a>
### Q9: How do you optimize memory usage in Go?
**Answer:**

```go
// 1. Preallocate slices
// Bad
var s []int
for i := 0; i < 10000; i++ {
    s = append(s, i)  // Multiple reallocations
}

// Good
s := make([]int, 0, 10000)
for i := 0; i < 10000; i++ {
    s = append(s, i)  // No reallocation
}

// 2. Use strings.Builder for string concatenation
// Bad
var s string
for i := 0; i < 1000; i++ {
    s += "x"  // Creates new string each time
}

// Good
var sb strings.Builder
sb.Grow(1000)  // Preallocate
for i := 0; i < 1000; i++ {
    sb.WriteString("x")
}
result := sb.String()

// 3. Use sync.Pool for frequently allocated objects
var bufferPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 4096)
    },
}

func process(data []byte) {
    buf := bufferPool.Get().([]byte)
    defer bufferPool.Put(buf)
    
    // Use buf...
}

// 4. Avoid pointers in large slices (better cache locality)
// Worse for large slices
type Item struct {
    data *Data  // Pointer chasing
}

// Better for large slices
type Item struct {
    data Data  // Inline data
}

// 5. Struct field ordering (reduce padding)
// Bad (24 bytes due to padding)
type Bad struct {
    a bool    // 1 byte + 7 padding
    b int64   // 8 bytes
    c bool    // 1 byte + 7 padding
}

// Good (10 bytes, may round to 16)
type Good struct {
    b int64   // 8 bytes
    a bool    // 1 byte
    c bool    // 1 byte
}

// 6. Use []byte instead of string for mutations
// Strings are immutable, []byte is not

// 7. Stream large files instead of loading into memory
func processLargeFile(filename string) error {
    file, _ := os.Open(filename)
    defer file.Close()
    
    scanner := bufio.NewScanner(file)
    for scanner.Scan() {
        processLine(scanner.Bytes())  // Process line by line
    }
    return scanner.Err()
}

// 8. Use smaller integer types when appropriate
type Compact struct {
    age   uint8   // 0-255 is enough
    count int32   // If int64 not needed
}

// 9. Clear slice references to allow GC
func process(items []*LargeObject) {
    for i := range items {
        process(items[i])
        items[i] = nil  // Allow GC to collect
    }
}
```

<a id="q10"></a>
### Q10: How do you optimize CPU performance?
**Answer:**

```go
// 1. Avoid allocations in hot paths
// Bad
func process(data []int) int {
    result := make([]int, len(data))  // Allocation every call
    // ...
}

// Good - reuse buffer
func process(data []int, buf []int) int {
    buf = buf[:0]  // Reset, keep capacity
    // ...
}

// 2. Use value receivers for small structs
type Point struct {
    X, Y int
}

// Good for small struct - no indirection
func (p Point) Distance() float64 {
    return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}

// 3. Inline small functions (compiler may do this)
//go:noinline  // Prevent inlining for benchmarking
func add(a, b int) int { return a + b }

// 4. Use efficient data structures
// Map lookup: O(1)
// Slice search: O(n)
// Binary search sorted slice: O(log n)

// 5. Batch operations
// Bad - many small writes
for _, item := range items {
    file.Write(serialize(item))
}

// Good - batch writes
var buf bytes.Buffer
for _, item := range items {
    buf.Write(serialize(item))
}
file.Write(buf.Bytes())

// 6. Parallel processing for CPU-bound tasks
func parallelProcess(items []Item) []Result {
    numCPU := runtime.NumCPU()
    results := make([]Result, len(items))
    
    var wg sync.WaitGroup
    chunkSize := (len(items) + numCPU - 1) / numCPU
    
    for i := 0; i < numCPU; i++ {
        start := i * chunkSize
        end := start + chunkSize
        if end > len(items) {
            end = len(items)
        }
        
        wg.Add(1)
        go func(start, end int) {
            defer wg.Done()
            for j := start; j < end; j++ {
                results[j] = processItem(items[j])
            }
        }(start, end)
    }
    
    wg.Wait()
    return results
}

// 7. Avoid interface{} in hot paths
// Interface calls are slower than direct calls

// 8. Use SIMD-friendly code patterns
// Process data in chunks that fit CPU cache lines

// 9. Profile and optimize hotspots only
// Don't optimize prematurely - profile first!

// 10. Use compiler optimizations
// go build -gcflags="-m"  // See escape analysis
// go build -gcflags="-l"  // Disable inlining (for debugging)
```

<a id="q11"></a>
### Q11: What are common performance pitfalls in Go?
**Answer:**

```go
// 1. Defer in tight loops
// Bad
for i := 0; i < 1000000; i++ {
    f, _ := os.Open(file)
    defer f.Close()  // Defers pile up!
}

// Good
for i := 0; i < 1000000; i++ {
    func() {
        f, _ := os.Open(file)
        defer f.Close()  // Executes at function end
    }()
}

// 2. Using + for string concatenation in loops
// (Already covered in memory optimization)

// 3. Not setting slice capacity
// Bad
s := []int{}
for i := 0; i < n; i++ {
    s = append(s, i)  // Many reallocations
}

// 4. Unnecessary memory copies
// Bad
func process(data []byte) {
    copy := append([]byte{}, data...)  // Unnecessary copy
}

// 5. Regex compilation in loops
// Bad
for _, s := range strings {
    matched, _ := regexp.MatchString(pattern, s)  // Recompiles each time!
}

// Good
re := regexp.MustCompile(pattern)  // Compile once
for _, s := range strings {
    matched := re.MatchString(s)
}

// 6. Lock contention
// Bad - one global lock
var mu sync.Mutex
var cache = make(map[string]string)

// Good - sharded locks
type ShardedCache struct {
    shards [256]struct {
        mu   sync.RWMutex
        data map[string]string
    }
}

func (c *ShardedCache) Get(key string) string {
    shard := &c.shards[hash(key)%256]
    shard.mu.RLock()
    defer shard.mu.RUnlock()
    return shard.data[key]
}

// 7. Goroutine leaks
// Bad - goroutine never exits
func bad() {
    ch := make(chan int)
    go func() {
        for v := range ch {  // Never closes!
            process(v)
        }
    }()
    // ch is never closed, goroutine leaks
}

// Good - use context or close channel
func good(ctx context.Context) {
    ch := make(chan int)
    go func() {
        for {
            select {
            case <-ctx.Done():
                return
            case v, ok := <-ch:
                if !ok {
                    return
                }
                process(v)
            }
        }
    }()
}

// 8. JSON marshal/unmarshal overhead
// Use streaming for large JSON
// Consider binary formats (protobuf, msgpack) for performance

// 9. Time.Now() in tight loops
// Cache time for batches if microsecond precision not needed

// 10. Reflection in hot paths
// Use code generation or generics instead
```

---

[← Back to Go Index](README.md)

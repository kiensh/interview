# Observability in Go

## Table of Contents

### Logging
- [Q1: What are best practices for logging in Go?](#q1)
- [Q2: How do you use zerolog?](#q2)
- [Q3: How do you use zap?](#q3)

### Metrics
- [Q4: How do you expose Prometheus metrics in Go?](#q4)
- [Q5: What types of metrics should you collect?](#q5)
- [Q6: How do you create custom metrics?](#q6)

### Tracing
- [Q7: What is distributed tracing?](#q7)
- [Q8: How do you implement OpenTelemetry in Go?](#q8)
- [Q9: How do you propagate trace context?](#q9)
- [Q10: How do you correlate logs, metrics, and traces?](#q10)

---

## Logging

<a id="q1"></a>
### Q1: What are best practices for logging in Go?
**Answer:**

```go
// Best Practices:
// 1. Use structured logging (JSON)
// 2. Include context (request ID, user ID)
// 3. Use appropriate log levels
// 4. Don't log sensitive data
// 5. Include timestamps
// 6. Make logs machine-parseable

// Log levels
/*
DEBUG - Development details
INFO  - Normal operations
WARN  - Potential issues
ERROR - Errors that need attention
FATAL - Application cannot continue
*/

// Standard library logging (basic)
import "log"

func init() {
    log.SetFlags(log.LstdFlags | log.Lshortfile)
}

func example() {
    log.Println("Info message")
    log.Printf("User %d logged in", userID)
    log.Fatal("Fatal error, exiting")  // Calls os.Exit(1)
}

// Structured logging pattern
type LogEntry struct {
    Timestamp string `json:"timestamp"`
    Level     string `json:"level"`
    Message   string `json:"message"`
    Service   string `json:"service"`
    RequestID string `json:"request_id,omitempty"`
    UserID    string `json:"user_id,omitempty"`
    Error     string `json:"error,omitempty"`
    Duration  int64  `json:"duration_ms,omitempty"`
}

// Context-aware logging
type Logger interface {
    Debug(msg string, fields ...Field)
    Info(msg string, fields ...Field)
    Warn(msg string, fields ...Field)
    Error(msg string, fields ...Field)
    With(fields ...Field) Logger
}

// Middleware for request logging
func LoggingMiddleware(logger Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            requestID := r.Header.Get("X-Request-ID")
            if requestID == "" {
                requestID = uuid.New().String()
            }
            
            // Add to context
            ctx := context.WithValue(r.Context(), "request_id", requestID)
            
            // Wrap response writer to capture status
            wrapped := &responseWriter{ResponseWriter: w}
            
            next.ServeHTTP(wrapped, r.WithContext(ctx))
            
            logger.Info("HTTP request",
                String("method", r.Method),
                String("path", r.URL.Path),
                Int("status", wrapped.status),
                Duration("duration", time.Since(start)),
                String("request_id", requestID),
            )
        })
    }
}

// Sensitive data filtering
func sanitizeLog(data map[string]interface{}) map[string]interface{} {
    sensitiveKeys := []string{"password", "token", "secret", "api_key", "credit_card"}
    
    for key := range data {
        for _, sensitive := range sensitiveKeys {
            if strings.Contains(strings.ToLower(key), sensitive) {
                data[key] = "[REDACTED]"
            }
        }
    }
    return data
}
```

<a id="q2"></a>
### Q2: How do you use zerolog?
**Answer:**

```go
import "github.com/rs/zerolog"
import "github.com/rs/zerolog/log"

// Configure zerolog
func initZerolog() {
    // Set global level
    zerolog.SetGlobalLevel(zerolog.InfoLevel)
    
    // Pretty console output for development
    if os.Getenv("ENV") == "development" {
        log.Logger = log.Output(zerolog.ConsoleWriter{Out: os.Stderr})
    }
    
    // Add default fields
    log.Logger = log.With().
        Str("service", "my-service").
        Str("version", "1.0.0").
        Logger()
}

// Basic logging
func zerologExamples() {
    // Simple message
    log.Info().Msg("Server started")
    
    // With fields
    log.Info().
        Str("user", "alice").
        Int("attempt", 3).
        Msg("Login attempt")
    
    // With error
    err := doSomething()
    if err != nil {
        log.Error().
            Err(err).
            Str("operation", "doSomething").
            Msg("Operation failed")
    }
    
    // Structured data
    log.Info().
        Dict("user", zerolog.Dict().
            Str("id", "123").
            Str("name", "Alice")).
        Msg("User info")
    
    // Array
    log.Info().
        Strs("tags", []string{"go", "api", "web"}).
        Msg("Tagged item")
}

// Context-aware logging
func zerologWithContext(ctx context.Context) {
    // Get logger from context
    logger := zerolog.Ctx(ctx)
    logger.Info().Msg("Processing request")
}

// HTTP middleware
func ZerologMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // Create sub-logger with request context
        logger := log.With().
            Str("request_id", r.Header.Get("X-Request-ID")).
            Str("method", r.Method).
            Str("path", r.URL.Path).
            Logger()
        
        // Add logger to context
        ctx := logger.WithContext(r.Context())
        
        next.ServeHTTP(w, r.WithContext(ctx))
        
        logger.Info().
            Dur("duration", time.Since(start)).
            Msg("Request completed")
    })
}

// Sampling for high-volume logs
func zerologSampling() {
    sampled := log.Sample(&zerolog.BurstSampler{
        Burst:       5,
        Period:      time.Second,
        NextSampler: &zerolog.BasicSampler{N: 100},
    })
    
    // Only logs 5/sec burst, then 1/100
    for i := 0; i < 1000; i++ {
        sampled.Info().Msg("High volume log")
    }
}

// Hook for enrichment
type ServiceHook struct{}

func (h ServiceHook) Run(e *zerolog.Event, level zerolog.Level, msg string) {
    e.Str("hostname", hostname)
    e.Str("env", os.Getenv("ENV"))
}

log.Logger = log.Hook(ServiceHook{})
```

<a id="q3"></a>
### Q3: How do you use zap?
**Answer:**

```go
import "go.uber.org/zap"
import "go.uber.org/zap/zapcore"

// Configure zap
func initZap() (*zap.Logger, error) {
    // Production config
    config := zap.Config{
        Level:       zap.NewAtomicLevelAt(zap.InfoLevel),
        Development: false,
        Encoding:    "json",
        EncoderConfig: zapcore.EncoderConfig{
            TimeKey:        "timestamp",
            LevelKey:       "level",
            NameKey:        "logger",
            CallerKey:      "caller",
            MessageKey:     "message",
            StacktraceKey:  "stacktrace",
            LineEnding:     zapcore.DefaultLineEnding,
            EncodeLevel:    zapcore.LowercaseLevelEncoder,
            EncodeTime:     zapcore.ISO8601TimeEncoder,
            EncodeDuration: zapcore.MillisDurationEncoder,
            EncodeCaller:   zapcore.ShortCallerEncoder,
        },
        OutputPaths:      []string{"stdout"},
        ErrorOutputPaths: []string{"stderr"},
    }
    
    return config.Build()
}

// Quick setup
func quickSetup() {
    // Development (pretty print)
    logger, _ := zap.NewDevelopment()
    
    // Production (JSON)
    logger, _ := zap.NewProduction()
    
    defer logger.Sync()
}

// Basic logging
func zapExamples(logger *zap.Logger) {
    // Structured logging
    logger.Info("User logged in",
        zap.String("user_id", "123"),
        zap.String("ip", "192.168.1.1"),
        zap.Int("attempt", 1),
    )
    
    // With error
    logger.Error("Failed to connect",
        zap.Error(err),
        zap.String("host", "localhost"),
        zap.Int("port", 5432),
    )
    
    // Sugar logger (slower but more convenient)
    sugar := logger.Sugar()
    sugar.Infof("User %s logged in from %s", userID, ip)
    sugar.Infow("User logged in",
        "user_id", userID,
        "ip", ip,
    )
}

// Child logger with context
func zapWithContext(logger *zap.Logger, requestID string) {
    // Create child logger with fields
    reqLogger := logger.With(
        zap.String("request_id", requestID),
        zap.String("service", "api"),
    )
    
    reqLogger.Info("Processing request")
    reqLogger.Info("Request completed")  // Includes request_id and service
}

// HTTP middleware
func ZapMiddleware(logger *zap.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            
            wrapped := &responseWriter{ResponseWriter: w, status: http.StatusOK}
            next.ServeHTTP(wrapped, r)
            
            logger.Info("HTTP request",
                zap.String("method", r.Method),
                zap.String("path", r.URL.Path),
                zap.Int("status", wrapped.status),
                zap.Duration("duration", time.Since(start)),
                zap.String("remote_addr", r.RemoteAddr),
            )
        })
    }
}

// Dynamic level change
func dynamicLevel() {
    atom := zap.NewAtomicLevel()
    
    logger := zap.New(zapcore.NewCore(
        zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig()),
        zapcore.Lock(os.Stdout),
        atom,
    ))
    
    // Change level at runtime
    atom.SetLevel(zap.DebugLevel)
    
    // HTTP endpoint to change level
    http.HandleFunc("/log-level", atom.ServeHTTP)
}

// Sampling for high throughput
func zapSampling() {
    config := zap.NewProductionConfig()
    config.Sampling = &zap.SamplingConfig{
        Initial:    100,           // First 100 per second
        Thereafter: 100,           // Then every 100th
    }
    logger, _ := config.Build()
}
```

---

## Metrics

<a id="q4"></a>
### Q4: How do you expose Prometheus metrics in Go?
**Answer:**

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

// Register and expose metrics
func main() {
    // Create metrics
    requestsTotal := prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "path", "status"},
    )
    
    requestDuration := prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request duration in seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "path"},
    )
    
    // Register metrics
    prometheus.MustRegister(requestsTotal)
    prometheus.MustRegister(requestDuration)
    
    // Expose /metrics endpoint
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}

// Middleware to record metrics
func MetricsMiddleware(
    requestsTotal *prometheus.CounterVec,
    requestDuration *prometheus.HistogramVec,
) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            start := time.Now()
            
            wrapped := &responseWriter{ResponseWriter: w, status: http.StatusOK}
            next.ServeHTTP(wrapped, r)
            
            duration := time.Since(start).Seconds()
            status := strconv.Itoa(wrapped.status)
            
            requestsTotal.WithLabelValues(r.Method, r.URL.Path, status).Inc()
            requestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
        })
    }
}

// Default process and Go runtime metrics
func registerDefaultMetrics() {
    // These are registered by default:
    // - go_goroutines
    // - go_gc_duration_seconds
    // - go_memstats_*
    // - process_cpu_seconds_total
    // - process_resident_memory_bytes
    
    // Custom registry without defaults
    registry := prometheus.NewRegistry()
    registry.MustRegister(myMetric)
    
    // Handler with custom registry
    http.Handle("/metrics", promhttp.HandlerFor(registry, promhttp.HandlerOpts{}))
}
```

<a id="q5"></a>
### Q5: What types of metrics should you collect?
**Answer:**

```go
// RED Method (Request-focused)
// Rate - Requests per second
// Errors - Error rate
// Duration - Request latency

// USE Method (Resource-focused)
// Utilization - % time resource is busy
// Saturation - Work queued
// Errors - Error count

// Four Golden Signals (Google SRE)
// Latency, Traffic, Errors, Saturation

var (
    // Counter - only increases
    requestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "status"},
    )
    
    // Gauge - can increase or decrease
    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "active_connections",
            Help: "Number of active connections",
        },
    )
    
    // Histogram - distribution of values
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "Request duration in seconds",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "path"},
    )
    
    // Summary - similar to histogram with quantiles
    requestDurationSummary = prometheus.NewSummaryVec(
        prometheus.SummaryOpts{
            Name:       "http_request_duration_summary",
            Help:       "Request duration summary",
            Objectives: map[float64]float64{0.5: 0.05, 0.9: 0.01, 0.99: 0.001},
        },
        []string{"method"},
    )
)

// Business metrics examples
var (
    ordersCreated = prometheus.NewCounter(prometheus.CounterOpts{
        Name: "orders_created_total",
        Help: "Total orders created",
    })
    
    orderValue = prometheus.NewHistogram(prometheus.HistogramOpts{
        Name:    "order_value_dollars",
        Help:    "Order value distribution",
        Buckets: []float64{10, 25, 50, 100, 250, 500, 1000},
    })
    
    inventoryLevel = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "inventory_level",
            Help: "Current inventory level",
        },
        []string{"product_id"},
    )
    
    paymentProcessingTime = prometheus.NewHistogram(prometheus.HistogramOpts{
        Name:    "payment_processing_seconds",
        Help:    "Payment processing time",
        Buckets: []float64{0.1, 0.5, 1, 2, 5, 10},
    })
)
```

<a id="q6"></a>
### Q6: How do you create custom metrics?
**Answer:**

```go
// Custom collector for dynamic metrics
type DatabaseCollector struct {
    db              *sql.DB
    connectionPool  *prometheus.Desc
    queryDuration   *prometheus.Desc
}

func NewDatabaseCollector(db *sql.DB) *DatabaseCollector {
    return &DatabaseCollector{
        db: db,
        connectionPool: prometheus.NewDesc(
            "db_connection_pool_size",
            "Database connection pool statistics",
            []string{"state"},  // labels
            nil,
        ),
        queryDuration: prometheus.NewDesc(
            "db_query_duration_seconds",
            "Database query duration",
            []string{"query_type"},
            nil,
        ),
    }
}

func (c *DatabaseCollector) Describe(ch chan<- *prometheus.Desc) {
    ch <- c.connectionPool
    ch <- c.queryDuration
}

func (c *DatabaseCollector) Collect(ch chan<- prometheus.Metric) {
    stats := c.db.Stats()
    
    ch <- prometheus.MustNewConstMetric(
        c.connectionPool,
        prometheus.GaugeValue,
        float64(stats.OpenConnections),
        "open",
    )
    
    ch <- prometheus.MustNewConstMetric(
        c.connectionPool,
        prometheus.GaugeValue,
        float64(stats.InUse),
        "in_use",
    )
    
    ch <- prometheus.MustNewConstMetric(
        c.connectionPool,
        prometheus.GaugeValue,
        float64(stats.Idle),
        "idle",
    )
}

// Register custom collector
prometheus.MustRegister(NewDatabaseCollector(db))

// Timer helper
func timeOperation(histogram prometheus.Histogram, operation func() error) error {
    timer := prometheus.NewTimer(histogram)
    defer timer.ObserveDuration()
    return operation()
}

// Usage
err := timeOperation(queryDuration, func() error {
    return db.QueryRow("SELECT ...").Scan(&result)
})

// Metric with timestamp
func recordMetricWithTimestamp(gauge prometheus.Gauge, value float64, t time.Time) {
    gauge.Set(value)
    // For custom timestamps, use prometheus.NewMetricWithTimestamp
}

// Pushgateway for batch jobs
import "github.com/prometheus/client_golang/prometheus/push"

func batchJobMetrics() {
    registry := prometheus.NewRegistry()
    
    jobDuration := prometheus.NewGauge(prometheus.GaugeOpts{
        Name: "batch_job_duration_seconds",
    })
    registry.MustRegister(jobDuration)
    
    start := time.Now()
    // Run job...
    jobDuration.Set(time.Since(start).Seconds())
    
    // Push to Pushgateway
    push.New("http://pushgateway:9091", "batch_job").
        Gatherer(registry).
        Grouping("instance", hostname).
        Push()
}
```

---

## Tracing

<a id="q7"></a>
### Q7: What is distributed tracing?
**Answer:**

Distributed tracing tracks requests across multiple services:

```go
/*
Concepts:
- Trace: Full journey of a request
- Span: Single unit of work within a trace
- Context: Carries trace information across services

Example trace:
┌─────────────────────────────────────────────────────────────┐
│ Trace ID: abc123                                            │
├─────────────────────────────────────────────────────────────┤
│ Span: API Gateway (10ms)                                    │
│   └─ Span: Auth Service (2ms)                               │
│   └─ Span: Order Service (50ms)                             │
│       └─ Span: Database Query (5ms)                         │
│       └─ Span: Payment Service (40ms)                       │
│           └─ Span: External Payment API (35ms)              │
│   └─ Span: Notification Service (3ms)                       │
└─────────────────────────────────────────────────────────────┘

Benefits:
- Understand request flow
- Identify bottlenecks
- Debug distributed failures
- Performance optimization
*/

// Span attributes to include:
// - service.name
// - http.method, http.url, http.status_code
// - db.system, db.statement
// - error (boolean)
// - Custom business attributes
```

<a id="q8"></a>
### Q8: How do you implement OpenTelemetry in Go?
**Answer:**

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
    "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/attribute"
    semconv "go.opentelemetry.io/otel/semconv/v1.17.0"
)

// Initialize OpenTelemetry
func initTracer(ctx context.Context, serviceName string) (func(), error) {
    // Create OTLP exporter
    exporter, err := otlptracehttp.New(ctx,
        otlptracehttp.WithEndpoint("localhost:4318"),
        otlptracehttp.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    // Create trace provider
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName(serviceName),
            semconv.ServiceVersion("1.0.0"),
            attribute.String("environment", "production"),
        )),
        trace.WithSampler(trace.AlwaysSample()),
    )
    
    // Set global provider
    otel.SetTracerProvider(tp)
    
    // Return cleanup function
    return func() {
        tp.Shutdown(context.Background())
    }, nil
}

// Create spans
func processOrder(ctx context.Context, orderID string) error {
    tracer := otel.Tracer("order-service")
    
    // Start span
    ctx, span := tracer.Start(ctx, "processOrder")
    defer span.End()
    
    // Add attributes
    span.SetAttributes(
        attribute.String("order.id", orderID),
        attribute.String("order.type", "standard"),
    )
    
    // Child span for database
    ctx, dbSpan := tracer.Start(ctx, "database.query")
    order, err := getOrderFromDB(ctx, orderID)
    if err != nil {
        dbSpan.RecordError(err)
        dbSpan.SetStatus(codes.Error, err.Error())
    }
    dbSpan.End()
    
    // Child span for payment
    ctx, paymentSpan := tracer.Start(ctx, "payment.process")
    paymentSpan.SetAttributes(
        attribute.Float64("payment.amount", order.Total),
    )
    err = processPayment(ctx, order)
    paymentSpan.End()
    
    return err
}

// HTTP middleware with tracing
import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

func main() {
    // Wrap handler
    handler := otelhttp.NewHandler(myHandler, "server")
    http.ListenAndServe(":8080", handler)
}

// HTTP client with tracing
func makeRequest(ctx context.Context, url string) error {
    client := &http.Client{
        Transport: otelhttp.NewTransport(http.DefaultTransport),
    }
    
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := client.Do(req)
    // Trace is automatically propagated
    return err
}

// gRPC with tracing
import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

func main() {
    // Server
    server := grpc.NewServer(
        grpc.UnaryInterceptor(otelgrpc.UnaryServerInterceptor()),
        grpc.StreamInterceptor(otelgrpc.StreamServerInterceptor()),
    )
    
    // Client
    conn, _ := grpc.Dial(address,
        grpc.WithUnaryInterceptor(otelgrpc.UnaryClientInterceptor()),
        grpc.WithStreamInterceptor(otelgrpc.StreamClientInterceptor()),
    )
}
```

<a id="q9"></a>
### Q9: How do you propagate trace context?
**Answer:**

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
)

// Set up propagation
func initPropagation() {
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))
}

// Extract context from HTTP request
func extractContext(r *http.Request) context.Context {
    return otel.GetTextMapPropagator().Extract(
        r.Context(),
        propagation.HeaderCarrier(r.Header),
    )
}

// Inject context into HTTP request
func injectContext(ctx context.Context, req *http.Request) {
    otel.GetTextMapPropagator().Inject(
        ctx,
        propagation.HeaderCarrier(req.Header),
    )
}

// HTTP middleware
func TracingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract trace context from incoming request
        ctx := extractContext(r)
        
        // Start span
        tracer := otel.Tracer("http-server")
        ctx, span := tracer.Start(ctx, r.URL.Path)
        defer span.End()
        
        // Continue with traced context
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// HTTP client
func tracedHTTPRequest(ctx context.Context, url string) (*http.Response, error) {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    
    // Inject trace context into outgoing request
    injectContext(ctx, req)
    
    return http.DefaultClient.Do(req)
}

// Kafka propagation
func produceWithTrace(ctx context.Context, p *kafka.Producer, topic string, msg []byte) {
    headers := make([]kafka.Header, 0)
    
    // Inject trace context into Kafka headers
    otel.GetTextMapPropagator().Inject(ctx, &kafkaHeaderCarrier{headers: &headers})
    
    p.Produce(&kafka.Message{
        TopicPartition: kafka.TopicPartition{Topic: &topic},
        Headers:        headers,
        Value:          msg,
    }, nil)
}

func consumeWithTrace(msg *kafka.Message) context.Context {
    carrier := &kafkaHeaderCarrier{headers: &msg.Headers}
    return otel.GetTextMapPropagator().Extract(context.Background(), carrier)
}

type kafkaHeaderCarrier struct {
    headers *[]kafka.Header
}

func (c *kafkaHeaderCarrier) Get(key string) string {
    for _, h := range *c.headers {
        if h.Key == key {
            return string(h.Value)
        }
    }
    return ""
}

func (c *kafkaHeaderCarrier) Set(key, value string) {
    *c.headers = append(*c.headers, kafka.Header{Key: key, Value: []byte(value)})
}

func (c *kafkaHeaderCarrier) Keys() []string {
    keys := make([]string, len(*c.headers))
    for i, h := range *c.headers {
        keys[i] = h.Key
    }
    return keys
}
```

<a id="q10"></a>
### Q10: How do you correlate logs, metrics, and traces?
**Answer:**

```go
// Add trace ID to logs
func correlatedLogging(ctx context.Context) {
    span := trace.SpanFromContext(ctx)
    traceID := span.SpanContext().TraceID().String()
    spanID := span.SpanContext().SpanID().String()
    
    // Zerolog
    log.Info().
        Str("trace_id", traceID).
        Str("span_id", spanID).
        Msg("Processing request")
    
    // Zap
    logger.Info("Processing request",
        zap.String("trace_id", traceID),
        zap.String("span_id", spanID),
    )
}

// Middleware for correlation
func CorrelationMiddleware(logger zerolog.Logger) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx := r.Context()
            span := trace.SpanFromContext(ctx)
            
            // Create correlated logger
            correlatedLogger := logger.With().
                Str("trace_id", span.SpanContext().TraceID().String()).
                Str("span_id", span.SpanContext().SpanID().String()).
                Logger()
            
            ctx = correlatedLogger.WithContext(ctx)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// Add trace ID to metrics (exemplars)
func recordMetricWithExemplar(ctx context.Context, histogram *prometheus.HistogramVec, value float64) {
    span := trace.SpanFromContext(ctx)
    
    histogram.WithLabelValues("method").
        (prometheus.ExemplarObserver).ObserveWithExemplar(
            value,
            prometheus.Labels{"trace_id": span.SpanContext().TraceID().String()},
        )
}

// Unified observability setup
type Observability struct {
    Logger  zerolog.Logger
    Tracer  trace.Tracer
    Metrics *Metrics
}

func NewObservability(serviceName string) (*Observability, func(), error) {
    // Init tracer
    cleanup, err := initTracer(context.Background(), serviceName)
    if err != nil {
        return nil, nil, err
    }
    
    // Init logger
    logger := zerolog.New(os.Stdout).With().
        Str("service", serviceName).
        Timestamp().
        Logger()
    
    // Init metrics
    metrics := registerMetrics(serviceName)
    
    return &Observability{
        Logger:  logger,
        Tracer:  otel.Tracer(serviceName),
        Metrics: metrics,
    }, cleanup, nil
}

func (o *Observability) WithContext(ctx context.Context) context.Context {
    span := trace.SpanFromContext(ctx)
    
    // Create logger with trace context
    logger := o.Logger.With().
        Str("trace_id", span.SpanContext().TraceID().String()).
        Str("span_id", span.SpanContext().SpanID().String()).
        Logger()
    
    return logger.WithContext(ctx)
}

/*
Correlation in practice:
1. Trace ID links: log → trace → metrics
2. Query in Grafana: {trace_id="abc123"}
3. Jump from log to trace visualization
4. See metrics at specific trace time

Log entry example:
{
    "timestamp": "2024-01-15T10:30:00Z",
    "level": "info",
    "message": "Order processed",
    "service": "order-service",
    "trace_id": "abc123def456",
    "span_id": "789xyz",
    "order_id": "order-001",
    "duration_ms": 45
}
*/
```

---

[← Back to Go Index](README.md)

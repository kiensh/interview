# Gin Advanced Topics

## Table of Contents

### Request Validation
- [Q1: How do you validate request data in Gin?](#q1)
- [Q2: How do you create custom validators?](#q2)
- [Q3: How do you handle validation errors?](#q3)

### Gin Context
- [Q4: What is gin.Context and what can it do?](#q4)
- [Q5: How do you use context values?](#q5)
- [Q6: How do you handle request lifecycle?](#q6)

### Graceful Shutdown
- [Q7: How do you implement graceful shutdown?](#q7)
- [Q8: How do you handle in-flight requests during shutdown?](#q8)

### Panic Recovery
- [Q9: How does Gin handle panics?](#q9)
- [Q10: How do you customize panic recovery?](#q10)

---

## Request Validation

<a id="q1"></a>
### Q1: How do you validate request data in Gin?
**Answer:**

Gin uses `go-playground/validator` for validation:

```go
// Basic validation tags
type CreateUserRequest struct {
    // Required field
    Name string `json:"name" binding:"required"`
    
    // Required with format validation
    Email string `json:"email" binding:"required,email"`
    
    // Min/max length
    Password string `json:"password" binding:"required,min=8,max=100"`
    
    // Numeric range
    Age int `json:"age" binding:"required,gte=0,lte=150"`
    
    // One of allowed values
    Role string `json:"role" binding:"required,oneof=admin user guest"`
    
    // Optional with validation
    Phone string `json:"phone" binding:"omitempty,e164"`
    
    // URL validation
    Website string `json:"website" binding:"omitempty,url"`
    
    // UUID validation
    RefCode string `json:"ref_code" binding:"omitempty,uuid"`
}

// Common validation tags
/*
required     - Field must be present and not empty
omitempty    - Skip validation if empty
email        - Valid email format
url          - Valid URL format
uuid         - Valid UUID format
alpha        - Alphabetic characters only
alphanum     - Alphanumeric characters only
numeric      - Numeric characters only
min=n        - Minimum length/value
max=n        - Maximum length/value
len=n        - Exact length
gte=n        - Greater than or equal
lte=n        - Less than or equal
gt=n         - Greater than
lt=n         - Less than
eq=n         - Equal to
ne=n         - Not equal to
oneof=a b c  - One of listed values
contains=x   - Contains substring
startswith=x - Starts with
endswith=x   - Ends with
datetime=fmt - Date/time format
e164         - E.164 phone format
*/

// Handler with validation
r.POST("/users", func(c *gin.Context) {
    var req CreateUserRequest
    
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "error": err.Error(),
        })
        return
    }
    
    // Validation passed
    c.JSON(http.StatusCreated, gin.H{"message": "User created"})
})

// Nested struct validation
type Address struct {
    Street  string `json:"street" binding:"required"`
    City    string `json:"city" binding:"required"`
    ZipCode string `json:"zip_code" binding:"required,len=5"`
}

type CreateOrderRequest struct {
    CustomerID string  `json:"customer_id" binding:"required,uuid"`
    Items      []Item  `json:"items" binding:"required,min=1,dive"`
    Shipping   Address `json:"shipping" binding:"required"`
}

type Item struct {
    ProductID string `json:"product_id" binding:"required"`
    Quantity  int    `json:"quantity" binding:"required,min=1"`
}

// dive tag validates each slice element
```

<a id="q2"></a>
### Q2: How do you create custom validators?
**Answer:**

```go
import (
    "github.com/gin-gonic/gin/binding"
    "github.com/go-playground/validator/v10"
)

// Register custom validator at startup
func init() {
    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        // Simple custom validation
        v.RegisterValidation("notblank", notBlankValidator)
        
        // Validation with parameter
        v.RegisterValidation("customdate", customDateValidator)
        
        // Struct-level validation
        v.RegisterStructValidation(userStructValidation, User{})
    }
}

// Custom validator: not blank (not just whitespace)
func notBlankValidator(fl validator.FieldLevel) bool {
    return strings.TrimSpace(fl.Field().String()) != ""
}

// Custom validator with parameter
func customDateValidator(fl validator.FieldLevel) bool {
    dateStr := fl.Field().String()
    format := fl.Param()  // e.g., "2006-01-02"
    
    _, err := time.Parse(format, dateStr)
    return err == nil
}

// Struct-level validation (cross-field)
func userStructValidation(sl validator.StructLevel) {
    user := sl.Current().Interface().(User)
    
    // Password confirmation
    if user.Password != user.ConfirmPassword {
        sl.ReportError(user.ConfirmPassword, "ConfirmPassword", 
            "confirm_password", "eqfield", "Password")
    }
    
    // Conditional validation
    if user.Type == "business" && user.CompanyName == "" {
        sl.ReportError(user.CompanyName, "CompanyName",
            "company_name", "required_if", "Type=business")
    }
}

// Usage
type User struct {
    Username        string `json:"username" binding:"required,notblank"`
    Password        string `json:"password" binding:"required,min=8"`
    ConfirmPassword string `json:"confirm_password" binding:"required"`
    Type            string `json:"type" binding:"required,oneof=personal business"`
    CompanyName     string `json:"company_name"`
    BirthDate       string `json:"birth_date" binding:"omitempty,customdate=2006-01-02"`
}

// Custom tag with custom error message
func registerCustomValidations(v *validator.Validate) {
    v.RegisterValidation("strongpassword", func(fl validator.FieldLevel) bool {
        password := fl.Field().String()
        hasUpper := regexp.MustCompile(`[A-Z]`).MatchString(password)
        hasLower := regexp.MustCompile(`[a-z]`).MatchString(password)
        hasNumber := regexp.MustCompile(`[0-9]`).MatchString(password)
        hasSpecial := regexp.MustCompile(`[!@#$%^&*]`).MatchString(password)
        return hasUpper && hasLower && hasNumber && hasSpecial
    })
}

type RegisterRequest struct {
    Password string `json:"password" binding:"required,min=8,strongpassword"`
}

// Translation for error messages
func registerTranslations(v *validator.Validate, trans ut.Translator) {
    v.RegisterTranslation("strongpassword", trans, 
        func(ut ut.Translator) error {
            return ut.Add("strongpassword", 
                "{0} must contain uppercase, lowercase, number, and special character", true)
        },
        func(ut ut.Translator, fe validator.FieldError) string {
            t, _ := ut.T("strongpassword", fe.Field())
            return t
        },
    )
}
```

<a id="q3"></a>
### Q3: How do you handle validation errors?
**Answer:**

```go
import (
    "github.com/go-playground/validator/v10"
)

// Parse validation errors
func formatValidationErrors(err error) []ValidationError {
    var errors []ValidationError
    
    if validationErrors, ok := err.(validator.ValidationErrors); ok {
        for _, e := range validationErrors {
            errors = append(errors, ValidationError{
                Field:   e.Field(),
                Tag:     e.Tag(),
                Value:   e.Value(),
                Message: getErrorMessage(e),
            })
        }
    }
    
    return errors
}

type ValidationError struct {
    Field   string      `json:"field"`
    Tag     string      `json:"tag"`
    Value   interface{} `json:"value,omitempty"`
    Message string      `json:"message"`
}

func getErrorMessage(e validator.FieldError) string {
    switch e.Tag() {
    case "required":
        return fmt.Sprintf("%s is required", e.Field())
    case "email":
        return fmt.Sprintf("%s must be a valid email", e.Field())
    case "min":
        return fmt.Sprintf("%s must be at least %s characters", e.Field(), e.Param())
    case "max":
        return fmt.Sprintf("%s must be at most %s characters", e.Field(), e.Param())
    case "gte":
        return fmt.Sprintf("%s must be at least %s", e.Field(), e.Param())
    case "lte":
        return fmt.Sprintf("%s must be at most %s", e.Field(), e.Param())
    case "oneof":
        return fmt.Sprintf("%s must be one of: %s", e.Field(), e.Param())
    default:
        return fmt.Sprintf("%s failed on %s validation", e.Field(), e.Tag())
    }
}

// Handler with detailed error response
r.POST("/users", func(c *gin.Context) {
    var req CreateUserRequest
    
    if err := c.ShouldBindJSON(&req); err != nil {
        errors := formatValidationErrors(err)
        
        c.JSON(http.StatusBadRequest, gin.H{
            "error":   "Validation failed",
            "details": errors,
        })
        return
    }
    
    c.JSON(http.StatusCreated, gin.H{"message": "User created"})
})

// Response example:
/*
{
    "error": "Validation failed",
    "details": [
        {
            "field": "Email",
            "tag": "email",
            "value": "invalid",
            "message": "Email must be a valid email"
        },
        {
            "field": "Password",
            "tag": "min",
            "message": "Password must be at least 8 characters"
        }
    ]
}
*/

// Middleware for validation error handling
func ValidationErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        
        if len(c.Errors) > 0 {
            for _, e := range c.Errors {
                if validationErrors, ok := e.Err.(validator.ValidationErrors); ok {
                    errors := formatValidationErrors(validationErrors)
                    c.JSON(http.StatusBadRequest, gin.H{
                        "error":   "Validation failed",
                        "details": errors,
                    })
                    return
                }
            }
        }
    }
}
```

---

## Gin Context

<a id="q4"></a>
### Q4: What is gin.Context and what can it do?
**Answer:**

`gin.Context` is the core of Gin, passed to every handler:

```go
func handler(c *gin.Context) {
    // === REQUEST DATA ===
    // Path parameters
    id := c.Param("id")
    
    // Query parameters
    query := c.Query("q")
    page := c.DefaultQuery("page", "1")
    
    // Form data
    name := c.PostForm("name")
    
    // Headers
    auth := c.GetHeader("Authorization")
    
    // Cookies
    cookie, _ := c.Cookie("session")
    
    // Request body
    body, _ := c.GetRawData()
    
    // Bind data
    var req Request
    c.ShouldBindJSON(&req)
    
    // === REQUEST INFO ===
    c.Request           // *http.Request
    c.Request.Method    // HTTP method
    c.Request.URL       // URL
    c.Request.Header    // Headers
    c.ClientIP()        // Client IP
    c.ContentType()     // Content-Type header
    c.FullPath()        // Matched route pattern
    
    // === RESPONSE ===
    c.Writer            // http.ResponseWriter
    
    // Set headers
    c.Header("X-Custom", "value")
    
    // Set cookies
    c.SetCookie("name", "value", 3600, "/", "domain", false, true)
    
    // Response types
    c.JSON(200, data)
    c.XML(200, data)
    c.String(200, "text")
    c.HTML(200, "template", data)
    c.File("path")
    c.Redirect(302, "url")
    
    // === FLOW CONTROL ===
    c.Next()            // Call next handler
    c.Abort()           // Stop chain
    c.AbortWithStatus(code)
    c.AbortWithStatusJSON(code, obj)
    c.IsAborted()       // Check if aborted
    
    // === CONTEXT VALUES ===
    c.Set("key", value)
    value, exists := c.Get("key")
    c.MustGet("key")    // Panics if not exists
    
    // Typed getters
    c.GetString("key")
    c.GetInt("key")
    c.GetBool("key")
    c.GetStringSlice("key")
    c.GetStringMap("key")
    
    // === ERRORS ===
    c.Error(err)        // Add error
    c.Errors             // All errors
    
    // === METADATA ===
    c.Keys              // All context values
    c.Params            // All path parameters
    c.HandlerName()     // Current handler name
    c.HandlerNames()    // All handler names in chain
}
```

<a id="q5"></a>
### Q5: How do you use context values?
**Answer:**

```go
// Setting values in middleware
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        
        user, err := validateToken(token)
        if err != nil {
            c.AbortWithStatus(http.StatusUnauthorized)
            return
        }
        
        // Store user in context
        c.Set("user", user)
        c.Set("userID", user.ID)
        c.Set("roles", user.Roles)
        
        c.Next()
    }
}

// Using values in handler
func GetProfile(c *gin.Context) {
    // Get with type assertion
    user, exists := c.Get("user")
    if !exists {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Not authenticated"})
        return
    }
    
    u := user.(*User)
    c.JSON(http.StatusOK, u)
}

// Typed getters
func handler(c *gin.Context) {
    userID := c.GetString("userID")
    isAdmin := c.GetBool("isAdmin")
    roles := c.GetStringSlice("roles")
    settings := c.GetStringMap("settings")
    
    // MustGet panics if key doesn't exist
    user := c.MustGet("user").(*User)
}

// Context key type safety
type contextKey string

const (
    UserKey    contextKey = "user"
    RequestKey contextKey = "requestID"
)

func SetUser(c *gin.Context, user *User) {
    c.Set(string(UserKey), user)
}

func GetUser(c *gin.Context) (*User, bool) {
    user, exists := c.Get(string(UserKey))
    if !exists {
        return nil, false
    }
    return user.(*User), true
}

// Passing Go context
func handler(c *gin.Context) {
    // Get Go context (with deadline, cancellation)
    ctx := c.Request.Context()
    
    // Add values and pass to services
    ctx = context.WithValue(ctx, "userID", c.GetString("userID"))
    
    // Use context for database calls
    result, err := db.QueryContext(ctx, "SELECT ...")
    
    // Use context for HTTP calls
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
}

// Copy context for goroutines
func handler(c *gin.Context) {
    // DON'T use c directly in goroutine after handler returns
    // c is reused by Gin
    
    // DO copy the context
    cCopy := c.Copy()
    
    go func() {
        // Safe to use cCopy
        userID := cCopy.GetString("userID")
        processInBackground(userID)
    }()
    
    c.JSON(http.StatusAccepted, gin.H{"status": "processing"})
}
```

<a id="q6"></a>
### Q6: How do you handle request lifecycle?
**Answer:**

```go
// Request lifecycle
/*
1. Client sends request
2. Router matches route
3. Middleware chain executes (before)
4. Handler executes
5. Middleware chain executes (after)
6. Response sent to client
*/

func LifecycleMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // BEFORE handler
        requestID := uuid.New().String()
        c.Set("requestID", requestID)
        c.Header("X-Request-ID", requestID)
        
        start := time.Now()
        log.Printf("[%s] Started %s %s", requestID, c.Request.Method, c.Request.URL.Path)
        
        // Execute next handlers
        c.Next()
        
        // AFTER handler (response has been written)
        duration := time.Since(start)
        status := c.Writer.Status()
        log.Printf("[%s] Completed %d in %v", requestID, status, duration)
        
        // Check for errors
        if len(c.Errors) > 0 {
            log.Printf("[%s] Errors: %v", requestID, c.Errors)
        }
    }
}

// Abort flow
func AuthRequired() gin.HandlerFunc {
    return func(c *gin.Context) {
        if !isAuthenticated(c) {
            // This stops the chain
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Authentication required",
            })
            return
            // Next() is NOT called
            // Remaining handlers are skipped
            // But "after" code in previous middleware still runs
        }
        c.Next()
    }
}

// Multiple middleware ordering
r.Use(Middleware1()) // Runs 1st (before), 3rd (after)
r.Use(Middleware2()) // Runs 2nd (before), 2nd (after)
r.GET("/", Handler)  // Runs in the middle

/*
Order of execution:
Middleware1 BEFORE → Middleware2 BEFORE → Handler → Middleware2 AFTER → Middleware1 AFTER
*/

// Cleanup with defer
func ResourceMiddleware(db *sql.DB) gin.HandlerFunc {
    return func(c *gin.Context) {
        conn, err := db.Conn(c.Request.Context())
        if err != nil {
            c.AbortWithError(500, err)
            return
        }
        defer conn.Close()  // Always cleaned up
        
        c.Set("dbConn", conn)
        c.Next()
    }
}

// Timeout handling
func TimeoutMiddleware(timeout time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx, cancel := context.WithTimeout(c.Request.Context(), timeout)
        defer cancel()
        
        c.Request = c.Request.WithContext(ctx)
        
        done := make(chan struct{})
        go func() {
            c.Next()
            close(done)
        }()
        
        select {
        case <-done:
            // Completed normally
        case <-ctx.Done():
            c.AbortWithStatusJSON(http.StatusGatewayTimeout, gin.H{
                "error": "Request timeout",
            })
        }
    }
}
```

---

## Graceful Shutdown

<a id="q7"></a>
### Q7: How do you implement graceful shutdown?
**Answer:**

```go
package main

import (
    "context"
    "log"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"
    
    "github.com/gin-gonic/gin"
)

func main() {
    router := gin.Default()
    
    router.GET("/", func(c *gin.Context) {
        time.Sleep(5 * time.Second)  // Simulate long request
        c.String(http.StatusOK, "Hello World")
    })
    
    srv := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }
    
    // Start server in goroutine
    go func() {
        if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            log.Fatalf("listen: %s\n", err)
        }
    }()
    
    // Wait for interrupt signal
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    log.Println("Shutting down server...")
    
    // Give outstanding requests time to complete
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    if err := srv.Shutdown(ctx); err != nil {
        log.Fatal("Server forced to shutdown:", err)
    }
    
    log.Println("Server exiting")
}

// With cleanup functions
func main() {
    router := gin.Default()
    
    // Initialize resources
    db, _ := sql.Open("postgres", connStr)
    cache := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
    
    srv := &http.Server{
        Addr:    ":8080",
        Handler: router,
    }
    
    go func() {
        if err := srv.ListenAndServe(); err != http.ErrServerClosed {
            log.Fatal(err)
        }
    }()
    
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    log.Println("Shutdown signal received")
    
    // Create shutdown context
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    
    // Shutdown HTTP server
    if err := srv.Shutdown(ctx); err != nil {
        log.Printf("HTTP server shutdown error: %v", err)
    }
    
    // Close database
    if err := db.Close(); err != nil {
        log.Printf("Database close error: %v", err)
    }
    
    // Close cache
    if err := cache.Close(); err != nil {
        log.Printf("Cache close error: %v", err)
    }
    
    log.Println("Server shutdown complete")
}
```

<a id="q8"></a>
### Q8: How do you handle in-flight requests during shutdown?
**Answer:**

```go
// Track active requests
type Server struct {
    router     *gin.Engine
    httpServer *http.Server
    wg         sync.WaitGroup
}

func (s *Server) RequestTracker() gin.HandlerFunc {
    return func(c *gin.Context) {
        s.wg.Add(1)
        defer s.wg.Done()
        c.Next()
    }
}

func (s *Server) Shutdown(ctx context.Context) error {
    // Stop accepting new connections
    if err := s.httpServer.Shutdown(ctx); err != nil {
        return err
    }
    
    // Wait for in-flight requests
    done := make(chan struct{})
    go func() {
        s.wg.Wait()
        close(done)
    }()
    
    select {
    case <-done:
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

// Health check that considers shutdown
type HealthChecker struct {
    shuttingDown atomic.Bool
}

func (h *HealthChecker) SetShuttingDown() {
    h.shuttingDown.Store(true)
}

func (h *HealthChecker) HealthHandler(c *gin.Context) {
    if h.shuttingDown.Load() {
        // Return unhealthy so load balancer stops sending traffic
        c.JSON(http.StatusServiceUnavailable, gin.H{
            "status": "shutting_down",
        })
        return
    }
    
    c.JSON(http.StatusOK, gin.H{
        "status": "healthy",
    })
}

func main() {
    health := &HealthChecker{}
    router := gin.Default()
    router.GET("/health", health.HealthHandler)
    
    srv := &http.Server{Addr: ":8080", Handler: router}
    
    go srv.ListenAndServe()
    
    quit := make(chan os.Signal, 1)
    signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
    <-quit
    
    // Mark as shutting down (load balancer will stop sending traffic)
    health.SetShuttingDown()
    
    // Wait for load balancer to detect unhealthy
    time.Sleep(5 * time.Second)
    
    // Now shutdown
    ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
    defer cancel()
    srv.Shutdown(ctx)
}

// Draining connections before shutdown
func main() {
    router := gin.Default()
    
    // Track connections
    connTracker := &ConnectionTracker{}
    router.Use(connTracker.Middleware())
    
    srv := &http.Server{
        Addr:    ":8080",
        Handler: router,
        ConnState: func(conn net.Conn, state http.ConnState) {
            switch state {
            case http.StateNew:
                connTracker.Add()
            case http.StateClosed, http.StateHijacked:
                connTracker.Remove()
            }
        },
    }
    
    // ... shutdown handling
}

type ConnectionTracker struct {
    count atomic.Int64
}

func (ct *ConnectionTracker) Add() {
    ct.count.Add(1)
}

func (ct *ConnectionTracker) Remove() {
    ct.count.Add(-1)
}

func (ct *ConnectionTracker) Count() int64 {
    return ct.count.Load()
}

func (ct *ConnectionTracker) WaitForDrain(timeout time.Duration) bool {
    deadline := time.Now().Add(timeout)
    for ct.Count() > 0 {
        if time.Now().After(deadline) {
            return false
        }
        time.Sleep(100 * time.Millisecond)
    }
    return true
}
```

---

## Panic Recovery

<a id="q9"></a>
### Q9: How does Gin handle panics?
**Answer:**

```go
// gin.Recovery() middleware catches panics
r := gin.Default()  // Includes Recovery middleware

// Or add manually
r := gin.New()
r.Use(gin.Recovery())

// What Recovery does:
// 1. Catches panic
// 2. Logs stack trace
// 3. Returns 500 Internal Server Error
// 4. Doesn't crash the server

// Example panic
r.GET("/panic", func(c *gin.Context) {
    panic("Something went wrong!")
})

// Response:
// HTTP 500 Internal Server Error
// (In development mode, includes stack trace)

// Panic in goroutine (NOT recovered by Gin)
r.GET("/async-panic", func(c *gin.Context) {
    go func() {
        panic("This will crash the server!")  // NOT recovered!
    }()
    c.String(200, "OK")
})

// Safe async with recovery
r.GET("/safe-async", func(c *gin.Context) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("Recovered from panic: %v", r)
            }
        }()
        panic("This is safely recovered")
    }()
    c.String(200, "OK")
})
```

<a id="q10"></a>
### Q10: How do you customize panic recovery?
**Answer:**

```go
// Custom recovery handler
r.Use(gin.CustomRecovery(func(c *gin.Context, recovered interface{}) {
    // Log the panic
    log.Printf("Panic recovered: %v", recovered)
    log.Printf("Stack trace:\n%s", debug.Stack())
    
    // Custom response
    c.JSON(http.StatusInternalServerError, gin.H{
        "error":      "Internal server error",
        "request_id": c.GetString("requestID"),
    })
}))

// Recovery with alerting
func AlertingRecovery() gin.HandlerFunc {
    return gin.CustomRecoveryWithWriter(gin.DefaultErrorWriter, func(c *gin.Context, err interface{}) {
        // Get request info
        requestID := c.GetString("requestID")
        path := c.Request.URL.Path
        method := c.Request.Method
        
        // Log
        log.Printf("[PANIC] %s %s - %v", method, path, err)
        
        // Send alert (Slack, PagerDuty, etc.)
        go sendAlert(AlertData{
            Type:      "panic",
            RequestID: requestID,
            Path:      path,
            Method:    method,
            Error:     fmt.Sprint(err),
            Stack:     string(debug.Stack()),
        })
        
        // Response
        c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
            "error":      "An unexpected error occurred",
            "request_id": requestID,
        })
    })
}

// Recovery with metrics
func MetricsRecovery(panicCounter prometheus.Counter) gin.HandlerFunc {
    return gin.CustomRecovery(func(c *gin.Context, err interface{}) {
        // Increment panic counter
        panicCounter.Inc()
        
        c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
            "error": "Internal server error",
        })
    })
}

// Recovery that includes error in response (development only)
func DevRecovery() gin.HandlerFunc {
    return gin.CustomRecovery(func(c *gin.Context, err interface{}) {
        c.JSON(http.StatusInternalServerError, gin.H{
            "error":   "Internal server error",
            "panic":   fmt.Sprint(err),
            "stack":   string(debug.Stack()),
        })
    })
}

// Conditional recovery based on environment
func RecoveryMiddleware() gin.HandlerFunc {
    if os.Getenv("ENV") == "development" {
        return DevRecovery()
    }
    return AlertingRecovery()
}

// Recovery with sentry
import "github.com/getsentry/sentry-go"

func SentryRecovery() gin.HandlerFunc {
    return gin.CustomRecovery(func(c *gin.Context, err interface{}) {
        sentry.CurrentHub().Recover(err)
        sentry.Flush(time.Second * 5)
        
        c.AbortWithStatus(http.StatusInternalServerError)
    })
}
```

---

[← Back to Go Index](README.md)

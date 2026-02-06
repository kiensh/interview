# Gin Framework Basics

## Table of Contents

### Introduction
- [Q1: What is Gin and why use it?](#q1)
- [Q2: How do you set up a basic Gin application?](#q2)

### Routing
- [Q3: How do you define routes in Gin?](#q3)
- [Q4: How do you handle route parameters and query strings?](#q4)
- [Q5: How do you group routes?](#q5)

### Middleware
- [Q6: What is middleware in Gin?](#q6)
- [Q7: How do you create custom middleware?](#q7)
- [Q8: What are common built-in middlewares?](#q8)

### Request & Response
- [Q9: How do you bind request data?](#q9)
- [Q10: How do you send different types of responses?](#q10)
- [Q11: How do you handle errors in Gin?](#q11)
- [Q12: How do you use Gin's error handling features?](#q12)

---

## Introduction

<a id="q1"></a>
### Q1: What is Gin and why use it?
**Answer:**

Gin is a high-performance HTTP web framework for Go:

| Feature | Description |
|---------|-------------|
| Performance | One of the fastest Go frameworks |
| Middleware | Built-in support for middleware chain |
| Routing | Fast radix tree based routing |
| Binding | Easy request data binding and validation |
| Rendering | JSON, XML, HTML response rendering |
| Error handling | Centralized error management |

**When to use Gin:**
- Building REST APIs
- Microservices
- Web applications
- Need for performance
- Familiar MVC-style routing

**Gin vs net/http:**
```go
// net/http
http.HandleFunc("/users", func(w http.ResponseWriter, r *http.Request) {
    if r.Method != http.MethodGet {
        http.Error(w, "Method not allowed", http.StatusMethodNotAllowed)
        return
    }
    json.NewEncoder(w).Encode(users)
})

// Gin - cleaner, more features
r := gin.Default()
r.GET("/users", func(c *gin.Context) {
    c.JSON(http.StatusOK, users)
})
```

<a id="q2"></a>
### Q2: How do you set up a basic Gin application?
**Answer:**

```go
package main

import (
    "github.com/gin-gonic/gin"
    "net/http"
)

func main() {
    // Create router with default middleware (Logger, Recovery)
    r := gin.Default()
    
    // Or create without middleware
    r := gin.New()
    r.Use(gin.Logger())
    r.Use(gin.Recovery())
    
    // Define routes
    r.GET("/ping", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{
            "message": "pong",
        })
    })
    
    // Run server
    r.Run(":8080")  // Default: localhost:8080
}

// Production setup
func main() {
    // Set release mode
    gin.SetMode(gin.ReleaseMode)
    
    r := gin.New()
    
    // Custom logger
    r.Use(gin.LoggerWithConfig(gin.LoggerConfig{
        SkipPaths: []string{"/health"},
    }))
    r.Use(gin.Recovery())
    
    // Trust only specific proxies
    r.SetTrustedProxies([]string{"192.168.1.0/24"})
    
    // Health check
    r.GET("/health", func(c *gin.Context) {
        c.String(http.StatusOK, "OK")
    })
    
    // Custom HTTP server for more control
    srv := &http.Server{
        Addr:         ":8080",
        Handler:      r,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 10 * time.Second,
    }
    
    srv.ListenAndServe()
}

// Application structure
/*
myapp/
├── main.go
├── config/
│   └── config.go
├── handlers/
│   ├── user.go
│   └── order.go
├── middleware/
│   ├── auth.go
│   └── logging.go
├── models/
│   └── user.go
├── routes/
│   └── routes.go
└── services/
    └── user_service.go
*/

// routes/routes.go
func SetupRoutes(r *gin.Engine, h *handlers.Handler) {
    api := r.Group("/api/v1")
    {
        users := api.Group("/users")
        {
            users.GET("", h.ListUsers)
            users.POST("", h.CreateUser)
            users.GET("/:id", h.GetUser)
            users.PUT("/:id", h.UpdateUser)
            users.DELETE("/:id", h.DeleteUser)
        }
    }
}
```

---

## Routing

<a id="q3"></a>
### Q3: How do you define routes in Gin?
**Answer:**

```go
func setupRoutes(r *gin.Engine) {
    // HTTP Methods
    r.GET("/users", listUsers)
    r.POST("/users", createUser)
    r.PUT("/users/:id", updateUser)
    r.PATCH("/users/:id", patchUser)
    r.DELETE("/users/:id", deleteUser)
    r.HEAD("/users", headUsers)
    r.OPTIONS("/users", optionsUsers)
    
    // Handle multiple methods
    r.Any("/resource", handleAny)  // All methods
    r.Match([]string{"GET", "POST"}, "/match", handleMatch)
    
    // Static files
    r.Static("/assets", "./assets")
    r.StaticFS("/static", http.Dir("static"))
    r.StaticFile("/favicon.ico", "./favicon.ico")
    
    // NoRoute - 404 handler
    r.NoRoute(func(c *gin.Context) {
        c.JSON(http.StatusNotFound, gin.H{
            "error": "Resource not found",
        })
    })
    
    // NoMethod - 405 handler
    r.NoMethod(func(c *gin.Context) {
        c.JSON(http.StatusMethodNotAllowed, gin.H{
            "error": "Method not allowed",
        })
    })
}

// Handler function signature
func listUsers(c *gin.Context) {
    users := []User{...}
    c.JSON(http.StatusOK, users)
}

// Handler as struct method
type UserHandler struct {
    service UserService
}

func (h *UserHandler) List(c *gin.Context) {
    users, err := h.service.List(c.Request.Context())
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, users)
}

// Route with handler struct
handler := &UserHandler{service: userService}
r.GET("/users", handler.List)
r.POST("/users", handler.Create)
```

<a id="q4"></a>
### Q4: How do you handle route parameters and query strings?
**Answer:**

```go
// Path parameters
r.GET("/users/:id", func(c *gin.Context) {
    id := c.Param("id")
    c.JSON(http.StatusOK, gin.H{"id": id})
})

// Multiple parameters
r.GET("/users/:userID/posts/:postID", func(c *gin.Context) {
    userID := c.Param("userID")
    postID := c.Param("postID")
    c.JSON(http.StatusOK, gin.H{
        "user_id": userID,
        "post_id": postID,
    })
})

// Wildcard parameter (catches rest of path)
r.GET("/files/*filepath", func(c *gin.Context) {
    filepath := c.Param("filepath")
    // /files/docs/readme.md -> filepath = "/docs/readme.md"
    c.String(http.StatusOK, "File: %s", filepath)
})

// Query parameters
// GET /search?q=golang&page=1&limit=10
r.GET("/search", func(c *gin.Context) {
    query := c.Query("q")              // "golang"
    page := c.DefaultQuery("page", "1") // "1" (with default)
    limit := c.Query("limit")          // "10"
    
    c.JSON(http.StatusOK, gin.H{
        "query": query,
        "page":  page,
        "limit": limit,
    })
})

// Query array
// GET /tags?tag=go&tag=web&tag=api
r.GET("/tags", func(c *gin.Context) {
    tags := c.QueryArray("tag")  // ["go", "web", "api"]
    c.JSON(http.StatusOK, gin.H{"tags": tags})
})

// Query map
// GET /filters?filter[status]=active&filter[type]=user
r.GET("/filters", func(c *gin.Context) {
    filters := c.QueryMap("filter")  // map[string]string
    c.JSON(http.StatusOK, gin.H{"filters": filters})
})

// Bind query to struct
type SearchParams struct {
    Query string `form:"q" binding:"required"`
    Page  int    `form:"page,default=1"`
    Limit int    `form:"limit,default=10"`
}

r.GET("/search", func(c *gin.Context) {
    var params SearchParams
    if err := c.ShouldBindQuery(&params); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, params)
})
```

<a id="q5"></a>
### Q5: How do you group routes?
**Answer:**

```go
func setupRoutes(r *gin.Engine) {
    // Basic grouping
    api := r.Group("/api")
    {
        api.GET("/users", listUsers)
        api.POST("/users", createUser)
    }
    
    // Versioned API
    v1 := r.Group("/api/v1")
    {
        v1.GET("/users", v1ListUsers)
    }
    
    v2 := r.Group("/api/v2")
    {
        v2.GET("/users", v2ListUsers)
    }
    
    // Nested groups
    admin := r.Group("/admin")
    {
        admin.Use(AuthMiddleware())  // Group middleware
        
        users := admin.Group("/users")
        {
            users.GET("", adminListUsers)
            users.DELETE("/:id", adminDeleteUser)
        }
        
        settings := admin.Group("/settings")
        {
            settings.GET("", getSettings)
            settings.PUT("", updateSettings)
        }
    }
    
    // Group with middleware
    authorized := r.Group("/")
    authorized.Use(AuthRequired())
    {
        authorized.POST("/orders", createOrder)
        authorized.GET("/profile", getProfile)
    }
    
    // Public routes
    public := r.Group("/public")
    {
        public.GET("/products", listProducts)
        public.GET("/categories", listCategories)
    }
}

// Middleware specific to group
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Authorization required",
            })
            return
        }
        c.Next()
    }
}

// Route organization by domain
func SetupUserRoutes(rg *gin.RouterGroup) {
    users := rg.Group("/users")
    {
        users.GET("", ListUsers)
        users.POST("", CreateUser)
        users.GET("/:id", GetUser)
        users.PUT("/:id", UpdateUser)
        users.DELETE("/:id", DeleteUser)
    }
}

func SetupOrderRoutes(rg *gin.RouterGroup) {
    orders := rg.Group("/orders")
    {
        orders.GET("", ListOrders)
        orders.POST("", CreateOrder)
        orders.GET("/:id", GetOrder)
    }
}

// main.go
api := r.Group("/api/v1")
SetupUserRoutes(api)
SetupOrderRoutes(api)
```

---

## Middleware

<a id="q6"></a>
### Q6: What is middleware in Gin?
**Answer:**

Middleware are functions that run before/after handlers:

```go
/*
Request Flow:
Client → Middleware1 → Middleware2 → Handler → Middleware2 → Middleware1 → Client

Middleware Chain:
┌─────────────────────────────────────────────────────────────┐
│ Logger (before) → Auth (before) → Handler → Auth (after) → Logger (after) │
└─────────────────────────────────────────────────────────────┘
*/

// Middleware signature
func MyMiddleware() gin.HandlerFunc {
    // Setup (runs once when middleware is registered)
    
    return func(c *gin.Context) {
        // Before handler
        start := time.Now()
        
        c.Next()  // Call next handler
        
        // After handler
        duration := time.Since(start)
        log.Printf("Request took %v", duration)
    }
}

// Using middleware
r := gin.New()

// Global middleware (applies to all routes)
r.Use(gin.Logger())
r.Use(gin.Recovery())
r.Use(MyMiddleware())

// Group middleware
api := r.Group("/api")
api.Use(AuthMiddleware())

// Route-specific middleware
r.GET("/admin", AdminOnly(), adminHandler)

// Multiple middlewares
r.GET("/protected", Auth(), RateLimit(), Audit(), handler)
```

<a id="q7"></a>
### Q7: How do you create custom middleware?
**Answer:**

```go
// Logging middleware
func LoggingMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path
        method := c.Request.Method
        
        c.Next()  // Process request
        
        latency := time.Since(start)
        status := c.Writer.Status()
        
        log.Printf("[%s] %s %s %d %v",
            method, path, c.ClientIP(), status, latency)
    }
}

// Authentication middleware
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := c.GetHeader("Authorization")
        if token == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Missing authorization token",
            })
            return
        }
        
        // Validate token
        userID, err := validateToken(token)
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Invalid token",
            })
            return
        }
        
        // Set user in context
        c.Set("userID", userID)
        c.Next()
    }
}

// Rate limiting middleware
func RateLimitMiddleware(rps int) gin.HandlerFunc {
    limiter := rate.NewLimiter(rate.Limit(rps), rps)
    
    return func(c *gin.Context) {
        if !limiter.Allow() {
            c.AbortWithStatusJSON(http.StatusTooManyRequests, gin.H{
                "error": "Rate limit exceeded",
            })
            return
        }
        c.Next()
    }
}

// CORS middleware
func CORSMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Access-Control-Allow-Origin", "*")
        c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
        c.Header("Access-Control-Allow-Headers", "Content-Type, Authorization")
        
        if c.Request.Method == "OPTIONS" {
            c.AbortWithStatus(http.StatusNoContent)
            return
        }
        
        c.Next()
    }
}

// Request ID middleware
func RequestIDMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        requestID := c.GetHeader("X-Request-ID")
        if requestID == "" {
            requestID = uuid.New().String()
        }
        
        c.Set("requestID", requestID)
        c.Header("X-Request-ID", requestID)
        c.Next()
    }
}

// Timeout middleware
func TimeoutMiddleware(timeout time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx, cancel := context.WithTimeout(c.Request.Context(), timeout)
        defer cancel()
        
        c.Request = c.Request.WithContext(ctx)
        
        finished := make(chan struct{})
        go func() {
            c.Next()
            close(finished)
        }()
        
        select {
        case <-finished:
        case <-ctx.Done():
            c.AbortWithStatusJSON(http.StatusGatewayTimeout, gin.H{
                "error": "Request timeout",
            })
        }
    }
}

// Error handling middleware
func ErrorHandlerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        
        // Check for errors after handler
        if len(c.Errors) > 0 {
            err := c.Errors.Last()
            c.JSON(-1, gin.H{"error": err.Error()})
        }
    }
}
```

<a id="q8"></a>
### Q8: What are common built-in middlewares?
**Answer:**

```go
import "github.com/gin-gonic/gin"

// Logger - logs request details
r.Use(gin.Logger())

// Custom logger config
r.Use(gin.LoggerWithConfig(gin.LoggerConfig{
    Formatter: func(param gin.LogFormatterParams) string {
        return fmt.Sprintf("[%s] %s %s %d %s\n",
            param.TimeStamp.Format(time.RFC3339),
            param.Method,
            param.Path,
            param.StatusCode,
            param.Latency,
        )
    },
    Output:    os.Stdout,
    SkipPaths: []string{"/health", "/metrics"},
}))

// Recovery - recovers from panics
r.Use(gin.Recovery())

// Custom recovery
r.Use(gin.CustomRecovery(func(c *gin.Context, recovered interface{}) {
    if err, ok := recovered.(string); ok {
        c.JSON(http.StatusInternalServerError, gin.H{
            "error": err,
        })
    }
    c.AbortWithStatus(http.StatusInternalServerError)
}))

// Basic Auth
r.Use(gin.BasicAuth(gin.Accounts{
    "admin": "secret",
    "user":  "password",
}))

// Basic auth for specific routes
authorized := r.Group("/admin", gin.BasicAuth(gin.Accounts{
    "admin": "admin123",
}))

// Third-party middlewares
import "github.com/gin-contrib/cors"
import "github.com/gin-contrib/gzip"
import "github.com/gin-contrib/requestid"

// CORS
r.Use(cors.New(cors.Config{
    AllowOrigins:     []string{"https://example.com"},
    AllowMethods:     []string{"GET", "POST", "PUT", "DELETE"},
    AllowHeaders:     []string{"Origin", "Content-Type", "Authorization"},
    ExposeHeaders:    []string{"Content-Length"},
    AllowCredentials: true,
    MaxAge:           12 * time.Hour,
}))

// Gzip compression
r.Use(gzip.Gzip(gzip.DefaultCompression))

// Request ID
r.Use(requestid.New())
```

---

## Request & Response

<a id="q9"></a>
### Q9: How do you bind request data?
**Answer:**

```go
// JSON binding
type CreateUserRequest struct {
    Name     string `json:"name" binding:"required"`
    Email    string `json:"email" binding:"required,email"`
    Password string `json:"password" binding:"required,min=8"`
    Age      int    `json:"age" binding:"gte=0,lte=130"`
}

r.POST("/users", func(c *gin.Context) {
    var req CreateUserRequest
    
    // ShouldBindJSON - returns error, doesn't abort
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // BindJSON - aborts with 400 on error
    // if err := c.BindJSON(&req); err != nil { return }
    
    c.JSON(http.StatusCreated, req)
})

// Form binding
type LoginForm struct {
    Username string `form:"username" binding:"required"`
    Password string `form:"password" binding:"required"`
}

r.POST("/login", func(c *gin.Context) {
    var form LoginForm
    if err := c.ShouldBind(&form); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    // ShouldBind auto-detects content type
})

// URI binding (path parameters)
type UserURI struct {
    ID int `uri:"id" binding:"required,min=1"`
}

r.GET("/users/:id", func(c *gin.Context) {
    var uri UserURI
    if err := c.ShouldBindUri(&uri); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, gin.H{"id": uri.ID})
})

// Header binding
type AuthHeader struct {
    Token string `header:"Authorization" binding:"required"`
}

r.GET("/protected", func(c *gin.Context) {
    var header AuthHeader
    if err := c.ShouldBindHeader(&header); err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Missing token"})
        return
    }
})

// Multiple bindings
type Request struct {
    ID     int    `uri:"id" binding:"required"`
    Query  string `form:"q"`
    Filter string `form:"filter"`
}

r.GET("/items/:id", func(c *gin.Context) {
    var req Request
    if err := c.ShouldBindUri(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    if err := c.ShouldBindQuery(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
})

// Custom validator
import "github.com/go-playground/validator/v10"

var validate *validator.Validate

func init() {
    validate = validator.New()
    validate.RegisterValidation("customtag", customValidation)
}

func customValidation(fl validator.FieldLevel) bool {
    return fl.Field().String() != "invalid"
}

// Raw body
r.POST("/raw", func(c *gin.Context) {
    body, err := c.GetRawData()
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    c.String(http.StatusOK, string(body))
})
```

<a id="q10"></a>
### Q10: How do you send different types of responses?
**Answer:**

```go
// JSON response
r.GET("/json", func(c *gin.Context) {
    c.JSON(http.StatusOK, gin.H{
        "message": "Hello",
        "status":  "success",
    })
})

// Struct to JSON
type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

c.JSON(http.StatusOK, User{ID: 1, Name: "Alice"})

// IndentedJSON (pretty print)
c.IndentedJSON(http.StatusOK, data)

// SecureJSON (prevents JSON hijacking)
c.SecureJSON(http.StatusOK, data)
// Output: while(1);{"data":"value"}

// JSONP (for cross-domain requests)
c.JSONP(http.StatusOK, data)
// With ?callback=fn: fn({"data":"value"})

// AsciiJSON (escapes non-ASCII)
c.AsciiJSON(http.StatusOK, gin.H{"message": "こんにちは"})

// PureJSON (no HTML escaping)
c.PureJSON(http.StatusOK, gin.H{"html": "<b>Hello</b>"})

// XML response
c.XML(http.StatusOK, gin.H{"message": "Hello"})

// YAML response
c.YAML(http.StatusOK, gin.H{"message": "Hello"})

// String response
c.String(http.StatusOK, "Hello %s", name)

// HTML response
c.HTML(http.StatusOK, "index.html", gin.H{
    "title": "Home",
})

// File response
c.File("./files/document.pdf")

// File attachment (download)
c.FileAttachment("./files/document.pdf", "download.pdf")

// Data from reader
c.DataFromReader(http.StatusOK, contentLength, contentType, reader, extraHeaders)

// Redirect
c.Redirect(http.StatusMovedPermanently, "https://example.com")
c.Redirect(http.StatusFound, "/login")

// Set headers
c.Header("X-Custom-Header", "value")
c.Header("Content-Type", "application/json")

// Set cookie
c.SetCookie("token", "abc123", 3600, "/", "localhost", false, true)

// Stream response
c.Stream(func(w io.Writer) bool {
    w.Write([]byte("data: message\n\n"))
    return true  // Continue streaming
})

// No content
c.Status(http.StatusNoContent)
```

<a id="q11"></a>
### Q11: How do you handle errors in Gin?
**Answer:**

```go
// Basic error response
r.GET("/user/:id", func(c *gin.Context) {
    user, err := userService.GetByID(c.Param("id"))
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
            return
        }
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Internal error"})
        return
    }
    c.JSON(http.StatusOK, user)
})

// Using c.Error() for error collection
r.GET("/user/:id", func(c *gin.Context) {
    user, err := userService.GetByID(c.Param("id"))
    if err != nil {
        c.Error(err)  // Add error to context
        c.AbortWithStatusJSON(http.StatusInternalServerError, gin.H{
            "error": "Failed to get user",
        })
        return
    }
    c.JSON(http.StatusOK, user)
})

// Custom error type
type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
}

func (e APIError) Error() string {
    return e.Message
}

var (
    ErrNotFound     = APIError{Code: "NOT_FOUND", Message: "Resource not found"}
    ErrUnauthorized = APIError{Code: "UNAUTHORIZED", Message: "Authentication required"}
    ErrBadRequest   = APIError{Code: "BAD_REQUEST", Message: "Invalid request"}
)

// Error handling middleware
func ErrorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        
        for _, err := range c.Errors {
            switch e := err.Err.(type) {
            case APIError:
                c.JSON(getStatusCode(e.Code), e)
            default:
                c.JSON(http.StatusInternalServerError, gin.H{
                    "code":    "INTERNAL_ERROR",
                    "message": "Internal server error",
                })
            }
        }
    }
}

// Abort vs Next
func middleware(c *gin.Context) {
    if unauthorized {
        c.Abort()  // Stop middleware chain
        return
    }
    c.Next()  // Continue to next handler
}

// AbortWith variants
c.Abort()                                    // Just abort
c.AbortWithStatus(http.StatusUnauthorized)   // Abort with status
c.AbortWithStatusJSON(401, gin.H{})          // Abort with JSON
c.AbortWithError(500, err)                   // Abort with error
```

<a id="q12"></a>
### Q12: How do you use Gin's error handling features?
**Answer:**

```go
// gin.Error structure
type Error struct {
    Err  error       // Original error
    Type ErrorType   // Error type flag
    Meta interface{} // Additional metadata
}

// Error types
const (
    ErrorTypeBind    = 1 << 0  // Binding error
    ErrorTypeRender  = 1 << 1  // Rendering error
    ErrorTypePrivate = 1 << 2  // Private error (not shown to client)
    ErrorTypePublic  = 1 << 3  // Public error (shown to client)
)

// Adding errors
r.POST("/users", func(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        // Add binding error
        c.Error(err).SetType(gin.ErrorTypeBind)
        c.AbortWithStatus(http.StatusBadRequest)
        return
    }
    
    if err := validate(req); err != nil {
        // Add public error (shown to client)
        c.Error(err).SetType(gin.ErrorTypePublic)
        c.AbortWithStatus(http.StatusBadRequest)
        return
    }
    
    user, err := userService.Create(req)
    if err != nil {
        // Add private error (for logging only)
        c.Error(err).SetType(gin.ErrorTypePrivate).SetMeta(gin.H{
            "request": req,
        })
        c.AbortWithStatus(http.StatusInternalServerError)
        return
    }
    
    c.JSON(http.StatusCreated, user)
})

// Error handling middleware
func ErrorMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        
        if len(c.Errors) > 0 {
            // Log all errors
            for _, e := range c.Errors {
                log.Printf("Error: %v, Type: %d, Meta: %v", 
                    e.Err, e.Type, e.Meta)
            }
            
            // Return only public errors
            publicErrors := c.Errors.ByType(gin.ErrorTypePublic)
            if len(publicErrors) > 0 {
                c.JSON(-1, gin.H{
                    "errors": publicErrors.Errors(),
                })
                return
            }
            
            // Generic error for private errors
            c.JSON(-1, gin.H{
                "error": "An error occurred",
            })
        }
    }
}

// Checking errors
errors := c.Errors                    // All errors
bindErrors := c.Errors.ByType(gin.ErrorTypeBind)
lastError := c.Errors.Last()
errorStrings := c.Errors.Errors()     // []string
errorJSON := c.Errors.JSON()          // interface{}

// Recovery with custom error handling
r.Use(gin.CustomRecovery(func(c *gin.Context, err interface{}) {
    // Log the panic
    log.Printf("Panic recovered: %v\n%s", err, debug.Stack())
    
    c.JSON(http.StatusInternalServerError, gin.H{
        "error": "Internal server error",
    })
}))
```

---

[← Back to Go Index](README.md)

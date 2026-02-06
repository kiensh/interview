# Gin Features

## Table of Contents

### Templates
- [Q1: How do you use HTML templates in Gin?](#q1)
- [Q2: How do you use custom template functions?](#q2)

### Caching
- [Q3: How do you implement response caching?](#q3)
- [Q4: What caching strategies are available?](#q4)

### Compression
- [Q5: How do you enable gzip compression?](#q5)

### Security Headers
- [Q6: How do you add security headers?](#q6)

### File Handling
- [Q7: How do you handle file uploads?](#q7)
- [Q8: How do you serve static files and downloads?](#q8)

### Sessions & Cookies
- [Q9: How do you manage sessions in Gin?](#q9)
- [Q10: How do you configure cookies securely?](#q10)
- [Q11: What session storage backends are available?](#q11)

---

## Templates

<a id="q1"></a>
### Q1: How do you use HTML templates in Gin?
**Answer:**

```go
// Load templates
func main() {
    r := gin.Default()
    
    // Load all templates in directory
    r.LoadHTMLGlob("templates/*")
    
    // Or load specific files
    r.LoadHTMLFiles("templates/index.html", "templates/user.html")
    
    // Render template
    r.GET("/", func(c *gin.Context) {
        c.HTML(http.StatusOK, "index.html", gin.H{
            "title":   "Home",
            "message": "Welcome!",
        })
    })
    
    r.Run()
}

// templates/index.html
/*
<!DOCTYPE html>
<html>
<head>
    <title>{{ .title }}</title>
</head>
<body>
    <h1>{{ .message }}</h1>
</body>
</html>
*/

// Nested templates with layouts
// templates/layouts/base.html
/*
{{ define "base" }}
<!DOCTYPE html>
<html>
<head>
    <title>{{ template "title" . }}</title>
</head>
<body>
    <nav>{{ template "nav" . }}</nav>
    <main>{{ template "content" . }}</main>
    <footer>{{ template "footer" . }}</footer>
</body>
</html>
{{ end }}
*/

// templates/pages/home.html
/*
{{ template "base" . }}

{{ define "title" }}Home{{ end }}

{{ define "content" }}
<h1>Welcome, {{ .user.Name }}</h1>
<p>{{ .message }}</p>
{{ end }}
*/

// Load with patterns
r.LoadHTMLGlob("templates/**/*")

r.GET("/", func(c *gin.Context) {
    c.HTML(http.StatusOK, "pages/home.html", gin.H{
        "user":    user,
        "message": "Hello!",
    })
})

// Custom delimiters (avoid conflict with Vue/Angular)
r := gin.Default()
r.Delims("{[{", "}]}")
r.LoadHTMLGlob("templates/*")

// Template conditionals and loops
/*
{{ if .isAdmin }}
    <a href="/admin">Admin Panel</a>
{{ end }}

{{ range .items }}
    <li>{{ .Name }} - ${{ .Price }}</li>
{{ else }}
    <li>No items found</li>
{{ end }}

{{ with .user }}
    <p>Name: {{ .Name }}</p>
    <p>Email: {{ .Email }}</p>
{{ end }}
*/
```

<a id="q2"></a>
### Q2: How do you use custom template functions?
**Answer:**

```go
import "html/template"

func main() {
    r := gin.Default()
    
    // Register custom functions BEFORE LoadHTMLGlob
    r.SetFuncMap(template.FuncMap{
        "formatDate": formatDate,
        "upper":      strings.ToUpper,
        "lower":      strings.ToLower,
        "currency":   formatCurrency,
        "safe":       safeHTML,
        "truncate":   truncate,
    })
    
    r.LoadHTMLGlob("templates/*")
    r.Run()
}

// Custom functions
func formatDate(t time.Time) string {
    return t.Format("Jan 2, 2006")
}

func formatCurrency(amount float64) string {
    return fmt.Sprintf("$%.2f", amount)
}

func safeHTML(s string) template.HTML {
    return template.HTML(s)  // Marks as safe, won't be escaped
}

func truncate(s string, length int) string {
    if len(s) <= length {
        return s
    }
    return s[:length] + "..."
}

// Usage in template
/*
<p>Created: {{ formatDate .CreatedAt }}</p>
<p>Price: {{ currency .Price }}</p>
<p>Name: {{ upper .Name }}</p>
<p>Description: {{ truncate .Description 100 }}</p>
<div>{{ safe .HTMLContent }}</div>
*/

// Math and comparison functions
r.SetFuncMap(template.FuncMap{
    "add": func(a, b int) int { return a + b },
    "sub": func(a, b int) int { return a - b },
    "mul": func(a, b int) int { return a * b },
    "div": func(a, b int) int { return a / b },
    "mod": func(a, b int) int { return a % b },
    "seq": func(start, end int) []int {
        s := make([]int, end-start+1)
        for i := range s {
            s[i] = start + i
        }
        return s
    },
})

// Usage
/*
{{ range seq 1 10 }}
    <option value="{{ . }}">Page {{ . }}</option>
{{ end }}

<p>Total: {{ add .subtotal .tax }}</p>
*/
```

---

## Caching

<a id="q3"></a>
### Q3: How do you implement response caching?
**Answer:**

```go
import "github.com/gin-contrib/cache"
import "github.com/gin-contrib/cache/persistence"

func main() {
    r := gin.Default()
    
    // In-memory cache store
    store := persistence.NewInMemoryStore(time.Minute)
    
    // Cache specific routes
    r.GET("/expensive", cache.CachePage(store, time.Hour, expensiveHandler))
    
    r.Run()
}

func expensiveHandler(c *gin.Context) {
    // This response will be cached for 1 hour
    result := performExpensiveOperation()
    c.JSON(http.StatusOK, result)
}

// Redis cache store
import "github.com/gin-contrib/cache/persistence"

func main() {
    store := persistence.NewRedisCache("localhost:6379", "", time.Hour)
    
    r.GET("/data", cache.CachePage(store, time.Minute*5, dataHandler))
}

// Custom cache key
r.GET("/users/:id", cache.CachePageWithoutQuery(store, time.Minute, 
    func(c *gin.Context) {
        // Cache key will include the full URL path
    }))

// Manual caching
func CacheMiddleware(store persistence.CacheStore, duration time.Duration) gin.HandlerFunc {
    return func(c *gin.Context) {
        key := c.Request.URL.String()
        
        // Try cache
        if cached, err := store.Get(key, nil); err == nil {
            c.Data(http.StatusOK, "application/json", cached.([]byte))
            return
        }
        
        // Wrap response writer to capture response
        w := &responseCapture{ResponseWriter: c.Writer}
        c.Writer = w
        
        c.Next()
        
        // Store in cache
        if c.Writer.Status() == http.StatusOK {
            store.Set(key, w.body.Bytes(), duration)
        }
    }
}

type responseCapture struct {
    gin.ResponseWriter
    body bytes.Buffer
}

func (w *responseCapture) Write(b []byte) (int, error) {
    w.body.Write(b)
    return w.ResponseWriter.Write(b)
}
```

<a id="q4"></a>
### Q4: What caching strategies are available?
**Answer:**

```go
// Cache-Control headers
func CacheControlMiddleware(maxAge int) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Only cache GET requests
        if c.Request.Method == "GET" {
            c.Header("Cache-Control", fmt.Sprintf("public, max-age=%d", maxAge))
        }
        c.Next()
    }
}

// No cache for dynamic content
func NoCacheMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Cache-Control", "no-cache, no-store, must-revalidate")
        c.Header("Pragma", "no-cache")
        c.Header("Expires", "0")
        c.Next()
    }
}

// ETag caching
func ETagMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        
        if c.Writer.Status() == http.StatusOK {
            body := c.Writer.(*responseCapture).body.Bytes()
            etag := fmt.Sprintf(`"%x"`, md5.Sum(body))
            
            c.Header("ETag", etag)
            
            if match := c.GetHeader("If-None-Match"); match == etag {
                c.AbortWithStatus(http.StatusNotModified)
            }
        }
    }
}

// Conditional caching based on user
func UserAwareCaching(store persistence.CacheStore) gin.HandlerFunc {
    return func(c *gin.Context) {
        userID := c.GetString("userID")
        
        // Include user in cache key for personalized content
        key := fmt.Sprintf("%s:%s", userID, c.Request.URL.String())
        
        if cached, err := store.Get(key, nil); err == nil {
            c.Data(http.StatusOK, "application/json", cached.([]byte))
            return
        }
        
        c.Next()
        // ... cache response
    }
}

// Cache invalidation
type CacheManager struct {
    store persistence.CacheStore
}

func (cm *CacheManager) InvalidatePattern(pattern string) {
    // Implementation depends on store type
    // Redis supports SCAN with pattern matching
}

func (cm *CacheManager) InvalidateUser(userID string) {
    cm.InvalidatePattern(fmt.Sprintf("%s:*", userID))
}
```

---

## Compression

<a id="q5"></a>
### Q5: How do you enable gzip compression?
**Answer:**

```go
import "github.com/gin-contrib/gzip"

func main() {
    r := gin.Default()
    
    // Default compression
    r.Use(gzip.Gzip(gzip.DefaultCompression))
    
    // Best compression (slower)
    r.Use(gzip.Gzip(gzip.BestCompression))
    
    // Best speed (less compression)
    r.Use(gzip.Gzip(gzip.BestSpeed))
    
    // Custom compression level (1-9)
    r.Use(gzip.Gzip(gzip.BestCompression, gzip.WithExcludedExtensions([]string{".pdf", ".mp4"})))
    
    // Exclude paths
    r.Use(gzip.Gzip(gzip.DefaultCompression, gzip.WithExcludedPaths([]string{"/api/stream"})))
    
    // Exclude paths with regex
    r.Use(gzip.Gzip(gzip.DefaultCompression, gzip.WithExcludedPathsRegexs([]string{"^/api/.*"})))
    
    r.Run()
}

// Custom gzip middleware
func GzipMiddleware(level int) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Check if client accepts gzip
        if !strings.Contains(c.GetHeader("Accept-Encoding"), "gzip") {
            c.Next()
            return
        }
        
        // Create gzip writer
        gz, err := gzip.NewWriterLevel(c.Writer, level)
        if err != nil {
            c.Next()
            return
        }
        defer gz.Close()
        
        c.Header("Content-Encoding", "gzip")
        c.Header("Vary", "Accept-Encoding")
        
        c.Writer = &gzipWriter{Writer: gz, ResponseWriter: c.Writer}
        c.Next()
    }
}

type gzipWriter struct {
    io.Writer
    gin.ResponseWriter
}

func (w *gzipWriter) Write(data []byte) (int, error) {
    return w.Writer.Write(data)
}

// Only compress responses above minimum size
func SmartGzip(minSize int) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Capture response first
        buffer := &bytes.Buffer{}
        c.Writer = &bufferWriter{ResponseWriter: c.Writer, buffer: buffer}
        
        c.Next()
        
        if buffer.Len() >= minSize {
            // Compress
            c.Header("Content-Encoding", "gzip")
            gz := gzip.NewWriter(c.Writer)
            gz.Write(buffer.Bytes())
            gz.Close()
        } else {
            // Write uncompressed
            c.Writer.Write(buffer.Bytes())
        }
    }
}
```

---

## Security Headers

<a id="q6"></a>
### Q6: How do you add security headers?
**Answer:**

```go
import "github.com/gin-contrib/secure"

func main() {
    r := gin.Default()
    
    r.Use(secure.New(secure.Config{
        // Prevent clickjacking
        FrameDeny: true,
        
        // Content type sniffing protection
        ContentTypeNosniff: true,
        
        // XSS protection
        BrowserXssFilter: true,
        
        // HSTS (HTTP Strict Transport Security)
        STSSeconds:           315360000,  // 10 years
        STSIncludeSubdomains: true,
        
        // Content Security Policy
        ContentSecurityPolicy: "default-src 'self'; script-src 'self' 'unsafe-inline'",
        
        // Referrer policy
        ReferrerPolicy: "strict-origin-when-cross-origin",
        
        // Only allow HTTPS
        SSLRedirect: true,
        SSLHost:     "example.com",
        
        // Custom headers
        CustomFrameOptionsValue: "SAMEORIGIN",
    }))
    
    r.Run()
}

// Manual security headers
func SecurityHeadersMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        // Prevent clickjacking
        c.Header("X-Frame-Options", "DENY")
        
        // Prevent content type sniffing
        c.Header("X-Content-Type-Options", "nosniff")
        
        // XSS protection
        c.Header("X-XSS-Protection", "1; mode=block")
        
        // HSTS
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        
        // Content Security Policy
        c.Header("Content-Security-Policy", "default-src 'self'")
        
        // Referrer Policy
        c.Header("Referrer-Policy", "strict-origin-when-cross-origin")
        
        // Permissions Policy
        c.Header("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        
        c.Next()
    }
}

// Environment-specific headers
func SecurityHeaders(env string) gin.HandlerFunc {
    return func(c *gin.Context) {
        // Always set these
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-Frame-Options", "DENY")
        
        // Production only
        if env == "production" {
            c.Header("Strict-Transport-Security", "max-age=31536000")
            c.Header("Content-Security-Policy", "default-src 'self'")
        }
        
        c.Next()
    }
}
```

---

## File Handling

<a id="q7"></a>
### Q7: How do you handle file uploads?
**Answer:**

```go
// Single file upload
r.POST("/upload", func(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    // Save file
    dst := filepath.Join("./uploads", file.Filename)
    if err := c.SaveUploadedFile(file, dst); err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{
        "filename": file.Filename,
        "size":     file.Size,
    })
})

// Multiple file upload
r.POST("/upload-multiple", func(c *gin.Context) {
    form, err := c.MultipartForm()
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    files := form.File["files"]
    
    for _, file := range files {
        dst := filepath.Join("./uploads", file.Filename)
        c.SaveUploadedFile(file, dst)
    }
    
    c.JSON(http.StatusOK, gin.H{
        "uploaded": len(files),
    })
})

// File upload with validation
func uploadWithValidation(c *gin.Context) {
    file, err := c.FormFile("file")
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "No file uploaded"})
        return
    }
    
    // Check file size (10MB limit)
    if file.Size > 10*1024*1024 {
        c.JSON(http.StatusBadRequest, gin.H{"error": "File too large"})
        return
    }
    
    // Check file type
    allowedTypes := map[string]bool{
        "image/jpeg": true,
        "image/png":  true,
        "image/gif":  true,
    }
    
    f, _ := file.Open()
    defer f.Close()
    
    buffer := make([]byte, 512)
    f.Read(buffer)
    contentType := http.DetectContentType(buffer)
    
    if !allowedTypes[contentType] {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid file type"})
        return
    }
    
    // Generate unique filename
    ext := filepath.Ext(file.Filename)
    newFilename := fmt.Sprintf("%s%s", uuid.New().String(), ext)
    dst := filepath.Join("./uploads", newFilename)
    
    c.SaveUploadedFile(file, dst)
    c.JSON(http.StatusOK, gin.H{"filename": newFilename})
}

// Set max multipart memory
r.MaxMultipartMemory = 8 << 20 // 8 MiB

// Process file without saving
r.POST("/process", func(c *gin.Context) {
    file, _ := c.FormFile("file")
    f, _ := file.Open()
    defer f.Close()
    
    // Read content
    content, _ := io.ReadAll(f)
    
    // Process content...
    
    c.JSON(http.StatusOK, gin.H{"processed": true})
})
```

<a id="q8"></a>
### Q8: How do you serve static files and downloads?
**Answer:**

```go
// Serve static directory
r.Static("/assets", "./assets")
r.Static("/css", "./public/css")
r.Static("/js", "./public/js")

// Serve with custom filesystem
r.StaticFS("/static", http.Dir("static"))

// Serve single file
r.StaticFile("/favicon.ico", "./resources/favicon.ico")

// Serve file in handler
r.GET("/document/:id", func(c *gin.Context) {
    id := c.Param("id")
    filePath := fmt.Sprintf("./documents/%s.pdf", id)
    
    if _, err := os.Stat(filePath); os.IsNotExist(err) {
        c.JSON(http.StatusNotFound, gin.H{"error": "File not found"})
        return
    }
    
    c.File(filePath)
})

// Force download (Content-Disposition: attachment)
r.GET("/download/:filename", func(c *gin.Context) {
    filename := c.Param("filename")
    filePath := filepath.Join("./files", filename)
    
    // FileAttachment sets Content-Disposition header
    c.FileAttachment(filePath, filename)
})

// Serve from memory/reader
r.GET("/generate-pdf", func(c *gin.Context) {
    // Generate PDF
    pdfBuffer := generatePDF()
    
    c.DataFromReader(
        http.StatusOK,
        int64(pdfBuffer.Len()),
        "application/pdf",
        pdfBuffer,
        map[string]string{
            "Content-Disposition": `attachment; filename="report.pdf"`,
        },
    )
})

// Streaming large files
r.GET("/stream/:id", func(c *gin.Context) {
    file, _ := os.Open("./large-file.zip")
    defer file.Close()
    
    fileInfo, _ := file.Stat()
    
    c.DataFromReader(
        http.StatusOK,
        fileInfo.Size(),
        "application/octet-stream",
        file,
        map[string]string{
            "Content-Disposition": `attachment; filename="download.zip"`,
        },
    )
})

// Range requests for video/audio
r.GET("/video/:id", func(c *gin.Context) {
    filePath := "./videos/" + c.Param("id")
    http.ServeFile(c.Writer, c.Request, filePath)
})
```

---

## Sessions & Cookies

<a id="q9"></a>
### Q9: How do you manage sessions in Gin?
**Answer:**

```go
import "github.com/gin-contrib/sessions"
import "github.com/gin-contrib/sessions/cookie"

func main() {
    r := gin.Default()
    
    // Cookie-based sessions
    store := cookie.NewStore([]byte("secret-key"))
    r.Use(sessions.Sessions("mysession", store))
    
    r.GET("/set", func(c *gin.Context) {
        session := sessions.Default(c)
        session.Set("user", "alice")
        session.Set("authenticated", true)
        session.Save()
        
        c.JSON(http.StatusOK, gin.H{"status": "session set"})
    })
    
    r.GET("/get", func(c *gin.Context) {
        session := sessions.Default(c)
        user := session.Get("user")
        auth := session.Get("authenticated")
        
        c.JSON(http.StatusOK, gin.H{
            "user":          user,
            "authenticated": auth,
        })
    })
    
    r.GET("/clear", func(c *gin.Context) {
        session := sessions.Default(c)
        session.Clear()
        session.Save()
        
        c.JSON(http.StatusOK, gin.H{"status": "session cleared"})
    })
    
    r.GET("/delete", func(c *gin.Context) {
        session := sessions.Default(c)
        session.Delete("user")
        session.Save()
        
        c.JSON(http.StatusOK, gin.H{"status": "key deleted"})
    })
    
    r.Run()
}

// Session options
store := cookie.NewStore([]byte("secret"))
store.Options(sessions.Options{
    Path:     "/",
    Domain:   "example.com",
    MaxAge:   86400,           // 1 day
    Secure:   true,            // HTTPS only
    HttpOnly: true,            // No JavaScript access
    SameSite: http.SameSiteLaxMode,
})

// Flash messages
r.POST("/login", func(c *gin.Context) {
    session := sessions.Default(c)
    
    if !authenticate(c) {
        session.AddFlash("Invalid credentials")
        session.Save()
        c.Redirect(http.StatusFound, "/login")
        return
    }
    
    session.Set("user", "alice")
    session.Save()
    c.Redirect(http.StatusFound, "/dashboard")
})

r.GET("/login", func(c *gin.Context) {
    session := sessions.Default(c)
    flashes := session.Flashes()
    session.Save()  // Clear flashes after reading
    
    c.HTML(http.StatusOK, "login.html", gin.H{
        "flashes": flashes,
    })
})
```

<a id="q10"></a>
### Q10: How do you configure cookies securely?
**Answer:**

```go
// Setting cookies
r.GET("/set-cookie", func(c *gin.Context) {
    c.SetCookie(
        "token",           // name
        "abc123",          // value
        3600,              // maxAge (seconds)
        "/",               // path
        "example.com",     // domain
        true,              // secure (HTTPS only)
        true,              // httpOnly (no JS access)
    )
    
    c.String(http.StatusOK, "Cookie set")
})

// Reading cookies
r.GET("/get-cookie", func(c *gin.Context) {
    token, err := c.Cookie("token")
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "No cookie"})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{"token": token})
})

// Deleting cookies
r.GET("/delete-cookie", func(c *gin.Context) {
    // Set MaxAge to -1 to delete
    c.SetCookie("token", "", -1, "/", "example.com", true, true)
    c.String(http.StatusOK, "Cookie deleted")
})

// SameSite attribute
r.GET("/secure-cookie", func(c *gin.Context) {
    c.SetSameSite(http.SameSiteStrictMode)
    c.SetCookie("session", "value", 3600, "/", "", true, true)
})

// Signed cookies (prevent tampering)
import "github.com/gorilla/securecookie"

var hashKey = []byte("very-secret-key-for-signing")
var blockKey = []byte("16-byte-block-key!")  // AES-128
var s = securecookie.New(hashKey, blockKey)

func setSignedCookie(c *gin.Context, name string, value interface{}) error {
    encoded, err := s.Encode(name, value)
    if err != nil {
        return err
    }
    c.SetCookie(name, encoded, 3600, "/", "", true, true)
    return nil
}

func getSignedCookie(c *gin.Context, name string, dst interface{}) error {
    cookie, err := c.Cookie(name)
    if err != nil {
        return err
    }
    return s.Decode(name, cookie, dst)
}

// Usage
r.GET("/set", func(c *gin.Context) {
    userData := map[string]string{"user": "alice", "role": "admin"}
    setSignedCookie(c, "session", userData)
    c.String(http.StatusOK, "Set signed cookie")
})

r.GET("/get", func(c *gin.Context) {
    var userData map[string]string
    if err := getSignedCookie(c, "session", &userData); err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid cookie"})
        return
    }
    c.JSON(http.StatusOK, userData)
})
```

<a id="q11"></a>
### Q11: What session storage backends are available?
**Answer:**

```go
import (
    "github.com/gin-contrib/sessions"
    "github.com/gin-contrib/sessions/cookie"
    "github.com/gin-contrib/sessions/redis"
    "github.com/gin-contrib/sessions/memstore"
)

// Cookie store (client-side)
// Pros: No server storage needed
// Cons: Size limit (~4KB), data sent every request
func cookieStore() sessions.Store {
    return cookie.NewStore([]byte("secret"))
}

// In-memory store
// Pros: Fast
// Cons: Lost on restart, doesn't scale
func memoryStore() sessions.Store {
    return memstore.NewStore([]byte("secret"))
}

// Redis store (recommended for production)
// Pros: Persistent, scalable, shared across instances
// Cons: External dependency
func redisStore() (sessions.Store, error) {
    store, err := redis.NewStore(
        10,                    // max idle connections
        "tcp",                 // network
        "localhost:6379",      // address
        "",                    // password
        []byte("secret-key"),  // auth key
    )
    return store, err
}

// Redis with options
func redisStoreWithOptions() (sessions.Store, error) {
    store, err := redis.NewStoreWithDB(
        10,
        "tcp",
        "localhost:6379",
        "",
        "1",                   // Redis DB number
        []byte("secret-key"),
    )
    if err != nil {
        return nil, err
    }
    
    store.Options(sessions.Options{
        Path:     "/",
        MaxAge:   86400 * 7,   // 7 days
        Secure:   true,
        HttpOnly: true,
    })
    
    return store, nil
}

// Multiple stores (e.g., different session types)
func main() {
    r := gin.Default()
    
    // User session (Redis)
    userStore, _ := redis.NewStore(10, "tcp", "localhost:6379", "", []byte("user-secret"))
    r.Use(sessions.Sessions("user_session", userStore))
    
    // Admin session (separate Redis DB)
    adminStore, _ := redis.NewStoreWithDB(10, "tcp", "localhost:6379", "", "1", []byte("admin-secret"))
    
    admin := r.Group("/admin")
    admin.Use(sessions.Sessions("admin_session", adminStore))
    
    r.Run()
}

// Custom session store (implement sessions.Store interface)
type CustomStore struct {
    // Your storage implementation
}

func (s *CustomStore) Get(r *http.Request, name string) (*sessions.Session, error) {
    // Retrieve session
}

func (s *CustomStore) New(r *http.Request, name string) (*sessions.Session, error) {
    // Create new session
}

func (s *CustomStore) Save(r *http.Request, w http.ResponseWriter, session *sessions.Session) error {
    // Save session
}
```

---

[← Back to Go Index](README.md)

# Gin Authentication, Testing & Deployment

## Table of Contents

### Database Integration
- [Q1: How do you integrate databases with Gin?](#q1)
- [Q2: How do you manage database connections?](#q2)

### Authentication
- [Q3: How do you implement JWT authentication?](#q3)
- [Q4: How do you implement role-based access control?](#q4)
- [Q5: How do you implement OAuth2?](#q5)

### Testing
- [Q6: How do you write unit tests for Gin handlers?](#q6)
- [Q7: How do you test with mocked dependencies?](#q7)
- [Q8: How do you test middleware?](#q8)

### Deployment
- [Q9: How do you build and deploy Gin applications?](#q9)
- [Q10: What are performance optimization tips?](#q10)

---

## Database Integration

<a id="q1"></a>
### Q1: How do you integrate databases with Gin?
**Answer:**

```go
// Global database connection
var db *sql.DB

func main() {
    var err error
    db, err = sql.Open("postgres", "postgres://user:pass@localhost/db")
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()
    
    r := gin.Default()
    r.GET("/users", getUsers)
    r.Run()
}

func getUsers(c *gin.Context) {
    rows, err := db.QueryContext(c.Request.Context(), "SELECT id, name FROM users")
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    defer rows.Close()
    
    var users []User
    for rows.Next() {
        var u User
        rows.Scan(&u.ID, &u.Name)
        users = append(users, u)
    }
    
    c.JSON(http.StatusOK, users)
}

// Dependency injection (better approach)
type Handler struct {
    db *sql.DB
}

func NewHandler(db *sql.DB) *Handler {
    return &Handler{db: db}
}

func (h *Handler) GetUsers(c *gin.Context) {
    // Use h.db
}

func main() {
    db, _ := sql.Open("postgres", connStr)
    handler := NewHandler(db)
    
    r := gin.Default()
    r.GET("/users", handler.GetUsers)
    r.Run()
}

// With GORM
type Handler struct {
    db *gorm.DB
}

func (h *Handler) GetUsers(c *gin.Context) {
    var users []User
    if err := h.db.Find(&users).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, users)
}

func (h *Handler) CreateUser(c *gin.Context) {
    var user User
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    if err := h.db.Create(&user).Error; err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusCreated, user)
}

// Transaction in handler
func (h *Handler) TransferFunds(c *gin.Context) {
    var req TransferRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    err := h.db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Model(&Account{}).Where("id = ?", req.FromID).
            Update("balance", gorm.Expr("balance - ?", req.Amount)).Error; err != nil {
            return err
        }
        
        if err := tx.Model(&Account{}).Where("id = ?", req.ToID).
            Update("balance", gorm.Expr("balance + ?", req.Amount)).Error; err != nil {
            return err
        }
        
        return nil
    })
    
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{"status": "transferred"})
}
```

<a id="q2"></a>
### Q2: How do you manage database connections?
**Answer:**

```go
// Connection pool configuration
func setupDB() *sql.DB {
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        log.Fatal(err)
    }
    
    // Pool settings
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(25)
    db.SetConnMaxLifetime(5 * time.Minute)
    db.SetConnMaxIdleTime(5 * time.Minute)
    
    // Verify connection
    if err := db.Ping(); err != nil {
        log.Fatal(err)
    }
    
    return db
}

// Health check endpoint
func (h *Handler) HealthCheck(c *gin.Context) {
    ctx, cancel := context.WithTimeout(c.Request.Context(), 2*time.Second)
    defer cancel()
    
    if err := h.db.PingContext(ctx); err != nil {
        c.JSON(http.StatusServiceUnavailable, gin.H{
            "status":   "unhealthy",
            "database": "down",
        })
        return
    }
    
    c.JSON(http.StatusOK, gin.H{
        "status":   "healthy",
        "database": "up",
    })
}

// Middleware to add DB to context
func DBMiddleware(db *sql.DB) gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Set("db", db)
        c.Next()
    }
}

// Get DB from context
func GetDB(c *gin.Context) *sql.DB {
    return c.MustGet("db").(*sql.DB)
}

// Repository pattern
type UserRepository interface {
    FindByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, user *User) error
    Update(ctx context.Context, user *User) error
    Delete(ctx context.Context, id string) error
}

type userRepository struct {
    db *sql.DB
}

func (r *userRepository) FindByID(ctx context.Context, id string) (*User, error) {
    var user User
    err := r.db.QueryRowContext(ctx, 
        "SELECT id, name, email FROM users WHERE id = $1", id).
        Scan(&user.ID, &user.Name, &user.Email)
    
    if err == sql.ErrNoRows {
        return nil, ErrNotFound
    }
    return &user, err
}

// Service layer
type UserService struct {
    repo UserRepository
}

func (s *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    return s.repo.FindByID(ctx, id)
}

// Handler uses service
type UserHandler struct {
    service *UserService
}

func (h *UserHandler) GetUser(c *gin.Context) {
    user, err := h.service.GetUser(c.Request.Context(), c.Param("id"))
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            c.JSON(http.StatusNotFound, gin.H{"error": "User not found"})
            return
        }
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    c.JSON(http.StatusOK, user)
}
```

---

## Authentication

<a id="q3"></a>
### Q3: How do you implement JWT authentication?
**Answer:**

```go
import "github.com/golang-jwt/jwt/v5"

var jwtSecret = []byte(os.Getenv("JWT_SECRET"))

// Claims structure
type Claims struct {
    UserID string `json:"user_id"`
    Email  string `json:"email"`
    Role   string `json:"role"`
    jwt.RegisteredClaims
}

// Generate token
func GenerateToken(user *User) (string, error) {
    claims := Claims{
        UserID: user.ID,
        Email:  user.Email,
        Role:   user.Role,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            Issuer:    "my-app",
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

// Validate token
func ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
        }
        return jwtSecret, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, errors.New("invalid token")
}

// Auth middleware
func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        authHeader := c.GetHeader("Authorization")
        if authHeader == "" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Authorization header required",
            })
            return
        }
        
        // Extract token from "Bearer <token>"
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Invalid authorization header format",
            })
            return
        }
        
        claims, err := ValidateToken(parts[1])
        if err != nil {
            c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{
                "error": "Invalid or expired token",
            })
            return
        }
        
        // Set user info in context
        c.Set("userID", claims.UserID)
        c.Set("userEmail", claims.Email)
        c.Set("userRole", claims.Role)
        
        c.Next()
    }
}

// Login handler
func (h *AuthHandler) Login(c *gin.Context) {
    var req LoginRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
        return
    }
    
    user, err := h.userService.Authenticate(req.Email, req.Password)
    if err != nil {
        c.JSON(http.StatusUnauthorized, gin.H{"error": "Invalid credentials"})
        return
    }
    
    token, err := GenerateToken(user)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to generate token"})
        return
    }
    
    c.JSON(http.StatusOK, gin.H{
        "token": token,
        "user":  user,
    })
}

// Protected route
r.GET("/profile", AuthMiddleware(), func(c *gin.Context) {
    userID := c.GetString("userID")
    c.JSON(http.StatusOK, gin.H{"user_id": userID})
})

// Refresh token
func (h *AuthHandler) RefreshToken(c *gin.Context) {
    claims := c.MustGet("claims").(*Claims)
    
    // Check if token is about to expire (e.g., within 1 hour)
    if time.Until(claims.ExpiresAt.Time) > time.Hour {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Token not eligible for refresh"})
        return
    }
    
    // Generate new token
    user, _ := h.userService.GetByID(claims.UserID)
    newToken, _ := GenerateToken(user)
    
    c.JSON(http.StatusOK, gin.H{"token": newToken})
}
```

<a id="q4"></a>
### Q4: How do you implement role-based access control?
**Answer:**

```go
// Role constants
const (
    RoleAdmin  = "admin"
    RoleUser   = "user"
    RoleGuest  = "guest"
)

// Permission-based middleware
func RequireRole(roles ...string) gin.HandlerFunc {
    return func(c *gin.Context) {
        userRole := c.GetString("userRole")
        
        for _, role := range roles {
            if userRole == role {
                c.Next()
                return
            }
        }
        
        c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
            "error": "Insufficient permissions",
        })
    }
}

// Usage
r.GET("/admin/users", AuthMiddleware(), RequireRole(RoleAdmin), adminListUsers)
r.GET("/profile", AuthMiddleware(), RequireRole(RoleAdmin, RoleUser), getProfile)

// Permission-based (more granular)
type Permission string

const (
    PermissionReadUsers   Permission = "users:read"
    PermissionWriteUsers  Permission = "users:write"
    PermissionDeleteUsers Permission = "users:delete"
    PermissionReadOrders  Permission = "orders:read"
)

var rolePermissions = map[string][]Permission{
    RoleAdmin: {PermissionReadUsers, PermissionWriteUsers, PermissionDeleteUsers, PermissionReadOrders},
    RoleUser:  {PermissionReadUsers, PermissionReadOrders},
    RoleGuest: {},
}

func RequirePermission(required ...Permission) gin.HandlerFunc {
    return func(c *gin.Context) {
        userRole := c.GetString("userRole")
        userPermissions := rolePermissions[userRole]
        
        for _, req := range required {
            hasPermission := false
            for _, perm := range userPermissions {
                if perm == req {
                    hasPermission = true
                    break
                }
            }
            if !hasPermission {
                c.AbortWithStatusJSON(http.StatusForbidden, gin.H{
                    "error": fmt.Sprintf("Missing permission: %s", req),
                })
                return
            }
        }
        
        c.Next()
    }
}

// Usage
r.DELETE("/users/:id", AuthMiddleware(), RequirePermission(PermissionDeleteUsers), deleteUser)

// Resource-based authorization
func CanAccessResource(c *gin.Context, resourceOwnerID string) bool {
    userID := c.GetString("userID")
    userRole := c.GetString("userRole")
    
    // Admin can access anything
    if userRole == RoleAdmin {
        return true
    }
    
    // Users can only access their own resources
    return userID == resourceOwnerID
}

func (h *Handler) GetOrder(c *gin.Context) {
    order, err := h.orderService.GetByID(c.Param("id"))
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{"error": "Order not found"})
        return
    }
    
    if !CanAccessResource(c, order.UserID) {
        c.JSON(http.StatusForbidden, gin.H{"error": "Access denied"})
        return
    }
    
    c.JSON(http.StatusOK, order)
}
```

<a id="q5"></a>
### Q5: How do you implement OAuth2?
**Answer:**

```go
import "golang.org/x/oauth2"
import "golang.org/x/oauth2/google"

var googleOauthConfig = &oauth2.Config{
    ClientID:     os.Getenv("GOOGLE_CLIENT_ID"),
    ClientSecret: os.Getenv("GOOGLE_CLIENT_SECRET"),
    RedirectURL:  "http://localhost:8080/auth/google/callback",
    Scopes: []string{
        "https://www.googleapis.com/auth/userinfo.email",
        "https://www.googleapis.com/auth/userinfo.profile",
    },
    Endpoint: google.Endpoint,
}

// Initiate OAuth flow
func (h *AuthHandler) GoogleLogin(c *gin.Context) {
    // Generate state token for CSRF protection
    state := generateRandomState()
    
    // Store state in session
    session := sessions.Default(c)
    session.Set("oauth_state", state)
    session.Save()
    
    url := googleOauthConfig.AuthCodeURL(state)
    c.Redirect(http.StatusTemporaryRedirect, url)
}

// OAuth callback
func (h *AuthHandler) GoogleCallback(c *gin.Context) {
    // Verify state
    session := sessions.Default(c)
    savedState := session.Get("oauth_state")
    if c.Query("state") != savedState {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Invalid state"})
        return
    }
    
    // Exchange code for token
    code := c.Query("code")
    token, err := googleOauthConfig.Exchange(context.Background(), code)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{"error": "Failed to exchange token"})
        return
    }
    
    // Get user info from Google
    client := googleOauthConfig.Client(context.Background(), token)
    resp, err := client.Get("https://www.googleapis.com/oauth2/v2/userinfo")
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": "Failed to get user info"})
        return
    }
    defer resp.Body.Close()
    
    var googleUser GoogleUser
    json.NewDecoder(resp.Body).Decode(&googleUser)
    
    // Find or create user
    user, err := h.userService.FindOrCreateFromOAuth("google", googleUser.ID, googleUser.Email, googleUser.Name)
    if err != nil {
        c.JSON(http.StatusInternalServerError, gin.H{"error": err.Error()})
        return
    }
    
    // Generate JWT
    jwtToken, _ := GenerateToken(user)
    
    // Redirect to frontend with token
    c.Redirect(http.StatusTemporaryRedirect, 
        fmt.Sprintf("http://localhost:3000/oauth-success?token=%s", jwtToken))
}

type GoogleUser struct {
    ID      string `json:"id"`
    Email   string `json:"email"`
    Name    string `json:"name"`
    Picture string `json:"picture"`
}

// Routes
auth := r.Group("/auth")
{
    auth.GET("/google", h.GoogleLogin)
    auth.GET("/google/callback", h.GoogleCallback)
}
```

---

## Testing

<a id="q6"></a>
### Q6: How do you write unit tests for Gin handlers?
**Answer:**

```go
import (
    "net/http"
    "net/http/httptest"
    "testing"
    "github.com/gin-gonic/gin"
    "github.com/stretchr/testify/assert"
)

func TestGetUser(t *testing.T) {
    // Set Gin to test mode
    gin.SetMode(gin.TestMode)
    
    // Create router
    r := gin.Default()
    r.GET("/users/:id", GetUser)
    
    // Create test request
    req, _ := http.NewRequest("GET", "/users/1", nil)
    
    // Create response recorder
    w := httptest.NewRecorder()
    
    // Perform request
    r.ServeHTTP(w, req)
    
    // Assertions
    assert.Equal(t, http.StatusOK, w.Code)
    
    var response User
    json.Unmarshal(w.Body.Bytes(), &response)
    assert.Equal(t, "1", response.ID)
}

// Test with JSON body
func TestCreateUser(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    r := gin.Default()
    r.POST("/users", CreateUser)
    
    user := CreateUserRequest{
        Name:  "Alice",
        Email: "alice@example.com",
    }
    jsonBody, _ := json.Marshal(user)
    
    req, _ := http.NewRequest("POST", "/users", bytes.NewBuffer(jsonBody))
    req.Header.Set("Content-Type", "application/json")
    
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusCreated, w.Code)
}

// Test with query parameters
func TestSearchUsers(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    r := gin.Default()
    r.GET("/users", SearchUsers)
    
    req, _ := http.NewRequest("GET", "/users?q=alice&page=1", nil)
    w := httptest.NewRecorder()
    
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusOK, w.Code)
}

// Test with headers
func TestProtectedRoute(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    r := gin.Default()
    r.GET("/protected", AuthMiddleware(), ProtectedHandler)
    
    token, _ := GenerateToken(&User{ID: "1", Email: "test@test.com"})
    
    req, _ := http.NewRequest("GET", "/protected", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusOK, w.Code)
}

// Table-driven tests
func TestUserAPI(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    tests := []struct {
        name           string
        method         string
        path           string
        body           interface{}
        expectedStatus int
    }{
        {"Get existing user", "GET", "/users/1", nil, http.StatusOK},
        {"Get non-existing user", "GET", "/users/999", nil, http.StatusNotFound},
        {"Create valid user", "POST", "/users", CreateUserRequest{Name: "Test", Email: "test@test.com"}, http.StatusCreated},
        {"Create invalid user", "POST", "/users", CreateUserRequest{Name: ""}, http.StatusBadRequest},
    }
    
    r := setupRouter()
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            var body io.Reader
            if tt.body != nil {
                jsonBody, _ := json.Marshal(tt.body)
                body = bytes.NewBuffer(jsonBody)
            }
            
            req, _ := http.NewRequest(tt.method, tt.path, body)
            if tt.body != nil {
                req.Header.Set("Content-Type", "application/json")
            }
            
            w := httptest.NewRecorder()
            r.ServeHTTP(w, req)
            
            assert.Equal(t, tt.expectedStatus, w.Code)
        })
    }
}
```

<a id="q7"></a>
### Q7: How do you test with mocked dependencies?
**Answer:**

```go
// Interface for dependency
type UserService interface {
    GetByID(ctx context.Context, id string) (*User, error)
    Create(ctx context.Context, user *User) error
}

// Handler with dependency
type UserHandler struct {
    service UserService
}

// Mock service for testing
type MockUserService struct {
    mock.Mock
}

func (m *MockUserService) GetByID(ctx context.Context, id string) (*User, error) {
    args := m.Called(ctx, id)
    if args.Get(0) == nil {
        return nil, args.Error(1)
    }
    return args.Get(0).(*User), args.Error(1)
}

func (m *MockUserService) Create(ctx context.Context, user *User) error {
    args := m.Called(ctx, user)
    return args.Error(0)
}

// Test with mock
func TestGetUser_Success(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    // Setup mock
    mockService := new(MockUserService)
    expectedUser := &User{ID: "1", Name: "Alice"}
    mockService.On("GetByID", mock.Anything, "1").Return(expectedUser, nil)
    
    // Create handler with mock
    handler := &UserHandler{service: mockService}
    
    // Setup router
    r := gin.Default()
    r.GET("/users/:id", handler.GetUser)
    
    // Make request
    req, _ := http.NewRequest("GET", "/users/1", nil)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    // Assertions
    assert.Equal(t, http.StatusOK, w.Code)
    
    var response User
    json.Unmarshal(w.Body.Bytes(), &response)
    assert.Equal(t, "Alice", response.Name)
    
    // Verify mock was called
    mockService.AssertExpectations(t)
}

func TestGetUser_NotFound(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    mockService := new(MockUserService)
    mockService.On("GetByID", mock.Anything, "999").Return(nil, ErrNotFound)
    
    handler := &UserHandler{service: mockService}
    
    r := gin.Default()
    r.GET("/users/:id", handler.GetUser)
    
    req, _ := http.NewRequest("GET", "/users/999", nil)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusNotFound, w.Code)
    mockService.AssertExpectations(t)
}

// Test with test database
func TestIntegration_CreateUser(t *testing.T) {
    // Setup test database
    db := setupTestDB(t)
    defer teardownTestDB(db)
    
    handler := &UserHandler{service: NewUserService(db)}
    
    r := gin.Default()
    r.POST("/users", handler.Create)
    
    body := `{"name": "Alice", "email": "alice@test.com"}`
    req, _ := http.NewRequest("POST", "/users", strings.NewReader(body))
    req.Header.Set("Content-Type", "application/json")
    
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusCreated, w.Code)
    
    // Verify in database
    var count int
    db.QueryRow("SELECT COUNT(*) FROM users WHERE email = $1", "alice@test.com").Scan(&count)
    assert.Equal(t, 1, count)
}
```

<a id="q8"></a>
### Q8: How do you test middleware?
**Answer:**

```go
// Test authentication middleware
func TestAuthMiddleware_ValidToken(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    r := gin.Default()
    r.Use(AuthMiddleware())
    r.GET("/protected", func(c *gin.Context) {
        userID := c.GetString("userID")
        c.JSON(http.StatusOK, gin.H{"user_id": userID})
    })
    
    token, _ := GenerateToken(&User{ID: "123"})
    
    req, _ := http.NewRequest("GET", "/protected", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusOK, w.Code)
    
    var response map[string]string
    json.Unmarshal(w.Body.Bytes(), &response)
    assert.Equal(t, "123", response["user_id"])
}

func TestAuthMiddleware_MissingToken(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    r := gin.Default()
    r.Use(AuthMiddleware())
    r.GET("/protected", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })
    
    req, _ := http.NewRequest("GET", "/protected", nil)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusUnauthorized, w.Code)
}

func TestAuthMiddleware_InvalidToken(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    r := gin.Default()
    r.Use(AuthMiddleware())
    r.GET("/protected", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{})
    })
    
    req, _ := http.NewRequest("GET", "/protected", nil)
    req.Header.Set("Authorization", "Bearer invalid-token")
    
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Equal(t, http.StatusUnauthorized, w.Code)
}

// Test rate limiting middleware
func TestRateLimitMiddleware(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    r := gin.Default()
    r.Use(RateLimitMiddleware(2))  // 2 requests per second
    r.GET("/", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{"status": "ok"})
    })
    
    // First two requests should succeed
    for i := 0; i < 2; i++ {
        req, _ := http.NewRequest("GET", "/", nil)
        w := httptest.NewRecorder()
        r.ServeHTTP(w, req)
        assert.Equal(t, http.StatusOK, w.Code)
    }
    
    // Third request should be rate limited
    req, _ := http.NewRequest("GET", "/", nil)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    assert.Equal(t, http.StatusTooManyRequests, w.Code)
}

// Test logging middleware
func TestLoggingMiddleware(t *testing.T) {
    gin.SetMode(gin.TestMode)
    
    var logBuffer bytes.Buffer
    logger := log.New(&logBuffer, "", 0)
    
    r := gin.Default()
    r.Use(LoggingMiddleware(logger))
    r.GET("/test", func(c *gin.Context) {
        c.JSON(http.StatusOK, gin.H{})
    })
    
    req, _ := http.NewRequest("GET", "/test", nil)
    w := httptest.NewRecorder()
    r.ServeHTTP(w, req)
    
    assert.Contains(t, logBuffer.String(), "GET")
    assert.Contains(t, logBuffer.String(), "/test")
    assert.Contains(t, logBuffer.String(), "200")
}
```

---

## Deployment

<a id="q9"></a>
### Q9: How do you build and deploy Gin applications?
**Answer:**

```dockerfile
# Dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./
RUN go mod download

# Copy source
COPY . .

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Runtime stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

# Copy binary from builder
COPY --from=builder /app/main .
COPY --from=builder /app/templates ./templates
COPY --from=builder /app/static ./static

EXPOSE 8080

CMD ["./main"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8080:8080"
    environment:
      - GIN_MODE=release
      - DATABASE_URL=postgres://user:pass@db:5432/app
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine

volumes:
  pgdata:
```

```go
// Configuration from environment
type Config struct {
    Port        string `env:"PORT" envDefault:"8080"`
    DatabaseURL string `env:"DATABASE_URL,required"`
    RedisURL    string `env:"REDIS_URL,required"`
    JWTSecret   string `env:"JWT_SECRET,required"`
    Environment string `env:"ENVIRONMENT" envDefault:"development"`
}

func LoadConfig() (*Config, error) {
    var cfg Config
    if err := env.Parse(&cfg); err != nil {
        return nil, err
    }
    return &cfg, nil
}

func main() {
    cfg, err := LoadConfig()
    if err != nil {
        log.Fatal(err)
    }
    
    // Set Gin mode based on environment
    if cfg.Environment == "production" {
        gin.SetMode(gin.ReleaseMode)
    }
    
    r := gin.New()
    r.Use(gin.Recovery())
    
    // Production logging
    if cfg.Environment == "production" {
        r.Use(gin.LoggerWithConfig(gin.LoggerConfig{
            Formatter: JSONLogFormatter,
            Output:    os.Stdout,
        }))
    } else {
        r.Use(gin.Logger())
    }
    
    // Setup routes
    setupRoutes(r, cfg)
    
    // Start server
    r.Run(":" + cfg.Port)
}
```

<a id="q10"></a>
### Q10: What are performance optimization tips?
**Answer:**

```go
// 1. Use gin.ReleaseMode in production
gin.SetMode(gin.ReleaseMode)

// 2. Disable unnecessary features
r := gin.New()  // Instead of gin.Default()
r.Use(gin.Recovery())  // Add only needed middleware

// 3. Reuse JSON encoder
var jsonPool = sync.Pool{
    New: func() interface{} {
        return json.NewEncoder(nil)
    },
}

// 4. Stream large responses
func streamLargeData(c *gin.Context) {
    c.Stream(func(w io.Writer) bool {
        // Write chunks
        w.Write(chunk)
        return hasMoreData
    })
}

// 5. Enable gzip compression
r.Use(gzip.Gzip(gzip.DefaultCompression))

// 6. Use connection pooling
db.SetMaxOpenConns(25)
db.SetMaxIdleConns(25)
db.SetConnMaxLifetime(5 * time.Minute)

// 7. Cache responses
r.GET("/data", cache.CachePage(store, time.Minute, handler))

// 8. Optimize JSON responses
type User struct {
    ID    string `json:"id"`
    Name  string `json:"name"`
    // Use json:"-" to exclude fields from serialization
    Password string `json:"-"`
}

// 9. Use appropriate timeouts
srv := &http.Server{
    Addr:         ":8080",
    Handler:      r,
    ReadTimeout:  10 * time.Second,
    WriteTimeout: 10 * time.Second,
    IdleTimeout:  120 * time.Second,
}

// 10. Profile your application
import _ "net/http/pprof"
go func() {
    http.ListenAndServe("localhost:6060", nil)
}()

// 11. Use structured logging
import "github.com/rs/zerolog"
log := zerolog.New(os.Stdout).With().Timestamp().Logger()

// 12. Batch database operations
func createUsers(c *gin.Context) {
    var users []User
    c.ShouldBindJSON(&users)
    
    // Batch insert
    tx := db.Begin()
    for _, u := range users {
        tx.Create(&u)
    }
    tx.Commit()
}

// 13. Use context cancellation
func handler(c *gin.Context) {
    ctx := c.Request.Context()
    
    result, err := slowOperation(ctx)
    if err != nil {
        if ctx.Err() == context.Canceled {
            return  // Client disconnected
        }
        c.JSON(500, gin.H{"error": err.Error()})
        return
    }
    
    c.JSON(200, result)
}

// 14. Minimize allocations
// Bad
func handler(c *gin.Context) {
    data := make(map[string]interface{})  // Allocates every request
    data["status"] = "ok"
    c.JSON(200, data)
}

// Good
var okResponse = gin.H{"status": "ok"}  // Reuse

func handler(c *gin.Context) {
    c.JSON(200, okResponse)
}
```

---

[← Back to Go Index](README.md)

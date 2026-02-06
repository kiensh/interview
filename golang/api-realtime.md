# API & Real-time Communication

## Table of Contents

### REST API Design
- [Q1: How do you design RESTful APIs in Go?](#q1)
- [Q2: How do you handle API versioning?](#q2)
- [Q3: How do you implement rate limiting?](#q3)

### gRPC
- [Q4: What is gRPC and when should you use it?](#q4)
- [Q5: How do you implement gRPC services in Go?](#q5)
- [Q6: What are gRPC streaming modes?](#q6)
- [Q7: What are gRPC interceptors?](#q7)

### Real-time Communication
- [Q8: How do you implement WebSockets in Go?](#q8)
- [Q9: How do you implement Server-Sent Events (SSE)?](#q9)
- [Q10: How do you scale WebSocket connections?](#q10)

### API Best Practices
- [Q11: How do you handle API authentication?](#q11)
- [Q12: How do you document APIs?](#q12)
- [Q13: How do you handle API errors consistently?](#q13)
- [Q14: What are best practices for API pagination?](#q14)

---

## REST API Design

<a id="q1"></a>
### Q1: How do you design RESTful APIs in Go?
**Answer:**

```go
// RESTful resource structure
// GET    /users       - List users
// POST   /users       - Create user
// GET    /users/:id   - Get user
// PUT    /users/:id   - Update user
// DELETE /users/:id   - Delete user

// Using net/http (Go 1.22+ with enhanced routing)
mux := http.NewServeMux()

// Pattern matching (Go 1.22+)
mux.HandleFunc("GET /users", listUsers)
mux.HandleFunc("POST /users", createUser)
mux.HandleFunc("GET /users/{id}", getUser)
mux.HandleFunc("PUT /users/{id}", updateUser)
mux.HandleFunc("DELETE /users/{id}", deleteUser)

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")  // Go 1.22+
    // ...
}

// Handler with standard patterns
type UserHandler struct {
    service UserService
}

func (h *UserHandler) List(w http.ResponseWriter, r *http.Request) {
    // Parse query params
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    
    users, total, err := h.service.List(r.Context(), page, limit)
    if err != nil {
        respondError(w, http.StatusInternalServerError, err.Error())
        return
    }
    
    respondJSON(w, http.StatusOK, PaginatedResponse{
        Data:  users,
        Total: total,
        Page:  page,
        Limit: limit,
    })
}

func (h *UserHandler) Get(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    
    user, err := h.service.GetByID(r.Context(), id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            respondError(w, http.StatusNotFound, "user not found")
            return
        }
        respondError(w, http.StatusInternalServerError, err.Error())
        return
    }
    
    respondJSON(w, http.StatusOK, user)
}

func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        respondError(w, http.StatusBadRequest, "invalid request body")
        return
    }
    
    if err := validate(req); err != nil {
        respondError(w, http.StatusBadRequest, err.Error())
        return
    }
    
    user, err := h.service.Create(r.Context(), req)
    if err != nil {
        respondError(w, http.StatusInternalServerError, err.Error())
        return
    }
    
    respondJSON(w, http.StatusCreated, user)
}

// Helper functions
func respondJSON(w http.ResponseWriter, status int, data interface{}) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(status)
    json.NewEncoder(w).Encode(data)
}

func respondError(w http.ResponseWriter, status int, message string) {
    respondJSON(w, status, map[string]string{"error": message})
}
```

<a id="q2"></a>
### Q2: How do you handle API versioning?
**Answer:**

```go
// Method 1: URL Path versioning (most common)
// /api/v1/users
// /api/v2/users

mux := http.NewServeMux()

// V1 handlers
mux.HandleFunc("GET /api/v1/users", v1ListUsers)
mux.HandleFunc("GET /api/v1/users/{id}", v1GetUser)

// V2 handlers
mux.HandleFunc("GET /api/v2/users", v2ListUsers)
mux.HandleFunc("GET /api/v2/users/{id}", v2GetUser)

// Method 2: Header versioning
// Accept: application/vnd.myapi.v1+json

func versionedHandler(w http.ResponseWriter, r *http.Request) {
    accept := r.Header.Get("Accept")
    
    switch {
    case strings.Contains(accept, "vnd.myapi.v2"):
        v2Handler(w, r)
    case strings.Contains(accept, "vnd.myapi.v1"):
        v1Handler(w, r)
    default:
        // Default to latest stable version
        v1Handler(w, r)
    }
}

// Method 3: Query parameter versioning
// /api/users?version=1

func versionedByQuery(w http.ResponseWriter, r *http.Request) {
    version := r.URL.Query().Get("version")
    
    switch version {
    case "2":
        v2Handler(w, r)
    default:
        v1Handler(w, r)
    }
}

// Version router middleware
func VersionRouter(versions map[string]http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Extract version from path
        parts := strings.Split(r.URL.Path, "/")
        if len(parts) > 2 && strings.HasPrefix(parts[2], "v") {
            version := parts[2]
            if handler, ok := versions[version]; ok {
                // Strip version from path for downstream handlers
                r.URL.Path = "/" + strings.Join(append(parts[:2], parts[3:]...), "/")
                handler.ServeHTTP(w, r)
                return
            }
        }
        http.NotFound(w, r)
    })
}

// Backward compatibility pattern
type UserV1 struct {
    ID   string `json:"id"`
    Name string `json:"name"`
}

type UserV2 struct {
    ID        string `json:"id"`
    FirstName string `json:"first_name"`
    LastName  string `json:"last_name"`
    Name      string `json:"name"` // Computed for backward compat
}

func (u *UserV2) ToV1() UserV1 {
    return UserV1{
        ID:   u.ID,
        Name: u.FirstName + " " + u.LastName,
    }
}
```

<a id="q3"></a>
### Q3: How do you implement rate limiting?
**Answer:**

```go
import "golang.org/x/time/rate"

// Token bucket rate limiter
type RateLimiter struct {
    visitors map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(r rate.Limit, burst int) *RateLimiter {
    return &RateLimiter{
        visitors: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    burst,
    }
}

func (rl *RateLimiter) GetLimiter(ip string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    limiter, exists := rl.visitors[ip]
    if !exists {
        limiter = rate.NewLimiter(rl.rate, rl.burst)
        rl.visitors[ip] = limiter
    }
    
    return limiter
}

// Middleware
func RateLimitMiddleware(rl *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := getIP(r)
            limiter := rl.GetLimiter(ip)
            
            if !limiter.Allow() {
                w.Header().Set("Retry-After", "1")
                http.Error(w, "Too Many Requests", http.StatusTooManyRequests)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

func getIP(r *http.Request) string {
    // Check X-Forwarded-For header
    if xff := r.Header.Get("X-Forwarded-For"); xff != "" {
        return strings.Split(xff, ",")[0]
    }
    // Check X-Real-IP header
    if xri := r.Header.Get("X-Real-IP"); xri != "" {
        return xri
    }
    // Fall back to RemoteAddr
    ip, _, _ := net.SplitHostPort(r.RemoteAddr)
    return ip
}

// Usage
func main() {
    // 10 requests per second, burst of 20
    limiter := NewRateLimiter(10, 20)
    
    handler := RateLimitMiddleware(limiter)(myHandler)
    http.ListenAndServe(":8080", handler)
}

// Sliding window rate limiter with Redis
type RedisRateLimiter struct {
    client    *redis.Client
    limit     int
    window    time.Duration
}

func (rl *RedisRateLimiter) Allow(ctx context.Context, key string) (bool, error) {
    now := time.Now().UnixNano()
    windowStart := now - rl.window.Nanoseconds()
    
    pipe := rl.client.Pipeline()
    
    // Remove old entries
    pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(windowStart, 10))
    
    // Count current entries
    countCmd := pipe.ZCard(ctx, key)
    
    // Add new entry
    pipe.ZAdd(ctx, key, &redis.Z{Score: float64(now), Member: now})
    
    // Set expiry
    pipe.Expire(ctx, key, rl.window)
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }
    
    return countCmd.Val() < int64(rl.limit), nil
}
```

---

## gRPC

<a id="q4"></a>
### Q4: What is gRPC and when should you use it?
**Answer:**

gRPC is a high-performance RPC framework using Protocol Buffers:

| Feature | REST/JSON | gRPC |
|---------|-----------|------|
| Protocol | HTTP/1.1 | HTTP/2 |
| Payload | JSON (text) | Protobuf (binary) |
| Performance | Good | Excellent |
| Streaming | Limited | Native support |
| Code Generation | Optional | Required |
| Browser Support | Native | Requires proxy |
| Human Readable | Yes | No |

**Use gRPC when:**
- Internal microservices communication
- High-performance requirements
- Bidirectional streaming needed
- Strong typing across services
- Multiple language support needed

**Use REST when:**
- Public APIs
- Browser clients (without gRPC-Web)
- Simple CRUD operations
- Human-readable debugging important

```protobuf
// user.proto
syntax = "proto3";
package user;
option go_package = "./pb";

service UserService {
    rpc GetUser(GetUserRequest) returns (User);
    rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
    rpc CreateUser(CreateUserRequest) returns (User);
}

message User {
    string id = 1;
    string name = 2;
    string email = 3;
    int64 created_at = 4;
}

message GetUserRequest {
    string id = 1;
}

message ListUsersRequest {
    int32 page = 1;
    int32 limit = 2;
}

message ListUsersResponse {
    repeated User users = 1;
    int32 total = 2;
}

message CreateUserRequest {
    string name = 1;
    string email = 2;
}
```

```bash
# Generate Go code
protoc --go_out=. --go-grpc_out=. user.proto
```

<a id="q5"></a>
### Q5: How do you implement gRPC services in Go?
**Answer:**

```go
// Server implementation
type userServer struct {
    pb.UnimplementedUserServiceServer
    store UserStore
}

func (s *userServer) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    user, err := s.store.GetByID(ctx, req.Id)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            return nil, status.Error(codes.NotFound, "user not found")
        }
        return nil, status.Error(codes.Internal, err.Error())
    }
    
    return &pb.User{
        Id:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt.Unix(),
    }, nil
}

func (s *userServer) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
    // Validation
    if req.Name == "" {
        return nil, status.Error(codes.InvalidArgument, "name is required")
    }
    
    user, err := s.store.Create(ctx, User{
        Name:  req.Name,
        Email: req.Email,
    })
    if err != nil {
        return nil, status.Error(codes.Internal, err.Error())
    }
    
    return &pb.User{
        Id:        user.ID,
        Name:      user.Name,
        Email:     user.Email,
        CreatedAt: user.CreatedAt.Unix(),
    }, nil
}

// Start gRPC server
func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }
    
    grpcServer := grpc.NewServer()
    pb.RegisterUserServiceServer(grpcServer, &userServer{store: NewUserStore()})
    
    // Enable reflection for debugging tools
    reflection.Register(grpcServer)
    
    log.Println("gRPC server listening on :50051")
    if err := grpcServer.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}

// Client usage
func main() {
    conn, err := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf("failed to connect: %v", err)
    }
    defer conn.Close()
    
    client := pb.NewUserServiceClient(conn)
    
    ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    defer cancel()
    
    user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: "123"})
    if err != nil {
        // Handle gRPC errors
        st, ok := status.FromError(err)
        if ok {
            switch st.Code() {
            case codes.NotFound:
                log.Println("User not found")
            default:
                log.Printf("RPC error: %v", st.Message())
            }
        }
        return
    }
    
    fmt.Printf("User: %+v\n", user)
}
```

<a id="q6"></a>
### Q6: What are gRPC streaming modes?
**Answer:**

```protobuf
service StreamService {
    // Server streaming - server sends multiple responses
    rpc ServerStream(Request) returns (stream Response);
    
    // Client streaming - client sends multiple requests
    rpc ClientStream(stream Request) returns (Response);
    
    // Bidirectional streaming - both send multiple messages
    rpc BidirectionalStream(stream Request) returns (stream Response);
}
```

```go
// Server streaming implementation
func (s *server) ServerStream(req *pb.Request, stream pb.StreamService_ServerStreamServer) error {
    for i := 0; i < 10; i++ {
        if err := stream.Send(&pb.Response{
            Message: fmt.Sprintf("Response %d", i),
        }); err != nil {
            return err
        }
        time.Sleep(time.Second)
    }
    return nil
}

// Client streaming implementation
func (s *server) ClientStream(stream pb.StreamService_ClientStreamServer) error {
    var messages []string
    
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            // Client finished sending
            return stream.SendAndClose(&pb.Response{
                Message: fmt.Sprintf("Received %d messages", len(messages)),
            })
        }
        if err != nil {
            return err
        }
        messages = append(messages, req.Message)
    }
}

// Bidirectional streaming implementation
func (s *server) BidirectionalStream(stream pb.StreamService_BidirectionalStreamServer) error {
    for {
        req, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        
        // Process and respond
        if err := stream.Send(&pb.Response{
            Message: "Echo: " + req.Message,
        }); err != nil {
            return err
        }
    }
}

// Client-side server streaming
func clientServerStream(client pb.StreamServiceClient) {
    stream, err := client.ServerStream(context.Background(), &pb.Request{})
    if err != nil {
        log.Fatal(err)
    }
    
    for {
        resp, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
        fmt.Println(resp.Message)
    }
}

// Client-side bidirectional streaming
func clientBidirectional(client pb.StreamServiceClient) {
    stream, err := client.BidirectionalStream(context.Background())
    if err != nil {
        log.Fatal(err)
    }
    
    // Send goroutine
    go func() {
        for i := 0; i < 10; i++ {
            stream.Send(&pb.Request{Message: fmt.Sprintf("Message %d", i)})
        }
        stream.CloseSend()
    }()
    
    // Receive loop
    for {
        resp, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            log.Fatal(err)
        }
        fmt.Println(resp.Message)
    }
}
```

<a id="q7"></a>
### Q7: What are gRPC interceptors?
**Answer:**

Interceptors are middleware for gRPC:

```go
// Unary interceptor (single request-response)
func loggingUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    start := time.Now()
    
    // Call handler
    resp, err := handler(ctx, req)
    
    // Log
    log.Printf("Method: %s, Duration: %v, Error: %v",
        info.FullMethod, time.Since(start), err)
    
    return resp, err
}

// Stream interceptor
func loggingStreamInterceptor(
    srv interface{},
    ss grpc.ServerStream,
    info *grpc.StreamServerInfo,
    handler grpc.StreamHandler,
) error {
    start := time.Now()
    
    err := handler(srv, ss)
    
    log.Printf("Stream: %s, Duration: %v, Error: %v",
        info.FullMethod, time.Since(start), err)
    
    return err
}

// Authentication interceptor
func authUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (interface{}, error) {
    // Skip auth for certain methods
    if info.FullMethod == "/user.UserService/Login" {
        return handler(ctx, req)
    }
    
    // Get token from metadata
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        return nil, status.Error(codes.Unauthenticated, "missing metadata")
    }
    
    tokens := md.Get("authorization")
    if len(tokens) == 0 {
        return nil, status.Error(codes.Unauthenticated, "missing token")
    }
    
    // Validate token
    userID, err := validateToken(tokens[0])
    if err != nil {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }
    
    // Add user to context
    ctx = context.WithValue(ctx, "userID", userID)
    
    return handler(ctx, req)
}

// Recovery interceptor
func recoveryUnaryInterceptor(
    ctx context.Context,
    req interface{},
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler,
) (resp interface{}, err error) {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("Panic recovered: %v", r)
            err = status.Error(codes.Internal, "internal error")
        }
    }()
    
    return handler(ctx, req)
}

// Register interceptors
server := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        recoveryUnaryInterceptor,
        loggingUnaryInterceptor,
        authUnaryInterceptor,
    ),
    grpc.ChainStreamInterceptor(
        loggingStreamInterceptor,
    ),
)

// Client interceptor
func clientLoggingInterceptor(
    ctx context.Context,
    method string,
    req, reply interface{},
    cc *grpc.ClientConn,
    invoker grpc.UnaryInvoker,
    opts ...grpc.CallOption,
) error {
    start := time.Now()
    err := invoker(ctx, method, req, reply, cc, opts...)
    log.Printf("Client call: %s, Duration: %v", method, time.Since(start))
    return err
}

conn, _ := grpc.Dial("localhost:50051",
    grpc.WithUnaryInterceptor(clientLoggingInterceptor),
)
```

---

## Real-time Communication

<a id="q8"></a>
### Q8: How do you implement WebSockets in Go?
**Answer:**

```go
import "github.com/gorilla/websocket"

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        // Allow all origins (customize for production)
        return true
    },
}

// Simple WebSocket handler
func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := upgrader.Upgrade(w, r, nil)
    if err != nil {
        log.Printf("Upgrade error: %v", err)
        return
    }
    defer conn.Close()
    
    for {
        messageType, message, err := conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err, 
                websocket.CloseGoingAway, 
                websocket.CloseAbnormalClosure) {
                log.Printf("Read error: %v", err)
            }
            break
        }
        
        // Echo message back
        if err := conn.WriteMessage(messageType, message); err != nil {
            log.Printf("Write error: %v", err)
            break
        }
    }
}

// Chat room with Hub pattern
type Client struct {
    hub  *Hub
    conn *websocket.Conn
    send chan []byte
}

type Hub struct {
    clients    map[*Client]bool
    broadcast  chan []byte
    register   chan *Client
    unregister chan *Client
    mu         sync.RWMutex
}

func NewHub() *Hub {
    return &Hub{
        clients:    make(map[*Client]bool),
        broadcast:  make(chan []byte),
        register:   make(chan *Client),
        unregister: make(chan *Client),
    }
}

func (h *Hub) Run() {
    for {
        select {
        case client := <-h.register:
            h.mu.Lock()
            h.clients[client] = true
            h.mu.Unlock()
            
        case client := <-h.unregister:
            h.mu.Lock()
            if _, ok := h.clients[client]; ok {
                delete(h.clients, client)
                close(client.send)
            }
            h.mu.Unlock()
            
        case message := <-h.broadcast:
            h.mu.RLock()
            for client := range h.clients {
                select {
                case client.send <- message:
                default:
                    close(client.send)
                    delete(h.clients, client)
                }
            }
            h.mu.RUnlock()
        }
    }
}

func (c *Client) readPump() {
    defer func() {
        c.hub.unregister <- c
        c.conn.Close()
    }()
    
    c.conn.SetReadLimit(512)
    c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(60 * time.Second))
        return nil
    })
    
    for {
        _, message, err := c.conn.ReadMessage()
        if err != nil {
            break
        }
        c.hub.broadcast <- message
    }
}

func (c *Client) writePump() {
    ticker := time.NewTicker(54 * time.Second)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()
    
    for {
        select {
        case message, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if !ok {
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }
            
            if err := c.conn.WriteMessage(websocket.TextMessage, message); err != nil {
                return
            }
            
        case <-ticker.C:
            c.conn.SetWriteDeadline(time.Now().Add(10 * time.Second))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

<a id="q9"></a>
### Q9: How do you implement Server-Sent Events (SSE)?
**Answer:**

```go
// SSE handler
func sseHandler(w http.ResponseWriter, r *http.Request) {
    // Set headers for SSE
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    w.Header().Set("Access-Control-Allow-Origin", "*")
    
    // Get flusher
    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "SSE not supported", http.StatusInternalServerError)
        return
    }
    
    // Create client channel
    messageChan := make(chan string)
    
    // Register client
    clients.Add(messageChan)
    defer clients.Remove(messageChan)
    
    // Listen for client disconnect
    ctx := r.Context()
    
    for {
        select {
        case <-ctx.Done():
            return
        case msg := <-messageChan:
            fmt.Fprintf(w, "data: %s\n\n", msg)
            flusher.Flush()
        }
    }
}

// SSE with event types and IDs
func sseWithEvents(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    
    flusher := w.(http.Flusher)
    
    // Send event with ID and type
    sendEvent := func(id, eventType, data string) {
        if id != "" {
            fmt.Fprintf(w, "id: %s\n", id)
        }
        if eventType != "" {
            fmt.Fprintf(w, "event: %s\n", eventType)
        }
        fmt.Fprintf(w, "data: %s\n\n", data)
        flusher.Flush()
    }
    
    // Send events
    for i := 0; ; i++ {
        select {
        case <-r.Context().Done():
            return
        default:
            sendEvent(
                strconv.Itoa(i),
                "update",
                fmt.Sprintf(`{"count": %d}`, i),
            )
            time.Sleep(time.Second)
        }
    }
}

// Client manager for broadcasting
type SSEClientManager struct {
    clients map[chan string]bool
    mu      sync.RWMutex
}

func (m *SSEClientManager) Add(client chan string) {
    m.mu.Lock()
    m.clients[client] = true
    m.mu.Unlock()
}

func (m *SSEClientManager) Remove(client chan string) {
    m.mu.Lock()
    delete(m.clients, client)
    close(client)
    m.mu.Unlock()
}

func (m *SSEClientManager) Broadcast(message string) {
    m.mu.RLock()
    defer m.mu.RUnlock()
    
    for client := range m.clients {
        select {
        case client <- message:
        default:
            // Client not ready, skip
        }
    }
}

// JavaScript client
/*
const eventSource = new EventSource('/events');

eventSource.onmessage = (event) => {
    console.log('Message:', event.data);
};

eventSource.addEventListener('update', (event) => {
    console.log('Update:', event.data);
});

eventSource.onerror = (error) => {
    console.error('SSE Error:', error);
};
*/
```

<a id="q10"></a>
### Q10: How do you scale WebSocket connections?
**Answer:**

```go
// Challenge: WebSocket connections are stateful
// Solutions:

// 1. Sticky sessions (load balancer)
// - Route same client to same server
// - Use client IP or cookie

// 2. Pub/Sub with Redis
type RedisHub struct {
    rdb      *redis.Client
    pubsub   *redis.PubSub
    clients  map[*Client]bool
    mu       sync.RWMutex
    channel  string
}

func NewRedisHub(redisAddr, channel string) *RedisHub {
    rdb := redis.NewClient(&redis.Options{Addr: redisAddr})
    
    hub := &RedisHub{
        rdb:     rdb,
        clients: make(map[*Client]bool),
        channel: channel,
    }
    
    // Subscribe to Redis channel
    hub.pubsub = rdb.Subscribe(context.Background(), channel)
    
    go hub.subscribeLoop()
    
    return hub
}

func (h *RedisHub) subscribeLoop() {
    ch := h.pubsub.Channel()
    
    for msg := range ch {
        // Broadcast to local clients
        h.mu.RLock()
        for client := range h.clients {
            select {
            case client.send <- []byte(msg.Payload):
            default:
            }
        }
        h.mu.RUnlock()
    }
}

func (h *RedisHub) Broadcast(message []byte) {
    // Publish to Redis (reaches all servers)
    h.rdb.Publish(context.Background(), h.channel, message)
}

// 3. Room-based scaling with Redis
type Room struct {
    ID       string
    clients  map[*Client]bool
    rdb      *redis.Client
    mu       sync.RWMutex
}

func (r *Room) Join(client *Client) {
    r.mu.Lock()
    r.clients[client] = true
    r.mu.Unlock()
    
    // Subscribe to room channel
    go r.subscribeClient(client)
}

func (r *Room) Broadcast(message []byte) {
    // Publish to room-specific channel
    r.rdb.Publish(context.Background(), "room:"+r.ID, message)
}

// 4. Connection state in Redis
type ConnectionState struct {
    UserID    string
    ServerID  string
    RoomIDs   []string
    ConnectedAt time.Time
}

func (h *Hub) RegisterConnection(userID string) error {
    state := ConnectionState{
        UserID:      userID,
        ServerID:    h.serverID,
        ConnectedAt: time.Now(),
    }
    
    data, _ := json.Marshal(state)
    return h.rdb.Set(context.Background(), 
        "conn:"+userID, data, 24*time.Hour).Err()
}

// 5. Horizontal scaling architecture
/*
                    ┌─────────────┐
                    │Load Balancer│
                    │ (Sticky)    │
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌────────────┐  ┌────────────┐  ┌────────────┐
    │  Server 1  │  │  Server 2  │  │  Server 3  │
    │  (WS Hub)  │  │  (WS Hub)  │  │  (WS Hub)  │
    └─────┬──────┘  └─────┬──────┘  └─────┬──────┘
          └───────────────┼───────────────┘
                          ▼
                   ┌────────────┐
                   │   Redis    │
                   │  Pub/Sub   │
                   └────────────┘
*/
```

---

## API Best Practices

<a id="q11"></a>
### Q11: How do you handle API authentication?
**Answer:**

```go
// JWT Authentication middleware
func JWTAuthMiddleware(secretKey []byte) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Get token from header
            authHeader := r.Header.Get("Authorization")
            if authHeader == "" {
                http.Error(w, "Missing authorization header", http.StatusUnauthorized)
                return
            }
            
            // Parse "Bearer <token>"
            parts := strings.Split(authHeader, " ")
            if len(parts) != 2 || parts[0] != "Bearer" {
                http.Error(w, "Invalid authorization header", http.StatusUnauthorized)
                return
            }
            
            // Validate token
            token, err := jwt.Parse(parts[1], func(token *jwt.Token) (interface{}, error) {
                if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                    return nil, fmt.Errorf("unexpected signing method")
                }
                return secretKey, nil
            })
            
            if err != nil || !token.Valid {
                http.Error(w, "Invalid token", http.StatusUnauthorized)
                return
            }
            
            // Extract claims
            claims, ok := token.Claims.(jwt.MapClaims)
            if !ok {
                http.Error(w, "Invalid claims", http.StatusUnauthorized)
                return
            }
            
            // Add user to context
            ctx := context.WithValue(r.Context(), "userID", claims["sub"])
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}

// Generate JWT token
func GenerateToken(userID string, secretKey []byte) (string, error) {
    claims := jwt.MapClaims{
        "sub": userID,
        "iat": time.Now().Unix(),
        "exp": time.Now().Add(24 * time.Hour).Unix(),
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(secretKey)
}

// API Key authentication
func APIKeyMiddleware(validKeys map[string]bool) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            apiKey := r.Header.Get("X-API-Key")
            if apiKey == "" {
                apiKey = r.URL.Query().Get("api_key")
            }
            
            if !validKeys[apiKey] {
                http.Error(w, "Invalid API key", http.StatusUnauthorized)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}
```

<a id="q12"></a>
### Q12: How do you document APIs?
**Answer:**

```go
// Using swaggo/swag for Swagger documentation

// @title           User API
// @version         1.0
// @description     User management API
// @host            localhost:8080
// @BasePath        /api/v1

// @securityDefinitions.apikey Bearer
// @in header
// @name Authorization

// Handler documentation
// @Summary      Get user by ID
// @Description  Get detailed user information
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        id   path      string  true  "User ID"
// @Success      200  {object}  User
// @Failure      404  {object}  ErrorResponse
// @Failure      500  {object}  ErrorResponse
// @Security     Bearer
// @Router       /users/{id} [get]
func GetUser(w http.ResponseWriter, r *http.Request) {
    // Implementation
}

// @Summary      Create user
// @Description  Create a new user
// @Tags         users
// @Accept       json
// @Produce      json
// @Param        user  body      CreateUserRequest  true  "User data"
// @Success      201   {object}  User
// @Failure      400   {object}  ErrorResponse
// @Failure      500   {object}  ErrorResponse
// @Router       /users [post]
func CreateUser(w http.ResponseWriter, r *http.Request) {
    // Implementation
}

// Model documentation
type User struct {
    ID        string    `json:"id" example:"123"`
    Name      string    `json:"name" example:"John Doe"`
    Email     string    `json:"email" example:"john@example.com"`
    CreatedAt time.Time `json:"created_at"`
}

type CreateUserRequest struct {
    Name  string `json:"name" binding:"required" example:"John Doe"`
    Email string `json:"email" binding:"required,email" example:"john@example.com"`
}

type ErrorResponse struct {
    Error string `json:"error" example:"user not found"`
}

// Generate docs: swag init
// Serve docs at /swagger/*
```

<a id="q13"></a>
### Q13: How do you handle API errors consistently?
**Answer:**

```go
// Standard error response
type APIError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Details any    `json:"details,omitempty"`
}

// Error codes
const (
    ErrCodeValidation   = "VALIDATION_ERROR"
    ErrCodeNotFound     = "NOT_FOUND"
    ErrCodeUnauthorized = "UNAUTHORIZED"
    ErrCodeForbidden    = "FORBIDDEN"
    ErrCodeConflict     = "CONFLICT"
    ErrCodeInternal     = "INTERNAL_ERROR"
)

// Custom error type
type AppError struct {
    StatusCode int
    Code       string
    Message    string
    Err        error
}

func (e *AppError) Error() string {
    return e.Message
}

// Error constructors
func NewValidationError(message string, details any) *AppError {
    return &AppError{
        StatusCode: http.StatusBadRequest,
        Code:       ErrCodeValidation,
        Message:    message,
    }
}

func NewNotFoundError(resource string) *AppError {
    return &AppError{
        StatusCode: http.StatusNotFound,
        Code:       ErrCodeNotFound,
        Message:    fmt.Sprintf("%s not found", resource),
    }
}

// Error handler middleware
func ErrorHandler(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                log.Printf("Panic: %v", err)
                respondError(w, &AppError{
                    StatusCode: http.StatusInternalServerError,
                    Code:       ErrCodeInternal,
                    Message:    "Internal server error",
                })
            }
        }()
        next.ServeHTTP(w, r)
    })
}

func respondError(w http.ResponseWriter, err *AppError) {
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(err.StatusCode)
    json.NewEncoder(w).Encode(APIError{
        Code:    err.Code,
        Message: err.Message,
    })
}

// Usage in handler
func GetUser(w http.ResponseWriter, r *http.Request) {
    user, err := userService.Get(r.Context(), id)
    if err != nil {
        if errors.Is(err, ErrUserNotFound) {
            respondError(w, NewNotFoundError("user"))
            return
        }
        respondError(w, &AppError{
            StatusCode: http.StatusInternalServerError,
            Code:       ErrCodeInternal,
            Message:    "Failed to get user",
            Err:        err,
        })
        return
    }
    respondJSON(w, http.StatusOK, user)
}
```

<a id="q14"></a>
### Q14: What are best practices for API pagination?
**Answer:**

```go
// Offset-based pagination
type PaginationParams struct {
    Page  int `form:"page" binding:"min=1"`
    Limit int `form:"limit" binding:"min=1,max=100"`
}

type PaginatedResponse[T any] struct {
    Data       []T  `json:"data"`
    Page       int  `json:"page"`
    Limit      int  `json:"limit"`
    Total      int  `json:"total"`
    TotalPages int  `json:"total_pages"`
    HasMore    bool `json:"has_more"`
}

func Paginate[T any](data []T, page, limit, total int) PaginatedResponse[T] {
    totalPages := (total + limit - 1) / limit
    return PaginatedResponse[T]{
        Data:       data,
        Page:       page,
        Limit:      limit,
        Total:      total,
        TotalPages: totalPages,
        HasMore:    page < totalPages,
    }
}

// Cursor-based pagination (better for large datasets)
type CursorParams struct {
    Cursor string `form:"cursor"`
    Limit  int    `form:"limit" binding:"min=1,max=100"`
}

type CursorResponse[T any] struct {
    Data       []T    `json:"data"`
    NextCursor string `json:"next_cursor,omitempty"`
    HasMore    bool   `json:"has_more"`
}

func (s *UserService) ListWithCursor(ctx context.Context, cursor string, limit int) (*CursorResponse[User], error) {
    // Decode cursor (e.g., base64 encoded ID or timestamp)
    var lastID string
    if cursor != "" {
        decoded, _ := base64.StdEncoding.DecodeString(cursor)
        lastID = string(decoded)
    }
    
    // Query with cursor
    query := `SELECT * FROM users WHERE id > $1 ORDER BY id LIMIT $2`
    users, err := s.db.Query(ctx, query, lastID, limit+1)
    if err != nil {
        return nil, err
    }
    
    hasMore := len(users) > limit
    if hasMore {
        users = users[:limit]
    }
    
    var nextCursor string
    if hasMore && len(users) > 0 {
        lastUser := users[len(users)-1]
        nextCursor = base64.StdEncoding.EncodeToString([]byte(lastUser.ID))
    }
    
    return &CursorResponse[User]{
        Data:       users,
        NextCursor: nextCursor,
        HasMore:    hasMore,
    }, nil
}

// Link header pagination (REST standard)
func SetPaginationHeaders(w http.ResponseWriter, baseURL string, page, limit, total int) {
    totalPages := (total + limit - 1) / limit
    var links []string
    
    if page > 1 {
        links = append(links, fmt.Sprintf(`<%s?page=1&limit=%d>; rel="first"`, baseURL, limit))
        links = append(links, fmt.Sprintf(`<%s?page=%d&limit=%d>; rel="prev"`, baseURL, page-1, limit))
    }
    
    if page < totalPages {
        links = append(links, fmt.Sprintf(`<%s?page=%d&limit=%d>; rel="next"`, baseURL, page+1, limit))
        links = append(links, fmt.Sprintf(`<%s?page=%d&limit=%d>; rel="last"`, baseURL, totalPages, limit))
    }
    
    if len(links) > 0 {
        w.Header().Set("Link", strings.Join(links, ", "))
    }
    
    w.Header().Set("X-Total-Count", strconv.Itoa(total))
    w.Header().Set("X-Total-Pages", strconv.Itoa(totalPages))
}
```

---

[← Back to Go Index](README.md)

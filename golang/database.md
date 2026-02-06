# Database Integration

## Table of Contents

### Database Basics
- [Q1: How do you connect to SQL databases in Go?](#q1)
- [Q2: How do you handle database transactions?](#q2)
- [Q3: What is connection pooling and how do you configure it?](#q3)

### ORMs and Query Builders
- [Q4: How do you use GORM?](#q4)
- [Q5: How do you use sqlx?](#q5)
- [Q6: How do you use sqlc?](#q6)
- [Q7: When should you use ORM vs raw SQL?](#q7)

### NoSQL Databases
- [Q8: How do you work with Redis in Go?](#q8)
- [Q9: How do you work with MongoDB in Go?](#q9)

### Best Practices
- [Q10: How do you handle database migrations?](#q10)
- [Q11: How do you implement caching strategies?](#q11)
- [Q12: How do you prevent SQL injection?](#q12)

---

## Database Basics

<a id="q1"></a>
### Q1: How do you connect to SQL databases in Go?
**Answer:**

```go
import (
    "database/sql"
    _ "github.com/lib/pq"           // PostgreSQL
    _ "github.com/go-sql-driver/mysql" // MySQL
)

// PostgreSQL connection
func connectPostgres() (*sql.DB, error) {
    connStr := "host=localhost port=5432 user=postgres password=secret dbname=mydb sslmode=disable"
    
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, err
    }
    
    // Verify connection
    if err := db.Ping(); err != nil {
        return nil, err
    }
    
    return db, nil
}

// MySQL connection
func connectMySQL() (*sql.DB, error) {
    connStr := "user:password@tcp(localhost:3306)/dbname?parseTime=true"
    return sql.Open("mysql", connStr)
}

// Query single row
func getUser(db *sql.DB, id int) (*User, error) {
    var user User
    err := db.QueryRow(
        "SELECT id, name, email FROM users WHERE id = $1", id,
    ).Scan(&user.ID, &user.Name, &user.Email)
    
    if err == sql.ErrNoRows {
        return nil, ErrNotFound
    }
    if err != nil {
        return nil, err
    }
    return &user, nil
}

// Query multiple rows
func listUsers(db *sql.DB) ([]User, error) {
    rows, err := db.Query("SELECT id, name, email FROM users ORDER BY id")
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    var users []User
    for rows.Next() {
        var u User
        if err := rows.Scan(&u.ID, &u.Name, &u.Email); err != nil {
            return nil, err
        }
        users = append(users, u)
    }
    
    // Check for errors during iteration
    if err := rows.Err(); err != nil {
        return nil, err
    }
    
    return users, nil
}

// Insert with returning ID
func createUser(db *sql.DB, name, email string) (int, error) {
    var id int
    err := db.QueryRow(
        "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
        name, email,
    ).Scan(&id)
    return id, err
}

// Update
func updateUser(db *sql.DB, id int, name string) error {
    result, err := db.Exec(
        "UPDATE users SET name = $1 WHERE id = $2",
        name, id,
    )
    if err != nil {
        return err
    }
    
    rowsAffected, _ := result.RowsAffected()
    if rowsAffected == 0 {
        return ErrNotFound
    }
    return nil
}

// Context-aware queries
func getUserCtx(ctx context.Context, db *sql.DB, id int) (*User, error) {
    var user User
    err := db.QueryRowContext(ctx,
        "SELECT id, name, email FROM users WHERE id = $1", id,
    ).Scan(&user.ID, &user.Name, &user.Email)
    return &user, err
}
```

<a id="q2"></a>
### Q2: How do you handle database transactions?
**Answer:**

```go
// Basic transaction
func transferFunds(db *sql.DB, fromID, toID int, amount float64) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback()  // Rollback if not committed
    
    // Deduct from sender
    _, err = tx.Exec(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
        amount, fromID,
    )
    if err != nil {
        return err
    }
    
    // Add to receiver
    _, err = tx.Exec(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
        amount, toID,
    )
    if err != nil {
        return err
    }
    
    return tx.Commit()
}

// Transaction with context
func transferFundsCtx(ctx context.Context, db *sql.DB, fromID, toID int, amount float64) error {
    tx, err := db.BeginTx(ctx, nil)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Queries...
    
    return tx.Commit()
}

// Transaction with isolation level
func transferFundsSerializable(ctx context.Context, db *sql.DB, fromID, toID int, amount float64) error {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
        ReadOnly:  false,
    })
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    // Queries...
    
    return tx.Commit()
}

// Transaction helper pattern
func WithTransaction(db *sql.DB, fn func(*sql.Tx) error) error {
    tx, err := db.Begin()
    if err != nil {
        return err
    }
    
    defer func() {
        if p := recover(); p != nil {
            tx.Rollback()
            panic(p)
        }
    }()
    
    if err := fn(tx); err != nil {
        tx.Rollback()
        return err
    }
    
    return tx.Commit()
}

// Usage
err := WithTransaction(db, func(tx *sql.Tx) error {
    if _, err := tx.Exec("INSERT INTO orders ..."); err != nil {
        return err
    }
    if _, err := tx.Exec("UPDATE inventory ..."); err != nil {
        return err
    }
    return nil
})

// Savepoints (PostgreSQL)
func complexTransaction(tx *sql.Tx) error {
    // Main operations
    if _, err := tx.Exec("INSERT INTO orders ..."); err != nil {
        return err
    }
    
    // Savepoint for optional operation
    _, err := tx.Exec("SAVEPOINT optional_op")
    if err != nil {
        return err
    }
    
    if _, err := tx.Exec("INSERT INTO audit_log ..."); err != nil {
        // Rollback only the savepoint, not the whole transaction
        tx.Exec("ROLLBACK TO SAVEPOINT optional_op")
    }
    
    return nil
}
```

<a id="q3"></a>
### Q3: What is connection pooling and how do you configure it?
**Answer:**

```go
// database/sql has built-in connection pooling
func configurePool(db *sql.DB) {
    // Maximum number of open connections
    db.SetMaxOpenConns(25)
    
    // Maximum number of idle connections
    db.SetMaxIdleConns(25)
    
    // Maximum lifetime of a connection
    db.SetConnMaxLifetime(5 * time.Minute)
    
    // Maximum time a connection can be idle (Go 1.15+)
    db.SetConnMaxIdleTime(5 * time.Minute)
}

// Pool statistics
func monitorPool(db *sql.DB) {
    stats := db.Stats()
    
    fmt.Printf("Open connections: %d\n", stats.OpenConnections)
    fmt.Printf("In use: %d\n", stats.InUse)
    fmt.Printf("Idle: %d\n", stats.Idle)
    fmt.Printf("Wait count: %d\n", stats.WaitCount)
    fmt.Printf("Wait duration: %v\n", stats.WaitDuration)
}

// Connection pool best practices
/*
1. MaxOpenConns: Based on database server capacity
   - PostgreSQL default max_connections = 100
   - Leave room for other applications
   - Consider: (max_connections - 10) / num_app_instances

2. MaxIdleConns: Usually same as MaxOpenConns
   - Lower value = more connection churn
   - Higher value = more memory usage

3. ConnMaxLifetime: Prevent stale connections
   - Set lower than DB server timeout
   - Helps with load balancer rotation
   - 5-10 minutes is common

4. ConnMaxIdleTime: Clean up idle connections
   - Similar to ConnMaxLifetime
   - Helps reduce resource usage during low traffic
*/

// Production configuration example
func NewDB(dsn string) (*sql.DB, error) {
    db, err := sql.Open("postgres", dsn)
    if err != nil {
        return nil, err
    }
    
    // Pool configuration
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(25)
    db.SetConnMaxLifetime(5 * time.Minute)
    db.SetConnMaxIdleTime(5 * time.Minute)
    
    // Verify connection
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    if err := db.PingContext(ctx); err != nil {
        return nil, err
    }
    
    return db, nil
}
```

---

## ORMs and Query Builders

<a id="q4"></a>
### Q4: How do you use GORM?
**Answer:**

```go
import "gorm.io/gorm"
import "gorm.io/driver/postgres"

// Model definition
type User struct {
    gorm.Model           // ID, CreatedAt, UpdatedAt, DeletedAt
    Name     string      `gorm:"size:255;not null"`
    Email    string      `gorm:"uniqueIndex;not null"`
    Age      int
    Profile  Profile     // Has One
    Orders   []Order     // Has Many
}

type Profile struct {
    gorm.Model
    UserID  uint
    Bio     string
    Avatar  string
}

type Order struct {
    gorm.Model
    UserID  uint
    Total   float64
    Status  string
}

// Connect
func connectGorm() (*gorm.DB, error) {
    dsn := "host=localhost user=postgres password=secret dbname=mydb port=5432"
    db, err := gorm.Open(postgres.Open(dsn), &gorm.Config{})
    if err != nil {
        return nil, err
    }
    
    // Auto migrate
    db.AutoMigrate(&User{}, &Profile{}, &Order{})
    
    return db, nil
}

// CRUD operations
func gormExamples(db *gorm.DB) {
    // Create
    user := User{Name: "Alice", Email: "alice@example.com", Age: 30}
    result := db.Create(&user)
    fmt.Println(user.ID)  // Auto-filled
    fmt.Println(result.RowsAffected)
    
    // Create with associations
    user := User{
        Name:  "Bob",
        Email: "bob@example.com",
        Profile: Profile{Bio: "Developer"},
    }
    db.Create(&user)
    
    // Read
    var u User
    db.First(&u, 1)                        // By primary key
    db.First(&u, "email = ?", "alice@example.com")  // By condition
    
    // Find multiple
    var users []User
    db.Where("age > ?", 18).Find(&users)
    
    // Select specific columns
    db.Select("name", "email").Find(&users)
    
    // Update
    db.Model(&user).Update("name", "Alice Updated")
    db.Model(&user).Updates(User{Name: "New Name", Age: 31})
    db.Model(&user).Updates(map[string]interface{}{"name": "New", "age": 32})
    
    // Delete (soft delete with gorm.Model)
    db.Delete(&user, 1)
    
    // Hard delete
    db.Unscoped().Delete(&user, 1)
}

// Associations
func gormAssociations(db *gorm.DB) {
    // Eager loading
    var user User
    db.Preload("Profile").Preload("Orders").First(&user, 1)
    
    // Nested preload
    db.Preload("Orders.Items").First(&user, 1)
    
    // Conditional preload
    db.Preload("Orders", "status = ?", "completed").First(&user, 1)
    
    // Joins
    var users []User
    db.Joins("Profile").Find(&users)
}

// Transactions
func gormTransaction(db *gorm.DB) error {
    return db.Transaction(func(tx *gorm.DB) error {
        if err := tx.Create(&User{Name: "Alice"}).Error; err != nil {
            return err
        }
        
        if err := tx.Create(&Order{UserID: 1, Total: 100}).Error; err != nil {
            return err
        }
        
        return nil
    })
}

// Raw SQL
func gormRawSQL(db *gorm.DB) {
    var users []User
    db.Raw("SELECT * FROM users WHERE age > ?", 18).Scan(&users)
    
    db.Exec("UPDATE users SET age = ? WHERE id = ?", 30, 1)
}

// Hooks
func (u *User) BeforeCreate(tx *gorm.DB) error {
    // Validation or modification before create
    if u.Name == "" {
        return errors.New("name is required")
    }
    return nil
}

func (u *User) AfterCreate(tx *gorm.DB) error {
    // Actions after create
    log.Printf("User %d created", u.ID)
    return nil
}
```

<a id="q5"></a>
### Q5: How do you use sqlx?
**Answer:**

```go
import "github.com/jmoiron/sqlx"

// Model with db tags
type User struct {
    ID        int       `db:"id"`
    Name      string    `db:"name"`
    Email     string    `db:"email"`
    CreatedAt time.Time `db:"created_at"`
}

// Connect
func connectSqlx() (*sqlx.DB, error) {
    db, err := sqlx.Connect("postgres", 
        "host=localhost user=postgres password=secret dbname=mydb sslmode=disable")
    return db, err
}

// Get single row
func getUser(db *sqlx.DB, id int) (*User, error) {
    var user User
    err := db.Get(&user, "SELECT * FROM users WHERE id = $1", id)
    if err == sql.ErrNoRows {
        return nil, ErrNotFound
    }
    return &user, err
}

// Get multiple rows
func listUsers(db *sqlx.DB) ([]User, error) {
    var users []User
    err := db.Select(&users, "SELECT * FROM users ORDER BY id")
    return users, err
}

// Named queries
func createUserNamed(db *sqlx.DB, user *User) error {
    query := `INSERT INTO users (name, email) 
              VALUES (:name, :email) 
              RETURNING id, created_at`
    
    rows, err := db.NamedQuery(query, user)
    if err != nil {
        return err
    }
    defer rows.Close()
    
    if rows.Next() {
        rows.Scan(&user.ID, &user.CreatedAt)
    }
    return nil
}

// Named exec
func updateUserNamed(db *sqlx.DB, user *User) error {
    query := `UPDATE users SET name = :name, email = :email WHERE id = :id`
    _, err := db.NamedExec(query, user)
    return err
}

// In clause expansion
func getUsersByIDs(db *sqlx.DB, ids []int) ([]User, error) {
    query, args, err := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)
    if err != nil {
        return nil, err
    }
    
    // Rebind for PostgreSQL ($1, $2 instead of ?)
    query = db.Rebind(query)
    
    var users []User
    err = db.Select(&users, query, args...)
    return users, err
}

// Transaction
func transferSqlx(db *sqlx.DB, fromID, toID int, amount float64) error {
    tx, err := db.Beginx()
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    _, err = tx.Exec("UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, fromID)
    if err != nil {
        return err
    }
    
    _, err = tx.Exec("UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, toID)
    if err != nil {
        return err
    }
    
    return tx.Commit()
}

// Struct scanning with joins
type UserWithProfile struct {
    User
    ProfileBio string `db:"profile_bio"`
}

func getUserWithProfile(db *sqlx.DB, id int) (*UserWithProfile, error) {
    var result UserWithProfile
    query := `
        SELECT u.*, p.bio as profile_bio 
        FROM users u 
        LEFT JOIN profiles p ON p.user_id = u.id 
        WHERE u.id = $1`
    
    err := db.Get(&result, query, id)
    return &result, err
}
```

<a id="q6"></a>
### Q6: How do you use sqlc?
**Answer:**

```sql
-- queries.sql
-- name: GetUser :one
SELECT * FROM users WHERE id = $1;

-- name: ListUsers :many
SELECT * FROM users ORDER BY id LIMIT $1 OFFSET $2;

-- name: CreateUser :one
INSERT INTO users (name, email)
VALUES ($1, $2)
RETURNING *;

-- name: UpdateUser :exec
UPDATE users SET name = $1, email = $2 WHERE id = $3;

-- name: DeleteUser :exec
DELETE FROM users WHERE id = $1;

-- name: GetUsersByIDs :many
SELECT * FROM users WHERE id = ANY($1::int[]);
```

```yaml
# sqlc.yaml
version: "2"
sql:
  - engine: "postgresql"
    queries: "queries.sql"
    schema: "schema.sql"
    gen:
      go:
        package: "db"
        out: "internal/db"
        sql_package: "pgx/v5"
```

```go
// Generated code usage
func main() {
    ctx := context.Background()
    
    conn, err := pgx.Connect(ctx, "postgres://...")
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close(ctx)
    
    queries := db.New(conn)
    
    // Create user (type-safe!)
    user, err := queries.CreateUser(ctx, db.CreateUserParams{
        Name:  "Alice",
        Email: "alice@example.com",
    })
    
    // Get user
    user, err := queries.GetUser(ctx, 1)
    
    // List users
    users, err := queries.ListUsers(ctx, db.ListUsersParams{
        Limit:  10,
        Offset: 0,
    })
    
    // Update
    err = queries.UpdateUser(ctx, db.UpdateUserParams{
        ID:    1,
        Name:  "Alice Updated",
        Email: "alice@example.com",
    })
}

// Transaction with sqlc
func transferWithSqlc(ctx context.Context, conn *pgx.Conn, fromID, toID int32, amount float64) error {
    tx, err := conn.Begin(ctx)
    if err != nil {
        return err
    }
    defer tx.Rollback(ctx)
    
    queries := db.New(tx)
    
    // Use generated methods
    if err := queries.DeductBalance(ctx, db.DeductBalanceParams{
        ID:     fromID,
        Amount: amount,
    }); err != nil {
        return err
    }
    
    if err := queries.AddBalance(ctx, db.AddBalanceParams{
        ID:     toID,
        Amount: amount,
    }); err != nil {
        return err
    }
    
    return tx.Commit(ctx)
}
```

<a id="q7"></a>
### Q7: When should you use ORM vs raw SQL?
**Answer:**

| Use ORM (GORM) | Use Query Builder (sqlx) | Use Code Gen (sqlc) |
|----------------|--------------------------|---------------------|
| Rapid prototyping | Medium complexity | Complex queries |
| Simple CRUD | Need SQL control | Maximum type safety |
| Associations needed | Performance critical | Compile-time checks |
| Less SQL expertise | Team knows SQL | Large codebases |

```go
// ORM pros: Easy associations, migrations, hooks
// ORM cons: Magic, N+1 queries, learning curve

// GORM - good for
db.Preload("Orders.Items").Where("active = ?", true).Find(&users)

// sqlx - good for
query := `
    SELECT u.*, COUNT(o.id) as order_count
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    WHERE u.active = true
    GROUP BY u.id
    HAVING COUNT(o.id) > 10
`
db.Select(&results, query)

// sqlc - good for
// Compile-time SQL validation
// Auto-generated type-safe Go code
// No runtime reflection

// Decision matrix:
/*
1. Team SQL experience?
   - Low → ORM
   - High → sqlx/sqlc

2. Query complexity?
   - Simple CRUD → ORM
   - Complex joins/aggregates → sqlx/sqlc

3. Performance requirements?
   - Standard → Any
   - High → sqlx/sqlc

4. Type safety needs?
   - Standard → ORM/sqlx
   - Maximum → sqlc

5. Codebase size?
   - Small → Any
   - Large → sqlc (catches errors at compile time)
*/
```

---

## NoSQL Databases

<a id="q8"></a>
### Q8: How do you work with Redis in Go?
**Answer:**

```go
import "github.com/redis/go-redis/v9"

// Connect
func connectRedis() *redis.Client {
    client := redis.NewClient(&redis.Options{
        Addr:     "localhost:6379",
        Password: "",
        DB:       0,
        PoolSize: 10,
    })
    return client
}

// Basic operations
func redisBasics(ctx context.Context, rdb *redis.Client) {
    // String
    err := rdb.Set(ctx, "key", "value", time.Hour).Err()
    val, err := rdb.Get(ctx, "key").Result()
    
    // Check if key exists
    if errors.Is(err, redis.Nil) {
        fmt.Println("Key does not exist")
    }
    
    // Set with expiration
    rdb.SetEx(ctx, "session:123", "user_data", 24*time.Hour)
    
    // Increment
    rdb.Incr(ctx, "counter")
    rdb.IncrBy(ctx, "counter", 10)
    
    // Hash
    rdb.HSet(ctx, "user:1", "name", "Alice", "email", "alice@example.com")
    name, _ := rdb.HGet(ctx, "user:1", "name").Result()
    user, _ := rdb.HGetAll(ctx, "user:1").Result()  // map[string]string
    
    // List
    rdb.RPush(ctx, "queue", "item1", "item2")
    item, _ := rdb.LPop(ctx, "queue").Result()
    
    // Set
    rdb.SAdd(ctx, "tags", "go", "redis", "database")
    tags, _ := rdb.SMembers(ctx, "tags").Result()
    
    // Sorted Set
    rdb.ZAdd(ctx, "leaderboard", redis.Z{Score: 100, Member: "player1"})
    top, _ := rdb.ZRevRangeWithScores(ctx, "leaderboard", 0, 9).Result()
    
    // Delete
    rdb.Del(ctx, "key1", "key2")
    
    // TTL
    ttl, _ := rdb.TTL(ctx, "key").Result()
    rdb.Expire(ctx, "key", time.Hour)
}

// Pipeline (batch operations)
func redisPipeline(ctx context.Context, rdb *redis.Client) {
    pipe := rdb.Pipeline()
    
    pipe.Set(ctx, "key1", "value1", 0)
    pipe.Set(ctx, "key2", "value2", 0)
    pipe.Incr(ctx, "counter")
    
    cmds, err := pipe.Exec(ctx)
    // Process results...
}

// Transaction (MULTI/EXEC)
func redisTransaction(ctx context.Context, rdb *redis.Client) error {
    // Optimistic locking with WATCH
    err := rdb.Watch(ctx, func(tx *redis.Tx) error {
        val, err := tx.Get(ctx, "balance").Int()
        if err != nil {
            return err
        }
        
        if val < 100 {
            return errors.New("insufficient balance")
        }
        
        _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
            pipe.DecrBy(ctx, "balance", 100)
            pipe.IncrBy(ctx, "spent", 100)
            return nil
        })
        return err
    }, "balance")
    
    return err
}

// Pub/Sub
func redisPubSub(ctx context.Context, rdb *redis.Client) {
    // Subscriber
    pubsub := rdb.Subscribe(ctx, "channel1")
    defer pubsub.Close()
    
    ch := pubsub.Channel()
    go func() {
        for msg := range ch {
            fmt.Printf("Received: %s\n", msg.Payload)
        }
    }()
    
    // Publisher
    rdb.Publish(ctx, "channel1", "Hello!")
}

// Caching pattern
func getUser(ctx context.Context, rdb *redis.Client, db *sql.DB, userID string) (*User, error) {
    // Try cache first
    data, err := rdb.Get(ctx, "user:"+userID).Bytes()
    if err == nil {
        var user User
        json.Unmarshal(data, &user)
        return &user, nil
    }
    
    // Cache miss - fetch from DB
    user, err := getUserFromDB(db, userID)
    if err != nil {
        return nil, err
    }
    
    // Store in cache
    data, _ = json.Marshal(user)
    rdb.Set(ctx, "user:"+userID, data, time.Hour)
    
    return user, nil
}
```

<a id="q9"></a>
### Q9: How do you work with MongoDB in Go?
**Answer:**

```go
import "go.mongodb.org/mongo-driver/mongo"
import "go.mongodb.org/mongo-driver/bson"

// Connect
func connectMongo(ctx context.Context) (*mongo.Client, error) {
    client, err := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))
    if err != nil {
        return nil, err
    }
    
    // Ping
    if err := client.Ping(ctx, nil); err != nil {
        return nil, err
    }
    
    return client, nil
}

// Model
type User struct {
    ID        primitive.ObjectID `bson:"_id,omitempty"`
    Name      string             `bson:"name"`
    Email     string             `bson:"email"`
    Age       int                `bson:"age"`
    Tags      []string           `bson:"tags"`
    CreatedAt time.Time          `bson:"created_at"`
}

// CRUD operations
func mongoCRUD(ctx context.Context, coll *mongo.Collection) {
    // Insert one
    user := User{
        Name:      "Alice",
        Email:     "alice@example.com",
        Age:       30,
        Tags:      []string{"developer", "go"},
        CreatedAt: time.Now(),
    }
    result, err := coll.InsertOne(ctx, user)
    id := result.InsertedID.(primitive.ObjectID)
    
    // Insert many
    users := []interface{}{user1, user2, user3}
    result, err := coll.InsertMany(ctx, users)
    
    // Find one
    var found User
    err = coll.FindOne(ctx, bson.M{"email": "alice@example.com"}).Decode(&found)
    
    // Find by ID
    objID, _ := primitive.ObjectIDFromHex("...")
    err = coll.FindOne(ctx, bson.M{"_id": objID}).Decode(&found)
    
    // Find many
    cursor, err := coll.Find(ctx, bson.M{"age": bson.M{"$gte": 18}})
    defer cursor.Close(ctx)
    
    var users []User
    if err = cursor.All(ctx, &users); err != nil {
        return
    }
    
    // Update one
    filter := bson.M{"_id": id}
    update := bson.M{"$set": bson.M{"name": "Alice Updated"}}
    result, err := coll.UpdateOne(ctx, filter, update)
    
    // Update many
    filter := bson.M{"age": bson.M{"$lt": 18}}
    update := bson.M{"$set": bson.M{"status": "minor"}}
    result, err := coll.UpdateMany(ctx, filter, update)
    
    // Delete
    result, err := coll.DeleteOne(ctx, bson.M{"_id": id})
    result, err := coll.DeleteMany(ctx, bson.M{"status": "inactive"})
}

// Aggregation pipeline
func mongoAggregation(ctx context.Context, coll *mongo.Collection) {
    pipeline := []bson.M{
        {"$match": bson.M{"status": "active"}},
        {"$group": bson.M{
            "_id":   "$department",
            "count": bson.M{"$sum": 1},
            "avg_salary": bson.M{"$avg": "$salary"},
        }},
        {"$sort": bson.M{"count": -1}},
        {"$limit": 10},
    }
    
    cursor, err := coll.Aggregate(ctx, pipeline)
    defer cursor.Close(ctx)
    
    var results []bson.M
    cursor.All(ctx, &results)
}

// Indexes
func createIndexes(ctx context.Context, coll *mongo.Collection) {
    // Single field index
    coll.Indexes().CreateOne(ctx, mongo.IndexModel{
        Keys: bson.M{"email": 1},
        Options: options.Index().SetUnique(true),
    })
    
    // Compound index
    coll.Indexes().CreateOne(ctx, mongo.IndexModel{
        Keys: bson.D{
            {Key: "status", Value: 1},
            {Key: "created_at", Value: -1},
        },
    })
    
    // Text index
    coll.Indexes().CreateOne(ctx, mongo.IndexModel{
        Keys: bson.M{"content": "text"},
    })
}

// Transaction
func mongoTransaction(ctx context.Context, client *mongo.Client) error {
    session, err := client.StartSession()
    if err != nil {
        return err
    }
    defer session.EndSession(ctx)
    
    _, err = session.WithTransaction(ctx, func(sessCtx mongo.SessionContext) (interface{}, error) {
        coll1 := client.Database("db").Collection("accounts")
        coll2 := client.Database("db").Collection("transactions")
        
        // Operations within transaction
        _, err := coll1.UpdateOne(sessCtx, ...)
        if err != nil {
            return nil, err
        }
        
        _, err = coll2.InsertOne(sessCtx, ...)
        return nil, err
    })
    
    return err
}
```

---

## Best Practices

<a id="q10"></a>
### Q10: How do you handle database migrations?
**Answer:**

```go
// Using golang-migrate
import "github.com/golang-migrate/migrate/v4"

// Migration files
// migrations/
//   000001_create_users_table.up.sql
//   000001_create_users_table.down.sql
//   000002_add_email_index.up.sql
//   000002_add_email_index.down.sql

// 000001_create_users_table.up.sql
/*
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
*/

// 000001_create_users_table.down.sql
/*
DROP TABLE users;
*/

// Run migrations programmatically
func runMigrations(dbURL string) error {
    m, err := migrate.New(
        "file://migrations",
        dbURL,
    )
    if err != nil {
        return err
    }
    
    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        return err
    }
    
    return nil
}

// Embed migrations in binary
import "github.com/golang-migrate/migrate/v4/source/iofs"

//go:embed migrations/*.sql
var migrations embed.FS

func runEmbeddedMigrations(db *sql.DB) error {
    source, err := iofs.New(migrations, "migrations")
    if err != nil {
        return err
    }
    
    driver, err := postgres.WithInstance(db, &postgres.Config{})
    if err != nil {
        return err
    }
    
    m, err := migrate.NewWithInstance("iofs", source, "postgres", driver)
    if err != nil {
        return err
    }
    
    return m.Up()
}

// CLI commands
// migrate -path migrations -database "postgres://..." up
// migrate -path migrations -database "postgres://..." down 1
// migrate -path migrations -database "postgres://..." version
// migrate create -ext sql -dir migrations -seq create_orders_table
```

<a id="q11"></a>
### Q11: How do you implement caching strategies?
**Answer:**

```go
// Cache-Aside (Lazy Loading)
func getUser(ctx context.Context, cache *redis.Client, db *sql.DB, id string) (*User, error) {
    // 1. Check cache
    data, err := cache.Get(ctx, "user:"+id).Bytes()
    if err == nil {
        var user User
        json.Unmarshal(data, &user)
        return &user, nil
    }
    
    // 2. Cache miss - load from DB
    user, err := loadUserFromDB(db, id)
    if err != nil {
        return nil, err
    }
    
    // 3. Store in cache
    data, _ := json.Marshal(user)
    cache.Set(ctx, "user:"+id, data, time.Hour)
    
    return user, nil
}

// Write-Through
func updateUser(ctx context.Context, cache *redis.Client, db *sql.DB, user *User) error {
    // 1. Update DB
    if err := updateUserInDB(db, user); err != nil {
        return err
    }
    
    // 2. Update cache
    data, _ := json.Marshal(user)
    return cache.Set(ctx, "user:"+user.ID, data, time.Hour).Err()
}

// Cache Invalidation
func deleteUser(ctx context.Context, cache *redis.Client, db *sql.DB, id string) error {
    // 1. Delete from DB
    if err := deleteUserFromDB(db, id); err != nil {
        return err
    }
    
    // 2. Invalidate cache
    return cache.Del(ctx, "user:"+id).Err()
}

// Cache with singleflight (prevent thundering herd)
import "golang.org/x/sync/singleflight"

var group singleflight.Group

func getUserSingleflight(ctx context.Context, cache *redis.Client, db *sql.DB, id string) (*User, error) {
    // Check cache first
    if data, err := cache.Get(ctx, "user:"+id).Bytes(); err == nil {
        var user User
        json.Unmarshal(data, &user)
        return &user, nil
    }
    
    // Deduplicate concurrent requests for same key
    result, err, _ := group.Do("user:"+id, func() (interface{}, error) {
        // Double-check cache (another request might have populated it)
        if data, err := cache.Get(ctx, "user:"+id).Bytes(); err == nil {
            var user User
            json.Unmarshal(data, &user)
            return &user, nil
        }
        
        // Load from DB
        user, err := loadUserFromDB(db, id)
        if err != nil {
            return nil, err
        }
        
        // Populate cache
        data, _ := json.Marshal(user)
        cache.Set(ctx, "user:"+id, data, time.Hour)
        
        return user, nil
    })
    
    if err != nil {
        return nil, err
    }
    return result.(*User), nil
}

// TTL strategies
const (
    ShortTTL  = 5 * time.Minute   // Frequently changing data
    MediumTTL = 1 * time.Hour     // Regular data
    LongTTL   = 24 * time.Hour    // Rarely changing data
)

// Jitter to prevent cache stampede
func ttlWithJitter(base time.Duration) time.Duration {
    jitter := time.Duration(rand.Int63n(int64(base / 10)))
    return base + jitter
}
```

<a id="q12"></a>
### Q12: How do you prevent SQL injection?
**Answer:**

```go
// NEVER do this - vulnerable to SQL injection
func badQuery(db *sql.DB, name string) {
    query := "SELECT * FROM users WHERE name = '" + name + "'"
    db.Query(query)  // DANGEROUS!
}

// ALWAYS use parameterized queries
func goodQuery(db *sql.DB, name string) {
    // PostgreSQL uses $1, $2, etc.
    db.Query("SELECT * FROM users WHERE name = $1", name)
    
    // MySQL uses ?
    db.Query("SELECT * FROM users WHERE name = ?", name)
}

// Named parameters (sqlx)
func namedQuery(db *sqlx.DB, params map[string]interface{}) {
    query := "SELECT * FROM users WHERE name = :name AND age > :age"
    db.NamedQuery(query, params)
}

// GORM is safe by default
func gormQuery(db *gorm.DB, name string) {
    // Safe
    db.Where("name = ?", name).Find(&users)
    
    // Also safe
    db.Where(&User{Name: name}).Find(&users)
    
    // DANGEROUS - raw string interpolation
    db.Where(fmt.Sprintf("name = '%s'", name)).Find(&users)  // DON'T DO THIS
}

// Dynamic column names (careful!)
func dynamicColumn(db *sql.DB, column, value string) error {
    // Whitelist allowed columns
    allowedColumns := map[string]bool{
        "name": true, "email": true, "status": true,
    }
    
    if !allowedColumns[column] {
        return errors.New("invalid column")
    }
    
    // Column name can be interpolated (it's validated)
    // But value must still be parameterized
    query := fmt.Sprintf("SELECT * FROM users WHERE %s = $1", column)
    _, err := db.Query(query, value)
    return err
}

// IN clauses
func inClause(db *sqlx.DB, ids []int) {
    // Bad - building string
    // query := fmt.Sprintf("WHERE id IN (%s)", strings.Join(...))
    
    // Good - use sqlx.In
    query, args, _ := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)
    query = db.Rebind(query)
    db.Select(&users, query, args...)
}

// LIKE queries
func likeQuery(db *sql.DB, search string) {
    // Escape special characters in LIKE
    search = strings.ReplaceAll(search, "%", "\\%")
    search = strings.ReplaceAll(search, "_", "\\_")
    
    db.Query("SELECT * FROM users WHERE name LIKE $1", "%"+search+"%")
}
```

---

[← Back to Go Index](README.md)

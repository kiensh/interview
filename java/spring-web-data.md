# Spring Web & Data (JPA, JDBC, WebFlux, Security)

## Table of Contents

### Spring Data JPA
- [Q1: What is Spring Data JPA?](#q1)
- [Q2: What is the difference between JpaRepository, CrudRepository, and PagingAndSortingRepository?](#q2)

### Spring JDBC
- [Q3: What is Spring JDBC and when would you use it over JPA?](#q3)
- [Q4: How do you use JdbcTemplate?](#q4)
- [Q5: What is the difference between JdbcTemplate and NamedParameterJdbcTemplate?](#q5)

### Hibernate & ORM
- [Q6: What is ORM and how does Hibernate implement it?](#q6)
- [Q7: What is the N+1 problem and how do you solve it?](#q7)

### Reactive Programming (WebFlux)
- [Q8: What is reactive programming and Spring WebFlux?](#q8)
- [Q9: What is the difference between Mono and Flux?](#q9)
- [Q10: How do you handle backpressure in reactive streams?](#q10)
- [Q11: When should you use WebFlux vs Spring MVC?](#q11)

### Spring Security
- [Q12: What is Spring Security and how does it work?](#q12)
- [Q13: What is the difference between authentication and authorization?](#q13)
- [Q14: How do you implement JWT authentication in Spring Boot?](#q14)
- [Q15: What are the common Spring Security annotations?](#q15)
- [Q16: How do you configure CORS in Spring Security?](#q16)

---

## Spring Data JPA

<a id="q1"></a>
### Q1: What is Spring Data JPA?
**Answer:**
An abstraction layer over JPA that reduces boilerplate code by:
- Providing repository interfaces with CRUD operations
- Query derivation from method names
- Pagination and sorting support
- Custom query support via @Query

```java
public interface UserRepository extends JpaRepository<User, Long> {
    // Query derived from method name
    List<User> findByEmailAndStatus(String email, Status status);
    
    // Custom query
    @Query("SELECT u FROM User u WHERE u.createdAt > :date")
    List<User> findRecentUsers(@Param("date") LocalDate date);
}
```

<a id="q2"></a>
### Q2: What is the difference between JpaRepository, CrudRepository, and PagingAndSortingRepository?
**Answer:**
```
Repository (marker)
    └── CrudRepository (basic CRUD)
            └── PagingAndSortingRepository (+ paging/sorting)
                    └── JpaRepository (+ JPA specific: flush, batch)
```

Use `JpaRepository` for most cases as it includes all features.

---

## Spring JDBC

<a id="q3"></a>
### Q3: What is Spring JDBC and when would you use it over JPA?
**Answer:**
Spring JDBC provides a simplified abstraction over raw JDBC.

| Use JDBC | Use JPA |
|----------|---------|
| Complex native SQL queries | CRUD operations on entities |
| Bulk operations (batch inserts) | Object-relational mapping |
| Stored procedure calls | Lazy loading, caching |
| Maximum performance control | Rapid development |

<a id="q4"></a>
### Q4: How do you use JdbcTemplate?
**Answer:**
```java
@Repository
public class UserJdbcRepository {
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // Query for list
    public List<User> findAll() {
        return jdbcTemplate.query("SELECT * FROM users",
            (rs, rowNum) -> new User(rs.getLong("id"), rs.getString("name")));
    }
    
    // Query for single value
    public Integer count() {
        return jdbcTemplate.queryForObject("SELECT COUNT(*) FROM users", Integer.class);
    }
    
    // Update
    public int update(Long id, String name) {
        return jdbcTemplate.update("UPDATE users SET name = ? WHERE id = ?", name, id);
    }
    
    // Batch
    jdbcTemplate.batchUpdate("INSERT INTO users (name) VALUES (?)",
        new BatchPreparedStatementSetter() {
            public void setValues(PreparedStatement ps, int i) throws SQLException {
                ps.setString(1, users.get(i).getName());
            }
            public int getBatchSize() { return users.size(); }
        });
}
```

<a id="q5"></a>
### Q5: What is the difference between JdbcTemplate and NamedParameterJdbcTemplate?
**Answer:**
| JdbcTemplate | NamedParameterJdbcTemplate |
|--------------|---------------------------|
| Uses `?` positional placeholders | Uses named parameters `:name` |
| Order of parameters matters | Order doesn't matter |
| Harder to read with many params | More readable |

```java
// JdbcTemplate
jdbcTemplate.query("SELECT * FROM users WHERE name = ? AND age > ?",
    new Object[]{"John", 25}, rowMapper);

// NamedParameterJdbcTemplate
String sql = "SELECT * FROM users WHERE name = :name AND age > :age";
MapSqlParameterSource params = new MapSqlParameterSource()
    .addValue("name", "John")
    .addValue("age", 25);
namedTemplate.query(sql, params, rowMapper);
```

---

## Hibernate & ORM

<a id="q6"></a>
### Q6: What is ORM and how does Hibernate implement it?
**Answer:**
**ORM (Object-Relational Mapping)**: Technique to map objects to database tables.

| Concept | Java | Database |
|---------|------|----------|
| Class | Entity | Table |
| Field | Property | Column |
| Object | Instance | Row |
| Association | Reference | Foreign Key |

**JPA** is the specification, **Hibernate** is an implementation.

<a id="q7"></a>
### Q7: What is the N+1 problem and how do you solve it?
**Answer:**
N+1 problem: 1 query to fetch N entities, then N additional queries for related entities.

```java
List<Author> authors = authorRepository.findAll(); // 1 query
for (Author author : authors) {
    author.getBooks().size(); // N queries!
}
```

**Solutions:**
```java
// 1. JOIN FETCH (JPQL)
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();

// 2. @EntityGraph
@EntityGraph(attributePaths = {"books"})
List<Author> findAll();

// 3. @BatchSize (Hibernate)
@BatchSize(size = 10)
@OneToMany
private List<Book> books;
```

---

## Reactive Programming (WebFlux)

<a id="q8"></a>
### Q8: What is reactive programming and Spring WebFlux?
**Answer:**
Reactive programming focuses on asynchronous data streams with automatic backpressure handling.

**Key characteristics:**
- **Non-blocking**: Doesn't block threads waiting for I/O
- **Backpressure**: Consumer controls data flow rate
- **Event-driven**: React to data as it arrives

```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable String id) {
        return userService.findById(id);
    }
    
    @GetMapping
    public Flux<User> getAllUsers() {
        return userService.findAll();
    }
    
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<User> streamUsers() {
        return userService.findAll().delayElements(Duration.ofSeconds(1));
    }
}
```

<a id="q9"></a>
### Q9: What is the difference between Mono and Flux?
**Answer:**
| Mono<T> | Flux<T> |
|---------|---------|
| 0 or 1 element | 0 to N elements |
| Like Optional but async | Like List but async |
| Single result operations | Collection/stream operations |

```java
Mono<User> user = Mono.just(new User("Alice"));
Flux<Integer> numbers = Flux.just(1, 2, 3, 4, 5);

// Converting
Mono<List<User>> listMono = userFlux.collectList();
Flux<User> userFlux = userMono.flux();
```

<a id="q10"></a>
### Q10: How do you handle backpressure in reactive streams?
**Answer:**
| Strategy | Description | Use Case |
|----------|-------------|----------|
| `onBackpressureBuffer()` | Buffer elements | Temporary spikes |
| `onBackpressureDrop()` | Drop elements | Real-time data |
| `onBackpressureLatest()` | Keep only latest | Status updates |
| `limitRate(n)` | Request N at a time | Controlled processing |

```java
Flux.range(1, 1000)
    .onBackpressureBuffer(100)
    .subscribe(this::process);

Flux.interval(Duration.ofMillis(1))
    .onBackpressureDrop(dropped -> log.warn("Dropped: {}", dropped))
    .subscribe(this::slowProcess);
```

<a id="q11"></a>
### Q11: When should you use WebFlux vs Spring MVC?
**Answer:**
| Use WebFlux | Use Spring MVC |
|-------------|----------------|
| High concurrency, many connections | Traditional request-response |
| Streaming data | Blocking I/O (JDBC) |
| Microservices with reactive DBs | Simple CRUD applications |
| Event-driven architecture | Team familiar with imperative |

**WebFlux drawbacks:** Steeper learning curve, harder to debug, limited blocking library support.

---

## Spring Security

<a id="q12"></a>
### Q12: What is Spring Security and how does it work?
**Answer:**
Spring Security provides authentication, authorization, and protection against common attacks.

**Security Filter Chain:**
```
HTTP Request → Security Filter Chain → Controller
                     │
                     ├── SecurityContextPersistenceFilter
                     ├── UsernamePasswordAuthenticationFilter
                     ├── ExceptionTranslationFilter
                     └── FilterSecurityInterceptor
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .build();
    }
}
```

<a id="q13"></a>
### Q13: What is the difference between authentication and authorization?
**Answer:**
| Authentication | Authorization |
|----------------|---------------|
| Who are you? | What can you do? |
| Verify identity | Verify permissions |
| Login process | Access control |
| 401 Unauthorized | 403 Forbidden |

<a id="q14"></a>
### Q14: How do you implement JWT authentication in Spring Boot?
**Answer:**
```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String authHeader = request.getHeader("Authorization");
        
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }
        
        String jwt = authHeader.substring(7);
        String username = jwtService.extractUsername(jwt);
        
        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            if (jwtService.isTokenValid(jwt, userDetails)) {
                UsernamePasswordAuthenticationToken authToken = 
                    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        filterChain.doFilter(request, response);
    }
}

// Security config
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/auth/**").permitAll()
            .anyRequest().authenticated())
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
        .build();
}
```

<a id="q15"></a>
### Q15: What are the common Spring Security annotations?
**Answer:**
| Annotation | Description |
|------------|-------------|
| `@EnableWebSecurity` | Enable Spring Security |
| `@EnableMethodSecurity` | Enable method-level security |
| `@PreAuthorize` | Check before method execution |
| `@PostAuthorize` | Check after method execution |
| `@Secured` | Simple role-based check |

```java
@PreAuthorize("hasRole('ADMIN')")
public void adminOnly() { }

@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
public void adminOrManager() { }

@PreAuthorize("#userId == authentication.principal.id")
public User getUser(@PathVariable Long userId) { }

@PreAuthorize("@securityService.canAccess(#id)")
public Resource getResource(@PathVariable Long id) { }
```

<a id="q16"></a>
### Q16: How do you configure CORS in Spring Security?
**Answer:**
```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .csrf(csrf -> csrf.disable())
        .build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(Arrays.asList("http://localhost:3000"));
    config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(Arrays.asList("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

**Controller-level:**
```java
@RestController
@CrossOrigin(origins = "http://localhost:3000")
public class ApiController { }
```

---

[← Back to Java Index](README.md)

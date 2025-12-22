# Spring Framework Interview Questions & Answers

---

## Table of Contents

### [Dependency Injection](#dependency-injection)
- [Q36: What is Dependency Injection (DI)?](#q36)
- [Q37: Why is Constructor Injection preferred over Field Injection?](#q37)

### [Spring Annotations](#spring-annotations)
- [Q38: What is the difference between @Component, @Service, @Repository, and @Controller?](#q38)
- [Q39: What is the difference between @Autowired, @Inject, and @Resource?](#q39)
- [Q40: What does @Qualifier do?](#q40)

### [Spring Boot](#spring-boot)
- [Q41: What are the advantages of Spring Boot?](#q41)
- [Q42: What is Spring Boot Auto-configuration?](#q42)
- [Q43: What is the difference between application.properties and application.yml?](#q43)

### [@Transactional Annotation](#transactional-annotation)
- [Q44: How does @Transactional work?](#q44)
- [Q45: What are Transaction Propagation levels?](#q45)
- [Q46: Why might @Transactional not work?](#q46)

### [Spring Data JPA](#spring-data-jpa)
- [Q47: What is Spring Data JPA?](#q47)
- [Q48: What is the difference between JpaRepository, CrudRepository, and PagingAndSortingRepository?](#q48)

### [Hibernate vs ORM](#hibernate-vs-orm)
- [Q49: What is ORM and how does Hibernate implement it?](#q49)
- [Q50: What is the N+1 problem and how do you solve it?](#q50)

### [Bean Lifecycle](#bean-lifecycle)
- [Q51: What is a Spring Bean?](#q51)
- [Q52: What are the Bean scopes in Spring?](#q52)
- [Q53: What is the Bean lifecycle?](#q53)

### [IoC (Inversion of Control)](#ioc-inversion-of-control)
- [Q54: What is IoC and how does Spring implement it?](#q54)

### [AOP (Aspect-Oriented Programming)](#aop-aspect-oriented-programming)
- [Q55: What is AOP and what are its key concepts?](#q55)
- [Q56: What are the types of Advice in Spring AOP?](#q56)

### [SOLID, DRY, DDD](#solid-dry-ddd)
- [Q57: What are the SOLID principles?](#q57)
- [Q58: What is DDD (Domain-Driven Design)?](#q58)

### [Testing with Spring](#testing-with-spring)
- [Q59: What is the difference between @WebMvcTest and @SpringBootTest?](#q59)
- [Q60: How do you test a REST controller with MockMvc?](#q60)
- [Q61: What is @DataJpaTest used for?](#q61)

---

## Dependency Injection

<a id="q36"></a>
### Q36: What is Dependency Injection (DI)?
**Answer:**
DI is a design pattern where objects receive their dependencies from external sources rather than creating them internally.

**Benefits:**
- Loose coupling between classes
- Easier unit testing (can inject mocks)
- Better code reusability
- Follows Single Responsibility Principle

**Types in Spring:**
1. **Constructor Injection** (recommended)
2. **Setter Injection**
3. **Field Injection** (not recommended)

```java
// Constructor Injection (preferred)
@Service
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

<a id="q37"></a>
### Q37: Why is Constructor Injection preferred over Field Injection?
**Answer:**
1. **Immutability**: Dependencies can be final
2. **Required dependencies are explicit**: Won't compile without them
3. **Easier testing**: No reflection needed to inject mocks
4. **Prevents circular dependencies**: Fails fast at startup
5. **Framework independence**: Works without Spring context

---

## Spring Annotations

<a id="q38"></a>
### Q38: What is the difference between @Component, @Service, @Repository, and @Controller?
**Answer:**
All are specializations of `@Component`:

| Annotation | Layer | Special Features |
|------------|-------|------------------|
| @Component | Generic | Base annotation for Spring-managed beans |
| @Service | Service/Business | Semantic marker for business logic |
| @Repository | Data Access | Exception translation to DataAccessException |
| @Controller | Web | Used with @RequestMapping for MVC |
| @RestController | Web | @Controller + @ResponseBody combined |

<a id="q39"></a>
### Q39: What is the difference between @Autowired, @Inject, and @Resource?
**Answer:**
| @Autowired | @Inject | @Resource |
|------------|---------|-----------|
| Spring annotation | JSR-330 (Java standard) | JSR-250 (Java standard) |
| By type, then by name | By type, then by name | By name, then by type |
| Has `required` attribute | No required attribute | Has `name` attribute |

<a id="q40"></a>
### Q40: What does @Qualifier do?
**Answer:**
Used when multiple beans of the same type exist to specify which one to inject:

```java
@Component("emailNotifier")
public class EmailNotifier implements Notifier { }

@Component("smsNotifier")
public class SmsNotifier implements Notifier { }

@Service
public class NotificationService {
    @Autowired
    @Qualifier("emailNotifier")
    private Notifier notifier;
}
```

---

## Spring Boot

<a id="q41"></a>
### Q41: What are the advantages of Spring Boot?
**Answer:**
1. **Auto-configuration**: Automatically configures based on classpath
2. **Starter dependencies**: Simplified dependency management
3. **Embedded servers**: No need to deploy WAR files
4. **Production-ready features**: Health checks, metrics, externalized config
5. **No XML configuration**: Java-based configuration
6. **Opinionated defaults**: Convention over configuration

<a id="q42"></a>
### Q42: What is Spring Boot Auto-configuration?
**Answer:**
Auto-configuration automatically configures Spring beans based on:
- Classes present on classpath
- Existing bean definitions
- Property values

```java
// Conditional annotations used internally
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.url")
```

Disable specific auto-configuration:
```java
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```

<a id="q43"></a>
### Q43: What is the difference between application.properties and application.yml?
**Answer:**
Both serve the same purpose, different formats:

```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost/db
```

```yaml
# application.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://localhost/db
```

YAML advantages: hierarchical structure, less repetition, supports lists naturally.

---

## @Transactional Annotation

<a id="q44"></a>
### Q44: How does @Transactional work?
**Answer:**
Spring creates a proxy around the annotated method that:
1. Opens a transaction before method execution
2. Commits if method completes successfully
3. Rolls back if RuntimeException is thrown

```java
@Transactional
public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
    accountRepository.debit(fromId, amount);
    accountRepository.credit(toId, amount);
    // Both operations in same transaction
}
```

**Key attributes:**
- `propagation`: How transaction relates to existing transaction
- `isolation`: Database isolation level
- `rollbackFor`: Exceptions that trigger rollback
- `readOnly`: Optimization hint for read-only transactions

<a id="q45"></a>
### Q45: What are Transaction Propagation levels?
**Answer:**
| Propagation | Behavior |
|-------------|----------|
| REQUIRED (default) | Join existing or create new |
| REQUIRES_NEW | Always create new, suspend existing |
| SUPPORTS | Join existing, or run without if none |
| NOT_SUPPORTED | Run without transaction, suspend existing |
| MANDATORY | Must have existing, else exception |
| NEVER | Must not have existing, else exception |
| NESTED | Nested transaction with savepoint |

<a id="q46"></a>
### Q46: Why might @Transactional not work?
**Answer:**
1. **Self-invocation**: Calling @Transactional method from same class (proxy bypass)
2. **Not public**: Spring proxies only work with public methods
3. **Wrong exception**: By default, only unchecked exceptions roll back
4. **Proxy mode**: JDK proxy requires interface, CGLIB for classes

```java
// Won't work - self-invocation
public void methodA() {
    methodB(); // Direct call, bypasses proxy
}

@Transactional
public void methodB() { }
```

---

## Spring Data JPA

<a id="q47"></a>
### Q47: What is Spring Data JPA?
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

<a id="q48"></a>
### Q48: What is the difference between JpaRepository, CrudRepository, and PagingAndSortingRepository?
**Answer:**
```
Repository (marker)
    └── CrudRepository (basic CRUD)
            └── PagingAndSortingRepository (+ paging/sorting)
                    └── JpaRepository (+ JPA specific: flush, batch)
```

Use `JpaRepository` for most cases as it includes all features.

---

## Hibernate vs ORM

<a id="q49"></a>
### Q49: What is ORM and how does Hibernate implement it?
**Answer:**
**ORM (Object-Relational Mapping)**: Technique to map objects to database tables.

**JPA**: Java specification/standard for ORM.

**Hibernate**: Implementation of JPA specification.

| Concept | Java | Database |
|---------|------|----------|
| Class | Entity | Table |
| Field | Property | Column |
| Object | Instance | Row |
| Association | Reference | Foreign Key |

<a id="q50"></a>
### Q50: What is the N+1 problem and how do you solve it?
**Answer:**
N+1 problem: 1 query to fetch N entities, then N additional queries to fetch related entities.

```java
// N+1 problem
List<Author> authors = authorRepository.findAll(); // 1 query
for (Author author : authors) {
    author.getBooks().size(); // N queries
}
```

**Solutions:**
1. **JOIN FETCH** (JPQL):
```java
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();
```

2. **@EntityGraph**:
```java
@EntityGraph(attributePaths = {"books"})
List<Author> findAll();
```

3. **@BatchSize** (Hibernate):
```java
@BatchSize(size = 10)
@OneToMany
private List<Book> books;
```

---

## Bean Lifecycle

<a id="q51"></a>
### Q51: What is a Spring Bean?
**Answer:**
A Bean is an object managed by the Spring IoC container. Spring handles:
- Instantiation
- Dependency injection
- Lifecycle management
- Destruction

<a id="q52"></a>
### Q52: What are the Bean scopes in Spring?
**Answer:**
| Scope | Description |
|-------|-------------|
| singleton (default) | One instance per Spring container |
| prototype | New instance each time requested |
| request | One instance per HTTP request |
| session | One instance per HTTP session |
| application | One instance per ServletContext |
| websocket | One instance per WebSocket session |

<a id="q53"></a>
### Q53: What is the Bean lifecycle?
**Answer:**
1. Instantiation
2. Populate properties (DI)
3. BeanNameAware.setBeanName()
4. BeanFactoryAware.setBeanFactory()
5. ApplicationContextAware.setApplicationContext()
6. BeanPostProcessor.postProcessBeforeInitialization()
7. @PostConstruct
8. InitializingBean.afterPropertiesSet()
9. Custom init-method
10. BeanPostProcessor.postProcessAfterInitialization()
11. Bean ready to use
12. @PreDestroy
13. DisposableBean.destroy()
14. Custom destroy-method

---

## IoC (Inversion of Control)

<a id="q54"></a>
### Q54: What is IoC and how does Spring implement it?
**Answer:**
**IoC (Inversion of Control)**: Design principle where control of object creation and lifecycle is transferred from application code to a container/framework.

**Traditional approach:**
```java
public class UserService {
    private UserRepository repo = new UserRepositoryImpl(); // Tight coupling
}
```

**IoC approach:**
```java
public class UserService {
    private UserRepository repo; // Injected by container
    
    public UserService(UserRepository repo) {
        this.repo = repo;
    }
}
```

**Spring's IoC Container** (ApplicationContext) manages:
- Bean instantiation
- Bean configuration
- Bean assembly (wiring dependencies)

---

## AOP (Aspect-Oriented Programming)

<a id="q55"></a>
### Q55: What is AOP and what are its key concepts?
**Answer:**
AOP allows separation of cross-cutting concerns (logging, security, transactions) from business logic.

**Key concepts:**
- **Aspect**: Module containing cross-cutting concern code
- **Join Point**: Point in execution (method call, exception)
- **Pointcut**: Expression that matches join points
- **Advice**: Action taken at join point (before, after, around)
- **Weaving**: Linking aspects with target objects

```java
@Aspect
@Component
public class LoggingAspect {
    
    @Around("execution(* com.example.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        long duration = System.currentTimeMillis() - start;
        log.info("{} executed in {}ms", joinPoint.getSignature(), duration);
        return result;
    }
}
```

<a id="q56"></a>
### Q56: What are the types of Advice in Spring AOP?
**Answer:**
| Advice | Execution |
|--------|-----------|
| @Before | Before method execution |
| @After | After method (regardless of outcome) |
| @AfterReturning | After successful return |
| @AfterThrowing | After exception thrown |
| @Around | Wraps method, full control |

---

## SOLID, DRY, DDD

<a id="q57"></a>
### Q57: What are the SOLID principles?
**Answer:**
1. **S - Single Responsibility**: Class should have only one reason to change
2. **O - Open/Closed**: Open for extension, closed for modification
3. **L - Liskov Substitution**: Subtypes must be substitutable for base types
4. **I - Interface Segregation**: Many specific interfaces > one general interface
5. **D - Dependency Inversion**: Depend on abstractions, not concretions

<a id="q58"></a>
### Q58: What is DDD (Domain-Driven Design)?
**Answer:**
DDD is an approach to software development that focuses on the business domain.

**Key concepts:**
- **Entity**: Object with identity (e.g., User with ID)
- **Value Object**: Immutable object without identity (e.g., Address)
- **Aggregate**: Cluster of entities treated as unit
- **Aggregate Root**: Entry point to aggregate
- **Repository**: Abstraction for data access
- **Domain Service**: Business logic not belonging to entity
- **Bounded Context**: Explicit boundary within which domain model exists

---

## Testing with Spring

<a id="q59"></a>
### Q59: What is the difference between @WebMvcTest and @SpringBootTest?
**Answer:**
| @WebMvcTest | @SpringBootTest |
|-------------|-----------------|
| Loads only web layer | Loads full application context |
| Fast, focused | Slower, comprehensive |
| Auto-configures MockMvc | Need to configure MockMvc |
| Tests controllers only | Integration tests |
| Dependencies must be mocked | Can use real dependencies |

<a id="q60"></a>
### Q60: How do you test a REST controller with MockMvc?
**Answer:**
```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void shouldReturnUser() throws Exception {
        when(userService.findById(1L)).thenReturn(new User(1L, "John"));
        
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("John"));
    }
}
```

<a id="q61"></a>
### Q61: What is @DataJpaTest used for?
**Answer:**
- Tests JPA repositories
- Configures in-memory database
- Scans for @Entity classes
- Configures Spring Data JPA repositories
- Transactions are rolled back after each test

```java
@DataJpaTest
class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void shouldFindByEmail() {
        userRepository.save(new User("test@email.com"));
        
        Optional<User> found = userRepository.findByEmail("test@email.com");
        
        assertThat(found).isPresent();
    }
}
```

---

Good luck with your interview! 🚀

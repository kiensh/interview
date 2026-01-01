# Spring Core (DI, IoC, Boot, AOP, Transactions)

## Table of Contents

### Dependency Injection & IoC
- [Q1: What is Dependency Injection (DI)?](#q1)
- [Q2: Why is Constructor Injection preferred over Field Injection?](#q2)
- [Q3: What is IoC and how does Spring implement it?](#q3)

### Spring Annotations
- [Q4: What is the difference between @Component, @Service, @Repository, and @Controller?](#q4)
- [Q5: What is the difference between @Autowired, @Inject, and @Resource?](#q5)
- [Q6: What does @Qualifier do?](#q6)

### Spring Boot
- [Q7: What are the advantages of Spring Boot?](#q7)
- [Q8: What is Spring Boot Auto-configuration?](#q8)
- [Q9: What is the difference between application.properties and application.yml?](#q9)

### Bean Lifecycle
- [Q10: What is a Spring Bean?](#q10)
- [Q11: What are the Bean scopes in Spring?](#q11)
- [Q12: What is the Bean lifecycle?](#q12)
- [Q13: What are the different ways to define a bean in Spring?](#q13)

### AOP (Aspect-Oriented Programming)
- [Q14: What is AOP and what are its key concepts?](#q14)
- [Q15: What are the types of Advice in Spring AOP?](#q15)
- [Q16: What is the difference between Spring AOP and AspectJ?](#q16)
- [Q17: How do you write a pointcut expression?](#q17)

### Transactions
- [Q18: How does @Transactional work?](#q18)
- [Q19: What are Transaction Propagation levels?](#q19)
- [Q20: Why might @Transactional not work?](#q20)

### Testing
- [Q21: What is the difference between @WebMvcTest and @SpringBootTest?](#q21)
- [Q22: How do you test a REST controller with MockMvc?](#q22)
- [Q23: What is @DataJpaTest used for?](#q23)

---

## Dependency Injection & IoC

<a id="q1"></a>
### Q1: What is Dependency Injection (DI)?
**Answer:**
DI is a design pattern where objects receive their dependencies from external sources rather than creating them internally.

**Benefits:**
- Loose coupling between classes
- Easier unit testing (can inject mocks)
- Better code reusability
- Follows Single Responsibility Principle

**Types in Spring:**
```java
// 1. Constructor Injection (preferred)
@Service
public class UserService {
    private final UserRepository userRepository;
    
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}

// 2. Setter Injection
@Autowired
public void setUserRepository(UserRepository repo) { this.repo = repo; }

// 3. Field Injection (not recommended)
@Autowired
private UserRepository userRepository;
```

<a id="q2"></a>
### Q2: Why is Constructor Injection preferred over Field Injection?
**Answer:**
1. **Immutability**: Dependencies can be `final`
2. **Required dependencies are explicit**: Won't compile without them
3. **Easier testing**: No reflection needed to inject mocks
4. **Prevents circular dependencies**: Fails fast at startup
5. **Framework independence**: Works without Spring context

```java
@Service
@RequiredArgsConstructor  // Lombok generates constructor
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;
}
```

<a id="q3"></a>
### Q3: What is IoC and how does Spring implement it?
**Answer:**
**IoC (Inversion of Control)**: Design principle where control of object creation is transferred from application code to a container.

```java
// Traditional approach (tight coupling)
public class UserService {
    private UserRepository repo = new UserRepositoryImpl();
}

// IoC approach (loose coupling)
public class UserService {
    private UserRepository repo;  // Injected by container
    public UserService(UserRepository repo) { this.repo = repo; }
}
```

**Spring's IoC Container** (ApplicationContext) manages bean instantiation, configuration, and assembly.

---

## Spring Annotations

<a id="q4"></a>
### Q4: What is the difference between @Component, @Service, @Repository, and @Controller?
**Answer:**
All are specializations of `@Component`:

| Annotation | Layer | Special Features |
|------------|-------|------------------|
| @Component | Generic | Base annotation for Spring-managed beans |
| @Service | Service/Business | Semantic marker for business logic |
| @Repository | Data Access | Exception translation to DataAccessException |
| @Controller | Web | Used with @RequestMapping for MVC |
| @RestController | Web | @Controller + @ResponseBody combined |

<a id="q5"></a>
### Q5: What is the difference between @Autowired, @Inject, and @Resource?
**Answer:**
| @Autowired | @Inject | @Resource |
|------------|---------|-----------|
| Spring annotation | JSR-330 (Java standard) | JSR-250 (Java standard) |
| By type, then by name | By type, then by name | By name, then by type |
| Has `required` attribute | No required attribute | Has `name` attribute |

<a id="q6"></a>
### Q6: What does @Qualifier do?
**Answer:**
Used when multiple beans of the same type exist:

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

<a id="q7"></a>
### Q7: What are the advantages of Spring Boot?
**Answer:**
1. **Auto-configuration**: Automatically configures based on classpath
2. **Starter dependencies**: Simplified dependency management
3. **Embedded servers**: No need to deploy WAR files
4. **Production-ready features**: Health checks, metrics, externalized config
5. **No XML configuration**: Java-based configuration
6. **Opinionated defaults**: Convention over configuration

<a id="q8"></a>
### Q8: What is Spring Boot Auto-configuration?
**Answer:**
Auto-configuration automatically configures Spring beans based on classpath, existing beans, and properties.

```java
// Conditional annotations used internally
@ConditionalOnClass(DataSource.class)
@ConditionalOnMissingBean(DataSource.class)
@ConditionalOnProperty(name = "spring.datasource.url")

// Disable specific auto-configuration
@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
```

<a id="q9"></a>
### Q9: What is the difference between application.properties and application.yml?
**Answer:**
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

## Bean Lifecycle

<a id="q10"></a>
### Q10: What is a Spring Bean?
**Answer:**
A Bean is an object managed by the Spring IoC container. Spring handles instantiation, dependency injection, lifecycle management, and destruction.

<a id="q11"></a>
### Q11: What are the Bean scopes in Spring?
**Answer:**
| Scope | Description |
|-------|-------------|
| singleton (default) | One instance per Spring container |
| prototype | New instance each time requested |
| request | One instance per HTTP request |
| session | One instance per HTTP session |
| application | One instance per ServletContext |

<a id="q12"></a>
### Q12: What is the Bean lifecycle?
**Answer:**
1. Instantiation
2. Populate properties (DI)
3. BeanNameAware, BeanFactoryAware, ApplicationContextAware
4. BeanPostProcessor.postProcessBeforeInitialization()
5. @PostConstruct
6. InitializingBean.afterPropertiesSet()
7. Custom init-method
8. BeanPostProcessor.postProcessAfterInitialization()
9. Bean ready to use
10. @PreDestroy
11. DisposableBean.destroy()
12. Custom destroy-method

<a id="q13"></a>
### Q13: What are the different ways to define a bean in Spring?
**Answer:**
```java
// 1. Component Scanning
@Component  // or @Service, @Repository, @Controller
public class UserService { }

// 2. @Bean in @Configuration class
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserService(userRepository());
    }
}

// 3. @Import for modular configuration
@Configuration
@Import({DatabaseConfig.class, SecurityConfig.class})
public class AppConfig { }
```

---

## AOP (Aspect-Oriented Programming)

<a id="q14"></a>
### Q14: What is AOP and what are its key concepts?
**Answer:**
AOP allows separation of cross-cutting concerns (logging, security, transactions) from business logic.

**Key concepts:**
- **Aspect**: Module containing cross-cutting concern code
- **Join Point**: Point in execution (method call, exception)
- **Pointcut**: Expression that matches join points
- **Advice**: Action taken at join point
- **Weaving**: Linking aspects with target objects

```java
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object logExecutionTime(ProceedingJoinPoint joinPoint) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        log.info("{} executed in {}ms", joinPoint.getSignature(), 
                 System.currentTimeMillis() - start);
        return result;
    }
}
```

<a id="q15"></a>
### Q15: What are the types of Advice in Spring AOP?
**Answer:**
| Advice | Execution |
|--------|-----------|
| @Before | Before method execution |
| @After | After method (regardless of outcome) |
| @AfterReturning | After successful return |
| @AfterThrowing | After exception thrown |
| @Around | Wraps method, full control |

<a id="q16"></a>
### Q16: What is the difference between Spring AOP and AspectJ?
**Answer:**
| Feature | Spring AOP | AspectJ |
|---------|------------|---------|
| Weaving | Runtime (proxy-based) | Compile-time or load-time |
| Join points | Method execution only | Methods, fields, constructors |
| Scope | Spring beans only | Any Java class |
| Self-invocation | Not intercepted | Works correctly |

```java
// Self-invocation problem in Spring AOP
@Service
public class OrderService {
    @Transactional
    public void createOrder() {
        processPayment();  // DOES NOT trigger @Transactional (bypasses proxy)
    }
    
    @Transactional
    public void processPayment() { }
}
```

<a id="q17"></a>
### Q17: How do you write a pointcut expression?
**Answer:**
```java
// Any method in service package
@Before("execution(* com.example.service.*.*(..))")

// Methods with @Transactional annotation
@Before("@annotation(org.springframework.transaction.annotation.Transactional)")

// All methods in classes annotated with @Service
@Before("@within(org.springframework.stereotype.Service)")

// Specific bean
@Before("bean(userService)")

// Combining pointcuts
@Pointcut("execution(* com.example.service.*.*(..))")
public void serviceLayer() { }

@Before("serviceLayer() && args(userId, ..)")
public void beforeServiceWithUserId(Long userId) { }
```

---

## Transactions

<a id="q18"></a>
### Q18: How does @Transactional work?
**Answer:**
Spring creates a proxy that:
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

**Key attributes:** `propagation`, `isolation`, `rollbackFor`, `readOnly`

<a id="q19"></a>
### Q19: What are Transaction Propagation levels?
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

<a id="q20"></a>
### Q20: Why might @Transactional not work?
**Answer:**
1. **Self-invocation**: Calling @Transactional method from same class (proxy bypass)
2. **Not public**: Spring proxies only work with public methods
3. **Wrong exception**: By default, only unchecked exceptions roll back
4. **Proxy mode**: JDK proxy requires interface, CGLIB for classes

---

## Testing

<a id="q21"></a>
### Q21: What is the difference between @WebMvcTest and @SpringBootTest?
**Answer:**
| @WebMvcTest | @SpringBootTest |
|-------------|-----------------|
| Loads only web layer | Loads full application context |
| Fast, focused | Slower, comprehensive |
| Auto-configures MockMvc | Need to configure MockMvc |
| Tests controllers only | Integration tests |
| Dependencies must be mocked | Can use real dependencies |

<a id="q22"></a>
### Q22: How do you test a REST controller with MockMvc?
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

<a id="q23"></a>
### Q23: What is @DataJpaTest used for?
**Answer:**
- Tests JPA repositories
- Configures in-memory database
- Scans for @Entity classes
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

[← Back to Java Index](README.md)

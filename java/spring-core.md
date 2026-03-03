# Spring Core (DI, IoC, Boot, Bean, AOP, Transactions)

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
- [Q18: Why might @Transactional not work?](#q18)

---

## Dependency Injection & IoC

<a id="q1"></a>
### Q1: What is Dependency Injection (DI)?
**Answer:**
Dependency Injection means an object receives its dependencies from outside instead of constructing them internally.

```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```

**Why DI matters in real systems:**
- Reduces coupling between components.
- Improves testability (inject mocks/fakes).
- Makes wiring explicit and environment-specific (prod vs test).
- Supports clean layering and easier refactoring.

<a id="q2"></a>
### Q2: Why is Constructor Injection preferred over Field Injection?
**Answer:**
Constructor injection is preferred because dependencies become explicit and immutable.

| Constructor Injection | Field Injection |
|----------------------|-----------------|
| Supports `final` fields | Mutable dependencies |
| Fails fast on missing required deps | Missing dependency may fail later |
| Easy plain unit testing | Usually needs reflection/container |
| Clear API contract | Hidden required dependencies |

```java
@Service
public class OrderService {
    private final PaymentClient paymentClient;
    private final OrderRepository orderRepository;

    public OrderService(PaymentClient paymentClient, OrderRepository orderRepository) {
        this.paymentClient = paymentClient;
        this.orderRepository = orderRepository;
    }
}
```

**Practical tip:** Use constructor injection for required dependencies; optional deps can use setter/object provider.

<a id="q3"></a>
### Q3: What is IoC and how does Spring implement it?
**Answer:**
**IoC (Inversion of Control)** means object creation/lifecycle moves from application code to a container.

Spring implements IoC through `BeanFactory`/`ApplicationContext`, which handles:
1. Bean creation
2. Dependency resolution and injection
3. Lifecycle callbacks
4. Proxy wrapping (AOP, transactions, security)

```java
AnnotationConfigApplicationContext context =
    new AnnotationConfigApplicationContext(AppConfig.class);
UserService service = context.getBean(UserService.class);
```

**Interview angle:** IoC is the principle; DI is one concrete mechanism.

---

## Spring Annotations

<a id="q4"></a>
### Q4: What is the difference between @Component, @Service, @Repository, and @Controller?
**Answer:**
All are stereotype annotations discovered by component scanning, but each signals intent.

| Annotation | Typical layer | Extra behavior |
|------------|---------------|----------------|
| `@Component` | Generic | Base stereotype |
| `@Service` | Business/service | Semantic marker for service logic |
| `@Repository` | Persistence | Exception translation into `DataAccessException` |
| `@Controller` | MVC web | Works with request mapping/view resolution |
| `@RestController` | REST web | `@Controller` + `@ResponseBody` |

Use the most specific stereotype so code communicates architecture clearly.

<a id="q5"></a>
### Q5: What is the difference between @Autowired, @Inject, and @Resource?
**Answer:**
| Annotation | Standard | Primary resolution strategy | Notes |
|------------|----------|-----------------------------|-------|
| `@Autowired` | Spring | by type (then qualifiers/name) | Supports `required=false` |
| `@Inject` | JSR-330 | by type | Portable, no `required` flag |
| `@Resource` | JSR-250 | by name (then type) | Useful for name-based injection |

```java
@Autowired
@Qualifier("emailNotifier")
private Notifier notifier;
```

In Spring-heavy projects, `@Autowired` is most common; use standards if portability is required.

<a id="q6"></a>
### Q6: What does @Qualifier do?
**Answer:**
`@Qualifier` disambiguates beans when multiple candidates share the same type.

```java
@Component("emailNotifier")
class EmailNotifier implements Notifier {}

@Component("smsNotifier")
class SmsNotifier implements Notifier {}

@Service
class NotificationService {
    private final Notifier notifier;

    NotificationService(@Qualifier("emailNotifier") Notifier notifier) {
        this.notifier = notifier;
    }
}
```

**Alternative:** Mark one implementation as `@Primary` and use `@Qualifier` only for explicit overrides.

---

## Spring Boot

<a id="q7"></a>
### Q7: What are the advantages of Spring Boot?
**Answer:**
Spring Boot optimizes developer productivity while keeping Spring ecosystem power.

Key advantages:
1. Auto-configuration based on classpath + properties.
2. Starter dependencies reduce version-management friction.
3. Embedded server support (jar deployment).
4. Externalized configuration by profile/environment.
5. Production features via Actuator (health, metrics, info).
6. Convention-over-configuration for faster onboarding.

**Trade-off:** Defaults are productive, but teams must still understand what Boot auto-configured under the hood.

<a id="q8"></a>
### Q8: What is Spring Boot Auto-configuration?
**Answer:**
Auto-configuration conditionally creates beans using rules such as:
- `@ConditionalOnClass`
- `@ConditionalOnMissingBean`
- `@ConditionalOnProperty`

```java
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class App {}
```

**How to debug it:** use `--debug` or inspect condition evaluation report to see why a bean was or was not created.

**Best practice:** Prefer overriding with your own bean rather than blindly excluding large auto-config modules.

<a id="q9"></a>
### Q9: What is the difference between application.properties and application.yml?
**Answer:**
Both define externalized config; they are functionally equivalent in Spring Boot.

```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/app
```

```yaml
# application.yml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/app
```

| `properties` | `yml` |
|--------------|-------|
| Verbose for nested values | Cleaner for nested structures/lists |
| Familiar Java style | More concise but indentation-sensitive |

Choose one style consistently in a codebase.

---

## Bean Lifecycle

<a id="q10"></a>
### Q10: What is a Spring Bean?
**Answer:**
A Spring Bean is an object managed by the IoC container, including creation, wiring, lifecycle, and optional proxying.

Not every Java object is a bean; only objects registered in context (`@Component`, `@Bean`, XML, etc.) are container-managed.

<a id="q11"></a>
### Q11: What are the Bean scopes in Spring?
**Answer:**
| Scope | Meaning | Notes |
|-------|---------|-------|
| `singleton` (default) | One instance per container | Shared across requests; must be thread-safe |
| `prototype` | New instance per lookup | Container does not fully manage destroy phase |
| `request` | One per HTTP request | Web context only |
| `session` | One per HTTP session | Web context only |
| `application` | One per servlet context | Web context only |
| `websocket` | One per WebSocket session | Web context only |

**Pitfall:** Injecting `prototype` into `singleton` directly yields one instance at construction time unless using providers/object factories.

<a id="q12"></a>
### Q12: What is the Bean lifecycle?
**Answer:**
Typical lifecycle (simplified):
1. Bean definition registered.
2. Bean instantiated.
3. Dependencies injected.
4. `BeanNameAware` / `BeanFactoryAware` callbacks.
5. `BeanPostProcessor#postProcessBeforeInitialization`.
6. Init callbacks (`@PostConstruct`, `InitializingBean`, custom init method).
7. `BeanPostProcessor#postProcessAfterInitialization` (proxy may be returned here).
8. Bean ready for use.
9. On shutdown: `@PreDestroy`, `DisposableBean`, custom destroy method.

```java
@Component
class Client {
    @PostConstruct
    void init() {}

    @PreDestroy
    void cleanup() {}
}
```

<a id="q13"></a>
### Q13: What are the different ways to define a bean in Spring?
**Answer:**
1. **Component scanning**
```java
@Service
class UserService {}
```

2. **Java config with `@Bean`**
```java
@Configuration
class AppConfig {
    @Bean
    UserService userService(UserRepository repo) {
        return new UserService(repo);
    }
}
```

3. **Import modular configs**
```java
@Configuration
@Import({DbConfig.class, SecurityConfig.class})
class AppConfig {}
```

4. **Functional/Programmatic registration** (advanced bootstrap cases).

---

## AOP (Aspect-Oriented Programming)

<a id="q14"></a>
### Q14: What is AOP and what are its key concepts?
**Answer:**
AOP separates cross-cutting concerns (logging, security, transactions, tracing) from core business logic.

| Concept | Meaning |
|---------|---------|
| Aspect | Module containing cross-cutting logic |
| Join point | Interceptable execution point (Spring AOP: method execution) |
| Pointcut | Expression selecting join points |
| Advice | Action around join points (`@Before`, `@Around`, etc.) |
| Weaving | Applying aspect logic to target |

```java
@Aspect
@Component
class TimingAspect {
    @Around("execution(* com.example.service..*(..))")
    public Object time(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long tookMs = (System.nanoTime() - start) / 1_000_000;
            System.out.println(pjp.getSignature() + " took " + tookMs + "ms");
        }
    }
}
```

<a id="q15"></a>
### Q15: What are the types of Advice in Spring AOP?
**Answer:**
| Advice | Runs when |
|--------|-----------|
| `@Before` | before target method |
| `@After` | after target method (success or exception) |
| `@AfterReturning` | after successful return |
| `@AfterThrowing` | after exception |
| `@Around` | wraps call and controls invocation |

**Guideline:** Use `@Around` only when you need full control (timing/retry/short-circuit); otherwise prefer narrower advice.

<a id="q16"></a>
### Q16: What is the difference between Spring AOP and AspectJ?
**Answer:**
| Feature | Spring AOP | AspectJ |
|---------|------------|---------|
| Weaving | Runtime proxies | Compile-time or load-time weaving |
| Join points | Method execution on Spring beans | Methods, constructors, fields, more |
| Scope | Container-managed beans only | Any class |
| Self-invocation | Not intercepted by proxy | Can be intercepted |
| Complexity | Simpler operationally | More power, more setup |

Use Spring AOP for most enterprise cross-cutting concerns; choose AspectJ when you need non-proxy join points.

<a id="q17"></a>
### Q17: How do you write a pointcut expression?
**Answer:**
Common forms:

```java
// Any method in service package and subpackages
@Before("execution(* com.example.service..*(..))")
public void beforeServiceMethods() {}

// Methods annotated with @Transactional
@Before("@annotation(org.springframework.transaction.annotation.Transactional)")
public void beforeTransactionalMethods() {}

// Beans with @Service stereotype
@Before("@within(org.springframework.stereotype.Service)")
public void beforeServiceBeans() {}

// Composition
@Pointcut("execution(* com.example.service..*(..))")
private void serviceLayer() {}

@Before("serviceLayer() && args(userId, ..)")
public void audit(Long userId) {}
```

**Best practice:** Keep pointcuts centralized and named; avoid fragile copy-paste strings across aspects.

---

## Transactions

<a id="q18"></a>
### Q18: Why might @Transactional not work?
**Answer:**
Common causes:
1. **Self-invocation**: method in same class bypasses proxy.
2. **Wrong visibility/finality**: proxy cannot intercept as expected (`private`, `final`, etc., depending on proxy type).
3. **Called outside Spring context**: object manually created with `new`.
4. **Wrong exception expectations**: checked exceptions do not roll back by default.
5. **Async boundary**: transaction context does not automatically flow to new thread.
6. **Misconfigured transaction manager**: incorrect datasource/manager wiring.

```java
@Service
class OrderService {
    @Transactional
    public void createOrder() {
        validate(); // self-invocation issue if validate is expected transactional separately
    }

    @Transactional
    public void validate() {}
}
```

**Fix patterns:** split methods into separate beans, ensure proxy-managed calls, and set `rollbackFor` intentionally when needed.

---

[← Back to Java Index](README.md)

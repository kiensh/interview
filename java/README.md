# Java & Spring Interview Questions

A comprehensive collection of Java and Spring Framework interview questions organized by topic.

## Contents

| File | Topics | Questions |
|------|--------|-----------|
| [core.md](core.md) | OOP, Interfaces, Keywords, Generics, Exceptions, Serialization, Immutability, SOLID/GRASP | 29 |
| [collections.md](collections.md) | ArrayList, HashMap, TreeMap, HashSet, Collections Internals | 10 |
| [concurrency.md](concurrency.md) | Threads, Synchronization, Locks, Executors, Atomic, Memory | 16 |
| [jvm.md](jvm.md) | JVM Architecture, ClassLoader, Memory Areas, Execution Engine, GC, Security | 27 |
| [modern-java.md](modern-java.md) | Streams, Functional Programming, Java 8/11/17 Features | 11 |
| [spring-core.md](spring-core.md) | DI, IoC, Annotations, Boot, Bean Lifecycle, AOP, Transactions | 20 |
| [spring-web-data.md](spring-web-data.md) | JPA, JDBC, Hibernate, WebFlux, Security | 16 |
| [testing.md](testing.md) | JUnit, Mockito, Spring Boot Testing, Test Slices | 15 |
| [tools.md](tools.md) | Maven, Gradle, Logging (SLF4J/Logback), Task Scheduling | 10 |

**Total: 154 Questions**

---

## Quick Navigation

### Java Core
- **OOP**: Four pillars, Interfaces vs Abstract Classes, SOLID, GRASP
- **Generics**: Type erasure, Bounded types, Wildcards, PECS
- **Exceptions**: Checked/Unchecked, try-with-resources, Exception chaining
- **Serialization**: Serializable, transient, serialVersionUID

### Collections & Data Structures
- ArrayList vs LinkedList, HashMap vs TreeMap
- HashMap internals (buckets, tree conversion)
- ConcurrentHashMap, TreeMap (Red-Black Tree)

### Concurrency
- Thread lifecycle, Synchronized vs Volatile
- ReentrantLock, ReadWriteLock
- ExecutorService, CountDownLatch, Semaphore
- Atomic classes, CAS, Memory (Heap vs Stack)

### JVM & Performance
- JVM Architecture (JDK vs JRE vs JVM, Components)
- ClassLoader (Delegation, Visibility, Uniqueness, Loading phases)
- Memory Areas (Method Area, Heap, Stack, PC Registers, Native Stack)
- Execution Engine (Interpreter, JIT Compiler, Optimizations)
- Garbage Collection (Serial, Parallel, G1, ZGC)
- GC Roots, Stop-the-World, References (Weak, Soft, Phantom)

### Modern Java
- Stream API (map, flatMap, filter, parallel)
- Functional interfaces, Lambdas, Optional
- Java 8/11/17 features (Records, Sealed Classes, Pattern Matching)

### Spring Framework
- **Core**: DI, IoC, Bean lifecycle, Annotations, Auto-configuration
- **AOP**: Aspects, Pointcuts, Advice types
- **Data**: JPA repositories, JdbcTemplate, N+1 problem
- **Web**: WebFlux (Mono/Flux), Security, JWT, CORS

### Testing
- Testing pyramid, Mocking vs Stubbing
- JUnit 5 (annotations, lifecycle, parameterized tests, assertions)
- Mockito (@Mock, @InjectMocks, verify, ArgumentCaptor)
- Spring Boot Test Slices (@WebMvcTest, @DataJpaTest, @SpringBootTest)

### Tools & DevOps
- Build tools (Maven vs Gradle, dependencies, plugins)
- Logging (SLF4J, Logback, log levels, centralized logging)
- Task Scheduling (@Scheduled, ShedLock, distributed locks)

---

[← Back to Main Index](../README.md)

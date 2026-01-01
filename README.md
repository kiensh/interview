# Interview Questions & Answers

A comprehensive collection of interview questions and answers for software engineering roles.

## Contents

| File | Topics | Questions |
|------|--------|-----------|
| [java.md](java.md) | Java Core | 76 |
| [spring.md](spring.md) | Spring Framework | 45 |
| [database/](database/) | Databases | 38 |
| [caching.md](caching.md) | Caching (Redis, Caffeine) | 5 |
| [messaging.md](messaging.md) | Messaging (Kafka) | 4 |
| [microservices.md](microservices.md) | Microservices Architecture | 27 |
| [design-patterns.md](design-patterns.md) | Design Patterns | 21 |
| [testing.md](testing.md) | Testing (JUnit, Mockito) | 15 |
| [tools.md](tools.md) | DevOps & Tools | 10 |

**Total: 241 Questions**

## Topics Covered

### Java Core
- OOP (Object-Oriented Programming)
- Data Structures (Lists, Maps, Sets, Collections)
- Keywords (static, final, volatile, etc.)
- Interfaces and Abstract Classes
- Garbage Collection
- Memory (Heap and Stack)
- Multithreading (lifecycle, locks, concurrency utilities)
- Immutable Classes
- Atomic Classes
- Stream API
- Functional Programming
- Java Version Features (8, 11, 17)
- Collections Internals (ArrayList, HashMap, ConcurrentHashMap, LinkedList, TreeMap)
- JVM Internals (Class Loading, Memory Model)
- Exception Handling
- Serialization & Deserialization
- Generics
- SOLID & GRASP Principles
- Security (Truststore & Keystore)

### Spring Framework
- Dependency Injection
- Spring Annotations
- Spring Boot
- @Transactional Annotation
- Spring Data JPA
- Spring JDBC
- Hibernate & ORM
- Bean Lifecycle & Scopes
- Bean Definition & Injection Types
- IoC (Inversion of Control)
- AOP (Aspect-Oriented Programming)
- AOP Pointcut Expressions
- Spring AOP vs AspectJ
- Reactive Programming (Mono, WebFlux)
- Spring Security (JWT, OAuth2, CORS)
- Testing with Spring

### Databases ([database/](database/))
- SQL vs NoSQL
- Normalization & Denormalization
- Indexing (B-Tree, Hash, GIN, GiST, Clustered/Non-Clustered)
- Partitioning (Range, List, Hash)
- Locking (Optimistic vs Pessimistic)
- ACID vs BASE
- Transactions & Isolation Levels
- Transaction Propagation & Savepoints
- Two-Phase Commit & Distributed Transactions
- Joins & Stored Procedures
- Connection Pooling
- CAP Theorem
- Sharding & Replication
- Deadlocks & Query Optimization

### Caching
- Redis & Memcached
- Caching strategies (Cache-Aside, Write-Through, Write-Behind)
- Ehcache & Caffeine
- Spring Boot caching annotations

### Messaging
- Apache Kafka concepts
- Kafka with Spring Boot
- Kafka vs Message Queues

### Microservices Architecture
- Domain-Driven Design (DDD)
- Bounded Contexts & Aggregates
- Service Discovery & Registry (Eureka)
- Communication Patterns (sync/async)
- API Gateway
- Event Sourcing & CQRS
- Saga Pattern (Choreography vs Orchestration)
- Resiliency Patterns (Circuit Breaker, Retry, Bulkhead)
- Distributed Transactions & Outbox Pattern
- Authentication & OAuth2
- Distributed Tracing & Centralized Logging
- High Availability (99.99% uptime)
- API Versioning

### Design Patterns
- Creational: Singleton, Factory, Abstract Factory, Builder, Prototype
- Structural: Adapter, Decorator, Facade, Proxy, Composite
- Behavioral: Observer, Strategy, Template Method, Chain of Responsibility, Command
- Concurrency: Thread Pool, Future/Promise, Producer-Consumer
- Architectural: MVC, Clean Architecture, Repository

### Testing
- Unit, Integration, E2E Testing
- Testing Pyramid
- Mocking vs Stubbing
- JUnit 5 Annotations & Lifecycle
- Mockito (@Mock, @InjectMocks, ArgumentCaptor)
- Spring Boot Test Slices (@WebMvcTest, @DataJpaTest)

### DevOps & Tools
- Build Tools (Maven vs Gradle)
- Gradle Dependency Management & Plugins
- Logging Frameworks (Log4j, SLF4J, Logback)
- Centralized Logging (ELK Stack)
- Task Scheduling & Distributed Locking

## Usage

Each markdown file contains:
- A table of contents with anchor links
- Questions with detailed answers
- Code examples where applicable
- Comparison tables for easy reference

## License

MIT

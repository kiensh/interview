# Interview Questions & Answers

A comprehensive collection of interview questions and answers for software engineering roles.

## Contents

| Section | Topics | Questions |
|---------|--------|-----------|
| [python/](python/) | **Python & Django** | **~80** |
| [frontend/](frontend/) | **JavaScript, TypeScript, Node.js, React, Vue.js** | **~80** |
| [behavioral/](behavioral/) | **Behavioral Interview Questions** | **~25** |
| [golang/](golang/) | **Go Language & Gin Framework** | **~155** |
| [java/](java/) | Java Core, Concurrency, Modern Java | ~50 |
| [database/](database/) | SQL, NoSQL, Optimization | ~38 |
| [messaging.md](messaging.md) | Messaging (Kafka) | 4 |
| [microservices.md](microservices.md) | Microservices Architecture | 27 |
| [design-patterns.md](design-patterns.md) | Design Patterns | 21 |

**Total: ~480+ Questions**

## Topics Covered

### Python & Django ([python/](python/))
- **Python Core**: Data types, Functions, OOP, Decorators, Generators
- **Advanced**: Closures, Context managers, GIL, Memory management
- **Django**: MTV architecture, Models, Views, Templates, ORM
- **Django REST**: Serializers, ViewSets, Authentication, Pagination
- **Testing**: unittest, pytest, Mocking, Django test client

### Frontend ([frontend/](frontend/))
- **JavaScript**: ES6+, Closures, Promises, Event loop, Prototypes
- **TypeScript**: Types, Interfaces, Generics, Type guards, Utility types
- **Node.js**: Event loop, Express.js, Middleware, Streams, Clustering
- **React**: Components, Hooks, State, Context, Performance
- **Vue.js**: Composition API, Reactivity, Vuex/Pinia, Routing

### Behavioral ([behavioral/](behavioral/))
- Work pressure & stress handling
- Conflict resolution
- Teamwork & collaboration
- Problem solving & initiative
- Career motivation

### Go Language ([golang/](golang/))
- **Core**: Data types, Variables, Control structures, Functions, Pointers, Packages
- **Concurrency**: Goroutines, Channels, Select, Mutex, WaitGroup, sync package
- **Types & Errors**: Structs, Interfaces, Type assertions, Error handling, Custom errors
- **Standard Library**: fmt, os, io, strings, time, net/http, encoding/json
- **Testing**: Unit tests, Table-driven tests, Benchmarks, Mocking
- **Advanced**: Modules, Reflection, Generics, Context package
- **Patterns**: Design patterns, Concurrency patterns, Worker pools, Pipelines
- **Performance**: Profiling (pprof), Memory optimization, CPU optimization
- **APIs**: REST, gRPC, WebSockets, Server-Sent Events, Rate limiting
- **Databases**: SQL (GORM, sqlx, sqlc), NoSQL (Redis, MongoDB), Transactions, Migrations
- **Messaging**: Kafka, RabbitMQ, Event sourcing, CQRS, Pub/Sub
- **Observability**: Structured logging (zerolog, zap), Prometheus metrics, OpenTelemetry
- **Gin Framework**: Routing, Middleware, Validation, Authentication, Sessions, Testing

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

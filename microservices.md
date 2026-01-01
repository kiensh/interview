# Microservices Interview Questions & Answers

---

## Table of Contents

### [Microservices Fundamentals](#microservices-fundamentals)
- [Q1: What are microservices and how do they differ from monolithic architecture?](#q1)
- [Q2: What are the advantages and disadvantages of microservices?](#q2)

### [Domain-Driven Design (DDD)](#domain-driven-design-ddd)
- [Q3: What is Domain-Driven Design?](#q3)
- [Q4: What are Bounded Contexts?](#q4)
- [Q5: What is an Aggregate and Aggregate Root?](#q5)

### [Service Discovery & Registry](#service-discovery--registry)
- [Q6: What is service discovery?](#q6)
- [Q7: What is the difference between client-side and server-side discovery?](#q7)
- [Q8: How does Netflix Eureka work?](#q8)

### [Communication Patterns](#communication-patterns)
- [Q9: What are the different ways microservices communicate?](#q9)
- [Q10: What is the difference between synchronous and asynchronous communication?](#q10)
- [Q11: What is an API Gateway?](#q11)

### [Event Sourcing & CQRS](#event-sourcing--cqrs)
- [Q12: What is Event Sourcing?](#q12)
- [Q13: What is CQRS?](#q13)
- [Q14: When should you use Event Sourcing and CQRS?](#q14)

### [Saga Pattern](#saga-pattern)
- [Q15: What is the Saga pattern?](#q15)
- [Q16: What is the difference between Choreography and Orchestration?](#q16)

### [Resiliency Patterns](#resiliency-patterns)
- [Q17: What is the Circuit Breaker pattern?](#q17)
- [Q18: What are Retry and Timeout patterns?](#q18)
- [Q19: What is the Bulkhead pattern?](#q19)

### [Transaction Management](#transaction-management)
- [Q20: How do you handle distributed transactions?](#q20)
- [Q21: What is the Outbox pattern?](#q21)

### [Security](#security)
- [Q22: How do you implement authentication in microservices?](#q22)
- [Q23: What is OAuth2 and how is it used?](#q23)

### [Monitoring & Observability](#monitoring--observability)
- [Q24: What is distributed tracing?](#q24)
- [Q25: How do you implement centralized logging?](#q25)
- [Q26: How do you achieve 99.99% uptime?](#q26)

### [Versioning](#versioning)
- [Q27: How do you version microservices APIs?](#q27)

---

## Microservices Fundamentals

<a id="q1"></a>
### Q1: What are microservices and how do they differ from monolithic architecture?
**Answer:**

| Monolithic | Microservices |
|------------|---------------|
| Single deployable unit | Multiple independent services |
| Shared database | Database per service |
| Tight coupling | Loose coupling |
| Scale entire application | Scale individual services |
| Single technology stack | Polyglot (multiple languages/DBs) |
| Simpler development | Complex distributed system |
| Single point of failure | Fault isolation |

```
Monolithic:
┌─────────────────────────────────────┐
│           Application               │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   │
│  │User │ │Order│ │Pay  │ │Ship │   │
│  └─────┘ └─────┘ └─────┘ └─────┘   │
│                                     │
│         ┌─────────────┐             │
│         │  Database   │             │
│         └─────────────┘             │
└─────────────────────────────────────┘

Microservices:
┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│User  │  │Order │  │Pay   │  │Ship  │
│Svc   │  │Svc   │  │Svc   │  │Svc   │
└──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
   │         │         │         │
┌──┴───┐  ┌──┴───┐  ┌──┴───┐  ┌──┴───┐
│ DB1  │  │ DB2  │  │ DB3  │  │ DB4  │
└──────┘  └──────┘  └──────┘  └──────┘
```

<a id="q2"></a>
### Q2: What are the advantages and disadvantages of microservices?
**Answer:**

**Advantages:**
| Benefit | Description |
|---------|-------------|
| Independent deployment | Deploy services without affecting others |
| Technology flexibility | Choose best tech for each service |
| Scalability | Scale services independently |
| Fault isolation | Failure in one service doesn't crash all |
| Team autonomy | Small teams own services |
| Faster development | Parallel development by teams |

**Disadvantages:**
| Challenge | Description |
|-----------|-------------|
| Distributed complexity | Network latency, failures |
| Data consistency | No ACID across services |
| Testing complexity | Integration testing harder |
| Operational overhead | Many services to monitor |
| Debugging difficulty | Distributed tracing needed |
| Initial overhead | Infrastructure setup |

---

## Domain-Driven Design (DDD)

<a id="q3"></a>
### Q3: What is Domain-Driven Design?
**Answer:**
DDD is an approach to software development that focuses on the core business domain.

**Key concepts:**

| Concept | Description |
|---------|-------------|
| **Domain** | The subject area the software addresses |
| **Ubiquitous Language** | Common language between devs and domain experts |
| **Entity** | Object with identity (e.g., User with ID) |
| **Value Object** | Immutable object without identity (e.g., Address) |
| **Aggregate** | Cluster of entities treated as unit |
| **Repository** | Abstraction for data access |
| **Domain Service** | Business logic not belonging to entities |
| **Domain Event** | Something that happened in the domain |

```java
// Entity - has identity
@Entity
public class Order {
    @Id
    private UUID id;
    private CustomerId customerId;
    private List<OrderItem> items;
    private OrderStatus status;
    
    public void addItem(Product product, int quantity) {
        items.add(new OrderItem(product, quantity));
        DomainEvents.raise(new OrderItemAdded(this.id, product.getId()));
    }
}

// Value Object - immutable, no identity
public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}

// Domain Service
@Service
public class PricingService {
    public Money calculateTotal(Order order, Customer customer) {
        Money subtotal = order.getSubtotal();
        Money discount = calculateDiscount(customer, subtotal);
        return subtotal.subtract(discount);
    }
}
```

<a id="q4"></a>
### Q4: What are Bounded Contexts?
**Answer:**
A Bounded Context is an explicit boundary within which a domain model exists. Different contexts may have different models for the same concept.

```
E-commerce System:
┌────────────────────┐  ┌────────────────────┐
│   Sales Context    │  │  Shipping Context  │
│                    │  │                    │
│  ┌─────────────┐   │  │  ┌─────────────┐   │
│  │   Product   │   │  │  │   Product   │   │
│  │ - name      │   │  │  │ - weight    │   │
│  │ - price     │   │  │  │ - dimensions│   │
│  │ - category  │   │  │  │ - fragile   │   │
│  └─────────────┘   │  │  └─────────────┘   │
└────────────────────┘  └────────────────────┘

│                        │
│  Context Map           │
│  (Anti-Corruption      │
│   Layer)               │
└────────────────────────┘
```

**Relationships between contexts:**
| Pattern | Description |
|---------|-------------|
| Shared Kernel | Shared subset of domain model |
| Customer-Supplier | Upstream/downstream relationship |
| Conformist | Downstream conforms to upstream |
| Anti-Corruption Layer | Translation layer between contexts |
| Open Host Service | Published API for integration |

```java
// Anti-Corruption Layer - translates between contexts
@Service
public class ShippingAntiCorruptionLayer {
    
    private final SalesClient salesClient;
    
    public ShippingProduct translateProduct(String salesProductId) {
        SalesProduct salesProduct = salesClient.getProduct(salesProductId);
        
        return ShippingProduct.builder()
            .id(salesProduct.getId())
            .weight(extractWeight(salesProduct))
            .dimensions(extractDimensions(salesProduct))
            .fragile(isFragile(salesProduct.getCategory()))
            .build();
    }
}
```

<a id="q5"></a>
### Q5: What is an Aggregate and Aggregate Root?
**Answer:**
An **Aggregate** is a cluster of domain objects treated as a single unit. The **Aggregate Root** is the entry point that ensures consistency.

**Rules:**
1. External objects reference only the aggregate root
2. Aggregate root enforces invariants
3. Delete aggregate root → delete everything inside
4. One transaction = one aggregate

```java
// Order is the Aggregate Root
@Entity
public class Order {  // Aggregate Root
    @Id
    private OrderId id;
    
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();  // Part of aggregate
    
    private OrderStatus status;
    private Money total;
    
    // All modifications go through aggregate root
    public void addItem(ProductId productId, int quantity, Money price) {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        
        OrderItem item = new OrderItem(productId, quantity, price);
        items.add(item);
        recalculateTotal();
    }
    
    public void confirm() {
        if (items.isEmpty()) {
            throw new IllegalStateException("Cannot confirm empty order");
        }
        this.status = OrderStatus.CONFIRMED;
        DomainEvents.raise(new OrderConfirmed(this.id));
    }
    
    private void recalculateTotal() {
        this.total = items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}

// OrderItem - part of aggregate, not directly accessible
@Entity
public class OrderItem {
    @Id
    private OrderItemId id;
    private ProductId productId;
    private int quantity;
    private Money unitPrice;
    
    Money getSubtotal() {
        return unitPrice.multiply(quantity);
    }
}

// Repository for aggregate root only
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
    // No OrderItemRepository!
}
```

---

## Service Discovery & Registry

<a id="q6"></a>
### Q6: What is service discovery?
**Answer:**
Service discovery automatically detects network locations of service instances, essential when services scale dynamically.

```
Without Service Discovery:
┌─────────────┐
│ Service A   │──── hardcoded: http://192.168.1.10:8080 ────✗
└─────────────┘     (IP changes when service restarts)

With Service Discovery:
┌─────────────┐       ┌─────────────────┐       ┌─────────────┐
│ Service A   │──────►│ Service Registry│◄──────│ Service B   │
└─────────────┘ query │ (Eureka/Consul) │ register └─────────────┘
                      │ B: 10.0.1.5:8080 │
                      │ B: 10.0.1.6:8080 │
                      └─────────────────┘
```

<a id="q7"></a>
### Q7: What is the difference between client-side and server-side discovery?
**Answer:**

| Client-Side Discovery | Server-Side Discovery |
|----------------------|----------------------|
| Client queries registry directly | Load balancer queries registry |
| Client does load balancing | Load balancer handles routing |
| More complex client logic | Simpler clients |
| Netflix Eureka + Ribbon | AWS ALB, Kubernetes |

**Client-Side (Netflix Eureka):**
```
┌──────────┐    ┌──────────────┐    ┌──────────┐
│ Service A│───►│Service       │    │Service B │
│          │    │Registry      │◄───│ instance1│
│ (client) │    │(Eureka)      │◄───│ instance2│
└────┬─────┘    └──────────────┘    └──────────┘
     │                   
     │ Direct call with client-side load balancing
     └────────────────────────────────────────────►
```

**Server-Side (Kubernetes):**
```
┌──────────┐    ┌──────────────┐    ┌──────────┐
│ Service A│───►│ Load Balancer│───►│Service B │
│          │    │ (kube-proxy) │───►│ instance1│
│          │    │              │───►│ instance2│
└──────────┘    └──────────────┘    └──────────┘
```

<a id="q8"></a>
### Q8: How does Netflix Eureka work?
**Answer:**

```java
// Eureka Server
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}

// application.yml
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

```java
// Eureka Client (Service)
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}

// application.yml
spring:
  application:
    name: user-service
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
```

```java
// Using discovered service with WebClient
@Service
public class OrderService {
    
    @Autowired
    private WebClient.Builder webClientBuilder;
    
    public Mono<User> getUser(String userId) {
        return webClientBuilder.build()
            .get()
            .uri("http://user-service/users/{id}", userId)  // Service name, not IP
            .retrieve()
            .bodyToMono(User.class);
    }
}

// Or with RestTemplate + @LoadBalanced
@Configuration
public class Config {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

---

## Communication Patterns

<a id="q9"></a>
### Q9: What are the different ways microservices communicate?
**Answer:**

| Pattern | Type | Use Case |
|---------|------|----------|
| REST/HTTP | Synchronous | Request-response |
| gRPC | Synchronous | High-performance, internal |
| Message Queue | Asynchronous | Event-driven, decoupled |
| Event Streaming | Asynchronous | Real-time, high volume |

```
REST (Synchronous):
Service A ──HTTP Request──► Service B
         ◄──HTTP Response──

Message Queue (Asynchronous):
Service A ──publish──► [Queue] ──consume──► Service B
                         │
                         └──consume──► Service C

Event Streaming (Kafka):
Producer ──► [Topic: orders] ──► Consumer Group 1
                    │
                    └──► Consumer Group 2
```

<a id="q10"></a>
### Q10: What is the difference between synchronous and asynchronous communication?
**Answer:**

| Synchronous | Asynchronous |
|-------------|--------------|
| Caller waits for response | Caller doesn't wait |
| Tight coupling in time | Loose coupling |
| Simpler to implement | More complex |
| Cascading failures | Fault tolerant |
| REST, gRPC | Message queues, events |

```java
// Synchronous - REST call
public Order createOrder(OrderRequest request) {
    // Blocks until response
    User user = userClient.getUser(request.getUserId());  // Wait
    Payment payment = paymentClient.process(request);     // Wait
    Inventory inv = inventoryClient.reserve(request);     // Wait
    
    return orderRepository.save(new Order(user, payment, inv));
}

// Asynchronous - Event-driven
@Transactional
public void createOrder(OrderRequest request) {
    Order order = new Order(request);
    order.setStatus(OrderStatus.PENDING);
    orderRepository.save(order);
    
    // Publish event, don't wait
    eventPublisher.publish(new OrderCreatedEvent(order.getId(), request));
}

@KafkaListener(topics = "order-created")
public void handleOrderCreated(OrderCreatedEvent event) {
    // Process asynchronously
    inventoryService.reserve(event.getOrderId(), event.getItems());
}
```

<a id="q11"></a>
### Q11: What is an API Gateway?
**Answer:**
An API Gateway is a single entry point for all clients, handling cross-cutting concerns.

**Responsibilities:**
- Request routing
- Authentication/Authorization
- Rate limiting
- Load balancing
- SSL termination
- Response caching
- Request/Response transformation

```
                    ┌─────────────────┐
Mobile App ────────►│                 │────► User Service
                    │   API Gateway   │────► Order Service
Web App ───────────►│   (Kong/Zuul)   │────► Payment Service
                    │                 │────► Notification Service
External API ──────►└─────────────────┘
```

**Spring Cloud Gateway:**
```java
@SpringBootApplication
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
    
    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("user-service", r -> r
                .path("/api/users/**")
                .filters(f -> f
                    .stripPrefix(1)
                    .addRequestHeader("X-Request-Source", "gateway")
                    .circuitBreaker(c -> c.setName("userCircuitBreaker")))
                .uri("lb://user-service"))
            .route("order-service", r -> r
                .path("/api/orders/**")
                .filters(f -> f.stripPrefix(1))
                .uri("lb://order-service"))
            .build();
    }
}
```

---

## Event Sourcing & CQRS

<a id="q12"></a>
### Q12: What is Event Sourcing?
**Answer:**
Event Sourcing stores all changes to application state as a sequence of events, rather than storing current state.

```
Traditional (State):
┌─────────────────┐
│ Account         │
│ balance: $500   │  ← Only current state
└─────────────────┘

Event Sourcing (Events):
┌─────────────────────────────────────────┐
│ Events:                                 │
│ 1. AccountOpened(id=123)                │
│ 2. MoneyDeposited(amount=1000)          │
│ 3. MoneyWithdrawn(amount=300)           │
│ 4. MoneyWithdrawn(amount=200)           │
│                                         │
│ Current Balance = replay events = $500  │
└─────────────────────────────────────────┘
```

```java
// Events
public sealed interface AccountEvent {
    String accountId();
    Instant timestamp();
}

public record AccountOpened(String accountId, String owner, Instant timestamp) 
    implements AccountEvent {}

public record MoneyDeposited(String accountId, BigDecimal amount, Instant timestamp) 
    implements AccountEvent {}

public record MoneyWithdrawn(String accountId, BigDecimal amount, Instant timestamp) 
    implements AccountEvent {}

// Aggregate rebuilds state from events
public class Account {
    private String id;
    private BigDecimal balance = BigDecimal.ZERO;
    private List<AccountEvent> uncommittedEvents = new ArrayList<>();
    
    // Rebuild from event history
    public static Account fromHistory(List<AccountEvent> history) {
        Account account = new Account();
        history.forEach(account::apply);
        return account;
    }
    
    public void deposit(BigDecimal amount) {
        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        apply(new MoneyDeposited(id, amount, Instant.now()));
    }
    
    private void apply(AccountEvent event) {
        // Update state based on event
        switch (event) {
            case AccountOpened e -> this.id = e.accountId();
            case MoneyDeposited e -> this.balance = balance.add(e.amount());
            case MoneyWithdrawn e -> this.balance = balance.subtract(e.amount());
        }
        uncommittedEvents.add(event);
    }
}

// Event Store
public interface EventStore {
    void save(String aggregateId, List<Event> events, int expectedVersion);
    List<Event> getEvents(String aggregateId);
}
```

<a id="q13"></a>
### Q13: What is CQRS?
**Answer:**
CQRS (Command Query Responsibility Segregation) separates read and write models.

```
Traditional:
┌─────────────┐     ┌──────────────┐
│   Service   │────►│   Database   │
│ (CRUD)      │◄────│ (single model)│
└─────────────┘     └──────────────┘

CQRS:
          Commands (Write)                 Queries (Read)
┌─────────────────────────────┐   ┌─────────────────────────────┐
│    Command Handler          │   │     Query Handler           │
│    (business logic)         │   │     (simple reads)          │
└────────────┬────────────────┘   └────────────┬────────────────┘
             │                                  │
             ▼                                  ▼
┌─────────────────────┐          ┌─────────────────────────────┐
│   Write Database    │ ──sync─► │   Read Database             │
│   (normalized)      │          │   (denormalized for queries)│
└─────────────────────┘          └─────────────────────────────┘
```

```java
// Commands (Write side)
public record CreateOrderCommand(String customerId, List<OrderItem> items) {}

@Service
public class OrderCommandHandler {
    
    @Autowired
    private EventStore eventStore;
    
    public String handle(CreateOrderCommand cmd) {
        Order order = new Order(cmd.customerId(), cmd.items());
        order.validate();
        
        eventStore.save(order.getId(), order.getUncommittedEvents());
        return order.getId();
    }
}

// Queries (Read side)
public record OrderSummary(String orderId, String status, BigDecimal total) {}

@Service
public class OrderQueryHandler {
    
    @Autowired
    private OrderReadRepository readRepository;  // Optimized for reads
    
    public List<OrderSummary> getOrdersByCustomer(String customerId) {
        return readRepository.findByCustomerId(customerId);
    }
}

// Event handler updates read model
@Component
public class OrderProjection {
    
    @EventHandler
    public void on(OrderCreatedEvent event) {
        OrderReadModel readModel = new OrderReadModel(
            event.orderId(),
            event.status(),
            event.total()
        );
        readRepository.save(readModel);
    }
}
```

<a id="q14"></a>
### Q14: When should you use Event Sourcing and CQRS?
**Answer:**

**Use Event Sourcing when:**
- Need complete audit trail
- Need to replay/reconstruct past states
- Complex domain with many state transitions
- Event-driven architecture

**Use CQRS when:**
- Read and write patterns differ significantly
- Need different read models for different clients
- High read-to-write ratio
- Complex queries on normalized data

**Avoid when:**
- Simple CRUD operations
- Small teams
- Simple domain
- Performance not critical

---

## Saga Pattern

<a id="q15"></a>
### Q15: What is the Saga pattern?
**Answer:**
Saga is a pattern for managing distributed transactions across multiple services using a sequence of local transactions with compensating actions.

```
Order Saga:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│1. Create │───►│2. Reserve│───►│3. Process│───►│4. Ship   │
│   Order  │    │ Inventory│    │  Payment │    │   Order  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │               │
     ▼               ▼               ▼
Compensate:    Compensate:    Compensate:
Cancel Order   Release Stock  Refund Payment

If Step 3 fails:
1. Refund Payment (compensate step 3)
2. Release Inventory (compensate step 2)  
3. Cancel Order (compensate step 1)
```

<a id="q16"></a>
### Q16: What is the difference between Choreography and Orchestration?
**Answer:**

| Choreography | Orchestration |
|--------------|---------------|
| Services react to events | Central coordinator controls flow |
| Decentralized | Centralized |
| Loose coupling | Tighter coupling to orchestrator |
| Harder to track | Easy to monitor |
| No single point of failure | Orchestrator is SPOF |

**Choreography (Event-driven):**
```
┌─────────┐ OrderCreated  ┌───────────┐ InventoryReserved ┌─────────┐
│ Order   │──────────────►│ Inventory │──────────────────►│ Payment │
│ Service │               │ Service   │                   │ Service │
└─────────┘               └───────────┘                   └─────────┘
     ▲                          ▲                              │
     │                          │                              │
     └──── OrderCancelled ◄─────┴───── PaymentFailed ◄─────────┘
```

```java
// Choreography - each service publishes/subscribes
@Service
public class OrderService {
    @Transactional
    public void createOrder(OrderRequest request) {
        Order order = orderRepository.save(new Order(request));
        eventPublisher.publish(new OrderCreatedEvent(order));
    }
    
    @KafkaListener(topics = "payment-failed")
    public void onPaymentFailed(PaymentFailedEvent event) {
        Order order = orderRepository.findById(event.getOrderId());
        order.cancel();
        orderRepository.save(order);
    }
}

@Service
public class InventoryService {
    @KafkaListener(topics = "order-created")
    public void onOrderCreated(OrderCreatedEvent event) {
        try {
            inventory.reserve(event.getItems());
            eventPublisher.publish(new InventoryReservedEvent(event.getOrderId()));
        } catch (Exception e) {
            eventPublisher.publish(new InventoryReservationFailedEvent(event.getOrderId()));
        }
    }
}
```

**Orchestration (Central coordinator):**
```
                    ┌─────────────────┐
                    │ Saga Orchestrator│
                    └────────┬────────┘
           ┌─────────────────┼─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
     ┌───────────┐    ┌───────────┐    ┌───────────┐
     │  Order    │    │ Inventory │    │  Payment  │
     │  Service  │    │  Service  │    │  Service  │
     └───────────┘    └───────────┘    └───────────┘
```

```java
// Orchestration - central coordinator
@Service
public class OrderSagaOrchestrator {
    
    public void executeSaga(OrderRequest request) {
        SagaExecution saga = new SagaExecution();
        
        try {
            // Step 1: Create Order
            Order order = orderService.create(request);
            saga.addCompensation(() -> orderService.cancel(order.getId()));
            
            // Step 2: Reserve Inventory
            inventoryService.reserve(order.getItems());
            saga.addCompensation(() -> inventoryService.release(order.getItems()));
            
            // Step 3: Process Payment
            paymentService.process(order.getPaymentDetails());
            saga.addCompensation(() -> paymentService.refund(order.getId()));
            
            // Step 4: Ship Order
            shippingService.ship(order);
            
        } catch (Exception e) {
            saga.compensate();  // Execute compensations in reverse order
            throw new SagaFailedException(e);
        }
    }
}
```

---

## Resiliency Patterns

<a id="q17"></a>
### Q17: What is the Circuit Breaker pattern?
**Answer:**
Circuit Breaker prevents cascading failures by stopping calls to a failing service.

**States:**
```
                    Success
        ┌─────────────────────────────┐
        │                             │
        ▼         Failures > threshold│
     CLOSED ──────────────────────► OPEN
        ▲                             │
        │                             │ Timeout
        │         Success             ▼
        └─────────────────────── HALF_OPEN
                                      │
                                      │ Failure
                                      ▼
                                    OPEN
```

```java
// Using Resilience4j
@Configuration
public class CircuitBreakerConfig {
    
    @Bean
    public CircuitBreaker circuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)           // Open if 50% fail
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowSize(10)              // Last 10 calls
            .minimumNumberOfCalls(5)            // Min calls before calculating
            .build();
        
        return CircuitBreaker.of("paymentService", config);
    }
}

@Service
public class PaymentService {
    
    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
    public Payment processPayment(PaymentRequest request) {
        return paymentClient.process(request);
    }
    
    public Payment fallbackPayment(PaymentRequest request, Exception e) {
        // Queue for later processing
        paymentQueue.add(request);
        return Payment.pending(request.getId());
    }
}
```

<a id="q18"></a>
### Q18: What are Retry and Timeout patterns?
**Answer:**

**Retry Pattern:**
```java
@Service
public class ExternalService {
    
    @Retry(name = "externalApi", fallbackMethod = "fallback")
    public Response callExternalApi(Request request) {
        return externalClient.call(request);
    }
}

// Configuration
resilience4j.retry:
  instances:
    externalApi:
      maxAttempts: 3
      waitDuration: 1s
      exponentialBackoffMultiplier: 2  # 1s, 2s, 4s
      retryExceptions:
        - java.io.IOException
        - java.net.SocketTimeoutException
      ignoreExceptions:
        - com.example.BusinessException
```

**Timeout Pattern:**
```java
@Service
public class SlowService {
    
    @TimeLimiter(name = "slowService", fallbackMethod = "fallback")
    public CompletableFuture<Response> callSlowService() {
        return CompletableFuture.supplyAsync(() -> {
            return slowClient.call();
        });
    }
}

// Configuration
resilience4j.timelimiter:
  instances:
    slowService:
      timeoutDuration: 5s
      cancelRunningFuture: true
```

<a id="q19"></a>
### Q19: What is the Bulkhead pattern?
**Answer:**
Bulkhead isolates failures by partitioning resources, preventing one failing component from taking down the entire system.

```
Without Bulkhead:
┌───────────────────────────────────────┐
│        Thread Pool (100 threads)       │
│ ┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐┌───┐  │
│ │ A ││ A ││ A ││ B ││ B ││ C ││ C │  │
│ └───┘└───┘└───┘└───┘└───┘└───┘└───┘  │
└───────────────────────────────────────┘
If Service A hangs, all 100 threads blocked!

With Bulkhead:
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│Pool A (30)  │ │Pool B (40)  │ │Pool C (30)  │
│┌───┐┌───┐   │ │┌───┐┌───┐   │ │┌───┐┌───┐   │
││ A ││ A │   │ ││ B ││ B │   │ ││ C ││ C │   │
│└───┘└───┘   │ │└───┘└───┘   │ │└───┘└───┘   │
└─────────────┘ └─────────────┘ └─────────────┘
If Service A hangs, only 30 threads affected!
```

```java
@Service
public class OrderService {
    
    @Bulkhead(name = "paymentBulkhead", type = Bulkhead.Type.THREADPOOL)
    public CompletableFuture<Payment> processPayment(Order order) {
        return CompletableFuture.supplyAsync(() -> 
            paymentClient.process(order));
    }
    
    @Bulkhead(name = "inventoryBulkhead", type = Bulkhead.Type.SEMAPHORE)
    public Inventory checkInventory(String productId) {
        return inventoryClient.check(productId);
    }
}

// Configuration
resilience4j.bulkhead:
  instances:
    paymentBulkhead:
      maxConcurrentCalls: 10
      maxWaitDuration: 500ms
    inventoryBulkhead:
      maxConcurrentCalls: 20

resilience4j.thread-pool-bulkhead:
  instances:
    paymentBulkhead:
      maxThreadPoolSize: 10
      coreThreadPoolSize: 5
      queueCapacity: 100
```

---

## Transaction Management

<a id="q20"></a>
### Q20: How do you handle distributed transactions?
**Answer:**

| Approach | Description | Trade-offs |
|----------|-------------|------------|
| **2PC** | Two-phase commit | Strong consistency, blocking |
| **Saga** | Compensating transactions | Eventually consistent |
| **Outbox** | Event + data in same DB transaction | Reliable delivery |
| **TCC** | Try-Confirm-Cancel | Explicit reservation |

**Best practice: Avoid distributed transactions when possible!**

Design strategies:
1. Keep transactions within service boundaries
2. Use eventual consistency
3. Design idempotent operations
4. Use saga for cross-service operations

<a id="q21"></a>
### Q21: What is the Outbox pattern?
**Answer:**
The Outbox pattern ensures reliable event publishing by storing events in the same database transaction as business data.

```
┌─────────────────────────────────────────────────────────┐
│                     Order Service                        │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │            Single DB Transaction                 │   │
│  │  ┌──────────────┐    ┌────────────────────┐     │   │
│  │  │ orders table │    │ outbox table       │     │   │
│  │  │              │    │ (event_type,       │     │   │
│  │  │ id: 123      │    │  payload,          │     │   │
│  │  │ status: NEW  │    │  processed: false) │     │   │
│  │  └──────────────┘    └────────────────────┘     │   │
│  └─────────────────────────────────────────────────┘   │
│                           │                             │
│                           ▼                             │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Message Relay (polls outbox, publishes to Kafka) │   │
│  └─────────────────────────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────┘
                            │
                            ▼
                    ┌──────────────┐
                    │    Kafka     │
                    └──────────────┘
```

```java
@Service
public class OrderService {
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // Save order
        Order order = orderRepository.save(new Order(request));
        
        // Save event to outbox (same transaction!)
        OutboxEvent event = new OutboxEvent(
            "OrderCreated",
            objectMapper.writeValueAsString(new OrderCreatedEvent(order)),
            false
        );
        outboxRepository.save(event);
        
        return order;
    }
}

// Message relay - polls and publishes
@Scheduled(fixedDelay = 1000)
@Transactional
public void publishOutboxEvents() {
    List<OutboxEvent> events = outboxRepository.findByProcessedFalse();
    
    for (OutboxEvent event : events) {
        kafkaTemplate.send(event.getEventType(), event.getPayload());
        event.setProcessed(true);
        outboxRepository.save(event);
    }
}
```

---

## Security

<a id="q22"></a>
### Q22: How do you implement authentication in microservices?
**Answer:**

**Common patterns:**

| Pattern | Description |
|---------|-------------|
| API Gateway Auth | Gateway validates tokens, passes user info |
| Service-to-Service | Mutual TLS or service tokens |
| Token Propagation | Pass user token through service calls |

```
┌────────┐     ┌─────────────┐     ┌─────────────┐
│ Client │────►│ API Gateway │────►│  Services   │
└────────┘     │ (validates  │     │             │
   JWT         │  JWT token) │     │             │
               └─────────────┘     └─────────────┘
                     │
                     ▼
              ┌─────────────┐
              │ Auth Server │
              │  (Keycloak) │
              └─────────────┘
```

<a id="q23"></a>
### Q23: What is OAuth2 and how is it used?
**Answer:**

**OAuth2 Flows:**
| Flow | Use Case |
|------|----------|
| Authorization Code | Web apps with backend |
| Client Credentials | Service-to-service |
| Resource Owner Password | Trusted apps |
| Implicit (deprecated) | SPAs (use PKCE instead) |

```java
// Resource Server configuration
@Configuration
@EnableResourceServer
public class ResourceServerConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated())
            .build();
    }
}

// Service-to-service with Client Credentials
@Configuration
public class OAuth2ClientConfig {
    
    @Bean
    public WebClient webClient(OAuth2AuthorizedClientManager clientManager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction oauth2 =
            new ServletOAuth2AuthorizedClientExchangeFilterFunction(clientManager);
        oauth2.setDefaultClientRegistrationId("internal-service");
        
        return WebClient.builder()
            .apply(oauth2.oauth2Configuration())
            .build();
    }
}
```

---

## Monitoring & Observability

<a id="q24"></a>
### Q24: What is distributed tracing?
**Answer:**
Distributed tracing tracks requests across multiple services using correlation IDs.

```
Request: GET /orders/123
Trace ID: abc-123

┌──────────────────────────────────────────────────────────────┐
│ API Gateway (trace: abc-123, span: 1)                        │
│   └── Order Service (trace: abc-123, span: 2)                │
│         ├── User Service (trace: abc-123, span: 3)           │
│         └── Inventory Service (trace: abc-123, span: 4)      │
│               └── Database (trace: abc-123, span: 5)         │
└──────────────────────────────────────────────────────────────┘
```

```java
// Spring Cloud Sleuth + Zipkin
// Automatically adds trace/span IDs to logs

// application.yml
spring:
  sleuth:
    sampler:
      probability: 1.0  # Sample 100% of requests
  zipkin:
    base-url: http://zipkin:9411

// Logs automatically include trace ID
// 2024-01-15 10:30:45.123 INFO [order-service,abc-123,span-2] ...
```

<a id="q25"></a>
### Q25: How do you implement centralized logging?
**Answer:**

**ELK Stack (Elasticsearch, Logstash, Kibana):**
```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Services │───►│ Logstash │───►│Elastic-  │───►│ Kibana   │
│ (logs)   │    │(process) │    │search    │    │(visualize)│
└──────────┘    └──────────┘    └──────────┘    └──────────┘
```

```java
// Structured logging with correlation ID
@Slf4j
@RestController
public class OrderController {
    
    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        MDC.put("orderId", id);  // Add to all logs in this context
        
        log.info("Fetching order", 
            StructuredArguments.kv("orderId", id),
            StructuredArguments.kv("action", "fetch"));
        
        return orderService.findById(id);
    }
}

// logback-spring.xml for JSON output
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>spanId</includeMdcKeyName>
    </encoder>
</appender>
```

<a id="q26"></a>
### Q26: How do you achieve 99.99% uptime?
**Answer:**
99.99% uptime = max 52 minutes downtime per year.

| Strategy | Description |
|----------|-------------|
| **Redundancy** | Multiple instances, multi-region |
| **Load Balancing** | Distribute traffic, health checks |
| **Auto-scaling** | Handle traffic spikes |
| **Circuit Breakers** | Prevent cascading failures |
| **Health Checks** | Detect and remove unhealthy instances |
| **Blue-Green/Canary** | Zero-downtime deployments |
| **Database HA** | Replication, automatic failover |
| **Monitoring** | Proactive alerting |
| **Chaos Engineering** | Test failure scenarios |

```yaml
# Kubernetes deployment for HA
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: order-service
        resources:
          requests:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
      affinity:
        podAntiAffinity:  # Spread across nodes
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
```

---

## Versioning

<a id="q27"></a>
### Q27: How do you version microservices APIs?
**Answer:**

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| URL Path | `/v1/users` | Clear, cacheable | URL pollution |
| Query Param | `/users?version=1` | Easy to default | Can be ignored |
| Header | `Accept: application/vnd.api.v1+json` | Clean URLs | Hidden |
| Content Negotiation | `Accept: application/json;version=1` | RESTful | Complex |

```java
// URL Path versioning (most common)
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    @GetMapping("/{id}")
    public UserV1 getUser(@PathVariable Long id) { }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    @GetMapping("/{id}")
    public UserV2 getUser(@PathVariable Long id) { }
}

// Header versioning
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(value = "/{id}", headers = "X-API-Version=1")
    public UserV1 getUserV1(@PathVariable Long id) { }
    
    @GetMapping(value = "/{id}", headers = "X-API-Version=2")
    public UserV2 getUserV2(@PathVariable Long id) { }
}
```

**Backward compatibility strategies:**
- Add new fields (don't remove)
- Make new fields optional
- Support old and new versions simultaneously
- Deprecate before removing
- Use API Gateway for routing

---

Good luck with your interview! 🚀

# Messaging & Kafka Interview Questions & Answers

---

## Table of Contents

- [Q1: What is Apache Kafka?](#q1)
- [Q2: What are Kafka's key concepts?](#q2)
- [Q3: How do you use Kafka with Spring Boot?](#q3)
- [Q4: What is the difference between Kafka and message queues?](#q4)

---

<a id="q1"></a>
### Q1: What is Apache Kafka?
**Answer:**
Apache Kafka is a distributed event streaming platform for high-throughput, fault-tolerant messaging.

**Key features:**
- Distributed and scalable
- High throughput (millions of messages/sec)
- Durable storage (persists to disk)
- Real-time processing
- Exactly-once semantics

```
┌───────────────────────────────────────────────────────┐
│                    Kafka Cluster                       │
│  ┌─────────────────────────────────────────────────┐  │
│  │              Topic: orders                       │  │
│  │  ┌───────────┬───────────┬───────────┐          │  │
│  │  │Partition 0│Partition 1│Partition 2│          │  │
│  │  │ [0,1,2,3] │ [0,1,2]   │ [0,1,2,3,4]│         │  │
│  │  └───────────┴───────────┴───────────┘          │  │
│  └─────────────────────────────────────────────────┘  │
│                                                       │
│  Broker 1        Broker 2        Broker 3            │
└───────────────────────────────────────────────────────┘
        ▲                                   │
        │ Produce                           │ Consume
        │                                   ▼
   ┌─────────┐                      ┌───────────────┐
   │ Producer│                      │Consumer Group │
   └─────────┘                      │  ┌────┬────┐  │
                                    │  │ C1 │ C2 │  │
                                    │  └────┴────┘  │
                                    └───────────────┘
```

<a id="q2"></a>
### Q2: What are Kafka's key concepts?
**Answer:**

| Concept | Description |
|---------|-------------|
| **Topic** | Category/feed name for messages |
| **Partition** | Ordered, immutable sequence of messages |
| **Offset** | Unique ID for message within partition |
| **Producer** | Publishes messages to topics |
| **Consumer** | Reads messages from topics |
| **Consumer Group** | Group of consumers sharing workload |
| **Broker** | Kafka server in the cluster |
| **Replication** | Copies of partitions for fault tolerance |

**Message ordering:**
- Ordering guaranteed within a partition only
- Use same key for related messages → same partition

```java
// Producer sends with key for ordering
producer.send(new ProducerRecord<>("orders", 
    orderId,        // Key - determines partition
    orderData));    // Value

// Same orderId always goes to same partition
// Messages for same order processed in order
```

**Consumer groups:**
```
Topic with 3 partitions:
┌────┐ ┌────┐ ┌────┐
│ P0 │ │ P1 │ │ P2 │
└──┬─┘ └──┬─┘ └──┬─┘
   │      │      │
Consumer Group A:
   │      │      │
   ▼      ▼      ▼
┌────┐ ┌────┐ ┌────┐
│ C1 │ │ C2 │ │ C3 │  ← Each consumer gets one partition
└────┘ └────┘ └────┘

Consumer Group B (different group):
   │      │      │
   ▼      ▼      ▼
┌─────────────────┐
│       C1        │  ← Single consumer gets all partitions
└─────────────────┘
```

<a id="q3"></a>
### Q3: How do you use Kafka with Spring Boot?
**Answer:**

**Dependencies:**
```groovy
implementation 'org.springframework.kafka:spring-kafka'
```

**Configuration:**
```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: my-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.example.events"
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
```

**Producer:**
```java
@Service
public class OrderEventProducer {
    
    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void sendOrderCreated(Order order) {
        OrderCreatedEvent event = new OrderCreatedEvent(order);
        
        kafkaTemplate.send("order-events", order.getId(), event)
            .whenComplete((result, ex) -> {
                if (ex == null) {
                    log.info("Sent: {} to partition {}", 
                        event, result.getRecordMetadata().partition());
                } else {
                    log.error("Failed to send", ex);
                }
            });
    }
    
    // Synchronous send
    public void sendSync(OrderEvent event) throws Exception {
        kafkaTemplate.send("order-events", event).get(10, TimeUnit.SECONDS);
    }
}
```

**Consumer:**
```java
@Component
public class OrderEventConsumer {
    
    @KafkaListener(topics = "order-events", groupId = "order-processor")
    public void handleOrderEvent(OrderEvent event) {
        log.info("Received: {}", event);
        processOrder(event);
    }
    
    // With manual acknowledgment
    @KafkaListener(topics = "order-events", groupId = "order-processor")
    public void handleWithAck(OrderEvent event, Acknowledgment ack) {
        try {
            processOrder(event);
            ack.acknowledge();  // Manual commit
        } catch (Exception e) {
            // Don't ack - will be redelivered
            throw e;
        }
    }
    
    // Batch processing
    @KafkaListener(topics = "order-events", groupId = "batch-processor")
    public void handleBatch(List<OrderEvent> events) {
        log.info("Received batch of {} events", events.size());
        events.forEach(this::processOrder);
    }
    
    // With message metadata
    @KafkaListener(topics = "order-events")
    public void handleWithMetadata(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {
        log.info("Received from partition {} at offset {}: {}", 
            partition, offset, event);
    }
}
```

<a id="q4"></a>
### Q4: What is the difference between Kafka and message queues?
**Answer:**

| Feature | Kafka | Message Queue (RabbitMQ, SQS) |
|---------|-------|------------------------------|
| Model | Log-based (append-only) | Queue-based (FIFO) |
| Persistence | Always persisted | Usually transient |
| Consumption | Pull-based | Push or Pull |
| Replay | Yes (read from any offset) | No (consumed = deleted) |
| Consumer model | Consumer groups | Competing consumers |
| Ordering | Per partition | Per queue |
| Throughput | Very high (millions/sec) | Medium |
| Use case | Event streaming, log aggregation | Task queues, RPC |

**When to use Kafka:**
- Event sourcing
- Log aggregation
- Stream processing
- High throughput requirements
- Need to replay events

**When to use Message Queues:**
- Task distribution
- RPC-style communication
- Simple pub/sub
- Guaranteed delivery to one consumer
- Complex routing logic

```
Kafka - Event Log:
┌─────────────────────────────────────────┐
│ [0] [1] [2] [3] [4] [5] [6] [7] →       │
└─────────────────────────────────────────┘
  ↑           ↑               ↑
Consumer A  Consumer B    New messages
(offset 2)  (offset 5)    
                          
Messages persist, consumers track their position

Message Queue:
┌─────────────────────────────────────────┐
│ [msg1] [msg2] [msg3] → Consumer         │
└─────────────────────────────────────────┘
              ↓
           Deleted after consumption
```

---

Good luck with your interview! 🚀

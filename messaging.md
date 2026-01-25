# Messaging & Kafka Interview Questions & Answers

---

## Table of Contents

- [Q1: What is Apache Kafka?](#q1)
- [Q2: What are Kafka's key concepts?](#q2)
- [Q3: How do you use Kafka with Spring Boot?](#q3)
- [Q4: What is the difference between Kafka and message queues?](#q4)
- [Q5: How does Kafka achieve high availability with replication?](#q5)
- [Q6: How do partition keys work and how do you design them?](#q6)
- [Q7: What is log compaction and when should you use it?](#q7)
- [Q8: What happens when a Kafka broker fails?](#q8)
- [Q9: What are the different acks settings and their trade-offs?](#q9)
- [Q10: How does the idempotent producer work and why is it important?](#q10)
- [Q11: How do you tune producer for high throughput vs low latency?](#q11)
- [Q12: How do you handle consumer offset management?](#q12)
- [Q13: What is consumer lag and how do you handle it?](#q13)
- [Q14: What are the partition assignment strategies?](#q14)
- [Q15: How do Kafka transactions work for exactly-once semantics?](#q15)
- [Q16: What is Schema Registry and why is it important?](#q16)
- [Q17: How do you configure Kafka security?](#q17)
- [Q18: What are the key Kafka metrics to monitor?](#q18)

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

<a id="q5"></a>
### Q5: How does Kafka achieve high availability with replication?
**Answer:**

Kafka uses partition replication across multiple brokers to ensure high availability and data durability.

**Key Concepts:**

| Term | Description |
|------|-------------|
| Replication Factor | Number of copies of each partition (typically 3) |
| Leader | Partition replica that handles all reads/writes |
| Follower | Replicas that replicate data from the leader |
| ISR (In-Sync Replicas) | Followers that are fully caught up with the leader |
| min.insync.replicas | Minimum ISR count required for writes to succeed |

**How Replication Works:**

```
Topic: orders (replication-factor=3)

Partition 0:
├── Broker 1: Leader    ← All reads/writes go here
├── Broker 2: Follower (ISR)
└── Broker 3: Follower (ISR)

Partition 1:
├── Broker 2: Leader
├── Broker 1: Follower (ISR)
└── Broker 3: Follower (ISR)
```

**Write Flow:**
1. Producer sends message to partition leader
2. Leader writes to local log
3. Followers fetch and replicate the message
4. Once enough replicas acknowledge (based on `acks`), write is confirmed

**Configuration for Durability:**

| Setting | Recommended | Purpose |
|---------|-------------|---------|
| replication.factor | 3 | Survive 2 broker failures |
| min.insync.replicas | 2 | Require at least 2 replicas for writes |
| acks | all | Wait for all ISR to acknowledge |

**ISR Dynamics:**
- Follower falls out of ISR if it falls too far behind (`replica.lag.time.max.ms`)
- When follower catches up, it rejoins ISR
- If ISR count < `min.insync.replicas`, writes fail (data safety over availability)

<a id="q6"></a>
### Q6: How do partition keys work and how do you design them?
**Answer:**

Partition keys determine which partition a message goes to, affecting ordering and load distribution.

**How Partitioning Works:**

```
Default: partition = hash(key) % num_partitions

Message with key="user-123" → hash("user-123") % 6 → Partition 3
Message with key="user-456" → hash("user-456") % 6 → Partition 1
Message with key=null       → Round-robin distribution
```

**Key Design Principles:**

| Principle | Explanation |
|-----------|-------------|
| Ordering Guarantee | Messages with same key always go to same partition → ordered processing |
| Cardinality | High cardinality keys = better load distribution |
| Hot Partitions | Avoid keys that cause uneven distribution |

**Common Key Strategies:**

| Use Case | Key Strategy | Example |
|----------|--------------|---------|
| User events | User ID | `user-12345` |
| Order processing | Order ID | `order-67890` |
| Multi-tenant | Tenant + Entity ID | `tenant-A:user-123` |
| Time-series | Sensor ID | `sensor-west-01` |
| Geographic | Region | `us-east`, `eu-west` |

**Avoiding Hot Partitions:**

```
❌ Bad: Using country as key
   - "US" gets 60% of traffic → one partition overloaded

✅ Good: Using user_id as key
   - Even distribution across partitions

✅ Good: Compound key for ordering within groups
   - key = "customer-123" (all orders for customer in same partition)
```

**When to Use Null Keys:**
- When ordering doesn't matter
- Want maximum parallelism and even distribution
- Log aggregation, metrics collection

**Trade-offs:**

| More Partitions | Fewer Partitions |
|-----------------|------------------|
| Higher parallelism | Lower overhead |
| Better throughput | Simpler management |
| More rebalancing time | Faster rebalancing |
| More file handles | Lower resource usage |

<a id="q7"></a>
### Q7: What is log compaction and when should you use it?
**Answer:**

Log compaction retains only the latest value for each key, rather than all historical messages.

**Retention vs Compaction:**

| Retention-based | Compaction-based |
|-----------------|------------------|
| Delete messages older than X days | Keep latest value per key |
| Time or size-based cleanup | Key-based cleanup |
| All history within window | Only latest state |
| Use: Event streams, logs | Use: State, CDC, snapshots |

**How Compaction Works:**

```
Before Compaction:
Offset 0: key=A, value=1
Offset 1: key=B, value=2
Offset 2: key=A, value=3  ← newer value for A
Offset 3: key=C, value=4
Offset 4: key=B, value=5  ← newer value for B
Offset 5: key=A, value=null (tombstone)

After Compaction:
Offset 3: key=C, value=4
Offset 4: key=B, value=5
(key=A deleted due to tombstone)
```

**Use Cases:**

| Use Case | Why Compaction |
|----------|----------------|
| Database CDC | Latest row state for each primary key |
| User profiles | Current profile, not history |
| Configuration | Latest config values |
| Cache rebuilding | Rebuild cache from topic |
| Kafka Streams state stores | KTable changelog topics |

**Configuration:**

| Setting | Description |
|---------|-------------|
| `cleanup.policy=compact` | Enable compaction |
| `cleanup.policy=compact,delete` | Compact + delete old segments |
| `min.cleanable.dirty.ratio` | Trigger compaction when dirty ratio exceeds |
| `delete.retention.ms` | How long to keep tombstones |
| `segment.ms` | Closed segments are eligible for compaction |

**Tombstones (Deletion):**
- Message with `value=null` marks key for deletion
- Tombstone retained for `delete.retention.ms`
- Consumers can see the deletion event
- After retention, key is fully removed

**Important Considerations:**
- Compaction is asynchronous (not immediate)
- Active segment is never compacted
- Messages may exist temporarily after "deletion"
- Ordering within a key is preserved

<a id="q8"></a>
### Q8: What happens when a Kafka broker fails?
**Answer:**

Kafka handles broker failures through automatic leader election and consumer rebalancing.

**Broker Failure Scenarios:**

**1. Leader Broker Fails:**

```
Before Failure:
Partition 0: Broker 1 (Leader) ← fails
            Broker 2 (Follower, ISR)
            Broker 3 (Follower, ISR)

After Failure:
Partition 0: Broker 2 (New Leader) ← elected from ISR
            Broker 3 (Follower, ISR)
            Broker 1 (offline)
```

**Leader Election Process:**
1. Controller detects broker failure (via ZooKeeper/KRaft)
2. Controller selects new leader from ISR
3. Controller updates metadata
4. Clients refresh metadata and connect to new leader
5. Brief unavailability during election (typically milliseconds)

**2. Follower Broker Fails:**

```
Before: ISR = [Broker 1 (leader), Broker 2, Broker 3]
Broker 3 fails...
After:  ISR = [Broker 1 (leader), Broker 2]

- Writes continue if ISR >= min.insync.replicas
- When Broker 3 recovers, it catches up and rejoins ISR
```

**3. Impact on Producers:**

| Scenario | acks=1 | acks=all |
|----------|--------|----------|
| Leader fails mid-write | Message may be lost | Message safe if ISR acknowledged |
| ISR < min.insync.replicas | Writes succeed | Writes fail (NotEnoughReplicas) |

**4. Impact on Consumers:**

```
Consumer Group Rebalancing:

Before: Consumer 1 → Partitions [0, 1]
        Consumer 2 → Partitions [2, 3]
        
Consumer 2 fails...

After:  Consumer 1 → Partitions [0, 1, 2, 3] (rebalanced)
```

**Recovery Timeline:**

| Phase | Duration | Description |
|-------|----------|-------------|
| Detection | 1-10 sec | `session.timeout.ms` for consumers |
| Election | ~ms | Controller elects new leader |
| Metadata propagation | ~ms | Clients get new leader info |
| Consumer rebalance | seconds | `max.poll.interval.ms` timeout |

**Unclean Leader Election:**
- If all ISR replicas fail, can elect out-of-sync replica
- `unclean.leader.election.enable=false` (default): Wait for ISR replica
- `unclean.leader.election.enable=true`: Possible data loss but availability

<a id="q9"></a>
### Q9: What are the different acks settings and their trade-offs?
**Answer:**

The `acks` (acknowledgments) setting controls durability vs performance trade-off for producers.

**Acks Settings:**

| acks | Behavior | Durability | Performance |
|------|----------|------------|-------------|
| `0` | Fire and forget, no acknowledgment | Lowest | Highest throughput |
| `1` | Wait for leader acknowledgment | Medium | Good throughput |
| `all` / `-1` | Wait for all ISR acknowledgment | Highest | Lower throughput |

**acks=0 (Fire and Forget):**

```
Producer → Broker (Leader)
    ↓
No response waited

- No delivery confirmation
- Possible message loss (network issues, broker crash)
- Use case: Metrics, logs where some loss is acceptable
```

**acks=1 (Leader Only):**

```
Producer → Broker (Leader) → Ack
                ↓
        Async replication to followers

- Confirmed when leader writes to local log
- Risk: Leader crashes before replication → message lost
- Use case: Balance of durability and performance
```

**acks=all (All In-Sync Replicas):**

```
Producer → Broker (Leader) → Replicate → All ISR Ack → Ack to Producer

- Confirmed when all ISR replicas acknowledge
- Combined with min.insync.replicas for strong guarantees
- Use case: Financial transactions, critical data
```

**Combining with min.insync.replicas:**

```
Configuration:
- replication.factor = 3
- min.insync.replicas = 2
- acks = all

Scenarios:
✅ ISR = [B1, B2, B3]: Write succeeds (3 >= 2)
✅ ISR = [B1, B2]:     Write succeeds (2 >= 2)
❌ ISR = [B1]:         Write fails NotEnoughReplicasException (1 < 2)
```

**Performance Comparison:**

| Setting | Latency | Throughput | Data Safety |
|---------|---------|------------|-------------|
| acks=0 | ~0.5ms | Highest | None |
| acks=1 | ~2-5ms | High | Leader only |
| acks=all | ~5-15ms | Medium | Full (with min.insync.replicas) |

**Recommendations:**

| Use Case | Recommended Setting |
|----------|---------------------|
| Logs, metrics (loss acceptable) | acks=0 or acks=1 |
| General application events | acks=1 |
| Financial, orders, critical data | acks=all + min.insync.replicas=2 |

<a id="q10"></a>
### Q10: How does the idempotent producer work and why is it important?
**Answer:**

Idempotent producer ensures exactly-once delivery semantics by preventing duplicate messages from retries.

**The Duplicate Problem:**

```
Without Idempotence:
1. Producer sends message
2. Broker writes message, sends ack
3. Ack lost (network issue)
4. Producer retries (thinks it failed)
5. Broker writes DUPLICATE message

Result: Same message written twice
```

**How Idempotent Producer Works:**

```
With Idempotence:
1. Producer gets Producer ID (PID) from broker
2. Each message gets sequence number per partition
3. Broker tracks: (PID, Partition) → last sequence number

Message: {PID: 1000, Partition: 0, Sequence: 5, data: "..."}

Broker logic:
- If sequence == expected: Write and ack
- If sequence < expected: Already received, ack without writing (dedup)
- If sequence > expected: Out of order error
```

**Enabling Idempotence:**

```
Producer Config:
enable.idempotence = true

// Automatically sets:
// acks = all
// retries = Integer.MAX_VALUE
// max.in.flight.requests.per.connection <= 5
```

**What Idempotence Guarantees:**

| Guarantee | Scope |
|-----------|-------|
| No duplicates | Within a single producer session |
| Ordering | Per partition (even with retries) |
| Exactly-once | Producer to broker (not end-to-end) |

**Limitations:**

| Limitation | Explanation |
|------------|-------------|
| Single session | New producer instance = new PID = possible duplicates |
| Producer-side only | Doesn't prevent consumer duplicates |
| Not across partitions | Only guarantees within single partition |

**When Producer Restarts:**

```
Session 1: PID=1000, Sequence=0,1,2,3... (crash)
Session 2: PID=1001, Sequence=0,1,2,3... (new PID)

- If crash happened after send but before ack
- Message might be duplicated across sessions
- Solution: Use transactions for cross-session guarantees
```

**Idempotent vs Transactional Producer:**

| Feature | Idempotent | Transactional |
|---------|------------|---------------|
| Deduplication | ✅ | ✅ |
| Ordering | Per partition | Per partition |
| Atomicity | Single message | Multiple messages |
| Cross-session | ❌ | ✅ |
| Performance | ~Same as acks=all | Slight overhead |

<a id="q11"></a>
### Q11: How do you tune producer for high throughput vs low latency?
**Answer:**

Producer tuning involves trade-offs between throughput, latency, and durability.

**Key Configuration Parameters:**

| Parameter | Effect on Throughput | Effect on Latency |
|-----------|---------------------|-------------------|
| `batch.size` | ↑ larger = higher | ↑ larger = higher |
| `linger.ms` | ↑ longer = higher | ↑ longer = higher |
| `compression.type` | ↑ compression = higher | Slight increase |
| `buffer.memory` | ↑ larger = higher | Prevents blocking |
| `acks` | ↓ lower = higher | ↓ lower = lower |

**High Throughput Configuration:**

```
# Batch more messages
batch.size = 65536          # 64KB (default 16KB)
linger.ms = 50              # Wait up to 50ms to fill batch

# Compression
compression.type = lz4      # or snappy, zstd

# Buffer
buffer.memory = 67108864    # 64MB

# Parallelism
max.in.flight.requests.per.connection = 5

# Durability trade-off
acks = 1                    # or 0 for maximum throughput
```

**Low Latency Configuration:**

```
# Send immediately
batch.size = 16384          # Smaller batches
linger.ms = 0               # No waiting

# No compression (CPU overhead)
compression.type = none

# Quick acknowledgment
acks = 1

# Fewer in-flight requests
max.in.flight.requests.per.connection = 1
```

**Batching Explained:**

```
linger.ms = 0:
Messages sent immediately as individual requests
[M1] → [M2] → [M3] → (3 network round trips)

linger.ms = 50:
Messages batched and sent together
[M1, M2, M3] → (1 network round trip)
```

**Compression Comparison:**

| Type | Compression Ratio | CPU Usage | Speed |
|------|-------------------|-----------|-------|
| none | 1x | None | Fastest |
| snappy | ~2x | Low | Fast |
| lz4 | ~2.5x | Low | Fast |
| gzip | ~3x | High | Slow |
| zstd | ~3.5x | Medium | Medium |

**Buffer Memory Management:**

```
buffer.memory = 32MB (default)

If buffer fills up:
- max.block.ms: How long to block before throwing exception
- Default: 60000ms (60 seconds)

Signs buffer is too small:
- TimeoutException on send()
- High "buffer-available-bytes" metric fluctuation
```

**Tuning Checklist:**

| Goal | Settings |
|------|----------|
| Maximum throughput | linger.ms=100, batch.size=128KB, compression=lz4, acks=1 |
| Minimum latency | linger.ms=0, batch.size=16KB, compression=none, acks=1 |
| Balanced | linger.ms=5-20, batch.size=32KB, compression=snappy, acks=all |

<a id="q12"></a>
### Q12: How do you handle consumer offset management?
**Answer:**

Offset management determines message delivery semantics and processing guarantees.

**Offset Basics:**

```
Partition: [msg0][msg1][msg2][msg3][msg4][msg5]
                              ↑
                        committed offset = 3
                        (messages 0-2 processed)
```

**Auto Commit vs Manual Commit:**

| Auto Commit | Manual Commit |
|-------------|---------------|
| `enable.auto.commit=true` | `enable.auto.commit=false` |
| Commits every `auto.commit.interval.ms` | Explicit commit call |
| At-most-once risk | At-least-once or exactly-once |
| Simpler code | Full control |

**Auto Commit Risks:**

```
Auto Commit Scenario:
1. Poll messages [5, 6, 7]
2. Auto-commit fires → offset=8
3. Processing message 6 → crash!
4. Restart → reads from offset 8
5. Messages 6, 7 LOST (at-most-once)
```

**Manual Commit Strategies:**

**1. Synchronous Commit (Safest):**
```
while (true) {
    messages = consumer.poll()
    for (msg in messages) {
        process(msg)
    }
    consumer.commitSync()  // Blocks until confirmed
}
// At-least-once: Message processed before commit
```

**2. Asynchronous Commit (Better throughput):**
```
while (true) {
    messages = consumer.poll()
    for (msg in messages) {
        process(msg)
    }
    consumer.commitAsync()  // Non-blocking
}
// Risk: Commit might fail silently
```

**3. Commit Per Message (Fine-grained):**
```
while (true) {
    messages = consumer.poll()
    for (msg in messages) {
        process(msg)
        consumer.commitSync(msg.offset + 1)  // Commit after each
    }
}
// Highest overhead, most granular control
```

**4. Commit Per Batch:**
```
while (true) {
    messages = consumer.poll()
    process_batch(messages)
    consumer.commitSync()  // Commit after batch
}
// Balance of safety and performance
```

**Seeking to Specific Offsets:**

| Method | Use Case |
|--------|----------|
| `seekToBeginning()` | Reprocess all messages |
| `seekToEnd()` | Skip to latest, ignore backlog |
| `seek(partition, offset)` | Resume from specific point |
| `offsetsForTimes()` | Find offset by timestamp |

**auto.offset.reset Settings:**

| Setting | Behavior | Use Case |
|---------|----------|----------|
| `earliest` | Start from beginning | New consumer, need all data |
| `latest` | Start from end | Only want new messages |
| `none` | Throw exception | Explicit offset management |

**Delivery Semantics Summary:**

| Semantics | How to Achieve |
|-----------|----------------|
| At-most-once | Auto-commit or commit before processing |
| At-least-once | Manual commit after processing |
| Exactly-once | Transactions + idempotent processing |

<a id="q13"></a>
### Q13: What is consumer lag and how do you handle it?
**Answer:**

Consumer lag is the difference between the latest message offset and the consumer's committed offset.

**Understanding Lag:**

```
Partition:    [0][1][2][3][4][5][6][7][8][9]
                            ↑           ↑
                     Consumer Offset   Latest Offset
                         (5)              (9)
                         
Consumer Lag = 9 - 5 = 4 messages behind
```

**Causes of Consumer Lag:**

| Cause | Description |
|-------|-------------|
| Slow processing | Consumer can't keep up with production rate |
| Consumer downtime | Consumer offline, messages accumulate |
| Rebalancing | Partitions reassigned during rebalance |
| Network issues | Slow fetches from broker |
| GC pauses | Long garbage collection stops processing |

**Monitoring Lag:**

```bash
# Kafka CLI
kafka-consumer-groups.sh --describe --group my-group

GROUP     TOPIC     PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
my-group  orders    0          1000            1050            50
my-group  orders    1          2000            2000            0
my-group  orders    2          1500            1600            100
```

**Key Metrics:**

| Metric | Description |
|--------|-------------|
| `records-lag` | Messages behind per partition |
| `records-lag-max` | Maximum lag across partitions |
| `records-consumed-rate` | Consumption throughput |
| `fetch-rate` | Fetch requests per second |

**Handling Lag:**

**1. Scale Consumers (Horizontal):**
```
Before: 1 consumer → 6 partitions (overloaded)
After:  6 consumers → 6 partitions (1:1 mapping)

Note: Consumers cannot exceed partition count
```

**2. Optimize Processing:**
```
- Batch processing instead of per-message
- Async I/O for external calls
- Parallel processing within consumer
- Reduce processing time per message
```

**3. Increase Fetch Size:**
```
fetch.min.bytes = 1048576      # 1MB minimum fetch
fetch.max.wait.ms = 500        # Wait up to 500ms
max.partition.fetch.bytes = 1048576  # Per partition
```

**4. Handle Backpressure:**
```
- Pause consumption when downstream is slow
- consumer.pause(partitions)
- consumer.resume(partitions) when ready
```

**5. Skip or Sample (Emergency):**
```
- Seek to end: consumer.seekToEnd(partitions)
- Process only recent: seek to timestamp
- Sample messages (process every Nth)
```

**Lag Alerting Thresholds:**

| Lag Level | Action |
|-----------|--------|
| < 1000 | Normal operation |
| 1000 - 10000 | Warning, investigate |
| 10000 - 100000 | Critical, scale consumers |
| > 100000 | Emergency, consider skipping |

**Prevention:**

| Strategy | Implementation |
|----------|----------------|
| Right-size partitions | More partitions = more parallelism |
| Capacity planning | Monitor and scale proactively |
| Auto-scaling | Scale consumers based on lag metrics |
| Circuit breakers | Prevent cascade failures |

<a id="q14"></a>
### Q14: What are the partition assignment strategies?
**Answer:**

Partition assignment strategies determine how partitions are distributed among consumers in a group.

**Built-in Strategies:**

**1. Range Assignment (Default):**

```
Topics: T1 (3 partitions), T2 (3 partitions)
Consumers: C1, C2

Assignment:
C1 → T1-P0, T1-P1, T2-P0, T2-P1
C2 → T1-P2, T2-P2

Pros: Co-locates same partition numbers across topics
Cons: Uneven distribution possible
```

**2. RoundRobin Assignment:**

```
Topics: T1 (3 partitions), T2 (3 partitions)
Consumers: C1, C2

All partitions sorted, distributed round-robin:
C1 → T1-P0, T1-P2, T2-P1
C2 → T1-P1, T2-P0, T2-P2

Pros: Even distribution
Cons: No partition affinity across topics
```

**3. Sticky Assignment:**

```
Initial:
C1 → P0, P1, P2
C2 → P3, P4, P5

C2 leaves (rebalance):
C1 → P0, P1, P2, P3, P4, P5  (keeps original + gets C2's)

C2 rejoins (rebalance):
C1 → P0, P1, P2  (keeps original)
C2 → P3, P4, P5  (gets back its partitions)

Pros: Minimizes partition movement during rebalance
Cons: Slightly more complex
```

**4. Cooperative Sticky (Incremental Rebalance):**

```
Traditional Rebalance:
1. All consumers stop
2. All partitions revoked
3. Reassignment calculated
4. All partitions assigned
→ Full stop-the-world

Cooperative Rebalance:
1. Calculate new assignment
2. Only revoke partitions that need to move
3. Assign revoked partitions to new owners
→ Minimal disruption
```

**Comparison:**

| Strategy | Distribution | Rebalance Impact | Use Case |
|----------|--------------|------------------|----------|
| Range | May be uneven | High | Co-located processing |
| RoundRobin | Even | High | Simple, even load |
| Sticky | Even | Medium | Stateful consumers |
| CooperativeSticky | Even | Low | Production recommended |

**Rebalance Impact:**

```
Traditional (Eager):
Time: ──────[STOP]──────[REBALANCE]──────[RESUME]──────
             ↑ all processing stops       ↑ all resume

Cooperative (Incremental):
C1:   ──────────────────────────────────────────────
C2:   ──────[revoke P3]──────────────────────────────
C3:   ─────────────────────[assign P3]───────────────
             ↑ only affected partitions stop
```

**Configuration:**

```
# Consumer config
partition.assignment.strategy = 
    org.apache.kafka.clients.consumer.CooperativeStickyAssignor

# Multiple strategies (fallback)
partition.assignment.strategy = 
    org.apache.kafka.clients.consumer.CooperativeStickyAssignor,
    org.apache.kafka.clients.consumer.RangeAssignor
```

**Choosing a Strategy:**

| Scenario | Recommended Strategy |
|----------|---------------------|
| Simple workloads | RoundRobin |
| Stateful processing (local cache) | Sticky |
| Large consumer groups | CooperativeSticky |
| Minimize downtime | CooperativeSticky |
| Cross-topic partition affinity | Range |

<a id="q15"></a>
### Q15: How do Kafka transactions work for exactly-once semantics?
**Answer:**

Kafka transactions enable atomic writes across multiple partitions and exactly-once semantics for stream processing.

**Transaction Guarantees:**

| Guarantee | Description |
|-----------|-------------|
| Atomicity | All messages in transaction succeed or all fail |
| Cross-partition | Atomic writes to multiple partitions |
| Exactly-once | No duplicates even across producer restarts |
| Read committed | Consumers only see committed messages |

**Transaction Flow:**

```
1. Producer: beginTransaction()
2. Producer: send(topic1, message1)
3. Producer: send(topic2, message2)
4. Producer: sendOffsetsToTransaction(offsets)  // For consume-transform-produce
5. Producer: commitTransaction() or abortTransaction()

Broker:
- Messages written with "uncommitted" marker
- On commit: marker changed to "committed"
- On abort: messages ignored by consumers
```

**Transactional Producer Setup:**

```
# Producer configuration
transactional.id = my-transactional-producer  # Required, must be unique
enable.idempotence = true                     # Automatically enabled
acks = all                                    # Automatically set
```

**Transactional Consumer Setup:**

```
# Consumer configuration
isolation.level = read_committed  # Only see committed messages

# Alternative
isolation.level = read_uncommitted  # See all messages (default)
```

**Consume-Transform-Produce Pattern:**

```
Transaction Scope:
┌─────────────────────────────────────────────────┐
│ 1. Consume from input topic                     │
│ 2. Process/Transform                            │
│ 3. Produce to output topic                      │
│ 4. Commit consumer offsets                      │
│                                                 │
│ All atomic - either all succeed or all rollback │
└─────────────────────────────────────────────────┘
```

**Transaction States:**

```
Empty → Initializing → Ready → InTransaction → CommittingTransaction → Ready
                                    ↓
                         AbortingTransaction → Ready
```

**Exactly-Once Semantics (EOS):**

| Level | Scope | How |
|-------|-------|-----|
| Idempotent Producer | Single partition | PID + sequence number |
| Transactions | Multiple partitions | Transaction coordinator |
| EOS in Streams | End-to-end | Consume + process + produce atomic |

**Failure Scenarios:**

| Failure | Transaction Behavior |
|---------|---------------------|
| Producer crash before commit | Transaction times out, aborted |
| Producer crash after commit started | Coordinator completes commit |
| Broker crash | Transaction log replicated, recovery continues |
| Network partition | Transaction timeout, abort |

**Transaction Timeout:**

```
transaction.timeout.ms = 60000  # Default 1 minute

If transaction not completed within timeout:
- Coordinator aborts transaction
- Producer gets InvalidProducerEpoch on next operation
- Must reinitialize producer
```

**Performance Considerations:**

| Aspect | Impact |
|--------|--------|
| Latency | ~10-20ms overhead per transaction |
| Throughput | Batch multiple messages per transaction |
| Storage | Transaction markers stored in log |

**When to Use Transactions:**

| Use Case | Need Transactions? |
|----------|-------------------|
| Log aggregation | No |
| Metrics collection | No |
| Stream processing (exactly-once) | Yes |
| Financial operations | Yes |
| Multi-topic atomic writes | Yes |

<a id="q16"></a>
### Q16: What is Schema Registry and why is it important?
**Answer:**

Schema Registry is a centralized service for managing and validating message schemas, enabling schema evolution without breaking consumers.

**Why Schema Registry:**

```
Without Schema Registry:
Producer → {"name": "John", "age": 30} → Consumer
Producer → {"name": "John", "years": 30} → Consumer ❌ (breaking change)

With Schema Registry:
Producer → [schema-id][binary data] → Consumer
           ↓
    Schema Registry validates compatibility
```

**Key Concepts:**

| Concept | Description |
|---------|-------------|
| Schema | Structure definition (Avro, Protobuf, JSON Schema) |
| Subject | Name under which schema is registered (usually topic-name-value) |
| Version | Sequential version number for schema evolution |
| Schema ID | Global unique identifier for a schema |
| Compatibility | Rules for schema evolution |

**How It Works:**

```
1. Producer registers schema (if new):
   POST /subjects/orders-value/versions
   → Returns schema_id: 42

2. Producer serializes message:
   [Magic Byte][Schema ID: 42][Avro Binary Data]

3. Consumer deserializes:
   - Reads schema_id from message
   - Fetches schema from registry (cached)
   - Deserializes using schema
```

**Compatibility Modes:**

| Mode | Allowed Changes | Use Case |
|------|-----------------|----------|
| BACKWARD | Delete fields, add optional fields | Default, safe |
| FORWARD | Add fields, delete optional fields | Producer-first deploys |
| FULL | Add/delete optional fields only | Strictest evolution |
| NONE | Any change | Development only |

**Compatibility Examples (Avro):**

```
BACKWARD Compatible (consumer can read old with new schema):
v1: {name: string, age: int}
v2: {name: string, age: int, email: string = ""} ✅ (default provided)

BACKWARD Incompatible:
v1: {name: string, age: int}
v2: {name: string, age: int, email: string} ❌ (required field added)

FORWARD Compatible (consumer can read new with old schema):
v1: {name: string, age: int, email: string}
v2: {name: string, age: int} ✅ (field removed)
```

**Schema Evolution Best Practices:**

| Do | Don't |
|----|-------|
| Add fields with defaults | Add required fields |
| Use optional/nullable fields | Remove required fields |
| Use union types for flexibility | Change field types |
| Plan schema from start | Rename fields |

**Serialization Formats:**

| Format | Binary | Schema Evolution | Performance |
|--------|--------|------------------|-------------|
| Avro | Yes | Excellent | Fast |
| Protobuf | Yes | Excellent | Fast |
| JSON Schema | No | Limited | Slower |

**Subject Naming Strategies:**

| Strategy | Subject Name | Use Case |
|----------|--------------|----------|
| TopicName | `<topic>-value` | Default, simple |
| RecordName | `<record-name>` | Share schema across topics |
| TopicRecordName | `<topic>-<record>` | Multiple record types per topic |

**Benefits:**

| Benefit | Description |
|---------|-------------|
| Decoupling | Producers/consumers evolve independently |
| Validation | Prevent incompatible schemas |
| Documentation | Central schema catalog |
| Efficiency | Binary serialization (Avro/Protobuf) |
| Versioning | Track schema history |

<a id="q17"></a>
### Q17: How do you configure Kafka security?
**Answer:**

Kafka security covers authentication, authorization, and encryption.

**Security Components:**

| Component | Purpose | Mechanisms |
|-----------|---------|------------|
| Authentication | Verify identity | SASL, SSL/TLS client certs |
| Authorization | Control access | ACLs |
| Encryption | Protect data | SSL/TLS (transit), custom (at rest) |

**Authentication Mechanisms:**

**1. SASL/PLAIN (Username/Password):**
```
# Broker
listeners=SASL_PLAINTEXT://localhost:9092
sasl.enabled.mechanisms=PLAIN
sasl.mechanism.inter.broker.protocol=PLAIN

# Client
security.protocol=SASL_PLAINTEXT
sasl.mechanism=PLAIN
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule required \
    username="client" password="client-secret";
```

**2. SASL/SCRAM (Salted Challenge Response):**
```
# More secure than PLAIN (password not sent in clear)
sasl.enabled.mechanisms=SCRAM-SHA-256,SCRAM-SHA-512
```

**3. SASL/GSSAPI (Kerberos):**
```
# Enterprise environments with existing Kerberos infrastructure
sasl.enabled.mechanisms=GSSAPI
sasl.kerberos.service.name=kafka
```

**4. mTLS (Mutual TLS):**
```
# Client authentication via certificates
ssl.client.auth=required
ssl.keystore.location=/path/to/keystore.jks
ssl.truststore.location=/path/to/truststore.jks
```

**Encryption in Transit (TLS):**

```
# Broker configuration
listeners=SSL://localhost:9093
ssl.keystore.location=/path/to/kafka.server.keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password
ssl.truststore.location=/path/to/kafka.server.truststore.jks
ssl.truststore.password=truststore-password

# Client configuration
security.protocol=SSL
ssl.truststore.location=/path/to/client.truststore.jks
ssl.truststore.password=truststore-password
```

**Authorization (ACLs):**

```bash
# Grant producer access
kafka-acls.sh --add --allow-principal User:producer1 \
    --operation Write --topic orders

# Grant consumer access
kafka-acls.sh --add --allow-principal User:consumer1 \
    --operation Read --topic orders \
    --group order-processors

# Grant admin access
kafka-acls.sh --add --allow-principal User:admin \
    --operation All --topic '*'
```

**ACL Operations:**

| Operation | Description |
|-----------|-------------|
| Read | Consume from topic |
| Write | Produce to topic |
| Create | Create topics |
| Delete | Delete topics |
| Alter | Modify topic config |
| Describe | View topic metadata |
| ClusterAction | Cluster-level operations |
| All | All operations |

**Security Protocols:**

| Protocol | Authentication | Encryption |
|----------|----------------|------------|
| PLAINTEXT | None | None |
| SSL | Optional (mTLS) | Yes |
| SASL_PLAINTEXT | SASL | None |
| SASL_SSL | SASL | Yes |

**Encryption at Rest:**

```
# Kafka doesn't provide native at-rest encryption
# Options:
1. Filesystem encryption (dm-crypt, LUKS)
2. Cloud provider encryption (AWS EBS, GCP PD)
3. Application-level encryption (encrypt before produce)
```

**Security Best Practices:**

| Practice | Recommendation |
|----------|----------------|
| Protocol | Use SASL_SSL in production |
| Authentication | SCRAM or Kerberos (not PLAIN) |
| ACLs | Principle of least privilege |
| Certificates | Rotate regularly |
| Secrets | Use vault/secrets manager |
| Network | Isolate Kafka in private network |

<a id="q18"></a>
### Q18: What are the key Kafka metrics to monitor?
**Answer:**

Monitoring Kafka involves broker, producer, consumer, and topic-level metrics.

**Critical Broker Metrics:**

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| UnderReplicatedPartitions | Partitions not fully replicated | > 0 |
| OfflinePartitionsCount | Partitions without leader | > 0 |
| ActiveControllerCount | Should be 1 per cluster | != 1 |
| RequestHandlerAvgIdlePercent | Handler thread utilization | < 0.3 (70% utilized) |

**Broker Performance Metrics:**

| Metric | Description | Healthy Range |
|--------|-------------|---------------|
| MessagesInPerSec | Incoming message rate | Depends on workload |
| BytesInPerSec | Incoming byte rate | < network capacity |
| BytesOutPerSec | Outgoing byte rate | < network capacity |
| RequestsPerSec | Request rate by type | Stable baseline |
| TotalTimeMs | Request latency (produce, fetch) | < 100ms |

**Producer Metrics:**

| Metric | Description | What to Watch |
|--------|-------------|---------------|
| record-send-rate | Messages sent per second | Throughput |
| record-error-rate | Failed sends per second | Should be ~0 |
| request-latency-avg | Average request time | Latency SLA |
| batch-size-avg | Average batch size | Batching efficiency |
| buffer-available-bytes | Free buffer memory | Not approaching 0 |
| record-queue-time-avg | Time in buffer | Backpressure indicator |

**Consumer Metrics:**

| Metric | Description | What to Watch |
|--------|-------------|---------------|
| records-lag | Messages behind per partition | Growing lag |
| records-lag-max | Maximum lag across partitions | Alert threshold |
| records-consumed-rate | Consumption throughput | Match production rate |
| fetch-latency-avg | Fetch request latency | Network/broker issues |
| commit-latency-avg | Offset commit latency | Commit performance |
| rebalance-latency-avg | Time spent in rebalance | Stability |

**Consumer Group Lag Monitoring:**

```bash
# CLI check
kafka-consumer-groups.sh --describe --group my-group

# Key columns:
# - CURRENT-OFFSET: Consumer position
# - LOG-END-OFFSET: Latest message
# - LAG: Difference (messages behind)

# JMX metric path
kafka.consumer:type=consumer-fetch-manager-metrics,
    client-id=*,topic=*,partition=*
    records-lag
```

**Topic/Partition Metrics:**

| Metric | Description |
|--------|-------------|
| Size | Disk usage per partition |
| MessageCount | Messages in partition |
| LogStartOffset | Earliest available offset |
| LogEndOffset | Latest offset |

**ZooKeeper Metrics (if applicable):**

| Metric | Description | Alert |
|--------|-------------|-------|
| AvgRequestLatency | ZK request latency | > 100ms |
| OutstandingRequests | Pending requests | Growing |
| NumAliveConnections | Connected clients | Dropping |

**Monitoring Stack:**

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Kafka JMX     │ → │   Prometheus    │ → │    Grafana      │
│   Metrics       │    │   (scraping)    │    │   (dashboards)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘

Alternative: Kafka → JMX Exporter → Prometheus → Grafana
Alternative: Kafka → Datadog Agent → Datadog
```

**Key Alerts to Configure:**

| Alert | Condition | Severity |
|-------|-----------|----------|
| Under-replicated partitions | > 0 for 5 min | Critical |
| Offline partitions | > 0 | Critical |
| Consumer lag | > threshold | Warning/Critical |
| Broker down | ActiveControllerCount != 1 | Critical |
| High request latency | p99 > SLA | Warning |
| Disk usage | > 80% | Warning |
| Producer errors | error-rate > 0 | Warning |

**Dashboard Essentials:**

| Panel | Metrics |
|-------|---------|
| Cluster Health | Active controllers, under-replicated, offline |
| Throughput | Messages/bytes in/out per broker |
| Latency | Produce/fetch request latency p50, p99 |
| Consumer Lag | Lag by consumer group and topic |
| Resources | CPU, memory, disk, network per broker |

---

Good luck with your interview! 🚀

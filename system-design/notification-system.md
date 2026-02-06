# Notification System Design

## Table of Contents

### Section 1: Fundamentals & Requirements
- [Q1: What are the core components of a notification system?](#q1)
- [Q2: What are the different types of notifications and their characteristics?](#q2)
- [Q3: How do you gather functional and non-functional requirements?](#q3)
- [Q4: What are the key metrics to consider?](#q4)
- [Q5: Push vs Pull model - when to use which?](#q5)

### Section 2: High-Level Architecture
- [Q6: How would you design the high-level architecture?](#q6)
- [Q7: How do you handle multiple notification channels?](#q7)
- [Q8: What role do message queues play in the architecture?](#q8)
- [Q9: How do you design the notification service API?](#q9)
- [Q10: How do you handle user preferences and subscription management?](#q10)

### Section 3: Scalability & Performance
- [Q11: How do you scale to millions of users?](#q11)
- [Q12: How do you handle traffic spikes?](#q12)
- [Q13: What caching strategies would you use?](#q13)
- [Q14: How do you implement rate limiting?](#q14)
- [Q15: How do you optimize for low latency delivery?](#q15)

### Section 4: Reliability & Delivery Guarantees
- [Q16: How do you ensure at-least-once delivery?](#q16)
- [Q17: How do you handle failed deliveries and retries?](#q17)
- [Q18: What is a dead letter queue and when do you use it?](#q18)
- [Q19: How do you prevent duplicate notifications?](#q19)
- [Q20: How do you design for high availability?](#q20)

### Section 5: Data Storage & Analytics
- [Q21: What database schema would you use?](#q21)
- [Q22: How do you track delivery status and analytics?](#q22)
- [Q23: How do you handle notification history and archival?](#q23)

### Section 6: Advanced Topics
- [Q24: How do you implement priority queues for urgent notifications?](#q24)
- [Q25: How do you handle cross-region notification delivery?](#q25)

---

## Section 1: Fundamentals & Requirements

<a id="q1"></a>
### Q1: What are the core components of a notification system?

**Answer:**

A notification system consists of several key components working together:

```
┌─────────────────────────────────────────────────────────────────┐
│                      NOTIFICATION SYSTEM                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐   ┌──────────┐   ┌─────────────────────────┐      │
│  │ Clients  │──▶│   API    │──▶│   Notification Service  │      │
│  │(Services)│   │ Gateway  │   │   - Validation          │      │
│  └──────────┘   └──────────┘   │   - Rate Limiting       │      │
│                                │   - Template Processing │      │
│                                └────────────┬────────────┘      │
│                                             │                   │
│                                             ▼                   │
│  ┌──────────┐   ┌──────────┐   ┌─────────────────────────┐      │
│  │   User   │◀─▶│Preference│◀─▶│      Message Queue      │      │
│  │ Database │   │ Service  │   │     (Kafka/RabbitMQ)    │      │
│  └──────────┘   └──────────┘   └────────────┬────────────┘      │
│                                             │                   │
│                  ┌──────────────────────────┼─────────────┐     │
│                  │                          │             │     │
│                  ▼                          ▼             ▼     │
│       ┌──────────────┐           ┌─────────────┐    ┌───────┐   │
│       │ Push Service │           │ SMS Service │    │ Email │   │
│       │  (APNs/FCM)  │           │  (Twilio)   │    │Service│   │
│       └──────────────┘           └─────────────┘    └───────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Core Components:**

| Component | Responsibility |
|-----------|----------------|
| **API Gateway** | Authentication, rate limiting, request routing |
| **Notification Service** | Core logic, validation, template rendering, routing |
| **Message Queue** | Decoupling, buffering, reliable delivery |
| **User/Device Store** | User preferences, device tokens, contact info |
| **Channel Workers** | Push, SMS, Email handlers with provider integrations |
| **Analytics Service** | Delivery tracking, metrics, reporting |

---

<a id="q2"></a>
### Q2: What are the different types of notifications and their characteristics?

**Answer:**

**By Channel:**

| Channel | Latency | Cost | Reach | Rich Content | Use Case |
|---------|---------|------|-------|--------------|----------|
| **Push** | Low (ms) | Free | App users only | Limited | Real-time alerts (ride arriving) |
| **SMS** | Medium (s) | High | Universal | Text only | OTP, critical alerts |
| **Email** | High (min) | Low | Universal | Rich HTML | Receipts, marketing |
| **In-App** | Instant | Free | Active users | Rich | Feature announcements |
| **WebSocket** | Instant | Free | Connected users | Any | Live updates |

**By Priority:**

| Type | Examples | SLA | Handling |
|------|----------|-----|----------|
| **Critical** | OTP, payment failure, security alerts | < 1 second | Dedicated high-priority queue |
| **Transactional** | Order confirmation, ride updates | < 30 seconds | Standard processing |
| **Marketing** | Promotions, recommendations | Minutes to hours | Batch processing, rate limited |

**By Trigger:**

- **Event-driven**: Triggered by user actions (order placed, ride completed)
- **Scheduled**: Sent at specific times (daily digest, appointment reminders)
- **Batch**: Bulk campaigns to user segments

---

<a id="q3"></a>
### Q3: How do you gather functional and non-functional requirements?

**Answer:**

**Functional Requirements:**

1. **Multi-channel delivery**: Support push, SMS, email, in-app notifications
2. **User preferences**: Allow users to opt-in/out per channel and notification type
3. **Template management**: Support dynamic content with personalization
4. **Scheduling**: Send immediately or at scheduled times
5. **Targeting**: Send to individual users, segments, or broadcast
6. **Tracking**: Track delivery status (sent, delivered, read, failed)

**Non-Functional Requirements:**

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| **Throughput** | 1M+ notifications/minute | Peak traffic during campaigns |
| **Latency** | < 100ms for critical, < 5s for standard | User experience |
| **Availability** | 99.99% | Critical business function |
| **Durability** | No message loss | Financial/legal implications |
| **Scalability** | 10x current load | Business growth |

**Capacity Estimation (example for Grab-scale):**

```
Users: 100M registered users
DAU: 30M daily active users
Notifications per user per day: 10 average

Daily notifications: 30M × 10 = 300M/day
Peak QPS: 300M / 86400 × 10 (peak factor) ≈ 35,000/second

Storage (30 days retention):
- 300M × 30 days × 1KB avg = 9TB
```

---

<a id="q4"></a>
### Q4: What are the key metrics to consider?

**Answer:**

**Delivery Metrics:**

| Metric | Definition | Target |
|--------|------------|--------|
| **Delivery Rate** | Successfully delivered / Total sent | > 99% |
| **Bounce Rate** | Failed deliveries / Total sent | < 1% |
| **Latency P50/P99** | Time from request to delivery | P50 < 100ms, P99 < 1s |
| **Throughput** | Notifications processed per second | Based on capacity |

**Engagement Metrics:**

| Metric | Definition | Purpose |
|--------|------------|---------|
| **Open Rate** | Opened / Delivered | Measure relevance |
| **Click-through Rate** | Clicked / Opened | Measure effectiveness |
| **Opt-out Rate** | Unsubscribes / Sent | Monitor user fatigue |
| **Conversion Rate** | Actions taken / Sent | Business impact |

**System Health Metrics:**

| Metric | What to Monitor |
|--------|-----------------|
| **Queue Depth** | Backlog size, processing lag |
| **Error Rate** | 4xx, 5xx responses, provider errors |
| **Resource Usage** | CPU, memory, connections per service |
| **Provider Status** | APNs, FCM, Twilio availability |

**Alerting Thresholds:**

- Queue depth > 100K messages: Warning
- Delivery rate < 95%: Critical
- Latency P99 > 5s: Warning
- Provider error rate > 5%: Critical

---

<a id="q5"></a>
### Q5: Push vs Pull model - when to use which?

**Answer:**

**Push Model:**
```
Server ──────▶ Client
(Server initiates delivery)
```

**Pull Model:**
```
Client ──────▶ Server
(Client requests updates)
```

**Comparison:**

| Aspect | Push | Pull |
|--------|------|------|
| **Latency** | Real-time | Depends on polling interval |
| **Resource Usage** | Efficient (send when needed) | Wasteful (frequent empty polls) |
| **Complexity** | Higher (manage connections) | Lower (stateless) |
| **Scalability** | Connection management overhead | Easier to scale |
| **Offline Handling** | Requires queuing | Natural (client pulls when ready) |
| **Battery Impact** | Lower (wake on message) | Higher (constant polling) |

**When to Use Push:**
- Real-time requirements (ride updates, chat messages)
- Time-sensitive alerts (OTP, security)
- Mobile notifications (APNs/FCM are push-based)

**When to Use Pull:**
- Non-urgent updates (notification history)
- Web dashboards with refresh
- Batch data synchronization

**Hybrid Approach (Common in Practice):**
```
┌───────────────────────────────────────────────────┐
│                                                   │
│  Push: "You have new notifications"               │
│            │                                      │
│            ▼                                      │
│  Pull: Client fetches full notification content   │
│                                                   │
└───────────────────────────────────────────────────┘
```

This reduces push payload size while ensuring real-time awareness.

---

## Section 2: High-Level Architecture

<a id="q6"></a>
### Q6: How would you design the high-level architecture?

**Answer:**

**System Architecture:**

```
                               ┌─────────────────┐
                               │    Internal     │
                               │    Services     │
                               │ (Orders, Rides) │
                               └────────┬────────┘
                                        │
                                        ▼
┌──────────────────────────────────────────────────────────────┐
│                         API GATEWAY                          │
│                 (Auth, Rate Limiting, Routing)               │
└───────────────────────────────┬──────────────────────────────┘
                                │
                                ▼
┌──────────────────────────────────────────────────────────────┐
│                    NOTIFICATION SERVICE                      │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐  │
│  │ Validator │  │ Template  │  │  Router   │  │ Scheduler │  │
│  │           │  │  Engine   │  │           │  │           │  │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘  │
└───────────────────────────────┬──────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        │                       │                       │
        ▼                       ▼                       ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│ High Priority │       │Standard Queue │       │  Batch Queue  │
│     Queue     │       │               │       │               │
└───────┬───────┘       └───────┬───────┘       └───────┬───────┘
        │                       │                       │
        ▼                       ▼                       ▼
┌──────────────────────────────────────────────────────────────┐
│                         WORKER POOL                          │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐  │
│  │Push Worker│  │SMS Worker │  │Email      │  │In-App     │  │
│  │           │  │           │  │Worker     │  │Worker     │  │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  │
└────────┼──────────────┼──────────────┼──────────────┼────────┘
         │              │              │              │
         ▼              ▼              ▼              ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ APNs/FCM │   │  Twilio  │   │ SendGrid │   │ WebSocket│
   └──────────┘   └──────────┘   └──────────┘   └──────────┘

┌──────────────────────────────────────────────────────────────┐
│                         DATA STORES                          │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐  │
│  │User/Device│  │   Notif   │  │ Template  │  │ Analytics │  │
│  │   Store   │  │  History  │  │   Store   │  │   Store   │  │
│  │ Postgres  │  │ Cassandra │  │   Redis   │  │ ClickHouse│  │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘  │
└──────────────────────────────────────────────────────────────┘
```                                                             

**Component Responsibilities:**

| Layer | Component | Responsibility |
|-------|-----------|----------------|
| **Ingestion** | API Gateway | Auth, rate limiting, request validation |
| **Processing** | Notification Service | Template rendering, routing decisions |
| **Queuing** | Message Queues | Buffering, priority handling, durability |
| **Delivery** | Channel Workers | Provider-specific delivery logic |
| **Storage** | Data Stores | User data, history, templates, analytics |

---

<a id="q7"></a>
### Q7: How do you handle multiple notification channels?

**Answer:**

**Channel Abstraction Pattern:**

```
┌────────────────────────────────────────────────────────┐
│                 NOTIFICATION REQUEST                   │
│  {                                                     │
│    user_id: "123",                                     │
│    type: "RIDE_ARRIVING",                              │
│    channels: ["push", "sms"],  // determined by prefs  │
│    data: { driver: "John", eta: "2 min" }              │
│  }                                                     │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────┐
│                    CHANNEL ROUTER                      │
│                                                        │
│  1. Fetch user preferences                             │
│  2. Filter enabled channels                            │
│  3. Apply channel-specific rules                       │
│  4. Fan-out to channel queues                          │
└──────────────────────────┬─────────────────────────────┘
                           │
       ┌───────────────────┼───────────────────┐
       ▼                   ▼                   ▼
 ┌───────────┐       ┌───────────┐       ┌───────────┐
 │Push Queue │       │ SMS Queue │       │Email Queue│
 └─────┬─────┘       └─────┬─────┘       └─────┬─────┘
       │                   │                   │
       ▼                   ▼                   ▼
 ┌───────────┐       ┌───────────┐       ┌───────────┐
 │Push Worker│       │SMS Worker │       │Email      │
 │           │       │           │       │Worker     │
 │ - Format  │       │ - Format  │       │ - Format  │
 │ - APNs    │       │ - Twilio  │       │ - SMTP    │
 │ - FCM     │       │ - Nexmo   │       │ - SendGrid│
 └───────────┘       └───────────┘       └───────────┘
```

**Channel Selection Logic:**

| Factor | Consideration |
|--------|---------------|
| **User Preference** | Respect opt-in/opt-out settings |
| **Notification Type** | OTP → SMS only, Promo → Email preferred |
| **Urgency** | Critical → All enabled channels |
| **Cost** | SMS expensive, prefer push when possible |
| **Reachability** | No device token? Fall back to SMS/Email |

**Fallback Strategy:**

```
Primary: Push Notification
    │
    ├── Success → Done
    │
    └── Failed (no token, disabled)
            │
            ▼
        Secondary: SMS
            │
            ├── Success → Done
            │
            └── Failed (no phone)
                    │
                    ▼
                Tertiary: Email
```

---

<a id="q8"></a>
### Q8: What role do message queues play in the architecture?

**Answer:**

**Why Message Queues Are Essential:**

```
WITHOUT QUEUE:                    WITH QUEUE:

Service ──────▶ Provider          Service ──▶ Queue ──▶ Worker ──▶ Provider

Problems:                         Benefits:
- Tight coupling                  - Decoupling
- No buffering                    - Buffering spikes
- Blocking on slow provider       - Async processing
- Lost messages on failure        - Durability
- Hard to scale                   - Independent scaling
```

**Queue Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                   MESSAGE QUEUE CLUSTER                     │
│                      (Kafka/RabbitMQ)                       │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Topic: notifications.high-priority                     │ │
│  │ Partitions: 16 | Replication: 3 | Retention: 7 days    │ │
│  │ [OTP, Security Alerts, Payment Failures]               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Topic: notifications.standard                          │ │
│  │ Partitions: 32 | Replication: 3 | Retention: 7 days    │ │
│  │ [Order Updates, Ride Status, Receipts]                 │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Topic: notifications.batch                             │ │
│  │ Partitions: 8 | Replication: 3 | Retention: 3 days     │ │
│  │ [Marketing, Recommendations, Digests]                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Topic: notifications.dlq (Dead Letter Queue)           │ │
│  │ [Failed messages for investigation]                    │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**Kafka vs RabbitMQ:**

| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| **Throughput** | Very high (millions/sec) | High (tens of thousands/sec) |
| **Ordering** | Per partition | Per queue |
| **Retention** | Configurable (days/weeks) | Until consumed |
| **Replay** | Yes (offset-based) | No |
| **Use Case** | Event streaming, high volume | Task queues, complex routing |

**For Notification System:** Kafka is preferred for high-throughput scenarios with replay capability for debugging.

---

<a id="q9"></a>
### Q9: How do you design the notification service API?

**Answer:**

**API Endpoints:**

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/notifications` | POST | Send a notification |
| `/v1/notifications/batch` | POST | Send batch notifications |
| `/v1/notifications/{id}` | GET | Get notification status |
| `/v1/notifications/{id}/cancel` | POST | Cancel scheduled notification |
| `/v1/users/{id}/preferences` | GET/PUT | Manage user preferences |
| `/v1/templates` | CRUD | Manage notification templates |

**Send Notification Request:**

```json
// POST /v1/notifications
{
  "recipient": {
    "user_id": "user_123"
    // OR for batch: "segment_id": "active_users"
  },
  "notification": {
    "type": "RIDE_ARRIVING",
    "template_id": "ride_eta_v2",
    "data": {
      "driver_name": "John",
      "eta_minutes": 2,
      "vehicle": "Toyota Camry - ABC 123"
    }
  },
  "channels": ["push", "sms"],
  "options": {
    "priority": "high",
    "scheduled_at": null,
    "ttl_seconds": 300,
    "idempotency_key": "ride_123_arriving"
  }
}
```

**Response:**

```json
{
  "notification_id": "notif_abc123",
  "status": "queued",
  "channels": {
    "push": { "status": "queued", "device_count": 2 },
    "sms": { "status": "queued", "phone": "+84***456" }
  },
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Status Webhook (for async tracking):**

```json
// POST {callback_url}
{
  "notification_id": "notif_abc123",
  "channel": "push",
  "status": "delivered",
  "delivered_at": "2024-01-15T10:30:02Z",
  "metadata": {
    "device_id": "device_xyz",
    "provider": "fcm"
  }
}
```

---

<a id="q10"></a>
### Q10: How do you handle user preferences and subscription management?

**Answer:**

**Preference Data Model:**

```
┌─────────────────────────────────────────────────────────────┐
│                     USER PREFERENCES                        │
├─────────────────────────────────────────────────────────────┤
│ user_id: "user_123"                                         │
│                                                             │
│ global_settings:                                            │
│   quiet_hours: { start: "22:00", end: "08:00", tz: "+7" }   │
│   language: "vi"                                            │
│                                                             │
│ channel_settings:                                           │
│   push:  { enabled: true, devices: ["dev_1", "dev_2"] }     │
│   sms:   { enabled: true, phone: "+84123456789" }           │
│   email: { enabled: true, address: "user@example.com" }     │
│                                                             │
│ notification_types:                                         │
│   RIDE_UPDATES:   { push: true,  sms: false, email: false } │
│   PAYMENT_ALERTS: { push: true,  sms: true,  email: true  } │
│   PROMOTIONS:     { push: true,  sms: false, email: true  } │
│   ORDER_UPDATES:  { push: true,  sms: false, email: true  } │
└─────────────────────────────────────────────────────────────┘
```

**Preference Resolution Flow:**

```
Notification Request
        │
        ▼
┌───────────────────┐
│ Fetch User Prefs  │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐     ┌────────────────────────────────┐
│ Check Global      │────▶│ Quiet Hours? → Queue for later │
│ Settings          │     │ Unsubscribed? → Skip entirely  │
└─────────┬─────────┘     └────────────────────────────────┘
          │
          ▼
┌───────────────────┐
│ Check Notification│────▶ Which channels enabled for this type?
│ Type Settings     │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ Check Channel     │────▶ Is channel globally enabled?
│ Settings          │      Valid contact info available?
└─────────┬─────────┘
          │
          ▼
    Final Channel List
```

**Preference Caching Strategy:**

| Store | Purpose | TTL |
|-------|---------|-----|
| **Redis** | Hot cache for active users | 1 hour |
| **PostgreSQL** | Source of truth | Permanent |
| **Local Cache** | Worker-level cache | 5 minutes |

**Cache Invalidation:**
- User updates preferences → Publish event → Invalidate cache
- Use cache-aside pattern with version stamps

---

## Section 3: Scalability & Performance

<a id="q11"></a>
### Q11: How do you scale to millions of users?

**Answer:**

**Scaling Strategies:**

```
┌─────────────────────────────────────────────────────────────┐
│                    HORIZONTAL SCALING                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  API Layer:        Load Balancer                            │
│                   /     |     \                             │
│                API-1  API-2  API-3  ... API-N               │
│                (Stateless - scale infinitely)               │
│                                                             │
│  Queue Layer:      Kafka Cluster                            │
│                   Partition 0 ──▶ Consumer Group A          │
│                   Partition 1 ──▶ Consumer Group A          │
│                   Partition N ──▶ Consumer Group A          │
│                (Add partitions for parallelism)             │
│                                                             │
│  Worker Layer:                                              │
│                Worker-1  Worker-2  Worker-3  ... Worker-N   │
│                (Scale based on queue depth)                 │
│                                                             │
│  Data Layer:                                                │
│                ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│                │ Shard 0 │  │ Shard 1 │  │ Shard 2 │        │
│                │ Users   │  │ Users   │  │ Users   │        │
│                │  A-H    │  │  I-P    │  │  Q-Z    │        │
│                └─────────┘  └─────────┘  └─────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Scaling by Component:**

| Component | Scaling Strategy | Trigger |
|-----------|------------------|---------|
| **API Servers** | Horizontal (add instances) | CPU > 70%, latency > threshold |
| **Message Queue** | Add partitions, brokers | Lag > threshold |
| **Workers** | Horizontal (auto-scale) | Queue depth, processing time |
| **Database** | Read replicas, sharding | Query latency, connections |
| **Cache** | Redis Cluster | Memory usage, hit rate |

**Partitioning Strategy for Kafka:**

```
Partition Key Options:
├── user_id: Even distribution, user-level ordering
├── notification_type: Group similar notifications
└── channel: Separate queues per channel

Recommended: user_id % num_partitions
- Ensures all notifications for a user go to same partition
- Maintains ordering per user
- Even distribution across partitions
```

**Database Sharding:**

| Strategy | Pros | Cons |
|----------|------|------|
| **By User ID** | Even distribution, simple routing | Cross-user queries hard |
| **By Region** | Data locality, compliance | Uneven load |
| **By Time** | Easy archival | Hot partition for recent data |

---

<a id="q12"></a>
### Q12: How do you handle traffic spikes?

**Answer:**

**Traffic Spike Scenarios:**

| Scenario | Example | Scale |
|----------|---------|-------|
| **Marketing Campaign** | Flash sale announcement | 10M notifications in minutes |
| **System Event** | App update prompt | All users simultaneously |
| **Regional Event** | Weather alert | Millions in one region |
| **Viral Moment** | Trending promotion | Unpredictable spike |

**Handling Strategies:**

```
┌─────────────────────────────────────────────────────────────┐
│                  TRAFFIC SPIKE HANDLING                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. ADMISSION CONTROL                                       │
│     ┌─────────────────────────────────────────────────┐     │
│     │ Rate Limiter (per client, global)               │     │
│     │ - Reject excess with 429                        │     │
│     │ - Return Retry-After header                     │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  2. QUEUE BUFFERING                                         │
│     ┌─────────────────────────────────────────────────┐     │
│     │ Message Queue absorbs spike                     │     │
│     │ - Kafka can handle millions/sec                 │     │
│     │ - Consumers process at sustainable rate         │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  3. BACKPRESSURE                                            │
│     ┌─────────────────────────────────────────────────┐     │
│     │ Monitor queue depth                             │     │
│     │ - Slow down producers if lag too high           │     │
│     │ - Reject low-priority notifications             │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
│  4. AUTO-SCALING                                            │
│     ┌─────────────────────────────────────────────────┐     │
│     │ Scale workers based on queue depth              │     │
│     │ - Pre-warm for known events                     │     │
│     │ - Scale down after spike subsides               │     │
│     └─────────────────────────────────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Campaign Scheduling Best Practice:**

```
Instead of:                    Do this:

Send 10M at once               Batch over time
     │                              │
     ▼                              ▼
┌─────────┐                   ┌─────────┐
│  SPIKE  │                   │ Batch 1 │ (1M users, T+0)
│ 10M/min │                   │ Batch 2 │ (1M users, T+5min)
└─────────┘                   │ Batch 3 │ (1M users, T+10min)
                              │   ...   │
                              │Batch 10 │ (1M users, T+45min)
                              └─────────┘
```

**Provider Rate Limits:**

| Provider | Rate Limit | Strategy |
|----------|------------|----------|
| **APNs** | No hard limit, but throttles | Token-based batching |
| **FCM** | 1000 msg/sec per project | Multiple projects, queuing |
| **Twilio SMS** | Varies by number type | Number pooling, queuing |
| **SendGrid** | Varies by plan | Dedicated IPs, warming |

---

<a id="q13"></a>
### Q13: What caching strategies would you use?

**Answer:**

**What to Cache:**

| Data | Cache Location | TTL | Invalidation |
|------|----------------|-----|--------------|
| **User Preferences** | Redis + Local | 1h / 5m | On update event |
| **Device Tokens** | Redis | 24h | On token refresh |
| **Templates** | Local + Redis | 1h | On template update |
| **Rate Limit Counters** | Redis | Window size | Auto-expire |
| **User Segments** | Redis | 15m | On segment update |

**Caching Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                   MULTI-LAYER CACHING                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Request ──▶ L1: Local Cache (in-memory)                    │
│                   │                                         │
│                   │ Miss                                    │
│                   ▼                                         │
│             L2: Redis Cluster                               │
│                   │                                         │
│                   │ Miss                                    │
│                   ▼                                         │
│             L3: Database                                    │
│                   │                                         │
│                   │ Populate cache on read                  │
│                   ▼                                         │
│             Return + Cache at L1 & L2                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Cache-Aside Pattern (Read):**

```
1. Check cache for user preferences
2. If HIT → Return cached data
3. If MISS:
   a. Query database
   b. Store in cache with TTL
   c. Return data
```

**Write-Through Pattern (Update):**

```
1. Update database
2. Update cache (or invalidate)
3. Publish invalidation event for other instances
```

**Cache Warming:**

```
For scheduled campaigns:
1. Query target user segment
2. Pre-load user preferences into cache
3. Pre-load device tokens into cache
4. Execute campaign with warm cache
```

---

<a id="q14"></a>
### Q14: How do you implement rate limiting?

**Answer:**

**Rate Limiting Layers:**

```
┌─────────────────────────────────────────────────────────────┐
│                   RATE LIMITING LAYERS                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Layer 1: API Gateway                                       │
│  ├── Per-client rate limit (e.g., 1000 req/sec per service) │
│  └── Global rate limit (protect system)                     │
│                                                             │
│  Layer 2: Per-User Rate Limit                               │
│  ├── Max notifications per user per hour                    │
│  └── Prevent notification spam to single user               │
│                                                             │
│  Layer 3: Per-Channel Rate Limit                            │
│  ├── SMS: Max 5/hour per user (cost control)                │
│  └── Push: Max 20/hour per user (prevent fatigue)           │
│                                                             │
│  Layer 4: Provider Rate Limit                               │
│  ├── Respect FCM/APNs/Twilio limits                         │
│  └── Queue excess, retry with backoff                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Rate Limiting Algorithms:**

| Algorithm | Description | Use Case |
|-----------|-------------|----------|
| **Token Bucket** | Tokens added at fixed rate, consumed per request | Smooth traffic with burst allowance |
| **Sliding Window** | Count requests in rolling time window | Precise rate limiting |
| **Fixed Window** | Count requests in fixed time buckets | Simple, some edge cases |
| **Leaky Bucket** | Process at fixed rate, queue excess | Smooth output rate |

**Token Bucket Implementation (Conceptual):**

```
┌───────────────────────────────────────┐
│           TOKEN BUCKET                │
├───────────────────────────────────────┤
│  Capacity: 100 tokens                 │
│  Refill Rate: 10 tokens/second        │
│  Current Tokens: 45                   │
├───────────────────────────────────────┤
│                                       │
│  Request arrives:                     │
│  1. Calculate tokens since last req   │
│  2. Add refilled tokens (cap at max)  │
│  3. If tokens >= 1: Allow, decrement  │
│  4. If tokens < 1: Reject (429)       │
│                                       │
└───────────────────────────────────────┘
```

**Redis-based Distributed Rate Limiting:**

```
Key: rate_limit:{client_id}:{window}
Value: request_count
TTL: window_size

Atomic operation:
1. INCR key
2. If count == 1: SET TTL
3. If count > limit: REJECT
```

---

<a id="q15"></a>
### Q15: How do you optimize for low latency delivery?

**Answer:**

**Latency Breakdown:**

```
Total Latency = API Processing + Queue Wait + Worker Processing + Provider Delivery

┌─────────────────────────────────────────────────────────────┐
│                     LATENCY BUDGET                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  API Processing:      10-20ms                               │
│  ├── Auth/Validation: 2ms                                   │
│  ├── Preference Fetch: 5ms (cached)                         │
│  └── Queue Produce:   5ms                                   │
│                                                             │
│  Queue Wait:          0-100ms (depends on load)             │
│                                                             │
│  Worker Processing:   10-30ms                               │
│  ├── Dequeue:         5ms                                   │
│  ├── Template Render: 5ms                                   │
│  └── Provider Call:   10-20ms                               │
│                                                             │
│  Provider Delivery:   50-500ms (external, varies)           │
│                                                             │
│  TOTAL:               70-650ms typical                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Optimization Strategies:**

| Area | Optimization | Impact |
|------|--------------|--------|
| **API Layer** | Connection pooling, async I/O | -10ms |
| **Caching** | Cache preferences, templates locally | -20ms |
| **Queue** | Separate high-priority queue | -50ms for critical |
| **Workers** | Pre-warmed connections to providers | -15ms |
| **Batching** | Batch multiple notifications per API call | Higher throughput |
| **Geography** | Deploy workers near provider endpoints | -30ms |

**Priority Queue Implementation:**

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  High Priority Queue ────▶ Dedicated Workers (always on)    │
│  (OTP, Security)           Low latency, high resources      │
│                                                             │
│  Standard Queue ─────────▶ Auto-scaled Workers              │
│  (Transactional)           Balance latency/cost             │
│                                                             │
│  Batch Queue ────────────▶ Batch Workers                    │
│  (Marketing)               Optimize throughput, not latency │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Connection Optimization:**

```
Bad: Create connection per request
     Request → New Connection → Provider → Close
     (100-200ms overhead)

Good: Connection pooling
     Request → Pool.Get() → Provider → Pool.Return()
     (5-10ms overhead)

Best: Persistent connections with multiplexing
     HTTP/2 to FCM, persistent SMTP connections
```

---

## Section 4: Reliability & Delivery Guarantees

<a id="q16"></a>
### Q16: How do you ensure at-least-once delivery?

**Answer:**

**Delivery Guarantees Comparison:**

| Guarantee | Description | Trade-off |
|-----------|-------------|-----------|
| **At-most-once** | Fire and forget, may lose messages | Fastest, no duplicates |
| **At-least-once** | Retry until confirmed, may duplicate | Reliable, possible duplicates |
| **Exactly-once** | Deliver exactly once | Complex, expensive |

**At-Least-Once Implementation:**

```
┌──────────────────────────────────────────────────────────────┐
│               AT-LEAST-ONCE DELIVERY FLOW                    │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Producer writes to queue with acknowledgment             │
│     ┌──────────┐        ┌──────────┐                         │
│     │ Producer │──MSG──▶│  Queue   │──ACK──▶ Producer        │
│     └──────────┘        └──────────┘                         │
│     (Retry if no ACK)                                        │
│                                                              │
│  2. Consumer processes with manual commit                    │
│     ┌──────────┐        ┌──────────┐        ┌──────────┐     │
│     │  Queue   │──MSG──▶│ Consumer │──SEND─▶│ Provider │     │
│     └──────────┘        └──────────┘        └──────────┘     │
│           ▲                   │                   │          │
│           │                   │                   │          │
│           └───────COMMIT──────┘◀──────ACK────────┘           │
│     (Only commit after provider confirms)                    │
│                                                              │
│  3. If consumer crashes before commit                        │
│     → Message redelivered to another consumer                │
│     → May result in duplicate delivery                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**Key Implementation Points:**

| Component | Strategy |
|-----------|----------|
| **Producer** | Wait for queue ACK before returning success |
| **Queue** | Replicated storage (Kafka replication factor 3) |
| **Consumer** | Manual offset commit after successful processing |
| **Provider** | Verify delivery confirmation before commit |

**Kafka Consumer Configuration:**

```
enable.auto.commit = false  // Manual commit only
acks = all                  // Wait for all replicas
retries = MAX_INT           // Retry forever on failure
```

---

<a id="q17"></a>
### Q17: How do you handle failed deliveries and retries?

**Answer:**

**Retry Strategy:**

```
┌─────────────────────────────────────────────────────────────┐
│                       RETRY FLOW                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Attempt 1 ──▶ Failed                                       │
│                  │                                          │
│                  ▼                                          │
│  Wait 1 second ──▶ Attempt 2 ──▶ Failed                     │
│                                    │                        │
│                                    ▼                        │
│  Wait 2 seconds ──▶ Attempt 3 ──▶ Failed                    │
│                                    │                        │
│                                    ▼                        │
│  Wait 4 seconds ──▶ Attempt 4 ──▶ Failed                    │
│                                    │                        │
│                                    ▼                        │
│  Wait 8 seconds ──▶ Attempt 5 ──▶ Failed                    │
│                                    │                        │
│                                    ▼                        │
│                    Move to Dead Letter Queue                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Exponential Backoff with Jitter:**

```
delay = min(base * 2^attempt + random_jitter, max_delay)

Example:
- Base: 1 second
- Max: 60 seconds
- Jitter: 0-500ms random

Attempt 1: 1s + jitter
Attempt 2: 2s + jitter
Attempt 3: 4s + jitter
Attempt 4: 8s + jitter
Attempt 5: 16s + jitter
...
Attempt N: 60s + jitter (capped)
```

**Error Classification:**

| Error Type | Action | Examples |
|------------|--------|----------|
| **Transient** | Retry with backoff | Timeout, 503, rate limit |
| **Permanent** | Move to DLQ | Invalid token, unsubscribed |
| **Partial** | Retry subset | Batch with some failures |

**Retry Queue Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Main Queue ──▶ Worker ──▶ Provider                         │
│                   │                                         │
│                   │ Failed (transient)                      │
│                   ▼                                         │
│  Retry Queue (with delay) ──▶ Worker ──▶ Provider           │
│                   │                                         │
│                   │ Failed (max retries)                    │
│                   ▼                                         │
│  Dead Letter Queue ──▶ Manual Investigation / Alerting      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

<a id="q18"></a>
### Q18: What is a dead letter queue and when do you use it?

**Answer:**

**Dead Letter Queue (DLQ) Definition:**

A DLQ is a separate queue that stores messages that could not be processed successfully after all retry attempts.

```
┌─────────────────────────────────────────────────────────────┐
│                  DEAD LETTER QUEUE FLOW                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Main Queue                                                 │
│      │                                                      │
│      ▼                                                      │
│  ┌───────────────────────────────────────────────────┐      │
│  │ Consumer attempts processing                      │      │
│  │                                                   │      │
│  │  ┌─────────────────────────────────────────────┐  │      │
│  │  │ Retry Logic                                 │  │      │
│  │  │ - Max retries: 5                            │  │      │
│  │  │ - Backoff: exponential                      │  │      │
│  │  │ - Timeout: 30s per attempt                  │  │      │
│  │  └─────────────────────────────────────────────┘  │      │
│  │                                                   │      │
│  │  If all retries exhausted:                        │      │
│  │  → Move to DLQ with metadata                      │      │
│  └───────────────────────────────────────────────────┘      │
│                                                             │
│  Dead Letter Queue                                          │
│      │                                                      │
│      ├──▶ Alert on-call engineer                            │
│      ├──▶ Store for investigation                           │
│      └──▶ Manual replay after fix                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**DLQ Message Contents:**

```json
{
  "original_message": { "..." : "..." },
  "metadata": {
    "original_queue": "notifications.push",
    "first_attempt": "2024-01-15T10:00:00Z",
    "last_attempt": "2024-01-15T10:05:00Z",
    "attempt_count": 5,
    "last_error": "FCM: InvalidRegistration",
    "error_code": "PERMANENT_FAILURE"
  }
}
```

**When Messages Go to DLQ:**

| Reason | Example | Action |
|--------|---------|--------|
| **Invalid data** | Malformed notification payload | Fix producer, discard |
| **Invalid recipient** | Expired device token | Update user record, discard |
| **Persistent failure** | Provider consistently fails | Investigate provider |
| **Poison message** | Causes consumer crash | Debug and fix consumer |

**DLQ Operations:**

| Operation | Purpose |
|-----------|---------|
| **Monitor** | Alert when DLQ size grows |
| **Investigate** | Analyze failure patterns |
| **Replay** | Reprocess after fix |
| **Purge** | Remove unrecoverable messages |

---

<a id="q19"></a>
### Q19: How do you prevent duplicate notifications?

**Answer:**

**Why Duplicates Happen:**

```
Scenario 1: Producer retry
┌──────────┐        ┌──────────┐
│ Producer │──MSG──▶│  Queue   │
└──────────┘   ✗    └──────────┘
     │      ACK lost
     │
     └──MSG──▶ (Retry = Duplicate in queue)


Scenario 2: Consumer crash
┌──────────┐        ┌──────────┐        ┌──────────┐
│  Queue   │──MSG──▶│ Consumer │──SEND─▶│ Provider │
└──────────┘        └──────────┘        └──────────┘
                         │                  ✓
                      CRASH before commit
                         │
                         ▼
                   Redelivered to new consumer = Duplicate send
```

**Deduplication Strategies:**

```
┌────────────────────────────────────────────────────────┐
│            IDEMPOTENCY KEY DEDUPLICATION               │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Request includes:                                     │
│  {                                                     │
│    "idempotency_key": "order_123_confirmed",           │
│    ...                                                 │
│  }                                                     │
│                                                        │
│  Processing:                                           │
│  1. Check if key exists in dedup store                 │
│  2. If exists → Return cached result (no processing)   │
│  3. If not exists:                                     │
│     a. Process notification                            │
│     b. Store key with result (TTL: 24h)                │
│     c. Return result                                   │
│                                                        │
└────────────────────────────────────────────────────────┘
```

**Deduplication Store (Redis):**

```
Key: dedup:{idempotency_key}
Value: { status: "sent", notification_id: "xxx", sent_at: "..." }
TTL: 24 hours

Operations:
- SET NX (set if not exists) for atomic check-and-set
- EXPIRE for automatic cleanup
```

**Content-Based Deduplication:**

```
For notifications without explicit idempotency key:

Generate key from:
- user_id
- notification_type
- content_hash
- time_window (e.g., 5-minute bucket)

Example: SHA256(user_123 + RIDE_ARRIVING + hash(content) + 2024-01-15T10:05)
```

**Deduplication Levels:**

| Level | Implementation | Trade-off |
|-------|----------------|-----------|
| **Producer** | Idempotency key in request | Requires client cooperation |
| **Queue** | Kafka exactly-once semantics | Complex, performance overhead |
| **Consumer** | Check before send | Additional latency |
| **Provider** | Collapse key (FCM) | Provider-specific |

---

<a id="q20"></a>
### Q20: How do you design for high availability?

**Answer:**

**High Availability Architecture:**

```
┌───────────────────────────────────────────────────────────────┐
│                   MULTI-REGION DEPLOYMENT                     │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  Region A (Primary)              Region B (Secondary)         │
│  ┌─────────────────────┐         ┌─────────────────────┐      │
│  │                     │         │                     │      │
│  │  ┌───────────────┐  │         │  ┌───────────────┐  │      │
│  │  │  API Servers  │  │◀───────▶│  │  API Servers  │  │      │
│  │  │   (Active)    │  │         │  │   (Standby)   │  │      │
│  │  └───────────────┘  │         │  └───────────────┘  │      │
│  │         │           │         │         │           │      │
│  │         ▼           │         │         ▼           │      │
│  │  ┌───────────────┐  │         │  ┌───────────────┐  │      │
│  │  │     Kafka     │  │◀───────▶│  │     Kafka     │  │      │
│  │  │   (Leader)    │  │  Mirror │  │  (Follower)   │  │      │
│  │  └───────────────┘  │         │  └───────────────┘  │      │
│  │         │           │         │         │           │      │
│  │         ▼           │         │         ▼           │      │
│  │  ┌───────────────┐  │         │  ┌───────────────┐  │      │
│  │  │    Workers    │  │         │  │    Workers    │  │      │
│  │  │   (Active)    │  │         │  │   (Standby)   │  │      │
│  │  └───────────────┘  │         │  └───────────────┘  │      │
│  │         │           │         │         │           │      │
│  │         ▼           │         │         ▼           │      │
│  │  ┌───────────────┐  │         │  ┌───────────────┐  │      │
│  │  │   Database    │  │◀───────▶│  │   Database    │  │      │
│  │  │   (Primary)   │  │  Sync   │  │   (Replica)   │  │      │
│  │  └───────────────┘  │         │  └───────────────┘  │      │
│  │                     │         │                     │      │
│  └─────────────────────┘         └─────────────────────┘      │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**HA Strategies by Component:**

| Component | Strategy | RTO/RPO |
|-----------|----------|---------|
| **API Gateway** | Multiple instances, health checks | Seconds |
| **Notification Service** | Stateless, auto-scaling | Seconds |
| **Kafka** | Replication factor 3, rack awareness | Zero data loss |
| **Database** | Primary-replica, auto-failover | Minutes, near-zero loss |
| **Redis** | Cluster mode, sentinel | Seconds |

**Failure Scenarios and Handling:**

| Failure | Detection | Recovery |
|---------|-----------|----------|
| **Single server** | Health check | Load balancer removes |
| **Availability zone** | Zone health | Traffic shifts to other AZs |
| **Region** | Global health check | DNS failover to secondary |
| **Kafka broker** | Leader election | Automatic, partition reassignment |
| **Database primary** | Heartbeat timeout | Promote replica |

**Circuit Breaker Pattern:**

```
┌─────────────────────────────────────────────────────────────┐
│                  CIRCUIT BREAKER STATES                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CLOSED ────▶ (failures > threshold) ────▶ OPEN             │
│    │                                         │              │
│    │                                         │              │
│    │◀── (success) ◀── HALF-OPEN ◀── (timeout)│              │
│                                                             │
│  CLOSED: Normal operation, track failures                   │
│  OPEN: Fail fast, don't call provider                       │
│  HALF-OPEN: Allow limited traffic to test recovery          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

Use circuit breakers for external provider calls (FCM, Twilio) to prevent cascade failures.

---

## Section 5: Data Storage & Analytics

<a id="q21"></a>
### Q21: What database schema would you use?

**Answer:**

**Data Store Selection:**

| Data Type | Store | Reason |
|-----------|-------|--------|
| **User/Device Data** | PostgreSQL | Relational, ACID, complex queries |
| **Notification History** | Cassandra/ScyllaDB | High write throughput, time-series |
| **Templates** | PostgreSQL + Redis | Structured data + caching |
| **Analytics** | ClickHouse/BigQuery | Columnar, aggregation |
| **Rate Limits/Dedup** | Redis | Fast, TTL support |

**Core Tables (PostgreSQL):**

```
┌─────────────────────────────────────────────────────────────┐
│                          USERS                              │
├─────────────────────────────────────────────────────────────┤
│ id (PK)           │ UUID                                    │
│ email             │ VARCHAR(255)                            │
│ phone             │ VARCHAR(20)                             │
│ timezone          │ VARCHAR(50)                             │
│ language          │ VARCHAR(10)                             │
│ created_at        │ TIMESTAMP                               │
│ updated_at        │ TIMESTAMP                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                         DEVICES                             │
├─────────────────────────────────────────────────────────────┤
│ id (PK)           │ UUID                                    │
│ user_id (FK)      │ UUID                                    │
│ platform          │ ENUM('ios', 'android', 'web')           │
│ push_token        │ VARCHAR(500)                            │
│ token_updated_at  │ TIMESTAMP                               │
│ app_version       │ VARCHAR(20)                             │
│ is_active         │ BOOLEAN                                 │
│ created_at        │ TIMESTAMP                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     USER_PREFERENCES                        │
├─────────────────────────────────────────────────────────────┤
│ user_id (PK)      │ UUID                                    │
│ channel           │ ENUM('push', 'sms', 'email')            │
│ notif_type        │ VARCHAR(50)                             │
│ enabled           │ BOOLEAN                                 │
│ quiet_start       │ TIME                                    │
│ quiet_end         │ TIME                                    │
│ updated_at        │ TIMESTAMP                               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                        TEMPLATES                            │
├─────────────────────────────────────────────────────────────┤
│ id (PK)           │ UUID                                    │
│ name              │ VARCHAR(100)                            │
│ type              │ VARCHAR(50)                             │
│ channel           │ ENUM('push', 'sms', 'email')            │
│ language          │ VARCHAR(10)                             │
│ title_template    │ TEXT                                    │
│ body_template     │ TEXT                                    │
│ version           │ INTEGER                                 │
│ is_active         │ BOOLEAN                                 │
│ created_at        │ TIMESTAMP                               │
└─────────────────────────────────────────────────────────────┘
```

**Notification History (Cassandra):**

```
┌─────────────────────────────────────────────────────────────┐
│                  NOTIFICATIONS_BY_USER                      │
├─────────────────────────────────────────────────────────────┤
│ Partition Key: user_id                                      │
│ Clustering Key: created_at DESC, notification_id            │
├─────────────────────────────────────────────────────────────┤
│ user_id           │ UUID                                    │
│ notification_id   │ UUID                                    │
│ type              │ VARCHAR                                 │
│ channel           │ VARCHAR                                 │
│ title             │ VARCHAR                                 │
│ body              │ VARCHAR                                 │
│ status            │ VARCHAR                                 │
│ created_at        │ TIMESTAMP                               │
│ delivered_at      │ TIMESTAMP                               │
│ read_at           │ TIMESTAMP                               │
│ metadata          │ MAP<VARCHAR, VARCHAR>                   │
└─────────────────────────────────────────────────────────────┘

Query pattern: Get recent notifications for user
CQL: SELECT * FROM notifications_by_user 
     WHERE user_id = ? 
     ORDER BY created_at DESC 
     LIMIT 50;
```

---

<a id="q22"></a>
### Q22: How do you track delivery status and analytics?

**Answer:**

**Delivery Status State Machine:**

```
┌─────────────────────────────────────────────────────────────┐
│                NOTIFICATION STATUS FLOW                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  CREATED ──▶ QUEUED ──▶ SENT ──▶ DELIVERED ──▶ READ         │
│     │          │         │           │                      │
│     │          │         │           └──▶ CLICKED           │
│     │          │         │                                  │
│     │          │         └──▶ BOUNCED (soft/hard)           │
│     │          │                                            │
│     │          └──▶ FAILED (max retries)                    │
│     │                                                       │
│     └──▶ CANCELLED (by user/system)                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Status Tracking Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Worker ──▶ Send to Provider                                │
│                │                                            │
│                ▼                                            │
│  Provider Callback/Webhook ──▶ Status Update Service        │
│                                      │                      │
│                                      ├──▶ Update Cassandra  │
│                                      │    (notif history)   │
│                                      │                      │
│                                      ├──▶ Publish to Kafka  │
│                                      │    (analytics stream)│
│                                      │                      │
│                                      └──▶ Update Redis      │
│                                           (real-time status)│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Analytics Data Pipeline:**

```
┌─────────────────────────────────────────────────────────────┐
│                    ANALYTICS PIPELINE                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Status Events ──▶ Kafka ──▶ Stream Processor ──▶ ClickHouse│
│                             (Flink/Spark)      (Analytics)  │
│                                  │                          │
│                                  ├──▶ Real-time aggregates  │
│                                  │    - Delivery rate/min   │
│                                  │    - Error rate/min      │
│                                  │                          │
│                                  └──▶ Batch aggregates      │
│                                       - Daily/weekly reports│
│                                       - Campaign performance│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Key Analytics Queries:**

| Query | Purpose | Store |
|-------|---------|-------|
| Delivery rate last hour | Real-time monitoring | Redis/ClickHouse |
| Campaign performance | Business metrics | ClickHouse |
| User notification history | User experience | Cassandra |
| Failure patterns | Debugging | ClickHouse |
| Provider performance | Vendor management | ClickHouse |

**Dashboard Metrics:**

```
┌─────────────────────────────────────────────────────────────┐
│                   REAL-TIME DASHBOARD                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Throughput:    12,345 notifications/sec                    │
│  Delivery Rate: 99.2%                                       │
│  Avg Latency:   125ms (P50), 450ms (P99)                    │
│  Queue Depth:   5,432 messages                              │
│                                                             │
│  By Channel:                                                │
│  ├── Push:  8,000/sec  (99.5% delivered)                    │
│  ├── SMS:   2,000/sec  (98.8% delivered)                    │
│  └── Email: 2,345/sec  (99.0% delivered)                    │
│                                                             │
│  Errors (last hour):                                        │
│  ├── Invalid token:     234                                 │
│  ├── Rate limited:      56                                  │
│  └── Provider timeout:  12                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

<a id="q23"></a>
### Q23: How do you handle notification history and archival?

**Answer:**

**Data Lifecycle:**

```
┌─────────────────────────────────────────────────────────────┐
│                     DATA LIFECYCLE                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Hot Data (0-7 days)                                        │
│  ├── Store: Cassandra (primary cluster)                     │
│  ├── Access: Frequent (user queries, real-time analytics)   │
│  └── Performance: Optimized for read/write                  │
│                                                             │
│  Warm Data (7-90 days)                                      │
│  ├── Store: Cassandra (secondary cluster) or compress       │
│  ├── Access: Occasional (historical queries, reports)       │
│  └── Performance: Acceptable latency                        │
│                                                             │
│  Cold Data (90+ days)                                       │
│  ├── Store: Object storage (S3/GCS) in Parquet format       │
│  ├── Access: Rare (compliance, audits)                      │
│  └── Performance: High latency acceptable                   │
│                                                             │
│  Archived Data (1+ years)                                   │
│  ├── Store: Glacier/Archive storage                         │
│  ├── Access: Very rare (legal holds)                        │
│  └── Cost: Minimal                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Archival Process:**

```
┌─────────────────────────────────────────────────────────────┐
│                    ARCHIVAL PIPELINE                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Daily Job (runs at low-traffic hours):                     │
│                                                             │
│  1. Query Cassandra for records older than threshold        │
│     SELECT * FROM notifications                             │
│     WHERE created_at < (now - 90 days)                      │
│                                                             │
│  2. Transform to Parquet format (columnar, compressed)      │
│     - Partition by date and notification_type               │
│     - Compress with Snappy/Zstd                             │
│                                                             │
│  3. Upload to object storage                                │
│     s3://notification-archive/year=2024/month=01/day=15/    │
│                                                             │
│  4. Verify upload integrity (checksums)                     │
│                                                             │
│  5. Delete from Cassandra (TTL or explicit delete)          │
│                                                             │
│  6. Update data catalog (for query access)                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Cassandra TTL Strategy:**

```
Insert with TTL:
- Hot data: No TTL (managed by archival job)
- OR: TTL of 90 days (automatic expiration)

Trade-off:
- TTL: Simpler, but no archival
- Archival job: More complex, but preserves history
```

**Querying Archived Data:**

| Tool | Use Case |
|------|----------|
| **Athena/Presto** | Ad-hoc queries on S3 Parquet files |
| **Spark** | Large-scale analytics jobs |
| **Data Catalog** | Schema discovery, partition pruning |

---

## Section 6: Advanced Topics

<a id="q24"></a>
### Q24: How do you implement priority queues for urgent notifications?

**Answer:**

**Priority Levels:**

| Priority | Examples | SLA | Handling |
|----------|----------|-----|----------|
| **P0 - Critical** | OTP, security alerts, payment failure | < 1s | Dedicated queue, always-on workers |
| **P1 - High** | Ride arriving, order ready | < 10s | Priority queue, auto-scaled workers |
| **P2 - Normal** | Order confirmation, receipts | < 60s | Standard queue |
| **P3 - Low** | Marketing, recommendations | < 1h | Batch queue, rate limited |

**Multi-Queue Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                  PRIORITY QUEUE SYSTEM                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Incoming Request                                           │
│        │                                                    │
│        ▼                                                    │
│  ┌─────────────────┐                                        │
│  │ Priority Router │                                        │
│  └────────┬────────┘                                        │
│           │                                                 │
│     ┌─────┼─────┬─────────┐                                 │
│     │     │     │         │                                 │
│     ▼     ▼     ▼         ▼                                 │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                            │
│  │ P0  │ │ P1  │ │ P2  │ │ P3  │  Queues                    │
│  └──┬──┘ └──┬──┘ └──┬──┘ └──┬──┘                            │
│     │       │       │       │                               │
│     ▼       ▼       ▼       ▼                               │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐                            │
│  │ 10  │ │ 20  │ │ 50  │ │  5  │  Workers                   │
│  │fixed│ │auto │ │auto │ │batch│                            │
│  └─────┘ └─────┘ └─────┘ └─────┘                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Worker Allocation Strategy:**

| Queue | Worker Type | Scaling |
|-------|-------------|---------|
| **P0** | Dedicated, always running | Fixed (over-provisioned) |
| **P1** | Dedicated pool | Auto-scale aggressively |
| **P2** | Shared pool | Auto-scale normally |
| **P3** | Batch workers | Scale on schedule |

**Kafka Topic Configuration:**

```
Topic: notifications.p0-critical
- Partitions: 8
- Replication: 3
- Retention: 1 day
- Consumer group: critical-workers (fixed size)

Topic: notifications.p1-high
- Partitions: 16
- Replication: 3
- Consumer group: high-priority-workers (auto-scaled)

Topic: notifications.p2-normal
- Partitions: 32
- Replication: 3
- Consumer group: normal-workers (auto-scaled)

Topic: notifications.p3-batch
- Partitions: 8
- Replication: 3
- Consumer group: batch-workers (scheduled)
```

**Priority Escalation:**

```
If P2 notification waiting > 30 seconds:
  → Automatically promote to P1 queue
  
If P1 notification waiting > 10 seconds:
  → Alert on-call, consider scaling
  
P0 queue depth > 100:
  → Page on-call immediately
```

---

<a id="q25"></a>
### Q25: How do you handle cross-region notification delivery?

**Answer:**

**Multi-Region Architecture:**

```
┌───────────────────────────────────────────────────────────────┐
│                  GLOBAL NOTIFICATION SYSTEM                   │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                     GLOBAL LAYER                        │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │  │
│  │  │  Global LB  │  │ User Region │  │   Global    │      │  │
│  │  │  (Route53)  │  │   Mapping   │  │   Config    │      │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘      │  │
│  └─────────────────────────────────────────────────────────┘  │
│                             │                                 │
│         ┌───────────────────┼───────────────────┐             │
│         │                   │                   │             │
│         ▼                   ▼                   ▼             │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐      │
│  │ Region: SGP │     │ Region: HKG │     │ Region: JKT │      │
│  │             │     │             │     │             │      │
│  │ ┌─────────┐ │     │ ┌─────────┐ │     │ ┌─────────┐ │      │
│  │ │   API   │ │     │ │   API   │ │     │ │   API   │ │      │
│  │ └─────────┘ │     │ └─────────┘ │     │ └─────────┘ │      │
│  │ ┌─────────┐ │     │ ┌─────────┐ │     │ ┌─────────┐ │      │
│  │ │  Kafka  │◀┼─────┼▶│  Kafka  │◀┼─────┼▶│  Kafka  │ │      │
│  │ └─────────┘ │     │ └─────────┘ │     │ └─────────┘ │      │
│  │ ┌─────────┐ │     │ ┌─────────┐ │     │ ┌─────────┐ │      │
│  │ │ Workers │ │     │ │ Workers │ │     │ │ Workers │ │      │
│  │ └─────────┘ │     │ └─────────┘ │     │ └─────────┘ │      │
│  │ ┌─────────┐ │     │ ┌─────────┐ │     │ ┌─────────┐ │      │
│  │ │   DB    │◀┼─────┼▶│   DB    │◀┼─────┼▶│   DB    │ │      │
│  │ │(replica)│ │     │ │(replica)│ │     │ │(primary)│ │      │
│  │ └─────────┘ │     │ └─────────┘ │     │ └─────────┘ │      │
│  └─────────────┘     └─────────────┘     └─────────────┘      │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

**Routing Strategies:**

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **User-based** | Route to user's home region | Data locality, compliance |
| **Latency-based** | Route to nearest region | Minimize latency |
| **Load-based** | Route to least loaded region | Balance capacity |
| **Failover** | Route to backup on failure | High availability |

**Cross-Region Communication:**

```
Scenario: User in Vietnam, notification originated in Singapore

Option 1: Route to User's Region
┌─────────┐       ┌─────────┐       ┌─────────┐
│ SGP API │──MSG─▶│   SGP   │──MSG─▶│   VN    │──▶ User
│         │       │  Kafka  │       │ Workers │
└─────────┘       └─────────┘       └─────────┘
                  (Cross-region replication)

Option 2: Send from Origin Region
┌─────────┐       ┌─────────┐
│ SGP API │──MSG─▶│   SGP   │─────────────────▶ User
│         │       │ Workers │
└─────────┘       └─────────┘
(Higher latency to user, simpler architecture)

Recommended: Option 1 for latency-sensitive, Option 2 for simplicity
```

**Data Replication:**

| Data Type | Replication Strategy |
|-----------|---------------------|
| **User Preferences** | Async replication, eventual consistency |
| **Device Tokens** | Sync to all regions (critical for delivery) |
| **Templates** | Replicate on update (infrequent changes) |
| **Notification History** | Write to local, async replicate |

**Challenges and Solutions:**

| Challenge | Solution |
|-----------|----------|
| **Network latency** | Edge locations, persistent connections |
| **Data consistency** | Eventual consistency with conflict resolution |
| **Failover** | Automatic DNS failover, health checks |
| **Cost** | Minimize cross-region data transfer |
| **Compliance** | Data residency requirements per region |

**Disaster Recovery:**

```
Primary Region Failure:

1. Health check detects failure (< 30 seconds)
2. DNS failover to secondary region (< 60 seconds)
3. Secondary region takes over traffic
4. Kafka MirrorMaker ensures message continuity
5. RPO: < 1 minute (async replication lag)
6. RTO: < 5 minutes (full recovery)
```

---

[← Back to System Design Index](README.md) | [← Back to Main Index](../README.md)

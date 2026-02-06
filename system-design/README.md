# System Design Interview Questions

Comprehensive Q&A documents for system design interviews, focused on distributed systems and scalable architectures.

## Topics

| File | Questions | Description |
|------|-----------|-------------|
| [notification-system.md](notification-system.md) | 25 | Push notifications, SMS, email delivery at scale |
| [rate-limiter.md](rate-limiter.md) | 20 | Rate limiting algorithms, distributed implementation |
| [vol1-chap1.md](vol1-chap1.md) | - | System design fundamentals |

## Topic Overviews

### Notification System

- **Fundamentals**: Components, requirements, metrics
- **Architecture**: High-level design, multi-channel delivery, message queues
- **Scalability**: Horizontal scaling, traffic spikes, caching, rate limiting
- **Reliability**: Delivery guarantees, retries, dead letter queues, deduplication
- **Data Storage**: Schema design, analytics, archival
- **Advanced**: Priority queues, cross-region delivery

### Rate Limiter

- **Fundamentals**: Why rate limiting, algorithms overview
- **Algorithms**: Token Bucket, Leaky Bucket, Fixed/Sliding Window
- **High-Level Design**: Components, request flow, rule storage
- **Distributed**: Redis implementation, race conditions, consistency
- **Optimization**: Memory usage, data structures
- **Advanced**: Multi-tier limiting, headers, client handling

---

[← Back to Main Index](../README.md)

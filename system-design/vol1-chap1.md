

# Scale from Zero to Millions of Users

- `Load Balancer`
- `Database Replication` (Master-Slave)
- `Cache`
  - **Decide when to use cache**
  - **Expiration policies**
  - **Consistency**
  - **Mitigate failures**: A single cache server can be a single point of failure (SPOF). Use distributed caching systems like Redis or Memcached with replication and clustering.
  - **Eviction policies**: Implement strategies to evict stale or less frequently used data from the cache to free up space. Common eviction policies include LRU (Least Recently Used), LFU (Least Frequently Used), and FIFO (First In, First Out).
- `CDN` (Content Delivery Network)
  - **Cost considerations**
  - **Setting an appropriate TTL (Time to Live)**
  - **CDN fallback**
  - **Invalidating files**: Invalidate the CDN object using APIs provided by CDN vendors. Or Use object versioning to serve a different version of the object
- `Convert to Stateless Web Servers`
  - **Store session data in a distributed cache** (e.g., Redis, Memcached) or a database
  - Use JWT (JSON Web Tokens) for session management
- `Data Center Distribution`
  - **Geo-Distributed Databases**
  - Data Partitioning
  - Data Consistency Models
  - **Data Synchronization**
- `Asynchronous Processing`
  - **Message Queues** (e.g., RabbitMQ, Kafka): Decouple components and handle high-throughput tasks asynchronously.
  - **Background Workers**: Process time-consuming tasks in the background to improve user experience.
- `Logging, Metrics, and Monitoring`
  - Centralized Logging: Use tools like ELK Stack (Elasticsearch, Logstash, Kibana) or Splunk for aggregating and analyzing logs.
  - Metrics Collection: Use Prometheus, Grafana, or similar tools to collect and visualize system metrics.
  - Alerting: Set up alerts for critical system metrics to proactively address issues.
- `Database Sharding`
  - **Horizontal Partitioning**: Distribute data across multiple database instances based on a shard key (e.g., user ID, geographic region).
  - Shard Management: Implement strategies for adding/removing shards and rebalancing data.
  - Cross-Shard Queries: Design the system to handle queries that span multiple shards efficiently.
  - **Resharding data**: When shard exhaustion happens, it requires updating the sharding function and moving data around. `Consistent hashing`, which will be discussed in Chapter 5, is a commonly used technique to solve this problem
  - **Celebrity problem**: need to allocate a shard for each celebrity. Each shard might even require further partition
  - **Join and de-normalization**: Joins across shards are expensive. De-normalization can help reduce the need for joins but may lead to data duplication and consistency challenges.

# Summary

To conclude this chapter, we provide a summary of how we scale our system to support millions of users:
- Keep web tier stateless
- Build redundancy at every tier
- Cache data as much as you can
- Support multiple data centers
- Host static assets in CDN
- Scale your data tier by sharding
- Split tiers into individual services
- Monitor your system and use automation tools


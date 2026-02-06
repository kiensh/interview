# Rate Limiter System Design

## Table of Contents

### Fundamentals
- [Q1: What is rate limiting and why do we need it?](#q1)
- [Q2: What are the common rate limiting algorithms?](#q2)
- [Q3: Where should rate limiters be placed in the architecture?](#q3)
- [Q4: What's the difference between rate limiting, throttling, and load shedding?](#q4)

### Algorithm Deep Dive
- [Q5: How does the Token Bucket algorithm work?](#q5)
- [Q6: How does the Sliding Window algorithm work?](#q6)
- [Q7: Compare rate limiting algorithms - when to use which?](#q7)
- [Q8: How do you choose the right algorithm for different scenarios?](#q8)

### High-Level Design
- [Q9: Design a rate limiter - what are the core components?](#q9)
- [Q10: Walk through the request flow in a rate limiter](#q10)
- [Q11: How do you define and store rate limiting rules?](#q11)

### Distributed Rate Limiting
- [Q12: What are the challenges of rate limiting in distributed systems?](#q12)
- [Q13: How do you implement distributed rate limiting with Redis?](#q13)
- [Q14: How do you handle race conditions in distributed rate limiting?](#q14)
- [Q15: Compare centralized vs distributed rate limiting approaches](#q15)

### Data Storage & Optimization
- [Q16: What Redis data structures are used for rate limiting?](#q16)
- [Q17: How do you optimize memory usage in rate limiters?](#q17)

### Advanced Topics
- [Q18: How do you implement multi-tier rate limiting?](#q18)
- [Q19: What rate limit headers should APIs return?](#q19)
- [Q20: How should clients handle rate limit responses?](#q20)

---

## Fundamentals

<a id="q1"></a>
### Q1: What is rate limiting and why do we need it?

**Answer:**

Rate limiting is a technique to control the rate of requests a client can make to a service within a specified time window. It acts as a gatekeeper that protects your system from being overwhelmed.

**Why We Need Rate Limiting:**

| Reason | Description |
|--------|-------------|
| **Prevent DoS attacks** | Block malicious users flooding your system |
| **Reduce cost** | Limit expensive operations (API calls, DB queries) |
| **Prevent resource starvation** | Ensure fair usage among all users |
| **Control data flow** | Match rate between producer and consumer |
| **Manage API quotas** | Enforce tiered pricing plans |

**Real-World Examples:**

```
┌─────────────────────────────────────────────────────────────┐
│                    RATE LIMITING EXAMPLES                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  GitHub API:        5,000 requests/hour (authenticated)     │
│  Twitter API:       300 requests/15 min (per endpoint)      │
│  Stripe API:        100 requests/sec (live mode)            │
│  Google Maps API:   50 requests/sec                         │
│  AWS API Gateway:   10,000 requests/sec (default)           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**What Happens Without Rate Limiting:**

```
Normal Traffic:        With Attack (No Rate Limit):
    │                      │││││││││││││││
    │                      │││││││││││││││
    ▼                      ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
┌────────┐             ┌────────┐
│ Server │  ──OK──▶    │ Server │  ──CRASH──▶  503 for everyone
└────────┘             └────────┘
```

---

<a id="q2"></a>
### Q2: What are the common rate limiting algorithms?

**Answer:**

**Overview of Algorithms:**

```
┌─────────────────────────────────────────────────────────────┐
│                 RATE LIMITING ALGORITHMS                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Token Bucket        - Allows bursts, smooth average     │
│  2. Leaky Bucket        - Strict constant rate output       │
│  3. Fixed Window        - Simple, but boundary spike issue  │
│  4. Sliding Window Log  - Accurate, but memory intensive    │
│  5. Sliding Window Counter - Balance of accuracy & memory   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**1. Token Bucket:**
```
Bucket Capacity: 4 tokens
Refill Rate: 1 token/second

Time 0:  [●●●●] 4 tokens  → Request arrives → [●●●○] 3 tokens (ALLOW)
Time 1:  [●●●●] 4 tokens  → 3 requests     → [●○○○] 1 token  (ALLOW)
Time 2:  [●●○○] 2 tokens  → 3 requests     → [○○○○] 0 tokens (1 DENIED)
```

**2. Leaky Bucket:**
```
Queue Size: 4, Process Rate: 1 req/sec

Requests:  ●●●●●●  (6 arrive at once)
Queue:     [●●●●] ← 2 dropped (queue full)
Output:    ●........●........●........●  (constant rate)
```

**3. Fixed Window:**
```
Window: 1 minute, Limit: 100 requests

|-------- Minute 1 --------|-------- Minute 2 --------|
     [50 requests]              [100 requests]
                    ↑
              Window boundary
              
Problem: 50 req at 0:59 + 100 req at 1:00 = 150 req in 1 sec!
```

**4. Sliding Window Log:**
```
Window: 1 minute, Limit: 3 requests

Timestamps: [10:00:15, 10:00:30, 10:00:45]

At 10:01:00: Remove 10:00:15 (older than 1 min)
             Log: [10:00:30, 10:00:45] → 2 requests → ALLOW
```

**5. Sliding Window Counter:**
```
Current Window: 70% through
Previous Window Count: 100
Current Window Count: 30
Limit: 100

Weighted Count = 100 × 0.3 + 30 × 1.0 = 60 → ALLOW (< 100)
```

**Quick Comparison:**

| Algorithm | Memory | Accuracy | Burst Handling |
|-----------|--------|----------|----------------|
| Token Bucket | Low | Good | Allows controlled bursts |
| Leaky Bucket | Low | Good | No bursts (smooths traffic) |
| Fixed Window | Low | Poor | Boundary spike issue |
| Sliding Log | High | Excellent | Accurate but expensive |
| Sliding Counter | Medium | Good | Good balance |

---

<a id="q3"></a>
### Q3: Where should rate limiters be placed in the architecture?

**Answer:**

Rate limiters can be placed at multiple points in your architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                 RATE LIMITER PLACEMENT                      │
│                                                             │
│  ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐    │
│  │ Client │────▶│  CDN/  │────▶│  API   │────▶│ Service│    │
│  │        │     │  Edge  │     │Gateway │     │        │    │
│  └────────┘     └────────┘     └────────┘     └────────┘    │
│       │              │              │              │        │
│       ▼              ▼              ▼              ▼        │
│    Client         Layer 1       Layer 2       Layer 3       │
│    Side           (Edge)       (Gateway)     (Service)      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Placement Options:**

| Location | Pros | Cons | Use Case |
|----------|------|------|----------|
| **Client-side** | Reduces unnecessary requests | Easily bypassed | Mobile apps, SDKs |
| **CDN/Edge** | Blocks attacks early | Limited customization | DDoS protection |
| **API Gateway** | Centralized, flexible | Single point of failure | Most common choice |
| **Service-level** | Fine-grained control | Complex to manage | Microservices |

**API Gateway Rate Limiting (Most Common):**

```
┌─────────────────────────────────────────────────────────────┐
│                      API GATEWAY                            │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                  RATE LIMITER                       │    │
│  │                                                     │    │
│  │  1. Extract client identifier (API key, IP, user)   │    │
│  │  2. Check rate limit rules                          │    │
│  │  3. Query/update counter in Redis                   │    │
│  │  4. Allow or reject request                         │    │
│  │                                                     │    │
│  └─────────────────────────────────────────────────────┘    │
│                          │                                  │
│              ┌───────────┴───────────┐                      │
│              ▼                       ▼                      │
│        ┌──────────┐           ┌──────────┐                  │
│        │ Service A│           │ Service B│                  │
│        └──────────┘           └──────────┘                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Multi-Layer Defense:**

```
Layer 1 (Edge/CDN):
  - IP-based blocking
  - Geographic restrictions
  - Bot detection

Layer 2 (API Gateway):
  - API key validation
  - Per-client rate limits
  - Endpoint-specific limits

Layer 3 (Service):
  - Business logic limits
  - Resource-specific throttling
  - Per-user quotas
```

---

<a id="q4"></a>
### Q4: What's the difference between rate limiting, throttling, and load shedding?

**Answer:**

```
┌─────────────────────────────────────────────────────────────┐
│            RATE LIMITING vs THROTTLING vs LOAD SHEDDING     │
│                                                             │
│  Rate Limiting:    "You can only make 100 requests/minute"  │
│  Throttling:       "Slow down, I'll process your request"   │
│  Load Shedding:    "System overloaded, dropping requests"   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Detailed Comparison:**

| Aspect | Rate Limiting | Throttling | Load Shedding |
|--------|--------------|------------|---------------|
| **Purpose** | Enforce quotas | Slow down clients | Protect system |
| **Response** | 429 (reject) | Delayed processing | 503 (drop) |
| **Trigger** | Client exceeds limit | Approaching limit | System overload |
| **Scope** | Per-client | Per-client | Global |
| **Proactive** | Yes | Yes | Reactive |

**Visual Representation:**

```
RATE LIMITING:
Requests: ●●●●●●●●●● (10 arrive)
Limit: 5
Result:   ●●●●● ✓    ●●●●● ✗ (rejected immediately)

THROTTLING:
Requests: ●●●●●●●●●● (10 arrive)
Limit: 5/sec
Result:   ●●●●● ✓ now    ●●●●● ✓ later (queued/delayed)

LOAD SHEDDING:
System Load: 95% (critical)
Requests: ●●●●●●●●●● (10 arrive)
Result:   ●●●●●● ✓ (high priority)    ●●●● ✗ (low priority dropped)
```

**When to Use Each:**

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Rate Limiting:                                             │
│  ├── API quota enforcement                                  │
│  ├── Preventing abuse                                       │
│  └── Fair usage policies                                    │
│                                                             │
│  Throttling:                                                │
│  ├── Smoothing bursty traffic                               │
│  ├── Matching producer/consumer rates                       │
│  └── Graceful degradation                                   │
│                                                             │
│  Load Shedding:                                             │
│  ├── Emergency protection                                   │
│  ├── Cascading failure prevention                           │
│  └── Priority-based request handling                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Algorithm Deep Dive

<a id="q5"></a>
### Q5: How does the Token Bucket algorithm work?

**Answer:**

Token Bucket is the most widely used rate limiting algorithm. It allows controlled bursts while maintaining an average rate.

**How It Works:**

```
┌─────────────────────────────────────────────────────────────┐
│                    TOKEN BUCKET ALGORITHM                   │
│                                                             │
│  Parameters:                                                │
│  ├── Bucket Size (capacity): Maximum tokens stored          │
│  ├── Refill Rate: Tokens added per time unit                │
│  └── Tokens Required: Tokens consumed per request           │
│                                                             │
│  ┌─────────────────────────────────────────┐                │
│  │              TOKEN BUCKET               │                │
│  │                                         │                │
│  │    Capacity: 10 tokens                  │                │
│  │    ┌─────────────────────┐              │                │
│  │    │ ● ● ● ● ● ● ○ ○ ○ ○ │ ← 6 tokens   │                │
│  │    └─────────────────────┘              │                │
│  │              ▲                          │                │
│  │              │ Refill: 2 tokens/sec     │                │
│  │                                         │                │
│  │    Request arrives:                     │                │
│  │    - If tokens >= 1: Allow, remove 1    │                │
│  │    - If tokens < 1: Reject (429)        │                │
│  │                                         │                │
│  └─────────────────────────────────────────┘                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Step-by-Step Example:**

```
Config: Capacity=4, Refill=1/sec, Initial=4

Time    Tokens    Event                  Result
─────────────────────────────────────────────────
0.0s    4         Request               ALLOW → 3 tokens
0.1s    3         Request               ALLOW → 2 tokens
0.2s    2         Request               ALLOW → 1 token
0.3s    1         Request               ALLOW → 0 tokens
0.4s    0         Request               DENY (429)
1.0s    1         +1 token (refill)     
1.1s    1         Request               ALLOW → 0 tokens
2.0s    1         +1 token (refill)
3.0s    2         +1 token (refill)
4.0s    3         +1 token (refill)
5.0s    4         +1 token (capped)     Max capacity reached
```

**Implementation (Go):**

```go
type TokenBucket struct {
    capacity     float64       // Max tokens
    tokens       float64       // Current tokens
    refillRate   float64       // Tokens per second
    lastRefill   time.Time     // Last refill timestamp
    mu           sync.Mutex
}

func NewTokenBucket(capacity, refillRate float64) *TokenBucket {
    return &TokenBucket{
        capacity:   capacity,
        tokens:     capacity,  // Start full
        refillRate: refillRate,
        lastRefill: time.Now(),
    }
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    // Refill tokens based on elapsed time
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    tb.tokens = min(tb.capacity, tb.tokens + elapsed*tb.refillRate)
    tb.lastRefill = now
    
    // Check if request can be allowed
    if tb.tokens >= 1 {
        tb.tokens--
        return true
    }
    return false
}
```

**Why Token Bucket is Popular:**

| Advantage | Description |
|-----------|-------------|
| **Allows bursts** | Can handle traffic spikes up to bucket capacity |
| **Memory efficient** | Only stores: tokens, lastRefill, capacity, rate |
| **Simple** | Easy to understand and implement |
| **Flexible** | Easy to adjust capacity and rate |

---

<a id="q6"></a>
### Q6: How does the Sliding Window algorithm work?

**Answer:**

There are two variants: **Sliding Window Log** and **Sliding Window Counter**.

**Sliding Window Log:**

```
┌─────────────────────────────────────────────────────────────┐
│                   SLIDING WINDOW LOG                        │
│                                                             │
│  Stores timestamp of every request in the window            │
│                                                             │
│  Window: 1 minute, Limit: 5 requests                        │
│                                                             │
│  Timeline:                                                  │
│  ────────────────────────────────────────────────────▶      │
│  10:00:15  10:00:30  10:00:45  10:01:00  10:01:10           │
│     ●         ●         ●         ●         ●               │
│                                                             │
│  At 10:01:20, new request arrives:                          │
│  1. Remove timestamps older than 10:00:20                   │
│  2. Count: [10:00:30, 10:00:45, 10:01:00, 10:01:10]         │
│  3. Count = 4 < 5 → ALLOW, add 10:01:20                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Implementation (Sliding Window Log):**

```go
type SlidingWindowLog struct {
    windowSize time.Duration
    limit      int
    timestamps []time.Time
    mu         sync.Mutex
}

func (sw *SlidingWindowLog) Allow() bool {
    sw.mu.Lock()
    defer sw.mu.Unlock()
    
    now := time.Now()
    windowStart := now.Add(-sw.windowSize)
    
    // Remove old timestamps
    validTimestamps := []time.Time{}
    for _, ts := range sw.timestamps {
        if ts.After(windowStart) {
            validTimestamps = append(validTimestamps, ts)
        }
    }
    sw.timestamps = validTimestamps
    
    // Check limit
    if len(sw.timestamps) < sw.limit {
        sw.timestamps = append(sw.timestamps, now)
        return true
    }
    return false
}
```

**Sliding Window Counter (Optimized):**

```
┌─────────────────────────────────────────────────────────────┐
│                 SLIDING WINDOW COUNTER                      │
│                                                             │
│  Uses weighted average of current and previous window       │
│                                                             │
│  |---Previous Window---|---Current Window---|               │
│  |        100 req      |      30 req       |                │
│  |                     |====70%====|       |                │
│                              ↑                              │
│                         Current position                    │
│                                                             │
│  Formula:                                                   │
│  Count = PrevCount × (1 - position%) + CurrCount × 1.0      │
│  Count = 100 × 0.3 + 30 × 1.0 = 60 requests                 │
│                                                             │
│  If limit = 100: 60 < 100 → ALLOW                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Implementation (Sliding Window Counter):**

```go
type SlidingWindowCounter struct {
    windowSize   time.Duration
    limit        int
    prevCount    int
    currCount    int
    windowStart  time.Time
    mu           sync.Mutex
}

func (sw *SlidingWindowCounter) Allow() bool {
    sw.mu.Lock()
    defer sw.mu.Unlock()
    
    now := time.Now()
    
    // Check if we need to slide the window
    if now.Sub(sw.windowStart) >= sw.windowSize {
        sw.prevCount = sw.currCount
        sw.currCount = 0
        sw.windowStart = now.Truncate(sw.windowSize)
    }
    
    // Calculate weighted count
    elapsed := now.Sub(sw.windowStart)
    position := float64(elapsed) / float64(sw.windowSize)
    weightedCount := float64(sw.prevCount)*(1-position) + float64(sw.currCount)
    
    if int(weightedCount) < sw.limit {
        sw.currCount++
        return true
    }
    return false
}
```

**Comparison:**

| Aspect | Sliding Window Log | Sliding Window Counter |
|--------|-------------------|----------------------|
| **Memory** | O(requests in window) | O(1) - just 2 counters |
| **Accuracy** | Exact | Approximate |
| **Best for** | Low traffic, exact limits | High traffic, scalable |

---

<a id="q7"></a>
### Q7: Compare rate limiting algorithms - when to use which?

**Answer:**

**Comprehensive Comparison:**

```
┌─────────────────────────────────────────────────────────────┐
│              ALGORITHM COMPARISON MATRIX                    │
│                                                             │
│             Memory   Accuracy   Burst   Complexity   Best   │
│             ──────   ────────   ─────   ──────────   ────   │
│  Token      O(1)     Good       Yes     Low          APIs   │
│  Bucket     ●○○○○    ●●●●○      ●●●●●   ●○○○○               │
│                                                             │
│  Leaky      O(N)     Good       No      Medium       Queue  │
│  Bucket     ●●○○○    ●●●●○      ●○○○○   ●●○○○               │
│                                                             │
│  Fixed      O(1)     Poor       Edge    Low          Simple │
│  Window     ●○○○○    ●●○○○      ●●●○○   ●○○○○               │
│                                                             │
│  Sliding    O(N)     Exact      No      High         Exact  │
│  Log        ●●●●●    ●●●●●      ●●○○○   ●●●●○               │
│                                                             │
│  Sliding    O(1)     Good       Some    Medium       Scale  │
│  Counter    ●○○○○    ●●●●○      ●●●○○   ●●○○○               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Decision Tree:**

```
                    Need rate limiting?
                           │
              ┌────────────┴────────────┐
              ▼                         ▼
      Allow bursts?              Need exact count?
           │                           │
     ┌─────┴─────┐              ┌──────┴──────┐
     ▼           ▼              ▼             ▼
    Yes          No            Yes            No
     │           │              │             │
     ▼           ▼              ▼             ▼
  Token       Leaky         Sliding       Sliding
  Bucket      Bucket        Window Log    Window Counter
```

**Use Case Recommendations:**

| Use Case | Recommended | Reason |
|----------|-------------|--------|
| **Public API** | Token Bucket | Allows bursts, good UX |
| **Payment processing** | Sliding Window Log | Exact counting required |
| **Real-time streaming** | Leaky Bucket | Smooth constant output |
| **High-traffic API** | Sliding Window Counter | Memory efficient |
| **Simple internal service** | Fixed Window | Easy to implement |
| **DDoS protection** | Token Bucket + Fixed Window | Defense in depth |

---

<a id="q8"></a>
### Q8: How do you choose the right algorithm for different scenarios?

**Answer:**

**Scenario-Based Selection:**

```
┌─────────────────────────────────────────────────────────────┐
│                 SCENARIO: E-COMMERCE API                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Requirements:                                              │
│  - Handle flash sales (traffic spikes)                      │
│  - Different limits per endpoint                            │
│  - Premium users get higher limits                          │
│                                                             │
│  Recommendation: Token Bucket                               │
│  - Allows burst during flash sales                          │
│  - Easy to configure different bucket sizes                 │
│  - Simple to implement tiered limits                        │
│                                                             │
│  Config:                                                    │
│  - /products: capacity=100, refill=50/sec                   │
│  - /checkout: capacity=10, refill=5/sec (stricter)          │
│  - Premium: 2x capacity                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              SCENARIO: FINANCIAL TRADING API                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Requirements:                                              │
│  - Exact request counting (regulatory compliance)           │
│  - No tolerance for over-limit requests                     │
│  - Audit trail needed                                       │
│                                                             │
│  Recommendation: Sliding Window Log                         │
│  - Exact counting, no approximation                         │
│  - Timestamps provide audit trail                           │
│  - Strict enforcement                                       │
│                                                             │
│  Trade-off: Higher memory usage (acceptable for compliance) │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                SCENARIO: VIDEO STREAMING                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Requirements:                                              │
│  - Constant bitrate delivery                                │
│  - No buffering/stuttering                                  │
│  - Smooth playback experience                               │
│                                                             │
│  Recommendation: Leaky Bucket                               │
│  - Outputs at constant rate                                 │
│  - Smooths bursty input                                     │
│  - Prevents buffer overflow                                 │
│                                                             │
│  Config:                                                    │
│  - Bucket size: 5 seconds of video                          │
│  - Leak rate: video bitrate                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Multi-Algorithm Strategy:**

For large systems, combine algorithms at different layers:

```
┌─────────────────────────────────────────────────────────────┐
│                  MULTI-LAYER RATE LIMITING                  │
│                                                             │
│  Layer 1: Edge (Fixed Window)                               │
│  ├── Simple IP-based blocking                               │
│  └── Stop obvious attacks early                             │
│                                                             │
│  Layer 2: API Gateway (Token Bucket)                        │
│  ├── Per-client rate limits                                 │
│  └── Allows legitimate bursts                               │
│                                                             │
│  Layer 3: Service (Sliding Window Counter)                  │
│  ├── Per-user, per-resource limits                          │
│  └── Fine-grained control                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## High-Level Design

<a id="q9"></a>
### Q9: Design a rate limiter - what are the core components?

**Answer:**

**System Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                    RATE LIMITER SYSTEM                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    API GATEWAY                      │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐ │    │
│  │  │  Auth   │  │  Rate   │  │ Routing │  │ Logging │ │    │
│  │  │         │  │ Limiter │  │         │  │         │ │    │
│  │  └─────────┘  └────┬────┘  └─────────┘  └─────────┘ │    │
│  └────────────────────┼────────────────────────────────┘    │
│                       │                                     │
│                       ▼                                     │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                RATE LIMITER SERVICE                 │    │
│  │                                                     │    │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐       │     │
│  │  │   Rule    │  │  Counter  │  │ Decision  │       │     │
│  │  │  Engine   │  │   Store   │  │  Engine   │       │     │
│  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘       │     │
│  │        │              │              │              │    │
│  └────────┼──────────────┼──────────────┼─────────────┘     │
│           │              │              │                   │
│           ▼              ▼              ▼                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │    Rules    │  │    Redis    │  │   Metrics   │          │
│  │  Database   │  │   Cluster   │  │ (Prometheus)│          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Core Components:**

| Component | Responsibility | Technology |
|-----------|---------------|------------|
| **Rule Engine** | Load and match rate limit rules | In-memory cache |
| **Counter Store** | Track request counts | Redis |
| **Decision Engine** | Allow/deny based on rules | Application logic |
| **Config Store** | Persist rate limit rules | PostgreSQL/etcd |
| **Metrics** | Monitor rate limit stats | Prometheus |

**Component Details:**

```
┌─────────────────────────────────────────────────────────────┐
│                      RULE ENGINE                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Responsibilities:                                          │
│  ├── Load rules from config store                           │
│  ├── Cache rules in memory                                  │
│  ├── Match incoming request to applicable rules             │
│  └── Support rule priority (specific > general)             │
│                                                             │
│  Rule Matching Order:                                       │
│  1. User + Endpoint specific                                │
│  2. User general                                            │
│  3. Endpoint specific                                       │
│  4. Global default                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     COUNTER STORE                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Key Format: rate_limit:{client_id}:{endpoint}:{window}     │
│                                                             │
│  Example Keys:                                              │
│  - rate_limit:user_123:/api/orders:1706140800               │
│  - rate_limit:api_key_abc:/api/search:1706140800            │
│                                                             │
│  Operations:                                                │
│  - INCR: Increment counter atomically                       │
│  - GET: Read current count                                  │
│  - EXPIRE: Auto-cleanup old windows                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

<a id="q10"></a>
### Q10: Walk through the request flow in a rate limiter

**Answer:**

**Request Flow Diagram:**

```
┌─────────────────────────────────────────────────────────────┐
│                    REQUEST FLOW                             │
│                                                             │
│  Client                                                     │
│    │                                                        │
│    │ 1. HTTP Request                                        │
│    ▼                                                        │
│  ┌─────────────────────────────────────────────────────┐    │
│  │                    API GATEWAY                      │    │
│  │                                                     │    │
│  │  2. Extract identifier (API key, IP, User ID)      │     │
│  │                       │                             │    │
│  │                       ▼                             │    │
│  │  3. ┌─────────────────────────────────┐            │     │
│  │     │        RATE LIMITER             │            │     │
│  │     │                                 │            │     │
│  │     │  a. Load applicable rules       │            │     │
│  │     │  b. Query counter from Redis    │◀───┐       │     │
│  │     │  c. Apply algorithm             │    │       │     │
│  │     │  d. Increment counter           │────┘       │     │
│  │     │  e. Return allow/deny           │            │     │
│  │     │                                 │            │     │
│  │     └────────────┬────────────────────┘            │     │
│  │                  │                                  │    │
│  │        ┌─────────┴─────────┐                       │     │
│  │        ▼                   ▼                       │     │
│  │     ALLOW              DENY (429)                  │     │
│  │        │                   │                       │     │
│  └────────┼───────────────────┼───────────────────────┘     │
│           │                   │                             │
│           ▼                   ▼                             │
│  4. Route to Service    Return Error Response               │
│           │                   │                             │
│           ▼                   │                             │
│  ┌─────────────┐              │                             │
│  │   Service   │              │                             │
│  └──────┬──────┘              │                             │
│         │                     │                             │
│         ▼                     ▼                             │
│  5. Response to Client (200 or 429)                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Detailed Steps:**

```
Step 1: Request Arrives
─────────────────────────────────────────────────
POST /api/orders HTTP/1.1
Host: api.example.com
X-API-Key: abc123
Content-Type: application/json

Step 2: Extract Identifier
─────────────────────────────────────────────────
Priority:
1. X-API-Key header → "abc123"
2. Authorization Bearer token → extract user_id
3. X-Forwarded-For / Client IP → "203.0.113.42"

Identifier: "api_key:abc123"

Step 3: Load Rules
─────────────────────────────────────────────────
Rules for api_key:abc123:
- Global: 1000 req/hour
- /api/orders: 100 req/min
- /api/orders POST: 10 req/min (most specific, use this)

Step 4: Check Counter (Redis)
─────────────────────────────────────────────────
Key: rate_limit:abc123:POST:/api/orders:202401251430
GET → 8 requests in current window
Limit: 10, Current: 8 → ALLOW

Step 5: Update Counter
─────────────────────────────────────────────────
INCR rate_limit:abc123:POST:/api/orders:202401251430
New value: 9

Step 6: Add Rate Limit Headers
─────────────────────────────────────────────────
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 1
X-RateLimit-Reset: 1706186460

Step 7: Forward to Service or Reject
─────────────────────────────────────────────────
If ALLOW: Forward request to Order Service
If DENY: Return 429 Too Many Requests
```

---

<a id="q11"></a>
### Q11: How do you define and store rate limiting rules?

**Answer:**

**Rule Structure:**

```json
{
  "rule_id": "rule_001",
  "name": "API Orders Endpoint Limit",
  "description": "Limit order creation requests",
  "match": {
    "client_type": "api_key",
    "endpoint": "/api/orders",
    "method": "POST"
  },
  "limit": {
    "requests": 100,
    "window": "1m",
    "algorithm": "token_bucket",
    "burst": 20
  },
  "action": {
    "on_limit": "reject",
    "status_code": 429,
    "retry_after": true
  },
  "priority": 100,
  "enabled": true
}
```

**Rule Hierarchy:**

```
┌─────────────────────────────────────────────────────────────┐
│                     RULE HIERARCHY                          │
│                                                             │
│  Priority (higher = more specific):                         │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ P1000: User + Endpoint + Method                      │   │
│  │        user:123 + POST /api/orders                   │   │
│  └─────────────────────────────────────────────────────┘    │
│                         │                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ P500:  User + Endpoint                               │   │
│  │        user:123 + /api/orders                        │   │
│  └─────────────────────────────────────────────────────┘    │
│                         │                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ P200:  Endpoint + Method                             │   │
│  │        POST /api/orders (all users)                  │   │
│  └─────────────────────────────────────────────────────┘    │
│                         │                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ P100:  User general                                  │   │
│  │        user:123 (all endpoints)                      │   │
│  └─────────────────────────────────────────────────────┘    │
│                         │                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ P10:   Tier-based                                    │   │
│  │        tier:premium (1000 req/min)                   │   │
│  └─────────────────────────────────────────────────────┘    │
│                         │                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ P1:    Global default                                │   │
│  │        All requests (100 req/min)                    │   │
│  └─────────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Storage Options:**

| Storage | Pros | Cons | Use Case |
|---------|------|------|----------|
| **Config file** | Simple, version controlled | Requires restart | Small, static rules |
| **Database** | Dynamic updates, audit log | Latency | Complex rule management |
| **etcd/Consul** | Distributed, watch support | Additional infra | Microservices |
| **Redis** | Fast, distributed | Persistence concerns | Hybrid (rules + counters) |

**Database Schema:**

```sql
CREATE TABLE rate_limit_rules (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    
    -- Matching criteria
    client_type     VARCHAR(50),  -- 'api_key', 'user', 'ip'
    client_id       VARCHAR(255), -- specific client or NULL for all
    endpoint        VARCHAR(255), -- '/api/orders' or '*'
    method          VARCHAR(10),  -- 'POST', 'GET', '*'
    
    -- Limit configuration
    requests        INT NOT NULL,
    window_seconds  INT NOT NULL,
    algorithm       VARCHAR(50) DEFAULT 'token_bucket',
    burst_size      INT,
    
    -- Behavior
    priority        INT DEFAULT 0,
    enabled         BOOLEAN DEFAULT true,
    
    -- Audit
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_rules_matching 
ON rate_limit_rules(client_type, endpoint, method, priority DESC);
```

---

## Distributed Rate Limiting

<a id="q12"></a>
### Q12: What are the challenges of rate limiting in distributed systems?

**Answer:**

**The Problem:**

```
┌─────────────────────────────────────────────────────────────┐
│              DISTRIBUTED RATE LIMITING CHALLENGE            │
│                                                             │
│  Client making 10 requests with limit of 5:                 │
│                                                             │
│               ┌─────────────────────┐                       │
│               │      Client         │                       │
│               │   10 requests       │                       │
│               └──────────┬──────────┘                       │
│                          │                                  │
│            ┌─────────────┼─────────────┐                    │
│            ▼             ▼             ▼                    │
│       ┌────────┐    ┌────────┐    ┌────────┐                │
│       │Server 1│    │Server 2│    │Server 3│                │
│       │ 3 req  │    │ 4 req  │    │ 3 req  │                │
│       │Local: 3│    │Local: 4│    │Local: 3│                │
│       │ALLOW ✓ │    │ALLOW ✓ │    │ALLOW ✓ │                │
│       └────────┘    └────────┘    └────────┘                │
│                                                             │
│  Problem: Each server sees only its local count!            │
│  Total allowed: 10 (but limit was 5)                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Key Challenges:**

| Challenge | Description | Impact |
|-----------|-------------|--------|
| **Consistency** | Counters must be synchronized | Over/under limiting |
| **Latency** | Central store adds network hop | Slower requests |
| **Availability** | What if counter store is down? | System failure |
| **Race conditions** | Concurrent increments | Exceeding limits |
| **Scalability** | Hot keys under high load | Bottleneck |

**Race Condition Example:**

```
Time    Server A              Server B              Redis
────────────────────────────────────────────────────────────
T1      GET counter           GET counter           counter=4
T2      Read: 4               Read: 4               
T3      4 < 5? Yes            4 < 5? Yes            
T4      INCR counter          INCR counter          
T5                                                  counter=6!
────────────────────────────────────────────────────────────
Result: Both requests allowed, but limit was 5!
```

**Solutions Overview:**

```
┌─────────────────────────────────────────────────────────────┐
│                     SOLUTIONS                               │
│                                                             │
│  1. Atomic Operations (Redis INCR)                          │
│     └── Check-and-increment in single operation             │
│                                                             │
│  2. Lua Scripts (Redis)                                     │
│     └── Complex logic executed atomically                   │
│                                                             │
│  3. Redis Cell (Rate limiting module)                       │
│     └── Built-in GCRA algorithm                             │
│                                                             │
│  4. Sticky Sessions                                         │
│     └── Route same client to same server                    │
│                                                             │
│  5. Local + Sync                                            │
│     └── Local counters with periodic sync                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

<a id="q13"></a>
### Q13: How do you implement distributed rate limiting with Redis?

**Answer:**

**Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│             REDIS-BASED DISTRIBUTED RATE LIMITER            │
│                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                      │
│  │Server 1 │  │Server 2 │  │Server 3 │                      │
│  └────┬────┘  └────┬────┘  └────┬────┘                      │
│       │            │            │                           │
│       └────────────┼────────────┘                           │
│                    │                                        │
│                    ▼                                        │
│       ┌────────────────────────┐                            │
│       │     REDIS CLUSTER      │                            │
│       │                        │                            │
│       │  ┌──────┐  ┌──────┐   │                             │
│       │  │Node 1│  │Node 2│   │                             │
│       │  │Master│  │Master│   │                             │
│       │  └──┬───┘  └──┬───┘   │                             │
│       │     │         │       │                             │
│       │  ┌──┴───┐  ┌──┴───┐   │                             │
│       │  │Replica│ │Replica│   │                            │
│       │  └──────┘  └──────┘   │                             │
│       │                        │                            │
│       └────────────────────────┘                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Implementation - Fixed Window with Redis:**

```go
func (rl *RateLimiter) AllowFixedWindow(ctx context.Context, key string) (bool, error) {
    // Key format: rate_limit:{client}:{window_timestamp}
    windowKey := fmt.Sprintf("rate_limit:%s:%d", key, time.Now().Unix()/60)
    
    pipe := rl.redis.Pipeline()
    incr := pipe.Incr(ctx, windowKey)
    pipe.Expire(ctx, windowKey, 2*time.Minute) // Cleanup buffer
    
    _, err := pipe.Exec(ctx)
    if err != nil {
        return false, err
    }
    
    count := incr.Val()
    return count <= rl.limit, nil
}
```

**Implementation - Token Bucket with Lua Script:**

```lua
-- token_bucket.lua
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

-- Get current state
local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or capacity
local last_refill = tonumber(data[2]) or now

-- Calculate tokens to add
local elapsed = now - last_refill
local tokens_to_add = elapsed * refill_rate
tokens = math.min(capacity, tokens + tokens_to_add)

-- Check if request can be allowed
local allowed = false
if tokens >= requested then
    tokens = tokens - requested
    allowed = true
end

-- Update state
redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', key, 3600) -- 1 hour TTL

return {allowed and 1 or 0, tokens}
```

**Go Code to Execute Lua Script:**

```go
var tokenBucketScript = redis.NewScript(`
    -- Lua script from above
`)

func (rl *RateLimiter) AllowTokenBucket(ctx context.Context, key string) (bool, int, error) {
    now := float64(time.Now().UnixNano()) / 1e9
    
    result, err := tokenBucketScript.Run(ctx, rl.redis, 
        []string{key},
        rl.capacity,
        rl.refillRate,
        now,
        1, // tokens requested
    ).Result()
    
    if err != nil {
        return false, 0, err
    }
    
    res := result.([]interface{})
    allowed := res[0].(int64) == 1
    remaining := int(res[1].(int64))
    
    return allowed, remaining, nil
}
```

**Using Redis Cell (GCRA Algorithm):**

```go
// Redis Cell module provides CL.THROTTLE command
// Install: https://github.com/brandur/redis-cell

func (rl *RateLimiter) AllowRedisCell(ctx context.Context, key string) (bool, int, int, error) {
    // CL.THROTTLE key max_burst count_per_period period [quantity]
    // Example: CL.THROTTLE user:123 10 100 60 1
    // Allow 100 requests per 60 seconds with burst of 10
    
    result, err := rl.redis.Do(ctx, "CL.THROTTLE", 
        key,
        rl.burst,        // max_burst
        rl.limit,        // count_per_period
        rl.windowSecs,   // period in seconds
        1,               // quantity (tokens requested)
    ).Result()
    
    if err != nil {
        return false, 0, 0, err
    }
    
    // Response: [limited, limit, remaining, retry_after, reset_after]
    res := result.([]interface{})
    limited := res[0].(int64) == 1
    remaining := int(res[2].(int64))
    retryAfter := int(res[3].(int64))
    
    return !limited, remaining, retryAfter, nil
}
```

---

<a id="q14"></a>
### Q14: How do you handle race conditions in distributed rate limiting?

**Answer:**

**The Race Condition Problem:**

```
┌─────────────────────────────────────────────────────────────┐
│                   RACE CONDITION                            │
│                                                             │
│  Naive approach (GET then SET):                             │
│                                                             │
│  Server A                    Server B                       │
│     │                           │                           │
│     │── GET counter ──────────▶│                            │
│     │◀─────── 4 ───────────────│                            │
│     │                           │── GET counter ──▶         │
│     │                           │◀────── 4 ───────          │
│     │                           │                           │
│     │   4 < 5? ALLOW           │   4 < 5? ALLOW             │
│     │                           │                           │
│     │── SET counter=5 ────────▶│                            │
│     │                           │── SET counter=5 ──▶       │
│     │                           │                           │
│  Result: Both allowed, but only 1 should be!                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Solution 1: Atomic INCR with Check:**

```go
func (rl *RateLimiter) AllowAtomic(ctx context.Context, key string) (bool, error) {
    // INCR is atomic - increment first, then check
    count, err := rl.redis.Incr(ctx, key).Result()
    if err != nil {
        return false, err
    }
    
    // Set expiry on first request
    if count == 1 {
        rl.redis.Expire(ctx, key, rl.window)
    }
    
    // Check after increment
    if count > rl.limit {
        return false, nil // Over limit
    }
    
    return true, nil
}
```

**Solution 2: Lua Script (Complex Logic):**

```lua
-- Atomic check-and-increment with all logic in one operation
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call('GET', key)
if current and tonumber(current) >= limit then
    return {0, tonumber(current), -1}  -- Denied
end

local new_count = redis.call('INCR', key)
if new_count == 1 then
    redis.call('EXPIRE', key, window)
end

local ttl = redis.call('TTL', key)
return {1, new_count, ttl}  -- Allowed
```

**Solution 3: Redis Transactions (WATCH/MULTI/EXEC):**

```go
func (rl *RateLimiter) AllowWithWatch(ctx context.Context, key string) (bool, error) {
    // Optimistic locking with WATCH
    txf := func(tx *redis.Tx) error {
        count, err := tx.Get(ctx, key).Int()
        if err != nil && err != redis.Nil {
            return err
        }
        
        if count >= rl.limit {
            return ErrRateLimited
        }
        
        _, err = tx.TxPipelined(ctx, func(pipe redis.Pipeliner) error {
            pipe.Incr(ctx, key)
            pipe.Expire(ctx, key, rl.window)
            return nil
        })
        return err
    }
    
    // Retry on WATCH failure (key changed)
    for i := 0; i < 3; i++ {
        err := rl.redis.Watch(ctx, txf, key)
        if err == nil {
            return true, nil
        }
        if err == redis.TxFailedErr {
            continue // Retry
        }
        return false, err
    }
    
    return false, ErrRetryExhausted
}
```

**Comparison of Solutions:**

| Solution | Atomicity | Complexity | Performance | Use Case |
|----------|-----------|------------|-------------|----------|
| **INCR + Check** | Partial | Low | Best | Simple counters |
| **Lua Script** | Full | Medium | Good | Complex algorithms |
| **WATCH/MULTI** | Full | High | Variable | Rare updates |
| **Redis Cell** | Full | Low | Good | Production ready |

---

<a id="q15"></a>
### Q15: Compare centralized vs distributed rate limiting approaches

**Answer:**

**Centralized Approach:**

```
┌─────────────────────────────────────────────────────────────┐
│                  CENTRALIZED RATE LIMITING                  │
│                                                             │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐             │
│  │Server 1│  │Server 2│  │Server 3│  │Server 4│             │
│  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘             │
│      │           │           │           │                  │
│      └───────────┴─────┬─────┴───────────┘                  │
│                        │                                    │
│                        ▼                                    │
│              ┌──────────────────┐                           │
│              │  Central Redis   │                           │
│              │  (Single Source  │                           │
│              │   of Truth)      │                           │
│              └──────────────────┘                           │
│                                                             │
│  Pros: Accurate counting, simple logic                      │
│  Cons: Single point of failure, latency                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Distributed Approach:**

```
┌─────────────────────────────────────────────────────────────┐
│                 DISTRIBUTED RATE LIMITING                   │
│                                                             │
│  ┌────────────────┐  ┌────────────────┐                     │
│  │   Server 1     │  │   Server 2     │                     │
│  │ ┌────────────┐ │  │ ┌────────────┐ │                     │
│  │ │Local Cache │ │  │ │Local Cache │ │                     │
│  │ │ count: 25  │ │  │ │ count: 23  │ │                     │
│  │ └────────────┘ │  │ └────────────┘ │                     │
│  └───────┬────────┘  └───────┬────────┘                     │
│          │                   │                              │
│          └─────────┬─────────┘                              │
│                    │ Periodic Sync                          │
│                    ▼                                        │
│          ┌──────────────────┐                               │
│          │  Redis Cluster   │                               │
│          │  (Aggregated)    │                               │
│          └──────────────────┘                               │
│                                                             │
│  Pros: Low latency, high availability                       │
│  Cons: Approximate counting, sync complexity                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Hybrid Approach (Recommended):**

```
┌─────────────────────────────────────────────────────────────┐
│                   HYBRID RATE LIMITING                      │
│                                                             │
│  Request arrives                                            │
│       │                                                     │
│       ▼                                                     │
│  ┌─────────────────────────────────────┐                    │
│  │         LOCAL CHECK (Fast)          │                    │
│  │  Local token bucket with fraction   │                    │
│  │  of total limit (limit / N servers) │                    │
│  └──────────────────┬──────────────────┘                    │
│                     │                                       │
│            ┌────────┴────────┐                              │
│            ▼                 ▼                              │
│      Local ALLOW       Local DENY                           │
│            │                 │                              │
│            ▼                 │                              │
│  ┌─────────────────┐         │                              │
│  │  REDIS CHECK    │         │                              │
│  │  (Accurate)     │         │                              │
│  └────────┬────────┘         │                              │
│           │                  │                              │
│      ┌────┴────┐             │                              │
│      ▼         ▼             ▼                              │
│    ALLOW     DENY          DENY                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Implementation of Hybrid Approach:**

```go
type HybridRateLimiter struct {
    localLimiter  *TokenBucket    // Fast local check
    globalLimiter *RedisLimiter   // Accurate global check
    localRatio    float64         // Fraction of limit for local
}

func (h *HybridRateLimiter) Allow(ctx context.Context, key string) (bool, error) {
    // Step 1: Fast local check (non-blocking)
    if !h.localLimiter.Allow() {
        return false, nil // Definitely over limit
    }
    
    // Step 2: Accurate global check (may add latency)
    return h.globalLimiter.Allow(ctx, key)
}

func NewHybridRateLimiter(totalLimit int, numServers int, redis *redis.Client) *HybridRateLimiter {
    // Each server gets fraction of limit locally
    localLimit := float64(totalLimit) / float64(numServers)
    
    return &HybridRateLimiter{
        localLimiter:  NewTokenBucket(localLimit, localLimit/60),
        globalLimiter: NewRedisLimiter(redis, totalLimit),
        localRatio:    1.0 / float64(numServers),
    }
}
```

**Comparison Summary:**

| Aspect | Centralized | Distributed | Hybrid |
|--------|-------------|-------------|--------|
| **Accuracy** | Exact | Approximate | Good |
| **Latency** | Higher | Lower | Low (local hit) |
| **Availability** | SPOF risk | High | High |
| **Complexity** | Low | Medium | Higher |
| **Best for** | Small scale | Large scale | Production |

---

## Data Storage & Optimization

<a id="q16"></a>
### Q16: What Redis data structures are used for rate limiting?

**Answer:**

**Data Structures Overview:**

```
┌─────────────────────────────────────────────────────────────┐
│              REDIS DATA STRUCTURES FOR RATE LIMITING        │
│                                                             │
│  1. String (Counter)     - Fixed/Sliding window counter     │
│  2. Hash                 - Token bucket state               │
│  3. Sorted Set (ZSET)    - Sliding window log               │
│  4. List                 - Leaky bucket queue               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**1. String - Simple Counter:**

```
Use Case: Fixed Window Counter

Commands:
  INCR rate_limit:user123:1706140800
  EXPIRE rate_limit:user123:1706140800 60

Key Format: rate_limit:{client}:{window_timestamp}
Value: Integer count

Memory: ~50 bytes per key
```

```go
func fixedWindowCounter(ctx context.Context, rdb *redis.Client, key string, limit int) bool {
    windowKey := fmt.Sprintf("%s:%d", key, time.Now().Unix()/60)
    
    count, _ := rdb.Incr(ctx, windowKey).Result()
    if count == 1 {
        rdb.Expire(ctx, windowKey, 2*time.Minute)
    }
    
    return count <= int64(limit)
}
```

**2. Hash - Token Bucket State:**

```
Use Case: Token Bucket Algorithm

Key: rate_limit:bucket:user123
Fields:
  - tokens: 7.5
  - last_refill: 1706140800.123

Commands:
  HMSET rate_limit:bucket:user123 tokens 7.5 last_refill 1706140800.123
  HMGET rate_limit:bucket:user123 tokens last_refill

Memory: ~100 bytes per key
```

```go
type BucketState struct {
    Tokens     float64 `redis:"tokens"`
    LastRefill float64 `redis:"last_refill"`
}

func tokenBucketWithHash(ctx context.Context, rdb *redis.Client, key string) bool {
    var state BucketState
    rdb.HGetAll(ctx, key).Scan(&state)
    
    // Calculate and update...
    rdb.HSet(ctx, key, "tokens", newTokens, "last_refill", now)
    return allowed
}
```

**3. Sorted Set - Sliding Window Log:**

```
Use Case: Exact sliding window counting

Key: rate_limit:log:user123
Members: Request timestamps (score = timestamp)

Commands:
  ZADD rate_limit:log:user123 1706140800.123 "req_uuid_1"
  ZREMRANGEBYSCORE rate_limit:log:user123 0 1706140740  -- Remove old
  ZCARD rate_limit:log:user123  -- Count requests

Memory: ~80 bytes per request (high for busy clients)
```

```go
func slidingWindowLog(ctx context.Context, rdb *redis.Client, key string, limit int, window time.Duration) bool {
    now := float64(time.Now().UnixNano()) / 1e9
    windowStart := now - window.Seconds()
    
    pipe := rdb.Pipeline()
    
    // Remove old entries
    pipe.ZRemRangeByScore(ctx, key, "0", fmt.Sprintf("%f", windowStart))
    
    // Count current entries
    countCmd := pipe.ZCard(ctx, key)
    
    pipe.Exec(ctx)
    
    if countCmd.Val() < int64(limit) {
        // Add new entry
        rdb.ZAdd(ctx, key, redis.Z{Score: now, Member: uuid.New().String()})
        rdb.Expire(ctx, key, window+time.Minute)
        return true
    }
    
    return false
}
```

**4. List - Leaky Bucket Queue:**

```
Use Case: Request queuing with constant drain rate

Key: rate_limit:queue:user123
Elements: Serialized requests

Commands:
  LPUSH rate_limit:queue:user123 "{request_data}"
  RPOP rate_limit:queue:user123  -- Consumer drains at constant rate
  LLEN rate_limit:queue:user123

Memory: Variable based on request size
```

**Comparison:**

| Structure | Algorithm | Memory | Operations | Accuracy |
|-----------|-----------|--------|------------|----------|
| String | Fixed Window | O(1) | INCR, EXPIRE | Low |
| Hash | Token Bucket | O(1) | HGET, HSET | Good |
| Sorted Set | Sliding Log | O(n) | ZADD, ZREM, ZCARD | Exact |
| List | Leaky Bucket | O(n) | LPUSH, RPOP | Good |

---

<a id="q17"></a>
### Q17: How do you optimize memory usage in rate limiters?

**Answer:**

**Memory Optimization Strategies:**

```
┌─────────────────────────────────────────────────────────────┐
│               MEMORY OPTIMIZATION STRATEGIES                │
│                                                             │
│  1. Choose efficient algorithm (Token Bucket = O(1))        │
│  2. Set appropriate TTLs (auto-cleanup)                     │
│  3. Use sliding window counter instead of log               │
│  4. Compress keys (short prefixes)                          │
│  5. Use Redis Cluster for horizontal scaling                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**1. Key Compression:**

```
Before: rate_limit:user:authentication:endpoint:/api/v1/orders:window:1706140800
After:  rl:u:a:/api/v1/orders:1706140800

Savings: ~50% key size reduction
```

```go
func compressKey(clientType, clientID, endpoint string, window int64) string {
    // Use short prefixes
    typePrefix := map[string]string{
        "user":    "u",
        "api_key": "k",
        "ip":      "i",
    }[clientType]
    
    // Hash long endpoints
    endpointHash := endpoint
    if len(endpoint) > 20 {
        h := fnv.New32a()
        h.Write([]byte(endpoint))
        endpointHash = fmt.Sprintf("%x", h.Sum32())
    }
    
    return fmt.Sprintf("rl:%s:%s:%s:%d", typePrefix, clientID, endpointHash, window)
}
```

**2. TTL Strategy:**

```
┌─────────────────────────────────────────────────────────────┐
│                    TTL BEST PRACTICES                       │
│                                                             │
│  Window Size    │ Recommended TTL  │ Reason                 │
│  ───────────────────────────────────────────────────────────│
│  1 second       │ 2-5 seconds      │ Quick cleanup          │
│  1 minute       │ 2-3 minutes      │ Buffer for processing  │
│  1 hour         │ 2 hours          │ Handle clock skew      │
│  1 day          │ 25-26 hours      │ Timezone safety        │
│                                                             │
│  Rule: TTL = window_size × 2 (minimum)                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**3. Sliding Window Counter vs Log:**

```
Scenario: 1000 users, 100 req/min limit, 60-second window

Sliding Window Log:
- Each user: up to 100 timestamps stored
- Memory per user: 100 × 80 bytes = 8 KB
- Total: 1000 × 8 KB = 8 MB

Sliding Window Counter:
- Each user: 2 integers (prev_count, curr_count)
- Memory per user: ~100 bytes
- Total: 1000 × 100 bytes = 100 KB

Savings: 98.75% reduction!
```

**4. Memory Monitoring:**

```go
func monitorRateLimiterMemory(ctx context.Context, rdb *redis.Client) {
    // Get memory usage for rate limit keys
    var cursor uint64
    var totalMemory int64
    var keyCount int64
    
    for {
        keys, newCursor, _ := rdb.Scan(ctx, cursor, "rl:*", 1000).Result()
        
        for _, key := range keys {
            mem, _ := rdb.MemoryUsage(ctx, key).Result()
            totalMemory += mem
            keyCount++
        }
        
        cursor = newCursor
        if cursor == 0 {
            break
        }
    }
    
    log.Printf("Rate limiter keys: %d, Total memory: %d bytes", keyCount, totalMemory)
}
```

**5. Redis Cluster Sharding:**

```
┌─────────────────────────────────────────────────────────────┐
│                  REDIS CLUSTER SHARDING                     │
│                                                             │
│  Hash slots: 0-16383 distributed across nodes               │
│                                                             │
│  Key: rl:{user123}:endpoint:window                          │
│       └── Hash tag ensures same user → same node            │
│                                                             │
│  Node 1 (slots 0-5460)      Node 2 (slots 5461-10922)       │
│  ┌─────────────────────┐    ┌─────────────────────┐         │
│  │ rl:{user001}:*      │    │ rl:{user501}:*      │         │
│  │ rl:{user002}:*      │    │ rl:{user502}:*      │         │
│  │ ...                 │    │ ...                 │         │
│  └─────────────────────┘    └─────────────────────┘         │
│                                                             │
│  Benefits:                                                  │
│  - Horizontal scaling                                       │
│  - Data locality for same user                              │
│  - Automatic rebalancing                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Advanced Topics

<a id="q18"></a>
### Q18: How do you implement multi-tier rate limiting?

**Answer:**

**Multi-Tier Architecture:**

```
┌─────────────────────────────────────────────────────────────┐
│                  MULTI-TIER RATE LIMITING                   │
│                                                             │
│  Request                                                    │
│     │                                                       │
│     ▼                                                       │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Tier 1: GLOBAL LIMIT                                │    │
│  │ 10,000 req/sec across all clients                   │    │
│  │ Purpose: Protect infrastructure                     │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │ PASS                              │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Tier 2: PER-CLIENT LIMIT                            │    │
│  │ 1,000 req/min per API key                           │    │
│  │ Purpose: Fair usage, prevent abuse                  │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │ PASS                              │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Tier 3: PER-ENDPOINT LIMIT                          │    │
│  │ /search: 100 req/min, /orders: 50 req/min           │    │
│  │ Purpose: Protect expensive operations               │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │ PASS                              │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Tier 4: PER-USER LIMIT                              │    │
│  │ 10 orders/hour per user                             │    │
│  │ Purpose: Business logic limits                      │    │
│  └──────────────────────┬──────────────────────────────┘    │
│                         │ PASS                              │
│                         ▼                                   │
│                    ALLOW REQUEST                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Implementation:**

```go
type Tier struct {
    Name       string
    KeyFunc    func(r *http.Request) string
    Limit      int
    Window     time.Duration
    Algorithm  string
}

type MultiTierLimiter struct {
    tiers  []Tier
    redis  *redis.Client
}

func NewMultiTierLimiter(redis *redis.Client) *MultiTierLimiter {
    return &MultiTierLimiter{
        redis: redis,
        tiers: []Tier{
            {
                Name:      "global",
                KeyFunc:   func(r *http.Request) string { return "global" },
                Limit:     10000,
                Window:    time.Second,
                Algorithm: "token_bucket",
            },
            {
                Name:      "per_client",
                KeyFunc:   func(r *http.Request) string { return r.Header.Get("X-API-Key") },
                Limit:     1000,
                Window:    time.Minute,
                Algorithm: "sliding_window",
            },
            {
                Name:      "per_endpoint",
                KeyFunc:   func(r *http.Request) string { 
                    return r.Header.Get("X-API-Key") + ":" + r.URL.Path 
                },
                Limit:     100,
                Window:    time.Minute,
                Algorithm: "sliding_window",
            },
        },
    }
}

func (m *MultiTierLimiter) Allow(r *http.Request) (bool, string) {
    for _, tier := range m.tiers {
        key := fmt.Sprintf("rl:%s:%s", tier.Name, tier.KeyFunc(r))
        
        allowed := m.checkTier(r.Context(), key, tier)
        if !allowed {
            return false, tier.Name // Return which tier blocked
        }
    }
    return true, ""
}

func (m *MultiTierLimiter) checkTier(ctx context.Context, key string, tier Tier) bool {
    switch tier.Algorithm {
    case "token_bucket":
        return m.tokenBucketCheck(ctx, key, tier.Limit, tier.Window)
    case "sliding_window":
        return m.slidingWindowCheck(ctx, key, tier.Limit, tier.Window)
    default:
        return m.fixedWindowCheck(ctx, key, tier.Limit, tier.Window)
    }
}
```

**Tier Configuration by Plan:**

```json
{
  "plans": {
    "free": {
      "global_limit": 100,
      "per_minute": 60,
      "per_endpoint": {
        "/api/search": 10,
        "/api/orders": 5
      }
    },
    "pro": {
      "global_limit": 1000,
      "per_minute": 600,
      "per_endpoint": {
        "/api/search": 100,
        "/api/orders": 50
      }
    },
    "enterprise": {
      "global_limit": 10000,
      "per_minute": 6000,
      "per_endpoint": {
        "/api/search": 1000,
        "/api/orders": 500
      }
    }
  }
}
```

---

<a id="q19"></a>
### Q19: What rate limit headers should APIs return?

**Answer:**

**Standard Headers:**

```
┌─────────────────────────────────────────────────────────────┐
│                 RATE LIMIT HEADERS                          │
│                                                             │
│  Standard (IETF draft-polli-ratelimit-headers):             │
│                                                             │
│  RateLimit-Limit:     100      # Max requests allowed       │
│  RateLimit-Remaining: 45       # Requests left in window    │
│  RateLimit-Reset:     30       # Seconds until reset        │
│                                                             │
│  Legacy (still widely used):                                │
│                                                             │
│  X-RateLimit-Limit:     100                                 │
│  X-RateLimit-Remaining: 45                                  │
│  X-RateLimit-Reset:     1706140860  # Unix timestamp        │
│                                                             │
│  On Rate Limit Exceeded (429):                              │
│                                                             │
│  Retry-After: 30               # Seconds to wait            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Implementation:**

```go
type RateLimitInfo struct {
    Limit     int
    Remaining int
    Reset     time.Time
}

func SetRateLimitHeaders(w http.ResponseWriter, info RateLimitInfo) {
    // Standard headers (IETF draft)
    w.Header().Set("RateLimit-Limit", strconv.Itoa(info.Limit))
    w.Header().Set("RateLimit-Remaining", strconv.Itoa(info.Remaining))
    w.Header().Set("RateLimit-Reset", strconv.Itoa(int(time.Until(info.Reset).Seconds())))
    
    // Legacy headers (for compatibility)
    w.Header().Set("X-RateLimit-Limit", strconv.Itoa(info.Limit))
    w.Header().Set("X-RateLimit-Remaining", strconv.Itoa(info.Remaining))
    w.Header().Set("X-RateLimit-Reset", strconv.FormatInt(info.Reset.Unix(), 10))
}

func RateLimitMiddleware(limiter *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            key := extractClientKey(r)
            
            allowed, info := limiter.Check(r.Context(), key)
            SetRateLimitHeaders(w, info)
            
            if !allowed {
                w.Header().Set("Retry-After", strconv.Itoa(int(time.Until(info.Reset).Seconds())))
                http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}
```

**Response Examples:**

```
Successful Request (200 OK):
─────────────────────────────────────────────────
HTTP/1.1 200 OK
Content-Type: application/json
RateLimit-Limit: 100
RateLimit-Remaining: 45
RateLimit-Reset: 30

{"data": "..."}


Rate Limited (429 Too Many Requests):
─────────────────────────────────────────────────
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
RateLimit-Limit: 100
RateLimit-Remaining: 0
RateLimit-Reset: 30
Retry-After: 30

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please retry after 30 seconds.",
    "retry_after": 30,
    "limit": 100,
    "reset_at": "2024-01-25T10:35:00Z"
  }
}
```

**Multi-Tier Headers:**

```
When multiple limits apply, show the most restrictive:

RateLimit-Limit: 100
RateLimit-Remaining: 5
RateLimit-Reset: 30
RateLimit-Policy: 100;w=60, 1000;w=3600

Policy format: {limit};w={window_seconds}
- 100 requests per 60 seconds
- 1000 requests per 3600 seconds (1 hour)
```

---

<a id="q20"></a>
### Q20: How should clients handle rate limit responses?

**Answer:**

**Client-Side Best Practices:**

```
┌─────────────────────────────────────────────────────────────┐
│              CLIENT RATE LIMIT HANDLING                     │
│                                                             │
│  1. Read rate limit headers on every response               │
│  2. Track remaining quota locally                           │
│  3. Implement exponential backoff on 429                    │
│  4. Add jitter to prevent thundering herd                   │
│  5. Queue requests when approaching limit                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Retry Strategy with Exponential Backoff:**

```
┌─────────────────────────────────────────────────────────────┐
│              EXPONENTIAL BACKOFF WITH JITTER                │
│                                                             │
│  Attempt 1: Wait 1s  + random(0-500ms)  = 1.0-1.5s          │
│  Attempt 2: Wait 2s  + random(0-500ms)  = 2.0-2.5s          │
│  Attempt 3: Wait 4s  + random(0-500ms)  = 4.0-4.5s          │
│  Attempt 4: Wait 8s  + random(0-500ms)  = 8.0-8.5s          │
│  Attempt 5: Wait 16s + random(0-500ms)  = 16.0-16.5s        │
│  Max wait:  60s (cap)                                       │
│                                                             │
│  Formula: min(base × 2^attempt + jitter, max_delay)         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Go Client Implementation:**

```go
type RateLimitedClient struct {
    client        *http.Client
    baseURL       string
    remaining     int
    resetTime     time.Time
    mu            sync.Mutex
    maxRetries    int
    baseDelay     time.Duration
    maxDelay      time.Duration
}

func (c *RateLimitedClient) Do(req *http.Request) (*http.Response, error) {
    var lastErr error
    
    for attempt := 0; attempt <= c.maxRetries; attempt++ {
        // Check if we should wait before making request
        c.mu.Lock()
        if c.remaining <= 0 && time.Now().Before(c.resetTime) {
            waitTime := time.Until(c.resetTime)
            c.mu.Unlock()
            time.Sleep(waitTime)
        } else {
            c.mu.Unlock()
        }
        
        resp, err := c.client.Do(req)
        if err != nil {
            lastErr = err
            continue
        }
        
        // Update rate limit info from headers
        c.updateRateLimitInfo(resp)
        
        if resp.StatusCode != http.StatusTooManyRequests {
            return resp, nil
        }
        
        // Handle 429 - Rate Limited
        resp.Body.Close()
        
        delay := c.calculateDelay(resp, attempt)
        time.Sleep(delay)
        lastErr = fmt.Errorf("rate limited")
    }
    
    return nil, fmt.Errorf("max retries exceeded: %w", lastErr)
}

func (c *RateLimitedClient) updateRateLimitInfo(resp *http.Response) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    if remaining := resp.Header.Get("RateLimit-Remaining"); remaining != "" {
        c.remaining, _ = strconv.Atoi(remaining)
    }
    
    if reset := resp.Header.Get("RateLimit-Reset"); reset != "" {
        seconds, _ := strconv.Atoi(reset)
        c.resetTime = time.Now().Add(time.Duration(seconds) * time.Second)
    }
}

func (c *RateLimitedClient) calculateDelay(resp *http.Response, attempt int) time.Duration {
    // Use Retry-After header if present
    if retryAfter := resp.Header.Get("Retry-After"); retryAfter != "" {
        if seconds, err := strconv.Atoi(retryAfter); err == nil {
            return time.Duration(seconds) * time.Second
        }
    }
    
    // Exponential backoff with jitter
    delay := c.baseDelay * time.Duration(1<<attempt)
    if delay > c.maxDelay {
        delay = c.maxDelay
    }
    
    // Add jitter (0-25% of delay)
    jitter := time.Duration(rand.Int63n(int64(delay / 4)))
    return delay + jitter
}
```

**Request Queuing for High-Volume Clients:**

```go
type RequestQueue struct {
    client     *RateLimitedClient
    queue      chan *QueuedRequest
    rateLimit  int
    window     time.Duration
}

type QueuedRequest struct {
    Request  *http.Request
    Response chan *http.Response
    Error    chan error
}

func (q *RequestQueue) Start() {
    // Calculate delay between requests to stay under limit
    delay := q.window / time.Duration(q.rateLimit)
    
    ticker := time.NewTicker(delay)
    defer ticker.Stop()
    
    for {
        select {
        case req := <-q.queue:
            <-ticker.C // Wait for next slot
            
            go func(qr *QueuedRequest) {
                resp, err := q.client.Do(qr.Request)
                if err != nil {
                    qr.Error <- err
                } else {
                    qr.Response <- resp
                }
            }(req)
        }
    }
}

func (q *RequestQueue) Enqueue(req *http.Request) (*http.Response, error) {
    qr := &QueuedRequest{
        Request:  req,
        Response: make(chan *http.Response, 1),
        Error:    make(chan error, 1),
    }
    
    q.queue <- qr
    
    select {
    case resp := <-qr.Response:
        return resp, nil
    case err := <-qr.Error:
        return nil, err
    }
}
```

**Summary of Client Best Practices:**

| Practice | Benefit |
|----------|---------|
| Read headers | Know limits before hitting them |
| Local tracking | Avoid unnecessary 429s |
| Exponential backoff | Reduce server load during recovery |
| Jitter | Prevent thundering herd |
| Request queue | Stay within limits proactively |
| Circuit breaker | Fail fast when service degraded |

---

[← Back to System Design Index](README.md)

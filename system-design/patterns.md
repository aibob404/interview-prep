# System Design Patterns

## 1. Caching

### Cache-Aside (Lazy Loading)
Application manages cache directly:
```
Read:  Check cache → miss → read DB → populate cache → return
Write: Write DB → invalidate (or update) cache
```
- Cache only stores what's actually requested
- Cold start: first request always hits DB
- Cache invalidation complexity: when to evict?

### Read-Through
Cache sits in front of DB; on miss, cache fetches from DB automatically:
```
Read:  Check cache → miss → cache reads DB → stores → returns
Write: Goes through cache to DB (write-through) or directly to DB (cache-aside write)
```
- Simpler application code
- Used in: Hibernate second-level cache, Redis with read-through clients

### Write-Through
Every write goes to both cache and DB synchronously:
- No stale data in cache
- Higher write latency (waits for DB)
- Writes for data never read still go to cache (wasted)

### Write-Behind (Write-Back)
Write to cache immediately, persist to DB asynchronously:
- Very low write latency
- Risk of data loss if cache fails before flush
- Good for: counters, analytics, non-critical data

### Cache Invalidation Strategies
```
TTL (Time-to-live):      key expires after N seconds — simple, may serve stale
Event-based:             publish event on write → cache subscriber deletes key
Write-through:           update cache on every DB write
Cache-aside invalidation: delete cache key on write, re-populate on next read
```

### Thundering Herd
When cache expires, many requests hit DB simultaneously:
```java
// Solution 1: Lock on cache miss — only one request fetches from DB
String value = cache.get(key);
if (value == null) {
    Lock lock = locks.get(key);
    lock.lock();
    try {
        value = cache.get(key); // double-check after lock
        if (value == null) {
            value = db.fetch(key);
            cache.set(key, value, TTL);
        }
    } finally { lock.unlock(); }
}

// Solution 2: Probabilistic early expiration — recompute before TTL expires
// Solution 3: Background refresh — keep serving stale while refreshing asynchronously
```

### Cache Levels
| Level | Technology | Latency | Size |
|-------|-----------|---------|------|
| In-process | Caffeine, Guava | <1ms | Limited by JVM heap |
| Distributed | Redis, Memcached | ~1ms | Virtually unlimited |
| CDN | Cloudflare, Akamai | <10ms | Edge nodes globally |

---

## 2. Message Queues

### When to Use
- **Decoupling:** producer doesn't know about consumers
- **Backpressure:** consumer sets its own pace
- **Retry:** failed messages can be re-queued
- **Fan-out:** one event → multiple consumers
- **Async processing:** fire-and-forget for non-critical paths

### Kafka Architecture
```
Topics → Partitions → Leaders/Replicas
Consumer Group → Partition per consumer instance
Offset tracking → per partition per consumer group
```

```
Topic "orders" with 3 partitions:
  P0: [msg1, msg4, msg7, ...]
  P1: [msg2, msg5, msg8, ...]
  P2: [msg3, msg6, msg9, ...]

Consumer Group "invoice-service": 3 instances
  Instance A → P0
  Instance B → P1
  Instance C → P2
  (each partition consumed by exactly one instance in a group)
```

**Key properties:**
- Messages retained for configurable period (not deleted on consume)
- Replay: reset offset to re-process messages
- High throughput: sequential disk writes, batching, compression
- Ordering: guaranteed within a partition, not across partitions

**Partitioning key:** determines which partition a message goes to. Same key → same partition → order preserved.

### RabbitMQ Architecture
```
Producer → Exchange → (binding rules) → Queue → Consumer
```
- **Direct exchange:** route by exact routing key
- **Fanout exchange:** broadcast to all bound queues
- **Topic exchange:** pattern matching (`order.*`, `*.created`)
- **Headers exchange:** route by message headers

### Kafka vs RabbitMQ

| | Kafka | RabbitMQ |
|---|---|---|
| Model | Log (pull) | Queue (push) |
| Retention | Days/weeks | Until consumed |
| Ordering | Per partition | Per queue |
| Throughput | Very high | High |
| Replay | Yes | No (once consumed) |
| Use case | Event streaming, analytics | Task queue, RPC |

### Exactly-Once Semantics
- **At-most-once:** may lose messages (fire and forget)
- **At-least-once:** may deliver duplicates (acknowledge after processing)
- **Exactly-once:** Kafka transactions + idempotent producer + transactional consumers

**Idempotent consumers** (practical approach):
```java
// Track processed message IDs
public void handleMessage(Message msg) {
    if (processedIds.contains(msg.getId())) return; // deduplicate
    process(msg);
    processedIds.add(msg.getId());
}
```

---

## 3. Rate Limiting

### Token Bucket Algorithm
Tokens accumulate at a fixed rate up to a maximum. Each request consumes a token. Allows bursts up to bucket capacity.

```
Bucket: max 100 tokens, refill 10/sec
Request arrives: take 1 token (if available) or reject
Burst: 100 requests immediately if bucket is full
Sustained: 10 req/sec after burst
```

### Sliding Window Log
Track timestamps of recent requests. Count requests in window. Most accurate, high memory use.

### Leaky Bucket
Requests enter a queue (bucket), processed at fixed rate (leak). Smooths traffic, no bursts. Used in: traffic shaping, API gateways.

### Fixed Window Counter
Increment counter per time window (e.g., per minute). Simple but boundary problem: burst at window edge can double rate.

### Distributed Rate Limiting
```java
// Redis-based token bucket with Lua script (atomic)
local key = KEYS[1]
local max_tokens = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local requested = tonumber(ARGV[4])

local tokens = tonumber(redis.call("GET", key) or max_tokens)
-- calculate tokens added since last access ...
if tokens >= requested then
    redis.call("SET", key, tokens - requested)
    return 1 -- allowed
else
    return 0 -- rejected
end
```

---

## 4. Circuit Breaker

States: **Closed** (normal) → **Open** (failing) → **Half-Open** (probe)

```
[Closed]  ─── failures exceed threshold ──→ [Open]
               (reject all requests immediately)
[Open]    ─── timeout expires ──→ [Half-Open]
               (allow one probe request)
[Half-Open] ─── probe succeeds ──→ [Closed]
[Half-Open] ─── probe fails ──→ [Open]
```

```java
// Using Resilience4j
@Bean
public CircuitBreakerConfig circuitBreakerConfig() {
    return CircuitBreakerConfig.custom()
        .failureRateThreshold(50)           // open if 50% fail
        .waitDurationInOpenState(Duration.ofSeconds(30))
        .slidingWindowSize(10)              // evaluate last 10 calls
        .minimumNumberOfCalls(5)
        .permittedNumberOfCallsInHalfOpenState(3)
        .build();
}

@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResult charge(PaymentRequest req) {
    return paymentClient.charge(req);
}

public PaymentResult paymentFallback(PaymentRequest req, Exception e) {
    return PaymentResult.queued(req.getOrderId()); // degraded mode
}
```

---

## 5. Retry with Exponential Backoff

```java
// Base: 100ms, multiplier: 2, max: 30s, jitter: ±25%
// Attempt 1: 100ms
// Attempt 2: 200ms ± jitter
// Attempt 3: 400ms ± jitter
// Attempt 4: 800ms ± jitter

// Jitter prevents synchronized retries (thundering herd after outage)

RetryConfig config = RetryConfig.custom()
    .maxAttempts(5)
    .waitDuration(Duration.ofMillis(100))
    .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(100, 2.0, 0.5, 30_000))
    .retryOnException(ex -> ex instanceof ServiceUnavailableException)
    .build();
```

---

## 6. Outbox Pattern

Solves the dual-write problem: atomically save to DB AND publish an event:

```java
// Problem: write to DB and publish to Kafka — what if publish fails?
orderRepo.save(order);         // TX committed
eventBus.publish(OrderPlaced); // fails → event lost!

// Solution: write event to outbox table in same transaction
@Transactional
public void placeOrder(Order order) {
    orderRepo.save(order);
    outboxRepo.save(new OutboxEvent("ORDER_PLACED", order.toJson()));
    // Both in same DB transaction — atomically committed
}

// Separate process: poll outbox table, publish to Kafka, mark published
// (or use Debezium CDC to capture DB changes → publish to Kafka)
```

---

## 7. CQRS (Command Query Responsibility Segregation)

Separate write model (commands) from read model (queries):

```
Write side:                    Read side:
  Command → Handler              Query → Handler
       ↓                               ↑
  Domain Model                   Read Model (view)
       ↓         sync/async            ↑
  Event Store  ──────────────→  Projection
```

**When to use:**
- Read and write models have very different shapes
- Read performance must be optimized independently
- Complex domain with rich business logic

**When NOT to use:**
- Simple CRUD — adds enormous complexity without benefit
- Most applications

---

## 8. Bulkhead Pattern

Isolate components so failure in one doesn't cascade:

```java
// Thread pool bulkhead — payment service has its own pool
@Bulkhead(name = "paymentService",
          type = Bulkhead.Type.THREADPOOL,
          fallbackMethod = "paymentFallback")
public CompletableFuture<PaymentResult> charge(PaymentRequest req) {
    return paymentClient.chargeAsync(req);
}

// Semaphore bulkhead — limit concurrent calls
BulkheadConfig config = BulkheadConfig.custom()
    .maxConcurrentCalls(20)
    .maxWaitDuration(Duration.ofMillis(500))
    .build();
```

Like bulkheads in a ship — a hole in one section doesn't sink the whole ship.

---

## 9. Service Discovery

### Client-Side Discovery
Client queries service registry (Eureka, Consul), chooses instance, calls directly:
```
Client → Registry: "where is order-service?" → [IP1, IP2, IP3]
Client picks IP2 → calls directly
```

### Server-Side Discovery
Client calls load balancer; load balancer queries registry and routes:
```
Client → Load Balancer → Registry → pick instance → forward
```

---

## 10. Common Interview Questions

| Question | Answer |
|----------|--------|
| What is the difference between cache-aside and write-through caching? | **Cache-aside** (lazy loading): the application checks the cache first; on a miss it reads from the DB and populates the cache. The app manages cache writes explicitly. Stale data is possible until TTL expires or manual invalidation. Simple and effective for read-heavy workloads. **Write-through**: every write goes to the cache and DB atomically (or the cache writes through to the DB). The cache is always consistent with the DB, but every write pays the cache write cost, even for data that is rarely read. Use cache-aside as the default; write-through when read-after-write consistency is critical. |
| What is a cache stampede and how do you prevent it? | A cache stampede (thundering herd) happens when many concurrent requests experience a cache miss simultaneously — e.g., when a popular cache entry expires — and all fire database queries at once, overwhelming the DB. Prevention strategies: (1) **Mutex/lock on miss** — one request rebuilds, others wait; (2) **Probabilistic early expiration** (XFetch) — a request randomly decides to refresh slightly before TTL expires, spreading the refresh across time; (3) **Background refresh** — a background job refreshes entries before they expire; (4) **Stale-while-revalidate** — serve the stale value immediately while a background task refreshes it. |
| What is the Outbox pattern and what problem does it solve? | The problem: you need to update your database and publish a message to Kafka/RabbitMQ atomically — but database transactions and message brokers are different systems and cannot share a transaction. The Outbox pattern: write the event to an `outbox` table in the **same database transaction** as the business data change. A separate **relay process** (Debezium CDC, or a polling job) reads committed outbox rows and publishes them to the broker, then marks them as sent. This guarantees at-least-once delivery without dual-write races, using only the database's ACID guarantees. |
| What are the three states of a circuit breaker and how do they work? | **Closed**: the circuit is operational; requests pass through normally. A failure counter tracks errors. When failures exceed a threshold within a time window, the circuit trips to Open. **Open**: all requests fail immediately without attempting the downstream call. After a configured timeout, the circuit moves to Half-Open. **Half-Open**: a limited number of probe requests are allowed through. If they succeed, the circuit resets to Closed. If they fail, it returns to Open. This prevents cascading failures by giving a struggling downstream service time to recover instead of being hammered with requests during an outage. |
| When would you choose Kafka over RabbitMQ? | **Kafka**: choose when you need high throughput (millions of events/sec), message **replay** (consumers can rewind and re-read events), long retention (days/weeks), event sourcing, or multiple independent consumer groups reading the same stream. Kafka is a distributed commit log. **RabbitMQ**: choose for task queues with complex routing (topic/fanout exchanges), **request-reply patterns (RPC)**, per-message acknowledgment and re-queuing, or when messages should be consumed and discarded. RabbitMQ is a traditional message broker. Key rule: if you need consumers to replay events or have multiple independent consumers, Kafka. If you need smart routing with guaranteed single delivery, RabbitMQ. |
| What is the difference between at-least-once and exactly-once message delivery? | **At-least-once** (ALO): the broker ensures the message is delivered, but may deliver it more than once if an acknowledgment is lost and the broker retries. The consumer must be **idempotent** to handle duplicates safely. **Exactly-once** (EO): the message is delivered and processed exactly one time, even under retries. In Kafka, EO delivery requires producer transactions (`enable.idempotence=true` + `transactional.id`) and a transactional consumer that commits offsets atomically with the processing result. EO is complex and has throughput implications — prefer idempotent consumers + ALO in most cases. |
| What is the bulkhead pattern and when do you apply it? | The bulkhead pattern isolates different workloads into separate resource pools — thread pools, connection pools, or service instances — so that failures or overload in one area do not exhaust resources shared with others. Named after the watertight compartments in a ship hull: one flooded compartment does not sink the ship. Example: maintain separate HTTP thread pools for "critical checkout API calls" and "non-critical recommendation calls." If recommendations become slow, they consume their own pool — checkout remains unaffected. Implemented in Resilience4j via `Bulkhead` and `ThreadPoolBulkhead`. |
| What is CQRS and what are its tradeoffs? | CQRS (Command Query Responsibility Segregation) separates the write model (commands that change state) from the read model (queries that return data). The read model can be a denormalized projection — a separate table, a Redis cache, or an Elasticsearch index — optimized purely for querying. This allows read and write sides to scale, evolve, and be optimized independently. Tradeoffs: increased complexity (two models to maintain), eventual consistency between write and read sides (reads may lag slightly). Use CQRS when read and write traffic patterns are very different, or when queries require data shapes that differ significantly from the normalized write model. |
| How does distributed rate limiting work? | Each incoming request increments a counter stored in a centralized store like Redis. The most robust approach uses the **token bucket** algorithm: a bucket holds up to N tokens; each request consumes one token; tokens are refilled at a fixed rate. A Lua script executes the check-and-decrement atomically in Redis, preventing race conditions. For user-level rate limiting, the key includes the user ID. For IP-level limiting, it includes the client IP. Challenges: Redis latency adds to every request; a Redis failure can either block all requests (fail-closed) or allow all (fail-open) — choose based on risk tolerance. |
| What is exponential backoff with jitter and why is jitter important? | **Exponential backoff** retries a failed request with increasing delays: 1s, 2s, 4s, 8s, 16s... This avoids hammering a struggling service. **Jitter** adds randomness to the delay (e.g., `random(0, delay)`). Without jitter, all retrying clients that failed at the same time back off for the same duration and then all retry simultaneously — creating a new thundering herd at each retry wave. Jitter spreads retries over time, smoothing the load on the recovering service. The "full jitter" strategy (`random(0, cap)`) is most effective; "decorrelated jitter" (`random(base, prevDelay * 3)`) also works well. |

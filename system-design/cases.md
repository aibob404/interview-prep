# System Design Cases

## Template: How to Answer

1. **Requirements** (2 min): functional + non-functional
2. **Capacity estimation** (3 min): QPS, storage, bandwidth
3. **High-level diagram** (5 min): main components
4. **Data model** (5 min): key tables/documents
5. **API design** (3 min): main endpoints
6. **Deep dive** (10 min): the interesting parts — bottlenecks, trade-offs
7. **Failure scenarios** (2 min): what breaks and how

---

## 1. URL Shortener (bit.ly)

### Requirements
- Shorten a URL → short code
- Redirect short URL to original
- Optional: analytics, expiry, custom alias
- Scale: 100M URLs/day written, 10B redirects/day

### Capacity Estimation
```
Write QPS:  100M / 86400 ≈ 1,200 writes/sec
Read QPS:   10B / 86400 ≈ 115,000 reads/sec (read:write ≈ 100:1)
Storage:    100M URLs/day × 365 × 5 years = 182B URLs
            Each URL: ~500 bytes → 182B × 500 = 91 TB
```

### High-Level Design
```
Client → CDN/Cache → Redirect Service → Cache (Redis) → DB (Postgres)
                  ↑
            Short URL API → ID Generator → DB
```

### Short Code Generation
```
Option 1: Base62 encode a unique ID
  ID = 12345678 → base62(12345678) = "5Rk3P" (7 chars = 62^7 = 3.5T URLs)

Option 2: MD5 hash of long URL → take first 7 chars
  Risk: collision; if collision, try next 7 chars

Option 3: Distributed ID generator (Snowflake)
  64-bit: timestamp (41b) + datacenter (5b) + machine (5b) + sequence (12b)
  Globally unique, sortable, no coordination needed
```

### Data Model
```sql
CREATE TABLE urls (
    id          BIGINT PRIMARY KEY,
    short_code  CHAR(7) UNIQUE NOT NULL,
    long_url    TEXT NOT NULL,
    created_at  TIMESTAMP NOT NULL,
    expires_at  TIMESTAMP,
    user_id     BIGINT,
    click_count BIGINT DEFAULT 0
);

CREATE INDEX idx_urls_short_code ON urls(short_code);
```

### Redirect Flow
```
GET /abc123
  → Check Redis cache (short_code → long_url), TTL 1 hour
  → Cache hit: 301/302 redirect (301 cacheable, 302 not)
  → Cache miss: DB lookup → populate cache → redirect
  → Async: log click event to Kafka for analytics
```

**301 vs 302:**
- 301 Permanent: browser caches redirect → fewer server hits, but can't change destination
- 302 Temporary: browser always asks server → enables analytics counting, A/B testing

### Bottlenecks & Solutions
- Read bottleneck → Redis cache (hot URLs stay in memory)
- Write bottleneck → shard by short_code hash
- Analytics → write to Kafka, process asynchronously with Flink/Spark

---

## 2. Real-Time Chat (WhatsApp/Slack)

### Requirements
- 1:1 and group messaging
- Online/offline status
- Message delivery receipts (sent, delivered, read)
- Scale: 1B users, 100M DAU, 1B messages/day

### Capacity Estimation
```
Message QPS:   1B / 86400 ≈ 12,000 msg/sec
Storage:       1B msg/day × 100 bytes × 365 days = 3.6 TB/year
Connection:    100M concurrent WebSocket connections
```

### Connection Architecture
```
Client ──WebSocket──→ Chat Server (stateful!)
                          ↓ publish
                      Message Queue (Kafka)
                          ↓ consume
                      Chat Server (recipient's server)
                          ↓
                      Client (WebSocket push)
```

**Why WebSocket?** Persistent bidirectional connection — server can push without polling. HTTP long-polling is an alternative but less efficient.

### Message Delivery
```
1. Sender sends message → Chat Server A
2. Server A persists to DB (Cassandra)
3. Server A publishes to Kafka topic (partitioned by conversation_id)
4. Recipient's Chat Server B consumes from Kafka
5. Server B pushes to recipient via WebSocket
6. Recipient acknowledges → Server B sends "delivered" receipt back
7. Recipient reads → "read" receipt sent

Offline case:
  Recipient not connected → message stored in DB
  On reconnect → client fetches unread messages (REST API)
  Push notification via FCM/APNs
```

### Data Model (Cassandra)
```cql
-- Messages by conversation (read pattern: all messages in a conversation)
CREATE TABLE messages (
    conversation_id UUID,
    created_at      TIMESTAMP,
    message_id      UUID,
    sender_id       UUID,
    content         TEXT,
    type            TEXT,  -- text, image, video
    status          TEXT,  -- sent, delivered, read
    PRIMARY KEY ((conversation_id), created_at, message_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- User inbox (which conversations have new messages)
CREATE TABLE user_conversations (
    user_id         UUID,
    conversation_id UUID,
    last_message_at TIMESTAMP,
    unread_count    INT,
    PRIMARY KEY (user_id, last_message_at, conversation_id)
) WITH CLUSTERING ORDER BY (last_message_at DESC);
```

**Why Cassandra?** Write-heavy, high throughput, partition by conversation_id for scalability.

### Online Status
```
User connects → write status=online, timestamp to Redis (TTL 60s)
User sends heartbeat every 30s → refresh TTL
User disconnects → TTL expires → status=offline

// Check status: Redis O(1) lookup
// Fan-out to friends: publish to Pub/Sub, friends' servers subscribe
```

---

## 3. Payment System

### Requirements
- Process payments between users (or user to merchant)
- Idempotency — never double-charge
- ACID consistency — money can't disappear or be created
- Scale: 1M transactions/day (10 TPS average, 100 TPS peak)

### Core Principles
- **Idempotency:** each payment request has unique idempotency key; server stores result; duplicate request returns same result
- **Exactly-once processing:** critical — a payment must execute exactly once
- **Audit trail:** every state change logged immutably

### Architecture
```
Client → API Gateway → Payment Service → Payment DB (Postgres)
                           ↓
                    External Payment Processor (Stripe, Adyen)
                           ↓
                    Webhook Handler → Event Queue → Notification Service
```

### Double-Entry Bookkeeping
```sql
-- Never update balances directly; record debits and credits
CREATE TABLE ledger_entries (
    id              BIGINT PRIMARY KEY,
    account_id      BIGINT NOT NULL,
    transaction_id  BIGINT NOT NULL,
    type            TEXT CHECK (type IN ('DEBIT', 'CREDIT')),
    amount          NUMERIC(19,4) NOT NULL,
    balance_after   NUMERIC(19,4) NOT NULL,
    created_at      TIMESTAMP NOT NULL
);

-- Balance = SUM of all entries for account_id
-- Every transfer = 1 DEBIT + 1 CREDIT in same transaction
```

### Idempotency Pattern
```java
@Transactional
public PaymentResult processPayment(PaymentRequest req, String idempotencyKey) {
    // Check if already processed
    Optional<PaymentResult> existing = idempotencyRepo.find(idempotencyKey);
    if (existing.isPresent()) return existing.get(); // return same result

    // Process
    PaymentResult result = doProcessPayment(req);

    // Store result with idempotency key
    idempotencyRepo.save(idempotencyKey, result, Duration.ofDays(7));
    return result;
}
```

### Payment State Machine
```
INITIATED → PROCESSING → COMPLETED
                       → FAILED
                       → REQUIRES_ACTION (3DS, etc.)
         → CANCELLED (before processing)
COMPLETED → REFUNDING → REFUNDED
```

---

## 4. Notification Service

### Requirements
- Multi-channel: email, SMS, push notification
- Reliable delivery — at-least-once
- Templated messages with personalization
- Rate limiting per user per channel
- Scale: 10M notifications/hour

### Architecture
```
Trigger (order, payment, etc.)
    ↓
Notification API Service
    ↓
Message Queue (Kafka, partitioned by user_id)
    ↓ consumers
Channel Workers:
    Email Worker    → SendGrid/SES
    SMS Worker      → Twilio
    Push Worker     → FCM/APNs
    ↓
Delivery Tracking DB
```

### Template Engine
```java
// Template stored in DB or S3
"Hello {{firstName}}, your order #{{orderId}} has been {{status}}."

// Rendered at send time with user data
Map<String, String> variables = Map.of(
    "firstName", user.getFirstName(),
    "orderId", order.getId().toString(),
    "status", order.getStatus().getDisplayName()
);
String rendered = templateEngine.render(templateId, variables);
```

### Reliability
- **Retry:** exponential backoff on channel failure
- **Dead Letter Queue:** failed after N retries → DLQ → alert + manual review
- **Idempotency:** deduplicate by notification_id at channel worker level
- **Rate limiting:** Redis counter per `user:channel:hour` — max N notifications/hour

### Priority Queues
```
HIGH priority:   OTP codes, security alerts → separate queue, dedicated workers
MEDIUM priority: Order confirmations, payment receipts
LOW priority:    Marketing, newsletters → bulk processing, lower cost provider
```

---

## 5. Design Checklist for Any System

### Functional
- [ ] What exactly does the system do?
- [ ] What are the API endpoints?
- [ ] What is the read/write ratio?

### Non-Functional
- [ ] Expected QPS, peak QPS
- [ ] Latency requirements (p99)
- [ ] Availability requirement (99.9%? 99.99%?)
- [ ] Data durability requirements
- [ ] Consistency requirements (strong? eventual?)

### Data
- [ ] What data needs to be stored?
- [ ] What are the read patterns? (by ID? range query? full-text?)
- [ ] What is the data size?
- [ ] SQL vs NoSQL? Why?

### Reliability
- [ ] What happens when a service goes down?
- [ ] What happens when the DB goes down?
- [ ] What happens during a network partition?
- [ ] How do you handle duplicate requests?
- [ ] How do you recover from failures?

### Scalability
- [ ] Where are the bottlenecks?
- [ ] What can be cached?
- [ ] What can be sharded?
- [ ] What can be async?

# System Design Fundamentals

## 1. How to Approach a System Design Interview

1. **Clarify requirements** (5 min): functional, non-functional (scale, latency, availability)
2. **Estimate scale** (5 min): users, QPS, storage, bandwidth
3. **High-level design** (10 min): components, data flow, APIs
4. **Deep dive** (15 min): data model, bottlenecks, trade-offs
5. **Wrap up** (5 min): failure scenarios, monitoring, evolution

---

## 2. CAP Theorem

A distributed system can guarantee **at most 2 of 3**:
- **Consistency (C):** every read returns the most recent write
- **Availability (A):** every request gets a (non-error) response
- **Partition Tolerance (P):** system works despite network partition

**Network partitions are unavoidable** in production — so the real choice is **CP vs AP**:

| System | Type | Behavior during partition |
|--------|------|--------------------------|
| PostgreSQL (single node) | CA | Not distributed — no partition |
| Apache Zookeeper | CP | Rejects writes if quorum lost |
| etcd, Consul | CP | Rejects writes during partition |
| Cassandra, DynamoDB | AP | Serves potentially stale data |
| Redis Cluster | AP (default) | Tunable with WAIT |
| MySQL (with replication) | CP or AP | Depends on sync/async replication |

> CAP is a simplification. PACELC extends it: during normal operation (no partition), there's still a trade-off between **latency** and **consistency**.

---

## 3. Consistency Models

From strongest to weakest:

### Linearizability (Strict Consistency)
Reads always return the most recent write. Every operation appears to execute instantaneously at a single point in time.
- Expensive (requires coordination)
- Used in: leader-based consensus (Raft, Paxos), ZooKeeper

### Sequential Consistency
All operations appear in some sequential order consistent with each thread's program order. Weaker than linearizability — no real-time guarantee.

### Causal Consistency
Operations that are causally related are seen in the same order by all nodes. Concurrent (causally unrelated) operations may be seen in different orders.
- "Your post will appear after you wrote it" — but two independent posts may appear in different order for different users

### Eventual Consistency
Given no new writes, all replicas will converge to the same value. Reads may return stale data.
- Used in: DNS, shopping carts, social feeds, S3
- High availability, low latency

### Read-Your-Writes Consistency
After you write, you'll always read your own writes. Others may still see old data.

---

## 4. Distributed Transactions

### Two-Phase Commit (2PC)
```
Phase 1 (Prepare): Coordinator asks all participants to prepare
Phase 2 (Commit):  If all say "ready" → commit; else → abort

Problems:
- Coordinator single point of failure (blocking)
- Long-held locks during prepare phase
- Network partition can leave participants in uncertain state
```

### Saga Pattern (Preferred for Microservices)
Break transaction into sequence of local transactions; each publishes event or message for the next step. On failure, execute **compensating transactions**:

```
PlaceOrder → ReserveInventory → ChargePayment → SendConfirmation
     ↓ (failure)
CancelConfirmation ← RefundPayment ← ReleaseInventory ← CancelOrder
```

**Choreography:** services react to events (decoupled, harder to trace)
**Orchestration:** central saga orchestrator commands each step (easier to trace, single point of control)

---

## 5. Replication

### Leader-Follower (Primary-Replica)
```
Writes → Leader → replicates to → Followers
Reads  → Leader (consistent) or Followers (stale possible)
```
- **Synchronous replication:** write blocks until at least one follower confirms → no data loss, higher latency
- **Asynchronous replication:** write returns immediately → possible data loss on leader failure, lower latency

### Multi-Leader Replication
Multiple nodes accept writes. Conflict resolution required (last-write-wins, CRDTs, application logic). Used in: multi-datacenter setups, collaborative editing.

### Leaderless (Quorum)
Any node accepts writes. Quorum: W + R > N ensures at least one node has latest data.
- N=3, W=2, R=2: tolerates 1 failure, R+W=4>3
- **Sloppy quorum:** during partition, writes accepted by available nodes (hinted handoff later) — favors availability

---

## 6. Partitioning / Sharding

### Range Partitioning
Data split by key range (e.g., A-M, N-Z). Simple, supports range queries. Risk: **hot spots** (all writes go to recent range, e.g., time-series data).

### Hash Partitioning
`partition = hash(key) % num_partitions`. Even distribution, eliminates hot spots. Destroys key ordering — no range queries.

### Consistent Hashing
Keys and nodes placed on a virtual ring. Each key assigned to the next node clockwise. Adding/removing a node only rebalances ~1/N of keys (vs hash mod which rebalances everything).
- Used in: Cassandra, DynamoDB, load balancers, CDNs

```
Virtual Ring:
  0 ─── Node A ─── Node B ─── Node C ─── 360
        key1   key2        key3
key1 → Node A; key2 → Node B; key3 → Node C
Adding Node D between B and C: only key3 moves
```

### Hotspot Mitigation
- Append random suffix to hot key: `celeb_userId_01`, `celeb_userId_02`
- Read-aside caching for hot rows
- Time-based partitioning with multiple recent partitions

---

## 7. Load Balancing

### Algorithms
| Algorithm | Use case |
|-----------|---------|
| Round Robin | Equal servers, stateless |
| Weighted Round Robin | Servers with different capacities |
| Least Connections | Long-lived connections (WebSockets) |
| IP Hash | Session affinity (sticky sessions) |
| Random | Simple, surprisingly effective |
| Resource-based | Route to server with most CPU/memory available |

### Load Balancer Layers
- **L4 (Transport):** TCP/UDP level, fast, no application context (HAProxy, NLB)
- **L7 (Application):** HTTP level, routing by URL/header, SSL termination, compression (Nginx, ALB)

---

## 8. Availability and Reliability

### Availability Numbers
| Availability | Downtime/year | Downtime/month |
|-------------|-------------|--------------|
| 99% | 3.65 days | 7.2 hours |
| 99.9% (three 9s) | 8.7 hours | 43 minutes |
| 99.99% (four 9s) | 52 minutes | 4.3 minutes |
| 99.999% (five 9s) | 5 minutes | 26 seconds |

### SLA, SLO, SLI
- **SLA (Agreement):** contract with customer; breach has consequences
- **SLO (Objective):** internal target (must be tighter than SLA)
- **SLI (Indicator):** the actual measured metric (latency p99, error rate)

### Error Budget
`Error budget = 1 - SLO`
Team has 0.1% budget to burn before SLO breach. When budget is low, reduce risk (no new deployments).

---

## 9. Scalability Patterns

### Vertical Scaling
Add more CPU/RAM to existing server. Simple, no code changes. Hard ceiling — expensive at the top.

### Horizontal Scaling
Add more servers. Requires stateless services, load balancer, shared session storage.

### Read Scaling
- **Read replicas:** offload reads from primary
- **Caching:** serve reads from cache (Redis, CDN)
- **CQRS:** separate read model optimized for queries

### Write Scaling
- **Sharding:** distribute writes across nodes
- **Write-ahead log + async processing:** accept writes fast, process asynchronously
- **Command buffering:** batch small writes

---

## 10. Common Interview Questions

| Question | Answer |
|----------|--------|
| Explain the CAP theorem and its practical implications | CAP states that a distributed system can guarantee at most two of: **Consistency** (every read returns the most recent write), **Availability** (every request gets a non-error response), and **Partition tolerance** (the system continues despite network partitions). Since network partitions are unavoidable in distributed systems, the real choice is **CP vs AP**: CP systems (HBase, Zookeeper) return an error or wait during a partition to preserve consistency; AP systems (Cassandra, DynamoDB) keep responding but may return stale data. Most real systems choose a tradeoff along the spectrum rather than a hard CP or AP position. |
| What is eventual consistency and when is it acceptable? | Eventual consistency guarantees that if no new writes occur, all replicas will converge to the same value — but reads may return stale data in the interim. It enables high availability and low latency by allowing replicas to accept writes without coordinating with each other. Acceptable for: social media likes/views, product recommendations, DNS propagation, shopping cart totals. Not acceptable for: bank balances, inventory counts where overselling must be prevented, or any operation requiring a "read your own writes" guarantee without session affinity. |
| How does consistent hashing work and why is it used? | In consistent hashing, both data keys and server nodes are mapped onto a virtual ring (0–2³² positions) using a hash function. A key is assigned to the first node clockwise from its hash position. When a node is added or removed, only the keys between the new/removed node and its predecessor need to be moved — typically K/N keys (K=total keys, N=nodes). This is far less disruption than modular hashing (`key % N`), which remaps almost all keys. Virtual nodes (each physical server gets multiple ring positions) improve load distribution and allow weighted assignment. Used in Cassandra, DynamoDB, and CDNs. |
| What is the Saga pattern and how does it handle distributed transactions? | A Saga is a sequence of local transactions, each publishing an event or sending a command to trigger the next step. If a step fails, compensating transactions are executed in reverse order to undo the completed steps. Two coordination styles: **Choreography** — each service listens to events and decides its own next action (decentralized, but harder to track flow); **Orchestration** — a central coordinator (orchestrator) explicitly commands each service (clearer flow, single point of failure risk). Sagas trade ACID atomicity for eventual consistency across services, which is the standard approach in microservices. |
| How do you handle hot spots (write hot keys) in a database? | A hot spot occurs when many writes hit the same key or partition — common with sequential IDs or time-based partitions. Solutions: (1) **Key suffix randomization** — append a random suffix (1–N) to spread writes across N partitions, then aggregate on read; (2) **Time-based bucketing with a multiplier** — partition by `hour + random_shard`; (3) **Caching** — absorb reads in a cache layer so the hot DB row is hit less; (4) **Counter aggregation** — batch increments and write totals periodically instead of per-event; (5) use a purpose-built system like Redis for high-frequency counters. |
| What is the difference between L4 and L7 load balancing? | **L4 (transport-layer)** load balancing operates on TCP/UDP — it routes packets based on IP address and port without inspecting the content. It is very fast and protocol-agnostic but cannot make routing decisions based on content. **L7 (application-layer)** load balancing reads the HTTP request — URL path, headers, cookies, and body — enabling: URL-based routing (`/api` → service A, `/static` → CDN), SSL termination, sticky sessions via cookie, and A/B testing. L7 is slower due to parsing overhead but far more flexible. AWS ALB is L7; NLB is L4. |
| What are read replicas and what are their limitations? | Read replicas are copies of the primary database that serve read queries. The primary streams its WAL (write-ahead log) to replicas, which apply it asynchronously. Benefits: scale read throughput horizontally, reduce primary load, provide geographic proximity for read-heavy workloads. Limitations: **replication lag** — replicas may be seconds behind, causing stale reads; **not suitable for reads that must follow a write** (e.g., "read your own writes" after a POST); no write scaling — all writes still go to the primary. For write scaling you need sharding or a distributed database. |
| What architectural properties are required to achieve 99.99% availability? | 99.99% allows only ~52 minutes of downtime per year. Required properties: (1) **No single point of failure** — redundant instances for every component (app, cache, DB, load balancer); (2) **Multi-AZ or multi-region** deployment with automatic failover; (3) **Health checks + auto-scaling** to replace unhealthy instances; (4) **Graceful degradation** — circuit breakers, fallbacks, feature flags; (5) **Zero-downtime deployments** (blue/green or rolling); (6) **Chaos engineering** to proactively discover failure modes before production does. Each additional 9 roughly requires 10× more operational investment. |
| What is the difference between database sharding and replication? | **Replication** copies the same data to multiple nodes. All nodes have the same rows. It improves **read throughput** and provides **redundancy/failover**. **Sharding** horizontally partitions data — each shard holds a disjoint subset of rows (e.g., users A–M on shard 1, N–Z on shard 2). It scales **write throughput** and **storage** since each shard handles a fraction of the total data. Sharding adds complexity: cross-shard queries require scatter-gather, transactions spanning shards require distributed coordination. Most production systems use both: each shard is replicated for redundancy. |

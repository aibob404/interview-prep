# Spring Data & JPA

## 1. JPA vs Hibernate

| | JPA | Hibernate |
|---|---|---|
| Type | Specification (JSR 338) | Implementation (most popular) |
| Package | `javax.persistence` / `jakarta.persistence` | `org.hibernate` |
| Portability | High (swap implementations) | Lower (Hibernate-specific features) |

Use JPA annotations for portability; use Hibernate-specific features (e.g., `@BatchSize`, `@NaturalId`) only when needed.

---

## 2. Entity Mapping

```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_customer", columnList = "customer_id"),
    @Index(name = "idx_orders_status_created", columnList = "status, created_at")
})
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE,
                    generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "order_seq",
                       allocationSize = 50) // fetch 50 IDs at a time — reduces DB calls
    private Long id;

    @Column(nullable = false, length = 50)
    @Enumerated(EnumType.STRING) // never use ORDINAL — breaks on enum reordering
    private OrderStatus status;

    @Column(name = "created_at", nullable = false, updatable = false)
    private Instant createdAt;

    @Version // optimistic locking
    private Long version;

    @ManyToOne(fetch = FetchType.LAZY) // always lazy for @ManyToOne
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL,
               orphanRemoval = true, fetch = FetchType.LAZY)
    private List<OrderItem> items = new ArrayList<>();
}
```

### ID Generation Strategies

| Strategy | Behavior | Use case |
|----------|---------|---------|
| `AUTO` | JPA decides | Default, not recommended |
| `IDENTITY` | DB auto-increment | MySQL, PostgreSQL serial |
| `SEQUENCE` | DB sequence | PostgreSQL — preferred (batching possible) |
| `TABLE` | JPA-managed counter table | Avoid — performance |
| `UUID` | UUID-based | Distributed systems, no DB round-trip |

### `@GeneratedValue` + `SEQUENCE` vs `IDENTITY`
- `SEQUENCE` with `allocationSize > 1`: Hibernate pre-fetches IDs in batches — fewer DB round-trips, supports JDBC batch inserts
- `IDENTITY`: Hibernate must flush after each insert to get the generated ID → **disables JDBC batching**

---

## 3. Associations

### @OneToMany / @ManyToOne
```java
// OWNER SIDE: always @ManyToOne (the FK column lives in this table)
@Entity
public class OrderItem {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order; // owner
}

// INVERSE SIDE: @OneToMany with mappedBy
@Entity
public class Order {
    @OneToMany(mappedBy = "order", // matches field name in OrderItem
               cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    // Bidirectional helpers — always maintain both sides!
    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }
    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}
```

### @ManyToMany
```java
@Entity
public class Student {
    @ManyToMany
    @JoinTable(name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>(); // Use Set, not List (avoids duplicates)
}

// For extra columns on join table — use explicit join entity
@Entity
public class Enrollment {
    @ManyToOne @JoinColumn(name = "student_id") private Student student;
    @ManyToOne @JoinColumn(name = "course_id") private Course course;
    private Instant enrolledAt;
    private String grade;
}
```

### Fetch Types
```java
// @OneToMany / @ManyToMany: LAZY by default (correct)
// @ManyToOne / @OneToOne: EAGER by default (CHANGE TO LAZY!)

@ManyToOne(fetch = FetchType.LAZY) // always override
private User user;
```

**Why EAGER is dangerous:**
- Loads data you may not need
- Causes N+1 queries in lists
- Can trigger cartesian product joins with multiple EAGER collections

---

## 4. The N+1 Problem

```java
// Query fetches N orders
List<Order> orders = orderRepo.findAll(); // 1 query

for (Order o : orders) {
    System.out.println(o.getCustomer().getName()); // N queries — one per order!
}
// Total: N+1 queries
```

### Solutions

#### JOIN FETCH in JPQL
```java
@Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.status = :status")
List<Order> findWithCustomer(@Param("status") OrderStatus status);

// Multiple collections — Hibernate will throw MultipleBagFetchException for bags (List)
// Use Set for multiple fetches, or fetch one collection at a time
```

#### @EntityGraph
```java
@EntityGraph(attributePaths = {"customer", "items"})
List<Order> findByStatus(OrderStatus status);

// Or define named entity graph
@NamedEntityGraph(name = "Order.full",
    attributeNodes = {
        @NamedAttributeNode("customer"),
        @NamedAttributeNode(value = "items", subgraph = "items.product")
    },
    subgraphs = @NamedSubgraph(name = "items.product",
        attributeNodes = @NamedAttributeNode("product"))
)
@Entity
public class Order { ... }

@EntityGraph("Order.full")
Optional<Order> findById(Long id);
```

#### @BatchSize (Hibernate)
```java
// Loads items in batches instead of one-by-one
@OneToMany(mappedBy = "order")
@BatchSize(size = 25) // load items for 25 orders in one IN query
private List<OrderItem> items;
```

#### Hibernate `hibernate.default_batch_fetch_size`
```properties
spring.jpa.properties.hibernate.default_batch_fetch_size=25
```
Applies batch loading globally — good default for most apps.

---

## 5. Transactions

### @Transactional Internals
`@Transactional` works via **AOP proxy** — when you call a transactional method, Spring intercepts the call, opens a transaction, calls the real method, then commits or rolls back.

```java
@Service
@Transactional // class-level: applies to all public methods
public class OrderService {

    @Transactional(readOnly = true) // hint to DB: no flush, optimized
    public Order findById(Long id) {
        return repo.findById(id).orElseThrow();
    }

    @Transactional(
        propagation = Propagation.REQUIRED,     // default: join existing or create new
        isolation = Isolation.READ_COMMITTED,   // default in most DBs
        rollbackFor = PaymentException.class,   // add checked exceptions
        timeout = 10                            // seconds
    )
    public void placeOrder(OrderRequest req) {
        // ...
    }
}
```

### Propagation Levels

| Propagation | Behavior |
|-------------|---------|
| `REQUIRED` (default) | Join existing TX; create new if none |
| `REQUIRES_NEW` | Always create new TX; suspend existing |
| `SUPPORTS` | Join if exists; run non-transactional if none |
| `NOT_SUPPORTED` | Always run non-transactional; suspend existing |
| `MANDATORY` | Must join existing TX; exception if none |
| `NEVER` | Must run non-transactional; exception if TX exists |
| `NESTED` | Nested TX with savepoint; partial rollback possible |

### Isolation Levels

| Isolation | Dirty Read | Non-repeatable Read | Phantom Read |
|-----------|-----------|--------------------|----|
| `READ_UNCOMMITTED` | ✓ | ✓ | ✓ |
| `READ_COMMITTED` | ✗ | ✓ | ✓ |
| `REPEATABLE_READ` | ✗ | ✗ | ✓ |
| `SERIALIZABLE` | ✗ | ✗ | ✗ |

Most DBs default to `READ_COMMITTED`. MySQL InnoDB defaults to `REPEATABLE_READ`.

### Rollback Behavior
```java
// Default: rolls back on RuntimeException and Error
// Checked exceptions do NOT trigger rollback by default!

@Transactional(rollbackFor = IOException.class) // add checked exception
@Transactional(noRollbackFor = OptimisticLockException.class) // exclude

// Common mistake: catching exception inside @Transactional hides it
@Transactional
public void process() {
    try {
        repo.save(entity);
    } catch (DataAccessException e) {
        log.error("error", e); // TX is NOT marked for rollback!
        // must rethrow to trigger rollback
    }
}
```

### @Transactional Pitfalls

```java
// PITFALL 1: Self-invocation bypasses proxy
@Service
public class Service {
    @Transactional
    public void outer() {
        inner(); // NOT transactional — direct call, not through proxy!
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void inner() { ... }
}

// PITFALL 2: @Transactional on private method — ignored
@Transactional
private void doWork() { } // Spring AOP can't intercept private methods

// PITFALL 3: @Transactional on non-Spring bean — ignored
// PITFALL 4: Transaction-bound operations after transaction commits
```

---

## 6. Optimistic vs Pessimistic Locking

### Optimistic Locking
```java
@Entity
public class Account {
    @Version
    private Long version; // Hibernate increments on update
}

// On save: UPDATE account SET balance=?, version=? WHERE id=? AND version=?
// If version doesn't match → OptimisticLockException → retry or show error
```
Best for: **low contention**, read-heavy, user-facing operations.

### Pessimistic Locking
```java
// SELECT ... FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Account> findById(Long id);

// With timeout
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "javax.persistence.lock.timeout", value = "3000"))
Optional<Account> findById(Long id);
```
Best for: **high contention**, critical sections, financial operations.

---

## 7. Second-Level Cache

```java
// Entity cache
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product { ... }

// Collection cache
@OneToMany
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
private List<Tag> tags;

// Query cache
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
List<Product> findByCategory(String category);
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
```

**Cache strategies:**
- `READ_ONLY`: immutable data, highest performance
- `NONSTRICT_READ_WRITE`: rare updates, eventual consistency acceptable
- `READ_WRITE`: transactional updates, locks during write
- `TRANSACTIONAL`: full transactional cache (JTA required)

---

## 8. Spring Data Repositories

```java
// Basic
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Derived queries — parsed from method name
    List<Order> findByStatusAndCustomerId(OrderStatus status, Long customerId);
    Optional<Order> findFirstByCustomerIdOrderByCreatedAtDesc(Long customerId);
    long countByStatus(OrderStatus status);
    void deleteByStatusAndCreatedAtBefore(OrderStatus status, Instant cutoff);

    // JPQL
    @Query("SELECT o FROM Order o JOIN FETCH o.customer WHERE o.id = :id")
    Optional<Order> findWithCustomer(@Param("id") Long id);

    // Native query
    @Query(value = "SELECT * FROM orders WHERE total > :min", nativeQuery = true)
    List<Order> findHighValueOrders(@Param("min") BigDecimal min);

    // Projections — only fetch needed columns
    @Query("SELECT o.id as id, o.status as status FROM Order o")
    List<OrderSummary> findAllSummaries();

    // Streaming large result sets
    @Query("SELECT o FROM Order o")
    @QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "1000"))
    Stream<Order> streamAll();

    // Pagination
    Page<Order> findByStatus(OrderStatus status, Pageable pageable);
    Slice<Order> findByCustomerId(Long customerId, Pageable pageable); // no count query
}

// Projection interface
interface OrderSummary {
    Long getId();
    OrderStatus getStatus();
}
```

---

## 9. Common Interview Questions

| Question | Answer |
|----------|--------|
| What is the N+1 problem and how do you fix it? | The N+1 problem occurs when loading a list of N entities causes N additional queries to fetch their lazy-loaded associations. Example: loading 100 `Order` entities, then accessing `order.getItems()` for each — Hibernate fires 1 query for orders + 100 queries for items = 101 total. Fixes: (1) **JOIN FETCH** in JPQL: `SELECT o FROM Order o JOIN FETCH o.items` — one query with a join; (2) **`@EntityGraph`** on the repository method; (3) **`@BatchSize(size=50)`** on the collection — Hibernate loads 50 associations per query instead of 1; (4) projection DTOs that fetch only needed columns. |
| What does `@Transactional` do mechanically? | Spring generates an AOP proxy around the bean. When the annotated method is called through the proxy, the proxy opens a transaction (or joins an existing one, depending on propagation), invokes the real method, and commits on success. On an uncaught exception that triggers rollback, the proxy rolls back instead. The proxy approach means `@Transactional` has no effect on self-invocations (calling the method within the same class bypasses the proxy). |
| What is the default rollback behavior of `@Transactional`? | By default, Spring rolls back on **unchecked exceptions** (`RuntimeException` and its subclasses) and on `Error`. It does **not** roll back on checked exceptions. This is a Spring convention (not a JPA requirement). You can override it with `rollbackFor = Exception.class` to roll back on any exception, or `noRollbackFor` to suppress rollback for specific cases. Always verify this when your service methods throw checked exceptions. |
| What is the difference between optimistic and pessimistic locking in JPA? | **Optimistic locking** (via `@Version`) assumes conflicts are rare. Hibernate includes a `version` column in every `UPDATE` — if the version has changed since the entity was read, the update fails with `OptimisticLockException`. No DB lock is held during the transaction. **Pessimistic locking** (`PESSIMISTIC_WRITE`) issues a `SELECT ... FOR UPDATE`, immediately acquiring a row-level DB lock that blocks other writers until the transaction commits. Use optimistic for read-heavy, low-contention scenarios; use pessimistic when you must prevent any concurrent modification during a business operation. |
| What is the difference between `REQUIRED` and `REQUIRES_NEW` propagation? | `REQUIRED` (default): if a transaction already exists, the method joins it. If not, a new one is started. All work is committed or rolled back together. `REQUIRES_NEW`: always suspends any existing transaction and starts a fresh, independent one. The inner transaction commits or rolls back independently of the outer one. Use `REQUIRES_NEW` when you need an operation (e.g., logging, audit) to persist even if the outer transaction rolls back. Be aware that it holds two database connections simultaneously. |
| Why is `FetchType.EAGER` dangerous and when does it cause problems? | EAGER loading fetches the association unconditionally every time the parent entity is loaded — even when the association is not needed. Two problems: (1) **performance** — unnecessary data is always fetched; (2) **N+1 with multiple eager collections** — if an entity has two `@OneToMany(fetch=EAGER)` collections, Hibernate cannot fetch both in a single JOIN (cartesian product issue) and may resort to multiple queries. Always use `LAZY` as the default and fetch eagerly only when needed via `JOIN FETCH` or `@EntityGraph`. |
| What is `orphanRemoval` in JPA and how does it differ from `CascadeType.REMOVE`? | `orphanRemoval = true` on a `@OneToMany` relationship automatically deletes a child entity from the database when it is removed from the parent's collection: `order.getItems().remove(item)` → Hibernate deletes the `item` row. `CascadeType.REMOVE` deletes all children when the parent is deleted. `orphanRemoval` handles the case of removing a child from the collection while keeping the parent alive — a common pattern in aggregate root management. |
| Why is `SEQUENCE` ID generation preferred over `IDENTITY` for batch inserts? | With `IDENTITY` (database auto-increment), the database generates the ID only **after** the insert. Hibernate must flush each insert immediately to get the generated ID back — this disables JDBC batch inserts. With `SEQUENCE`, Hibernate pre-fetches a range of IDs from the sequence before inserting, can batch multiple inserts in a single JDBC statement, and significantly improves throughput for bulk operations. Always use `SEQUENCE` with an appropriate `allocationSize` (e.g., 50) for high-volume entities. |
| What is the Hibernate second-level cache and when should you enable it? | The first-level cache (Session cache) lives per transaction and is enabled by default. The **second-level cache** is a `SessionFactory`-level (application-wide) cache shared across transactions. When an entity is loaded, it is stored in the L2 cache. Subsequent reads in different sessions avoid the database entirely. Use it for **read-mostly, rarely-changing** reference data (e.g., country codes, product categories). Do not use it for frequently updated data — stale reads and cache invalidation complexity outweigh the benefits. Configure with EhCache or Caffeine via `@Cache` on the entity. |
| What does `@Version` do and how does it prevent lost updates? | `@Version` marks a numeric or timestamp field that Hibernate automatically includes in every `UPDATE`: `UPDATE order SET status=?, version=version+1 WHERE id=? AND version=?`. If another transaction updated the row and incremented the version between your read and write, the `WHERE version=?` clause matches zero rows, and Hibernate throws `OptimisticLockException`. The caller must catch it and retry or inform the user of a conflict. This prevents the **lost update** anomaly without holding any database locks. |

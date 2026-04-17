# Spring Core

## 1. IoC Container

**Inversion of Control:** the framework controls object creation and lifecycle — your code declares dependencies, Spring resolves them. This inverts the traditional "code creates its own dependencies" model.

### BeanFactory vs ApplicationContext

| | BeanFactory | ApplicationContext |
|---|---|---|
| Lazy initialization | Yes (default) | No (eager by default) |
| Bean post-processors | Manual | Auto-detected |
| Event publishing | No | Yes |
| i18n support | No | Yes |
| AOP integration | No | Yes |

> Always use `ApplicationContext`. `BeanFactory` is a low-level SPI for framework authors.

---

## 2. Dependency Injection Types

### Constructor Injection (Preferred)
```java
@Service
public class OrderService {
    private final OrderRepository repo;
    private final PaymentGateway gateway;

    // No @Autowired needed for single constructor (Spring 4.3+)
    public OrderService(OrderRepository repo, PaymentGateway gateway) {
        this.repo = Objects.requireNonNull(repo);
        this.gateway = Objects.requireNonNull(gateway);
    }
}
```
**Why preferred:**
- Dependencies are `final` — immutability guaranteed
- Null-safe at construction time
- Unit testing without Spring context (`new OrderService(mockRepo, mockGateway)`)
- Makes circular dependencies visible at startup (not silently proxied)

### Setter Injection (Optional Dependencies)
```java
@Service
public class NotificationService {
    private EmailSender emailSender;

    @Autowired(required = false) // optional dependency
    public void setEmailSender(EmailSender emailSender) {
        this.emailSender = emailSender;
    }
}
```

### Field Injection (Avoid)
```java
@Service
public class BadService {
    @Autowired private UserRepository repo; // hidden dependency, untestable
}
```
**Problems:** can't make field `final`, requires reflection for testing, hides dependencies.

---

## 3. Bean Lifecycle

```
Instantiation
    → Dependency Injection
    → BeanNameAware.setBeanName()
    → BeanFactoryAware.setBeanFactory()
    → ApplicationContextAware.setApplicationContext()
    → @PostConstruct / InitializingBean.afterPropertiesSet() / init-method
    → Bean Ready for Use
    → @PreDestroy / DisposableBean.destroy() / destroy-method
```

```java
@Component
public class ConnectionPool implements InitializingBean, DisposableBean {

    @PostConstruct
    public void init() {
        // Runs after all dependencies are injected
        pool.warmUp();
    }

    @Override
    public void afterPropertiesSet() {
        // Same as @PostConstruct but couples to Spring API
    }

    @PreDestroy
    public void shutdown() {
        pool.close();
    }
}
```

---

## 4. Bean Scopes

| Scope | Instance per | Notes |
|-------|-------------|-------|
| `singleton` | ApplicationContext | Default. Shared state — must be thread-safe |
| `prototype` | Each injection point | New instance every time; Spring doesn't manage lifecycle after creation |
| `request` | HTTP request | Web-aware contexts only |
| `session` | HTTP session | Web-aware contexts only |
| `application` | ServletContext | Web-aware, one per app |
| `websocket` | WebSocket session | |

### Scoped Bean in Singleton — The Proxy Problem
```java
// PROBLEM: prototype bean injected into singleton → same instance every time!
@Service
public class SingletonService {
    @Autowired
    private PrototypeBean bean; // Only injected ONCE at startup
}

// SOLUTION 1: Inject ApplicationContext and getBean manually
@Autowired
private ApplicationContext context;
public void doWork() {
    PrototypeBean bean = context.getBean(PrototypeBean.class); // new each time
}

// SOLUTION 2: Scoped proxy
@Component
@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class PrototypeBean { ... }
// Now singleton holds a proxy that creates a new bean on each method call
```

---

## 5. Configuration Approaches

### Java-based Configuration
```java
@Configuration
public class AppConfig {

    @Bean
    @Primary // used when multiple beans of same type exist
    public DataSource primaryDataSource(
            @Value("${db.url}") String url,
            @Value("${db.username}") String username) {
        return DataSourceBuilder.create()
            .url(url)
            .username(username)
            .build();
    }

    @Bean("secondaryDs")
    @DependsOn("primaryDataSource") // explicit ordering
    public DataSource secondaryDataSource() { ... }
}

// Injection by qualifier
@Service
public class ReportService {
    @Autowired
    @Qualifier("secondaryDs")
    private DataSource reportingDataSource;
}
```

### @ComponentScan
```java
@Configuration
@ComponentScan(
    basePackages = "com.example",
    excludeFilters = @ComponentScan.Filter(
        type = FilterType.ANNOTATION,
        classes = Repository.class
    )
)
public class AppConfig { }
```

### Condition-Based Beans
```java
@Bean
@ConditionalOnProperty(name = "feature.email", havingValue = "true")
public EmailService emailService() { ... }

@Bean
@ConditionalOnMissingBean(CacheManager.class)
public CacheManager defaultCacheManager() { ... }

@Bean
@ConditionalOnClass(name = "com.redis.clients.jedis.Jedis")
public RedisCache redisCache() { ... }

// Custom condition
@Bean
@Conditional(ProductionCondition.class)
public SecurityFilter strictSecurityFilter() { ... }
```

---

## 6. AOP — Aspect-Oriented Programming

### Core Concepts
- **Aspect:** module encapsulating cross-cutting concern
- **Pointcut:** expression defining which methods to intercept
- **Advice:** code to run (when and what)
- **Join point:** moment during execution (method call, field access)
- **Weaving:** applying aspects to target objects

### Advice Types
```java
@Aspect
@Component
public class SecurityAspect {

    @Before("execution(* com.example.service.*.*(..))")
    public void checkAuth(JoinPoint jp) {
        // runs before method; cannot prevent execution (throw exception to block)
    }

    @After("execution(* com.example..*(..))")
    public void audit(JoinPoint jp) {
        // runs after method (regardless of outcome)
    }

    @AfterReturning(pointcut = "execution(* com.example..*(..))", returning = "result")
    public void logResult(Object result) {
        // runs after successful return
    }

    @AfterThrowing(pointcut = "execution(* com.example..*(..))", throwing = "ex")
    public void logError(Exception ex) {
        // runs after exception thrown
    }

    @Around("@annotation(com.example.Timed)")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed(); // actually invoke the method
        } finally {
            log.info("{} took {}ms", pjp.getSignature(),
                     (System.nanoTime() - start) / 1_000_000);
        }
    }
}
```

### Pointcut Expressions
```java
// Method execution in package
"execution(* com.example.service.*.*(..))"
// Any method in type
"within(com.example.service.*)"
// Methods with specific annotation
"@annotation(org.springframework.transaction.annotation.Transactional)"
// Methods with annotated parameter
"@args(com.example.Validated)"
// Bean name pattern
"bean(userService)"
// Combine with &&, ||, !
"execution(* com.example..*Service.*(..)) && !execution(* *.get*(..))"
```

### AOP Pitfalls
```java
// PITFALL 1: Self-invocation bypasses proxy!
@Service
public class OrderService {
    @Transactional
    public void placeOrder(Order order) {
        validateOrder(order);
        // ...
    }

    @Transactional(propagation = REQUIRES_NEW)
    public void validateOrder(Order order) {
        // Called directly — NOT through proxy → @Transactional ignored!
    }
}

// FIX: Inject self or use AspectJ weaving (not just Spring proxy)
@Autowired
private OrderService self;
self.validateOrder(order); // goes through proxy

// PITFALL 2: AOP only works on Spring-managed beans
// PITFALL 3: private methods can't be intercepted by Spring AOP (proxy-based)
```

---

## 7. Spring Events

```java
// Define event
public class OrderPlacedEvent extends ApplicationEvent {
    private final Order order;
    public OrderPlacedEvent(Object source, Order order) {
        super(source);
        this.order = order;
    }
}

// Publish
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void placeOrder(Order order) {
        // business logic
        publisher.publishEvent(new OrderPlacedEvent(this, order));
    }
}

// Listen
@Component
public class EmailNotificationListener {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.send(event.getOrder().getCustomerEmail(), "Order confirmed");
    }

    @EventListener
    @Async // handle in different thread
    public void updateInventory(OrderPlacedEvent event) {
        inventoryService.reserve(event.getOrder().getItems());
    }

    @TransactionalEventListener(phase = AFTER_COMMIT) // fire only after TX commits
    public void sendReceipt(OrderPlacedEvent event) { ... }
}
```

---

## 8. Profiles

```java
@Configuration
@Profile("production")
public class ProductionConfig {
    @Bean
    public DataSource dataSource() { /* real DB */ }
}

@Configuration
@Profile("!production") // everything except production
public class DevelopmentConfig {
    @Bean
    public DataSource dataSource() { /* H2 in-memory */ }
}
```

Activate: `spring.profiles.active=production` or `-Dspring.profiles.active=production`

---

## 9. Common Interview Questions

| Question | Answer |
|----------|--------|
| What is the difference between `BeanFactory` and `ApplicationContext`? | `BeanFactory` is the basic IoC container — it instantiates beans lazily (on first request) and provides only core DI. `ApplicationContext` extends it with: eager singleton initialization at startup, `ApplicationEvent` publishing, AOP support, i18n (`MessageSource`), `@PostConstruct`/`@PreDestroy` lifecycle, and `BeanPostProcessor` auto-detection. In practice, always use `ApplicationContext` (or let Spring Boot create it). `BeanFactory` is only relevant in resource-constrained environments. |
| Why is constructor injection preferred over field injection? | Constructor injection makes dependencies explicit, mandatory, and immutable (fields can be `final`). The class is fully initialized after construction — no partial state. It fails fast at startup if a dependency is missing. It is easily testable without Spring: just call `new MyService(dep)`. Field injection (`@Autowired` on a field) hides dependencies, allows partially constructed objects, requires reflection (or Spring) to set private fields, and makes the class harder to test in isolation. |
| What is a circular dependency in Spring and how do you resolve it? | A circular dependency occurs when bean A requires bean B in its constructor, and bean B requires bean A in its constructor — Spring cannot satisfy either. For constructor injection, Spring throws `BeanCurrentlyInCreationException` at startup. Solutions: (1) redesign — extract a third class that both depend on; (2) use `@Lazy` on one injection point so the dependency is resolved as a proxy on first use; (3) switch one injection to setter-based (Spring can inject after construction). Field injection hides the cycle and should not be the fix. |
| What is a Spring AOP proxy and how does it work? | Spring wraps a bean in a **proxy** object at startup. All external calls to the bean go through the proxy, which checks whether any advice (before, after, around) applies to the called method. Two proxy types: **JDK dynamic proxy** (used when the bean implements at least one interface — the proxy implements the same interfaces); **CGLIB proxy** (used when no interface — the proxy is a generated subclass). The real bean instance sits inside the proxy. |
| What is the AOP self-invocation problem? | When a bean calls its own method internally (e.g., `this.someMethod()`), the call bypasses the proxy entirely — it goes directly to the real object. Any advice on `someMethod()` (e.g., `@Transactional`, `@Cacheable`) is not applied. Common mistake: calling a `@Transactional` method from another method in the same class. Solutions: inject the bean into itself via `@Autowired` and call the dependency reference; or use `AopContext.currentProxy()` (requires `exposeProxy=true`); or refactor the logic into a separate bean. |
| What is the Spring bean lifecycle in order? | (1) Instantiate — constructor called; (2) Populate properties — `@Autowired` fields and setters injected; (3) `BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware` callbacks; (4) `BeanPostProcessor.postProcessBeforeInitialization()`; (5) `@PostConstruct` method; (6) `InitializingBean.afterPropertiesSet()`; (7) Bean is ready for use; (8) `@PreDestroy` on context close; (9) `DisposableBean.destroy()`. |
| What does `@Primary` do and when should you use it? | `@Primary` marks a bean as the default candidate when Spring finds multiple beans of the same type during autowiring. If a `@Service` depends on `PaymentGateway` and two beans implement it — one annotated `@Primary` — Spring injects the primary one without requiring a qualifier. Use it when one implementation is the natural default and others are special cases. If you always need to be explicit, use `@Qualifier` instead. |
| What is `@Conditional` and how does autoconfiguration use it? | `@Conditional` tells Spring to only register a bean if a given condition evaluates to true at context startup. Spring Boot builds its entire autoconfiguration mechanism on top of it: `@ConditionalOnClass` (registers only if a class is on the classpath), `@ConditionalOnMissingBean` (backs off if you already defined one), `@ConditionalOnProperty`, etc. This allows Spring Boot starters to configure sensible defaults that are automatically replaced by user configuration. |
| How do you correctly inject a prototype-scoped bean into a singleton-scoped bean? | The problem: a singleton is created once, so Spring injects the prototype once at startup — the singleton always uses the same instance, defeating the purpose of prototype scope. Solutions: (1) **Scoped proxy** — annotate the prototype with `@Scope(value="prototype", proxyMode=ScopedProxyMode.TARGET_CLASS)`; the proxy is injected into the singleton and creates a new instance on every method call; (2) **`ApplicationContext.getBean()`** — inject the context and call `getBean(MyPrototype.class)` every time you need a fresh instance; (3) **`ObjectFactory<T>` / `Provider<T>`** — inject a factory and call `get()` each time. |

# Spring Boot

## 1. Autoconfiguration

Spring Boot automatically configures your application based on the dependencies on the classpath.

### How It Works
```
@SpringBootApplication
    ↓ includes
@EnableAutoConfiguration
    ↓ triggers
AutoConfigurationImportSelector
    ↓ reads
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
(or spring.factories in older versions)
    ↓ loads
~150+ autoconfiguration classes, each annotated with conditions
```

### Example: DataSource Autoconfiguration
```java
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ ... })
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

Spring Boot backs off if you define your own `DataSource` bean — this is the core of "convention over configuration".

### Exclusion and Override
```java
// Exclude a specific autoconfiguration
@SpringBootApplication(exclude = SecurityAutoConfiguration.class)

// Override: just define your own bean — autoconfiguration backs off
@Bean
public DataSource myDataSource() { ... } // disables DataSourceAutoConfiguration's bean
```

---

## 2. @SpringBootApplication

```java
@SpringBootApplication // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication app = new SpringApplication(Application.class);
        app.setAdditionalProfiles("local");
        app.run(args);
    }
}
```

### SpringApplication Customization
```java
SpringApplication app = new SpringApplication(Application.class);
app.setBannerMode(Banner.Mode.OFF);
app.setDefaultProperties(Map.of("server.port", "9090"));
app.addListeners(new ApplicationPidFileWriter());
ConfigurableApplicationContext ctx = app.run(args);
```

---

## 3. Externalized Configuration

### Property Priority (highest to lowest)
1. Command line arguments `--server.port=8080`
2. `SPRING_APPLICATION_JSON` environment variable
3. Servlet/system environment variables
4. OS environment variables
5. `application-{profile}.properties` outside jar
6. `application.properties` outside jar
7. `application-{profile}.properties` inside jar
8. `application.properties` inside jar
9. `@PropertySource` annotations
10. Default properties

### `application.yml` Example
```yaml
server:
  port: 8080
  shutdown: graceful  # wait for active requests before shutdown

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}       # environment variable
    password: ${DB_PASS}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000

  jpa:
    open-in-view: false          # always disable — prevents lazy loading in view layer
    show-sql: false
    properties:
      hibernate:
        default_batch_fetch_size: 25
        format_sql: true

logging:
  level:
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
```

### @ConfigurationProperties (Preferred Over @Value)
```java
@ConfigurationProperties(prefix = "payment")
@Validated // triggers JSR-303 validation
public class PaymentProperties {

    @NotNull
    private String apiKey;

    @Min(1) @Max(60)
    private int timeoutSeconds = 30;

    @Valid
    private RetryConfig retry = new RetryConfig();

    @Data
    public static class RetryConfig {
        private int maxAttempts = 3;
        private Duration backoff = Duration.ofSeconds(1);
    }
}

// Register it
@Configuration
@EnableConfigurationProperties(PaymentProperties.class)
public class PaymentConfig { }
```

**@Value vs @ConfigurationProperties:**
- `@Value`: simple, per-field injection, expressions supported (`#{}` for SpEL, `${}` for properties)
- `@ConfigurationProperties`: type-safe binding, IDE support, validation, refactoring-safe

---

## 4. Starters

Starters are convenience dependency descriptors that pull in a coherent set of dependencies.

| Starter | Brings in |
|---------|-----------|
| `spring-boot-starter-web` | Spring MVC, Tomcat, Jackson |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA, JDBC |
| `spring-boot-starter-security` | Spring Security |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, Testcontainers |
| `spring-boot-starter-actuator` | Metrics, health, info endpoints |
| `spring-boot-starter-cache` | Spring Cache abstraction |
| `spring-boot-starter-validation` | Jakarta Bean Validation, Hibernate Validator |

---

## 5. Actuator

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,env,beans,loggers
  endpoint:
    health:
      show-details: when-authorized  # or 'always' for dev
    shutdown:
      enabled: true  # POST /actuator/shutdown

  # Prometheus scraping
  metrics:
    export:
      prometheus:
        enabled: true
```

### Key Endpoints
| Endpoint | Description |
|----------|-------------|
| `/actuator/health` | Application health status, DB, disk, custom checks |
| `/actuator/info` | App version, git info |
| `/actuator/metrics` | JVM, HTTP, custom metrics (Micrometer) |
| `/actuator/prometheus` | Metrics in Prometheus format |
| `/actuator/env` | All environment properties |
| `/actuator/beans` | All Spring beans |
| `/actuator/loggers` | View/change log levels at runtime |
| `/actuator/threaddump` | Thread dump |
| `/actuator/heapdump` | Heap dump |
| `/actuator/httptrace` | Recent HTTP exchanges |

### Custom Health Indicator
```java
@Component
public class PaymentGatewayHealth implements HealthIndicator {

    @Override
    public Health health() {
        try {
            gatewayClient.ping();
            return Health.up()
                .withDetail("latency", "12ms")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

### Custom Metrics
```java
@Service
public class OrderService {
    private final Counter orderCounter;
    private final Timer orderTimer;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.placed")
            .tag("region", "eu")
            .description("Total orders placed")
            .register(registry);

        this.orderTimer = Timer.builder("orders.processing.time")
            .description("Order processing duration")
            .register(registry);
    }

    public Order placeOrder(OrderRequest req) {
        return orderTimer.record(() -> {
            Order order = processOrder(req);
            orderCounter.increment();
            return order;
        });
    }
}
```

---

## 6. Testing

### @SpringBootTest
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class OrderIntegrationTest {

    @Autowired TestRestTemplate restTemplate;

    @LocalServerPort int port;

    @Test
    void shouldCreateOrder() {
        ResponseEntity<Order> response = restTemplate.postForEntity(
            "/api/orders", new CreateOrderRequest(...), Order.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    }
}
```

### Slice Tests (Faster)
```java
// Test only web layer (no service/repository)
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    @Autowired MockMvc mockMvc;
    @MockBean OrderService orderService; // mock service layer

    @Test
    void shouldReturnOrder() throws Exception {
        when(orderService.findById(1L)).thenReturn(mockOrder());
        mockMvc.perform(get("/api/orders/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.status").value("PENDING"));
    }
}

// Test only JPA layer (in-memory DB)
@DataJpaTest
class OrderRepositoryTest {
    @Autowired TestEntityManager em;
    @Autowired OrderRepository repo;
    // Tests run in transaction, rolled back after each test
}
```

### Testcontainers Integration
```java
@SpringBootTest
@Testcontainers
class OrderRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

---

## 7. Graceful Shutdown

```yaml
server.shutdown: graceful
spring.lifecycle.timeout-per-shutdown-phase: 30s
```

On `SIGTERM`:
1. Stop accepting new requests
2. Wait for active requests to complete (up to 30s)
3. Shutdown Spring context (beans destroyed, connections closed)
4. JVM exits

---

## 8. Common Interview Questions

| Question | Answer |
|----------|--------|
| How does Spring Boot autoconfiguration work? | Spring Boot scans `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 3) for a list of `@AutoConfiguration` classes. Each class is annotated with `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`, etc. At startup, Spring evaluates these conditions and registers only the beans whose conditions pass. If you define your own `DataSource` bean, `DataSourceAutoConfiguration` detects it via `@ConditionalOnMissingBean(DataSource.class)` and backs off. This is the "opinionated defaults, easily overridden" model. |
| What replaced `spring.factories` in Spring Boot 3? | In Boot 2, auto-configurations were registered in `META-INF/spring.factories` under the `EnableAutoConfiguration` key. Boot 3 replaced this with `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` — a dedicated file listing auto-configuration class names one per line. The old `spring.factories` mechanism still works for backward compatibility but is deprecated for autoconfiguration. The new file is more explicit and avoids the key–value parsing overhead. |
| What is `@ConfigurationProperties` and why prefer it over `@Value`? | `@ConfigurationProperties` binds a whole group of properties from `application.yml` or `application.properties` to a typed POJO. Benefits: IDE auto-completion and documentation, JSR-303 validation (`@Validated`), hierarchical binding of nested objects, and binding collections/maps. `@Value` injects a single property via SpEL and has no type safety or validation support. For any non-trivial configuration (more than one or two properties), `@ConfigurationProperties` is cleaner and more maintainable. |
| What is the Open Session In View anti-pattern, and why set `spring.jpa.open-in-view=false`? | OSIV keeps the Hibernate Session (database connection) open for the entire HTTP request lifecycle, including the view rendering layer. This allows lazy loading in templates or JSON serializers but is harmful in production: it holds a DB connection for the duration of every request, even during template rendering, drastically reducing connection pool capacity. Disabling it (`false`) forces you to load all needed data in the service layer before closing the transaction, resulting in explicit, predictable database access. |
| How do you create a custom Actuator health indicator? | Implement the `HealthIndicator` interface and register it as a Spring bean. Override `health()` to return `Health.up().withDetail("key", "value").build()` when the dependency is healthy, or `Health.down().withDetail("error", message).build()` when it is not. Spring Boot automatically discovers it and aggregates it into the `/actuator/health` response. For async checks, implement `ReactiveHealthIndicator` instead. |
| How do you expose custom application metrics with Micrometer? | Inject `MeterRegistry` into your component. Use `Counter.builder("name").register(registry).increment()` for counts, `Timer.builder("name").register(registry).record(...)` for durations, and `Gauge.builder("name", supplier, ...)` for current values. Tag your metrics with dimensions (e.g., `tag("endpoint", "/api/orders")`) for filtering in Prometheus/Grafana. Micrometer is vendor-neutral — the same code exports to Prometheus, Datadog, CloudWatch, etc., by changing only the dependency. |
| What is the difference between `@SpringBootTest` and `@WebMvcTest`? | `@SpringBootTest` loads the **full application context** — all beans, datasources, security, and configurations. It is used for end-to-end integration tests but is slow because it starts the entire application. `@WebMvcTest` loads only the **web layer** — controllers, filters, `MockMvc`, and related beans — mocking everything else with `@MockBean`. It is much faster and focused. Use `@WebMvcTest` for testing controller logic, request mapping, and HTTP serialization; use `@SpringBootTest` for testing cross-layer interactions. |
| How does Spring Boot graceful shutdown work? | When a SIGTERM is received (e.g., from Kubernetes during pod termination), Spring Boot (2.3+) stops accepting new requests, waits for in-flight requests to complete (up to a configurable timeout: `server.shutdown.timeout`), and then proceeds to destroy beans and close the application context. Enable it with `server.shutdown=graceful`. Without graceful shutdown, in-flight requests get cut off immediately, causing errors for clients and potentially corrupting ongoing database transactions. |
| What is Testcontainers and what problem does it solve? | Testcontainers starts real **Docker containers** (PostgreSQL, Redis, Kafka, etc.) programmatically during tests and stops them after. It solves the "it works on my machine" problem by eliminating environment-specific test databases or in-memory fakes that behave differently from production. Tests run against the real database engine, catching SQL dialect issues, migration problems, and driver-specific behavior that H2 or mocks would hide. Spring Boot 3.1+ has built-in `@ServiceConnection` support for zero-config Testcontainers setup. |
| What is `@DynamicPropertySource` and when do you use it? | `@DynamicPropertySource` registers Spring `Environment` properties at test startup time, after the test infrastructure (like Testcontainers) has started. Use it to inject dynamic values — like the JDBC URL of a freshly started container — into the application context before any beans are created: `registry.add("spring.datasource.url", container::getJdbcUrl)`. Without it, you would have to use a fixed port or a custom `ApplicationContextInitializer`. |

# Java Standards

Java conventions for Entur applications. Read [CONVENTIONS.md](../CONVENTIONS.md) first for cross-language standards.

## Runtime and Build

- **Java version**: 21 or newer (LTS releases, or latest stable like 25)
- **Build tool**: Gradle with Kotlin DSL (`build.gradle.kts`)
- **Framework**: Spring Boot 3.x
- **Dependency management**: Gradle version catalogs (`gradle/libs.versions.toml`)
- **JDK distribution**: Liberica JDK (preferred) or Eclipse Temurin

## Project Setup

### build.gradle.kts

```kotlin
plugins {
    java
    id("org.springframework.boot") version libs.versions.springBoot
    id("io.spring.dependency-management") version libs.versions.springDependencyManagement
}

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY build/libs/*.jar app.jar

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## Logging

Use [entur/cloud-logging](https://github.com/entur/cloud-logging) -- Entur's standard logging library for JVM applications on GCP. It provides plug-and-play structured JSON logging, on-demand logging for cost reduction, request-response logging, and human-readable output during local development.

### Setup

Import the BOM and the GCP web starter:

```kotlin
// build.gradle.kts
val cloudLoggingVersion = "x.y.z"  // check Maven Central for latest

dependencies {
    implementation(platform("no.entur.logging.cloud:bom:$cloudLoggingVersion"))
    testImplementation(platform("no.entur.logging.cloud:bom:$cloudLoggingVersion"))

    // Base logging (required)
    implementation("no.entur.logging.cloud:spring-boot-starter-gcp-web")
    testImplementation("no.entur.logging.cloud:spring-boot-starter-gcp-web-test")
}
```

Remove any existing `logback.xml` or `logback-spring.xml` -- cloud-logging provides its own configuration automatically.

### Usage

Use standard SLF4J logging -- cloud-logging handles the JSON formatting, GCP severity mapping, and correlation-id propagation:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private static final Logger LOG = LoggerFactory.getLogger(RouteService.class);

LOG.info("Route found for id {}", routeId);
LOG.warn("Retry attempt {} for route {}", attempt, routeId);
LOG.error("Failed to fetch route {}", routeId, exception);
```

Configure log levels via Spring properties:

```yaml
# application.yml
logging:
  level:
    root: INFO
    no.entur.myapp: INFO
```

### Optional: Request-Response Logging

Log HTTP request and response bodies (Logbook-based):

```kotlin
// build.gradle.kts
dependencies {
    implementation("no.entur.logging.cloud:request-response-spring-boot-starter-gcp-web")
    testImplementation("no.entur.logging.cloud:request-response-spring-boot-starter-gcp-web-test")
}
```

Exclude noisy endpoints:

```yaml
logbook:
  exclude:
    - /actuator/**
    - /v3/api-docs/**
```

### Optional: On-Demand Logging

Reduce logging costs by only capturing full logs for problematic requests. When enabled, happy-case requests log at a higher threshold (e.g. WARN), but if an error occurs, all buffered log statements for that request (including INFO) are flushed:

```kotlin
// build.gradle.kts
dependencies {
    implementation("no.entur.logging.cloud:on-demand-spring-boot-starter-gcp-web")
}
```

```yaml
# application.yml
entur:
  logging:
    http:
      ondemand:
        enabled: true
        success:
          level: warn                            # happy case: only log WARN+
        failure:
          level: info                            # failure: flush all INFO+ logs
          http:
            status-code:
              equal-or-higher-than: 400          # trigger on 4xx/5xx
          logger:
            level: error                         # trigger on ERROR log statements
```

### Local Development

In test scope, cloud-logging automatically provides human-readable colored output. You can toggle the output format:

```yaml
entur:
  logging:
    style: humanReadablePlain    # humanReadablePlain | humanReadableJson | machineReadableJson
```

### DevOpsLogger (Additional Severity Levels)

For operational severity beyond standard SLF4J levels:

```java
import no.entur.logging.cloud.api.DevOpsLogger;
import no.entur.logging.cloud.api.DevOpsLoggerFactory;

private static final DevOpsLogger LOGGER = DevOpsLoggerFactory.getLogger(MyService.class);

LOGGER.errorTellMeTomorrow("Non-urgent error");        // ERROR level
LOGGER.errorInterruptMyDinner("Critical error");        // CRITICAL level
LOGGER.errorWakeMeUpRightNow("System down");            // ALERT level
```

## Application Configuration

### application.yml (defaults)

```yaml
server:
  port: 8080

spring:
  application:
    name: ${APPLICATION_NAME:my-application}

management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  endpoint:
    health:
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db
  metrics:
    tags:
      application: ${spring.application.name}
```

### Health Checks

Spring Boot Actuator provides the health endpoints expected by Kubernetes:

- Liveness: `/actuator/health/liveness`
- Readiness: `/actuator/health/readiness`

These are the defaults in the Entur common Helm chart. Do not change these paths unless you also update the Helm values.

## Coding Patterns

### REST Controllers

```java
@RestController
@RequestMapping("/api/v1/routes")
public class RouteController {

    private final RouteService routeService;

    public RouteController(RouteService routeService) {
        this.routeService = routeService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<RouteResponse> getRoute(@PathVariable String id) {
        return routeService.findById(id)
                .map(ResponseEntity::ok)
                .orElse(ResponseEntity.notFound().build());
    }

    @PostMapping
    public ResponseEntity<RouteResponse> createRoute(
            @Valid @RequestBody CreateRouteRequest request) {
        RouteResponse created = routeService.create(request);
        URI location = URI.create("/api/v1/routes/" + created.id());
        return ResponseEntity.created(location).body(created);
    }
}
```

### Service Layer

```java
@Service
public class RouteService {

    private final RouteRepository routeRepository;

    public RouteService(RouteRepository routeRepository) {
        this.routeRepository = routeRepository;
    }

    @Transactional(readOnly = true)
    public Optional<RouteResponse> findById(String id) {
        return routeRepository.findById(id)
                .map(RouteMapper::toResponse);
    }

    @Transactional
    public RouteResponse create(CreateRouteRequest request) {
        Route route = RouteMapper.toEntity(request);
        Route saved = routeRepository.save(route);
        return RouteMapper.toResponse(saved);
    }
}
```

### Key Principles

- Use constructor injection (not field injection with `@Autowired`)
- Use Java records for DTOs and value objects
- Use `Optional` for return types that may be absent -- never return null
- Use `@Transactional(readOnly = true)` for read operations
- Validate inputs at the controller boundary with `@Valid`
- Use a mapper layer to convert between entities and DTOs

### Exception Handling

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger LOG = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        LOG.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
                .body(new ErrorResponse("VALIDATION_ERROR", message));
    }
}

public record ErrorResponse(String code, String message) {}
```

## Testing

### Unit Tests

```java
@ExtendWith(MockitoExtension.class)
class RouteServiceTest {

    @Mock
    private RouteRepository routeRepository;

    @InjectMocks
    private RouteService routeService;

    @Test
    @DisplayName("findById returns route when it exists")
    void findById_existingRoute_returnsRoute() {
        // Arrange
        Route route = TestFixtures.aRoute().build();
        when(routeRepository.findById("route-1")).thenReturn(Optional.of(route));

        // Act
        Optional<RouteResponse> result = routeService.findById("route-1");

        // Assert
        assertThat(result).isPresent();
        assertThat(result.get().id()).isEqualTo("route-1");
    }
}
```

### Integration Tests

```java
@SpringBootTest
@Testcontainers
class RouteRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private RouteRepository routeRepository;

    @Test
    void savesAndRetrievesRoute() {
        Route route = new Route("route-1", "Oslo S", "Bergen");
        routeRepository.save(route);

        Optional<Route> found = routeRepository.findById("route-1");
        assertThat(found).isPresent();
    }
}
```

### Test Libraries

- JUnit 5 for test framework
- AssertJ for fluent assertions (preferred over Hamcrest)
- Mockito for mocking
- Testcontainers for integration tests with databases and message brokers
- Spring Boot Test for application context tests

## Redis (Memorystore)

Entur uses **Google Memorystore for Redis** as a managed key-value store. Infrastructure is provisioned via the `terraform-google-memorystore` Terraform module (see [terraform/modules.md](terraform/modules.md#memorystore-redis)). Connection credentials (`REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`) are injected via Kubernetes secrets.

### When to Use Redis

| Use Case | Redis? | Notes |
|----------|--------|-------|
| Caching (HTTP responses, DB queries) | Yes | Primary use case. Reduces load on PostgreSQL |
| Session storage | Yes | Shared sessions across pods |
| Rate limiting / counters | Yes | Atomic `INCR` with TTL |
| Distributed locks | Yes | Use Redisson or Spring Integration |
| Idempotency keys (Kafka dedup) | Yes | `SET key value NX EX ttl` pattern |
| Primary data store | **No** | Use PostgreSQL. Redis is not durable by default |
| Complex queries / joins | **No** | Use PostgreSQL |
| Large objects (> 1 MB) | **No** | Use Cloud Storage |

### Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
}
```

### Configuration

```yaml
# application.yml
spring:
  data:
    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD}
      timeout: 2000ms              # connection + command timeout
      connect-timeout: 1000ms      # TCP connection timeout
      lettuce:
        pool:
          max-active: 8            # max concurrent connections
          max-idle: 8
          min-idle: 2
          max-wait: 1000ms         # max wait for a connection from the pool
```

### Spring Cache Abstraction

The simplest approach -- use `@Cacheable` annotations with Redis as the backing store:

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheConfiguration cacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .disableCachingNullValues()
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(new GenericJackson2JsonRedisSerializer())
            );
    }

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory)
            .cacheDefaults(cacheConfiguration())
            .withCacheConfiguration("routes",
                cacheConfiguration().entryTtl(Duration.ofHours(1)))   // per-cache TTL
            .withCacheConfiguration("stops",
                cacheConfiguration().entryTtl(Duration.ofMinutes(30)))
            .build();
    }
}
```

```java
@Service
public class RouteService {

    @Cacheable(value = "routes", key = "#id")
    public Route findById(String id) {
        return routeRepository.findById(id).orElseThrow();
    }

    @CacheEvict(value = "routes", key = "#id")
    public void update(String id, UpdateRouteRequest request) {
        // Cache entry is evicted after update
    }

    @CacheEvict(value = "routes", allEntries = true)
    public void refreshAll() {
        // Evict all entries in the "routes" cache
    }
}
```

### Direct RedisTemplate Usage

For more control (counters, locks, sets, hashes):

```java
@Component
public class RateLimiter {

    private final StringRedisTemplate redis;

    public RateLimiter(StringRedisTemplate redis) {
        this.redis = redis;
    }

    public boolean isAllowed(String clientId, int maxRequests, Duration window) {
        String key = "rate:" + clientId;
        Long count = redis.opsForValue().increment(key);
        if (count == 1) {
            redis.expire(key, window);
        }
        return count <= maxRequests;
    }
}
```

### Key Naming Conventions

Use a consistent prefix scheme to avoid collisions and enable monitoring:

```text
{app}:{domain}:{id}           -- entity cache
{app}:rate:{clientId}          -- rate limiting
{app}:lock:{resource}          -- distributed locks
{app}:dedup:{messageId}        -- idempotency keys
```

Examples: `products-api:route:ENT:Route:123`, `products-api:rate:partner-xyz`

### Best Practices

- **Always set TTLs** -- never store keys without expiration. Unbounded growth will exhaust memory and cause eviction of other keys.
- **Use `allkeys-lfu`** eviction policy (configured in Terraform) -- least-frequently-used keys are evicted first when memory is full.
- **Keep values small** -- serialize to JSON, avoid storing entire entity graphs. Aim for < 100 KB per key.
- **Use `NX` (set-if-not-exists)** for distributed locks and idempotency: `SET key value NX EX 60`.
- **Handle failures gracefully** -- Redis is a cache, not a primary store. If Redis is unavailable, fall back to the database. Never let a Redis outage cause the application to fail.
- **Avoid `KEYS *`** in production -- it blocks the single-threaded Redis server. Use `SCAN` for iteration.
- **Use pipelining** for batch operations to reduce round trips.
- **Namespace keys** with the application name to avoid collisions when multiple apps share a Redis instance.
- **Monitor memory** -- set alerts on `used_memory` vs `maxmemory`. See [observability.md](observability.md) for metrics and alerting.
- **Do not use Redis as a message queue** -- use Kafka instead. Redis Pub/Sub has no persistence or delivery guarantees.

### Testing

Use Testcontainers for integration tests:

```java
@SpringBootTest
@Testcontainers
class RedisCacheIntegrationTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureRedis(DynamicPropertyRegistry registry) {
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", () -> redis.getMappedPort(6379));
        registry.add("spring.data.redis.password", () -> "");
    }
}
```

For unit tests, mock the cache or use `@MockBean` on `RedisTemplate`:

```java
@WebMvcTest(RouteController.class)
class RouteControllerTest {

    @MockBean
    private RouteService routeService;  // caching is transparent

    // Test controller behavior -- caching is an implementation detail
}
```

## Artifactory (JFrog)

Entur uses JFrog Artifactory as the artifact repository for internal packages. Configure Gradle to resolve from Artifactory:

```kotlin
// build.gradle.kts
repositories {
    val entur_artifactory_user: String? by project
    val entur_artifactory_password: String? by project

    maven {
        name = "Entur JFrog"
        url = URI("https://entur2.jfrog.io/entur2/entur-release-standard/")
        credentials {
            username = entur_artifactory_user ?: System.getenv("ARTIFACTORY_AUTH_USER")
            password = entur_artifactory_password ?: System.getenv("ARTIFACTORY_AUTH_TOKEN")
        }
    }
}
```

Set credentials locally in `$HOME/.gradle/gradle.properties` or as environment variables. In GitHub Actions, use the organization secrets `ARTIFACTORY_AUTH_USER` and `ARTIFACTORY_AUTH_TOKEN`.

## Spring MVC vs WebFlux

**Default to Spring MVC** unless you have a demonstrated need for high-concurrency I/O.

Choose **WebFlux** when the entire stack can be end-to-end non-blocking (R2DBC, reactive HTTP clients) and you need efficient handling of many concurrent I/O operations, streaming, SSE, or WebSockets.

Choose **MVC** when the stack is mostly blocking (JDBC/JPA, legacy SDKs), workloads are CPU-bound, or the team prefers imperative code. MVC has a lower learning curve, more intuitive debugging, and broader library compatibility.

Risks of WebFlux: steep learning curve (Mono/Flux, backpressure), harder debugging (reactive stack traces), testing complexity (StepVerifier), and mixing blocking code with reactive degrades performance.

## Connection Pool Sizing

When using HPA with Cloud SQL, total database connections = `number_of_pods * max_pool_size_per_pod`. HikariCP defaults to a pool size of 10. With 5 pods: `5 * 10 = 50` connections consumed.

Each connection consumes RAM on the database. Ensure the Cloud SQL instance tier's `max_connections` (minus 3 reserved for superuser) can handle the worst-case HPA pod count. See [Terraform modules](terraform/modules.md) for Cloud SQL instance sizing.

## Rate Limiting

Under heavy load, all threads can become busy waiting for database connections, causing HTTP 503 errors. Implement rate limiting to protect the service:

- **Spring approach**: Extend `OncePerRequestFilter` to create a QoS filter that returns HTTP 503 when requests exceed a per-second limit
- **Resilience4j approach**: Use `@RateLimiter` annotation from Resilience4j

The **client-side connection timeout must be shorter than the server-side timeout** to ensure clients are properly notified of errors.

## Dependencies

### Common Dependencies

| Dependency | Purpose |
|-----------|---------|
| `spring-boot-starter-web` | REST API |
| `spring-boot-starter-actuator` | Health checks, metrics |
| `spring-boot-starter-data-jpa` | Database access (JPA) |
| `spring-boot-starter-validation` | Input validation |
| `no.entur.logging.cloud:spring-boot-starter-gcp-web` | Structured logging (cloud-logging) |
| `micrometer-registry-prometheus` | Prometheus metrics |
| `spring-cloud-gcp-starter` | GCP integration |
| `spring-cloud-gcp-starter-secretmanager` | Secret Manager integration |
| `flyway-core` | Database migrations |
| `postgresql` | PostgreSQL driver |
| `org.entur.data:entur-kafka-spring-starter` | Kafka producer/consumer with Aiven defaults ([docs](kafka.md)) |
| `org.entur.openapi:entur-springdoc-starter` | Entur OpenAPI extensions for springdoc ([docs](api-design.md#entur-springdoc-starter)) |
| `org.entur.metrics:metrics-spring-boot-starter` | Prometheus metrics with Entur defaults ([docs](observability.md#entur-metrics-starter-spring-boot)) |

### Cloud SQL Connectivity

When using Cloud SQL, the application connects via the Cloud SQL proxy sidecar (configured in Helm). Configure the datasource to connect to `localhost`:

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/${DB_NAME}
    username: ${PG_USER}
    password: ${PG_PASSWORD}
```

The `PG_USER`, `PG_PASSWORD`, and `DB_NAME` values come from Kubernetes secrets created by the Terraform `terraform-google-sql-db` module.

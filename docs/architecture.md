# Architecture Standards

Guidelines for application architecture at Entur. All services run on Google Kubernetes Engine (GKE) in `europe-west1`.

## Design Methodology

Entur's official architectural methodology combines **Domain-Driven Design (DDD)** with **Hexagonal Architecture**:

- **DDD** establishes a common/ubiquitous language between technical and domain experts, produces testable code by design, and implies SOLID principles
- **Hexagonal Architecture** (ports and adapters) keeps the domain core independent of frameworks, databases, and external services -- outer layers depend on inner layers, never the reverse
- **Reactive Systems** principles (from the Reactive Manifesto) guide the operational design: responsive, elastic, resilient, and event-driven

## Service Architecture

### Microservice Principles

- Each service owns its data -- no shared databases between services
- Services communicate via well-defined APIs (REST or gRPC) or asynchronous messaging (Kafka)
- Services are independently deployable
- Each service has its own repository, CI/CD pipeline, and Kubernetes namespace
- Follow the golden path: repository name = application name = namespace = Helm release name
- **"You build it, you run it"** -- each team is responsible for operating its deployed services
- All versions must support rollback to the previous version
- Maintain backwards compatibility so consumers can update at their own pace
- Applications must start even when dependencies are unavailable -- never crash because a dependency is missing

### Layered Architecture (within a service)

```text
Controller                 # HTTP request handling, DTO transformation
       |
    Mapper                 # DTO <-> Domain transformation
       |
   Service                 # Business logic, validation, orchestration
       |
     DAO                   # Data access, SQL queries
       |
   Entity                  # Database table definition
       |
  Domain Model             # Domain data classes, value objects
```

- **Controllers** handle HTTP concerns only: receive DTOs, call mappers, return DTOs with status codes
- **Mappers** transform between generated DTOs and domain models at the API boundary
- **Services** contain business logic and orchestrate between DAOs. Define service interfaces with implementations
- **DAOs** handle data persistence (SQL queries via Exposed or Spring Data)
- **Entities** define database table structure (Exposed table objects or JPA entities)
- **Domain Models** are plain Kotlin data classes with no framework dependencies

### Key Principles

- Depend inward: outer layers depend on inner layers, never the reverse
- Use dependency injection for testability
- Keep controllers thin -- delegate to services
- Don't let database entities leak into API responses (use DTOs/mappers)
- Service layer works exclusively with domain models, never with DTOs
- Define service interfaces separately from implementations for testability

### Package-by-Feature

Organize code by domain feature, not by technical layer. Each feature package contains all its layers:

```text
org.entur.myapp/
  common/
    api/                   # Shared API utilities (ErrorResponse, GlobalExceptionHandler)
    base/                  # Base interfaces (BaseDAO, BaseService)
    exception/             # Typed exceptions
    utils/                 # Shared utilities, constants
  config/                  # Spring configuration classes
  version/                 # Feature package
    Version.kt             # Domain model (data class)
    VersionController.kt   # REST controller (implements generated interface)
    VersionDAO.kt           # Data access object
    VersionEntity.kt        # Database table definition (Exposed)
    VersionInputValidator.kt # Input validation rules
    VersionMapper.kt        # DTO <-> Domain mapper
    VersionService.kt       # Service interface
    VersionServiceImpl.kt   # Service implementation
  priceableobject/         # Another feature package
    PriceableObjectEntity.kt
    PriceableObjectDAO.kt
    fareprice/             # Sub-feature
      FarePriceEntity.kt
```

### Mapper Pattern

Use dedicated mapper classes to transform between generated DTOs and domain models. This is the preferred pattern when using contract-first OpenAPI development:

```text
Controller Layer       ← receives/returns DTOs
       ↓
   Mapper Layer        ← transforms DTO ↔ Domain
       ↓
  Service Layer        ← works ONLY with domain models
       ↓
    DAO Layer           ← transforms Domain ↔ Entity/ResultRow
```

Benefits:

- API contracts (DTOs) are separated from business logic (domain models)
- Changes to API don't ripple through the entire codebase
- Read-only fields (like `created`, `changed`) are correctly handled at the boundary
- Transformation logic is testable in isolation

### Composition Over Inheritance

Prefer composing functionality via constructor injection over inheritance hierarchies:

```kotlin
@Service
class VersionServiceImpl(private val versionDAO: VersionDAO) : VersionService {
    // Business logic delegates to DAO -- composition, not inheritance
}
```

Entities compose relationships using extension functions rather than inheritance:

```kotlin
object PriceableObjectEntity : LongIdTable("priceable_object") {
    fun Join.joinPriceableObjectWithChildren() =
        Join(this)
            .joinFarePriceToPriceableObject()
            .joinLimitingRuleToFarePrice()
            .withVersionsForPriceableObjects()
}
```

## Infrastructure Architecture

### GCP Project Structure

- Each application has a GCP project per environment (dev, tst, prd)
- The `terraform-google-init` module discovers project configuration
- Shared networking is managed centrally (VPC, subnets)

### Data Stores

| Need | Service | Provisioning |
|------|---------|-------------|
| Relational data | Cloud SQL (PostgreSQL) | `terraform-google-sql-db` module |
| Caching | Memorystore (Redis) | `terraform-google-memorystore` module |
| Object storage | Cloud Storage | [`terraform-google-cloud-storage`](https://github.com/entur/terraform-google-cloud-storage) module |
| Event streaming | Apache Kafka (Aiven-hosted) | Provisioned via Aiven; see [docs/kafka.md](kafka.md) |
| Analytics | BigQuery | Use `google_bigquery_dataset` / `table` resources |

For Cloud SQL and Memorystore, always use the Entur Terraform modules -- they handle naming, networking, secret creation, and Kubernetes integration automatically.

### Connectivity

- Applications connect to Cloud SQL via the **Cloud SQL proxy sidecar** (configured in Helm with `postgres.enabled: true`)
- Applications connect to Memorystore via the **private IP** (automatically configured by the Terraform module and exposed as `REDIS_HOST` environment variable)
- Inter-service communication within the cluster uses Kubernetes service DNS

## Asynchronous Messaging

Entur uses **Apache Kafka on Aiven** as the primary event streaming platform. See [docs/kafka.md](kafka.md) for full configuration and usage documentation.

### Kafka Patterns

- Use Kafka for event-driven communication between services
- Each consuming service has its own consumer group (fan-out pattern)
- Use dead-letter topics (DLT) for messages that fail processing after retries
- Use non-blocking retry with exponential backoff via `entur-kafka-spring-starter`
- Use message keys for partition ordering when message ordering matters within an entity
- Use the `entur-kafka-spring-starter` library (`org.entur.data:entur-kafka-spring-starter`) for all Kafka integration

### Cluster Selection

- **Internal clusters** (`*_INT`): for apps running inside the Kubernetes VPC
- **Public clusters** (`*_PUBLIC_*_INT`): for local development or apps outside the VPC
- **External clusters** (`*_EXT`): for external partner integrations
- Authentication: SASL/SCRAM-SHA-512 over TLS

### Message Design

- Use **Avro** (default) or **Protobuf** for message serialization -- schemas are managed via Confluent Schema Registry
- Include a correlation ID for tracing (automatically added by the Kafka starter as `X-Correlation-Id` header)
- Keep messages self-contained -- include all necessary data (avoid requiring the consumer to call back to the producer)
- Use the Gradle Avro plugin (`com.github.davidmc24.gradle.plugin.avro`) to generate classes from `.avsc` schema files

## Database Design

### PostgreSQL Conventions

- Use `snake_case` for table and column names
- Use singular table names: `route`, `stop_place` (not `routes`, `stop_places`)
- Always include `id`, `created`, `changed` (or `created_at`, `updated_at`) columns
- Use UUID or ULID for primary keys (not auto-incrementing integers) for distributed systems, or `Long` auto-increment for simpler domains
- Use Flyway (Java/Kotlin) for database migrations
- Prefix migration files with version numbers: `V1__create_route_table.sql`
- Organize migrations in subdirectories: `schema/` for DDL, `reusablefunctions/` for stored procedures, `kotlin/` for code-based migrations
- Use database views for complex read queries spanning multiple tables

### SQL Library Choice (Kotlin)

For Kotlin projects, prefer **JetBrains Exposed SQL-DSL** over JPA/Hibernate:

- Provides a typesafe Kotlin DSL close to SQL (not an ORM)
- Works naturally with Kotlin immutable `data class` models
- Better control over queries, joins, and subqueries
- Lightweight -- no magic, no lazy loading, no proxy objects

Define tables as `object` extending `LongIdTable`:

```kotlin
object VersionEntity : LongIdTable("version") {
    val netexId = varchar("netex_id", 70)
    val created = timestamp("created")
    val status = varchar("status", 70)
    val startDate = date("start_date")
    val endDate = date("end_date").nullable()
}
```

For Java projects, **Spring Data JPA** remains the default choice.

### Migration Best Practices

- Migrations must be backward-compatible (support rolling deployments)
- Separate schema changes from data migrations
- Never modify an existing migration that has been applied
- Test migrations against a copy of production data before deploying
- Use `baseline-on-migrate` in test configuration to handle existing schemas
- Configure Flyway migration locations per environment if needed

## Resilience Patterns

### Circuit Breaker

Use circuit breakers for calls to external services to prevent cascade failures:

```java
// Spring Boot with Resilience4j
@CircuitBreaker(name = "externalService", fallbackMethod = "fallback")
public RouteData fetchFromExternalService(String id) { ... }
```

### Retry with Backoff

Retry transient failures with exponential backoff:

```java
@Retry(name = "externalService", fallbackMethod = "fallback")
public RouteData fetchFromExternalService(String id) { ... }
```

### Timeouts

- Set explicit timeouts on all outgoing HTTP calls
- Connect timeout: 5 seconds
- Read timeout: 30 seconds (adjust based on expected response times)
- Never use infinite timeouts

### Graceful Degradation

- Design services to degrade gracefully when dependencies are unavailable
- Return cached data when the source is temporarily unavailable
- Return partial results rather than failing entirely

## Production Hardening

All production services must meet these requirements:

### Multiple Replicas

Production applications must run with more than 1 pod. Nodes can be downscaled at any time and workloads may need to restart.

### Horizontal Pod Autoscaler (HPA)

The common Helm chart automatically configures HPA in production:

- Default: scales between `replicas` and `maxReplicas` (default 10)
- Scaling metric: CPU utilization (default 80%)
- Customize via `hpa.spec` in Helm values
- Favor horizontal scaling over vertical scaling -- horizontal scaling handles spikes and scales down during low traffic

### Pod Disruption Budget (PDB)

The common Helm chart automatically creates a PDB in production:

- Default: `minAvailable: 50%`
- Ensures high availability during node maintenance and deployments
- In dev/tst with a single pod, set `minAvailable: 0`

### Pod Anti-Affinity

Distribute workloads across different nodes to reduce impact during node upgrades, errors, or scaling events. The common Helm chart configures this automatically.

### Zone Anti-Affinity

Distribute workloads across different availability zones in the regional cluster to survive zonal incidents. Configure this for critical production services.

### Vertical Pod Autoscaler (VPA)

VPA is enabled by default for resource recommendations. It monitors actual resource usage and suggests optimal CPU/memory requests. VPA recommendations take weeks to stabilize for new deployments.

### Resource Sizing

- Set CPU request for normal load; do not set CPU limit (CPU is compressible, allow bursting)
- Set memory request and limit to the same value (memory is incompressible, exceeding the limit causes OOM kills)
- Start with small resources and tune based on VPA recommendations

## Application Lifecycle

### Deprecating an Application

1. Verify traffic is zero using the Grafana traffic dashboard
2. Scale down to 0 replicas in GCP Console (soft delete for quick recovery)
3. Clean up Apigee proxies (undeploy in all environments, request deletion in `#talk-utviklerplattform`)
4. Request domain name removal if applicable

### Deleting an Application

1. Delete the `.entur` folder in the repository and follow the self-service workflow
2. Archive the GitHub repository
3. Clean up container artifacts if they may be accidentally used elsewhere

A deleted GCP project can be restored within 30 days; contact `#talk-utviklerplattform` for help.

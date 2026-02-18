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
- Services communicate via well-defined APIs (REST or gRPC) or asynchronous messaging (Pub/Sub)
- Services are independently deployable
- Each service has its own repository, CI/CD pipeline, and Kubernetes namespace
- Follow the golden path: repository name = application name = namespace = Helm release name
- **"You build it, you run it"** -- each team is responsible for operating its deployed services
- All versions must support rollback to the previous version
- Maintain backwards compatibility so consumers can update at their own pace
- Applications must start even when dependencies are unavailable -- never crash because a dependency is missing

### Layered Architecture (within a service)

```text
Controller / Handler       # HTTP request handling, input validation
       |
Service / Use Case         # Business logic, orchestration
       |
Repository / Client        # Data access, external service calls
       |
Domain / Model             # Domain entities, value objects
```

- **Controllers** handle HTTP concerns only: parse requests, validate input, format responses
- **Services** contain business logic and orchestrate between repositories and clients
- **Repositories** handle data persistence (database queries, cache access)
- **Clients** handle outgoing HTTP/gRPC calls to other services
- **Models/Entities** are plain objects with no framework dependencies

### Key Principles

- Depend inward: outer layers depend on inner layers, never the reverse
- Use dependency injection for testability
- Keep controllers thin -- delegate to services
- Don't let database entities leak into API responses (use DTOs/mappers)

## Infrastructure Architecture

### GCP Project Structure

- Each application has a GCP project per environment (dev, tst, prd)
- The `terraform-google-init` module discovers project configuration
- Shared networking is managed centrally (VPC, subnets)

### Data Stores

| Need | Service | Terraform Module |
|------|---------|-----------------|
| Relational data | Cloud SQL (PostgreSQL) | `terraform-google-sql-db` |
| Caching | Memorystore (Redis) | `terraform-google-memorystore` |
| Object storage | Cloud Storage | [`terraform-google-cloud-storage`](https://github.com/entur/terraform-google-cloud-storage) |
| Event streaming | Pub/Sub | Use `google_pubsub_topic` / `subscription` resources |
| Analytics | BigQuery | Use `google_bigquery_dataset` / `table` resources |

For Cloud SQL and Memorystore, always use the Entur Terraform modules -- they handle naming, networking, secret creation, and Kubernetes integration automatically.

### Connectivity

- Applications connect to Cloud SQL via the **Cloud SQL proxy sidecar** (configured in Helm with `postgres.enabled: true`)
- Applications connect to Memorystore via the **private IP** (automatically configured by the Terraform module and exposed as `REDIS_HOST` environment variable)
- Inter-service communication within the cluster uses Kubernetes service DNS

## Asynchronous Messaging

### Pub/Sub Patterns

- Use Pub/Sub for event-driven communication between services
- Each consuming service has its own subscription (fan-out pattern)
- Use dead-letter topics for messages that fail processing
- Set appropriate acknowledgment deadlines and retry policies
- Use ordering keys when message ordering matters

### Message Design

- Use JSON or Protobuf for message serialization
- Include a message type/version field for forward compatibility
- Include a correlation ID for tracing
- Keep messages self-contained -- include all necessary data (avoid requiring the consumer to call back to the producer)

## Database Design

### PostgreSQL Conventions

- Use `snake_case` for table and column names
- Use singular table names: `route`, `stop_place` (not `routes`, `stop_places`)
- Always include `id`, `created_at`, `updated_at` columns
- Use UUID or ULID for primary keys (not auto-incrementing integers) for distributed systems
- Use Flyway (Java/Kotlin) for database migrations
- Prefix migration files with version numbers: `V1__create_route_table.sql`

### Migration Best Practices

- Migrations must be backward-compatible (support rolling deployments)
- Separate schema changes from data migrations
- Never modify an existing migration that has been applied
- Test migrations against a copy of production data before deploying

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

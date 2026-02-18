# Observability Standards

All Entur services must expose health checks, Prometheus metrics, and distributed tracing. This enables monitoring, alerting, and debugging across the platform.

## Health Checks

Every service must expose liveness and readiness probes. These are used by Kubernetes to manage pod lifecycle.

### Liveness Probe

Answers "is the process running and not deadlocked?"

- Return `200 OK` if the application is alive
- Do NOT check external dependencies (database, cache) -- a slow database should not cause restarts
- Default path: `/actuator/health/liveness` (Spring Boot) or custom path for Go/Python

### Readiness Probe

Answers "is the application ready to serve traffic?"

- Return `200 OK` if the application can serve requests
- Check only **private resources** owned by this service (its own database connection pool, internal cache)
- **Never check shared or external services** in readiness probes -- if a shared service is down, all pods will be removed from routing simultaneously, making the entire service completely unavailable
- Return `503 Service Unavailable` if a private dependency is down
- Default path: `/actuator/health/readiness` (Spring Boot) or custom path for Go/Python

### Helm Configuration

The Entur common Helm chart configures probes automatically. Defaults:

```yaml
common:
  container:
    probes:
      enabled: true
      liveness:
        path: /actuator/health/liveness
      readiness:
        path: /actuator/health/readiness
```

For non-Spring-Boot services (Go, Python), override the paths:

```yaml
common:
  container:
    probes:
      liveness:
        path: /health/liveness
      readiness:
        path: /health/readiness
```

## Prometheus Metrics

### Enabling Metrics

#### Spring Boot

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  metrics:
    tags:
      application: ${spring.application.name}
```

Add the `micrometer-registry-prometheus` dependency.

#### Go

```go
import "github.com/prometheus/client_golang/prometheus/promhttp"

mux.Handle("GET /metrics", promhttp.Handler())
```

#### Python

```python
from prometheus_client import start_http_server, Counter

REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'path', 'status'])
```

### Metrics Helm Configuration

```yaml
common:
  container:
    prometheus:
      enabled: true
      path: /actuator/prometheus    # Spring Boot default
      # path: /metrics              # Go / Python
```

### Standard Metrics

All services should expose at minimum:

| Metric | Type | Description |
|--------|------|-------------|
| `http_server_requests_seconds` | Histogram | Request duration by method, path, status |
| `jvm_memory_used_bytes` | Gauge | JVM memory usage (Java/Kotlin) |
| `process_cpu_usage` | Gauge | CPU usage |
| `db_pool_active_connections` | Gauge | Database connection pool usage |

Spring Boot with Actuator and Micrometer provides all of these automatically.

### Custom Metrics

Name custom metrics following Prometheus conventions:

- Use `snake_case`
- Include unit as suffix: `_seconds`, `_bytes`, `_total`
- Use labels for dimensions (e.g. `route`, `status`)

```java
// Java/Kotlin
Counter.builder("routes_processed_total")
    .description("Total routes processed")
    .tag("status", "success")
    .register(meterRegistry)
    .increment();
```

```go
// Go
routesProcessed := promauto.NewCounterVec(prometheus.CounterOpts{
    Name: "routes_processed_total",
    Help: "Total routes processed",
}, []string{"status"})

routesProcessed.WithLabelValues("success").Inc()
```

## Distributed Tracing

### Tracing with Spring Boot

Use Micrometer Tracing (successor to Spring Cloud Sleuth) with OpenTelemetry:

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0    # 100% sampling in dev/tst, lower in prd

# Add dependencies:
# micrometer-tracing-bridge-otel
# opentelemetry-exporter-otlp
```

### Tracing with Go

Use OpenTelemetry SDK:

```go
import "go.opentelemetry.io/otel"

tracer := otel.Tracer("my-service")
ctx, span := tracer.Start(ctx, "findRoute")
defer span.End()
```

### Trace Propagation

- Use W3C Trace Context headers (`traceparent`, `tracestate`) for cross-service propagation
- Include `traceId` in all log entries for log-trace correlation
- Google Cloud Trace automatically ingests traces from the OpenTelemetry exporter

## Google Cloud Profiler

Cloud Profiler provides continuous, low-overhead production profiling. It samples CPU usage, memory allocation, and lock contention, mapping resource cost back to specific source code.

### Enabling Profiler

1. Update the trigger in the `.entur` directory to enable the Profiler API for the application
2. Attach the profiler agent to your application

For Java (Spring Boot on GKE), start the JVM with the Cloud Profiler agent:

```bash
java \
  -agentpath:/path/to/profiler_java_agent.so \
  -Dcom.google.cprof.service=my-service-name \
  -Dcom.google.cprof.service_version=1.0.0 \
  -cprof_project_id <your_app_project_id> \
  -jar app.jar
```

Use the **application's own GCP project ID**, not the project where the Kubernetes cluster runs.

View profiles in the Google Cloud Console under **Profiler** for the relevant project.

## Grafana Dashboards

Key dashboards for operations:

- **VPA recommendations**: Search for "kubernetes-vpa-recommendations" in `grafana.entur.org` -- select cluster, namespace, and target to see recommended CPU/memory settings
- **PDB compliance**: Search for "kubernetes-poddisruptionbudget" in `grafana.entur.org` -- find deployments missing proper PDB configuration
- **Traffic per service**: Search for "traffic-pr-service" in `grafana.entur.org` -- verify traffic levels before deprecating a service

Use `kubectl describe vpa <deployment-name> -n <namespace>` for VPA recommendations via CLI. Note that VPA recommendations take weeks to stabilize for new deployments.

## Alerting

### Recommended Alerts

| Alert | Condition | Severity |
|-------|-----------|----------|
| High error rate | 5xx rate > 5% for 5 minutes | Critical |
| High latency | p99 latency > 5s for 10 minutes | Warning |
| Pod restarts | > 3 restarts in 15 minutes | Warning |
| CPU saturation | CPU usage > 80% for 10 minutes | Warning |
| Memory saturation | Memory usage > 85% for 5 minutes | Critical |
| Health check failing | Readiness probe failing for 3 minutes | Critical |

Configure alerts in your monitoring setup (Google Cloud Monitoring or Prometheus AlertManager).

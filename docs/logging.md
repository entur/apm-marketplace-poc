# Structured Logging Standards

All Entur services must use structured logging in JSON format. This enables efficient log querying in Google Cloud Logging and correlation across distributed services.

## Format

All log output must be JSON, written to stdout. Google Cloud Logging automatically ingests stdout from Kubernetes pods.

### Required Fields

| Field | Description | Example |
|-------|-------------|---------|
| `timestamp` | ISO 8601 timestamp | `2025-01-15T10:30:00.123Z` |
| `severity` / `level` | Log level | `INFO`, `WARN`, `ERROR` |
| `message` / `msg` | Human-readable log message | `"Request processed"` |

### Recommended Fields

| Field | Description | Example |
|-------|-------------|---------|
| `logger` | Logger name (class/package) | `no.entur.myapp.RouteService` |
| `traceId` | Distributed trace ID | `abc123def456` |
| `spanId` | Current span ID | `789ghi` |
| `requestId` | HTTP request correlation ID | `req-abc-123` |
| `application` | Application name | `my-application` |
| `environment` | Runtime environment | `dev`, `tst`, `prd` |

## Implementation by Language

### Java / Kotlin (Spring Boot)

Use [entur/cloud-logging](https://github.com/entur/cloud-logging) -- Entur's standard logging library for JVM applications. It provides plug-and-play structured JSON logging for GCP with no manual `logback.xml` configuration required.

```kotlin
// build.gradle.kts -- add the BOM and GCP web starter
dependencies {
    implementation(platform("no.entur.logging.cloud:bom:$cloudLoggingVersion"))
    implementation("no.entur.logging.cloud:spring-boot-starter-gcp-web")
    testImplementation("no.entur.logging.cloud:spring-boot-starter-gcp-web-test")
}
```

Use standard SLF4J -- cloud-logging handles JSON output, GCP severity mapping, and correlation automatically:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private static final Logger LOG = LoggerFactory.getLogger(RouteService.class);

LOG.info("Route found for id {}", routeId);
LOG.error("Failed to fetch route {}", routeId, exception);
```

Optional features (add the corresponding starters):

- **Request-response logging** (`request-response-spring-boot-starter-gcp-web`) -- Logbook-based HTTP body logging
- **On-demand logging** (`on-demand-spring-boot-starter-gcp-web`) -- reduce logging costs by buffering and only flushing full logs for failed requests

See [java.md](java.md) for full setup details.

### Go

Use [entur/go-logging](https://github.com/entur/go-logging) -- Entur's standard logging SDK for Go on GCP:

```go
import "github.com/entur/go-logging"

logging.Info().Str("routeId", routeId).Str("origin", origin).Msg("route found")
logging.Error().Err(err).Str("routeId", routeId).Msg("failed to fetch route")
```

The SDK handles JSON output, caller location, and GCP-compatible log levels automatically. Default log level is read from the `LOG_LEVEL` environment variable (defaults to `warning` if unset). See [go.md](go.md) for full usage details.

### Python

Use the standard `logging` module with JSON formatting:

```python
import logging
import json_log_formatter

formatter = json_log_formatter.JSONFormatter()
handler = logging.StreamHandler()
handler.setFormatter(formatter)

logger = logging.getLogger("my_application")
logger.addHandler(handler)
logger.setLevel(logging.INFO)

logger.info("Route found", extra={"route_id": route_id, "origin": origin})
```

## Log Levels

| Level | Use for | Example |
|-------|---------|---------|
| `ERROR` | Unexpected failures requiring attention | Database connection lost, unhandled exception |
| `WARN` | Expected but unusual conditions | Retry attempt, deprecation warning, rate limit approached |
| `INFO` | Normal operational events | Request processed, job completed, startup/shutdown |
| `DEBUG` | Diagnostic detail for troubleshooting | Query parameters, cache hit/miss, detailed flow |

### Guidelines

- **Production** should run at `INFO` level by default
- **Never log secrets**, tokens, passwords, or PII (personally identifiable information)
- **Never log payment details** (PCI-DSS compliance)
- **Never log request/response bodies** at INFO level -- use DEBUG
- **Log at the boundary**: log when entering/exiting the system (HTTP requests, message consumption), not inside every method
- **Include context**: always include enough context to trace a log entry back to a specific request or operation
- **Don't log and throw**: either log the error or propagate it, not both (to avoid duplicate log entries)
- **Prevent log injection**: all logging components must appropriately encode user-supplied data to prevent log forging
- Session tokens must only appear in logs in irreversible, hashed form

### Security Events to Log

Log these events for security monitoring and incident investigation:

- Successful and failed authentication attempts
- Access control failures (unauthorized access attempts)
- Input validation failures
- Deserialization failures
- Application startup and shutdown

## Correlation

### Distributed Tracing

- Spring Boot applications should use Micrometer Tracing to propagate trace context
- Go applications should propagate the `X-Cloud-Trace-Context` header
- Include `traceId` and `spanId` in all log entries for cross-service correlation

### Request IDs

- Generate a unique request ID at the ingress point if one is not present in the `X-Request-ID` header
- Propagate the request ID through all downstream calls
- Include the request ID in all log entries and error responses

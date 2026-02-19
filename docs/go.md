# Go Standards

Go conventions for Entur applications. Read [CONVENTIONS.md](../CONVENTIONS.md) first for cross-language standards.

## Runtime and Build

- **Go version**: latest stable (currently 1.23+)
- **Modules**: Go modules (`go.mod`)
- **Linting**: `golangci-lint`
- **Framework**: standard library `net/http` (Go 1.22+ routing) or minimal frameworks only

## Project Structure

```text
my-service/
  cmd/
    my-service/
      main.go               # Entry point
  internal/
    handler/                # HTTP handlers
    service/                # Business logic
    repository/             # Data access
    model/                  # Domain types
    config/                 # Configuration loading
  pkg/                      # Public reusable packages (if any)
  Dockerfile
  go.mod
  go.sum
  .golangci.yml
```

- `internal/` for application-private code (enforced by Go compiler)
- `cmd/` for application entry points
- `pkg/` only for code intended to be imported by other projects (rare)

## Dockerfile

```dockerfile
# Build stage
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server ./cmd/my-service

# Runtime stage
FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/server /server
EXPOSE 8080
ENTRYPOINT ["/server"]
```

- Multi-stage build to minimize image size
- `distroless` base image for security (no shell, no package manager)
- `nonroot` variant runs as non-root user

## Application Patterns

### Main Entry Point

```go
package main

import (
    "context"
    "net/http"
    "os"
    "os/signal"
    "syscall"
    "time"

    "github.com/entur/go-logging"
)

func main() {
    cfg, err := config.Load()
    if err != nil {
        logging.Error().Err(err).Msg("failed to load configuration")
        os.Exit(1)
    }

    mux := http.NewServeMux()
    registerRoutes(mux, cfg)

    server := &http.Server{
        Addr:         ":" + cfg.Port,
        Handler:      mux,
        ReadTimeout:  10 * time.Second,
        WriteTimeout: 30 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    // Graceful shutdown
    ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
    defer stop()

    go func() {
        logging.Info().Msgf("server starting on port %s", cfg.Port)
        if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
            logging.Error().Err(err).Msg("server failed")
            os.Exit(1)
        }
    }()

    <-ctx.Done()
    logging.Info().Msg("shutting down gracefully")

    shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    if err := server.Shutdown(shutdownCtx); err != nil {
        logging.Error().Err(err).Msg("forced shutdown")
    }
}
```

### Health Checks

```go
func registerRoutes(mux *http.ServeMux, cfg *config.Config) {
    mux.HandleFunc("GET /health/liveness", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"status":"UP"}`))
    })

    mux.HandleFunc("GET /health/readiness", func(w http.ResponseWriter, r *http.Request) {
        // Check dependencies (database, etc.)
        if err := cfg.DB.PingContext(r.Context()); err != nil {
            w.WriteHeader(http.StatusServiceUnavailable)
            w.Write([]byte(`{"status":"DOWN"}`))
            return
        }
        w.WriteHeader(http.StatusOK)
        w.Write([]byte(`{"status":"UP"}`))
    })
}
```

Update Helm values to point to these custom paths:

```yaml
common:
  container:
    probes:
      liveness:
        path: /health/liveness
      readiness:
        path: /health/readiness
```

### HTTP Handlers

```go
type RouteHandler struct {
    service *RouteService
    logger  *logging.Logger
}

func NewRouteHandler(service *RouteService) *RouteHandler {
    return &RouteHandler{
        service: service,
        logger:  logging.New(),
    }
}

func (h *RouteHandler) GetRoute(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")

    route, err := h.service.FindByID(r.Context(), id)
    if err != nil {
        h.logger.Error().Err(err).Str("id", id).Msg("failed to find route")
        http.Error(w, "internal server error", http.StatusInternalServerError)
        return
    }
    if route == nil {
        http.Error(w, "route not found", http.StatusNotFound)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(route)
}
```

### Error Handling

```go
// Define sentinel errors for known conditions
var (
    ErrRouteNotFound = errors.New("route not found")
    ErrInvalidInput  = errors.New("invalid input")
)

// Wrap errors with context
func (s *RouteService) FindByID(ctx context.Context, id string) (*Route, error) {
    route, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return nil, fmt.Errorf("finding route %s: %w", id, err)
    }
    return route, nil
}
```

- Always wrap errors with `fmt.Errorf("context: %w", err)` to preserve the error chain
- Use sentinel errors for known, expected error conditions
- Use `errors.Is()` and `errors.As()` to check error types
- Never ignore errors -- handle or propagate every one

### Configuration

```go
type Config struct {
    Port        string `env:"PORT" envDefault:"8080"`
    DatabaseURL string `env:"DATABASE_URL,required"`
    LogLevel    string `env:"LOG_LEVEL" envDefault:"info"`
}

func Load() (*Config, error) {
    cfg := &Config{}
    if err := env.Parse(cfg); err != nil {
        return nil, fmt.Errorf("parsing config: %w", err)
    }
    return cfg, nil
}
```

Use environment variables for all configuration. Use a library like `caarlos0/env` or parse `os.Getenv` directly.

## Testing

```go
func TestRouteService_FindByID(t *testing.T) {
    t.Run("returns route when found", func(t *testing.T) {
        repo := &mockRepo{
            routes: map[string]*Route{"r1": {ID: "r1", Origin: "Oslo"}},
        }
        svc := NewRouteService(repo)

        route, err := svc.FindByID(context.Background(), "r1")

        require.NoError(t, err)
        assert.Equal(t, "r1", route.ID)
        assert.Equal(t, "Oslo", route.Origin)
    })

    t.Run("returns nil when not found", func(t *testing.T) {
        repo := &mockRepo{routes: map[string]*Route{}}
        svc := NewRouteService(repo)

        route, err := svc.FindByID(context.Background(), "unknown")

        require.NoError(t, err)
        assert.Nil(t, route)
    })
}
```

- Use `testing` package with `t.Run` for subtests
- Use `testify/require` for fatal assertions, `testify/assert` for non-fatal
- Use table-driven tests for multiple input scenarios
- Write mocks by hand (implement the interface) or use `testify/mock`
- Use `testcontainers-go` for integration tests with databases

## Redis (Memorystore)

Entur uses **Google Memorystore for Redis** as a managed key-value store. Infrastructure is provisioned via the `terraform-google-memorystore` Terraform module (see [terraform/modules.md](terraform/modules.md#memorystore-redis)). Connection credentials (`REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD`) are injected via Kubernetes secrets.

### Client Library

Use [`go-redis/redis`](https://github.com/redis/go-redis) (v9+):

```bash
go get github.com/redis/go-redis/v9
```

### Connection Setup

```go
package redis

import (
    "context"
    "fmt"
    "time"

    "github.com/redis/go-redis/v9"
)

func NewClient(cfg *Config) (*redis.Client, error) {
    client := redis.NewClient(&redis.Options{
        Addr:         fmt.Sprintf("%s:%s", cfg.RedisHost, cfg.RedisPort),
        Password:     cfg.RedisPassword,
        DB:           0,
        DialTimeout:  1 * time.Second,
        ReadTimeout:  2 * time.Second,
        WriteTimeout: 2 * time.Second,
        PoolSize:     10,
        MinIdleConns: 2,
    })

    ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
    defer cancel()

    if err := client.Ping(ctx).Err(); err != nil {
        return nil, fmt.Errorf("connecting to redis: %w", err)
    }
    return client, nil
}
```

### Configuration

Load connection details from environment variables:

```go
type Config struct {
    RedisHost     string `env:"REDIS_HOST,required"`
    RedisPort     string `env:"REDIS_PORT" envDefault:"6379"`
    RedisPassword string `env:"REDIS_PASSWORD,required"`
}
```

### Basic Operations

```go
// SET with TTL
err := client.Set(ctx, "myapp:route:123", routeJSON, 10*time.Minute).Err()

// GET
val, err := client.Get(ctx, "myapp:route:123").Result()
if errors.Is(err, redis.Nil) {
    // Key does not exist -- cache miss, fall back to database
} else if err != nil {
    // Redis error -- log and fall back to database
}

// DELETE
err = client.Del(ctx, "myapp:route:123").Err()

// SET if not exists (distributed lock / idempotency)
ok, err := client.SetNX(ctx, "myapp:lock:import-job", "owner-1", 60*time.Second).Result()
if ok {
    // Lock acquired
}

// Atomic increment (rate limiting)
count, err := client.Incr(ctx, "myapp:rate:client-xyz").Result()
if count == 1 {
    client.Expire(ctx, "myapp:rate:client-xyz", 1*time.Minute)
}
```

### Cache-Aside Pattern

```go
type RouteCache struct {
    redis *redis.Client
    repo  RouteRepository
    ttl   time.Duration
}

func (c *RouteCache) GetRoute(ctx context.Context, id string) (*Route, error) {
    key := "myapp:route:" + id

    // Try cache first
    val, err := c.redis.Get(ctx, key).Bytes()
    if err == nil {
        var route Route
        if err := json.Unmarshal(val, &route); err == nil {
            return &route, nil
        }
    }

    // Cache miss or error -- fall back to database
    route, err := c.repo.FindByID(ctx, id)
    if err != nil {
        return nil, err
    }
    if route == nil {
        return nil, nil
    }

    // Store in cache (best-effort -- don't fail if Redis is down)
    if data, err := json.Marshal(route); err == nil {
        _ = c.redis.Set(ctx, key, data, c.ttl).Err()
    }

    return route, nil
}

func (c *RouteCache) InvalidateRoute(ctx context.Context, id string) {
    _ = c.redis.Del(ctx, "myapp:route:"+id).Err()
}
```

### Key Naming Conventions

```text
{app}:{domain}:{id}           -- entity cache
{app}:rate:{clientId}          -- rate limiting
{app}:lock:{resource}          -- distributed locks
{app}:dedup:{messageId}        -- idempotency keys
```

### Best Practices

- **Always set TTLs** -- never store keys without expiration. Unbounded growth exhausts memory.
- **Handle failures gracefully** -- Redis is a cache, not a primary store. If Redis is unavailable, fall back to the database. Never let a Redis outage cause the application to crash.
- **Keep values small** -- serialize to JSON, aim for < 100 KB per key.
- **Use `SetNX`** for distributed locks and idempotency keys.
- **Avoid `Keys("*")`** in production -- it blocks the single-threaded Redis server. Use `Scan` for iteration.
- **Namespace keys** with the application name to avoid collisions when multiple apps share a Redis instance.
- **Use pipelining** (`client.Pipeline()`) for batch operations to reduce round trips.
- **Do not use Redis as a message queue** -- use Kafka instead.

### Health Check with Redis

Include Redis in the readiness probe:

```go
mux.HandleFunc("GET /health/readiness", func(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()

    // Check database
    if err := db.PingContext(ctx); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        w.Write([]byte(`{"status":"DOWN","reason":"database"}`))
        return
    }

    // Check Redis (only if it's a private resource for this service)
    if err := redisClient.Ping(ctx).Err(); err != nil {
        w.WriteHeader(http.StatusServiceUnavailable)
        w.Write([]byte(`{"status":"DOWN","reason":"redis"}`))
        return
    }

    w.WriteHeader(http.StatusOK)
    w.Write([]byte(`{"status":"UP"}`))
})
```

**Note**: Only include Redis in the readiness check if the Redis instance is a **private resource owned by this service**. If Redis is shared across services, do not check it in the readiness probe -- see [observability.md](observability.md#readiness-probe).

### Testing

Use Testcontainers for integration tests:

```go
import (
    "github.com/testcontainers/testcontainers-go"
    "github.com/testcontainers/testcontainers-go/wait"
)

func setupRedis(t *testing.T) *redis.Client {
    ctx := context.Background()
    container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
        ContainerRequest: testcontainers.ContainerRequest{
            Image:        "redis:7-alpine",
            ExposedPorts: []string{"6379/tcp"},
            WaitingFor:   wait.ForListeningPort("6379/tcp"),
        },
        Started: true,
    })
    require.NoError(t, err)
    t.Cleanup(func() { container.Terminate(ctx) })

    host, _ := container.Host(ctx)
    port, _ := container.MappedPort(ctx, "6379")

    return redis.NewClient(&redis.Options{
        Addr: fmt.Sprintf("%s:%s", host, port.Port()),
    })
}
```

## Prometheus Metrics

```go
import "github.com/prometheus/client_golang/prometheus/promhttp"

mux.Handle("GET /metrics", promhttp.Handler())
```

Update Helm values:

```yaml
common:
  container:
    prometheus:
      enabled: true
      path: /metrics
```

## Logging

Use [entur/go-logging](https://github.com/entur/go-logging) -- Entur's standard logging SDK for Go services on GCP. It provides structured JSON output, automatic caller location, optional stacktraces, and colorful local output.

### Install

```bash
go get github.com/entur/go-logging
```

### Usage

```go
import "github.com/entur/go-logging"

// Global logger (includes caller info automatically)
logging.Info().Msg("request processed")
logging.Error().Err(err).Str("query", queryName).Msg("database query failed")

// Formatted messages
logging.Info().Msgf("processed %d routes in %dms", count, elapsed.Milliseconds())

// Instance logger (for dependency injection)
logger := logging.New()
logger.Info().Msg("request processed")

// Instance logger with custom level
logger = logging.New(logging.WithLevel(logging.DebugLevel))
logger.Debug().Msg("cache miss")

// Disable caller info when not needed
logger = logging.New(logging.WithNoCaller())
```

### Log Level

The default log level is read from the `LOG_LEVEL` environment variable. If not set, it defaults to `warning`.

Valid values: `fatal`, `panic`, `error`, `warning`, `info`, `debug`, `trace` (or short forms: `ftl`, `pnc`, `err`, `wrn`, `inf`, `dbg`, `trc`).

Set it via Helm environment variables:

```yaml
common:
  container:
    env:
      - name: LOG_LEVEL
        value: "info"    # info for dev/tst, warning for prd
```

### Errors with Stacktraces

Use `logging.NewStackTraceError()` to capture stacktraces at the point of error creation:

```go
// New error with stacktrace
err := logging.NewStackTraceError("route not found: %s", routeID)

// Wrap an existing error with stacktrace
err = logging.NewStackTraceError("%w", existingErr)

// Log it -- stacktrace is included automatically
logging.Error().Err(err).Msg("an internal error occurred")
```

See [logging.md](logging.md) for general logging standards.

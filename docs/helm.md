# Entur Common Helm Chart

Reference: [entur/helm-charts](https://github.com/entur/helm-charts)

The Entur `common` Helm chart is the standard base chart for deploying applications to Kubernetes. It provides sensible defaults for Spring Boot applications and can be configured for Go, Python, or any containerized service.

## Naming Convention

Application name = Git repository name = backend URL (`yourapp.entur.io`). Must be:

- Unique across Entur
- Max 63 characters
- Only characters `[a-z0-9]` and dash `-`

## Setup

### Chart.yaml

```yaml
apiVersion: v2
name: my-application
version: 0.1.0
dependencies:
  - name: common
    version: "1.x.x"
    repository: "https://entur.github.io/helm-charts"
    alias: common
```

Run `helm dependency update` after creating or modifying `Chart.yaml`.

### Directory Structure

```text
helm/
  my-application/
    Chart.yaml
    Chart.lock
    values.yaml           # Default values
    env/
      dev.yaml            # Dev overrides
      tst.yaml            # Test overrides
      prd.yaml            # Production overrides
    tests/                # Helm unit tests (optional)
      deployment_test.yaml
```

## Required Values

Every deployment must set these values:

```yaml
# values.yaml
common:
  app: my-application
  shortname: myapp          # Max 10 characters, used for GCP 2.0 app ID
  team: my-team             # Team name without "team-" prefix
  container:
    image: my-application   # Docker image name (without registry/tag)
```

Environment-specific:

```yaml
# env/dev.yaml
common:
  env: dev
```

```yaml
# env/prd.yaml
common:
  env: prd
```

## Container Configuration

### Resources

```yaml
common:
  container:
    cpu: 0.2          # CPU request in cores (200m)
    memory: 256        # Memory request in Mi
    memoryLimit: 256   # Set equal to memory request
```

Best practice for resource limits:

- **CPU limit**: Do not set. CPU is compressible -- pods are throttled, not killed. Allow bursting.
- **Memory limit**: Set equal to memory request. Memory is incompressible -- exceeding the limit causes OOM kills.

Start with small values and let VPA recommend optimal settings. See [Resource Sizing Best Practices](#resource-sizing-best-practices) below.

### Replicas and Scaling

```yaml
common:
  container:
    replicas: 2              # Desired replicas (set to 1 for Recreate strategy)

  deployment:
    maxReplicas: 10          # HPA maximum (default: 10)
    minAvailable: "50%"      # PDB minimum available (default: 50%)
```

- In `prd`, HPA and PDB are enabled automatically when replicas > 1
- Setting `replicas: 1` uses Recreate deployment strategy (no PDB, no HPA)
- HPA scales on CPU utilization (80% threshold by default)
- **PDB percentage gotcha**: Kubernetes rounds `minAvailable` percentages up. With 3 replicas and `minAvailable: 80%`, 80% of 3 = 2.4, rounded to 3 -- effectively preventing all disruption. Use `50%` (the default) or ensure enough replicas.
- In dev/tst environments with a single pod, set `minAvailable: 0` to allow cluster operations

### Health Probes

Default paths (Spring Boot):

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

**Probe design rules:**

- **Liveness probe**: Verify the application responds to requests within a reasonable timeframe. Do not check external dependencies -- a failing liveness probe causes pod restarts.
- **Readiness probe**: Check only **private resources** (own database, internal cache). Never check shared or external services in readiness probes -- if the shared service is down, all pods will be removed from routing simultaneously, making the entire service unavailable.

For non-Spring-Boot applications (Go, Python):

```yaml
common:
  container:
    probes:
      liveness:
        path: /health/liveness
      readiness:
        path: /health/readiness
```

Custom probe spec (full Kubernetes probe configuration):

```yaml
common:
  container:
    probes:
      spec:
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Prometheus Metrics

```yaml
common:
  container:
    prometheus:
      enabled: true
      path: /actuator/prometheus    # Spring Boot
      # path: /metrics              # Go / Python
```

### Ports

```yaml
common:
  service:
    internalPort: 8080    # Container port (default)
    externalPort: 80      # Service port (default)
```

## Networking

### Ingress

```yaml
common:
  ingress:
    enabled: true                    # Default: true
    trafficType: api                 # Required: "api", "public", or "http2"
```

| Traffic Type | Description |
|-------------|-------------|
| `api` | Internal API traffic (default for backend services) |
| `public` | Public-facing traffic (internet-accessible) |
| `http2` | gRPC / HTTP/2 traffic |

### gRPC

```yaml
common:
  grpc: true
  ingress:
    trafficType: http2
```

When `grpc: true`, the chart automatically:

- Sets appropriate annotations for gRPC
- Configures gRPC health checking probes

## Database (Cloud SQL Proxy)

Enable the Cloud SQL proxy sidecar:

```yaml
common:
  postgres:
    enabled: true
```

This injects a Cloud SQL proxy sidecar container that handles authentication and connectivity to Cloud SQL. The application connects to `localhost:5432`.

Database credentials are provided as environment variables from Kubernetes secrets (created by the Terraform `terraform-google-sql-db` module):

- `PG_USER` -- database username
- `PG_PASSWORD` -- database password
- Database name is configured via the application's own configuration

## Secrets (ExternalSecrets)

Sync secrets from Google Secret Manager to Kubernetes:

```yaml
common:
  secrets:
    my-app-secrets:           # K8s Secret name
      - API_KEY               # Secret Manager secret -> env var name
      - EXTERNAL_SERVICE_KEY
    database-credentials:     # Another K8s Secret
      - PG_USER
      - PG_PASSWORD
```

Each entry creates an ExternalSecret that syncs the named secrets from Google Secret Manager into a Kubernetes Secret, which is then mounted as environment variables.

## CronJobs

```yaml
common:
  cron:
    enabled: true
    schedule: "0 2 * * *"    # Daily at 02:00 UTC
```

## Environment Variables

Set additional environment variables:

```yaml
common:
  container:
    envFrom:
      - configMapRef:
          name: my-config
    env:
      - name: SPRING_PROFILES_ACTIVE
        value: "cloud"
      - name: JAVA_TOOL_OPTIONS
        value: "-Xmx512m"
```

## Custom HPA Spec

Override the default HPA configuration:

```yaml
common:
  hpa:
    spec:
      minReplicas: 2
      maxReplicas: 20
      metrics:
        - type: Resource
          resource:
            name: cpu
            target:
              type: Utilization
              averageUtilization: 70
```

## Resource Sizing Best Practices

| Resource | Recommendation |
|----------|---------------|
| CPU request | Set for normal load with some overhead. CPU is compressible -- pods are throttled, not killed. |
| CPU limit | **Do not set.** CPU bursting is allowed and preferred. Nodes are often underutilized. |
| Memory request | Set as close as possible to real usage including burst usage for critical processes. |
| Memory limit | **Set equal to memory request.** Memory is incompressible -- exceeding the limit causes OOM kills. |

Use VPA recommendations (enabled on all clusters) to tune resource settings over time. See [observability](observability.md) for Grafana dashboards.

Example:

```yaml
common:
  container:
    cpu: 0.3         # request only, no limit
    memory: 512      # request
    memoryLimit: 512  # limit = request
```

## Local Debugging

```bash
# Lint (check for errors):
helm lint helm/my-application/ -f helm/my-application/env/dev.yaml

# Template (render K8s YAML locally):
helm template my-application helm/my-application/ -f helm/my-application/env/dev.yaml
```

## Complete Example (Spring Boot)

```yaml
# values.yaml
common:
  app: route-service
  shortname: routesvc
  team: journey-planning
  container:
    image: route-service
    cpu: 0.2
    memory: 256
    replicas: 2
    probes:
      enabled: true
    prometheus:
      enabled: true
  service:
    internalPort: 8080
  ingress:
    enabled: true
    trafficType: api
  postgres:
    enabled: true
  secrets:
    route-service-secrets:
      - API_KEY
```

```yaml
# env/dev.yaml
common:
  env: dev
  container:
    replicas: 1
    cpu: 0.1
    memory: 128
```

```yaml
# env/prd.yaml
common:
  env: prd
  container:
    replicas: 2
    cpu: 0.5
    memory: 512
  deployment:
    maxReplicas: 10
```

## Complete Example (Go Service)

```yaml
# values.yaml
common:
  app: stop-lookup
  shortname: stoplkup
  team: data-platform
  container:
    image: stop-lookup
    cpu: 0.1
    memory: 64
    replicas: 2
    probes:
      liveness:
        path: /health/liveness
      readiness:
        path: /health/readiness
    prometheus:
      enabled: true
      path: /metrics
  service:
    internalPort: 8080
  ingress:
    enabled: true
    trafficType: api
```

## Helm Unit Testing

Use [helm-unittest](https://github.com/helm-unittest/helm-unittest) for chart testing:

```yaml
# tests/deployment_test.yaml
suite: deployment tests
templates:
  - templates/deployment.yaml
tests:
  - it: should set the correct image
    set:
      common.container.image: my-app
    asserts:
      - contains:
          path: spec.template.spec.containers
          content:
            image: my-app
```

Run in CI with:

```yaml
jobs:
  helm-unittest:
    uses: entur/gha-helm/.github/workflows/unittest.yml@v1
```

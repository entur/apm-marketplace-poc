# Containerization with Docker

All Entur applications deployed to Kubernetes are packaged as Docker container images. This document covers Dockerfile conventions, base images, CI/CD integration, and security best practices.

## Conventions

- The Dockerfile lives at the **repository root**
- The image name follows the golden path convention: **repository name = application name = Docker image name**
- Images are pushed to **Google Artifact Registry** via the [gha-docker](https://github.com/entur/gha-docker) reusable workflows
- Dockerfiles are linted with **Hadolint** via the `gha-docker` lint workflow

## Base Images by Language

Prefer **distroless** or **slim-musl** images over full Alpine when possible. These images have a smaller attack surface (no shell, no package manager), smaller size, and better security. Use Alpine only when you need a shell, package manager, or in-container debugging.

| Language | Recommended | Alternative |
|----------|------------|-------------|
| Java/Kotlin | `bellsoft/liberica-runtime-container:jre-25-cds-slim-musl` | `eclipse-temurin:21-jre-alpine` |
| Go | `gcr.io/distroless/static-debian12:nonroot` | `golang:1.23-alpine` (build only) |
| Node.js | `gcr.io/distroless/nodejs24-debian12` | `node:24-alpine` |
| Python | `gcr.io/distroless/python3-debian12` | `python:3.12-slim` |

For Java/Kotlin, the **Liberica Runtime Container with CDS** is preferred because it supports Class Data Sharing for faster startup and is optimized for containerized JVM workloads.

Pin base image versions to a specific tag or digest. Never use `latest`.

## Dockerfile Examples

### Java / Kotlin (Preferred: Multi-Stage with Layered JAR and CDS)

The preferred approach uses four Docker stages for optimal build caching, image size, and startup performance:

1. **Bundler** -- bundle OpenAPI spec (if contract-first)
2. **Builder** -- compile and package the application
3. **Layers** -- extract Spring Boot layered JAR
4. **Run** -- minimal runtime image with CDS support

```dockerfile
# Stage 1: Bundle OpenAPI specification (contract-first only)
FROM node:25-slim AS bundler
WORKDIR /app
COPY specs specs
RUN npx @redocly/cli bundle specs/products.yaml --output specs/openapi.json

# Stage 2: Build the application
FROM gradle:9.3.1-jdk25-alpine AS builder
WORKDIR /app
COPY build.gradle.kts settings.gradle.kts ./
COPY gradle/libs.versions.toml gradle/libs.versions.toml

# Download dependencies first (cache layer)
RUN --mount=type=secret,id=ARTIFACTORY_AUTH_USER,env=ARTIFACTORY_AUTH_USER  \
    --mount=type=secret,id=ARTIFACTORY_AUTH_TOKEN,env=ARTIFACTORY_AUTH_TOKEN \
    gradle dependencies --no-daemon

COPY src src
COPY --from=bundler /app/specs/openapi.json specs/openapi.json

RUN --mount=type=secret,id=ARTIFACTORY_AUTH_USER,env=ARTIFACTORY_AUTH_USER  \
    --mount=type=secret,id=ARTIFACTORY_AUTH_TOKEN,env=ARTIFACTORY_AUTH_TOKEN \
    gradle bootJar -x clean -x bundleOpenApiSpecification --no-daemon

# Stage 3: Extract layered JAR
FROM bellsoft/liberica-runtime-container:jre-25-cds-slim-musl AS layers
WORKDIR /app
COPY --from=builder /app/build/libs/my-app.jar my-app.jar
RUN java -Djarmode=tools -jar my-app.jar extract --layers --launcher

# Stage 4: Final runtime image
FROM bellsoft/liberica-runtime-container:jre-25-cds-slim-musl AS run
LABEL maintainer="Team Name <team@entur.org>"
EXPOSE 8086
WORKDIR /app

# Copy layers in order of change frequency (least to most)
COPY --from=layers /app/my-app/dependencies/ ./
COPY --from=layers /app/my-app/internal-dependencies/ ./
COPY --from=layers /app/my-app/spring-boot-loader/ ./
COPY --from=layers /app/my-app/application/ .

ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "org.springframework.boot.loader.launch.JarLauncher"]
```

Key practices in this approach:

- **Dependency caching**: Copy build files first, download dependencies, then copy source -- changes to source code don't invalidate the dependency cache layer
- **Build secrets**: Use `--mount=type=secret` for Artifactory credentials instead of `ARG`/`ENV` (secrets don't persist in image layers)
- **Layered JAR**: Spring Boot's layered JAR splits the application into dependency layers, so only the changed layer is rebuilt when pushing new images
- **CDS (Class Data Sharing)**: The Liberica CDS image pre-computes class metadata for faster startup
- **`-XX:MaxRAMPercentage=75.0`**: Let the JVM use 75% of container memory, leaving room for the OS and native memory

### Java / Kotlin (Simple)

For simpler projects without multi-stage builds:

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

### Go

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

### Python

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

FROM python:3.12-slim
WORKDIR /app
COPY --from=builder /install /usr/local
COPY . .

# Non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

EXPOSE 8080
ENTRYPOINT ["python", "-m", "my_service"]
```

## Best Practices

### Use Multi-Stage Builds

Separate the build stage from the runtime stage to exclude build tools, source code, and intermediate artifacts from the final image. This reduces image size and attack surface.

> **Note:** Multi-stage builds in GitHub Actions do not support GitHub caching for the application build step and do not support GitHub secrets injection. If you need these, split your build into separate workflow steps instead of Docker stages.

### Run as Non-Root

All containers must run as a non-root user:

- **Java/Kotlin**: Create a user with `addgroup`/`adduser` and switch with `USER`
- **Go**: Use the `nonroot` variant of distroless (runs as UID 65532 by default)
- **Python**: Create a user with `groupadd`/`useradd` and switch with `USER`

The Entur common Helm chart enforces `runAsNonRoot: true` in the pod security context.

### Minimize Image Size

- Use Alpine or slim base images
- Remove package manager caches (`--no-cache` for apk, `--no-cache-dir` for pip)
- Avoid installing unnecessary packages
- Copy only the artifacts needed for runtime

### Do Not Store Secrets in Images

Never include secrets, credentials, or environment-specific configuration in the Docker image. Use:

- **Google Secret Manager** with ExternalSecrets in Helm for runtime secrets
- **Environment variables** injected via Helm values for non-sensitive configuration

### Pin Dependencies

- Pin base image tags to specific versions (not `latest`)
- Pin build tool versions in the build stage
- Use lock files (`go.sum`, `gradle.lockfile`, `requirements.txt` with pinned versions)

### Expose the Correct Port

All Entur applications listen on port `8080` by default. Ensure the `EXPOSE` directive matches and the application binds to that port.

### Health Check Endpoints

The application inside the container must expose liveness and readiness endpoints. The standard paths are:

- `GET /actuator/health/liveness` -- returns HTTP `200` when the process is alive
- `GET /actuator/health/readiness` -- returns HTTP `200` when the service is ready for traffic

For non-Spring applications, expose equivalent endpoints at these paths or configure custom paths in your Helm values. See [Helm guide](helm.md) for probe configuration and [observability](observability.md) for monitoring details.

## CI/CD Integration

Docker images are built, scanned, and pushed using Entur's [gha-docker](https://github.com/entur/gha-docker) reusable workflows.

### Dockerfile Linting

```yaml
docker-lint:
  uses: entur/gha-docker/.github/workflows/lint.yml@v1
```

Runs Hadolint against the Dockerfile. Fix all warnings before merging.

### Build

```yaml
docker-build:
  uses: entur/gha-docker/.github/workflows/build.yml@v1
```

Builds the image and outputs an `image_artifact` for scanning and pushing.

### Security Scan

```yaml
docker-scan:
  needs: [docker-build]
  uses: entur/gha-security/.github/workflows/docker-scan.yml@v2
  secrets: inherit
  with:
    image_artifact: ${{ needs.docker-build.outputs.image_artifact }}
```

Scans the built image for known vulnerabilities. See [security](security.md) for allowlist configuration.

### Push

```yaml
docker-push:
  needs: [docker-build, docker-scan]
  uses: entur/gha-docker/.github/workflows/push.yml@v1
```

Pushes the image to Google Artifact Registry. The output `image_and_tag` is used by downstream Helm deploy jobs.

### Full CI Pipeline Example

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  docker-lint:
    uses: entur/gha-docker/.github/workflows/lint.yml@v1

  docker-build:
    uses: entur/gha-docker/.github/workflows/build.yml@v1

  docker-scan:
    needs: [docker-build]
    uses: entur/gha-security/.github/workflows/docker-scan.yml@v2
    secrets: inherit
    with:
      image_artifact: ${{ needs.docker-build.outputs.image_artifact }}

  docker-push:
    if: github.event_name != 'pull_request'
    needs: [docker-build, docker-scan]
    uses: entur/gha-docker/.github/workflows/push.yml@v1
```

For complete CI/CD pipeline examples including Helm deployment, see [CI/CD workflows](cicd/workflows.md).

## Hadolint Configuration

To suppress specific Hadolint rules, create a `.hadolint.yaml` file in the repository root:

```yaml
ignored:
  - DL3018  # Pin versions in apk add
```

Only suppress rules with justification. The `gha-docker` lint workflow picks up this file automatically.

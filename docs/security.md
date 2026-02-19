# Security Standards

Security conventions for all Entur services. Follow these patterns for authentication, authorization, secret management, and vulnerability scanning.

Entur uses a three-tier security requirement model based on OWASP ASVS:

- **Level 1**: Mandatory for all services
- **Level 2**: Required for services handling money or personal information
- **Level 3**: Nice-to-have improvements

## Secret Management

### Rules

- **Never hardcode secrets** in source code, configuration files, Dockerfiles, or CI workflows
- **Never commit secrets** to Git -- not even in "test" configurations
- **Use Google Secret Manager** for all secrets (database passwords, API keys, tokens)
- **Reference secrets via ExternalSecrets** in Kubernetes (configured in Helm values)

### Creating Secrets (Terraform)

Secrets are created automatically by Entur Terraform modules (e.g. `terraform-google-sql-db` creates `PG_USER` and `PG_PASSWORD`). For custom secrets:

```hcl
resource "google_secret_manager_secret" "api_key" {
  secret_id = "${module.init.app.id}-API_KEY"
  project   = module.init.app.project_id

  replication {
    auto {}
  }

  labels = module.init.labels
}
```

### Consuming Secrets (Helm)

```yaml
# helm/<app>/env/dev.yaml
common:
  secrets:
    my-secret:               # ExternalSecret name -> creates K8s Secret
      - API_KEY              # Secret Manager secret name -> mounted as env var
      - EXTERNAL_SERVICE_KEY
```

This creates an ExternalSecret that syncs from Google Secret Manager to a Kubernetes Secret, which is then mounted as environment variables in the pod.

### Consuming Secrets (Application Code)

Access secrets as environment variables:

```java
// Java/Kotlin - via Spring property
@Value("${API_KEY}")
private String apiKey;
```

```go
// Go
apiKey := os.Getenv("API_KEY")
```

### Local Development

Access secrets locally via `gcloud` without storing them in files:

```bash
APP_PASSWORD=$(gcloud secrets versions access latest --secret <secret-name> --project <project-id>)
export APP_PASSWORD
```

Do not store secrets locally in files on your development machine.

### Secret Rotation

Rotate secrets at least every 90 days (does not apply to OAuth access/refresh tokens, which should have shorter lifetimes). Secret sync to Kubernetes happens hourly; redeployment is typically required to pick up updated secrets.

## Authentication Patterns

### Service-to-Service (Internal)

- Use GCP service account identity for internal service-to-service authentication
- Use Workload Identity to bind Kubernetes service accounts to GCP service accounts
- The Terraform `init` module provides the necessary service account outputs

### External API Authentication

- Use OAuth 2.0 / OpenID Connect for user-facing APIs
- Validate JWT tokens at the API gateway or in the application
- Never build custom authentication -- use established libraries and protocols
- Use Entur's OIDC auth libraries: `oidc-auth-resource-server` for token validation, `oidc-auth-client` for obtaining tokens

### Authorization

Entur uses a centralized authorization system based on **Permission Store** and **Permission Client**. See [authorization.md](authorization.md) for complete details.

Key concepts:

- **Business Capabilities**: operation + access level (LES, OPPRETT, ENDRE, SLETT) for endpoint-level access
- **Responsibility Sets**: operation + responsibility type + object key for data-level access
- **Agreements**: link responsibility sets to organisations
- Use `@PreAuthorize("hasPermission('operation', 'access')")` in Spring controllers
- Use `LOCAL_TEST_CACHE` with test users for testing
- Use `IN_MEMORY` cache with WebSocket push notifications in production

## HTTP Security Headers

All HTTP responses must include these headers:

- **`Content-Type`** with a safe charset (UTF-8 or ISO-8859-1) for text content types
- **`X-Content-Type-Options: nosniff`** to prevent MIME-type sniffing
- **`Strict-Transport-Security: max-age=15724800; includeSubdomains`** on all responses

For web applications (not REST APIs), also set:

- **`Content-Security-Policy`** to mitigate XSS attacks
- **`X-Frame-Options`** or `Content-Security-Policy: frame-ancestors` to prevent clickjacking

## Input Validation

- Validate all input at API boundaries using allow lists (positive validation) when possible
- Use **Data Transfer Objects (DTOs)** to prevent mass parameter assignment -- never bind input directly to domain objects
- Only accept HTTP methods actually used by the application; log requests with unexpected methods
- Do not use the `Origin` header for authentication or access control decisions
- Validate and sanitize URL redirects using an allow list
- For structured data, enforce a strongly typed schema including allowed characters, length, and pattern

## Dependency Scanning

### Dependabot

Dependabot is automatically enabled for all Entur repositories. Configuration is in `.github/dependabot.yml`. Use Dependabot instead of Renovate.

Recommended ecosystems to configure: `docker`, `helm`, `github-actions`, `terraform`, `npm`, `gradle`, `gomod`, `pip`.

Vulnerabilities found by Dependabot must be triaged, evaluated, and fixed within **30 days** of discovery. Manage findings in the repository's **Security** tab under Dependabot.

### Code Scanning (CodeQL)

Every repository must have a `.github/workflows/codeql.yml` file:

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'    # Weekly on Monday

jobs:
  code-scan:
    uses: entur/gha-security/.github/workflows/code-scan.yml@v2
    secrets: inherit
```

The workflow name **must** be `codeql.yml`. It triggers on PRs, pushes to main, and weekly.

### Docker Image Scanning

Add to your CI pipeline after building the Docker image:

```yaml
jobs:
  docker-build:
    uses: entur/gha-docker/.github/workflows/build.yml@v1
  docker-scan:
    needs: [docker-build]
    uses: entur/gha-security/.github/workflows/docker-scan.yml@v2
    secrets: inherit
    with:
      image_artifact: ${{ needs.docker-build.outputs.image_artifact }}
```

### Allowlisting False Positives

#### Code Scan Allowlist (`.entur/security/codescan.yml`)

```yaml
apiVersion: entur.io/securitytools/v1
kind: CodeScanConfig
metadata:
  id: my-application
spec:
  allowlist:
    - cwe: "cwe-080"
      comment: "False positive: XSS not possible in this context"
      reason: "false_positive"        # false_positive | wont_fix | test
  notifications:
    severityThreshold: "high"         # low | medium | high | critical
    outputs:
      slack:
        enabled: true
        channelId: "C01ABCDEFGH"
      pullRequest:
        enabled: true
```

#### Docker Scan Allowlist (`.entur/security/dockerscan.yml`)

```yaml
apiVersion: entur.io/securitytools/v1
kind: DockerScanConfig
metadata:
  id: my-application
spec:
  allowlist:
    - cve: "CVE-2024-1234"
      comment: "Not exploitable in our configuration"
      reason: "false_positive"
  notifications:
    severityThreshold: "high"
    outputs:
      slack:
        enabled: true
        channelId: "C01ABCDEFGH"
      pullRequest:
        enabled: false
```

## Container Security

### Dockerfile Best Practices

See [Docker guide](docker.md) for complete Dockerfile conventions and examples.

- Use minimal base images (`eclipse-temurin:21-jre-alpine` for Java, `distroless` for Go)
- Run as non-root user
- Don't install unnecessary packages
- Don't store secrets in the image
- Use multi-stage builds to exclude build tools from the runtime image
- Pin base image versions (not `latest`)

```dockerfile
# Java example
FROM eclipse-temurin:21-jre-alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
WORKDIR /app
COPY build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Kubernetes Security

- Pods run as non-root by default (enforced by common Helm chart)
- Use network policies to restrict pod-to-pod communication where appropriate
- Use Workload Identity instead of mounting service account keys
- Never mount the default Kubernetes service account token unless required

## IAM and Permissions

Only use IAM roles from the [approved list](terraform/iam-roles.md). If you need a role that is not listed, request it in `#talk-utviklerplattform` on Slack.

Common roles for applications:

| Role | Use case |
|------|----------|
| `roles/secretmanager.secretAccessor` | Read secrets from Secret Manager |
| `roles/cloudsql.client` | Connect to Cloud SQL via proxy |
| `roles/pubsub.publisher` | Publish messages to Pub/Sub |
| `roles/pubsub.subscriber` | Subscribe to Pub/Sub topics |
| `roles/storage.objectViewer` | Read from Cloud Storage buckets |
| `roles/bigquery.dataViewer` | Query BigQuery datasets |

## Vulnerability Management

Use the GitHub **Security** tab (enabled by default on all repositories) to review code vulnerabilities, dependency vulnerabilities, and committed secrets. The organization-level security overview is at `https://github.com/orgs/entur/security/overview`.

When dismissing a vulnerability alert, always provide a reason:

- **Fix already started**: Include a link to the PR or Jira ticket
- **False positive**: Explain why it does not apply

Teams managing projects are responsible for handling all security findings in their repositories.

## Access Control

- Enforce access control on the server side (zero trust model: "never trust, always verify")
- Users and data attributes used by access controls must not be manipulable by end users
- Apply the principle of least privilege -- users and services only access what they specifically need
- All accounts must be single-purpose (one user or one application per account)
- Access controls must fail securely, including when exceptions occur
- Use attribute/feature-based authorization checks in code, not role-based checks
- Each team is responsible for controlling access to its own data (not the platform team)
- Grant IAM permissions at the **group level** rather than individual users when possible
- Manage data access through Terraform in a CD pipeline for auditability and version control

## Security Checklist

Before submitting a PR, verify:

- [ ] No secrets or credentials in source code or configuration files
- [ ] Secrets are stored in Google Secret Manager, referenced via ExternalSecrets
- [ ] Dockerfile runs as non-root user
- [ ] Base images are pinned to specific versions
- [ ] Dependencies are scanned (CodeQL + Docker scan in CI)
- [ ] IAM roles are from the approved list only
- [ ] Input validation is in place at API boundaries
- [ ] Error responses don't leak internal details (stack traces, database errors)
- [ ] Authentication and authorization are properly configured
- [ ] CORS is configured restrictively (not `*` in production)

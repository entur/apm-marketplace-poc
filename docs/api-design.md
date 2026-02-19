# API Design Standards

Guidelines for designing REST and gRPC APIs at Entur.

## REST API Conventions

### URL Structure

```text
https://<service>.entur.org/api/v{version}/{resource}
```

- Use kebab-case for URL path segments: `/stop-places`, `/journey-plans`
- Use plural nouns for resource collections: `/routes`, `/stops`
- Use path parameters for resource identity: `/routes/{id}`
- Use query parameters for filtering, sorting, and pagination: `/routes?origin=Oslo&limit=20`
- Version the API in the URL path: `/api/v1/`, `/api/v2/`

### HTTP Methods

| Method | Purpose | Idempotent | Response |
|--------|---------|------------|----------|
| `GET` | Retrieve a resource or collection | Yes | `200 OK` |
| `POST` | Create a new resource | No | `201 Created` with `Location` header |
| `PUT` | Replace a resource entirely | Yes | `200 OK` or `204 No Content` |
| `PATCH` | Partially update a resource | No | `200 OK` |
| `DELETE` | Remove a resource | Yes | `204 No Content` |

### Status Codes

#### Success

| Code | Meaning | Use for |
|------|---------|---------|
| `200` | OK | Successful GET, PUT, PATCH |
| `201` | Created | Successful POST (include `Location` header) |
| `204` | No Content | Successful DELETE or PUT with no response body |

#### Client Errors

| Code | Meaning | Use for |
|------|---------|---------|
| `400` | Bad Request | Malformed request, validation error |
| `401` | Unauthorized | Missing or invalid authentication |
| `403` | Forbidden | Authenticated but lacks permission |
| `404` | Not Found | Resource does not exist |
| `409` | Conflict | Conflict with current state (duplicate, version mismatch) |
| `422` | Unprocessable Entity | Semantically invalid request |
| `429` | Too Many Requests | Rate limit exceeded |

#### Server Errors

| Code | Meaning | Use for |
|------|---------|---------|
| `500` | Internal Server Error | Unexpected server failure |
| `502` | Bad Gateway | Upstream service failure |
| `503` | Service Unavailable | Service temporarily unavailable |
| `504` | Gateway Timeout | Upstream service timeout |

### Error Response Format

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Origin stop place is required",
    "details": [
      {
        "field": "origin",
        "message": "must not be blank"
      }
    ]
  }
}
```

- Always return a structured error body for 4xx and 5xx responses
- Include a machine-readable `code` and a human-readable `message`
- Never expose stack traces, SQL queries, or internal details in error responses

### Pagination

Use cursor-based pagination for large collections:

```json
GET /api/v1/routes?limit=20&cursor=eyJpZCI6MTAwfQ

{
  "data": [...],
  "pagination": {
    "limit": 20,
    "cursor": "eyJpZCI6MTIwfQ",
    "hasMore": true
  }
}
```

Alternatively, offset-based pagination is acceptable for simpler use cases:

```json
GET /api/v1/routes?page=2&size=20

{
  "data": [...],
  "page": 2,
  "size": 20,
  "totalElements": 150,
  "totalPages": 8
}
```

### Request and Response Bodies

- Use `camelCase` for JSON field names
- Use ISO 8601 for dates and times: `"2025-01-15T10:30:00Z"`
- Use ISO 4217 for currency codes: `"NOK"`
- Wrap collections in a named field (not raw arrays): `{"data": [...]}`
- Include only necessary fields -- don't return the entire database row

### Content Negotiation

- Default to `application/json`
- Set `Content-Type: application/json` on all JSON responses
- Accept `Accept` header for content negotiation when supporting multiple formats

## API Documentation

### OpenAPI / Swagger

All REST APIs must have an OpenAPI specification. There are two approaches:

#### Contract-First (Preferred for Kotlin)

Define the API specification before writing code. Use **OpenAPI Generator** to generate interfaces and DTOs:

1. Write modular OpenAPI specs in `specs/`
2. Bundle with **Redocly CLI** into a single spec
3. Generate Kotlin Spring interfaces and DTOs at build time
4. Implement the generated interfaces in your controllers

This approach ensures the API contract is always in sync with implementation and enables parallel frontend/backend development.

See the [Contract-First OpenAPI](#contract-first-openapi) section below for full details.

#### Code-First (Simpler Projects)

Annotate controllers and let springdoc-openapi generate the specification:

```java
// Spring Boot - springdoc-openapi
@Operation(summary = "Find route by ID")
@ApiResponse(responseCode = "200", description = "Route found")
@ApiResponse(responseCode = "404", description = "Route not found")
@GetMapping("/{id}")
public ResponseEntity<RouteResponse> getRoute(@PathVariable String id) { ... }
```

Serve the OpenAPI spec at `/api-docs` or `/v3/api-docs` (Spring Boot default).

### Entur Springdoc Starter

For code-first APIs, use the **`entur-springdoc-starter`** (`org.entur.openapi:entur-springdoc-starter`) alongside springdoc-openapi. This starter extends springdoc with Entur-specific OpenAPI extensions that are required by the Entur developer portal and API gateway.

#### Setup

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-api:<version>")
    implementation("org.entur.openapi:entur-springdoc-starter:<version>")  // check Artifactory for latest
}
```

The starter is published to Entur's JFrog Artifactory. Check [Artifactory](https://entur2.jfrog.io) for the latest version. See [java.md](java.md#artifactory-jfrog) for repository configuration.

Configure springdoc defaults:

```yaml
# application.yml
springdoc:
  default-produces-media-type: application/json
  default-consumes-media-type: application/json
```

#### `x-entur-metadata` Extension

Every API must declare `x-entur-metadata` on its `info` object. This identifies the API in Entur's developer portal. Use the `EnturMetadata` class provided by the starter.

Kotlin:

```kotlin
@Configuration
class OpenApiConfig {
    @Bean
    fun openApi(): OpenAPI {
        return OpenAPI()
            .info(Info()
                .title("Items API")
                .version("1.0.0")
                .description("Manage items")
                .enturMetadata(EnturMetadata()
                    .id("items")                    // unique API identifier in Entur
                )
            )
    }
}
```

Java:

```java
import static org.entur.openapi.EnturMetadataKt.enturMetadata;

@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI openApi() {
        var info = new Info()
                .title("Items API")
                .version("1.0.0")
                .description("Manage items");

        enturMetadata(info, new EnturMetadata().id("items"));

        return new OpenAPI().info(info);
    }
}
```

The `parentId` field is available for APIs that are developed independently across microservices but should appear as a single specification to consumers:

```kotlin
EnturMetadata()
    .id("items-subset")
    .parentId("items")    // content shown under the parent API, not standalone
```

#### `x-entur-permissions` Extension (Automatic)

The starter **automatically generates** `x-entur-permissions` on every operation based on the `@PreAuthorize` annotation. No code changes are needed -- if your controller already uses `@PreAuthorize("hasPermission('resource', 'les')")`, the extension is added to the OpenAPI spec at runtime.

The parser understands:

- `hasPermission('resource', 'access')` -- single permission
- `hasPermission('a', 'les') AND hasPermission('b', 'les')` -- all required (`all`)
- `hasPermission('a', 'les') OR hasPermission('b', 'les')` -- any sufficient (`any`)
- Nested combinations of `AND`/`OR`
- Non-permission nodes like `hasAnyAuthority('partner')` are skipped

Example controller:

```kotlin
@GetMapping("/items")
@Operation(summary = "Get items")
@PreAuthorize("hasPermission('items', 'les') || hasPermission('items-global', 'les')")
fun getItems(): List<Item> { ... }
```

This produces the following in the OpenAPI spec:

```json
"x-entur-permissions": {
  "value": {
    "any": ["items:les", "items-global:les"]
  }
}
```

#### `@EnturPermissions` Override

If the auto-generated permissions from `@PreAuthorize` are insufficient or you need to override them, use the `@EnturPermissions` annotation:

```kotlin
@GetMapping("/items")
@PreAuthorize("hasPermission('items', 'les') || hasPermission('items-global', 'les')")
@EnturPermissions(
    description = "Requires read access to items or global items",
    value = EnturPermissionsValue(any = [
        EnturPermissionsValue("items:les"),
        EnturPermissionsValue("something-else:les"),
    ])
)
fun getItems(): List<Item> { ... }
```

If only `description` is set (without `value`), the `@PreAuthorize` annotation is still used for the permission value.

#### `@SchemaExample` Annotation

Adding examples to OpenAPI schemas using standard Swagger annotations can be verbose. The starter provides `@SchemaExample` to generate examples from actual class instances, ensuring examples compile and stay in sync with the schema.

Kotlin:

```kotlin
data class Item(
    val id: String,
    val description: String
) {
    companion object {
        @SchemaExample
        @JvmStatic
        fun example(): Item = Item("foo", "Foo")
    }
}
```

Java:

```java
public record Item(String id, String description) {
    @SchemaExample
    public static Item example() {
        return new Item("foo", "Foo");
    }
}
```

The example method must be:

- `static` (`@JvmStatic` in Kotlin companion objects)
- Annotated with `@SchemaExample`
- Return an instance of the enclosing class

The example is serialized using the application's `ObjectMapper` and added to the schema in the OpenAPI spec.

#### Custom TypeNameResolver

The starter supports a custom `TypeNameResolver` bean for controlling how generic types appear in the OpenAPI schema. This is useful for cleaning up names of generic wrapper types:

```kotlin
@Bean
fun customTypeNameResolver(): TypeNameResolver {
    return object : TypeNameResolver() {
        override fun nameForGenericType(
            type: JavaType,
            options: Set<Options?>?
        ): String? {
            val name = super.nameForGenericType(type, options)
            // "PageDtoItem" becomes "PageItem"
            return if (type.rawClass?.isAssignableFrom(PageDto::class.java) == true) {
                name.replaceFirst("PageDto", "Page")
            } else {
                name
            }
        }
    }
}
```

## gRPC APIs

For gRPC services:

- Define services and messages in `.proto` files
- Use `PascalCase` for message and service names, `snake_case` for field names
- Version proto packages: `entur.myservice.v1`
- Enable gRPC in the Helm chart:

```yaml
common:
  grpc: true
  ingress:
    trafficType: http2
```

- Use gRPC health checking protocol (`grpc.health.v1.Health`)
- The common Helm chart automatically configures gRPC probes when `grpc: true`

## Exposing APIs

### Domain Patterns

| Application Type | Production | Test | Development |
|------------------|-----------|------|-------------|
| Frontend / Web | `.entur.no` | `.staging.entur.no` | `.dev.entur.no` |
| API | `.entur.io` | `.staging.entur.io` | `.dev.entur.io` |

### Apigee API Gateway

External APIs are exposed through **Apigee** at `api.entur.io`. Set `ingress.trafficType: api` in Helm to route through Apigee (the application is not directly accessible from the internet).

| Environment | URL pattern |
|-------------|-------------|
| Production | `https://api.entur.io/<api-path>/<version>` |
| Test | `https://api.staging.entur.io/<api-path>/<version>` |
| Development | `https://api.dev.entur.io/<api-path>/<version>` |

**Limitation**: 10 MB maximum per single request through Apigee. Use streaming for larger payloads.

For Apigee proxy configuration, see the "API Gateway Apigee X" documentation or request help in `#talk-utviklerplattform`.

### Internal Service URLs

For cluster-internal communication (not exposed to the internet):

- Same namespace: `http://<service-name>`
- Cross-namespace (same environment): `http://<service-name>.<namespace>.entur.internal`
- Only TCP ports 80 and 443 are available between services

Set `ingress.enabled: false` in Helm for internal-only services.

### gRPC External Limitations

gRPC is supported for cluster-internal and frontend-external traffic. **Apigee does not yet support gRPC API proxies** -- external gRPC APIs require separate platform configuration. Contact `#talk-utviklerplattform` for gRPC external exposure.

## API Versioning

- Use URL path versioning: `/api/v1/`, `/api/v2/`
- Increment the major version only for breaking changes
- Support the previous version for a deprecation period (minimum 6 months)
- Breaking changes include: removing fields, changing field types, changing URL structure, changing error codes

## Contract-First OpenAPI

The preferred approach for Kotlin Spring Boot APIs is **contract-first development** using OpenAPI Generator.

### Directory Structure

```text
specs/
  products.yaml                    # Main entry point
  openapi.json                     # Bundled output (gitignored)
  schemas/
    _index.yaml                   # Schema index
    Version.yaml                  # Individual schemas
    enums/
      VersionStatus.yaml          # Enum definitions
  parameters/
    _index.yaml                   # Parameter index
    header/
      et-client-name.yaml
      x-correlation-id.yaml
    path/
      id.yaml
  paths/
    VersionV3.yaml                # Path definitions
```

### Main Spec File

```yaml
openapi: "3.0.3"
info:
  version: "1.0.0"
  title: My API
  contact:
    name: Team Name
    email: team@entur.org
servers:
  - url: "https://api.entur.io/my-api"
    description: "Production environment"
  - url: "https://api.staging.entur.io/my-api"
    description: "Staging environment"
  - url: "https://api.dev.entur.io/my-api"
    description: "Development environment"
tags:
  - name: version
    description: Version management endpoints
paths:
  /v3/versions/{id}:
    $ref: "./paths/VersionV3.yaml#/~1v3~1versions~1{id}"
```

### Schema with Read-Only Fields

Use `readOnly: true` for server-generated fields like `created` and `changed`:

```yaml
type: object
required:
  - id
  - status
  - startDate
properties:
  id:
    pattern: "^([A-Z]{3}):Version:([0-9A-Za-z_\\-]*)$"
    type: string
  status:
    $ref: "./enums/VersionStatus.yaml"
  startDate:
    type: string
    format: date
  endDate:
    type: string
    format: date
allOf:
  - type: object
    properties:
      created:
        type: string
        format: date-time
        readOnly: true
      changed:
        type: string
        format: date-time
        readOnly: true
```

### Bundling and Code Generation (Gradle)

```kotlin
// build.gradle.kts
openApiGenerate {
    validateSpec = true
    inputSpec = "specs/openapi.json"
    outputDir = "$generatedSourcesDir/openapi"
    generatorName = "kotlin-spring"
    apiPackage = "org.entur.myapp.api"
    modelPackage = "org.entur.myapp.dto"
    configOptions = mapOf(
        "interfaceOnly" to "true",         // Generate interfaces, not implementations
        "useBeanValidation" to "true",     // Add Jakarta validation annotations
        "useSpringBoot3" to "true",        // Spring Boot 3 / Jakarta EE
        "exceptionHandler" to "false",     // Use custom exception handler
        "useTags" to "true",               // Group by tags
    )
    typeMappings = mapOf(
        "java.time.OffsetDateTime" to "java.time.ZonedDateTime",
        "kotlin.Float" to "java.math.BigDecimal",
    )
}
```

Bundling uses Redocly CLI (via Docker or npm):

```kotlin
// Gradle task using Docker
register("bundleOpenApiSpecification", Exec::class) {
    commandLine("docker", "run", "--rm",
        "-v", "$projectDirPath/specs:/spec",
        "redocly/cli", "bundle", "products.yaml", "-o", "openapi.json")
}

// Build chain
compileKotlin { dependsOn("openApiGenerate") }
openApiGenerate { dependsOn("bundleOpenApiSpecification") }
```

### API Spec Linting in CI

Lint the API specification on PRs that modify `specs/`:

```yaml
name: lint-api
on:
  pull_request:
    paths:
      - 'specs/**'
jobs:
  api-lint:
    uses: entur/gha-api/.github/workflows/lint.yml@v5
    with:
      spec: specs/*.yaml
```

### Publishing to Developer Portal

After deploying to production, publish the bundled spec to Entur's developer portal:

```yaml
openapi-publish:
  uses: entur/gha-api/.github/workflows/publish.yml@v5
  with:
    artifact: openapi-spec
    visibility: partner       # partner | public | internal
```

## Rate Limiting and Resilience

- Implement retry logic with exponential backoff for outgoing HTTP calls
- Set reasonable timeouts on all HTTP clients (connect: 5s, read: 30s)
- Use circuit breakers for calls to external services
- Return `429 Too Many Requests` with `Retry-After` header when rate limiting

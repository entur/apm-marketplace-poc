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

All REST APIs must have an OpenAPI specification:

```java
// Spring Boot - springdoc-openapi
@Operation(summary = "Find route by ID")
@ApiResponse(responseCode = "200", description = "Route found")
@ApiResponse(responseCode = "404", description = "Route not found")
@GetMapping("/{id}")
public ResponseEntity<RouteResponse> getRoute(@PathVariable String id) { ... }
```

Serve the OpenAPI spec at `/api-docs` or `/v3/api-docs` (Spring Boot default).

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

## Rate Limiting and Resilience

- Implement retry logic with exponential backoff for outgoing HTTP calls
- Set reasonable timeouts on all HTTP clients (connect: 5s, read: 30s)
- Use circuit breakers for calls to external services
- Return `429 Too Many Requests` with `Retry-After` header when rate limiting

# Authorization with Permission Store

Entur uses a centralized authorization system consisting of **Permission Store** (backend service) and **Permission Client** (Java SDK). This document covers how to integrate authorization into your application, grant access between apps/clients/partners, and configure test users.

## Overview

| Component | Repository | Purpose |
|-----------|-----------|---------|
| **Permission Store** | `permission-store` | Central backend that stores and serves authorization data (permissions, agreements, responsibility sets) |
| **Permission Client** (Java) | `permission-client` | Spring Boot SDK for consuming permissions, caching, and Spring Security integration |
| **Permission Client** (Node) | `permission-client-node` | Node.js SDK for consuming permissions |

### How It Works

1. **Applications register** with Permission Store and declare which permissions (business capabilities) they need
2. **Users authenticate** via OIDC/JWT and receive tokens with tenant information (authority + subject)
3. **Permission Client** caches permissions from Permission Store and evaluates access locally
4. **Controllers** use `@PreAuthorize` annotations to enforce access control at the endpoint level

```text
User (JWT token)
       ↓
  Application
  (Permission Client SDK)
       ↓ polls/websocket
  Permission Store
  (central permission database)
```

## Permission Types

### Business Capabilities

Business capabilities are the primary authorization model. They define **what operations** a user can perform on a service.

A business capability is a pair of: **operation** + **access level**

| Access Level | Norwegian | Description |
|-------------|-----------|-------------|
| `LES` | Les | Read access |
| `OPPRETT` | Opprett | Create access |
| `ENDRE` | Endre | Update/modify access |
| `SLETT` | Slett | Delete access |

Example: A user with business capability `product-api-access` + `LES,ENDRE` can read and modify products but not create or delete them.

### Responsibility Sets

Responsibility sets provide **data-level access control** -- who can access which specific data objects. They combine:

- **Operation** (same as business capability)
- **Responsibility Type** (e.g., `product.organisation`)
- **Responsibility Key** (e.g., a specific organisation ID)
- **Access Level** (LES, OPPRETT, ENDRE, SLETT)

Responsibility sets are linked to users via **Agreements** (see below).

Use responsibility sets when you need to restrict access to specific data partitions (e.g., "user X can edit products belonging to organisation Y").

## Adding Authorization to Your Application

### Prerequisites

Your application must have Auth0 internal M2M credentials provisioned to authenticate with Permission Store. In your self-service application manifest, ensure:

```yaml
spec:
  auth0:
    internal:
      enabled: true
      type: m2m
```

See [self-service.md](self-service.md) for details on application manifests.

### 1. Add Dependencies

```kotlin
// build.gradle.kts
dependencies {
    implementation(libs.bundles.entur.auth)
    // This bundle typically includes:
    // - org.entur.auth.client:oidc-client-spring-boot
    // - org.entur.auth.resource-server:oidc-rs-spring-boot-web-config
    // - org.entur.auth:permission-client

    testImplementation(libs.entur.auth.oidc)
    // - org.entur.auth.resource-server:oidc-rs-spring-boot-web-test
}
```

Or explicitly:

```groovy
// build.gradle
implementation 'org.entur.auth:permission-client:3.x.x'
implementation 'org.entur.auth.resource-server:oidc-rs-spring-boot-web-config:2.x.x'
implementation 'org.entur.auth.client:oidc-client-spring-boot:2.x.x'
```

### 2. Configure application.yml

```yaml
entur:
  auth:
    tenants:
      environment: dev              # dev | tst | prd
      include: internal, partner    # Which tenants to accept tokens from

  # OIDC client credentials for machine-to-machine calls to Permission Store
  clients:
    auth0:
      permission-store:
        clientId: ${sm@MNG_AUTH0_INT_CLIENT_ID}
        secret: ${sm@MNG_AUTH0_INT_CLIENT_SECRET}
        domain: internal.dev.entur.org
        audience: https://api.dev.entur.io

  permission:
    scheduler: ws                    # ws (websocket) or poll
    bean: client                     # client | oidc | jwt
    permission-cache:
      type: IN_MEMORY                # IN_MEMORY | LOCAL_TEST_CACHE | FULL_ACCESS
      url: https://api.dev.entur.io/permission-store/v1
      application-name: my-application
      refresh-rate: 300              # Seconds between full cache refreshes
      auth-qualifier: "permission-store"
    businessCapabilities:
      - my-operation, LES, ENDRE     # Compact format: operation, access1, access2
```

### 3. Protect Endpoints

Use `@PreAuthorize` with `hasPermission()`:

```kotlin
@RestController
class VersionController(
    private val versionService: VersionService,
) : VersionApi {

    // Business capability check: operation + access
    @PreAuthorize("hasPermission('product-api-access', 'endre')")
    override fun createVersion(request: CreateVersionRequest): ResponseEntity<Version> {
        // Only users with 'product-api-access' + 'endre' access can reach this
    }

    @PreAuthorize("hasPermission('product-api-access', 'les')")
    override fun getVersionById(id: String): ResponseEntity<Version> {
        // Read-only access
    }
}
```

For responsibility set checks:

```java
// Check if user has access to a specific data object
@PreAuthorize("hasPermission(#organisationId, 'product.organisation', 'endre')")
public ResponseEntity<?> updateProduct(@PathVariable String organisationId, ...) {
    // Only users with responsibility set access for this organisation
}
```

### 4. Programmatic Access Checks

Inject `AuthorizeTenant` for programmatic permission checks:

```java
@Service
public class MyService {
    private final AuthorizeTenant authorizeTenant;

    public MyService(AuthorizeTenant authorizeTenant) {
        this.authorizeTenant = authorizeTenant;
    }

    public void doSomething(Authentication auth) {
        // Check business capability
        boolean canEdit = authorizeTenant.checkBusinessCapabilityPermission(
            auth.getPrincipal(), "my-operation", "endre"
        );

        // Get all permissions for a user
        Set<Permission> permissions = authorizeTenant.getPermissions(auth.getPrincipal());

        // Check responsibility set
        boolean canAccessOrg = authorizeTenant.checkResponsibilitySetPermission(
            auth.getPrincipal(), "my-operation", new ObjectId("org-123"), "les"
        );
    }
}
```

## Granting Access Between Apps, Clients, and Partners

### Defining Business Capabilities

In your application's `application.yml`, declare which business capabilities your application uses:

```yaml
entur:
  permission:
    businessCapabilities:
      # Compact format (preferred):
      - my-operation, LES, OPPRETT, ENDRE, SLETT

      # Verbose format:
      - operation: my-admin-operation
        access: LES, ENDRE
        description: "Administrative access to my-app"
```

These are registered with Permission Store when your application starts.

### Defining Responsibility Types

For data-level access control, define responsibility types:

```yaml
entur:
  permission:
    responsibilityTypes:
      # Compact format (preferred):
      - product.organisation, LES, ENDRE

      # Verbose format:
      - name: product.id
        access: LES, OPPRETT, ENDRE, SLETT
```

### Setting User Permissions in Permission Store

Permissions are managed through Permission Store's API. When your application registers and posts its permissions, Permission Store tracks which users (identified by authority + subject) have which access levels.

The Permission Store API endpoints:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/applications` | Register application |
| `POST` | `/applications/{id}/permissions` | Set permissions for all tenants |
| `GET` | `/applications/{id}/permissions` | Get current permissions |
| `GET` | `/applications/{id}/changes` | Get permission changes (delta) |

### Managing Agreements (Responsibility Sets)

Agreements link responsibility sets to organisations, enabling data-level access:

```java
// Programmatic agreement management via AuthorizeTenant
authorizeTenant.storeAgreement(
    Agreement.builder()
        .operation("product")
        .access(Access.ENDRE)
        .responsibilityType("organisation")
        .responsibilityKey("ENT:Organisation:123")
        .organisationId("org-456")
        .build()
);

// Query agreements
Set<Agreement> agreements = authorizeTenant.getAgreements(permission);

// Delete agreement
authorizeTenant.deleteAgreement(agreementId);
```

Agreement management is also available via the Permission Store REST API:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/agreements` | List agreements (with filters) |
| `POST` | `/agreements` | Create agreement(s) |
| `DELETE` | `/agreements` | Delete agreement |

## Cache Types

### IN_MEMORY (Production)

The default production cache. Polls or receives WebSocket push notifications from Permission Store to keep permissions in sync:

```yaml
entur:
  permission:
    permission-cache:
      type: IN_MEMORY
      url: https://api.dev.entur.io/permission-store/v1
      refresh-rate: 300            # Full refresh every 5 minutes
      delta-refresh-rate: 60       # Delta refresh every 1 minute (optional)
```

For push notifications instead of polling:

```yaml
entur:
  permission:
    scheduler: ws                  # Use WebSocket push
```

### LOCAL_TEST_CACHE (Testing)

For integration tests, define test users with specific permissions directly in configuration:

```yaml
entur:
  permission:
    permission-cache:
      type: LOCAL_TEST_CACHE
    test-users:
      - authority: internal
        subject: fullAccess
        business-capability-permissions:
          - operation: product-api-access
            access: les, endre
      - authority: internal
        subject: readAccess
        business-capability-permissions:
          - operation: product-api-access
            access: les
      - authority: partner
        subject: orgAdmin
        organisationId: "1"
        responsibility-permissions:
          - operation: product
            responsibilityType: organisation
            access: les, endre
        user-information:
          email: test@example.com
          givenName: Test
          familyName: User
```

### FULL_ACCESS (Development)

Grants all permissions to all users. Use only for local development:

```yaml
entur:
  permission:
    permission-cache:
      type: FULL_ACCESS
```

## Test Configuration

### Controller Tests

Use Entur's test authentication library with `TenantJsonWebToken`:

```kotlin
@WebMvcTest(VersionController::class)
@Import(VersionMapper::class)
@ExtendWith(TenantJsonWebToken::class)
class VersionControllerTests : BaseControllerTest() {

    @MockkBean
    lateinit var versionService: VersionService

    @Test
    fun `internal user with full access can read`(
        @InternalTenant(clientId = "fullAccess") token: String,
    ) {
        every { versionService.find("ENT:Version:1") } returns someVersion()

        mockMvc.perform(
            get("/v3/versions/ENT:Version:1")
                .header("Authorization", token)
        ).andExpect(status().isOk)
    }

    @Test
    fun `traveller tenant is unauthorized`(
        @TravellerTenant(clientId = "user") token: String,
    ) {
        mockMvc.perform(
            get("/v3/versions/ENT:Version:1")
                .header("Authorization", token)
        ).andExpect(status().isUnauthorized)
    }

    @Test
    fun `read-only user cannot create`(
        @InternalTenant(clientId = "readAccess") token: String,
    ) {
        mockMvc.perform(
            post("/v3/versions")
                .header("Authorization", token)
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"id": "ENT:Version:001", "status": "DRAFT"}""")
        ).andExpect(status().isForbidden)
    }
}
```

### Integration Tests

For integration tests with `LOCAL_TEST_CACHE`, define test users in `src/test/resources/application.yml`:

```yaml
entur:
  auth:
    tenants:
      include: partner, internal
      environment: mock              # Use mock OIDC providers
    lazy-load: true
  permission:
    permission-cache:
      type: LOCAL_TEST_CACHE
    test-users:
      - authority: internal
        subject: fullAccess
        business-capability-permissions:
          - operation: product-api-access
            access: les, endre
      - authority: internal
        subject: readAccess
        business-capability-permissions:
          - operation: product-api-access
            access: les
```

## Access Aliases

Map Norwegian access names to custom names if your domain uses different terminology:

```yaml
entur:
  permission:
    permission-cache:
      access-aliases:
        opprett: create
        les: read
        endre: change, admin       # Multiple aliases supported
        slett: delete
```

## Internal Has Full Access

Grant all permissions to internal (Entur employee) tenants automatically:

```yaml
entur:
  permission:
    permission-cache:
      internalHasFullAccess: true   # Default: false
```

When enabled, any token from the `internal` tenant authority is granted full access to all business capabilities. Useful during development or for internal-only services.

## Environment-Specific Configuration

### Permission Store URLs

| Environment | Permission Store URL |
|-------------|---------------------|
| dev | `http://permission-store.dev.entur.internal` or `https://api.dev.entur.io/permission-store/v1` |
| tst | `http://permission-store.tst.entur.internal` or `https://api.staging.entur.io/permission-store/v1` |
| prd | `http://permission-store.prd.entur.internal` or `https://api.entur.io/permission-store/v1` |

Use the internal URL (`.entur.internal`) for cluster-internal communication. Set via Helm configmap:

```yaml
# helm/my-app/env/values-kub-ent-dev.yaml
common:
  configmap:
    data:
      ENTUR_PERMISSION_PERMISSIONCACHE_URL: "http://permission-store.dev.entur.internal"
```

### Auth0 Domains

| Environment | Domain | Audience |
|-------------|--------|----------|
| dev | `internal.dev.entur.org` | `https://api.dev.entur.io` |
| tst | `internal.staging.entur.org` | `https://api.staging.entur.io` |
| prd | `internal.entur.org` | `https://api.entur.io` |

Set via Helm configmap:

```yaml
# helm/my-app/env/values-kub-ent-tst.yaml
common:
  configmap:
    data:
      ENTUR_AUTH_TENANTS_ENVIRONMENT: "tst"
      ENTUR_CLIENTS_AUTH0_PERMISSIONSTORE_AUDIENCE: "https://api.staging.entur.io"
      ENTUR_CLIENTS_AUTH0_PERMISSIONSTORE_DOMAIN: "internal.staging.entur.org"
```

## Authentication Modes

Permission Client supports three authentication modes for obtaining tokens to call Permission Store:

| Mode | Config | When to Use |
|------|--------|-------------|
| `oidc` | `bean: oidc` | Recommended for production with `oidc-auth-client` |
| `client` | `bean: client` | Direct client credentials (Auth0 Machine-to-Machine) |
| `jwt` | `bean: jwt` | When using JWT resource server without OIDC client |

```yaml
entur:
  permission:
    bean: client    # or oidc, jwt
```

## Permission Store Architecture

Permission Store is a Spring Boot application backed by PostgreSQL with:

- **Hibernate Envers** for full audit trail of all permission changes
- **WebSocket (STOMP/SockJS)** for push notifications to connected clients
- **ShedLock** for distributed scheduling
- **LZ4-compressed Kryo serialization** for efficient permission cache
- **Apigee API Gateway** for external access with rate limiting (1200 rpm)

### Domain Model

```text
Application
  └── ApplicationInstance
        ├── BusinessCapability (operation + access)
        │     └── BusinessCapabilityPermission (tenant binding)
        └── Responsibility (operation + responsibilityType)
              └── ResponsibilitySet (+ objectKey)
                    ├── ResponsibilityPermission (tenant binding)
                    └── Agreement (+ organisationId)
```

### Automatic Cleanup

Permission Store automatically cleans up unused resources:

- Applications not refreshed in 30 days
- Permissions not refreshed in 30 days
- Responsibility sets with no agreements for 30 days

This means applications must regularly refresh their registration with Permission Store (handled automatically by Permission Client).

## Quick Reference

### Minimal Setup Checklist

1. Ensure `auth0.internal.enabled: true` and `auth0.internal.type: m2m` in your self-service application manifest
2. Add `permission-client` + `oidc-auth` dependencies
3. Configure `entur.auth.tenants` in `application.yml`
4. Configure `entur.clients.auth0.permission-store` credentials
5. Configure `entur.permission.permission-cache` with URL and application name
6. Declare `businessCapabilities` used by your application
7. Add `@PreAuthorize("hasPermission('operation', 'access')")` to protected endpoints
8. Add `entur-springdoc-starter` to auto-document permissions in the OpenAPI spec (see [api-design.md](api-design.md#x-entur-permissions-extension-automatic))
9. Configure `LOCAL_TEST_CACHE` with test users in test `application.yml`
10. Set environment-specific Permission Store URL and auth config via Helm configmap

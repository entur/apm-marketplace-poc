# Kotlin Standards

Kotlin conventions for Entur applications. Read [CONVENTIONS.md](../CONVENTIONS.md) first for cross-language standards, and [java.md](java.md) for shared JVM patterns (Spring Boot, testing, dependencies).

## Runtime and Build

- **Kotlin version**: latest stable (currently 2.x)
- **Java target**: 21
- **Build tool**: Gradle with Kotlin DSL (`build.gradle.kts`)
- **Framework**: Spring Boot 3.x with Kotlin support
- **Linting**: Ktlint

### build.gradle.kts

```kotlin
plugins {
    kotlin("jvm") version libs.versions.kotlin
    kotlin("plugin.spring") version libs.versions.kotlin
    id("org.springframework.boot") version libs.versions.springBoot
    id("io.spring.dependency-management") version libs.versions.springDependencyManagement
}

kotlin {
    jvmToolchain(21)
    compilerOptions {
        freeCompilerArgs.addAll("-Xjsr305=strict")
    }
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

The `-Xjsr305=strict` flag enables strict null-safety interop with Spring's nullability annotations.

## Kotlin-Specific Patterns

### Data Classes for DTOs

```kotlin
data class RouteResponse(
    val id: String,
    val origin: String,
    val destination: String,
    val departureTime: Instant,
)

data class CreateRouteRequest(
    @field:NotBlank val origin: String,
    @field:NotBlank val destination: String,
    @field:NotNull val departureTime: Instant,
)
```

### REST Controllers

```kotlin
@RestController
@RequestMapping("/api/v1/routes")
class RouteController(
    private val routeService: RouteService,
) {
    @GetMapping("/{id}")
    fun getRoute(@PathVariable id: String): ResponseEntity<RouteResponse> =
        routeService.findById(id)
            ?.let { ResponseEntity.ok(it) }
            ?: ResponseEntity.notFound().build()

    @PostMapping
    fun createRoute(@Valid @RequestBody request: CreateRouteRequest): ResponseEntity<RouteResponse> {
        val created = routeService.create(request)
        val location = URI.create("/api/v1/routes/${created.id}")
        return ResponseEntity.created(location).body(created)
    }
}
```

### Service Layer

```kotlin
@Service
class RouteService(
    private val routeRepository: RouteRepository,
) {
    @Transactional(readOnly = true)
    fun findById(id: String): RouteResponse? =
        routeRepository.findByIdOrNull(id)?.toResponse()

    @Transactional
    fun create(request: CreateRouteRequest): RouteResponse {
        val route = request.toEntity()
        val saved = routeRepository.save(route)
        return saved.toResponse()
    }
}
```

### Key Kotlin Principles

- Use `data class` for DTOs, value objects, and request/response types
- Use primary constructor injection (not `@Autowired`)
- Use Kotlin null-safety instead of `Optional` -- return `T?` not `Optional<T>`
- Use `findByIdOrNull()` (from Spring Data Kotlin extensions) instead of `findById().orElse(null)`
- Use expression-body functions for simple transformations
- Use trailing commas in parameter lists and collections (improves diffs)
- Use `val` over `var` wherever possible
- Use `sealed class` or `sealed interface` for restricted type hierarchies

### Extension Functions for Mapping

```kotlin
// Keep mapping logic in extension functions, close to the domain
fun Route.toResponse() = RouteResponse(
    id = id,
    origin = origin,
    destination = destination,
    departureTime = departureTime,
)

fun CreateRouteRequest.toEntity() = Route(
    origin = origin,
    destination = destination,
    departureTime = departureTime,
)
```

### Coroutines (WebFlux)

If using Spring WebFlux with coroutines:

```kotlin
@RestController
@RequestMapping("/api/v1/routes")
class RouteController(
    private val routeService: RouteService,
) {
    @GetMapping("/{id}")
    suspend fun getRoute(@PathVariable id: String): ResponseEntity<RouteResponse> =
        routeService.findById(id)
            ?.let { ResponseEntity.ok(it) }
            ?: ResponseEntity.notFound().build()
}
```

Only use coroutines if the project already uses WebFlux. Do not mix WebFlux and MVC.

## Testing in Kotlin

```kotlin
@ExtendWith(MockitoExtension::class)
class RouteServiceTest {

    @Mock
    private lateinit var routeRepository: RouteRepository

    @InjectMocks
    private lateinit var routeService: RouteService

    @Test
    fun `findById returns route when it exists`() {
        // Arrange
        val route = aRoute(id = "route-1")
        whenever(routeRepository.findByIdOrNull("route-1")).thenReturn(route)

        // Act
        val result = routeService.findById("route-1")

        // Assert
        assertThat(result).isNotNull
        assertThat(result!!.id).isEqualTo("route-1")
    }

    @Test
    fun `findById returns null when route does not exist`() {
        whenever(routeRepository.findByIdOrNull("unknown")).thenReturn(null)

        val result = routeService.findById("unknown")

        assertThat(result).isNull()
    }
}
```

- Use backtick method names for readable test descriptions
- Use `whenever` (mockito-kotlin) instead of `when` (reserved keyword in Kotlin)
- Add `mockito-kotlin` dependency for Kotlin-friendly Mockito extensions

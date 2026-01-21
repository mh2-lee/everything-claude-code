---
name: coding-standards
description: Universal coding standards, best practices, and patterns for Kotlin and Spring Boot development.
---

# Coding Standards & Best Practices

Universal coding standards applicable across all Kotlin/Spring Boot projects.

## Code Quality Principles

### 1. Readability First
- Code is read more than written
- Clear variable and function names
- Self-documenting code preferred over comments
- Consistent formatting (ktlint)

### 2. KISS (Keep It Simple, Stupid)
- Simplest solution that works
- Avoid over-engineering
- No premature optimization
- Easy to understand > clever code

### 3. DRY (Don't Repeat Yourself)
- Extract common logic into functions
- Create reusable components
- Share utilities across modules
- Avoid copy-paste programming

### 4. YAGNI (You Aren't Gonna Need It)
- Don't build features before they're needed
- Avoid speculative generality
- Add complexity only when required
- Start simple, refactor when needed

## Kotlin Standards

### Variable Naming

```kotlin
// ✅ GOOD: Descriptive names
val marketSearchQuery = "election"
val isUserAuthenticated = true
val totalRevenue = BigDecimal("1000.00")

// ❌ BAD: Unclear names
val q = "election"
val flag = true
val x = BigDecimal("1000.00")
```

### Function Naming

```kotlin
// ✅ GOOD: Verb-noun pattern
suspend fun fetchMarketData(marketId: Long): Market
fun calculateSimilarity(a: List<Double>, b: List<Double>): Double
fun isValidEmail(email: String): Boolean

// ❌ BAD: Unclear or noun-only
suspend fun market(id: Long): Market
fun similarity(a: List<Double>, b: List<Double>): Double
fun email(e: String): Boolean
```

### Immutability Pattern (CRITICAL)

```kotlin
// ✅ ALWAYS prefer val over var
val user = User(name = "John", email = "john@example.com")

// ✅ Use data class copy() for modifications
val updatedUser = user.copy(name = "Jane")

// ✅ Immutable collections
val items = listOf("a", "b", "c")
val updatedItems = items + "d"

// ❌ NEVER mutate directly when possible
var user = User(...)  // Avoid var
user.name = "New Name"  // Avoid mutation

// ❌ Mutable collections (use only when necessary)
val mutableItems = mutableListOf<String>()
mutableItems.add("item")  // Avoid if possible
```

### Null Safety

```kotlin
// ✅ GOOD: Safe call operator
val name = user?.name ?: "Unknown"

// ✅ GOOD: let for null checks
user?.let {
    processUser(it)
}

// ✅ GOOD: Elvis operator with return/throw
val email = user?.email ?: return
val id = user?.id ?: throw IllegalArgumentException("User ID required")

// ❌ BAD: !! operator (avoid unless absolutely certain)
val name = user!!.name  // Can throw NPE

// ❌ BAD: Java-style null checks
if (user != null) {
    if (user.name != null) {
        println(user.name)
    }
}
```

### Error Handling

```kotlin
// ✅ GOOD: Result type for expected failures
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val cause: Throwable? = null) : Result<Nothing>()
}

fun fetchUser(id: Long): Result<User> {
    return try {
        val user = userRepository.findById(id)
            .orElse(null) ?: return Result.Error("User not found")
        Result.Success(user)
    } catch (e: Exception) {
        logger.error("Failed to fetch user", e)
        Result.Error("Failed to fetch user", e)
    }
}

// ✅ GOOD: runCatching for simple cases
val result = runCatching { riskyOperation() }
    .getOrElse { defaultValue }

// ❌ BAD: Silent exception swallowing
try {
    riskyOperation()
} catch (e: Exception) {
    // Empty catch block - BAD!
}
```

### Extension Functions

```kotlin
// ✅ GOOD: Extension functions for common operations
fun String.toSlug(): String =
    this.lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")
        .trim('-')

fun <T> List<T>.secondOrNull(): T? =
    if (size >= 2) this[1] else null

// Usage
val slug = "Hello World!".toSlug()  // "hello-world"
val second = listOf(1, 2, 3).secondOrNull()  // 2
```

### Data Classes

```kotlin
// ✅ GOOD: Immutable data class
data class CreateUserRequest(
    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Invalid email format")
    val email: String,

    @field:NotBlank(message = "Name is required")
    @field:Size(min = 1, max = 100)
    val name: String,

    val role: UserRole = UserRole.USER
)

// ✅ GOOD: Entity with JPA
@Entity
@Table(name = "users")
data class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, unique = true)
    val email: String,

    @Column(nullable = false)
    val name: String,

    @Enumerated(EnumType.STRING)
    val status: UserStatus = UserStatus.ACTIVE,

    @Column(nullable = false, updatable = false)
    val createdAt: LocalDateTime = LocalDateTime.now()
)
```

### Sealed Classes for State

```kotlin
// ✅ GOOD: Sealed class for finite states
sealed class MarketStatus {
    object Draft : MarketStatus()
    object Active : MarketStatus()
    data class Resolved(val outcome: String) : MarketStatus()
    object Closed : MarketStatus()
}

// Usage with when (exhaustive)
fun handleStatus(status: MarketStatus): String = when (status) {
    is MarketStatus.Draft -> "Market is in draft"
    is MarketStatus.Active -> "Market is active"
    is MarketStatus.Resolved -> "Market resolved: ${status.outcome}"
    is MarketStatus.Closed -> "Market is closed"
}
```

## Spring Boot Best Practices

### Controller Layer

```kotlin
@RestController
@RequestMapping("/api/markets")
class MarketController(
    private val marketService: MarketService
) {
    @GetMapping
    fun getMarkets(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "10") size: Int
    ): ResponseEntity<ApiResponse<Page<MarketDto>>> {
        val markets = marketService.findAll(PageRequest.of(page, size))
        return ResponseEntity.ok(ApiResponse.success(markets))
    }

    @GetMapping("/{slug}")
    fun getMarket(@PathVariable slug: String): ResponseEntity<ApiResponse<MarketDto>> {
        val market = marketService.findBySlug(slug)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(ApiResponse.success(market))
    }

    @PostMapping
    fun createMarket(
        @Valid @RequestBody request: CreateMarketRequest
    ): ResponseEntity<ApiResponse<MarketDto>> {
        val market = marketService.create(request)
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(market))
    }
}
```

### Service Layer

```kotlin
@Service
class MarketService(
    private val marketRepository: MarketRepository,
    private val eventPublisher: ApplicationEventPublisher
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    @Transactional(readOnly = true)
    fun findAll(pageable: Pageable): Page<MarketDto> {
        return marketRepository.findAll(pageable).map { it.toDto() }
    }

    @Transactional(readOnly = true)
    fun findBySlug(slug: String): MarketDto? {
        return marketRepository.findBySlug(slug)?.toDto()
    }

    @Transactional
    fun create(request: CreateMarketRequest): MarketDto {
        logger.info("Creating market: ${request.name}")

        val market = Market(
            name = request.name,
            slug = request.name.toSlug(),
            description = request.description,
            status = MarketStatus.DRAFT
        )

        val saved = marketRepository.save(market)
        eventPublisher.publishEvent(MarketCreatedEvent(saved))

        return saved.toDto()
    }
}
```

### Repository Layer

```kotlin
interface MarketRepository : JpaRepository<Market, Long> {
    fun findBySlug(slug: String): Market?
    fun findByStatus(status: MarketStatus): List<Market>

    @Query("SELECT m FROM Market m WHERE m.name LIKE %:query% OR m.description LIKE %:query%")
    fun searchByQuery(@Param("query") query: String, pageable: Pageable): Page<Market>

    @Query("SELECT m FROM Market m WHERE m.status = :status ORDER BY m.createdAt DESC")
    fun findRecentByStatus(
        @Param("status") status: MarketStatus,
        pageable: Pageable
    ): Page<Market>
}
```

## API Design Standards

### REST API Conventions

```
GET    /api/markets              # List all markets (paginated)
GET    /api/markets/:slug        # Get specific market
POST   /api/markets              # Create new market
PUT    /api/markets/:slug        # Update market (full)
PATCH  /api/markets/:slug        # Update market (partial)
DELETE /api/markets/:slug        # Delete market

# Query parameters for filtering
GET /api/markets?status=ACTIVE&page=0&size=10&sort=createdAt,desc
```

### Response Format

```kotlin
// ✅ GOOD: Consistent response structure
data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val error: ErrorDetails? = null,
    val meta: Meta? = null
) {
    companion object {
        fun <T> success(data: T, meta: Meta? = null) = ApiResponse(
            success = true,
            data = data,
            meta = meta
        )

        fun error(message: String, code: String? = null) = ApiResponse<Nothing>(
            success = false,
            error = ErrorDetails(message, code)
        )
    }
}

data class ErrorDetails(
    val message: String,
    val code: String? = null,
    val details: Map<String, Any>? = null
)

data class Meta(
    val total: Long,
    val page: Int,
    val size: Int,
    val totalPages: Int
)
```

### Input Validation

```kotlin
// ✅ GOOD: Jakarta Bean Validation
data class CreateMarketRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 1, max = 200, message = "Name must be 1-200 characters")
    val name: String,

    @field:Size(max = 2000, message = "Description must be at most 2000 characters")
    val description: String? = null,

    @field:NotNull(message = "End date is required")
    @field:Future(message = "End date must be in the future")
    val endDate: LocalDateTime,

    @field:NotEmpty(message = "At least one category is required")
    val categories: List<String>
)

// Global exception handler
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidationException(ex: MethodArgumentNotValidException): ResponseEntity<ApiResponse<Nothing>> {
        val errors = ex.bindingResult.fieldErrors.associate {
            it.field to (it.defaultMessage ?: "Invalid value")
        }
        return ResponseEntity.badRequest().body(
            ApiResponse.error("Validation failed").copy(
                error = ErrorDetails(
                    message = "Validation failed",
                    code = "VALIDATION_ERROR",
                    details = errors
                )
            )
        )
    }
}
```

## File Organization

### Project Structure (Package by Feature)

```
src/main/kotlin/com/example/
├── Application.kt                  # @SpringBootApplication
├── config/                         # Configuration classes
│   ├── SecurityConfig.kt
│   ├── CacheConfig.kt
│   └── WebConfig.kt
├── market/                         # Market feature
│   ├── MarketController.kt
│   ├── MarketService.kt
│   ├── MarketRepository.kt
│   ├── Market.kt                   # Entity
│   ├── MarketDto.kt               # DTOs
│   └── MarketMapper.kt            # Entity <-> DTO mapping
├── user/                           # User feature
│   ├── UserController.kt
│   ├── UserService.kt
│   └── ...
└── common/                         # Shared code
    ├── exception/
    │   ├── GlobalExceptionHandler.kt
    │   └── Exceptions.kt
    ├── security/
    │   └── JwtTokenProvider.kt
    └── util/
        └── Extensions.kt
```

### File Naming

```
src/main/kotlin/
├── MarketController.kt          # PascalCase for classes
├── MarketService.kt
├── MarketRepository.kt
├── Market.kt                    # Entity
├── MarketDto.kt                 # DTO suffix
├── MarketMapper.kt              # Mapper suffix
└── Extensions.kt                # Utility extensions

src/main/resources/
├── application.yml              # Main config
├── application-dev.yml          # Dev profile
├── application-prod.yml         # Prod profile
└── db/migration/                # Flyway migrations
    ├── V1__create_users.sql
    └── V2__create_markets.sql
```

## Comments & Documentation

### When to Comment

```kotlin
// ✅ GOOD: Explain WHY, not WHAT
// Use exponential backoff to avoid overwhelming the API during outages
val delay = minOf(1000L * 2.0.pow(retryCount).toLong(), 30000L)

// Deliberately using mutable list here for performance with 10k+ items
val items = mutableListOf<Item>()

// ❌ BAD: Stating the obvious
// Increment counter by 1
count++

// Set name to user's name
name = user.name
```

### KDoc for Public APIs

```kotlin
/**
 * Searches markets using semantic similarity.
 *
 * @param query Natural language search query
 * @param limit Maximum number of results (default: 10)
 * @return List of markets sorted by similarity score
 * @throws SearchException if OpenAI API fails or Redis unavailable
 *
 * @sample
 * ```kotlin
 * val results = searchMarkets("election", 5)
 * println(results[0].name) // "Trump vs Biden"
 * ```
 */
fun searchMarkets(query: String, limit: Int = 10): List<Market>
```

## Performance Best Practices

### Database Queries

```kotlin
// ✅ GOOD: Select only needed columns with projections
interface MarketSummary {
    val id: Long
    val name: String
    val status: MarketStatus
}

@Query("SELECT m.id as id, m.name as name, m.status as status FROM Market m")
fun findAllSummaries(): List<MarketSummary>

// ✅ GOOD: Use pagination
fun findAll(pageable: Pageable): Page<Market>

// ✅ GOOD: Fetch relations in single query (avoid N+1)
@EntityGraph(attributePaths = ["categories", "creator"])
fun findBySlug(slug: String): Market?

// ❌ BAD: Select everything
fun findAll(): List<Market>  // Can return millions of rows
```

### Caching

```kotlin
@Service
class MarketService(
    private val marketRepository: MarketRepository
) {
    @Cacheable("markets", key = "#slug")
    fun findBySlug(slug: String): Market? {
        return marketRepository.findBySlug(slug)
    }

    @CacheEvict("markets", key = "#market.slug")
    @Transactional
    fun update(market: Market): Market {
        return marketRepository.save(market)
    }
}
```

## Testing Standards

### Test Structure (Given-When-Then)

```kotlin
@Test
fun `should calculate similarity correctly`() {
    // Given
    val vector1 = listOf(1.0, 0.0, 0.0)
    val vector2 = listOf(0.0, 1.0, 0.0)

    // When
    val similarity = calculateCosineSimilarity(vector1, vector2)

    // Then
    assertThat(similarity).isEqualTo(0.0)
}
```

### Test Naming

```kotlin
// ✅ GOOD: Descriptive test names (backtick syntax)
@Test
fun `should return empty list when no markets match query`() { }

@Test
fun `should throw exception when API key is missing`() { }

@Test
fun `should fall back to substring search when Redis unavailable`() { }

// ❌ BAD: Vague test names
@Test
fun `works`() { }

@Test
fun `test search`() { }
```

## Code Smell Detection

Watch for these anti-patterns:

### 1. Long Functions
```kotlin
// ❌ BAD: Function > 50 lines
fun processMarketData() {
    // 100 lines of code
}

// ✅ GOOD: Split into smaller functions
fun processMarketData() {
    val validated = validateData()
    val transformed = transformData(validated)
    return saveData(transformed)
}
```

### 2. Deep Nesting
```kotlin
// ❌ BAD: 5+ levels of nesting
if (user != null) {
    if (user.isAdmin) {
        if (market != null) {
            if (market.isActive) {
                // Do something
            }
        }
    }
}

// ✅ GOOD: Early returns
user ?: return
if (!user.isAdmin) return
market ?: return
if (!market.isActive) return

// Do something
```

### 3. Magic Numbers
```kotlin
// ❌ BAD: Unexplained numbers
if (retryCount > 3) { }
delay(500)

// ✅ GOOD: Named constants
companion object {
    private const val MAX_RETRIES = 3
    private const val DEBOUNCE_DELAY_MS = 500L
}

if (retryCount > MAX_RETRIES) { }
delay(DEBOUNCE_DELAY_MS)
```

**Remember**: Code quality is not negotiable. Clear, maintainable code enables rapid development and confident refactoring.

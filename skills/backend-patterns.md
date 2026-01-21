---
name: backend-patterns
description: Backend architecture patterns, API design, database optimization, and server-side best practices for Spring Boot and Kotlin.
---

# Backend Development Patterns

Backend architecture patterns and best practices for scalable Spring Boot applications with Kotlin.

## API Design Patterns

### RESTful API Structure

```kotlin
// Resource-based URLs
GET    /api/markets                 # List resources
GET    /api/markets/{id}            # Get single resource
POST   /api/markets                 # Create resource
PUT    /api/markets/{id}            # Replace resource
PATCH  /api/markets/{id}            # Update resource
DELETE /api/markets/{id}            # Delete resource

// Query parameters for filtering, sorting, pagination
GET /api/markets?status=active&sort=volume,desc&page=0&size=20
```

### Controller Layer

```kotlin
@RestController
@RequestMapping("/api/markets")
class MarketController(
    private val marketService: MarketService
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    @GetMapping
    fun findAll(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int,
        @RequestParam(required = false) status: MarketStatus?
    ): ResponseEntity<Page<MarketResponse>> {
        val pageable = PageRequest.of(page, size)
        val markets = marketService.findAll(status, pageable)
        return ResponseEntity.ok(markets.map { it.toResponse() })
    }

    @GetMapping("/{id}")
    fun findById(@PathVariable id: Long): ResponseEntity<MarketResponse> {
        val market = marketService.findById(id)
        return ResponseEntity.ok(market.toResponse())
    }

    @PostMapping
    fun create(@Valid @RequestBody request: CreateMarketRequest): ResponseEntity<MarketResponse> {
        val market = marketService.create(request)
        val location = URI.create("/api/markets/${market.id}")
        return ResponseEntity.created(location).body(market.toResponse())
    }

    @PutMapping("/{id}")
    fun update(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateMarketRequest
    ): ResponseEntity<MarketResponse> {
        val market = marketService.update(id, request)
        return ResponseEntity.ok(market.toResponse())
    }

    @DeleteMapping("/{id}")
    fun delete(@PathVariable id: Long): ResponseEntity<Void> {
        marketService.delete(id)
        return ResponseEntity.noContent().build()
    }
}
```

### Service Layer Pattern

```kotlin
@Service
@Transactional(readOnly = true)
class MarketService(
    private val marketRepository: MarketRepository,
    private val eventPublisher: ApplicationEventPublisher
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    fun findAll(status: MarketStatus?, pageable: Pageable): Page<Market> {
        return status?.let { marketRepository.findByStatus(it, pageable) }
            ?: marketRepository.findAll(pageable)
    }

    fun findById(id: Long): Market {
        return marketRepository.findByIdOrNull(id)
            ?: throw EntityNotFoundException("Market not found: $id")
    }

    @Transactional
    fun create(request: CreateMarketRequest): Market {
        val market = Market(
            name = request.name,
            description = request.description,
            status = MarketStatus.PENDING
        )

        val saved = marketRepository.save(market)
        eventPublisher.publishEvent(MarketCreatedEvent(saved))

        logger.info("Market created: ${saved.id}")
        return saved
    }

    @Transactional
    fun update(id: Long, request: UpdateMarketRequest): Market {
        val market = findById(id)

        val updated = market.copy(
            name = request.name ?: market.name,
            description = request.description ?: market.description
        )

        return marketRepository.save(updated)
    }

    @Transactional
    fun delete(id: Long) {
        val market = findById(id)
        marketRepository.delete(market)
        logger.info("Market deleted: $id")
    }
}
```

### Repository Pattern

```kotlin
interface MarketRepository : JpaRepository<Market, Long> {

    fun findByStatus(status: MarketStatus, pageable: Pageable): Page<Market>

    fun findByNameContainingIgnoreCase(name: String): List<Market>

    @Query("""
        SELECT m FROM Market m
        WHERE m.status = :status
        AND m.createdAt >= :since
        ORDER BY m.volume DESC
    """)
    fun findActiveMarketsSince(
        @Param("status") status: MarketStatus,
        @Param("since") since: Instant
    ): List<Market>

    @Query(value = """
        SELECT * FROM markets m
        WHERE m.status = 'ACTIVE'
        AND m.volume > :minVolume
        LIMIT :limit
    """, nativeQuery = true)
    fun findHighVolumeMarkets(
        @Param("minVolume") minVolume: Long,
        @Param("limit") limit: Int
    ): List<Market>
}
```

## Database Patterns

### Entity Design

```kotlin
@Entity
@Table(name = "markets")
data class Market(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, length = 200)
    val name: String,

    @Column(columnDefinition = "TEXT")
    val description: String?,

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    val status: MarketStatus = MarketStatus.PENDING,

    @Column(nullable = false)
    val volume: Long = 0,

    @CreatedDate
    @Column(nullable = false, updatable = false)
    val createdAt: Instant = Instant.now(),

    @LastModifiedDate
    @Column(nullable = false)
    val updatedAt: Instant = Instant.now(),

    @Version
    val version: Long = 0
)

enum class MarketStatus {
    PENDING, ACTIVE, CLOSED, CANCELLED
}
```

### N+1 Query Prevention

```kotlin
// BAD: N+1 query problem
val markets = marketRepository.findAll()
markets.forEach { market ->
    val creator = userRepository.findById(market.creatorId)  // N queries!
}

// GOOD: Use JOIN FETCH
@Query("""
    SELECT m FROM Market m
    JOIN FETCH m.creator
    WHERE m.status = :status
""")
fun findWithCreator(@Param("status") status: MarketStatus): List<Market>

// GOOD: Use @EntityGraph
@EntityGraph(attributePaths = ["creator", "positions"])
fun findByStatus(status: MarketStatus): List<Market>

// GOOD: Batch fetch with IN clause
val markets = marketRepository.findAll()
val creatorIds = markets.map { it.creatorId }.distinct()
val creators = userRepository.findAllById(creatorIds)
val creatorMap = creators.associateBy { it.id }
```

### Transaction Management

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val inventoryService: InventoryService,
    private val paymentService: PaymentService
) {
    @Transactional
    fun createOrder(request: CreateOrderRequest): Order {
        // All operations in single transaction
        val order = orderRepository.save(Order(request))

        inventoryService.reserve(order.items)
        paymentService.charge(order.totalAmount)

        return order
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun processPayment(orderId: Long) {
        // Independent transaction
    }

    @Transactional(readOnly = true)
    fun findOrder(id: Long): Order {
        // Read-only transaction for better performance
        return orderRepository.findByIdOrNull(id)
            ?: throw EntityNotFoundException("Order not found")
    }
}
```

## Caching Strategies

### Spring Cache with Redis

```kotlin
@Configuration
@EnableCaching
class CacheConfig {

    @Bean
    fun cacheManager(redisConnectionFactory: RedisConnectionFactory): CacheManager {
        val config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair
                    .fromSerializer(GenericJackson2JsonRedisSerializer())
            )

        return RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(config)
            .withCacheConfiguration("markets", config.entryTtl(Duration.ofMinutes(5)))
            .build()
    }
}

@Service
class MarketService(private val marketRepository: MarketRepository) {

    @Cacheable(value = ["markets"], key = "#id")
    fun findById(id: Long): Market {
        return marketRepository.findByIdOrNull(id)
            ?: throw EntityNotFoundException("Market not found")
    }

    @CachePut(value = ["markets"], key = "#result.id")
    @Transactional
    fun update(id: Long, request: UpdateMarketRequest): Market {
        val market = findById(id)
        return marketRepository.save(market.copy(name = request.name))
    }

    @CacheEvict(value = ["markets"], key = "#id")
    @Transactional
    fun delete(id: Long) {
        marketRepository.deleteById(id)
    }

    @CacheEvict(value = ["markets"], allEntries = true)
    fun clearCache() {
        // Clear all market cache entries
    }
}
```

## Error Handling Patterns

### Global Exception Handler

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    private val logger = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(EntityNotFoundException::class)
    fun handleNotFound(e: EntityNotFoundException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse(
                code = "NOT_FOUND",
                message = e.message ?: "Resource not found"
            ))
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(e: MethodArgumentNotValidException): ResponseEntity<ErrorResponse> {
        val errors = e.bindingResult.fieldErrors.map { error ->
            FieldError(error.field, error.defaultMessage ?: "Invalid value")
        }

        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(ErrorResponse(
                code = "VALIDATION_ERROR",
                message = "Validation failed",
                errors = errors
            ))
    }

    @ExceptionHandler(OptimisticLockingFailureException::class)
    fun handleOptimisticLock(e: OptimisticLockingFailureException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .status(HttpStatus.CONFLICT)
            .body(ErrorResponse(
                code = "CONFLICT",
                message = "Resource was modified by another request"
            ))
    }

    @ExceptionHandler(Exception::class)
    fun handleGeneric(e: Exception): ResponseEntity<ErrorResponse> {
        logger.error("Unexpected error", e)

        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse(
                code = "INTERNAL_ERROR",
                message = "An unexpected error occurred"
            ))
    }
}

data class ErrorResponse(
    val code: String,
    val message: String,
    val errors: List<FieldError>? = null,
    val timestamp: Instant = Instant.now()
)

data class FieldError(
    val field: String,
    val message: String
)
```

### Custom Business Exceptions

```kotlin
sealed class BusinessException(
    override val message: String,
    val code: String
) : RuntimeException(message)

class InsufficientBalanceException(userId: Long, required: BigDecimal) :
    BusinessException("Insufficient balance for user $userId. Required: $required", "INSUFFICIENT_BALANCE")

class MarketClosedException(marketId: Long) :
    BusinessException("Market $marketId is closed", "MARKET_CLOSED")

class DuplicateEmailException(email: String) :
    BusinessException("Email already exists: $email", "DUPLICATE_EMAIL")
```

## Authentication & Authorization

### Spring Security with JWT

```kotlin
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
class SecurityConfig(
    private val jwtTokenProvider: JwtTokenProvider
) {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        return http
            .csrf { it.disable() }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/auth/**").permitAll()
                    .requestMatchers("/api/public/**").permitAll()
                    .requestMatchers("/actuator/health").permitAll()
                    .anyRequest().authenticated()
            }
            .addFilterBefore(
                JwtAuthenticationFilter(jwtTokenProvider),
                UsernamePasswordAuthenticationFilter::class.java
            )
            .build()
    }
}

@Component
class JwtTokenProvider(
    @Value("\${jwt.secret}") private val secret: String,
    @Value("\${jwt.expiration}") private val expiration: Long
) {
    private val key: SecretKey = Keys.hmacShaKeyFor(secret.toByteArray())

    fun createToken(userId: Long, roles: List<String>): String {
        val now = Date()
        val validity = Date(now.time + expiration)

        return Jwts.builder()
            .setSubject(userId.toString())
            .claim("roles", roles)
            .setIssuedAt(now)
            .setExpiration(validity)
            .signWith(key)
            .compact()
    }

    fun validateToken(token: String): Boolean {
        return try {
            Jwts.parserBuilder().setSigningKey(key).build().parseClaimsJws(token)
            true
        } catch (e: JwtException) {
            false
        }
    }

    fun getUserId(token: String): Long {
        val claims = Jwts.parserBuilder().setSigningKey(key).build()
            .parseClaimsJws(token).body
        return claims.subject.toLong()
    }
}
```

### Method-Level Security

```kotlin
@Service
class MarketService {

    @PreAuthorize("hasRole('ADMIN')")
    fun deleteMarket(id: Long) {
        // Only admins can delete
    }

    @PreAuthorize("hasRole('USER') and #userId == authentication.principal.id")
    fun getUserOrders(userId: Long): List<Order> {
        // Users can only access their own orders
    }

    @PostAuthorize("returnObject.creatorId == authentication.principal.id or hasRole('ADMIN')")
    fun findMarket(id: Long): Market {
        // Return only if user owns the market or is admin
    }
}
```

## Event-Driven Patterns

### Application Events

```kotlin
// Define events
data class MarketCreatedEvent(val market: Market)
data class OrderCompletedEvent(val order: Order)

// Publish events
@Service
class MarketService(
    private val eventPublisher: ApplicationEventPublisher
) {
    @Transactional
    fun create(request: CreateMarketRequest): Market {
        val market = marketRepository.save(Market(request))
        eventPublisher.publishEvent(MarketCreatedEvent(market))
        return market
    }
}

// Listen to events
@Component
class MarketEventListener(
    private val notificationService: NotificationService,
    private val searchIndexService: SearchIndexService
) {
    @EventListener
    fun handleMarketCreated(event: MarketCreatedEvent) {
        notificationService.notifyFollowers(event.market)
    }

    @Async
    @EventListener
    fun indexMarket(event: MarketCreatedEvent) {
        searchIndexService.index(event.market)
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun afterCommit(event: MarketCreatedEvent) {
        // Execute after transaction commits
    }
}
```

## Logging & Monitoring

### Structured Logging

```kotlin
@Aspect
@Component
class LoggingAspect {

    @Around("@within(org.springframework.web.bind.annotation.RestController)")
    fun logControllerMethods(joinPoint: ProceedingJoinPoint): Any? {
        val logger = LoggerFactory.getLogger(joinPoint.target.javaClass)
        val methodName = joinPoint.signature.name
        val args = joinPoint.args.contentToString()

        val startTime = System.currentTimeMillis()

        return try {
            val result = joinPoint.proceed()
            val duration = System.currentTimeMillis() - startTime

            logger.info("method={} args={} duration={}ms status=success", methodName, args, duration)
            result
        } catch (e: Exception) {
            val duration = System.currentTimeMillis() - startTime
            logger.error("method={} args={} duration={}ms status=error error={}",
                methodName, args, duration, e.message)
            throw e
        }
    }
}
```

### Micrometer Metrics

```kotlin
@Service
class MarketService(
    private val marketRepository: MarketRepository,
    private val meterRegistry: MeterRegistry
) {
    private val createCounter = Counter.builder("market.created")
        .description("Number of markets created")
        .register(meterRegistry)

    private val findTimer = Timer.builder("market.find")
        .description("Time to find market")
        .register(meterRegistry)

    @Transactional
    fun create(request: CreateMarketRequest): Market {
        val market = marketRepository.save(Market(request))
        createCounter.increment()
        return market
    }

    fun findById(id: Long): Market {
        return findTimer.recordCallable {
            marketRepository.findByIdOrNull(id)
                ?: throw EntityNotFoundException("Market not found")
        }!!
    }
}
```

**Remember**: Spring Boot patterns enable scalable, maintainable server-side applications. Choose patterns that fit your complexity level.

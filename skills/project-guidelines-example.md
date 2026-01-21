# Project Guidelines Skill (Example)

This is an example of a project-specific skill. Use this as a template for your own projects.

Based on a real production application architecture pattern for enterprise Spring Boot applications.

---

## When to Use

Reference this skill when working on the specific project it's designed for. Project skills contain:
- Architecture overview
- File structure
- Code patterns
- Testing requirements
- Deployment workflow

---

## Architecture Overview

**Tech Stack:**
- **Backend**: Spring Boot 3.x, Kotlin, WebFlux (reactive)
- **Database**: PostgreSQL (primary), Redis (cache)
- **AI**: Claude API with tool calling and structured output
- **Deployment**: Kubernetes (GKE/EKS)
- **Testing**: JUnit5, Testcontainers, MockK

**Services:**
```
┌─────────────────────────────────────────────────────────────┐
│                      API Gateway                             │
│  Spring Cloud Gateway + Rate Limiting                       │
│  Deployed: Kubernetes Ingress                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Application Service                       │
│  Spring Boot 3.x + Kotlin + WebFlux                         │
│  Deployed: Kubernetes Deployment                            │
└─────────────────────────────────────────────────────────────┘
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        ┌──────────┐   ┌──────────┐   ┌──────────┐
        │PostgreSQL│   │  Claude  │   │  Redis   │
        │ Database │   │   API    │   │  Cache   │
        └──────────┘   └──────────┘   └──────────┘
```

---

## File Structure

```
project/
├── src/
│   ├── main/
│   │   ├── kotlin/com/example/
│   │   │   ├── Application.kt           # @SpringBootApplication
│   │   │   ├── config/                  # Configuration classes
│   │   │   │   ├── SecurityConfig.kt
│   │   │   │   ├── CacheConfig.kt
│   │   │   │   ├── WebConfig.kt
│   │   │   │   └── ClaudeConfig.kt
│   │   │   ├── market/                  # Market domain
│   │   │   │   ├── MarketController.kt
│   │   │   │   ├── MarketService.kt
│   │   │   │   ├── MarketRepository.kt
│   │   │   │   ├── Market.kt            # Entity
│   │   │   │   ├── MarketDto.kt         # DTOs
│   │   │   │   └── MarketMapper.kt
│   │   │   ├── user/                    # User domain
│   │   │   │   ├── UserController.kt
│   │   │   │   ├── UserService.kt
│   │   │   │   └── ...
│   │   │   ├── ai/                      # AI integration
│   │   │   │   ├── ClaudeService.kt
│   │   │   │   └── AnalysisModels.kt
│   │   │   └── common/                  # Shared code
│   │   │       ├── exception/
│   │   │       │   ├── GlobalExceptionHandler.kt
│   │   │       │   └── Exceptions.kt
│   │   │       ├── security/
│   │   │       │   └── JwtTokenProvider.kt
│   │   │       └── util/
│   │   │           └── Extensions.kt
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       └── db/migration/            # Flyway migrations
│   │           ├── V1__create_users.sql
│   │           └── V2__create_markets.sql
│   └── test/
│       └── kotlin/com/example/
│           ├── market/
│           │   ├── MarketControllerTest.kt
│           │   ├── MarketServiceTest.kt
│           │   └── MarketRepositoryTest.kt
│           ├── integration/
│           │   └── MarketIntegrationTest.kt
│           └── TestConfig.kt
├── build.gradle.kts
├── settings.gradle.kts
├── docker-compose.yml
└── k8s/                                 # Kubernetes manifests
    ├── deployment.yaml
    ├── service.yaml
    └── configmap.yaml
```

---

## Code Patterns

### API Response Format (Kotlin)

```kotlin
// Sealed class for type-safe API responses
sealed class ApiResponse<out T> {
    data class Success<T>(
        val data: T,
        val meta: Meta? = null
    ) : ApiResponse<T>()

    data class Error(
        val error: String,
        val code: String? = null,
        val details: Map<String, Any>? = null
    ) : ApiResponse<Nothing>()

    companion object {
        fun <T> ok(data: T, meta: Meta? = null): ApiResponse<T> =
            Success(data, meta)

        fun fail(error: String, code: String? = null): ApiResponse<Nothing> =
            Error(error, code)
    }
}

data class Meta(
    val total: Long,
    val page: Int,
    val limit: Int,
    val totalPages: Int
)
```

### Controller Pattern

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
        return ResponseEntity.ok(ApiResponse.ok(markets))
    }

    @GetMapping("/{slug}")
    fun getMarket(@PathVariable slug: String): ResponseEntity<ApiResponse<MarketDto>> {
        val market = marketService.findBySlug(slug)
            ?: return ResponseEntity.notFound().build()
        return ResponseEntity.ok(ApiResponse.ok(market))
    }

    @PostMapping
    fun createMarket(
        @Valid @RequestBody request: CreateMarketRequest
    ): ResponseEntity<ApiResponse<MarketDto>> {
        val market = marketService.create(request)
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.ok(market))
    }
}
```

### Claude AI Integration (Kotlin)

```kotlin
@Service
class ClaudeService(
    @Value("\${anthropic.api-key}")
    private val apiKey: String
) {
    private val client = Anthropic(apiKey)
    private val logger = LoggerFactory.getLogger(javaClass)

    suspend fun analyzeContent(content: String): AnalysisResult {
        val response = client.messages.create(
            model = "claude-sonnet-4-5-20250514",
            maxTokens = 1024,
            messages = listOf(
                MessageParam.builder()
                    .role(MessageParam.Role.USER)
                    .content(content)
                    .build()
            ),
            tools = listOf(
                ToolParam.builder()
                    .name("provide_analysis")
                    .description("Provide structured analysis")
                    .inputSchema(AnalysisResult.jsonSchema())
                    .build()
            ),
            toolChoice = ToolChoice.tool("provide_analysis")
        )

        val toolUse = response.content
            .filterIsInstance<ContentBlock.ToolUse>()
            .first()

        return objectMapper.convertValue(toolUse.input, AnalysisResult::class.java)
    }
}

data class AnalysisResult(
    val summary: String,
    val keyPoints: List<String>,
    val confidence: Double
) {
    companion object {
        fun jsonSchema(): Map<String, Any> = mapOf(
            "type" to "object",
            "properties" to mapOf(
                "summary" to mapOf("type" to "string"),
                "keyPoints" to mapOf(
                    "type" to "array",
                    "items" to mapOf("type" to "string")
                ),
                "confidence" to mapOf("type" to "number")
            ),
            "required" to listOf("summary", "keyPoints", "confidence")
        )
    }
}
```

### Service Layer Pattern

```kotlin
@Service
class MarketService(
    private val marketRepository: MarketRepository,
    private val cacheService: CacheService,
    private val eventPublisher: ApplicationEventPublisher
) {
    private val logger = LoggerFactory.getLogger(javaClass)

    @Transactional(readOnly = true)
    fun findAll(pageable: Pageable): Page<MarketDto> {
        return marketRepository.findAll(pageable).map { it.toDto() }
    }

    @Transactional(readOnly = true)
    @Cacheable("markets", key = "#slug")
    fun findBySlug(slug: String): MarketDto? {
        return marketRepository.findBySlug(slug)?.toDto()
    }

    @Transactional
    @CacheEvict("markets", key = "#result.slug")
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

    @Transactional
    @CacheEvict("markets", key = "#slug")
    fun update(slug: String, request: UpdateMarketRequest): MarketDto {
        val market = marketRepository.findBySlug(slug)
            ?: throw EntityNotFoundException("Market not found: $slug")

        val updated = market.copy(
            name = request.name ?: market.name,
            description = request.description ?: market.description,
            status = request.status ?: market.status
        )

        return marketRepository.save(updated).toDto()
    }
}

// Extension function for slug generation
fun String.toSlug(): String =
    this.lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")
        .trim('-')
```

---

## Testing Requirements

### Unit Tests (JUnit5 + MockK)

```bash
# Run all tests
./gradlew test

# Run with coverage
./gradlew test jacocoTestReport

# Run specific test class
./gradlew test --tests "MarketServiceTest"
```

**Test structure:**
```kotlin
@ExtendWith(MockKExtension::class)
class MarketServiceTest {

    @MockK
    lateinit var marketRepository: MarketRepository

    @MockK
    lateinit var cacheService: CacheService

    @MockK
    lateinit var eventPublisher: ApplicationEventPublisher

    @InjectMockKs
    lateinit var marketService: MarketService

    @BeforeEach
    fun setup() {
        clearAllMocks()
    }

    @Test
    fun `should create market successfully`() {
        // Given
        val request = CreateMarketRequest(
            name = "Test Market",
            description = "Test description"
        )
        val savedMarket = Market(
            id = 1L,
            name = "Test Market",
            slug = "test-market",
            description = "Test description",
            status = MarketStatus.DRAFT
        )

        every { marketRepository.save(any()) } returns savedMarket
        every { eventPublisher.publishEvent(any<MarketCreatedEvent>()) } just Runs

        // When
        val result = marketService.create(request)

        // Then
        assertThat(result.name).isEqualTo("Test Market")
        assertThat(result.slug).isEqualTo("test-market")
        verify { marketRepository.save(any()) }
        verify { eventPublisher.publishEvent(any<MarketCreatedEvent>()) }
    }

    @Test
    fun `should throw exception when market not found`() {
        // Given
        every { marketRepository.findBySlug("unknown") } returns null

        // When & Then
        assertThatThrownBy { marketService.update("unknown", UpdateMarketRequest()) }
            .isInstanceOf(EntityNotFoundException::class.java)
            .hasMessage("Market not found: unknown")
    }
}
```

### Integration Tests (Testcontainers)

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class MarketIntegrationTest {

    companion object {
        @Container
        val postgres = PostgreSQLContainer("postgres:15")

        @Container
        val redis = GenericContainer("redis:7")
            .withExposedPorts(6379)

        @JvmStatic
        @DynamicPropertySource
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
            registry.add("spring.redis.host", redis::getHost)
            registry.add("spring.redis.port") { redis.getMappedPort(6379) }
        }
    }

    @Autowired
    lateinit var restTemplate: TestRestTemplate

    @Autowired
    lateinit var marketRepository: MarketRepository

    @BeforeEach
    fun setup() {
        marketRepository.deleteAll()
    }

    @Test
    fun `full market creation workflow`() {
        // Create market
        val createRequest = CreateMarketRequest(
            name = "Test Market",
            description = "Test description"
        )
        val createResponse = restTemplate.postForEntity(
            "/api/markets",
            createRequest,
            object : ParameterizedTypeReference<ApiResponse<MarketDto>>() {}
        )

        assertThat(createResponse.statusCode).isEqualTo(HttpStatus.CREATED)
        assertThat(createResponse.body?.data?.name).isEqualTo("Test Market")

        // Fetch market
        val getResponse = restTemplate.exchange(
            "/api/markets/test-market",
            HttpMethod.GET,
            null,
            object : ParameterizedTypeReference<ApiResponse<MarketDto>>() {}
        )

        assertThat(getResponse.statusCode).isEqualTo(HttpStatus.OK)
        assertThat(getResponse.body?.data?.slug).isEqualTo("test-market")
    }
}
```

### Controller Tests (@WebMvcTest)

```kotlin
@WebMvcTest(MarketController::class)
class MarketControllerTest {

    @Autowired
    lateinit var mockMvc: MockMvc

    @MockkBean
    lateinit var marketService: MarketService

    @Test
    fun `GET api_markets should return markets successfully`() {
        // Given
        val markets = PageImpl(listOf(
            MarketDto(id = 1, name = "Test Market", slug = "test-market")
        ))
        every { marketService.findAll(any()) } returns markets

        // When & Then
        mockMvc.perform(get("/api/markets"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.data.content[0].name").value("Test Market"))
    }

    @Test
    fun `POST api_markets should validate request body`() {
        val invalidRequest = """{"name": "", "description": ""}"""

        mockMvc.perform(
            post("/api/markets")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidRequest)
        )
            .andExpect(status().isBadRequest)
    }
}
```

---

## Deployment Workflow

### Pre-Deployment Checklist

- [ ] All tests passing locally (`./gradlew test`)
- [ ] Build succeeds (`./gradlew build`)
- [ ] Detekt passes (`./gradlew detekt`)
- [ ] No hardcoded secrets
- [ ] Environment variables documented
- [ ] Database migrations ready (Flyway)

### Deployment Commands

```bash
# Build Docker image
./gradlew bootBuildImage --imageName=myapp:latest

# Run locally with Docker Compose
docker-compose up -d

# Deploy to Kubernetes
kubectl apply -f k8s/

# Check deployment status
kubectl rollout status deployment/myapp
```

### Environment Variables

```yaml
# application.yml
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/mydb}
    username: ${DATABASE_USER:postgres}
    password: ${DATABASE_PASSWORD:}
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

anthropic:
  api-key: ${ANTHROPIC_API_KEY:}

jwt:
  secret: ${JWT_SECRET:}
  expiration: ${JWT_EXPIRATION:86400000}
```

### Kubernetes Deployment

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: database-url
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: anthropic-api-key
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
```

---

## Critical Rules

1. **No emojis** in code, comments, or documentation
2. **Immutability** - use val, data class copy(), immutable collections
3. **TDD** - write tests before implementation
4. **80% coverage** minimum
5. **Many small files** - 200-400 lines typical, 800 max
6. **No println** in production code (use logger)
7. **Proper error handling** with sealed classes or Result type
8. **Input validation** with Jakarta Bean Validation (@Valid, @NotBlank)
9. **Null safety** - leverage Kotlin's type system, avoid !!

---

## Related Skills

- `coding-standards.md` - General Kotlin/Spring Boot best practices
- `backend-patterns.md` - API and database patterns
- `tdd-workflow/` - Test-driven development methodology
- `security-review/` - Security best practices

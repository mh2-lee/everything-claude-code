---
name: tdd-workflow
description: Use this skill when writing new features, fixing bugs, or refactoring code. Enforces test-driven development with 80%+ coverage including unit, integration, and E2E tests for Kotlin/Spring Boot.
---

# Test-Driven Development Workflow

This skill ensures all Kotlin/Spring Boot code development follows TDD principles with comprehensive test coverage.

## When to Activate

- Writing new features or functionality
- Fixing bugs or issues
- Refactoring existing code
- Adding API endpoints
- Creating new services or repositories

## Core Principles

### 1. Tests BEFORE Code
ALWAYS write tests first, then implement code to make tests pass.

### 2. Coverage Requirements
- Minimum 80% coverage (unit + integration + E2E)
- All edge cases covered
- Error scenarios tested
- Boundary conditions verified

### 3. Test Types

#### Unit Tests
- Individual functions and utilities
- Service layer logic
- Pure functions
- Helpers and utilities

#### Integration Tests
- API endpoints (@WebMvcTest, @SpringBootTest)
- Database operations (@DataJpaTest)
- Service interactions
- External API calls

#### E2E Tests (Selenium)
- Critical user flows
- Complete workflows
- Browser automation
- UI interactions

## TDD Workflow Steps

### Step 1: Write User Stories
```
As a [role], I want to [action], so that [benefit]

Example:
As a user, I want to search for markets semantically,
so that I can find relevant markets even without exact keywords.
```

### Step 2: Generate Test Cases
For each user story, create comprehensive test cases:

```kotlin
@ExtendWith(MockitoExtension::class)
class MarketServiceTest {

    @Mock
    lateinit var marketRepository: MarketRepository

    @Mock
    lateinit var searchService: SearchService

    @InjectMocks
    lateinit var marketService: MarketService

    @Test
    fun `should return relevant markets for query`() {
        // Given
        val query = "election"
        val expectedMarkets = listOf(
            Market(id = 1, name = "Trump vs Biden", slug = "trump-biden"),
            Market(id = 2, name = "Election 2024", slug = "election-2024")
        )
        whenever(searchService.search(query)).thenReturn(expectedMarkets)

        // When
        val results = marketService.searchMarkets(query)

        // Then
        assertThat(results).hasSize(2)
        assertThat(results[0].name).contains("Trump")
    }

    @Test
    fun `should handle empty query gracefully`() {
        // Given & When & Then
        assertThatThrownBy { marketService.searchMarkets("") }
            .isInstanceOf(IllegalArgumentException::class.java)
            .hasMessage("Query cannot be empty")
    }

    @Test
    fun `should fall back to substring search when Redis unavailable`() {
        // Given
        val query = "election"
        whenever(searchService.search(query)).thenThrow(RedisConnectionException::class.java)
        whenever(marketRepository.findByNameContaining(query)).thenReturn(listOf(
            Market(id = 1, name = "Election Market", slug = "election-market")
        ))

        // When
        val results = marketService.searchMarkets(query)

        // Then
        assertThat(results).hasSize(1)
        verify(marketRepository).findByNameContaining(query)
    }
}
```

### Step 3: Run Tests (They Should Fail)
```bash
./gradlew test
# Tests should fail - we haven't implemented yet
```

### Step 4: Implement Code
Write minimal code to make tests pass:

```kotlin
@Service
class MarketService(
    private val marketRepository: MarketRepository,
    private val searchService: SearchService
) {
    fun searchMarkets(query: String): List<Market> {
        require(query.isNotBlank()) { "Query cannot be empty" }

        return try {
            searchService.search(query)
        } catch (e: Exception) {
            logger.warn("Search service unavailable, falling back to substring search", e)
            marketRepository.findByNameContaining(query)
        }
    }
}
```

### Step 5: Run Tests Again
```bash
./gradlew test
# Tests should now pass
```

### Step 6: Refactor
Improve code quality while keeping tests green:
- Remove duplication
- Improve naming
- Optimize performance
- Enhance readability

### Step 7: Verify Coverage
```bash
./gradlew test jacocoTestReport
# Verify 80%+ coverage achieved
open build/reports/jacoco/test/html/index.html
```

## Testing Patterns

### Unit Test Pattern (JUnit5 + Mockito)
```kotlin
@ExtendWith(MockitoExtension::class)
class UserServiceTest {

    @Mock
    lateinit var userRepository: UserRepository

    @Mock
    lateinit var passwordEncoder: PasswordEncoder

    @InjectMocks
    lateinit var userService: UserService

    @Test
    fun `should create user with encoded password`() {
        // Given
        val request = CreateUserRequest(
            email = "test@example.com",
            password = "password123",
            name = "Test User"
        )
        val encodedPassword = "encoded_password_hash"
        whenever(passwordEncoder.encode(request.password)).thenReturn(encodedPassword)
        whenever(userRepository.save(any())).thenAnswer { it.arguments[0] }

        // When
        val result = userService.createUser(request)

        // Then
        assertThat(result.email).isEqualTo("test@example.com")
        assertThat(result.password).isEqualTo(encodedPassword)
        verify(passwordEncoder).encode("password123")
    }

    @Test
    fun `should throw exception when email already exists`() {
        // Given
        val request = CreateUserRequest(
            email = "existing@example.com",
            password = "password123",
            name = "Test User"
        )
        whenever(userRepository.findByEmail(request.email)).thenReturn(
            User(id = 1, email = request.email, password = "hash", name = "Existing")
        )

        // When & Then
        assertThatThrownBy { userService.createUser(request) }
            .isInstanceOf(EmailAlreadyExistsException::class.java)
    }
}
```

### Controller Integration Test Pattern (@WebMvcTest)
```kotlin
@WebMvcTest(MarketController::class)
class MarketControllerTest {

    @Autowired
    lateinit var mockMvc: MockMvc

    @MockBean
    lateinit var marketService: MarketService

    @Test
    fun `GET api_markets should return markets successfully`() {
        // Given
        val markets = listOf(
            Market(id = 1, name = "Test Market", slug = "test-market")
        )
        whenever(marketService.findAll()).thenReturn(markets)

        // When & Then
        mockMvc.perform(get("/api/markets"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.success").value(true))
            .andExpect(jsonPath("$.data").isArray)
            .andExpect(jsonPath("$.data[0].name").value("Test Market"))
    }

    @Test
    fun `GET api_markets_search should validate query parameter`() {
        mockMvc.perform(get("/api/markets/search"))
            .andExpect(status().isBadRequest)
    }

    @Test
    fun `POST api_markets should validate request body`() {
        val invalidRequest = """{"name": "", "slug": ""}"""

        mockMvc.perform(
            post("/api/markets")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidRequest)
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.success").value(false))
    }
}
```

### Repository Integration Test Pattern (@DataJpaTest)
```kotlin
@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class MarketRepositoryTest {

    companion object {
        @Container
        val postgres = PostgreSQLContainer("postgres:15")
            .withDatabaseName("testdb")

        @JvmStatic
        @DynamicPropertySource
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }

    @Autowired
    lateinit var marketRepository: MarketRepository

    @BeforeEach
    fun setup() {
        marketRepository.deleteAll()
    }

    @Test
    fun `should find markets by status`() {
        // Given
        marketRepository.saveAll(listOf(
            Market(name = "Active Market", slug = "active", status = MarketStatus.ACTIVE),
            Market(name = "Closed Market", slug = "closed", status = MarketStatus.CLOSED)
        ))

        // When
        val activeMarkets = marketRepository.findByStatus(MarketStatus.ACTIVE)

        // Then
        assertThat(activeMarkets).hasSize(1)
        assertThat(activeMarkets[0].name).isEqualTo("Active Market")
    }

    @Test
    fun `should search markets by name containing`() {
        // Given
        marketRepository.saveAll(listOf(
            Market(name = "Election 2024", slug = "election-2024", status = MarketStatus.ACTIVE),
            Market(name = "Sports Event", slug = "sports-event", status = MarketStatus.ACTIVE)
        ))

        // When
        val results = marketRepository.findByNameContaining("Election")

        // Then
        assertThat(results).hasSize(1)
        assertThat(results[0].slug).isEqualTo("election-2024")
    }
}
```

### Full Integration Test Pattern (@SpringBootTest)
```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
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
            slug = "test-market",
            description = "Test description"
        )
        val createResponse = restTemplate.postForEntity(
            "/api/markets",
            createRequest,
            ApiResponse::class.java
        )
        assertThat(createResponse.statusCode).isEqualTo(HttpStatus.CREATED)

        // Fetch market
        val getResponse = restTemplate.getForEntity(
            "/api/markets/test-market",
            ApiResponse::class.java
        )
        assertThat(getResponse.statusCode).isEqualTo(HttpStatus.OK)
        assertThat(getResponse.body?.success).isTrue()
    }
}
```

### E2E Test Pattern (Selenium)
```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class MarketE2ETest {

    @LocalServerPort
    private var port: Int = 0

    private lateinit var driver: WebDriver

    @BeforeEach
    fun setup() {
        WebDriverManager.chromedriver().setup()
        val options = ChromeOptions().apply {
            addArguments("--headless")
            addArguments("--no-sandbox")
        }
        driver = ChromeDriver(options)
    }

    @AfterEach
    fun teardown() {
        driver.quit()
    }

    @Test
    fun `user can search and filter markets`() {
        // Navigate to markets page
        driver.get("http://localhost:$port/markets")

        // Verify page loaded
        val title = driver.findElement(By.tagName("h1"))
        assertThat(title.text).contains("Markets")

        // Search for markets
        val searchInput = driver.findElement(By.cssSelector("[data-testid='search-input']"))
        searchInput.sendKeys("election")

        // Wait for debounce and results
        Thread.sleep(600)

        // Verify search results displayed
        val wait = WebDriverWait(driver, Duration.ofSeconds(5))
        val results = wait.until(
            ExpectedConditions.presenceOfAllElementsLocatedBy(
                By.cssSelector("[data-testid='market-card']")
            )
        )
        assertThat(results).hasSizeGreaterThan(0)
    }
}
```

## Test File Organization

```
src/
├── main/kotlin/com/example/
│   ├── Application.kt
│   ├── market/
│   │   ├── MarketController.kt
│   │   ├── MarketService.kt
│   │   ├── MarketRepository.kt
│   │   └── Market.kt
│   └── user/
│       └── ...
└── test/kotlin/com/example/
    ├── market/
    │   ├── MarketControllerTest.kt      # @WebMvcTest
    │   ├── MarketServiceTest.kt         # Unit tests
    │   ├── MarketRepositoryTest.kt      # @DataJpaTest
    │   └── MarketIntegrationTest.kt     # @SpringBootTest
    ├── e2e/
    │   ├── MarketE2ETest.kt             # Selenium E2E
    │   └── AuthE2ETest.kt
    └── TestConfig.kt                     # Test configurations
```

## Mocking External Services

### Repository Mock (Mockito-Kotlin)
```kotlin
@Mock
lateinit var marketRepository: MarketRepository

@Test
fun `should find by status`() {
    whenever(marketRepository.findByStatus(MarketStatus.ACTIVE)).thenReturn(
        listOf(Market(id = 1, name = "Test", slug = "test", status = MarketStatus.ACTIVE))
    )

    val results = marketService.getActiveMarkets()

    assertThat(results).hasSize(1)
    verify(marketRepository).findByStatus(MarketStatus.ACTIVE)
}
```

### External API Mock (MockK)
```kotlin
@Test
fun `should handle OpenAI API failure`() {
    val openAiService = mockk<OpenAiService>()
    every { openAiService.generateEmbedding(any()) } throws OpenAiException("API Error")

    val service = SearchService(openAiService, marketRepository)

    assertThatThrownBy { service.semanticSearch("query") }
        .isInstanceOf(SearchException::class.java)
}
```

### Redis Mock
```kotlin
@MockBean
lateinit var redisTemplate: RedisTemplate<String, Any>

@Test
fun `should cache market data`() {
    val opsForValue = mock<ValueOperations<String, Any>>()
    whenever(redisTemplate.opsForValue()).thenReturn(opsForValue)
    whenever(opsForValue.get("market:test")).thenReturn(null)

    marketService.getMarket("test")

    verify(opsForValue).set(eq("market:test"), any(), any())
}
```

## Test Coverage Configuration

### JaCoCo Configuration (build.gradle.kts)
```kotlin
plugins {
    jacoco
}

jacoco {
    toolVersion = "0.8.11"
}

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required.set(true)
        html.required.set(true)
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                minimum = "0.80".toBigDecimal()
            }
        }
    }
}

tasks.check {
    dependsOn(tasks.jacocoTestCoverageVerification)
}
```

## Common Testing Mistakes to Avoid

### ❌ WRONG: Testing Implementation Details
```kotlin
// Don't test internal state
assertThat(service.cache.size).isEqualTo(5)
```

### ✅ CORRECT: Test Behavior
```kotlin
// Test observable behavior
val result = service.getMarkets()
assertThat(result).hasSize(5)
```

### ❌ WRONG: No Test Isolation
```kotlin
// Tests depend on each other
@Test fun `creates user`() { /* creates user with id 1 */ }
@Test fun `updates same user`() { /* depends on previous test */ }
```

### ✅ CORRECT: Independent Tests
```kotlin
// Each test sets up its own data
@BeforeEach
fun setup() {
    testData = createTestData()
}

@Test
fun `creates user`() { /* ... */ }
@Test
fun `updates user`() { /* ... */ }
```

## Continuous Testing

### Watch Mode During Development
```bash
./gradlew test --continuous
# Tests run automatically on file changes
```

### Pre-Commit Hook
```bash
# Runs before every commit
./gradlew test && ./gradlew detekt
```

### CI/CD Integration (GitHub Actions)
```yaml
- name: Run Tests
  run: ./gradlew test jacocoTestReport

- name: Upload Coverage
  uses: codecov/codecov-action@v3
  with:
    files: build/reports/jacoco/test/jacocoTestReport.xml
```

## Best Practices

1. **Write Tests First** - Always TDD
2. **One Assert Per Test** - Focus on single behavior
3. **Descriptive Test Names** - Use backtick syntax with clear descriptions
4. **Given-When-Then** - Clear test structure
5. **Mock External Dependencies** - Isolate unit tests
6. **Test Edge Cases** - Null, empty, invalid, large data
7. **Test Error Paths** - Not just happy paths
8. **Keep Tests Fast** - Unit tests < 50ms each
9. **Clean Up After Tests** - Use @BeforeEach/@AfterEach
10. **Review Coverage Reports** - Identify gaps

## Success Metrics

- 80%+ code coverage achieved
- All tests passing (green)
- No skipped or disabled tests
- Fast test execution (< 30s for unit tests)
- E2E tests cover critical user flows
- Tests catch bugs before production

---

**Remember**: Tests are not optional. They are the safety net that enables confident refactoring, rapid development, and production reliability.

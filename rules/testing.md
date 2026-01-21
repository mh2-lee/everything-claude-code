# Testing Requirements

## Minimum Test Coverage: 80%

Test Types (ALL required):
1. **Unit Tests** - Individual functions, services, utilities (JUnit5/Kotest)
2. **Integration Tests** - API endpoints, database operations (@SpringBootTest)
3. **E2E Tests** - Critical user flows (optional: Selenium, Playwright)

## Test-Driven Development

MANDATORY workflow:
1. Write test first (RED)
2. Run test - it should FAIL
3. Write minimal implementation (GREEN)
4. Run test - it should PASS
5. Refactor (IMPROVE)
6. Verify coverage (80%+)

## JUnit5 + Kotlin Testing

### Unit Test Example

```kotlin
@ExtendWith(MockitoExtension::class)
class UserServiceTest {

    @Mock
    lateinit var userRepository: UserRepository

    @InjectMocks
    lateinit var userService: UserService

    @Test
    fun `should create user with valid data`() {
        // Given
        val request = CreateUserRequest("test@example.com", "John")
        val savedUser = User(1L, "test@example.com", "John")

        whenever(userRepository.save(any())).thenReturn(savedUser)

        // When
        val result = userService.create(request)

        // Then
        assertThat(result.email).isEqualTo("test@example.com")
        verify(userRepository).save(any())
    }

    @Test
    fun `should throw exception for duplicate email`() {
        // Given
        val request = CreateUserRequest("existing@example.com", "John")
        whenever(userRepository.existsByEmail(request.email)).thenReturn(true)

        // When & Then
        assertThrows<DuplicateEmailException> {
            userService.create(request)
        }
    }
}
```

### Integration Test Example

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
class UserControllerIntegrationTest {

    @Autowired
    lateinit var restTemplate: TestRestTemplate

    @Autowired
    lateinit var userRepository: UserRepository

    @BeforeEach
    fun setup() {
        userRepository.deleteAll()
    }

    @Test
    fun `POST users should create new user`() {
        // Given
        val request = CreateUserRequest("test@example.com", "John")

        // When
        val response = restTemplate.postForEntity(
            "/api/users",
            request,
            UserResponse::class.java
        )

        // Then
        assertThat(response.statusCode).isEqualTo(HttpStatus.CREATED)
        assertThat(response.body?.email).isEqualTo("test@example.com")
        assertThat(userRepository.count()).isEqualTo(1)
    }

    @Test
    fun `GET users should return all users`() {
        // Given
        userRepository.saveAll(listOf(
            User(email = "user1@test.com", name = "User 1"),
            User(email = "user2@test.com", name = "User 2")
        ))

        // When
        val response = restTemplate.getForEntity(
            "/api/users",
            Array<UserResponse>::class.java
        )

        // Then
        assertThat(response.statusCode).isEqualTo(HttpStatus.OK)
        assertThat(response.body).hasSize(2)
    }
}
```

### Repository Test with @DataJpaTest

```kotlin
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryTest {

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
    lateinit var userRepository: UserRepository

    @Test
    fun `should find user by email`() {
        // Given
        val user = userRepository.save(User(email = "test@example.com", name = "Test"))

        // When
        val found = userRepository.findByEmail("test@example.com")

        // Then
        assertThat(found).isNotNull
        assertThat(found?.id).isEqualTo(user.id)
    }
}
```

## Kotest Alternative (BDD Style)

```kotlin
class UserServiceSpec : BehaviorSpec({
    val userRepository = mockk<UserRepository>()
    val userService = UserService(userRepository)

    given("a valid user request") {
        val request = CreateUserRequest("test@example.com", "John")

        `when`("creating a new user") {
            every { userRepository.save(any()) } returns User(1L, "test@example.com", "John")

            then("should return created user") {
                val result = userService.create(request)
                result.email shouldBe "test@example.com"
            }
        }
    }

    given("an existing email") {
        val request = CreateUserRequest("existing@example.com", "John")

        `when`("creating user with duplicate email") {
            every { userRepository.existsByEmail(request.email) } returns true

            then("should throw DuplicateEmailException") {
                shouldThrow<DuplicateEmailException> {
                    userService.create(request)
                }
            }
        }
    }
})
```

## MockMvc for Controller Testing

```kotlin
@WebMvcTest(UserController::class)
class UserControllerTest {

    @Autowired
    lateinit var mockMvc: MockMvc

    @MockBean
    lateinit var userService: UserService

    @Test
    fun `should return 400 for invalid email`() {
        val invalidRequest = """{"email": "invalid", "name": "John"}"""

        mockMvc.perform(
            post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(invalidRequest)
        )
            .andExpect(status().isBadRequest)
            .andExpect(jsonPath("$.errors[0].field").value("email"))
    }

    @Test
    fun `should return 200 with user list`() {
        whenever(userService.findAll()).thenReturn(listOf(
            UserDto(1L, "user@test.com", "User")
        ))

        mockMvc.perform(get("/api/users"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$[0].email").value("user@test.com"))
    }
}
```

## Running Tests

```bash
# Run all tests
./gradlew test

# Run with coverage report
./gradlew test jacocoTestReport

# Run specific test class
./gradlew test --tests "UserServiceTest"

# Run specific test method
./gradlew test --tests "UserServiceTest.should create user with valid data"

# Integration tests only
./gradlew integrationTest
```

## Troubleshooting Test Failures

1. Use **tdd-guide** agent
2. Check test isolation (use @BeforeEach cleanup)
3. Verify mocks are correctly configured
4. Check Testcontainers are running
5. Fix implementation, not tests (unless tests are wrong)

## Agent Support

- **tdd-guide** - Use PROACTIVELY for new features, enforces write-tests-first
- **e2e-runner** - E2E testing specialist (Selenium/Playwright)

---
name: security-review
description: Use this skill when adding authentication, handling user input, working with secrets, creating API endpoints, or implementing payment/sensitive features. Provides comprehensive security checklist and patterns for Kotlin/Spring Boot.
---

# Security Review Skill

This skill ensures all Kotlin/Spring Boot code follows security best practices and identifies potential vulnerabilities.

## When to Activate

- Implementing authentication or authorization
- Handling user input or file uploads
- Creating new API endpoints
- Working with secrets or credentials
- Implementing payment features
- Storing or transmitting sensitive data
- Integrating third-party APIs

## Security Checklist

### 1. Secrets Management

#### ❌ NEVER Do This
```kotlin
val apiKey = "sk-proj-xxxxx"  // Hardcoded secret
val dbPassword = "password123" // In source code
```

#### ✅ ALWAYS Do This
```kotlin
// Using @Value
@Value("\${openai.api-key}")
private lateinit var apiKey: String

// Or ConfigurationProperties (recommended)
@ConfigurationProperties(prefix = "app.secrets")
data class SecretsProperties(
    val openaiApiKey: String,
    val databaseUrl: String
)

// Verify secrets exist at startup
@PostConstruct
fun validateSecrets() {
    require(apiKey.isNotBlank()) { "openai.api-key not configured" }
}
```

#### Verification Steps
- [ ] No hardcoded API keys, tokens, or passwords
- [ ] All secrets in environment variables or Vault
- [ ] `application-local.yml` in .gitignore
- [ ] No secrets in git history
- [ ] Production secrets in hosting platform or Kubernetes secrets

### 2. Input Validation

#### Always Validate User Input
```kotlin
// Define validation with Jakarta Bean Validation
data class CreateUserRequest(
    @field:Email(message = "Invalid email format")
    @field:NotBlank(message = "Email is required")
    val email: String,

    @field:NotBlank(message = "Name is required")
    @field:Size(min = 1, max = 100, message = "Name must be 1-100 characters")
    val name: String,

    @field:Min(0, message = "Age must be positive")
    @field:Max(150, message = "Age must be realistic")
    val age: Int
)

// Validate in controller
@PostMapping("/users")
fun createUser(@Valid @RequestBody request: CreateUserRequest): ResponseEntity<User> {
    return ResponseEntity.ok(userService.create(request))
}

// Global exception handler for validation errors
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidationException(ex: MethodArgumentNotValidException): ResponseEntity<ErrorResponse> {
        val errors = ex.bindingResult.fieldErrors.map {
            FieldError(it.field, it.defaultMessage ?: "Invalid value")
        }
        return ResponseEntity.badRequest().body(ErrorResponse(
            success = false,
            errors = errors
        ))
    }
}
```

#### File Upload Validation
```kotlin
@PostMapping("/upload")
fun uploadFile(@RequestParam("file") file: MultipartFile): ResponseEntity<UploadResponse> {
    // Size check (5MB max)
    val maxSize = 5 * 1024 * 1024L
    if (file.size > maxSize) {
        throw FileTooLargeException("File too large (max 5MB)")
    }

    // Type check
    val allowedTypes = listOf("image/jpeg", "image/png", "image/gif")
    if (file.contentType !in allowedTypes) {
        throw InvalidFileTypeException("Invalid file type")
    }

    // Extension check
    val allowedExtensions = listOf(".jpg", ".jpeg", ".png", ".gif")
    val extension = file.originalFilename?.lowercase()?.let {
        it.substring(it.lastIndexOf("."))
    }
    if (extension !in allowedExtensions) {
        throw InvalidFileExtensionException("Invalid file extension")
    }

    return ResponseEntity.ok(fileService.store(file))
}
```

#### Verification Steps
- [ ] All user inputs validated with @Valid annotations
- [ ] File uploads restricted (size, type, extension)
- [ ] No direct use of user input in queries
- [ ] Whitelist validation (not blacklist)
- [ ] Error messages don't leak sensitive info

### 3. SQL Injection Prevention

#### ❌ NEVER Concatenate SQL
```kotlin
// DANGEROUS - SQL Injection vulnerability
@Query("SELECT * FROM users WHERE email = '$email'", nativeQuery = true)
fun findByEmail(email: String): User?

// Also dangerous
val query = "SELECT * FROM users WHERE email = '${userEmail}'"
entityManager.createNativeQuery(query)
```

#### ✅ ALWAYS Use Parameterized Queries
```kotlin
// Safe - Spring Data JPA methods
interface UserRepository : JpaRepository<User, Long> {
    fun findByEmail(email: String): User?
}

// Safe - parameterized JPQL query
@Query("SELECT u FROM User u WHERE u.email = :email")
fun findByEmail(@Param("email") email: String): User?

// Safe - native query with parameters
@Query("SELECT * FROM users WHERE email = :email", nativeQuery = true)
fun findByEmailNative(@Param("email") email: String): User?

// Safe - Criteria API
fun findByFilters(filters: UserFilters): List<User> {
    val cb = entityManager.criteriaBuilder
    val query = cb.createQuery(User::class.java)
    val root = query.from(User::class.java)

    val predicates = mutableListOf<Predicate>()
    filters.email?.let { predicates.add(cb.equal(root.get<String>("email"), it)) }

    query.where(*predicates.toTypedArray())
    return entityManager.createQuery(query).resultList
}
```

#### Verification Steps
- [ ] All database queries use parameterized queries
- [ ] No string concatenation in SQL
- [ ] Spring Data JPA methods used correctly
- [ ] Native queries use @Param annotations

### 4. Authentication & Authorization

#### JWT Token Handling
```kotlin
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
class SecurityConfig {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()) }
            .sessionManagement { it.sessionCreationPolicy(SessionCreationPolicy.STATELESS) }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/public/**").permitAll()
                    .requestMatchers("/api/admin/**").hasRole("ADMIN")
                    .requestMatchers("/api/**").authenticated()
                    .anyRequest().permitAll()
            }
            .oauth2ResourceServer { it.jwt { } }
        return http.build()
    }

    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder(12)
}
```

#### Authorization Checks
```kotlin
@Service
class UserService(private val userRepository: UserRepository) {

    // Method-level security
    @PreAuthorize("hasRole('ADMIN')")
    fun deleteUser(userId: Long) {
        userRepository.deleteById(userId)
    }

    // Check ownership
    @PreAuthorize("hasRole('ADMIN') or @userSecurity.isOwner(#userId)")
    fun getUser(userId: Long): User {
        return userRepository.findById(userId)
            .orElseThrow { EntityNotFoundException("User not found") }
    }
}

@Component
class UserSecurity {
    fun isOwner(userId: Long): Boolean {
        val authentication = SecurityContextHolder.getContext().authentication
        val currentUserId = (authentication.principal as UserDetails).username.toLong()
        return currentUserId == userId
    }
}
```

#### Verification Steps
- [ ] JWT tokens validated on every request
- [ ] @PreAuthorize on all sensitive methods
- [ ] Role-based access control implemented
- [ ] Session management configured
- [ ] Password encoder is BCrypt (strength 12+)

### 5. XSS Prevention

#### Sanitize HTML (Thymeleaf)
```html
<!-- ✅ SAFE: th:text escapes content -->
<p th:text="${userInput}"></p>

<!-- ❌ DANGEROUS: th:utext does NOT escape -->
<p th:utext="${userInput}"></p>

<!-- ✅ If HTML needed, sanitize server-side first -->
<p th:utext="${sanitizedHtml}"></p>
```

```kotlin
// Server-side HTML sanitization
import org.owasp.html.PolicyFactory
import org.owasp.html.Sanitizers

@Service
class HtmlSanitizer {
    private val policy: PolicyFactory = Sanitizers.FORMATTING.and(Sanitizers.LINKS)

    fun sanitize(html: String): String {
        return policy.sanitize(html)
    }
}
```

#### Content Security Policy
```kotlin
@Configuration
class SecurityHeadersConfig : WebMvcConfigurer {
    override fun addInterceptors(registry: InterceptorRegistry) {
        registry.addInterceptor(SecurityHeadersInterceptor())
    }
}

class SecurityHeadersInterceptor : HandlerInterceptor {
    override fun preHandle(request: HttpServletRequest, response: HttpServletResponse, handler: Any): Boolean {
        response.setHeader("Content-Security-Policy", "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'")
        response.setHeader("X-Content-Type-Options", "nosniff")
        response.setHeader("X-Frame-Options", "DENY")
        response.setHeader("X-XSS-Protection", "1; mode=block")
        return true
    }
}
```

#### Verification Steps
- [ ] User-provided HTML sanitized with OWASP Java HTML Sanitizer
- [ ] Thymeleaf th:text used (not th:utext)
- [ ] CSP headers configured
- [ ] No unvalidated dynamic content rendering

### 6. CSRF Protection

#### Spring Security CSRF (enabled by default for forms)
```kotlin
@Configuration
@EnableWebSecurity
class SecurityConfig {
    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { csrf ->
                // For APIs with JWT, often disabled
                csrf.ignoringRequestMatchers("/api/**")
                // For forms, use cookie-based token
                csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            }
        return http.build()
    }
}
```

#### Thymeleaf Form with CSRF
```html
<form th:action="@{/submit}" method="post">
    <!-- CSRF token automatically included by Thymeleaf -->
    <input type="text" name="data" />
    <button type="submit">Submit</button>
</form>
```

#### Verification Steps
- [ ] CSRF protection enabled for form submissions
- [ ] SameSite=Strict on all cookies
- [ ] API endpoints use JWT (stateless, no CSRF needed)

### 7. Rate Limiting

#### Rate Limiting with Bucket4j
```kotlin
@Configuration
class RateLimitConfig {
    @Bean
    fun rateLimitBuckets(): Map<String, Bucket> {
        return mapOf(
            "api" to Bucket.builder()
                .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
                .build(),
            "search" to Bucket.builder()
                .addLimit(Bandwidth.classic(10, Refill.intervally(10, Duration.ofMinutes(1))))
                .build()
        )
    }
}

@Component
class RateLimitInterceptor(
    private val buckets: Map<String, Bucket>
) : HandlerInterceptor {

    override fun preHandle(request: HttpServletRequest, response: HttpServletResponse, handler: Any): Boolean {
        val bucketKey = if (request.requestURI.contains("/search")) "search" else "api"
        val bucket = buckets[bucketKey]!!

        return if (bucket.tryConsume(1)) {
            true
        } else {
            response.status = 429
            response.writer.write("Too many requests")
            false
        }
    }
}
```

#### Resilience4j Rate Limiter
```kotlin
@RestController
class SearchController(private val searchService: SearchService) {

    @GetMapping("/api/search")
    @RateLimiter(name = "search", fallbackMethod = "searchFallback")
    fun search(@RequestParam q: String): ResponseEntity<SearchResponse> {
        return ResponseEntity.ok(searchService.search(q))
    }

    fun searchFallback(q: String, ex: Exception): ResponseEntity<SearchResponse> {
        return ResponseEntity.status(429).body(SearchResponse(
            success = false,
            error = "Too many search requests. Please try again later."
        ))
    }
}
```

#### Verification Steps
- [ ] Rate limiting on all API endpoints
- [ ] Stricter limits on expensive operations (search, AI calls)
- [ ] IP-based rate limiting
- [ ] User-based rate limiting (authenticated)

### 8. Sensitive Data Exposure

#### Logging
```kotlin
// ❌ WRONG: Logging sensitive data
logger.info("User login: email=$email, password=$password")
logger.info("Payment: cardNumber=$cardNumber, cvv=$cvv")

// ✅ CORRECT: Redact sensitive data
logger.info("User login: email=${email.maskEmail()}, userId=$userId")
logger.info("Payment: last4=${card.last4}, userId=$userId")

// Extension function for masking
fun String.maskEmail(): String {
    val atIndex = indexOf('@')
    if (atIndex <= 1) return "***@***"
    return "${first()}${"*".repeat(atIndex - 1)}${substring(atIndex)}"
}
```

#### Error Messages
```kotlin
// Global exception handler
@RestControllerAdvice
class GlobalExceptionHandler {

    private val logger = LoggerFactory.getLogger(javaClass)

    // ❌ WRONG: Exposing internal details
    @ExceptionHandler(Exception::class)
    fun handleException(ex: Exception): ResponseEntity<ErrorResponse> {
        return ResponseEntity.status(500).body(ErrorResponse(
            error = ex.message,  // Might expose internal details!
            stackTrace = ex.stackTraceToString()  // Never expose!
        ))
    }

    // ✅ CORRECT: Generic error messages
    @ExceptionHandler(Exception::class)
    fun handleException(ex: Exception): ResponseEntity<ErrorResponse> {
        logger.error("Internal error", ex)  // Log full details server-side
        return ResponseEntity.status(500).body(ErrorResponse(
            success = false,
            error = "An error occurred. Please try again."
        ))
    }
}
```

#### Verification Steps
- [ ] No passwords, tokens, or secrets in logs
- [ ] Error messages generic for users
- [ ] Detailed errors only in server logs
- [ ] No stack traces exposed to users

### 9. Dependency Security

#### Regular Updates
```bash
# Check for vulnerable dependencies
./gradlew dependencyCheckAnalyze

# View dependency tree
./gradlew dependencies

# Update dependencies
./gradlew dependencyUpdates
```

#### Gradle Configuration
```kotlin
// build.gradle.kts
plugins {
    id("org.owasp.dependencycheck") version "9.0.0"
    id("com.github.ben-manes.versions") version "0.50.0"
}

dependencyCheck {
    failBuildOnCVSS = 7.0f  // Fail on high severity
    suppressionFile = "owasp-suppressions.xml"
}
```

#### Verification Steps
- [ ] Dependencies up to date
- [ ] No known vulnerabilities (OWASP check clean)
- [ ] gradle.lockfile committed
- [ ] Dependabot enabled on GitHub
- [ ] Regular security updates

### 10. Database Security

#### JPA Entity Security
```kotlin
@Entity
@Table(name = "users")
@SQLDelete(sql = "UPDATE users SET deleted = true WHERE id = ?")
@Where(clause = "deleted = false")
data class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, unique = true)
    val email: String,

    @Column(nullable = false)
    @JsonIgnore  // Never serialize password
    val password: String,

    @Column(nullable = false)
    val deleted: Boolean = false
)
```

#### Pessimistic Locking for Financial Operations
```kotlin
interface AccountRepository : JpaRepository<Account, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    fun findByIdWithLock(@Param("id") id: Long): Account?
}

@Service
class TransferService(private val accountRepository: AccountRepository) {

    @Transactional
    fun transfer(fromId: Long, toId: Long, amount: BigDecimal) {
        val from = accountRepository.findByIdWithLock(fromId)
            ?: throw EntityNotFoundException("Source account not found")
        val to = accountRepository.findByIdWithLock(toId)
            ?: throw EntityNotFoundException("Target account not found")

        require(from.balance >= amount) { "Insufficient balance" }

        from.balance -= amount
        to.balance += amount

        accountRepository.saveAll(listOf(from, to))
    }
}
```

## Security Testing

### Automated Security Tests
```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class SecurityTest {

    @Autowired
    lateinit var restTemplate: TestRestTemplate

    @Test
    fun `unauthenticated request should return 401`() {
        val response = restTemplate.getForEntity("/api/protected", String::class.java)
        assertThat(response.statusCode).isEqualTo(HttpStatus.UNAUTHORIZED)
    }

    @Test
    fun `user without admin role should get 403 on admin endpoint`() {
        val headers = HttpHeaders().apply {
            setBearerAuth(userToken)
        }
        val entity = HttpEntity<String>(headers)
        val response = restTemplate.exchange("/api/admin", HttpMethod.GET, entity, String::class.java)
        assertThat(response.statusCode).isEqualTo(HttpStatus.FORBIDDEN)
    }

    @Test
    fun `invalid input should return 400`() {
        val request = CreateUserRequest(email = "not-an-email", name = "", age = -1)
        val response = restTemplate.postForEntity("/api/users", request, ErrorResponse::class.java)
        assertThat(response.statusCode).isEqualTo(HttpStatus.BAD_REQUEST)
    }
}
```

## Pre-Deployment Security Checklist

Before ANY production deployment:

- [ ] **Secrets**: No hardcoded secrets, all in env vars
- [ ] **Input Validation**: All user inputs validated with @Valid
- [ ] **SQL Injection**: All queries parameterized
- [ ] **XSS**: User content sanitized, th:text used
- [ ] **CSRF**: Protection enabled for forms
- [ ] **Authentication**: Spring Security configured
- [ ] **Authorization**: @PreAuthorize on sensitive methods
- [ ] **Rate Limiting**: Enabled on all endpoints
- [ ] **HTTPS**: Enforced in production
- [ ] **Security Headers**: CSP, X-Frame-Options configured
- [ ] **Error Handling**: No sensitive data in errors
- [ ] **Logging**: No sensitive data logged
- [ ] **Dependencies**: Up to date, no vulnerabilities
- [ ] **CORS**: Properly configured
- [ ] **File Uploads**: Validated (size, type)
- [ ] **Passwords**: BCrypt with strength 12+

## Resources

- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [OWASP Java HTML Sanitizer](https://github.com/OWASP/java-html-sanitizer)
- [Bucket4j Rate Limiting](https://github.com/bucket4j/bucket4j)

---

**Remember**: Security is not optional. One vulnerability can compromise the entire platform. When in doubt, err on the side of caution.

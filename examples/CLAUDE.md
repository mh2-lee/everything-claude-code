# Example Project CLAUDE.md

This is an example project-level CLAUDE.md file for Spring Boot + Kotlin projects. Place this in your project root.

## Project Overview

[Brief description of your project - what it does, tech stack]

- **Framework**: Spring Boot 3.x
- **Language**: Kotlin
- **Build Tool**: Gradle (Kotlin DSL)
- **Database**: PostgreSQL
- **Cache**: Redis

## Critical Rules

### 1. Code Organization

- Many small files over few large files
- High cohesion, low coupling
- 200-400 lines typical, 800 max per file
- Organize by feature/domain, not by layer

```
src/main/kotlin/com/example/
├── user/
│   ├── UserController.kt
│   ├── UserService.kt
│   ├── UserRepository.kt
│   └── User.kt
├── order/
│   └── ...
└── common/
    ├── config/
    └── exception/
```

### 2. Code Style

- Use `val` over `var` (immutability)
- Use data class `copy()` for updates
- No println - use SLF4J logger
- Proper error handling with exception classes
- Input validation with Jakarta Validation

### 3. Testing

- TDD: Write tests first
- 80% minimum coverage
- Unit tests with JUnit5 + Mockito
- Integration tests with @SpringBootTest
- Use Testcontainers for database tests

### 4. Security

- No hardcoded secrets
- Environment variables for sensitive data (application.yml with `${VAR}`)
- Validate all user inputs
- Use Spring Security for auth
- Parameterized queries (JPA handles this)

## File Structure

```
src/
├── main/
│   ├── kotlin/com/example/
│   │   ├── Application.kt
│   │   ├── user/
│   │   ├── order/
│   │   └── common/
│   └── resources/
│       ├── application.yml
│       ├── application-local.yml
│       └── db/migration/        # Flyway migrations
└── test/
    └── kotlin/com/example/
```

## Key Patterns

### API Response Format

```kotlin
data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val error: ErrorDetail? = null
)
```

### Error Handling

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {
    @ExceptionHandler(EntityNotFoundException::class)
    fun handleNotFound(e: EntityNotFoundException): ResponseEntity<ApiResponse<Nothing>> {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ApiResponse(success = false, error = ErrorDetail("NOT_FOUND", e.message)))
    }
}
```

### Service Pattern

```kotlin
@Service
@Transactional(readOnly = true)
class UserService(private val userRepository: UserRepository) {

    fun findById(id: Long): User =
        userRepository.findByIdOrNull(id)
            ?: throw EntityNotFoundException("User not found: $id")

    @Transactional
    fun create(request: CreateUserRequest): User {
        val user = User(name = request.name, email = request.email)
        return userRepository.save(user)
    }
}
```

## Environment Variables

```yaml
# application.yml
spring:
  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/myapp}
    username: ${DATABASE_USERNAME:postgres}
    password: ${DATABASE_PASSWORD:postgres}

  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}

jwt:
  secret: ${JWT_SECRET}
  expiration: ${JWT_EXPIRATION:86400000}
```

## Available Commands

- `/tdd` - Test-driven development workflow
- `/plan` - Create implementation plan
- `/code-review` - Review code quality
- `/build-fix` - Fix Gradle build errors

## Build Commands

```bash
# Build
./gradlew build

# Run tests
./gradlew test

# Run with coverage
./gradlew test jacocoTestReport

# Run application
./gradlew bootRun

# Build Docker image
./gradlew bootBuildImage
```

## Git Workflow

- Conventional commits: `feat:`, `fix:`, `refactor:`, `docs:`, `test:`
- Never commit to main directly
- PRs require review
- All tests must pass before merge

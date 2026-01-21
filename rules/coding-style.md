# Coding Style

## Immutability (CRITICAL)

ALWAYS use immutable data, NEVER mutate:

```kotlin
// WRONG: Mutation
fun updateUser(user: User, name: String): User {
    user.name = name  // MUTATION!
    return user
}

// CORRECT: Immutability with data class copy
fun updateUser(user: User, name: String): User {
    return user.copy(name = name)
}

// CORRECT: Use val, not var
val users = listOf(user1, user2)  // Immutable list
val mutableUsers = mutableListOf(user1)  // Only when necessary
```

## File Organization

MANY SMALL FILES > FEW LARGE FILES:
- High cohesion, low coupling
- 200-400 lines typical, 800 max
- One class per file (Kotlin convention)
- Organize by feature/domain, not by layer

```
src/main/kotlin/com/example/
├── user/
│   ├── UserController.kt
│   ├── UserService.kt
│   ├── UserRepository.kt
│   └── User.kt
├── order/
│   ├── OrderController.kt
│   └── ...
```

## Error Handling

ALWAYS handle errors comprehensively:

```kotlin
fun riskyOperation(): Result<Data> {
    return try {
        val result = performOperation()
        Result.success(result)
    } catch (e: Exception) {
        logger.error("Operation failed", e)
        Result.failure(ApiException("User-friendly message", e))
    }
}

// Or use Spring's exception handling
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException::class)
    fun handleNotFound(e: EntityNotFoundException): ResponseEntity<ErrorResponse> {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse(e.message ?: "Resource not found"))
    }
}
```

## Input Validation

ALWAYS validate user input:

```kotlin
// Using Jakarta Validation (Bean Validation)
data class CreateUserRequest(
    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Invalid email format")
    val email: String,

    @field:Min(0, message = "Age must be positive")
    @field:Max(150, message = "Age must be under 150")
    val age: Int
)

@PostMapping("/users")
fun createUser(@Valid @RequestBody request: CreateUserRequest): ResponseEntity<User> {
    return ResponseEntity.ok(userService.create(request))
}
```

## Kotlin Idioms

USE Kotlin idioms properly:

```kotlin
// Use data classes for DTOs
data class UserDto(
    val id: Long,
    val name: String,
    val email: String
)

// Use sealed classes for state
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String) : Result<Nothing>()
}

// Use extension functions
fun String.toSlug(): String =
    this.lowercase().replace(" ", "-")

// Use scope functions appropriately
val user = userRepository.findById(id)?.let { user ->
    user.copy(lastLogin = Instant.now())
} ?: throw EntityNotFoundException("User not found")

// Prefer expression body for simple functions
fun double(x: Int): Int = x * 2
```

## Null Safety

LEVERAGE Kotlin's null safety:

```kotlin
// Use nullable types explicitly
fun findUser(id: Long): User?  // May return null

// Use safe calls
val length = user?.name?.length ?: 0

// Use require/check for preconditions
fun processOrder(order: Order) {
    require(order.items.isNotEmpty()) { "Order must have items" }
    check(order.status == OrderStatus.PENDING) { "Order already processed" }
}

// Avoid !! operator - use proper null handling instead
// BAD: user!!.name
// GOOD: user?.name ?: "Unknown"
```

## Code Quality Checklist

Before marking work complete:
- [ ] Code is readable and well-named
- [ ] Functions are small (<50 lines)
- [ ] Files are focused (<800 lines)
- [ ] No deep nesting (>4 levels)
- [ ] Proper error handling
- [ ] No println statements (use logger)
- [ ] No hardcoded values
- [ ] Immutable patterns used (val, data class copy)
- [ ] Null safety properly handled
- [ ] Kotlin idioms followed

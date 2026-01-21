---
name: frontend-patterns
description: Server-side rendering patterns with Thymeleaf, HTMX integration, and API response patterns for Spring Boot applications.
---

# Frontend & API Response Patterns

Patterns for server-side rendering with Thymeleaf and API response design in Spring Boot applications.

## Thymeleaf Template Patterns

### Base Layout Template

```html
<!-- templates/layout/base.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      th:fragment="layout(title, content)">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title th:replace="${title}">Default Title</title>
    <link rel="stylesheet" th:href="@{/css/main.css}">
    <script src="https://unpkg.com/htmx.org@1.9.10"></script>
</head>
<body>
    <nav th:replace="~{layout/nav :: nav}"></nav>

    <main class="container">
        <div th:replace="${content}">Content</div>
    </main>

    <footer th:replace="~{layout/footer :: footer}"></footer>

    <script th:src="@{/js/main.js}"></script>
</body>
</html>
```

### Page Template with Layout

```html
<!-- templates/market/list.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout/base :: layout(~{::title}, ~{::content})}">
<head>
    <title>Markets</title>
</head>
<body>
    <div th:fragment="content">
        <h1>Markets</h1>

        <!-- Search form -->
        <form th:action="@{/markets}" method="get" class="search-form">
            <input type="text" name="query" th:value="${query}" placeholder="Search markets...">
            <button type="submit">Search</button>
        </form>

        <!-- Market list -->
        <div class="market-list" id="market-list">
            <div th:each="market : ${markets}" class="market-card">
                <h3 th:text="${market.name}">Market Name</h3>
                <p th:text="${market.description}">Description</p>
                <span th:text="${market.status}"
                      th:class="'status status-' + ${market.status.name().toLowerCase()}">
                    Status
                </span>
                <a th:href="@{/markets/{id}(id=${market.id})}">View Details</a>
            </div>

            <div th:if="${#lists.isEmpty(markets)}" class="empty-state">
                <p>No markets found.</p>
            </div>
        </div>

        <!-- Pagination -->
        <nav th:if="${totalPages > 1}" class="pagination">
            <a th:if="${currentPage > 0}"
               th:href="@{/markets(page=${currentPage - 1}, query=${query})}">
                Previous
            </a>
            <span th:each="page : ${#numbers.sequence(0, totalPages - 1)}"
                  th:class="${page == currentPage} ? 'active'">
                <a th:href="@{/markets(page=${page}, query=${query})}"
                   th:text="${page + 1}">1</a>
            </span>
            <a th:if="${currentPage < totalPages - 1}"
               th:href="@{/markets(page=${currentPage + 1}, query=${query})}">
                Next
            </a>
        </nav>
    </div>
</body>
</html>
```

### Form with Validation

```html
<!-- templates/market/form.html -->
<form th:action="@{/markets}" th:object="${marketForm}" method="post">
    <div class="form-group" th:classappend="${#fields.hasErrors('name')} ? 'has-error'">
        <label for="name">Name</label>
        <input type="text" id="name" th:field="*{name}" required>
        <span th:if="${#fields.hasErrors('name')}"
              th:errors="*{name}"
              class="error-message">
        </span>
    </div>

    <div class="form-group" th:classappend="${#fields.hasErrors('description')} ? 'has-error'">
        <label for="description">Description</label>
        <textarea id="description" th:field="*{description}" rows="4"></textarea>
        <span th:if="${#fields.hasErrors('description')}"
              th:errors="*{description}"
              class="error-message">
        </span>
    </div>

    <div class="form-group">
        <label for="status">Status</label>
        <select id="status" th:field="*{status}">
            <option th:each="status : ${T(com.example.MarketStatus).values()}"
                    th:value="${status}"
                    th:text="${status.displayName}">
            </option>
        </select>
    </div>

    <!-- CSRF token (auto-included by Spring Security) -->
    <button type="submit">Create Market</button>
</form>
```

## HTMX Integration Patterns

### Partial Updates

```html
<!-- List with HTMX partial loading -->
<div id="market-list">
    <div th:each="market : ${markets}"
         th:fragment="market-item"
         class="market-card"
         th:id="'market-' + ${market.id}">
        <h3 th:text="${market.name}">Name</h3>
        <button hx-delete th:attr="hx-delete=@{/markets/{id}(id=${market.id})}"
                hx-target th:attr="hx-target='#market-' + ${market.id}"
                hx-swap="outerHTML"
                hx-confirm="Are you sure?">
            Delete
        </button>
    </div>
</div>

<!-- Infinite scroll -->
<div hx-get="/markets?page=1"
     hx-trigger="revealed"
     hx-swap="afterend">
    Loading more...
</div>
```

### Live Search

```html
<input type="search"
       name="query"
       hx-get="/markets/search"
       hx-trigger="keyup changed delay:300ms"
       hx-target="#search-results"
       hx-indicator="#search-spinner"
       placeholder="Search markets...">

<span id="search-spinner" class="htmx-indicator">Searching...</span>

<div id="search-results">
    <!-- Results loaded here -->
</div>
```

### Form Submission with HTMX

```html
<form hx-post="/markets"
      hx-target="#market-list"
      hx-swap="beforeend"
      hx-on::after-request="this.reset()">
    <input type="text" name="name" required>
    <textarea name="description"></textarea>
    <button type="submit">Add Market</button>
</form>
```

## Controller Patterns for Views

### View Controller

```kotlin
@Controller
@RequestMapping("/markets")
class MarketViewController(
    private val marketService: MarketService
) {
    @GetMapping
    fun list(
        @RequestParam(defaultValue = "") query: String,
        @RequestParam(defaultValue = "0") page: Int,
        model: Model
    ): String {
        val pageable = PageRequest.of(page, 20)
        val markets = if (query.isBlank()) {
            marketService.findAll(pageable)
        } else {
            marketService.search(query, pageable)
        }

        model.addAttribute("markets", markets.content)
        model.addAttribute("currentPage", page)
        model.addAttribute("totalPages", markets.totalPages)
        model.addAttribute("query", query)

        return "market/list"
    }

    @GetMapping("/{id}")
    fun detail(@PathVariable id: Long, model: Model): String {
        val market = marketService.findById(id)
        model.addAttribute("market", market)
        return "market/detail"
    }

    @GetMapping("/new")
    fun createForm(model: Model): String {
        model.addAttribute("marketForm", MarketForm())
        return "market/form"
    }

    @PostMapping
    fun create(
        @Valid @ModelAttribute("marketForm") form: MarketForm,
        bindingResult: BindingResult,
        redirectAttributes: RedirectAttributes
    ): String {
        if (bindingResult.hasErrors()) {
            return "market/form"
        }

        val market = marketService.create(form.toRequest())
        redirectAttributes.addFlashAttribute("message", "Market created successfully")
        return "redirect:/markets/${market.id}"
    }

    // HTMX partial endpoint
    @GetMapping("/search")
    fun search(
        @RequestParam query: String,
        model: Model
    ): String {
        val markets = marketService.search(query, PageRequest.of(0, 10))
        model.addAttribute("markets", markets.content)
        return "market/list :: market-item"  // Return fragment only
    }
}
```

## API Response Patterns

### Standard Response Wrapper

```kotlin
data class ApiResponse<T>(
    val success: Boolean,
    val data: T? = null,
    val error: ErrorDetail? = null,
    val meta: Meta? = null
)

data class ErrorDetail(
    val code: String,
    val message: String,
    val details: List<FieldError>? = null
)

data class Meta(
    val page: Int? = null,
    val size: Int? = null,
    val totalElements: Long? = null,
    val totalPages: Int? = null
)

// Extension functions for easy response creation
fun <T> T.toSuccessResponse(): ApiResponse<T> = ApiResponse(success = true, data = this)

fun <T> Page<T>.toPagedResponse(): ApiResponse<List<T>> = ApiResponse(
    success = true,
    data = this.content,
    meta = Meta(
        page = this.number,
        size = this.size,
        totalElements = this.totalElements,
        totalPages = this.totalPages
    )
)
```

### REST API Controller

```kotlin
@RestController
@RequestMapping("/api/markets")
class MarketApiController(
    private val marketService: MarketService
) {
    @GetMapping
    fun findAll(
        @RequestParam(defaultValue = "0") page: Int,
        @RequestParam(defaultValue = "20") size: Int
    ): ApiResponse<List<MarketResponse>> {
        val markets = marketService.findAll(PageRequest.of(page, size))
        return markets.map { it.toResponse() }.toPagedResponse()
    }

    @GetMapping("/{id}")
    fun findById(@PathVariable id: Long): ApiResponse<MarketResponse> {
        val market = marketService.findById(id)
        return market.toResponse().toSuccessResponse()
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun create(@Valid @RequestBody request: CreateMarketRequest): ApiResponse<MarketResponse> {
        val market = marketService.create(request)
        return market.toResponse().toSuccessResponse()
    }

    @PutMapping("/{id}")
    fun update(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateMarketRequest
    ): ApiResponse<MarketResponse> {
        val market = marketService.update(id, request)
        return market.toResponse().toSuccessResponse()
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun delete(@PathVariable id: Long) {
        marketService.delete(id)
    }
}
```

### DTO Mapping

```kotlin
// Response DTO
data class MarketResponse(
    val id: Long,
    val name: String,
    val description: String?,
    val status: String,
    val volume: Long,
    val createdAt: Instant,
    val updatedAt: Instant
)

// Extension function for mapping
fun Market.toResponse(): MarketResponse = MarketResponse(
    id = this.id,
    name = this.name,
    description = this.description,
    status = this.status.name,
    volume = this.volume,
    createdAt = this.createdAt,
    updatedAt = this.updatedAt
)

// Request DTO with validation
data class CreateMarketRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(max = 200, message = "Name must be under 200 characters")
    val name: String,

    @field:Size(max = 2000, message = "Description must be under 2000 characters")
    val description: String?
)

// Form DTO for Thymeleaf
data class MarketForm(
    var name: String = "",
    var description: String = "",
    var status: MarketStatus = MarketStatus.PENDING
) {
    fun toRequest() = CreateMarketRequest(name, description)
}
```

## Error Response Patterns

### Consistent Error Format

```kotlin
@RestControllerAdvice
class ApiExceptionHandler {

    @ExceptionHandler(EntityNotFoundException::class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    fun handleNotFound(e: EntityNotFoundException): ApiResponse<Nothing> {
        return ApiResponse(
            success = false,
            error = ErrorDetail(
                code = "NOT_FOUND",
                message = e.message ?: "Resource not found"
            )
        )
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleValidation(e: MethodArgumentNotValidException): ApiResponse<Nothing> {
        val fieldErrors = e.bindingResult.fieldErrors.map { error ->
            FieldError(error.field, error.defaultMessage ?: "Invalid value")
        }

        return ApiResponse(
            success = false,
            error = ErrorDetail(
                code = "VALIDATION_ERROR",
                message = "Request validation failed",
                details = fieldErrors
            )
        )
    }
}
```

## Static Resources Configuration

```kotlin
@Configuration
class WebConfig : WebMvcConfigurer {

    override fun addResourceHandlers(registry: ResourceHandlerRegistry) {
        registry.addResourceHandler("/static/**")
            .addResourceLocations("classpath:/static/")
            .setCacheControl(CacheControl.maxAge(365, TimeUnit.DAYS))

        registry.addResourceHandler("/webjars/**")
            .addResourceLocations("classpath:/META-INF/resources/webjars/")
    }

    override fun addViewControllers(registry: ViewControllerRegistry) {
        registry.addViewController("/").setViewName("redirect:/markets")
        registry.addViewController("/login").setViewName("auth/login")
    }
}
```

**Remember**: Choose between server-side rendering (Thymeleaf + HTMX) for traditional web apps or pure REST APIs for SPA/mobile backends. Both patterns work well with Spring Boot.

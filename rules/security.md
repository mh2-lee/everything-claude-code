# Security Guidelines

## Mandatory Security Checks

Before ANY commit:
- [ ] No hardcoded secrets (API keys, passwords, tokens)
- [ ] All user inputs validated (@Valid, @NotBlank, etc.)
- [ ] SQL injection prevention (parameterized queries, Spring Data JPA)
- [ ] XSS prevention (Thymeleaf th:text, sanitized HTML)
- [ ] CSRF protection enabled (Spring Security)
- [ ] Authentication/authorization verified (@PreAuthorize)
- [ ] Rate limiting on all endpoints (Bucket4j, Resilience4j)
- [ ] Error messages don't leak sensitive data

## Secret Management

```kotlin
// NEVER: Hardcoded secrets
val apiKey = "sk-proj-xxxxx"

// ALWAYS: Environment variables via Spring
@Value("\${openai.api-key}")
private lateinit var apiKey: String

// Or with ConfigurationProperties
@ConfigurationProperties(prefix = "app.secrets")
data class SecretsProperties(
    val openaiApiKey: String,
    val jwtSecret: String
)

// Validate at startup
@PostConstruct
fun validateSecrets() {
    require(apiKey.isNotBlank()) { "openai.api-key not configured" }
}
```

## Security Response Protocol

If security issue found:
1. STOP immediately
2. Use **security-reviewer** agent
3. Fix CRITICAL issues before continuing
4. Rotate any exposed secrets
5. Review entire codebase for similar issues

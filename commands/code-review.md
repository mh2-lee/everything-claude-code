# Code Review

Comprehensive security and quality review of uncommitted changes:

1. Get changed files: `git diff --name-only HEAD`

2. For each changed file, check for:

**Security Issues (CRITICAL):**
- Hardcoded credentials, API keys, tokens
- SQL injection vulnerabilities (raw queries)
- Missing input validation
- Insecure dependencies
- Path traversal risks
- Secrets in application.yml without `${ENV_VAR}`

**Code Quality (HIGH):**
- Functions > 50 lines
- Files > 800 lines
- Nesting depth > 4 levels
- Missing error handling
- `println` or `System.out` statements
- TODO/FIXME comments
- Missing KDoc for public APIs
- Using `var` when `val` would work
- Using `!!` instead of proper null handling

**Kotlin Best Practices (MEDIUM):**
- Mutation patterns (use `copy()` for data classes)
- Missing `@Transactional` on write operations
- Not using Kotlin idioms (let, apply, also, run)
- Missing tests for new code
- Unused imports or variables

**Spring Boot Specific:**
- Missing `@Valid` on request body
- Missing `@Transactional(readOnly = true)` for read operations
- Circular dependencies
- N+1 query issues
- Missing logging on service methods

3. Generate report with:
   - Severity: CRITICAL, HIGH, MEDIUM, LOW
   - File location and line numbers
   - Issue description
   - Suggested fix

4. Block commit if CRITICAL or HIGH issues found

## Example Output

```
╔═══════════════════════════════════════════════════════════════╗
║                    CODE REVIEW REPORT                         ║
╠═══════════════════════════════════════════════════════════════╣
║ Files Reviewed: 5                                             ║
║ Issues Found: 3                                               ║
╠═══════════════════════════════════════════════════════════════╣
║ CRITICAL: 0                                                   ║
║ HIGH: 1                                                       ║
║ MEDIUM: 2                                                     ║
║ LOW: 0                                                        ║
╚═══════════════════════════════════════════════════════════════╝

HIGH: UserService.kt:45
  Issue: Missing @Transactional on write operation
  Fix: Add @Transactional annotation to create() method

MEDIUM: OrderController.kt:23
  Issue: Using println instead of logger
  Fix: Replace with logger.info()

MEDIUM: MarketService.kt:67
  Issue: Using !! operator
  Fix: Use ?.let { } ?: throw EntityNotFoundException()
```

Never approve code with security vulnerabilities!

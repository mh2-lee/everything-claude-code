# Refactor Clean

Safely identify and remove dead code with test verification:

1. Run dead code analysis tools:
   - IntelliJ IDEA: Analyze → Inspect Code (unused declarations)
   - Detekt: Find unused private members and parameters
   - Gradle: `./gradlew dependencies` for unused dependencies

2. Generate comprehensive report in `.reports/dead-code-analysis.md`

3. Categorize findings by severity:
   - SAFE: Test files, unused utilities
   - CAUTION: API endpoints, services
   - DANGER: Config files, main entry points, @Bean definitions

4. Propose safe deletions only

5. Before each deletion:
   - Run full test suite: `./gradlew test`
   - Verify tests pass
   - Apply change
   - Re-run tests
   - Rollback if tests fail

6. Show summary of cleaned items

## Useful Commands

```bash
# Run all tests before refactoring
./gradlew test

# Check for unused dependencies
./gradlew dependencies --configuration compileClasspath

# Run Detekt for code analysis
./gradlew detekt

# Find unused imports (via IntelliJ or ktlint)
./gradlew ktlintCheck
```

## Detekt Configuration

```yaml
# detekt.yml
style:
  UnusedPrivateMember:
    active: true
  UnusedPrivateClass:
    active: true
  UnusedImports:
    active: true
```

## Common Dead Code Patterns in Kotlin/Spring

- Unused `@Service`, `@Component` classes
- Private functions never called
- Unused constructor parameters
- Dead `@Bean` definitions
- Commented-out code blocks
- Unused `data class` properties

Never delete code without running tests first!

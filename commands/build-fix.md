# Build and Fix

Incrementally fix Kotlin compilation and Gradle build errors:

1. Run build: `./gradlew build` or `./gradlew compileKotlin`

2. Parse error output:
   - Group by file
   - Sort by severity (errors before warnings)

3. For each error:
   - Show error context (5 lines before/after)
   - Explain the issue
   - Propose fix
   - Apply fix
   - Re-run build
   - Verify error resolved

4. Common Kotlin/Spring errors:
   - **Unresolved reference**: Missing import or dependency
   - **Type mismatch**: Check nullable types, use `?.` or `!!`
   - **Bean not found**: Check `@Component`, `@Service`, `@Repository` annotations
   - **Circular dependency**: Use `@Lazy` or restructure
   - **Missing @Transactional**: Add to service methods modifying data

5. Stop if:
   - Fix introduces new errors
   - Same error persists after 3 attempts
   - User requests pause

6. Show summary:
   - Errors fixed
   - Errors remaining
   - New errors introduced

## Useful Commands

```bash
# Full build
./gradlew build

# Compile only (faster)
./gradlew compileKotlin

# Compile tests
./gradlew compileTestKotlin

# Clean and rebuild
./gradlew clean build

# Check for dependency issues
./gradlew dependencies

# Run with stacktrace for detailed errors
./gradlew build --stacktrace
```

Fix one error at a time for safety!

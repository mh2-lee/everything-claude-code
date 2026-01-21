# Test Coverage

Analyze test coverage and generate missing tests:

1. Run tests with coverage: `./gradlew test jacocoTestReport`

2. Analyze coverage report (`build/reports/jacoco/test/html/index.html`)

3. Identify files below 80% coverage threshold

4. For each under-covered file:
   - Analyze untested code paths
   - Generate unit tests for functions
   - Generate integration tests for APIs
   - Generate E2E tests for critical flows

5. Verify new tests pass

6. Show before/after coverage metrics

7. Ensure project reaches 80%+ overall coverage

## Commands

```bash
# Run tests with coverage
./gradlew test jacocoTestReport

# View HTML report
open build/reports/jacoco/test/html/index.html

# Check coverage in CI (fails if below threshold)
./gradlew jacocoTestCoverageVerification
```

## Gradle Configuration

```kotlin
// build.gradle.kts
plugins {
    jacoco
}

jacoco {
    toolVersion = "0.8.11"
}

tasks.jacocoTestReport {
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

## Focus Areas

- Happy path scenarios
- Error handling (exceptions, edge cases)
- Edge cases (null, empty, boundary values)
- Boundary conditions
- All branches in conditional logic

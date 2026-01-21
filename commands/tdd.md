---
description: Enforce test-driven development workflow. Scaffold interfaces, generate tests FIRST, then implement minimal code to pass. Ensure 80%+ coverage.
---

# TDD Command

This command invokes the **tdd-guide** agent to enforce test-driven development methodology.

## What This Command Does

1. **Scaffold Interfaces** - Define types/interfaces first
2. **Generate Tests First** - Write failing tests (RED)
3. **Implement Minimal Code** - Write just enough to pass (GREEN)
4. **Refactor** - Improve code while keeping tests green (REFACTOR)
5. **Verify Coverage** - Ensure 80%+ test coverage

## When to Use

Use `/tdd` when:
- Implementing new features
- Adding new functions/services
- Fixing bugs (write test that reproduces bug first)
- Refactoring existing code
- Building critical business logic

## How It Works

The tdd-guide agent will:

1. **Define interfaces** for inputs/outputs
2. **Write tests that will FAIL** (because code doesn't exist yet)
3. **Run tests** and verify they fail for the right reason
4. **Write minimal implementation** to make tests pass
5. **Run tests** and verify they pass
6. **Refactor** code while keeping tests green
7. **Check coverage** and add more tests if below 80%

## TDD Cycle

```
RED → GREEN → REFACTOR → REPEAT

RED:      Write a failing test
GREEN:    Write minimal code to pass
REFACTOR: Improve code, keep tests passing
REPEAT:   Next feature/scenario
```

## Example Usage

```
User: /tdd I need a function to calculate market liquidity score

Agent (tdd-guide):
# TDD Session: Market Liquidity Score Calculator

## Step 1: Define Interface (SCAFFOLD)
```

```kotlin
// src/main/kotlin/com/example/market/LiquidityCalculator.kt
package com.example.market

import java.time.Instant

data class MarketData(
    val totalVolume: Long,
    val bidAskSpread: Double,
    val activeTraders: Int,
    val lastTradeTime: Instant
)

fun calculateLiquidityScore(market: MarketData): Double {
    // TODO: Implementation
    throw NotImplementedError("Not implemented")
}
```

## Step 2: Write Failing Test (RED)

```kotlin
// src/test/kotlin/com/example/market/LiquidityCalculatorTest.kt
package com.example.market

import org.junit.jupiter.api.Test
import org.junit.jupiter.api.assertThrows
import org.assertj.core.api.Assertions.assertThat
import java.time.Instant

class LiquidityCalculatorTest {

    @Test
    fun `should return high score for liquid market`() {
        // Given
        val market = MarketData(
            totalVolume = 100_000,
            bidAskSpread = 0.01,
            activeTraders = 500,
            lastTradeTime = Instant.now()
        )

        // When
        val score = calculateLiquidityScore(market)

        // Then
        assertThat(score).isGreaterThan(80.0)
        assertThat(score).isLessThanOrEqualTo(100.0)
    }

    @Test
    fun `should return low score for illiquid market`() {
        // Given
        val market = MarketData(
            totalVolume = 100,
            bidAskSpread = 0.5,
            activeTraders = 2,
            lastTradeTime = Instant.now().minusSeconds(86400) // 1 day ago
        )

        // When
        val score = calculateLiquidityScore(market)

        // Then
        assertThat(score).isLessThan(30.0)
        assertThat(score).isGreaterThanOrEqualTo(0.0)
    }

    @Test
    fun `should handle edge case - zero volume`() {
        // Given
        val market = MarketData(
            totalVolume = 0,
            bidAskSpread = 0.0,
            activeTraders = 0,
            lastTradeTime = Instant.now()
        )

        // When
        val score = calculateLiquidityScore(market)

        // Then
        assertThat(score).isEqualTo(0.0)
    }
}
```

## Step 3: Run Tests - Verify FAIL

```bash
./gradlew test --tests "LiquidityCalculatorTest"

FAIL com.example.market.LiquidityCalculatorTest
  ✕ should return high score for liquid market
    NotImplementedError: Not implemented

1 test failed, 0 passed
```

✅ Tests fail as expected. Ready to implement.

## Step 4: Implement Minimal Code (GREEN)

```kotlin
// src/main/kotlin/com/example/market/LiquidityCalculator.kt
package com.example.market

import java.time.Duration
import java.time.Instant
import kotlin.math.max
import kotlin.math.min

fun calculateLiquidityScore(market: MarketData): Double {
    // Handle zero volume edge case
    if (market.totalVolume == 0L) {
        return 0.0
    }

    // Calculate component scores (0-100 scale)
    val volumeScore = min(market.totalVolume / 1000.0, 100.0)
    val spreadScore = max(100 - (market.bidAskSpread * 1000), 0.0)
    val traderScore = min(market.activeTraders / 10.0, 100.0)

    // Recent activity bonus
    val hoursSinceLastTrade = Duration.between(market.lastTradeTime, Instant.now()).toHours()
    val recencyScore = max(100 - (hoursSinceLastTrade * 10), 0.0)

    // Weighted average
    val score = (
        volumeScore * 0.4 +
        spreadScore * 0.3 +
        traderScore * 0.2 +
        recencyScore * 0.1
    )

    return score.coerceIn(0.0, 100.0)
}
```

## Step 5: Run Tests - Verify PASS

```bash
./gradlew test --tests "LiquidityCalculatorTest"

PASS com.example.market.LiquidityCalculatorTest
  ✓ should return high score for liquid market (12 ms)
  ✓ should return low score for illiquid market (3 ms)
  ✓ should handle edge case - zero volume (2 ms)

3 tests passed
```

✅ All tests passing!

## Step 6: Refactor (IMPROVE)

```kotlin
// src/main/kotlin/com/example/market/LiquidityCalculator.kt
package com.example.market

import java.time.Duration
import java.time.Instant
import kotlin.math.max
import kotlin.math.min

private object Weights {
    const val VOLUME = 0.4
    const val SPREAD = 0.3
    const val TRADERS = 0.2
    const val RECENCY = 0.1
}

private object ScaleFactors {
    const val VOLUME = 1000.0
    const val SPREAD = 1000.0
    const val TRADERS = 10.0
    const val RECENCY_PENALTY = 10.0
}

fun calculateLiquidityScore(market: MarketData): Double {
    if (market.totalVolume == 0L) return 0.0

    val volumeScore = (market.totalVolume / ScaleFactors.VOLUME).coerceAtMost(100.0)
    val spreadScore = (100 - market.bidAskSpread * ScaleFactors.SPREAD).coerceIn(0.0, 100.0)
    val traderScore = (market.activeTraders / ScaleFactors.TRADERS).coerceAtMost(100.0)

    val hoursSinceLastTrade = Duration.between(market.lastTradeTime, Instant.now()).toHours()
    val recencyScore = (100 - hoursSinceLastTrade * ScaleFactors.RECENCY_PENALTY).coerceIn(0.0, 100.0)

    val weightedScore =
        volumeScore * Weights.VOLUME +
        spreadScore * Weights.SPREAD +
        traderScore * Weights.TRADERS +
        recencyScore * Weights.RECENCY

    return weightedScore.coerceIn(0.0, 100.0)
}
```

## Step 7: Verify Tests Still Pass

```bash
./gradlew test --tests "LiquidityCalculatorTest"

PASS com.example.market.LiquidityCalculatorTest
  ✓ should return high score for liquid market (10 ms)
  ✓ should return low score for illiquid market (2 ms)
  ✓ should handle edge case - zero volume (1 ms)

3 tests passed
```

✅ Refactoring complete, tests still passing!

## Step 8: Check Coverage

```bash
./gradlew test jacocoTestReport

File                      | % Lines | % Branches
--------------------------|---------|----------
LiquidityCalculator.kt    |   100%  |   100%

Coverage: 100% ✅ (Target: 80%)
```

✅ TDD session complete!

## TDD Best Practices

**DO:**
- ✅ Write the test FIRST, before any implementation
- ✅ Run tests and verify they FAIL before implementing
- ✅ Write minimal code to make tests pass
- ✅ Refactor only after tests are green
- ✅ Add edge cases and error scenarios
- ✅ Aim for 80%+ coverage (100% for critical code)

**DON'T:**
- ❌ Write implementation before tests
- ❌ Skip running tests after each change
- ❌ Write too much code at once
- ❌ Ignore failing tests
- ❌ Test implementation details (test behavior)
- ❌ Mock everything (prefer integration tests)

## Test Types to Include

**Unit Tests** (Function-level):
- Happy path scenarios
- Edge cases (null, empty, max values)
- Error conditions
- Boundary values

**Integration Tests** (Service-level):
- API endpoints with @SpringBootTest
- Database operations with @DataJpaTest
- External service calls

**E2E Tests** (use `/e2e` command):
- Critical user flows
- Multi-step processes
- Full stack integration

## Coverage Requirements

- **80% minimum** for all code
- **100% required** for:
  - Financial calculations
  - Authentication logic
  - Security-critical code
  - Core business logic

## Running Tests

```bash
# Run all tests
./gradlew test

# Run specific test class
./gradlew test --tests "LiquidityCalculatorTest"

# Run with coverage
./gradlew test jacocoTestReport

# View coverage report
open build/reports/jacoco/test/html/index.html
```

## Important Notes

**MANDATORY**: Tests must be written BEFORE implementation. The TDD cycle is:

1. **RED** - Write failing test
2. **GREEN** - Implement to pass
3. **REFACTOR** - Improve code

Never skip the RED phase. Never write code before tests.

## Related Commands

- Use `/plan` first to understand what to build
- Use `/tdd` to implement with tests
- Use `/build-fix` if build errors occur
- Use `/code-review` to review implementation
- Use `/test-coverage` to verify coverage

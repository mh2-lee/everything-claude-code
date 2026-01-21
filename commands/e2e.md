---
description: Generate and run end-to-end tests with Selenium or Playwright. Creates test journeys, runs tests, captures screenshots, and generates reports.
---

# E2E Command

This command invokes the **e2e-runner** agent to generate, maintain, and execute end-to-end tests.

## What This Command Does

1. **Generate Test Journeys** - Create E2E tests for user flows
2. **Run E2E Tests** - Execute tests across browsers
3. **Capture Artifacts** - Screenshots on failures
4. **Generate Reports** - HTML reports with results
5. **Identify Flaky Tests** - Quarantine unstable tests

## When to Use

Use `/e2e` when:
- Testing critical user journeys (login, checkout, payments)
- Verifying multi-step flows work end-to-end
- Testing UI interactions and navigation
- Validating integration between frontend and backend
- Preparing for production deployment

## How It Works

The e2e-runner agent will:

1. **Analyze user flow** and identify test scenarios
2. **Generate test code** using Page Object Model pattern
3. **Run tests** across browsers
4. **Capture failures** with screenshots
5. **Generate report** with results and artifacts

## Example Usage - Selenium with Kotlin

```
User: /e2e Test the market search and view flow

Agent (e2e-runner):
# E2E Test Generation: Market Search and View Flow

## Test Scenario Identified

**User Journey:** Market Search → View Results → Click Market → View Details

## Generated Test Code
```

```kotlin
// src/test/kotlin/e2e/MarketSearchTest.kt
package e2e

import io.github.bonigarcia.wdm.WebDriverManager
import org.junit.jupiter.api.*
import org.junit.jupiter.api.Assertions.*
import org.openqa.selenium.By
import org.openqa.selenium.WebDriver
import org.openqa.selenium.chrome.ChromeDriver
import org.openqa.selenium.chrome.ChromeOptions
import org.openqa.selenium.support.ui.ExpectedConditions
import org.openqa.selenium.support.ui.WebDriverWait
import java.time.Duration

@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class MarketSearchTest {

    private lateinit var driver: WebDriver
    private lateinit var wait: WebDriverWait
    private val baseUrl = System.getenv("BASE_URL") ?: "http://localhost:8080"

    @BeforeAll
    fun setup() {
        WebDriverManager.chromedriver().setup()
        val options = ChromeOptions().apply {
            addArguments("--headless")
            addArguments("--no-sandbox")
            addArguments("--disable-dev-shm-usage")
        }
        driver = ChromeDriver(options)
        wait = WebDriverWait(driver, Duration.ofSeconds(10))
    }

    @AfterAll
    fun teardown() {
        driver.quit()
    }

    @Test
    fun `user can search markets and view details`() {
        // 1. Navigate to markets page
        driver.get("$baseUrl/markets")

        // Verify page loaded
        wait.until(ExpectedConditions.titleContains("Markets"))
        val heading = driver.findElement(By.tagName("h1"))
        assertTrue(heading.text.contains("Markets"))

        // 2. Perform search
        val searchInput = driver.findElement(By.cssSelector("[data-testid='search-input']"))
        searchInput.sendKeys("election")

        // Wait for results
        wait.until(ExpectedConditions.presenceOfElementLocated(
            By.cssSelector("[data-testid='market-card']")
        ))

        // 3. Verify search results
        val marketCards = driver.findElements(By.cssSelector("[data-testid='market-card']"))
        assertTrue(marketCards.isNotEmpty(), "Should have search results")

        // 4. Click first result
        val firstMarketTitle = marketCards.first().findElement(By.tagName("h3")).text
        marketCards.first().click()

        // 5. Verify market details page
        wait.until(ExpectedConditions.urlContains("/markets/"))

        val detailsTitle = wait.until(
            ExpectedConditions.presenceOfElementLocated(By.cssSelector("[data-testid='market-name']"))
        )
        assertTrue(detailsTitle.isDisplayed)

        // 6. Verify chart renders
        val chart = driver.findElement(By.cssSelector("[data-testid='price-chart']"))
        assertTrue(chart.isDisplayed)
    }

    @Test
    fun `search with no results shows empty state`() {
        driver.get("$baseUrl/markets")

        val searchInput = driver.findElement(By.cssSelector("[data-testid='search-input']"))
        searchInput.sendKeys("xyznonexistentmarket123456")

        wait.until(ExpectedConditions.presenceOfElementLocated(
            By.cssSelector("[data-testid='no-results']")
        ))

        val noResults = driver.findElement(By.cssSelector("[data-testid='no-results']"))
        assertTrue(noResults.text.contains("no", ignoreCase = true))
    }
}
```

## Page Object Model Pattern

```kotlin
// src/test/kotlin/e2e/pages/MarketsPage.kt
package e2e.pages

import org.openqa.selenium.By
import org.openqa.selenium.WebDriver
import org.openqa.selenium.WebElement
import org.openqa.selenium.support.FindBy
import org.openqa.selenium.support.PageFactory
import org.openqa.selenium.support.ui.ExpectedConditions
import org.openqa.selenium.support.ui.WebDriverWait
import java.time.Duration

class MarketsPage(private val driver: WebDriver) {

    private val wait = WebDriverWait(driver, Duration.ofSeconds(10))

    @FindBy(css = "[data-testid='search-input']")
    lateinit var searchInput: WebElement

    @FindBy(css = "[data-testid='market-card']")
    lateinit var marketCards: List<WebElement>

    init {
        PageFactory.initElements(driver, this)
    }

    fun navigate(baseUrl: String) {
        driver.get("$baseUrl/markets")
        wait.until(ExpectedConditions.titleContains("Markets"))
    }

    fun searchMarkets(query: String) {
        searchInput.clear()
        searchInput.sendKeys(query)
        wait.until(ExpectedConditions.presenceOfElementLocated(
            By.cssSelector("[data-testid='market-card']")
        ))
    }

    fun getMarketCount(): Int = marketCards.size

    fun clickFirstMarket() {
        marketCards.first().click()
    }
}
```

## Running Tests

```bash
# Run all E2E tests
./gradlew e2eTest

# Run specific test class
./gradlew e2eTest --tests "e2e.MarketSearchTest"

# Run with specific browser
BASE_URL=http://localhost:8080 ./gradlew e2eTest

# Run in headed mode (see browser)
HEADLESS=false ./gradlew e2eTest
```

## Gradle Configuration

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("org.seleniumhq.selenium:selenium-java:4.15.0")
    testImplementation("io.github.bonigarcia:webdrivermanager:5.6.2")
    testImplementation("org.junit.jupiter:junit-jupiter:5.10.0")
}

tasks.register<Test>("e2eTest") {
    useJUnitPlatform()
    include("**/e2e/**")
    testLogging {
        events("passed", "skipped", "failed")
    }
}
```

## Test Report

```
╔══════════════════════════════════════════════════════════════╗
║                    E2E Test Results                          ║
╠══════════════════════════════════════════════════════════════╣
║ Status:     ✅ ALL TESTS PASSED                              ║
║ Total:      3 tests                                          ║
║ Passed:     3 (100%)                                         ║
║ Failed:     0                                                ║
║ Duration:   12.4s                                            ║
╚══════════════════════════════════════════════════════════════╝

Report: build/reports/tests/e2eTest/index.html
```

## Critical Flows to Test

**CRITICAL (Must Always Pass):**
1. User can login/logout
2. User can browse listings
3. User can search
4. User can view details
5. User can complete purchase flow
6. User can view order history

**IMPORTANT:**
1. User profile updates
2. Filter and sort functionality
3. Pagination
4. Mobile responsive layout

## Best Practices

**DO:**
- ✅ Use Page Object Model for maintainability
- ✅ Use data-testid attributes for selectors
- ✅ Wait for elements, not arbitrary timeouts
- ✅ Test critical user journeys end-to-end
- ✅ Run tests before merging to main

**DON'T:**
- ❌ Use brittle selectors (CSS classes can change)
- ❌ Test implementation details
- ❌ Run tests against production
- ❌ Ignore flaky tests
- ❌ Test every edge case with E2E (use unit tests)

## CI/CD Integration

```yaml
# .github/workflows/e2e.yml
- name: Run E2E tests
  run: ./gradlew e2eTest
  env:
    BASE_URL: http://localhost:8080

- name: Upload test report
  if: always()
  uses: actions/upload-artifact@v3
  with:
    name: e2e-report
    path: build/reports/tests/e2eTest/
```

## Related Commands

- Use `/plan` to identify critical journeys to test
- Use `/tdd` for unit tests (faster, more granular)
- Use `/e2e` for integration and user journey tests
- Use `/code-review` to verify test quality

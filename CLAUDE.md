# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Selenium-based e2e automation framework for testing the [Rahul Shetty Academy e-commerce platform](https://rahulshettyacademy.com/client). The project is wired to a Jenkins CI/CD pipeline triggered by GitHub pushes.

**Stack:** Java 8, Maven, TestNG 7.10.2, Cucumber 7.15.0, Selenium 4.21.0, ExtentReports 5.1.1, WebDriverManager 5.8.0

## Commands

### Run Tests

```bash
# Default regression suite
mvn clean test

# Specific Maven profiles
mvn clean test -P Regression
mvn clean test -P Purchase
mvn clean test -P CucumberTests
mvn clean test -P ErrorValidation

# Override browser at runtime (chrome | firefox | edge | chrome-headless)
mvn clean test -P Regression -Dbrowser=chrome-headless

# Run a single test class (bypasses suite XML, uses Surefire directly)
mvn clean test -Dtest=SubmitOrderTest
mvn clean test -Dtest=ErrorValidationsTest#LoginErrorValidation

# Build without running tests
mvn clean install -DskipTests
```

### Test Profiles → Suite Mapping

| Profile | Suite File | TestNG Group |
|---|---|---|
| Regression | testSuites/testng.xml | (all) |
| Purchase | testSuites/Purchase.xml | Purchase |
| ErrorValidation | testSuites/ErrorValidationTests.xml | ErrorHandling |
| CucumberTests | TestNGTestRunner.java | @ErrorValidation tag only |

### Reports

- **ExtentReports HTML:** `reports/index.html` (also where failure screenshots land)
- **Cucumber HTML:** `target/cucumber.html`
- **TestNG results:** `test-output/`

## Architecture

### Page Object Model

All page objects live in `src/main/java/riteshsharma/pageobjects/` and extend `AbstractComponent`, which centralizes:
- PageFactory initialization
- Explicit wait helpers (`waitForElementToAppear`, `waitForWebElementToAppear`, `waitForElementToDisappear`)
- Navigation to Cart and Orders pages

Page flow: `LandingPage` → `ProductCatalogue` → `CartPage` → `CheckoutPage` → `ConfirmationPage` / `OrderPage`

### Test Layer

`src/test/java/riteshsharma/TestComponents/`:
- **BaseTest** — WebDriver lifecycle (`@BeforeMethod`/`@AfterMethod`), reads browser from `GlobalData.properties` (overridable via `-Dbrowser=`), JSON test data parsing, screenshot capture
- **Listeners** — TestNG listener wiring ExtentReports; uses `ThreadLocal<ExtentTest>` for thread-safe parallel reporting; accesses the test instance's `driver` field via reflection on failure to capture screenshots
- **Retry** — Retries failed tests once (`maxTry=1`)

`src/test/java/riteshsharma/tests/`:
- **SubmitOrderTest** — Data-driven purchase flow via `@DataProvider` (reads `PurchaseOrder.json`); `OrderHistoryTest` declares `dependsOnMethods={"submitOrder"}` and uses hardcoded credentials (not from DataProvider)
- **ErrorValidationsTest** — Login and cart error message validation; `LoginErrorValidation` uses `retryAnalyzer=Retry.class`
- **StandAloneTest** — Scratch exploration script with a `main()` method; not wired to any suite or profile, not intended to be run via Maven

### BDD / Cucumber

`src/test/java/cucumber/` contains feature files and `TestNGTestRunner`. `StepDefinitionImpl` **extends BaseTest** to reuse driver setup — step definitions share the same `@BeforeMethod`/`@AfterMethod` lifecycle.

**Important:** `TestNGTestRunner` is hardcoded to `tags = "@ErrorValidation"`. `SubmitOrder.feature` uses `@Regression` and will **not** run under the `CucumberTests` profile despite existing in the same directory.

### Test Data

`src/test/java/riteshsharma/data/PurchaseOrder.json` — two datasets (email/password/product). `DataReader` uses Jackson `ObjectMapper` to parse into `List<HashMap<String, String>>`, fed via `@DataProvider`.

### Configuration

`src/main/java/riteshsharma/resources/GlobalData.properties` — single property: `browser=chrome`. Override at runtime with `-Dbrowser=<value>`.

## Key Patterns

- **Java 8 Streams** are used to find `WebElement`s by visible text (e.g., matching product names in a list).
- **Actions API** is used for the country autocomplete dropdown on CheckoutPage — it types characters, not selecting via `<select>`.
- **Parallel execution** is configured in `testng.xml` with `parallel="tests"` at suite level; ExtentReports uses `ThreadLocal` to keep reports isolated per thread.
- **Headless mode** requires `-Dbrowser=chrome-headless`; BaseTest sets a fixed window size (`1440x900`) when headless is active.
- **`waitForElementToDisappear`** currently uses `Thread.sleep(1000)` — the WebDriverWait-based implementation is commented out in `AbstractComponent.java`.

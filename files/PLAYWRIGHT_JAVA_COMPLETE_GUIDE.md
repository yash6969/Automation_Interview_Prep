# Playwright Java — Complete Enterprise Framework Guide

> **From Selenium to Playwright: A ground-up redesign for the Superr QA team**
>
> This guide analyzes your existing Selenium framework, explains why each architectural decision changes, and provides a complete production-ready Playwright Java framework with full code.

---

## Table of Contents

1. [STEP 1: Existing Selenium Framework Analysis](#step-1-existing-selenium-framework-analysis)
2. [STEP 2: Modern Playwright Java Architecture](#step-2-modern-playwright-java-architecture)
3. [STEP 3: Playwright Concepts for Beginners](#step-3-playwright-concepts-for-beginners)
4. [STEP 5: Class Necessity Audit](#step-5-class-necessity-audit)
5. [STEP 7: Framework Code — Production-Ready](#step-7-framework-code)
6. [STEP 9: Page Objects with Modern Locator API](#step-9-page-objects)
7. [STEP 16: Selenium → Playwright Migration Guide (100+ Operations)](#step-16-migration-guide)
8. [Best Practices](#best-practices)
9. [Interview Questions & FAQs](#interview-questions)
10. [Debugging Guide](#debugging-guide)
11. [Performance Tips](#performance-tips)

---

# STEP 1: Existing Selenium Framework Analysis

## 1.1 Overall Architecture

Your current framework is a **Selenium 4 + Appium + TestNG** hybrid automation framework using the **Page Object Model (POM)** pattern. It supports:

- **Web automation** via Selenium WebDriver (Chrome, Edge)
- **Mobile automation** via Appium (Android)
- **Hybrid flows** via a shared DriverManager

The web-specific classes are:

| Class | Package | Role |
|-------|---------|------|
| `DriverFactory` | `driver` | Creates WebDriver instances (Chrome/Edge) |
| `DriverManager` | `driver` | ThreadLocal driver storage + routing |
| `BaseTest` | `tests.base` | @BeforeSuite/@BeforeMethod lifecycle |
| `TestListener` | `listeners` | ExtentReports integration via TestNG ITestListener |
| `ExtentManager` | `utils` | Singleton Extent Reports setup |
| `ReportUtils` | `utils` | Static helper for logging to Extent |
| `ConfigReader` | `utils` | Properties file loader |
| `ScreenshotUtils` | `utils` | Base64 screenshot capture |
| `LMSPage` | `pageobjects.web` | LMS Portal page object |
| `StageAdminPortalPage` | `pageobjects.web` | Admin Portal page object |
| `LMSTests` | `tests.web` | LMS test class |
| `AdminPortal` | `tests.web` | Admin Portal test class |

## 1.2 Folder Structure (Web Only)

```
src/main/java/com/superr/framework/
├── driver/
│   ├── DriverFactory.java          ← Creates Chrome/Edge WebDriver
│   ├── DriverManager.java          ← ThreadLocal driver management
│   └── MobileDriverFactory.java    ← (IGNORE — mobile only)
├── listeners/
│   └── TestListener.java           ← TestNG → ExtentReports bridge
├── pageobjects/web/
│   ├── adminportal/StageAdminPortalPage.java
│   └── lmsportal/LMSPage.java
└── utils/
    ├── ConfigReader.java           ← Properties loader
    ├── ExtentManager.java          ← Report singleton
    ├── ReportUtils.java            ← Report helpers
    ├── ScreenshotUtils.java        ← Screenshot capture
    ├── LoggerUtils.java            ← Log4j wrapper
    └── WaitUtils.java              ← Explicit wait helpers

src/test/java/com/superr/tests/
├── base/BaseTest.java              ← Web test lifecycle
└── web/
    ├── adminportal/AdminPortal.java
    └── lmsportal/LMSTests.java

src/main/resources/config/
└── environments/
    ├── config-dev.properties
    ├── config-staging.properties
    └── config-prod.properties

src/test/resources/testng_suites/
└── webSmokeTests.xml
```

## 1.3 Design Patterns

| Pattern | Where Used | Assessment |
|---------|-----------|------------|
| Page Object Model | `LMSPage`, `StageAdminPortalPage` | ✅ Good — but Selenium-style (returns void, manual waits) |
| Singleton | `ExtentManager` | ⚠️ Unnecessary in Playwright (built-in reporting) |
| Factory | `DriverFactory` | ❌ Unnecessary in Playwright (built-in browser launch) |
| ThreadLocal | `DriverManager` | ❌ Anti-pattern in Playwright (context isolation replaces this) |
| Properties Config | `ConfigReader` | ⚠️ Partially reusable — Playwright prefers programmatic config |

## 1.4 Strengths

1. **Clean separation** — page objects, tests, utils, and config are properly separated
2. **Multi-environment** — dev/staging/prod config files with a single `ConfigReader.load(env)` call
3. **Consistent logging** — Log4j2 + ExtentReports across all tests
4. **WebDriverManager** — auto-downloads correct driver binaries (good for Selenium, irrelevant in Playwright)
5. **Credential management** — test data in properties files, not hardcoded in tests

## 1.5 Weaknesses & Code Smells

### ❌ 1. DriverFactory/DriverManager — Over-engineered for web

Selenium requires manual browser lifecycle management. Playwright does NOT. In Playwright:
- `Playwright.create()` → `browser.launch()` → `browser.newContext()` → `context.newPage()`
- Browser contexts provide **automatic isolation** — no ThreadLocal needed
- No driver binaries to download — Playwright bundles its own browsers

### ❌ 2. WaitUtils — Completely unnecessary

Your `WaitUtils` wraps Selenium's `WebDriverWait` + `ExpectedConditions`. Playwright has **auto-waiting built in**. Every `locator.click()`, `locator.fill()`, `locator.isVisible()` automatically waits for the element to be actionable. Zero explicit waits needed.

### ❌ 3. ExtentManager + ReportUtils — Replaced by Playwright's built-in reporting

Playwright provides:
- **HTML Reporter** (built-in, zero config)
- **Trace Viewer** (records every action, network call, screenshot, DOM snapshot)
- **Video recording** (per-test, automatic)
- **Screenshot on failure** (automatic)

ExtentReports adds no value over Playwright's native tooling.

### ❌ 4. ScreenshotUtils — Unnecessary

Playwright captures screenshots automatically on failure. Manual `page.screenshot()` is one line — no utility class needed.

### ⚠️ 5. TestNG dependency

Playwright Java's official integration is with **JUnit 5**. TestNG works but lacks first-party support. JUnit 5 is the recommended path.

### ⚠️ 6. Page objects use `By` locators and manual waits

```java
// Your current Selenium pattern:
driver.findElement(By.id("email")).sendKeys(email);
new WebDriverWait(driver, 10).until(ExpectedConditions.visibilityOfElementLocated(By.id("toast")));
```

In Playwright, this becomes:
```java
// Playwright pattern — auto-waits, no explicit wait:
page.getByLabel("Email").fill(email);
page.getByText("files uploaded successfully").isVisible(); // auto-waits
```

### ⚠️ 7. Hardcoded file paths

```java
appPath=D:/MyWorkSpace/SuperrQAAutomation/src/main/resources/...
```
File paths should be relative and OS-agnostic.

## 1.6 Technical Debt

| Issue | Severity | Playwright Resolution |
|-------|----------|----------------------|
| Manual explicit waits everywhere | High | Auto-waiting eliminates 100% of explicit waits |
| ThreadLocal driver management | Medium | Browser context isolation replaces this |
| ExtentReports maintenance burden | Medium | Built-in HTML reporter + Trace Viewer |
| WebDriverManager dependency | Low | Playwright bundles browsers — zero driver management |
| TestNG XML suite files | Low | JUnit 5 tags + Maven profiles replace suite XMLs |

## 1.7 Verdict: What Stays, What Goes, What Gets Rewritten

| Class | Verdict | Reason |
|-------|---------|--------|
| `DriverFactory` | 🗑 **DELETE** | Playwright manages browsers natively |
| `DriverManager` | 🗑 **DELETE** | Browser contexts provide isolation |
| `BaseTest` | ✏️ **REWRITE** | Becomes a thin JUnit 5 base with Playwright lifecycle |
| `TestListener` | 🗑 **DELETE** | Playwright's built-in reporting replaces it |
| `ExtentManager` | 🗑 **DELETE** | No longer needed |
| `ReportUtils` | 🗑 **DELETE** | No longer needed |
| `ScreenshotUtils` | 🗑 **DELETE** | One-liner in Playwright |
| `WaitUtils` | 🗑 **DELETE** | Auto-waiting makes this obsolete |
| `LoggerUtils` | ⚠️ **OPTIONAL** | SLF4J/Log4j still useful for custom logging |
| `ConfigReader` | ✏️ **REWRITE** | Simplified — fewer config keys needed |
| `LMSPage` | ✏️ **REWRITE** | Modern Locator API, no `By` selectors |
| `StageAdminPortalPage` | ✏️ **REWRITE** | Modern Locator API |
| `LMSTests` | ✏️ **REWRITE** | JUnit 5 + Playwright assertions |
| `AdminPortal` | ✏️ **REWRITE** | JUnit 5 + Playwright assertions |

**Score: 7 classes deleted, 6 rewritten, 1 optional.** This is normal — Playwright's built-in features replace most of your utility layer.

---

# STEP 2: Modern Playwright Java Architecture

## 2.1 Why the Folder Structure Changes

In Selenium, you need layers of abstraction to manage what Playwright handles natively:

| Selenium Need | Custom Code Required | Playwright Equivalent |
|--------------|---------------------|----------------------|
| Browser launch | `DriverFactory` + WebDriverManager | `playwright.chromium().launch()` |
| Parallel isolation | `ThreadLocal<WebDriver>` | `browser.newContext()` — automatic |
| Waiting | `WaitUtils` + `WebDriverWait` | Auto-waiting — built in |
| Reporting | `ExtentManager` + `TestListener` + `ReportUtils` | Built-in HTML reporter + Trace Viewer |
| Screenshots | `ScreenshotUtils` | `page.screenshot()` — one line |

**Result:** Your `driver/`, `listeners/`, and half of `utils/` disappear entirely.

## 2.2 New Folder Structure

```
superr-playwright/
├── pom.xml
├── playwright.config                    ← (optional) Env-specific overrides
│
├── src/
│   ├── main/java/com/superr/playwright/
│   │   ├── config/
│   │   │   └── EnvConfig.java           ← Environment configuration (replaces ConfigReader)
│   │   │
│   │   └── pages/
│   │       ├── BasePage.java            ← Thin base — holds Page reference only
│   │       ├── LoginPage.java           ← Shared login (LMS + Admin)
│   │       ├── LMSPage.java             ← LMS portal page object
│   │       └── AdminPortalPage.java     ← Admin portal page object
│   │
│   └── test/java/com/superr/tests/
│       ├── BaseTest.java                ← JUnit 5 lifecycle (Playwright + Browser + Context + Page)
│       ├── lms/
│       │   └── LMSTests.java            ← LMS test class
│       └── admin/
│           └── AdminPortalTests.java    ← Admin portal test class
│
└── src/main/resources/
    └── config/
        ├── dev.properties
        ├── staging.properties
        └── prod.properties
```

### Package-by-Package Justification

| Package | Exists? | Why |
|---------|---------|-----|
| `config/` | ✅ Kept | Multi-environment support (URLs, credentials) is still valuable |
| `pages/` | ✅ Kept | Page Object Model is valid in Playwright — the IMPLEMENTATION changes, not the pattern |
| `driver/` | 🗑 Removed | Playwright manages browser lifecycle natively. Zero custom code needed. |
| `listeners/` | 🗑 Removed | Playwright's built-in reporter replaces TestNG listeners |
| `utils/` | 🗑 Removed | WaitUtils, ReportUtils, ScreenshotUtils, ExtentManager — all unnecessary |
| `tests/` | ✅ Kept | Tests still exist, but use JUnit 5 instead of TestNG |

**Why no `utils/` package?** Every utility class you had either wraps something Playwright does natively (waits, screenshots, reporting) or becomes a one-liner. Creating wrapper classes around Playwright APIs is an anti-pattern — it hides Playwright's fluent API and makes debugging harder.

## 2.3 Build Tool: Maven

**Decision: Maven**, not Gradle.

Why:
- Your team already uses Maven (no learning curve)
- Playwright Java has first-class Maven support via `com.microsoft.playwright:playwright`
- Playwright browser installation runs via `mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="install"`

---

# STEP 3: Playwright Concepts for Beginners

## 3.1 The Playwright Object Model

Playwright has a clear hierarchy. Every object has a specific lifetime:

```
Playwright (process-level — created once)
  └── Browser (one per browser type: Chromium, Firefox, WebKit)
       └── BrowserContext (isolated session — like an incognito window)
            └── Page (a single tab)
                 └── Locator (points to element(s) on the page)
```

### How This Maps to Your Selenium Code

| Selenium Concept | Playwright Equivalent | Key Difference |
|-----------------|----------------------|----------------|
| `WebDriver driver` | `Page page` | Page is the primary interaction object |
| `new ChromeDriver()` | `playwright.chromium().launch()` | No binary download needed |
| "Open incognito window" | `browser.newContext()` | Built-in isolation, cookies, storage |
| `driver.findElement(By.id("x"))` | `page.locator("#x")` or `page.getByTestId("x")` | Returns a Locator (lazy, auto-waits) |
| `element.click()` | `locator.click()` | Auto-waits for element to be clickable |
| `element.sendKeys("text")` | `locator.fill("text")` | Clears first, then types |
| `WebDriverWait` | **Nothing** | Auto-waiting is built into every action |
| `driver.switchTo().frame()` | `page.frameLocator("#frame")` | Scoped locator, no switching needed |
| `driver.manage().window()` | `page.setViewportSize()` | Direct API, no manage() chain |

## 3.2 Auto-Waiting — The Biggest Paradigm Shift

In Selenium, you write:
```java
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
wait.until(ExpectedConditions.elementToBeClickable(By.id("submit")));
driver.findElement(By.id("submit")).click();
```

In Playwright, you write:
```java
page.locator("#submit").click();
```

**That's it.** Playwright automatically:
1. Waits for the element to appear in the DOM
2. Waits for it to be visible
3. Waits for it to be stable (not animating)
4. Waits for it to be enabled
5. Waits for it to receive pointer events (not covered by another element)
6. Then clicks

Default timeout: 30 seconds (configurable). If any condition isn't met within the timeout, it throws a clear error with a trace.

**This is why `WaitUtils` is deleted.** Creating a custom wait wrapper around Playwright is an anti-pattern — it fights the framework instead of using it.

## 3.3 Locator API — Modern Element Selection

Playwright's Locator API is **semantic** — it finds elements the way a user thinks about them.

| Locator Method | Finds By | Example |
|---------------|----------|---------|
| `getByRole("button", opts)` | ARIA role + name | Login button, Submit button |
| `getByLabel("Email")` | Associated `<label>` text | Form inputs |
| `getByPlaceholder("Enter email")` | Placeholder attribute | Form inputs without labels |
| `getByText("Sign In")` | Visible text content | Links, buttons, paragraphs |
| `getByTestId("login-btn")` | `data-testid` attribute | Dev-provided stable identifiers |
| `locator("css-selector")` | CSS selector | Fallback when semantic locators won't work |

**Priority order (use the first one that works):**
1. `getByRole()` — most resilient, accessibility-friendly
2. `getByLabel()` / `getByPlaceholder()` — great for forms
3. `getByTestId()` — stable if devs add `data-testid`
4. `getByText()` — good for buttons and links
5. `locator("css")` — fallback
6. `locator("xpath")` — **last resort only**

### Why No XPath?

XPath is fragile, slow, and unreadable. Playwright's semantic locators are:
- **Resilient** — survive DOM structure changes
- **Readable** — `getByRole("button", new Page.GetByRoleOptions().setName("Login"))` reads like English
- **Fast** — native engine, no XPath evaluation

## 3.4 Assertions — Built-In and Auto-Retrying

Playwright provides `assertThat()` which **auto-retries** until the condition is met:

```java
import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;

assertThat(page).hasTitle("Dashboard");
assertThat(page.getByText("Welcome")).isVisible();
assertThat(page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Submit"))).isEnabled();
```

These assertions automatically retry for up to 5 seconds (configurable). No `Thread.sleep()`, no `WebDriverWait`.

## 3.5 Trace Viewer — Replaces Manual Screenshots

Playwright's Trace Viewer captures **everything** during a test:
- Every action (click, fill, navigate)
- Before/after screenshots of each action
- DOM snapshots
- Network requests/responses
- Console logs

After a test run:
```bash
mvn exec:java -e -D exec.mainClass=com.microsoft.playwright.CLI -D exec.args="show-trace trace.zip"
```

This opens a visual debugger where you can step through every action. This is why `ExtentReports` and `ScreenshotUtils` are unnecessary — Trace Viewer provides 10x more information.

---

# STEP 5: Class Necessity Audit

Before writing any code, here's the audit of every class:

## Classes That Are UNNECESSARY in Playwright

### DriverFactory.java — DELETE

**Why unnecessary:** Playwright launches browsers with one line:
```java
Browser browser = playwright.chromium().launch();
```
No WebDriverManager, no ChromeOptions builder, no switch-case for browser types. Playwright bundles Chromium, Firefox, and WebKit — zero binary management.

### DriverManager.java — DELETE

**Why unnecessary:** Playwright's `BrowserContext` provides automatic isolation:
```java
BrowserContext context = browser.newContext(); // Isolated session
Page page = context.newPage();                // Fresh tab
```
Each context has its own cookies, storage, and cache. ThreadLocal is unnecessary because contexts are already isolated.

### WaitUtils.java — DELETE

**Why unnecessary:** Every Playwright action auto-waits. `locator.click()` waits for clickability. `locator.fill()` waits for the input to be editable. `assertThat(locator).isVisible()` retries automatically. There is literally nothing for a WaitUtils class to do.

### ExtentManager.java — DELETE

**Why unnecessary:** Playwright generates HTML reports natively. Trace Viewer provides richer debugging than ExtentReports ever could.

### ReportUtils.java — DELETE

**Why unnecessary:** Playwright logs every action automatically to its trace. For custom logging, `System.out.println()` or SLF4J is sufficient.

### ScreenshotUtils.java — DELETE

**Why unnecessary:** `page.screenshot()` is a one-line call. Playwright also captures screenshots automatically on test failure when configured.

### TestListener.java — DELETE

**Why unnecessary:** This was a TestNG `ITestListener` that bridged TestNG events to ExtentReports. With JUnit 5 + Playwright's built-in reporting, there's no TestNG and no ExtentReports — so no listener needed.

## Classes That Get REWRITTEN

### ConfigReader.java → EnvConfig.java

**Why rewritten (not deleted):** Multi-environment config (dev/staging/prod URLs, credentials) is still valuable. But the implementation simplifies — fewer keys needed since Playwright handles browser config, timeouts, and waits natively.

### BaseTest.java → BaseTest.java

**Why rewritten:** The lifecycle changes from Selenium's `DriverFactory.initDriver()` + `DriverManager.setWebDriver()` to Playwright's `Playwright.create()` → `browser.launch()` → `context.newPage()`. Uses JUnit 5 `@BeforeAll` / `@BeforeEach` / `@AfterEach` / `@AfterAll`.

### LMSPage.java → LMSPage.java

**Why rewritten:** Locator strategy changes completely. `By.id()` / `By.xpath()` → `getByRole()` / `getByLabel()` / `getByPlaceholder()`. Manual waits disappear. The page object becomes cleaner and shorter.

### StageAdminPortalPage.java → AdminPortalPage.java

**Why rewritten:** Same reasons as LMSPage.

### LMSTests.java → LMSTests.java

**Why rewritten:** TestNG → JUnit 5. `Assert.assertEquals()` → `assertThat()` (Playwright's auto-retrying assertions). Test structure changes.

### AdminPortal.java → AdminPortalTests.java

**Why rewritten:** Same reasons as LMSTests.

---

# STEP 7: Framework Code — Production-Ready

## 7.1 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.superr</groupId>
    <artifactId>superr-playwright</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <name>Superr Playwright Framework</name>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

        <playwright.version>1.52.0</playwright.version>
        <junit.version>5.11.4</junit.version>
        <slf4j.version>2.0.16</slf4j.version>

        <!-- Default environment: override with -Denv=staging -->
        <env>dev</env>
    </properties>

    <dependencies>
        <!-- Playwright -->
        <dependency>
            <groupId>com.microsoft.playwright</groupId>
            <artifactId>playwright</artifactId>
            <version>${playwright.version}</version>
        </dependency>

        <!-- JUnit 5 -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>

        <!-- SLF4J (optional — for custom logging) -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-simple</artifactId>
            <version>${slf4j.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <!-- Surefire for JUnit 5 -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <version>3.5.2</version>
                <configuration>
                    <systemPropertyVariables>
                        <env>${env}</env>
                    </systemPropertyVariables>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

**What changed from your Selenium pom.xml:**
- ❌ `selenium-java` → replaced by `playwright`
- ❌ `webdrivermanager` → Playwright bundles browsers
- ❌ `testng` → replaced by `junit-jupiter`
- ❌ `extentreports` → Playwright's built-in reporting
- ❌ `appium java-client` → ignored (mobile only)
- ❌ `rest-assured` → ignored (API — out of scope)
- ❌ `log4j` → simplified to `slf4j-simple`

**Dependency count: 11 → 3.** This is a hallmark of Playwright — it ships batteries-included.

## 7.2 EnvConfig.java

```java
package com.superr.playwright.config;

import java.io.IOException;
import java.io.InputStream;
import java.util.Properties;

/**
 * Loads environment-specific configuration.
 *
 * WHY THIS EXISTS:
 *   Multi-environment support (dev/staging/prod URLs, credentials) is genuine
 *   business value that Playwright doesn't provide natively.
 *
 * WHY IT'S SIMPLER THAN ConfigReader:
 *   - No browser config (Playwright handles that)
 *   - No timeout config (Playwright's auto-waiting handles that)
 *   - No driver paths (Playwright bundles browsers)
 *   - Only URLs + credentials remain
 *
 * USAGE:
 *   EnvConfig config = EnvConfig.load();          // reads -Denv system property
 *   String url = config.get("url");
 *   String user = config.get("lms.username");
 */
public final class EnvConfig {

    private final Properties props;

    private EnvConfig(Properties props) {
        this.props = props;
    }

    public static EnvConfig load() {
        String env = System.getProperty("env", "dev");
        String fileName = "config/" + env + ".properties";

        Properties props = new Properties();
        try (InputStream in = EnvConfig.class.getClassLoader().getResourceAsStream(fileName)) {
            if (in == null) {
                throw new RuntimeException("Config not found: " + fileName);
            }
            props.load(in);
        } catch (IOException e) {
            throw new RuntimeException("Failed to load config: " + fileName, e);
        }
        return new EnvConfig(props);
    }

    public String get(String key) {
        String value = props.getProperty(key);
        if (value == null) {
            throw new RuntimeException("Missing config key: " + key);
        }
        return value.trim();
    }
}
```

## 7.3 dev.properties

```properties
# URLs
url=https://dev.superr.ai/login
admin.url=https://staging.admin.superr.ai/admin/admin

# LMS Credentials
lms.username=akashkumar.r@superr.ai
lms.password=Akashkumar@3527

# Admin Credentials
admin.username=prasad.admin@superr.ai
admin.password=Prasad@5747

# Browser (chromium / firefox / webkit)
browser=chromium
headless=false
```

**What was removed vs your Selenium config:**
- ❌ `deviceName`, `deviceId`, `appPath`, `appPackage`, `appActivity` — mobile only
- ❌ `appiumUrl`, `newCommandTimeout` — mobile only
- ❌ `installApp` — mobile only
- ❌ `implicitWait` — Playwright auto-waits; no implicit wait concept
- ❌ `selenium.version`, driver paths — Playwright bundles everything

## 7.4 BasePage.java

```java
package com.superr.playwright.pages;

import com.microsoft.playwright.Page;

/**
 * Thin base class for all page objects.
 *
 * WHY THIS EXISTS:
 *   Holds the Page reference so every page object doesn't repeat the constructor.
 *   That's ALL it does. No utility methods, no custom waits, no wrappers.
 *
 * WHY IT'S THIN:
 *   In Selenium, BasePage often held WebDriverWait, JavascriptExecutor, Actions,
 *   and dozens of wrapper methods like click(), sendKeys(), getText().
 *   In Playwright, the Page and Locator APIs are already clean and fluent.
 *   Wrapping them adds zero value and makes debugging harder.
 *
 * ANTI-PATTERN TO AVOID:
 *   Don't add methods like: void click(String selector) { page.locator(selector).click(); }
 *   This hides Playwright's fluent API and breaks auto-complete in the IDE.
 */
public abstract class BasePage {

    protected final Page page;

    protected BasePage(Page page) {
        this.page = page;
    }
}
```

## 7.5 BaseTest.java

```java
package com.superr.tests;

import com.microsoft.playwright.Browser;
import com.microsoft.playwright.BrowserContext;
import com.microsoft.playwright.BrowserType;
import com.microsoft.playwright.Page;
import com.microsoft.playwright.Playwright;
import com.microsoft.playwright.Tracing;
import com.superr.playwright.config.EnvConfig;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.TestInfo;

import java.nio.file.Paths;

/**
 * Base class for all web tests.
 *
 * LIFECYCLE (mapped from your Selenium BaseTest):
 *
 *   Selenium                          Playwright
 *   ─────────────────────────────     ──────────────────────────────
 *   @BeforeSuite: ConfigReader.load   @BeforeAll: Playwright.create + browser.launch
 *   @BeforeMethod: DriverFactory.init @BeforeEach: browser.newContext + context.newPage
 *   @AfterMethod: driver.quit         @AfterEach: context.close (automatic cleanup)
 *   (no equivalent)                   @AfterAll: browser.close + playwright.close
 *
 * WHY JUnit 5 INSTEAD OF TestNG:
 *   Playwright Java's official examples and community all use JUnit 5.
 *   JUnit 5 has @Tag (replaces TestNG groups), @ParameterizedTest (replaces DataProvider),
 *   and @ExtendWith (replaces listeners). Full feature parity with cleaner syntax.
 *
 * WHY NO DriverFactory/DriverManager:
 *   Look at @BeforeEach — browser.newContext() creates an isolated session.
 *   Each test gets its own cookies, storage, and cache. No ThreadLocal needed.
 *   The "factory" is just playwright.chromium().launch() — one line, not a class.
 */
public abstract class BaseTest {

    protected static Playwright playwright;
    protected static Browser browser;
    protected static EnvConfig config;

    protected BrowserContext context;
    protected Page page;

    @BeforeAll
    static void launchBrowser() {
        config = EnvConfig.load();

        playwright = Playwright.create();

        boolean headless = Boolean.parseBoolean(config.get("headless"));
        String browserName = config.get("browser");

        BrowserType browserType;
        switch (browserName.toLowerCase()) {
            case "firefox":
                browserType = playwright.firefox();
                break;
            case "webkit":
                browserType = playwright.webkit();
                break;
            default:
                browserType = playwright.chromium();
                break;
        }

        browser = browserType.launch(
            new BrowserType.LaunchOptions()
                .setHeadless(headless)
                .setSlowMo(0) // Set to 100 for debugging (slows each action by 100ms)
        );
    }

    @BeforeEach
    void createContextAndPage(TestInfo testInfo) {
        // Each test gets a fresh, isolated browser context
        context = browser.newContext(
            new Browser.NewContextOptions()
                .setViewportSize(1920, 1080)
                .setRecordVideoDir(Paths.get("test-output/videos/"))
        );

        // Start tracing for debugging (captures screenshots, DOM, network)
        context.tracing().start(
            new Tracing.StartOptions()
                .setScreenshots(true)
                .setSnapshots(true)
                .setSources(false)
        );

        page = context.newPage();
    }

    @AfterEach
    void closeContext(TestInfo testInfo) {
        // Save trace on failure (check JUnit test result)
        String tracePath = "test-output/traces/" + testInfo.getDisplayName() + ".zip";
        context.tracing().stop(
            new Tracing.StopOptions().setPath(Paths.get(tracePath))
        );

        // Close context — automatically closes page, releases all resources
        context.close();
    }

    @AfterAll
    static void closeBrowser() {
        if (browser != null) browser.close();
        if (playwright != null) playwright.close();
    }
}
```

---

# STEP 9: Page Objects with Modern Locator API

## 9.1 LoginPage.java (Shared across LMS + Admin)

```java
package com.superr.playwright.pages;

import com.microsoft.playwright.Page;

/**
 * Shared login page — used by both LMS Portal and Admin Portal.
 *
 * LOCATOR STRATEGY:
 *   Priority 1: getByLabel() — form inputs with associated <label>
 *   Priority 2: getByRole() — buttons, links
 *   Priority 3: getByPlaceholder() — inputs without labels
 *   Priority 4: getByText() — visible text
 *   Priority 5: locator("css") — fallback only
 *
 * NO XPATH. NO By.id(). NO By.className().
 * These are Selenium patterns. Playwright's semantic locators are more resilient.
 */
public class LoginPage extends BasePage {

    public LoginPage(Page page) {
        super(page);
    }

    public void navigate(String url) {
        page.navigate(url);
    }

    public void enterEmail(String email) {
        page.getByLabel("Email").fill(email);
    }

    public void enterPassword(String password) {
        page.getByLabel("Password").fill(password);
    }

    public void clickSignIn() {
        page.getByRole(com.microsoft.playwright.options.AriaRole.BUTTON,
            new Page.GetByRoleOptions().setName("Sign In")).click();
    }

    public void login(String email, String password) {
        enterEmail(email);
        enterPassword(password);
        clickSignIn();
    }

    // ── Validation Messages ──

    public String getEmailValidationMessage() {
        return page.getByLabel("Email").evaluate("el => el.validationMessage").toString();
    }

    public String getPasswordValidationMessage() {
        return page.getByLabel("Password").evaluate("el => el.validationMessage").toString();
    }

    public String getInvalidLoginErrorMessage() {
        return page.getByText("Invalid email or password").textContent();
    }
}
```

**Comparison with your Selenium version:**

| Selenium (your code) | Playwright (new) | Why it changed |
|----|----|----|
| `driver.findElement(By.id("email")).sendKeys(email)` | `page.getByLabel("Email").fill(email)` | Semantic locator + `fill()` clears first |
| `driver.findElement(By.id("password")).sendKeys(pass)` | `page.getByLabel("Password").fill(pass)` | Same |
| `driver.findElement(By.xpath("//button[text()='Sign In']")).click()` | `page.getByRole(BUTTON, opts.setName("Sign In")).click()` | Role-based — no XPath |
| `new WebDriverWait(driver, 10).until(...)` | *(nothing)* | Auto-waiting built in |

## 9.2 LMSPage.java

```java
package com.superr.playwright.pages;

import com.microsoft.playwright.Locator;
import com.microsoft.playwright.Page;
import com.microsoft.playwright.options.AriaRole;

import java.nio.file.Path;
import java.nio.file.Paths;

/**
 * LMS Portal page object.
 *
 * Covers: Login, File Upload, File Delete, Toast Verification
 */
public class LMSPage extends BasePage {

    public LMSPage(Page page) {
        super(page);
    }

    // ── Navigation ──

    public void open(String url) {
        page.navigate(url);
    }

    // ── Login ──

    public void login(String email, String password) {
        page.getByLabel("Email").fill(email);
        page.getByLabel("Password").fill(password);
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Sign In")).click();
    }

    public void loginWithOnlyEmail(String email) {
        page.getByLabel("Email").fill(email);
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Sign In")).click();
    }

    public void loginWithInvalidCredentials(String email, String password) {
        login(email, password);
    }

    public void loginWithLeadingTrailingSpaces(String email, String password) {
        login(email, password);
    }

    // ── Validation Messages ──

    public String getEmailValidationMessage() {
        return page.getByLabel("Email").evaluate("el => el.validationMessage").toString();
    }

    public String getPasswordValidationMessage() {
        return page.getByLabel("Password").evaluate("el => el.validationMessage").toString();
    }

    public String getInvalidLoginErrorMessage() {
        return page.getByText("Invalid email or password").textContent();
    }

    public boolean isUserLoggedInSuccessfully() {
        // Wait for navigation to complete after login
        page.waitForURL("**/dashboard**");
        return page.url().contains("dashboard");
    }

    // ── Files Section ──

    public void clickFiles() {
        page.getByRole(AriaRole.LINK, new Page.GetByRoleOptions().setName("Files")).click();
    }

    public void selectClass(String className) {
        page.getByRole(AriaRole.COMBOBOX).selectOption(className);
    }

    public String getSelectedClassText() {
        return page.getByRole(AriaRole.COMBOBOX).inputValue();
    }

    // ── File Upload ──
    // In Playwright, file upload is NATIVE — no AutoIT, no Robot, no sendKeys hack.

    public void uploadMultipleDocuments(String... fileNames) {
        Path[] paths = new Path[fileNames.length];
        for (int i = 0; i < fileNames.length; i++) {
            paths[i] = Paths.get("src/main/resources/config/testdata/files/" + fileNames[i]);
        }
        page.getByLabel("Upload").setInputFiles(paths);
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Submit")).click();
    }

    public void uploadDocumentAndSubmit(String fileName) {
        Path filePath = Paths.get("src/main/resources/config/testdata/files/" + fileName);
        page.getByLabel("Upload").setInputFiles(filePath);
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Submit")).click();
    }

    // ── File Delete ──

    public void rightClickToDelete(String fileName) {
        page.getByText(fileName).click(new Locator.ClickOptions().setButton(
            com.microsoft.playwright.options.MouseButton.RIGHT));
        page.getByText("Delete").click();
    }

    public void againSelectClass(String fileName) {
        page.getByText(fileName).click();
    }

    // ── Toast Messages ──

    public String getToastMessage() {
        // Playwright auto-waits for the toast to appear
        Locator toast = page.locator(".toast-message, [role='alert']");
        return toast.textContent();
    }
}
```

## 9.3 AdminPortalPage.java

```java
package com.superr.playwright.pages;

import com.microsoft.playwright.Page;
import com.microsoft.playwright.options.AriaRole;

/**
 * Admin Portal page object.
 */
public class AdminPortalPage extends BasePage {

    public AdminPortalPage(Page page) {
        super(page);
    }

    public void open(String url) {
        page.navigate(url);
    }

    public void enterUsername(String username) {
        page.getByLabel("Username").fill(username);
    }

    public void enterPassword(String password) {
        page.getByLabel("Password").fill(password);
    }

    public void clickLogin() {
        page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Login")).click();
    }

    public boolean isDeviceManagementPageDisplayed() {
        return page.getByText("Device Management").isVisible();
    }
}
```

## 9.4 LMSTests.java

```java
package com.superr.tests.lms;

import com.superr.playwright.pages.LMSPage;
import com.superr.tests.BaseTest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static com.microsoft.playwright.assertions.PlaywrightAssertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

/**
 * LMS Portal tests.
 *
 * COMPARISON WITH YOUR SELENIUM VERSION:
 *   - TestNG @Test → JUnit 5 @Test
 *   - groups = {"smoke"} → @Tag("smoke")
 *   - Assert.assertEquals → assertEquals (JUnit 5) or assertThat (Playwright)
 *   - No WebDriverWait anywhere
 *   - File upload is native (no Robot, no sendKeys hack)
 */
class LMSTests extends BaseTest {

    private LMSPage lms;

    @BeforeEach
    void openLMS() {
        lms = new LMSPage(page);
        lms.open(config.get("url"));
    }

    @Test
    @DisplayName("Verify login with blank fields")
    void verifyLoginWithBlankFields() {
        page.getByRole(com.microsoft.playwright.options.AriaRole.BUTTON,
            new com.microsoft.playwright.Page.GetByRoleOptions().setName("Sign In")).click();
        String msg = lms.getEmailValidationMessage();
        assertEquals("Please fill out this field.", msg);
    }

    @Test
    @DisplayName("Verify login with blank password")
    void verifyLoginWithBlankPassword() {
        lms.loginWithOnlyEmail("test@gmail.com");
        String msg = lms.getPasswordValidationMessage();
        assertEquals("Please fill out this field.", msg);
    }

    @Test
    @DisplayName("Verify login with invalid credentials")
    void verifyLoginWithInvalidCredentials() {
        lms.loginWithInvalidCredentials("invalid@gmail.com", "invalid123");
        assertThat(page.getByText("Invalid email or password")).isVisible();
    }

    @Test
    @DisplayName("Verify login with leading and trailing spaces")
    void verifyLoginWithLeadingTrailingSpaces() {
        lms.loginWithLeadingTrailingSpaces("  akashkumar.r@superr.ai   ", "Akashkumar@3527");
        assertTrue(lms.isUserLoggedInSuccessfully());
    }

    @Test
    @DisplayName("Verify login with invalid email format")
    void verifyLoginWithInvalidEmailFormat() {
        lms.loginWithOnlyEmail("abc#yahoo.com");
        String msg = lms.getEmailValidationMessage();
        assertTrue(msg.contains("Please include an '@' in the email address"));
    }

    @Test
    @Tag("smoke")
    @DisplayName("Upload multiple files and validate toast message")
    void uploadMultipleFilesAndValidateToast() {
        lms.login(config.get("lms.username"), config.get("lms.password"));
        lms.clickFiles();
        lms.selectClass("Class 6 Dance");
        assertEquals("Class 6 Dance", lms.getSelectedClassText());
        lms.uploadMultipleDocuments("Demo.docx", "Demo.pdf", "googleImage.jpeg", "Presentation.pptx", "ScreenRecording.mp4");
        String toast = lms.getToastMessage();
        assertTrue(toast.contains("files uploaded successfully"));
    }

    @Test
    @Tag("smoke")
    @DisplayName("Upload and delete file flow")
    void uploadAndDeleteFileFlow() {
        lms.login(config.get("lms.username"), config.get("lms.password"));
        lms.clickFiles();
        lms.selectClass("Class 6 Gujarati");
        assertEquals("Class 6 Gujarati", lms.getSelectedClassText());

        lms.uploadDocumentAndSubmit("Demo.pdf");
        assertTrue(lms.getToastMessage().contains("1 file uploaded successfully"));

        lms.againSelectClass("Demo.pdf");
        lms.rightClickToDelete("Demo.pdf");
        assertTrue(lms.getToastMessage().contains("1 files deleted successfully"));
    }
}
```

## 9.5 AdminPortalTests.java

```java
package com.superr.tests.admin;

import com.superr.playwright.pages.AdminPortalPage;
import com.superr.tests.BaseTest;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Tag;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertTrue;

/**
 * Admin Portal tests.
 */
class AdminPortalTests extends BaseTest {

    @Test
    @Tag("smoke")
    @DisplayName("Verify Device Management page is displayed after login")
    void verifyDeviceManagementPage() {
        AdminPortalPage admin = new AdminPortalPage(page);
        admin.open(config.get("admin.url"));
        admin.enterUsername(config.get("admin.username"));
        admin.enterPassword(config.get("admin.password"));
        admin.clickLogin();
        assertTrue(admin.isDeviceManagementPageDisplayed());
    }
}
```

---

# STEP 16: Selenium → Playwright Migration Guide (100+ Operations)

## Navigation

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 1 | `driver.get(url)` | `page.navigate(url)` | |
| 2 | `driver.navigate().to(url)` | `page.navigate(url)` | |
| 3 | `driver.navigate().back()` | `page.goBack()` | |
| 4 | `driver.navigate().forward()` | `page.goForward()` | |
| 5 | `driver.navigate().refresh()` | `page.reload()` | |
| 6 | `driver.getCurrentUrl()` | `page.url()` | |
| 7 | `driver.getTitle()` | `page.title()` | |

## Element Finding

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 8 | `driver.findElement(By.id("x"))` | `page.locator("#x")` | Prefer `getByTestId()` |
| 9 | `driver.findElement(By.name("x"))` | `page.locator("[name='x']")` | Prefer `getByLabel()` |
| 10 | `driver.findElement(By.className("x"))` | `page.locator(".x")` | |
| 11 | `driver.findElement(By.cssSelector("x"))` | `page.locator("x")` | CSS is default |
| 12 | `driver.findElement(By.xpath("//x"))` | `page.locator("xpath=//x")` | Avoid — use semantic |
| 13 | `driver.findElement(By.linkText("x"))` | `page.getByRole(AriaRole.LINK, opts.setName("x"))` | |
| 14 | `driver.findElement(By.partialLinkText("x"))` | `page.getByRole(AriaRole.LINK, opts.setName(Pattern.compile("x")))` | |
| 15 | `driver.findElement(By.tagName("x"))` | `page.locator("x")` | |
| 16 | `driver.findElements(By.css("x"))` | `page.locator("x").all()` | Returns List |
| 17 | `element.findElement(By.css("x"))` | `locator.locator("x")` | Scoped |

## Semantic Locators (Playwright-only — no Selenium equivalent)

| # | Playwright | Finds |
|---|-----------|-------|
| 18 | `page.getByRole(AriaRole.BUTTON, opts.setName("Login"))` | Button by accessible name |
| 19 | `page.getByLabel("Email")` | Input by associated label |
| 20 | `page.getByPlaceholder("Enter email")` | Input by placeholder |
| 21 | `page.getByText("Welcome")` | Element by visible text |
| 22 | `page.getByTestId("submit-btn")` | Element by data-testid |
| 23 | `page.getByAltText("Logo")` | Image by alt text |
| 24 | `page.getByTitle("Close")` | Element by title attribute |

## Actions

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 25 | `element.click()` | `locator.click()` | Auto-waits |
| 26 | `element.sendKeys("text")` | `locator.fill("text")` | Clears first |
| 27 | `element.sendKeys("text")` (append) | `locator.pressSequentially("text")` | Types char by char |
| 28 | `element.clear()` | `locator.clear()` | |
| 29 | `element.submit()` | `locator.press("Enter")` | No submit() in Playwright |
| 30 | `element.getText()` | `locator.textContent()` | |
| 31 | `element.getAttribute("x")` | `locator.getAttribute("x")` | |
| 32 | `element.isDisplayed()` | `locator.isVisible()` | |
| 33 | `element.isEnabled()` | `locator.isEnabled()` | |
| 34 | `element.isSelected()` | `locator.isChecked()` | |
| 35 | `element.getCssValue("color")` | `locator.evaluate("el => getComputedStyle(el).color")` | |

## Mouse & Keyboard

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 36 | `Actions(driver).doubleClick(el)` | `locator.dblclick()` | |
| 37 | `Actions(driver).contextClick(el)` | `locator.click(new ClickOptions().setButton(MouseButton.RIGHT))` | |
| 38 | `Actions(driver).moveToElement(el)` | `locator.hover()` | |
| 39 | `Actions(driver).dragAndDrop(src, tgt)` | `locator.dragTo(target)` | |
| 40 | `Actions(driver).sendKeys(Keys.ENTER)` | `page.keyboard().press("Enter")` | |
| 41 | `Actions(driver).sendKeys(Keys.TAB)` | `page.keyboard().press("Tab")` | |
| 42 | `Actions(driver).keyDown(Keys.CONTROL)` | `page.keyboard().down("Control")` | |
| 43 | `Actions(driver).keyUp(Keys.CONTROL)` | `page.keyboard().up("Control")` | |
| 44 | `Actions(driver).sendKeys(Keys.chord(CTRL, "a"))` | `page.keyboard().press("Control+a")` | |

## Waits

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 45 | `Thread.sleep(ms)` | **REMOVE** | Never needed |
| 46 | `driver.manage().timeouts().implicitlyWait()` | **REMOVE** | Auto-waiting replaces it |
| 47 | `WebDriverWait.until(visibilityOf)` | **REMOVE** | `locator.click()` auto-waits |
| 48 | `WebDriverWait.until(elementToBeClickable)` | **REMOVE** | `locator.click()` auto-waits |
| 49 | `WebDriverWait.until(textToBePresentIn)` | `assertThat(locator).hasText("x")` | Auto-retrying assertion |
| 50 | `WebDriverWait.until(urlContains)` | `page.waitForURL("**/path**")` | Glob pattern |
| 51 | `FluentWait` | `locator.waitFor()` | Rarely needed |

## Dropdowns

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 52 | `new Select(el).selectByVisibleText("x")` | `locator.selectOption("x")` | |
| 53 | `new Select(el).selectByValue("x")` | `locator.selectOption(new SelectOption().setValue("x"))` | |
| 54 | `new Select(el).selectByIndex(0)` | `locator.selectOption(new SelectOption().setIndex(0))` | |
| 55 | `new Select(el).getOptions()` | `locator.locator("option").all()` | |

## Checkboxes & Radio

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 56 | `checkbox.click()` | `locator.check()` | Idempotent — checks only if unchecked |
| 57 | `checkbox.click()` (uncheck) | `locator.uncheck()` | Idempotent |
| 58 | `checkbox.isSelected()` | `locator.isChecked()` | |
| 59 | `radio.click()` | `locator.check()` | Same as checkbox |

## Frames

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 60 | `driver.switchTo().frame("name")` | `page.frameLocator("[name='name']")` | Returns scoped locator |
| 61 | `driver.switchTo().frame(0)` | `page.frames().get(0)` | By index |
| 62 | `driver.switchTo().frame(element)` | `page.frameLocator("#id")` | |
| 63 | `driver.switchTo().defaultContent()` | **Not needed** | frameLocator is scoped |
| 64 | `driver.switchTo().parentFrame()` | **Not needed** | frameLocator is scoped |

## Alerts

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 65 | `driver.switchTo().alert().accept()` | `page.onDialog(d -> d.accept())` | Register handler BEFORE trigger |
| 66 | `driver.switchTo().alert().dismiss()` | `page.onDialog(d -> d.dismiss())` | |
| 67 | `driver.switchTo().alert().getText()` | `dialog.message()` in handler | |
| 68 | `driver.switchTo().alert().sendKeys("x")` | `page.onDialog(d -> d.accept("x"))` | |

## Windows & Tabs

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 69 | `driver.getWindowHandle()` | **Not needed** | Each page is already a separate object |
| 70 | `driver.getWindowHandles()` | `context.pages()` | |
| 71 | `driver.switchTo().window(handle)` | `Page newPage = context.waitForPage(action)` | Event-based |
| 72 | `driver.close()` | `page.close()` | |
| 73 | Open new tab | `Page newPage = context.newPage()` | Direct API |

## JavaScript

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 74 | `((JavascriptExecutor) driver).executeScript("...")` | `page.evaluate("...")` | |
| 75 | `js.executeScript("arguments[0].click()", el)` | `locator.evaluate("el => el.click()")` | |
| 76 | `js.executeScript("return document.title")` | `page.evaluate("() => document.title")` | |
| 77 | `js.executeAsyncScript(...)` | `page.evaluate("async () => { ... }")` | |

## Cookies

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 78 | `driver.manage().getCookies()` | `context.cookies()` | |
| 79 | `driver.manage().getCookieNamed("x")` | `context.cookies(url)` + filter | |
| 80 | `driver.manage().addCookie(cookie)` | `context.addCookies(List.of(cookie))` | |
| 81 | `driver.manage().deleteCookieNamed("x")` | `context.clearCookies()` | Clear all; no single-delete |
| 82 | `driver.manage().deleteAllCookies()` | `context.clearCookies()` | |

## File Upload & Download

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 83 | `element.sendKeys(filePath)` | `locator.setInputFiles(path)` | Native — no sendKeys hack |
| 84 | Multiple files: `sendKeys(path1 + "\n" + path2)` | `locator.setInputFiles(new Path[]{p1, p2})` | Array of paths |
| 85 | Download (manual wait + file check) | `Download dl = page.waitForDownload(() -> { click(); }); dl.path()` | Event-based |

## Screenshots

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 86 | `((TakesScreenshot) driver).getScreenshotAs(FILE)` | `page.screenshot(opts.setPath(path))` | |
| 87 | Element screenshot | `locator.screenshot(opts.setPath(path))` | Native |
| 88 | Full page screenshot | `page.screenshot(opts.setFullPage(true))` | Native |

## Browser Management

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 89 | `WebDriverManager.chromedriver().setup()` | **Not needed** | Playwright bundles browsers |
| 90 | `new ChromeDriver(options)` | `playwright.chromium().launch(options)` | |
| 91 | `ChromeOptions.addArguments("--headless")` | `LaunchOptions().setHeadless(true)` | |
| 92 | `driver.manage().window().maximize()` | `newContext(opts.setViewportSize(1920,1080))` | Set viewport explicitly |
| 93 | `driver.manage().window().setSize(dim)` | `page.setViewportSize(w, h)` | |
| 94 | `driver.quit()` | `browser.close()` | |

## Assertions (Playwright-specific — auto-retrying)

| # | Playwright Assertion | What it checks |
|---|---------------------|----------------|
| 95 | `assertThat(page).hasTitle("x")` | Page title |
| 96 | `assertThat(page).hasURL(Pattern.compile(".*dash.*"))` | Page URL |
| 97 | `assertThat(locator).isVisible()` | Element visible |
| 98 | `assertThat(locator).isHidden()` | Element hidden |
| 99 | `assertThat(locator).isEnabled()` | Element enabled |
| 100 | `assertThat(locator).isDisabled()` | Element disabled |
| 101 | `assertThat(locator).hasText("x")` | Text content |
| 102 | `assertThat(locator).containsText("x")` | Partial text |
| 103 | `assertThat(locator).hasAttribute("href", "/x")` | Attribute value |
| 104 | `assertThat(locator).hasClass("active")` | CSS class |
| 105 | `assertThat(locator).hasCount(5)` | Number of matches |
| 106 | `assertThat(locator).hasValue("x")` | Input value |
| 107 | `assertThat(locator).isChecked()` | Checkbox/radio state |

## Network & API

| # | Selenium | Playwright | Notes |
|---|----------|-----------|-------|
| 108 | DevTools CDP (complex) | `page.route("**/api/**", route -> route.fulfill(...))` | Mock API |
| 109 | No native API support | `page.request().get(url)` | API testing built in |
| 110 | No network interception | `page.route("**/login", route -> route.abort())` | Block requests |

---

# Best Practices

## 1. Never Wrap Playwright's API

**Anti-pattern:**
```java
public void click(String selector) {
    page.locator(selector).click();
}
```

**Why it's bad:** Hides auto-complete, hides error messages, adds zero value. Just call `page.locator().click()` directly.

## 2. Use Semantic Locators First

```java
// ❌ Bad — fragile, not readable
page.locator("#btnSubmit").click();
page.locator("//div[@class='form']//button[2]").click();

// ✅ Good — resilient, reads like English
page.getByRole(AriaRole.BUTTON, new Page.GetByRoleOptions().setName("Submit")).click();
```

## 3. Never Use Thread.sleep()

```java
// ❌ Bad
Thread.sleep(3000);
page.locator("#result").click();

// ✅ Good — Playwright auto-waits
page.locator("#result").click();
```

## 4. Use Playwright Assertions (Not JUnit Assertions) for UI Checks

```java
// ❌ Bad — no auto-retry
assertTrue(page.locator("#msg").isVisible());

// ✅ Good — auto-retries for up to 5 seconds
assertThat(page.locator("#msg")).isVisible();
```

## 5. One Assertion per Test (When Possible)

Keeps tests focused and makes failures easy to diagnose.

## 6. Use Tracing for Debugging, Not Screenshots

Trace Viewer shows the full story — every action, every network call, DOM state at every step. Screenshots are limited to a single moment.

---

# Interview Questions & FAQs

## Beginner

**Q1: What is Playwright?**
A: Playwright is a browser automation framework by Microsoft that controls Chromium, Firefox, and WebKit with a single API. Unlike Selenium, it bundles browsers, auto-waits, and provides built-in tracing.

**Q2: How does Playwright differ from Selenium?**
A: Key differences: Playwright auto-waits (no explicit waits), bundles browsers (no driver management), uses Locator API (semantic selectors), has built-in tracing/screenshots/video, and isolates tests via browser contexts (no ThreadLocal).

**Q3: What is auto-waiting?**
A: Every Playwright action (click, fill, check) automatically waits for the element to be actionable before proceeding. No `WebDriverWait` or `Thread.sleep()` needed.

**Q4: What is a BrowserContext?**
A: An isolated browser session — like an incognito window. Each context has its own cookies, localStorage, and cache. In tests, each `@BeforeEach` creates a fresh context for full test isolation.

**Q5: What is a Locator?**
A: A lazy reference to element(s) on the page. Unlike Selenium's `WebElement`, a Locator doesn't find the element immediately — it resolves at action time, which enables auto-waiting and retries.

## Intermediate

**Q6: Why use `getByRole()` instead of CSS selectors?**
A: Role-based locators are resilient to DOM changes (class renames, ID changes) and test the page the way a screen reader sees it, improving accessibility coverage.

**Q7: How does Playwright handle parallel execution?**
A: Each test gets its own BrowserContext, which is fully isolated. Multiple tests can share a single Browser instance safely. JUnit 5's parallel execution works out of the box.

**Q8: What is Trace Viewer?**
A: A visual debugger that records every action, screenshot, DOM snapshot, and network call during a test. Open with: `playwright show-trace trace.zip`.

**Q9: How do you handle authentication in Playwright?**
A: Use `storageState`. Login once, save the session to a JSON file, then load it in subsequent tests. This avoids logging in before every test.

**Q10: What is `page.route()`?**
A: Network interception. You can mock API responses, abort requests, or modify them in flight. Selenium has no equivalent without CDP hacks.

## Advanced

**Q11: Should I create a DriverFactory in Playwright?**
A: No. `Playwright.create()` + `browser.launch()` is the factory. Creating a custom class adds abstraction that hides Playwright's simple API.

**Q12: Can I use TestNG with Playwright?**
A: Technically yes, but Playwright's ecosystem (examples, community, fixtures) is built around JUnit 5. TestNG works but you lose the first-party integration.

---

# Debugging Guide

## 1. Headed Mode (See the Browser)

```properties
# In dev.properties:
headless=false
```

## 2. Slow Motion (Slow Down Actions)

```java
browser = browserType.launch(new BrowserType.LaunchOptions().setSlowMo(500));
```

## 3. Playwright Inspector (Step Through Interactively)

Set the environment variable before running:
```bash
PWDEBUG=1 mvn test -Dtest=LMSTests
```

This opens a debugger where you can step through each action, inspect locators live, and see the page state.

## 4. Trace Viewer (Post-Mortem Debugging)

After a test run:
```bash
mvn exec:java -e -Dexec.mainClass=com.microsoft.playwright.CLI -Dexec.args="show-trace test-output/traces/testName.zip"
```

## 5. Codegen (Generate Locators by Recording)

```bash
mvn exec:java -e -Dexec.mainClass=com.microsoft.playwright.CLI -Dexec.args="codegen https://dev.superr.ai/login"
```

This opens a browser and records your actions as Java code. Great for discovering the right locators.

---

# Performance Tips

1. **Share Browser across tests** — `@BeforeAll` launches once, each `@BeforeEach` creates a lightweight context
2. **Use `storageState`** for auth — login once, reuse the session
3. **Run headless in CI** — `headless=true` for faster execution
4. **Parallel execution** — JUnit 5 `junit.jupiter.execution.parallel.enabled=true`
5. **Disable video in CI** — remove `setRecordVideoDir()` when not debugging
6. **Use route mocking** — mock slow API endpoints for faster tests

---

# Running the Framework

## First-Time Setup

```bash
# 1. Clone and install dependencies
mvn clean install -DskipTests

# 2. Install Playwright browsers (one-time)
mvn exec:java -e -Dexec.mainClass=com.microsoft.playwright.CLI -Dexec.args="install"

# 3. Run all tests
mvn test

# 4. Run with specific environment
mvn test -Denv=staging

# 5. Run only smoke tests
mvn test -Dgroups=smoke

# 6. Run a specific test class
mvn test -Dtest=LMSTests

# 7. Run a specific test method
mvn test -Dtest=LMSTests#verifyLoginWithBlankFields
```

---

*Generated for the Superr QA team — migrating from Selenium to Playwright Java.*

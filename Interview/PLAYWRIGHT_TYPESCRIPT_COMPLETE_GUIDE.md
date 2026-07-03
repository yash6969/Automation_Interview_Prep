# Playwright TypeScript — Complete Enterprise Framework Guide

> **The native Playwright experience — TypeScript is Playwright's primary language.**
>
> This guide covers the same testing scenarios as the Java guide but in TypeScript,
> which is Playwright's first-class, most fully supported language.

---

## Table of Contents

1. [Why TypeScript?](#why-typescript)
2. [Project Setup](#project-setup)
3. [Configuration](#configuration)
4. [Page Objects](#page-objects)
5. [Tests](#tests)
6. [Running Tests](#running-tests)
7. [Key Differences from Java](#key-differences-from-java)
8. [Migration Guide (Selenium Java → Playwright TS)](#migration-guide)
9. [Best Practices](#best-practices)
10. [Interview Questions](#interview-questions)

---

# Why TypeScript?

Playwright was built in TypeScript. The TS version gets:
- Features first (Java lags by ~1-2 months)
- Best documentation and examples
- Built-in test runner (`@playwright/test`) with fixtures, parallelism, retries
- First-class `playwright.config.ts` with full type safety
- Auto-generated HTML reports
- VS Code extension with live debugging

If you're starting fresh with Playwright, TypeScript is the recommended language.

---

# Project Setup

## Initialize

```bash
npm init playwright@latest
# Choose: TypeScript, tests folder, GitHub Actions CI, install browsers
```

This generates:

```
superr-playwright-ts/
├── playwright.config.ts          ← Central configuration
├── package.json
├── tests/
│   ├── lms.spec.ts               ← LMS Portal tests
│   └── admin.spec.ts             ← Admin Portal tests
├── pages/
│   ├── base.page.ts              ← Base page object
│   ├── login.page.ts             ← Login page object
│   ├── lms.page.ts               ← LMS page object
│   └── admin-portal.page.ts      ← Admin portal page object
├── config/
│   ├── dev.config.ts             ← Dev environment data
│   └── staging.config.ts         ← Staging environment data
└── test-results/                 ← Auto-generated reports, traces, videos
```

## package.json

```json
{
  "name": "superr-playwright-ts",
  "version": "1.0.0",
  "devDependencies": {
    "@playwright/test": "^1.52.0"
  },
  "scripts": {
    "test": "npx playwright test",
    "test:headed": "npx playwright test --headed",
    "test:debug": "npx playwright test --debug",
    "test:ui": "npx playwright test --ui",
    "test:smoke": "npx playwright test --grep @smoke",
    "report": "npx playwright show-report",
    "codegen": "npx playwright codegen"
  }
}
```

**Compare with your Selenium pom.xml:** 11 dependencies → 1 dependency. That's it.

---

# Configuration

## playwright.config.ts

```typescript
import { defineConfig, devices } from '@playwright/test';

/**
 * Central Playwright configuration.
 *
 * REPLACES IN YOUR SELENIUM FRAMEWORK:
 *   - config-dev.properties (partially)
 *   - DriverFactory.java (browser launch)
 *   - BaseTest.java (timeouts, setup)
 *   - ExtentManager.java (reporting)
 *   - webSmokeTests.xml (suite configuration)
 *
 * All in ONE file. Type-safe. Auto-complete in VS Code.
 */
export default defineConfig({
  testDir: './tests',
  timeout: 30_000,
  expect: { timeout: 5_000 },

  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,

  reporter: [
    ['html', { open: 'never' }],
    ['list'],
  ],

  use: {
    baseURL: process.env.BASE_URL || 'https://dev.superr.ai',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    headless: true,
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
});
```

## Environment Config

```typescript
// config/env.config.ts

interface EnvConfig {
  url: string;
  adminUrl: string;
  lms: { username: string; password: string };
  admin: { username: string; password: string };
}

const configs: Record<string, EnvConfig> = {
  dev: {
    url: 'https://dev.superr.ai/login',
    adminUrl: 'https://staging.admin.superr.ai/admin/admin',
    lms: { username: 'akashkumar.r@superr.ai', password: 'Akashkumar@3527' },
    admin: { username: 'prasad.admin@superr.ai', password: 'Prasad@5747' },
  },
  staging: {
    url: 'https://staging.superr.ai/login',
    adminUrl: 'https://staging.admin.superr.ai/admin/admin',
    lms: { username: 'akashkumar.r@superr.ai', password: 'Akashkumar@3527' },
    admin: { username: 'prasad.admin@superr.ai', password: 'Prasad@5747' },
  },
};

export function getConfig(): EnvConfig {
  const env = process.env.ENV || 'dev';
  const config = configs[env];
  if (!config) throw new Error(`Unknown environment: ${env}`);
  return config;
}
```

---

# Page Objects

## base.page.ts

```typescript
import { Page } from '@playwright/test';

/**
 * Thin base — holds Page reference only.
 * No wrapper methods. Playwright's API is already clean.
 */
export abstract class BasePage {
  constructor(protected readonly page: Page) {}
}
```

## login.page.ts

```typescript
import { BasePage } from './base.page';

export class LoginPage extends BasePage {
  async navigate(url: string) {
    await this.page.goto(url);
  }

  async login(email: string, password: string) {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Sign In' }).click();
  }

  async getEmailValidationMessage(): Promise<string> {
    return await this.page.getByLabel('Email').evaluate(
      (el: HTMLInputElement) => el.validationMessage
    );
  }

  async getPasswordValidationMessage(): Promise<string> {
    return await this.page.getByLabel('Password').evaluate(
      (el: HTMLInputElement) => el.validationMessage
    );
  }
}
```

## lms.page.ts

```typescript
import { BasePage } from './base.page';
import path from 'path';

export class LMSPage extends BasePage {
  async open(url: string) {
    await this.page.goto(url);
  }

  async login(email: string, password: string) {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByLabel('Password').fill(password);
    await this.page.getByRole('button', { name: 'Sign In' }).click();
  }

  async loginWithOnlyEmail(email: string) {
    await this.page.getByLabel('Email').fill(email);
    await this.page.getByRole('button', { name: 'Sign In' }).click();
  }

  async getEmailValidationMessage(): Promise<string> {
    return await this.page.getByLabel('Email').evaluate(
      (el: HTMLInputElement) => el.validationMessage
    );
  }

  async getPasswordValidationMessage(): Promise<string> {
    return await this.page.getByLabel('Password').evaluate(
      (el: HTMLInputElement) => el.validationMessage
    );
  }

  async getInvalidLoginErrorMessage(): Promise<string> {
    return await this.page.getByText('Invalid email or password').textContent() ?? '';
  }

  async isUserLoggedInSuccessfully(): Promise<boolean> {
    await this.page.waitForURL('**/dashboard**');
    return this.page.url().includes('dashboard');
  }

  async clickFiles() {
    await this.page.getByRole('link', { name: 'Files' }).click();
  }

  async selectClass(className: string) {
    await this.page.getByRole('combobox').selectOption(className);
  }

  async getSelectedClassText(): Promise<string> {
    return await this.page.getByRole('combobox').inputValue();
  }

  async uploadMultipleDocuments(...fileNames: string[]) {
    const paths = fileNames.map(f => path.resolve('test-data', f));
    await this.page.getByLabel('Upload').setInputFiles(paths);
    await this.page.getByRole('button', { name: 'Submit' }).click();
  }

  async uploadDocumentAndSubmit(fileName: string) {
    const filePath = path.resolve('test-data', fileName);
    await this.page.getByLabel('Upload').setInputFiles(filePath);
    await this.page.getByRole('button', { name: 'Submit' }).click();
  }

  async rightClickToDelete(fileName: string) {
    await this.page.getByText(fileName).click({ button: 'right' });
    await this.page.getByText('Delete').click();
  }

  async getToastMessage(): Promise<string> {
    return await this.page.locator('.toast-message, [role="alert"]').textContent() ?? '';
  }
}
```

## admin-portal.page.ts

```typescript
import { BasePage } from './base.page';

export class AdminPortalPage extends BasePage {
  async open(url: string) {
    await this.page.goto(url);
  }

  async enterUsername(username: string) {
    await this.page.getByLabel('Username').fill(username);
  }

  async enterPassword(password: string) {
    await this.page.getByLabel('Password').fill(password);
  }

  async clickLogin() {
    await this.page.getByRole('button', { name: 'Login' }).click();
  }

  async isDeviceManagementPageDisplayed(): Promise<boolean> {
    return await this.page.getByText('Device Management').isVisible();
  }
}
```

---

# Tests

## lms.spec.ts

```typescript
import { test, expect } from '@playwright/test';
import { LMSPage } from '../pages/lms.page';
import { getConfig } from '../config/env.config';

const config = getConfig();

test.describe('LMS Portal', () => {
  let lms: LMSPage;

  test.beforeEach(async ({ page }) => {
    lms = new LMSPage(page);
    await lms.open(config.url);
  });

  test('Verify login with blank fields', async ({ page }) => {
    await page.getByRole('button', { name: 'Sign In' }).click();
    const msg = await lms.getEmailValidationMessage();
    expect(msg).toBe('Please fill out this field.');
  });

  test('Verify login with blank password', async ({ page }) => {
    await lms.loginWithOnlyEmail('test@gmail.com');
    const msg = await lms.getPasswordValidationMessage();
    expect(msg).toBe('Please fill out this field.');
  });

  test('Verify login with invalid credentials', async ({ page }) => {
    await lms.login('invalid@gmail.com', 'invalid123');
    await expect(page.getByText('Invalid email or password')).toBeVisible();
  });

  test('Verify login with leading and trailing spaces', async () => {
    await lms.login('  akashkumar.r@superr.ai   ', 'Akashkumar@3527');
    expect(await lms.isUserLoggedInSuccessfully()).toBeTruthy();
  });

  test('Verify login with invalid email format', async () => {
    await lms.loginWithOnlyEmail('abc#yahoo.com');
    const msg = await lms.getEmailValidationMessage();
    expect(msg).toContain("Please include an '@' in the email address");
  });

  test('Upload multiple files and validate toast @smoke', async () => {
    await lms.login(config.lms.username, config.lms.password);
    await lms.clickFiles();
    await lms.selectClass('Class 6 Dance');
    expect(await lms.getSelectedClassText()).toBe('Class 6 Dance');
    await lms.uploadMultipleDocuments('Demo.docx', 'Demo.pdf', 'googleImage.jpeg', 'Presentation.pptx', 'ScreenRecording.mp4');
    const toast = await lms.getToastMessage();
    expect(toast).toContain('files uploaded successfully');
  });

  test('Upload and delete file flow @smoke', async () => {
    await lms.login(config.lms.username, config.lms.password);
    await lms.clickFiles();
    await lms.selectClass('Class 6 Gujarati');
    expect(await lms.getSelectedClassText()).toBe('Class 6 Gujarati');

    await lms.uploadDocumentAndSubmit('Demo.pdf');
    expect(await lms.getToastMessage()).toContain('1 file uploaded successfully');

    await lms.rightClickToDelete('Demo.pdf');
    expect(await lms.getToastMessage()).toContain('1 files deleted successfully');
  });
});
```

## admin.spec.ts

```typescript
import { test, expect } from '@playwright/test';
import { AdminPortalPage } from '../pages/admin-portal.page';
import { getConfig } from '../config/env.config';

const config = getConfig();

test.describe('Admin Portal', () => {
  test('Verify Device Management page @smoke', async ({ page }) => {
    const admin = new AdminPortalPage(page);
    await admin.open(config.adminUrl);
    await admin.enterUsername(config.admin.username);
    await admin.enterPassword(config.admin.password);
    await admin.clickLogin();
    expect(await admin.isDeviceManagementPageDisplayed()).toBeTruthy();
  });
});
```

---

# Running Tests

```bash
# Run all tests
npx playwright test

# Run in headed mode (see the browser)
npx playwright test --headed

# Run with UI mode (visual debugger)
npx playwright test --ui

# Run specific file
npx playwright test tests/lms.spec.ts

# Run smoke tests only
npx playwright test --grep @smoke

# Run with specific browser
npx playwright test --project=firefox

# Run with staging environment
ENV=staging npx playwright test

# Debug mode (step through)
npx playwright test --debug

# Generate code by recording
npx playwright codegen https://dev.superr.ai/login

# View last report
npx playwright show-report

# View trace file
npx playwright show-trace test-results/trace.zip
```

---

# Key Differences from Java

| Aspect | Java | TypeScript |
|--------|------|------------|
| Test runner | JUnit 5 (external) | `@playwright/test` (built-in) |
| Config | `properties` files + `EnvConfig.java` | `playwright.config.ts` (type-safe) |
| Page objects | Synchronous methods | `async/await` throughout |
| Assertions | `assertThat(locator).isVisible()` | `await expect(locator).toBeVisible()` |
| Fixtures | Manual `@BeforeEach` / `@AfterEach` | Built-in `{ page, context, browser }` fixtures |
| Parallelism | JUnit 5 config | Built-in `fullyParallel: true` |
| Reporting | Manual setup | Automatic HTML report |
| Browser install | `mvn exec:java ... CLI install` | `npx playwright install` |
| Dependencies | 3 (playwright, junit, slf4j) | 1 (`@playwright/test`) |

---

# Best Practices

1. **Use `async/await`** — every Playwright call is async in TypeScript
2. **Use `expect` from `@playwright/test`** — auto-retrying, Playwright-aware
3. **Use `test.describe`** — groups related tests
4. **Use `test.beforeEach`** — setup runs before each test with fresh `page`
5. **Use `@smoke` tags in test names** — filter with `--grep @smoke`
6. **Use `page.goto()` not `page.navigate()`** — TypeScript API uses `goto`
7. **Use `test.use({ storageState })` for auth** — skip login in every test
8. **Use `toBeVisible()` not `isVisible()`** — the assertion auto-retries, the method doesn't

---

# Interview Questions

**Q1: Why TypeScript for Playwright?**
A: Playwright was built in TypeScript. The TS version gets features first, has the best documentation, the built-in test runner, and the richest ecosystem.

**Q2: What are fixtures in Playwright?**
A: Fixtures are dependency-injected test parameters. `{ page }` gives you a fresh page per test. `{ context }` gives you the browser context. `{ browser }` gives you the browser instance. Custom fixtures can provide logged-in pages, test data, etc.

**Q3: How does parallelism work?**
A: `fullyParallel: true` runs all tests in parallel. Each test gets its own browser context (isolated session). Workers control concurrency — default is 50% of CPU cores.

**Q4: What is `storageState`?**
A: A JSON file containing cookies and localStorage. Login once in a `globalSetup`, save to file, then load in tests. Every test starts already authenticated — no login page needed.

**Q5: What is UI Mode?**
A: `npx playwright test --ui` opens a visual test runner where you can watch tests execute, see timeline of actions, inspect DOM at any step, and re-run individual tests. It's like a built-in Trace Viewer for live debugging.

---

*Generated for the Superr QA team — Playwright TypeScript edition.*

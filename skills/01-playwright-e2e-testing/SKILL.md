---
name: playwright-e2e-testing
description: End-to-end, API, and responsive testing for web applications using Playwright with TypeScript. Use when asked to write, run, debug, or maintain Playwright tests for UI behavior, form submissions, user flows, API validation, responsive design, or visual regression.
source: https://github.com/fugazi/test-automation-skills-agents
tier: best-in-class
---

# Playwright E2E Testing (TypeScript)

Comprehensive toolkit for end-to-end testing of web applications using Playwright with TypeScript. Enables robust UI testing, API validation, and responsive design verification following best practices.

## When to Use

- Write E2E tests for user flows, forms, navigation, and authentication
- API testing via `request` fixture or network interception during UI tests
- Responsive testing across mobile, tablet, and desktop viewports
- Debug flaky tests using traces, screenshots, videos, and Playwright Inspector
- Setup test infrastructure with Page Object Model and fixtures
- Mock/intercept APIs for isolated, deterministic testing
- Visual regression testing with screenshot comparisons

## Prerequisites

| Requirement     | Details                             |
|-----------------|-------------------------------------|
| Node.js         | v18+                                |
| Playwright      | `@playwright/test`                  |
| TypeScript      | `typescript` + `ts-node`            |
| Browsers        | `npx playwright install`            |

```bash
npm init playwright@latest
# or add to existing project
npm install -D @playwright/test && npx playwright install
```

## First Questions to Ask

1. **App URL**: Local dev server command + port, or staging URL?
2. **Critical flows**: Which user journeys must be covered (happy path + error states)?
3. **Browsers/devices**: Chrome, Firefox, Safari? Mobile viewports?
4. **API strategy**: Real backend, mocked responses, or hybrid?
5. **Test data**: Seed data available? Reset/cleanup strategy?

---

## Core Principles

### 1. Locator Strategy (Priority Order)

| Priority | Locator                | Example                                   |
|----------|------------------------|-------------------------------------------|
| 1        | Role + accessible name | `getByRole('button', { name: 'Submit' })` |
| 2        | Label                  | `getByLabel('Email')`                     |
| 3        | Placeholder            | `getByPlaceholder('Enter email')`         |
| 4        | Text                   | `getByText('Welcome')`                    |
| 5        | Test ID                | `getByTestId('submit-btn')`               |
| 6        | CSS (avoid)            | `locator('.btn-primary')`                 |

### 2. Auto-Waiting — Never Use sleep()

```typescript
// ✅ Web-first assertions (auto-retry)
await expect(page.getByRole('alert')).toBeVisible();
await expect(page).toHaveURL(/dashboard/);

// ❌ Never do this
await page.waitForTimeout(2000);
```

### 3. Test Structure with Steps

```typescript
test('checkout flow', async ({ page }) => {
  await test.step('Add item to cart', async () => {
    await page.goto('/products/1');
    await page.getByRole('button', { name: 'Add to Cart' }).click();
  });
  await test.step('Complete checkout', async () => {
    await page.goto('/checkout');
    await page.getByRole('button', { name: 'Pay Now' }).click();
  });
  await test.step('Verify confirmation', async () => {
    await expect(page.getByRole('heading')).toContainText('Order Confirmed');
  });
});
```

---

## Key Workflows

### Forms & Navigation

```typescript
test('user can login', async ({ page }) => {
  await page.goto('/login');
  await page.getByLabel('Email').fill('user@test.com');
  await page.getByLabel('Password').fill('password123');
  await page.getByRole('button', { name: 'Sign in' }).click();
  await expect(page).toHaveURL(/.*dashboard/);
});
```

### API Testing (Request Fixture)

```typescript
test('API health check', async ({ request }) => {
  const response = await request.get('/api/health');
  expect(response.ok()).toBeTruthy();
  expect(await response.json()).toMatchObject({ status: 'ok' });
});
```

### API Mocking & Interception

```typescript
test('handles API error', async ({ page }) => {
  await page.route('**/api/users', (route) =>
    route.fulfill({ status: 500, body: JSON.stringify({ error: 'Server error' }) })
  );
  await page.goto('/users');
  await expect(page.getByRole('alert')).toContainText('Something went wrong');
});
```

### Responsive Testing

```typescript
const viewports = [
  { width: 375, height: 667, name: 'mobile' },
  { width: 768, height: 1024, name: 'tablet' },
  { width: 1280, height: 720, name: 'desktop' },
];

for (const vp of viewports) {
  test(`navigation works on ${vp.name}`, async ({ page }) => {
    await page.setViewportSize(vp);
    await page.goto('/');
    if (vp.width < 768) {
      await page.getByRole('button', { name: /menu/i }).click();
    }
    await page.getByRole('link', { name: 'About' }).click();
    await expect(page).toHaveURL(/about/);
  });
}
```

---

## Configuration (playwright.config.ts)

```typescript
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  retries: process.env.CI ? 2 : 0,
  reporter: [['html'], ['junit', { outputFile: 'results.xml' }]],
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: devices['Desktop Chrome'] },
    { name: 'mobile', use: devices['Pixel 5'] },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

---

## Troubleshooting

| Problem                | Solution                                                       |
|------------------------|----------------------------------------------------------------|
| Element not found      | Use `PWDEBUG=1`, verify with `getByRole`                      |
| Timeout waiting        | Check for overlays, use `waitFor()` with specific condition   |
| Flaky tests            | Add `test.step()`, use proper waits, disable animations       |
| Strict mode violation  | Use `.first()`, `.filter()`, or more specific locator         |
| CI fails, local passes | Check `baseURL`, timeouts, `webServer` config                 |
| API mock not working   | Use `**/api/...` glob, verify with `page.on('request')`       |

## CLI Quick Reference

| Command                           | Description                   |
|-----------------------------------|-------------------------------|
| `npx playwright test`             | Run all tests headless        |
| `npx playwright test --ui`        | Open interactive UI mode      |
| `npx playwright test --debug`     | Run with Playwright Inspector |
| `npx playwright test -g "login"`  | Run tests matching pattern    |
| `npx playwright show-report`      | Open HTML report              |
| `npx playwright codegen`          | Generate tests by recording   |
| `PWDEBUG=1 npx playwright test`   | Debug with Inspector          |

---

## Verification Checklist

- [ ] Test files follow `*.spec.ts` naming convention
- [ ] Page Objects injected via fixtures (no `new PageObject()` in specs)
- [ ] Locators use `getByRole()`, `getByTestId()`, or `getByText()` — no raw CSS
- [ ] No `page.waitForTimeout()` or `sleep()` calls
- [ ] Each test sets up and tears down its own state — no shared mutable state
- [ ] Error/empty/loading states tested alongside happy paths
- [ ] All tests pass (`npx playwright test` exits 0)
- [ ] No skipped tests (`test.skip` / `test.fixme`) in main suite

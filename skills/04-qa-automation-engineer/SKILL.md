---
name: qa-automation-engineer
description: Senior QA automation engineer role — architect scalable, maintainable test suites integrated with CI/CD. Covers test pyramid design, framework selection, test data management, parallel execution, flaky test governance, and quality reporting dashboards.
source: https://github.com/rohitg00/awesome-claude-code-toolkit
tier: best-in-class
---

# QA Automation Engineer

Acts as a senior QA automation engineer who architects scalable, maintainable test suites that prevent regressions and integrate seamlessly with CI/CD pipelines.

## When to Use

- Designing or auditing a test automation strategy from scratch
- Setting up CI/CD test pipelines (GitHub Actions, GitLab CI, Jenkins)
- Choosing test frameworks and configuration for a new project
- Implementing test data management strategies
- Creating quality dashboards and reporting for teams
- Establishing go/no-go quality gates before releases
- Governing flaky tests across a large test suite

---

## Test Pyramid Architecture

Structure every test suite using the pyramid model. Deviation from this is a smell:

```
         /\
        /E2E\         10% — slow, expensive, high confidence
       /------\
      /  Integ  \     20% — service/API/DB boundaries
     /------------\
    /  Unit Tests  \   70% — fast, isolated, numerous
   /-----------------\
```

**Organization principle**: group by **feature**, not by test type:
```
src/
  features/
    checkout/
      checkout.unit.test.ts        # unit tests
      checkout.integration.test.ts # integration tests
      checkout.e2e.spec.ts         # e2e tests
    auth/
      auth.unit.test.ts
      auth.integration.test.ts
```

---

## Framework Configuration

### Browser Testing (Playwright)
```typescript
// playwright.config.ts
export default defineConfig({
  retries: process.env.CI ? 2 : 0,        // max 2 retries in CI
  workers: process.env.CI ? 2 : undefined, // constrain parallelism in CI
  reporter: [
    ['html', { outputFolder: 'playwright-report' }],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    ['github'],                             // GitHub Actions annotations
  ],
  use: {
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
});
```

### Unit/Integration Testing (Vitest/Jest)
```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    pool: 'threads',
    poolOptions: { threads: { maxThreads: 4 } },
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov', 'html'],
      thresholds: { lines: 80, functions: 80, branches: 75 },
    },
    reporters: ['verbose', 'junit'],
  },
});
```

### Performance Testing (k6)
```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 100 },  // ramp up
    { duration: '5m', target: 100 },  // sustained load
    { duration: '2m', target: 0 },    // ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95th percentile under 500ms
    http_req_failed: ['rate<0.01'],    // error rate under 1%
  },
};
```

---

## Test Data Management

| Strategy           | When to Use                              | Implementation                          |
|--------------------|------------------------------------------|-----------------------------------------|
| Factory pattern    | Flexible, varied test data               | `faker.js`, `fishery`, custom builders  |
| DB transactions    | Integration tests needing real DB        | Roll back after each test               |
| Seed once          | Reference/lookup data                    | `beforeAll` with idempotent seeds       |
| API seeding        | E2E tests needing app state              | Call setup APIs before test, cleanup after |
| No real PII        | All environments                         | Always use synthetic data               |

```typescript
// Factory pattern example
import { Factory } from 'fishery';
import { faker } from '@faker-js/faker';

const userFactory = Factory.define<User>(() => ({
  id: faker.string.uuid(),
  email: faker.internet.email(),
  name: faker.person.fullName(),
  role: 'user',
}));

// In tests
const adminUser = userFactory.build({ role: 'admin' });
const users = userFactory.buildList(5);
```

---

## CI/CD Integration

### Pipeline Structure

```yaml
# .github/workflows/quality.yml
name: Quality Gates

on: [push, pull_request]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: unit-coverage
          path: coverage/

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:integration

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npx playwright install --with-deps
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/

  nightly-regression:
    runs-on: ubuntu-latest
    if: github.event_name == 'schedule'
    steps:
      - run: npx playwright test --project=all
```

### Execution Schedule

| Trigger      | Tests to Run                            | Time Budget |
|--------------|-----------------------------------------|-------------|
| Every commit | Unit tests                              | 10 min      |
| Every PR     | Unit + Integration tests                | 30 min      |
| Nightly      | Full regression (E2E + performance)     | 2 hours     |
| Pre-release  | All tests including accessibility       | No limit    |

---

## Quality Gates

Define and enforce these before any deployment:

```typescript
// quality-gate.config.ts
export const qualityGate = {
  coverage: {
    lines: 80,
    functions: 80,
    branches: 75,
  },
  testResults: {
    maxFailures: 0,          // zero tolerance for failures
    maxSkipped: 5,           // alert if more than 5 tests skipped
    maxFlaky: 3,             // alert if more than 3 flaky in last 7 days
  },
  performance: {
    p95ResponseTime: 500,    // ms
    errorRate: 0.01,         // 1%
    maxMemoryLeak: 50,       // MB over 10 min soak
  },
};
```

---

## Flaky Test Governance

Track flaky tests as first-class defects:

1. **Detect**: configure retry counts (max 2 in CI); log every retry as a potential flake
2. **Track**: maintain a flaky test registry with first-seen date, frequency, owner
3. **SLA**: flaky tests in critical paths must be fixed within 1 sprint
4. **Quarantine**: if not fixed in SLA, quarantine with a tracking issue (never silent skip)
5. **Trend**: review flaky test count weekly — rising count is a pipeline health metric

---

## Reporting Dashboard

Publish these metrics to your team weekly:

| Metric                    | Target           | Alert Threshold  |
|---------------------------|------------------|------------------|
| Test pass rate            | ≥ 98%            | < 95%            |
| Code coverage (lines)     | ≥ 80%            | < 75%            |
| E2E execution time        | ≤ 30 min         | > 45 min         |
| Flaky test count          | 0 (critical path)| > 3 in 7 days    |
| Mean time to detect (MTTD)| ≤ 1 build        | > 3 builds       |
| Defect escape rate        | ≤ 2%             | > 5%             |

---

## Verification Checklist

- [ ] Test pyramid ratios respected (70/20/10)
- [ ] Tests organized by feature, not by type
- [ ] CI runs unit tests per commit, integration per PR, E2E nightly
- [ ] No real customer/PII data in test environments
- [ ] Factory pattern used for test data (not hardcoded fixtures)
- [ ] Coverage thresholds enforced as CI quality gates
- [ ] Flaky test registry exists and is reviewed weekly
- [ ] HTML + JUnit reports generated and archived per CI run
- [ ] Performance baselines defined and monitored
- [ ] Build fails on test failures — no manual overrides without issue tracking

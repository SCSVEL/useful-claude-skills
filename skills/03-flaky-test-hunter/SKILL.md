---
name: flaky-test-hunter
description: Detect, diagnose, and eliminate intermittent (flaky) test failures through pattern recognition and root cause analysis. Use when tests pass locally but fail in CI, fail non-deterministically, or have been quarantined. Never treats symptoms — always investigates root cause first.
source: https://github.com/fugazi/test-automation-skills-agents
tier: best-in-class
---

# Flaky Test Hunter

A test reliability detective: identify, analyze, and eliminate intermittent failures. A flaky test is worse than no test — it erodes trust in the entire suite.

## When to Use

- Tests pass locally but fail intermittently in CI
- Tests fail at random without code changes
- Test suite has a growing list of quarantined/skipped tests
- Team is losing confidence in the test suite
- PR checks are unreliable (must re-run to get green)

## Core Rules (Non-Negotiable)

1. **Always investigate root cause before prescribing any fix** — never treat a symptom
2. Never use arbitrary delays (`sleep`, `waitForTimeout`) as a primary solution
3. Disable tests only with a tracking issue and documented reason
4. Verify every fix with 10+ consecutive CI runs before closure

---

## Root Cause Taxonomy

### 1. Race Conditions / Timing Issues

**Symptoms**: Test passes when run alone, fails when parallelized; failures correlate with CI load

**Diagnoses**:
```typescript
// ❌ Timing assumption — element might not exist yet
await page.click('#submit-btn');

// ✅ Wait for the condition you actually need
await page.waitForSelector('#submit-btn', { state: 'visible' });
await page.click('#submit-btn');

// ✅ Even better: web-first assertion first
await expect(page.getByRole('button', { name: 'Submit' })).toBeEnabled();
await page.getByRole('button', { name: 'Submit' }).click();
```

**Fix pattern**: Replace timing assumptions with explicit condition waits.

### 2. Shared State Pollution

**Symptoms**: Test passes in isolation, fails when run in suite; order-dependent failures

**Diagnoses**:
```typescript
// ❌ Shared mutable state between tests
let user: User;
beforeAll(async () => { user = await createUser(); });
afterAll(async () => { await deleteUser(user.id); });

// ✅ Isolated state per test
beforeEach(async () => { user = await createUser(); });
afterEach(async () => { await deleteUser(user.id); });
```

**Fix pattern**: Move setup/teardown from `beforeAll/afterAll` to `beforeEach/afterEach`. Use database transactions that roll back after each test.

### 3. External Dependency Flakiness

**Symptoms**: Failures correlate with network calls; error messages reference timeouts or 503s

**Diagnoses**:
```typescript
// ❌ Real external API in unit/integration test
const result = await fetch('https://api.stripe.com/v1/charges');

// ✅ Mock external dependencies at the boundary
await page.route('**/api.stripe.com/**', route =>
  route.fulfill({ status: 200, body: JSON.stringify(mockChargeResponse) })
);
```

**Fix pattern**: Mock external APIs using `page.route()`, `msw`, or a test double. Reserve real calls for contract tests only.

### 4. Async / Promise Handling

**Symptoms**: Uncaught promise rejections; tests complete before async operations finish

**Diagnoses**:
```typescript
// ❌ Missing await — test completes before assertion runs
test('saves user', () => {
  saveUser({ name: 'Alice' });
  expect(db.users).toHaveLength(1); // may be 0
});

// ✅ Properly awaited
test('saves user', async () => {
  await saveUser({ name: 'Alice' });
  expect(db.users).toHaveLength(1);
});
```

### 5. Date/Time Sensitivity

**Symptoms**: Tests fail near midnight, weekends, month/year boundaries

**Fix pattern**: Freeze time in tests using `jest.useFakeTimers()`, `vi.setSystemTime()`, or `sinon.useFakeTimers()`. Never test against `new Date()` directly.

```typescript
beforeEach(() => {
  vi.setSystemTime(new Date('2024-06-15T12:00:00Z'));
});
afterEach(() => {
  vi.useRealTimers();
});
```

### 6. Resource Cleanup Failures

**Symptoms**: "Port already in use", "database locked", file access errors on second run

**Fix pattern**: Ensure cleanup runs even when tests fail using `try/finally` or framework teardown hooks.

---

## Investigation Workflow

### Step 1: Reproduce the Flakiness

```bash
# Run the specific test 20 times to confirm and measure flake rate
for i in {1..20}; do npx playwright test tests/checkout.spec.ts --reporter=line; done

# Or use Playwright's built-in repeat
npx playwright test tests/checkout.spec.ts --repeat-each=20
```

### Step 2: Isolate the Cause

```bash
# Check if test is order-dependent
npx playwright test --shard=1/1 --repeat-each=10  # isolated
npx playwright test --repeat-each=10               # full suite

# Enable full tracing for failed runs
npx playwright test --trace=on

# View trace for last failure
npx playwright show-report
```

### Step 3: Classify the Root Cause

Use the taxonomy above. Check in this order:
1. Is there a `waitForTimeout` or `sleep`? → Race condition
2. Does it fail only in parallel? → Shared state
3. Does it involve external URLs or services? → External dependency
4. Does it involve dates/times? → Time sensitivity
5. Does it fail only after a previous test? → State pollution

### Step 4: Fix and Verify

Apply the appropriate fix pattern. Then verify:
```bash
# Must pass 10 consecutive runs in CI before closing
npx playwright test tests/flaky.spec.ts --repeat-each=10 --retries=0
```

---

## Quarantine Protocol

When a fix isn't immediately available, quarantine properly — never silently skip:

```typescript
test.skip('FLAKY-123: intermittent timeout on checkout button click — race condition under investigation', async ({ page }) => {
  // test body preserved for reference
});
```

Quarantine requirements:
- Tracking issue created with reproduction steps
- `skip` annotation includes the issue ID and one-line description
- Review scheduled within 1 sprint
- Remove from quarantine only after 10+ green CI runs

---

## Prevention Guidelines

Add to your project's CLAUDE.md or testing conventions:

```markdown
## Test Reliability Rules
- No `page.waitForTimeout()` or `sleep()` — use web-first assertions or `waitFor` conditions
- No shared mutable state in `beforeAll` — use `beforeEach` with cleanup
- Mock all external HTTP calls — use `page.route()` for Playwright, MSW for unit tests
- Freeze time whenever testing date/time logic
- All tests must pass 10× in isolation before merging
- Quarantined tests require a tracking issue — no silent skips
```

---

## Verification Checklist

- [ ] Root cause identified (not just symptom patched)
- [ ] No arbitrary delays introduced as the fix
- [ ] Test passes 10+ consecutive times in CI with `--retries=0`
- [ ] If quarantined: tracking issue created with reproduction steps
- [ ] Prevention pattern documented in project testing conventions
- [ ] Related tests checked for the same root cause pattern

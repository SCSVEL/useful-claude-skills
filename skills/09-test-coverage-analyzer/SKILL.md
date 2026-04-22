---
name: test-coverage-analyzer
description: Analyze test coverage gaps, prioritize by risk, and generate missing tests. Goes beyond line coverage percentages — maps untested code to business risk, identifies mutation-surviving test weaknesses, and produces an actionable coverage improvement plan.
source: https://github.com/qdhenry/Claude-Command-Suite + proffesor-for-testing/agentic-qe
tier: best-in-class
---

# Test Coverage Analyzer

Coverage percentages lie. 80% line coverage can still miss the 20% that contains all the bugs. This skill maps coverage gaps to business risk and builds an actionable improvement plan.

## When to Use

- Coverage is below threshold but you don't know where to start
- Coverage is "high" but bugs are still escaping to production
- Before a release: identify critical paths with zero test coverage
- After adding a large feature with few tests
- When mutation testing reveals tests that don't catch defects

---

## Step 1: Gather Coverage Data

```bash
# JavaScript/TypeScript (Vitest)
npx vitest run --coverage
# Outputs: coverage/lcov.info, coverage/index.html

# JavaScript/TypeScript (Jest)
npx jest --coverage --coverageReporters=lcov,text-summary

# Python (pytest)
pytest --cov=src --cov-report=html --cov-report=term-missing

# Java (Maven + JaCoCo)
mvn test jacoco:report
# Report: target/site/jacoco/index.html
```

---

## Step 2: Risk-Weighted Gap Analysis

Not all uncovered code is equal. Prioritize by:

```
Coverage Priority Score = Business Risk × Failure Impact × Defect Probability

Business Risk:
  5 = Payment, auth, data integrity, PII
  4 = Core user flows (checkout, signup, core CRUD)
  3 = Secondary features
  2 = Admin/internal tools
  1 = Cosmetic/logging

Failure Impact:
  5 = Data loss, security breach, revenue loss
  4 = Feature completely broken
  3 = Feature degraded
  2 = Minor inconvenience
  1 = No user impact

Defect Probability:
  5 = Complex logic, many branches, recent changes
  4 = Multiple integration points
  3 = Moderate complexity
  2 = Simple, stable code
  1 = Trivial getters/setters
```

**Coverage Gap Report template**:
```markdown
| Module                    | Line % | Branch % | Risk | Priority | Uncovered Critical Paths           |
|---------------------------|--------|----------|------|----------|------------------------------------|
| payment/processor.ts      | 45%    | 30%      | 5    | P0       | refund flow, failed auth, retry     |
| auth/jwt.ts               | 62%    | 45%      | 5    | P0       | token refresh, expiry edge cases   |
| cart/checkout.ts          | 78%    | 60%      | 4    | P1       | concurrent checkout, coupon stack  |
| notifications/email.ts    | 20%    | 10%      | 3    | P2       | template rendering, bounce handling|
| utils/format.ts           | 95%    | 90%      | 1    | P4       | minor edge cases                   |
```

---

## Step 3: Identify What Lines Mean

Line coverage tells you what ran. Branch coverage tells you what decisions were tested.

```typescript
// This function has 100% line coverage but 50% branch coverage
// if you only test the truthy path
function getDiscount(user: User): number {
  if (user.isPremium) {            // branch 1: true ✅  false ❌
    return 0.20;
  }
  return 0;
}

// What you need:
test('premium users get 20% discount', () => {
  expect(getDiscount({ isPremium: true })).toBe(0.20);
});
test('regular users get no discount', () => {
  expect(getDiscount({ isPremium: false })).toBe(0);
});
```

**Key insight**: Always target branch/condition coverage, not just line coverage.

---

## Step 4: Mutation Testing

Mutation testing proves tests actually *catch* bugs, not just *run* code.

```bash
# JavaScript/TypeScript (Stryker)
npm install --save-dev @stryker-mutator/core @stryker-mutator/vitest-runner

# stryker.config.mjs
export default {
  testRunner: 'vitest',
  mutate: ['src/**/*.ts', '!src/**/*.spec.ts'],
  thresholds: { high: 80, low: 60, break: 50 },
  reporters: ['html', 'clear-text'],
};

npx stryker run
```

**Interpreting mutation score**:
```
Killed mutants: your tests caught the bug — good
Survived mutants: your tests didn't catch the bug — gap to fix
No coverage: not even run — absolute gap
Timeout: possible infinite loop in source — investigate
```

**Common survived mutants and what they mean**:
```typescript
// Mutant: changed > to >= — your test didn't check the boundary
if (age > 18) { ... }  →  if (age >= 18) { ... }   // add test for age=18

// Mutant: removed condition — your test didn't check when this is false
if (user.isActive && user.hasPermission) { ... }    // test the && cases

// Mutant: changed + to - — your test input produces same result both ways
return total + tax;  →  return total - tax;         // use values where tax > 0
```

---

## Step 5: Generate Missing Tests

For each coverage gap, generate tests systematically:

### Template for uncovered function

```typescript
// Source: payment/processor.ts - refund() - 0% coverage
async function refund(orderId: string, amount: number): Promise<RefundResult> {
  if (amount <= 0) throw new Error('Refund amount must be positive');
  if (amount > order.total) throw new Error('Cannot refund more than order total');
  // ...payment gateway call...
}

// Generated tests:
describe('refund', () => {
  it('successfully refunds valid amount', async () => {
    const order = await createOrder({ total: 100 });
    const result = await refund(order.id, 50);
    expect(result.status).toBe('success');
    expect(result.refundedAmount).toBe(50);
  });

  it('throws for zero amount', async () => {
    const order = await createOrder({ total: 100 });
    await expect(refund(order.id, 0)).rejects.toThrow('must be positive');
  });

  it('throws when refund exceeds order total', async () => {
    const order = await createOrder({ total: 100 });
    await expect(refund(order.id, 150)).rejects.toThrow('Cannot refund more');
  });

  it('handles payment gateway failure', async () => {
    vi.spyOn(gateway, 'refund').mockRejectedValue(new Error('Gateway timeout'));
    await expect(refund(orderId, 50)).rejects.toThrow('Gateway timeout');
  });
});
```

---

## Step 6: Coverage Improvement Roadmap

Structure improvement as a sprint-ready backlog:

```markdown
## Coverage Improvement Plan

### Sprint 1 (P0 — Critical Path Coverage)
- [ ] payment/processor.ts: add refund flow tests (est. 4h, +15% branch)
- [ ] auth/jwt.ts: add token refresh + expiry tests (est. 2h, +20% branch)

### Sprint 2 (P1 — Core Flows)
- [ ] cart/checkout.ts: concurrent checkout race condition tests (est. 3h)
- [ ] cart/checkout.ts: coupon stacking edge cases (est. 2h)

### Sprint 3 (P2 — Secondary Features)
- [ ] notifications/email.ts: template rendering + bounce handling (est. 3h)

### Continuous
- [ ] Enforce coverage thresholds in CI (fail if drops below 80% lines, 70% branches)
- [ ] Run Stryker mutation analysis weekly, target mutation score ≥ 75%
- [ ] Add coverage badge to README
```

---

## CI Enforcement

```yaml
# .github/workflows/coverage.yml
- name: Check coverage thresholds
  run: |
    npx vitest run --coverage \
      --coverage.thresholds.lines=80 \
      --coverage.thresholds.branches=70 \
      --coverage.thresholds.functions=80
```

```typescript
// vitest.config.ts — fail CI if thresholds not met
coverage: {
  thresholds: {
    lines: 80,
    branches: 70,
    functions: 80,
    statements: 80,
    // Per-file thresholds for critical modules
    'src/payment/**': { lines: 90, branches: 85 },
    'src/auth/**': { lines: 90, branches: 85 },
  },
},
```

---

## Verification Checklist

- [ ] Coverage report generated (HTML + LCOV)
- [ ] Gaps mapped to business risk (not just alphabetical by file)
- [ ] Critical paths (payment, auth, core flows) at ≥ 90% branch coverage
- [ ] Mutation score ≥ 75% for critical modules
- [ ] All P0 coverage gaps have test tickets created
- [ ] CI fails when coverage drops below thresholds
- [ ] Coverage trends tracked week-over-week (not just a snapshot)
- [ ] Tests generated for coverage gaps actually assert on outcomes

---
name: ui-comprehensive-tester
description: Comprehensive UI testing across web, mobile, and desktop applications. Intelligently selects testing tools based on platform (Playwright for web, Mobile MCP for iOS/Android). Tests functional behavior, usability, accessibility, responsive design, error states, and edge cases.
source: https://github.com/darcyegb/ClaudeCodeAgents
tier: best-in-class
---

# UI Comprehensive Tester

Performs thorough UI testing across web, mobile, and desktop applications. Selects the optimal testing approach for the platform and produces detailed, actionable reports.

## When to Use

- Testing a feature's full UI behavior (happy paths + edge cases)
- Cross-browser and cross-device compatibility verification
- Accessibility audits (WCAG 2.1 AA)
- Responsive design validation across breakpoints
- Usability issue identification
- Regression testing after UI changes

---

## Tool Selection by Platform

| Platform               | Tool                | Use Case                                          |
|------------------------|---------------------|---------------------------------------------------|
| Web (complex)          | Playwright          | Cross-browser, network interception, traces       |
| Web (simple/readonly)  | Puppeteer           | Lightweight automation, screenshots               |
| iOS / Android          | Mobile MCP          | Native gestures, device-specific behaviors        |
| Desktop (Electron)     | Playwright          | Electron apps via CDP                             |

---

## Web UI Testing

### Comprehensive Coverage Areas

For every feature, test across these dimensions:

**1. Functional Coverage**
- [ ] Happy path (valid inputs, expected flow)
- [ ] Form validation (required fields, format errors, length limits)
- [ ] Navigation (links, back/forward, deep links)
- [ ] Interactive elements (buttons, dropdowns, modals, tabs)
- [ ] Data display (correct values, formatting, empty states)
- [ ] CRUD operations (create, read, update, delete flows)

**2. Error States**
- [ ] Network failure (offline mode, 500 errors)
- [ ] Unauthorized/forbidden access
- [ ] Not found (404 pages, deleted resources)
- [ ] Validation errors (inline messages, field highlighting)
- [ ] Timeout handling (slow responses, loading spinners)

**3. Responsive Design**
```typescript
const breakpoints = {
  mobile:  { width: 375,  height: 812  },  // iPhone SE
  tablet:  { width: 768,  height: 1024 },  // iPad
  laptop:  { width: 1280, height: 720  },  // Standard laptop
  desktop: { width: 1920, height: 1080 },  // Full HD
};

for (const [name, size] of Object.entries(breakpoints)) {
  test(`${featureName} renders correctly on ${name}`, async ({ page }) => {
    await page.setViewportSize(size);
    await page.goto('/feature');

    // Take full-page screenshot for visual review
    await page.screenshot({ fullPage: true, path: `screenshots/${name}.png` });

    // Verify no horizontal overflow
    const hasOverflow = await page.evaluate(() =>
      document.documentElement.scrollWidth > document.documentElement.clientWidth
    );
    expect(hasOverflow).toBe(false);
  });
}
```

**4. Accessibility (WCAG 2.1 AA)**
```typescript
import { checkA11y } from 'axe-playwright';

test('page has no accessibility violations', async ({ page }) => {
  await page.goto('/feature');
  await checkA11y(page, undefined, {
    detailedReport: true,
    detailedReportOptions: { html: true },
  });
});

// Manual checks beyond axe:
// - Tab order is logical and visible
// - All images have meaningful alt text
// - Color contrast ratio ≥ 4.5:1 for body text
// - Form fields have associated labels
// - Error messages are announced to screen readers
// - Modal dialogs trap focus correctly
// - Skip navigation link present
```

**5. Performance (User-Perceived)**
```typescript
test('page loads within performance budget', async ({ page }) => {
  await page.goto('/feature');

  const metrics = await page.evaluate(() => ({
    fcp: performance.getEntriesByName('first-contentful-paint')[0]?.startTime,
    domInteractive: performance.timing.domInteractive - performance.timing.navigationStart,
    loadComplete: performance.timing.loadEventEnd - performance.timing.navigationStart,
  }));

  expect(metrics.fcp).toBeLessThan(1800);        // FCP < 1.8s
  expect(metrics.domInteractive).toBeLessThan(3000); // TTI < 3s
});
```

---

## Mobile Testing Patterns

```typescript
// iOS-specific touch interactions
test('swipe to delete item', async ({ page }) => {
  await page.setViewportSize({ width: 390, height: 844 }); // iPhone 14

  // Touch/swipe simulation
  const item = page.locator('[data-testid="list-item"]').first();
  const box = await item.boundingBox();

  await page.touchscreen.tap(box.x + box.width / 2, box.y + box.height / 2);

  // Simulate swipe left
  await page.touchscreen.tap(box.x + box.width - 10, box.y + box.height / 2);
  // Use page.mouse for swipe gestures in Playwright
  await page.mouse.move(box.x + box.width - 10, box.y + box.height / 2);
  await page.mouse.down();
  await page.mouse.move(box.x + 10, box.y + box.height / 2, { steps: 10 });
  await page.mouse.up();

  await expect(page.getByRole('button', { name: 'Delete' })).toBeVisible();
});

// Test critical mobile-specific behaviors
const mobileChecks = [
  'Tap targets are at least 44×44 CSS pixels',
  'No horizontal scroll at 375px viewport',
  'Inputs trigger appropriate mobile keyboard (email, tel, numeric)',
  'Pinch-to-zoom not disabled (no user-scalable=no)',
  'Fixed headers do not obscure content when scrolling',
];
```

---

## Bug Severity Classification

Use this consistent taxonomy in reports:

| Severity | Definition                                              | Example                              |
|----------|---------------------------------------------------------|--------------------------------------|
| Critical | Blocks core user flow; data loss or security issue      | Login broken; payment loop           |
| High     | Major feature broken; significant UX degradation        | Form submit fails; wrong data shown  |
| Medium   | Feature works but with noticeable issues                | Misaligned layout; slow load (>3s)   |
| Low      | Cosmetic or minor inconvenience                         | Typo; slight padding off             |

---

## Test Execution Report Template

```markdown
# UI Test Report: [Feature Name]
**Date**: [YYYY-MM-DD] | **Tester**: [Name/Agent] | **Build**: [version]

## Summary
| Category        | Passed | Failed | Blocked |
|----------------|--------|--------|---------|
| Functional      | 12     | 2      | 0       |
| Responsive      | 8      | 1      | 0       |
| Accessibility   | 15     | 3      | 0       |
| Error States    | 6      | 0      | 0       |
| **Total**       | **41** | **6**  | **0**   |

## Critical Issues
1. **[CRITICAL]** Submit button unresponsive on iOS Safari 17
   - Steps: Navigate to /checkout → Fill form → Tap Submit
   - Expected: Form submits, redirect to confirmation
   - Actual: Nothing happens, no error shown
   - Screenshot: [attached]

## Recommendations
- Fix critical iOS issue before release
- Medium issues can ship with tracking tickets
```

---

## Verification Checklist

- [ ] Happy path tested end-to-end on all target browsers
- [ ] All error states have visible, accessible error messages
- [ ] Tested at mobile (375px), tablet (768px), and desktop (1280px) breakpoints
- [ ] No horizontal overflow at any breakpoint
- [ ] Accessibility audit passes (axe-core zero violations, WCAG AA)
- [ ] Tab order is logical; keyboard-only navigation works
- [ ] Form inputs have associated labels
- [ ] Interactive elements meet 44×44px touch target minimum
- [ ] Performance budget met (FCP < 1.8s, TTI < 3s)
- [ ] Report produced with severity-classified issues

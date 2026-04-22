---
name: qa-manual-istqb
description: Complete ISTQB Foundation Level aligned workflow for QA test engineers covering test planning, test design techniques, bug reporting, regression suites, traceability, and metrics. Use when creating test plans, generating test cases, writing bug reports, or conducting exploratory testing.
source: https://github.com/fugazi/test-automation-skills-agents
tier: best-in-class
---

# ISTQB Manual & Automation QA Toolkit

Full ISTQB CTFL-aligned workflow: **Test Planning → Test Analysis → Test Design → Test Implementation → Test Execution → Test Completion**

## When to Use

- Creating or reviewing **test plans** and **test strategies**
- Generating **test conditions** and **test cases** from requirements
- Applying **test design techniques** (EP, BVA, decision tables, state transitions)
- Writing **bug reports** and managing **defect lifecycle**
- Building **regression suites** with risk-based selection
- Creating **traceability matrices** (requirements ↔ tests ↔ defects)
- Conducting **exploratory testing** sessions with charters
- Estimating test effort using ISTQB techniques
- Reviewing testware through **static testing** practices

---

## Inputs to Collect First

Always gather these before starting:

- **Test basis**: requirements, user stories, acceptance criteria, designs, risk register, defect history
- **Scope**: in-scope/out-of-scope features, target platforms/browsers/devices, locales, integrations
- **Quality risks**: what can fail, impact, likelihood, regulatory/compliance requirements
- **Constraints**: deadlines, environments, data availability, tooling, access/roles, CI/CD expectations
- **Definitions**: severity vs priority scale, test levels and types, entry/exit criteria

---

## Workflows

### 1. Create a Test Plan

1. Identify test objectives, scope, assumptions, and constraints from the test basis
2. Define test levels and types (functional + change-related + key non-functional)
3. Choose test design techniques per area (see Test Design Techniques section)
4. Specify environments, test data, tooling, and configuration management needs
5. Define entry/exit criteria, deliverables, and reporting cadence/metrics
6. Add a risk matrix and mitigation actions; prioritize testing by risk

**Test Plan Structure:**
```markdown
# Test Plan: [Project Name] [Release]

## 1. Introduction
- Objectives, scope, out-of-scope

## 2. Test Items
- Features to be tested, versions

## 3. Risk & Mitigation
- Quality risks (impact × likelihood), mitigations

## 4. Test Approach
- Levels: unit / integration / system / acceptance
- Types: functional / performance / security / accessibility

## 5. Entry & Exit Criteria
- Entry: build deployed, smoke passed, data seeded
- Exit: 95% cases executed, 0 critical open, coverage ≥ threshold

## 6. Environments & Data
- Environments, credentials, test data strategy

## 7. Schedule & Effort Estimate
- Milestones, resource allocation

## 8. Deliverables
- Test cases, bug reports, summary report, traceability matrix
```

### 2. Generate Test Conditions and Test Cases

1. Convert the test basis into **test conditions** (WHAT to test) before writing step-by-step cases
2. For each condition, pick a technique:

| Technique              | Use When                                             |
|------------------------|------------------------------------------------------|
| Equivalence Partitioning | Input validation, categories of valid/invalid data |
| Boundary Value Analysis  | Numeric ranges, min/max thresholds                 |
| Decision Tables          | Business rules with multiple conditions            |
| State Transition         | Lifecycle flows (order states, auth flows)         |
| Use Case / Scenario      | End-to-end user journeys                           |
| Exploratory Charter      | Learning, discovering unknown unknowns             |

3. Write test cases: atomic, unambiguous, traceable to requirement/user story IDs
4. Add expected results: observable and measurable (define the test oracle)
5. Tag priority and risk to support risk-based regression selection

**Test Case Template:**
```csv
ID, Title, Preconditions, Steps, Expected Result, Priority, Tags, Requirement ID
TC-001, "Login with valid credentials", "User registered", "1. Go to /login\n2. Enter valid email\n3. Enter valid password\n4. Click Sign In", "Redirected to /dashboard, user name shown", High, smoke regression, US-042
```

### 3. Build and Maintain Regression Suites

1. Define suite tiers:
   - **Smoke**: critical paths, < 10 min, run on every deploy
   - **Sanity**: build verification, < 30 min, run on every PR
   - **Regression**: broad coverage, < 2 hours, run nightly
   - **Full**: complete coverage, run before release

2. Select tests using: risk × frequency × criticality × defect history
3. Tag tests consistently (`@smoke`, `@regression`, `@critical`)
4. Review the suite regularly: remove obsolete coverage, add tests for escaped defects

### 4. Write Effective Bug Reports

1. Reproduce reliably; reduce to minimal steps
2. Capture environment details (build, OS, browser, account role, data conditions)
3. Describe expected vs actual behavior with impact statement
4. Attach evidence (screenshots, console logs, network traces)
5. Set severity and priority consistently using defined scale

**Bug Report Template:**
```markdown
# Bug Report: [Short Title]

**ID:** BUG-XXX  
**Severity:** Critical / High / Medium / Low  
**Priority:** P1 / P2 / P3  
**Environment:** [OS, Browser, App version, Test data]  
**Reporter:** [Name] | **Date:** [YYYY-MM-DD]

## Summary
[One sentence description of the defect]

## Steps to Reproduce
1. Navigate to [URL]
2. [Action]
3. [Action]

## Expected Result
[What should happen]

## Actual Result
[What actually happens]

## Impact
[Who is affected and how severely]

## Evidence
[Screenshots, logs, trace files]
```

### 5. Conduct Static Testing (Reviews)

1. Schedule reviews early (shift-left): requirements, designs, test plans, test cases
2. Use checklists for consistency
3. Document findings with severity and actionability
4. Track defects found in static testing separately (prevention vs detection)

### 6. Estimate Test Effort

Apply one or more techniques:
- **Expert judgment / historical data**: benchmark against similar past features
- **Test point analysis**: count test conditions × complexity × automation factor
- **Work breakdown structure**: decompose into tasks, estimate each, sum + contingency

Always add 15-25% contingency for risks and unknowns.

### 7. Monitor Test Progress and Metrics

Track these metrics throughout execution:

| Metric                   | Formula / Definition                              |
|--------------------------|---------------------------------------------------|
| Test Execution Rate      | Cases executed / Total cases × 100%              |
| Pass Rate                | Cases passed / Cases executed × 100%             |
| Defect Detection Rate    | Defects found / Total defects × 100%             |
| Defect Removal Efficiency| Defects found in testing / All defects × 100%   |
| Requirements Coverage    | Requirements with ≥1 test / Total requirements   |
| Defect Age               | Days from open to close (avg by severity)        |

---

## Test Design Techniques Reference

### Equivalence Partitioning
Divide inputs into partitions where all values behave the same way. Test one value per partition.
- Valid partition: age 18-65 → test 35
- Invalid partitions: age <18, >65 → test 10, test 70

### Boundary Value Analysis
Test the edges of each partition (min, min+1, max-1, max).
- Age range 18-65 → test 17, 18, 19, 64, 65, 66

### Decision Tables
For features driven by combinations of conditions:
```
Condition 1 | Y | Y | N | N
Condition 2 | Y | N | Y | N
Action A    | X |   | X |  
Action B    |   | X |   | X
```

### State Transition
Map states → events → transitions → actions. Test all valid transitions and key invalid ones.

---

## Quality Gates (Self-Check)

- [ ] **Test plan** includes scope, approach, risks, environments, entry/exit criteria, deliverables, metrics
- [ ] **Test cases** are traceable, atomic, deterministic, with clear oracles and data
- [ ] **Automation** is maintainable: stable locators, minimal flake, independent tests, clear assertions
- [ ] **Regression** is risk-based, tagged, and curated with clear add/remove rules
- [ ] **Bug reports** are reproducible, actionable, contain evidence + environment + impact
- [ ] **Static testing** reviews are documented with findings tracked to resolution
- [ ] **Estimates** include contingency and are refined as scope clarifies
- [ ] **Coverage matrix** maintained — no uncovered requirements
- [ ] **Both positive and negative scenarios** are tested
- [ ] **Boundary values, empty inputs, and extreme values** are covered

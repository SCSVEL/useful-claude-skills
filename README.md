# my-claude-skills

A curated collection of best-in-class Claude Code skills for QA, testing, and quality engineering — sourced from the top open-source repositories and adapted for production use.

---

## Top 10 QA Skills

| # | Skill | What It Does | Best For |
|---|-------|--------------|----------|
| 1 | [Playwright E2E Testing](skills/01-playwright-e2e-testing/SKILL.md) | TypeScript E2E tests: POM, locators, API mocking, responsive, CI config | Writing/debugging browser automation tests |
| 2 | [ISTQB Manual QA](skills/02-qa-manual-istqb/SKILL.md) | Full ISTQB test planning, design techniques (EP/BVA/decision tables), bug reports, metrics | Structured manual QA, test plans, exploratory testing |
| 3 | [Flaky Test Hunter](skills/03-flaky-test-hunter/SKILL.md) | Root cause analysis and fixes for non-deterministic tests | Stabilizing unreliable CI pipelines |
| 4 | [QA Automation Engineer](skills/04-qa-automation-engineer/SKILL.md) | Test pyramid architecture, CI/CD integration, framework config, quality gates, dashboards | Setting up or auditing a full automation strategy |
| 5 | [API Testing](skills/05-api-testing/SKILL.md) | REST/GraphQL testing, schema validation, auth flows, contract testing | API test coverage from scratch or audit |
| 6 | [Task Completion Validator](skills/06-task-completion-validator/SKILL.md) | Verify implementations actually work end-to-end, not stubbed/hardcoded | Pre-merge QA gate on any feature |
| 7 | [UI Comprehensive Tester](skills/07-ui-comprehensive-tester/SKILL.md) | Cross-platform UI testing (web/mobile), accessibility WCAG 2.1 AA, responsive, performance | Release readiness testing |
| 8 | [Adversarial Audit](skills/08-adversarial-audit/SKILL.md) | Attack surface mapping, abuse cases, auth bypass, injection, business logic exploits | Pre-release security testing |
| 9 | [Test Coverage Analyzer](skills/09-test-coverage-analyzer/SKILL.md) | Risk-weighted gap analysis, mutation testing, coverage improvement roadmap | Improving test effectiveness, not just percentages |
| 10 | [Code Review QA](skills/10-code-review-qa/SKILL.md) | Multi-lens review: correctness, security, testability, maintainability, performance | PR reviews, audit of legacy code |

---

## How to Use These Skills

### Option A — Load directly as context

Paste the contents of a SKILL.md into your Claude conversation before starting a task. The skill primes Claude with the right mental model, patterns, and verification checklists.

### Option B — Claude Code slash commands

Copy a SKILL.md into your project's `.claude/commands/` directory:

```
your-project/
└── .claude/
    └── commands/
        ├── playwright-test.md      ← contents of skill 01
        ├── api-test.md             ← contents of skill 05
        └── code-review.md         ← contents of skill 10
```

Then invoke with `/playwright-test`, `/api-test`, etc. in Claude Code.

### Option C — CLAUDE.md reference

Add a line to your project's `CLAUDE.md` pointing to the relevant skills:

```markdown
## Testing
For E2E tests, follow: [Playwright E2E Testing skill](../my-claude-skills/skills/01-playwright-e2e-testing/SKILL.md)
For code review: [Code Review QA skill](../my-claude-skills/skills/10-code-review-qa/SKILL.md)
```

---

## Skill Selection Guide

| Situation | Recommended Skills |
|-----------|-------------------|
| New project, setting up testing | 4 → 1 → 5 → 2 |
| Tests keep failing in CI randomly | 3 (Flaky Test Hunter) |
| About to ship a feature | 6 → 7 → 8 |
| Coverage is low, don't know where to start | 9 |
| PR review before merge | 10 → 6 |
| Writing API tests | 5 |
| Accessibility audit needed | 7 |
| Security review | 8 |
| Building test strategy from scratch | 4 → 2 |

---

## Sources

These skills are curated and synthesized from:

- [fugazi/test-automation-skills-agents](https://github.com/fugazi/test-automation-skills-agents) — Production-grade Playwright, API, ISTQB, and flaky test skills
- [neonwatty/qa-skills](https://github.com/neonwatty/qa-skills) — QA pipeline skills for Claude Code with adversarial and resilience audits
- [darcyegb/ClaudeCodeAgents](https://github.com/darcyegb/ClaudeCodeAgents) — QA agents: task validator, UI tester, compliance checker
- [rohitg00/awesome-claude-code-toolkit](https://github.com/rohitg00/awesome-claude-code-toolkit) — QA automation engineer agent with pyramid + CI/CD patterns
- [qdhenry/Claude-Command-Suite](https://github.com/qdhenry/Claude-Command-Suite) — Professional slash commands for testing, coverage, mutation testing
- [proffesor-for-testing/agentic-qe](https://github.com/proffesor-for-testing/agentic-qe) — Agentic QE Fleet: 60 agents, 75 skills across all QE domains
- [anthropics/skills](https://github.com/anthropics/skills) — Official Anthropic webapp-testing skill (Playwright Python)
- [travisvn/awesome-claude-skills](https://github.com/travisvn/awesome-claude-skills) — Curated community skills directory

---

Owner: [SCSVEL](https://github.com/SCSVEL) | Project context: [MyHomeDoctor](https://github.com/SCSVEL/MyHomeDoctor)

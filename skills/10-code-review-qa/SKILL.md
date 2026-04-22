---
name: code-review-qa
description: Structured QA-focused code review that catches over-engineering, logic errors, security gaps, missing tests, and naming issues before merge. Performs a multi-lens review: correctness, security, testability, maintainability, and performance.
source: https://github.com/darcyegb/ClaudeCodeAgents + https://github.com/qdhenry/Claude-Command-Suite
tier: best-in-class
---

# Code Review QA

A senior-engineer quality lens applied to code before merge: correctness, security, testability, maintainability, and performance — in that order.

## When to Use

- Before merging any non-trivial PR
- After implementing a bug fix (verify fix doesn't introduce regressions)
- When reviewing externally contributed code
- As a self-review step before requesting human review
- When auditing legacy code for quality issues

---

## Review Lens 1: Correctness

The most critical lens. Does the code actually do what it claims?

**Questions to answer**:
- Does every code path produce a correct result?
- Are all edge cases handled (null, empty, zero, max values)?
- Are concurrent operations safe?
- Is the logic equivalent to the specification/ticket?

**Common correctness bugs**:
```typescript
// ❌ Off-by-one: should be <=, not <
for (let i = 0; i < items.length - 1; i++) {
  process(items[i]); // misses last item
}

// ❌ Mutation of input parameter (unexpected side effect)
function sortUsers(users: User[]): User[] {
  return users.sort((a, b) => a.name.localeCompare(b.name)); // mutates original array!
}
// ✅ Non-mutating
function sortUsers(users: User[]): User[] {
  return [...users].sort((a, b) => a.name.localeCompare(b.name));
}

// ❌ Floating point comparison
if (price === 0.1 + 0.2) { ... } // 0.1 + 0.2 = 0.30000000000000004
// ✅ Use a tolerance or library
if (Math.abs(price - 0.30) < Number.EPSILON) { ... }

// ❌ Race condition: read-modify-write without atomicity
async function incrementCounter(id: string) {
  const current = await db.counters.get(id);       // read
  await db.counters.update(id, current.value + 1); // write — concurrent calls will conflict
}
// ✅ Atomic operation
await db.counters.increment(id, 1);
```

---

## Review Lens 2: Security

**Input validation**:
```typescript
// ❌ Trusting user input directly
app.get('/user/:id', async (req, res) => {
  const user = await db.query(`SELECT * FROM users WHERE id = ${req.params.id}`); // SQL injection
});

// ✅ Parameterized query
app.get('/user/:id', async (req, res) => {
  const user = await db.query('SELECT * FROM users WHERE id = $1', [req.params.id]);
});

// ❌ Reflected XSS
res.send(`<h1>Hello ${req.query.name}</h1>`); // XSS if name = <script>...

// ❌ Insecure direct object reference
app.get('/document/:id', async (req, res) => {
  const doc = await db.documents.get(req.params.id); // no ownership check!
  res.json(doc);
});
// ✅ Check ownership
app.get('/document/:id', async (req, res) => {
  const doc = await db.documents.getForUser(req.params.id, req.user.id);
  if (!doc) return res.status(404).json({ error: 'Not found' });
  res.json(doc);
});
```

**Security checklist for every PR**:
- [ ] No SQL/NoSQL injection via string concatenation
- [ ] No hardcoded secrets, API keys, or passwords
- [ ] User input sanitized/validated before use
- [ ] Authorization checks on every resource access
- [ ] No sensitive data logged or included in error responses
- [ ] Dependencies: no newly introduced known-vulnerable packages

---

## Review Lens 3: Testability

**Questions**:
- Can this code be unit tested without spinning up databases or external services?
- Are dependencies injectable (not created inside functions)?
- Are side effects isolated from business logic?

```typescript
// ❌ Untestable — hard dependency on database and external service
async function registerUser(email: string, password: string) {
  const hashedPw = await bcrypt.hash(password, 10);
  await new Database().users.create({ email, password: hashedPw });
  await new EmailService().sendWelcome(email); // side effect mixed with logic
  return true;
}

// ✅ Testable — dependencies injected, pure transformation separated
async function registerUser(
  email: string,
  password: string,
  deps: { db: UserRepository; email: EmailService; hash: HashFn }
) {
  const hashedPw = await deps.hash(password);
  const user = await deps.db.create({ email, password: hashedPw });
  await deps.email.sendWelcome(email);
  return user;
}
```

**Testability checklist**:
- [ ] Dependencies injected or mockable at seams
- [ ] Pure functions separated from I/O operations
- [ ] No direct `new Service()` calls inside functions being tested
- [ ] Side effects (email, DB writes) happen in predictable, isolated places

---

## Review Lens 4: Maintainability

**Over-engineering detection**:
```typescript
// ❌ Premature abstraction for 2 use cases
class AbstractBaseEntityFactoryStrategyProvider<T extends BaseEntity> {
  // 50 lines for something that could be 5 lines
}

// ❌ Unnecessary indirection
const getUserById = (id: string) => userService.findById(id);
// ↑ Just use userService.findById(id) directly

// ❌ Magic numbers
if (retryCount > 3) { ... }    // What's special about 3?
// ✅ Named constant
const MAX_RETRIES = 3;
if (retryCount > MAX_RETRIES) { ... }
```

**Naming checklist**:
- [ ] Functions named for what they DO (verb: `getUser`, `validateEmail`, `processOrder`)
- [ ] Booleans named as predicates (`isActive`, `hasPermission`, `canDelete`)
- [ ] No single-letter variables outside tight loops
- [ ] No misleading names (`temp`, `data`, `info`, `stuff`, `obj`)
- [ ] No abbreviations that require domain knowledge to decode

**Complexity checks**:
- Cyclomatic complexity > 10 per function → likely needs decomposition
- Functions > 50 lines → likely does more than one thing
- Files > 300 lines → likely needs splitting
- Nesting depth > 3 → extract nested logic to named functions

---

## Review Lens 5: Performance

```typescript
// ❌ N+1 query pattern
const orders = await db.orders.findAll();
for (const order of orders) {
  order.user = await db.users.findById(order.userId); // N queries!
}
// ✅ Batch fetch
const orders = await db.orders.findAllWithUsers(); // JOIN or batch

// ❌ Unnecessary re-computation in loop
const users = await db.users.findAll();
for (const user of users) {
  const config = JSON.parse(fs.readFileSync('config.json')); // reads file every iteration!
  process(user, config);
}
// ✅ Hoist constant outside loop
const config = JSON.parse(fs.readFileSync('config.json'));
for (const user of users) {
  process(user, config);
}

// ❌ Missing index for frequently queried column
// In migration: just created users.email column with no index
// SELECT * FROM users WHERE email = ? — full table scan on every login!
// ✅ Add index: CREATE INDEX users_email_idx ON users(email);
```

---

## Review Output Format

```markdown
## Code Review: [PR Title / Feature Name]

### Overall: APPROVED / CHANGES REQUESTED / BLOCKED

---

### Must Fix (blocks merge)
- **[SECURITY]** `auth/login.ts:42` — Password compared with `==` not constant-time comparison → timing attack possible
- **[CORRECTNESS]** `cart/discount.ts:87` — Floating point comparison `=== 0.1 + 0.2` will always be false

### Should Fix (this PR)
- **[TESTABILITY]** `payment/processor.ts` — `new StripeClient()` created inside function, not injectable
- **[MAINTAINABILITY]** `utils/helpers.ts:120` — Function is 90 lines doing 4 distinct things; split into named functions

### Suggestions (future/optional)
- **[PERFORMANCE]** `users/list.ts:35` — N+1 pattern if user list grows; consider eager loading
- **[NAMING]** `data` parameter in `processData()` (line 58) — name the actual type

### Positive Notes
- Clean separation of validation from business logic in `order.service.ts`
- Good use of transaction wrapping in `order.repository.ts`
```

---

## Verification Checklist

- [ ] All code paths traced for correctness (including null/empty/concurrent paths)
- [ ] No SQL injection, XSS, or IDOR vulnerabilities
- [ ] No hardcoded secrets or credentials
- [ ] Authorization checks present on all resource operations
- [ ] Dependencies are injectable (testable without real services)
- [ ] No magic numbers; constants named
- [ ] Functions single-purpose and < 50 lines
- [ ] No N+1 query patterns
- [ ] Review output includes severity classification and file:line references
- [ ] Positive aspects noted (not only defects)

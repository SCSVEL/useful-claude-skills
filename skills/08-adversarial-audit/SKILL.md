---
name: adversarial-audit
description: Map economic/security attack surface, generate abuse cases, and stress-test assumptions from an adversarial perspective. Use before release to identify how real-world bad actors could exploit the system — covers auth bypass, rate limit evasion, data exfiltration, business logic abuse, and injection vectors.
source: https://github.com/neonwatty/qa-skills
tier: best-in-class
---

# Adversarial Audit

Maps attack surface and generates abuse cases from an adversarial perspective. Goes beyond functional testing — asks "how would a determined attacker misuse this?"

## When to Use

- Pre-release security review of new features
- After adding authentication, payment, or permissions logic
- When a feature handles user-supplied input
- When a feature has monetary or data value to attackers
- To generate negative test cases that functional QA misses

## Scope Clarification First

Before starting, answer:
1. What is the economic/data value this feature controls?
2. Who are the likely threat actors? (script kiddies, competitors, insiders, bots)
3. What's the blast radius if this is exploited?
4. What existing mitigations are in place?

---

## Attack Surface Mapping

### 1. Authentication & Session Attacks

**Test cases to generate**:
```markdown
- [ ] Access protected endpoint with no token → expect 401
- [ ] Access with malformed JWT (tampered signature) → expect 401
- [ ] Access with expired JWT → expect 401 with "expired" message
- [ ] Access with JWT signed by different secret → expect 401
- [ ] Replay a revoked token (after logout) → expect 401
- [ ] JWT algorithm confusion: change alg to "none" → expect rejection
- [ ] Brute force login: 100 attempts in 60s → expect lockout or rate limit
- [ ] Login with SQL in email field: `' OR '1'='1` → no bypass, no 500
- [ ] Session fixation: reuse pre-auth session after login → new token issued
```

### 2. Authorization & Privilege Escalation

**Test cases to generate**:
```markdown
- [ ] User A accesses User B's resource by changing ID in URL → expect 403/404
- [ ] Regular user calls admin endpoint → expect 403
- [ ] User modifies their own role via profile update API → expect 403/400
- [ ] User accesses resource of deleted/suspended account → expect 403
- [ ] IDOR: iterate sequential IDs to access other users' data
      GET /api/orders/1, /api/orders/2... → all should be 403 for non-owner
- [ ] Mass assignment: POST /api/users with {"role": "admin"} in body → role unchanged
- [ ] Horizontal escalation across tenants (if multi-tenant)
```

### 3. Input Injection Vectors

**Test cases for every user-input field**:
```markdown
SQL Injection:
- [ ] `'; DROP TABLE users; --`
- [ ] `' UNION SELECT password FROM users--`
- [ ] `1 OR 1=1`

XSS:
- [ ] `<script>alert('xss')</script>`
- [ ] `<img src=x onerror=alert(1)>`
- [ ] `javascript:alert(1)` in URL fields

Path Traversal:
- [ ] `../../../etc/passwd` in file upload names
- [ ] `..%2F..%2Fetc%2Fpasswd` (URL-encoded)

Command Injection (if server executes commands):
- [ ] `; cat /etc/passwd`
- [ ] `$(whoami)`
- [ ] `| ls -la`

SSRF (if server makes HTTP requests with user input):
- [ ] `http://localhost/admin`
- [ ] `http://169.254.169.254/latest/meta-data/` (AWS metadata)
- [ ] `file:///etc/passwd`
```

### 4. Rate Limiting & Denial of Service

**Test cases**:
```markdown
- [ ] Rapid fire API calls: 1000 requests/minute → expect 429 with Retry-After
- [ ] Large payload: upload 100MB file → expect 413 before processing
- [ ] Deeply nested JSON: `{"a":{"b":{"c":...}}}` (1000 levels) → no stack overflow
- [ ] Very long strings: 10MB string in text field → handled gracefully
- [ ] Concurrent duplicate requests: 50 simultaneous POST /api/orders → no duplicate orders
- [ ] Regex DoS (ReDoS): supply pathological input to regex-validated fields
```

### 5. Business Logic Abuse

**Test cases (context-specific)**:
```markdown
E-commerce:
- [ ] Apply coupon code multiple times on same order
- [ ] Negative quantity in cart: quantity=-1 → negative total?
- [ ] Race condition: checkout while item goes out of stock simultaneously
- [ ] Price manipulation: modify price in frontend before submit

Account/Credits:
- [ ] Transfer funds to self → balance unchanged?
- [ ] Transfer 0 or negative amount
- [ ] Race condition: simultaneous withdrawals exceeding balance

File Upload:
- [ ] Upload .php/.exe/.svg with malicious content
- [ ] Upload file claiming to be image but containing script
- [ ] Zip bomb: upload 1KB zip that extracts to 1GB
- [ ] SVG with embedded JavaScript
```

### 6. Data Exfiltration & Information Leakage

**Test cases**:
```markdown
- [ ] Error messages expose stack traces, DB schema, or internal paths → should not
- [ ] API response includes fields not needed by the client (over-fetching)
- [ ] Timing attacks: does login response time differ for valid vs invalid email?
- [ ] Cache headers on sensitive responses (set Cache-Control: no-store)
- [ ] Internal IDs (database PKs) exposed in URLs → consider UUIDs or slugs
- [ ] API returns deleted records (soft delete not filtered)
- [ ] Verbose 404: does "user not found" vs "wrong password" enumerate users?
```

---

## Abuse Case Template

For each finding, document:

```markdown
## Abuse Case: [Name]

**Category**: Auth / AuthZ / Injection / Rate Limit / Business Logic / Data Leak
**Severity**: Critical / High / Medium / Low
**Likelihood**: High / Medium / Low (considering threat actor sophistication)

### Attack Scenario
[Describe how a real attacker would exploit this, step by step]

### Test Case
1. [Step 1]
2. [Step 2]
3. Observe: [what actually happens]

### Expected (Secure) Behavior
[What should happen]

### Actual Behavior
[What currently happens — if vulnerability confirmed]

### Recommended Mitigation
[Specific fix, e.g., "Add rate limiting middleware at the /api/login route"]

### Risk if Unmitigated
[Business/data impact: "Attacker can enumerate all user emails via timing attack"]
```

---

## Prioritization Matrix

After generating abuse cases, prioritize by:

```
Impact × Likelihood = Risk Score

Critical × High     = P0 — fix before deploy
Critical × Medium   = P1 — fix this sprint
High × High         = P1 — fix this sprint
High × Medium       = P2 — fix next sprint
Medium × Any        = P2/P3 — schedule
Low × Any           = P4 — track in backlog
```

---

## Verification Checklist

- [ ] Authentication: token validation, expiry, revocation all tested
- [ ] Authorization: horizontal and vertical privilege escalation tested
- [ ] All user-input fields tested with SQL injection, XSS, path traversal
- [ ] Rate limiting tested and confirmed enforced
- [ ] Business logic edge cases and race conditions documented
- [ ] Error messages do not leak internal details
- [ ] All P0/P1 findings have assigned owners and fix ETAs
- [ ] At least one re-test performed after mitigations are applied

---
name: api-testing
description: REST and GraphQL API testing using Playwright request fixture (TypeScript) or REST Assured (Java). Covers schema validation, authentication flows, error handling, pagination, idempotency, contract testing, and rate limiting. Use when writing, auditing, or debugging API tests.
source: https://github.com/fugazi/test-automation-skills-agents
tier: best-in-class
---

# API Testing (REST & GraphQL)

Comprehensive API test coverage for REST and GraphQL endpoints with schema validation, auth testing, contract enforcement, and edge case coverage.

## When to Use

- Writing API tests for new endpoints
- Auditing existing API test coverage
- Testing authentication and authorization flows
- Implementing contract tests between services
- Validating error handling and edge cases
- Setting up API test infrastructure for a project

## Four Foundational Principles

1. **Schema validate every response** — structure and types, not just status codes
2. **Test all HTTP status codes** — success AND all error states
3. **Always test authentication** — protected endpoints require auth tests
4. **Data-driven testing** — valid, invalid, boundary, and empty values for every input

---

## TypeScript (Playwright request fixture)

### Setup

```typescript
// api.setup.ts
import { test as base, expect } from '@playwright/test';

type APIFixtures = {
  apiContext: APIRequestContext;
  authToken: string;
};

export const test = base.extend<APIFixtures>({
  authToken: async ({ request }, use) => {
    const response = await request.post('/api/auth/login', {
      data: { email: process.env.TEST_EMAIL, password: process.env.TEST_PASSWORD },
    });
    const { token } = await response.json();
    await use(token);
  },
  apiContext: async ({ playwright, authToken }, use) => {
    const ctx = await playwright.request.newContext({
      baseURL: process.env.API_BASE_URL || 'http://localhost:3000',
      extraHTTPHeaders: { Authorization: `Bearer ${authToken}` },
    });
    await use(ctx);
    await ctx.dispose();
  },
});
```

### CRUD Endpoint Tests

```typescript
import { test, expect } from './api.setup';
import { z } from 'zod';

// Schema definition — validate structure, not just status
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1),
  role: z.enum(['user', 'admin']),
  createdAt: z.string().datetime(),
});

test.describe('Users API', () => {
  test('GET /users returns paginated list', async ({ apiContext }) => {
    const response = await apiContext.get('/api/users?page=1&limit=10');
    expect(response.status()).toBe(200);

    const body = await response.json();
    expect(body).toMatchObject({
      data: expect.any(Array),
      pagination: {
        page: 1,
        limit: 10,
        total: expect.any(Number),
      },
    });
    // Validate each item against schema
    body.data.forEach((user: unknown) => UserSchema.parse(user));
  });

  test('POST /users creates user with valid data', async ({ apiContext }) => {
    const payload = { email: 'new@test.com', name: 'New User', role: 'user' };
    const response = await apiContext.post('/api/users', { data: payload });
    expect(response.status()).toBe(201);

    const user = await response.json();
    UserSchema.parse(user); // schema validation
    expect(user.email).toBe(payload.email);
    expect(response.headers()['location']).toContain(`/api/users/${user.id}`);
  });

  test('POST /users returns 400 for invalid email', async ({ apiContext }) => {
    const response = await apiContext.post('/api/users', {
      data: { email: 'not-an-email', name: 'Test' },
    });
    expect(response.status()).toBe(400);

    const error = await response.json();
    expect(error).toMatchObject({
      error: expect.any(String),
      field: 'email',
    });
  });

  test('DELETE /users/:id is idempotent', async ({ apiContext }) => {
    const createRes = await apiContext.post('/api/users', {
      data: { email: 'delete@test.com', name: 'To Delete' },
    });
    const { id } = await createRes.json();

    const firstDelete = await apiContext.delete(`/api/users/${id}`);
    expect(firstDelete.status()).toBe(204);

    const secondDelete = await apiContext.delete(`/api/users/${id}`);
    expect(secondDelete.status()).toBe(404); // or 204 — document your contract
  });
});
```

### Authentication Tests

```typescript
test.describe('Auth', () => {
  test('returns 401 without token', async ({ request }) => {
    const response = await request.get('/api/users');
    expect(response.status()).toBe(401);
  });

  test('returns 403 for insufficient role', async ({ request }) => {
    // Use a regular user token
    const response = await request.get('/api/admin/settings', {
      headers: { Authorization: `Bearer ${userToken}` },
    });
    expect(response.status()).toBe(403);
  });

  test('returns 401 for expired token', async ({ request }) => {
    const response = await request.get('/api/users', {
      headers: { Authorization: 'Bearer expired.jwt.token' },
    });
    expect(response.status()).toBe(401);
    const body = await response.json();
    expect(body.error).toMatch(/expired/i);
  });
});
```

---

## Java (REST Assured)

### Setup

```java
// ApiTestBase.java
@BeforeAll
static void setup() {
    RestAssured.baseURI = System.getenv().getOrDefault("API_BASE_URL", "http://localhost:3000");
    RestAssured.requestSpecification = new RequestSpecBuilder()
        .addHeader("Authorization", "Bearer " + getAuthToken())
        .addHeader("Content-Type", "application/json")
        .build();
}
```

### CRUD Tests with AssertJ

```java
@Test
void createUser_validPayload_returns201() {
    String payload = """
        {"email": "test@example.com", "name": "Test User", "role": "user"}
        """;

    given()
        .body(payload)
    .when()
        .post("/api/users")
    .then()
        .statusCode(201)
        .body("id", matchesPattern("[0-9a-f-]{36}"))
        .body("email", equalTo("test@example.com"))
        .body("createdAt", notNullValue())
        .header("Location", containsString("/api/users/"));
}

@Test
void getUser_invalidId_returns404() {
    given()
    .when()
        .get("/api/users/00000000-0000-0000-0000-000000000000")
    .then()
        .statusCode(404)
        .body("error", not(emptyString()));
}
```

---

## GraphQL Testing

```typescript
test.describe('GraphQL API', () => {
  const GQL_URL = '/graphql';

  test('query users with pagination', async ({ apiContext }) => {
    const response = await apiContext.post(GQL_URL, {
      data: {
        query: `
          query GetUsers($limit: Int, $offset: Int) {
            users(limit: $limit, offset: $offset) {
              nodes { id email name }
              totalCount
              pageInfo { hasNextPage endCursor }
            }
          }
        `,
        variables: { limit: 10, offset: 0 },
      },
    });

    expect(response.status()).toBe(200);
    const { data, errors } = await response.json();
    expect(errors).toBeUndefined(); // GraphQL always returns 200, check errors field
    expect(data.users.nodes).toHaveLength(10);
  });

  test('returns validation error for invalid input', async ({ apiContext }) => {
    const response = await apiContext.post(GQL_URL, {
      data: {
        query: `mutation { createUser(email: "bad") { id } }`,
      },
    });

    const { errors } = await response.json();
    expect(errors).toBeDefined();
    expect(errors[0].extensions.code).toBe('BAD_USER_INPUT');
  });
});
```

---

## Contract Testing (Consumer-Driven)

```typescript
// pact.spec.ts — consumer side
import { PactV3, MatchersV3 } from '@pact-foundation/pact';

const provider = new PactV3({
  consumer: 'frontend',
  provider: 'users-service',
});

test('users service contract', async () => {
  await provider
    .given('users exist')
    .uponReceiving('a request for user list')
    .withRequest({ method: 'GET', path: '/api/users' })
    .willRespondWith({
      status: 200,
      body: {
        data: MatchersV3.eachLike({
          id: MatchersV3.uuid(),
          email: MatchersV3.email(),
          name: MatchersV3.string(),
        }),
      },
    })
    .executeTest(async (mockServer) => {
      const response = await fetch(`${mockServer.url}/api/users`);
      expect(response.ok).toBe(true);
    });
});
```

---

## Edge Cases to Always Test

| Scenario                        | What to Assert                              |
|---------------------------------|---------------------------------------------|
| Empty collection                | 200 with `data: []`, not 404               |
| Single item                     | Correct structure, pagination `total: 1`   |
| Max page size exceeded          | 400 with clear error                        |
| Concurrent creates (same data)  | Idempotency or conflict handling (409)     |
| Very long strings               | 400 or truncated — document the behavior   |
| Special characters in IDs       | Proper URL encoding, no 500s               |
| Rate limiting                   | 429 with `Retry-After` header              |
| Payload too large               | 413 with clear message                     |

---

## Verification Checklist

- [ ] All CRUD operations tested (GET, POST, PUT/PATCH, DELETE)
- [ ] Schema validation on every response (not just status code)
- [ ] 401 tested for all protected endpoints without token
- [ ] 403 tested for role-based access control
- [ ] 400 tested for all invalid input combinations
- [ ] Idempotency verified for DELETE and PUT
- [ ] Edge cases covered (empty, boundary, special chars)
- [ ] GraphQL error field checked (not just HTTP status)
- [ ] Contract tests published to Pact Broker (if used)
- [ ] All tests pass without real external service calls in CI

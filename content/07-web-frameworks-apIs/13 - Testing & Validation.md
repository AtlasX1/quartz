# Testing & Validation of APIs: Ensuring Correctness and Reliability

## Introduction & Purpose

Testing APIs is fundamentally about answering: **"Does this API behave correctly under various conditions?"** But "correct" means different things at different levels.

Unit tests verify individual functions work. Integration tests verify components work together. E2E tests verify the entire system works. Contract tests verify services can communicate. Each level catches different categories of bugs.

This document explores the testing pyramid for APIs: unit, integration, E2E, and contract testing—and when each matters.

## Core Concepts & Internal Architecture

### The Testing Pyramid

```
              E2E Tests
           (few, slow)
          /           \
       Integration
       (moderate)
      /           \
  Unit Tests
  (many, fast)
```

**Unit tests (base):** Fast, isolated, many
**Integration tests (middle):** Moderate speed, test components together
**E2E tests (top):** Slow, test entire system

**Ideal distribution:**
```
Unit tests: 70%
Integration tests: 20%
E2E tests: 10%
```

**Reasoning:** Unit tests catch most bugs quickly; integration tests catch interaction bugs; E2E tests catch system-level bugs.

### Unit Testing APIs

**Unit tests** test functions in isolation:

```typescript
// userService.test.ts
import { UserService } from './userService'
import { Database } from './database'

// Mock the database
jest.mock('./database')

describe('UserService', () => {
    let service: UserService
    let mockDb: jest.Mocked<Database>

    beforeEach(() => {
        mockDb = Database as jest.Mocked<typeof Database>
        service = new UserService(mockDb)
    })

    describe('getUser', () => {
        it('returns user when found', async () => {
            // Arrange
            mockDb.findById.mockResolvedValue({ id: 1, name: 'Alice' })

            // Act
            const user = await service.getUser(1)

            // Assert
            expect(user).toEqual({ id: 1, name: 'Alice' })
            expect(mockDb.findById).toHaveBeenCalledWith(1)
        })

        it('throws error when user not found', async () => {
            // Arrange
            mockDb.findById.mockResolvedValue(null)

            // Act & Assert
            await expect(service.getUser(1)).rejects.toThrow('User not found')
        })
    })
})
```

**Architectural principles:**

1. **Isolation:** Mock external dependencies (database, APIs)
2. **Assertions:** Verify both return value and side effects
3. **Clarity:** Use Arrange-Act-Assert pattern

### Integration Testing

**Integration tests** verify multiple components work together:

```typescript
// userController.integration.test.ts
import { createTestApp } from './test-setup'
import { Database } from './database'
import { request } from 'supertest'

describe('UserController Integration', () => {
    let app
    let db: Database

    beforeAll(async () => {
        db = new Database()
        await db.connect()
        app = createTestApp(db)
    })

    afterAll(async () => {
        await db.disconnect()
    })

    afterEach(async () => {
        await db.clear()  // Clean up test data
    })

    describe('POST /users', () => {
        it('creates user and persists to database', async () => {
            // Act
            const response = await request(app)
                .post('/users')
                .send({ name: 'Alice', email: 'alice@example.com' })

            // Assert: HTTP response
            expect(response.status).toBe(201)
            expect(response.body).toHaveProperty('id')
            expect(response.body.name).toBe('Alice')

            // Assert: Data persisted
            const user = await db.users.findById(response.body.id)
            expect(user).toBeDefined()
            expect(user.email).toBe('alice@example.com')
        })

        it('validates email format', async () => {
            // Act
            const response = await request(app)
                .post('/users')
                .send({ name: 'Bob', email: 'invalid-email' })

            // Assert
            expect(response.status).toBe(400)
            expect(response.body.errors).toContainEqual(
                expect.objectContaining({ field: 'email' })
            )
        })
    })
})
```

**Architectural principles:**

1. **Real database:** Use test database, not mocks
2. **Clean state:** Start each test fresh
3. **Full stack:** Test from HTTP request to database

### End-to-End (E2E) Testing

**E2E tests** verify entire system workflows:

```typescript
// user.e2e.test.ts
import { test } from '@playwright/test'

test.describe('User workflow', () => {
    test('user can sign up and view profile', async ({ page }) => {
        // Navigate to signup
        await page.goto('https://app.example.com/signup')

        // Fill form
        await page.fill('input[name="name"]', 'Charlie')
        await page.fill('input[name="email"]', 'charlie@example.com')
        await page.fill('input[name="password"]', 'SecurePassword123')
        
        // Submit
        await page.click('button[type="submit"]')

        // Wait for navigation to dashboard
        await page.waitForURL('https://app.example.com/dashboard')

        // Verify profile shows correct name
        const userName = await page.textContent('[data-testid="user-name"]')
        expect(userName).toBe('Charlie')
    })
})
```

**Characteristics:**
- Tests real browsers
- Tests full user workflows
- Slow (10-30 seconds per test)
- Catch issues unit/integration tests miss

### Schema and Contract Validation

**Schema validation** tests verify API contracts:

```typescript
// API schema validation
import Ajv from 'ajv'

describe('User API Schema', () => {
    const ajv = new Ajv()
    const userSchema = {
        type: 'object',
        properties: {
            id: { type: 'number' },
            name: { type: 'string' },
            email: { type: 'string', format: 'email' }
        },
        required: ['id', 'name', 'email']
    }

    const validate = ajv.compile(userSchema)

    it('response matches schema', async () => {
        // Make request
        const response = await request(app)
            .get('/users/1')

        // Validate response body matches schema
        const valid = validate(response.body)
        
        if (!valid) {
            console.error('Validation errors:', validate.errors)
        }
        expect(valid).toBe(true)
    })
})
```

### Contract Testing (Pact)

**Contract testing** verifies service boundaries work correctly:

```typescript
// Consumer (Client) test
import { Pact } from '@pact-foundation/pact'

describe('API Client', () => {
    const provider = new Pact({ consumer: 'WebApp', provider: 'UserAPI' })

    it('can fetch user', async () => {
        // Define expected interaction
        await provider.addInteraction({
            state: 'user with id 1 exists',
            uponReceiving: 'a request for user 1',
            withRequest: {
                method: 'GET',
                path: '/users/1'
            },
            willRespondWith: {
                status: 200,
                body: Matchers.eachLike({
                    id: 1,
                    name: 'Alice'
                })
            }
        })

        // Test client code
        const client = new ApiClient(provider.mockService.baseUrl)
        const user = await client.getUser(1)

        expect(user.name).toBe('Alice')
    })

    afterAll(() => provider.finalize())
})
```

**Provider verifies contract:**
```typescript
describe('User API Provider', () => {
    // Verify server can satisfy the contract
    it('satisfies contract with WebApp', () => {
        return verifyProvider({
            providerBaseUrl: 'http://localhost:3000',
            pactUrls: ['./pacts/WebApp-UserAPI.json']
        })
    })
})
```

**Architectural benefit:** Prevents "client expects X, server provides Y" mismatches without requiring full integration test setup.

### Performance and Load Testing

**Performance tests** verify API meets performance requirements:

```typescript
import { check } from 'k6'
import http from 'k6/http'

export let options = {
    thresholds: {
        http_req_duration: ['p(95)<500'],  // 95% under 500ms
        http_req_failed: ['<1%']  // <1% failure rate
    }
}

export default function() {
    const response = http.get('http://localhost:3000/users')
    
    check(response, {
        'status is 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
        'has user data': (r) => {
            try {
                const body = JSON.parse(r.body)
                return Array.isArray(body.data)
            } catch {
                return false
            }
        }
    })
}
```

**Runs scenarios:**
```
Ramp up: 0 → 100 users over 1 minute
Sustain: 100 users for 5 minutes
Ramp down: 100 → 0 users over 1 minute

Reports:
- Response time percentiles (p50, p95, p99)
- Error rate
- Throughput (requests/second)
```

## Common Problems & Failure Scenarios

### Problem 1: Test Flakiness

**Scenario:**

```
Test passes: 90% of the time
Test fails: 10% of the time (randomly)

Cause: Race condition, timing issue, or external dependency variability
```

**Common causes:**
- Timing assumptions (expecting API to respond in 10ms, sometimes takes 50ms)
- Shared test state (tests interfere with each other)
- External dependencies (network calls to real services)

**Solutions:**
1. Use database transactions that rollback
2. Avoid hard-coded timeouts; use explicit waits
3. Mock external services
4. Run tests in isolation

### Problem 2: Test Data Management

**Scenario:**

```
Test 1 creates user "Alice"
Test 2 expects only existing users
    ↓
Test 1 ran, "Alice" exists
    ↓
Test 2 fails
```

**Root cause:** Shared test database; tests have side effects on each other.

**Solution: Clean slate**

```typescript
beforeEach(async () => {
    // Clear all data before each test
    await database.truncate()
})

afterEach(async () => {
    // Or: Wrap in transaction and rollback
    await database.rollbackTransaction()
})
```

### Problem 3: Incomplete Test Coverage

**Scenario:**

```
Code coverage: 90% (appears well-tested)
But: Only happy path tested
Error handling untested
Edge cases untested
    ↓
Bugs in error handling / edge cases
```

**Root cause:** Metrics (coverage %) don't guarantee quality.

**Solution:** Test behaviors, not just coverage

```typescript
describe('UserService', () => {
    describe('getUser', () => {
        // Happy path
        it('returns user when found', ...)
        
        // Error handling
        it('throws error when user not found', ...)
        
        // Edge cases
        it('handles null id', ...)
        it('handles negative id', ...)
        it('handles very large id', ...)
        
        // Authorization
        it('rejects unauthorized requests', ...)
    })
})
```

### Problem 4: Over-Testing Implementation Details

**Scenario:**

```typescript
// BAD: Tests implementation detail
it('calls database.query exactly once', () => {
    expect(mockDb.query).toHaveBeenCalledTimes(1)
})

// If implementation changes (optimization batches queries)
// Test fails even though behavior unchanged
```

**Root cause:** Testing "how" instead of "what".

**Better:** Test behavior

```typescript
// GOOD: Tests behavior
it('returns user with correct data', async () => {
    const user = await service.getUser(1)
    expect(user.name).toBe('Alice')
})

// If implementation changes but returns same data
// Test still passes
```

## Design Decisions & Trade-offs

### Trade-off 1: Unit Tests + Mocks vs. Integration Tests

**Unit tests (mocked):**
- Pros: Fast, isolate bugs to specific unit
- Cons: Mocks diverge from reality; miss interaction bugs

**Integration tests:**
- Pros: Test real components together
- Cons: Slower, harder to debug

**Best practice:** Use both. Mocked unit tests for quick feedback; integration tests to verify mocks are realistic.

### Trade-off 2: E2E Tests vs. Integration Tests

**E2E (through UI):**
- Pros: Tests real user workflows
- Cons: Very slow, fragile, hard to debug

**Integration (API level):**
- Pros: Faster, more stable
- Cons: Doesn't test UI/browser issues

**Best practice:** Rely on integration tests; use E2E for critical workflows only.

## When to Test

### What to Test

- ✅ Business logic (core functionality)
- ✅ Error handling (validation, edge cases)
- ✅ Authorization (who can do what)
- ✅ Data persistence (writes to database work)
- ⚠️ UI interactions (only critical paths, via E2E)
- ❌ Framework internals (framework already tested)
- ❌ Third-party library behavior (library already tested)

### Test Distribution

```
API / Service:
  Unit: 70%
  Integration: 25%
  E2E: 5%

UI / Frontend:
  Unit: 40%
  Integration: 40%
  E2E: 20%

Critical system:
  Unit: 60%
  Integration: 30%
  E2E: 10%
```

---

## Conclusion

**Testing is about confidence.** Good tests catch bugs early, enable refactoring, and document expected behavior.

Key principles:

1. **Test pyramid:** Many fast unit tests; fewer slow integration/E2E tests
2. **Isolation:** Mock external dependencies
3. **Clarity:** Use descriptive test names; Arrange-Act-Assert pattern
4. **Coverage:** Not just code coverage; behavior coverage
5. **Performance:** Tests should be fast (milliseconds for units, seconds for integration)
6. **Contracts:** Verify service boundaries work correctly

The goal is not 100% coverage. The goal is catching bugs before production.

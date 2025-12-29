# Testing & Debugging: Quality Assurance and Production Troubleshooting

## Conceptual Overview

Testing and debugging are distinct disciplines with different purposes. Testing validates correctness during development; debugging identifies root causes of failures in production or during development. Node.js provides sophisticated tools for both: Jest/Mocha for unit tests, Supertest for HTTP API testing, Node Inspector for debugging, and profiling tools for performance analysis. Understanding when to apply each tool and how to structure tests for maintainability is essential for building reliable systems.

The challenge isn't writing tests—it's writing tests that provide value (catching regressions, documenting behavior) without becoming maintenance burdens. Similarly, debugging requires understanding the execution model, memory model, and profiling techniques to find subtle issues in concurrent systems.

---

## Internal Mechanics: Test Isolation and Mocking

### Test Frameworks: Jest vs. Mocha vs. Vitest

**Jest**:
- Built-in test runner, assertion library, mocking, coverage
- Automatic test discovery and parallelization
- Snapshot testing for UI and serialized output
- Strengths: Complete out-of-box; excellent for large projects
- Weaknesses: Slower startup (Node.js module loading); configuration complexity

**Mocha**:
- Minimal test framework; requires external libraries for assertions (chai) and mocking (sinon)
- Modular; compose testing stack
- Strengths: Lightweight; flexible; fast startup
- Weaknesses: More configuration; fragmented ecosystem

**Vitest**:
- Modern test framework; built on Vite
- Speed (uses native ES modules)
- Jest-compatible API
- Strengths: Fast; modern tooling; TypeScript first-class
- Weaknesses: Younger; smaller ecosystem

### Test Structure: Arrange-Act-Assert

```javascript
describe('UserService', () => {
  // Arrange: Set up test data and dependencies
  let userService;
  let mockDatabase;
  
  beforeEach(() => {
    mockDatabase = { getUser: jest.fn() };
    userService = new UserService(mockDatabase);
  });
  
  it('should return user when found', async () => {
    // Arrange
    const userId = 1;
    const expectedUser = { id: 1, name: 'Alice' };
    mockDatabase.getUser.mockResolvedValue(expectedUser);
    
    // Act
    const user = await userService.getUser(userId);
    
    // Assert
    expect(user).toEqual(expectedUser);
    expect(mockDatabase.getUser).toHaveBeenCalledWith(userId);
  });
});
```

### Mocking and Stubbing

**Mocking**: Replacing a dependency with a mock that records calls and returns predetermined values.

```javascript
const mockFetch = jest.fn()
  .mockResolvedValue({ json: async () => ({ id: 1 }) });
```

**Stubbing**: Replacing implementation without recording calls (simpler than mocking).

**Spying**: Wrapping real implementation to observe calls without changing behavior.

```javascript
const spy = jest.spyOn(database, 'query');
// spy records calls but delegates to original database.query
```

---

## Testing Layers

### Unit Tests: Fast, Isolated

Test a single unit (function, class, module) in isolation. Dependencies are mocked.

**Characteristics**:
- Run in milliseconds
- No I/O (files, network, database)
- 70-80% of test suite

**Example**:
```javascript
test('calculateDiscount applies percentage correctly', () => {
  const discount = calculateDiscount(100, 20);  // 20% off
  expect(discount).toBe(20);
});
```

**Challenge**: Overuse of mocks can lead to tests that pass but code fails in integration (mock doesn't accurately represent reality).

### Integration Tests: Medium Speed, Realistic

Test multiple components working together. Use real dependencies (test database, mock HTTP servers).

**Characteristics**:
- Run in seconds
- Use test database (SQLite, in-memory)
- Mock external services (HTTP APIs)
- 15-25% of test suite

**Example**:
```javascript
test('POST /users creates user in database', async () => {
  const response = await request(app)
    .post('/users')
    .send({ email: 'test@example.com' });
  
  expect(response.status).toBe(201);
  
  const user = await db.getUser('test@example.com');
  expect(user).toBeDefined();
});
```

### End-to-End (E2E) Tests: Slow, Real System

Test the entire application flow, often through the UI. No mocking.

**Characteristics**:
- Run in minutes
- Use real infrastructure (database, external services)
- Flaky; environment-dependent
- 5-15% of test suite

**Example** (Playwright):
```javascript
test('user can create account and login', async ({ page }) => {
  await page.goto('http://localhost:3000/signup');
  await page.fill('[name="email"]', 'test@example.com');
  await page.click('[type="submit"]');
  
  expect(page.url()).toContain('/dashboard');
});
```

---

## Debugging: Finding Production Issues

### Debug Protocol: Node Inspector

Node.js can run with `--inspect` flag, enabling remote debugging:

```bash
node --inspect app.js
```

Connect from Chrome DevTools (chrome://inspect), Firefox DevTools, or VS Code.

**Features**:
- Breakpoints with conditional logic
- Step through execution (step-in, step-over, step-out)
- Watch expressions and REPL in DevTools console
- Call stack and scope inspection

**Real scenario**: Production crash; attach debugger to staging replica, reproduce locally.

### Profiling: Identifying Performance Bottlenecks

**CPU Profiling**: Identify hot functions consuming CPU time.

```bash
node --prof app.js  # Generates v8.log
node --prof-process v8.log > profile.txt  # Converts to readable format
```

**Flame graphs**: Visualize CPU time by call stack.

```bash
# Use clinic.js for automatic flame graph generation
npx clinic.js doctor -- node app.js
npx clinic.js doctor  # Generates HTML visualization
```

**Memory Profiling**: Find memory leaks and allocation hotspots.

```javascript
const inspector = require('inspector');
const fs = require('fs');

const session = new inspector.Session();
session.connect();

session.post('HeapProfiler.startSampling', {}, () => {
  setTimeout(() => {
    session.post('HeapProfiler.stopSampling', {}, (err, { profile }) => {
      fs.writeFileSync('profile.json', JSON.stringify(profile));
      session.disconnect();
    });
  }, 5000);
});
```

---

## Problems & Challenges

### 1. Brittle Tests Due to Tight Coupling

**The problem**: Tests depend on implementation details rather than behavior.

```javascript
// Brittle: Tests internal state
test('user is added to list', () => {
  userService.addUser({ name: 'Alice' });
  expect(userService.users.length).toBe(1);  // Tests implementation
});

// Better: Tests behavior
test('user can be retrieved after adding', async () => {
  await userService.addUser({ name: 'Alice' });
  const user = await userService.getUser('Alice');
  expect(user.name).toBe('Alice');  // Tests behavior
});
```

### 2. Flaky Tests: Non-Deterministic Failures

**The problem**: Timing-dependent tests fail intermittently.

```javascript
// Flaky: Race condition
test('data loads', async () => {
  fetchData();  // No await
  setTimeout(() => {
    expect(dataLoaded).toBe(true);  // Sometimes data isn't loaded in time
  }, 100);  // Arbitrary delay
});

// Better: Wait for actual condition
test('data loads', async () => {
  const promise = fetchData();
  await expect(promise).resolves.toBeDefined();
});
```

### 3. Test Infrastructure Overhead

**The problem**: Setting up test database, mocking external services, and tear-down takes more time than the test itself.

### 4. Memory Leaks in Long-Running Tests

**The problem**: Test suite accumulates memory across tests; later tests fail with OOM.

### 5. Debugging Production Crashes Without Reproduction

**The problem**: Error occurs in production but cannot be reproduced locally (race condition, specific environment, specific data).

### 6. Performance Regressions: Slow Code Not Caught by Tests

**The problem**: Code changes that degrade performance pass all tests because tests don't measure performance.

---

## Solutions & Architectural Approaches

### 1. Use Factory Functions for Test Data

Avoid duplicating test data setup:

```javascript
function createTestUser(overrides = {}) {
  return {
    id: 1,
    email: 'test@example.com',
    name: 'Test User',
    ...overrides
  };
}

test('user email is required', async () => {
  const user = createTestUser({ email: null });
  expect(() => validateUser(user)).toThrow();
});
```

### 2. Use Test Containers for Database Tests

Run real database in Docker for integration tests:

```javascript
describe('UserRepository with PostgreSQL', () => {
  let container;
  let db;
  
  beforeAll(async () => {
    container = await GenericContainer.from('postgres:13')
      .withExposedPorts(5432)
      .withEnvironment({ POSTGRES_PASSWORD: 'password' })
      .start();
    
    db = new Database({ host: container.getHost(), port: container.getMappedPort(5432) });
  });
  
  afterAll(async () => {
    await container.stop();
  });
});
```

### 3. Instrument Code with Async Hooks for Debugging

Track async execution flow:

```javascript
const asyncHooks = require('async_hooks');
const fs = require('fs');

const hook = asyncHooks.createHook({
  init: (asyncId, type, triggerAsyncId) => {
    fs.writeSync(1, `Init ${asyncId} (type: ${type}) triggered by ${triggerAsyncId}\n`);
  },
  before: (asyncId) => {
    fs.writeSync(1, `Before ${asyncId}\n`);
  },
  after: (asyncId) => {
    fs.writeSync(1, `After ${asyncId}\n`);
  }
});

hook.enable();
```

### 4. Use Assertion Libraries for Better Error Messages

```javascript
// Chai provides readable assertions
expect(user).to.deep.include({ email: 'test@example.com' });
// Error: expected { id: 1, name: 'Alice' } to deeply include { email: '...' }

// vs. basic jest
expect(user).toEqual({ email: 'test@example.com' });
// Error: Objects are not equal
```

### 5. Profile Regularly, Not Just Before Deployment

Integrate profiling into CI/CD:

```bash
# Benchmark script
node --prof benchmark.js
node --prof-process v8.log > profile.txt

# Compare with baseline
if grep -q "hot_function.*>10%" profile.txt; then
  echo "Performance regression detected"
  exit 1
fi
```

### 6. Use Mock Servers for External API Testing

```javascript
const nock = require('nock');

beforeEach(() => {
  nock('https://api.external.com')
    .get('/users/1')
    .reply(200, { id: 1, name: 'Alice' });
});

test('fetches user from external API', async () => {
  const user = await externalService.getUser(1);
  expect(user.name).toBe('Alice');
});
```

### 7. Document Testing Strategy

Clarify which layer tests each scenario:

```
API Coverage (E2E/Integration Tests):
  ✓ GET /users/:id - happy path
  ✓ GET /users/:id - not found
  ✓ POST /users - validation

Business Logic (Unit Tests):
  ✓ calculateDiscount - normal case
  ✓ calculateDiscount - edge cases
  ✓ calculateDiscount - invalid input

Database (Integration Tests):
  ✓ User.create - saves to database
  ✓ User.update - updates existing
  ✓ User.delete - soft delete
```

---

## Trade-offs & Limitations

### Test Speed vs. Coverage

**Fast tests**: Rely on mocks; run quickly but miss integration bugs.

**Comprehensive tests**: Include integration/E2E; slower but catch real issues.

**Practical approach**: 70% unit (mocked), 25% integration, 5% E2E.

### Local Debugging vs. Production Debugging

**Local**: Full control, reproducible, slow (requires environment setup).

**Production**: Real environment, real data, risks (must be non-intrusive).

### Profiling Overhead

**Sampling profiler** (`--prof`): Low overhead; statistical approximation.

**Instrumentation profiler**: High overhead; accurate; suitable for local debugging.

---

## Common Pitfalls

### 1. Writing Tests Instead of Understanding Requirements

```javascript
// Anti-pattern: Test doesn't validate real requirement
test('function returns something', () => {
  expect(calculatePrice(100)).toBeDefined();
});

// Better: Test validates requirement
test('calculatePrice applies 10% tax', () => {
  expect(calculatePrice(100)).toBe(110);
});
```

### 2. Over-Mocking and Testing Mock Behavior

```javascript
// Anti-pattern: Tests mock behavior, not code behavior
const mockDb = { query: jest.fn().mockResolvedValue({ data: 'mocked' }) };
expect(mockDb.query()).resolves.toEqual({ data: 'mocked' });
// This test passes but doesn't test actual integration

// Better: Test with real database or realistic mock
const user = await db.query('SELECT * FROM users WHERE id = 1');
expect(user).toHaveProperty('email');
```

### 3. Test Infrastructure Taking Longer Than Tests

```javascript
// Anti-pattern: 30-second setup for 10 tests
describe('UserService', () => {
  beforeAll(async () => {
    // Wait for database, cache, queues to start
    // Takes 30 seconds
  });
  
  test('simple function', () => { /* 1ms */ });
});

// Better: Use smaller, focused test setup
test('calculateDiscount', () => { /* no setup */ });
```

### 4. Not Testing Async Error Cases

```javascript
// Anti-pattern: Only tests happy path
test('fetches user', async () => {
  const user = await getUser(1);
  expect(user).toBeDefined();
});

// Better: Also test errors
test('throws when user not found', async () => {
  await expect(getUser(99999)).rejects.toThrow('User not found');
});
```

### 5. Ignoring Flaky Tests

```javascript
// Anti-pattern: Tests fail occasionally; developers skip them or disable them
describe.skip('flaky test', () => { /* ... */ });

// Better: Fix the root cause
// If test relies on timing, use actual async/await or condition polling
```

---

## How This Affects System Architecture

Testing and debugging practices influence architecture:

- **Testability**: Dependency injection, separation of concerns enable easier testing.
- **Observability**: Logging, tracing, and profiling enable faster debugging in production.
- **Confidence**: Test coverage and integration tests reduce fear of refactoring.
- **Performance**: Regular profiling prevents gradual degradation.

---

## Key Takeaways

1. **Test pyramid: 70% unit, 25% integration, 5% E2E provides fast feedback with real coverage.**
2. **Mock external dependencies but test real business logic; over-mocking creates false positives.**
3. **Flaky tests are worse than no tests; eliminate race conditions through proper async patterns.**
4. **Profiling is essential for performance; arbitrary "feels fast" judgments fail at scale.**
5. **Debug production issues using Node Inspector or heap snapshots; attach debugger to staging replicas.**
6. **Brittle tests create maintenance burden; test behavior, not implementation details.**

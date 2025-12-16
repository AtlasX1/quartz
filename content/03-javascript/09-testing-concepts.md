# Testing Concepts: L4 Engineering Guide

## Part 1: Testing Fundamentals

### 1.1 Testing Types & Scope

**Unit tests** validate individual functions/components in isolation. They're fast, focused, and run frequently. Mocking external dependencies isolates units.

**Integration tests** validate multiple components working together. They catch interaction bugs but are slower than unit tests.

**End-to-End (E2E) tests** validate complete user workflows through the application. They're slow but catch real-world bugs, especially UI interactions.

**Pyramid principle**: Many unit tests, fewer integration tests, fewest E2E tests (due to speed and complexity).

```javascript
// Unit test (Jest)
describe('add function', () => {
  it('should add two numbers', () => {
    expect(add(2, 3)).toBe(5);
  });

  it('should handle negative numbers', () => {
    expect(add(-5, 3)).toBe(-2);
  });
});

// Integration test
describe('UserService integration', () => {
  it('should fetch and cache user', async () => {
    const service = new UserService(mockApi);
    const user1 = await service.getUser(1);
    const user2 = await service.getUser(1);
    expect(mockApi.fetch).toHaveBeenCalledTimes(1); // Cached
  });
});

// E2E test (Cypress)
describe('Login flow', () => {
  it('should login and display dashboard', () => {
    cy.visit('/login');
    cy.get('input[name="email"]').type('user@example.com');
    cy.get('input[name="password"]').type('password');
    cy.get('button[type="submit"]').click();
    cy.url().should('include', '/dashboard');
  });
});
```

### 1.2 Test Structure & Assertions

**Arrange-Act-Assert pattern**:
1. **Arrange**: Set up test data and conditions
2. **Act**: Execute the code being tested
3. **Assert**: Verify the results

```javascript
describe('removeItem', () => {
  it('should remove item from array', () => {
    // Arrange
    const arr = [1, 2, 3];
    
    // Act
    const result = removeItem(arr, 2);
    
    // Assert
    expect(result).toEqual([1, 3]);
    expect(arr).toEqual([1, 2, 3]); // Original untouched (immutable)
  });
});
```

### 1.3 Testing Async Code

Async operations require special handling: Promises resolve asynchronously; tests must wait.

```javascript
// Async/await syntax (recommended)
test('should fetch user', async () => {
  const user = await fetchUser(1);
  expect(user.name).toBe('John');
});

// Promise syntax
test('should fetch user with promise', () => {
  return fetchUser(1).then(user => {
    expect(user.name).toBe('John');
  });
});

// Done callback
test('should fetch user with done', (done) => {
  fetchUser(1).then(user => {
    expect(user.name).toBe('John');
    done();
  });
});

// Fake timers for setTimeout/setInterval
jest.useFakeTimers();
test('should debounce function', () => {
  const fn = jest.fn();
  const debounced = debounce(fn, 300);
  debounced();
  debounced();
  debounced();
  expect(fn).not.toHaveBeenCalled();
  
  jest.advanceTimersByTime(300);
  expect(fn).toHaveBeenCalledTimes(1);
});
jest.useRealTimers();
```

---

## Part 2: Mocking & Spying

### 2.1 Mock Functions

**Mock functions** replace real implementations, tracking calls and return values. Useful for isolating units and controlling behavior.

```javascript
// Creating mock functions
const mockFn = jest.fn();
const mockFnWithImpl = jest.fn(() => 'default value');

// Tracking calls
mockFn(1, 2, 3);
expect(mockFn).toHaveBeenCalledWith(1, 2, 3);
expect(mockFn).toHaveBeenCalledTimes(1);
expect(mockFn.mock.calls).toEqual([[1, 2, 3]]);
expect(mockFn.mock.results[0].value).toBe('result');

// Conditional implementations
const mockApi = jest.fn((id) => {
  if (id === 1) return { name: 'John' };
  throw new Error('Not found');
});

// Mock implementations by call
mockFn
  .mockReturnValueOnce('first call')
  .mockReturnValueOnce('second call')
  .mockReturnValue('default');

expect(mockFn()).toBe('first call');
expect(mockFn()).toBe('second call');
expect(mockFn()).toBe('default');
```

### 2.2 Module Mocking

Mock entire modules to control their exports and behavior.

```javascript
// Original module: api.js
export const fetchUser = async (id) => { /* ... */ };

// In test file
jest.mock('./api', () => ({
  fetchUser: jest.fn()
}));

import { fetchUser } from './api';

test('should handle API error', async () => {
  fetchUser.mockRejectedValue(new Error('Network error'));
  
  const result = await getUserOrDefault(1);
  expect(result).toEqual({ name: 'Default' });
});

// Partial mocking (mock specific exports)
jest.mock('./helpers', () => ({
  ...jest.requireActual('./helpers'),
  expensiveHelper: jest.fn(() => 'mocked')
}));
```

### 2.3 Spies

**Spies** wrap existing functions, tracking calls without replacing implementations. Useful for monitoring function behavior.

```javascript
const obj = {
  method: () => 'original'
};

const spy = jest.spyOn(obj, 'method');
obj.method();

expect(spy).toHaveBeenCalled();
expect(spy.mock.results[0].value).toBe('original');

// Spy with mock implementation
spy.mockReturnValue('mocked');
expect(obj.method()).toBe('mocked');

spy.mockRestore(); // Restore original implementation
```

---

## Part 3: Coverage & Best Practices

### 3.1 Test Coverage

**Code coverage** measures what percentage of code is executed by tests. Metrics:
- **Statement coverage**: % of statements executed
- **Branch coverage**: % of conditional branches executed
- **Function coverage**: % of functions called
- **Line coverage**: % of lines executed

```javascript
// Running coverage in Jest
// package.json
{
  "scripts": {
    "test:coverage": "jest --coverage"
  }
}

// .jestrc.js
module.exports = {
  collectCoverageFrom: ['src/**/*.js'],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  }
};
```

### 3.2 Testing Best Practices

**Isolation**: Test one thing per test; mock external dependencies.

**Clarity**: Use descriptive test names; follow Arrange-Act-Assert pattern.

**Determinism**: Tests should always pass/fail; avoid flakiness (timing issues, random data).

**Speed**: Unit tests should be fast; slow tests discourage frequent running.

```javascript
// Bad: testing multiple things
test('user', () => {
  const user = new User('John');
  expect(user.name).toBe('John');
  expect(user.email).toBe('john@example.com');
  expect(user.isAdmin).toBe(false);
});

// Good: focused tests
describe('User', () => {
  test('should set name', () => {
    const user = new User('John');
    expect(user.name).toBe('John');
  });

  test('should default to non-admin', () => {
    const user = new User('John');
    expect(user.isAdmin).toBe(false);
  });
});

// Bad: flaky test (depends on timing)
test('should handle async', (done) => {
  setTimeout(() => {
    expect(data).toBeDefined();
    done();
  }, 100); // Flaky on slow machines
});

// Good: controlled timing
test('should handle async', async () => {
  const data = await fetchData();
  expect(data).toBeDefined();
});
```

---

## Interview Questions

**Q1: Explain the testing pyramid. Why is it important?**

The testing pyramid advocates many unit tests (fast, isolated), fewer integration tests, fewest E2E tests (slow, brittle). This ratio maximizes coverage while minimizing feedback time. Top-heavy pyramids (many E2E) are slow and expensive.

**Q2: What's the difference between mocking and spying?**

Mocks replace implementations entirely; spies wrap existing functions, tracking calls without replacing behavior. Spies are useful for monitoring; mocks are useful for isolation and controlling behavior.

**Q3: How do you test asynchronous code?**

Use `async/await` syntax with `await` (cleanest), return Promises (older), or `done` callbacks. For timers, use `jest.useFakeTimers()` to advance time deterministically. Mock Promise-returning functions to control resolution/rejection.

**Q4: What's code coverage and how do you achieve 80% coverage?**

Code coverage measures what % of code is executed by tests. Achieve 80% by: writing unit tests for critical paths, testing edge cases and error conditions, using coverage reports to identify gaps, and iteratively adding tests for uncovered branches.

---

## Key Takeaways

1. **Unit tests are fast and focused** - Mock dependencies; test one thing per test
2. **Integration tests catch interaction bugs** - Slower but essential for real workflows
3. **E2E tests validate user journeys** - Slowest and brittlest; use sparingly
4. **Mocks isolate units; spies monitor behavior** - Choose based on testing goal
5. **Async tests require proper handling** - Use `async/await`, fake timers, or Promises
6. **Test coverage indicates gaps** - Aim for 80%+ coverage of critical paths
7. **Arrange-Act-Assert pattern improves clarity** - Consistent structure aids readability
# Testing React Components: L4 Engineering Guide

## Part 1: React Testing Strategies & Frameworks

### 1.1 Testing Pyramid & Test Types

**Testing pyramid:** many unit tests (60%), fewer integration tests (30%), minimal E2E tests (10%). Unit tests are fast and cheap; E2E tests are slow but test real user flows.

**React Testing Library** emphasizes testing components like users do: interact with rendered output, not internal state/methods. This encourages accessible component design and tests that catch real bugs.

```jsx
// Unit test: component with props
import { render, screen } from "@testing-library/react";
import Button from "./Button";

describe("Button", () => {
  test("renders button with text", () => {
    render(<Button>Click me</Button>);
    expect(screen.getByRole("button", { name: "Click me" })).toBeInTheDocument();
  });

  test("calls onClick when clicked", () => {
    const handleClick = jest.fn();
    render(<Button onClick={handleClick}>Click me</Button>);
    screen.getByRole("button").click();
    expect(handleClick).toHaveBeenCalledTimes(1);
  });
});

// Integration test: component with state and side effects
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import UserForm from "./UserForm";

describe("UserForm", () => {
  test("submits form data", async () => {
    const handleSubmit = jest.fn();
    render(<UserForm onSubmit={handleSubmit} />);

    await userEvent.type(screen.getByLabelText("Name"), "Alice");
    await userEvent.type(screen.getByLabelText("Email"), "alice@example.com");
    await userEvent.click(screen.getByRole("button", { name: "Submit" }));

    await waitFor(() => {
      expect(handleSubmit).toHaveBeenCalledWith({
        name: "Alice",
        email: "alice@example.com"
      });
    });
  });
});
```

### 1.2 Component Testing Best Practices

**Query priority:** `getByRole` (recommended) → `getByLabelText` → `getByPlaceholderText` → `getByText`. This ensures accessible components; tests that require inaccessible selectors indicate accessibility issues.

**Avoid testing:** implementation details (state, methods), snapshot tests (brittle), testing library internals.

```jsx
// ✅ Good: test user interactions and outputs
test("increments counter", async () => {
  render(<Counter initialValue={0} />);
  expect(screen.getByText("Count: 0")).toBeInTheDocument();

  await userEvent.click(screen.getByRole("button", { name: "Increment" }));
  expect(screen.getByText("Count: 1")).toBeInTheDocument();
});

// ❌ Bad: testing implementation details
test("state updates correctly", () => {
  const { result } = renderHook(() => useCounter());
  act(() => result.current.increment());
  expect(result.current.count).toBe(1); // Testing hook state directly
});

// Testing async operations
test("displays fetched data", async () => {
  render(<UserProfile userId="1" />);
  expect(screen.getByText("Loading...")).toBeInTheDocument();

  await waitFor(() => {
    expect(screen.getByText("Alice")).toBeInTheDocument();
  });
});

// Testing error states
test("displays error message on failure", async () => {
  // Mock API to fail
  global.fetch = jest.fn(() =>
    Promise.reject(new Error("Network error"))
  );

  render(<UserProfile userId="1" />);
  await waitFor(() => {
    expect(screen.getByText("Network error")).toBeInTheDocument();
  });
});
```

---

## Part 2: Advanced Testing Patterns

### 2.1 Mocking & Test Doubles

**Mocking** replaces real dependencies with test doubles (stubs, mocks, fakes). Essential for testing in isolation and controlling side effects.

```jsx
// Mocking API calls
jest.mock("./api", () => ({
  fetchUser: jest.fn()
}));

import { fetchUser } from "./api";

test("loads user on mount", async () => {
  fetchUser.mockResolvedValue({ id: 1, name: "Alice" });

  render(<UserProfile userId="1" />);
  await screen.findByText("Alice");

  expect(fetchUser).toHaveBeenCalledWith("1");
});

// Mocking child components
jest.mock("./ExpensiveComponent", () => {
  return function DummyComponent() {
    return <div>Mocked</div>;
  };
});

// Spying on module functions
import * as api from "./api";
jest.spyOn(api, "fetchUser").mockResolvedValue({ name: "Alice" });

// Mocking timers
jest.useFakeTimers();

test("debounces search input", async () => {
  render(<SearchInput onSearch={jest.fn()} />);
  const input = screen.getByRole("textbox");

  await userEvent.type(input, "query");
  expect(onSearch).not.toHaveBeenCalled();

  jest.runAllTimers();
  expect(onSearch).toHaveBeenCalledWith("query");

  jest.useRealTimers();
});
```

### 2.2 Testing Hooks & Context

**Hooks testing** requires `renderHook` from React Testing Library. Test hooks in isolation, then test components using those hooks.

```jsx
// Testing custom hook
import { renderHook, act, waitFor } from "@testing-library/react";
import useCounter from "./useCounter";

test("increments counter", () => {
  const { result } = renderHook(() => useCounter(0));
  expect(result.current.count).toBe(0);

  act(() => result.current.increment());
  expect(result.current.count).toBe(1);
});

// Testing hooks with dependencies
test("updates when dependency changes", async () => {
  const { result, rerender } = renderHook(
    ({ userId }) => useUser(userId),
    { initialProps: { userId: "1" } }
  );

  await waitFor(() => expect(result.current.data).toBeDefined());

  rerender({ userId: "2" });
  await waitFor(() => expect(result.current.data.id).toBe("2"));
});

// Testing Context provider
import { render } from "@testing-library/react";
import { UserProvider } from "./UserContext";

test("provides user data to children", () => {
  render(
    <UserProvider initialUser={{ name: "Alice" }}>
      <UserProfile />
    </UserProvider>
  );

  expect(screen.getByText("Alice")).toBeInTheDocument();
});
```

### 2.3 Test Coverage & Coverage-Driven Development

**Code coverage** measures how much code is executed by tests. Aim for 80%+ coverage; diminishing returns beyond 90%.

**Caution:** high coverage doesn't guarantee good tests. A test covering all lines but not asserting outputs is meaningless.

```jsx
// Coverage report in package.json
{
  "scripts": {
    "test:coverage": "jest --coverage"
  }
}

// Coverage thresholds
{
  "jest": {
    "coverageThreshold": {
      "global": {
        "statements": 80,
        "branches": 75,
        "functions": 80,
        "lines": 80
      }
    }
  }
}

// Identifying untested paths
test("handles all user roles", () => {
  ["admin", "user", "guest"].forEach(role => {
    const { unmount } = render(<Dashboard userRole={role} />);
    expect(screen.getByText(new RegExp(role, "i"))).toBeInTheDocument();
    unmount();
  });
});
```

---

## Interview Questions

**Q1: Explain the difference between unit, integration, and E2E tests.**

Unit tests test individual components/functions in isolation. Integration tests test multiple components working together. E2E tests test complete user flows through real browser. Unit tests are fastest, cheapest; E2E tests are slowest but test real scenarios. Testing pyramid: many unit, fewer integration, minimal E2E.

**Q2: Why is testing component implementation details a bad idea?**

Testing implementation details (state, methods) couples tests to component internals. When you refactor internals (keeping behavior same), tests break. Instead, test component outputs and user interactions. This ensures tests verify actual user-facing behavior, not implementation.

**Q3: When should you mock dependencies vs testing real integration?**

Mock external dependencies (APIs, timers, expensive computations) to isolate component logic. Test real integration with other components you control. Mock third-party libraries to avoid network calls and control behavior. Balance: fewer mocks = more realistic tests; more mocks = faster, more isolated tests.

**Q4: How do you test asynchronous behavior and side effects?**

Use `waitFor()` to wait for async operations. Use `jest.useFakeTimers()` for timer-based code. Mock fetch/axios and use `.mockResolvedValue()` for success, `.mockRejectedValue()` for errors. Test loading states, success states, and error states separately.

---

## Key Takeaways

1. **React Testing Library emphasizes user-centric tests** - Query by role, label, text (not implementation)
2. **Query priority guides accessible component design** - Enforces ARIA landmarks and labels
3. **Avoid testing implementation details** - Test outputs and interactions, not state/methods
4. **Mock external dependencies** - Control behavior, avoid side effects during tests
5. **Test async operations with waitFor** - Wait for promises, updates, side effects
6. **Use hooks testing for custom hooks** - Test in isolation with renderHook
7. **Coverage is metric, not goal** - 80%+ coverage; diminishing returns beyond 90%
8. **Test pyramid: many unit, fewer integration, minimal E2E** - Optimize for speed and cost
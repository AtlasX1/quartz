# Hooks & State Management: L4 Engineering Guide

## Part 1: React Hooks Fundamentals

### 1.1 Hooks Rules & State Management

**Hooks** enable functional components to use state and side effects. **Hooks rules:** only call hooks from React functions (components, custom hooks), and always call them at the top level (not in conditionals/loops). Violating rules causes bugs: hooks maintain state by call order.

**State updates are asynchronous.** React batches state updates for performance; state changes don't reflect immediately after `setState()`. Functional updates avoid stale state issues.

```jsx
// useState: basic state hook
const [count, setCount] = React.useState(0);

// State updates are asynchronous and batched
function handleClick() {
  setCount(count + 1); // Scheduled, not immediate
  setCount(count + 1); // Batched with previous update
  console.log(count); // Still 0 (state not updated yet)
}

// Functional update: handles async batching
setCount(prevCount => prevCount + 1); // Always operates on latest state

// useReducer: complex state with multiple updates
const [state, dispatch] = React.useReducer(reducer, initialState);
dispatch({ type: "INCREMENT", payload: 1 });

function reducer(state, action) {
  switch (action.type) {
    case "INCREMENT":
      return { ...state, count: state.count + action.payload };
    case "RESET":
      return initialState;
    default:
      return state;
  }
}
```

### 1.2 Custom Hooks & Hook Composition

**Custom hooks** extract component logic into reusable functions. They're just JavaScript functions that call other hooks. Custom hooks enable code reuse and separation of concerns.

```jsx
// Custom hook: useWindowSize
function useWindowSize() {
  const [size, setSize] = React.useState({
    width: window.innerWidth,
    height: window.innerHeight
  });

  React.useEffect(() => {
    const handleResize = () => {
      setSize({ width: window.innerWidth, height: window.innerHeight });
    };
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return size;
}

// Usage
function ResponsiveComponent() {
  const { width, height } = useWindowSize();
  return <div>{width}x{height}</div>;
}

// Custom hook: useFetch
function useFetch(url) {
  const [data, setData] = React.useState(null);
  const [loading, setLoading] = React.useState(true);
  const [error, setError] = React.useState(null);

  React.useEffect(() => {
    fetch(url)
      .then(r => r.json())
      .then(data => { setData(data); setLoading(false); })
      .catch(err => { setError(err); setLoading(false); });
  }, [url]);

  return { data, loading, error };
}
```

### 1.3 Hook Dependencies & Stale Closures

**Dependency arrays** tell React when to re-run effects. Missing dependencies cause stale closures (referencing old values). Extra dependencies cause unnecessary re-runs.

**Stale closure problem:** Effects capture variables at definition time. If a variable changes but isn't in dependencies, the effect sees the old value.

```jsx
// Stale closure: missing dependency
function Counter() {
  const [count, setCount] = React.useState(0);

  React.useEffect(() => {
    const timer = setTimeout(() => {
      console.log(`Count: ${count}`); // Logs 0, always (stale closure)
    }, 1000);
  }, []); // Missing dependency: count

  return (
    <div>
      {count}
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

// Fixed with dependency
React.useEffect(() => {
  const timer = setTimeout(() => {
    console.log(`Count: ${count}`); // Logs current count
  }, 1000);
}, [count]); // Dependency: count

// useCallback prevents unnecessary re-renders
const memoizedCallback = React.useCallback(() => {
  // callback definition
}, [dependency]);

// useMemo prevents expensive recomputation
const memoizedValue = React.useMemo(() => {
  return expensiveComputation(a, b);
}, [a, b]);
```

---

## Part 2: State Management Patterns

### 2.1 Context API & Provider Pattern

**Context API** enables sharing state without prop drilling. **Provider** makes state available to all descendants; **Consumer** (or `useContext` hook) accesses state.

**Performance caveat:** Context changes cause all consumers to re-render, even if they don't use the changed value. Use multiple contexts for different concerns.

```jsx
// Create context
const UserContext = React.createContext();

// Provider component
function UserProvider({ children }) {
  const [user, setUser] = React.useState(null);

  return (
    <UserContext.Provider value={{ user, setUser }}>
      {children}
    </UserContext.Provider>
  );
}

// Consumer hook
function useUser() {
  const context = React.useContext(UserContext);
  if (!context) throw new Error("useUser must be within UserProvider");
  return context;
}

// Usage
function App() {
  return (
    <UserProvider>
      <Header />
      <Dashboard />
    </UserProvider>
  );
}

function Dashboard() {
  const { user } = useUser(); // Access context
  return <div>Welcome, {user?.name}</div>;
}
```

### 2.2 Reducer Pattern & Action Dispatching

**useReducer** manages complex state with multiple related actions. Reducer pattern is familiar to Redux users and scales better than multiple `useState` calls.

```jsx
// Reducer function
function appReducer(state, action) {
  switch (action.type) {
    case "SET_USER":
      return { ...state, user: action.payload };
    case "SET_LOADING":
      return { ...state, loading: action.payload };
    case "SET_ERROR":
      return { ...state, error: action.payload };
    default:
      return state;
  }
}

// Initial state
const initialState = { user: null, loading: false, error: null };

// Component using reducer
function App() {
  const [state, dispatch] = React.useReducer(appReducer, initialState);

  const fetchUser = async (id) => {
    dispatch({ type: "SET_LOADING", payload: true });
    try {
      const user = await api.getUser(id);
      dispatch({ type: "SET_USER", payload: user });
    } catch (error) {
      dispatch({ type: "SET_ERROR", payload: error.message });
    }
  };

  return <Dashboard state={state} onFetch={fetchUser} />;
}
```

---

## Interview Questions

**Q1: Explain hooks rules and why they're important.**

Hooks must be called only from React functions (components, custom hooks) at the top level (not conditionally). Hooks maintain state by call order; violating rules breaks this order, causing bugs. Rules ensure consistent hook behavior across renders.

**Q2: What's the stale closure problem and how do you solve it?**

Stale closure occurs when effects capture outdated variable values. Solution: include all variables used in effects in the dependency array. If a variable is missing, the effect sees the old value, causing bugs. Use `useCallback` and `useMemo` to control dependencies.

**Q3: Explain the difference between Context API and state management libraries like Redux.**

Context API is built-in; good for small-medium applications and simple state sharing. Redux/Zustand provide predictable state management, time-travel debugging, middleware. Use Context for simple cases; use libraries for complex apps with multiple interacting state pieces.

**Q4: When should you use useReducer instead of useState?**

Use `useReducer` when: state has multiple related fields, updates depend on current state (avoids multiple `setState` calls), logic is complex. Use `useState` for independent state values. `useReducer` scales better for complex state machines.

---

## Key Takeaways

1. **Hooks enable state in functional components** - Call only at top level, not conditionally
2. **State updates are asynchronous and batched** - Use functional updates for correct state
3. **Custom hooks extract reusable logic** - Just JavaScript functions calling hooks
4. **Dependencies control when effects run** - Missing dependencies cause stale closures
5. **useReducer manages complex state** - Multiple related actions and predictable updates
6. **Context API avoids prop drilling** - But all consumers re-render on any value change
7. **useCallback and useMemo optimize** - Prevent unnecessary re-renders and recomputation
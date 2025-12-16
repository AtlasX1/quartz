# State Management Patterns in React: L4 Engineering Guide

## Part 1: State Management Approaches & Tradeoffs

### 1.1 Built-In Solutions: Context API vs Local State

**Local state** (`useState`, `useReducer`) manages component-specific data. **Context API** shares data across components without prop drilling. **Global state** (Redux, Zustand) manages application-wide data with predictable updates.

**Guidelines:** Use local state for component-specific data. Use Context for moderate sharing (user auth, theme). Use global state libraries for complex apps with many interacting state pieces, time-travel debugging needs.

```jsx
// Local state: simplest solution
function Counter() {
  const [count, setCount] = React.useState(0);
  return (
    <div>
      Count: {count}
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

// Context API: sharing without prop drilling
const ThemeContext = React.createContext();

function App() {
  const [theme, setTheme] = React.useState("light");
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Header />
      <MainContent />
      <Footer />
    </ThemeContext.Provider>
  );
}

function MainContent() {
  const { theme } = React.useContext(ThemeContext);
  return <div className={theme}>Content</div>;
}

// Problem: Context causes all consumers to re-render on any value change
// Solution: split contexts by concern or use state management library
const UserContext = React.createContext(); // User data
const NotificationContext = React.createContext(); // Notifications
```

### 1.2 Redux: Predictable State Management

**Redux** centralizes state in a single store, uses pure functions (reducers) to update state, and enforces unidirectional data flow. Enables time-travel debugging, middleware, predictable state shape.

**Learning curve:** Redux requires boilerplate (actions, reducers, selectors). Modern alternatives (Zustand, Jotai) are simpler.

```jsx
// Redux store setup
import { createStore } from "redux";

// Action types
const INCREMENT = "INCREMENT";
const SET_USER = "SET_USER";

// Actions
const increment = () => ({ type: INCREMENT });
const setUser = (user) => ({ type: SET_USER, payload: user });

// Reducer
const initialState = { count: 0, user: null };
function rootReducer(state = initialState, action) {
  switch (action.type) {
    case INCREMENT:
      return { ...state, count: state.count + 1 };
    case SET_USER:
      return { ...state, user: action.payload };
    default:
      return state;
  }
}

const store = createStore(rootReducer);

// Usage with React
import { useSelector, useDispatch } from "react-redux";

function Counter() {
  const count = useSelector(state => state.count);
  const dispatch = useDispatch();

  return (
    <div>
      Count: {count}
      <button onClick={() => dispatch(increment())}>+</button>
    </div>
  );
}

// Selectors: reusable state queries
const selectCount = (state) => state.count;
const selectUser = (state) => state.user;

// Reselect: memoized selectors prevent unnecessary re-renders
import { createSelector } from "reselect";

const selectUserPosts = createSelector(
  (state) => state.user,
  (state) => state.posts,
  (user, posts) => posts.filter(p => p.userId === user.id)
);
```

---

## Part 2: Modern State Management Libraries

### 2.1 Zustand: Minimal State Management

**Zustand** is a lightweight alternative to Redux. Simpler API, no boilerplate, built-in middleware.

```jsx
import { create } from "zustand";

// Create store
const useStore = create((set) => ({
  count: 0,
  user: null,

  // Actions
  increment: () => set((state) => ({ count: state.count + 1 })),
  setUser: (user) => set({ user }),
  reset: () => set({ count: 0, user: null })
}));

// Usage in components
function Counter() {
  const count = useStore((state) => state.count);
  const increment = useStore((state) => state.increment);

  return (
    <div>
      Count: {count}
      <button onClick={increment}>+</button>
    </div>
  );
}

// Middleware for persistence
import { persist } from "zustand/middleware";

const useStore = create(
  persist(
    (set) => ({ count: 0, increment: () => set((s) => ({ count: s.count + 1 })) }),
    { name: "store" } // localStorage key
  )
);

// DevTools
import { devtools } from "zustand/middleware";

const useStore = create(
  devtools((set) => ({
    count: 0,
    increment: () => set((s) => ({ count: s.count + 1 }))
  }), { name: "CountStore" })
);
```

### 2.2 Jotai: Primitive State Management

**Jotai** treats state as atoms (primitives) that compose. Enables fine-grained reactivity: only components reading changed atoms re-render.

```jsx
import { atom, useAtom } from "jotai";

// Define atoms
const countAtom = atom(0);
const userAtom = atom(null);

// Derived atoms (computed state)
const doubleCountAtom = atom((get) => get(countAtom) * 2);

// Usage
function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const [doubleCount] = useAtom(doubleCountAtom);

  return (
    <div>
      Count: {count} (Double: {doubleCount})
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

// Async atoms
const userAtom = atom(async (get) => {
  const response = await fetch("/api/user");
  return response.json();
});

function User() {
  const [user] = useAtom(userAtom);
  return <div>{user?.name}</div>;
}

// Fine-grained updates: only countAtom consumers re-render
// If nameAtom changes, Counter doesn't re-render
const nameAtom = atom("Alice");

function Profile() {
  const [name, setName] = useAtom(nameAtom);
  return <div>{name}</div>;
}
```

### 2.3 When to Use State Management Libraries

**Use local state:** for component-specific data (form inputs, UI toggles).

**Use Context:** for moderate sharing (user auth, theme, notifications).

**Use state library (Redux, Zustand, Jotai):** for complex apps with:
- Multiple sources of truth that interact
- Complex state updates (undo/redo, time-travel debugging)
- Many components reading/updating same state
- Need for middleware (logging, persistence, devtools)

```jsx
// Decision tree example
function App() {
  // Global state: use library
  const user = useStore((s) => s.user);
  const notifications = useStore((s) => s.notifications);

  // Shared across many components: use Context
  return (
    <ThemeContext.Provider value={theme}>
      {/* Component-specific: use useState */}
      <Page />
    </ThemeContext.Provider>
  );
}

function Page() {
  const [isOpen, setIsOpen] = React.useState(false); // Local state
  const theme = React.useContext(ThemeContext); // Context
  const user = useStore((s) => s.user); // Global state
}
```

---

## Interview Questions

**Q1: When should you use Context API vs a state management library like Redux or Zustand?**

Use Context for moderate sharing (user auth, theme) that doesn't change frequently. Use state libraries for complex apps with multiple interacting state pieces, frequent updates, or time-travel debugging. Context causes all consumers to re-render on any value change; libraries handle fine-grained updates better.

**Q2: What are advantages of Redux over Context API?**

Redux provides time-travel debugging, middleware (logging, persistence), devtools, predictable state shape, and memoized selectors that prevent unnecessary re-renders. Redux enforces unidirectional data flow. Trade-off: Redux requires more boilerplate.

**Q3: Explain the difference between Zustand and Jotai.**

Zustand is a single central store (like Redux, simpler). Jotai treats state as atoms that compose; enables fine-grained reactivity (only components reading changed atoms re-render). Zustand is more familiar to Redux users; Jotai is more flexible for complex dependency graphs.

**Q4: How do you prevent unnecessary re-renders with Context?**

Split contexts by concern (one for user, one for notifications). Use `useCallback` for context values. Use memoization on components. Better solution: use state management library with fine-grained updates (Zustand, Jotai).

---

## Key Takeaways

1. **Local state (useState) for component-specific data** - Simplest, no prop drilling within component
2. **Context API avoids prop drilling** - Good for moderate sharing; all consumers re-render on value change
3. **Redux provides predictable, debuggable state** - Time-travel debugging, middleware, complex state
4. **Zustand: lightweight Redux alternative** - Minimal boilerplate, built-in middleware
5. **Jotai: atomic state composition** - Fine-grained reactivity, only affected components re-render
6. **Selector memoization (Reselect) prevents re-renders** - Memoized selectors return same reference if input unchanged
7. **Choose based on complexity** - Simple apps: local state + Context; complex apps: state library
8. **DevTools and middleware enable debugging** - Redux/Zustand devtools, time-travel debugging, persistence
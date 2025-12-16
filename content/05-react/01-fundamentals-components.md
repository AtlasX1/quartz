# React Fundamentals & Components: L4 Engineering Guide

## Part 1: React Core Concepts

### 1.1 Virtual DOM & Reconciliation

React maintains a **Virtual DOM** (in-memory representation of the actual DOM) to optimize updates. When state/props change, React creates a new Virtual DOM tree, diffs it against the previous one, and applies minimal DOM changes.

**Reconciliation** (diffing algorithm) compares two Virtual DOM trees to determine what changed. React uses **keys** (in lists) to maintain element identity across renders, preventing unnecessary remounting.

**Performance implications:** The Virtual DOM doesn't magically optimize everything. Unnecessary re-renders still waste CPU. React's reconciliation is $O(n)$ (linear) with heuristics: elements with different types become different subtrees; elements with keys maintain identity.

```jsx
// Keyed list: maintains component state across reorders
function UserList({ users }) {
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li> // key maintains identity
      ))}
    </ul>
  );
}

// Without key: can cause state loss and bugs
{users.map(user => <li>{user.name}</li>)} // Bad: index as key
```

### 1.2 Components & Composition

**Components** are functions or classes returning JSX (React syntax for describing UI). React emphasizes **composition over inheritance**—combine simple components to build complex UIs.

**Unidirectional data flow:** Data flows down (props); events flow up (callbacks). This makes data flow predictable and easier to reason about.

```jsx
// Functional component (modern standard)
function UserCard({ user, onDelete }) {
  return (
    <div className="card">
      <h2>{user.name}</h2>
      <button onClick={() => onDelete(user.id)}>Delete</button>
    </div>
  );
}

// Composition: combine components
function UserList({ users, onDelete }) {
  return (
    <div>
      {users.map(user => (
        <UserCard key={user.id} user={user} onDelete={onDelete} />
      ))}
    </div>
  );
}
```

### 1.3 JSX & Transpilation

**JSX** is syntactic sugar for `React.createElement()`. Transpilers (Babel) convert JSX to function calls at build time.

```jsx
// JSX
const element = <div className="card"><h1>Hello</h1></div>;

// Equivalent JavaScript
const element = React.createElement(
  "div",
  { className: "card" },
  React.createElement("h1", null, "Hello")
);
```

---

## Part 2: Functional Components & Props

### 2.1 Props & Component Contracts

**Props** are immutable inputs to components, enabling component customization. Props should define a clear contract: what data is required, what's optional, what types are expected.

**Prop drilling** (passing props through many intermediary components) can become problematic in deep component trees. Context API or state management libraries address this.

```jsx
// Props contract with TypeScript
interface UserCardProps {
  user: { id: number; name: string; email: string };
  onDelete: (id: number) => void;
  onEdit?: (user: User) => void;
}

function UserCard({ user, onDelete, onEdit }: UserCardProps) {
  return (
    <div>
      <h2>{user.name}</h2>
      <p>{user.email}</p>
      <button onClick={() => onDelete(user.id)}>Delete</button>
      {onEdit && <button onClick={() => onEdit(user)}>Edit</button>}
    </div>
  );
}

// DefaultProps & prop validation (legacy, TypeScript preferred)
UserCard.defaultProps = {
  onEdit: undefined
};
```

### 2.2 Children & Composition Patterns

**Children** (content passed between opening/closing tags) enables flexible composition. `children` is a special prop containing everything between tags.

```jsx
// Children pattern
function Card({ title, children }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="content">{children}</div>
    </div>
  );
}

// Usage
<Card title="User">
  <p>User details here</p>
  <button>Action</button>
</Card>

// Render props pattern (alternative composition)
function DataProvider({ render }) {
  const [data, setData] = React.useState(null);
  return render(data);
}

<DataProvider render={data => <div>{data}</div>} />
```

---

## Part 3: Component Lifecycle

### 3.1 Functional Component Lifecycle with Useeffect

**Functional components** don't have lifecycle methods. Instead, **`useEffect`** hook handles side effects (API calls, subscriptions, DOM manipulation).

`useEffect` runs after every render by default; dependency arrays control when it runs. Dependencies: empty array (runs once), no array (runs every render), specific dependencies (runs when deps change).

```jsx
// Runs after every render (avoid: performance issue)
useEffect(() => {
  console.log("Rendered");
});

// Runs once on mount (common for initialization)
useEffect(() => {
  fetchUser();
}, []);

// Runs when specific dependencies change
useEffect(() => {
  subscribeToUpdates(userId);
  return () => unsubscribe(); // Cleanup function
}, [userId]);

// Multiple effects for different concerns
useEffect(() => { /* title side effect */ }, [title]);
useEffect(() => { /* auth side effect */ }, [userId]);
```

### 3.2 Cleanup & Memory Leak Prevention

**Cleanup functions** (returned from `useEffect`) run when the component unmounts or before the effect runs again. Essential for preventing memory leaks.

Memory leaks occur from: subscriptions not unsubscribed, timers not cleared, event listeners not removed, closures retaining large objects.

```jsx
// Event listener memory leak
useEffect(() => {
  const handleResize = () => console.log(window.innerWidth);
  window.addEventListener("resize", handleResize);
  // Missing cleanup: listener never removed
}, []);

// Fixed with cleanup
useEffect(() => {
  const handleResize = () => console.log(window.innerWidth);
  window.addEventListener("resize", handleResize);
  return () => window.removeEventListener("resize", handleResize); // Cleanup
}, []);

// Subscription memory leak
useEffect(() => {
  const subscription = dataStore.subscribe(() => {
    // Handle update
  });
  return () => subscription.unsubscribe(); // Cleanup
}, []);
```

---

## Interview Questions

**Q1: Explain the Virtual DOM and how reconciliation works.**

React maintains a Virtual DOM (in-memory tree). When state changes, React creates a new Virtual DOM, diffs it against the old one, and applies minimal DOM changes. Reconciliation uses keys to maintain element identity; elements without keys can lose state during reorders.

**Q2: What's prop drilling and how do you avoid it?**

Prop drilling is passing props through many intermediary components, leading to tight coupling and maintenance issues. Avoid with: Context API (global state), state management libraries (Redux, Zustand), or restructuring component hierarchy.

**Q3: Explain useEffect cleanup functions. Why are they important?**

Cleanup functions run when components unmount or before the effect runs again. Essential for preventing memory leaks: removing event listeners, unsubscribing from observables, clearing timers. Every effect with side effects should have a cleanup function.

**Q4: When should you use children vs render props vs component composition?**

Use `children` for simple containment (Card, Modal). Use render props for data sharing without intermediate components. Use component composition for flexibility. Generally, `children` is simpler; use render props/composition only when necessary.

---

## Key Takeaways

1. **Virtual DOM optimizes updates** - React diffs trees and applies minimal changes
2. **Keys maintain element identity** - Prevent state loss during list reorders
3. **Props define component contracts** - Unidirectional data flow (top to bottom)
4. **useEffect manages side effects** - Dependency arrays control when effects run
5. **Cleanup functions prevent memory leaks** - Always clean up subscriptions and listeners
6. **Composition over inheritance** - Combine simple components to build complex UIs
7. **Children enable flexible composition** - Generic containers without tight coupling
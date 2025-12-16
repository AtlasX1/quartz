# Performance Optimization in React: L4 Engineering Guide

## Part 1: Rendering Performance & Memoization

### 1.1 React Reconciliation & Re-render Minimization

**React reconciliation** (diffing algorithm) compares old and new component trees. Reconciliation is $O(n)$ but heuristic-based: assumes siblings have stable keys and different component types rarely produce similar trees.

**Re-renders happen** when parent state/props change or component state changes. To minimize re-renders: memoize components, use `useCallback` for stable function refs, split state into smaller chunks, or use state management libraries.

```jsx
// Problem: expensive component re-renders unnecessarily
function Parent() {
  const [count, setCount] = React.useState(0);
  const [name, setName] = React.useState("Alice");

  // Child re-renders even when count doesn't affect it
  return (
    <>
      <Child name={name} />
      <button onClick={() => setCount(count + 1)}>Count: {count}</button>
    </>
  );
}

// Solution 1: React.memo prevents re-renders if props unchanged
const Child = React.memo(function Child({ name }) {
  console.log("Child rendered"); // Logs only when name changes
  return <div>{name}</div>;
});

// Solution 2: Move state closer to where it's used
function Parent() {
  return (
    <>
      <NameComponent />
      <CountComponent />
    </>
  );
}
```

### 1.2 useCallback & useMemo for Stable References

**useCallback** memoizes function references; returns same function if dependencies unchanged. Prevents child re-renders when function props are compared.

**useMemo** memoizes computed values; returns same value if dependencies unchanged. Prevents expensive recomputation.

```jsx
// Problem: inline functions cause re-renders
function Parent() {
  const [count, setCount] = React.useState(0);

  // handleClick is recreated every render -> Child always re-renders
  return (
    <Child
      onClick={() => console.log(count)}
      items={[1, 2, 3]}
    />
  );
}

// Solution: useCallback for stable functions
function Parent() {
  const [count, setCount] = React.useState(0);

  const handleClick = React.useCallback(() => {
    console.log(count); // Captures count in closure
  }, [count]); // Re-creates only when count changes

  const items = React.useMemo(() => [1, 2, 3], []); // Stable array

  return <Child onClick={handleClick} items={items} />;
}

const Child = React.memo(function Child({ onClick, items }) {
  // Only re-renders when onClick or items reference actually changes
  return <button onClick={onClick}>Items: {items.length}</button>;
});

// useMemo: prevent expensive computation
function ProductList({ products, filter }) {
  const filtered = React.useMemo(() => {
    return products.filter(p => p.category === filter);
  }, [products, filter]); // Recomputes only when dependencies change

  return filtered.map(p => <ProductItem key={p.id} product={p} />);
}
```

---

## Part 2: Advanced Performance Strategies

### 2.1 Code Splitting & Lazy Loading

**Code splitting** breaks bundle into chunks, loading only necessary code. **Lazy loading** defers component loading until needed, reducing initial bundle size.

React provides `React.lazy()` and `Suspense` for component-level code splitting. Route-based splitting (splitting by route) is most effective.

```jsx
// Code splitting with React.lazy
const HeavyComponent = React.lazy(() => import("./HeavyComponent"));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <HeavyComponent />
    </Suspense>
  );
}

// Route-based splitting (Next.js/React Router)
const Dashboard = React.lazy(() => import("./pages/Dashboard"));
const Settings = React.lazy(() => import("./pages/Settings"));

function Routes() {
  return (
    <Switch>
      <Route path="/dashboard">
        <Suspense fallback={<Loading />}>
          <Dashboard />
        </Suspense>
      </Route>
      <Route path="/settings">
        <Suspense fallback={<Loading />}>
          <Settings />
        </Suspense>
      </Route>
    </Switch>
  );
}

// Prefetch: load before user needs it
React.useEffect(() => {
  const controller = new AbortController();
  import("./HeavyComponent");
  return () => controller.abort();
}, []);
```

### 2.2 Windowing & Virtual Scrolling

**Windowing** (virtual scrolling) renders only visible items in a large list. Instead of rendering 10,000 items, render only ~20 visible ones. Dramatically improves performance for large lists.

Libraries: `react-window`, `react-virtualized`, or manual implementation.

```jsx
// Manual windowing example
function VirtualList({ items, itemHeight, windowHeight }) {
  const [scrollTop, setScrollTop] = React.useState(0);

  const startIndex = Math.floor(scrollTop / itemHeight);
  const endIndex = Math.ceil((scrollTop + windowHeight) / itemHeight);
  const visibleItems = items.slice(startIndex, endIndex);

  return (
    <div
      style={{ height: windowHeight, overflow: "auto" }}
      onScroll={(e) => setScrollTop(e.target.scrollTop)}
    >
      <div style={{ height: items.length * itemHeight }}>
        <div style={{ transform: `translateY(${startIndex * itemHeight}px)` }}>
          {visibleItems.map((item, i) => (
            <div key={startIndex + i} style={{ height: itemHeight }}>
              {item.name}
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}

// Using react-window
import { FixedSizeList } from "react-window";

function List({ items }) {
  return (
    <FixedSizeList
      height={600}
      itemCount={items.length}
      itemSize={35}
      width="100%"
    >
      {({ index, style }) => (
        <div style={style}>{items[index].name}</div>
      )}
    </FixedSizeList>
  );
}
```

### 2.3 Suspense for Data Fetching

**Suspense** lets components "suspend" rendering while data loads. Wrapping component shows fallback UI until all suspended children resolve.

```jsx
// Component that suspends
const UserProfile = ({ userId }) => {
  const user = use(fetchUser(userId)); // Suspends while fetching
  return <div>{user.name}</div>;
};

// Suspense boundary
function App({ userId }) {
  return (
    <Suspense fallback={<Loading />}>
      <UserProfile userId={userId} />
    </Suspense>
  );
}

// Nested Suspense for granular loading
function Dashboard() {
  return (
    <div>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>
      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>
    </div>
  );
}
```

---

## Interview Questions

**Q1: Explain React's reconciliation algorithm and how keys affect it.**

React uses heuristic-based diffing ($O(n)$ complexity). Keys help React identify which items have changed. Without keys or with index keys in dynamic lists, React may reuse DOM nodes incorrectly, causing bugs or performance issues. Stable, unique keys are essential for list performance.

**Q2: When should you use React.memo, useCallback, and useMemo?**

Use `React.memo` when component is expensive and props rarely change. Use `useCallback` when passing functions to memoized children. Use `useMemo` for expensive computations. However, premature memoization can hurt performance (memoization itself has overhead). Profile first, then optimize.

**Q3: Explain code splitting and its performance impact.**

Code splitting reduces initial bundle size by loading code on-demand. Route-based splitting is most effective. Trade-off: slightly slower navigation (network request + parsing), but faster initial load. Essential for large applications.

**Q4: What's the difference between windowing and infinite scroll?**

Windowing renders only visible items; scales to millions of items ($O(1)$ rendered items). Infinite scroll appends all loaded items; scales poorly ($O(n)$ rendered items). Windowing is more performant but requires explicit implementation.

---

## Key Takeaways

1. **React reconciliation is $O(n)$ heuristic-based** - Stable keys are essential for correctness
2. **Memoization has overhead** - Profile before optimizing; premature memoization hurts performance
3. **useCallback and useMemo stabilize references** - Use when passing to memoized children
4. **Code splitting reduces initial bundle** - Load routes/features on-demand
5. **Windowing enables rendering large lists** - Only render visible items
6. **Suspense coordinates async operations** - Load data in parallel; show fallback during loading
7. **Move state closer to usage** - Reduces unnecessary re-renders of unrelated components
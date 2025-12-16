# Advanced React Concepts: L4 Engineering Guide

## Part 1: Concurrent Rendering & Fiber Architecture

### 1.1 React Fiber: Rendering Engine Architecture

**React Fiber** breaks rendering into small units (fibers) that can be paused and resumed. Enables prioritization: high-priority updates (user input) interrupt low-priority updates (data fetching).

**Before Fiber:** rendering was synchronous; blocking long renders froze UI.
**After Fiber:** rendering is interruptible; high-priority work preempts low-priority work.

```jsx
// Fiber architecture (internal, not exposed in API)
// React parses component tree into fiber tree (linked list)
// Each fiber: component, props, state, children, sibling, parent
// Rendering phases:
// 1. Render phase (can pause/resume/restart): pure, no side effects
// 2. Commit phase (atomic): apply DOM changes, run effects

// Concurrent rendering enables:
// - Prioritized updates: user input > data fetching > CPU-bound work
// - Interruptible rendering: pause for user input, resume later
// - Suspense: pause rendering until async data available

// Time slicing: yield to browser for UI responsiveness
// React renders for ~5ms, yields to browser (~1ms tasks), resumes
```

### 1.2 useTransition for Non-Blocking Updates

**useTransition** marks updates as non-blocking (low-priority). Useful for expensive operations (filtering large lists, searches) that shouldn't freeze UI.

```jsx
// useTransition for expensive state updates
function SearchList({ items }) {
  const [query, setQuery] = React.useState("");
  const [isPending, startTransition] = React.useTransition();

  const filteredItems = React.useMemo(() => {
    if (!query) return items;
    return items.filter(item => item.name.includes(query)); // Expensive: O(n)
  }, [query, items]);

  const handleSearch = (e) => {
    const newQuery = e.target.value;
    // Start non-blocking update
    startTransition(() => {
      setQuery(newQuery);
    });
  };

  return (
    <>
      <input
        value={query}
        onChange={handleSearch}
        placeholder="Search..."
      />
      {isPending && <p>Searching...</p>}
      <ul>
        {filteredItems.map(item => (
          <li key={item.id}>{item.name}</li>
        ))}
      </ul>
    </>
  );
}

// useDeferredValue: defer state update
function SearchResults({ query }) {
  const deferredQuery = React.useDeferredValue(query);
  const [results, setResults] = React.useState([]);

  React.useEffect(() => {
    const search = async () => {
      const results = await api.search(deferredQuery);
      setResults(results);
    };
    search();
  }, [deferredQuery]);

  return <ul>{results.map(r => <li key={r.id}>{r.name}</li>)}</ul>;
}

// Input updates immediately; search deferred
<input value={query} onChange={e => setQuery(e.target.value)} />
<SearchResults query={query} />
```

---

## Part 2: Advanced React 18+ Features

### 2.1 Suspense for Async Operations

**Suspense** pauses component rendering until async dependencies resolve. Enables waterfall-free (parallel) data fetching and graceful loading states.

```jsx
// Component that suspends
const UserProfile = ({ userId }) => {
  // This "throws" a promise that Suspense catches
  const user = use(fetchUser(userId));
  return <div>{user.name}</div>;
};

// Suspense boundary handles loading
function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <UserProfile userId="123" />
    </Suspense>
  );
}

// Parallel suspense boundaries (independent loading)
function Dashboard() {
  return (
    <div>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>

      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>

      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>
    </div>
  );
}

// Avoid waterfall: fetch in parallel
// ❌ Waterfall: fetch Header, then Sidebar (sequential)
// ✅ Parallel: fetch both simultaneously
function App() {
  return (
    <>
      <Suspense fallback={<HeaderSkeleton />}>
        <Header />
      </Suspense>
      <Suspense fallback={<ContentSkeleton />}>
        <MainContent />
      </Suspense>
    </>
  );
}
```

### 2.2 React Server Components (RSC)

**Server Components** render on server, send minimal JavaScript to client. Enables direct database access, secrets, reduced bundle size.

```jsx
// Server Component (by default in Next.js)
async function PostList() {
  // Direct database access (only runs on server)
  const posts = await db.post.findMany();
  
  // No JavaScript sent to client (no hydration needed)
  return (
    <ul>
      {posts.map(post => (
        <li key={post.id}>{post.title}</li>
      ))}
    </ul>
  );
}

// Client Component: enables interactivity
"use client"; // Mark as client component
import { useState } from "react";

export function SearchBox() {
  const [query, setQuery] = useState("");
  return (
    <input
      value={query}
      onChange={(e) => setQuery(e.target.value)}
      placeholder="Search posts..."
    />
  );
}

// Mixing Server + Client Components
// Server Component (no "use client")
async function PostPage() {
  const posts = await db.post.findMany();

  return (
    <div>
      <SearchBox /> {/* Client component: bundled to client */}
      <PostList posts={posts} /> {/* Server component: rendered server */}
    </div>
  );
}

// Server Actions: call server functions from client
"use server"

export async function deletePost(postId) {
  await db.post.delete(postId);
  // Data refetch happens server-side
}

// Client component calls server action
"use client"
export function DeleteButton({ postId }) {
  const handleDelete = async () => {
    await deletePost(postId);
  };
  return <button onClick={handleDelete}>Delete</button>;
}
```

### 2.3 Error Boundaries & Error Handling

**Error Boundaries** catch errors in descendant components, preventing full app crash. Only catch render-phase errors, not async errors or event handlers.

```jsx
// Error Boundary class component
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    console.error("Error caught:", error, errorInfo);
    // Log to error reporting service
  }

  render() {
    if (this.state.hasError) {
      return <div>Something went wrong: {this.state.error.message}</div>;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <DangerousComponent />
</ErrorBoundary>

// Granular error boundaries
<ErrorBoundary>
  <Header />
  <ErrorBoundary>
    <MainContent />
  </ErrorBoundary>
  <ErrorBoundary>
    <Sidebar />
  </ErrorBoundary>
</ErrorBoundary>

// Handling async errors (not caught by error boundary)
function AsyncComponent() {
  const [error, setError] = React.useState(null);

  React.useEffect(() => {
    fetch("/api/data")
      .catch(err => setError(err)); // Handle manually
  }, []);

  if (error) return <div>Error: {error.message}</div>;
  return <div>Data</div>;
}
```

---

## Interview Questions

**Q1: Explain React Fiber and how it enables concurrent rendering.**

React Fiber breaks rendering into small units that can be paused/resumed. High-priority updates (user input) can interrupt low-priority updates (data fetching). Before Fiber, rendering was synchronous and blocking. Fiber enables interruptible rendering via time slicing: React renders for ~5ms, yields to browser, resumes.

**Q2: What's the difference between useTransition and useDeferredValue?**

`useTransition` marks state update as non-blocking; provides `isPending` flag. `useDeferredValue` defers a derived value. Both reduce update priority. useTransition useful for expensive operations; useDeferredValue for derived state. useDeferredValue returns old value while new value computes.

**Q3: Explain React Server Components and their benefits.**

Server Components render on server, send minimal JavaScript to client. Direct database access (no API needed), secrets stay server-side, reduced bundle size. Trade-off: less interactivity (need client components for event handlers). Combines SSR benefits (SEO) with reduced JavaScript.

**Q4: What errors do Error Boundaries catch? What don't they catch?**

Error Boundaries catch render-phase errors (constructor, render, lifecycle methods). They don't catch: async errors (setTimeout, fetch), event handler errors, Server Component errors. Handle async/event errors manually with try/catch or error state.

---

## Key Takeaways

1. **React Fiber enables concurrent rendering** - Interruptible rendering via time slicing
2. **High-priority updates preempt low-priority work** - User input > data fetching > CPU-bound
3. **useTransition marks non-blocking updates** - Prevents UI freeze on expensive operations
4. **useDeferredValue defers value computation** - New value computes in background
5. **Suspense pauses rendering until dependencies resolve** - Enables parallel data fetching
6. **Server Components reduce bundle and enable direct DB access** - Server-rendered, no hydration needed
7. **Server Actions call server functions from client** - RPC-like pattern for mutations
8. **Error Boundaries catch render-phase errors** - Don't catch async/event errors (handle manually)
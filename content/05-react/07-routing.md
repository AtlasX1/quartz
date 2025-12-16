# Client-Side Routing in React: L4 Engineering Guide

## Part 1: Routing Fundamentals & Implementation

### 1.1 Client-Side Routing vs Server-Side Routing

**Client-side routing** renders different components based on URL without full page reloads. JavaScript intercepts navigation, updates DOM, and changes browser history. **Single-page applications** (SPAs) use client-side routing.

**Server-side routing** serves different HTML per URL; each navigation requires server request. Slower but simpler, better for traditional multi-page apps.

**Trade-offs:** Client-side routing enables fast navigation, smooth transitions, offline support; requires JavaScript, larger initial bundle, complexity managing browser history.

```jsx
// Basic client-side routing with window.location
function Router() {
  const [path, setPath] = React.useState(window.location.pathname);

  React.useEffect(() => {
    const handlePopState = () => {
      setPath(window.location.pathname);
    };
    window.addEventListener("popstate", handlePopState);
    return () => window.removeEventListener("popstate", handlePopState);
  }, []);

  const navigate = (newPath) => {
    window.history.pushState({}, "", newPath);
    setPath(newPath);
  };

  return (
    <>
      <nav>
        <button onClick={() => navigate("/")}>Home</button>
        <button onClick={() => navigate("/about")}>About</button>
        <button onClick={() => navigate("/contact")}>Contact</button>
      </nav>

      {path === "/" && <Home />}
      {path === "/about" && <About />}
      {path === "/contact" && <Contact />}
    </>
  );
}

// React Router (industry standard)
import { BrowserRouter, Routes, Route, Link, useNavigate } from "react-router-dom";

function App() {
  return (
    <BrowserRouter>
      <nav>
        <Link to="/">Home</Link>
        <Link to="/about">About</Link>
      </nav>

      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/user/:id" element={<UserProfile />} />
      </Routes>
    </BrowserRouter>
  );
}

function UserProfile() {
  const { id } = useParams(); // Extract URL parameter
  const navigate = useNavigate();

  return (
    <div>
      <h1>User {id}</h1>
      <button onClick={() => navigate("/")}>Back Home</button>
    </div>
  );
}
```

### 1.2 Route Matching & Parameter Extraction

**Route matching** determines which component to render based on URL. **Path parameters** extract dynamic segments (e.g., `/user/:id` matches `/user/123`). **Query parameters** pass optional data (e.g., `/search?q=react&sort=date`).

```jsx
// Dynamic routes and parameters
<Routes>
  {/* Exact match */}
  <Route path="/" element={<Home />} />
  
  {/* Path parameter */}
  <Route path="/user/:id" element={<UserProfile />} />
  
  {/* Multiple parameters */}
  <Route path="/post/:postId/comment/:commentId" element={<Comment />} />
  
  {/* Wildcard catch-all */}
  <Route path="*" element={<NotFound />} />
</Routes>

function UserProfile() {
  const { id } = useParams();
  const [searchParams] = useSearchParams();
  const tab = searchParams.get("tab") || "profile";

  return <div>User {id}, Tab: {tab}</div>;
}

// Query parameter example
// URL: /search?q=react&sort=date
function SearchResults() {
  const [searchParams, setSearchParams] = useSearchParams();
  const query = searchParams.get("q");
  const sort = searchParams.get("sort") || "relevance";

  const updateParams = (newSort) => {
    setSearchParams({ q: query, sort: newSort });
  };

  return (
    <div>
      <h1>Results for "{query}"</h1>
      <button onClick={() => updateParams("date")}>Sort by Date</button>
    </div>
  );
}
```

---

## Part 2: Advanced Routing Patterns

### 2.1 Nested Routes & Layout Routes

**Nested routes** render components inside other components. **Layout routes** wrap multiple routes with shared UI (header, sidebar).

```jsx
// Nested routes
<Routes>
  <Route path="/dashboard" element={<DashboardLayout />}>
    <Route path="overview" element={<Overview />} />
    <Route path="analytics" element={<Analytics />} />
    <Route path="settings" element={<Settings />} />
  </Route>
</Routes>

function DashboardLayout() {
  return (
    <div className="dashboard">
      <Sidebar />
      <div className="content">
        <Outlet /> {/* Renders nested route component */}
      </div>
    </div>
  );
}

// Layout wrapping multiple routes
<Routes>
  <Route element={<PublicLayout />}>
    <Route path="/" element={<Home />} />
    <Route path="/about" element={<About />} />
    <Route path="/login" element={<Login />} />
  </Route>

  <Route element={<ProtectedLayout />}>
    <Route path="/dashboard" element={<Dashboard />} />
    <Route path="/profile" element={<Profile />} />
  </Route>
</Routes>

function PublicLayout() {
  return (
    <>
      <PublicHeader />
      <Outlet />
      <Footer />
    </>
  );
}

function ProtectedLayout() {
  return (
    <>
      <PrivateHeader />
      <Outlet />
    </>
  );
}
```

### 2.2 Protected Routes & Route Guards

**Protected routes** require authentication/authorization before rendering. Check permission, redirect to login if needed.

```jsx
// ProtectedRoute component
function ProtectedRoute({ isAuthenticated, children }) {
  return isAuthenticated ? children : <Navigate to="/login" replace />;
}

<Routes>
  <Route path="/login" element={<Login />} />
  <Route
    path="/dashboard"
    element={
      <ProtectedRoute isAuthenticated={isAuth}>
        <Dashboard />
      </ProtectedRoute>
    }
  />
</Routes>

// Advanced: role-based access control
function RoleProtectedRoute({ requiredRole, userRole, children }) {
  if (!userRole) return <Navigate to="/login" replace />;
  if (!requiredRole.includes(userRole)) return <Navigate to="/unauthorized" replace />;
  return children;
}

<Routes>
  <Route
    path="/admin"
    element={
      <RoleProtectedRoute requiredRole={["admin"]} userRole={currentUser?.role}>
        <AdminPanel />
      </RoleProtectedRoute>
    }
  />
</Routes>

// Route transition guards (warn before leaving)
function useConfirmExit(isDirty) {
  React.useEffect(() => {
    const handleBeforeUnload = (e) => {
      if (isDirty) {
        e.preventDefault();
        e.returnValue = "";
      }
    };
    window.addEventListener("beforeunload", handleBeforeUnload);
    return () => window.removeEventListener("beforeunload", handleBeforeUnload);
  }, [isDirty]);
}

function FormPage() {
  const [isDirty, setIsDirty] = React.useState(false);
  useConfirmExit(isDirty);
  // ...
}
```

### 2.3 Code Splitting with Routes

**Route-based code splitting** loads component chunks on-demand. Each route can have its own chunk, reducing initial bundle.

```jsx
import React from "react";
import { BrowserRouter, Routes, Route } from "react-router-dom";

// Lazy load routes
const Home = React.lazy(() => import("./pages/Home"));
const Dashboard = React.lazy(() => import("./pages/Dashboard"));
const Settings = React.lazy(() => import("./pages/Settings"));

function App() {
  return (
    <BrowserRouter>
      <Suspense fallback={<LoadingSpinner />}>
        <Routes>
          <Route path="/" element={<Home />} />
          <Route path="/dashboard" element={<Dashboard />} />
          <Route path="/settings" element={<Settings />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  );
}

// Prefetch routes on hover/demand
function LinkWithPrefetch({ to, children }) {
  const navigate = useNavigate();

  const handleMouseEnter = async () => {
    // Prefetch component chunk
    import(to);
  };

  return (
    <Link to={to} onMouseEnter={handleMouseEnter}>
      {children}
    </Link>
  );
}
```

---

## Interview Questions

**Q1: Explain client-side vs server-side routing. What are trade-offs?**

Client-side routing renders components without page reloads; enables fast navigation, offline support, smooth transitions. Requires more JavaScript, larger initial bundle, complex browser history management. Server-side routing requires full page reload per navigation but is simpler. SPAs use client-side; traditional sites use server-side.

**Q2: How do you handle dynamic routes with parameters?**

Use `useParams()` to extract URL parameters from path segments (e.g., `/user/:id`). Use `useSearchParams()` for query parameters. Both hooks return values that update when URL changes, triggering re-renders.

**Q3: How do you implement protected routes requiring authentication?**

Create a `ProtectedRoute` component that checks authentication status. If authenticated, render component; else redirect to login. For role-based access, check user role and redirect to unauthorized if insufficient permissions.

**Q4: Why and how do you implement route-based code splitting?**

Route-based splitting reduces initial bundle size; users load only components for routes they visit. Use `React.lazy()` with route components. Wrap routes in `Suspense` boundary to show loading state during chunk load.

---

## Key Takeaways

1. **Client-side routing enables SPAs** - Fast navigation without full page reloads
2. **window.history.pushState manages browser history** - Custom routing possible, but use libraries (React Router)
3. **Dynamic routes extract parameters from URL** - useParams() for segments, useSearchParams() for query params
4. **Nested routes organize complex UIs** - Use Outlet to render child routes within layouts
5. **Protected routes enforce authentication** - Redirect unauthenticated users to login
6. **Role-based access control checks permissions** - Redirect insufficient-permission users away
7. **Route-based code splitting reduces bundle** - Lazy load components per route with React.lazy() + Suspense
8. **useNavigate() for programmatic navigation** - Navigate based on user actions or logic
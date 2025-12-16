# Performance & Optimization: L4 Engineering Guide

## Part 1: Compiler Optimization & Configuration

### 1.1 TypeScript Compiler Performance

**Compilation time** depends on codebase size, complexity, and configuration. Slow compilations hinder developer experience (IDE lag, build delays).

**Optimization strategies:**

- **`skipLibCheck`**: Skip type checking of library files (`.d.ts`); speeds compilation significantly
- **`isolatedModules`**: Enable per-file compilation; each file compiles independently
- **`incremental`**: Store previous compilation results; recompile only changed files
- **`tsBuildInfoFile`**: Specify where to store incremental compilation data

```typescript
// tsconfig.json optimizations
{
  "compilerOptions": {
    "skipLibCheck": true,
    "incremental": true,
    "isolatedModules": true,
    "tsBuildInfoFile": ".tsbuildinfo",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true
  }
}

// Monorepo optimization: use project references
{
  "references": [
    { "path": "./packages/core" },
    { "path": "./packages/ui" },
    { "path": "./packages/api" }
  ]
}

// Build specific project
// tsc -b packages/core
```

### 1.2 Type Checking & Strictness Trade-offs

**Strict mode** enables all type checking flags but increases compilation time. **Progressive strictness** enables checks gradually.

```typescript
// Strict mode enables all checks
{
  "compilerOptions": {
    "strict": true // Equivalent to setting all strict checks
  }
}

// Equivalent to
{
  "compilerOptions": {
    "alwaysStrict": true,
    "noImplicitAny": true,
    "noImplicitThis": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}

// Progressive: start loose, tighten gradually
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": false, // Disable one check if needed
    "skipLibCheck": true // Speed up compilation
  }
}
```

---

## Part 2: Runtime Performance

### 2.1 Type Erasure & Transpilation Impact

**Type erasure** removes all type information during transpilation; types have zero runtime overhead. However, transpilation adds overhead (especially for `async`/`await`, decorators).

**Key principle:** TypeScript's static analysis catches bugs at compile time; types don't exist at runtime.

```typescript
// TypeScript source
interface User {
  id: number;
  name: string;
}

function getUser(id: number): User {
  return { id, name: "John" };
}

// Transpiled JavaScript (types erased)
function getUser(id) {
  return { id, name: "John" };
}

// Decorators transpilation overhead
@Trackable()
class User {
  @Validated
  name: string;
}

// Transpiles to (with overhead)
User = Trackable()(User);
__decorate([Validated], User.prototype, "name");

// Async/await transpilation
async function fetchUser() {
  return await fetch("/api/user");
}

// Transpiles to (with overhead)
function fetchUser() {
  return __awaiter(this, void 0, void 0, function* () {
    return yield fetch("/api/user");
  });
}
```

### 2.2 Runtime Type Checking & Validation

Since types don't exist at runtime, runtime validation requires explicit implementation (libraries like Zod, io-ts).

```typescript
// Without runtime validation (trust types)
interface User {
  id: number;
  name: string;
}

function processUser(user: User) {
  // Assumes user has id and name; no verification
  console.log(user.id, user.name);
}

// Problem: external data (API, JSON) bypasses type checking
const apiData = JSON.parse(jsonString); // type unknown
processUser(apiData); // Runtime error if data shape is wrong

// With runtime validation (schema validation)
import { z } from "zod";

const UserSchema = z.object({
  id: z.number(),
  name: z.string()
});

type User = z.infer<typeof UserSchema>;

function processUser(user: unknown) {
  const validated = UserSchema.parse(user); // Throws if invalid
  console.log(validated.id, validated.name); // Type-safe
}

// Usage
const apiData = JSON.parse(jsonString);
processUser(apiData); // Safe: validated at runtime
```

---

## Part 3: Build & Bundling Optimization

### 3.1 Tree Shaking & Dead Code Elimination

**Tree shaking** removes unused code during bundling, reducing bundle size. Requires ES modules and proper bundler configuration.

```typescript
// tree-shaking-enabled.ts
export function used() {
  return "This is used";
}

export function unused() {
  return "This is not used";
}

// main.ts
import { used } from "./tree-shaking-enabled";
console.log(used());

// Bundled result (unused removed)
// function used() { return "This is used"; }
// console.log(used());

// Side effects prevent tree shaking
// package.json
{
  "sideEffects": ["./src/polyfills.ts"] // These files always included
}

// Webpack config
module.exports = {
  mode: "production",
  optimization: {
    usedExports: true,
    sideEffects: true
  }
};
```

### 3.2 Code Splitting & Lazy Loading

**Code splitting** divides code into chunks loaded on demand, improving initial load time.

```typescript
// Route-based splitting
const routes = [
  { path: "/", component: () => import("./pages/Home") },
  { path: "/about", component: () => import("./pages/About") },
  { path: "/dashboard", component: () => import("./pages/Dashboard") }
];

// Feature-based splitting
const features = {
  analytics: () => import("./features/analytics"),
  reporting: () => import("./features/reporting"),
  premium: () => import("./features/premium")
};

// Conditional splitting (load premium features only for premium users)
async function initializeFeatures(user: User) {
  if (user.isPremium) {
    const { premium } = await import("./features/premium");
    premium.initialize();
  }
}

// Webpack chunk optimization
const config = {
  optimization: {
    splitChunks: {
      chunks: "all",
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: "vendors",
          priority: 10
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true
        }
      }
    }
  }
};
```

---

## Interview Questions

**Q1: How can you speed up TypeScript compilation?**

Use `skipLibCheck` to skip library type checking, enable `incremental` compilation, use `isolatedModules` for per-file compilation, use project references in monorepos, avoid overly complex types. Balance compilation speed with type safety via progressive strictness.

**Q2: Explain type erasure. Does TypeScript have runtime overhead?**

Type erasure removes all type information during transpilation; types don't exist at runtime. TypeScript itself has zero runtime overhead. However, transpilation (especially decorators, async/await) adds overhead. Use `module: "esnext"` to minimize transpilation overhead.

**Q3: How does tree shaking work? What prevents it?**

Tree shaking removes unused code during bundling. Requires ES modules and proper bundler configuration. Prevented by: CommonJS modules (dynamic), side effects, circular dependencies. Mark side-effect-free files in `package.json` ("sideEffects": false) to enable aggressive tree shaking.

**Q4: Explain code splitting and when to use it.**

Code splitting divides code into chunks loaded on demand. Use for: routes (load page code on navigation), features (load premium features if needed), vendor code (separate from app code). Benefits: smaller initial bundle, faster time-to-interactive, efficient caching.

---

## Key Takeaways

1. **`skipLibCheck` significantly speeds compilation** - Skip type checking of library files
2. **`incremental` compilation reuses previous results** - Faster rebuilds for watched development
3. **Types have zero runtime overhead** - Type erasure removes all type information
4. **Transpilation adds overhead** - Decorators, async/await require runtime helpers
5. **Runtime validation required for external data** - Types don't exist at runtime; validate JSON/API responses
6. **Tree shaking removes unused code** - ES modules required; mark side-effects in package.json
7. **Code splitting improves initial load time** - Load features/routes on demand
# Modules & Namespaces: L4 Engineering Guide

## Part 1: Module System Fundamentals

### 1.1 ES Modules & CommonJS

**ES Modules** (`import`/`export`) are the standardized module system. **CommonJS** (`require`/`module.exports`) is Node.js's original system.

**TypeScript supports both:** `module` compiler option specifies target. `esnext` preserves ES modules; `commonjs` transpiles to CommonJS.

**Key differences:**
- ES Modules: Static imports (analyzable before execution); async loading
- CommonJS: Dynamic imports (resolved at runtime); synchronous loading
- ES Modules: Named and default exports; CommonJS: single export object

```typescript
// ES Module
export interface User {
  id: number;
  name: string;
}

export const defaultConfig = { timeout: 5000 };

export default class UserService { }

// Import
import UserService, { User, defaultConfig } from "./user.js";
import * as UserModule from "./user.js"; // Namespace import

// Re-export (aggregation)
export { User, defaultConfig } from "./user.js";
export * from "./auth.js";

// Dynamic import
const module = await import("./user.js");
```

### 1.2 Module Resolution & Path Mapping

**Module resolution** locates modules during import. TypeScript's `moduleResolution` option specifies the algorithm.

**Path mapping** (`paths` in `tsconfig.json`) creates aliases for common import paths, improving readability and maintainability.

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "moduleResolution": "node",
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"],
      "@components/*": ["src/components/*"],
      "@utils/*": ["src/utils/*"],
      "@types/*": ["src/types/*"]
    }
  }
}

// Usage with aliases
import { User } from "@types/user";
import { Button } from "@components/Button";
import { formatDate } from "@utils/date";

// Without aliases (relative paths)
import { User } from "../../../types/user";
import { Button } from "../../components/Button";
import { formatDate } from "../utils/date";
```

---

## Part 2: Namespaces & Ambient Types

### 2.1 Namespaces (Legacy, Rarely Used)

**Namespaces** provide a way to organize code into scopes. ES modules largely supersede namespaces; they exist mainly for compatibility.

```typescript
// Namespace definition
namespace Geometry {
  export interface Point { x: number; y: number; }
  
  export function distance(a: Point, b: Point): number {
    return Math.sqrt((b.x - a.x) ** 2 + (b.y - a.y) ** 2);
  }
}

// Usage
const p1: Geometry.Point = { x: 0, y: 0 };
const p2: Geometry.Point = { x: 3, y: 4 };
Geometry.distance(p1, p2); // 5

// Namespace merging
namespace Geometry {
  export function midpoint(a: Point, b: Point): Point {
    return { x: (a.x + b.x) / 2, y: (a.y + b.y) / 2 };
  }
}

// Modern approach: ES modules instead
// namespace.ts
export interface Point { x: number; y: number; }
export function distance(a: Point, b: Point): number { /* ... */ }

// Import and use like namespaces
import * as Geometry from "./namespace.js";
```

### 2.2 Ambient Declarations & Type Definitions

**Ambient type declarations** (`declare`) define types for external JavaScript libraries without implementation.

**Type definition files** (`.d.ts`) provide type information for JavaScript libraries. DefinitelyTyped (@types/* packages) hosts community-maintained definitions.

```typescript
// Ambient declarations
declare global {
  interface Window {
    customApp: { config: any };
  }
}

declare module "external-lib" {
  export function process(data: unknown): Promise<string>;
}

// Using ambient types
window.customApp.config; // Type-safe access to global

// .d.ts file example (types/custom-lib.d.ts)
declare module "custom-lib" {
  export interface Config {
    apiUrl: string;
    timeout: number;
  }

  export function initialize(config: Config): void;
}

// Using library
import { initialize, Config } from "custom-lib";
const config: Config = { apiUrl: "http://api.example.com", timeout: 5000 };
initialize(config);
```

---

## Part 3: Advanced Module Patterns

### 3.1 Module Augmentation & Declaration Merging

**Module augmentation** extends library types without modifying source. **Declaration merging** combines multiple declarations of the same name.

```typescript
// Augment library types
declare module "express" {
  interface Request {
    user?: { id: number; name: string };
  }
}

// In application code
import { Request } from "express";
function handler(req: Request) {
  console.log(req.user?.name); // user property exists (augmented)
}

// Interface declaration merging
interface User {
  id: number;
  name: string;
}

interface User {
  email: string; // Merged
}

// Result: User has id, name, email

// Namespace merging
namespace Config {
  export const api = "http://api.example.com";
}

namespace Config {
  export const timeout = 5000; // Merged
}

// Result: Config.api and Config.timeout both exist
```

### 3.2 Lazy Loading & Code Splitting

**Lazy loading** defers module imports until needed. **Code splitting** divides code into chunks loaded on demand.

```typescript
// Lazy import using dynamic import
async function loadDashboard() {
  const { Dashboard } = await import("./Dashboard.js");
  return new Dashboard();
}

// Conditional lazy loading
async function loadModule(name: string) {
  if (name === "admin") {
    const { AdminPanel } = await import("./AdminPanel.js");
    return AdminPanel;
  } else {
    const { UserPanel } = await import("./UserPanel.js");
    return UserPanel;
  }
}

// Router-based code splitting (React Router example)
const routes = [
  {
    path: "/",
    component: () => import("./Home.js").then(m => m.default)
  },
  {
    path: "/dashboard",
    component: () => import("./Dashboard.js").then(m => m.default)
  }
];

// Webpack chunk naming
const feature = () => import(
  /* webpackChunkName: "feature-chunk" */
  "./Feature.js"
);
```

---

## Interview Questions

**Q1: Explain the difference between ES Modules and CommonJS.**

ES Modules use `import`/`export`; CommonJS uses `require`/`module.exports`. ES Modules are static (analyzable before execution); CommonJS is dynamic. ES Modules are async; CommonJS is sync. TypeScript transpiles ES modules to CommonJS for Node.js compatibility.

**Q2: What are path aliases and why are they useful?**

Path aliases (via `paths` in `tsconfig.json`) map long import paths to shorter aliases: `@components/*` → `src/components/*`. Benefits: readable imports, easier refactoring (change one path in config), reduce relative path complexity. Essential for large projects.

**Q3: Explain module augmentation. When would you use it?**

Module augmentation extends library types without modifying source via `declare module`. Used when a library doesn't provide type definitions or needs application-specific extensions (e.g., adding user property to Express Request). Enables type-safe access to augmented properties.

**Q4: How do you implement lazy loading in TypeScript? What are the benefits?**

Use dynamic `import()`: `const module = await import("./path.js")`. Benefits: smaller initial bundle, faster startup, load features on demand. Critical for large applications; reduces time-to-interactive.

---

## Key Takeaways

1. **ES Modules are the standard** - Static imports, async loading, standardized syntax
2. **CommonJS still used in Node.js** - TypeScript transpiles ES modules to CommonJS
3. **Path aliases improve readability** - Avoid deep relative path hierarchies
4. **Namespaces are legacy** - Use ES modules instead; more standard and composable
5. **Module augmentation extends library types** - Without modifying source libraries
6. **Dynamic imports enable lazy loading** - Load code on demand for smaller bundles
7. **Declaration merging combines multiple declarations** - Interfaces, namespaces automatically merge
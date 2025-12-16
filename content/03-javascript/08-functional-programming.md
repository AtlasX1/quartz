# Functional Programming in JavaScript: L4 Engineering Guide

## Part 1: Core FP Concepts

### 1.1 Pure Functions & Referential Transparency

A **pure function** produces the same output for the same inputs with no side effects. It doesn't modify external state, perform I/O, or depend on global variables.

**Referential transparency** means a function call can be replaced with its result without changing behavior. Pure functions enable this property.

**Advantages:**
- Easier testing: behavior is deterministic and isolated
- Easier reasoning: fewer hidden dependencies
- Optimization opportunities: caching/memoization, parallelization
- Easier composition: combining pure functions remains pure

**Trade-off:** Pure functions often require immutability and functional style, which can be less intuitive than imperative mutations.

```javascript
// Impure: modifies global state
let count = 0;
function incrementCount() {
  count++; // Side effect
  return count;
}

// Pure: no side effects
function increment(n) {
  return n + 1; // Referentially transparent
}

// Impure: depends on external input
function greet(name = globalUser.name) {
  return `Hello, ${name}`;
}

// Pure: all inputs explicit
function greetPure(name) {
  return `Hello, ${name}`;
}

// Impure: performs I/O
function fetchUser(id) {
  return fetch(`/api/users/${id}`).then(r => r.json()); // I/O side effect
}

// Pure: accepts dependency as parameter
function fetchUserPure(fetch, id) {
  return fetch(`/api/users/${id}`).then(r => r.json());
}
```

### 1.2 Immutability & Data Structures

**Immutable data** is never modified; operations create new values. This eliminates bugs from unexpected mutations and enables structural sharing (efficient memory usage).

**Benefits:**
- Eliminates mutation bugs: shared data can't cause unexpected changes
- Enables concurrency: no lock needed for immutable data
- Enables undo/redo: previous versions are preserved
- Simplifies debugging: state changes are explicit

```javascript
// Mutable arrays (imperative)
const arr = [1, 2, 3];
arr[0] = 10; // Modified original

// Immutable arrays (functional)
const arr2 = [1, 2, 3];
const arr3 = [10, ...arr2.slice(1)]; // New array created
// or: arr2.map((x, i) => i === 0 ? 10 : x)

// Mutable objects
const obj = { a: 1, b: 2 };
obj.a = 10; // Modified original

// Immutable objects
const obj2 = { a: 1, b: 2 };
const obj3 = { ...obj2, a: 10 }; // New object created

// Immutable library (Immer.js or similar)
import produce from 'immer';
const state = { users: [{ id: 1, name: 'John' }] };
const newState = produce(state, draft => {
  draft.users[0].name = 'Jane'; // Syntax of mutation, but immutable result
});
```

---

## Part 2: Higher-Order Functions & Composition

### 2.1 Higher-Order Functions

A **higher-order function** accepts functions as arguments or returns functions. This pattern enables abstraction, reusability, and composition.

**Common patterns:**
- **Decorators**: Enhance function behavior (logging, caching, error handling)
- **Currying**: Transform function into nested functions with one argument each
- **Partial application**: Fix some arguments, return function with remaining arguments

```javascript
// Decorator: adding logging to functions
function withLogging(fn) {
  return function(...args) {
    console.log(`Calling ${fn.name} with`, args);
    const result = fn(...args);
    console.log(`Result:`, result);
    return result;
  };
}

const add = (a, b) => a + b;
const addLogged = withLogging(add);
addLogged(5, 3); // Logs calls and result

// Currying: transform f(a, b) to f(a)(b)
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn(...args);
    } else {
      return (...nextArgs) => curried(...args, ...nextArgs);
    }
  };
}

const addCurried = curry((a, b, c) => a + b + c);
addCurried(1)(2)(3); // 6
addCurried(1, 2)(3); // 6

// Partial application: fix some arguments
function partial(fn, ...partialArgs) {
  return (...args) => fn(...partialArgs, ...args);
}

const add5 = partial(add, 5);
add5(3); // 8
```

### 2.2 Function Composition

**Composition** combines functions to create new functions. `compose(f, g)(x) = f(g(x))` (right-to-left, mathematical notation). `pipe(f, g)(x) = g(f(x))` (left-to-right, practical notation).

```javascript
// Compose and pipe utilities
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);
const pipe = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);

// Pure functions to compose
const toUpperCase = (str) => str.toUpperCase();
const addPrefix = (str) => `PREFIX_${str}`;
const reverse = (str) => str.split('').reverse().join('');

// Composition (right-to-left)
const composed = compose(reverse, addPrefix, toUpperCase);
composed('hello'); // 'OLLEH_XIFXERP' (uppercase → prefix → reverse)

// Piping (left-to-right)
const piped = pipe(toUpperCase, addPrefix, reverse);
piped('hello'); // Same result, more readable order

// Real-world example
const getUser = (id) => ({ id, name: 'John', email: 'john@example.com' });
const extractEmail = (user) => user.email;
const lowerEmail = (email) => email.toLowerCase();
const addDomain = (email) => `${email} (verified)`;

const processEmail = pipe(getUser, extractEmail, lowerEmail, addDomain);
processEmail(1); // 'john@example.com (verified)'
```

---

## Part 3: Declarative Style & Functional Patterns

### 3.1 Map, Filter, Reduce

**Declarative programming** describes what to compute, not how. Functional method chains are declarative.

```javascript
const users = [
  { id: 1, name: 'Alice', age: 30, active: true },
  { id: 2, name: 'Bob', age: 25, active: false },
  { id: 3, name: 'Charlie', age: 35, active: true }
];

// Imperative (how to do it)
const result = [];
for (const user of users) {
  if (user.active && user.age > 26) {
    result.push(user.name.toUpperCase());
  }
}

// Declarative (what to compute)
const result2 = users
  .filter(user => user.active && user.age > 26)
  .map(user => user.name.toUpperCase());

// Reduce for aggregation
const totalAge = users.reduce((sum, user) => sum + user.age, 0);
const agesByStatus = users.reduce((acc, user) => {
  const status = user.active ? 'active' : 'inactive';
  acc[status] = (acc[status] || 0) + 1;
  return acc;
}, {});
```

### 3.2 Avoiding Side Effects

Side effects (I/O, mutations, global state changes) should be isolated and minimized. Separate pure business logic from side-effect-heavy infrastructure code.

```javascript
// Bad: mixed logic and side effects
function getUserAndLog(id) {
  const user = fetchSync(`/api/users/${id}`); // I/O
  console.log('User fetched:', user); // Side effect
  user.lastAccessed = Date.now(); // Mutation
  saveSync(user); // I/O
  return user;
}

// Good: separate concerns
// Pure function
function enrichUser(user) {
  return { ...user, lastAccessed: Date.now() };
}

// Side-effect-heavy function (infrastructure)
async function getUserAndLog(id) {
  try {
    const user = await fetch(`/api/users/${id}`).then(r => r.json()); // I/O
    const enriched = enrichUser(user); // Pure
    console.log('User fetched:', enriched); // Side effect
    await fetch(`/api/users/${id}`, { 
      method: 'PUT', 
      body: JSON.stringify(enriched) 
    }); // I/O
    return enriched;
  } catch (error) {
    console.error('Failed to fetch user:', error);
    throw error;
  }
}
```

---

## Interview Questions

**Q1: What are pure functions? Why are they important?**

Pure functions produce the same output for the same inputs with no side effects. They're important because they're easier to test, reason about, and optimize. They enable referential transparency (replaceable with their results).

**Q2: Explain function composition. How does it differ from method chaining?**

Composition combines functions to create new functions: `compose(f, g)(x) = f(g(x))`. Method chaining calls methods sequentially on objects. Composition is more flexible for reuse and testing; chaining is more idiomatic for certain patterns.

**Q3: What's the difference between currying and partial application?**

Currying transforms `f(a, b, c)` into `f(a)(b)(c)`, always one argument per call. Partial application fixes some arguments and returns a function with remaining arguments. Currying is a specific pattern; partial application is more general.

**Q4: Compare imperative and declarative styles. When should you use each?**

Imperative describes how to compute results (loops, mutations). Declarative describes what to compute (`map`, `filter`). Declarative is more readable and compositional; imperative is sometimes more efficient. Use declarative for business logic; imperative for low-level optimizations.

---

## Key Takeaways

1. **Pure functions eliminate bugs** - Same input → same output; no side effects
2. **Immutability enables composition** - Operations create new values, preserving originals
3. **Higher-order functions enable abstraction** - Decorators, currying, partial application
4. **Function composition creates complex operations** - Combine simple pure functions
5. **Declarative style improves readability** - `map`, `filter`, `reduce` chains
6. **Side effects should be isolated** - Separate pure logic from I/O operations
7. **Functional patterns enhance testability** - Pure functions don't require mocks
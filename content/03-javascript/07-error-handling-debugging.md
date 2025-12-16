# Error Handling & Debugging: L4 Engineering Guide

## Part 1: Error Types & Handling

### 1.1 Error Hierarchy

JavaScript provides built-in error types inheriting from `Error`:

- **Error**: Base class for all errors
- **ReferenceError**: Accessing undefined variable
- **TypeError**: Type mismatch (calling non-function, accessing property of null/undefined)
- **RangeError**: Invalid numeric range (array too large, recursion depth exceeded)
- **SyntaxError**: Invalid syntax (compile-time, usually can't be caught)
- **AggregateError**: Multiple errors (returned by `Promise.any()`)

Each error has `message`, `name`, and `stack` properties. The `stack` property shows the call trace.

```javascript
// Error type examples
try {
  JSON.parse('invalid');
} catch (error) {
  console.log(error instanceof SyntaxError); // true in JSON parsing
}

try {
  const obj = null;
  obj.property; // Cannot read properties of null
} catch (error) {
  console.log(error instanceof TypeError); // true
}

try {
  Array(Math.pow(2, 32)); // Array too large
} catch (error) {
  console.log(error instanceof RangeError); // true
}
```

### 1.2 Try/Catch/Finally

**`try` block** contains code that might throw errors. **`catch` block** handles exceptions. **`finally` block** executes regardless of success/exception.

**Control flow:** If an error occurs, control jumps to `catch`, skipping remaining `try` code. `finally` always executes after `try`/`catch`, even if `catch` throws.

```javascript
function parseConfig(jsonString) {
  let config;
  try {
    config = JSON.parse(jsonString);
    if (!config.required) throw new Error('Missing required field');
    return config;
  } catch (error) {
    if (error instanceof SyntaxError) {
      console.error('Invalid JSON:', error.message);
    } else {
      console.error('Config error:', error.message);
    }
    return null; // Fallback
  } finally {
    console.log('Config parsing completed');
  }
}

// Rethrowing
try {
  riskyOperation();
} catch (error) {
  if (error instanceof TypeError) {
    // Handle TypeError
    console.error('Type error handled');
  } else {
    throw error; // Rethrow for outer handler
  }
}
```

### 1.3 Custom Errors

Extending `Error` creates domain-specific error types, improving error handling clarity.

```javascript
class ValidationError extends Error {
  constructor(message, field) {
    super(message);
    this.name = 'ValidationError';
    this.field = field;
  }
}

class NetworkError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.name = 'NetworkError';
    this.statusCode = statusCode;
  }
}

function validateEmail(email) {
  if (!email.includes('@')) {
    throw new ValidationError('Invalid email format', 'email');
  }
}

async function fetchUser(userId) {
  const response = await fetch(`/api/users/${userId}`);
  if (!response.ok) {
    throw new NetworkError('Failed to fetch user', response.status);
  }
  return response.json();
}

// Specific error handling
try {
  validateEmail('invalid');
} catch (error) {
  if (error instanceof ValidationError) {
    console.error(`${error.field} validation failed: ${error.message}`);
  } else {
    throw error;
  }
}
```

---

## Part 2: Debugging Techniques

### 2.1 Chrome DevTools

**Breakpoints** pause execution at specific lines:
- **Line breakpoint:** Pause at specific line
- **Conditional breakpoint:** Pause if condition is true
- **DOM breakpoint:** Pause when DOM element changes
- **Event listener breakpoint:** Pause on event

**Call stack** shows the function execution sequence. Inspecting variables in the stack reveals state at each level.

**Watch expressions** monitor variable changes without pausing.

```javascript
// Example: debugging a function with breakpoints
function calculateTotal(items) {
  let total = 0; // Set breakpoint here
  for (const item of items) {
    total += item.price; // Step through this loop
  }
  return total;
}

// In DevTools: Set breakpoint → F10 (step over) or F11 (step into)
// Watch `total` to see accumulation
calculateTotal([{ price: 10 }, { price: 20 }]);
```

### 2.2 Logging & Debugging Tools

**`console` methods** provide debugging output:
- `console.log()`: Basic logging
- `console.table()`: Formatted table output
- `console.time()` / `console.timeEnd()`: Performance timing
- `console.trace()`: Stack trace
- `console.assert()`: Assertion checking
- `console.group()` / `console.groupEnd()`: Grouped output

```javascript
// Advanced logging
const users = [
  { id: 1, name: 'Alice', age: 30 },
  { id: 2, name: 'Bob', age: 25 }
];

console.table(users); // Formatted table in DevTools

console.time('operation');
slowOperation();
console.timeEnd('operation'); // Logs elapsed time

console.assert(value > 0, 'Value must be positive');

console.group('User operations');
console.log('Fetching users...');
console.log('Users fetched');
console.groupEnd();
```

### 2.3 Profiling & Memory Analysis

**Performance Profiler** records CPU usage over time, identifying slow functions. **Memory Profiler** detects memory leaks by tracking heap size.

**Source Maps** map minified code back to original source, enabling debugging of transpiled/bundled code.

```javascript
// Performance.now() for manual profiling
const start = performance.now();
expensiveOperation();
const end = performance.now();
console.log(`Operation took ${end - start}ms`);

// Memory profiling (DevTools Memory tab)
// Take heap snapshot before/after operation
// Compare snapshots to identify unreleased objects
```

---

## Part 3: Error Handling Patterns

### 3.1 Defensive Programming

Validate inputs and handle edge cases explicitly rather than letting errors occur.

```javascript
// Unsafe: assumes input is valid
function add(a, b) {
  return a + b;
}

// Safe: validates and handles edge cases
function safeAdd(a, b) {
  if (typeof a !== 'number' || typeof b !== 'number') {
    throw new TypeError('Both arguments must be numbers');
  }
  if (!isFinite(a) || !isFinite(b)) {
    throw new RangeError('Arguments must be finite numbers');
  }
  return a + b;
}

safeAdd(5, 10); // 15
safeAdd('5', 10); // TypeError
safeAdd(Infinity, 10); // RangeError
```

### 3.2 Promise Error Handling

Unhandled promise rejections cause "Uncaught (in promise)" errors. Always attach `.catch()` or use `try...catch` in `async` functions.

```javascript
// Bad: unhandled rejection
Promise.reject(new Error('Failed'));
// Uncaught (in promise) Error: Failed

// Good: explicit error handling
Promise.reject(new Error('Failed')).catch(error => {
  console.error('Handled:', error);
});

// Good: async/await with try...catch
async function robustOperation() {
  try {
    await riskyOperation();
  } catch (error) {
    console.error('Operation failed:', error);
  }
}

// Global unhandled rejection handler
window.addEventListener('unhandledrejection', event => {
  console.error('Unhandled promise rejection:', event.reason);
  event.preventDefault(); // Prevent crash
});
```

---

## Interview Questions

**Q1: Explain the difference between Error types in JavaScript. How would you handle them differently?**

Error types indicate the error source: `TypeError` for type mismatches, `ReferenceError` for undefined variables, `SyntaxError` for parse errors. Handle each type specifically using `instanceof` checks; custom errors enable domain-specific handling.

**Q2: What happens in finally if catch throws an error?**

`finally` always executes after `catch`, even if `catch` throws. The new error from `catch` takes precedence over the original error. `finally` is useful for guaranteed cleanup.

**Q3: How do you debug asynchronous code? What tools help?**

Use async breakpoints in DevTools, set breakpoints in Promise `.then()` callbacks, use `console.time()` for timing, and inspect the call stack (which shows async contexts). Source maps are essential for debugging transpiled code.

**Q4: Explain memory leaks and how to detect them.**

Memory leaks occur when objects remain referenced but become unreachable. Causes: event listeners not removed, closures retaining large objects, circular references. Detect with Chrome DevTools Memory Profiler: take heap snapshots before/after operations and compare.

---

## Key Takeaways

1. **Error types indicate error source** - Use `instanceof` to distinguish and handle specifically
2. **`finally` guarantees cleanup** - Executes even if `try`/`catch` throws
3. **Custom errors improve clarity** - Create domain-specific error types
4. **DevTools breakpoints enable step-by-step debugging** - Inspect variables and call stack
5. **`console` methods aid logging** - `console.table()`, `console.time()`, `console.trace()`
6. **Unhandled promise rejections are silent crashes** - Always add `.catch()` or `try...catch`
7. **Source maps enable debugging of transpiled code** - Essential for production debugging
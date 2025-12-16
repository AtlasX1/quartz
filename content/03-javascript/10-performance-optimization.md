# Performance & Optimization: L4 Engineering Guide

## Part 1: JavaScript Engine Optimization

### 1.1 V8 Engine & Hidden Classes

Modern JavaScript engines (V8, SpiderMonkey) use **just-in-time (JIT) compilation** to convert frequently-executed JavaScript to machine code. **Hidden classes** track object property layouts; objects with identical structure share hidden classes for optimization.

**Optimization impact:**
- Consistent property order improves performance ($2-10x$ speedup for hot code)
- Adding/removing properties creates new hidden classes (deoptimization)
- Polymorphic call sites (different object types) prevent optimization

```javascript
// Optimized: consistent property order
function createUser(name, email) {
  return { name, email }; // Consistent order
}

const u1 = createUser('John', 'john@example.com');
const u2 = createUser('Jane', 'jane@example.com');
// u1 and u2 share hidden class → fast property access

// Deoptimized: inconsistent property addition
const obj1 = { a: 1 };
obj1.b = 2; // New hidden class created

const obj2 = { a: 1, b: 2 }; // Different construction path
// obj1 and obj2 don't share hidden class → slower access

// Deoptimized: polymorphic access
function processObject(obj) {
  return obj.value; // Optimized for one type
}

processObject({ value: 10 }); // Hidden class 1
processObject({ value: 20, type: 'number' }); // Hidden class 2
// Polymorphic site → deoptimized, falls back to slower lookup
```

### 1.2 Avoiding Deoptimization

**Performance cliffs** occur when optimized code is deoptimized:

- **Type instability**: Functions receiving different types
- **Hidden class pollution**: Adding properties dynamically
- **Megamorphic call sites**: Same method called with many different object types
- **Eval & Dynamic code**: Dynamic code generation prevents static analysis

```javascript
// Bad: type instability
function add(a, b) {
  return a + b;
}

add(5, 3); // Number
add('5', '3'); // String (deoptimizes for polymorphic dispatch)

// Good: consistent types
function addNumbers(a, b) {
  return a + b;
}

function concatenate(a, b) {
  return String(a) + String(b);
}

// Bad: dynamic property addition
function build() {
  const obj = {};
  for (let i = 0; i < 100; i++) {
    obj[`prop${i}`] = i; // Creates new hidden class each iteration
  }
  return obj;
}

// Good: declare properties upfront
function buildOptimized() {
  const obj = {};
  for (let i = 0; i < 100; i++) {
    obj.value = i; // Single hidden class
  }
  return obj;
}
```

---

## Part 2: Performance Optimization Techniques

### 2.1 Debouncing & Throttling

**Debouncing** delays execution until no events occur for a specified duration. Useful for search, resize, input.

**Throttling** limits execution frequency to once per interval. Useful for scroll, mousemove.

```javascript
// Debounce (delays until quiet)
function debounce(fn, delay) {
  let timeout;
  return function(...args) {
    clearTimeout(timeout);
    timeout = setTimeout(() => fn(...args), delay);
  };
}

const search = debounce(async (query) => {
  const results = await api.search(query);
  updateUI(results);
}, 300);

// User types: 'j', 'jo', 'joh', 'john'
// API called once (300ms after last keystroke)

// Throttle (limits frequency)
function throttle(fn, interval) {
  let lastCall = 0;
  return function(...args) {
    const now = Date.now();
    if (now - lastCall >= interval) {
      lastCall = now;
      fn(...args);
    }
  };
}

window.addEventListener('scroll', throttle(() => {
  console.log('Scrolling');
}, 100)); // Logs max every 100ms
```

### 2.2 Memoization & Caching

**Memoization** caches function results based on arguments. Effective for expensive pure functions.

```javascript
// Simple memoization
function memoize(fn) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}

const fibonacci = memoize((n) => {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
});

fibonacci(40); // ~1ms (memoized)
fibonacci(40); // ~0.001ms (cached)

// Memoization with max cache size
function memoizeWithLimit(fn, limit = 100) {
  const cache = new Map();
  return function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }
    if (cache.size >= limit) {
      cache.delete(cache.keys().next().value); // Remove oldest
    }
    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
```

### 2.3 Lazy Loading & Code Splitting

**Lazy loading** defers loading code until needed. **Code splitting** divides code into chunks loaded on demand.

```javascript
// Dynamic imports (lazy loading)
async function loadDataProcessor() {
  const { processData } = await import('./data-processor.js');
  processData();
}

// Conditional import
if (user.isPremium) {
  const { premiumFeatures } = await import('./premium.js');
}

// Router-based code splitting
const routes = [
  { path: '/', component: () => import('./Home.js') },
  { path: '/dashboard', component: () => import('./Dashboard.js') }
];

// Webpack: splitting with comments
const component = () => import(
  /* webpackChunkName: "feature-chunk" */
  './FeatureModule.js'
);
```

---

## Part 3: Memory & Profiling

### 3.1 Memory Management

**Memory lifecycle:** Allocate → Use → Release. JavaScript manages deallocation via garbage collection, but memory leaks occur when objects remain referenced.

**Common memory leak patterns:**

- **Event listeners not removed**: DOM nodes with listeners aren't garbage collected
- **Timers not cleared**: `setInterval` with closures retains memory
- **Circular references in closures**: Objects reference each other indefinitely
- **Global variables**: Accidentally created globals persist forever

```javascript
// Memory leak: event listener
const button = document.getElementById('btn');
const handler = () => { /* heavy work */ };
button.addEventListener('click', handler);
// When button is removed from DOM, if listener isn't removed, memory leaks

// Fix: remove listener
button.removeEventListener('click', handler);

// Memory leak: timer
let cache = new Array(1000000);
const id = setInterval(() => {
  console.log(cache.length); // Closure retains cache
}, 1000);

// Fix: clear timer and nullify
clearInterval(id);
cache = null;

// Memory leak: circular closure
function createCycle() {
  const obj = {};
  obj.self = obj; // Circular reference
  return obj;
}

// Modern GC handles simple cycles, but complex cases may leak
```

### 3.2 Profiling & Performance Monitoring

**`performance.now()`** provides high-resolution timing for measuring code execution.

**Chrome DevTools Performance tab** records execution timeline, showing rendering, JavaScript, and other metrics.

**`requestIdleCallback()`** schedules work when browser is idle, avoiding blocking.

```javascript
// Manual profiling
const start = performance.now();
expensiveOperation();
const end = performance.now();
console.log(`Execution time: ${end - start}ms`);

// PerformanceObserver for monitoring
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`${entry.name}: ${entry.duration}ms`);
  }
});

observer.observe({ entryTypes: ['measure', 'navigation'] });

// Mark and measure
performance.mark('operation-start');
expensiveOperation();
performance.mark('operation-end');
performance.measure('operation', 'operation-start', 'operation-end');

// requestIdleCallback for non-critical work
requestIdleCallback(() => {
  console.log('Running during idle time');
});
```

---

## Interview Questions

**Q1: Explain hidden classes in V8. Why do they matter?**

Hidden classes track object property layouts. Objects with identical structure share hidden classes, enabling property access optimization. Inconsistent property order or dynamic addition creates new hidden classes, causing deoptimization ($2-10x$ slowdown).

**Q2: What's the difference between debouncing and throttling?**

Debouncing delays execution until events stop occurring; throttling limits execution frequency. Debounce: one execution after quiet period. Throttle: max execution once per interval. Use debounce for autocomplete; throttle for scroll events.

**Q3: Explain memoization. What are its limitations?**

Memoization caches function results based on arguments, avoiding recomputation. Limitations: cache invalidation (stale data), memory overhead, only works for pure functions, hash collision handling (JSON.stringify is slow).

**Q4: How do you identify memory leaks?**

Use Chrome DevTools Memory Profiler: take heap snapshots before/after operations and compare. Look for retained objects that should be garbage collected. Common causes: event listeners not removed, timers not cleared, circular closures.

---

## Key Takeaways

1. **Hidden classes optimize property access** - Consistent object structure enables 2-10x speedups
2. **Type stability prevents deoptimization** - Polymorphic call sites cause performance cliffs
3. **Debounce delays; throttle limits frequency** - Choose based on event pattern
4. **Memoization caches pure function results** - Effective for expensive computations
5. **Lazy loading reduces initial bundle size** - Dynamic imports enable code splitting
6. **Memory leaks occur from retained references** - Remove listeners, clear timers, nullify closures
7. **`performance.now()` enables precise measurement** - Use for profiling hot code paths
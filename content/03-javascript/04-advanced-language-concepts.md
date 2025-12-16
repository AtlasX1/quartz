# Advanced Language Concepts: L4 Engineering Guide

## Part 1: Scope & Execution Context

### 1.1 Lexical vs Dynamic Scope

**Lexical scope** (static scope) determines variable accessibility based on syntactic code structure. A function can access variables from its enclosing scope, not its call site. JavaScript uses lexical scope exclusively.

**Dynamic scope** (hypothetically) would resolve variables based on the call stack at runtime. This is rare in modern languages and complicates reasoning about code.

```javascript
// Lexical scope example
let global = 'global';

function outer() {
  let outerVar = 'outer';
  
  function inner() {
    console.log(global, outerVar); // Accesses outer's scope
  }
  
  return inner;
}

const fn = outer();
let outerVar = 'shadow'; // Different variable
fn(); // Logs 'global outer' (NOT 'global shadow')
// fn uses lexical scope from outer, not call site
```

### 1.2 Execution Context Stack

JavaScript maintains an **execution context stack** (call stack). When a function is called, a new execution context is pushed; when it returns, the context is popped.

**Execution context contains:**
- Variable environment: `var` declarations and function declarations
- Lexical environment: `let`/`const` declarations
- `this` binding

**Stack trace** shows the call sequence when errors occur. Exceeding maximum stack depth causes **stack overflow** (infinite recursion).

```javascript
function a() {
  console.log('In a');
  b();
}

function b() {
  console.log('In b');
  c();
}

function c() {
  console.log('In c');
}

a();
// Stack: [Global] → [a] → [b] → [c] → [b] → [a] → [Global]
```

### 1.3 Hoisting & Temporal Dead Zone (TDZ)

**Hoisting** is the JavaScript parsing behavior of moving declarations to the top of their scope. This happens during compilation, before execution.

**For `var`:** Hoisted and initialized to `undefined`—accessing before declaration returns `undefined`, not an error.

**For `let`/`const`:** Hoisted but not initialized. Accessing before declaration throws **ReferenceError** because the variable is in the **Temporal Dead Zone (TDZ)**.

**For `function`:** Fully hoisted (declaration + body)—callable before declaration.

```javascript
console.log(x); // undefined (var hoisted, initialized to undefined)
console.log(y); // ReferenceError (let in TDZ)
console.log(fn()); // Works (function fully hoisted)

var x = 5;
let y = 10;
function fn() { return 'called'; }

// TDZ visualization
{
  // y is in TDZ here
  console.log(y); // ReferenceError
  let y = 10; // Declaration exits TDZ
}
```

---

## Part 2: Memory Model & Garbage Collection

### 2.1 Stack vs Heap

**Stack** stores primitive values and references to objects. Stack memory is automatically freed when functions return. Stack access is fast ($O(1)$) but limited in size.

**Heap** stores objects, arrays, and other reference types. Heap allocation is slower than stack, and freeing requires garbage collection.

**Memory model:**
- Function parameters and local primitives → Stack
- Local object references → Stack; the objects themselves → Heap
- Global variables → Global scope (heap-like behavior)

```javascript
function example() {
  const x = 5; // Primitive → Stack
  const obj = { value: 10 }; // Reference → Stack; object → Heap
  const arr = [1, 2, 3]; // Reference → Stack; array → Heap
}
// On return: Stack frame freed; Heap objects eligible for GC if unreferenced

// Memory leak (closure retains object)
function createLeak() {
  const largeObject = new Array(1000000);
  return function() {
    return largeObject.length; // Closure retains largeObject
  };
}
```

### 2.2 Garbage Collection & Memory Leaks

**Garbage collection (GC)** automatically frees unreferenced objects. Modern engines use **mark-and-sweep**: objects reachable from global scope or active functions are "marked"; unmarked objects are freed.

**Memory leaks** occur when objects remain referenced but become unreachable through normal code flow. Common causes:
- **Circular references in closures**: Functions retain large objects indefinitely
- **Event listeners not removed**: DOM nodes with attached listeners aren't garbage collected
- **Timers not cleared**: `setInterval`/`setTimeout` with closures
- **Global variable accumulation**: Accidental globals

```javascript
// Memory leak: event listener
const button = document.getElementById('btn');
button.addEventListener('click', function() {
  // If 'this' or closure holds large data, button removal doesn't free memory
});
// Solution: remove listener or use arrow function with manual cleanup

// Memory leak: timer
let bigData = new Array(1000000);
setInterval(() => {
  console.log(bigData.length);
}, 1000);
// bigData is retained indefinitely; even if we want to free it, we can't

// Solution: clear interval and nullify reference
const id = setInterval(() => { }, 1000);
clearInterval(id);
bigData = null;
```

---

## Part 3: Event Loop & Concurrency Model

### 3.1 Synchronous Execution & Call Stack

JavaScript executes synchronous code using a **single call stack**—one operation at a time. When a function calls another, the called function's context is pushed; when it returns, the context is popped.

Blocking operations (intensive loops, synchronous I/O) freeze the UI because the call stack can't process other events.

```javascript
function fibonacci(n) {
  if (n <= 1) return n;
  return fibonacci(n - 1) + fibonacci(n - 2);
}

// This blocks the UI for several seconds
const result = fibonacci(40);
console.log(result);

// Solution: use Web Workers or async decomposition
```

### 3.2 Event Loop & Task Queues

JavaScript's **event loop** allows asynchronous operations by coordinating the call stack, callback queues, and event system. Modern JavaScript distinguishes two queue types:

**Macrotask queue:** `setTimeout`, `setInterval`, `setImmediate` (Node.js), I/O operations
**Microtask queue:** Promise callbacks (`.then()`, `.catch()`), `queueMicrotask()`, mutation observers

**Event loop process:**
1. Execute all synchronous code (call stack)
2. Execute all microtasks (Promise callbacks)
3. Render (if needed)
4. Execute one macrotask (e.g., `setTimeout` callback)
5. Repeat from step 2

**Implication:** Microtasks execute before macrotasks; even if a Promise was created after `setTimeout`, the Promise's `.then()` executes first.

```javascript
console.log('1'); // Synchronous

setTimeout(() => console.log('2'), 0); // Macrotask

Promise.resolve().then(() => console.log('3')); // Microtask

console.log('4'); // Synchronous

// Output: 1, 4, 3, 2
// 1,4 (synchronous) → 3 (microtask) → 2 (macrotask)
```

---

## Part 4: Modules

### 4.1 CommonJS vs ES Modules

**CommonJS** (`require`/`module.exports`):
- Synchronous module loading; imports happen at runtime
- Used in Node.js
- `module.exports = value` exports; `const value = require('path')` imports

**ES Modules** (`import`/`export`):
- Asynchronous module loading; standardized JavaScript modules
- Supported in modern browsers and Node.js 12+
- Static imports allow tree-shaking (dead code elimination)

```javascript
// CommonJS
module.exports = { add: (a, b) => a + b };
const { add } = require('./math.js');

// ES Modules
export const add = (a, b) => a + b;
import { add } from './math.js';

// Dynamic imports (both)
const module = await import('./math.js');
```

### 4.2 Module Resolution & Tree-Shaking

**Module resolution** locates modules during import. ES Modules resolve relative paths (`./file.js`) and npm packages (`import React from 'react'`).

**Tree-shaking** (dead code elimination) is possible with ES Modules because imports are static. Bundlers analyze which exports are used and remove unused code.

```javascript
// math.js
export const add = (a, b) => a + b;
export const subtract = (a, b) => a - b;
export const multiply = (a, b) => a * b;

// main.js
import { add } from './math.js'; // Only add is imported

// Bundler analysis: subtract and multiply are unused → removed from bundle
// Result: smaller bundle size
```

---

## Interview Questions

**Q1: Explain the Temporal Dead Zone (TDZ) and why it exists.**

TDZ is the period between entering a scope and reaching a `let`/`const` declaration. Accessing variables in TDZ throws ReferenceError. TDZ forces explicit initialization order, preventing subtle bugs from hoisting.

**Q2: What's the difference between the call stack and event loop?**

The call stack executes synchronous code; the event loop coordinates asynchronous operations and UI updates. The event loop checks microtasks after each synchronous function returns, then processes one macrotask, then repeats.

**Q3: How do closures cause memory leaks?**

Closures retain references to outer scope variables indefinitely. If a closure is never garbage collected (event listeners not removed, timers not cleared), the retained variables are never freed, causing memory leaks.

**Q4: What's the advantage of ES Modules over CommonJS?**

ES Modules are static (imports are analyzable before execution), enabling tree-shaking and better bundling. CommonJS is dynamic (resolved at runtime), preventing static analysis. ES Modules are the standardized approach.

---

## Key Takeaways

1. **Lexical scope is determined statically** - Functions access their enclosing scope, not call site
2. **Hoisting moves declarations but not initializations** - TDZ prevents accessing `let`/`const` before declaration
3. **Stack is automatic; Heap requires garbage collection** - Unreferenced heap objects are freed automatically
4. **Event loop coordinates async operations** - Microtasks execute before macrotasks
5. **Closures retain memory** - Remove event listeners and clear timers to prevent leaks
6. **ES Modules enable tree-shaking** - Static imports allow bundlers to eliminate dead code
7. **Call stack can overflow** - Infinite recursion causes stack overflow errors
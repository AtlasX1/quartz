# Asynchronous JavaScript: L4 Engineering Guide

## Part 1: Callbacks & Promises

### 1.1 Callback Pattern & Inversion of Control

**Callbacks** are functions passed as arguments and executed later. They're fundamental to asynchronous JavaScript but create **inversion of control**: you pass your function to external code, losing control over its execution.

**Callback issues:**
- **Callback hell:** Deeply nested callbacks reduce readability
- **Error handling scattered:** Each callback needs its own error handling
- **Difficult composition:** Combining multiple async operations is cumbersome
- **Inversion of control:** External code controls when/how your callback executes

```javascript
// Callback example
function fetchUser(userId, callback) {
  setTimeout(() => {
    if (userId > 0) {
      callback(null, { id: userId, name: 'John' });
    } else {
      callback(new Error('Invalid user ID'));
    }
  }, 100);
}

// Callback hell (pyramid of doom)
fetchUser(1, (err, user) => {
  if (err) console.error(err);
  else {
    fetchUser(user.id, (err, friend) => {
      if (err) console.error(err);
      else {
        console.log(friend.name);
      }
    });
  }
});
```

### 1.2 Promise Lifecycle & States

A **Promise** represents an eventual result (or error) of an asynchronous operation. It has three states:

- **Pending:** Initial state; operation hasn't completed
- **Fulfilled:** Operation succeeded; `.then()` handlers execute
- **Rejected:** Operation failed; `.catch()` handlers execute

A promise transitions from Pending to Fulfilled/Rejected exactly once; state changes are irreversible.

```javascript
// Promise creation
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    if (Math.random() > 0.5) {
      resolve('Success'); // Transition to Fulfilled
    } else {
      reject(new Error('Failure')); // Transition to Rejected
    }
  }, 100);
});

// Handlers (chaining)
promise
  .then(result => console.log(result)) // Executed if fulfilled
  .catch(error => console.error(error)) // Executed if rejected
  .finally(() => console.log('Done')); // Always executed

// Promise chaining returns new promises
promise
  .then(result => result.toUpperCase()) // Returns new promise
  .then(upper => console.log(upper))
  .catch(error => console.error(error));
```

### 1.3 Promise Utility Methods

**`Promise.all()`** combines multiple promises; returns rejected if any promise rejects.

**`Promise.race()`** returns the first settled promise (fulfilled or rejected).

**`Promise.any()`** returns the first fulfilled promise; rejects with `AggregateError` if all reject.

**`Promise.allSettled()`** waits for all promises to settle (fulfill or reject); always returns results array.

```javascript
const p1 = Promise.resolve(1);
const p2 = new Promise(resolve => setTimeout(() => resolve(2), 100));
const p3 = Promise.reject('Error');

// Promise.all (all or nothing)
Promise.all([p1, p2]).then(values => console.log(values)); // [1, 2]
Promise.all([p1, p3]).catch(err => console.error(err)); // 'Error'

// Promise.race (first to settle)
Promise.race([p1, p2]).then(v => console.log(v)); // 1

// Promise.allSettled (all results)
Promise.allSettled([p1, p2, p3]).then(results => {
  console.log(results);
  // [
  //   { status: 'fulfilled', value: 1 },
  //   { status: 'fulfilled', value: 2 },
  //   { status: 'rejected', reason: 'Error' }
  // ]
});
```

---

## Part 2: Async/Await Syntax

### 2.1 Async Functions

An **async function** always returns a Promise. Inside an async function, `await` pauses execution until a Promise settles.

**Semantics:**
- `await` only works inside `async` functions (or top-level in modules)
- `await` unwraps Promise values: `const value = await promise` (not `const promise = await promise`)
- Errors thrown or rejected promises throw exceptions catchable by `try...catch`

```javascript
async function fetchData(url) {
  try {
    const response = await fetch(url);
    const data = await response.json();
    return data; // Implicitly wrapped in Promise
  } catch (error) {
    console.error('Fetch failed:', error);
    throw error; // Reject the returned Promise
  }
}

// Calling async function
fetchData('/api/users')
  .then(data => console.log(data))
  .catch(error => console.error(error));

// Or in another async function
async function displayUsers() {
  const users = await fetchData('/api/users');
  console.log(users);
}
```

### 2.2 Sequential vs Parallel Execution

**Sequential execution** awaits each operation before starting the next. Useful when operations depend on each other but slower when independent.

**Parallel execution** starts all operations simultaneously and waits for completion. Faster for independent operations.

```javascript
// Sequential (slow)
async function sequential() {
  const user = await fetchUser(1); // Waits 100ms
  const posts = await fetchPosts(user.id); // Waits 100ms
  // Total: ~200ms
  return { user, posts };
}

// Parallel (fast)
async function parallel() {
  const [user, posts] = await Promise.all([
    fetchUser(1),
    fetchPosts(1)
  ]); // Both run simultaneously; total ~100ms
  return { user, posts };
}

// Partially parallel
async function mixed() {
  const user = await fetchUser(1); // Wait for user first
  const [posts, friends] = await Promise.all([
    fetchPosts(user.id),
    fetchFriends(user.id)
  ]); // Then fetch posts and friends in parallel
  return { user, posts, friends };
}
```

### 2.3 Error Handling with Async/Await

Errors in `async` functions are caught by `try...catch`. Rejected promises thrown inside `async` functions propagate as exceptions.

```javascript
async function robustFetch(url) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }
    return await response.json();
  } catch (error) {
    if (error instanceof TypeError) {
      console.error('Network error:', error);
    } else {
      console.error('Request failed:', error);
    }
    return null; // Return default value
  } finally {
    console.log('Request completed');
  }
}
```

---

## Part 3: Event Loop & Concurrency

### 3.1 Microtasks vs Macrotasks

**Microtasks** (Promise callbacks, `queueMicrotask`, mutation observers) execute after all synchronous code and before macrotasks. Microtasks have higher priority.

**Macrotasks** (timers, I/O, rendering) execute one at a time, with microtasks running between each macrotask.

**Event loop phases:**
1. Execute call stack (synchronous code)
2. Execute all microtasks
3. Render (if needed)
4. Execute one macrotask
5. Go to step 2

```javascript
console.log('1'); // Synchronous

setTimeout(() => console.log('2'), 0); // Macrotask

Promise.resolve()
  .then(() => console.log('3'))
  .then(() => console.log('4')); // Microtasks

console.log('5'); // Synchronous

// Output: 1, 5, 3, 4, 2
// Sync (1, 5) → Microtasks (3, 4) → Macrotask (2)
```

### 3.2 Timer Behavior & Scheduling

**`setTimeout(callback, delay)`** schedules a callback after at least `delay` milliseconds, but the actual execution depends on the event loop being idle.

**`setImmediate(callback)`** (Node.js only) schedules callback in the check phase of the event loop, after I/O operations.

**`requestAnimationFrame(callback)`** schedules callback before the next repaint, synchronizing with the browser's rendering.

```javascript
setTimeout(() => console.log('setTimeout'), 0);
Promise.resolve().then(() => console.log('Promise'));
queueMicrotask(() => console.log('queueMicrotask'));

// Output: queueMicrotask, Promise, setTimeout
// All microtasks execute before macrotasks, despite setTimeout(0)
```

---

## Interview Questions

**Q1: What's the difference between `.then()/.catch()` and `async/await`?**

Both handle promises, but `async/await` is syntactic sugar for promise chains. `async/await` is more readable and enables `try...catch` error handling, making sequential async operations clearer. Promises are more flexible for complex composition.

**Q2: Explain the event loop. Why does `setTimeout(callback, 0)` not execute immediately?**

The event loop executes synchronous code, then microtasks (Promises), then one macrotask (`setTimeout`). `setTimeout(0)` is a macrotask; even with 0ms delay, it executes after all microtasks, not immediately.

**Q3: When should you use `Promise.all()` vs `Promise.allSettled()`?**

`Promise.all()` is all-or-nothing: if any promise rejects, the whole operation fails. Use it when all operations must succeed. `Promise.allSettled()` waits for all to settle (success or failure) and returns all results. Use it when you need all results regardless of failures.

**Q4: What's the performance difference between sequential and parallel async operations?**

Sequential: $T = T_1 + T_2 + ... + T_n$ (total time is sum of individual times). Parallel: $T = \max(T_1, T_2, ..., T_n)$ (total time is longest operation). Parallel is faster when operations are independent.

---

## Key Takeaways

1. **Promises manage async operations with guaranteed state** - Fulfill, reject, or stay pending; irreversible
2. **`async/await` simplifies promise handling** - More readable than `.then()` chains
3. **Microtasks execute before macrotasks** - Promise callbacks run before `setTimeout`
4. **Sequential vs parallel execution has different performance** - Use `Promise.all()` for independent operations
5. **Error handling with `try...catch` is cleaner in `async` functions** - Catch promise rejections as exceptions
6. **Inversion of control is a callback pitfall** - Promises and `async/await` restore control flow
7. **Timer delays are minimum, not guaranteed** - Actual execution depends on event loop availability
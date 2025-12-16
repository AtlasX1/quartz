# Control Flow & Functions: L4 Engineering Guide

## Part 1: Control Flow Statements

### 1.1 Conditional Logic

**`if/else` statements** evaluate boolean expressions to branch execution. JavaScript coerces condition expressions to booleans using truthy/falsy rules: `false`, `0`, `-0`, `0n`, `''`, `null`, `undefined`, `NaN` are falsy; all other values are truthy.

**Proper conditional structure:**

```javascript
// Good: explicit boolean
if (age >= 18) { /* adult logic */ }

// Avoid: implicit coercion
if (age) { /* coercion: 0 is falsy */ }

// Ternary operator for simple conditions
const status = age >= 18 ? 'adult' : 'minor';

// Nested ternaries reduce readability—use if/else instead
```

**`switch` statements** compare a value against multiple cases using strict equality. Execution falls through to the next case unless `break` is used. Default case catches non-matching cases.

**Efficiency note:** `switch` statements with many cases perform linear search; for large case sets, object maps offer $O(1)$ lookup.

```javascript
switch (day) {
  case 1: console.log('Monday'); break;
  case 2: console.log('Tuesday'); break;
  default: console.log('Other');
}

// Equivalent object map (more scalable)
const days = { 1: 'Monday', 2: 'Tuesday' };
console.log(days[day] || 'Other');
```

### 1.2 Loop Mechanisms

**`for` loop:** Traditional loop with initialization, condition, and increment. Optimal for indexed iteration.

**`while`/`do...while` loops:** Useful when iteration count is unknown or condition-based.

**`for...in` loop:** Iterates over enumerable properties (including inherited ones). Unreliable for arrays since order isn't guaranteed and may include non-numeric properties.

**`for...of` loop:** Iterates over iterable values (arrays, strings, Sets, Maps). Cleaner than `for` loop; respects array indices and order.

```javascript
const arr = [10, 20, 30];

// for loop (traditional)
for (let i = 0; i < arr.length; i++) console.log(arr[i]);

// for...of (recommended for arrays)
for (const value of arr) console.log(value);

// for...in (avoid for arrays, use for objects)
for (const key in { a: 1, b: 2 }) console.log(key);

// while (condition-based)
let count = 0;
while (count < 3) { console.log(count++); }
```

### 1.3 Loop Control & Short-Circuit Evaluation

**`break`** exits the loop entirely; **`continue`** skips to the next iteration.

**Short-circuit operators** (`&&`, `||`, `??`) evaluate conditionally:
- `condition && sideEffect()` executes `sideEffect()` only if condition is truthy
- `condition || defaultValue` returns `defaultValue` if condition is falsy
- `value ?? defaultValue` returns `defaultValue` only if value is `null`/`undefined`

```javascript
// Short-circuit example
function logIfTrue(flag) {
  flag && console.log('Logged');
  flag || console.log('Flag is falsy');
}

// Nullish coalescing
const config = userInput ?? { default: true };
```

---

## Part 2: Functions & Scope

### 2.1 Function Scope & Closures

A **closure** is a function that retains access to variables from its enclosing scope, even after the outer function returns. Closures are created automatically; every function is a closure.

**Closure mechanism:** When a function is created, it captures a reference to its lexical environment (outer scope variables). This reference persists even after the outer function completes.

**Practical implications:**

- Closures enable data encapsulation (private variables)
- Closures create memory retention risks if not managed carefully
- Every callback, event listener, and higher-order function result uses closures

```javascript
function counter() {
  let count = 0; // Captured by closure
  return function() {
    return ++count;
  };
}

const c = counter();
c(); // 1
c(); // 2
// count variable persists in closure even after counter() returns

// Data encapsulation pattern
function createBankAccount(initialBalance) {
  let balance = initialBalance; // Private variable
  
  return {
    deposit: (amount) => { balance += amount; return balance; },
    withdraw: (amount) => { balance -= amount; return balance; },
    getBalance: () => balance
  };
}

const account = createBankAccount(1000);
account.deposit(500); // 1500
// balance is inaccessible directly; only accessible through methods
```

### 2.2 Higher-Order Functions

A **higher-order function** accepts functions as arguments or returns functions. This pattern enables composition, abstraction, and functional programming.

**Common higher-order functions:**

- **`Array.map()`**: Transforms each element, returns new array
- **`Array.filter()`**: Retains elements matching predicate, returns new array
- **`Array.reduce()`**: Accumulates a single value from array elements
- **`Array.forEach()`**: Executes side effects on each element

**Performance characteristics:** `map` and `filter` create intermediate arrays; for multiple transformations, `reduce` is more efficient. `forEach` is slower than `for` loop but idiomatic for functional style.

```javascript
const users = [
  { name: 'Alice', age: 30 },
  { name: 'Bob', age: 25 },
  { name: 'Charlie', age: 35 }
];

// Chaining higher-order functions
users
  .filter(user => user.age > 26)
  .map(user => user.name)
  .forEach(name => console.log(name));

// Equivalent reduce (more efficient, single pass)
const names = users
  .filter(user => user.age > 26)
  .reduce((acc, user) => [...acc, user.name], []);
```

### 2.3 Pure vs Impure Functions

A **pure function** produces the same output for the same inputs and has no side effects. It doesn't modify external state, make API calls, or log.

An **impure function** has side effects: modifies global state, reads from external sources, or performs I/O.

**Advantages of pure functions:**
- Testability: Behavior is deterministic and isolated
- Reusability: No hidden dependencies
- Optimization: Compiler/engine can cache results (memoization)

**Trade-off:** Pure functions often require functional style (immutability, avoiding mutations), which can be less intuitive than imperative style.

```javascript
// Pure function
function add(a, b) {
  return a + b; // Deterministic, no side effects
}

// Impure function
let total = 0;
function addToTotal(amount) {
  total += amount; // Modifies external state
  console.log(total); // Side effect
}

// Pure array transformation
function addElement(arr, element) {
  return [...arr, element]; // New array, original untouched
}

// Impure array transformation
function addElementImpure(arr, element) {
  arr.push(element); // Mutates original
  return arr;
}
```

---

## Part 3: Exception Handling

### 3.1 Try/Catch/Finally

**`try` block** contains code that might throw errors. If an error occurs, control jumps to `catch`.

**`catch` block** handles the error. The error object contains `message`, `name`, and `stack` properties.

**`finally` block** executes regardless of success/exception; used for cleanup (closing files, releasing resources).

```javascript
function parseJSON(jsonString) {
  try {
    return JSON.parse(jsonString);
  } catch (error) {
    console.error('Parse failed:', error.message);
    return null;
  } finally {
    console.log('Parsing attempt completed');
  }
}

// Rethrowing errors
try {
  riskyOperation();
} catch (error) {
  if (error instanceof TypeError) {
    // Handle TypeError specifically
    console.error('Type error:', error);
  } else {
    throw error; // Rethrow for outer handler
  }
}
```

### 3.2 Error Types & Custom Errors

JavaScript provides built-in error types:
- **`Error`**: Base error class
- **`TypeError`**: Type mismatch (calling non-function, accessing undefined property)
- **`ReferenceError`**: Undefined variable access
- **`RangeError`**: Invalid numeric range
- **`SyntaxError`**: Invalid syntax (compile-time)
- **`AggregateError`**: Multiple errors (used by `Promise.any()`)

**Custom errors** extend Error for domain-specific handling:

```javascript
class ValidationError extends Error {
  constructor(message) {
    super(message);
    this.name = 'ValidationError';
  }
}

function validateEmail(email) {
  if (!email.includes('@')) {
    throw new ValidationError('Invalid email format');
  }
}

try {
  validateEmail('invalid');
} catch (error) {
  if (error instanceof ValidationError) {
    console.error('Validation failed:', error.message);
  }
}
```

---

## Interview Questions

**Q1: Explain closures. Why are they useful and what risks do they pose?**

Closures are functions retaining access to outer scope variables after the outer function returns. Useful for data encapsulation and callbacks. Risk: memory leaks if closures retain large objects unnecessarily.

**Q2: What's the difference between `map`, `filter`, and `reduce`? When should you use each?**

`map` transforms elements (one-to-one), `filter` selects elements (one-to-zero-or-one), `reduce` accumulates to a single value. Use `map` for transformation, `filter` for selection, `reduce` for aggregation or multi-pass efficiency.

**Q3: Compare pure and impure functions. When is it appropriate to use impure functions?**

Pure functions are deterministic and testable; impure functions have side effects. Pure functions are preferred for business logic; impure functions are necessary for I/O (API calls, logging, DOM manipulation) and state management.

**Q4: Explain try/catch/finally. What happens in finally if an error occurs?**

`try` contains risky code, `catch` handles errors, `finally` executes regardless. `finally` runs even if `catch` throws, allowing guaranteed cleanup logic.

---

## Key Takeaways

1. **Closures retain scope access** - Every function is a closure; leverage for encapsulation
2. **Higher-order functions enable composition** - `map`, `filter`, `reduce` provide declarative data transformation
3. **Pure functions are easier to test** - Prefer pure functions for business logic
4. **Short-circuit evaluation avoids unnecessary computation** - `&&`, `||`, `??` conditionally evaluate right operand
5. **`for...of` is preferred for arrays** - Cleaner than `for...in` or traditional `for` loops
6. **`finally` guarantees cleanup** - Useful for resource management
7. **Custom errors improve error handling** - Extend Error for domain-specific exception types
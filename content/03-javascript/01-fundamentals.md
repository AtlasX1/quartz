# JavaScript Fundamentals: L4 Engineering Guide

## Part 1: ECMAScript Standards & Syntax Evolution

### 1.1 JavaScript Execution Model

JavaScript operates as a **dynamically-typed, interpreted language** with just-in-time (JIT) compilation in modern engines (V8, SpiderMonkey, JavaScriptCore). The language standardization through ECMAScript specifications (ES5, ES6/ES2015 onwards) ensures consistency across environments.

**Key execution characteristics:**

- **Dynamic typing**: Variables don't declare types; types are determined at runtime. This enables flexibility but requires careful property access (accessing undefined properties returns undefined, not errors).
- **Lexical scoping**: Inner scopes can access outer scopes, but outer scopes cannot access inner scopes. This fundamental principle determines variable resolution.
- **Automatic semicolon insertion (ASI)**: JavaScript adds missing semicolons, but relying on ASI creates ambiguous code. Explicit semicolons are standard practice.

ECMAScript 2015 (ES6) marked a major shift introducing `let`/`const` block scoping, arrow functions, destructuring, and modules—moving from function-scoped (`var`) to block-scoped variables.

### 1.2 Variable Declaration & Hoisting

Three declaration mechanisms exist: `var`, `let`, and `const`.

**`var` (function-scoped):** Hoisted to function/global scope with `undefined` initialization. Reassignable and redeclarable within the same scope. Largely superseded by `let`/`const`.

**`let` (block-scoped):** Hoisted but enters Temporal Dead Zone (TDZ)—accessing before declaration throws ReferenceError. Reassignable but not redeclarable in the same scope.

**`const` (block-scoped):** Also subject to TDZ. Not reassignable after initialization, but the reference is immutable—objects/arrays are mutable if const-declared. Modern practice favors `const` by default; use `let` only when reassignment is necessary.

**Hoisting mechanics:** Variable and function declarations are processed during compilation phase before execution, moving declarations to the top of their scope. Functions are hoisted entirely (declaration + body); variables are hoisted with partial initialization (`var` → `undefined`, `let`/`const` → TDZ).

### 1.3 Arrow Functions vs Traditional Functions

Arrow functions (`=>`) differ from traditional functions in two critical ways:

1. **Lexical `this` binding**: Arrow functions inherit `this` from enclosing scope, not from call context. Traditional functions have dynamic `this`.
2. **No `arguments` object**: Arrow functions don't have `arguments`; use rest parameters instead.
3. **Cannot be constructors**: `new ArrowFunction()` throws TypeError.

**Practical implications:**

- Arrow functions are unsuitable for object methods or callbacks where `this` context matters
- Arrow functions are ideal for higher-order functions, callbacks, and functional programming patterns

**Example:**

```javascript
// Traditional function: dynamic this
const obj = {
  name: 'obj',
  method: function() { return this.name; }
};
obj.method(); // 'obj'
const fn = obj.method;
fn(); // undefined (this is global/undefined in strict mode)

// Arrow function: lexical this
const obj2 = {
  name: 'obj2',
  method: () => this.name // this is inherited from outer scope
};
// In global scope, arrow this refers to global object

// Constructor pattern
function Traditional() { this.value = 42; }
new Traditional(); // Works

const Arrow = () => { this.value = 42; };
new Arrow(); // TypeError
```

---

## Part 2: Data Types & Type Coercion

### 2.1 Primitive vs Reference Types

**Primitives** (stored by value):
- `string`, `number`, `bigint`, `boolean`, `undefined`, `symbol`, `null`
- Immutable; operations create new values
- Compared by value: `5 === 5` is true

**Reference types** (stored by reference):
- `Object`, `Array`, `Function`, `Date`, `RegExp`, `Map`, `Set`, `WeakMap`, `WeakSet`
- Mutable; operations modify existing values
- Compared by reference: `[1] === [1]` is false (different array objects)

**Example:**

```javascript
let a = 5;
let b = a;
b = 10;
console.log(a); // 5 (primitives don't share values)

let obj1 = { x: 5 };
let obj2 = obj1;
obj2.x = 10;
console.log(obj1.x); // 10 (objects share reference)

// Type coercion
'5' == 5; // true (loose equality, coercion occurs)
'5' === 5; // false (strict equality, no coercion)
null == undefined; // true (coercion rule)
null === undefined; // false
```

### 2.2 Type Coercion & Implicit Conversion

JavaScript automatically converts types in operations:

**Loose equality (`==`)** triggers coercion: `0 == false` is true; `'' == 0` is true. The spec defines complex coercion rules (ToPrimitive, ToNumber, ToString) but the behavior is unpredictable.

**Strict equality (`===`)** compares both value and type—recommended practice. Requires explicit conversion when types differ.

**Nullish coalescing (`??`)** returns right operand if left is `null` or `undefined` (but not `false`, `0`, `''`). Differs from `||` which treats all falsy values the same.

**Optional chaining (`?.`)** safely accesses nested properties; returns `undefined` if the chain is broken rather than throwing an error.

**Example:**

```javascript
// Coercion pitfalls
console.log(1 + '1'); // '11' (number coerced to string)
console.log('1' - 1); // 0 (string coerced to number)
console.log(true + 1); // 2 (boolean coerced to 1)

// Nullish coalescing
const x = null ?? 'default'; // 'default'
const y = 0 ?? 'default'; // 0 (not treated as nullish)

// Optional chaining
const obj = { a: { b: 5 } };
console.log(obj?.a?.b); // 5
console.log(obj?.x?.y); // undefined (not TypeError)
```

---

## Part 3: Operators & Expressions

### 3.1 Operator Categories

**Arithmetic operators** (`+`, `-`, `*`, `/`, `%`, `**`) perform mathematical operations. The `+` operator is overloaded: it performs addition for numbers but concatenation for strings.

**Comparison operators** (`<`, `>`, `<=`, `>=`) return booleans. String comparison uses lexicographic ordering (Unicode values).

**Logical operators** (`&&`, `||`, `!`) implement short-circuit evaluation:
- `&&` returns first falsy value or last value if all truthy
- `||` returns first truthy value or last value if all falsy

**Bitwise operators** (`&`, `|`, `^`, `~`, `<<`, `>>`, `>>>`) operate on 32-bit signed/unsigned integers. Rarely used in modern code but critical for performance-sensitive operations (game engines, graphics).

### 3.2 Spread & Rest Operators

The spread operator (`...`) expands iterables (arrays, strings) into individual elements. The rest operator uses the same syntax but collects arguments into an array.

**Spread in arrays:**
```javascript
const arr = [1, 2, 3];
const expanded = [0, ...arr, 4]; // [0, 1, 2, 3, 4]

// Spread in objects
const obj = { a: 1, b: 2 };
const merged = { ...obj, c: 3 }; // { a: 1, b: 2, c: 3 }
```

**Rest in parameters:**
```javascript
function sum(...numbers) {
  return numbers.reduce((a, b) => a + b, 0);
}
sum(1, 2, 3, 4); // 10

// Destructuring rest
const [first, ...rest] = [1, 2, 3, 4];
first; // 1
rest; // [2, 3, 4]
```

### 3.3 Destructuring

Destructuring extracts values from arrays/objects into distinct variables, improving readability and enabling concise pattern matching.

**Example:**

```javascript
// Array destructuring
const [a, b, , d] = [1, 2, 3, 4];
a; // 1, d; // 4

// Object destructuring
const { name, age } = { name: 'John', age: 30, email: 'john@example.com' };
name; // 'John'

// Nested destructuring
const { user: { name, profile: { age } } } = {
  user: { name: 'John', profile: { age: 30 } }
};

// Default values
const { status = 'active' } = {}; // 'active'
```

---

## Part 4: Function Declarations & Expressions

### 4.1 Declaration vs Expression

**Function declaration:** Hoisted entirely (name + body), executable before declaration.

```javascript
sayHi(); // Works—function hoisted
function sayHi() { console.log('Hi'); }
```

**Function expression:** Not hoisted; must be declared before use.

```javascript
greet(); // TypeError: greet is not a function
const greet = function() { console.log('Hello'); };
```

**Named function expressions** aid debugging (stack traces show the function name).

### 4.2 Default Parameters & Arguments

Modern ES6 syntax supports default parameters, evaluated left-to-right during execution:

```javascript
function greet(name = 'Guest', greeting = `Hello, ${name}`) {
  console.log(greeting);
}
greet(); // Hello, Guest
greet('Alice'); // Hello, Alice
greet('Bob', 'Hi'); // Hi
```

---

## Part 5: Scope & Execution Context

### 5.1 Lexical Scope & Variable Resolution

JavaScript uses **lexical (static) scoping**: variable resolution follows the syntactic structure of the code, not runtime call stack. Inner functions can access outer variables; outer functions cannot access inner variables.

**Scope chain:** When resolving a variable, JavaScript searches the local scope, then outer scopes sequentially until the variable is found or the global scope is reached. This determines accessibility.

**Example:**

```javascript
let global = 'global';

function outer() {
  let outerVar = 'outer';
  
  function inner() {
    let innerVar = 'inner';
    console.log(innerVar, outerVar, global); // All accessible
  }
  
  inner();
  console.log(innerVar); // ReferenceError—inner scope not accessible
}

outer();
```

### 5.2 Execution Context & `this`

**Execution context** (distinct from scope) is an abstract concept representing the environment where code executes. It contains:
- **Variable environment**: Local variables declared with `var`, function declarations
- **Lexical environment**: Local variables declared with `let`/`const`, function expressions
- **`this` binding**: Depends on how the function is called

**`this` binding rules (in order of precedence):**

1. **Arrow function:** Uses lexical `this` from enclosing scope
2. **`new` operator:** `this` refers to newly created object
3. **Explicit binding:** `call()`, `apply()`, `bind()` override `this`
4. **Method invocation:** `obj.method()` → `this` is `obj`
5. **Function call:** `function()` → `this` is global object (or `undefined` in strict mode)

**Example:**

```javascript
const obj = {
  name: 'obj',
  method: function() { return this.name; },
  arrow: () => this.name // this from outer scope
};

obj.method(); // 'obj'
const fn = obj.method;
fn(); // undefined (this is global/undefined in strict)

// Explicit binding
fn.call(obj); // 'obj'
fn.apply(obj); // 'obj'
const bound = fn.bind(obj);
bound(); // 'obj'
```

---

## Interview Questions

**Q1: Explain hoisting and Temporal Dead Zone (TDZ).**

Hoisting moves variable and function declarations to the top of their scope during compilation. `var` hoists with initialization to `undefined`; `let`/`const` hoist without initialization, entering TDZ. Accessing `let`/`const` in TDZ throws ReferenceError. `function` declarations fully hoist.

**Q2: What's the difference between `==` and `===`? When should you use each?**

`==` performs type coercion before comparison; `===` compares both value and type without coercion. Always use `===` for predictable behavior and to avoid coercion pitfalls. `==` should be avoided except for specific comparisons like `null == undefined`.

**Q3: How do arrow functions differ from traditional functions? When should you avoid them?**

Arrow functions have lexical `this` (inherited from outer scope), no `arguments` object, and cannot be constructors. Avoid arrow functions as object methods or constructors where dynamic `this` is needed.

**Q4: Explain destructuring and its practical advantages.**

Destructuring extracts values from arrays/objects into variables. Advantages: reduced variable assignment boilerplate, clearer intent, supports default values, enables pattern matching. Example: `const { name, age } = user;` vs `const name = user.name; const age = user.age;`

---

## Key Takeaways

1. **Primitives are immutable; references are mutable** - Understanding value vs reference semantics is fundamental
2. **`let`/`const` are block-scoped; `var` is function-scoped** - Prefer `const` by default for immutability
3. **Hoisting moves declarations but not initializations** - `var` hoists with `undefined`; `let`/`const` enter TDZ
4. **Arrow functions have lexical `this`; traditional functions have dynamic `this`** - Choose based on context
5. **Type coercion creates unpredictable behavior** - Use strict equality (`===`) to avoid coercion
6. **Spread/rest operators unify syntax** - Spread expands iterables; rest collects arguments
7. **Execution context determines `this` binding** - Five binding rules govern `this` resolution
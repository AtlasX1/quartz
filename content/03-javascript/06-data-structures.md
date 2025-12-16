# Data Structures in JavaScript: L4 Engineering Guide

## Part 1: Collections & Iterables

### 1.1 Arrays & Array Methods

**Arrays** store indexed values (0-based). They're objects with numeric keys and a `length` property. Arrays provide methods for transformation, filtering, and searching.

**Transformation methods** create new arrays: `map`, `filter`, `slice`, `concat`, `flat`, `flatMap`.

**Mutation methods** modify the original array: `push`, `pop`, `shift`, `unshift`, `splice`, `sort`, `reverse`.

**Iteration methods** execute callbacks: `forEach`, `map`, `filter`, `reduce`, `find`, `findIndex`, `some`, `every`.

**Performance characteristics:**
- Indexed access: $O(1)$
- Linear search: $O(n)$
- `sort`: $O(n \log n)$ (typically Timsort in modern engines)

```javascript
const arr = [1, 2, 3, 4, 5];

// Transformation (creates new arrays)
const doubled = arr.map(x => x * 2); // [2, 4, 6, 8, 10]
const evens = arr.filter(x => x % 2 === 0); // [2, 4]
const sum = arr.reduce((acc, x) => acc + x, 0); // 15

// Search
const index = arr.indexOf(3); // 2
const found = arr.find(x => x > 3); // 4

// Mutation (modifies original)
arr.push(6); // [1, 2, 3, 4, 5, 6]
arr.sort((a, b) => b - a); // [6, 5, 4, 3, 2, 1]

// Method chaining
const result = [1, 2, 3, 4, 5]
  .filter(x => x % 2 === 0)
  .map(x => x * 2)
  .reduce((acc, x) => acc + x, 0); // 12
```

### 1.2 Sets & Maps

**Set** stores unique values (no duplicates). Operations:
- `add`, `delete`, `has`: $O(1)$ average
- `size`: $O(1)$
- No indexing or ordering guarantee (though insertion order is preserved in practice)

**Map** stores key-value pairs with any type as key (not just strings like objects). Operations:
- `get`, `set`, `delete`, `has`: $O(1)$ average
- `size`: $O(1)$
- Iteration order is insertion order

**WeakSet & WeakMap** store weak references (objects eligible for garbage collection even if referenced). Useful for metadata associated with objects without preventing garbage collection.

```javascript
// Set: unique values
const set = new Set([1, 2, 2, 3, 3, 3]);
console.log(set.size); // 3
set.add(4);
console.log(set.has(2)); // true
set.forEach(value => console.log(value)); // 1, 2, 3, 4

// Map: key-value pairs with any key type
const map = new Map();
map.set('name', 'John');
map.set({ id: 1 }, 'value');
map.set(123, 'number key');
console.log(map.get('name')); // 'John'
console.log(map.size); // 3

// WeakMap: metadata association
const userData = new WeakMap();
let user = { id: 1 };
userData.set(user, 'metadata');
// When user is garbage collected, the WeakMap entry is removed
```

### 1.3 Iterables & Iterators

An **iterable** is an object implementing `Symbol.iterator` method, which returns an **iterator**. An **iterator** is an object with `next()` method returning `{value, done}`.

**For...of loops** work with iterables: Arrays, Strings, Sets, Maps, Generators.

```javascript
// Manual iterator implementation
const iterable = {
  data: [1, 2, 3],
  [Symbol.iterator]() {
    let index = 0;
    return {
      next: () => {
        if (index < this.data.length) {
          return { value: this.data[index++], done: false };
        } else {
          return { value: undefined, done: true };
        }
      }
    };
  }
};

// For...of uses the iterator
for (const value of iterable) {
  console.log(value); // 1, 2, 3
}

// Spread operator also uses iterables
const arr = [...iterable]; // [1, 2, 3]
```

---

## Part 2: Typed Arrays & Buffers

### 2.1 ArrayBuffer & DataView

**ArrayBuffer** is raw binary data; cannot be accessed directly. **DataView** provides typed access to ArrayBuffer contents, supporting different data types (Int8, Uint8, Int16, Float32, etc.).

**Use cases:** File processing, WebGL graphics, network protocols, audio processing.

```javascript
// Create 16-byte buffer
const buffer = new ArrayBuffer(16);

// Access with DataView
const view = new DataView(buffer);

// Write values
view.setInt32(0, 42); // Write 32-bit integer at byte 0
view.setFloat32(4, 3.14); // Write 32-bit float at byte 4

// Read values
console.log(view.getInt32(0)); // 42
console.log(view.getFloat32(4)); // 3.14...
```

### 2.2 Typed Array Views

**Typed arrays** (Int8Array, Uint8Array, Int16Array, Float32Array, etc.) provide fixed-type array views over ArrayBuffer. They're faster than DataView for homogeneous data.

```javascript
const buffer = new ArrayBuffer(12);

// Create typed array views
const int32View = new Int32Array(buffer);
const uint8View = new Uint8Array(buffer);

int32View[0] = 256; // Write value
console.log(uint8View[0]); // 0 (least significant byte)
console.log(uint8View[1]); // 1 (next byte)

// Operations
const arr = new Uint8Array([1, 2, 3]);
arr.fill(0); // [0, 0, 0]
arr.set([10, 20], 1); // [0, 10, 20]
```

---

## Part 3: Advanced Iteration

### 3.1 Generators

A **generator function** (marked with `*`) yields values one at a time, pausing and resuming execution.

**Use cases:** Lazy evaluation, creating iterables, state machines, coroutines.

```javascript
function* numberGenerator() {
  yield 1;
  yield 2;
  yield 3;
}

const gen = numberGenerator();
console.log(gen.next()); // { value: 1, done: false }
console.log(gen.next()); // { value: 2, done: false }
console.log(gen.next()); // { value: 3, done: false }
console.log(gen.next()); // { value: undefined, done: true }

// For...of works with generators
for (const value of numberGenerator()) {
  console.log(value); // 1, 2, 3
}

// Infinite generator
function* infinite() {
  let i = 0;
  while (true) yield i++;
}

const iter = infinite();
console.log(iter.next().value); // 0
console.log(iter.next().value); // 1
```

### 3.2 Async Generators & For-Await

**Async generators** combine generators and async functions, yielding Promises. **For-await...of** loops over async iterables.

```javascript
async function* fetchUsers() {
  for (let i = 1; i <= 3; i++) {
    const response = await fetch(`/api/users/${i}`);
    yield await response.json();
  }
}

// Consume async generator
async function displayUsers() {
  for await (const user of fetchUsers()) {
    console.log(user.name);
  }
}

displayUsers();
```

---

## Interview Questions

**Q1: What's the difference between Array and Set? When should you use each?**

Arrays store ordered values with possible duplicates and support indexing ($O(1)$ access). Sets store unique values without indexing ($O(1)$ lookup). Use Arrays for ordered data and indexing; use Sets for uniqueness and fast membership testing.

**Q2: Explain generators. Why are they useful?**

Generators pause and resume execution, yielding values lazily. Useful for: creating iterables without materializing entire sequences, managing state machines, and avoiding callback hell (though Promises/async-await are now preferred).

**Q3: What's the difference between Map and Object? When should you use Map?**

Objects use string keys; Maps support any type as keys. Objects have prototype inheritance; Maps don't. Maps maintain insertion order; Objects (historically) don't. Use Map when you need non-string keys or don't want prototype chains.

**Q4: Explain DataView vs Typed Arrays. Which should you use?**

DataView provides flexible, type-agnostic access to ArrayBuffer with endianness control. Typed arrays are faster but fixed-type. Use Typed Arrays for performance with homogeneous data; use DataView for heterogeneous or flexible data types.

---

## Key Takeaways

1. **Arrays are ordered with indexing; Sets are unordered with uniqueness** - Choose based on needs
2. **Map supports any type as key; Objects only support strings** - Use Map for complex key types
3. **Array methods create new arrays (immutable) or mutate original** - Be aware of mutation side effects
4. **Iterables and iterators enable for...of loops** - Implement `Symbol.iterator` for custom iterables
5. **Typed Arrays are fast, fixed-type views over buffers** - Use for binary data and performance
6. **Generators lazily yield values** - Useful for infinite sequences or state machines
7. **WeakMaps prevent garbage collection issues** - Use for object metadata
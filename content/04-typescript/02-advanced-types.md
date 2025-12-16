# Advanced Types & Type Manipulation: L4 Engineering Guide

## Part 1: Advanced Type Constructs

### 1.1 Mapped Types & Template Literal Types

**Mapped types** transform existing types into new types by iterating over properties. They enable DRY (don't repeat yourself) patterns for type derivation.

**Template literal types** use string interpolation in types, enabling sophisticated type-level string manipulation.

```typescript
// Mapped type: make all properties optional
type Partial<T> = {
  [K in keyof T]?: T[K];
};

type User = { name: string; age: number };
type PartialUser = Partial<User>; // { name?: string; age?: number }

// Mapped type: make all properties readonly
type Readonly<T> = {
  readonly [K in keyof T]: T[K];
};

// Template literal type
type Greeting<Name extends string> = `Hello, ${Name}!`;
type greeting1 = Greeting<"Alice">; // "Hello, Alice!"

// Union of template literals
type Status = "active" | "inactive";
type StatusMessage = `User is ${Status}`; // "User is active" | "User is inactive"
```

### 1.2 Conditional Types & Type Inference

**Conditional types** enable type-level if/else logic: `T extends U ? X : Y`.

**`infer` keyword** enables extracting types from complex structures, supporting sophisticated type manipulation.

```typescript
// Conditional type
type IsString<T> = T extends string ? true : false;
type A = IsString<"hello">; // true
type B = IsString<number>; // false

// Conditional with inference
type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type StringReturner = () => string;
type StringType = ReturnType<StringReturner>; // string

// Conditional distribution (over unions)
type Flatten<T> = T extends Array<infer U> ? U : T;
type Str = Flatten<string[]>; // string
type Num = Flatten<number>; // number
type Union = Flatten<string[] | number>; // string | number
```

### 1.3 Keyof & Index Signatures

**`keyof` operator** extracts keys from a type as a union. **`keyof T[K]` pattern** enables working with object properties type-safely.

**Index signatures** define object properties dynamically: `[key: string]: value`.

```typescript
type User = { name: string; age: number; email: string };
type UserKeys = keyof User; // "name" | "age" | "email"

// Function ensuring property exists and is typed correctly
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user: User = { name: "John", age: 30, email: "john@example.com" };
const name = getProperty(user, "name"); // string
// getProperty(user, "invalid"); // Error: "invalid" not in keyof User

// Index signature
type StringMap = {
  [key: string]: unknown; // Any string key, any value
};

const obj: StringMap = { a: 1, b: "string", c: true };
```

---

## Part 2: Generics & Type Constraints

### 2.1 Generic Constraints

**Generics** enable writing type-safe code that works with multiple types. **Constraints** limit which types can substitute generic parameters.

```typescript
// Generic without constraint
function identity<T>(x: T): T {
  return x;
}

// Generic with constraint
function getLength<T extends { length: number }>(x: T): number {
  return x.length;
}

getLength("hello"); // Works: strings have length
getLength([1, 2, 3]); // Works: arrays have length
// getLength(123); // Error: numbers don't have length property

// Multiple constraints via intersection
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return { ...a, ...b };
}

merge({ x: 1 }, { y: 2 }); // { x: 1, y: 2 }

// Keyof constraint
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}
```

### 2.2 Variance & Covariance

**Variance** describes how generic types relate when their type parameters differ. **Covariance** (output position) allows subtypes; **contravariance** (input position) allows supertypes; **invariance** requires exact match.

TypeScript's type system prioritizes practical usability over theoretical purity.

```typescript
// Covariance in return types
interface Animal { }
interface Dog extends Animal { bark(): void; }

function getAnimal(): Animal { return {}; }
function getDog(): Dog { return new Dog(); }

// getDog() can be assigned to getAnimal() (covariant)
const fn1: () => Animal = getDog(); // OK

// Contravariance in parameters
function handleAnimal(animal: Animal) { }
function handleDog(dog: Dog) { }

// handleAnimal can be assigned to handleDog (contravariant)
const fn2: (dog: Dog) => void = handleAnimal; // OK
fn2(new Dog()); // Safe: handleAnimal accepts Dog

// Invariance in mutable structures
// Arrays: invariant in TypeScript (strict)
const dogArray: Dog[] = [];
// const animalArray: Animal[] = dogArray; // Error: invariant
```

---

## Part 3: Utility Types

### 3.1 Built-In Utility Types

TypeScript provides utility types for common type transformations. These are implemented using mapped types, conditional types, and other advanced features.

```typescript
// Partial: all properties optional
type PartialUser = Partial<{ name: string; age: number }>;

// Required: all properties required
type RequiredUser = Required<{ name?: string; age?: number }>;

// Readonly: all properties immutable
type ReadonlyUser = Readonly<{ name: string; age: number }>;

// Pick: select specific properties
type UserPreview = Pick<User, "name" | "email">;

// Omit: exclude specific properties
type UserWithoutAge = Omit<User, "age">;

// Record: create object with specific keys
type Permissions = Record<"read" | "write" | "delete", boolean>;
// { read: boolean; write: boolean; delete: boolean }

// Exclude: remove types from union
type NonString = Exclude<string | number | boolean, string>;
// number | boolean

// Extract: keep only matching types
type StringOrNumber = Extract<string | number | boolean, string | number>;
// string | number
```

### 3.2 Type Predicates & Assertion Functions

**Type predicates** (`is` keyword) narrow types within conditionals. **Assertion functions** throw if a condition is false, narrowing the type afterward.

```typescript
// Type predicate
function isString(value: unknown): value is string {
  return typeof value === "string";
}

const input: unknown = "hello";
if (isString(input)) {
  console.log(input.toUpperCase()); // input: string (narrowed)
}

// Assertion function
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== "string") {
    throw new Error("Not a string");
  }
}

const data: unknown = "test";
assertIsString(data);
console.log(data.toUpperCase()); // data: string (narrowed)
```

---

## Interview Questions

**Q1: Explain mapped types and their practical use cases.**

Mapped types iterate over object properties to create new types. Practical examples: `Partial<T>` makes all properties optional, `Readonly<T>` makes all properties immutable, `Record<K, V>` creates objects with specific keys. Essential for reducing type boilerplate.

**Q2: What's the difference between covariance and contravariance?**

Covariance allows subtypes in return types (functions returning Dogs are assignable to functions returning Animals). Contravariance allows supertypes in parameter types (functions accepting Animals are assignable to functions accepting Dogs). Invariance requires exact type match (arrays in TypeScript).

**Q3: Explain `infer` in conditional types. Give a practical example.**

`infer` extracts types from complex structures within conditional types. Example: `type ReturnType<T> = T extends (...args: any[]) => infer R ? R : never` extracts the return type of a function. Enables sophisticated type-level programming.

**Q4: When should you use `Pick` vs `Partial`?**

`Pick` selects specific properties to include. `Partial` makes all properties optional. Use `Pick` to create a subset with the original optionality; use `Partial` when you want to relax required properties. Combined: `Partial<Pick<T, Keys>>` for optional subset.

---

## Key Takeaways

1. **Mapped types eliminate type boilerplate** - Transform types systematically
2. **Conditional types enable type-level logic** - `infer` enables type extraction
3. **Generic constraints prevent misuse** - `T extends U` restricts type parameters
4. **Variance describes subtype relationships** - Covariance in returns, contravariance in parameters
5. **Utility types are essential tools** - `Partial`, `Required`, `Pick`, `Omit`, `Record`
6. **Type predicates narrow types safely** - `value is T` enables type-safe narrowing
7. **Assertion functions narrow permanently** - Throw if condition false; type narrowed afterward
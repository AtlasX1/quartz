# Decorators & Metadata: L4 Engineering Guide

## Part 1: Decorator Fundamentals

### 1.1 Decorators as Higher-Order Functions

**Decorators** are higher-order functions that enhance or modify classes, methods, properties, and parameters. They execute at class definition time, before any instances are created.

**Decorator syntax:** `@decoratorName` applied before the decorated element. Under the hood, decorators are functions receiving the decorated element and returning a modified version.

**Metadata** provides information about types (via `reflect-metadata`), enabling runtime type inspection and validation frameworks.

```typescript
// Enable decorators and metadata reflection
// tsconfig.json: "experimentalDecorators": true, "emitDecoratorMetadata": true

// Simple decorator: logs when method is called
function LogMethod(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;

  descriptor.value = function(...args: any[]) {
    console.log(`Called: ${propertyKey}`);
    return original.apply(this, args);
  };

  return descriptor;
}

class Calculator {
  @LogMethod
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(5, 3); // Logs: "Called: add"

// Decorator factories: parameterized decorators
function Retry(times: number) {
  return function(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;

    descriptor.value = function(...args: any[]) {
      for (let i = 0; i < times; i++) {
        try {
          return original.apply(this, args);
        } catch (e) {
          if (i === times - 1) throw e;
        }
      }
    };

    return descriptor;
  };
}

class Service {
  @Retry(3)
  fetchData() {
    // Attempted up to 3 times before throwing
  }
}
```

### 1.2 Class & Property Decorators

**Class decorators** receive the constructor and can modify the class. **Property decorators** receive the target and property key.

```typescript
// Class decorator: adds a static property
function Trackable() {
  return function<T extends { new (...args: any[]): {} }>(constructor: T) {
    return class extends constructor {
      static _tracked = true;
    };
  };
}

@Trackable()
class User {
  constructor(public name: string) {}
}

console.log((User as any)._tracked); // true

// Property decorator: logs property access
function Logged(target: any, propertyKey: string) {
  let value: any;

  const getter = () => {
    console.log(`Accessed: ${propertyKey}`);
    return value;
  };

  const setter = (newValue: any) => {
    console.log(`Set: ${propertyKey} = ${newValue}`);
    value = newValue;
  };

  Object.defineProperty(target, propertyKey, {
    get: getter,
    set: setter
  });
}

class Entity {
  @Logged
  id: number = 0;
}

const entity = new Entity();
entity.id = 5; // Logs: "Set: id = 5"
console.log(entity.id); // Logs: "Accessed: id"
```

---

## Part 2: Metadata & Reflection

### 2.1 Reflect Metadata API

**`reflect-metadata` library** enables runtime type information (RTTI) via the Reflection API. The `emitDecoratorMetadata` compiler option automatically generates metadata from TypeScript types.

```typescript
import "reflect-metadata";

// Type metadata is auto-generated when decorators are used
class User {
  name: string = "";
  age: number = 0;
}

// Reflect API enables querying metadata
const ctor = Reflect.getMetadata("design:type", User.prototype, "name");
// ctor: String constructor

// Method parameter types
function ValidateParams(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const paramTypes = Reflect.getMetadata("design:paramtypes", target, propertyKey);
  const original = descriptor.value;

  descriptor.value = function(...args: any[]) {
    paramTypes.forEach((type: any, index: number) => {
      if (!(args[index] instanceof type)) {
        throw new Error(`Parameter ${index} should be ${type.name}`);
      }
    });
    return original.apply(this, args);
  };

  return descriptor;
}

class Calculator {
  @ValidateParams
  add(a: number, b: number): number {
    return a + b;
  }
}

const calc = new Calculator();
calc.add(5, 3); // OK
calc.add("5", 3); // Error: Parameter 0 should be Number
```

### 2.2 Custom Metadata with Decorators

Custom decorators can attach arbitrary metadata to classes, methods, or properties for frameworks to use (e.g., dependency injection, validation).

```typescript
// Store metadata on classes/methods
const validateMetadata = new Map<any, Map<string, any>>();

function Validate(validator: (value: any) => boolean) {
  return function(target: any, propertyKey: string) {
    if (!validateMetadata.has(target)) {
      validateMetadata.set(target, new Map());
    }
    validateMetadata.get(target)!.set(propertyKey, validator);
  };
}

class User {
  @Validate(v => typeof v === "string" && v.length > 0)
  name: string = "";

  @Validate(v => typeof v === "number" && v > 0)
  age: number = 0;
}

// Framework uses metadata
function validate(instance: any): boolean {
  const validators = validateMetadata.get(Object.getPrototypeOf(instance));
  if (!validators) return true;

  for (const [key, validator] of validators) {
    if (!validator(instance[key])) {
      console.error(`Validation failed: ${key}`);
      return false;
    }
  }
  return true;
}

const user = new User();
user.name = "John";
user.age = 30;
validate(user); // true
```

---

## Part 3: Advanced Decorator Patterns

### 3.1 Parameter & Accessor Decorators

**Parameter decorators** receive the target, property key, and parameter index. **Accessor decorators** modify getters/setters.

```typescript
// Parameter decorator: logs parameters
function LogParams(target: any, propertyKey: string, parameterIndex: number) {
  const existingMetadata = Reflect.getOwnMetadata("log:params", target, propertyKey) || [];
  existingMetadata.push(parameterIndex);
  Reflect.defineMetadata("log:params", existingMetadata, target, propertyKey);
}

function LogParamsDecorator(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const paramIndices = Reflect.getOwnMetadata("log:params", target, propertyKey) || [];
  const original = descriptor.value;

  descriptor.value = function(...args: any[]) {
    paramIndices.forEach((index: number) => {
      console.log(`Parameter ${index}: ${args[index]}`);
    });
    return original.apply(this, args);
  };

  return descriptor;
}

class Logger {
  @LogParamsDecorator
  log(@LogParams message: string, @LogParams level: string) {
    console.log(`${level}: ${message}`);
  }
}

// Accessor decorator: log access
function LogAccessor(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const original = descriptor.get;

  descriptor.get = function() {
    console.log(`Getting ${propertyKey}`);
    return original?.call(this);
  };

  return descriptor;
}

class Property {
  private _value: number = 0;

  @LogAccessor
  get value(): number {
    return this._value;
  }
}
```

### 3.2 Practical Decorator Patterns

**Common patterns:** validation, logging, memoization, dependency injection, authorization.

```typescript
// Memoization decorator
function Memoize(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;
  const cache = new Map();

  descriptor.value = function(...args: any[]) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      return cache.get(key);
    }

    const result = original.apply(this, args);
    cache.set(key, result);
    return result;
  };

  return descriptor;
}

// Authorization decorator
function RequireAuth(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const original = descriptor.value;

  descriptor.value = function(...args: any[]) {
    if (!isUserAuthenticated()) {
      throw new Error("Unauthorized");
    }
    return original.apply(this, args);
  };

  return descriptor;
}

class UserService {
  @RequireAuth
  @Memoize
  getUserProfile(userId: string) {
    // Fetches profile, checks auth, caches result
    return { id: userId, name: "John" };
  }
}
```

---

## Interview Questions

**Q1: Explain decorators and their execution timing.**

Decorators are higher-order functions applied at class definition time, before instances are created. They enhance or modify classes, methods, properties. Syntax: `@decorator`. Execution order: class decorators last, then methods, properties, parameters.

**Q2: What's the difference between decorators and middleware?**

Decorators modify class/method definitions at compile time. Middleware processes requests/responses at runtime. Decorators are compile-time transformations; middleware is runtime interception.

**Q3: How do you use `reflect-metadata` for runtime type information?**

`reflect-metadata` enables querying types at runtime. `Reflect.getMetadata("design:type", target, key)` retrieves type information automatically generated when `emitDecoratorMetadata` is enabled. Enables runtime validation and dependency injection.

**Q4: What are practical use cases for decorators?**

Validation (checking property values), logging (recording method calls), memoization (caching results), authorization (enforcing access control), dependency injection (providing dependencies). Frameworks like NestJS extensively use decorators for routing and middleware.

---

## Key Takeaways

1. **Decorators execute at class definition time** - Before instances are created
2. **Decorators are higher-order functions** - Receive element, return modified version
3. **Metadata enables runtime type information** - `reflect-metadata` + `emitDecoratorMetadata`
4. **Multiple decorators compose** - Applied in reverse order (bottom-up)
5. **Factory patterns enable parameterized decorators** - Return decorator function from factory
6. **Decorators enable frameworks** - Validation, logging, dependency injection, authorization
7. **Practical decorators: @Validate, @Log, @Memoize, @Auth** - Common patterns in production code
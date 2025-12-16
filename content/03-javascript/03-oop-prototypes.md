# Object-Oriented JavaScript & Prototypes: L4 Engineering Guide

## Part 1: Prototype-Based Inheritance

### 1.1 Prototype Chain & Delegation

JavaScript implements inheritance through **delegation** rather than classical inheritance. Objects delegate method/property lookups to their prototype when properties aren't found locally. This creates a prototype chain: `object → prototype → prototype.prototype → Object.prototype → null`.

When accessing a property, JavaScript checks:
1. Own properties (properties directly on the object)
2. Prototype's properties
3. Prototype's prototype's properties (chain continues)
4. Returns `undefined` if not found

**The `[[Prototype]]` slot** (not directly accessible) holds the reference to an object's prototype. Use `Object.getPrototypeOf(obj)` to access it.

```javascript
const parent = { greet: () => 'Hello' };
const child = Object.create(parent);
child.name = 'Child';

console.log(child.name); // Own property
console.log(child.greet()); // Delegated to parent (via prototype chain)
console.log(Object.getPrototypeOf(child) === parent); // true

// Prototype chain inspection
console.log(child.hasOwnProperty('name')); // true
console.log(child.hasOwnProperty('greet')); // false (inherited)
```

### 1.2 Constructors & the `new` Operator

The `new` operator creates a new object and sets its `[[Prototype]]` to the constructor function's `prototype` property. Constructor functions are conventionally capitalized.

**`new` process (4 steps):**
1. Create empty object
2. Set object's `[[Prototype]]` to `Constructor.prototype`
3. Execute constructor with `this` bound to the new object
4. Return the object (unless constructor explicitly returns an object)

```javascript
function Animal(name) {
  this.name = name;
}

Animal.prototype.speak = function() {
  return `${this.name} speaks`;
};

const dog = new Animal('Dog');
console.log(dog.name); // 'Dog' (own property)
console.log(dog.speak()); // 'Dog speaks' (inherited method)
console.log(dog instanceof Animal); // true
```

### 1.3 Prototype Pitfalls

**Shared prototype properties:** Arrays and objects in prototype are shared across instances, causing unintended mutations.

```javascript
function MyClass() {}
MyClass.prototype.items = []; // Shared array

const obj1 = new MyClass();
const obj2 = new MyClass();
obj1.items.push(1);
console.log(obj2.items); // [1] (unintended sharing)

// Solution: initialize in constructor
function MyClass2() {
  this.items = []; // Own property, not shared
}
```

---

## Part 2: ES6 Classes

### 2.1 Class Syntax & Semantics

ES6 classes are syntactic sugar over prototypes. They provide clearer inheritance syntax but operate identically to constructor functions and prototypes.

**Class structure:**
- Constructor method runs during instantiation
- Methods are added to prototype (shared across instances)
- Static methods belong to the class, not instances
- Private fields (`#fieldName`) are inaccessible outside the class

```javascript
class Animal {
  static species = 'Unknown'; // Static property
  #privateAge = 0; // Private field

  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} speaks`;
  }

  static info() {
    return `Species: ${Animal.species}`;
  }

  getAge() {
    return this.#privateAge;
  }

  setAge(age) {
    if (age < 0) throw new Error('Age cannot be negative');
    this.#privateAge = age;
  }
}

const dog = new Animal('Dog');
dog.speak(); // 'Dog speaks'
Animal.info(); // 'Species: Unknown'
dog.getAge(); // 0
// dog.#privateAge; // SyntaxError—private field inaccessible
```

### 2.2 Inheritance with `extends` & `super`

**`extends`** sets the prototype chain: `ChildClass.prototype → ParentClass.prototype`.

**`super`** calls the parent class constructor and methods, enabling method overriding.

```javascript
class Animal {
  constructor(name) {
    this.name = name;
  }

  speak() {
    return `${this.name} makes a sound`;
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name); // Call parent constructor
    this.breed = breed;
  }

  speak() {
    return `${super.speak()} - Woof!`; // Call parent method
  }
}

const dog = new Dog('Rex', 'Labrador');
console.log(dog.speak()); // 'Rex makes a sound - Woof!'
console.log(dog instanceof Animal); // true
console.log(dog instanceof Dog); // true
```

### 2.3 Composition vs Inheritance

**Inheritance** models "is-a" relationships but creates tight coupling and rigid hierarchies. Changes to parent classes affect all children.

**Composition** models "has-a" relationships and is more flexible. Objects combine behaviors from multiple mixins without hierarchy constraints.

```javascript
// Inheritance (rigid)
class Vehicle { move() { } }
class Car extends Vehicle { }
class Boat extends Vehicle { }
// Problem: what if we need an amphibious vehicle? Multiple inheritance isn't supported.

// Composition (flexible)
const canMove = { move() { console.log('Moving'); } };
const canSwim = { swim() { console.log('Swimming'); } };
const canFly = { fly() { console.log('Flying'); } };

const car = { ...canMove };
const plane = { ...canMove, ...canFly };
const amphibian = { ...canMove, ...canSwim };

// Mixins
function applyMixin(target, mixin) {
  Object.assign(target, mixin);
}

const boat = {};
applyMixin(boat, canSwim);
applyMixin(boat, canMove);
```

---

## Part 3: Advanced Prototype Patterns

### 3.1 Object.create() & OLOO Pattern

**OLOO (Objects Linking to Other Objects)** is a pattern using `Object.create()` to establish prototype relationships without classes/constructors.

```javascript
const AnimalPrototype = {
  init(name) {
    this.name = name;
    return this;
  },
  speak() {
    return `${this.name} speaks`;
  }
};

const DogPrototype = Object.create(AnimalPrototype);
DogPrototype.bark = function() {
  return `${this.name} barks`;
};

const dog = Object.create(DogPrototype).init('Rex');
console.log(dog.speak()); // 'Rex speaks'
console.log(dog.bark()); // 'Rex barks'
```

### 3.2 Getters & Setters

Property accessors enable custom logic when reading/writing properties, useful for validation or computed properties.

```javascript
class Person {
  constructor(firstName, lastName) {
    this._firstName = firstName;
    this._lastName = lastName;
  }

  get fullName() {
    return `${this._firstName} ${this._lastName}`;
  }

  set fullName(name) {
    const [first, last] = name.split(' ');
    this._firstName = first;
    this._lastName = last;
  }

  get age() {
    return 2025 - this._birthYear;
  }

  set age(years) {
    this._birthYear = 2025 - years;
  }
}

const person = new Person('John', 'Doe');
console.log(person.fullName); // 'John Doe'
person.fullName = 'Jane Smith';
console.log(person._firstName); // 'Jane'
```

---

## Interview Questions

**Q1: Explain the prototype chain and how property lookup works.**

When accessing a property, JavaScript checks own properties first, then the prototype, then the prototype's prototype (chain continues). If the property isn't found anywhere, `undefined` is returned. This enables delegation-based inheritance.

**Q2: What's the difference between class-based and prototype-based inheritance?**

Classes are syntactic sugar over prototypes; they provide clearer inheritance syntax but operate identically. Prototypes use constructor functions and explicit prototype chains; classes use `extends` and `super` for inheritance.

**Q3: When should you use composition over inheritance?**

Use composition for "has-a" relationships and flexibility. Inheritance is suitable for "is-a" relationships but creates rigid hierarchies. Composition via mixins allows combining multiple behaviors without hierarchy constraints.

**Q4: What happens when you call `new` on a constructor function?**

`new` creates a new object, sets its `[[Prototype]]` to `Constructor.prototype`, executes the constructor with `this` bound to the new object, and returns the object (unless the constructor explicitly returns an object).

---

## Key Takeaways

1. **Prototype chain enables delegation** - Property lookups traverse the chain until found
2. **`new` operator establishes prototype linkage** - Sets `[[Prototype]]` to constructor's `prototype` property
3. **ES6 classes are syntactic sugar** - Equivalent to constructor functions + prototypes
4. **Shared prototype properties create bugs** - Initialize mutable properties in the constructor
5. **Composition is more flexible than inheritance** - Avoid rigid class hierarchies when possible
6. **`super` enables parent method access** - Use for calling parent constructors and overriding methods
7. **Private fields (`#`) hide implementation** - Useful for encapsulation and preventing accidental access
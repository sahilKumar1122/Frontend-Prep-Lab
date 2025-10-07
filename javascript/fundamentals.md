# JavaScript Fundamentals

## Table of Contents
- [Data Types](#data-types)
- [Variables](#variables)
- [Functions](#functions)
- [Objects and Arrays](#objects-and-arrays)
- [this Keyword](#this-keyword)
- [Event Loop](#event-loop)

---

## Data Types

### What are the primitive data types in JavaScript?

**Answer:**

JavaScript has 7 primitive data types:
1. **String** - represents text data
2. **Number** - represents numeric values (both integers and floats)
3. **Boolean** - represents true/false
4. **Undefined** - variable declared but not assigned
5. **Null** - intentional absence of value
6. **Symbol** - unique identifier (ES6)
7. **BigInt** - large integers (ES2020)

**Code Example:**
```javascript
// Primitives
let str = "Hello";
let num = 42;
let bool = true;
let undef = undefined;
let nothing = null;
let sym = Symbol("unique");
let bigNum = 9007199254740991n;

// Check types
console.log(typeof str);     // "string"
console.log(typeof num);     // "number"
console.log(typeof bool);    // "boolean"
console.log(typeof undef);   // "undefined"
console.log(typeof nothing); // "object" (historical bug)
console.log(typeof sym);     // "symbol"
console.log(typeof bigNum);  // "bigint"
```

**Key Points:**
- Primitives are immutable
- `typeof null` returns "object" (this is a known bug)
- All other values are objects (including arrays and functions)

---

## Variables

### What's the difference between var, let, and const?

**Answer:**

| Feature | var | let | const |
|---------|-----|-----|-------|
| Scope | Function-scoped | Block-scoped | Block-scoped |
| Hoisting | Yes (initialized as undefined) | Yes (not initialized - TDZ) | Yes (not initialized - TDZ) |
| Re-declaration | Yes | No | No |
| Re-assignment | Yes | Yes | No |

**Code Example:**
```javascript
// var - function scoped
function varExample() {
  var x = 1;
  if (true) {
    var x = 2; // Same variable
    console.log(x); // 2
  }
  console.log(x); // 2
}

// let - block scoped
function letExample() {
  let x = 1;
  if (true) {
    let x = 2; // Different variable
    console.log(x); // 2
  }
  console.log(x); // 1
}

// const - block scoped, cannot reassign
const PI = 3.14159;
// PI = 3.14; // Error!

// But objects/arrays can be mutated
const obj = { name: "John" };
obj.name = "Jane"; // This is OK
obj.age = 30;      // This is OK
```

**Key Points:**
- Always prefer `const` by default
- Use `let` when you need to reassign
- Avoid `var` in modern JavaScript
- TDZ (Temporal Dead Zone) - accessing let/const before declaration throws error

---

## Functions

### What are the different ways to define functions in JavaScript?

**Answer:**

1. **Function Declaration**
2. **Function Expression**
3. **Arrow Function**
4. **Constructor Function**
5. **Generator Function**

**Code Example:**
```javascript
// 1. Function Declaration (hoisted)
function add(a, b) {
  return a + b;
}

// 2. Function Expression (not hoisted)
const subtract = function(a, b) {
  return a - b;
};

// 3. Arrow Function (ES6)
const multiply = (a, b) => a * b;

// Arrow function with multiple statements
const divide = (a, b) => {
  if (b === 0) throw new Error("Division by zero");
  return a / b;
};

// 4. Constructor Function
function Person(name, age) {
  this.name = name;
  this.age = age;
}

// 5. Generator Function
function* idGenerator() {
  let id = 1;
  while (true) {
    yield id++;
  }
}
```

**Key Points:**
- Function declarations are hoisted
- Arrow functions don't have their own `this`
- Arrow functions can't be used as constructors
- Generator functions return iterators

---

## Objects and Arrays

### How do you deep clone an object?

**Answer:**

There are several ways to deep clone an object in JavaScript:

**Code Example:**
```javascript
const original = {
  name: "John",
  address: {
    city: "New York",
    country: "USA"
  },
  hobbies: ["reading", "coding"]
};

// 1. JSON.parse/stringify (doesn't work with functions, dates, undefined, symbols)
const clone1 = JSON.parse(JSON.stringify(original));

// 2. Structured Clone (modern approach)
const clone2 = structuredClone(original);

// 3. Manual recursive clone
function deepClone(obj) {
  if (obj === null || typeof obj !== 'object') return obj;
  
  if (obj instanceof Date) return new Date(obj);
  if (obj instanceof Array) return obj.map(item => deepClone(item));
  
  const cloned = {};
  for (let key in obj) {
    if (obj.hasOwnProperty(key)) {
      cloned[key] = deepClone(obj[key]);
    }
  }
  return cloned;
}

const clone3 = deepClone(original);

// 4. Using lodash (third-party library)
// const clone4 = _.cloneDeep(original);
```

**Key Points:**
- `structuredClone()` is the modern built-in solution
- JSON method fails with functions, undefined, symbols, dates
- Consider using libraries like lodash for complex cases
- Shallow clone: `Object.assign()` or spread operator `{...obj}`

---

## this Keyword

### How does 'this' work in JavaScript?

**Answer:**

The value of `this` depends on how the function is called:
1. **Global context** - window (browser) or global (Node.js)
2. **Object method** - the object
3. **Constructor** - the new instance
4. **Arrow function** - lexical `this` (inherited from parent)
5. **Event handler** - the element
6. **call/apply/bind** - explicitly set

**Code Example:**
```javascript
// 1. Global context
console.log(this); // window (in browser)

// 2. Object method
const person = {
  name: "John",
  greet: function() {
    console.log(this.name); // "John"
  }
};
person.greet();

// 3. Constructor
function Car(brand) {
  this.brand = brand;
}
const myCar = new Car("Toyota");
console.log(myCar.brand); // "Toyota"

// 4. Arrow function (lexical this)
const obj = {
  name: "Alice",
  regularFunc: function() {
    console.log(this.name); // "Alice"
  },
  arrowFunc: () => {
    console.log(this.name); // undefined (or window.name)
  }
};

// 5. Explicit binding
function greet() {
  console.log(`Hello, ${this.name}`);
}
const user = { name: "Bob" };
greet.call(user);  // "Hello, Bob"
greet.apply(user); // "Hello, Bob"
const boundGreet = greet.bind(user);
boundGreet();      // "Hello, Bob"
```

**Key Points:**
- Arrow functions inherit `this` from their lexical scope
- Regular functions have dynamic `this` based on how they're called
- Use `bind()` to permanently set `this`
- In strict mode, `this` is `undefined` in global context

---

## Event Loop

### Explain the JavaScript Event Loop

**Answer:**

The Event Loop is JavaScript's mechanism for handling asynchronous operations despite being single-threaded.

**Components:**
1. **Call Stack** - executes synchronous code
2. **Web APIs** - handle async operations (setTimeout, fetch, etc.)
3. **Callback Queue (Task Queue)** - holds callbacks from Web APIs
4. **Microtask Queue** - holds promises, mutation observers
5. **Event Loop** - moves tasks from queues to call stack

**Code Example:**
```javascript
console.log('1. Sync start');

setTimeout(() => {
  console.log('2. Timeout callback');
}, 0);

Promise.resolve().then(() => {
  console.log('3. Promise callback');
});

console.log('4. Sync end');

// Output:
// 1. Sync start
// 4. Sync end
// 3. Promise callback
// 2. Timeout callback
```

**Execution Order:**
1. Synchronous code runs first
2. Microtasks (Promises) run next
3. Macrotasks (setTimeout, setInterval) run last

**Key Points:**
- JavaScript is single-threaded
- Microtasks have higher priority than macrotasks
- Event loop continuously checks if call stack is empty
- Promises are microtasks, setTimeout is macrotask

---

## Additional Topics to Practice

- Type coercion and equality (`==` vs `===`)
- Hoisting
- Spread and Rest operators
- Destructuring
- Template literals
- Default parameters
- Short-circuit evaluation
- Truthy and Falsy values

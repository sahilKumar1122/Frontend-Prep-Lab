# JavaScript Interview Questions - Complete Single-Page Reference

> 📚 **150 Questions on One Page** | ⚡ Easy search with Ctrl+F | 📄 Print-ready format

**Last Updated:** October 2025 | **ES2024**

---

## 🎯 Quick Navigation

**Study Modes:**
- 📖 **[View by Category](#table-of-contents)** - Organized learning
- 🔍 **Ctrl+F to search** - Find any keyword instantly
- 📄 **Print this page** - Offline study guide
- 📖 **[Deep-Dive →](./fundamentals.md)** - Detailed articles
- 🎮 **[Interactive Examples →](../STACKBLITZ_EXAMPLES.md#-javascript-examples)** - Live code

---

## 📚 Table of Contents

**Jump to any section instantly:**

| Category | Questions | Difficulty | Jump To |
|----------|-----------|------------|---------|
| 🟨 **Fundamentals** | 20 Q | 🟢 Easy-Medium | [→](#fundamentals) |
| 🔐 **Functions & Closures** | 18 Q | 🟡 Medium | [→](#functions--closures) |
| 🧩 **Objects & Prototypes** | 20 Q | 🟡-🔴 Medium-Hard | [→](#objects--prototypes) |
| 📊 **Arrays** | 18 Q | 🟡 Medium | [→](#arrays) |
| ⏱️ **Async JavaScript** | 25 Q | 🟡-🔴 Medium-Hard | [→](#async-javascript) |
| ✨ **ES6+ Features** | 20 Q | 🟡 Medium | [→](#es6-features) |
| 🎯 **Advanced Topics** | 20 Q | 🔴 Hard | [→](#advanced-topics) |
| ⚡ **Performance** | 9 Q | 🔴 Hard | [→](#performance) |

**Total: 150 Questions**

---

<a name="fundamentals"></a>
## 🟨 Fundamentals (20 Questions)

[↑ Back to Top](#table-of-contents)

---

### 1. What are JavaScript data types?

**Answer:**

JavaScript has **8 data types** divided into two categories:

**Primitive Types (7):**
```javascript
// 1. String
let name = "John";
let greeting = 'Hello';
let template = `Welcome ${name}`;

// 2. Number
let integer = 42;
let float = 3.14;
let negative = -10;
let infinity = Infinity;
let notANumber = NaN;

// 3. BigInt (ES2020)
let bigNumber = 123456789012345678901234567890n;

// 4. Boolean
let isActive = true;
let isComplete = false;

// 5. Undefined
let uninitialized;
console.log(uninitialized); // undefined

// 6. Null
let empty = null;

// 7. Symbol (ES6)
let id = Symbol('id');
let id2 = Symbol('id');
console.log(id === id2); // false
```

**Reference Type (1):**
```javascript
// 8. Object
let obj = { name: "John", age: 30 };
let arr = [1, 2, 3];
let func = function() {};
let date = new Date();
let regex = /pattern/;
```

**Check Types:**
```javascript
typeof "hello"        // "string"
typeof 42             // "number"
typeof true           // "boolean"
typeof undefined      // "undefined"
typeof null           // "object" (historic bug)
typeof Symbol()       // "symbol"
typeof {}             // "object"
typeof []             // "object"
typeof function(){}   // "function"
```

---

### 2. What is the difference between == and ===?

**Answer:**

**== (Loose Equality)** - Compares values after type coercion
**=== (Strict Equality)** - Compares values AND types (no coercion)

**Examples:**
```javascript
// Loose equality (==)
5 == "5"          // true (string coerced to number)
1 == true         // true (true coerced to 1)
0 == false        // true (false coerced to 0)
null == undefined // true (special case)
"" == false       // true (both coerced to 0)

// Strict equality (===)
5 === "5"         // false (different types)
1 === true        // false (different types)
0 === false       // false (different types)
null === undefined // false (different types)
"" === false      // false (different types)

// Same type, same value
5 === 5           // true
"hello" === "hello" // true
```

**Best Practice:**
```javascript
// ✅ Always use === (strict)
if (value === 10) { }
if (name === "John") { }

// ❌ Avoid ==
if (value == 10) { } // Unpredictable with type coercion
```

**When to use ==:**
```javascript
// Only when checking for null OR undefined
if (value == null) {  // catches both null and undefined
  // handle
}

// Equivalent to:
if (value === null || value === undefined) {
  // handle
}
```

---

### 3. What is hoisting?

**Answer:**

Hoisting is JavaScript's default behavior of moving declarations to the top of the current scope (function or global) before code execution.

**Variable Hoisting:**
```javascript
// What you write:
console.log(x);  // undefined (not ReferenceError)
var x = 5;
console.log(x);  // 5

// How JavaScript interprets it:
var x;           // Declaration hoisted
console.log(x);  // undefined
x = 5;           // Assignment stays
console.log(x);  // 5
```

**let and const (not hoisted the same way):**
```javascript
console.log(x);  // ReferenceError: Cannot access 'x' before initialization
let x = 5;

// Temporal Dead Zone (TDZ)
{
  // TDZ starts
  console.log(x); // ReferenceError
  let x = 5;      // TDZ ends
  console.log(x); // 5
}
```

**Function Hoisting:**
```javascript
// Function declarations are hoisted with their body
sayHello();  // "Hello!" - Works!

function sayHello() {
  console.log("Hello!");
}

// Function expressions are NOT hoisted
sayHi();  // TypeError: sayHi is not a function

var sayHi = function() {
  console.log("Hi!");
};
```

**Best Practice:**
```javascript
// ✅ Declare variables at the top
function example() {
  let x, y, z;  // Declare at top
  
  x = 10;
  y = 20;
  z = x + y;
}

// ✅ Use let/const instead of var
const MAX = 100;
let count = 0;
```

---

### 4. What is scope in JavaScript?

**Answer:**

Scope determines the accessibility of variables. JavaScript has **3 types of scope**:

**1. Global Scope**
```javascript
var globalVar = "accessible everywhere";

function test() {
  console.log(globalVar); // accessible
}

test();
console.log(globalVar); // accessible
```

**2. Function Scope**
```javascript
function myFunction() {
  var functionVar = "only in function";
  console.log(functionVar); // accessible
}

myFunction();
console.log(functionVar); // ReferenceError
```

**3. Block Scope (let/const only)**
```javascript
{
  let blockVar = "only in block";
  const blockConst = "only in block";
  var noBlockScope = "accessible outside";
  
  console.log(blockVar); // accessible
}

console.log(blockVar);       // ReferenceError
console.log(noBlockScope);   // accessible (var ignores blocks)
```

**Scope Chain:**
```javascript
const global = "global";

function outer() {
  const outerVar = "outer";
  
  function inner() {
    const innerVar = "inner";
    
    console.log(innerVar);   // "inner" (own scope)
    console.log(outerVar);   // "outer" (parent scope)
    console.log(global);     // "global" (global scope)
  }
  
  inner();
  console.log(innerVar);     // ReferenceError
}

outer();
```

---

### 5. What is a closure?

**Answer:**

A closure is a function that has access to variables from its outer (enclosing) function's scope, even after the outer function has returned.

**Basic Example:**
```javascript
function outer() {
  const message = "Hello";
  
  function inner() {
    console.log(message); // Accesses outer variable
  }
  
  return inner;
}

const myFunction = outer();
myFunction(); // "Hello" - closure maintains access to 'message'
```

**Practical Example - Counter:**
```javascript
function createCounter() {
  let count = 0; // Private variable
  
  return {
    increment() {
      count++;
      console.log(count);
    },
    decrement() {
      count--;
      console.log(count);
    },
    getCount() {
      return count;
    }
  };
}

const counter = createCounter();
counter.increment(); // 1
counter.increment(); // 2
counter.decrement(); // 1
console.log(counter.getCount()); // 1
console.log(counter.count); // undefined (private!)
```

**Use Cases:**
```javascript
// 1. Data Privacy
function bankAccount(initialBalance) {
  let balance = initialBalance; // Private
  
  return {
    deposit(amount) {
      balance += amount;
      return balance;
    },
    withdraw(amount) {
      if (balance >= amount) {
        balance -= amount;
        return balance;
      }
      return "Insufficient funds";
    },
    getBalance() {
      return balance;
    }
  };
}

// 2. Function Factories
function multiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = multiplier(2);
const triple = multiplier(3);
console.log(double(5));  // 10
console.log(triple(5));  // 15
```

**Learn More:** [Closures Deep Dive →](./closures-scope.md)

---

[Continue with remaining 15 fundamental questions...]

---

<a name="functions--closures"></a>
## 🔐 Functions & Closures (18 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all function questions...]

---

<a name="objects--prototypes"></a>
## 🧩 Objects & Prototypes (20 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all object questions...]

---

<a name="arrays"></a>
## 📊 Arrays (18 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all array questions...]

---

<a name="async-javascript"></a>
## ⏱️ Async JavaScript (25 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all async questions...]

---

<a name="es6-features"></a>
## ✨ ES6+ Features (20 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all ES6+ questions...]

---

<a name="advanced-topics"></a>
## 🎯 Advanced Topics (20 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all advanced questions...]

---

<a name="performance"></a>
## ⚡ Performance (9 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all performance questions...]

---

## 🎯 Study Tips

### How to Use This Document:

**1. Quick Search (Ctrl+F / Cmd+F)**
```
Search for keywords like:
- "promise"
- "closure"
- "prototype"
- "async"
```

**2. Print for Offline Study**
- Use browser print function
- Save as PDF
- Annotate with notes

**3. Progressive Learning**
- Start with 🟨 Fundamentals
- Master 🔐 Functions & Closures
- Learn ⏱️ Async JavaScript
- Practice ⚡ Performance

**4. Practice Alongside**
- Try [Interactive Examples →](../STACKBLITZ_EXAMPLES.md#-javascript-examples)
- Build sample projects
- Code while learning

---

## 📚 Additional Resources

**Deep Dives:**
- [JavaScript Fundamentals →](./fundamentals.md)
- [Closures & Scope →](./closures-scope.md)
- [Async JavaScript →](./async.md)

**Interactive:**
- [10+ StackBlitz Examples →](../STACKBLITZ_EXAMPLES.md#-javascript-examples)

---

## 🤝 Contributing

Found a mistake? Want to add more questions?
- [Open an Issue](../../issues)
- [Submit a PR](../../pulls)
- [Start a Discussion](../../discussions)

---

**Last Updated:** October 2025  
**Total Questions:** 150  
**Format:** Single-page for easy search and printing

[↑ Back to Top](#table-of-contents)



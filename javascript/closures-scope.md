# Closures & Scope

## Table of Contents
- [Scope](#scope)
- [Closures](#closures)
- [Lexical Scope](#lexical-scope)
- [Module Pattern](#module-pattern)
- [Common Pitfalls](#common-pitfalls)

---

## Scope

### What are the different types of scope in JavaScript?

**Answer:**

JavaScript has three types of scope:
1. **Global Scope** - accessible everywhere
2. **Function Scope** - accessible within function
3. **Block Scope** - accessible within block (let/const only)

**Code Example:**
```javascript
// Global Scope
var globalVar = "I'm global";
let globalLet = "I'm also global";

function scopeExample() {
  // Function Scope
  var functionVar = "I'm function scoped";
  let functionLet = "I'm also function scoped";
  
  if (true) {
    // Block Scope
    var blockVar = "I'm NOT block scoped (function scoped)";
    let blockLet = "I'm block scoped";
    const blockConst = "I'm also block scoped";
    
    console.log(blockLet); // Works
  }
  
  console.log(blockVar);  // Works (var is function-scoped)
  // console.log(blockLet); // Error! (let is block-scoped)
}

// Scope chain
let outer = "outer";

function outerFunc() {
  let middle = "middle";
  
  function innerFunc() {
    let inner = "inner";
    console.log(outer);  // Can access outer
    console.log(middle); // Can access middle
    console.log(inner);  // Can access inner
  }
  
  innerFunc();
  // console.log(inner); // Error! Cannot access inner
}
```

**Key Points:**
- Scope determines variable accessibility
- Inner scopes can access outer scopes
- Outer scopes cannot access inner scopes
- var is function-scoped, let/const are block-scoped

---

## Closures

### What is a closure and why is it useful?

**Answer:**

A closure is a function that has access to variables in its outer (enclosing) lexical scope, even after the outer function has returned.

**Code Example:**
```javascript
// Basic closure
function outerFunction(outerVariable) {
  return function innerFunction(innerVariable) {
    console.log('Outer:', outerVariable);
    console.log('Inner:', innerVariable);
  };
}

const newFunction = outerFunction('outside');
newFunction('inside');
// Outer: outside
// Inner: inside

// Practical example: Counter
function createCounter() {
  let count = 0; // Private variable
  
  return {
    increment: function() {
      count++;
      return count;
    },
    decrement: function() {
      count--;
      return count;
    },
    getCount: function() {
      return count;
    }
  };
}

const counter = createCounter();
console.log(counter.increment()); // 1
console.log(counter.increment()); // 2
console.log(counter.decrement()); // 1
console.log(counter.getCount());  // 1
// console.log(counter.count);    // undefined (private)

// Another example: Function factory
function createMultiplier(multiplier) {
  return function(number) {
    return number * multiplier;
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15

// Closure in loops (classic interview question)
// Wrong way:
for (var i = 1; i <= 3; i++) {
  setTimeout(function() {
    console.log(i); // Prints 4, 4, 4
  }, i * 1000);
}

// Correct way 1: IIFE
for (var i = 1; i <= 3; i++) {
  (function(i) {
    setTimeout(function() {
      console.log(i); // Prints 1, 2, 3
    }, i * 1000);
  })(i);
}

// Correct way 2: let (block scope)
for (let i = 1; i <= 3; i++) {
  setTimeout(function() {
    console.log(i); // Prints 1, 2, 3
  }, i * 1000);
}
```

**Key Points:**
- Closures maintain reference to outer scope
- Enable data privacy and encapsulation
- Used in callbacks, event handlers, higher-order functions
- Can cause memory leaks if not careful

---

## Lexical Scope

### What is lexical scope?

**Answer:**

Lexical scope means the scope is determined by where functions are defined, not where they're called.

**Code Example:**
```javascript
const value = "global";

function outer() {
  const value = "outer";
  
  function inner() {
    console.log(value); // Looks up the scope chain
  }
  
  return inner;
}

const innerFunc = outer();
innerFunc(); // "outer" (not "global")

// Another example
function createGreeter(greeting) {
  return function(name) {
    console.log(`${greeting}, ${name}!`);
  };
}

const sayHello = createGreeter("Hello");
const sayHi = createGreeter("Hi");

sayHello("John"); // "Hello, John!"
sayHi("Jane");    // "Hi, Jane!"

// Lexical scope with arrow functions
const obj = {
  value: "object value",
  regularFunc: function() {
    console.log(this.value); // "object value"
  },
  arrowFunc: () => {
    console.log(this.value); // undefined (lexical this)
  }
};
```

**Key Points:**
- Scope is determined at write time, not runtime
- JavaScript uses lexical (static) scope
- Arrow functions have lexical `this`

---

## Module Pattern

### How do closures enable the module pattern?

**Answer:**

The module pattern uses closures to create private variables and public APIs.

**Code Example:**
```javascript
// Revealing Module Pattern
const Calculator = (function() {
  // Private variables and functions
  let result = 0;
  
  function validate(num) {
    return typeof num === 'number';
  }
  
  // Public API
  return {
    add: function(num) {
      if (validate(num)) {
        result += num;
      }
      return this;
    },
    subtract: function(num) {
      if (validate(num)) {
        result -= num;
      }
      return this;
    },
    multiply: function(num) {
      if (validate(num)) {
        result *= num;
      }
      return this;
    },
    getResult: function() {
      return result;
    },
    reset: function() {
      result = 0;
      return this;
    }
  };
})();

// Usage
Calculator.add(10).subtract(3).multiply(2);
console.log(Calculator.getResult()); // 14
// console.log(Calculator.result);   // undefined (private)

// ES6 Module Pattern
class CounterModule {
  #count = 0; // Private field
  
  increment() {
    this.#count++;
    return this.#count;
  }
  
  decrement() {
    this.#count--;
    return this.#count;
  }
  
  get value() {
    return this.#count;
  }
}

const counter = new CounterModule();
console.log(counter.increment()); // 1
console.log(counter.value);       // 1
// console.log(counter.#count);   // Error (private)
```

**Key Points:**
- IIFE creates private scope
- Return object exposes public API
- Private members are truly private
- Method chaining possible with `return this`

---

## Common Pitfalls

### What are common closure mistakes?

**Answer:**

**Code Example:**
```javascript
// Pitfall 1: Shared variable in loop
function createFunctions() {
  const functions = [];
  
  for (var i = 0; i < 3; i++) {
    functions.push(function() {
      console.log(i);
    });
  }
  
  return functions;
}

const funcs = createFunctions();
funcs[0](); // 3 (not 0!)
funcs[1](); // 3 (not 1!)
funcs[2](); // 3 (not 2!)

// Solution 1: IIFE
function createFunctions() {
  const functions = [];
  
  for (var i = 0; i < 3; i++) {
    functions.push((function(index) {
      return function() {
        console.log(index);
      };
    })(i));
  }
  
  return functions;
}

// Solution 2: let
function createFunctions() {
  const functions = [];
  
  for (let i = 0; i < 3; i++) {
    functions.push(function() {
      console.log(i);
    });
  }
  
  return functions;
}

// Pitfall 2: Memory leaks
function createHugeObject() {
  const hugeArray = new Array(1000000).fill('data');
  
  return function() {
    // Even if we don't use hugeArray, it's kept in memory
    console.log('Function called');
  };
}

// Better: explicitly null out unused references
function createHugeObject() {
  let hugeArray = new Array(1000000).fill('data');
  const result = hugeArray[0];
  hugeArray = null; // Allow garbage collection
  
  return function() {
    console.log(result);
  };
}

// Pitfall 3: Accidental global variables
function oops() {
  // Forgot 'var', 'let', or 'const'
  leakedVariable = "I'm global!";
}

oops();
console.log(leakedVariable); // "I'm global!"

// Solution: use strict mode
'use strict';
function oops() {
  leakedVariable = "Error!"; // ReferenceError
}
```

**Key Points:**
- Be careful with closures in loops
- Consider memory implications
- Use `let`/`const` instead of `var`
- Enable strict mode to catch errors

---

## Interview Practice Questions

1. What is a closure and how does it work?
2. Explain the difference between scope and closure
3. How do you create private variables in JavaScript?
4. What's the output of functions created in a loop with var?
5. How do closures affect memory?
6. Implement a function that creates a counter using closures
7. What is the module pattern?
8. How do you avoid memory leaks with closures?

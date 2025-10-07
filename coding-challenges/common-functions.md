# Common Function Implementations

## Table of Contents
- [Debounce](#debounce)
- [Throttle](#throttle)
- [Deep Clone](#deep-clone)
- [Flatten Array](#flatten-array)
- [Curry](#curry)
- [Promise.all](#promiseall)

---

## Debounce

### Implement a debounce function

**Question:** Create a function that delays the execution until after a specified time has passed since the last invocation.

**Solution:**

```javascript
function debounce(func, delay) {
  let timeoutId;
  
  return function(...args) {
    // Clear the previous timeout
    clearTimeout(timeoutId);
    
    // Set a new timeout
    timeoutId = setTimeout(() => {
      func.apply(this, args);
    }, delay);
  };
}

// Usage
const search = debounce((query) => {
  console.log('Searching for:', query);
  // API call here
}, 300);

search('react'); // Waits 300ms
search('react hooks'); // Cancels previous, waits 300ms

// With immediate execution option
function debounceImmediate(func, delay, immediate = false) {
  let timeoutId;
  
  return function(...args) {
    const callNow = immediate && !timeoutId;
    
    clearTimeout(timeoutId);
    
    timeoutId = setTimeout(() => {
      timeoutId = null;
      if (!immediate) {
        func.apply(this, args);
      }
    }, delay);
    
    if (callNow) {
      func.apply(this, args);
    }
  };
}

// Test
const log = debounce((msg) => console.log(msg), 1000);
log('a');
log('b');
log('c'); // Only 'c' will be logged after 1 second
```

**Use Cases:**
- Search input
- Resize events
- Scroll events
- Form validation

---

## Throttle

### Implement a throttle function

**Question:** Create a function that ensures a function is called at most once in a specified time period.

**Solution:**

```javascript
function throttle(func, limit) {
  let inThrottle;
  
  return function(...args) {
    if (!inThrottle) {
      func.apply(this, args);
      inThrottle = true;
      
      setTimeout(() => {
        inThrottle = false;
      }, limit);
    }
  };
}

// Usage
const handleScroll = throttle(() => {
  console.log('Scroll event');
}, 1000);

window.addEventListener('scroll', handleScroll);

// Advanced throttle with trailing call
function throttleAdvanced(func, limit) {
  let inThrottle;
  let lastFunc;
  let lastRan;
  
  return function(...args) {
    if (!inThrottle) {
      func.apply(this, args);
      lastRan = Date.now();
      inThrottle = true;
    } else {
      clearTimeout(lastFunc);
      lastFunc = setTimeout(() => {
        if (Date.now() - lastRan >= limit) {
          func.apply(this, args);
          lastRan = Date.now();
        }
      }, limit - (Date.now() - lastRan));
    }
  };
}

// Test
let count = 0;
const throttled = throttle(() => console.log(++count), 1000);

setInterval(() => throttled(), 100);
// Logs: 1, 2, 3, 4... (once per second)
```

**Difference from Debounce:**
- **Debounce**: Waits for silence, then executes
- **Throttle**: Executes at regular intervals

---

## Deep Clone

### Implement a deep clone function

**Question:** Create a function that creates a deep copy of an object or array.

**Solution:**

```javascript
function deepClone(obj, hash = new WeakMap()) {
  // Handle null or primitive types
  if (obj === null || typeof obj !== 'object') {
    return obj;
  }
  
  // Handle circular references
  if (hash.has(obj)) {
    return hash.get(obj);
  }
  
  // Handle Date
  if (obj instanceof Date) {
    return new Date(obj);
  }
  
  // Handle RegExp
  if (obj instanceof RegExp) {
    return new RegExp(obj);
  }
  
  // Handle Array
  if (Array.isArray(obj)) {
    const arrCopy = [];
    hash.set(obj, arrCopy);
    
    obj.forEach((item, index) => {
      arrCopy[index] = deepClone(item, hash);
    });
    
    return arrCopy;
  }
  
  // Handle Object
  const objCopy = {};
  hash.set(obj, objCopy);
  
  Object.keys(obj).forEach(key => {
    objCopy[key] = deepClone(obj[key], hash);
  });
  
  return objCopy;
}

// Test
const original = {
  name: 'John',
  age: 30,
  address: {
    city: 'New York',
    zip: 10001
  },
  hobbies: ['reading', 'coding'],
  birthDate: new Date('1990-01-01')
};

const cloned = deepClone(original);
cloned.address.city = 'San Francisco';

console.log(original.address.city); // 'New York'
console.log(cloned.address.city);   // 'San Francisco'

// Handle circular reference
const circular = { a: 1 };
circular.self = circular;
const clonedCircular = deepClone(circular);
console.log(clonedCircular.self === clonedCircular); // true
```

---

## Flatten Array

### Flatten a nested array

**Question:** Convert a nested array into a single-level array.

**Solution:**

```javascript
// Method 1: Recursive
function flattenRecursive(arr) {
  const result = [];
  
  for (let item of arr) {
    if (Array.isArray(item)) {
      result.push(...flattenRecursive(item));
    } else {
      result.push(item);
    }
  }
  
  return result;
}

// Method 2: Reduce
function flattenReduce(arr) {
  return arr.reduce((acc, item) => {
    return acc.concat(
      Array.isArray(item) ? flattenReduce(item) : item
    );
  }, []);
}

// Method 3: Iterative with stack
function flattenIterative(arr) {
  const stack = [...arr];
  const result = [];
  
  while (stack.length) {
    const item = stack.pop();
    
    if (Array.isArray(item)) {
      stack.push(...item);
    } else {
      result.unshift(item);
    }
  }
  
  return result;
}

// Method 4: Using flat() (ES2019)
function flattenBuiltIn(arr, depth = Infinity) {
  return arr.flat(depth);
}

// Test
const nested = [1, [2, 3], [4, [5, 6]], [[7]], 8];

console.log(flattenRecursive(nested));
// [1, 2, 3, 4, 5, 6, 7, 8]

// Flatten to specific depth
function flattenToDepth(arr, depth = 1) {
  if (depth === 0) return arr;
  
  return arr.reduce((acc, item) => {
    return acc.concat(
      Array.isArray(item)
        ? flattenToDepth(item, depth - 1)
        : item
    );
  }, []);
}

console.log(flattenToDepth(nested, 1));
// [1, 2, 3, 4, [5, 6], [7], 8]
```

---

## Curry

### Implement function currying

**Question:** Transform a function with multiple arguments into a sequence of functions each taking a single argument.

**Solution:**

```javascript
// Basic curry
function curry(fn) {
  return function curried(...args) {
    if (args.length >= fn.length) {
      return fn.apply(this, args);
    } else {
      return function(...nextArgs) {
        return curried.apply(this, args.concat(nextArgs));
      };
    }
  };
}

// Usage
function sum(a, b, c) {
  return a + b + c;
}

const curriedSum = curry(sum);

console.log(curriedSum(1)(2)(3));       // 6
console.log(curriedSum(1, 2)(3));       // 6
console.log(curriedSum(1)(2, 3));       // 6
console.log(curriedSum(1, 2, 3));       // 6

// Practical example
function multiply(a, b, c) {
  return a * b * c;
}

const curriedMultiply = curry(multiply);
const multiplyBy2 = curriedMultiply(2);
const multiplyBy2And3 = multiplyBy2(3);

console.log(multiplyBy2And3(4)); // 24 (2 * 3 * 4)

// Advanced curry with placeholder
const _ = Symbol('placeholder');

function curryAdvanced(fn) {
  return function curried(...args) {
    const hasPlaceholder = args.some(arg => arg === _);
    
    if (!hasPlaceholder && args.length >= fn.length) {
      return fn.apply(this, args);
    }
    
    return function(...nextArgs) {
      const newArgs = args.map(arg =>
        arg === _ && nextArgs.length ? nextArgs.shift() : arg
      );
      return curried(...newArgs, ...nextArgs);
    };
  };
}

const curriedSum2 = curryAdvanced(sum);
console.log(curriedSum2(_, 2)(1, 3)); // 6
```

---

## Promise.all

### Implement Promise.all

**Question:** Implement the Promise.all functionality.

**Solution:**

```javascript
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('Argument must be an array'));
    }
    
    const results = [];
    let completed = 0;
    
    if (promises.length === 0) {
      return resolve(results);
    }
    
    promises.forEach((promise, index) => {
      // Handle non-promise values
      Promise.resolve(promise)
        .then(value => {
          results[index] = value;
          completed++;
          
          if (completed === promises.length) {
            resolve(results);
          }
        })
        .catch(error => {
          reject(error);
        });
    });
  });
}

// Test
const p1 = Promise.resolve(3);
const p2 = new Promise(resolve => setTimeout(() => resolve(42), 100));
const p3 = Promise.resolve('foo');

promiseAll([p1, p2, p3]).then(values => {
  console.log(values); // [3, 42, 'foo']
});

// Implement Promise.race
function promiseRace(promises) {
  return new Promise((resolve, reject) => {
    if (!Array.isArray(promises)) {
      return reject(new TypeError('Argument must be an array'));
    }
    
    promises.forEach(promise => {
      Promise.resolve(promise)
        .then(resolve)
        .catch(reject);
    });
  });
}

// Implement Promise.allSettled
function promiseAllSettled(promises) {
  return Promise.all(
    promises.map(promise =>
      Promise.resolve(promise)
        .then(value => ({ status: 'fulfilled', value }))
        .catch(reason => ({ status: 'rejected', reason }))
    )
  );
}

// Test
const p4 = Promise.reject('error');
promiseAllSettled([p1, p2, p4]).then(results => {
  console.log(results);
  // [
  //   { status: 'fulfilled', value: 3 },
  //   { status: 'fulfilled', value: 42 },
  //   { status: 'rejected', reason: 'error' }
  // ]
});
```

**Key Points:**
- Handle both promises and non-promise values
- Maintain order of results
- Fail fast on first rejection
- Handle edge cases (empty array, non-array)

---

## Additional Challenges

Try implementing these on your own:
- `bind()` function
- `pipe()` and `compose()` functions
- `memoize()` function
- `retry()` function for failed promises
- `promiseWithTimeout()`
- `deepEqual()` function
- `groupBy()` function
- `chunk()` array function

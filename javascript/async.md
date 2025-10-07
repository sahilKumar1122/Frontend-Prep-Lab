# Asynchronous JavaScript

## Table of Contents
- [Callbacks](#callbacks)
- [Promises](#promises)
- [Async/Await](#asyncawait)
- [Error Handling](#error-handling)
- [Common Patterns](#common-patterns)

---

## Callbacks

### What is a callback and what is callback hell?

**Answer:**

A callback is a function passed as an argument to another function, to be executed later.

**Code Example:**
```javascript
// Simple callback
function fetchData(callback) {
  setTimeout(() => {
    callback("Data received");
  }, 1000);
}

fetchData((data) => {
  console.log(data); // "Data received"
});

// Callback Hell (Pyramid of Doom)
fetchUser(userId, (user) => {
  fetchPosts(user.id, (posts) => {
    fetchComments(posts[0].id, (comments) => {
      fetchReplies(comments[0].id, (replies) => {
        // This keeps going...
        console.log(replies);
      });
    });
  });
});
```

**Key Points:**
- Callbacks are fundamental to async JavaScript
- Callback hell makes code hard to read and maintain
- Promises and async/await solve callback hell

---

## Promises

### Explain Promises and their states

**Answer:**

A Promise represents the eventual completion (or failure) of an asynchronous operation.

**States:**
1. **Pending** - initial state
2. **Fulfilled** - operation completed successfully
3. **Rejected** - operation failed

**Code Example:**
```javascript
// Creating a Promise
const myPromise = new Promise((resolve, reject) => {
  const success = true;
  
  setTimeout(() => {
    if (success) {
      resolve("Operation successful");
    } else {
      reject("Operation failed");
    }
  }, 1000);
});

// Consuming a Promise
myPromise
  .then(result => {
    console.log(result); // "Operation successful"
    return "Next step";
  })
  .then(result => {
    console.log(result); // "Next step"
  })
  .catch(error => {
    console.error(error);
  })
  .finally(() => {
    console.log("Cleanup");
  });

// Solving callback hell with Promises
fetchUser(userId)
  .then(user => fetchPosts(user.id))
  .then(posts => fetchComments(posts[0].id))
  .then(comments => fetchReplies(comments[0].id))
  .then(replies => console.log(replies))
  .catch(error => console.error(error));
```

**Key Points:**
- Promises are immutable once settled
- `.then()` returns a new Promise (chainable)
- `.catch()` catches errors in the chain
- `.finally()` executes regardless of outcome

---

## Async/Await

### What is async/await and how does it work?

**Answer:**

Async/await is syntactic sugar over Promises, making asynchronous code look synchronous.

**Code Example:**
```javascript
// Basic async function
async function fetchData() {
  return "Data"; // Automatically wrapped in Promise
}

fetchData().then(data => console.log(data));

// Using await
async function getUserData(userId) {
  try {
    const user = await fetchUser(userId);
    const posts = await fetchPosts(user.id);
    const comments = await fetchComments(posts[0].id);
    return comments;
  } catch (error) {
    console.error("Error:", error);
  }
}

// Multiple concurrent requests
async function fetchMultiple() {
  try {
    // Sequential (slow)
    const user1 = await fetchUser(1);
    const user2 = await fetchUser(2);
    
    // Concurrent (fast)
    const [user3, user4] = await Promise.all([
      fetchUser(3),
      fetchUser(4)
    ]);
    
    return { user1, user2, user3, user4 };
  } catch (error) {
    console.error(error);
  }
}

// Top-level await (ES2022)
const data = await fetchData();
console.log(data);
```

**Key Points:**
- `async` function always returns a Promise
- `await` can only be used inside async functions (or top-level)
- Use `Promise.all()` for concurrent operations
- Error handling with try/catch

---

## Error Handling

### How do you handle errors in async code?

**Answer:**

Different approaches for callbacks, Promises, and async/await.

**Code Example:**
```javascript
// 1. Callbacks - error-first pattern
function fetchData(callback) {
  setTimeout(() => {
    const error = null;
    const data = "Success";
    callback(error, data);
  }, 1000);
}

fetchData((error, data) => {
  if (error) {
    console.error(error);
    return;
  }
  console.log(data);
});

// 2. Promises - .catch()
fetchUser(123)
  .then(user => processUser(user))
  .then(result => console.log(result))
  .catch(error => {
    console.error("Error:", error);
  });

// 3. Async/Await - try/catch
async function getUserInfo(userId) {
  try {
    const user = await fetchUser(userId);
    const posts = await fetchPosts(user.id);
    return { user, posts };
  } catch (error) {
    console.error("Failed to get user info:", error);
    throw error; // Re-throw if needed
  }
}

// 4. Global error handlers
window.addEventListener('unhandledrejection', (event) => {
  console.error('Unhandled promise rejection:', event.reason);
});

// 5. Custom error wrapper
async function safeAsync(fn) {
  try {
    const data = await fn();
    return [null, data];
  } catch (error) {
    return [error, null];
  }
}

// Usage
const [error, data] = await safeAsync(() => fetchUser(123));
if (error) {
  console.error(error);
} else {
  console.log(data);
}
```

**Key Points:**
- Always handle errors in async operations
- Use try/catch with async/await
- Use .catch() with Promises
- Consider global error handlers for unhandled rejections

---

## Common Patterns

### Promise.all, Promise.race, Promise.allSettled, Promise.any

**Answer:**

Different methods for handling multiple Promises.

**Code Example:**
```javascript
const promise1 = Promise.resolve(3);
const promise2 = new Promise(resolve => setTimeout(() => resolve(42), 100));
const promise3 = Promise.reject("Error");

// Promise.all - waits for all, fails if any fails
Promise.all([promise1, promise2])
  .then(results => console.log(results)) // [3, 42]
  .catch(error => console.error(error));

Promise.all([promise1, promise3])
  .then(results => console.log(results))
  .catch(error => console.error(error)); // "Error"

// Promise.race - returns first settled (resolved or rejected)
Promise.race([promise1, promise2])
  .then(result => console.log(result)); // 3 (fastest)

// Promise.allSettled - waits for all, never rejects
Promise.allSettled([promise1, promise2, promise3])
  .then(results => {
    console.log(results);
    // [
    //   { status: 'fulfilled', value: 3 },
    //   { status: 'fulfilled', value: 42 },
    //   { status: 'rejected', reason: 'Error' }
    // ]
  });

// Promise.any - returns first fulfilled (ignores rejections)
Promise.any([promise3, promise1, promise2])
  .then(result => console.log(result)); // 3 (first fulfilled)

// Practical example: timeout wrapper
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) => {
    setTimeout(() => reject(new Error('Timeout')), ms);
  });
  return Promise.race([promise, timeout]);
}

// Usage
withTimeout(fetchUser(123), 5000)
  .then(user => console.log(user))
  .catch(error => console.error(error));
```

**Key Points:**
- `Promise.all()` - fails fast on first rejection
- `Promise.race()` - first to settle wins
- `Promise.allSettled()` - wait for all, get all results
- `Promise.any()` - first to fulfill wins

---

## Interview Practice Questions

1. What is the difference between callbacks, promises, and async/await?
2. How do you handle errors in async functions?
3. Explain the difference between Promise.all and Promise.allSettled
4. What happens when you don't await an async function?
5. Can you use await outside of an async function?
6. How do you make multiple API calls in parallel?
7. What is a microtask queue?
8. How does setTimeout differ from Promise in the event loop?

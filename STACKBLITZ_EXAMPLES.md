---
title: "Interactive Code Examples"
description: "Live, editable code examples for Angular, React, and JavaScript concepts"
category: "Resources"
tags: ["stackblitz", "examples", "interactive", "playground"]
featured: true
---

# 🎮 Interactive Code Examples

Live, editable code examples to practice and experiment with concepts covered in this guide.

---

## 📖 How to Use

1. **Click the "Open in StackBlitz" button** to view the example
2. **Edit the code** directly in the browser
3. **See changes live** in the preview pane
4. **Fork and save** your modifications (requires StackBlitz account)

---

## 🅰️ Angular Examples

### Core Concepts

#### 1. Component Basics
**Concepts:** Components, Templates, Data Binding

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-component-basics?file=src/app/user-card/user-card.component.ts)

```typescript
// Demonstrates:
// - @Component decorator
// - Property binding
// - Event binding
// - Template syntax
```

**Topics Covered:**
- Component creation
- Interpolation `{{ }}`
- Property binding `[property]`
- Event binding `(event)`
- Two-way binding `[(ngModel)]`

---

#### 2. Lifecycle Hooks
**Concepts:** Component Lifecycle, ngOnInit, ngOnDestroy

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-lifecycle-hooks-demo?file=src/app/lifecycle-demo/lifecycle-demo.component.ts)

```typescript
// Demonstrates:
// - ngOnInit for initialization
// - ngOnChanges for input tracking
// - ngOnDestroy for cleanup
// - Memory leak prevention
```

**Topics Covered:**
- Lifecycle hook execution order
- Cleanup patterns
- Memory leak prevention
- Input change detection

---

#### 3. Change Detection - OnPush Strategy
**Concepts:** Performance, ChangeDetectionStrategy

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-onpush-demo?file=src/app/performance/performance.component.ts)

```typescript
// Demonstrates:
// - OnPush change detection
// - Immutable updates
// - Performance comparison
// - markForCheck() usage
```

**Topics Covered:**
- Default vs OnPush strategy
- 90% performance improvement
- Immutable data patterns
- Manual change detection

---

#### 4. Dependency Injection
**Concepts:** Services, Providers, Injection Tokens

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-di-demo?file=src/app/services/user.service.ts)

```typescript
// Demonstrates:
// - providedIn: 'root'
// - Service injection
// - InjectionToken
// - Hierarchical injectors
```

**Topics Covered:**
- Service creation
- Provider scopes
- Custom injection tokens
- Dependency resolution

---

#### 5. RxJS Operators
**Concepts:** Observables, switchMap, debounceTime

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-rxjs-operators?file=src/app/search/search.component.ts)

```typescript
// Demonstrates:
// - switchMap for search
// - debounceTime for rate limiting
// - distinctUntilChanged
// - Error handling
```

**Topics Covered:**
- Search implementation
- API call optimization
- Operator chaining
- Memory management

---

#### 6. Reactive Forms
**Concepts:** FormBuilder, Validators, Dynamic Forms

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-reactive-forms?file=src/app/user-form/user-form.component.ts)

```typescript
// Demonstrates:
// - FormGroup and FormControl
// - Built-in validators
// - Custom validators
// - Dynamic form arrays
```

**Topics Covered:**
- Form creation with FormBuilder
- Validation patterns
- Cross-field validation
- FormArray for dynamic fields

---

#### 7. Router Guards
**Concepts:** Navigation, CanActivate, Route Protection

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-route-guards?file=src/app/guards/auth.guard.ts)

```typescript
// Demonstrates:
// - Functional guards
// - CanActivate implementation
// - Route protection
// - Navigation blocking
```

**Topics Covered:**
- Authentication guard
- Authorization patterns
- Redirect logic
- Guard composition

---

#### 8. Standalone Components (Angular 18)
**Concepts:** Modern Angular, Standalone, Signals

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-standalone-demo?file=src/main.ts)

```typescript
// Demonstrates:
// - Standalone components
// - bootstrapApplication
// - Signal-based state
// - Simplified architecture
```

**Topics Covered:**
- No NgModule needed
- Direct imports
- Signals API
- Modern patterns

---

### Advanced Examples

#### 9. NgRx State Management
**Concepts:** Redux Pattern, Actions, Reducers, Effects

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-ngrx-counter?file=src/app/store/counter.reducer.ts)

```typescript
// Demonstrates:
// - Store setup
// - Actions and reducers
// - Selectors
// - Effects for side effects
```

**Topics Covered:**
- Unidirectional data flow
- Immutable state updates
- Effect patterns
- DevTools integration

---

#### 10. Virtual Scrolling (CDK)
**Concepts:** Performance, Large Lists

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/angular-virtual-scroll?file=src/app/list/virtual-list.component.ts)

```typescript
// Demonstrates:
// - CDK Virtual Scroll
// - 10,000+ items
// - 99.85% DOM reduction
// - Smooth scrolling
```

**Topics Covered:**
- Virtual scrolling setup
- Performance benefits
- Dynamic item heights
- Viewport configuration

---

## ⚛️ React Examples

### Core Concepts

#### 11. useState Hook
**Concepts:** State Management, Hooks

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-usestate-demo?file=src/Counter.jsx)

```jsx
// Demonstrates:
// - useState basics
// - Functional updates
// - Multiple state variables
// - State with objects
```

**Topics Covered:**
- State initialization
- Updater functions
- Lazy initialization
- State patterns

---

#### 12. useEffect Hook
**Concepts:** Side Effects, Lifecycle

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-useeffect-demo?file=src/DataFetcher.jsx)

```jsx
// Demonstrates:
// - useEffect basics
// - Dependency array
// - Cleanup functions
// - Data fetching
```

**Topics Covered:**
- Effect execution
- Dependency management
- Memory leak prevention
- API integration

---

#### 13. Custom Hooks
**Concepts:** Reusable Logic, Hook Composition

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-custom-hooks?file=src/hooks/useFetch.js)

```jsx
// Demonstrates:
// - useFetch custom hook
// - useLocalStorage
// - useWindowSize
// - Hook patterns
```

**Topics Covered:**
- Hook extraction
- Reusability patterns
- State encapsulation
- Hook composition

---

#### 14. useContext & Context API
**Concepts:** Global State, Prop Drilling

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-context-demo?file=src/ThemeContext.jsx)

```jsx
// Demonstrates:
// - Context creation
// - Provider pattern
// - useContext hook
// - Theme switching
```

**Topics Covered:**
- Avoiding prop drilling
- Context best practices
- Multiple contexts
- Performance considerations

---

#### 15. useMemo & useCallback
**Concepts:** Performance Optimization, Memoization

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-memo-demo?file=src/ExpensiveComponent.jsx)

```jsx
// Demonstrates:
// - useMemo for values
// - useCallback for functions
// - React.memo for components
// - Performance comparison
```

**Topics Covered:**
- When to memoize
- Dependency arrays
- Performance profiling
- Optimization patterns

---

#### 16. useReducer Hook
**Concepts:** Complex State, Reducer Pattern

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-usereducer-demo?file=src/TodoList.jsx)

```jsx
// Demonstrates:
// - useReducer pattern
// - Complex state updates
// - Action dispatch
// - Reducer composition
```

**Topics Covered:**
- Reducer function
- Action types
- State immutability
- vs useState comparison

---

#### 17. React Router
**Concepts:** Navigation, Routing, Guards

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-router-demo?file=src/App.jsx)

```jsx
// Demonstrates:
// - Route configuration
// - Navigation
// - Protected routes
// - URL parameters
```

**Topics Covered:**
- BrowserRouter setup
- Route definition
- useNavigate hook
- useParams hook

---

#### 18. Form Handling
**Concepts:** Controlled Components, Validation

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-forms-demo?file=src/RegistrationForm.jsx)

```jsx
// Demonstrates:
// - Controlled inputs
// - Form validation
// - Error handling
// - Submit handling
```

**Topics Covered:**
- Controlled vs uncontrolled
- Input state management
- Validation patterns
- Form libraries (React Hook Form)

---

### Advanced Examples

#### 19. React 18 - useTransition
**Concepts:** Concurrent Features, Priority Updates

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-usetransition-demo?file=src/SearchWithTransition.jsx)

```jsx
// Demonstrates:
// - useTransition hook
// - Urgent vs non-urgent updates
// - Pending state
// - Responsive UI
```

**Topics Covered:**
- Concurrent rendering
- Update prioritization
- User experience optimization
- isPending indicator

---

#### 20. Error Boundaries
**Concepts:** Error Handling, Fallback UI

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/react-error-boundary?file=src/ErrorBoundary.jsx)

```jsx
// Demonstrates:
// - Error boundary class
// - Error catching
// - Fallback UI
// - Error logging
```

**Topics Covered:**
- componentDidCatch
- getDerivedStateFromError
- Error isolation
- Recovery patterns

---

## 🟨 JavaScript Examples

### Core Concepts

#### 21. Closures
**Concepts:** Scope, Lexical Environment

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-closures-demo?file=index.js)

```javascript
// Demonstrates:
// - Closure creation
// - Private variables
// - Function factories
// - Counter pattern
```

**Topics Covered:**
- Lexical scoping
- Data privacy
- Module pattern
- Practical use cases

---

#### 22. Promises & Async/Await
**Concepts:** Asynchronous Programming

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-async-demo?file=index.js)

```javascript
// Demonstrates:
// - Promise creation
// - Promise chaining
// - async/await syntax
// - Error handling
```

**Topics Covered:**
- Promise states
- .then() vs async/await
- Promise.all()
- Error propagation

---

#### 23. Event Loop & Microtasks
**Concepts:** Execution Model, Task Queue

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-event-loop-demo?file=index.js)

```javascript
// Demonstrates:
// - Call stack
// - Microtask queue
// - Macrotask queue
// - Execution order
```

**Topics Covered:**
- setTimeout vs Promise
- Microtask priority
- Task scheduling
- Event loop phases

---

#### 24. Array Methods
**Concepts:** Functional Programming, Iteration

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-array-methods?file=index.js)

```javascript
// Demonstrates:
// - map, filter, reduce
// - find, some, every
// - flatMap, flat
// - Method chaining
```

**Topics Covered:**
- Transformation patterns
- Filtering strategies
- Accumulation
- Performance considerations

---

#### 25. Prototypal Inheritance
**Concepts:** Prototype Chain, OOP

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-prototypes-demo?file=index.js)

```javascript
// Demonstrates:
// - Prototype chain
// - Object.create()
// - Class syntax
// - Inheritance patterns
```

**Topics Covered:**
- __proto__ vs prototype
- Constructor functions
- ES6 classes
- Inheritance hierarchy

---

#### 26. Debounce & Throttle
**Concepts:** Performance, Rate Limiting

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-debounce-throttle?file=index.js)

```javascript
// Demonstrates:
// - Debounce implementation
// - Throttle implementation
// - Use case comparison
// - Performance impact
```

**Topics Covered:**
- Search optimization
- Scroll handling
- API rate limiting
- Memory efficiency

---

#### 27. ES6+ Features
**Concepts:** Modern JavaScript, Syntax

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-es6-features?file=index.js)

```javascript
// Demonstrates:
// - Destructuring
// - Spread/Rest
// - Arrow functions
// - Template literals
```

**Topics Covered:**
- Modern syntax
- Code simplification
- Best practices
- Browser compatibility

---

#### 28. Module Patterns
**Concepts:** Code Organization, Encapsulation

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-module-patterns?file=index.js)

```javascript
// Demonstrates:
// - Module pattern
// - Revealing module
// - ES6 modules
// - Singleton pattern
```

**Topics Covered:**
- Code organization
- Private/public members
- Import/export
- Module benefits

---

### Advanced Examples

#### 29. Custom Promise Implementation
**Concepts:** Promises Internals

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-custom-promise?file=MyPromise.js)

```javascript
// Demonstrates:
// - Promise internals
// - State machine
// - then() implementation
// - Promise chaining
```

**Topics Covered:**
- Promise states
- Resolution mechanism
- Thenable chain
- Error propagation

---

#### 30. Web Workers
**Concepts:** Multi-threading, Performance

[![Open in StackBlitz](https://developer.stackblitz.com/img/open_in_stackblitz.svg)](https://stackblitz.com/edit/js-web-workers?file=index.js)

```javascript
// Demonstrates:
// - Worker creation
// - Message passing
// - Heavy computation
// - UI responsiveness
```

**Topics Covered:**
- Background processing
- Thread communication
- Use cases
- Limitations

---

## 🎯 Quick Start Templates

### Blank Templates (Start from Scratch)

- 🅰️ **[Angular Blank](https://stackblitz.com/fork/angular)** - Empty Angular project
- ⚛️ **[React Blank](https://stackblitz.com/fork/react)** - Empty React project
- 🟨 **[JavaScript Vanilla](https://stackblitz.com/fork/js)** - Plain JavaScript

### Interview Practice Templates

- 📝 **[Angular Interview Workspace](https://stackblitz.com/edit/angular-interview-prep)** - All examples in one project
- 📝 **[React Interview Workspace](https://stackblitz.com/edit/react-interview-prep)** - Complete React examples
- 📝 **[JavaScript Interview Workspace](https://stackblitz.com/edit/js-interview-prep)** - JS concepts collection

---

## 💡 Tips for Using StackBlitz

### For Learning:
1. **Read the code first** - Understand before editing
2. **Break things** - Best way to learn
3. **Add console.logs** - See execution flow
4. **Check the console** - Watch for errors

### For Practice:
1. **Fork examples** - Create your version
2. **Modify incrementally** - One change at a time
3. **Compare approaches** - Try different solutions
4. **Share with others** - Get feedback

### For Interviews:
1. **Practice live coding** - Simulate interview environment
2. **Explain out loud** - Practice talking through code
3. **Time yourself** - Build speed and confidence
4. **Review common patterns** - Recognize frequently used solutions

---

## 🔗 Additional Resources

### Official Documentation
- [StackBlitz Docs](https://developer.stackblitz.com/docs)
- [Angular StackBlitz](https://angular.io/guide/setup-local#stackblitz)
- [React Beta Docs](https://react.dev/learn)

### Video Tutorials
- [StackBlitz Tutorial Series](https://www.youtube.com/stackblitz)
- [Live Coding Sessions](https://www.youtube.com/results?search_query=stackblitz+tutorial)

### Community Examples
- [StackBlitz Community Projects](https://stackblitz.com/search?q=angular)
- [GitHub Examples Repository](../coding-challenges/)

---

## 🤝 Contributing Examples

Want to add more examples? See [CONTRIBUTING.md](../CONTRIBUTING.md)

**Example Contribution Checklist:**
- [ ] Clear, focused example (one concept)
- [ ] Well-commented code
- [ ] Working demo
- [ ] Description and learning outcomes
- [ ] Link to related documentation

---

**Last Updated:** October 2025  
**Total Examples:** 30+ interactive playgrounds



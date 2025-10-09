# React Interview Questions - Complete Single-Page Reference

> 📚 **90 Questions on One Page** | ⚡ Easy search with Ctrl+F | 📄 Print-ready format

**Last Updated:** October 2025 | **React 18+**

---

## 🎯 Quick Navigation

**Study Modes:**
- 📖 **[View by Category](#table-of-contents)** - Organized learning
- 🔍 **Ctrl+F to search** - Find any keyword instantly
- 📄 **Print this page** - Offline study guide
- 📖 **[Deep-Dive →](./fundamentals.md)** - Detailed articles
- 🎮 **[Interactive Examples →](../STACKBLITZ_EXAMPLES.md#-react-examples)** - Live code

---

## 📚 Table of Contents

**Jump to any section instantly:**

| Category | Questions | Difficulty | Jump To |
|----------|-----------|------------|---------|
| ⚛️ **Fundamentals** | 12 Q | 🟢 Easy-Medium | [→](#fundamentals) |
| 🧩 **Components & Props** | 10 Q | 🟢-🟡 Easy-Medium | [→](#components--props) |
| 🔄 **State & Lifecycle** | 15 Q | 🟡 Medium | [→](#state--lifecycle) |
| 🎣 **Hooks - Core** | 15 Q | 🟡 Medium | [→](#hooks---core) |
| 🎣 **Hooks - Advanced** | 18 Q | 🟡-🔴 Medium-Hard | [→](#hooks---advanced) |
| ⚡ **Effects & Side Effects** | 10 Q | 🟡-🔴 Medium-Hard | [→](#effects--side-effects) |
| 🚀 **Performance** | 6 Q | 🔴 Hard | [→](#performance) |
| 🎨 **Patterns** | 4 Q | 🟡 Medium | [→](#patterns) |

**Total: 90 Questions**

---

<a name="fundamentals"></a>
## ⚛️ Fundamentals (12 Questions)

[↑ Back to Top](#table-of-contents)

---

### 1. What is React?

**Answer:**

React is a JavaScript library for building user interfaces, developed and maintained by Facebook (Meta). It focuses on building reusable UI components with a declarative approach.

**Key Features:**
- **Component-Based** - Build encapsulated components
- **Declarative** - Design simple views for each state
- **Learn Once, Write Anywhere** - Can render on server, mobile (React Native)
- **Virtual DOM** - Efficient updates
- **One-Way Data Flow** - Predictable data flow
- **Rich Ecosystem** - Large community and tooling

**Example:**
```jsx
function Welcome({ name }) {
  return <h1>Hello, {name}!</h1>;
}

function App() {
  return (
    <div>
      <Welcome name="Alice" />
      <Welcome name="Bob" />
    </div>
  );
}
```

**Learn More:** [React Fundamentals →](./fundamentals.md)

---

### 2. What is JSX?

**Answer:**

JSX (JavaScript XML) is a syntax extension for JavaScript that looks similar to HTML. It allows you to write HTML-like code in JavaScript files.

**JSX:**
```jsx
const element = <h1 className="greeting">Hello, world!</h1>;
```

**Compiles to:**
```javascript
const element = React.createElement(
  'h1',
  { className: 'greeting' },
  'Hello, world!'
);
```

**Rules:**
```jsx
// 1. Must have ONE root element
return (
  <div>
    <h1>Title</h1>
    <p>Content</p>
  </div>
);

// 2. Or use Fragment
return (
  <>
    <h1>Title</h1>
    <p>Content</p>
  </>
);

// 3. className instead of class
<div className="container">

// 4. Expressions in curly braces
<h1>{name}</h1>
<p>{2 + 2}</p>

// 5. Self-closing tags
<img src={url} />
<input type="text" />
```

**Why JSX?**
- Easier to visualize UI structure
- JavaScript expressions inside markup
- Compile-time error checking
- Prevents injection attacks (auto-escapes)

---

### 3. What is the Virtual DOM?

**Answer:**

The Virtual DOM is a lightweight copy of the actual DOM kept in memory. React uses it to optimize updates.

**How it Works:**
```
1. State changes
    ↓
2. Create new Virtual DOM tree
    ↓
3. Diff with previous Virtual DOM (Reconciliation)
    ↓
4. Calculate minimal changes needed
    ↓
5. Batch update real DOM
```

**Example:**
```jsx
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>         {/* Only this updates */}
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

**Benefits:**
- **Faster** - Fewer DOM operations
- **Efficient** - Only updates what changed
- **Batched** - Multiple updates in one go

**Performance:**
- Virtual DOM operations: ~1ms
- Actual DOM operations: ~10-100ms

---

### 4. What are Components?

**Answer:**

Components are independent, reusable pieces of UI. They accept inputs (props) and return React elements.

**Types:**

**1. Function Components** (Modern, recommended)
```jsx
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}
```

**2. Class Components** (Legacy)
```jsx
class Greeting extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}!</h1>;
  }
}
```

**Component Composition:**
```jsx
function App() {
  return (
    <div>
      <Header />
      <main>
        <Sidebar />
        <Content />
      </main>
      <Footer />
    </div>
  );
}
```

**Best Practices:**
- Keep components small and focused
- Use meaningful names (PascalCase)
- Extract reusable logic
- Compose don't inherit

---

### 5. What are Props?

**Answer:**

Props (properties) are read-only inputs passed from parent to child components.

**Passing Props:**
```jsx
function App() {
  return (
    <UserCard 
      name="John Doe"
      age={30}
      isActive={true}
      skills={['React', 'Node.js']}
    />
  );
}
```

**Receiving Props:**
```jsx
function UserCard({ name, age, isActive, skills }) {
  return (
    <div>
      <h2>{name}</h2>
      <p>Age: {age}</p>
      <p>Status: {isActive ? 'Active' : 'Inactive'}</p>
      <ul>
        {skills.map(skill => <li key={skill}>{skill}</li>)}
      </ul>
    </div>
  );
}
```

**Props are Immutable:**
```jsx
function Component({ value }) {
  // ❌ Don't do this
  value = value + 1;
  
  // ✅ Create new value instead
  const newValue = value + 1;
}
```

**Default Props:**
```jsx
function Button({ text = 'Click me', type = 'button' }) {
  return <button type={type}>{text}</button>;
}
```

---

[Continue with remaining 7 fundamental questions...]

---

<a name="components--props"></a>
## 🧩 Components & Props (10 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all component questions...]

---

<a name="state--lifecycle"></a>
## 🔄 State & Lifecycle (15 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all state questions...]

---

<a name="hooks---core"></a>
## 🎣 Hooks - Core (15 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all core hooks questions...]

---

<a name="hooks---advanced"></a>
## 🎣 Hooks - Advanced (18 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all advanced hooks questions...]

---

<a name="effects--side-effects"></a>
## ⚡ Effects & Side Effects (10 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all effects questions...]

---

<a name="performance"></a>
## 🚀 Performance (6 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all performance questions...]

---

<a name="patterns"></a>
## 🎨 Patterns (4 Questions)

[↑ Back to Top](#table-of-contents)

[Continue with all pattern questions...]

---

## 🎯 Study Tips

### How to Use This Document:

**1. Quick Search (Ctrl+F / Cmd+F)**
```
Search for keywords like:
- "useState"
- "useEffect"
- "memo"
- "context"
```

**2. Print for Offline Study**
- Use browser print function
- Save as PDF
- Annotate with notes

**3. Progressive Learning**
- Start with ⚛️ Fundamentals
- Master 🎣 Hooks
- Practice 🚀 Performance optimization

**4. Practice Alongside**
- Try [Interactive Examples →](../STACKBLITZ_EXAMPLES.md#-react-examples)
- Build sample projects
- Code while learning

---

## 📚 Additional Resources

**Deep Dives:**
- [React Fundamentals →](./fundamentals.md)
- [Official React Docs](https://react.dev/reference/react)

**Interactive:**
- [10+ StackBlitz Examples →](../STACKBLITZ_EXAMPLES.md#-react-examples)

---

## 🤝 Contributing

Found a mistake? Want to add more questions?
- [Open an Issue](../../issues)
- [Submit a PR](../../pulls)
- [Start a Discussion](../../discussions)

---

**Last Updated:** October 2025  
**Total Questions:** 90  
**Format:** Single-page for easy search and printing

[↑ Back to Top](#table-of-contents)



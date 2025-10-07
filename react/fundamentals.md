# React Fundamentals

## Table of Contents
- [Core Concepts](#core-concepts)
- [Components](#components)
- [Props](#props)
- [State](#state)
- [Lifecycle](#lifecycle)
- [JSX](#jsx)

---

## Core Concepts

### What is React and why use it?

**Answer:**

React is a JavaScript library for building user interfaces, developed by Facebook.

**Key Features:**
- **Component-Based**: Build encapsulated components
- **Declarative**: Describe UI state, React updates DOM
- **Virtual DOM**: Efficient updates and rendering
- **One-Way Data Flow**: Predictable data flow
- **JSX**: JavaScript XML syntax

**Code Example:**
```javascript
import React from 'react';
import ReactDOM from 'react-dom/client';

// Simple component
function Welcome({ name }) {
  return <h1>Hello, {name}!</h1>;
}

// Render to DOM
const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<Welcome name="React" />);
```

**Benefits:**
- Reusable components
- Fast rendering with Virtual DOM
- Strong ecosystem and community
- Great developer experience
- SEO-friendly (with SSR)

---

## Components

### What are the different ways to create components?

**Answer:**

**1. Function Components (Modern)**
```javascript
// Basic function component
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}

// Arrow function component
const Greeting = ({ name }) => {
  return <h1>Hello, {name}!</h1>;
};

// With hooks
import { useState } from 'react';

function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>
        Increment
      </button>
    </div>
  );
}
```

**2. Class Components (Legacy)**
```javascript
import React, { Component } from 'react';

class Counter extends Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
  }
  
  increment = () => {
    this.setState({ count: this.state.count + 1 });
  }
  
  render() {
    return (
      <div>
        <p>Count: {this.state.count}</p>
        <button onClick={this.increment}>
          Increment
        </button>
      </div>
    );
  }
}
```

**Key Points:**
- Function components are now preferred
- Hooks enable state and lifecycle in functions
- Class components are legacy but still supported

---

## Props

### How do props work in React?

**Answer:**

Props (properties) are read-only data passed from parent to child components.

**Code Example:**
```javascript
// Parent component
function App() {
  return (
    <UserCard 
      name="John Doe"
      age={30}
      email="john@example.com"
      isActive={true}
    />
  );
}

// Child component
function UserCard({ name, age, email, isActive }) {
  return (
    <div className="user-card">
      <h2>{name}</h2>
      <p>Age: {age}</p>
      <p>Email: {email}</p>
      <p>Status: {isActive ? 'Active' : 'Inactive'}</p>
    </div>
  );
}

// Props with children
function Card({ title, children }) {
  return (
    <div className="card">
      <h3>{title}</h3>
      <div className="card-content">
        {children}
      </div>
    </div>
  );
}

// Usage
<Card title="My Card">
  <p>This is the content</p>
  <button>Click me</button>
</Card>

// Default props
function Button({ text = "Click", onClick }) {
  return <button onClick={onClick}>{text}</button>;
}

// Prop types (with prop-types library)
import PropTypes from 'prop-types';

Button.propTypes = {
  text: PropTypes.string,
  onClick: PropTypes.func.isRequired
};

// TypeScript props
interface ButtonProps {
  text?: string;
  onClick: () => void;
}

function Button({ text = "Click", onClick }: ButtonProps) {
  return <button onClick={onClick}>{text}</button>;
}
```

**Key Points:**
- Props are immutable (read-only)
- Pass data from parent to child
- Can pass any JavaScript value
- Use destructuring for cleaner code
- TypeScript recommended for type safety

---

## State

### What is state and how do you manage it?

**Answer:**

State is mutable data managed within a component.

**Code Example:**
```javascript
import { useState } from 'react';

// Simple state
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
      <button onClick={() => setCount(count - 1)}>-</button>
    </div>
  );
}

// Object state
function Form() {
  const [formData, setFormData] = useState({
    name: '',
    email: '',
    age: 0
  });
  
  const handleChange = (e) => {
    setFormData({
      ...formData,
      [e.target.name]: e.target.value
    });
  };
  
  return (
    <form>
      <input
        name="name"
        value={formData.name}
        onChange={handleChange}
      />
      <input
        name="email"
        value={formData.email}
        onChange={handleChange}
      />
    </form>
  );
}

// Functional updates
function Counter() {
  const [count, setCount] = useState(0);
  
  const increment = () => {
    // Correct way when new state depends on old state
    setCount(prevCount => prevCount + 1);
  };
  
  return <button onClick={increment}>{count}</button>;
}

// Multiple state updates
function Example() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');
  const [isActive, setIsActive] = useState(false);
  
  return (
    <div>
      <p>{count}</p>
      <input value={name} onChange={e => setName(e.target.value)} />
      <button onClick={() => setIsActive(!isActive)}>Toggle</button>
    </div>
  );
}
```

**Key Points:**
- State is local to component
- Updates are asynchronous
- Use functional updates when new state depends on old
- Never mutate state directly
- useState returns [value, setter]

---

## Lifecycle

### What are React lifecycle methods?

**Answer:**

Lifecycle methods let you run code at specific times in a component's life.

**Code Example:**
```javascript
// Class component lifecycle
class LifecycleDemo extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    console.log('1. Constructor');
  }
  
  static getDerivedStateFromProps(props, state) {
    console.log('2. getDerivedStateFromProps');
    return null;
  }
  
  componentDidMount() {
    console.log('3. componentDidMount');
    // Fetch data, setup subscriptions
  }
  
  shouldComponentUpdate(nextProps, nextState) {
    console.log('4. shouldComponentUpdate');
    return true;
  }
  
  getSnapshotBeforeUpdate(prevProps, prevState) {
    console.log('5. getSnapshotBeforeUpdate');
    return null;
  }
  
  componentDidUpdate(prevProps, prevState, snapshot) {
    console.log('6. componentDidUpdate');
  }
  
  componentWillUnmount() {
    console.log('7. componentWillUnmount');
    // Cleanup subscriptions, timers
  }
  
  render() {
    console.log('Render');
    return <div>{this.state.count}</div>;
  }
}

// Function component with hooks (equivalent)
import { useState, useEffect } from 'react';

function LifecycleDemo() {
  const [count, setCount] = useState(0);
  
  // componentDidMount + componentDidUpdate
  useEffect(() => {
    console.log('Component mounted or updated');
    
    // componentWillUnmount (cleanup)
    return () => {
      console.log('Component will unmount');
    };
  });
  
  // componentDidMount only (empty dependency array)
  useEffect(() => {
    console.log('Component mounted');
    fetchData();
  }, []);
  
  // componentDidUpdate when count changes
  useEffect(() => {
    console.log('Count changed:', count);
  }, [count]);
  
  return <div>{count}</div>;
}
```

**Lifecycle Phases:**
1. **Mounting**: Component being created
2. **Updating**: Component being re-rendered
3. **Unmounting**: Component being removed

**Key Points:**
- Use useEffect for side effects in function components
- Empty dependency array = componentDidMount
- No dependency array = runs on every render
- Return function = cleanup (componentWillUnmount)

---

## JSX

### What is JSX and how does it work?

**Answer:**

JSX is a syntax extension that looks like HTML but is JavaScript.

**Code Example:**
```javascript
// JSX
const element = <h1 className="greeting">Hello, world!</h1>;

// Compiles to:
const element = React.createElement(
  'h1',
  { className: 'greeting' },
  'Hello, world!'
);

// Expressions in JSX
const name = "John";
const element = <h1>Hello, {name}!</h1>;

// Attributes
const element = (
  <img 
    src={user.avatarUrl}
    alt={user.name}
    className="avatar"
  />
);

// Children
const element = (
  <div>
    <h1>Title</h1>
    <p>Paragraph</p>
  </div>
);

// Conditional rendering
const element = (
  <div>
    {isLoggedIn ? (
      <UserGreeting />
    ) : (
      <GuestGreeting />
    )}
  </div>
);

// Lists
const numbers = [1, 2, 3, 4, 5];
const listItems = numbers.map((number) =>
  <li key={number.toString()}>{number}</li>
);

// Fragments
return (
  <>
    <h1>Title</h1>
    <p>Paragraph</p>
  </>
);

// Comments
const element = (
  <div>
    {/* This is a comment */}
    <h1>Hello</h1>
  </div>
);
```

**JSX Rules:**
- Must return single parent element (or Fragment)
- className instead of class
- camelCase for attributes (onClick, not onclick)
- Close all tags (<br /> not <br>)
- Use {} for JavaScript expressions

**Key Points:**
- JSX is optional but highly recommended
- Compiles to React.createElement calls
- Closer to JavaScript than templates
- Babel transpiles JSX to JavaScript

---

## Interview Practice Questions

1. What is React and how is it different from Angular/Vue?
2. Explain the Virtual DOM
3. What's the difference between props and state?
4. What are React hooks and why were they introduced?
5. How do you handle events in React?
6. What is the key prop and why is it important?
7. Explain component lifecycle methods
8. What are controlled vs uncontrolled components?

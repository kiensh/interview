# React Basics Interview Questions

## Table of Contents

### Core Concepts
- [Q1: What is React and how does it work?](#q1)
- [Q2: What is JSX?](#q2)
- [Q3: What are components in React?](#q3)
- [Q4: What is the difference between props and state?](#q4)

### Hooks
- [Q5: What are React hooks?](#q5)
- [Q6: Explain useState and useReducer](#q6)
- [Q7: Explain useEffect and its cleanup](#q7)
- [Q8: What are useMemo and useCallback?](#q8)
- [Q9: What is useContext?](#q9)

### Advanced Patterns
- [Q10: What are controlled and uncontrolled components?](#q10)
- [Q11: How do you handle forms in React?](#q11)
- [Q12: What is component composition?](#q12)
- [Q13: How does React handle lists and keys?](#q13)

### Performance
- [Q14: How do you optimize React performance?](#q14)
- [Q15: What is React.memo and when to use it?](#q15)

---

## Core Concepts

<a id="q1"></a>
### Q1: What is React and how does it work?
**Answer:**

React is a JavaScript library for building user interfaces using a component-based architecture.

**Key concepts:**
| Concept | Description |
|---------|-------------|
| Virtual DOM | In-memory representation of real DOM |
| Reconciliation | Process of diffing and updating DOM |
| One-way data flow | Data flows from parent to child |
| Component-based | UI is built from reusable components |

```jsx
// React creates a Virtual DOM
// When state changes:
// 1. Creates new Virtual DOM tree
// 2. Diffs with previous tree (Reconciliation)
// 3. Updates only changed parts in real DOM

import React from 'react';
import ReactDOM from 'react-dom/client';

// Simple component
function App() {
    return <h1>Hello, React!</h1>;
}

// Render to DOM
const root = ReactDOM.createRoot(document.getElementById('root'));
root.render(<App />);

// Component with state - triggers re-render when state changes
function Counter() {
    const [count, setCount] = React.useState(0);
    
    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={() => setCount(count + 1)}>
                Increment
            </button>
        </div>
    );
}

// How Virtual DOM works:
// Initial render: Virtual DOM → Real DOM
// State change: New Virtual DOM → Diff → Patch Real DOM

// React 18 features
// Concurrent rendering - can interrupt rendering
// Automatic batching - groups state updates
// Suspense for data fetching

// Strict Mode (development only)
root.render(
    <React.StrictMode>
        <App />
    </React.StrictMode>
);
```

<a id="q2"></a>
### Q2: What is JSX?
**Answer:**

JSX is a syntax extension that allows writing HTML-like code in JavaScript.

```jsx
// JSX compiles to React.createElement calls

// JSX
const element = <h1 className="greeting">Hello, world!</h1>;

// Compiled to
const element = React.createElement(
    'h1',
    { className: 'greeting' },
    'Hello, world!'
);

// Embedding expressions
const name = 'John';
const element = <h1>Hello, {name}!</h1>;

// Any JavaScript expression works
const element = <h1>{2 + 2}</h1>;
const element = <h1>{user.firstName} {user.lastName}</h1>;
const element = <h1>{formatName(user)}</h1>;

// Conditional rendering
const element = <div>{isLoggedIn ? 'Welcome!' : 'Please log in'}</div>;
const element = <div>{isLoggedIn && <UserGreeting />}</div>;

// Lists
const items = ['Apple', 'Banana', 'Orange'];
const list = (
    <ul>
        {items.map((item, index) => (
            <li key={index}>{item}</li>
        ))}
    </ul>
);

// Attributes
// className instead of class
const element = <div className="container" />;

// htmlFor instead of for
const element = <label htmlFor="name">Name</label>;

// style takes an object
const element = <div style={{ color: 'red', fontSize: '16px' }} />;

// Boolean attributes
const element = <input disabled />;  // disabled={true}
const element = <input disabled={false} />;  // Not disabled

// Spread attributes
const props = { id: 'input', type: 'text', placeholder: 'Enter name' };
const element = <input {...props} />;

// Fragments - return multiple elements without wrapper
function Component() {
    return (
        <>
            <h1>Title</h1>
            <p>Content</p>
        </>
    );
}

// Or with key
function List({ items }) {
    return items.map(item => (
        <React.Fragment key={item.id}>
            <dt>{item.term}</dt>
            <dd>{item.description}</dd>
        </React.Fragment>
    ));
}

// JSX prevents injection attacks (escapes values)
const userInput = '<script>alert("xss")</script>';
const element = <div>{userInput}</div>;  // Safe - renders as text

// Render raw HTML (dangerous!)
const element = <div dangerouslySetInnerHTML={{ __html: htmlString }} />;
```

<a id="q3"></a>
### Q3: What are components in React?
**Answer:**

Components are reusable, self-contained pieces of UI.

```jsx
// Functional Component (recommended)
function Greeting({ name }) {
    return <h1>Hello, {name}!</h1>;
}

// Arrow function component
const Greeting = ({ name }) => <h1>Hello, {name}!</h1>;

// Class Component (legacy, still supported)
class Greeting extends React.Component {
    render() {
        return <h1>Hello, {this.props.name}!</h1>;
    }
}

// Using components
function App() {
    return (
        <div>
            <Greeting name="Alice" />
            <Greeting name="Bob" />
        </div>
    );
}

// Component with children
function Card({ title, children }) {
    return (
        <div className="card">
            <h2>{title}</h2>
            <div className="card-body">{children}</div>
        </div>
    );
}

// Usage
<Card title="Welcome">
    <p>This is the card content.</p>
    <button>Click me</button>
</Card>

// Component with default props
function Button({ text = 'Click', onClick, disabled = false }) {
    return (
        <button onClick={onClick} disabled={disabled}>
            {text}
        </button>
    );
}

// PropTypes for type checking (runtime)
import PropTypes from 'prop-types';

Button.propTypes = {
    text: PropTypes.string,
    onClick: PropTypes.func.isRequired,
    disabled: PropTypes.bool
};

Button.defaultProps = {
    text: 'Click',
    disabled: false
};

// TypeScript (compile-time type checking)
interface ButtonProps {
    text?: string;
    onClick: () => void;
    disabled?: boolean;
}

function Button({ text = 'Click', onClick, disabled = false }: ButtonProps) {
    return (
        <button onClick={onClick} disabled={disabled}>
            {text}
        </button>
    );
}

// Component organization
// Single responsibility - one reason to change
// Composition over inheritance
// Keep components small and focused
```

<a id="q4"></a>
### Q4: What is the difference between props and state?
**Answer:**

| Feature | Props | State |
|---------|-------|-------|
| Ownership | Parent owns | Component owns |
| Mutability | Read-only | Can be changed |
| Updates | From parent | With setState |
| Source | External | Internal |

```jsx
// Props - passed from parent, read-only
function UserCard({ user, onDelete }) {
    // Can read props
    console.log(user.name);
    
    // Cannot modify props
    // user.name = 'New Name'; // ERROR!
    
    return (
        <div>
            <h2>{user.name}</h2>
            <button onClick={() => onDelete(user.id)}>Delete</button>
        </div>
    );
}

// State - owned by component, can change
function Counter() {
    const [count, setCount] = useState(0);
    
    // State can be updated
    const increment = () => setCount(count + 1);
    const reset = () => setCount(0);
    
    return (
        <div>
            <p>Count: {count}</p>
            <button onClick={increment}>+1</button>
            <button onClick={reset}>Reset</button>
        </div>
    );
}

// State updates may be asynchronous (batched)
function Counter() {
    const [count, setCount] = useState(0);
    
    const handleClick = () => {
        // These may be batched
        setCount(count + 1);
        setCount(count + 1);
        console.log(count); // Still old value!
    };
    
    // Use functional update for sequential updates
    const handleClickCorrect = () => {
        setCount(prev => prev + 1);
        setCount(prev => prev + 1);
        // Now count increases by 2
    };
    
    return <button onClick={handleClickCorrect}>{count}</button>;
}

// Lifting state up
function Parent() {
    const [shared, setShared] = useState('');
    
    return (
        <div>
            <ChildA value={shared} onChange={setShared} />
            <ChildB value={shared} />
        </div>
    );
}

function ChildA({ value, onChange }) {
    return <input value={value} onChange={e => onChange(e.target.value)} />;
}

function ChildB({ value }) {
    return <p>Value: {value}</p>;
}

// Derived state - compute from props/state, don't store
function ProductList({ products, category }) {
    // Don't do this:
    // const [filtered, setFiltered] = useState([]);
    
    // Do this instead:
    const filtered = products.filter(p => p.category === category);
    
    return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

---

## Hooks

<a id="q5"></a>
### Q5: What are React hooks?
**Answer:**

Hooks let you use state and other React features in functional components.

```jsx
// Rules of Hooks:
// 1. Only call hooks at the top level (not in loops, conditions, nested functions)
// 2. Only call hooks from React functions (components or custom hooks)

// Built-in hooks
import {
    useState,      // State management
    useEffect,     // Side effects
    useContext,    // Context consumption
    useReducer,    // Complex state
    useCallback,   // Memoized callbacks
    useMemo,       // Memoized values
    useRef,        // Mutable ref object
    useLayoutEffect, // Synchronous effects
    useImperativeHandle, // Customize ref handle
    useDebugValue, // DevTools label
    useId,         // Unique IDs
    useTransition, // Non-blocking updates
    useDeferredValue, // Deferred value
} from 'react';

// Custom hook
function useLocalStorage(key, initialValue) {
    const [value, setValue] = useState(() => {
        const stored = localStorage.getItem(key);
        return stored ? JSON.parse(stored) : initialValue;
    });
    
    useEffect(() => {
        localStorage.setItem(key, JSON.stringify(value));
    }, [key, value]);
    
    return [value, setValue];
}

// Usage
function Settings() {
    const [theme, setTheme] = useLocalStorage('theme', 'light');
    
    return (
        <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
            Current theme: {theme}
        </button>
    );
}

// Custom hook for fetching
function useFetch(url) {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(true);
    const [error, setError] = useState(null);
    
    useEffect(() => {
        let cancelled = false;
        
        async function fetchData() {
            try {
                setLoading(true);
                const response = await fetch(url);
                const json = await response.json();
                if (!cancelled) {
                    setData(json);
                }
            } catch (err) {
                if (!cancelled) {
                    setError(err);
                }
            } finally {
                if (!cancelled) {
                    setLoading(false);
                }
            }
        }
        
        fetchData();
        
        return () => {
            cancelled = true;
        };
    }, [url]);
    
    return { data, loading, error };
}

// Usage
function UserProfile({ userId }) {
    const { data: user, loading, error } = useFetch(`/api/users/${userId}`);
    
    if (loading) return <div>Loading...</div>;
    if (error) return <div>Error: {error.message}</div>;
    return <div>{user.name}</div>;
}
```

<a id="q6"></a>
### Q6: Explain useState and useReducer
**Answer:**

```jsx
// useState - simple state
const [count, setCount] = useState(0);
const [user, setUser] = useState({ name: '', email: '' });
const [items, setItems] = useState([]);

// Lazy initialization (runs once)
const [data, setData] = useState(() => {
    const stored = localStorage.getItem('data');
    return stored ? JSON.parse(stored) : [];
});

// Updating state
setCount(5);                      // Direct value
setCount(prev => prev + 1);       // Functional update

// Object state - must spread to maintain other fields
setUser(prev => ({ ...prev, name: 'John' }));

// Array state
setItems([...items, newItem]);                    // Add
setItems(items.filter(item => item.id !== id));  // Remove
setItems(items.map(item =>                        // Update
    item.id === id ? { ...item, done: true } : item
));

// useReducer - complex state logic
const initialState = { count: 0, step: 1 };

function reducer(state, action) {
    switch (action.type) {
        case 'increment':
            return { ...state, count: state.count + state.step };
        case 'decrement':
            return { ...state, count: state.count - state.step };
        case 'setStep':
            return { ...state, step: action.payload };
        case 'reset':
            return initialState;
        default:
            throw new Error(`Unknown action: ${action.type}`);
    }
}

function Counter() {
    const [state, dispatch] = useReducer(reducer, initialState);
    
    return (
        <div>
            <p>Count: {state.count}</p>
            <p>Step: {state.step}</p>
            <button onClick={() => dispatch({ type: 'increment' })}>+</button>
            <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
            <button onClick={() => dispatch({ type: 'setStep', payload: 5 })}>
                Set step to 5
            </button>
            <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
        </div>
    );
}

// Lazy initialization with useReducer
const [state, dispatch] = useReducer(reducer, initialArg, init);

function init(initialArg) {
    return { count: initialArg };
}

// When to use which:
// useState: Independent pieces of state
// useReducer: Complex state logic, multiple sub-values, state depends on previous state
```

<a id="q7"></a>
### Q7: Explain useEffect and its cleanup
**Answer:**

```jsx
import { useEffect, useState } from 'react';

// useEffect runs after render
function Component() {
    const [data, setData] = useState(null);
    
    // Runs after every render
    useEffect(() => {
        console.log('Effect ran');
    });
    
    // Runs once on mount (empty dependency array)
    useEffect(() => {
        console.log('Mounted');
        fetchData().then(setData);
    }, []);
    
    // Runs when dependencies change
    useEffect(() => {
        console.log(`ID changed to ${id}`);
        fetchItem(id).then(setData);
    }, [id]);
    
    // Cleanup function
    useEffect(() => {
        const subscription = subscribeToUpdates();
        
        return () => {
            // Cleanup on unmount or before next effect
            subscription.unsubscribe();
        };
    }, []);
    
    return <div>{data}</div>;
}

// Timer example with cleanup
function Timer() {
    const [seconds, setSeconds] = useState(0);
    
    useEffect(() => {
        const interval = setInterval(() => {
            setSeconds(s => s + 1);
        }, 1000);
        
        return () => clearInterval(interval);
    }, []);
    
    return <div>Seconds: {seconds}</div>;
}

// Event listener with cleanup
function WindowSize() {
    const [size, setSize] = useState(window.innerWidth);
    
    useEffect(() => {
        const handleResize = () => setSize(window.innerWidth);
        window.addEventListener('resize', handleResize);
        
        return () => window.removeEventListener('resize', handleResize);
    }, []);
    
    return <div>Width: {size}</div>;
}

// Fetch with abort controller
function UserData({ userId }) {
    const [user, setUser] = useState(null);
    
    useEffect(() => {
        const controller = new AbortController();
        
        fetch(`/api/users/${userId}`, { signal: controller.signal })
            .then(res => res.json())
            .then(data => setUser(data))
            .catch(err => {
                if (err.name !== 'AbortError') {
                    console.error(err);
                }
            });
        
        return () => controller.abort();
    }, [userId]);
    
    return user ? <div>{user.name}</div> : <div>Loading...</div>;
}

// useLayoutEffect - synchronous, runs before paint
// Use for DOM measurements
function Tooltip({ targetRef }) {
    const [position, setPosition] = useState({ top: 0, left: 0 });
    
    useLayoutEffect(() => {
        const rect = targetRef.current.getBoundingClientRect();
        setPosition({ top: rect.bottom, left: rect.left });
    }, [targetRef]);
    
    return <div style={position}>Tooltip</div>;
}

// Common mistake: missing dependencies
function BadExample({ query }) {
    const [data, setData] = useState(null);
    
    // Bug: doesn't refetch when query changes
    useEffect(() => {
        fetchData(query).then(setData);
    }, []); // ESLint will warn about missing dependency
    
    // Fix: include query in dependencies
    useEffect(() => {
        fetchData(query).then(setData);
    }, [query]);
}
```

<a id="q8"></a>
### Q8: What are useMemo and useCallback?
**Answer:**

Both are used for memoization to prevent unnecessary recalculations/recreations.

```jsx
import { useMemo, useCallback, useState, memo } from 'react';

// useMemo - memoize computed values
function ExpensiveComponent({ items, filter }) {
    // Without useMemo: recalculates on every render
    // const filtered = items.filter(item => item.category === filter);
    
    // With useMemo: only recalculates when dependencies change
    const filtered = useMemo(() => {
        console.log('Filtering...');
        return items.filter(item => item.category === filter);
    }, [items, filter]);
    
    return <ul>{filtered.map(item => <li key={item.id}>{item.name}</li>)}</ul>;
}

// useCallback - memoize functions
function ParentComponent() {
    const [count, setCount] = useState(0);
    const [text, setText] = useState('');
    
    // Without useCallback: new function every render
    // const handleClick = () => setCount(c => c + 1);
    
    // With useCallback: same function reference unless dependencies change
    const handleClick = useCallback(() => {
        setCount(c => c + 1);
    }, []); // No dependencies - function never changes
    
    const handleTextChange = useCallback((newText) => {
        setText(newText);
    }, []); // Stable reference
    
    return (
        <div>
            <MemoizedButton onClick={handleClick} />
            <MemoizedInput onChange={handleTextChange} />
            <p>Count: {count}</p>
        </div>
    );
}

// useCallback is useful with memoized children
const MemoizedButton = memo(function Button({ onClick }) {
    console.log('Button rendered');
    return <button onClick={onClick}>Click</button>;
});

// When NOT to use useMemo/useCallback
function SimpleComponent({ name }) {
    // DON'T: overhead outweighs benefit for simple operations
    const greeting = useMemo(() => `Hello, ${name}`, [name]);
    
    // DO: just compute it
    const greeting = `Hello, ${name}`;
    
    return <div>{greeting}</div>;
}

// When to use:
// useMemo:
// 1. Expensive calculations
// 2. Reference equality for objects/arrays passed as props
// 3. Dependency of useEffect that needs stable reference

// useCallback:
// 1. Passing callbacks to memoized children
// 2. Callbacks as dependencies of useEffect
// 3. Custom hooks that expose handlers

// Reference equality example
function Parent() {
    const [count, setCount] = useState(0);
    
    // Without useMemo: new object every render, child re-renders
    // const config = { theme: 'dark' };
    
    // With useMemo: same reference, child doesn't re-render
    const config = useMemo(() => ({ theme: 'dark' }), []);
    
    return (
        <div>
            <button onClick={() => setCount(c => c + 1)}>{count}</button>
            <MemoizedChild config={config} />
        </div>
    );
}
```

<a id="q9"></a>
### Q9: What is useContext?
**Answer:**

useContext provides a way to pass data through the component tree without prop drilling.

```jsx
import { createContext, useContext, useState } from 'react';

// Create context with default value
const ThemeContext = createContext('light');

// Provider component
function ThemeProvider({ children }) {
    const [theme, setTheme] = useState('light');
    
    const toggleTheme = () => {
        setTheme(t => t === 'light' ? 'dark' : 'light');
    };
    
    return (
        <ThemeContext.Provider value={{ theme, toggleTheme }}>
            {children}
        </ThemeContext.Provider>
    );
}

// Consumer with useContext
function ThemedButton() {
    const { theme, toggleTheme } = useContext(ThemeContext);
    
    return (
        <button
            onClick={toggleTheme}
            style={{ background: theme === 'light' ? '#fff' : '#333' }}
        >
            Toggle Theme
        </button>
    );
}

// Usage
function App() {
    return (
        <ThemeProvider>
            <div>
                <ThemedButton />
                <ThemedComponent />
            </div>
        </ThemeProvider>
    );
}

// Multiple contexts
const UserContext = createContext(null);
const ThemeContext = createContext('light');

function App() {
    return (
        <UserContext.Provider value={currentUser}>
            <ThemeContext.Provider value="dark">
                <Content />
            </ThemeContext.Provider>
        </UserContext.Provider>
    );
}

// Custom hook for context (recommended pattern)
const AuthContext = createContext(null);

function AuthProvider({ children }) {
    const [user, setUser] = useState(null);
    
    const login = async (credentials) => {
        const user = await api.login(credentials);
        setUser(user);
    };
    
    const logout = () => {
        setUser(null);
    };
    
    return (
        <AuthContext.Provider value={{ user, login, logout }}>
            {children}
        </AuthContext.Provider>
    );
}

function useAuth() {
    const context = useContext(AuthContext);
    if (!context) {
        throw new Error('useAuth must be used within AuthProvider');
    }
    return context;
}

// Usage
function LoginButton() {
    const { user, login, logout } = useAuth();
    
    if (user) {
        return <button onClick={logout}>Logout {user.name}</button>;
    }
    return <button onClick={() => login({ email, password })}>Login</button>;
}

// Performance: Context re-renders all consumers
// Split contexts to avoid unnecessary re-renders
const UserContext = createContext(null);
const UserDispatchContext = createContext(null);

function UserProvider({ children }) {
    const [user, dispatch] = useReducer(userReducer, null);
    
    return (
        <UserContext.Provider value={user}>
            <UserDispatchContext.Provider value={dispatch}>
                {children}
            </UserDispatchContext.Provider>
        </UserContext.Provider>
    );
}
```

---

## Advanced Patterns

<a id="q10"></a>
### Q10: What are controlled and uncontrolled components?
**Answer:**

| Type | State location | Use case |
|------|----------------|----------|
| Controlled | React state | Full control, validation |
| Uncontrolled | DOM | Simple forms, file inputs |

```jsx
import { useState, useRef } from 'react';

// Controlled component - React controls the value
function ControlledInput() {
    const [value, setValue] = useState('');
    
    const handleChange = (e) => {
        // Can transform/validate before setting
        setValue(e.target.value.toUpperCase());
    };
    
    return (
        <input
            value={value}
            onChange={handleChange}
        />
    );
}

// Uncontrolled component - DOM controls the value
function UncontrolledInput() {
    const inputRef = useRef(null);
    
    const handleSubmit = (e) => {
        e.preventDefault();
        console.log(inputRef.current.value);
    };
    
    return (
        <form onSubmit={handleSubmit}>
            <input
                ref={inputRef}
                defaultValue="initial"
            />
            <button type="submit">Submit</button>
        </form>
    );
}

// File input is always uncontrolled
function FileInput() {
    const fileRef = useRef(null);
    
    const handleSubmit = () => {
        const file = fileRef.current.files[0];
        console.log(file);
    };
    
    return <input type="file" ref={fileRef} />;
}

// Controlled form with validation
function ControlledForm() {
    const [form, setForm] = useState({
        email: '',
        password: ''
    });
    const [errors, setErrors] = useState({});
    
    const handleChange = (e) => {
        const { name, value } = e.target;
        setForm(prev => ({ ...prev, [name]: value }));
        
        // Clear error when user types
        if (errors[name]) {
            setErrors(prev => ({ ...prev, [name]: '' }));
        }
    };
    
    const validate = () => {
        const newErrors = {};
        if (!form.email.includes('@')) {
            newErrors.email = 'Invalid email';
        }
        if (form.password.length < 8) {
            newErrors.password = 'Password must be at least 8 characters';
        }
        setErrors(newErrors);
        return Object.keys(newErrors).length === 0;
    };
    
    const handleSubmit = (e) => {
        e.preventDefault();
        if (validate()) {
            console.log('Submit:', form);
        }
    };
    
    return (
        <form onSubmit={handleSubmit}>
            <input
                name="email"
                value={form.email}
                onChange={handleChange}
            />
            {errors.email && <span>{errors.email}</span>}
            
            <input
                name="password"
                type="password"
                value={form.password}
                onChange={handleChange}
            />
            {errors.password && <span>{errors.password}</span>}
            
            <button type="submit">Submit</button>
        </form>
    );
}
```

<a id="q11"></a>
### Q11: How do you handle forms in React?
**Answer:**

```jsx
import { useState } from 'react';

// Basic form handling
function BasicForm() {
    const [formData, setFormData] = useState({
        username: '',
        email: '',
        role: 'user',
        newsletter: false,
        skills: []
    });
    
    const handleChange = (e) => {
        const { name, value, type, checked } = e.target;
        
        setFormData(prev => ({
            ...prev,
            [name]: type === 'checkbox' ? checked : value
        }));
    };
    
    const handleMultiSelect = (e) => {
        const values = Array.from(e.target.selectedOptions, opt => opt.value);
        setFormData(prev => ({ ...prev, skills: values }));
    };
    
    const handleSubmit = (e) => {
        e.preventDefault();
        console.log(formData);
    };
    
    return (
        <form onSubmit={handleSubmit}>
            {/* Text input */}
            <input
                type="text"
                name="username"
                value={formData.username}
                onChange={handleChange}
                placeholder="Username"
            />
            
            {/* Email input */}
            <input
                type="email"
                name="email"
                value={formData.email}
                onChange={handleChange}
                placeholder="Email"
            />
            
            {/* Select */}
            <select name="role" value={formData.role} onChange={handleChange}>
                <option value="user">User</option>
                <option value="admin">Admin</option>
                <option value="moderator">Moderator</option>
            </select>
            
            {/* Checkbox */}
            <label>
                <input
                    type="checkbox"
                    name="newsletter"
                    checked={formData.newsletter}
                    onChange={handleChange}
                />
                Subscribe to newsletter
            </label>
            
            {/* Multi-select */}
            <select
                name="skills"
                multiple
                value={formData.skills}
                onChange={handleMultiSelect}
            >
                <option value="react">React</option>
                <option value="node">Node.js</option>
                <option value="python">Python</option>
            </select>
            
            <button type="submit">Submit</button>
        </form>
    );
}

// Form with React Hook Form (popular library)
import { useForm } from 'react-hook-form';

function HookForm() {
    const {
        register,
        handleSubmit,
        formState: { errors, isSubmitting }
    } = useForm();
    
    const onSubmit = async (data) => {
        await submitToAPI(data);
    };
    
    return (
        <form onSubmit={handleSubmit(onSubmit)}>
            <input
                {...register('email', {
                    required: 'Email is required',
                    pattern: {
                        value: /\S+@\S+\.\S+/,
                        message: 'Invalid email'
                    }
                })}
            />
            {errors.email && <span>{errors.email.message}</span>}
            
            <input
                type="password"
                {...register('password', {
                    required: 'Password is required',
                    minLength: {
                        value: 8,
                        message: 'Password must be at least 8 characters'
                    }
                })}
            />
            {errors.password && <span>{errors.password.message}</span>}
            
            <button type="submit" disabled={isSubmitting}>
                {isSubmitting ? 'Submitting...' : 'Submit'}
            </button>
        </form>
    );
}
```

<a id="q12"></a>
### Q12: What is component composition?
**Answer:**

Composition is a pattern for reusing code by combining components.

```jsx
// Children prop - most basic composition
function Card({ children }) {
    return <div className="card">{children}</div>;
}

function App() {
    return (
        <Card>
            <h2>Title</h2>
            <p>Content goes here</p>
        </Card>
    );
}

// Named slots
function Layout({ header, sidebar, children, footer }) {
    return (
        <div className="layout">
            <header>{header}</header>
            <aside>{sidebar}</aside>
            <main>{children}</main>
            <footer>{footer}</footer>
        </div>
    );
}

function App() {
    return (
        <Layout
            header={<Nav />}
            sidebar={<SideMenu />}
            footer={<Footer />}
        >
            <MainContent />
        </Layout>
    );
}

// Compound components
const Tabs = ({ children, defaultTab }) => {
    const [activeTab, setActiveTab] = useState(defaultTab);
    
    return (
        <TabsContext.Provider value={{ activeTab, setActiveTab }}>
            <div className="tabs">{children}</div>
        </TabsContext.Provider>
    );
};

Tabs.List = function TabList({ children }) {
    return <div className="tab-list">{children}</div>;
};

Tabs.Tab = function Tab({ id, children }) {
    const { activeTab, setActiveTab } = useContext(TabsContext);
    return (
        <button
            className={activeTab === id ? 'active' : ''}
            onClick={() => setActiveTab(id)}
        >
            {children}
        </button>
    );
};

Tabs.Panel = function TabPanel({ id, children }) {
    const { activeTab } = useContext(TabsContext);
    return activeTab === id ? <div className="tab-panel">{children}</div> : null;
};

// Usage
<Tabs defaultTab="tab1">
    <Tabs.List>
        <Tabs.Tab id="tab1">Tab 1</Tabs.Tab>
        <Tabs.Tab id="tab2">Tab 2</Tabs.Tab>
    </Tabs.List>
    <Tabs.Panel id="tab1">Content 1</Tabs.Panel>
    <Tabs.Panel id="tab2">Content 2</Tabs.Panel>
</Tabs>

// Render props pattern
function MouseTracker({ render }) {
    const [position, setPosition] = useState({ x: 0, y: 0 });
    
    useEffect(() => {
        const handleMove = (e) => {
            setPosition({ x: e.clientX, y: e.clientY });
        };
        window.addEventListener('mousemove', handleMove);
        return () => window.removeEventListener('mousemove', handleMove);
    }, []);
    
    return render(position);
}

<MouseTracker
    render={({ x, y }) => <div>Mouse: {x}, {y}</div>}
/>

// Higher-Order Component (HOC)
function withAuth(WrappedComponent) {
    return function AuthenticatedComponent(props) {
        const { user } = useAuth();
        
        if (!user) {
            return <Navigate to="/login" />;
        }
        
        return <WrappedComponent {...props} user={user} />;
    };
}

const ProtectedDashboard = withAuth(Dashboard);
```

<a id="q13"></a>
### Q13: How does React handle lists and keys?
**Answer:**

Keys help React identify which items have changed, added, or removed.

```jsx
// Basic list rendering
function TodoList({ todos }) {
    return (
        <ul>
            {todos.map(todo => (
                <li key={todo.id}>{todo.text}</li>
            ))}
        </ul>
    );
}

// Why keys are important
// Without keys, React can't track items efficiently
// This can cause bugs with component state

// BAD: Using index as key
// Problems when items are reordered, inserted, or deleted
{items.map((item, index) => (
    <TodoItem key={index} item={item} /> // DON'T DO THIS
))}

// GOOD: Using unique identifier
{items.map(item => (
    <TodoItem key={item.id} item={item} /> // DO THIS
))}

// When index is okay:
// 1. List is static (never changes)
// 2. Items have no IDs
// 3. List is never reordered or filtered

// Keys must be unique among siblings
function Board({ data }) {
    return (
        <div>
            {/* These can have same keys - different lists */}
            <ul>
                {data.left.map(item => <li key={item.id}>{item.name}</li>)}
            </ul>
            <ul>
                {data.right.map(item => <li key={item.id}>{item.name}</li>)}
            </ul>
        </div>
    );
}

// Keys are not passed as props
function ListItem({ id, text }) {
    // 'id' is undefined - key is not a prop
    console.log(id); // undefined
    return <li>{text}</li>;
}

{items.map(item => (
    // Need to pass id separately if needed
    <ListItem key={item.id} id={item.id} text={item.text} />
))}

// Fragment with key
function Glossary({ items }) {
    return (
        <dl>
            {items.map(item => (
                <Fragment key={item.id}>
                    <dt>{item.term}</dt>
                    <dd>{item.description}</dd>
                </Fragment>
            ))}
        </dl>
    );
}

// Resetting component state with key
function ProfilePage({ userId }) {
    return (
        // Key change = component remounts with fresh state
        <Profile key={userId} userId={userId} />
    );
}
```

---

## Performance

<a id="q14"></a>
### Q14: How do you optimize React performance?
**Answer:**

```jsx
import { memo, useMemo, useCallback, lazy, Suspense } from 'react';

// 1. Code splitting with lazy loading
const HeavyComponent = lazy(() => import('./HeavyComponent'));

function App() {
    return (
        <Suspense fallback={<div>Loading...</div>}>
            <HeavyComponent />
        </Suspense>
    );
}

// 2. Memoization
const ExpensiveList = memo(function ExpensiveList({ items }) {
    return (
        <ul>
            {items.map(item => <li key={item.id}>{item.name}</li>)}
        </ul>
    );
});

// 3. Virtualization for long lists
import { FixedSizeList } from 'react-window';

function VirtualizedList({ items }) {
    return (
        <FixedSizeList
            height={400}
            itemCount={items.length}
            itemSize={50}
            width="100%"
        >
            {({ index, style }) => (
                <div style={style}>{items[index].name}</div>
            )}
        </FixedSizeList>
    );
}

// 4. Debounce expensive operations
function SearchInput({ onSearch }) {
    const [value, setValue] = useState('');
    
    // Debounce search
    const debouncedSearch = useMemo(
        () => debounce(onSearch, 300),
        [onSearch]
    );
    
    const handleChange = (e) => {
        setValue(e.target.value);
        debouncedSearch(e.target.value);
    };
    
    return <input value={value} onChange={handleChange} />;
}

// 5. Avoid inline objects/arrays
function Parent() {
    // BAD: new object every render
    // return <Child style={{ color: 'red' }} />;
    
    // GOOD: stable reference
    const style = useMemo(() => ({ color: 'red' }), []);
    return <Child style={style} />;
}

// 6. Use production builds
// npm run build (uses production mode)

// 7. Profiling
import { Profiler } from 'react';

function onRenderCallback(
    id, phase, actualDuration, baseDuration, startTime, commitTime
) {
    console.log({ id, phase, actualDuration });
}

<Profiler id="Navigation" onRender={onRenderCallback}>
    <Navigation />
</Profiler>

// 8. Use useTransition for non-urgent updates
function App() {
    const [isPending, startTransition] = useTransition();
    const [query, setQuery] = useState('');
    const [results, setResults] = useState([]);
    
    const handleChange = (e) => {
        setQuery(e.target.value); // Urgent
        
        startTransition(() => {
            setResults(filterItems(e.target.value)); // Non-urgent
        });
    };
    
    return (
        <div>
            <input value={query} onChange={handleChange} />
            {isPending ? 'Loading...' : <Results items={results} />}
        </div>
    );
}

// 9. Avoid unnecessary re-renders
// - Split components
// - Lift state only when needed
// - Use context sparingly
```

<a id="q15"></a>
### Q15: What is React.memo and when to use it?
**Answer:**

React.memo is a higher-order component that memoizes the rendered output.

```jsx
import { memo, useState, useCallback } from 'react';

// Basic memo usage
const ExpensiveComponent = memo(function ExpensiveComponent({ data }) {
    console.log('Rendering ExpensiveComponent');
    // Expensive render logic...
    return <div>{data.name}</div>;
});

// Component only re-renders if props change (shallow comparison)
function Parent() {
    const [count, setCount] = useState(0);
    const [user] = useState({ name: 'John' });
    
    return (
        <div>
            <button onClick={() => setCount(c => c + 1)}>{count}</button>
            {/* Won't re-render when count changes */}
            <ExpensiveComponent data={user} />
        </div>
    );
}

// Custom comparison function
const UserCard = memo(
    function UserCard({ user, onClick }) {
        return (
            <div onClick={onClick}>
                <h2>{user.name}</h2>
                <p>{user.email}</p>
            </div>
        );
    },
    (prevProps, nextProps) => {
        // Return true if props are equal (skip re-render)
        // Return false to re-render
        return prevProps.user.id === nextProps.user.id;
    }
);

// Common mistake: passing new object/function reference
function Parent() {
    const [count, setCount] = useState(0);
    
    // BAD: new function every render, memo doesn't help
    return <MemoizedChild onClick={() => console.log('click')} />;
    
    // GOOD: stable function reference
    const handleClick = useCallback(() => console.log('click'), []);
    return <MemoizedChild onClick={handleClick} />;
}

// When to use memo:
// 1. Component renders often with same props
// 2. Component renders expensive output
// 3. Parent re-renders frequently

// When NOT to use memo:
// 1. Component is simple/cheap to render
// 2. Props change frequently anyway
// 3. Component accepts children (usually invalidates memo)

// Children prop breaks memo
function Parent() {
    const [count, setCount] = useState(0);
    
    return (
        <MemoizedComponent>
            {/* New children every render */}
            <span>Child content</span>
        </MemoizedComponent>
    );
}

// Solution: memoize children or use composition
function Parent() {
    const [count, setCount] = useState(0);
    const children = useMemo(() => <span>Child content</span>, []);
    
    return <MemoizedComponent>{children}</MemoizedComponent>;
}

// Or restructure
function Parent() {
    const [count, setCount] = useState(0);
    
    return (
        <div>
            <button onClick={() => setCount(c => c + 1)}>{count}</button>
            <ExpensiveComponentWrapper />
        </div>
    );
}

// Children don't need count, won't re-render
function ExpensiveComponentWrapper() {
    return (
        <ExpensiveComponent>
            <span>Child content</span>
        </ExpensiveComponent>
    );
}
```

---

[← Node.js Basics](nodejs-basics.md) | [Back to Frontend Index](README.md) | [Vue.js Basics →](vuejs-basics.md)

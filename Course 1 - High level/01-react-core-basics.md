# React Mastery Guide — Part 1: Core Basics
## From Zero to Pro — 20 Years Experience Level

---

# 1. WHAT IS REACT?

**Definition:** React is a declarative, component-based JavaScript library for building user interfaces. Created by Facebook (2013). It's a *library*, not a framework — it handles only the View layer.

## Core Philosophy
- **Declarative** — You describe *what* the UI should look like for a given state, not *how* to mutate the DOM
- **Component-Based** — Build encapsulated components that manage their own state, compose them to make complex UIs
- **Learn Once, Write Anywhere** — React can render on server (Next.js), mobile (React Native), desktop (Electron)

## The Virtual DOM — Understand This Deeply

```
Real DOM:        Virtual DOM:
┌──────────┐     ┌──────────┐
│ <div>    │     │ <div>    │
│   <p>    │     │   <p>    │
│    Hi    │     │    Hi    │
│   </p>   │     │   </p>   │
│ </div>   │     │ </div>   │
└──────────┘     └──────────┘
     │                 │
     │    Diffing      │
     │  Algorithm      │
     └─────┬───────────┘
           ▼
    ┌──────────────────┐
    │ Batch updates to │
    │ real DOM (React  │
    │ Reconciliation)  │
    └──────────────────┘
```

**How Virtual DOM works:**
1. When state changes, React creates a new Virtual DOM tree
2. Diffing algorithm compares new Virtual DOM with previous snapshot
3. Calculates minimal set of DOM operations needed
4. Batches and applies updates to real DOM
5. This batch + minimal ops approach is why React is fast

**Interview Deep Dive:** The diffing algorithm uses two assumptions (O(n) complexity):
1. Elements of different types produce different trees
2. Keys on lists help identify stable elements

```jsx
// This is NOT React — this is what happens behind the scenes
// JSX compiles to React.createElement() calls
const element = React.createElement('div', { className: 'container' },
  React.createElement('h1', null, 'Hello World')
);

// Same as JSX:
const element = <div className="container"><h1>Hello World</h1></div>;
```

---

# 2. JSX — JAVASCRIPT XML

**Definition:** JSX is a syntax extension for JavaScript that looks like HTML. It's compiled by Babel to `React.createElement()` calls.

## Rules of JSX
1. **Single Root Element** — Must wrap siblings in one parent (use `<> </>` Fragment)
2. **Close All Tags** — Even self-closing: `<br />`
3. **camelCase Attributes** — `className`, `onClick`, `tabIndex`, `htmlFor`
4. **Curly Braces for JS** — `{expression}` not `"expression"`
5. **Comments** — `{/* comment */}` not `<!-- comment -->`

```jsx
// Fragment shorthand - no DOM node, just grouping
const App = () => (
  <>
    <Header />
    <Main />
    <Footer />
  </>
);

// Expressions in JSX
const UserCard = ({ user }) => (
  <div className="card">
    <h2>{user.name}</h2>
    <p>{user.bio || 'No bio'}</p>
    {user.isAdmin && <Badge type="admin" />}
    {user.skills.map(skill => <span key={skill}>{skill}</span>)}
  </div>
);
```

## Real-World Scenarios

**Scenario 1: Dynamic Class Names**
```jsx
// Utility function for conditional classes
const Button = ({ variant, disabled, children }) => (
  <button
    className={[
      'btn',                    // base
      `btn-${variant}`,         // 'btn-primary' | 'btn-danger'
      disabled && 'btn-disabled'// conditionally applied
    ].filter(Boolean).join(' ')}
    disabled={disabled}
  >
    {children}
  </button>
);
```

**Scenario 2: Rendering API Data Safely**
```jsx
const ProductList = () => {
  const [products, setProducts] = useState([]);
  const [error, setError] = useState(null);

  // ... fetch logic ...

  return (
    <div className="product-grid">
      {error && <ErrorBanner message={error} />}
      {!products.length && !error && <Spinner />}
      {products.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
    </div>
  );
};
```

**Scenario 3: Conditional Layout Structure**
```jsx
const Dashboard = ({ user, analytics }) => (
  <div className="dashboard">
    <Sidebar menuItems={getMenuForRole(user.role)} />
    <main>
      {user.role === 'admin' ? (
        <AdminAnalytics data={analytics} />
      ) : user.role === 'manager' ? (
        <TeamOverview data={analytics.teams} />
      ) : (
        <PersonalStats data={analytics.personal} />
      )}
    </main>
  </div>
);
```

## Interview Tips
- JSX is NOT HTML — it's syntactic sugar for `createElement`
- React doesn't require JSX (you can use `createElement` directly)
- Babel is the most common compiler for JSX
- The `key` prop in lists is NOT accessible in child component
- Boolean attributes like `disabled` work differently: `disabled={false}` removes the attribute

## Interview Questions

**Q1:** Why can't JSX return two sibling elements without a wrapper?
**A:** JSX compiles to `React.createElement()` which takes only one root element. Use `<Fragment>` or `<>` which compiles to `React.createElement(React.Fragment, null, ...)`.

**Q2:** What's the difference between `===` and `==` in JSX conditional rendering?
**A:** Use `===` always. `==` causes type coercion (e.g., `0 == false` is true). In `{count && <span>{count}</span>}`, if count is 0, React renders `0` not the span.

**Q3:** How does React handle `dangerouslySetInnerHTML` and why is it named that way?
**A:** React escapes all content by default to prevent XSS. `dangerouslySetInnerHTML` bypasses this — the name is intentional to warn developers. Only use with sanitized HTML (DOMPurify).

**Q4:** Why are `className` and `htmlFor` used instead of `class` and `for`?
**A:** JSX is closer to JavaScript than HTML. `class` and `for` are reserved words in JS. React DOM uses camelCase property naming convention.

**Q5:** How does JSX prevent injection attacks?
**A:** React escapes all values embedded in JSX before rendering. `{userInput}` becomes a string, not executable code. Only `dangerouslySetInnerHTML` bypasses this.

---

# 3. COMPONENTS — THE BUILDING BLOCKS

## Two Types of Components

### Functional Components (Modern — use these)
```jsx
// Simple functional component
const Greeting = ({ name }) => <h1>Hello, {name}!</h1>;

// With hooks (modern React)
const Counter = () => {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
};
```

### Class Components (Legacy — understand for interviews/codebases)
```jsx
class Counter extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
    // Binding is required in class components
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    this.setState(prevState => ({ count: prevState.count + 1 }));
  }

  render() {
    return <button onClick={this.handleClick}>{this.state.count}</button>;
  }
}
```

## Component Composition — The Real Power

```jsx
// Instead of inheritance, compose!
// WRONG:
class SpecialButton extends Button { /* ... */ }

// RIGHT:
const SpecialButton = ({ icon, children, ...props }) => (
  <Button {...props}>
    <Icon name={icon} />
    {children}
  </Button>
);
```

## Real-World Component Architecture

```jsx
// components/layout/PageLayout.jsx
const PageLayout = ({ header, sidebar, main, footer }) => (
  <div className="page-layout">
    {header && <header>{header}</header>}
    <div className="content">
      {sidebar && <aside>{sidebar}</aside>}
      <main>{main}</main>
    </div>
    {footer && <footer>{footer}</footer>}
  </div>
);

// Usage — composing components together
const DashboardPage = () => (
  <PageLayout
    header={<DashboardHeader />}
    sidebar={<NavigationMenu />}
    main={<Outlet />}  {/* React Router outlet */}
    footer={<DashboardFooter />}
  />
);
```

## Interview Questions

**Q1:** Functional vs Class components — which to use and why?
**A:** Always use functional components (since React 16.8+). They're simpler, use hooks, less boilerplate, no `this` binding issues, better tree-shaking, and the React team has said classes won't get new features.

**Q2:** What is a Pure Component?
**A:** `React.PureComponent` (class) or `React.memo` (functional) implements `shouldComponentUpdate` with shallow prop/state comparison. Useful for performance optimization when props/state are primitive or immutable.

**Q3:** Why should components be pure functions?
**A:** Pure functions always return the same output for the same input and have no side effects. This makes components predictable, testable, and enables React's performance optimizations (reconciliation, memoization).

**Q4:** What is the children prop pattern?
**A:** `children` is a special prop representing content between opening and closing tags. Enables component composition patterns like layouts, wrappers, and higher-order components.

**Q5:** How do you handle errors in a component?
**A:** Use Error Boundaries (class components with `componentDidCatch`). Functional components can't be Error Boundaries yet. Also use try-catch in event handlers and async operations.

---

# 4. PROPS — COMPONENT COMMUNICATION

**Definition:** Props (properties) are read-only data passed from parent to child. They enable component reusability.

## Prop Patterns

```jsx
// 1. Destructuring (cleanest)
const ProductCard = ({ title, price, image, onAddToCart }) => (
  <div className="product-card">
    <img src={image} alt={title} />
    <h3>{title}</h3>
    <p>${price}</p>
    <button onClick={() => onAddToCart(title)}>Add to Cart</button>
  </div>
);

// 2. Default Props
const Button = ({ variant = 'primary', size = 'md', children }) => (
  <button className={`btn btn-${variant} btn-${size}`}>{children}</button>
);

// 3. Spread Props (be careful!)
const Input = ({ label, error, ...inputProps }) => (
  <div className="field">
    <label>{label}</label>
    <input {...inputProps} className={error ? 'error' : ''} />
    {error && <span className="error-msg">{error}</span>}
  </div>
);
// Usage: <Input label="Email" type="email" placeholder="Enter email" required />

// 4. Render Props (advanced)
const DataFetcher = ({ url, render }) => {
  const [data, setData] = useState(null);
  // ... fetch logic ...
  return render(data);
};
// Usage:
// <DataFetcher
//   url="/api/users"
//   render={(data) => <UserList users={data} />}
// />
```

## Real-World Scenarios

**Scenario 1: Configuration Propagation**
```jsx
// Theme config flowing through props
const App = () => (
  <ThemeProvider theme={darkTheme}>
    <Dashboard config={dashboardConfig}>
      <Widget gridPosition={{ x: 0, y: 0, w: 6, h: 4 }}>
        <Chart type="line" data={revenueData} />
      </Widget>
    </Dashboard>
  </ThemeProvider>
);
// Props carry configuration down the tree
```

**Scenario 2: Callback Props for Child-to-Parent Communication**
```jsx
// Parent
const CheckoutForm = () => {
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState({});

  const handleNext = (stepData) => {
    setFormData(prev => ({ ...prev, ...stepData }));
    setStep(s => s + 1);
  };

  return (
    <div className="checkout">
      {step === 1 && <ShippingForm onNext={handleNext} />}
      {step === 2 && <PaymentForm onNext={handleNext} />}
      {step === 3 && <ReviewCart data={formData} onSubmit={placeOrder} />}
    </div>
  );
};

// Child
const ShippingForm = ({ onNext }) => {
  const [address, setAddress] = useState({});
  const handleSubmit = (e) => {
    e.preventDefault();
    onNext(address); // ← child calls parent's callback
  };
  return <form onSubmit={handleSubmit}>{/* ... */}</form>;
};
```

**Scenario 3: Polymorphic Components via Props**
```jsx
const Icon = ({ name, size = 24, className }) => {
  const IconComponent = iconMap[name]; // map of icon components
  return IconComponent ? (
    <IconComponent size={size} className={className} />
  ) : null;
};

const Button = ({ icon, variant, children, ...props }) => (
  <button className={`btn btn-${variant}`} {...props}>
    {icon && <Icon name={icon} size={16} />}
    {children}
  </button>
);

// Multiple usages:
<Button icon="shopping-cart" variant="primary">Add to Cart</Button>
<Button icon="trash" variant="danger">Delete</Button>
<Button variant="ghost">Cancel</Button>
```

## PropTypes (for type-checking — use TypeScript in production)

```jsx
import PropTypes from 'prop-types';

const UserProfile = ({ user, onUpdate, isActive }) => (
  <div className={`profile ${isActive ? 'active' : ''}`}>
    <h2>{user.name}</h2>
    <p>{user.email}</p>
  </div>
);

UserProfile.propTypes = {
  user: PropTypes.shape({
    id: PropTypes.number.isRequired,
    name: PropTypes.string.isRequired,
    email: PropTypes.string,
    role: PropTypes.oneOf(['admin', 'user', 'guest']),
    metadata: PropTypes.object,
  }).isRequired,
  onUpdate: PropTypes.func.isRequired,
  isActive: PropTypes.bool,
};

UserProfile.defaultProps = {
  isActive: false,
};
```

## Interview Tips
- Props are immutable (read-only) — never mutate props in child
- Prop drilling is a problem — Context API or state management solves it
- Use `React.memo` to prevent unnecessary re-renders on unchanged props
- Truthy/falsy checks with `&&` in JSX can render `0` unexpectedly — use ternary or `!!`
- Know the difference between controlled (state) and uncontrolled (ref) components

---

# 5. STATE — COMPONENT MEMORY

## Local State with useState

```jsx
const [state, setState] = useState(initialValue);

// Three ways to set state:
// 1. Direct value
setCount(10);

// 2. Functional updater (use when new state depends on old)
setCount(prev => prev + 1);

// 3. For objects — spread to maintain immutability
setUser(prev => ({ ...prev, name: 'John' }));
```

## Critical: State Updates Are Async & Batched

```jsx
const Counter = () => {
  const [count, setCount] = useState(0);

  const handleClick = () => {
    setCount(count + 1);  // Uses current count (0)
    setCount(count + 1);  // Still 0 — both use same stale value
    // Result: count is 1, not 2
  };

  const handleClickCorrect = () => {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1);
    // Result: count is 2 — functional updater uses correct previous value
  };

  // Reading state right after setting — BUG!
  const handleBug = () => {
    setCount(5);
    console.log(count); // ❌ Still old value! State not updated yet
  };

  // The only way to run code after state update:
  useEffect(() => {
    console.log('Count changed to:', count);
  }, [count]);
};
```

## Complex State Patterns

```jsx
// 1. Multiple state variables (preferred over single object for independent values)
const SignupForm = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [errors, setErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setIsSubmitting(true);
    try {
      // validate
      const validationErrors = validate({ email, password });
      if (Object.keys(validationErrors).length) {
        setErrors(validationErrors);
        return;
      }
      await api.register({ email, password });
    } finally {
      setIsSubmitting(false);
    }
  };
};

// 2. useReducer for complex state logic
const initialState = { count: 0, step: 1, max: 100 };

const reducer = (state, action) => {
  switch (action.type) {
    case 'INCREMENT':
      return { ...state, count: Math.min(state.count + state.step, state.max) };
    case 'DECREMENT':
      return { ...state, count: Math.max(state.count - state.step, 0) };
    case 'SET_STEP':
      return { ...state, step: action.payload };
    case 'RESET':
      return initialState;
    default:
      return state;
  }
};

const Counter = () => {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
      <input
        type="number"
        value={state.step}
        onChange={e => dispatch({ type: 'SET_STEP', payload: +e.target.value })}
      />
    </>
  );
};
```

## Real-World Scenarios

**Scenario 1: Shopping Cart State**
```jsx
const useShoppingCart = () => {
  const [cart, setCart] = useState({ items: [], total: 0 });

  const addItem = (product) => {
    setCart(prev => {
      const existing = prev.items.find(item => item.id === product.id);
      let newItems;

      if (existing) {
        newItems = prev.items.map(item =>
          item.id === product.id
            ? { ...item, quantity: item.quantity + 1 }
            : item
        );
      } else {
        newItems = [...prev.items, { ...product, quantity: 1 }];
      }

      const total = newItems.reduce((sum, item) => sum + item.price * item.quantity, 0);
      return { items: newItems, total };
    });
  };

  const removeItem = (productId) => {
    setCart(prev => {
      const newItems = prev.items.filter(item => item.id !== productId);
      const total = newItems.reduce((sum, item) => sum + item.price * item.quantity, 0);
      return { items: newItems, total };
    });
  };

  const updateQuantity = (productId, quantity) => {
    if (quantity <= 0) {
      removeItem(productId);
      return;
    }
    setCart(prev => {
      const newItems = prev.items.map(item =>
        item.id === productId ? { ...item, quantity } : item
      );
      const total = newItems.reduce((sum, item) => sum + item.price * item.quantity, 0);
      return { items: newItems, total };
    });
  };

  const clearCart = () => setCart({ items: [], total: 0 });

  return { cart, addItem, removeItem, updateQuantity, clearCart };
};
```

**Scenario 2: Multi-Step Form State**
```jsx
const useMultiStepForm = (steps) => {
  const [currentStep, setCurrentStep] = useState(0);
  const [formData, setFormData] = useState({});

  const next = (stepData) => {
    setFormData(prev => ({ ...prev, ...stepData }));
    setCurrentStep(i => Math.min(i + 1, steps.length - 1));
  };

  const back = () => setCurrentStep(i => Math.max(i - 1, 0));

  const goTo = (step) => setCurrentStep(step);

  return {
    currentStep,
    currentComponent: steps[currentStep],
    formData,
    next,
    back,
    goTo,
    isFirstStep: currentStep === 0,
    isLastStep: currentStep === steps.length - 1,
  };
};

// Usage
const CheckoutFlow = () => {
  const {
    currentComponent: StepComponent,
    formData,
    next,
    back,
    isFirstStep,
    isLastStep,
  } = useMultiStepForm([ShippingForm, PaymentForm, ReviewOrder, Confirmation]);

  return (
    <div className="checkout-flow">
      <ProgressBar currentStep={currentStep} totalSteps={4} />
      <StepComponent formData={formData} onNext={next} onBack={back} />
      <div className="navigation">
        {!isFirstStep && <button onClick={back}>Back</button>}
        {!isLastStep && <button onClick={handleStepSubmit}>Next</button>}
      </div>
    </div>
  );
};
```

**Scenario 3: Optimistic Updates**
```jsx
const TweetActions = ({ tweetId, initialLikes }) => {
  const [likes, setLikes] = useState(initialLikes);
  const [isLiked, setIsLiked] = useState(false);
  const [error, setError] = useState(null);

  const handleLike = async () => {
    // Optimistic update — update UI immediately
    setLikes(prev => isLiked ? prev - 1 : prev + 1);
    setIsLiked(prev => !prev);
    setError(null);

    try {
      // Actual API call
      await api.likeTweet(tweetId);
    } catch (err) {
      // Rollback on failure
      setLikes(prev => isLiked ? prev + 1 : prev - 1);
      setIsLiked(prev => !prev);
      setError('Failed to update like. Please try again.');
    }
  };

  return (
    <div>
      <button onClick={handleLike} className={isLiked ? 'liked' : ''}>
        ♥ {likes}
      </button>
      {error && <span className="error">{error}</span>}
    </div>
  );
};
```

## Interview Questions

**Q1:** Why is state mutation bad in React?
**A:** React detects changes via reference comparison. Mutating state (e.g., `arr.push(item)`) doesn't create a new reference, so React doesn't re-render. Always create new objects/arrays via spread, map, filter, or immutable libraries like Immer.

**Q2:** explain `setState` batching — when does React batch updates?
**A:** React batches state updates within event handlers, lifecycle methods, and effects. In React 18+, all updates are automatically batched including in timeouts, promises, and native events. Before 18, only React event handlers were batched.

**Q3:** What happens when you call `setState` in `useEffect`?
**A:** It triggers a re-render which runs `useEffect` cleanup (if dependencies change), then the effect again. This can cause infinite loops if you update state that's in the dependency array without condition.

**Q4:** How do you share state between components?
**A:** Lifting state up (common parent → props), Context API (for deep trees), or external state management (Redux, Zustand, Jotai). Choose based on scale: props for small, Context for medium, external lib for complex.

**Q5:** What is the `useRef` alternative for storing mutable values without re-render?
**A:** `useRef({ current: initialValue })` persists across renders without causing re-renders when changed. Use for timers, DOM refs, or any mutable value that shouldn't trigger UI updates.

---

# 6. EVENT HANDLING

React normalizes events across browsers using SyntheticEvent wrapper.

```jsx
const Form = () => {
  const handleSubmit = (e) => {
    e.preventDefault();  // Prevent default browser behavior
    // Form data is in e.target
    const formData = new FormData(e.target);
    console.log(Object.fromEntries(formData));
  };

  const handleChange = (e) => {
    // e.target.value — for input, select, textarea
    // e.target.checked — for checkboxes/radios
    const { name, value, type, checked } = e.target;
    setForm(prev => ({
      ...prev,
      [name]: type === 'checkbox' ? checked : value
    }));
  };

  // Debounced input
  const handleSearch = useCallback((e) => {
    // Debounce logic
    clearTimeout(timerRef.current);
    timerRef.current = setTimeout(() => {
      onSearch(e.target.value);
    }, 300);
  }, [onSearch]);

  return (
    <form onSubmit={handleSubmit}>
      <input name="email" onChange={handleChange} />
      <input name="agree" type="checkbox" onChange={handleChange} />
      <button type="submit">Submit</button>
    </form>
  );
};
```

## Event Pooling (React <17)
```jsx
// React <17: SyntheticEvent is pooled (reused) — async access fails
handleClick = (e) => {
  setTimeout(() => {
    console.log(e.target); // ❌ null — event was pooled
  }, 100);
};

// Fix: call e.persist() or extract values
handleClick = (e) => {
  const { target } = e;  // Extract before timeout
  setTimeout(() => {
    console.log(target); // ✅ works
  }, 100);
};

// React 17+: No pooling — events work in async code naturally
```

## Interview Questions

**Q1:** What is SyntheticEvent?
**A:** React's cross-browser wrapper around native DOM events. Provides consistent API across browsers. In React 17+, events are delegated to root node instead of document.

**Q2:** How do you pass custom parameters to event handlers?
**A:** `<button onClick={(e) => handleClick(id, e)}>` or use currying: `const handleClick = (id) => (e) => { ... }`.

**Q3:** Why avoid inline arrow functions in JSX?
**A:** Each render creates a new function, potentially breaking `React.memo` optimization. Use `useCallback` or extract to a named handler.

---

# 7. CONDITIONAL RENDERING — ALL 6 PATTERNS

```jsx
const ConditionalRender = ({ status, items, user }) => {
  // 1. If/Else (ternary)
  return (
    <div>
      <h1>{user.isAdmin ? 'Admin Panel' : 'Dashboard'}</h1>

      {/* 2. AND (&&) — render or nothing */}
      {status === 'loading' && <Spinner />}

      {/* 3. OR (||) — fallback */}
      {user.displayName || 'Anonymous'}

      {/* 4. IIFE - Immediately Invoked Function Expression */}
      {(() => {
        switch (status) {
          case 'loading': return <Spinner />;
          case 'error': return <Error />;
          case 'empty': return <EmptyState />;
          default: return <DataView />;
        }
      })()}

      {/* 5. Component with conditional logic inside */}
      <StatusRenderer status={status} />

      {/* 6. Conditional element in array */}
      {[
        <Header key="h" />,
        status !== 'loading' && <Content key="c" />,
        items.length > 0 && <Footer key="f" />,
      ].filter(Boolean)}
    </div>
  );
};
```

---

# 8. LISTS & KEYS

**Keys must be: stable, unique, and predictable.** Never use index as key if list can change.

```jsx
// ✅ GOOD: unique ID from data
{items.map(item => <Item key={item.id} data={item} />)}

// ✅ GOOD: stable ID generated once (useRef + uuid)
{items.map((item, index) => (
  <Item key={item.id || `item-${index}`} data={item} />
))}

// ❌ BAD: index as key (causes bugs with reordering/filtering)
{items.map((item, index) => <Item key={index} data={item} />)}

// ❌ BAD: Math.random() — keys change every render
{items.map(item => <Item key={Math.random()} data={item} />)}
```

## Key Impact on Reconciliation

```jsx
// WITHOUT keys — React reuses DOM nodes (wrong content may flash)
// WITH keys — React reorders/moves DOM nodes correctly

const TodoList = ({ todos }) => (
  <ul>
    {todos.map(todo => (
      <li key={todo.id}>
        <Checkbox checked={todo.done} />
        <span>{todo.text}</span>
        <button>✕</button>
      </li>
    ))}
  </ul>
);

// If todos is reordered:
// Without keys: React updates each <li>'s content (expensive, may cause focus/state loss)
// With keys: React moves DOM nodes (fast, preserves state)
```

## Interview Questions

**Q1:** Why is index as key problematic?
**A:** If items are reordered/filtered, the index changes causing React to unnecessarily unmount/remount components, losing state (e.g., input focus, scroll position, animation state).

**Q2:** Can two different items have the same key?
**A:** No. Keys must be unique among siblings. Duplicate keys cause unpredictable behavior — React may update wrong component.

**Q3:** What if data doesn't have a unique ID?
**A:** Generate one on data load: `items.map(item => ({ ...item, _key: nanoid() }))`, or use a combination of fields: `key={`${item.type}-${item.id}`}`.

---

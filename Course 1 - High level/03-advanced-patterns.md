# React Mastery Guide — Part 3: Advanced Patterns & State Management
## Enterprise-Level React Patterns

---

# 1. COMPOUND COMPONENTS PATTERN

**Definition:** A set of components that work together to create a cohesive UI, sharing implicit state via Context.

```jsx
// Accordion — The Classic Compound Component
const AccordionContext = createContext();

const Accordion = ({ children, defaultIndex = 0 }) => {
  const [activeIndex, setActiveIndex] = useState(defaultIndex);

  const value = useMemo(() => ({ activeIndex, setActiveIndex }), [activeIndex]);

  return (
    <AccordionContext.Provider value={value}>
      <div className="accordion">{children}</div>
    </AccordionContext.Provider>
  );
};

Accordion.Item = ({ index, children }) => {
  const { activeIndex, setActiveIndex } = useContext(AccordionContext);
  const isActive = activeIndex === index;

  return (
    <div className={`accordion-item ${isActive ? 'active' : ''}`}>
      {React.Children.map(children, child =>
        React.cloneElement(child, { isActive, onToggle: () => setActiveIndex(isActive ? null : index) })
      )}
    </div>
  );
};

Accordion.Header = ({ isActive, onToggle, children }) => (
  <button className="accordion-header" onClick={onToggle}>
    {children}
    <span className={`arrow ${isActive ? 'rotated' : ''}`}>▶</span>
  </button>
);

Accordion.Panel = ({ isActive, children }) => (
  <div className={`accordion-panel ${isActive ? 'show' : ''}`}>
    {children}
  </div>
);

// Usage
const FAQ = () => (
  <Accordion>
    <Accordion.Item index={0}>
      <Accordion.Header>What is React?</Accordion.Header>
      <Accordion.Panel>React is a JS library for building UIs.</Accordion.Panel>
    </Accordion.Item>
    <Accordion.Item index={1}>
      <Accordion.Header>What are hooks?</Accordion.Header>
      <Accordion.Panel>Hooks let you use state without classes.</Accordion.Panel>
    </Accordion.Item>
  </Accordion>
);
```

## Real-World: E-Commerce Product Configurator

```jsx
const ConfiguratorContext = createContext();

const ProductConfigurator = ({ children, basePrice }) => {
  const [selections, setSelections] = useState({});
  const totalPrice = useMemo(() => {
    const extras = Object.values(selections)
      .filter(Boolean)
      .reduce((sum, opt) => sum + (opt.price || 0), 0);
    return basePrice + extras;
  }, [selections, basePrice]);

  const value = useMemo(() => ({ selections, setSelections, totalPrice }), [selections, totalPrice]);

  return (
    <ConfiguratorContext.Provider value={value}>
      <div className="configurator">{children}</div>
    </ConfiguratorContext.Provider>
  );
};

Configurator.OptionGroup = ({ name, options, children }) => (
  <div className="option-group">
    <h4>{name}</h4>
    {options.map(opt => <Configurator.Option key={opt.id} groupName={name} option={opt} />)}
  </div>
);

Configurator.Option = ({ groupName, option }) => {
  const { selections, setSelections } = useContext(ConfiguratorContext);
  const isSelected = selections[groupName]?.id === option.id;

  return (
    <button
      className={`option ${isSelected ? 'selected' : ''}`}
      onClick={() => setSelections(prev => ({ ...prev, [groupName]: isSelected ? null : option }))}
    >
      {option.label} {option.price ? `(+$${option.price})` : ''}
    </button>
  );
};

Configurator.Summary = () => {
  const { selections, totalPrice } = useContext(ConfiguratorContext);
  return (
    <div className="summary">
      <h3>Total: ${totalPrice}</h3>
      <ul>
        {Object.entries(selections).map(([group, opt]) =>
          opt ? <li key={group}>{group}: {opt.label}</li> : null
        )}
      </ul>
    </div>
  );
};

// Usage
const LaptopConfigurator = () => (
  <ProductConfigurator basePrice={999}>
    <Configurator.OptionGroup
      name="RAM"
      options={[
        { id: '8gb', label: '8GB', price: 0 },
        { id: '16gb', label: '16GB', price: 200 },
        { id: '32gb', label: '32GB', price: 500 },
      ]}
    />
    <Configurator.OptionGroup
      name="Storage"
      options={[
        { id: '256', label: '256GB SSD', price: 0 },
        { id: '512', label: '512GB SSD', price: 150 },
        { id: '1tb', label: '1TB SSD', price: 350 },
      ]}
    />
    <Configurator.Summary />
  </ProductConfigurator>
);
```

# 2. CONTROLLED VS UNCONTROLLED COMPONENTS

## Uncontrolled (useRef — DOM manages state)
```jsx
const UncontrolledForm = () => {
  const inputRef = useRef(null);
  const handleSubmit = (e) => {
    e.preventDefault();
    alert(inputRef.current.value);
  };
  return (
    <form onSubmit={handleSubmit}>
      <input ref={inputRef} defaultValue="initial" />
      <button type="submit">Submit</button>
    </form>
  );
};
```

## Controlled (useState — React manages state)
```jsx
const ControlledForm = () => {
  const [value, setValue] = useState('initial');
  const handleSubmit = (e) => {
    e.preventDefault();
    alert(value);
  };
  return (
    <form onSubmit={handleSubmit}>
      <input value={value} onChange={e => setValue(e.target.value)} />
      <button type="submit">Submit</button>
    </form>
  );
};
```

## The useControlledSwitch Pattern
```jsx
// A component that works both controlled and uncontrolled
const useControlledSwitch = (controlledValue, defaultValue) => {
  const isControlled = controlledValue !== undefined;
  const [internalValue, setInternalValue] = useState(defaultValue);

  const value = isControlled ? controlledValue : internalValue;
  const setValue = (newValue) => {
    if (!isControlled) setInternalValue(newValue);
    // In controlled mode, parent manages state
  };

  return [value, setValue, isControlled];
};

const Toggle = ({ checked, defaultChecked, onChange }) => {
  const [isOn, setIsOn, isControlled] = useControlledSwitch(checked, defaultChecked || false);

  const handleClick = () => {
    const newValue = !isOn;
    setIsOn(newValue);
    onChange?.(newValue);
  };

  return (
    <button className={`toggle ${isOn ? 'on' : 'off'}`} onClick={handleClick}>
      {isOn ? 'ON' : 'OFF'}
    </button>
  );
};

// Uncontrolled usage: <Toggle defaultChecked={true} onChange={...} />
// Controlled usage: <Toggle checked={isChecked} onChange={setIsChecked} />
```

# 3. ERROR BOUNDARIES

**Error Boundaries catch JavaScript errors in their child tree, log them, and display fallback UI.**

```jsx
// ⚠️ Error Boundaries MUST be class components (React < 19 limitation)
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error) {
    // Update state so next render shows fallback
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log to monitoring service
    console.error('Error caught:', error, errorInfo);
    // Send to error tracking (Sentry, LogRocket, etc.)
    if (this.props.onError) {
      this.props.onError(error, errorInfo);
    }
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div className="error-boundary">
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message}</p>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Try Again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Usage
const App = () => (
  <ErrorBoundary fallback={<FullPageError />}>
    <ErrorBoundary fallback={<DashboardError />} onError={logToSentry}>
      <Dashboard />
    </ErrorBoundary>
    <ErrorBoundary fallback={<WidgetError />}>
      <AnalyticsWidget />
    </ErrorBoundary>
  </ErrorBoundary>
);
```

## With React 19 (captureError + new Error Boundaries)
```jsx
// React 19+: Error Boundaries can be functional with error hooks
// Also: use(promise) for Suspense
```

# 4. PORTALS — RENDER OUTSIDE DOM TREE

**Portals render children into a different DOM node, while maintaining React context.**

```jsx
// Modal using Portal
const ModalPortal = ({ children }) => {
  const [container] = useState(() => document.createElement('div'));

  useEffect(() => {
    const modalRoot = document.getElementById('modal-root');
    if (!modalRoot) {
      const root = document.createElement('div');
      root.id = 'modal-root';
      document.body.appendChild(root);
    }
    document.getElementById('modal-root').appendChild(container);
    return () => {
      document.getElementById('modal-root')?.removeChild(container);
    };
  }, [container]);

  return createPortal(children, container);
};

const Modal = ({ isOpen, onClose, title, children }) => {
  if (!isOpen) return null;

  return (
    <ModalPortal>
      <div className="modal-overlay" onClick={onClose}>
        <div className="modal-content" onClick={e => e.stopPropagation()}>
          <div className="modal-header">
            <h2>{title}</h2>
            <button onClick={onClose}>✕</button>
          </div>
          <div className="modal-body">{children}</div>
        </div>
      </div>
    </ModalPortal>
  );
};

// Usage — Modal renders at document.body level, not nested in DOM
const Page = () => {
  const [showModal, setShowModal] = useState(false);
  return (
    <>
      <button onClick={() => setShowModal(true)}>Show Terms</button>
      <Modal isOpen={showModal} onClose={() => setShowModal(false)} title="Terms">
        <p>All terms and conditions...</p>
      </Modal>
    </>
  );
};
```

# 5. REACT.MEMO — SMART RE-RENDER PREVENTION

```jsx
// Without React.memo: re-renders every time parent renders
const ExpensiveList = ({ items }) => (
  <ul>{items.map(item => <li key={item.id}>{item.name}</li>)}</ul>
);

// With React.memo: only re-renders when props change (shallow comparison)
const MemoizedList = React.memo(({ items }) => (
  <ul>{items.map(item => <li key={item.id}>{item.name}</li>)}</ul>
));

// Custom comparison function
const UserCard = React.memo(
  ({ user, onSelect }) => (
    <div className="card" onClick={() => onSelect(user.id)}>
      <Avatar src={user.avatar} />
      <h4>{user.name}</h4>
      <p>{user.email}</p>
    </div>
  ),
  (prev, next) => {
    return prev.user.id === next.user.id
      && prev.user.name === next.user.name
      && prev.user.avatar === next.user.avatar;
  }
);
```

# 6. CODE SPLITTING & LAZY LOADING

```jsx
import React, { Suspense, lazy } from 'react';

// Dynamic imports — code split at build time
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Analytics = lazy(() => import('./pages/Analytics'));
const Settings = lazy(() => import('./pages/Settings'));

// Loading fallback for each chunk
const PageLoader = () => <div className="page-loader"><Spinner /></div>;

const App = () => (
  <Suspense fallback={<PageLoader />}>
    <Routes>
      <Route path="/dashboard" element={<Dashboard />} />
      <Route path="/analytics" element={<Analytics />} />
      <Route path="/settings" element={<Settings />} />
    </Routes>
  </Suspense>
);

// Lazy load heavy components only when needed
const RichTextEditor = lazy(() => import('./RichTextEditor'));

const BlogEditor = () => {
  const [showEditor, setShowEditor] = useState(false);

  return (
    <div>
      <button onClick={() => setShowEditor(true)}>Start Writing</button>
      {showEditor && (
        <Suspense fallback={<EditorSkeleton />}>
          <RichTextEditor />
        </Suspense>
      )}
    </div>
  );
};
```

# 7. STATE MANAGEMENT COMPARISON

| Solution | Best For | Bundle Size | Learning Curve | Dev Experience |
|----------|----------|-------------|----------------|----------------|
| useState | Simple component state | 0kb | None | Excellent |
| useReducer | Complex component state | 0kb | Low | Good |
| Context API | Low-frequency global state | 0kb | Low | Good |
| Zustand | Medium-complexity global state | ~1kb | Low | Excellent |
| Redux Toolkit | Large-scale global state | ~12kb | Medium | Good (with RTK) |
| Jotai | Atomic state (like Recoil) | ~3kb | Low | Excellent |
| React Query/SWR | Server state (API data) | ~12kb | Low | Excellent |

## Zustand Example (Modern Choice)
```jsx
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

const useCartStore = create(
  devtools(
    persist(
      (set, get) => ({
        items: [],
        total: 0,
        itemCount: 0,

        addItem: (product) => set((state) => {
          const existing = state.items.find(item => item.id === product.id);
          let newItems;

          if (existing) {
            newItems = state.items.map(item =>
              item.id === product.id
                ? { ...item, quantity: item.quantity + 1 }
                : item
            );
          } else {
            newItems = [...state.items, { ...product, quantity: 1 }];
          }

          return {
            items: newItems,
            total: newItems.reduce((sum, i) => sum + i.price * i.quantity, 0),
            itemCount: newItems.reduce((sum, i) => sum + i.quantity, 0),
          };
        }),

        removeItem: (productId) => set((state) => {
          const newItems = state.items.filter(item => item.id !== productId);
          return {
            items: newItems,
            total: newItems.reduce((sum, i) => sum + i.price * i.quantity, 0),
            itemCount: newItems.reduce((sum, i) => sum + i.quantity, 0),
          };
        }),

        clearCart: () => set({ items: [], total: 0, itemCount: 0 }),
      }),
      { name: 'cart-storage' } // persist to localStorage
    )
  )
);

// Usage in component
const CartIcon = () => {
  const itemCount = useCartStore(state => state.itemCount); // Re-renders only when itemCount changes
  return <span>Cart ({itemCount})</span>;
};

const CheckoutButton = () => {
  const { items, total, clearCart } = useCartStore();
  return (
    <div>
      <p>Total: ${total}</p>
      <button onClick={handleCheckout}>Place Order</button>
    </div>
  );
};
```

# 8. PERFORMANCE OPTIMIZATION — THE SENIOR ENGINEER CHECKLIST

| Technique | What It Solves | How |
|-----------|---------------|-----|
| React.memo | Unnecessary child re-renders | Memoizes component |
| useMemo | Expensive recalculations | Memoizes values |
| useCallback | Stable function references | Memoizes callbacks |
| Virtualization | Large lists (10k+ items) | Only render visible items |
| Debouncing | Rapid input changes | Delay processing |
| Throttling | Scroll/resize events | Limit execution rate |
| Code Splitting | Large bundle size | Lazy load routes/components |
| Image Optimization | Large images | Lazy load, WebP, srcset |
| Web Workers | Heavy computation | Offload to worker thread |
| useTransition | Blocking updates | Mark non-urgent state updates |

## Virtualization Example (react-window)
```jsx
import { FixedSizeList } from 'react-window';

const ProductList = ({ products }) => {
  const Row = ({ index, style }) => {
    const product = products[index];
    return (
      <div style={style} className="product-row">
        <img src={product.thumbnail} alt={product.name} />
        <span>{product.name}</span>
        <span>${product.price}</span>
      </div>
    );
  };

  return (
    <FixedSizeList
      height={600}
      width="100%"
      itemCount={products.length}
      itemSize={80}
    >
      {Row}
    </FixedSizeList>
  );
};
```

## Profiling — How to Find Bottlenecks

```jsx
// 1. React DevTools Profiler — record interactions
// 2. Check what re-rendered and why

// 3. why-did-you-render library
import whyDidYouRender from '@welldone-software/why-did-you-render';
whyDidYouRender(React, { trackAllPureComponents: true });

// 4. Manual profiling
const ExpensiveComponent = React.memo(({ data }) => {
  const startTime = performance.now();
  // ... compute
  const duration = performance.now() - startTime;
  if (duration > 1) console.warn('Slow render:', duration, 'ms');
  return <div>{/* ... */}</div>;
});
```

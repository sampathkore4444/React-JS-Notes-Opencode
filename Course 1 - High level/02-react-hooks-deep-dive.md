# React Mastery Guide — Part 2: Hooks Deep Dive
## Master Every Hook Like a Senior Engineer

---

# 1. USEFFECT — THE SIDE EFFECT MASTER

**Definition:** useEffect synchronizes your component with external systems (API calls, subscriptions, DOM manipulations, timers).

## The Complete Mental Model

```
useEffect(setup, dependencies?)
  ├── setup: function that runs after render
  │   ├── runs on mount (initial render)
  │   ├── runs after dependency changes
  │   └── return cleanup function
  └── dependencies: array of values the effect depends on
      ├── undefined → runs after EVERY render
      ├── [] → runs only on mount/unmount
      └── [a, b] → runs when a or b change
```

## Every useEffect Pattern You'll Ever Need

```jsx
// PATTERN 1: Run once on mount (data fetching)
useEffect(() => {
  const fetchData = async () => {
    setLoading(true);
    try {
      const response = await api.getProducts();
      setProducts(response.data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  };
  fetchData();
}, []); // Empty array = componentDidMount

// PATTERN 2: Cleanup on unmount (subscriptions, listeners)
useEffect(() => {
  const subscription = websocket.subscribe('order-updates', handleUpdate);
  window.addEventListener('resize', handleResize);

  return () => {
    subscription.unsubscribe();
    window.removeEventListener('resize', handleResize);
  };
}, []); // Cleanup = componentWillUnmount

// PATTERN 3: Run when specific values change
useEffect(() => {
  document.title = `${unreadCount} unread messages`;
}, [unreadCount]);

// PATTERN 4: Debounced effect (search input)
useEffect(() => {
  const timer = setTimeout(() => {
    onSearch(searchTerm);
  }, 500);

  return () => clearTimeout(timer); // Cancel previous timer
}, [searchTerm]);

// PATTERN 5: Previous value tracking
useEffect(() => {
  if (prevCountRef.current !== undefined) {
    console.log(`Count changed from ${prevCountRef.current} to ${count}`);
  }
  prevCountRef.current = count;
}, [count]);

// PATTERN 6: Sequential dependent fetches
useEffect(() => {
  if (!userId) return;

  const fetchUserData = async () => {
    const user = await api.getUser(userId);
    const orders = await api.getUserOrders(user.id); // depends on user
    const recommendations = await api.getRecommendations(user.preferences);
    setData({ user, orders, recommendations });
  };
  fetchUserData();
}, [userId]);

// PATTERN 7: Race condition prevention
useEffect(() => {
  let cancelled = false;

  const fetchData = async () => {
    const result = await api.fetchProducts(category);
    if (!cancelled) { // ← Important: check before state update
      setProducts(result);
    }
  };
  fetchData();

  return () => {
    cancelled = true; // ← Prevents state update on unmounted component
  };
}, [category]);

// PATTERN 8: Polling / Interval
useEffect(() => {
  const fetchStatus = async () => {
    const status = await api.getOrderStatus(orderId);
    setOrderStatus(status);
    if (status === 'completed' || status === 'failed') {
      clearInterval(interval); // Stop polling
    }
  };

  fetchStatus(); // Immediate call
  const interval = setInterval(fetchStatus, 5000); // Then every 5s

  return () => clearInterval(interval);
}, [orderId]);

// PATTERN 9: Tracking state changes (componentDidUpdate)
const usePrevious = (value) => {
  const ref = useRef();
  useEffect(() => { ref.current = value; });
  return ref.current;
};

useEffect(() => {
  const prevUserId = usePrevious(userId);
  if (prevUserId !== userId && prevUserId !== undefined) {
    // User changed — log analytics
    analytics.track('user_switch', { from: prevUserId, to: userId });
  }
}, [userId]);

// PATTERN 10: Cleanup with AbortController (modern fetch)
useEffect(() => {
  const controller = new AbortController();

  const fetchProducts = async () => {
    try {
      const response = await fetch('/api/products', {
        signal: controller.signal
      });
      setProducts(await response.json());
    } catch (err) {
      if (err.name !== 'AbortError') {
        setError(err.message);
      }
    }
  };

  fetchProducts();
  return () => controller.abort(); // Cancel fetch on unmount/rerender
}, [category]);
```

## Common useEffect Mistakes

```jsx
// ❌ MISTAKE 1: Missing dependencies (stale closure)
const Counter = () => {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const interval = setInterval(() => {
      setCount(count + 1); // count is always 0 (stale closure)
    }, 1000);
    return () => clearInterval(interval);
  }, []); // Missing [count] — count is captured at mount time
  // FIX: use functional setState: setCount(c => c + 1)
};

// ❌ MISTAKE 2: Object/array in dependencies (infinite loop)
useEffect(() => {
  fetchData(filters);
}, [filters]); // filters is a new object every render → infinite loop
// FIX: use individual primitive deps or useMemo

// ❌ MISTAKE 3: Async effect without cleanup
useEffect(async () => {
  const data = await fetchData(); // ❌ async effect returns a Promise, not cleanup
}, []);
// FIX: define async function inside, call it, return cleanup
```

## Real-World Scenarios

**Scenario 1: Real-Time Dashboard with WebSocket + cleanup**
```jsx
const LiveDashboard = ({ userId }) => {
  const [metrics, setMetrics] = useState(null);
  const [connectionStatus, setConnectionStatus] = useState('disconnected');

  useEffect(() => {
    const ws = new WebSocket(`wss://api.example.com/dashboard/${userId}`);
    let heartbeatInterval;
    let reconnectTimer;
    let mounted = true;

    setConnectionStatus('connecting');

    ws.onopen = () => {
      if (!mounted) return;
      setConnectionStatus('connected');

      // Heartbeat to keep connection alive
      heartbeatInterval = setInterval(() => {
        ws.send(JSON.stringify({ type: 'ping' }));
      }, 30000);
    };

    ws.onmessage = (event) => {
      if (!mounted) return;
      try {
        const data = JSON.parse(event.data);
        setMetrics(prev => ({
          ...prev,
          [data.metric]: data.value,
          timestamp: data.timestamp,
        }));
      } catch (e) {
        console.error('Invalid message:', e);
      }
    };

    ws.onerror = () => {
      if (!mounted) return;
      setConnectionStatus('error');
    };

    ws.onclose = () => {
      setConnectionStatus('disconnected');
      // Auto reconnect with exponential backoff
      reconnectTimer = setTimeout(() => {
        // Reconnect logic
      }, 5000);
    };

    return () => {
      mounted = false;
      clearInterval(heartbeatInterval);
      clearTimeout(reconnectTimer);
      ws.close();
    };
  }, [userId]);

  return (
    <div className="dashboard">
      <StatusBadge status={connectionStatus} />
      {metrics && <MetricsDisplay data={metrics} />}
    </div>
  );
};
```

**Scenario 2: Form Auto-Save with Debounce**
```jsx
const DocumentEditor = ({ documentId }) => {
  const [content, setContent] = useState('');
  const [lastSaved, setLastSaved] = useState(null);
  const [dirty, setDirty] = useState(false);

  // Auto-save draft with debounce
  useEffect(() => {
    if (!dirty) return;

    const timer = setTimeout(async () => {
      try {
        await api.saveDraft(documentId, { content });
        setLastSaved(new Date());
        setDirty(false);
      } catch (err) {
        console.error('Auto-save failed:', err);
        // Could show a "Save failed" indicator
      }
    }, 2000); // 2 second debounce

    return () => clearTimeout(timer);
  }, [content, dirty, documentId]);

  // Warn user about unsaved changes
  useEffect(() => {
    const handleBeforeUnload = (e) => {
      if (dirty) {
        e.preventDefault();
        e.returnValue = 'You have unsaved changes.';
      }
    };
    window.addEventListener('beforeunload', handleBeforeUnload);
    return () => window.removeEventListener('beforeunload', handleBeforeUnload);
  }, [dirty]);

  return (
    <div className="editor">
      <textarea value={content} onChange={e => {
        setContent(e.target.value);
        setDirty(true);
      }} />
      <SaveIndicator lastSaved={lastSaved} dirty={dirty} />
    </div>
  );
};
```

**Scenario 3: Event Tracking / Analytics**
```jsx
const usePageTracking = (pageName, userProperties) => {
  useEffect(() => {
    analytics.pageView(pageName, userProperties);
  }, [pageName]); // Track when page changes

  useEffect(() => {
    if (!userProperties) return;
    analytics.identify(userProperties);
  }, [userProperties?.userId]); // Identify user on login
};

// Usage in component
const ProductPage = ({ productId }) => {
  const { product } = useProduct(productId);
  const [variant, setVariant] = useState(null);

  usePageTracking('product_detail', { productId, variant });

  useEffect(() => {
    const handleScroll = () => {
      // Track scroll depth for analytics
      const depth = Math.round((window.scrollY + window.innerHeight) /
        document.documentElement.scrollHeight * 100);
      if (depth >= 25 && depth >= 50 && depth >= 75 && depth >= 100) {
        analytics.track('scroll_depth', { productId, depth });
      }
    };
    window.addEventListener('scroll', handleScroll);
    return () => window.removeEventListener('scroll', handleScroll);
  }, [productId]);

  return <ProductDetail product={product} onVariantChange={setVariant} />;
};
```

## Interview Questions

**Q1:** How does useEffect decide whether to run the effect or skip it?
**A:** React compares each dependency with previous value using `Object.is`. If any changed, effect runs. If all same, skip.

**Q2:** What's the difference between `useEffect` and `useLayoutEffect`?
**A:** `useEffect` fires after paint (async, non-blocking). `useLayoutEffect` fires synchronously before paint (blocks visual updates). Use `useLayoutEffect` for DOM measurements/mutations that need to happen before user sees the update.

**Q3:** Why might an effect run twice in development?
**A:** React 18 StrictMode intentionally double-invokes effects in development to surface bugs with cleanup functions. It mounts → unmounts → remounts. This doesn't happen in production.

**Q4:** How do you handle cleanup when the effect depends on changing props?
**A:** Return a cleanup function from the effect. React runs the cleanup from the previous render before running the new effect (and on unmount).

**Q5:** What's the recommended way to fetch data with useEffect?
**A:** Inside the effect, define an async function and call it. Include cleanup to handle race conditions and abort requests. Consider using libraries like React Query/SWR which handle caching, deduplication, and more.

---

# 2. USECONTEXT — AVOID PROP DRILLING

## The Problem: Prop Drilling

```
App ──────────────────────────────────┐
 ├─ Dashboard (receives theme)       │
 │   └─ Widget (receives theme)      │ ← passing through multiple levels
 │       └─ Chart (receives theme)   │     just to reach Chart
```

## Context API Deep Dive

```jsx
// 1. CREATE CONTEXT
const ThemeContext = React.createContext('light'); // default value

// 2. PROVIDER (wraps component tree)
const App = () => (
  <ThemeContext.Provider value="dark">
    <Dashboard />
  </ThemeContext.Provider>
);

// 3. CONSUMER (hooks way — preferred)
const Chart = () => {
  const theme = useContext(ThemeContext);
  return <div className={`chart chart-${theme}`}>...</div>;
};
```

## Real-World Context Architecture

```jsx
// contexts/AuthContext.jsx
const AuthContext = createContext(null);

export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  // Check session on mount
  useEffect(() => {
    const checkAuth = async () => {
      try {
        const session = await api.checkSession();
        setUser(session.user);
      } catch {
        setUser(null);
      } finally {
        setLoading(false);
      }
    };
    checkAuth();
  }, []);

  const login = async (email, password) => {
    const { user, token } = await api.login(email, password);
    localStorage.setItem('token', token);
    setUser(user);
  };

  const logout = async () => {
    await api.logout();
    localStorage.removeItem('token');
    setUser(null);
  };

  const value = useMemo(() => ({
    user,
    loading,
    login,
    logout,
    isAuthenticated: !!user,
    role: user?.role,
  }), [user, loading]);

  return <AuthContext.Provider value={value}>{children}</AuthContext.Provider>;
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

## Performance: Context Splitting & Memoization

```jsx
// ❌ BAD: One big context that causes unnecessary re-renders
const AppContext = createContext();
const AppProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');
  const [notifications, setNotifications] = useState([]);
  // Any state change → ALL consumers re-render
};

// ✅ GOOD: Split contexts by concern
const UserContext = createContext();
const ThemeContext = createContext();
const NotificationContext = createContext();

// Even better: split value and setter
const UserStateContext = createContext();
const UserDispatchContext = createContext();

const UserProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  return (
    <UserStateContext.Provider value={user}>
      <UserDispatchContext.Provider value={setUser}>
        {children}
      </UserDispatchContext.Provider>
    </UserStateContext.Provider>
  );
};

const useUser = () => useContext(UserStateContext);
const useSetUser = () => useContext(UserDispatchContext);
```

## Interview Questions

**Q1:** Context vs Redux — when to use which?
**A:** Use Context for low-frequency updates (theme, auth, locale). Use Redux/Zustand for high-frequency global state (cart, real-time data) because Context causes all consumers to re-render on any value change.

**Q2:** How do you prevent unnecessary re-renders with Context?
**A:** Split contexts by domain; memoize context value with `useMemo`; split value/dispatch into separate contexts; or use libraries like Zustand that have fine-grained subscriptions.

**Q3:** Can you update context from a nested component?
**A:** Yes. Pass a setter function or dispatch function through context value. Best practice: use separate contexts for state and dispatch to prevent unnecessary re-renders.

---

# 3. USEREF — BEYOND DOM REFERENCES

**useRef** persists a mutable value across renders without causing re-renders when changed.

## All Use Cases

```jsx
const UseRefExamples = () => {
  // 1. DOM References
  const inputRef = useRef(null);
  const focusInput = () => inputRef.current?.focus();

  // 2. Store previous values
  const prevCountRef = useRef();
  useEffect(() => { prevCountRef.current = count; });

  // 3. Mutable instance variables (no re-render)
  const timerRef = useRef(null);
  const startTimer = () => {
    timerRef.current = setInterval(tick, 1000);
  };
  const stopTimer = () => {
    clearInterval(timerRef.current);
    timerRef.current = null;
  };

  // 4. Store callback (avoid stale closure in useEffect)
  const callbackRef = useRef(onChange);
  useEffect(() => { callbackRef.current = onChange; });
  useEffect(() => {
    const id = setInterval(() => {
      callbackRef.current(); // Always the latest callback
    }, 1000);
    return () => clearInterval(id);
  }, []); // No dependency on onChange!

  // 5. IntersectionObserver for infinite scroll
  const observerRef = useRef(null);
  const lastElementRef = useCallback(node => {
    if (observerRef.current) observerRef.current.disconnect();
    observerRef.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting) onLoadMore();
    });
    if (node) observerRef.current.observe(node);
  }, [onLoadMore]);

  // 6. Flag for mounted/unmounted state
  const isMounted = useRef(true);
  useEffect(() => () => { isMounted.current = false; }, []);

  // 7. Store previous props/state for comparison
  const prevPropsRef = useRef();
  useEffect(() => { prevPropsRef.current = props; });

  return (
    <>
      <input ref={inputRef} />
      <button onClick={focusInput}>Focus</button>
      <div ref={lastElementRef}>Load more...</div>
    </>
  );
};
```

## Interview Questions

**Q1:** What is the difference between `useRef` and `useState`?
**A:** `useRef` doesn't cause re-render when value changes; `useState` does. Use `useRef` for mutable values that shouldn't trigger UI updates (timers, DOM refs, callback references).

**Q2:** Can you use `useRef` to store a callback and avoid useEffect dependency issues?
**A:** Yes. Store callback in ref (`callbackRef.current = callback`), then use `callbackRef.current()` inside effect. This lets you keep effect dependencies empty while always calling latest callback.

---

# 4. USEMEMO & USECALLBACK — PERFORMANCE OPTIMIZATION

## useMemo — Memoize Computed Values

```jsx
// ❌ WITHOUT useMemo: recalculates every render
const TotalPrice = ({ items, discountCode }) => {
  const total = items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  const discount = calculateDiscount(discountCode, total); // Expensive!
  const finalPrice = total - discount;
  return <p>Total: ${finalPrice}</p>;
};

// ✅ WITH useMemo: only recalculates when dependencies change
const TotalPrice = ({ items, discountCode }) => {
  const total = useMemo(() =>
    items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    [items]
  );

  const discount = useMemo(() =>
    calculateDiscount(discountCode, total),
    [discountCode, total]
  );

  const finalPrice = useMemo(() => total - discount, [total, discount]);

  return <p>Total: ${finalPrice}</p>;
};
```

## useCallback — Memoize Functions

```jsx
// ❌ WITHOUT useCallback: new function created every render
// This breaks React.memo optimization on child components
const ProductList = ({ products }) => {
  const handleAddToCart = (product) => {
    dispatch({ type: 'ADD_TO_CART', payload: product });
  };
  return products.map(p => (
    <ProductCard key={p.id} product={p} onAddToCart={handleAddToCart} />
  ));
};

// ✅ WITH useCallback: stable function reference
const ProductList = ({ products }) => {
  const handleAddToCart = useCallback((product) => {
    dispatch({ type: 'ADD_TO_CART', payload: product });
  }, []); // If dispatch is stable (it is with useReducer or Redux)

  return products.map(p => (
    <ProductCard key={p.id} product={p} onAddToCart={handleAddToCart} />
  ));
};

// ✅ With dependencies:
const handleSubmit = useCallback(async (formData) => {
  setSubmitting(true);
  try {
    await api.updateUser(userId, { ...formData, email }); // depends on email
    setSuccess(true);
  } finally {
    setSubmitting(false);
  }
}, [userId, email]); // Must include all referenced values
```

## Real-World Scenario: Expensive Computation

```jsx
const DataTable = ({ rows, filters, sortColumn, sortDirection }) => {
  // Expensive filtering + sorting + pagination
  const processedData = useMemo(() => {
    console.time('processData');

    // 1. Filter
    let filtered = rows.filter(row => {
      return Object.entries(filters).every(([key, value]) => {
        if (!value) return true;
        return String(row[key]).toLowerCase().includes(value.toLowerCase());
      });
    });

    // 2. Sort
    if (sortColumn) {
      filtered.sort((a, b) => {
        const multiplier = sortDirection === 'asc' ? 1 : -1;
        if (a[sortColumn] < b[sortColumn]) return -1 * multiplier;
        if (a[sortColumn] > b[sortColumn]) return 1 * multiplier;
        return 0;
      });
    }

    console.timeEnd('processData');
    return filtered;
  }, [rows, filters, sortColumn, sortDirection]);

  // Pagination derived from filtered data
  const paginatedData = useMemo(() => {
    return processedData.slice(page * pageSize, (page + 1) * pageSize);
  }, [processedData, page, pageSize]);

  // Unique values for filter dropdowns
  const filterOptions = useMemo(() => {
    const options = {};
    Object.keys(filters).forEach(key => {
      options[key] = [...new Set(rows.map(r => r[key]))];
    });
    return options;
  }, [rows]);

  return <table>{/* render paginatedData */}</table>;
};
```

## The Rules: When to Memoize

| Scenario | Should You Memoize? | Why |
|----------|-------------------|-----|
| Array.map with children | ✅ Yes | Prevents child re-renders |
| Expensive calculations (>1ms) | ✅ Yes | Avoids repeated work |
| Values passed to Context | ✅ Yes | Prevents consumer re-renders |
| Simple primitives (strings, numbers) | ❌ No | No reference equality concern |
| Inline styles | ❌ No | Use className instead |
| Everything "just in case" | ❌ No | Adds overhead, hurts readability |

## Interview Questions

**Q1:** Does `useMemo` guarantee the value won't be recomputed?
**A:** No. React may clear the cache (e.g., memory pressure). Never write code that depends on `useMemo` not recomputing. It's a performance hint, not a guarantee.

**Q2:** What is the difference between `useMemo` and `useCallback`?
**A:** `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`. `useCallback` returns the memoized function itself; `useMemo` returns the result of calling the function.

**Q3:** Is `useMemo`/`useCallback` free?
**A:** No. They add memory overhead and comparison logic. Overusing them can actually harm performance. Only use when you've identified a performance bottleneck via profiling.

---

# 5. CUSTOM HOOKS — YOUR SECRET WEAPON

**Custom hooks extract reusable logic from components.** This is the most powerful pattern in React.

## The Custom Hook Pattern

```jsx
// A custom hook is just a function starting with "use"
// that uses other hooks internally
const useLocalStorage = (key, initialValue) => {
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(error);
      return initialValue;
    }
  });

  const setValue = useCallback((value) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(error);
    }
  }, [key, storedValue]);

  return [storedValue, setValue];
};
```

## Essential Custom Hooks Library

```jsx
// ---- HOOK 1: useAsync (Data Fetching) ----
const useAsync = (asyncFunction, immediate = true) => {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(false);

  const execute = useCallback(async (...args) => {
    setLoading(true);
    setError(null);
    try {
      const result = await asyncFunction(...args);
      setData(result);
      return result;
    } catch (err) {
      setError(err);
      throw err;
    } finally {
      setLoading(false);
    }
  }, [asyncFunction]);

  useEffect(() => {
    if (immediate) execute();
  }, [execute, immediate]);

  return { data, error, loading, execute };
};

// ---- HOOK 2: useDebounce ----
const useDebounce = (value, delay = 500) => {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
};

// ---- HOOK 3: useMediaQuery (Responsive) ----
const useMediaQuery = (query) => {
  const [matches, setMatches] = useState(() => window.matchMedia(query).matches);

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (e) => setMatches(e.matches);
    mediaQuery.addEventListener('change', handler);
    return () => mediaQuery.removeEventListener('change', handler);
  }, [query]);

  return matches;
};

// Usage: const isMobile = useMediaQuery('(max-width: 768px)');

// ---- HOOK 4: useIntersectionObserver ----
const useIntersectionObserver = (options = {}) => {
  const [entry, setEntry] = useState(null);
  const ref = useRef(null);

  useEffect(() => {
    const element = ref.current;
    if (!element) return;

    const observer = new IntersectionObserver(([entry]) => {
      setEntry(entry);
    }, { threshold: 0.1, ...options });

    observer.observe(element);
    return () => observer.disconnect();
  }, [options.threshold, options.rootMargin]);

  return [ref, entry];
};

// Usage: const [ref, entry] = useIntersectionObserver();
// if (entry?.isIntersecting) loadMore();

// ---- HOOK 5: useOnlineStatus ----
const useOnlineStatus = () => {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);
    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  return isOnline;
};

// ---- HOOK 6: useForm (Form State Management) ----
const useForm = (initialValues = {}) => {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});

  const handleChange = useCallback((e) => {
    const { name, value, type, checked } = e.target;
    const newValue = type === 'checkbox' ? checked : value;
    setValues(prev => ({ ...prev, [name]: newValue }));
    // Clear error when user types
    if (errors[name]) setErrors(prev => ({ ...prev, [name]: '' }));
  }, [errors]);

  const handleBlur = useCallback((e) => {
    setTouched(prev => ({ ...prev, [e.target.name]: true }));
  }, []);

  const setFieldValue = useCallback((name, value) => {
    setValues(prev => ({ ...prev, [name]: value }));
  }, []);

  const setFieldError = useCallback((name, error) => {
    setErrors(prev => ({ ...prev, [name]: error }));
  }, []);

  const resetForm = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
  }, [initialValues]);

  const validate = useCallback((validationRules) => {
    const newErrors = {};
    Object.entries(validationRules).forEach(([field, rules]) => {
      rules.some(rule => {
        if (rule.required && !values[field]) {
          newErrors[field] = rule.message;
          return true;
        }
        if (rule.pattern && !rule.pattern.test(values[field])) {
          newErrors[field] = rule.message;
          return true;
        }
        return false;
      });
    });
    setErrors(newErrors);
    return Object.keys(newErrors).length === 0;
  }, [values]);

  return {
    values, errors, touched,
    handleChange, handleBlur,
    setFieldValue, setFieldError,
    resetForm, validate,
    setValues, setErrors,
  };
};

// ---- HOOK 7: useTimeout ----
const useTimeout = (callback, delay) => {
  const savedCallback = useRef(callback);

  useEffect(() => { savedCallback.current = callback; }, [callback]);

  useEffect(() => {
    if (delay === null) return;
    const id = setTimeout(() => savedCallback.current(), delay);
    return () => clearTimeout(id);
  }, [delay]);
};

// ---- HOOK 8: useKeyPress ----
const useKeyPress = (targetKey) => {
  const [keyPressed, setKeyPressed] = useState(false);

  useEffect(() => {
    const downHandler = ({ key }) => {
      if (key === targetKey) setKeyPressed(true);
    };
    const upHandler = ({ key }) => {
      if (key === targetKey) setKeyPressed(false);
    };

    window.addEventListener('keydown', downHandler);
    window.addEventListener('keyup', upHandler);
    return () => {
      window.removeEventListener('keydown', downHandler);
      window.removeEventListener('keyup', upHandler);
    };
  }, [targetKey]);

  return keyPressed;
};

// ---- HOOK 9: useFetch (with caching) ----
const cache = new Map();

const useFetch = (url, options = {}) => {
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false;
    const fetchData = async () => {
      // Check cache
      if (cache.has(url)) {
        setData(cache.get(url));
        setLoading(false);
        return;
      }

      setLoading(true);
      try {
        const response = await fetch(url, options);
        if (!response.ok) throw new Error(`HTTP ${response.status}`);
        const json = await response.json();
        if (!cancelled) {
          cache.set(url, json);
          setData(json);
          setError(null);
        }
      } catch (err) {
        if (!cancelled) setError(err.message);
      } finally {
        if (!cancelled) setLoading(false);
      }
    };

    fetchData();
    return () => { cancelled = true; };
  }, [url]);

  const refetch = useCallback(async () => {
    cache.delete(url);
    // Re-trigger effect by changing a state
  }, [url]);

  return { data, error, loading, refetch };
};

// ---- HOOK 10: useUndo (History/Undo) ----
const useUndo = (initialPresent) => {
  const [state, setState] = useState({
    past: [],
    present: initialPresent,
    future: [],
  });

  const canUndo = state.past.length > 0;
  const canRedo = state.future.length > 0;

  const undo = useCallback(() => {
    setState(prev => {
      if (!prev.past.length) return prev;
      const previous = prev.past[prev.past.length - 1];
      const newPast = prev.past.slice(0, -1);
      return {
        past: newPast,
        present: previous,
        future: [prev.present, ...prev.future],
      };
    });
  }, []);

  const redo = useCallback(() => {
    setState(prev => {
      if (!prev.future.length) return prev;
      const next = prev.future[0];
      const newFuture = prev.future.slice(1);
      return {
        past: [...prev.past, prev.present],
        present: next,
        future: newFuture,
      };
    });
  }, []);

  const set = useCallback((newPresent) => {
    setState(prev => ({
      past: [...prev.past, prev.present],
      present: newPresent,
      future: [],
    }));
  }, []);

  const reset = useCallback((newPresent) => {
    setState({ past: [], present: newPresent, future: [] });
  }, []);

  return { state: state.present, set, reset, undo, redo, canUndo, canRedo };
};
```

## Custom Hook Composition Example

```jsx
//// hooks/useProducts.js
const useProducts = (category, searchTerm) => {
  const debouncedSearch = useDebounce(searchTerm, 300);
  const isOnline = useOnlineStatus();

  const url = useMemo(() => {
    const params = new URLSearchParams();
    if (category) params.set('category', category);
    if (debouncedSearch) params.set('q', debouncedSearch);
    return `/api/products?${params}`;
  }, [category, debouncedSearch]);

  const { data, loading, error, refetch } = useFetch(url);
  const [ref, entry] = useIntersectionObserver();

  // Infinite scroll
  useEffect(() => {
    if (entry?.isIntersecting && !loading && !error) {
      refetch();
    }
  }, [entry?.isIntersecting, loading, error, refetch]);

  return {
    products: data?.products ?? [],
    total: data?.total ?? 0,
    loading,
    error: !isOnline ? 'You are offline' : error,
    ref,
    refetch,
  };
};
```

## Interview Questions

**Q1:** What are the rules of custom hooks?
**A:** (1) Must start with "use". (2) Can only call hooks at the top level of a React function or another custom hook. (3) Should be pure — avoid side effects unrelated to the hook's purpose.

**Q2:** Can custom hooks take other hooks as parameters?
**A:** No. But they can accept functions that may use hooks internally, as long as the calling convention is respected.

**Q3:** How do you test a custom hook?
**A:** Use `renderHook` from `@testing-library/react-hooks` or `@testing-library/react` v14+. Test the hook's return values and state changes by calling its returned functions.

---

# 6. USEREDUCER — COMPLEX STATE MANAGEMENT

## Comparison: useState vs useReducer

| useState | useReducer |
|----------|------------|
| Simple state (primitives, small objects) | Complex state (nested objects, multiple fields) |
| Independent state variables | Interdependent state transitions |
| 2-3 related state values | State that depends on previous state in complex ways |
| Easy to read for simple cases | Better for complex logic, easier to test |

## Banking Transaction Example

```jsx
// State
const initialState = {
  balance: 10000,
  transactions: [],
  isProcessing: false,
  error: null,
  overdraftProtection: true,
};

// Actions
const ACTIONS = {
  DEPOSIT: 'DEPOSIT',
  WITHDRAW: 'WITHDRAW',
  TRANSFER_START: 'TRANSFER_START',
  TRANSFER_SUCCESS: 'TRANSFER_SUCCESS',
  TRANSFER_FAIL: 'TRANSFER_FAIL',
  TOGGLE_OVERDRAFT: 'TOGGLE_OVERDRAFT',
};

// Reducer — pure function, easy to test
const accountReducer = (state, action) => {
  switch (action.type) {
    case ACTIONS.DEPOSIT:
      return {
        ...state,
        balance: state.balance + action.payload.amount,
        transactions: [{
          id: nanoid(),
          type: 'deposit',
          amount: action.payload.amount,
          date: new Date(),
          description: action.payload.description,
        }, ...state.transactions],
      };

    case ACTIONS.WITHDRAW:
      // Check overdraft protection
      if (state.overdraftProtection && state.balance < action.payload.amount) {
        return {
          ...state,
          error: 'Insufficient funds. Transaction declined.',
        };
      }
      return {
        ...state,
        balance: state.balance - action.payload.amount,
        transactions: [{
          id: nanoid(),
          type: 'withdrawal',
          amount: -action.payload.amount,
          date: new Date(),
          description: action.payload.description,
        }, ...state.transactions],
        error: null,
      };

    case ACTIONS.TRANSFER_START:
      return { ...state, isProcessing: true, error: null };

    case ACTIONS.TRANSFER_SUCCESS:
      return {
        ...state,
        balance: state.balance - action.payload.amount,
        isProcessing: false,
        transactions: [{
          id: nanoid(),
          type: 'transfer_out',
          amount: -action.payload.amount,
          date: new Date(),
          description: `Transfer to ${action.payload.recipient}`,
        }, ...state.transactions],
      };

    case ACTIONS.TRANSFER_FAIL:
      return {
        ...state,
        isProcessing: false,
        error: action.payload.error,
      };

    case ACTIONS.TOGGLE_OVERDRAFT:
      return { ...state, overdraftProtection: !state.overdraftProtection };

    default:
      return state;
  }
};

// Component
const AccountDashboard = () => {
  const [state, dispatch] = useReducer(accountReducer, initialState);

  const handleTransfer = async (recipient, amount) => {
    dispatch({ type: ACTIONS.TRANSFER_START });
    try {
      await api.transfer(recipient, amount);
      dispatch({
        type: ACTIONS.TRANSFER_SUCCESS,
        payload: { recipient, amount },
      });
    } catch (error) {
      dispatch({
        type: ACTIONS.TRANSFER_FAIL,
        payload: { error: error.message },
      });
    }
  };

  return (
    <div className="account-dashboard">
      <Balance amount={state.balance} />
      <button onClick={() => dispatch({ type: ACTIONS.TOGGLE_OVERDRAFT })}>
        Overdraft Protection: {state.overdraftProtection ? 'ON' : 'OFF'}
      </button>
      {state.error && <ErrorBanner message={state.error} />}
      {state.isProcessing && <Spinner />}
      <TransactionHistory transactions={state.transactions} />
    </div>
  );
};

// Testing the reducer (pure function — easy!)
describe('accountReducer', () => {
  it('handles deposit', () => {
    const state = { balance: 1000, transactions: [], ... };
    const newState = accountReducer(state, {
      type: 'DEPOSIT',
      payload: { amount: 500, description: 'Salary' },
    });
    expect(newState.balance).toBe(1500);
    expect(newState.transactions).toHaveLength(1);
  });
});
```

---

# 7. RULES OF HOOKS — COMMANDMENTS

1. **Only call hooks at the top level** — not inside conditions, loops, or nested functions
2. **Only call hooks from React functions** — React function components or custom hooks
3. **Hooks must be called in the same order every render** — React relies on call order to maintain state between hooks

```jsx
// ❌ WRONG: Conditional hook
const Profile = ({ user }) => {
  if (user) {
    useEffect(() => { /* ... */ }); // Violation!
  }
};

// ✅ RIGHT: Move condition inside
const Profile = ({ user }) => {
  useEffect(() => {
    if (!user) return;
    // ...
  }, [user]);
};

// ❌ WRONG: Hook in loop
const Items = ({ items }) => {
  items.forEach(item => {
    const [state, setState] = useState(item); // Violation!
  });
};

// ✅ RIGHT: Use array state
const Items = ({ items }) => {
  const [states, setStates] = useState(items.map(item => ({ ...item })));
};
```

## Why This Matters

React hooks are implemented as a **linked list** per component. The order of hooks determines which state belongs to which hook call. If order changes between renders, React gets confused.

```jsx
// Render 1: useState('a') → 'a', useState('b') → 'b'
// Render 2: useState('a') → 'a',          ← still 'a' (matched by position)
//            useState('b') → 'b'          ← still 'b' (matched by position)
// ✅ Works because order is consistent

// ❌ BROKEN: If first render has 2 hooks, second has 1
// React would return the wrong state!
```

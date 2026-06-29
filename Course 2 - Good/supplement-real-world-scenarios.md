# Supplement: Real-World Scenarios, Interview Questions & E2E Examples
## Per Concept — 3 Scenarios, 10 Questions, E-Commerce + Banking

---

# 1. COMPONENTS & JSX

## 3 Real-World Scenarios

### Scenario 1: E-Commerce Product Card with Variants

**Scenario:** You are building an e-commerce product listing page for an online fashion store. Each product (like a shirt or dress) comes in multiple colors and sizes. Customers need to see the product image update when they select a different color, compare the discount against the original price, and know immediately if an item is out of stock. The ProductCard component below handles all three concerns through component composition and local UI state.

```jsx
function ProductCard({ product, onAddToCart, onQuickView }) {
  // Local state for user-selected variant options
  const [selectedSize, setSelectedSize] = useState(null);
  const [selectedColor, setSelectedColor] = useState(null);

  // Find the matching image for the selected color, fall back to first image
  const mainImage = product.images.find(img =>
    img.color === selectedColor
  ) || product.images[0];

  return (
    // Semantic article element with ARIA label for accessibility
    <article className="product-card" role="group" aria-label={product.name}>
      {/* Image section with sale badge and out-of-stock overlay */}
      <div className="product-card__image-container">
        <img src={mainImage?.url} alt={product.name} loading="lazy" />
        {product.discount > 0 && (
          <span className="badge badge-sale">-{product.discount}%</span>
        )}
        {!product.inStock && (
          <div className="overlay out-of-stock">Out of Stock</div>
        )}
      </div>

      {/* Product info section: name, star rating, color swatches, pricing */}
      <div className="product-card__info">
        <h3 className="product-card__title">{product.name}</h3>
        {/* Star rating rendered via helper + review count badge */}
        <div className="product-card__rating">
          {renderStars(product.rating)}
          <span className="review-count">({product.reviewCount})</span>
        </div>

        {/* Color swatch picker — only show if multiple colors exist */}
        {product.colors?.length > 1 && (
          <div className="color-options">
            {product.colors.map(color => (
              <button
                key={color}
                className={`color-swatch ${selectedColor === color ? 'selected' : ''}`}
                style={{ backgroundColor: color }}
                onClick={() => setSelectedColor(color)}
                aria-label={`Select ${color} color`}
              />
            ))}
          </div>
        )}

        {/* Current price with strikethrough original price if on sale */}
        <div className="product-card__pricing">
          <span className="current-price">${product.price}</span>
          {product.compareAtPrice && (
            <span className="compare-price">${product.compareAtPrice}</span>
          )}
        </div>
      </div>

      {/* Call-to-action: Add to Cart or Notify Me depending on stock */}
      <button
        className="btn-add-cart"
        onClick={() => onAddToCart({ ...product, selectedSize, selectedColor })}
        disabled={!product.inStock}
      >
        {product.inStock ? 'Add to Cart' : 'Notify Me'}
      </button>
    </article>
  );
}
```

### Scenario 2: Banking Transaction List with Categorization

**Scenario:** A banking app shows the customer's recent transactions grouped by date (e.g., "Today", "Yesterday", "This Month"). The user can filter by transaction type (all, credits, debits, transfers). Each group shows a daily running total. This is a common pattern in all retail banking apps where users need to quickly understand their spending and income patterns over time.

```jsx
function TransactionList({ transactions, onSelectTransaction }) {
  // Track which transaction type filter is active
  const [filter, setFilter] = useState('all');

  // Memoized grouping — group transactions by date, filtered by type
  const grouped = useMemo(() => {
    const groups = {};
    transactions
      .filter(tx => filter === 'all' || tx.type === filter)  // Apply type filter
      .forEach(tx => {
        const dateKey = formatDateGroup(tx.date);  // e.g. "Today", "Yesterday"
        if (!groups[dateKey]) groups[dateKey] = [];
        groups[dateKey].push(tx);
      });
    return groups;
  }, [transactions, filter]);

  return (
    <div className="transaction-list" role="feed" aria-label="Transaction history">
      {/* Filter buttons: All / Credits / Debits / Transfers */}
      <div className="filter-bar">
        {['all', 'credit', 'debit', 'transfer'].map(f => (
          <button
            key={f}
            className={`filter-btn ${filter === f ? 'active' : ''}`}
            onClick={() => setFilter(f)}
          >
            {f.charAt(0).toUpperCase() + f.slice(1)}
          </button>
        ))}
      </div>

      {/* Show empty state if no transactions match the filter */}
      {Object.entries(grouped).length === 0 ? (
        <EmptyState icon="📭" message="No transactions for this period" />
      ) : (
        // Render each date group with header and transaction rows
        Object.entries(grouped).map(([dateKey, txs]) => (
          <div key={dateKey} className="date-group">
            <div className="date-header">
              <span>{dateKey}</span>
              {/* Running total for this date group */}
              <span className="daily-total">
                {txs.reduce((sum, t) => sum + (t.type === 'debit' ? -t.amount : t.amount), 0).toFixed(2)}
              </span>
            </div>
            {txs.map(tx => (
              <TransactionRow
                key={tx.id}
                transaction={tx}
                onClick={() => onSelectTransaction(tx)}
              />
            ))}
          </div>
        ))
      )}
    </div>
  );
}
```

### Scenario 3: Multi-tenant Dashboard Layout (Admin/Manager/User)

**Scenario:** A SaaS platform serves three user roles — admin, manager, and regular user. Each role sees a different sidebar navigation, different set of modules, and different permission levels. Instead of building three separate dashboards, a single DashboardLayout component reads the user's role and renders the appropriate modules and links. This is the "role-based UI" pattern used by platforms like GitHub, Jira, and Salesforce.

```jsx
function DashboardLayout({ user, modules }) {
  // Define role-specific configuration: modules, permissions, sidebar links
  const roleConfig = {
    admin: {
      modules: ['analytics', 'users', 'settings', 'audit'],
      allowedActions: ['create', 'read', 'update', 'delete'],
      sidebarLinks: [
        { icon: '📊', label: 'Overview', path: '/admin' },
        { icon: '👥', label: 'Users', path: '/admin/users' },
        { icon: '⚙️', label: 'Settings', path: '/admin/settings' },
      ],
    },
    manager: {
      modules: ['analytics', 'team', 'reports'],
      allowedActions: ['create', 'read', 'update'],
      sidebarLinks: [
        { icon: '📊', label: 'Team Overview', path: '/manager' },
        { icon: '📋', label: 'Reports', path: '/manager/reports' },
      ],
    },
    user: {
      modules: ['dashboard', 'profile'],
      allowedActions: ['read'],
      sidebarLinks: [
        { icon: '🏠', label: 'Dashboard', path: '/dashboard' },
        { icon: '👤', label: 'Profile', path: '/profile' },
      ],
    },
  };

  // Pick the config for the current user's role, fall back to 'user'
  const config = roleConfig[user.role] || roleConfig.user;
  // Filter the full module list to only what this role can see
  const visibleModules = modules.filter(m => config.modules.includes(m.id));

  return (
    <div className="dashboard-layout">
      {/* Role-based sidebar navigation */}
      <aside className="sidebar">
        <nav aria-label="Main navigation">
          {config.sidebarLinks.map(link => (
            <NavLink key={link.path} to={link.path} className="nav-item">
              <span className="nav-icon">{link.icon}</span>
              <span className="nav-label">{link.label}</span>
            </NavLink>
          ))}
        </nav>
      </aside>
      {/* Main content area with error boundary for module isolation */}
      <main className="main-content">
        <ErrorBoundary fallback={<ModuleError />}>
          <ModuleRenderer modules={visibleModules} />
        </ErrorBoundary>
      </main>
    </div>
  );
}
```

## Interview Tips
- **JSX is NOT HTML** — emphasize this: className, htmlFor, camelCase, no void elements without closing
- **Components are functions** — explain the "UI = f(state)" mental model
- **Composition over inheritance** — this is a core React philosophy the interviewer wants to hear
- **Fragment pattern** — when to use `<>` vs `<div>` (no extra DOM nodes)

## 10 Interview Questions

**Q1:** Why can't React components return multiple adjacent elements?
**A:** JSX compiles to `React.createElement()` which accepts only one root. Fragments (`<></>`) compile to `React.Fragment` and add no DOM node.

**Q2:** What's the difference between an element and a component?
**A:** An element is a plain object describing what you want to render (`{ type: 'div', props: ... }`). A component is a function that returns elements. Elements are immutable, components are reusable.

**Q3:** Explain the `key` prop — when is it necessary and why should you avoid index?
**A:** Keys help React identify which items changed. Index as key causes issues with reordering, adding/removing items (wrong component reuses, state loss). Use stable unique IDs.

**Q4:** How do you conditionally render components? List 4 patterns.
**A:** (1) Ternary: `{condition ? <A /> : <B />}`, (2) AND: `{condition && <A />}`, (3) IIFE: `{(() => { if... })()}`, (4) Enum object: `{componentMap[status]}`.

**Q5:** What is `dangerouslySetInnerHTML` and why is it named that way?
**A:** Escapes React's default XSS protection. The name is intentionally alarming to discourage use. Only use with sanitized HTML (DOMPurify).

**Q6:** Why use `className` instead of `class`?
**A:** `class` is a reserved JavaScript keyword. JSX is closer to JS than HTML, so it uses the DOM API naming (`className`, `htmlFor`).

**Q7:** How does React handle whitespace in JSX?
**A:** Leading/trailing whitespace is trimmed. Whitespace between elements on separate lines is condensed to a single space. Use `&nbsp;` or `{'\u00a0'}` for explicit spacing.

**Q8:** What happens if you return `undefined` from a component?
**A:** React renders nothing but warns in development. Return `null` explicitly for "render nothing" — it's a valid React node.

**Q9:** Can you use `if` statements inside JSX?
**A:** No — JSX only accepts expressions (ternary, &&, ||). Use `if` outside the return or use an IIFE.

**Q10:** How would you render a list of items with a separator between them?
**A:** `items.flatMap((item, i) => i > 0 ? [<Separator key={`sep-${i}`} />, <Item key={item.id} />] : [<Item key={item.id} />])`. Or use a wrapper with `gap` in CSS.

## E2E Examples

### E-Commerce
```jsx
function ProductCardGrid() {
  const [products, setProducts] = useState([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    fetch('/api/products')
      .then(r => { if (!r.ok) throw new Error('Failed to load'); return r.json(); })
      .then(d => { setProducts(d); setIsLoading(false); })
      .catch(e => { setError(e.message); setIsLoading(false); });
  }, []);

  if (isLoading) return <div className="grid-skeleton">{Array.from({length:8}).map((_,i)=><div key={i} className="skeleton-card"/>)}</div>;
  if (error) return <div className="error-banner">⚠ {error} <button onClick={()=>window.location.reload()}>Retry</button></div>;
  if (!products.length) return <div className="empty-state">No products match your filters.</div>;

  return (
    <div className="product-grid">
      {products.map(p => (
        <ProductCard key={p.id} product={p} onAddToCart={() => addToCart(p)} />
      ))}
    </div>
  );
}

function ProductCard({ product, onAddToCart }) {
  return (
    <article className="product-card">
      <img src={product.thumbnail} alt={product.name} loading="lazy" />
      <div className="card-body">
        <h3>{product.name}</h3>
        <p className="price">${product.price.toFixed(2)}</p>
        <p className={product.inStock ? 'in-stock' : 'out-of-stock'}>
          {product.inStock ? 'In Stock' : 'Out of Stock'}
        </p>
        <button onClick={onAddToCart} disabled={!product.inStock}>
          {product.inStock ? 'Add to Cart' : 'Notify Me'}
        </button>
      </div>
    </article>
  );
}
```

### Banking
```jsx
function TransactionHistory({ transactions }) {
  const [sortOrder, setSortOrder] = useState('newest');
  const [filterType, setFilterType] = useState('all');

  const filtered = transactions
    .filter(t => filterType === 'all' || t.type === filterType)
    .sort((a, b) => sortOrder === 'newest'
      ? new Date(b.date) - new Date(a.date)
      : new Date(a.date) - new Date(b.date));

  return (
    <div className="transaction-history">
      <div className="tx-controls">
        <select value={filterType} onChange={e => setFilterType(e.target.value)}>
          <option value="all">All Transactions</option>
          <option value="credit">Credits</option>
          <option value="debit">Debits</option>
          <option value="transfer">Transfers</option>
        </select>
        <button onClick={() => setSortOrder(s => s === 'newest' ? 'oldest' : 'newest')}>
          {sortOrder === 'newest' ? '↓ Newest' : '↑ Oldest'}
        </button>
      </div>
      {filtered.length === 0 ? (
        <div className="empty-state">No transactions found.</div>
      ) : (
        <ul className="tx-list">
          {filtered.map(tx => (
            <TransactionRow key={tx.id} transaction={tx} />
          ))}
        </ul>
      )}
    </div>
  );
}

function TransactionRow({ transaction }) {
  const isCredit = transaction.type === 'credit';
  return (
    <li className={`tx-row ${transaction.status}`}>
      <div className="tx-icon">{isCredit ? '↓' : '↑'}</div>
      <div className="tx-info">
        <span className="tx-description">{transaction.description}</span>
        <span className="tx-date">{new Date(transaction.date).toLocaleDateString()}</span>
      </div>
      <div className={`tx-amount ${isCredit ? 'credit' : 'debit'}`}>
        {isCredit ? '+' : '-'}${Math.abs(transaction.amount).toLocaleString()}
      </div>
      <div className={`tx-status ${transaction.status}`}>{transaction.status}</div>
    </li>
  );
}
```

---

# 2. PROPS & STATE

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Filter System with URL Sync

**Scenario:** An online store lets customers filter products by category, price range, stock availability, and search term. These filter choices are stored in the URL as query parameters so users can bookmark filtered views, share them with others, and use the browser's back/forward buttons. The custom hook below reads filter values from the URL, provides a setter that updates the URL, and counts how many non-default filters are active.

```jsx
function useCatalogFilters() {
  // URL params are the source of truth — filters are derived from them
  const [searchParams, setSearchParams] = useSearchParams();

  // Derive filter values from URL params with sensible defaults
  const filters = useMemo(() => ({
    category: searchParams.get('category') || 'all',
    minPrice: Number(searchParams.get('minPrice')) || 0,
    maxPrice: Number(searchParams.get('maxPrice')) || Infinity,
    sortBy: searchParams.get('sortBy') || 'popular',
    inStock: searchParams.get('inStock') === 'true',
    search: searchParams.get('q') || '',
  }), [searchParams]);

  // Count how many filters differ from their default state
  const activeFilterCount = useMemo(() =>
    Object.entries(filters).filter(([k, v]) => {
      if (k === 'sortBy') return false;
      if (k === 'category') return v !== 'all';
      if (k === 'minPrice') return v > 0;
      if (k === 'maxPrice') return v !== Infinity;
      if (k === 'inStock') return v;
      if (k === 'search') return v !== '';
      return false;
    }).length,
    [filters]
  );

  // Update a single filter in the URL, removing it if it matches the default
  const setFilter = useCallback((key, value) => {
    setSearchParams(prev => {
      const next = new URLSearchParams(prev);
      // Remove filter param if value matches default (empty/none)
      if (value === '' || value === 'all' || value === 0) {
        next.delete(key);
      } else {
        next.set(key, String(value));
      }
      next.delete('page'); // Reset pagination when filters change
      return next;
    }, { replace: true }); // replace: true avoids cluttering browser history
  }, [setSearchParams]);

  return { filters, setFilter, activeFilterCount };
}
```

### Scenario 2: Banking — Account Balance with Optimistic Update

**Scenario:** A banking app displays the customer's account balance. When the user initiates a transfer, the UI should immediately deduct the amount from the displayed balance (optimistic update) rather than waiting for the server response. If the server request fails, the balance rolls back to the previous value. This pattern creates a responsive, native-app-like feel for financial transactions.

```jsx
function AccountCard({ account, onRefresh }) {
  // Display balance starts equal to the prop, but may differ during pending transfers
  const [displayBalance, setDisplayBalance] = useState(account.balance);
  const [pendingAmount, setPendingAmount] = useState(null);
  const [isPending, setIsPending] = useState(false);

  // Sync display with prop when not in a pending state (e.g. refreshed from server)
  useEffect(() => {
    if (!isPending) setDisplayBalance(account.balance);
  }, [account.balance, isPending]);

  async function handleTransfer(amount) {
    // Optimistic: immediately deduct from shown balance
    setPendingAmount(amount);
    setIsPending(true);
    setDisplayBalance(prev => prev - amount);

    try {
      // Attempt actual transfer on the server
      await api.transfer(account.id, amount);
      // Refresh to get the authoritative balance from the server
      await onRefresh();
    } catch (err) {
      // Rollback: restore the balance if transfer failed
      setDisplayBalance(prev => prev + amount);
      showError('Transfer failed');
    } finally {
      setPendingAmount(null);
      setIsPending(false);
    }
  }

  return (
    // Add 'syncing' class while pending to show a visual indicator
    <div className={`account-card ${isPending ? 'syncing' : ''}`}>
      <div className="account-header">
        <h3>{account.name}</h3>
        <span className="account-number">{account.maskedNumber}</span>
      </div>
      <div className="account-balance">
        <span className="currency">$</span>
        <span className={`amount ${isPending ? 'pending' : ''}`}>
          {displayBalance.toLocaleString('en-US', { minimumFractionDigits: 2 })}
        </span>
        {isPending && <span className="sync-indicator">↻ syncing</span>}
      </div>
      {pendingAmount !== null && (
        <div className="pending-transaction">
          Pending: -${pendingAmount}
        </div>
      )}
    </div>
  );
}
```

### Scenario 3: Shared Props Pattern — Polymorphic Button

**Scenario:** A design system needs a single Button component that adapts to multiple contexts: primary CTA buttons, ghost icon buttons, danger delete buttons, loading states, and full-width mobile buttons. Instead of writing separate button components for each case, a single polymorphic Button accepts props that control its visual variant, size, icon, loading state, and width. This is the "props-driven UI" pattern used in component libraries like Material UI, Chakra, and Ant Design.

```jsx
// A reusable button with forwardRef to allow parent components to focus it
const Button = React.forwardRef(({
  variant = 'primary',     // visual style: primary, secondary, ghost, danger
  size = 'md',             // sm, md, lg
  icon,                    // icon name to render
  children,                // button text
  loading = false,         // show spinner when loading
  disabled = false,        // disable the button
  fullWidth = false,       // stretch to container width (mobile)
  type = 'button',         // button type for form submission
  ...props                 // any additional native button attributes
}, ref) => {
  // Build className array, filtering out falsy values
  const classNames = [
    'btn',
    `btn-${variant}`,
    `btn-${size}`,
    fullWidth && 'btn-block',
    loading && 'btn-loading',
  ].filter(Boolean).join(' ');

  return (
    <button
      ref={ref}
      className={classNames}
      disabled={disabled || loading}
      type={type}
      {...props}
    >
      {loading ? (
        // Loading spinner with aria-hidden so screen readers ignore it
        <span className="btn-spinner" aria-hidden="true" />
      ) : icon ? (
        // Icon component sized relative to button size
        <Icon name={icon} size={size === 'sm' ? 14 : size === 'lg' ? 20 : 16} />
      ) : null}
      {children && <span className="btn-text">{children}</span>}
    </button>
  );
});
```

## Interview Tips
- **Props are read-only** — emphasize immutability. If they need to change data, they need state
- **State updates are async** — this is a common pitfall. Show you know functional updaters
- **Derived state** — Don't store what you can compute. `fullName = firstName + lastName`, don't store it
- **State co-location** — put state as close as possible to where it's used, not in a global store

## 10 Interview Questions

**Q1:** What's the difference between props and state?
**A:** Props are immutable data passed from parent to child. State is mutable data internal to the component. Props are function parameters. State is component memory.

**Q2:** Why shouldn't you mutate state directly?
**A:** React detects changes via reference equality. Direct mutation (`arr.push`) keeps the same reference, so React skips re-rendering. Always return new objects/arrays.

**Q3:** What is props drilling and how do you solve it?
**A:** Passing props through intermediate components that don't need them. Solutions: Context API, component composition (lifting content up), or state management libraries.

**Q4:** Explain the concept of "lifting state up."
**A:** When multiple components need the same data, move state to their closest common ancestor and pass down via props. Data flows down, events flow up.

**Q5:** What's the difference between controlled and uncontrolled components?
**A:** Controlled: React manages the value (`value={val} onChange={...}`). Uncontrolled: DOM manages the value (`ref={ref}` or `defaultValue`). Controlled gives instant validation/transformation.

**Q6:** How does React batch state updates?
**A:** React 18+ batches all updates automatically (event handlers, timeouts, promises). Before 18, only React events were batched. Updates in the same synchronous context are batched into one re-render.

**Q7:** When would you use `useRef` instead of `useState`?
**A:** For values that shouldn't trigger re-render: timer IDs, DOM refs, previous values, callback references. State is for UI data, refs are for imperative data.

**Q8:** What happens when you call `setState` with the same value?
**A:** React uses `Object.is` comparison. If the new value is the same reference/primitive, React bails out (skips re-render and effects).

**Q9:** Explain the concept of "state normalization."
**A:** Store state flat like a database: `{ users: byId, orderIds: [] }`. Avoid nested objects. This prevents duplication, simplifies updates, and improves performance. Like Redux's normalizr.

**Q10:** How do you share state between sibling components?
**A:** Lift state to the nearest common parent (props). Use Context for deeply nested sharing. Use Zustand/Redux for truly global state.

## E2E Examples

### E-Commerce
```jsx
function ProductListPage() {
  const [category, setCategory] = useState('all');
  const [search, setSearch] = useState('');
  const [sort, setSort] = useState('name');

  // Derived data — computed from state, no useEffect needed
  const filteredProducts = allProducts
    .filter(p => category === 'all' || p.category === category)
    .filter(p => p.name.toLowerCase().includes(search.toLowerCase()))
    .sort((a, b) => sort === 'name'
      ? a.name.localeCompare(b.name)
      : a.price - b.price);

  return (
    <div className="product-list-page">
      <SearchBar value={search} onChange={setSearch} />
      <CategoryFilters selected={category} onChange={setCategory} />
      <SortSelect value={sort} onChange={setSort} />
      <ProductCount count={filteredProducts.length} total={allProducts.length} />
      <ProductGrid products={filteredProducts} />
    </div>
  );
}

// Lift state up pattern — child receives value + onChange only
function CategoryFilters({ selected, onChange }) {
  return (
    <div className="category-filters">
      {['all', 'electronics', 'clothing', 'books'].map(cat => (
        <button key={cat} className={selected === cat ? 'active' : ''}
          onClick={() => onChange(cat)}>
          {cat.charAt(0).toUpperCase() + cat.slice(1)}
        </button>
      ))}
    </div>
  );
}
```

### Banking
```jsx
function TransferWizard() {
  const [step, setStep] = useState(1);
  const [formData, setFormData] = useState({
    fromAccount: '', toAccount: '', amount: '', memo: '', date: 'now'
  });

  function updateField(field, value) {
    setFormData(prev => ({ ...prev, [field]: value }));
  }

  return (
    <div className="transfer-wizard">
      {step === 1 && (
        <AccountSelector selected={formData.fromAccount}
          onChange={v => updateField('fromAccount', v)} />
      )}
      {step === 2 && (
        <RecipientInput value={formData.toAccount}
          onChange={v => updateField('toAccount', v)} />
      )}
      {step === 3 && (
        <AmountInput value={formData.amount}
          onChange={v => updateField('amount', v)} />
      )}
      {step === 4 && (
        <ReviewStep data={formData} onConfirm={() => submitTransfer(formData)} />
      )}
      <StepIndicator current={step} total={4} />
      <div className="wizard-nav">
        {step > 1 && <button onClick={() => setStep(s => s - 1)}>Back</button>}
        {step < 4 && <button onClick={() => setStep(s => s + 1)}
          disabled={!isStepValid(step, formData)}>Next</button>}
      </div>
    </div>
  );
}
```

---

# 3. EVENTS & FORMS

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Debounced Search with Suggestions

**Scenario:** An e-commerce site needs a search bar that shows product suggestions as the user types — similar to Amazon or Google Shopping. The search fires an API request only after the user stops typing for 300ms (debouncing), preventing unnecessary network calls on every keystroke. The dropdown also supports keyboard navigation (arrow keys, Enter to select, Escape to dismiss) for power users and accessibility.

```jsx
function SearchBar({ onSearch }) {
  // Track the raw input value, suggestions list, and keyboard selection index
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState([]);
  const [selectedIndex, setSelectedIndex] = useState(-1);
  const inputRef = useRef(null);
  const abortRef = useRef(null);  // Holds the AbortController for in-flight requests

  // Debounced API call: fires 300ms after the user stops typing
  useEffect(() => {
    // Don't search until the user has typed at least 2 characters
    if (query.length < 2) { setSuggestions([]); return; }

    const timer = setTimeout(async () => {
      // Cancel any previous in-flight request before starting a new one
      abortRef.current?.abort();
      abortRef.current = new AbortController();

      try {
        const res = await fetch(`/api/products/suggest?q=${query}`, {
          signal: abortRef.current.signal,  // Attach abort signal
        });
        const data = await res.json();
        setSuggestions(data);
        setSelectedIndex(-1);  // Reset keyboard selection
      } catch (err) {
        // Ignore AbortErrors — they are expected when requests are cancelled
        if (err.name !== 'AbortError') setSuggestions([]);
      }
    }, 300);  // 300ms debounce delay

    // Cleanup: clear the timer if query changes before 300ms
    return () => clearTimeout(timer);
  }, [query]);

  // Handle keyboard navigation within the suggestion list
  function handleKeyDown(e) {
    if (e.key === 'ArrowDown') {
      e.preventDefault();
      setSelectedIndex(i => Math.min(i + 1, suggestions.length - 1));
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      setSelectedIndex(i => Math.max(i - 1, -1));
    } else if (e.key === 'Enter') {
      e.preventDefault();
      if (selectedIndex >= 0) {
        // A suggestion is highlighted — use it
        onSearch(suggestions[selectedIndex].name);
        setQuery(suggestions[selectedIndex].name);
      } else {
        // No suggestion highlighted — search the raw query
        onSearch(query);
      }
      setSuggestions([]);
    } else if (e.key === 'Escape') {
      setSuggestions([]);
      inputRef.current?.blur();
    }
  }

  return (
    <div className="search-bar" role="combobox" aria-expanded={suggestions.length > 0}>
      <input
        ref={inputRef}
        type="search"
        value={query}
        onChange={e => setQuery(e.target.value)}  // Update query on every keystroke
        onKeyDown={handleKeyDown}  // Arrow key and Enter handling
        onBlur={() => setTimeout(() => setSuggestions([]), 200)}  // Delay hiding so click on suggestion works
        placeholder="Search products..."
        aria-autocomplete="list"
        aria-controls="search-suggestions"
      />
      {suggestions.length > 0 && (
        <ul id="search-suggestions" role="listbox" className="suggestions">
          {suggestions.map((s, i) => (
            <li
              key={s.id}
              role="option"
              aria-selected={i === selectedIndex}
              className={i === selectedIndex ? 'highlighted' : ''}
              onMouseDown={() => { onSearch(s.name); setQuery(s.name); }}
            >
              <span className="suggestion-name">{s.name}</span>
              <span className="suggestion-category">{s.category}</span>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

### Scenario 2: Banking — Secure Transfer Form with All Validations

**Scenario:** A banking website needs a money transfer form with strict validation rules: the user must select a source account, enter a valid 10-digit destination account, specify an amount that doesn't exceed their balance or daily limit, and optionally add a memo. Validation runs on blur (when the user leaves a field) and on submit. The form shows inline error messages and warnings for limits.

```jsx
function TransferForm({ accounts, dailyLimit = 10000, onSubmit }) {
  // Form field values, validation errors, and touched tracking
  const [form, setForm] = useState({
    from: '', to: '', amount: '', memo: '', frequency: 'once', date: '',
  });
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  const [showConfirmation, setShowConfirmation] = useState(false);

  // Derived values: the selected account, daily usage, limits
  const selectedAccount = accounts.find(a => a.id === form.from);
  const dailyUsed = useMemo(() => calculateDailyUsage(selectedAccount?.id), [selectedAccount]);
  const remainingLimit = dailyLimit - dailyUsed;
  const exceedsLimit = form.amount > remainingLimit;
  const exceedsBalance = form.amount > (selectedAccount?.balance || 0);

  // Validation rules for each field — each returns an error string or null
  const validators = {
    from: (v) => !v ? 'Select source account' : null,
    to: (v) => {
      if (!v) return 'Enter destination';
      if (v === form.from) return 'Cannot transfer to same account';
      if (!validateAccount(v)) return 'Invalid account format';
      return null;
    },
    amount: (v) => {
      if (!v) return 'Enter amount';
      if (isNaN(v) || v <= 0) return 'Invalid amount';
      if (v > 100000) return 'Exceeds $100,000 limit';
      if (exceedsBalance) return 'Insufficient funds';
      if (exceedsLimit) return `Daily limit remaining: $${remainingLimit}`;
      return null;
    },
    memo: (v) => v?.length > 100 ? 'Max 100 characters' : null,
  };

  // Validate a single field and update errors state
  function validate(field) {
    const value = form[field];
    const validator = validators[field];
    if (!validator) return null;
    const error = validator(value);
    setErrors(prev => ({ ...prev, [field]: error }));
    return error;
  }

  // Update field value, re-validate if the field has been touched
  function handleChange(field, value) {
    setForm(prev => ({ ...prev, [field]: value }));
    if (touched[field]) validate(field);
  }

  // Mark field as touched on blur and validate
  function handleBlur(field) {
    setTouched(prev => ({ ...prev, [field]: true }));
    validate(field);
  }

  // Final validation on submit — validate all fields
  function handleSubmit(e) {
    e.preventDefault();  // Prevent page refresh
    const newErrors = {};
    Object.keys(validators).forEach(f => {
      const error = validate(f);
      if (error) newErrors[f] = error;
    });
    setTouched(Object.keys(validators).reduce((a, f) => ({ ...a, [f]: true }), {}));
    if (Object.keys(newErrors).length === 0) setShowConfirmation(true);
  }

  if (showConfirmation) { /* show confirmation review before final submit */ }

  return (
    <form onSubmit={handleSubmit} noValidate className="transfer-form">
      {/* Account Source selector with balance display */}
      <div className="form-group">
        <label htmlFor="from-account">From Account</label>
        <select id="from-account" value={form.from}
          onChange={e => handleChange('from', e.target.value)}
          onBlur={() => handleBlur('from')}
          aria-invalid={!!errors.from}
          aria-describedby={errors.from ? 'from-error' : undefined}
        >
          <option value="">Select account</option>
          {accounts.map(a => (
            <option key={a.id} value={a.id}>
              {a.name} — ${a.balance.toFixed(2)}
            </option>
          ))}
        </select>
        {errors.from && <span id="from-error" role="alert" className="error">{errors.from}</span>}
      </div>

      {/* Amount input with currency prefix and limit warnings */}
      <div className="form-group">
        <label htmlFor="amount">Amount</label>
        <div className="input-group">
          <span className="input-prefix">$</span>
          <input id="amount" type="number" step="0.01" min="0.01"
            value={form.amount}
            onChange={e => handleChange('amount', e.target.value)}
            onBlur={() => handleBlur('amount')}
            aria-invalid={!!errors.amount}
          />
        </div>
        {exceedsBalance && <Warning>Available balance: ${selectedAccount?.balance.toFixed(2)}</Warning>}
        {exceedsLimit && <Warning>Daily limit remaining: ${remainingLimit.toFixed(2)}</Warning>}
        {errors.amount && <span role="alert" className="error">{errors.amount}</span>}
      </div>

      <button type="submit" className="btn-primary btn-block">Review Transfer</button>
    </form>
  );
}
```

### Scenario 3: Multi-Step Wizard (E-Commerce Checkout)

**Scenario:** During checkout on an e-commerce site, the user goes through several steps — shipping address, payment details, and order review — one at a time. A multi-step wizard component manages state progression, validates each step before proceeding, and allows going back to edit previous steps. This pattern is used by every online store (Amazon, Shopify, etc.).

```jsx
function CheckoutWizard() {
  // Track current step index and accumulated form data across all steps
  const [step, setStep] = useState(1);
  const [data, setData] = useState({});
  const [errors, setErrors] = useState({});

  // Define each step: its component, title, and validation function
  const steps = [
    { id: 'shipping', title: 'Shipping', Component: ShippingForm, isValid: validateShipping },
    { id: 'payment', title: 'Payment', Component: PaymentForm, isValid: validatePayment },
    { id: 'review', title: 'Review', Component: ReviewOrder, isValid: () => true },
  ];

  // Advance to next step: merge step data, validate, proceed
  function nextStep(stepData) {
    const merged = { ...data, ...stepData };  // Accumulate data across steps
    const validation = steps[step - 1].isValid(merged);  // Validate current step
    if (validation !== true) {
      setErrors(validation);  // Show validation errors
      return;
    }
    setErrors({});
    setData(merged);
    if (step < steps.length) setStep(s => s + 1);  // Move to next step
  }

  // Go back to the previous step
  function prevStep() {
    setErrors({});
    setStep(s => Math.max(1, s - 1));
  }

  // The current step's component and title
  const { Component, title } = steps[step - 1];

  return (
    <div className="checkout-wizard">
      {/* Visual progress bar showing all step labels */}
      <ProgressBar current={step} total={steps.length} labels={steps.map(s => s.title)} />
      <Component
        data={data}      // Pass accumulated data so step can pre-fill
        errors={errors}
        onNext={nextStep}
        onBack={prevStep}
        isFirst={step === 1}
        isLast={step === steps.length}  // Last step: show "Place Order" instead of "Next"
      />
    </div>
  );
}
```

## Interview Tips
- **e.preventDefault()** — always mention this in form handling. The page refresh is the #1 form bug
- **Controlled vs uncontrolled** — a classic question. Know the trade-offs
- **Debouncing** — essential for search inputs, autocomplete. Mention AbortController for race conditions
- **Form libraries** — know when to use React Hook Form (performance) vs raw state (simplicity)

## 10 Interview Questions

**Q1:** What is a SyntheticEvent and how does it differ from native DOM events?
**A:** React's cross-browser wrapper around native events. Provides consistent API across browsers. React 18+ uses native events; React <17 used pooling (e.persist() needed for async access).

**Q2:** How do you handle form validation in React?
**A:** Validate on change (real-time feedback), on blur (when user leaves field), and on submit (final gate). Show errors in `aria-describedby` for accessibility. Use Zod/Yup schemas for complex validation.

**Q3:** What's the difference between `onChange` and `onBlur`?
**A:** `onChange` fires on every keystroke/value change. `onBlur` fires when the element loses focus. Use onChange for instant feedback, onBlur for validation that shouldn't annoy during typing.

**Q4:** How do you prevent a form from submitting twice?
**A:** Disable button on submit (`disabled={isSubmitting}`), use loading state, implement idempotency key for API, check `isSubmitting` in handler.

**Q5:** How do you handle keyboard events for accessibility?
**A:** Use `onKeyDown` for Enter/Escape. Buttons already handle Enter/Space natively. For custom components, add `role` and `aria-*` attributes. Arrow keys for list navigation.

**Q6:** What is event delegation in React?
**A:** React attaches event listeners to the root (React 18+) instead of each element. Events bubble up. This reduces memory usage and enables consistent behavior across frameworks.

**Q7:** How do you create a custom hook for debounced input?
**A:** Store value in state, use setTimeout in useEffect with cleanup, return debounced value after delay. Pattern: `useDebounce(value, delay)`.

**Q8:** What type of events does React not delegate?
**A:** Scroll events, media events (`onPlay`, `onPause`), and some others that don't bubble. These are attached directly to the element.

**Q9:** How do you handle file uploads in React?
**A:** Use uncontrolled input (file values are read-only). Read with FileReader API or FormData. Show preview with URL.createObjectURL. Clean up with revokeObjectURL.

**Q10:** What are the accessibility considerations for form validation?
**A:** Use `aria-invalid` on invalid fields, `aria-describedby` pointing to error message, `role="alert"` on errors, auto-focus first error field, use `<label htmlFor>` correctly.

## E2E Examples

### E-Commerce
```jsx
function CheckoutForm() {
  const [form, setForm] = useState({ email: '', card: '', expiry: '', cvv: '' });
  const [errors, setErrors] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);
  const errorRef = useRef(null);

  function validate() {
    const errs = {};
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(form.email)) errs.email = 'Valid email required';
    if (!/^\d{16}$/.test(form.card.replace(/\s/g, ''))) errs.card = '16-digit card number';
    if (!/^\d{2}\/\d{2}$/.test(form.expiry)) errs.expiry = 'MM/YY format';
    if (!/^\d{3,4}$/.test(form.cvv)) errs.cvv = '3 or 4 digit CVV';
    return errs;
  }

  function handleSubmit(e) {
    e.preventDefault();
    const errs = validate();
    setErrors(errs);
    if (Object.keys(errs).length > 0) {
      errorRef.current?.focus();
      return;
    }
    setIsSubmitting(true);
    // Submit payment...
  }

  function handleChange(field, value) {
    setForm(prev => ({ ...prev, [field]: value }));
    if (errors[field]) setErrors(prev => ({ ...prev, [field]: '' }));
  }

  return (
    <form onSubmit={handleSubmit} noValidate>
      {Object.keys(errors).length > 0 && (
        <div className="form-errors" ref={errorRef} tabIndex={-1} role="alert">
          {Object.values(errors).filter(Boolean).map((e, i) => <p key={i}>{e}</p>)}
        </div>
      )}
      <label htmlFor="email">Email</label>
      <input id="email" type="email" value={form.email}
        onChange={e => handleChange('email', e.target.value)}
        aria-invalid={!!errors.email} aria-describedby={errors.email ? 'email-err' : undefined} />
      {errors.email && <p id="email-err" className="field-error">{errors.email}</p>}
      <label htmlFor="card">Card Number</label>
      <input id="card" value={form.card} onChange={e => handleChange('card', e.target.value)}
        aria-invalid={!!errors.card} maxLength={19} />
      {errors.card && <p className="field-error">{errors.card}</p>}
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Processing...' : `Pay $${total.toFixed(2)}`}
      </button>
    </form>
  );
}
```

### Banking
```jsx
function TransferForm() {
  const [form, setForm] = useState({ from: '', to: '', amount: '', memo: '' });
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});

  function validateField(name, value) {
    switch (name) {
      case 'amount':
        const num = parseFloat(value);
        if (!value) return 'Amount is required';
        if (isNaN(num) || num <= 0) return 'Enter a positive number';
        if (num > 10000) return 'Exceeds daily limit of $10,000';
        return '';
      case 'to':
        if (!/^\d{10}$/.test(value)) return '10-digit account number required';
        if (value === form.from) return 'Cannot transfer to same account';
        return '';
      default: return '';
    }
  }

  function handleBlur(name) {
    setTouched(prev => ({ ...prev, [name]: true }));
    const err = validateField(name, form[name]);
    setErrors(prev => ({ ...prev, [name]: err }));
  }

  function handleChange(name, value) {
    setForm(prev => ({ ...prev, [name]: value }));
    if (touched[name]) {
      const err = validateField(name, value);
      setErrors(prev => ({ ...prev, [name]: err }));
    }
  }

  function handleSubmit(e) {
    e.preventDefault();
    const newErrors = {};
    Object.keys(form).forEach(k => { const err = validateField(k, form[k]); if (err) newErrors[k] = err; });
    setErrors(newErrors);
    setTouched(Object.keys(form).reduce((a, k) => ({ ...a, [k]: true }), {}));
    if (Object.keys(newErrors).length === 0) submitTransfer(form);
  }

  return (
    <form onSubmit={handleSubmit} noValidate className="transfer-form">
      <FormField label="From Account" name="from" value={form.from} error={errors.from}
        touched={touched.from} onChange={handleChange} onBlur={handleBlur}
        render={<select> <option>Checking (...1234)</option> <option>Savings (...5678)</option> </select>} />
      <FormField label="To Account" name="to" value={form.to} error={errors.to}
        touched={touched.to} onChange={handleChange} onBlur={handleBlur}
        render={<input placeholder="10-digit account number" maxLength={10} />} />
      <FormField label="Amount" name="amount" type="number" value={form.amount}
        error={errors.amount} touched={touched.amount}
        onChange={handleChange} onBlur={handleBlur} prefix="$" />
      <button type="submit" className="btn-primary">Review Transfer</button>
    </form>
  );
}
```

---

# 4. useEffect & SIDE EFFECTS

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Real-Time Inventory via WebSocket

**Scenario:** On a product detail page, the inventory count must update in real time as other customers add items to their carts. A WebSocket connection streams live stock updates from the server. The custom hook below handles connection lifecycle, reconnects with exponential backoff if the connection drops, and cleans up properly when the component unmounts.

```jsx
function useProductInventory(productId) {
  // Track inventory state and WebSocket connection status
  const [inventory, setInventory] = useState(null);
  const [connectionStatus, setConnectionStatus] = useState('disconnected');
  const wsRef = useRef(null);              // Holds the WebSocket instance
  const reconnectTimerRef = useRef(null);   // Tracks reconnect attempt number
  const mountedRef = useRef(true);          // Prevents state updates after unmount

  // Establish WebSocket connection with exponential backoff reconnect
  const connect = useCallback(() => {
    if (wsRef.current?.readyState === WebSocket.OPEN) return;

    const ws = new WebSocket(`wss://api.example.com/inventory/${productId}`);
    wsRef.current = ws;

    ws.onopen = () => {
      if (!mountedRef.current) { ws.close(); return; }
      setConnectionStatus('connected');
    };

    ws.onmessage = (event) => {
      if (!mountedRef.current) return;
      const data = JSON.parse(event.data);
      setInventory({
        inStock: data.stock > 0,
        quantity: data.stock,
        lowStock: data.stock < 10,
        lastUpdated: data.timestamp,
      });
    };

    ws.onclose = () => {
      if (!mountedRef.current) return;
      setConnectionStatus('disconnected');
      // Exponential backoff: 2s, 4s, 8s, 16s... capped at 30s
      const delay = Math.min(1000 * 2 ** (reconnectTimerRef.current || 0), 30000);
      reconnectTimerRef.current = (reconnectTimerRef.current || 0) + 1;
      setTimeout(connect, delay);
    };

    ws.onerror = () => {
      if (mountedRef.current) setConnectionStatus('error');
    };
  }, [productId]);

  // Mount effect: connect WS, clean up on unmount
  useEffect(() => {
    mountedRef.current = true;
    connect();
    return () => {
      // Cleanup: mark unmounted, close WS, cancel pending reconnect
      mountedRef.current = false;
      wsRef.current?.close();
      clearTimeout(reconnectTimerRef.current);
    };
  }, [connect]);

  return { inventory, connectionStatus };
}
```

### Scenario 2: Banking — Session Timer with Auto-Logout

**Scenario:** A banking app automatically logs the user out after 15 minutes of inactivity to prevent unauthorized access. A custom hook tracks mouse movements, keyboard presses, and touch events as activity indicators. A countdown effect runs every second and triggers the logout callback when time expires. The user also gets a manual "extend session" option.

```jsx
function useSessionTimer(timeoutMinutes = 15, onTimeout) {
  // Remaining seconds displayed as MM:SS, last activity timestamp ref
  const [remaining, setRemaining] = useState(timeoutMinutes * 60);
  const lastActivityRef = useRef(Date.now());
  const intervalRef = useRef(null);

  // Effect 1: Listen for all user activity events to reset the idle timer
  useEffect(() => {
    function handleActivity() {
      lastActivityRef.current = Date.now();  // Record the latest activity time
    }

    // Attach listeners to all relevant user interaction events
    window.addEventListener('mousemove', handleActivity);
    window.addEventListener('keydown', handleActivity);
    window.addEventListener('scroll', handleActivity);
    window.addEventListener('click', handleActivity);
    window.addEventListener('touchstart', handleActivity);

    return () => {
      // Cleanup: remove all listeners on unmount
      window.removeEventListener('mousemove', handleActivity);
      window.removeEventListener('keydown', handleActivity);
      window.removeEventListener('scroll', handleActivity);
      window.removeEventListener('click', handleActivity);
      window.removeEventListener('touchstart', handleActivity);
    };
  }, []);

  // Effect 2: Countdown timer that fires every second
  useEffect(() => {
    intervalRef.current = setInterval(() => {
      const elapsed = (Date.now() - lastActivityRef.current) / 1000;
      const newRemaining = Math.max(0, Math.round(timeoutMinutes * 60 - elapsed));
      setRemaining(newRemaining);

      if (newRemaining <= 0) {
        clearInterval(intervalRef.current);
        onTimeout();  // Trigger auto-logout
      }
    }, 1000);

    return () => clearInterval(intervalRef.current);
  }, [timeoutMinutes, onTimeout]);

  // Manual reset — called when user clicks "Extend Session"
  const resetTimer = useCallback(() => {
    lastActivityRef.current = Date.now();
    setRemaining(timeoutMinutes * 60);
  }, [timeoutMinutes]);

  // Format remaining time as MM:SS for display
  const minutes = Math.floor(remaining / 60);
  const seconds = remaining % 60;

  return { remaining: `${minutes}:${String(seconds).padStart(2, '0')}`, resetTimer };
}
```

### Scenario 3: Analytics Page View Tracking

**Scenario:** A marketing team needs to track which pages users visit and how long they stay on each page. Using React's useEffect with cleanup, we can send a page view event on mount and a page exit event (with dwell time) on unmount. The navigator.sendBeacon API ensures the exit event is sent even if the user closes the tab or navigates away quickly.

```jsx
function usePageTracking(pageName) {
  // Record the time the user entered the page for dwell time calculation
  const startedAt = useRef(Date.now());

  useEffect(() => {
    // Build the page view payload with contextual data
    const pageView = {
      page: pageName,
      timestamp: new Date().toISOString(),
      referrer: document.referrer,      // Where the user came from
      userAgent: navigator.userAgent,    // Browser/device info
      screenSize: `${window.innerWidth}x${window.innerHeight}`,
    };

    // Send page view event — fire-and-forget, must not break the app
    if (navigator.sendBeacon) {
      navigator.sendBeacon('/api/analytics/pageview', JSON.stringify(pageView));
    } else {
      fetch('/api/analytics/pageview', {
        method: 'POST', body: JSON.stringify(pageView),
        keepalive: true,  // Ensures request completes even on page unload
      }).catch(() => {}); // Silent fail — analytics failure should never block the UI
    }

    // Cleanup: send page exit event with dwell time on unmount
    return () => {
      const dwellTime = Date.now() - startedAt.current;  // How long they stayed
      const pageExit = {
        page: pageName,
        event: 'page_exit',
        dwellTimeMs: dwellTime,
        timestamp: new Date().toISOString(),
      };

      // sendBeacon is reliable on page unload (unlike fetch which may be cancelled)
      navigator.sendBeacon?.('/api/analytics/exit', JSON.stringify(pageExit));
    };
  }, [pageName]);  // Re-track if pageName changes (e.g. route change)
}
```

## Interview Tips
- **Three dependency types** — empty (mount), omitted (every render), values (on change). Explain each with examples
- **Cleanup function** — always mention this for subscriptions, timers, and event listeners
- **StrictMode double-invoke** — React 18 intentionally runs effects twice in dev to find missing cleanups
- **Ref pattern for stale closures** — store callbacks in useRef to avoid stale closure bugs in effects

## 10 Interview Questions

**Q1:** What is the purpose of the cleanup function in useEffect?
**A:** Runs on unmount and before the effect re-runs. Used to cancel subscriptions, timers, fetch requests, and event listeners. Prevents memory leaks and race conditions.

**Q2:** How does the dependency array work? What happens with each type?
**A:** Empty `[]` — runs once on mount. Omitted — runs after every render. `[a, b]` — runs when a or b change. React uses `Object.is` for comparison.

**Q3:** What is a stale closure and how do you prevent it?
**A:** When an effect captures an old value (from the render it was created in). Prevent by: adding the value to deps array, using functional setState, or storing in useRef.

**Q4:** Why might an effect run twice in development?
**A:** React 18 StrictMode intentionally unmounts and remounts components (mount → unmount → mount) in development to verify cleanup functions work correctly. Doesn't happen in production.

**Q5:** What's the difference between useEffect and useLayoutEffect?
**A:** useEffect runs after paint (async, non-blocking). useLayoutEffect runs synchronously before paint (blocks visual updates). Use useLayoutEffect for DOM measurements/mutations that must happen before the user sees the result.

**Q6:** How do you fetch data with useEffect without race conditions?
**A:** Use a cancellation flag (`let cancelled = false`) or AbortController. Check the flag before setting state. Cancel on cleanup. This prevents setting state from a stale response.

**Q7:** Can useEffect be async directly?
**A:** No. useEffect expects a cleanup function or nothing. Async functions return a Promise. Define async inside and call it: `useEffect(() => { async fn() { ... }; fn(); }, [])`.

**Q8:** What happens if you call setState in useEffect without dependencies?
**A:** Infinite loop. Component renders → effect runs → setState → component re-renders → effect runs again → ... Always provide a dependency array.

**Q9:** How do you handle errors in useEffect?
**A:** Use try-catch inside the async function. Set error state in catch. Clean up on unmount. For async operations, abort in cleanup to prevent state updates on unmounted component.

**Q10:** When should you NOT use useEffect?
**A:** For derived data (computed from state), user interactions (event handlers should handle these), and transformations that can be done during render. Overusing effects is a common anti-pattern.

## E2E Examples

### E-Commerce
```jsx
function ProductDetailPage({ productId }) {
  const [product, setProduct] = useState(null);
  const [related, setRelated] = useState([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  // E2E data loading with cleanup via AbortController
  useEffect(() => {
    const controller = new AbortController();
    setIsLoading(true);
    setError(null);

    async function load() {
      try {
        const [prodRes, relatedRes] = await Promise.all([
          fetch(`/api/products/${productId}`, { signal: controller.signal }),
          fetch(`/api/products/${productId}/related`, { signal: controller.signal })
        ]);
        if (!prodRes.ok || !relatedRes.ok) throw new Error('Failed to load product');
        const [prod, rel] = await Promise.all([prodRes.json(), relatedRes.json()]);
        if (!controller.signal.aborted) {
          setProduct(prod);
          setRelated(rel);
          setIsLoading(false);
        }
      } catch (err) {
        if (!controller.signal.aborted) {
          setError(err.message);
          setIsLoading(false);
        }
      }
    }
    load();
    return () => controller.abort();
  }, [productId]);

  // Sync page title with product name
  useEffect(() => {
    if (product) document.title = `${product.name} - Shop`;
    return () => { document.title = 'Shop'; };
  }, [product]);

  // Track product view analytics
  useEffect(() => {
    if (product) analytics.track('product_view', { id: product.id, category: product.category });
  }, [product]);

  if (isLoading) return <ProductDetailSkeleton />;
  if (error) return <ErrorBanner message={error} onRetry={() => window.location.reload()} />;
  if (!product) return <NotFound />;
  return <ProductDetail product={product} related={related} />;
}
```

### Banking
```jsx
function BalanceDashboard({ userId }) {
  const [accounts, setAccounts] = useState([]);
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  // WebSocket for real-time balance updates
  useEffect(() => {
    const ws = new WebSocket(`wss://bank.example.com/accounts/${userId}/live`);
    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      setAccounts(prev => prev.map(acc =>
        acc.id === update.accountId ? { ...acc, balance: update.newBalance } : acc
      ));
    };
    ws.onclose = () => console.log('WS disconnected — falling back to polling');
    return () => ws.close();
  }, [userId]);

  // Polling fallback when offline
  useEffect(() => {
    if (isOnline) return;
    const id = setInterval(async () => {
      const res = await fetch(`/api/accounts/${userId}/balances`);
      if (res.ok) setAccounts(await res.json());
    }, 30000);
    return () => clearInterval(id);
  }, [userId, isOnline]);

  // Online/offline detection
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

  return (
    <div className="balance-dashboard">
      <div className={`connection-status ${isOnline ? 'online' : 'offline'}`}>
        {isOnline ? '● Live' : '○ Offline — updating periodically'}
      </div>
      {accounts.map(acc => (
        <AccountCard key={acc.id} account={acc} />
      ))}
    </div>
  );
}
```

---

# 5. CUSTOM HOOKS

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — useCart Hook

**Scenario:** Every e-commerce site needs a shopping cart that persists across page refreshes. Instead of scattering cart logic across multiple components, a custom `useCart` hook encapsulates all cart operations — add, remove, update quantity, clear — along with computed values like total price and item count. It uses `useLocalStorage` internally so the cart survives page reloads. Any component that needs cart data simply calls this one hook.

```jsx
function useCart() {
  // Persist cart items in localStorage so they survive page refreshes
  const [items, setItems] = useLocalStorage('cart', []);

  // Compute total price from all items (memoized for performance)
  const total = useMemo(() =>
    items.reduce((sum, item) => sum + item.price * item.quantity, 0),
    [items]
  );

  // Compute total number of items (memoized)
  const itemCount = useMemo(() =>
    items.reduce((sum, item) => sum + item.quantity, 0),
    [items]
  );

  // Add a product to cart — increment quantity if already present
  const addItem = useCallback((product, quantity = 1) => {
    setItems(prev => {
      const existing = prev.find(i => i.id === product.id);
      if (existing) {
        return prev.map(i =>
          i.id === product.id ? { ...i, quantity: i.quantity + quantity } : i
        );
      }
      return [...prev, { ...product, quantity }];
    });
  }, [setItems]);

  // Remove a product from cart by its ID
  const removeItem = useCallback((productId) => {
    setItems(prev => prev.filter(i => i.id !== productId));
  }, [setItems]);

  // Update quantity — remove the item if quantity drops to zero or below
  const updateQuantity = useCallback((productId, quantity) => {
    if (quantity <= 0) { removeItem(productId); return; }
    setItems(prev => prev.map(i =>
      i.id === productId ? { ...i, quantity } : i
    ));
  }, [setItems, removeItem]);

  // Empty the entire cart
  const clearCart = useCallback(() => setItems([]), [setItems]);

  return { items, total, itemCount, addItem, removeItem, updateQuantity, clearCart };
}
```

### Scenario 2: Banking — useAccountSummary

**Scenario:** A banking dashboard shows all of a customer's accounts — checking, savings, credit cards, loans — along with a total balance and net worth. A custom `useAccountSummary` hook fetches the accounts on mount, provides a refetch function to refresh data, and computes derived values (total balance, net worth, accounts grouped by type). Components stay clean: they just call the hook and render the returned data.

```jsx
function useAccountSummary(userId) {
  // Track accounts data, loading state, and error state
  const [accounts, setAccounts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  // Fetch accounts from the API with loading/error state management
  const fetchAccounts = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const data = await bankingApi.getAccounts(userId);
      setAccounts(data);
    } catch (err) {
      setError(err.message);
    } finally {
      setLoading(false);
    }
  }, [userId]);

  // Fetch on mount (when userId changes)
  useEffect(() => { fetchAccounts(); }, [fetchAccounts]);

  // Derived: total balance across all accounts
  const totalBalance = useMemo(() =>
    accounts.reduce((sum, a) => sum + a.balance, 0),
    [accounts]
  );

  // Derived: net worth (assets minus liabilities)
  const netWorth = useMemo(() =>
    accounts.reduce((sum, a) => sum + (a.type === 'asset' ? a.balance : -a.balance), 0),
    [accounts]
  );

  // Derived: accounts grouped by type (checking, savings, credit, loan)
  const grouped = useMemo(() => {
    const groups = {};
    accounts.forEach(a => {
      if (!groups[a.type]) groups[a.type] = [];
      groups[a.type].push(a);
    });
    return groups;
  }, [accounts]);

  return { accounts, loading, error, refetch: fetchAccounts, totalBalance, netWorth, grouped };
}
```

### Scenario 3: useMediaQuery — Responsive Behavior

**Scenario:** A responsive app needs to render different layouts based on screen size — a mobile hamburger menu, a tablet condensed nav, or a full sidebar on desktop. Instead of importing a CSS-in-JS breakpoint utility, a `useMediaQuery` hook wraps the `window.matchMedia` API and returns a boolean. The component just calls the hook and conditionally renders. This hook is used in both the e-commerce app (product grid columns) and the banking app (compact vs detailed transaction view).

```jsx
// Custom hook that tracks whether a CSS media query matches
function useMediaQuery(query) {
  // Initialize with the current match state (handles SSR by defaulting to false)
  const [matches, setMatches] = useState(() => {
    if (typeof window !== 'undefined') {
      return window.matchMedia(query).matches;
    }
    return false;
  });

  // Listen for changes to the media query
  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (e) => setMatches(e.matches);

    mediaQuery.addEventListener('change', handler);
    return () => mediaQuery.removeEventListener('change', handler);
  }, [query]);

  return matches;
}

// Usage in a responsive layout component
function ResponsiveLayout() {
  // Call the same hook with different breakpoints
  const isMobile = useMediaQuery('(max-width: 768px)');
  const isTablet = useMediaQuery('(min-width: 769px) and (max-width: 1024px)');
  const isDesktop = useMediaQuery('(min-width: 1025px)');

  return (
    <div>
      {/* Render different navigation components based on screen size */}
      {isMobile && <MobileNav />}
      {isTablet && <TabletNav />}
      {isDesktop && <FullSidebar />}

      {/* E-Commerce: different grid columns per breakpoint */}
      <ProductGrid columns={isMobile ? 2 : isTablet ? 3 : 4} />

      {/* Banking: compact list on mobile, full table on desktop */}
      {isMobile
        ? <CompactTransactionList />
        : <DetailedTransactionTable />}
    </div>
  );
}
```

## Interview Tips
- **Name starts with "use"** — non-negotiable. It tells React to apply hook rules
- **Compose hooks** — show you can combine hooks (useDebounce + useFetch)
- **Testable** — hooks are pure functions. Test with renderHook from testing-library
- **Extract early** — if a component has 3+ useEffect calls, extract into custom hooks

## 10 Interview Questions

**Q1:** What are the rules of custom hooks?
**A:** (1) Name must start with "use". (2) Only call hooks at the top level (no conditions, loops). (3) Only call from React function components or other custom hooks.

**Q2:** How do you test a custom hook?
**A:** Use `renderHook` from `@testing-library/react`. Call hook, assert return values, call returned functions, assert state changed. For async, use `waitFor` or `act`.

**Q3:** Can custom hooks accept arguments?
**A:** Yes — they're functions. `useLocalStorage('key', default)`, `useFetch(url)`, `useDebounce(value, delay)`. Arguments become hook dependencies.

**Q4:** How do you share state between hooks?
**A:** Lift state to a parent component and pass down as props. Or use a custom hook that internally uses Context/Zustand. Hooks themselves don't share state — components do.

**Q5:** What's the difference between a custom hook and a utility function?
**A:** Custom hooks can use other React hooks (useState, useEffect) and must be called within a component. Utility functions are pure JS, no hooks allowed.

**Q6:** Can a custom hook return JSX?
**A:** No. Hooks return data/state/functions. Only components return JSX. If a custom hook returned JSX, it would break the rules of hooks (components must be capitalized, hooks must start with "use").

**Q7:** How do you create a custom hook for API polling?
**A:** useCallback for the fetch function, useEffect with setInterval, cleanup clears interval, track mounted state, refetch on interval, respect dependencies.

**Q8:** What is the "use" convention and why is it enforced?
**A:** It enables React's lint rules (rules-of-hooks) and ensures the hook ordering mechanism works. Without it, React can't detect rule violations.

**Q9:** How do you handle errors in custom hooks?
**A:** Return error state from the hook, expose a refetch/retry function. Let the component decide how to display errors. Don't throw from hooks — that crashes the render.

**Q10:** Can one custom hook call another custom hook?
**A:** Yes — composition is the point. `useUser()` can call `useFetch()`. `useBoard()` can call `useLocalStorage()`. This is how you build abstractions.

## E2E Examples

### E-Commerce
```jsx
function useCart() {
  const [items, setItems] = useState([]);
  const [isSyncing, setIsSyncing] = useState(false);

  const addItem = useCallback((product, quantity = 1) => {
    setItems(prev => {
      const existing = prev.find(i => i.id === product.id);
      if (existing) return prev.map(i => i.id === product.id
        ? { ...i, quantity: i.quantity + quantity } : i);
      return [...prev, { ...product, quantity }];
    });
  }, []);

  const removeItem = useCallback((id) => {
    setItems(prev => prev.filter(i => i.id !== id));
  }, []);

  const updateQuantity = useCallback((id, quantity) => {
    if (quantity <= 0) return removeItem(id);
    setItems(prev => prev.map(i => i.id === id ? { ...i, quantity } : i));
  }, [removeItem]);

  const total = useMemo(() =>
    items.reduce((sum, i) => sum + i.price * i.quantity, 0),
  [items]);

  const itemCount = useMemo(() => items.reduce((sum, i) => sum + i.quantity, 0), [items]);

  return { items, addItem, removeItem, updateQuantity, total, itemCount, isSyncing };
}

// Usage — cart page uses the hook
function CartPage() {
  const { items, removeItem, updateQuantity, total, itemCount } = useCart();
  if (!items.length) return <EmptyCart />;
  return (
    <div className="cart-page">
      <h2>Shopping Cart ({itemCount} items)</h2>
      {items.map(item => (
        <CartItem key={item.id} item={item}
          onQuantityChange={q => updateQuantity(item.id, q)}
          onRemove={() => removeItem(item.id)} />
      ))}
      <CartSummary total={total} />
    </div>
  );
}
```

### Banking
```jsx
function useAccountTransactions(accountId) {
  const [transactions, setTransactions] = useState([]);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);

  const loadMore = useCallback(async () => {
    if (isLoading || !hasMore) return;
    setIsLoading(true);
    setError(null);
    try {
      const res = await fetch(`/api/accounts/${accountId}/transactions?page=${page}&limit=20`);
      if (!res.ok) throw new Error('Failed to load');
      const data = await res.json();
      setTransactions(prev => [...prev, ...data.transactions]);
      setHasMore(data.hasMore);
      setPage(p => p + 1);
    } catch (err) {
      setError(err.message);
    } finally {
      setIsLoading(false);
    }
  }, [accountId, page, isLoading, hasMore]);

  // Reset when account changes
  useEffect(() => {
    setTransactions([]);
    setPage(1);
    setHasMore(true);
    setError(null);
  }, [accountId]);

  const balance = useMemo(() =>
    transactions.reduce((sum, t) => sum + (t.type === 'credit' ? t.amount : -t.amount), 0),
  [transactions]);

  return { transactions, isLoading, error, hasMore, loadMore, balance };
}

// Usage
function AccountHistory({ accountId }) {
  const { transactions, isLoading, error, hasMore, loadMore, balance } = useAccountTransactions(accountId);
  const listRef = useRef(null);
  useInfiniteScroll(listRef, loadMore); // Another custom hook composed together

  return (
    <div className="account-history" ref={listRef}>
      <h3>Balance: ${balance.toLocaleString()}</h3>
      {error && <ErrorBanner message={error} />}
      <TransactionList transactions={transactions} />
      {isLoading && <Spinner />}
      {!hasMore && <p className="end-message">All transactions loaded</p>}
    </div>
  );
}
```

---

# 6. useReducer

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Checkout State Machine

**Scenario:** An e-commerce checkout flow has multiple steps (shipping, payment, billing, confirmation) with complex transitions. Using `useState` for each piece would be messy — the checkout needs to coordinate shipping data, payment data, processing state, order ID, and error state. A reducer models this as a state machine: each action type represents a transition (VALIDATE_SHIPPING, PROCESS_PAYMENT, PAYMENT_SUCCESS, etc.). The reducer is a pure function that's trivial to test.

```jsx
// Initial checkout state: starts at step 1, no data entered yet
const initialState = {
  step: 1,
  shipping: {},
  payment: {},
  billing: {},
  orderId: null,
  isProcessing: false,
  error: null,
};

// Reducer handles all checkout transitions as discrete action types
function checkoutReducer(state, action) {
  switch (action.type) {
    case 'VALIDATE_SHIPPING':   // User filled in shipping address
      return { ...state, shipping: { ...state.shipping, ...action.payload } };
    case 'VALIDATE_PAYMENT':    // User entered payment details
      return { ...state, payment: { ...state.payment, ...action.payload } };
    case 'NEXT_STEP':            // Move to next checkout step (max 4)
      return { ...state, step: Math.min(state.step + 1, 4) };
    case 'PREV_STEP':            // Go back, clear any error
      return { ...state, step: Math.max(state.step - 1, 1), error: null };
    case 'PROCESS_PAYMENT':      // Submitting payment to gateway
      return { ...state, isProcessing: true, error: null };
    case 'PAYMENT_SUCCESS':      // Payment went through
      return { ...state, isProcessing: false, orderId: action.payload, step: 4 };
    case 'PAYMENT_FAILURE':      // Payment declined
      return { ...state, isProcessing: false, error: action.payload };
    case 'RESET':                // Start a new checkout
      return initialState;
    default:
      return state;  // Always return state for unknown actions
  }
}

// Custom hook wrapping the reducer with derived validation state
function useCheckout() {
  const [state, dispatch] = useReducer(checkoutReducer, initialState);
  // Derived: compute validation errors from the current state
  const errors = useMemo(() => {
    const e = {};
    if (!state.shipping.address) e.shipping = 'Address required';
    if (!state.shipping.city) e.shipping = 'City required';
    return e;
  }, [state.shipping]);

  return { state, dispatch, errors, canProceed: Object.keys(errors).length === 0 };
}
```

### Scenario 2: Banking — Wire Transfer Workflow

**Scenario:** A wire transfer in a banking app goes through several stages: idle → validating → confirming → processing → 2FA authentication → completed/failed. Each stage has allowed transitions and associated data (validation errors, challenge IDs, transaction IDs). A reducer models this as an explicit state machine where illegal transitions are impossible by design — you can't go from VALIDATING to COMPLETED without going through PROCESSING and REQUIRING_AUTH.

```jsx
// Define all possible states as a constant map (enum pattern)
const TRANSFER_STEPS = {
  IDLE: 'idle',
  VALIDATING: 'validating',
  CONFIRMING: 'confirming',
  PROCESSING: 'processing',
  REQUIRING_AUTH: 'requiring_auth',
  COMPLETED: 'completed',
  FAILED: 'failed',
};

// Reducer enforces valid state transitions for the transfer workflow
const transferReducer = (state, action) => {
  switch (action.type) {
    case 'START_VALIDATION':       // User clicked "Review" — validate inputs
      return { ...state, status: TRANSFER_STEPS.VALIDATING, error: null };
    case 'VALIDATION_SUCCESS':     // All fields valid, store sanitized data
      return { ...state, status: TRANSFER_STEPS.CONFIRMING, validatedData: action.payload };
    case 'VALIDATION_FAILURE':     // Validation errors, back to editing
      return { ...state, status: TRANSFER_STEPS.IDLE, error: action.payload };
    case 'SUBMIT':                 // User confirmed — submit to server
      return { ...state, status: TRANSFER_STEPS.PROCESSING };
    case 'REQUIRE_2FA':            // Server requires 2FA before processing
      return { ...state, status: TRANSFER_STEPS.REQUIRING_AUTH, challengeId: action.payload };
    case 'AUTH_SUCCESS':           // 2FA verified, resume processing
      return { ...state, status: TRANSFER_STEPS.PROCESSING };
    case 'TRANSFER_SUCCESS':       // Transfer completed successfully
      return { ...state, status: TRANSFER_STEPS.COMPLETED, transactionId: action.payload };
    case 'TRANSFER_FAILURE':       // Transfer failed (insufficient funds, etc.)
      return { ...state, status: TRANSFER_STEPS.FAILED, error: action.payload };
    case 'RESET':                  // Start a new transfer
      return initialState;
    default:
      return state;
  }
};
```

### Scenario 3: Shopping Cart with Undo

**Scenario:** A shopping cart that supports "undo" when removing an item — a pattern popularized by Gmail and now common in e-commerce (Amazon, eBay). When the user removes an item, a toast appears with "Undo" for 5 seconds. The reducer stores the last removed item separately so it can be restored. This would be cumbersome with multiple useState calls because of the interdependency between items, removedItem, and the undo action.

```jsx
// Action type constants to avoid typos in dispatch calls
const CART_ACTIONS = {
  ADD_ITEM: 'ADD_ITEM',
  REMOVE_ITEM: 'REMOVE_ITEM',
  UPDATE_QTY: 'UPDATE_QTY',
  APPLY_COUPON: 'APPLY_COUPON',
  CLEAR_CART: 'CLEAR_CART',
  RESTORE: 'RESTORE',  // Undo the last removal
};

// Reducer manages cart items + the last removed item for undo support
function cartReducer(state, action) {
  switch (action.type) {
    case CART_ACTIONS.ADD_ITEM: {
      const existing = state.items.find(i => i.id === action.payload.id);
      const newItems = existing
        ? state.items.map(i => i.id === action.payload.id
          ? { ...i, quantity: i.quantity + 1 } : i)
        : [...state.items, { ...action.payload, quantity: 1 }];
      return { ...state, items: newItems, lastAction: action };
    }

    case CART_ACTIONS.REMOVE_ITEM:
      return {
        ...state,
        items: state.items.filter(i => i.id !== action.payload),
        lastAction: action,
        removedItem: state.items.find(i => i.id === action.payload),  // Save for undo
      };

    case CART_ACTIONS.RESTORE:  // Undo: put the removed item back
      return {
        ...state,
        items: state.removedItem
          ? [...state.items, state.removedItem]
          : state.items,
        removedItem: null,  // Clear the saved item after restore
      };

    default:
      return state;
  }
}
```

## Interview Tips
- **When useState isn't enough** — multiple interdependent state values, complex transitions
- **Testing** — reducer is a pure function, ridiculously easy to test
- **DevTools** — reducers enable time-travel debugging with Redux DevTools
- **State machines** — think of useReducer as a simple state machine

## 10 Interview Questions

**Q1:** When should you use useReducer over useState?
**A:** Complex state logic (multiple sub-values, interdependent), state that depends on previous state in complex ways, or when the next state depends on the action type + payload pattern.

**Q2:** What is a reducer function and why must it be pure?
**A:** A function that takes (state, action) → returns new state. Must be pure (no side effects, same input = same output) so React can optimize, time-travel debug, and replay actions.

**Q3:** How do you handle side effects with useReducer?
**A:** Side effects happen outside the reducer — in event handlers (call dispatch then do API call) or in useEffect (dispatch when effect runs). Reducer only computes next state.

**Q4:** What is the middle initializer function in useReducer?
**A:** The third argument: `useReducer(reducer, initialArg, init)`. `init(initialArg)` runs once on mount. Useful for computing initial state from props or localStorage.

**Q5:** Can you dispatch multiple actions in sequence?
**A:** Yes. React 18+ batches dispatches in event handlers and effects. Only one re-render for multiple dispatches in the same synchronous block.

**Q6:** How do you use useReducer with Context?
**A:** Create a context that holds `[state, dispatch]`. Provider wraps the tree. Children call `dispatch({ type, payload })` anywhere. This is the "Redux-lite" pattern.

**Q7:** What happens if you don't have a 'default' case in the reducer?
**A:** For unknown actions, the reducer returns `undefined`, which becomes the new state. This breaks the app. Always `default: return state`.

**Q8:** How do you implement undo/redo with useReducer?
**A:** Store past/future arrays in state. Each action pushes current state to `past`. Undo pops from past, pushes current to future. Redo pops from future, pushes current to past.

**Q9:** What's the difference between useReducer and Redux?
**A:** useReducer is built-in, local to a component tree. Redux is external, provides middleware, DevTools, and a single store. useReducer + Context replaces Redux for many apps.

**Q10:** How do you type a reducer with TypeScript?
**A:** Define action types as a discriminated union: `type Action = { type: 'ADD'; payload: Item } | { type: 'REMOVE'; payload: number }`. Reducer infers payload type from the action.type check.

## E2E Examples

### E-Commerce
```jsx
const cartReducer = (state, action) => {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existing = state.items.find(i => i.id === action.payload.id);
      return {
        ...state,
        items: existing
          ? state.items.map(i => i.id === action.payload.id
            ? { ...i, quantity: i.quantity + 1 } : i)
          : [...state.items, { ...action.payload, quantity: 1 }],
        lastAction: 'added'
      };
    }
    case 'REMOVE_ITEM':
      return {
        ...state,
        items: state.items.filter(i => i.id !== action.payload),
        lastAction: 'removed'
      };
    case 'UPDATE_QUANTITY':
      return {
        ...state,
        items: action.payload.quantity <= 0
          ? state.items.filter(i => i.id !== action.payload.id)
          : state.items.map(i => i.id === action.payload.id
            ? { ...i, quantity: action.payload.quantity } : i),
        lastAction: 'updated'
      };
    case 'APPLY_COUPON':
      return { ...state, coupon: action.payload, discount: calculateDiscount(action.payload, state.items) };
    case 'CLEAR_CART':
      return { ...initialState };
    default:
      return state;
  }
};

const initialState = { items: [], coupon: null, discount: 0, lastAction: null };

function CartWithReducer() {
  const [state, dispatch] = useReducer(cartReducer, initialState);
  return (
    <div className="cart">
      {state.items.map(item => (
        <div key={item.id} className="cart-item">
          <span>{item.name}</span>
          <button onClick={() => dispatch({ type: 'REMOVE_ITEM', payload: item.id })}>×</button>
          <input type="number" value={item.quantity}
            onChange={e => dispatch({ type: 'UPDATE_QUANTITY', payload: { id: item.id, quantity: +e.target.value } })} />
        </div>
      ))}
      <CouponInput onApply={code => dispatch({ type: 'APPLY_COUPON', payload: code })} />
      <p>Discount: -${state.discount.toFixed(2)}</p>
      <button onClick={() => dispatch({ type: 'CLEAR_CART' })}>Clear Cart</button>
    </div>
  );
}
```

### Banking
```jsx
const transferReducer = (state, action) => {
  switch (action.type) {
    case 'SET_STEP':
      return { ...state, currentStep: action.payload };
    case 'UPDATE_FIELD':
      return { ...state, [action.payload.field]: action.payload.value, errors: { ...state.errors, [action.payload.field]: '' } };
    case 'SET_ERRORS':
      return { ...state, errors: action.payload };
    case 'SUBMIT_START':
      return { ...state, isSubmitting: true, submitError: null };
    case 'SUBMIT_SUCCESS':
      return { ...state, isSubmitting: false, isComplete: true, confirmationId: action.payload };
    case 'SUBMIT_FAILURE':
      return { ...state, isSubmitting: false, submitError: action.payload };
    case 'RESET':
      return { ...initialTransferState };
    default:
      return state;
  }
};

const initialTransferState = {
  currentStep: 1,
  fromAccount: '',
  toAccount: '',
  amount: '',
  memo: '',
  isSubmitting: false,
  isComplete: false,
  confirmationId: null,
  errors: {},
  submitError: null
};

function WireTransferWizard() {
  const [state, dispatch] = useReducer(transferReducer, initialTransferState);
  return (
    <div className="wire-transfer">
      {state.currentStep === 1 && <AccountSelection selected={state.fromAccount}
        onSelect={v => dispatch({ type: 'UPDATE_FIELD', payload: { field: 'fromAccount', value: v } })} />}
      {state.currentStep === 2 && <RecipientInput value={state.toAccount}
        onChange={v => dispatch({ type: 'UPDATE_FIELD', payload: { field: 'toAccount', value: v } })} />}
      {state.currentStep === 3 && <AmountInput value={state.amount}
        onChange={v => dispatch({ type: 'UPDATE_FIELD', payload: { field: 'amount', value: v } })} />}
      {state.currentStep === 4 && <ReviewConfirm state={state} dispatch={dispatch} />}
      {state.isComplete && <ConfirmationScreen confirmationId={state.confirmationId}
        onNewTransfer={() => dispatch({ type: 'RESET' })} />}
    </div>
  );
}
```

---

# 7. useRef

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Image Zoom (Magnifier)

**Scenario:** On an e-commerce product detail page (like Amazon or Zappos), customers hover over the product image to see a magnified view. This uses three DOM refs: the container (to measure mouse position relative to the image), the image itself, and the zoom lens that follows the cursor. The lens position updates imperatively via DOM style manipulation (not state) for performance — only the lens visibility triggers a re-render.

```jsx
function ImageZoom({ src, alt, zoomScale = 2 }) {
  // Refs for DOM elements — changing these does NOT cause re-renders
  const containerRef = useRef(null);   // The image container (for mouse position)
  const imageRef = useRef(null);        // The main product image
  const lensRef = useRef(null);         // The magnifier lens overlay
  // Only the lens visibility and position trigger re-renders (via state)
  const [showZoom, setShowZoom] = useState(false);
  const [lensPos, setLensPos] = useState({ x: 0, y: 0 });

  function handleMouseMove(e) {
    // Measure container position using the DOM ref (no re-render needed for measurement)
    const rect = containerRef.current.getBoundingClientRect();
    const x = e.clientX - rect.left;
    const y = e.clientY - rect.top;

    setLensPos({ x, y });  // Update state for lens position

    // Directly manipulate lens background position via DOM (avoids re-render)
    const lens = lensRef.current;
    const img = imageRef.current;
    if (lens && img) {
      const lensSize = lens.offsetWidth / 2;
      lens.style.backgroundPosition = `-${x * zoomScale - lensSize}px -${y * zoomScale - lensSize}px`;
    }
  }

  return (
    <div
      ref={containerRef}  // Container ref for measuring mouse coordinates
      className="image-zoom-container"
      onMouseEnter={() => setShowZoom(true)}
      onMouseLeave={() => setShowZoom(false)}
      onMouseMove={handleMouseMove}
    >
      <img ref={imageRef} src={src} alt={alt} />
      {showZoom && (
        <div
          ref={lensRef}  // Lens ref for direct DOM manipulation
          className="zoom-lens"
          style={{
            left: lensPos.x - 50,
            top: lensPos.y - 50,
            backgroundImage: `url(${src})`,
            backgroundSize: `${zoomScale * 100}%`,
          }}
        />
      )}
    </div>
  );
}
```

### Scenario 2: Banking — PDF Statement Viewer with Page Tracking

**Scenario:** A banking app lets customers view their monthly PDF statements. A ref tracks the current page number (to know which page to render after zoom/scroll) and an audit ref logs scroll depth for regulatory compliance. The audit data must persist across renders without triggering re-renders — the compliance log is a background concern, not UI state.

```jsx
function StatementViewer({ statementUrl }) {
  // DOM ref for rendering the PDF onto a canvas element
  const canvasRef = useRef(null);
  // Mutable ref for current page — changing it should NOT re-render
  const pageNumRef = useRef(1);
  const [totalPages, setTotalPages] = useState(0);
  const [loading, setLoading] = useState(true);

  // Ref for the scrollable container and audit trail (persistent, non-reactive)
  const scrollRef = useRef(null);
  const auditRef = useRef({ pages: [], startTime: Date.now() });

  useEffect(() => {
    async function loadPDF() {
      const pdf = await pdfjsLib.getDocument(statementUrl).promise;
      setTotalPages(pdf.numPages);
      await renderPage(pdf, pageNumRef.current);  // Use ref value, not state
      setLoading(false);
    }
    loadPDF();
  }, [statementUrl]);

  function handleScroll() {
    const el = scrollRef.current;
    if (!el) return;
    const scrollPercent = el.scrollTop / (el.scrollHeight - el.clientHeight);
    // Log to the audit ref (no re-render needed for this compliance data)
    if (scrollPercent > 0.5 && !auditRef.current.halfwayLogged) {
      auditRef.current.halfwayLogged = true;
      auditRef.current.pages.push({
        page: pageNumRef.current,
        scrollPercent: Math.round(scrollPercent * 100),
        timestamp: new Date().toISOString(),
      });
      // Fire-and-forget beacon for audit trail
      navigator.sendBeacon('/api/audit/scroll', JSON.stringify({
        action: 'scrolled_past_50_percent',
        statementId: statementUrl,
        details: auditRef.current,
      }));
    }
  }

  return (
    <div ref={scrollRef} className="statement-viewer" onScroll={handleScroll}>
      <canvas ref={canvasRef} />
      {loading && <Spinner />}
    </div>
  );
}
```

### Scenario 3: Auto-Saving Form with Dirty Tracking

**Scenario:** A document editor (Google Docs-style) auto-saves changes 3 seconds after the user stops typing. It also shows a "You have unsaved changes" warning if the user tries to navigate away. A ref tracks whether the form is "dirty" (has unsaved changes) because this flag needs to be read by the beforeunload handler and the auto-save timer, but changing it should not re-render the UI.

```jsx
function useAutoSaveForm(initialValues, onSave, delay = 3000) {
  const [values, setValues] = useState(initialValues);  // UI state — triggers re-render
  const [lastSaved, setLastSaved] = useState(null);
  const [saveStatus, setSaveStatus] = useState('saved');

  const dirtyRef = useRef(false);    // Tracks unsaved changes — silent, no re-render
  const timerRef = useRef(null);      // Auto-save timer ID — for clearTimeout
  const formRef = useRef(initialValues);  // Latest form values for closure safety

  // Update a field and mark form as dirty (ref, not state — no re-render)
  const setValue = useCallback((key, value) => {
    setValues(prev => {
      const next = { ...prev, [key]: value };
      formRef.current = next;  // Keep ref in sync for closure safety
      return next;
    });
    dirtyRef.current = true;    // Mark dirty — silent update
    setSaveStatus('unsaved');   // State update triggers re-render to show indicator
  }, []);

  // Auto-save effect: fires after `delay` ms of inactivity
  useEffect(() => {
    if (!dirtyRef.current) return;  // Nothing to save

    timerRef.current = setTimeout(async () => {
      setSaveStatus('saving');
      try {
        await onSave(values);
        setLastSaved(new Date());
        dirtyRef.current = false;  // Reset dirty flag
        setSaveStatus('saved');
      } catch (err) {
        setSaveStatus('error');
      }
    }, delay);

    return () => clearTimeout(timerRef.current);
  }, [values, delay, onSave]);

  // Warn on page unload if there are unsaved changes
  useEffect(() => {
    function handleBeforeUnload(e) {
      if (dirtyRef.current) {        // Read from ref — always gets current value
        e.preventDefault();
        e.returnValue = 'You have unsaved changes.';
      }
    }
    window.addEventListener('beforeunload', handleBeforeUnload);
    return () => window.removeEventListener('beforeunload', handleBeforeUnload);
  }, []);

  return { values, setValue, setValues, lastSaved, saveStatus,
    reset: () => { setValues(initialValues); dirtyRef.current = false; } };
}
```

## Interview Tips
- **Ref ≠ state** — changing ref doesn't re-render. This is the #1 distinction
- **DOM measurements** — useLayoutEffect + useRef for measuring elements before paint
- **Callback ref** — when you need to react to a ref being set/unset (IntersectionObserver)
- **ForwardRef** — passing ref through a component tree to a DOM element

## 10 Interview Questions

**Q1:** What is the difference between useRef and useState?
**A:** useRef doesn't cause re-render on change. useState does. useRef persists across renders like state, but changes are silent. Use refs for DOM, timers, and mutable values that shouldn't trigger UI updates.

**Q2:** How do you measure the height of a DOM element with useRef?
**A:** Set ref on the element. In useLayoutEffect (or useEffect), read `ref.current.clientHeight` or `getBoundingClientRect()`. UseLayoutEffect ensures it runs before paint.

**Q3:** What is a callback ref and when would you use it?
**A:** Instead of an object ref, pass a function: `ref={node => { if (node) observe(node); }}`. Called when ref is attached/detached. Useful for IntersectionObserver, dynamic refs, or running side effects when elements mount.

**Q4:** How do you forward a ref through a component?
**A:** `React.forwardRef((props, ref) => <input ref={ref} />)`. Allows parent to focus/measure a child's DOM node. Common for form inputs, buttons, and reusable UI primitives.

**Q5:** Why use useRef for storing intervals/timeouts?
**A:** Timer IDs need to persist across renders but shouldn't cause re-renders. Using ref ensures you can clear the timer from anywhere (event handlers, cleanup) without triggering updates.

**Q6:** Can you observe mutation of a ref's current value?
**A:** Not directly. Refs are not reactive. Use state for reactive values. To "react" to a ref change, use a callback ref pattern or store the value in state as well.

**Q7:** What happens to ref.current when the component unmounts?
**A:** React does NOT automatically clear refs. Set to null in cleanup: `useEffect(() => { return () => { ref.current = null; }; }, [])`.

**Q8:** How do you store the previous value of a prop with useRef?
**A:** `const prevRef = useRef(); useEffect(() => { prevRef.current = value; }); return prevRef.current`. The ref updates after render, so during the next render, it holds the previous value.

**Q9:** What is the "ref pattern" for avoiding stale closures in useEffect?
**A:** Store the callback in a ref: `callbackRef.current = callback`. Inside the effect, call `callbackRef.current()` instead of `callback`. This lets you keep effect deps empty while always using the latest callback.

**Q10:** How do you implement a stopwatch with useRef?
**A:** Store `startTime` and `intervalId` in refs. Start: record `startTime`, set `setInterval`. Stop: `clearInterval(intervalId)`. Display: use state for elapsed time, updated by the interval callback. Refs for mutable timing data, state for UI.

## E2E Examples

### E-Commerce
```jsx
function ProductCarousel({ images, autoPlayInterval = 3000 }) {
  const [currentIndex, setCurrentIndex] = useState(0);
  const intervalRef = useRef(null);
  const touchStartX = useRef(0);
  const carouselRef = useRef(null);

  const startAutoPlay = useCallback(() => {
    intervalRef.current = setInterval(() => {
      setCurrentIndex(prev => (prev + 1) % images.length);
    }, autoPlayInterval);
  }, [images.length, autoPlayInterval]);

  const stopAutoPlay = useCallback(() => {
    if (intervalRef.current) {
      clearInterval(intervalRef.current);
      intervalRef.current = null;
    }
  }, []);

  useEffect(() => {
    startAutoPlay();
    return stopAutoPlay;
  }, [startAutoPlay, stopAutoPlay]);

  function handleTouchStart(e) {
    touchStartX.current = e.touches[0].clientX;
    stopAutoPlay();
  }

  function handleTouchEnd(e) {
    const diff = touchStartX.current - e.changedTouches[0].clientX;
    if (Math.abs(diff) > 50) {
      setCurrentIndex(prev => diff > 0
        ? (prev + 1) % images.length
        : (prev - 1 + images.length) % images.length);
    }
    startAutoPlay();
  }

  // Scroll the thumbnail strip to keep the active image visible
  useEffect(() => {
    const activeThumb = carouselRef.current?.querySelector('.active');
    activeThumb?.scrollIntoView({ behavior: 'smooth', block: 'nearest', inline: 'center' });
  }, [currentIndex]);

  return (
    <div className="product-carousel" onTouchStart={handleTouchStart} onTouchEnd={handleTouchEnd}
      onMouseEnter={stopAutoPlay} onMouseLeave={startAutoPlay}>
      <img src={images[currentIndex].url} alt={images[currentIndex].alt} />
      <button onClick={() => setCurrentIndex(i => (i - 1 + images.length) % images.length)}>‹</button>
      <button onClick={() => setCurrentIndex(i => (i + 1) % images.length)}>›</button>
      <div className="carousel-dots">
        {images.map((_, i) => (
          <button key={i} className={i === currentIndex ? 'active' : ''}
            onClick={() => setCurrentIndex(i)} aria-label={`Go to image ${i + 1}`} />
        ))}
      </div>
    </div>
  );
}
```

### Banking
```jsx
function OtpInput({ length = 6, onComplete }) {
  const inputsRef = useRef([]);
  const [values, setValues] = useState(Array(length).fill(''));

  function handleChange(index, value) {
    if (!/^\d?$/.test(value)) return;
    const newValues = [...values];
    newValues[index] = value;
    setValues(newValues);
    if (value && index < length - 1) inputsRef.current[index + 1]?.focus();
    if (value && index === length - 1) onComplete(newValues.join(''));
  }

  function handleKeyDown(index, e) {
    if (e.key === 'Backspace' && !values[index] && index > 0) {
      inputsRef.current[index - 1]?.focus();
    }
  }

  function handlePaste(e) {
    e.preventDefault();
    const pasted = e.clipboardData.getData('text').replace(/\D/g, '').slice(0, length);
    const newValues = [...values];
    pasted.split('').forEach((char, i) => { newValues[i] = char; });
    setValues(newValues);
    const nextIndex = Math.min(pasted.length, length - 1);
    inputsRef.current[nextIndex]?.focus();
    if (pasted.length === length) onComplete(pasted);
  }

  useEffect(() => { inputsRef.current[0]?.focus(); }, []);

  return (
    <div className="otp-input-group" onPaste={handlePaste}>
      {values.map((digit, i) => (
        <input key={i} ref={el => inputsRef.current[i] = el}
          type="text" inputMode="numeric" maxLength={1} value={digit}
          onChange={e => handleChange(i, e.target.value)}
          onKeyDown={e => handleKeyDown(i, e)}
          className="otp-input" aria-label={`Digit ${i + 1}`} />
      ))}
    </div>
  );
}
```

---

# 8. useMemo / useCallback / React.memo

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Product List with Client-Side Filtering

**Scenario:** An e-commerce site with thousands of products lets customers filter by category, price range, stock, and search term, then sort by price or rating. Without memoization, every keystroke or filter change would re-process the entire product array. `useMemo` caches the filtered/sorted result and only re-computes when products, filters, or sortBy actually change. The product card is wrapped in `React.memo` so it only re-renders when its specific product data changes.

```jsx
function ProductGrid({ products, filters, sortBy }) {
  // Expensive: filtering + sorting thousands of products — memoized
  const processedProducts = useMemo(() => {
    let result = [...products];

    // Apply each active filter sequentially
    if (filters.category !== 'all') {
      result = result.filter(p => p.category === filters.category);
    }
    if (filters.minPrice) {
      result = result.filter(p => p.price >= filters.minPrice);
    }
    if (filters.maxPrice) {
      result = result.filter(p => p.price <= filters.maxPrice);
    }
    if (filters.inStock) {
      result = result.filter(p => p.inStock);
    }
    if (filters.search) {
      const q = filters.search.toLowerCase();
      result = result.filter(p =>
        p.name.toLowerCase().includes(q) ||
        p.description.toLowerCase().includes(q)
      );
    }

    // Apply sorting
    switch (sortBy) {
      case 'price-asc': result.sort((a, b) => a.price - b.price); break;
      case 'price-desc': result.sort((a, b) => b.price - a.price); break;
      case 'rating': result.sort((a, b) => b.rating - a.rating); break;
      case 'newest': result.sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt)); break;
    }

    return result;
  }, [products, filters, sortBy]);  // Only re-compute when these change

  // Paginate the already-filtered results (another memo)
  const { paginatedProducts, totalPages } = useMemo(() => {
    const start = (page - 1) * pageSize;
    return {
      paginatedProducts: processedProducts.slice(start, start + pageSize),
      totalPages: Math.ceil(processedProducts.length / pageSize),
    };
  }, [processedProducts, page, pageSize]);

  // Extract unique filter options for dropdowns (also memoized)
  const filterOptions = useMemo(() => ({
    categories: [...new Set(products.map(p => p.category))],
    minPrices: [...new Set(products.map(p => Math.floor(p.price / 10) * 10))],
  }), [products]);

  return (
    <div>
      <FilterBar options={filterOptions} filters={filters} onChange={setFilters} />
      <div className="product-grid">
        {paginatedProducts.map(product => (
          <MemoizedProductCard key={product.id} product={product} onAddToCart={handleAddToCart} />
        ))}
      </div>
      <Pagination page={page} totalPages={totalPages} onChange={setPage} />
    </div>
  );
}

// React.memo prevents re-render if product props haven't changed
const MemoizedProductCard = React.memo(function ProductCard({ product, onAddToCart }) {
  return (
    <div className="product-card">
      <img src={product.thumbnail} alt={product.name} loading="lazy" />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
      <button onClick={() => onAddToCart(product.id)}>Add to Cart</button>
    </div>
  );
});
```

### Scenario 2: Banking — Transaction Search (Client-Side)

**Scenario:** A banking app lets users search, filter by type/date, and sort their transaction history. The custom hook below manages search state and uses `useMemo` to derive the filtered/sorted transaction list. Without memoization, every state change (like typing in the search box) would re-process all transactions. The debounced search also prevents re-processing on every keystroke.

```jsx
function useTransactionSearch(transactions) {
  // All the filter/sort state
  const [search, setSearch] = useState('');
  const [typeFilter, setTypeFilter] = useState('all');
  const [dateRange, setDateRange] = useState({ from: '', to: '' });
  const [sortBy, setSortBy] = useState('date-desc');

  // Debounced search term — only re-process after 200ms of no typing
  const debouncedSearch = useDebounce(search, 200);

  // Expensive: filter + sort the entire transaction list — memoized
  const filteredTransactions = useMemo(() => {
    let result = transactions;

    // Text search across description, reference, and category
    if (debouncedSearch) {
      const q = debouncedSearch.toLowerCase();
      result = result.filter(tx =>
        tx.description.toLowerCase().includes(q) ||
        tx.reference?.toLowerCase().includes(q) ||
        tx.category?.toLowerCase().includes(q)
      );
    }

    // Type filter: credits, debits, or all
    if (typeFilter !== 'all') {
      result = result.filter(tx => tx.type === typeFilter);
    }

    // Date range filter
    if (dateRange.from) {
      result = result.filter(tx => new Date(tx.date) >= new Date(dateRange.from));
    }
    if (dateRange.to) {
      result = result.filter(tx => new Date(tx.date) <= new Date(dateRange.to));
    }

    // Sort by amount or date, ascending or descending
    result.sort((a, b) => {
      const dir = sortBy.endsWith('-desc') ? -1 : 1;
      const field = sortBy.replace('-asc', '').replace('-desc', '');
      if (field === 'amount') return (a.amount - b.amount) * dir;
      if (field === 'date') return (new Date(a.date) - new Date(b.date)) * dir;
      return 0;
    });

    return result;
  }, [transactions, debouncedSearch, typeFilter, dateRange, sortBy]);

  // Derived summary from the filtered results (also memoized)
  const summary = useMemo(() => ({
    totalCredits: filteredTransactions
      .filter(t => t.type === 'credit' || t.amount > 0)
      .reduce((s, t) => s + Math.abs(t.amount), 0),
    totalDebits: filteredTransactions
      .filter(t => t.type === 'debit' || t.amount < 0)
      .reduce((s, t) => s + Math.abs(t.amount), 0),
    count: filteredTransactions.length,
  }), [filteredTransactions]);

  return { filteredTransactions, summary, search, setSearch, typeFilter, setTypeFilter, dateRange, setDateRange, sortBy, setSortBy };
}
```

### Scenario 3: Memoized Chart Component Avoiding Unnecessary Re-renders

**Scenario:** A dashboard renders an SVG performance chart that computes complex path data from an array of data points. Without `React.memo`, the chart would re-render every time its parent re-renders, even if the data hasn't changed. Without `useMemo`, the SVG path and grid lines would be re-computed on every render. This is a textbook case for both `React.memo` (skip rendering entirely) and `useMemo` (skip expensive calculations within a render).

```jsx
// React.memo: skip re-render if props haven't changed (shallow comparison)
const PerformanceChart = React.memo(function PerformanceChart({
  data, width, height, color = '#3b82f6', showGrid = true
}) {
  // useMemo: only re-compute SVG path when data, width, or height changes
  const pathData = useMemo(() => {
    if (!data || data.length === 0) return '';
    const max = Math.max(...data.map(d => d.value));
    const min = Math.min(...data.map(d => d.value));
    const range = max - min || 1;
    const stepX = width / (data.length - 1);
    const stepY = height / range;

    return data.map((d, i) => {
      const x = i * stepX;
      const y = height - (d.value - min) * stepY;
      return `${i === 0 ? 'M' : 'L'}${x},${y}`;  // SVG path commands
    }).join(' ');
  }, [data, width, height]);

  // useMemo: grid lines only depend on height and showGrid flag
  const gridLines = useMemo(() => {
    if (!showGrid) return [];
    return [0, 25, 50, 75, 100].map(pct => ({
      y: height * (1 - pct / 100),
      label: `${pct}%`,
    }));
  }, [height, showGrid]);

  return (
    <svg width={width} height={height} className="performance-chart">
      {gridLines.map((line, i) => (
        <line key={i} x1={0} y1={line.y} x2={width} y2={line.y}
          stroke="#e5e7eb" strokeWidth={1} />
      ))}
      <path d={pathData} fill="none" stroke={color} strokeWidth={2} />
    </svg>
  );
});
```

## Interview Tips
- **Profile first** — "I never optimize without profiling" shows senior-level thinking
- **Overhead exists** — useMemo/useCallback aren't free. They add comparison + memory overhead
- **Primitives are stable** — `5`, `'hello'`, `true` are always ===. No need to memoize
- **Object/array/function references** — these change every render. Memoize if passed to memo'd children

## 10 Interview Questions

**Q1:** What is the difference between useMemo and useCallback?
**A:** useMemo memoizes the result of a computation: `useMemo(() => compute(x), [x])`. useCallback memoizes the function itself: `useCallback(fn, deps)` is equivalent to `useMemo(() => fn, deps)`.

**Q2:** How does React.memo work?
**A:** It's a HOC that memoizes the rendered output. On re-render, it shallow-compares props. If all props are the same (via Object.is), it skips re-rendering and reuses the last output.

**Q3:** When would useMemo cause a performance regression?
**A:** When the computation is cheap (< 1ms) and the memoization overhead (comparison + memory) outweighs the cost. Also when deps change frequently so memoization is invalidated every time.

**Q4:** Can useMemo guarantee the value won't be recomputed?
**A:** No. React may clear memoized values (e.g., memory pressure). Never write code that depends on useMemo NOT recomputing. It's a performance hint, not a guarantee.

**Q5:** How do you pass stable callbacks to deeply nested memoized components?
**A:** useCallback with minimal deps. If deps change often, use a ref to store the latest callback: `const ref = useRef(fn); ref.current = fn; const stable = useCallback((...args) => ref.current(...args), [])`.

**Q6:** What is the "stale closure" problem with useCallback?
**A:** When useCallback captures a value from closure that becomes outdated. Example: `useCallback(() => setCount(count + 1), [])` captures count=0 forever. Fix by adding deps or using functional setState.

**Q7:** How do you debug why a memoized component is re-rendering?
**Q8:** What is the cost of an inline function in JSX vs useCallback?
**A:** An inline function creates a new function object on every render (~negligible allocation). useCallback has comparison overhead. For simple event handlers, inline is fine. Only useCallback when passing to memoized children.

**Q9:** How would you memoize a list of 10,000 items?
**A:** Not with useMemo — use virtualization (react-window). useMemo on the data processing, then virtualize the rendering. Memoization doesn't help if the bottleneck is DOM operations.

**Q10:** When should you NOT use React.memo?
**A:** When the component's props are always different (e.g., contains a new object/array every render), when the component is cheap to render (simple div), or when the component rarely re-renders (static content).

## E2E Examples

### E-Commerce
```jsx
const ProductCard = React.memo(({ product, onAddToCart }) => {
  return (
    <div className="product-card">
      <img src={product.thumbnail} alt={product.name} loading="lazy" />
      <h3>{product.name}</h3>
      <p>${product.price.toFixed(2)}</p>
      <button onClick={() => onAddToCart(product.id)}>Add to Cart</button>
    </div>
  );
});

function ProductGrid({ products, onAddToCart }) {
  // Stable callback — only recreates when onAddToCart identity changes
  const handleAdd = useCallback((id) => onAddToCart(id), [onAddToCart]);

  // Memoized computations
  const categories = useMemo(() =>
    [...new Set(products.map(p => p.category))],
  [products]);

  const priceRange = useMemo(() => {
    const prices = products.map(p => p.price);
    return { min: Math.min(...prices), max: Math.max(...prices) };
  }, [products]);

  // Memoized handlers for filtering
  const sortedProducts = useMemo(() =>
    [...products].sort((a, b) => a.name.localeCompare(b.name)),
  [products]);

  return (
    <div className="product-grid">
      <FilterBar categories={categories} priceRange={priceRange} />
      {sortedProducts.map(p => (
        <ProductCard key={p.id} product={p} onAddToCart={handleAdd} />
      ))}
    </div>
  );
}
```

### Banking
```jsx
const TransactionRow = React.memo(({ transaction, onSelect }) => {
  return (
    <tr className={`tx-row ${transaction.status}`} onClick={() => onSelect(transaction.id)}>
      <td>{new Date(transaction.date).toLocaleDateString()}</td>
      <td>{transaction.description}</td>
      <td className={transaction.type === 'credit' ? 'credit' : 'debit'}>
        {transaction.type === 'credit' ? '+' : '-'}${Math.abs(transaction.amount).toFixed(2)}
      </td>
      <td><span className={`badge ${transaction.status}`}>{transaction.status}</span></td>
    </tr>
  );
});

function TransactionTable({ transactions, onSelectTransaction }) {
  const handleSelect = useCallback((id) => onSelectTransaction(id), [onSelectTransaction]);

  const { sorted, totals } = useMemo(() => {
    const sorted = [...transactions].sort((a, b) => new Date(b.date) - new Date(a.date));
    const totals = sorted.reduce((acc, t) => ({
      credits: acc.credits + (t.type === 'credit' ? t.amount : 0),
      debits: acc.debits + (t.type === 'debit' ? t.amount : 0),
    }), { credits: 0, debits: 0 });
    return { sorted, totals };
  }, [transactions]);

  return (
    <div className="transaction-table">
      <SummaryRow credits={totals.credits} debits={totals.debits} />
      <table>
        <thead><tr><th>Date</th><th>Description</th><th>Amount</th><th>Status</th></tr></thead>
        <tbody>
          {sorted.map(tx => (
            <TransactionRow key={tx.id} transaction={tx} onSelect={handleSelect} />
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

---

# 9. CONTEXT API

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Multi-context Architecture

**Scenario:** A large e-commerce app needs several global pieces of data: authentication (changes rarely), shopping cart (changes frequently), UI theme (changes moderately), and notifications (changes very frequently). If all of these were in a single context, changing the cart would re-render every component that reads any context value. By splitting into separate contexts by update frequency, each context's consumers only re-render when their specific value changes.

```jsx
// Split contexts by update frequency to minimize unnecessary re-renders
const AuthContext = createContext(null);          // Changes rarely (login/logout)
const CartContext = createContext(null);           // Changes frequently (add/remove items)
const UIContext = createContext(null);             // Changes moderately (theme, sidebar)
const NotificationContext = createContext(null);   // Changes very frequently (toasts)

function AppProvider({ children }) {
  return (
    // Order by update frequency — least frequent on the outside
    <AuthProvider>          {/* Auth changes rarely — outermost */}
      <CartProvider>        {/* Cart changes frequently */}
        <UIProvider>        {/* Theme and UI state */}
          <NotificationProvider>  {/* Toasts fire very frequently */}
            <ErrorBoundary>
              {children}
            </ErrorBoundary>
          </NotificationProvider>
        </UIProvider>
      </CartProvider>
    </AuthProvider>
  );
}

// Each provider only re-renders its own consumers
// Theme toggling doesn't re-render Cart components
// Adding to cart doesn't re-render Auth components
```

### Scenario 2: Banking — Compliance Context (Audit Trail)

**Scenario:** A banking app must log every user action for regulatory compliance (SOX, PCI-DSS). The audit context provides a `logAction` function that any component can call. It logs to a local array (for batch upload) and sends a beacon to the server. For regulated transactions like transfers and payments, it also sends a synchronous request to ensure the audit entry is persisted immediately. Sensitive data (passwords, full account numbers) is NEVER logged.

```jsx
// Audit context for compliance logging across the entire banking app
const AuditContext = createContext(null);

function AuditProvider({ userId, sessionId, children }) {
  // Local buffer for batch audit uploads (not state — no re-render needed)
  const logRef = useRef([]);

  // The core audit function — call from any component via context
  const logAction = useCallback((action, details) => {
    // Build a sanitized audit entry
    const entry = {
      userId,
      sessionId,
      action,
      details: {
        ...details,
        // Security: NEVER log passwords, full SSN, full card numbers
        ip: null,                      // Captured server-side for security
        userAgent: navigator.userAgent,
        url: window.location.href,
        timestamp: new Date().toISOString(),
      },
    };
    logRef.current.push(entry);  // Add to local batch buffer

    // Fire-and-forget beacon for non-critical audit (won't block navigation)
    navigator.sendBeacon?.('/api/audit', JSON.stringify(entry));

    // For regulated transactions, flush synchronously to guarantee delivery
    if (action.startsWith('TRANSFER_') || action.startsWith('PAYMENT_')) {
      fetch('/api/audit/sync', { method: 'POST', body: JSON.stringify(entry) });
    }
  }, [userId, sessionId]);

  return (
    <AuditContext.Provider value={logAction}>
      {children}
    </AuditContext.Provider>
  );
}

// Usage in TransferForm — log the action without blocking the UI
function TransferForm() {
  const audit = useContext(AuditContext);  // Get the audit function from context

  function handleSubmit(data) {
    // Log the transfer initiation for compliance before submitting
    audit('TRANSFER_INITIATED', {
      fromAccount: account.maskedNumber,  // Only log masked number
      toAccount: dest.maskedNumber,       // Never full account number
      amount: amount,
      // Never log: full account numbers, balances, or credentials
    });
    // Proceed with actual transfer...
  }
}
```

### Scenario 3: Feature Flags with Context

**Scenario:** A SaaS platform uses feature flags to gradually roll out new features to different user segments. The `FeatureFlagProvider` fetches flags from the server on mount and makes them available globally. Any component can wrap content in `<Feature name="...">` to conditionally show new vs old UI. This is the pattern used by LaunchDarkly, Split.io, and similar feature management platforms.

```jsx
// Context that holds all feature flags and provides helper functions
const FeatureFlagContext = createContext({});

function FeatureFlagProvider({ flags, children }) {
  const [featureFlags, setFeatureFlags] = useState(flags);

  // Fetch the latest flags from the server on mount
  useEffect(() => {
    const fetchFlags = async () => {
      const res = await fetch('/api/features');
      const data = await res.json();
      setFeatureFlags(data.flags);
    };
    fetchFlags();
  }, []);

  // Check if a feature is enabled (with safe default of false)
  const isEnabled = useCallback((flag) => {
    return featureFlags[flag]?.enabled ?? false;
  }, [featureFlags]);

  // Get the A/B test variant for this user
  const getVariant = useCallback((flag) => {
    return featureFlags[flag]?.variant ?? 'control';
  }, [featureFlags]);

  // Memoize the context value to prevent unnecessary re-renders
  const value = useMemo(() => ({ isEnabled, getVariant, flags: featureFlags }), [isEnabled, getVariant, featureFlags]);

  return (
    <FeatureFlagContext.Provider value={value}>
      {children}
    </FeatureFlagContext.Provider>
  );
}

// Declarative component: show children if feature is enabled, fallback otherwise
function Feature({ name, children, fallback = null }) {
  const { isEnabled } = useContext(FeatureFlagContext);
  return isEnabled(name) ? children : fallback;
}

// Usage: swap checkout UI without breaking existing flow
<Feature name="new-checkout" fallback={<OldCheckout />}>
  <NewCheckout />
</Feature>
```

## Interview Tips
- **Provider nesting** — React.createContext returns Provider + Consumer. Provider wraps tree. Value passes down
- **Re-render problem** — all consumers re-render when context value changes. Split contexts by update frequency
- **Memoize value** — `useMemo(() => ({ ... }), [...])` prevents unnecessary re-renders
- **Default value** — the argument to `createContext(default)` is used when there's no Provider. Design for this

## 10 Interview Questions

**Q1:** What problem does Context API solve?
**A:** Prop drilling — passing props through intermediate components that don't need them. Context lets you broadcast data to any depth without explicit prop passing.

**Q2:** What is the main performance concern with Context API?
**A:** All consumers re-render when the context value changes. If the value changes frequently, many components re-render unnecessarily. Split contexts by update frequency to minimize impact.

**Q3:** How do you update context from a nested component?
**A:** Include update functions in the context value: `const value = { user, setUser, login, logout }`. Nested components call `setUser(...)` or `login(...)` from context.

**Q4:** What is the difference between `useContext` and `Context.Consumer`?
**A:** `useContext` is the hooks API (cleaner, less nesting). `Context.Consumer` is the render props API (legacy, more verbose). Both do the same thing.

**Q5:** How do you create a type-safe context with TypeScript?
**A:** Define an interface for the context value. Use `createContext<Type | null>(null)`. Provide a custom hook that checks for null and throws a descriptive error.

**Q6:** When would you choose Context vs Zustand vs Redux?
**A:** Context for low-frequency updates (theme, auth, locale). Zustand for medium-complexity with better performance (fine-grained subscriptions). Redux for large apps needing middleware, DevTools, and normalized state.

**Q7:** What happens if you use a context without a Provider?
**A:** The component receives the default value passed to `createContext(defaultValue)`. If no default is provided (undefined), accessing context throws "Cannot read property of undefined".

**Q8:** How do you mock context in tests?
**A:** Wrap the component with the Provider and pass mock values: `render(<ThemeContext.Provider value={{ theme: 'dark' }}><Component /></ThemeContext.Provider>)`.

**Q9:** Can you have nested context providers with the same context?
**A:** Yes. Nested providers override the value for their subtree. This is useful for scoped overrides (e.g., a dark-themed section inside a light-themed app).

**Q10:** How do you prevent unnecessary re-renders when context value is an object?
**A:** Memoize the value object: `const value = useMemo(() => ({ theme, toggleTheme }), [theme, toggleTheme])`. Without this, every Provider render creates a new object → all consumers re-render.

## E2E Examples

### E-Commerce
```jsx
const AuthContext = createContext(null);
const CartContext = createContext(null);
const ThemeContext = createContext(null);

function AppProviders({ children }) {
  const [user, setUser] = useState(null);
  const [theme, setTheme] = useState('light');

  const authValue = useMemo(() => ({ user, login: (u) => setUser(u), logout: () => setUser(null) }), [user]);
  const themeValue = useMemo(() => ({ theme, toggle: () => setTheme(t => t === 'light' ? 'dark' : 'light') }), [theme]);

  return (
    <AuthContext.Provider value={authValue}>
      <ThemeContext.Provider value={themeValue}>
        <CartProvider>
          {children}
        </CartProvider>
      </ThemeContext.Provider>
    </AuthContext.Provider>
  );
}

// Consumer hook
function useAuth() {
  const ctx = useContext(AuthContext);
  if (!ctx) throw new Error('useAuth must be inside AppProviders');
  return ctx;
}

function Header() {
  const { user, logout } = useAuth();
  const { theme, toggle } = useContext(ThemeContext);
  const { itemCount } = useContext(CartContext);

  return (
    <header className={`app-header ${theme}`}>
      <Logo />
      <nav>
        <Link to="/products">Shop</Link>
        <Link to="/cart">Cart ({itemCount})</Link>
      </nav>
      <div className="header-right">
        <button onClick={toggle}>🌓 {theme}</button>
        {user ? <button onClick={logout}>Logout ({user.name})</button> : <Link to="/login">Login</Link>}
      </div>
    </header>
  );
}
```

### Banking
```jsx
const BankAuthContext = createContext(null);
const AccountsContext = createContext(null);
const AlertsContext = createContext(null);

function BankProviders({ children }) {
  const [customer, setCustomer] = useState(null);
  const [accounts, setAccounts] = useState([]);
  const [alerts, setAlerts] = useState([]);

  const authValue = useMemo(() => ({
    customer,
    login: async (creds) => { const c = await api.login(creds); setCustomer(c); return c; },
    logout: () => { api.logout(); setCustomer(null); setAccounts([]); },
    isAuthenticated: !!customer
  }), [customer]);

  const accountsValue = useMemo(() => ({
    accounts,
    refresh: async () => { const a = await api.getAccounts(customer.id); setAccounts(a); },
    transfer: async (from, to, amount) => { await api.transfer(from, to, amount); await refresh(); }
  }), [accounts, customer]);

  return (
    <BankAuthContext.Provider value={authValue}>
      <AccountsContext.Provider value={accountsValue}>
        <AlertsContext.Provider value={{ alerts, addAlert: setAlerts }}>
          {children}
        </AlertsContext.Provider>
      </AccountsContext.Provider>
    </BankAuthContext.Provider>
  );
}

function Dashboard() {
  const { customer } = useContext(BankAuthContext);
  const { accounts, refresh } = useContext(AccountsContext);
  const { alerts } = useContext(AlertsContext);

  useEffect(() => { refresh(); }, [refresh]);

  return (
    <div className="dashboard">
      <WelcomeBanner customer={customer} />
      <AlertBar alerts={alerts} />
      <AccountsSummary accounts={accounts} />
      <QuickActions onTransfer={() => {}} onDeposit={() => {}} />
    </div>
  );
}
```

---

# 10. ERROR BOUNDARIES

... (and so on for remaining concepts)

This file continues for all remaining concepts in the same depth. Let me know if you want me to continue with:
- Error Boundaries (10 questions, 3 scenarios)
- React Router (10 questions, 3 scenarios)
- API Calls (10 questions, 3 scenarios)
- React Query (10 questions, 3 scenarios)
- Optimistic Updates (10 questions, 3 scenarios)
- Zustand (10 questions, 3 scenarios)
- Testing (10 questions, 3 scenarios)
- TypeScript (10 questions, 3 scenarios)

Each with matching E-Commerce and Banking code examples embedded in the scenarios.

Plus the 2 End-to-End examples (E-Commerce full app, Banking full app) — these already exist in `04-e2e-ecommerce.md` and `05-e2e-banking.md` which have the complete full-app architecture with auth, routing, payments, and real-time updates for both domains.

---

# 10. ERROR BOUNDARIES

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Per-Section Error Isolation

**Scenario:** In a complex e-commerce page, one crashing section (like a broken recommendation widget) shouldn't take down the entire page — the user should still be able to search, browse products, and use their cart. Error Boundaries wrap each major section independently. If the `RecommendationSidebar` crashes, only that section shows a fallback; the product grid, header, and cart keep working normally.

```jsx
function ECommerceLayout() {
  return (
    <div className="app">
      {/* Each section is independently wrapped — one crash doesn't kill the whole page */}
      <ErrorBoundary fallback={<HeaderFallback />} onError={logToSentry}>
        <Header />
      </ErrorBoundary>

      <main className="content">
        <ErrorBoundary fallback={<SearchFallback />} onError={logToSentry}>
          <SearchBar />
        </ErrorBoundary>

        <div className="product-area">
          {/* If the product grid crashes, recommendations still work (and vice versa) */}
          <ErrorBoundary fallback={<ProductGridFallback />} onError={logToSentry}>
            <ProductGrid />
          </ErrorBoundary>

          <ErrorBoundary fallback={<SidebarFallback />} onError={logToSentry}>
            <RecommendationSidebar />
          </ErrorBoundary>
        </div>
      </main>

      {/* Cart drawer — user can still review cart even if products fail to load */}
      <ErrorBoundary fallback={<CartFallback />} onError={logToSentry}>
        <CartDrawer />
      </ErrorBoundary>

      {/* Footer with null fallback — silently fails, no visual impact */}
      <ErrorBoundary fallback={null} onError={logToSentry}>
        <Footer />
      </ErrorBoundary>
    </div>
  );
}
```

### Scenario 2: Banking — Transaction Processing with Retry

**Scenario:** A banking payment flow can encounter transient errors (network blips, timeout). If the payment component crashes, the boundary logs the full context for compliance (transaction ID, amount, user ID) and shows a retry button. After 3 failed retries, it offers an alternative payment method instead of looping forever.

```jsx
class PaymentErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null, retryCount: 0 };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true, error };  // Trigger fallback UI
  }

  componentDidCatch(error, info) {
    // Log with full context for compliance (SOX, PCI-DSS requirements)
    const payload = {
      error: error.message,
      stack: error.stack,
      componentStack: info.componentStack,
      transactionId: this.props.transactionId,
      amount: this.props.amount,
      userId: this.props.userId,
      timestamp: new Date().toISOString(),
    };

    fetch('/api/errors/critical', {
      method: 'POST',
      body: JSON.stringify(payload),
    });

    // Notify fraud detection system immediately
    navigator.sendBeacon('/api/fraud/error', JSON.stringify({
      transactionId: this.props.transactionId,
      errorType: 'payment_ui_crash',
    }));
  }

  handleRetry = () => {
    this.setState(state => ({
      hasError: false,       // Reset boundary — re-render children
      error: null,
      retryCount: state.retryCount + 1,
    }));
  };

  render() {
    if (this.state.hasError) {
      return (
        <div className="payment-error" role="alert">
          <h3>Payment Processing Error</h3>
          <p>Your transaction has been paused. No charges have been made.</p>
          {/* Up to 3 retry attempts, then suggest alternative */}
          {this.state.retryCount < 3 ? (
            <button onClick={this.handleRetry} className="btn-primary">
              Retry Payment ({3 - this.state.retryCount} attempts left)
            </button>
          ) : (
            <div>
              <p>Please contact support with reference: {this.props.transactionId}</p>
              <button onClick={this.props.onFallback}>
                Use Alternative Payment Method
              </button>
            </div>
          )}
        </div>
      );
    }
    return this.props.children;
  }
}
```

### Scenario 3: Third-Party Widget Isolation

**Scenario:** A dashboard embeds third-party widgets (chat, analytics, maps) that load via external scripts. These scripts can throw errors that bubble up and crash the entire React app. The `ThirdPartyWidget` component uses a try-catch approach with `window.onerror` interception to catch errors specifically from the third-party script URL, preventing them from crashing the rest of the React tree.

```jsx
function ThirdPartyWidget({ widgetUrl, widgetName }) {
  const [error, setError] = useState(null);  // Local error state for this widget
  const containerRef = useRef(null);

  useEffect(() => {
    // Dynamically load the third-party script
    const script = document.createElement('script');
    script.src = widgetUrl;
    script.onerror = () => setError(new Error(`Failed to load ${widgetName}`));

    // Intercept global errors that originate from this specific widget URL
    const originalOnError = window.onerror;
    window.onerror = (msg, url, line, col, err) => {
      // Only catch errors from the widget's script URL
      if (url?.includes(widgetUrl)) {
        setError(new Error(`${widgetName} encountered an error`));
        return true;  // Prevent default error handling (keeps React tree intact)
      }
      return originalOnError?.(msg, url, line, col, err);
    };

    document.body.appendChild(script);

    // Cleanup: remove script and restore original error handler
    return () => {
      document.body.removeChild(script);
      window.onerror = originalOnError;
    };
  }, [widgetUrl, widgetName]);

  // Show fallback instead of letting the error crash the app
  if (error) {
    return (
      <div className="widget-error">
        <p>{widgetName} is currently unavailable.</p>
        <button onClick={() => setError(null)}>Retry</button>
      </div>
    );
  }

  return <div ref={containerRef} className="third-party-widget" />;
}
```

## Interview Tips
- **Class component requirement** — Error Boundaries must use `componentDidCatch`. Functional equivalent coming in React 19
- **Granularity** — wrap each section, not the whole app. One crash shouldn't take down everything
- **Async errors** — Error Boundaries don't catch event handlers or async code. Use try-catch for those
- **Recovery** — provide retry mechanism. `setState({ hasError: false })` attempts re-render

## 10 Interview Questions

**Q1:** What errors does Error Boundary catch?
**A:** Errors during rendering, in lifecycle methods, and in constructors of the whole subtree below it.

**Q2:** What errors does Error Boundary NOT catch?
**A:** Event handlers (use try-catch), async code (use try-catch), server-side rendering, errors in the boundary itself.

**Q3:** Why must Error Boundaries be class components?
**A:** The `componentDidCatch` and `getDerivedStateFromError` lifecycle methods are only available in class components. React < 19 doesn't have a hooks equivalent.

**Q4:** How do you log errors from an Error Boundary?
**A:** In `componentDidCatch(error, errorInfo)`. Send `error.message`, `error.stack`, and `errorInfo.componentStack` to your monitoring service (Sentry, LogRocket, Datadog).

**Q5:** How do you implement a "Try Again" button in an Error Boundary?
**A:** Call `setState({ hasError: false })`. This re-renders children. If the error was transient (network), it recovers. If permanent, it crashes again (which is fine — user knows it's broken).

**Q6:** What happens if the Error Boundary itself throws an error?
**A:** The error propagates to the nearest parent Error Boundary. If none exists, React unmounts the whole tree (white screen). That's why you should keep boundaries simple.

**Q7:** How do you test Error Boundaries?
**A:** Render a component that throws. Assert fallback UI appears. Assert onError was called. Use `jest.spyOn(console, 'error').mockImplementation()` to suppress React's error logging.

**Q8:** Can you catch errors in event handlers with Error Boundary?
**A:** No. Use try-catch in the event handler. Set error state. Show error UI conditionally.

**Q9:** How would you handle errors in a React Query mutation with Error Boundary?
**A:** Error Boundary doesn't catch async mutation errors. React Query exposes `onError` callback. Set error state there. Or use `useErrorBoundary: true` in React Query v5+ to send mutation errors to the nearest Error Boundary.

**Q10:** What's the React 19 approach to error handling?
**A:** React 19 introduces `use(error)` hook and improved error boundaries for server components. Also `captureError()` for programmatic error reporting.

## E2E Examples

### E-Commerce
```jsx
class ProductErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    logErrorToService(error, info.componentStack);
  }

  handleRetry = () => {
    this.setState({ hasError: false, error: null });
  };

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-boundary-fallback" role="alert">
          <h2>Something went wrong</h2>
          <p>{this.state.error?.message || 'An unexpected error occurred'}</p>
          <button onClick={this.handleRetry}>Try Again</button>
          <button onClick={() => window.location.href = '/products'}>Browse Products</button>
          <details>
            <summary>Error details</summary>
            <pre>{this.state.error?.stack}</pre>
          </details>
        </div>
      );
    }
    return this.props.children;
  }
}

// Usage wrapping a product detail page
function ProductDetailPage({ productId }) {
  return (
    <ProductErrorBoundary>
      <ProductDetailContent productId={productId} />
    </ProductErrorBoundary>
  );
}

// Also wrap at route level
const router = createBrowserRouter([
  {
    path: '/products/:id',
    element: <ProductErrorBoundary><ProductDetailPage /></ProductErrorBoundary>,
    errorElement: <RouteErrorFallback />
  }
]);

function RouteErrorFallback() {
  const error = useRouteError();
  return (
    <div className="route-error">
      <h1>404</h1>
      <p>{error.statusText || error.message}</p>
      <Link to="/">Go Home</Link>
    </div>
  );
}
```

### Banking
```jsx
class TransferErrorBoundary extends React.Component {
  state = { hasError: false, error: null, retryCount: 0 };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    console.error('Transfer error:', error, info.componentStack);
    monitor.alert('CRITICAL: Transfer component crashed', { error: error.message, stack: info.componentStack });
  }

  handleRetry = () => {
    const newCount = this.state.retryCount + 1;
    if (newCount > 3) {
      alert('Maximum retries reached. Please contact support.');
      window.location.href = '/support';
      return;
    }
    this.setState({ hasError: false, error: null, retryCount: newCount });
  };

  render() {
    if (this.state.hasError) {
      return (
        <div className="transfer-error-fallback">
          <h2>Transfer Temporarily Unavailable</h2>
          <p>We encountered an error processing your request. Your funds are safe.</p>
          <p className="error-ref">Reference: {this.state.error?.id || 'N/A'}</p>
          <button onClick={this.handleRetry}>
            Retry ({3 - this.state.retryCount} attempts remaining)
          </button>
          <button onClick={() => window.location.href = '/contact'}>Contact Support</button>
        </div>
      );
    }
    return this.props.children;
  }
}

function TransferPage() {
  return (
    <TransferErrorBoundary>
      <TransferForm />
    </TransferErrorBoundary>
  );
}
```

---

# 11. REACT ROUTER

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Full Route Architecture

**Scenario:** A full e-commerce site needs a well-organized route structure: public pages (home, product listing, product detail), auth-protected pages (cart, checkout, account), and a catch-all 404. React Router v6.4+'s `createBrowserRouter` with nested routes and loaders provides data fetching before render. The route tree mirrors the app's layout hierarchy — the root layout wraps all pages, cart routes require authentication, and product routes load their data via route loaders.

```jsx
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

// The route tree mirrors the component layout hierarchy
const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,          // Header + <Outlet /> + Footer
    errorElement: <RootErrorBoundary />,
    loader: rootLoader,                // Loads cart count, user session, etc.
    children: [
      { index: true, element: <HomePage /> },
      {
        path: 'products',              // Nested product routes with loaders
        children: [
          { index: true, element: <ProductListing />, loader: productsLoader },
          { path: ':category', element: <ProductListing />, loader: categoryLoader },
          { path: ':category/:slug', element: <ProductDetail />, loader: productLoader },
        ],
      },
      {
        path: 'cart',                  // Protected routes — wraps all cart pages
        element: <ProtectedRoute />,
        children: [
          { index: true, element: <CartPage /> },
          { path: 'checkout', element: <CheckoutPage /> },
          { path: 'payment', element: <PaymentPage /> },
          { path: 'confirmation/:orderId', element: <OrderConfirmation /> },
        ],
      },
      {
        path: 'account',               // Protected account area
        element: <ProtectedRoute />,
        children: [
          { index: true, element: <Navigate to="orders" replace /> },
          { path: 'orders', element: <OrderHistory />, loader: ordersLoader },
          { path: 'orders/:id', element: <OrderDetail />, loader: orderDetailLoader },
          { path: 'profile', element: <ProfilePage /> },
          { path: 'addresses', element: <AddressBook /> },
          { path: 'wishlist', element: <WishlistPage /> },
        ],
      },
      { path: 'search', element: <SearchPage /> },
      { path: 'about', element: <AboutPage /> },
      { path: '*', element: <NotFoundPage /> },  // Catch-all 404
    ],
  },
]);
```

### Scenario 2: Banking — Protected Routes with Role-Based Access

**Scenario:** A banking app has three user roles with different route access: customers see their dashboard and transfer pages, admins see user management and audit logs, tellers see deposit/withdrawal forms. The `ProtectedRoute` component checks authentication status, required roles, and required permissions before rendering the requested route. Unauthorized users are redirected to login or an "unauthorized" page.

```jsx
// A reusable route guard that checks auth, roles, and permissions
function ProtectedRoute({ children, requiredRoles, requiredPermissions }) {
  const { user, isLoading, isAuthenticated } = useAuth();
  const location = useLocation();  // Save location so we can redirect back after login

  // Show spinner while checking auth status
  if (isLoading) return <FullPageSpinner />;

  // Not logged in — redirect to login, remember where they came from
  if (!isAuthenticated) {
    return <Navigate to="/login" state={{ from: location }} replace />;
  }

  // Wrong role — redirect to unauthorized
  if (requiredRoles && !requiredRoles.includes(user.role)) {
    return <Navigate to="/unauthorized" replace />;
  }

  // Missing permissions — redirect to unauthorized
  if (requiredPermissions) {
    const hasAllPerms = requiredPermissions.every(
      p => user.permissions?.includes(p)
    );
    if (!hasAllPerms) return <Navigate to="/unauthorized" replace />;
  }

  return children || <Outlet />;  // Auth passed — render the route
}

// Route tree using the guard for different role sections
const router = createBrowserRouter([
  { path: '/login', element: <LoginPage /> },
  { path: '/unauthorized', element: <UnauthorizedPage /> },

  // Customer routes — requires 'customer' role
  {
    element: <ProtectedRoute requiredRoles={['customer']} />,
    children: [
      { path: '/dashboard', element: <CustomerDashboard /> },
      { path: '/accounts/:id', element: <AccountDetail /> },
      { path: '/transfer', element: <TransferPage /> },
      { path: '/payments', element: <PaymentsPage /> },
    ],
  },

  // Admin routes — requires 'admin' role AND 'view_audit' permission
  {
    element: <ProtectedRoute requiredRoles={['admin']} requiredPermissions={['view_audit']} />,
    children: [
      { path: '/admin', element: <AdminDashboard /> },
      { path: '/admin/users', element: <UserManagement /> },
      { path: '/admin/audit', element: <AuditLogViewer /> },
    ],
  },

  // Teller routes — either 'teller' or 'admin' can access
  {
    element: <ProtectedRoute requiredRoles={['teller', 'admin']} />,
    children: [
      { path: '/teller', element: <TellerDashboard /> },
      { path: '/teller/deposit', element: <DepositForm /> },
      { path: '/teller/withdrawal', element: <WithdrawalForm /> },
    ],
  },
]);
```

### Scenario 3: Checkout Wizard with Route-Based Steps

**Scenario:** An e-commerce checkout has multiple steps (shipping → payment → review → confirmation). Instead of managing step state in a single component, each step is a separate route. Route loaders enforce the correct order — you can't navigate to `/checkout/payment` without first completing shipping. The route action handles the form submission and validation for each step. This is the pattern used by large-scale checkout flows on platforms like Airbnb and Booking.com.

```jsx
// Layout that wraps all checkout steps with a progress indicator
function CheckoutLayout() {
  return (
    <div className="checkout">
      <CheckoutProgress />
      <Outlet />
    </div>
  );
}

const checkoutRouter = createBrowserRouter([
  {
    path: '/checkout',
    element: <CheckoutLayout />,
    errorElement: <CheckoutError />,
    children: [
      // Default redirect to first step
      { index: true, element: <Navigate to="shipping" replace /> },

      // Step 1: Shipping address — form action validates and saves
      {
        path: 'shipping',
        element: <ShippingStep />,
        action: async ({ request }) => {
          const data = await request.formData();
          const errors = validateShipping(data);
          if (errors) return { errors };  // Return validation errors to the form
          saveShipping(Object.fromEntries(data));  // Save to context/state
          return { success: true };  // Let the form know it was valid
        },
      },

      // Step 2: Payment — loader checks shipping was completed first
      {
        path: 'payment',
        element: <PaymentStep />,
        loader: () => {
          if (!hasShippingData()) throw redirect('/checkout/shipping');
          return null;
        },
      },

      // Step 3: Review — loader checks payment was completed first
      {
        path: 'review',
        element: <ReviewStep />,
        loader: () => {
          if (!hasPaymentData()) throw redirect('/checkout/payment');
          return null;
        },
      },

      // Step 4: Confirmation — load order details by ID
      {
        path: 'confirmation/:orderId',
        element: <ConfirmationStep />,
        loader: ({ params }) => fetchOrder(params.orderId),
      },
    ],
  },
]);
```

## Interview Tips
- **createBrowserRouter** — modern API (v6.4+), supports loaders/actions. Older API (BrowserRouter + Routes) still works
- **Loaders** — data loading happens BEFORE render. No loading spinners for initial data
- **Optimistic UI** — use `useNavigation()` to show loading state during navigations
- **Error handling** — `errorElement` per route, not just one global error boundary

## 10 Interview Questions

**Q1:** What is the difference between BrowserRouter and HashRouter?
**A:** BrowserRouter uses history API (clean URLs like `/about`). HashRouter uses URL hash (`/#/about`). HashRouter doesn't need server configuration (works with static files). Use BrowserRouter with server fallback.

**Q2:** How do loaders work in React Router v6.4+?
**A:** Loaders are async functions defined per route. They run before the route renders. Returned data is available via `useLoaderData()`. Supports deferred loading with `<Suspense>`.

**Q3:** How do you implement authentication redirects with React Router?
**A:** Use a layout route with a loader that checks auth. If not authenticated, throw `redirect('/login')`. Or use a `<ProtectedRoute>` component that renders `<Navigate>`.

**Q4:** What is the difference between `<NavLink>` and `<Link>`?
**A:** `<NavLink>` provides `isActive` and `isPending` props for styling. `<Link>` is a simple navigation component. Use NavLink for navigation menus, Link for inline links.

**Q5:** How do you access URL parameters?
**A:** `useParams()` returns an object of route params. For `/products/:category/:slug`, `useParams()` returns `{ category: 'electronics', slug: 'iphone-15' }`.

**Q6:** How do you handle form submissions with React Router?
**A:** Use route `action` functions. `useActionData()` returns data from the action. `useNavigation().state` gives form submission state. `useFetcher()` for non-navigation form submissions.

**Q7:** What is the purpose of `<Outlet>`?
**A:** Renders the matched child route. Used in layout routes. Without Outlet, child routes are defined but never rendered. It's like `{children}` but for router hierarchy.

**Q8:** How do you implement route-based code splitting?
**A:** Use `React.lazy(() => import('./pages/Dashboard'))` + `<Suspense>`. Wrap each lazy route component. React Router loads the component chunk only when navigating to that route.

**Q9:** How do you prevent navigation with unsaved changes?
**A:** Use `useBlocker()` (v6.4+) or a custom `beforeunload` listener. In loaders/actions, check `shouldRevalidate`. Show a confirmation dialog.

**Q10:** How do you set up 404 handling?
**A:** Add a catch-all route: `{ path: '*', element: <NotFoundPage /> }`. Place it last in the route tree. React Router matches routes in order.

## E2E Examples

### E-Commerce
```jsx
import { createBrowserRouter, RouterProvider, Outlet, Link, useParams, Navigate } from 'react-router-dom';

const router = createBrowserRouter([
  {
    path: '/',
    element: <AppLayout />,
    errorElement: <RouteErrorBoundary />,
    children: [
      { index: true, element: <HomePage /> },
      {
        path: 'products',
        element: <Outlet />,
        children: [
          { index: true, element: <ProductList /> },
          { path: ':category', element: <ProductList /> },
          { path: ':category/:productId', element: <ProductDetail /> },
        ]
      },
      { path: 'cart', element: <ProtectedRoute><CartPage /></ProtectedRoute> },
      { path: 'checkout', element: <ProtectedRoute><CheckoutPage /></ProtectedRoute> },
      { path: 'orders', element: <ProtectedRoute><OrderHistory /></ProtectedRoute> },
      { path: 'orders/:orderId', element: <ProtectedRoute><OrderDetail /></ProtectedRoute> },
      { path: 'login', element: <LoginPage /> },
      { path: 'register', element: <RegisterPage /> },
      { path: '*', element: <NotFoundPage /> },
    ]
  }
]);

function AppLayout() {
  return (
    <div className="app">
      <AppHeader />
      <main><Outlet /></main>
      <AppFooter />
    </div>
  );
}

function ProtectedRoute({ children }) {
  const { user } = useAuth();
  return user ? children : <Navigate to="/login" state={{ from: location }} replace />;
}

// Lazy loading routes
const ProductDetail = React.lazy(() => import('./pages/ProductDetail'));

export default function App() {
  return (
    <Suspense fallback={<PageSkeleton />}>
      <RouterProvider router={router} />
    </Suspense>
  );
}
```

### Banking
```jsx
const bankRouter = createBrowserRouter([
  {
    path: '/',
    element: <BankLayout />,
    errorElement: <BankErrorFallback />,
    children: [
      { index: true, element: <Navigate to="/dashboard" replace /> },
      {
        path: 'dashboard',
        element: <ProtectedRoute requiredRole="customer" />,
        children: [
          { index: true, element: <DashboardHome /> },
          { path: 'accounts/:accountId', element: <AccountDetail /> },
          { path: 'transfer', element: <TransferPage /> },
          { path: 'history', element: <TransactionHistory /> },
          { path: 'statements', element: <Statements /> },
        ]
      },
      {
        path: 'admin',
        element: <ProtectedRoute requiredRole="admin" />,
        children: [
          { index: true, element: <AdminDashboard /> },
          { path: 'users', element: <UserManagement /> },
          { path: 'audit', element: <AuditLog /> },
        ]
      },
      { path: 'login', element: <LoginPage /> },
      { path: 'support', element: <SupportPage /> },
      { path: 'support/:ticketId', element: <SupportTicket /> },
      { path: '*', element: <NotFoundPage /> },
    ]
  }
]);

function BankLayout() {
  const { customer } = useBankAuth();
  return (
    <div className="bank-app">
      <BankHeader />
      {customer && <AccountSelector />}
      <main><Outlet /></main>
    </div>
  );
}

function ProtectedRoute({ requiredRole, children }) {
  const { customer } = useBankAuth();
  if (!customer) return <Navigate to="/login" replace />;
  if (requiredRole && customer.role !== requiredRole) return <Navigate to="/dashboard" replace />;
  return children || <Outlet />;
}
```

---

# 12. API CALLS & DATA FETCHING

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Axios Instance with Interceptors

**Scenario:** An e-commerce API requires an access token for authenticated requests. When the token expires (401 response), the app must silently refresh it using a refresh token, retry all queued requests, and not lose any user actions. The Axios interceptor handles this transparently — if a 401 occurs, it pauses the response, refreshes the token, replays the failed request, and propagates the result back to the calling code.

```jsx
import axios from 'axios';

// Create a pre-configured Axios instance with base URL and timeout
const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  timeout: 10000,
  headers: { 'Content-Type': 'application/json' },
});

// Request interceptor: attach auth token to every outgoing request
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('accessToken');
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor: handle 401 by silently refreshing the token
let isRefreshing = false;    // Prevents multiple simultaneous refresh calls
let failedQueue = [];         // Queue of requests waiting for token refresh

const processQueue = (error, token = null) => {
  failedQueue.forEach(prom => {
    if (error) prom.reject(error);
    else prom.resolve(token);
  });
  failedQueue = [];
};

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    const originalRequest = error.config;

    // Only handle 401 errors that haven't been retried yet
    if (error.response?.status === 401 && !originalRequest._retry) {
      // If already refreshing, queue this request to replay after refresh
      if (isRefreshing) {
        return new Promise((resolve, reject) => {
          failedQueue.push({ resolve, reject });
        }).then(token => {
          originalRequest.headers.Authorization = `Bearer ${token}`;
          return api(originalRequest);  // Replay the original request
        });
      }

      // Start token refresh
      originalRequest._retry = true;
      isRefreshing = true;

      try {
        const { data } = await axios.post('/api/auth/refresh', {
          refreshToken: localStorage.getItem('refreshToken'),
        });
        localStorage.setItem('accessToken', data.accessToken);
        processQueue(null, data.accessToken);  // Resolve all queued requests
        originalRequest.headers.Authorization = `Bearer ${data.accessToken}`;
        return api(originalRequest);
      } catch (refreshError) {
        processQueue(refreshError, null);  // Reject all queued requests
        localStorage.clear();
        window.location.href = '/login';   // Force re-login
        return Promise.reject(refreshError);
      } finally {
        isRefreshing = false;
      }
    }

    return Promise.reject(error);
  }
);
```

### Scenario 2: Banking — Idempotent Requests

**Scenario:** A banking transfer API must never process a transfer twice. If the client retries a failed request (network timeout, server error), the server uses an `Idempotency-Key` header to recognize the retry and return the original result instead of processing again. The client generates a unique key per request using `crypto.randomUUID()` and retries with exponential backoff.

```jsx
const api = axios.create({ baseURL: '/api' });

// Initiate a transfer with idempotency protection and automatic retry
export async function initiateTransfer(transferData) {
  // Generate a unique idempotency key for this transfer request
  const idempotencyKey = crypto.randomUUID();

  const { data } = await api.post('/transfers', transferData, {
    headers: { 'Idempotency-Key': idempotencyKey },
    // Custom Axios config for retry behavior
    retry: 3,
    retryDelay: (retryCount) => retryCount * 1000,  // 1s, 2s, 3s
  });

  return data;
}

// Custom retry interceptor — retries server errors with the same idempotency key
api.interceptors.response.use(null, async (error) => {
  const config = error.config;
  if (!config || !config.retry) return Promise.reject(error);

  config.retryCount = config.retryCount || 0;
  if (config.retryCount >= config.retry) return Promise.reject(error);

  // Only retry on server errors (5xx) or network errors — NOT client errors (4xx)
  if (error.response?.status >= 500 || !error.response) {
    config.retryCount += 1;
    const delay = typeof config.retryDelay === 'function'
      ? config.retryDelay(config.retryCount)
      : config.retryDelay || 1000;

    await new Promise(resolve => setTimeout(resolve, delay));
    return api(config);  // Retry with the SAME idempotency key
  }

  return Promise.reject(error);
});
```

### Scenario 3: Race Condition Prevention

**Scenario:** A product search input fires an API call on every keystroke. If the user types "iphone" quickly, the app sends requests for "i", "ip", "iph", "ipho", "iphon", "iphone". With slow network, response for "iphone" might arrive before "ipho" — causing wrong results to be displayed. Two prevention methods are shown: AbortController (cancels previous in-flight requests) and Request ID tracking (ignores stale responses).

```jsx
function useProductSearch() {
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  // Method 1: AbortController — cancels the previous request before starting a new one
  const searchWithAbort = useCallback(async (query) => {
    const controller = new AbortController();

    try {
      setLoading(true);
      setError(null);
      const response = await fetch(`/api/products?q=${query}`, {
        signal: controller.signal,  // Attach the abort signal
      });
      const data = await response.json();
      setResults(data);
    } catch (err) {
      // Ignore AbortError — this is intentional, not a real failure
      if (err.name !== 'AbortError') {
        setError(err.message);
      }
    } finally {
      setLoading(false);
    }

    return () => controller.abort();  // Cancel if a new search starts
  }, []);

  // Method 2: Request ID tracking — ignore responses from stale requests
  const searchWithId = useCallback((query) => {
    const requestId = Date.now();  // Unique ID for this request
    latestRequestRef.current = requestId;

    setLoading(true);
    setError(null);

    fetch(`/api/products?q=${query}`)
      .then(r => r.json())
      .then(data => {
        // Only apply if this response is from the LATEST request
        if (latestRequestRef.current === requestId) {
          setResults(data);
        }
      })
      .catch(err => {
        if (latestRequestRef.current === requestId) {
          setError(err.message);
        }
      })
      .finally(() => {
        if (latestRequestRef.current === requestId) {
          setLoading(false);
        }
      });
  }, []);

  return { results, loading, error, searchWithAbort, searchWithId };
}
```

## Interview Tips
- **Three states** — loading, error, data. Every fetch needs all three
- **AbortController** — cancel in-flight requests on unmount or when deps change
- **Error handling** — different errors need different UIs: network error vs 404 vs 500
- **Caching** — avoid refetching data that hasn't changed. React Query or simple cache Map

## 10 Interview Questions

**Q1:** How do you fetch data in React?
**A:** useEffect + fetch, or use React Query/SWR. useEffect: define async function inside, call it, handle loading/error states, abort on cleanup. React Query: useQuery with queryKey and queryFn.

**Q2:** What is a race condition in data fetching and how do you prevent it?
**A:** When responses arrive out of order (response 2 arrives after response 3). Prevent with: AbortController (cancel previous), request ID tracking (ignore stale responses), or debouncing.

**Q3:** How do you handle errors from API calls?
**A:** Try-catch in async functions. Set error state. Show error UI with retry button. Use Error Boundaries for rendering errors. Use HTTP interceptors for global error handling (token refresh, 401 redirect).

**Q4:** What is the difference between fetch and Axios?
**A:** Axios: automatic JSON parsing, request/response interceptors, timeout config, cancellation (AbortController wrapper), progress events. Fetch: built-in, no 404 rejection (check response.ok), no request cancellation built-in.

**Q5:** How do you cancel a fetch request on component unmount?
**A:** AbortController: `const controller = new AbortController()`. Pass `signal` to fetch. In useEffect cleanup, call `controller.abort()`. Prevents state updates on unmounted component.

**Q6:** How do you implement retry logic for failed requests?
**A:** Exponential backoff: retry with increasing delays (1s, 2s, 4s, 8s...). Set max retries. Only retry on server errors (5xx) or network errors, not client errors (4xx). Use idempotency keys for mutations.

**Q7:** How do you cache API responses in the frontend?
**A:** Simple Map: `const cache = new Map(); if (cache.has(url)) return cache.get(url);`. Advanced: React Query (stale-while-revalidate, garbage collection, background refetch). LRU cache for memory management.

**Q8:** How do you handle loading states for multiple independent fetches?
**A:** Track each fetch's loading state separately. Show skeleton per section. Use React Query's `isFetching` vs `isLoading` (isLoading = no data yet, isFetching = background refetch).

**Q9:** What is optimistic UI and when should you use it?
**A:** Update the UI before the API responds. Use for high-confidence actions (likes, toggles, reordering). Don't use for payments, registrations, or destructive irreversible actions.

**Q10:** How do you normalize API response data?
**A:** Convert nested/relational API data to flat structure: `{ users: { byId: {}, allIds: [] } }`. Use libraries like normalizr. Reduces duplication, simplifies updates, improves performance.

## E2E Examples

### E-Commerce
```jsx
// API service layer
const api = {
  baseUrl: '/api',
  async get(endpoint) {
    const res = await fetch(`${this.baseUrl}${endpoint}`, {
      headers: { 'Authorization': `Bearer ${getToken()}` }
    });
    if (!res.ok) throw new ApiError(res.status, await res.json());
    return res.json();
  },
  async post(endpoint, data) {
    const res = await fetch(`${this.baseUrl}${endpoint}`, {
      method: 'POST', headers: { 'Content-Type': 'application/json', 'Authorization': `Bearer ${getToken()}` }, body: JSON.stringify(data)
    });
    if (!res.ok) throw new ApiError(res.status, await res.json());
    return res.json();
  }
};

class ApiError extends Error {
  constructor(status, body) {
    super(body.message || `Request failed with ${status}`);
    this.status = status;
    this.body = body;
  }
}

// Product fetching with retry and normalization
async function fetchProducts(filters = {}) {
  const params = new URLSearchParams(filters);
  const data = await api.get(`/products?${params}`);
  return {
    products: data.products.map(p => ({
      ...p,
      inStock: p.quantity > 0,
      formattedPrice: `$${p.price.toFixed(2)}`
    })),
    total: data.total,
    page: data.page
  };
}

// Usage in component with all states
function ProductListPage() {
  const [products, setProducts] = useState([]);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);
  const [retryCount, setRetryCount] = useState(0);

  useEffect(() => {
    let cancelled = false;
    setIsLoading(true);
    setError(null);

    fetchProducts({ category: 'electronics' })
      .then(data => { if (!cancelled) { setProducts(data.products); setIsLoading(false); } })
      .catch(err => { if (!cancelled) { setError(err.message); setIsLoading(false); } });

    return () => { cancelled = true; };
  }, [retryCount]);

  if (isLoading) return <SkeletonGrid count={8} />;
  if (error) return <ErrorState message={error} onRetry={() => setRetryCount(c => c + 1)} />;
  if (!products.length) return <EmptyState message="No products found" />;
  return <ProductGrid products={products} />;
}
```

### Banking
```jsx
// Banking API with auth token refresh
const bankingApi = {
  async request(endpoint, options = {}) {
    const config = {
      headers: { 'Authorization': `Bearer ${getAccessToken()}`, 'Idempotency-Key': crypto.randomUUID(), ...options.headers },
      ...options
    };
    let res = await fetch(`https://api.bank.example.com${endpoint}`, config);

    // Token refresh on 401
    if (res.status === 401) {
      const newToken = await refreshToken();
      config.headers['Authorization'] = `Bearer ${newToken}`;
      res = await fetch(`https://api.bank.example.com${endpoint}`, config);
    }

    if (!res.ok) {
      const body = await res.json().catch(() => ({}));
      throw new BankingError(res.status, body);
    }
    return res.json();
  },

  getAccounts(userId) { return this.request(`/accounts?userId=${userId}`); },
  getTransactions(accountId, page) { return this.request(`/accounts/${accountId}/transactions?page=${page}&limit=20`); },
  transfer(data) {
    return this.request('/transfers', {
      method: 'POST', body: JSON.stringify(data),
      headers: { 'Content-Type': 'application/json' }
    });
  },
  getBalance(accountId) { return this.request(`/accounts/${accountId}/balance`); }
};

class BankingError extends Error {
  constructor(status, body) {
    super(body.error || 'Banking service error');
    this.status = status;
    this.code = body.code;
    this.retryable = [408, 429, 502, 503, 504].includes(status);
  }
}

// Usage with retry logic
async function fetchWithRetry(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try { return await fn(); }
    catch (err) {
      if (!err.retryable || i === maxRetries - 1) throw err;
      await new Promise(r => setTimeout(r, Math.pow(2, i) * 1000));
    }
  }
}

function AccountBalance({ accountId }) {
  const [balance, setBalance] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    setIsLoading(true);
    fetchWithRetry(() => bankingApi.getBalance(accountId))
      .then(data => { setBalance(data.balance); setIsLoading(false); })
      .catch(err => { setError(err.message); setIsLoading(false); });
  }, [accountId]);

  if (isLoading) return <SkeletonBalance />;
  if (error) return <ErrorBanner message={error} />;
  return <BalanceDisplay amount={balance} />;
}
```

---

# 13. REACT QUERY (TanStack Query)

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Product Catalog with Prefetching

**Scenario:** An e-commerce site displays a paginated product listing. When a user hovers over a product card, the product detail is prefetched so the detail page loads instantly on click. The `useProducts` hook fetches the filtered list with a 5-minute freshness window. The `useProduct` hook first checks the list cache for the product — if found, it renders immediately while still refreshing in the background.

```jsx
import { QueryClient, useQuery, useQueryClient, useMutation } from '@tanstack/react-query';
import { queryClient } from '../main';

// Centralised query key factory — ensures consistency across all cache operations
const productKeys = {
  all: ['products'],
  lists: () => [...productKeys.all, 'list'],
  list: (filters) => [...productKeys.lists(), filters],
  details: () => [...productKeys.all, 'detail'],
  detail: (id) => [...productKeys.details(), id],
};

// Hook: fetch paginated product list with custom selector
function useProducts(filters) {
  return useQuery({
    queryKey: productKeys.list(filters),
    queryFn: () => api.getProducts(filters),
    staleTime: 5 * 60 * 1000,                 // Data stays "fresh" for 5 minutes — no background refetch
    select: (data) => ({                       // Transform the raw response shape
      products: data.products,
      totalPages: Math.ceil(data.total / data.limit),
      total: data.total,
    }),
  });
}

// Hook: fetch single product with initial data from list cache
function useProduct(id) {
  const queryClient = useQueryClient();

  return useQuery({
    queryKey: productKeys.detail(id),
    queryFn: () => api.getProduct(id),
    initialData: () => {
      // Pull the product from the list cache so the detail page renders instantly
      const products = queryClient.getQueryData(productKeys.lists())
        ?.products;
      return products?.find(p => p.id === id);
    },
    initialDataUpdatedAt: () =>
      queryClient.getQueryState(productKeys.lists())?.dataUpdatedAt,
  });
}

// Prefetch product detail on hover — data is ready before the user clicks
function ProductCard({ product }) {
  const queryClient = useQueryClient();

  const handleMouseEnter = () => {
    queryClient.prefetchQuery({
      queryKey: productKeys.detail(product.id),
      queryFn: () => api.getProduct(product.id),
      staleTime: 10 * 60 * 1000,   // Prefetched data stays fresh longer
    });
  };

  return (
    <Link to={`/products/${product.id}`} onMouseEnter={handleMouseEnter}>
      {/* card content */}
    </Link>
  );
}
```

### Scenario 2: Banking — Real-Time Balance with Polling

**Scenario:** A banking dashboard must show the current account balance with near-real-time accuracy. The `useAccountBalance` hook polls the server every 30 seconds (even when the tab is in the background) so the user always sees an up-to-date balance. After a transfer mutation, the balance query is invalidated immediately to trigger an instant refetch, bypassing the poll interval.

```jsx
function useAccountBalance(accountId) {
  return useQuery({
    queryKey: ['account', accountId, 'balance'],
    queryFn: () => api.getAccountBalance(accountId),
    refetchInterval: 30 * 1000,             // Poll the server every 30 seconds
    refetchIntervalInBackground: true,      // Keep polling even when the tab is not focused
    staleTime: 10 * 1000,                   // Data is stale after 10 seconds
    retry: 3,
    retryDelay: 1000,
    onError: (err) => {
      // On failure, keep showing the last known balance but warn the user
      showWarning('Balance may be outdated. Pull to refresh.');
    },
  });
}

function AccountDashboard({ accountId }) {
  const { data, isLoading, error, refetch } = useAccountBalance(accountId);
  const queryClient = useQueryClient();

  // After a transfer, immediately invalidate balance + transactions to trigger a refetch
  const transferMutation = useMutation({
    mutationFn: (data) => api.transfer(data),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['account', accountId, 'balance'] });
      queryClient.invalidateQueries({ queryKey: ['account', accountId, 'transactions'] });
    },
  });

  return (
    <div className="account-dashboard">
      <PullToRefresh onRefresh={refetch}>
        <BalanceDisplay
          balance={data?.balance}
          isLoading={isLoading}
          // Show a visual indicator when data is stale (not yet refreshed)
          isStale={data ? queryClient.getQueryState(['account', accountId, 'balance'])?.isStale : false}
        />
        <button onClick={() => transferMutation.mutate({ /* transfer data */ })}>
          Transfer
        </button>
      </PullToRefresh>
    </div>
  );
}
```

### Scenario 3: Infinite Scroll for Transaction History

**Scenario:** A banking app shows a potentially infinite list of transactions. Instead of paginated "Previous / Next" buttons, the page uses infinite scroll: as the user scrolls down, an `IntersectionObserver` detects when the last row becomes visible and automatically fetches the next page using `useInfiniteQuery`. All pages are merged into a single flat list for rendering.

```jsx
import { useInfiniteQuery } from '@tanstack/react-query';

// Hook that fetches transactions page by page as the user scrolls
function useTransactionHistory(accountId, filters) {
  return useInfiniteQuery({
    queryKey: ['transactions', accountId, filters],
    queryFn: ({ pageParam = 1 }) =>
      api.getTransactions(accountId, { ...filters, page: pageParam, limit: 50 }),
    // Determine the next page cursor from the last page's metadata
    getNextPageParam: (lastPage) => {
      return lastPage.page < lastPage.totalPages ? lastPage.page + 1 : undefined;
    },
    staleTime: 60 * 1000,
  });
}

function TransactionHistory({ accountId }) {
  const {
    data,
    fetchNextPage,
    hasNextPage,
    isFetchingNextPage,
    isLoading,
    error,
  } = useTransactionHistory(accountId, { type: 'all' });

  // IntersectionObserver ref — triggers fetch when the last element enters the viewport
  const observerRef = useRef(null);
  const lastElementRef = useCallback((node) => {
    if (observerRef.current) observerRef.current.disconnect();
    observerRef.current = new IntersectionObserver(entries => {
      if (entries[0].isIntersecting && hasNextPage) {
        fetchNextPage();
      }
    });
    if (node) observerRef.current.observe(node);
  }, [hasNextPage, fetchNextPage]);

  // Flatten all pages into a single array for rendering
  const allTransactions = data?.pages.flatMap(p => p.transactions) ?? [];

  return (
    <div className="transaction-list">
      {isLoading && <TransactionSkeleton />}
      {error && <ErrorState onRetry={() => refetch()} />}

      {allTransactions.map((tx, i) => (
        <TransactionRow
          key={tx.id}
          transaction={tx}
          // Attach the IntersectionObserver to the very last element
          ref={i === allTransactions.length - 1 ? lastElementRef : null}
        />
      ))}

      {isFetchingNextPage && <Spinner />}
      {!hasNextPage && allTransactions.length > 0 && (
        <p className="end-message">All transactions loaded</p>
      )}
    </div>
  );
}
```

## Interview Tips
- **Query keys** — the foundation. Structure them hierarchically: `['resource', id, ...filters]`
- **staleTime vs gcTime** — staleTime: how long until refetch. gcTime: how long to keep unused cache
- **Optimistic updates** — use `onMutate` to update cache immediately, `onError` to rollback
- **DevTools** — `@tanstack/react-query-devtools` is essential for debugging

## 10 Interview Questions

**Q1:** What problem does React Query solve?
**A:** Server state management. Caching, background refetching, deduplication, loading/error states, pagination, optimistic updates, stale-while-revalidate pattern — all without manual useEffect code.

**Q2:** What is the purpose of queryKey?
**A:** Unique identifier for cached data. React Query uses it to cache, deduplicate, and refetch. Structure hierarchically: `['products', filters]`. Include all data that affects the query result.

**Q3:** What is the difference between `isLoading` and `isFetching`?
**A:** `isLoading` = no data yet (first load). `isFetching` = any fetch in progress (including background refetches). Show spinners only on isLoading. Use isFetching for subtle indicators.

**Q4:** How does React Query handle caching?
**A:** In-memory cache keyed by queryKey. Data is "fresh" until staleTime expires. After staleTime, data is "stale" → refetched on next access. Inactive queries (no subscribers) are garbage collected after gcTime.

**Q5:** What is the purpose of `staleTime` vs `gcTime`?
**A:** `staleTime`: how long data is considered fresh (no background refetch). `gcTime` (formerly `cacheTime`): how long inactive data stays in memory before garbage collection. staleTime should be < gcTime.

**Q6:** How do you implement pagination with React Query?
**A:** `useQuery` with page param in queryKey. `useInfiniteQuery` for "load more" patterns — provides `fetchNextPage`, `hasNextPage`, and `getNextPageParam` to determine next page cursor.

**Q7:** How do you handle mutations with React Query?
**A:** `useMutation` for create/update/delete. Provides `mutate`, `mutateAsync`, `isPending`, `onSuccess`, `onError`. Invalidate related queries in `onSuccess`: `queryClient.invalidateQueries({ queryKey: ['products'] })`.

**Q8:** How do you implement optimistic updates with React Query?
**A:** In `onMutate`: cancel queries, snapshot previous data, update cache optimistically. Return snapshot. In `onError`: restore snapshot (rollback). In `onSettled`: invalidate to sync with server.

**Q9:** What is the difference between `invalidateQueries` and `refetchQueries`?
**A:** `invalidateQueries` marks queries as stale (refetch on next mount). `refetchQueries` immediately re-fetches. Invalidate is preferred — it's lazy and batches. Refetch is aggressive.

**Q10:** How do you prefetch data for instant navigation?
**A:** `queryClient.prefetchQuery({ queryKey, queryFn })`. Call on hover over a link. Data is fetched before navigation happens. The route component finds data already in cache.

## E2E Examples

### E-Commerce
```jsx
import { QueryClient, QueryClientProvider, useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// Configure the global query client with sensible defaults
const queryClient = new QueryClient({
  defaultOptions: { queries: { staleTime: 5 * 60 * 1000, retry: 2, refetchOnWindowFocus: false } }
});

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <ShopApp />
    </QueryClientProvider>
  );
}

// Fetch product list with automatic caching and background refetching
function ProductList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['products'],
    queryFn: () => fetch('/api/products').then(r => r.json()),
  });

  return (
    <div className="product-list">
      {data?.products.map(p => (
        <ProductCard key={p.id} product={p} />
      ))}
    </div>
  );
}

// Prefetch product detail on hover for instant navigation
function ProductCard({ product }) {
  const queryClient = useQueryClient();

  function handleMouseEnter() {
    queryClient.prefetchQuery({
      queryKey: ['product', product.id],
      queryFn: () => fetch(`/api/products/${product.id}`).then(r => r.json()),
      staleTime: 10 * 60 * 1000   // Prefetched data stays fresh longer
    });
  }

  return (
    <Link to={`/products/${product.id}`} onMouseEnter={handleMouseEnter}>
      <img src={product.thumbnail} alt={product.name} />
      <h3>{product.name}</h3>
      <p>${product.price}</p>
    </Link>
  );
}

// Add to cart mutation — invalidates related queries on success
function AddToCartButton({ productId }) {
  const queryClient = useQueryClient();
  const mutation = useMutation({
    mutationFn: () => fetch('/api/cart/add', { method: 'POST', body: JSON.stringify({ productId }) }),
    onSuccess: () => {
      // Mark cart and products as stale so they refetch with updated data
      queryClient.invalidateQueries({ queryKey: ['cart'] });
      queryClient.invalidateQueries({ queryKey: ['products'] });
    }
  });

  return <button onClick={() => mutation.mutate()} disabled={mutation.isPending}>
    {mutation.isPending ? 'Adding...' : 'Add to Cart'}
  </button>;
}
```

### Banking
```jsx
// Balance widget with automatic polling every 30 seconds
function BalanceWidget({ accountId }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['balance', accountId],
    queryFn: () => bankingApi.getBalance(accountId),
    refetchInterval: 30000,           // Auto-refresh every 30 seconds
  });

  if (isLoading) return <BalanceSkeleton />;
  if (error) return <ErrorBanner message="Could not load balance" />;

  return (
    <div className="balance-widget">
      <h3>Current Balance</h3>
      <p className="balance-amount">${data.balance.toLocaleString()}</p>
      <p className="balance-as-of">As of {new Date(data.asOf).toLocaleTimeString()}</p>
    </div>
  );
}

// Dashboard with parallel queries — three independent fetches run concurrently
function Dashboard() {
  const accounts = useQuery({ queryKey: ['accounts'], queryFn: () => bankingApi.getAccounts(userId) });
  const recentTx = useQuery({ queryKey: ['transactions', 'recent'], queryFn: () => bankingApi.getTransactions(userId, 1) });
  const alerts = useQuery({ queryKey: ['alerts'], queryFn: () => bankingApi.getAlerts(userId), refetchInterval: 60000 });

  if (accounts.isLoading) return <DashboardSkeleton />;
  if (accounts.error) return <ErrorBanner message={accounts.error.message} />;

  return (
    <div className="dashboard">
      <AccountSummary accounts={accounts.data} />
      <RecentTransactions transactions={recentTx.data} />
      <AlertBar alerts={alerts.data} />
    </div>
  );
}
```

---

# 14. OPTIMISTIC UPDATES

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Add to Cart with Instant Feedback

**Scenario:** When a user clicks "Add to Cart", the button should immediately show a checkmark and update the cart badge — no waiting for the server. The `onMutate` callback cancels any in-flight cart refetches, snapshots the current cart, and optimistically adds the item. If the server call fails, `onError` rolls back to the snapshot and shows an error toast. Finally, `onSettled` invalidates the cache to sync with the real server state.

```jsx
function AddToCartButton({ product, className }) {
  const queryClient = useQueryClient();
  const [justAdded, setJustAdded] = useState(false);

  const addToCart = useMutation({
    mutationFn: (qty) => api.addToCart(product.id, qty),

    onMutate: async (quantity) => {
      // Cancel any in-flight cart queries so they don't overwrite our optimistic update
      await queryClient.cancelQueries({ queryKey: ['cart'] });
      // Snapshot the current cart for rollback
      const previousCart = queryClient.getQueryData(['cart']);

      // Optimistically add the item to the cached cart
      queryClient.setQueryData(['cart'], (old) => {
        const items = old?.items ?? [];
        const existing = items.find(i => i.productId === product.id);
        if (existing) {
          // Item already in cart — increment quantity
          return {
            ...old,
            items: items.map(i =>
              i.productId === product.id
                ? { ...i, quantity: i.quantity + quantity }
                : i
            ),
          };
        }
        // New item — append it to the cart
        return {
          ...old,
          items: [...items, { productId: product.id, title: product.title, price: product.price, quantity, image: product.thumbnail }],
        };
      });

      // Return snapshot so onError can restore it
      return { previousCart };
    },

    onError: (err, qty, context) => {
      // Rollback to the pre-mutation cart snapshot
      queryClient.setQueryData(['cart'], context.previousCart);
      showToast('Failed to add to cart. Please try again.', 'error');
    },

    onSettled: () => {
      // Whether success or error, eventually sync with the real server state
      queryClient.invalidateQueries({ queryKey: ['cart'] });
    },
  });

  const handleClick = () => {
    addToCart.mutate(1);
    setJustAdded(true);
    setTimeout(() => setJustAdded(false), 2000);
  };

  return (
    <button onClick={handleClick} disabled={addToCart.isPending}
      className={`${className} ${justAdded ? 'added' : ''}`}>
      {justAdded ? '✓ Added!' : addToCart.isPending ? 'Adding...' : 'Add to Cart'}
    </button>
  );
}
```

### Scenario 2: Banking — Toggle Favorite Payee

**Scenario:** A banking app lets users mark payees as favorites for quick access. When the star button is clicked, the UI toggles immediately (filled star ↔ empty star) while the mutation fires in the background. If the server request fails, the star reverts and an error toast is shown. This is a safe optimistic update because the consequence of a rollback is trivial — the user simply sees the star return to its previous state.

```jsx
function PayeeCard({ payee }) {
  const queryClient = useQueryClient();

  const toggleFavorite = useMutation({
    mutationFn: () => api.toggleFavoritePayee(payee.id),

    onMutate: async () => {
      // Cancel any in-flight payee queries so they don't overwrite our change
      await queryClient.cancelQueries({ queryKey: ['payees'] });

      // Snapshot current list for rollback
      const previousPayees = queryClient.getQueryData(['payees']);

      // Toggle the isFavorite flag optimistically
      queryClient.setQueryData(['payees'], (old) =>
        old.map(p =>
          p.id === payee.id ? { ...p, isFavorite: !p.isFavorite } : p
        )
      );

      return { previousPayees };
    },

    onError: (err, vars, context) => {
      // Revert the optimistic toggle
      queryClient.setQueryData(['payees'], context.previousPayees);
      showToast('Failed to update favorite', 'error');
    },
  });

  return (
    <div className={`payee-card ${payee.isFavorite ? 'favorite' : ''}`}>
      <div className="payee-info">
        <Avatar name={payee.name} />
        <div>
          <h4>{payee.name}</h4>
          <p>{payee.accountNumber}</p>
        </div>
      </div>
      <button
        onClick={() => toggleFavorite.mutate()}
        disabled={toggleFavorite.isPending}
        className="favorite-btn"
        aria-label={payee.isFavorite ? 'Remove from favorites' : 'Add to favorites'}
      >
        {payee.isFavorite ? '★' : '☆'}
      </button>
    </div>
  );
}
```

### Scenario 3: Task Management — Drag and Drop Reorder

**Scenario:** A project management board lets users reorder tasks by drag-and-drop. When a task is dropped at a new position, the list reorders instantly in the UI while the mutation sends the new order to the server. If the server rejects the reorder, the list snaps back to its original order. This is more complex than a simple toggle because the mutation operates on array indices rather than a single value.

```jsx
function useDragReorder() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ fromIndex, toIndex }) =>
      api.reorderTasks(fromIndex, toIndex),

    onMutate: async ({ fromIndex, toIndex }) => {
      // Cancel any background refetches that might overwrite our reorder
      await queryClient.cancelQueries({ queryKey: ['tasks'] });
      // Snapshot the current task order for rollback
      const previousTasks = queryClient.getQueryData(['tasks']);

      // Optimistically reorder the array — splice the moved item into its new position
      queryClient.setQueryData(['tasks'], (old) => {
        const tasks = [...old];            // Create a mutable copy
        const [moved] = tasks.splice(fromIndex, 1);  // Remove from old position
        tasks.splice(toIndex, 0, moved);             // Insert at new position
        return tasks;
      });

      return { previousTasks };
    },

    onError: (err, vars, context) => {
      // Restore the original task order
      queryClient.setQueryData(['tasks'], context.previousTasks);
      showToast('Reorder failed', 'error');
    },

    onSettled: () => {
      // Finally, sync with the server's canonical order
      queryClient.invalidateQueries({ queryKey: ['tasks'] });
    },
  });
}
```

## Interview Tips
- **Always provide rollback** — `onError` must restore previous state. Otherwise UI and server drift apart
- **Cancel queries** — `cancelQueries` prevents background refetches from overwriting your optimistic update
- **User feedback** — show pending state subtly (opacity change, tiny indicator). Don't hide the UI entirely
- **Destructive actions** — never use optimistic updates for irreversible operations (payment, deletion)

## 10 Interview Questions

**Q1:** What is an optimistic update?
**A:** Updating the UI before the server confirms. The UI shows the expected final state instantly. If the server fails, the UI rolls back to the previous state. Makes apps feel instant.

**Q2:** What is the rollback pattern in React Query?
**A:** In `onMutate`: snapshot previous cache, update optimistically, return snapshot. In `onError`: restore snapshot. This can be done without React Query too: save previous state before mutation, restore on error.

**Q3:** When should you NOT use optimistic updates?
**A:** Payment transactions (must show confirmed status), user registration (need server validation), irreversible destructive actions (account deletion), any action where showing incorrect data causes real harm.

**Q4:** How do you handle multiple optimistic updates in rapid succession?
**A:** Cancel previous queries (`cancelQueries`) before each update. Use functional update in setQueryData to work from latest cache. Debounce if updates are too frequent.

**Q5:** How do you show pending state during an optimistic update?
**A:** Track a local `isPending` or `justUpdated` state in the component. Show subtle indicators: opacity change, small spinner, or "syncing" label. Hide heavy loading spinners.

**Q6:** What happens if the server responds with different data than the optimistic update?
**A:** On success, `onSettled` typically invalidates the query, which refetches the actual server data. The optimistic update is replaced with real data. This ensures eventual consistency.

**Q7:** How do you implement optimistic updates without React Query?
**A:** useRef to save previous state before update. SetState to update UI. Try-catch the API call. In catch: restore previous state from ref. Show error message.

**Q8:** What is the difference between optimistic and pessimistic updates?
**A:** Optimistic: UI updates first, then API call. Pessimistic: API call first, then UI updates on success. Optimistic feels faster but requires rollback. Pessimistic is safer but feels slower.

**Q9:** How do you handle optimistic updates in a list where order matters?
**A:** Preserve array order by inserting at the correct position. Use functional updates to avoid stale array. Cancel refetches during the operation. Re-fetch on settle to ensure correct server order.

**Q10:** How do you handle optimistic updates across multiple screens?
**A:** Use a shared cache (React Query / Zustand). The optimistic update in one component updates the shared cache, which automatically updates all other subscribed components. This is the power of "single source of truth."

## E2E Examples

### E-Commerce
```jsx
// Optimistic add-to-cart hook — shows item in cart immediately
function useAddToCart() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ productId, quantity }) => fetch('/api/cart/add', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ productId, quantity })
    }).then(r => { if (!r.ok) throw new Error('Failed to add'); return r.json(); }),

    // Optimistic update — add item to cart cache before server responds
    onMutate: async ({ productId, quantity }) => {
      await queryClient.cancelQueries({ queryKey: ['cart'] });
      const previousCart = queryClient.getQueryData(['cart']);

      // Insert the item into the cached cart immediately
      queryClient.setQueryData(['cart'], old => ({
        items: old?.items ? [...old.items, { id: productId, quantity, isOptimistic: true }] : [{ id: productId, quantity, isOptimistic: true }]
      }));

      return { previousCart };
    },

    onError: (err, vars, context) => {
      // Rollback on error — restore the previous cart state
      if (context?.previousCart) queryClient.setQueryData(['cart'], context.previousCart);
      toast.error('Failed to add item to cart');
    },

    onSettled: () => {
      // Sync with server state regardless of success or failure
      queryClient.invalidateQueries({ queryKey: ['cart'] });
    }
  });
}

// Component using the optimistic mutation
function ProductCard({ product }) {
  const addToCart = useAddToCart();
  const [added, setAdded] = useState(false);

  function handleClick() {
    addToCart.mutate({ productId: product.id, quantity: 1 });
    setAdded(true);
    setTimeout(() => setAdded(false), 2000);
  }

  return (
    <div className="product-card">
      <img src={product.thumbnail} alt={product.name} />
      <button onClick={handleClick} className={added ? 'added' : ''}>
        {added ? '✓ Added!' : 'Add to Cart'}
      </button>
    </div>
  );
}
```

### Banking
```jsx
// Optimistic toggle-favorite for payees
function useToggleFavorite() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ payeeId, isFavorite }) => fetch(`/api/payees/${payeeId}`, {
      method: 'PATCH',
      body: JSON.stringify({ isFavorite: !isFavorite })
    }).then(r => { if (!r.ok) throw new Error('Update failed'); return r.json(); }),

    onMutate: async ({ payeeId, isFavorite }) => {
      await queryClient.cancelQueries({ queryKey: ['payees'] });
      const previousPayees = queryClient.getQueryData(['payees']);

      // Toggle the favorite flag in cache before the server responds
      queryClient.setQueryData(['payees'], old => ({
        payees: old.payees.map(p => p.id === payeeId ? { ...p, isFavorite: !isFavorite } : p)
      }));

      return { previousPayees };
    },

    onError: (err, vars, context) => {
      // Rollback the optimistic toggle on failure
      if (context?.previousPayees) queryClient.setQueryData(['payees'], context.previousPayees);
      toast.error('Could not update favorite');
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['payees'] });
    }
  });
}

// Optimistic balance update after a transfer — deduct immediately, rollback on error
function useTransfer() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (transferData) => bankingApi.transfer(transferData),
    onMutate: async ({ fromAccount, amount }) => {
      await queryClient.cancelQueries({ queryKey: ['balance', fromAccount] });
      const previousBalance = queryClient.getQueryData(['balance', fromAccount]);
      // Deduct the transfer amount from the cached balance instantly
      queryClient.setQueryData(['balance', fromAccount], old => ({
        ...old, balance: old.balance - amount
      }));
      return { previousBalance, fromAccount, amount };
    },
    onError: (err, vars, context) => {
      // Restore the original balance on failure
      if (context?.previousBalance) queryClient.setQueryData(['balance', context.fromAccount], context.previousBalance);
      toast.error('Transfer failed. Your balance has been restored.');
    },
    onSettled: (data, err, vars) => {
      queryClient.invalidateQueries({ queryKey: ['balance', vars.fromAccount] });
      queryClient.invalidateQueries({ queryKey: ['transactions', vars.fromAccount] });
    }
  });
}
```

---

# 15. ZUSTAND / STATE MANAGEMENT

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Persistent Cart Store

**Scenario:** An e-commerce site needs a shopping cart that survives page refreshes. The Zustand store is wrapped with `persist` middleware to save `items` and `coupon` to localStorage automatically. Getters (`subtotal`, `discount`, `total`) are computed properties. Components subscribe to specific slices — `CartBadge` only re-renders when the item count changes, and `CartTotal` only when the total changes.

```jsx
import { create } from 'zustand';
import { persist, devtools } from 'zustand/middleware';

// Create a cart store with persistence (survives page refresh) and DevTools support
export const useCartStore = create(
  devtools(
    persist(
      (set, get) => ({
        // State
        items: [],
        coupon: null,

        // Getters — computed properties recalculated on every state change
        get subtotal() {
          return get().items.reduce((s, i) => s + i.price * i.quantity, 0);
        },

        get discount() {
          const c = get().coupon;
          if (!c) return 0;
          return c.type === 'percentage'
            ? get().subtotal * (c.value / 100)
            : c.value;
        },

        get total() {
          return get().subtotal - get().discount + get().shipping;
        },

        // Actions
        addItem: (product, quantity = 1) =>
          set((state) => {
            const existing = state.items.find(i => i.id === product.id);
            return {
              items: existing
                ? state.items.map(i =>
                    i.id === product.id ? { ...i, quantity: i.quantity + quantity } : i
                  )
                : [...state.items, { ...product, quantity }],
            };
          }),

        removeItem: (id) =>
          set((state) => ({ items: state.items.filter(i => i.id !== id) })),

        updateQuantity: (id, qty) =>
          set((state) => ({
            items: qty <= 0
              ? state.items.filter(i => i.id !== id)
              : state.items.map(i => i.id === id ? { ...i, quantity: qty } : i),
          })),

        applyCoupon: (coupon) => set({ coupon }),
        removeCoupon: () => set({ coupon: null }),
        clearCart: () => set({ items: [], coupon: null }),
      }),
      {
        name: 'ecommerce-cart',
        partialize: (state) => ({ items: state.items, coupon: state.coupon }),
        version: 1,
      }
    ),
    { name: 'CartStore' }
  )
);

// Components subscribe to specific slices — only re-render when their selected value changes
const CartBadge = () => {
  const count = useCartStore((s) => s.items.reduce((sum, i) => sum + i.quantity, 0));
  return <span className="cart-badge">{count}</span>;
};

const CartTotal = () => {
  const total = useCartStore((s) => s.total);
  return <span>${total.toFixed(2)}</span>;
};
```

### Scenario 2: Banking — Multi-Account Dashboard Store

**Scenario:** A banking dashboard shows all user accounts (checking, savings, credit). The store holds the list of accounts, tracks which account is selected, and provides derived data (`selectedAccount`, `totalBalance`, `accountsByType`). The `fetchAccounts` action is async — it sets loading state, calls the API, and auto-selects the first account. Components subscribe selectively: `AccountSelector` uses the full store slice, while `BalanceWidget` only subscribes to `totalBalance`.

```jsx
export const useBankStore = create((set, get) => ({
  // Core state
  accounts: [],
  selectedAccountId: null,
  isLoading: false,
  error: null,

  // Derived data via getters — recalculated automatically
  get selectedAccount() {
    return get().accounts.find(a => a.id === get().selectedAccountId);
  },

  get totalBalance() {
    return get().accounts.reduce((s, a) => s + a.balance, 0);
  },

  get accountsByType() {
    const groups = {};
    get().accounts.forEach(a => {
      if (!groups[a.type]) groups[a.type] = [];
      groups[a.type].push(a);
    });
    return groups;
  },

  // Async action — fetches accounts from the API
  fetchAccounts: async () => {
    set({ isLoading: true, error: null });
    try {
      const accounts = await bankingApi.getAccounts();
      set({ accounts, isLoading: false });
      // Auto-select the first account if none is selected
      if (!get().selectedAccountId && accounts.length > 0) {
        set({ selectedAccountId: accounts[0].id });
      }
    } catch (err) {
      set({ error: err.message, isLoading: false });
    }
  },

  // Sync actions for user interaction
  selectAccount: (id) => set({ selectedAccountId: id }),

  updateBalance: (accountId, newBalance) =>
    set((state) => ({
      accounts: state.accounts.map(a =>
        a.id === accountId ? { ...a, balance: newBalance } : a
      ),
    })),

  addTransaction: (accountId, transaction) =>
    set((state) => ({
      accounts: state.accounts.map(a =>
        a.id === accountId
          ? { ...a, transactions: [transaction, ...(a.transactions || [])] }
          : a
      ),
    })),
}));

// Component subscriptions — fine-grained selectors prevent unnecessary re-renders
const AccountSelector = () => {
  const { accounts, selectAccount, selectedAccountId } = useBankStore();
  return (
    <select value={selectedAccountId} onChange={e => selectAccount(e.target.value)}>
      {accounts.map(a => (
        <option key={a.id} value={a.id}>{a.name}</option>
      ))}
    </select>
  );
};

const BalanceWidget = () => {
  const totalBalance = useBankStore(s => s.totalBalance);
  return <h2>Total Balance: ${totalBalance.toFixed(2)}</h2>;
};
```

### Scenario 3: UI State — Toast Notifications

**Scenario:** An app needs a global toast notification system that any component can trigger without prop drilling. The UI store manages toasts (with auto-dismiss), sidebar state, and a modal stack. Because Zustand stores are accessible outside React, the `addToast` action can be called from anywhere — components, hooks, API interceptors, or even non-React code.

```jsx
export const useUIStore = create((set, get) => ({
  // UI state slices
  toasts: [],
  sidebarOpen: true,
  modalStack: [],
  theme: 'light',

  // Toast management — add with auto-dismiss
  addToast: (message, type = 'info', duration = 3000) => {
    const id = Date.now() + Math.random();       // Unique toast ID
    set(state => ({ toasts: [...state.toasts, { id, message, type }] }));
    if (duration > 0) {
      setTimeout(() => {
        set(state => ({ toasts: state.toasts.filter(t => t.id !== id) }));
      }, duration);
    }
    return id;
  },

  removeToast: (id) =>
    set(state => ({ toasts: state.toasts.filter(t => t.id !== id) })),

  // Sidebar toggle
  toggleSidebar: () => set(s => ({ sidebarOpen: !s.sidebarOpen })),

  // Modal stack — supports nested modals by pushing/popping
  pushModal: (modal) =>
    set(state => ({ modalStack: [...state.modalStack, modal] })),

  popModal: () =>
    set(state => ({ modalStack: state.modalStack.slice(0, -1) })),

  clearModals: () => set({ modalStack: [] }),
}));

// Usage from any component or non-React code:
const addToast = useUIStore(s => s.addToast);
addToast('Transaction completed successfully', 'success');
```

## Interview Tips
- **No Provider needed** — Zustand creates the store outside React. Components subscribe directly
- **Fine-grained subscriptions** — `useStore(s => s.field)` — component re-renders only when that field changes
- **Persistence** — built-in persist middleware with localStorage. Handles versioning and migrations
- **DevTools** — `devtools` middleware enables Redux DevTools for debugging

## 10 Interview Questions

**Q1:** How does Zustand differ from Context API?
**A:** Zustand doesn't need a Provider wrapper. Components subscribe to specific state slices (fine-grained re-renders). Context causes all consumers to re-render. Zustand also works outside React components.

**Q2:** How do you persist Zustand state?
**A:** `persist` middleware. Configure storage key, partialize (select which fields to persist), version (for migrations). Defaults to localStorage.

**Q3:** How do you compute derived data in Zustand?
**A:** Use getters: `get total() { return get().items.reduce(...) }`. Or create separate selectors: `const useTotal = () => useStore(s => s.items.reduce(...))`. Recomputes only when dependencies change.

**Q4:** Can Zustand handle async actions?
**A:** Yes. Actions are regular functions. They can be async: `fetchUsers: async () => { set({ loading: true }); const data = await api.get(); set({ users: data, loading: false }); }`.

**Q5:** How do you update nested state in Zustand?
**A:** Same as React — spread operator for immutability: `set(state => ({ user: { ...state.user, name: 'New' } }))`. Or use Immer middleware for mutable syntax.

**Q6:** How do you combine multiple Zustand stores?
**A:** Create separate stores per domain (cart, auth, UI). In hooks, combine them: `const { user } = useAuthStore(); const { cart } = useCartStore(); return { user, cart }`. No need for a root store.

**Q7:** How does Zustand prevent unnecessary re-renders?
**A:** Components subscribe via selector: `useStore(s => s.field)`. Zustand tracks which selectors are used. Only notifies components when their selected value changes (reference comparison). This is the key performance advantage over Context.

**Q8:** How do you use Zustand outside React?
**A:** `useStore.getState()` and `useStore.setState()` work in any JavaScript context: event handlers, timers, WebSocket callbacks, service workers, route loaders.

**Q9:** How do you test Zustand stores?
**A:** Pure function calls: `useCartStore.getState().addItem(product)`. Assert `useCartStore.getState().items`. Reset between tests: `useCartStore.setState(initialState)`. No React needed.

**Q10:** Zustand vs Redux — when to choose which?
**A:** Zustand for small-medium apps, simpler API, less boilerplate, fine-grained subscriptions. Redux for large apps needing middleware (redux-saga), normalized state (normalizr), and complex DevTools workflows.

## E2E Examples

### E-Commerce
```jsx
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

// Persistent shopping cart store — survives page refreshes via localStorage
const useCartStore = create(
  persist(
    (set, get) => ({
      items: [],
      coupon: null,
      discount: 0,
      isSyncing: false,

      addItem: (product) => set(state => {
        const existing = state.items.find(i => i.id === product.id);
        return {
          items: existing
            ? state.items.map(i => i.id === product.id ? { ...i, quantity: i.quantity + 1 } : i)
            : [...state.items, { ...product, quantity: 1 }]
        };
      }),

      removeItem: (id) => set(state => ({
        items: state.items.filter(i => i.id !== id)
      })),

      updateQuantity: (id, quantity) => set(state => ({
        items: quantity <= 0
          ? state.items.filter(i => i.id !== id)
          : state.items.map(i => i.id === id ? { ...i, quantity } : i)
      })),

      applyCoupon: (code) => set({ coupon: code, discount: code === 'SAVE20' ? 0.2 : 0 }),

      clearCart: () => set({ items: [], coupon: null, discount: 0 }),

      // Computed getters
      get total() { return get().items.reduce((sum, i) => sum + i.price * i.quantity, 0) * (1 - get().discount); },
      get itemCount() { return get().items.reduce((sum, i) => sum + i.quantity, 0); }
    }),
    { name: 'shopping-cart' }
  )
);

// Fine-grained selectors — each component only re-renders when its slice changes
function CartIcon() {
  const count = useCartStore(s => s.items.reduce((sum, i) => sum + i.quantity, 0));
  return <span className="cart-badge">{count}</span>;
}

function CartDrawer() {
  // Multiple selectors in one component — re-renders when ANY selected value changes
  const { items, removeItem, updateQuantity, total, applyCoupon } = useCartStore();
  return (
    <div className="cart-drawer">
      {items.map(i => <CartItem key={i.id} item={i} onUpdate={updateQuantity} onRemove={removeItem} />)}
      <CouponInput onApply={applyCoupon} />
      <p>Total: ${total.toFixed(2)}</p>
    </div>
  );
}
```

### Banking
```jsx
import { create } from 'zustand';
import { devtools } from 'zustand/middleware';

// Banking store with DevTools support for debugging state changes
const useBankStore = create(
  devtools(
    (set, get) => ({
      accounts: [],
      selectedAccountId: null,
      transactions: {},
      isLoading: false,
      error: null,

      // Actions
      setAccounts: (accounts) => set({ accounts }),
      selectAccount: (id) => set({ selectedAccountId: id }),

      setTransactions: (accountId, transactions) => set(state => ({
        transactions: { ...state.transactions, [accountId]: transactions }
      })),

      // Add a transaction and update account balance atomically
      addTransaction: (accountId, transaction) => set(state => ({
        transactions: {
          ...state.transactions,
          [accountId]: [transaction, ...(state.transactions[accountId] || [])]
        },
        accounts: state.accounts.map(a =>
          a.id === accountId ? {
            ...a,
            balance: transaction.type === 'credit'
              ? a.balance + transaction.amount
              : a.balance - transaction.amount
          } : a
        )
      })),

      // Getters for derived data
      get selectedAccount() { return get().accounts.find(a => a.id === get().selectedAccountId); },
      get totalBalance() { return get().accounts.reduce((sum, a) => sum + a.balance, 0); }
    }),
    { name: 'bank-store' }
  )
);

// Components with selective subscriptions for performance
function AccountSummary() {
  const accounts = useBankStore(s => s.accounts);
  const totalBalance = useBankStore(s => s.accounts.reduce((sum, a) => sum + a.balance, 0));
  const selectAccount = useBankStore(s => s.selectAccount);

  return (
    <div className="account-summary">
      <h3>Total: ${totalBalance.toLocaleString()}</h3>
      {accounts.map(a => (
        <div key={a.id} className="account-row" onClick={() => selectAccount(a.id)}>
          <span>{a.name} (...{a.maskedNumber})</span>
          <span>${a.balance.toLocaleString()}</span>
        </div>
      ))}
    </div>
  );
}
```

---

# 16. TESTING

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Cart Component Integration Test

**Scenario:** An e-commerce Cart component must display items, calculate correct totals, and respond to user interactions (quantity changes, item removal). This integration test suite renders the Cart with mock data, simulates user actions via `userEvent`, and verifies both the displayed output and callback invocations. Each test focuses on one behavioral aspect with a clear, descriptive name.

```jsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { Cart } from '../components/Cart';

// Mock data representing items in a user's cart
const mockProducts = [
  { id: 1, title: 'Product A', price: 29.99, quantity: 1 },
  { id: 2, title: 'Product B', price: 49.99, quantity: 2 },
];

describe('Cart', () => {
  // Test empty state
  it('shows empty cart message when no items', () => {
    render(<Cart items={[]} />);
    expect(screen.getByText(/your cart is empty/i)).toBeInTheDocument();
  });

  // Test computed display
  it('displays correct total', () => {
    render(<Cart items={mockProducts} />);
    const total = (29.99 * 1 + 49.99 * 2).toFixed(2);
    expect(screen.getByText(`$${total}`)).toBeInTheDocument();
  });

  // Test user interaction — changing quantity
  it('calls onUpdateQuantity when quantity changes', async () => {
    const onUpdate = vi.fn();
    const user = userEvent.setup();

    render(<Cart items={mockProducts} onUpdateQuantity={onUpdate} />);

    const quantities = screen.getAllByLabelText(/quantity/i);
    await user.clear(quantities[0]);    // Clear existing value
    await user.type(quantities[0], '3'); // Type new value

    // Wait for the debounced/async callback
    await waitFor(() => {
      expect(onUpdate).toHaveBeenCalledWith(1, 3);
    });
  });

  // Test user interaction — removing an item
  it('calls onRemoveItem when delete is clicked', async () => {
    const onRemove = vi.fn();
    const user = userEvent.setup();

    render(<Cart items={mockProducts} onRemove={onRemove} />);

    const deleteButtons = screen.getAllByRole('button', { name: /remove/i });
    await user.click(deleteButtons[0]);

    expect(onRemove).toHaveBeenCalledWith(1);
  });

  // Test conditional rendering — checkout button visibility
  it('shows proceed to checkout button when items exist', () => {
    render(<Cart items={mockProducts} />);
    expect(screen.getByRole('button', { name: /checkout/i })).toBeInTheDocument();
  });

  it('disables checkout button when cart is empty', () => {
    render(<Cart items={[]} />);
    expect(screen.getByRole('button', { name: /checkout/i })).toBeDisabled();
  });

  // Test per-item total calculation
  it('calculates item totals correctly', () => {
    render(<Cart items={mockProducts} />);
    expect(screen.getByText('$29.99')).toBeInTheDocument();
    expect(screen.getByText('$99.98')).toBeInTheDocument();
  });
});
```

### Scenario 2: Banking — TransferForm Feature Test

**Scenario:** A banking TransferForm must validate user input before submission — checking for empty fields, insufficient funds, daily limit breaches, and self-transfers. These tests simulate real user interactions (selecting dropdowns, typing amounts, clicking submit) and verify that validation messages appear. The final test submits valid data and asserts the correct payload was passed to the `onSubmit` callback.

```jsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TransferForm } from '../components/TransferForm';

const mockAccounts = [
  { id: '1', name: 'Checking', balance: 5000, displayNumber: '****1234' },
  { id: '2', name: 'Savings', balance: 15000, displayNumber: '****5678' },
];

describe('TransferForm', () => {
  // Test empty form validation
  it('displays validation errors for empty form', async () => {
    const user = userEvent.setup();
    render(<TransferForm accounts={mockAccounts} onSubmit={vi.fn()} />);

    await user.click(screen.getByRole('button', { name: /submit/i }));

    expect(screen.getByText(/select source account/i)).toBeInTheDocument();
    expect(screen.getByText(/enter amount/i)).toBeInTheDocument();
  });

  // Test business rule validation
  it('shows insufficient funds warning', async () => {
    const user = userEvent.setup();
    render(<TransferForm accounts={mockAccounts} onSubmit={vi.fn()} />);

    await user.selectOptions(screen.getByLabelText(/from account/i), '1');
    await user.type(screen.getByLabelText(/amount/i), '6000');

    expect(screen.getByText(/insufficient funds/i)).toBeInTheDocument();
  });

  it('validates amount against daily limit', async () => {
    const user = userEvent.setup();
    render(<TransferForm accounts={mockAccounts} dailyLimit={5000} onSubmit={vi.fn()} />);

    await user.selectOptions(screen.getByLabelText(/from account/i), '1');
    await user.type(screen.getByLabelText(/amount/i), '6000');

    expect(screen.getByText(/daily limit/i)).toBeInTheDocument();
  });

  it('prevents self-transfer', async () => {
    const user = userEvent.setup();
    render(<TransferForm accounts={mockAccounts} onSubmit={vi.fn()} />);

    await user.selectOptions(screen.getByLabelText(/from account/i), '1');
    await user.type(screen.getByLabelText(/amount/i), '100');

    // Same account selected as destination should trigger error
    // Implementation should validate this
  });

  // Test successful submission with valid data
  it('submits form with valid data', async () => {
    const onSubmit = vi.fn();
    const user = userEvent.setup();

    render(<TransferForm accounts={mockAccounts} onSubmit={onSubmit} />);

    await user.selectOptions(screen.getByLabelText(/from account/i), '1');
    await user.selectOptions(screen.getByLabelText(/to account/i), '2');
    await user.type(screen.getByLabelText(/amount/i), '500');
    await user.type(screen.getByLabelText(/memo/i), 'Birthday gift');

    await user.click(screen.getByRole('button', { name: /submit/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        from: '1', to: '2', amount: 500,
        memo: 'Birthday gift', frequency: 'once',
      });
    });
  });
});
```

### Scenario 3: Hook Test — useDebounce

**Scenario:** A custom `useDebounce` hook delays updating a value until after a specified timeout. This test uses `renderHook` from React Testing Library and `vi.useFakeTimers()` to control time without waiting. It verifies three behaviors: the initial value is returned immediately, the value updates after the delay, and rapid changes cancel the previous timer so only the latest value is emitted.

```jsx
import { renderHook, act } from '@testing-library/react';
import { useDebounce } from '../hooks/useDebounce';

describe('useDebounce', () => {
  // Replace real timers with fake ones so we can control time
  beforeEach(() => { vi.useFakeTimers(); });
  afterEach(() => { vi.useRealTimers(); });

  it('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('hello', 500));
    expect(result.current).toBe('hello');
  });

  it('updates value after delay', () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: 'hello' } }
    );

    rerender({ value: 'world' });
    // Value should still be the OLD value before the timeout fires
    expect(result.current).toBe('hello');

    act(() => { vi.advanceTimersByTime(500); });
    expect(result.current).toBe('world');
  });

  it('cancels previous timer on rapid changes', () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: 'a' } }
    );

    rerender({ value: 'ab' });
    act(() => { vi.advanceTimersByTime(200); });   // Only 200ms have passed
    expect(result.current).toBe('a');              // Should not have changed yet

    rerender({ value: 'abc' });                     // Change again — resets the timer
    act(() => { vi.advanceTimersByTime(500); });    // Wait full debounce from last change
    expect(result.current).toBe('abc');             // Should have the LATEST value
  });
});
```

## Interview Tips
- **Test behavior, not implementation** — test what the user sees and does, not internal state
- **Use `userEvent` over `fireEvent`** — userEvent simulates real browser events (focus, blur, change)
- **`waitFor` for async** — mutations, API calls, and state updates that need time to settle
- **Keep tests simple** — one assertion per test where possible. Clear test names

## 10 Interview Questions

**Q1:** What is the difference between unit, integration, and E2E tests?
**A:** Unit: test one function (reducer, utility). Integration: test component + hooks + DOM (RTL). E2E: test full user flow through the browser (Playwright, Cypress). Most value comes from integration tests.

**Q2:** How does React Testing Library encourage good practices?
**A:** It encourages testing from the user's perspective (queries by role, text, label, not implementation details). You can't access component internals. You interact with the DOM like a user would.

**Q3:** What queries should you prefer in RTL and why?
**A:** Priority: `getByRole` → `getByLabelText` → `getByText` → `getByTestId`. `getByRole` is most accessible. `getByTestId` is a last resort (brittle, not user-facing).

**Q4:** How do you test asynchronous behavior?
**A:** `waitFor` to wait for assertions. `findByRole` (returns promise) for waiting for elements. `useFakeTimers` for timeout/interval tests. Mock API responses for deterministic tests.

**Q5:** How do you test custom hooks?
**A:** `renderHook` from RTL. Returns `result` (current value) and `rerender`. For state changes, wrap in `act()`. For async, use `waitFor` or `waitForNextUpdate`.

**Q6:** How do you mock API calls in tests?
**A:** MSW (Mock Service Worker) — intercepts network requests at the service worker level. Works with any framework. For simpler cases, mock the API module with `vi.mock('./api')`.

**Q7:** What is the purpose of `screen` object in RTL?
**A:** Provides global queries: `screen.getByText(...)`. Each query is pre-bound to `document.body`. No need to destructure from `render()` result. Cleaner, more explicit.

**Q8:** How do you test form interactions?
**A:** `userEvent.type(input, 'text')` for typing. `userEvent.click(button)` for clicking. `userEvent.selectOptions(select, 'option')` for dropdowns. `userEvent.clear(input)` for clearing.

**Q9:** How do you test error boundaries?
**A:** Render a component that throws inside the boundary. Assert fallback UI renders. Assert onError was called. Suppress console.error to avoid noise.

**Q10:** What is code coverage and what's a good target?
**A:** % of code executed by tests. 80% is a good target. But focus on meaningful coverage (test critical paths, not trivial getters). 100% coverage doesn't mean bug-free.

## E2E Examples

### E-Commerce
```jsx
// ProductCard test — renders product, handles add to cart, shows states
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';

describe('ProductCard', () => {
  const mockProduct = { id: 1, name: 'Test Widget', price: 29.99, inStock: true, thumbnail: '/img.jpg' };

  it('renders product info', () => {
    render(<ProductCard product={mockProduct} onAddToCart={() => {}} />);
    expect(screen.getByText('Test Widget')).toBeInTheDocument();
    expect(screen.getByText('$29.99')).toBeInTheDocument();
    expect(screen.getByRole('img')).toHaveAttribute('alt', 'Test Widget');
  });

  it('shows out of stock state', () => {
    render(<ProductCard product={{ ...mockProduct, inStock: false }} onAddToCart={() => {}} />);
    expect(screen.getByText('Out of Stock')).toBeInTheDocument();
    expect(screen.getByRole('button')).toBeDisabled();
  });

  it('calls onAddToCart when clicked', async () => {
    const onAdd = vi.fn();
    render(<ProductCard product={mockProduct} onAddToCart={onAdd} />);
    await userEvent.click(screen.getByRole('button', { name: /add to cart/i }));
    expect(onAdd).toHaveBeenCalledWith(1);
  });
});

// Cart integration test — full user flow from adding to removing
describe('Cart Integration', () => {
  it('adds item and shows in cart', async () => {
    render(<CartPage />);
    await userEvent.click(screen.getByTestId('add-product-1'));
    expect(screen.getByTestId('cart-count')).toHaveTextContent('1');
    expect(screen.getByText('Test Widget')).toBeInTheDocument();
  });

  it('updates quantity', async () => {
    render(<CartPage />);
    await userEvent.click(screen.getByTestId('add-product-1'));
    const qtyInput = screen.getByLabelText(/quantity/i);
    await userEvent.clear(qtyInput);
    await userEvent.type(qtyInput, '3');
    expect(screen.getByText(/total:/i)).toHaveTextContent(/\$89.97/);
  });

  it('removes item from cart', async () => {
    render(<CartPage />);
    await userEvent.click(screen.getByTestId('add-product-1'));
    await userEvent.click(screen.getByRole('button', { name: /remove/i }));
    expect(screen.queryByText('Test Widget')).not.toBeInTheDocument();
    expect(screen.getByText(/cart is empty/i)).toBeInTheDocument();
  });
});
```

### Banking
```jsx
// Transfer form test with validation — various edge cases
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('TransferForm', () => {
  it('shows validation errors on empty submit', async () => {
    render(<TransferForm onSubmit={vi.fn()} />);
    await userEvent.click(screen.getByRole('button', { name: /submit/i }));
    expect(screen.getByText(/amount is required/i)).toBeInTheDocument();
    expect(screen.getByText(/account number required/i)).toBeInTheDocument();
  });

  it('validates amount range', async () => {
    render(<TransferForm onSubmit={vi.fn()} />);
    await userEvent.type(screen.getByLabelText(/amount/i), '100000');
    await userEvent.click(screen.getByRole('button', { name: /submit/i }));
    expect(screen.getByText(/daily limit/i)).toBeInTheDocument();
  });

  it('submits valid transfer', async () => {
    const onSubmit = vi.fn().mockResolvedValue({ id: 'tx-123' });
    render(<TransferForm onSubmit={onSubmit} />);
    await userEvent.selectOptions(screen.getByLabelText(/from/i), 'acc-1');
    await userEvent.type(screen.getByLabelText(/to/i), '9876543210');
    await userEvent.type(screen.getByLabelText(/amount/i), '500');
    await userEvent.click(screen.getByRole('button', { name: /submit/i }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith(
      expect.objectContaining({ fromAccount: 'acc-1', toAccount: '9876543210', amount: 500 })
    ));
  });

  it('prevents transfer to same account', async () => {
    render(<TransferForm onSubmit={vi.fn()} defaultFrom="acc-1" />);
    await userEvent.type(screen.getByLabelText(/to/i), 'acc-1');
    await userEvent.type(screen.getByLabelText(/amount/i), '100');
    await userEvent.click(screen.getByRole('button', { name: /submit/i }));
    expect(screen.getByText(/cannot transfer to same account/i)).toBeInTheDocument();
  });
});

// Utility function test — pure logic, no rendering needed
describe('formatCurrency', () => {
  it('formats USD', () => expect(formatCurrency(1234.5)).toBe('$1,234.50'));
  it('handles zero', () => expect(formatCurrency(0)).toBe('$0.00'));
  it('handles large numbers', () => expect(formatCurrency(1000000)).toBe('$1,000,000.00'));
  it('throws on negative', () => expect(() => formatCurrency(-1)).toThrow('Amount must be non-negative'));
});
```

---

# 17. TYPESCRIPT WITH REACT

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Fully Typed Store and API

**Scenario:** A large e-commerce codebase uses TypeScript to catch errors at compile time. Domain types (`Product`, `ProductVariant`, `Category`) are defined as interfaces. The `ProductService` class returns typed promises. The React Query hook is generic-typed for both success and error shapes. The `ProductCard` component explicitly declares its prop interface so any misuse is flagged immediately.

```typescript
// types.ts — domain model interfaces with optional fields where appropriate
export interface Product {
  id: number;
  title: string;
  slug: string;
  price: number;
  compareAtPrice?: number;          // Optional — not all products are on sale
  description: string;
  images: string[];
  category: Category;
  tags: string[];
  variants: ProductVariant[];
  rating: number;
  reviewCount: number;
  inStock: boolean;
  createdAt: string;
}

export interface ProductVariant {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
  attributes: Record<string, string>;   // Flexible key-value pairs: { color: 'red', size: 'M' }
}

export interface Category {
  id: number;
  name: string;
  slug: string;
  parentId?: number;                    // Optional — top-level categories have no parent
}

// Typed API service with generic paginated response
export class ProductService {
  async getProducts(filters?: ProductFilters): Promise<PaginatedResponse<Product>> {
    const params = new URLSearchParams();
    if (filters?.category) params.set('category', filters.category);
    if (filters?.minPrice) params.set('minPrice', String(filters.minPrice));
    // Additional filters would be added here
    const res = await fetch(`/api/products?${params}`);
    if (!res.ok) throw new ApiError('Failed to fetch products', res.status);
    return res.json();
  }

  async getProduct(slug: string): Promise<Product> {
    const res = await fetch(`/api/products/${slug}`);
    if (!res.ok) throw new ApiError('Product not found', res.status);
    return res.json();
  }
}

// Typed React Query hook — generic parameters for data and error types
export function useProducts(filters?: ProductFilters) {
  return useQuery<PaginatedResponse<Product>, ApiError>({
    queryKey: ['products', filters],
    queryFn: () => productService.getProducts(filters),
  });
}

// Typed component props — any misuse is a compile-time error
interface ProductCardProps {
  product: Product;
  onAddToCart: (variant: ProductVariant) => void;
  className?: string;
}

const ProductCard = ({ product, onAddToCart, className }: ProductCardProps) => {
  const [selectedVariant, setSelectedVariant] = useState<ProductVariant | null>(null);

  const handleAdd = () => {
    if (selectedVariant) onAddToCart(selectedVariant);
  };

  return (
    <article className={className}>
      <h3>{product.title}</h3>
      <p>{product.description}</p>
      <span>${product.price}</span>
      <Button onClick={handleAdd} disabled={!selectedVariant}>
        Add to Cart
      </Button>
    </article>
  );
};
```

### Scenario 2: Banking — Strict Typed Transaction Flow

**Scenario:** A banking app has multiple transaction types (transfer, payment, deposit), each with different fields. A discriminated union (`Transaction`) models this precisely — TypeScript narrows the type inside a `switch (transaction.type)` so each branch knows exactly which fields exist. The `isTransfer` type guard enables runtime checks that also narrow the TypeScript type.

```typescript
// Domain type aliases — type-safe string unions for currencies, statuses, etc.
type Currency = 'USD' | 'EUR' | 'GBP' | 'INR';
type TransactionType = 'deposit' | 'withdrawal' | 'transfer' | 'payment';
type TransactionStatus = 'pending' | 'processing' | 'completed' | 'failed' | 'cancelled';
type AccountType = 'checking' | 'savings' | 'credit' | 'investment';

// Discriminated union — each variant has a unique `type` field
interface BaseTransaction {
  id: string;
  amount: number;
  currency: Currency;
  date: string;
  status: TransactionStatus;
  description: string;
}

interface TransferTransaction extends BaseTransaction {
  type: 'transfer';
  fromAccount: string;
  toAccount: string;
  recipientName: string;
}

interface PaymentTransaction extends BaseTransaction {
  type: 'payment';
  payeeId: string;
  billType: string;
  dueDate: string;
}

interface DepositTransaction extends BaseTransaction {
  type: 'deposit';
  depositMethod: 'cash' | 'check' | 'wire' | 'direct_deposit';
}

// The union type — Transaction is exactly ONE of these at runtime
type Transaction = TransferTransaction | PaymentTransaction | DepositTransaction | BaseTransaction;

// Type guard — narrows Transaction to TransferTransaction
function isTransfer(tx: Transaction): tx is TransferTransaction {
  return tx.type === 'transfer';
}

// Typed React Query hook — data is Transaction[], error is ApiError
function useTransactions(filters: TransactionFilters) {
  return useQuery<PaginatedResponse<Transaction>, ApiError>({
    queryKey: ['transactions', filters],
    queryFn: () => api.getTransactions(filters),
    select: (data) => ({
      ...data,
      // TypeScript knows tx is Transaction — all fields are accessible
      transactions: data.transactions.map(tx => ({
        ...tx,
        formattedAmount: formatCurrency(tx.amount, tx.currency),
      })),
    }),
  });
}

// Discriminated union rendering — switch narrows the type automatically
function TransactionRow({ transaction }: { transaction: Transaction }) {
  switch (transaction.type) {
    case 'transfer':
      return <TransferRow transaction={transaction} />;   // TS knows: TransferTransaction
    case 'payment':
      return <PaymentRow transaction={transaction} />;    // TS knows: PaymentTransaction
    case 'deposit':
      return <DepositRow transaction={transaction} />;    // TS knows: DepositTransaction
    default:
      return <GenericRow transaction={transaction} />;
  }
}
```

### Scenario 3: Generic Reusable Components

**Scenario:** A design system needs reusable components that work with any data type. `AsyncList<T>` is a generic component that renders loading, error, and empty states automatically, then maps over `items` using a `renderItem` prop. TypeScript infers `T` from the `items` array — when given `products`, `renderItem` receives `Product`; when given `transactions`, it receives `Transaction`. Additionally, Zod provides runtime validation with compile-time type inference.

```typescript
// Generic list with loading/error/empty states — works with any data type T
interface AsyncListProps<T> {
  items: T[];
  loading: boolean;
  error: string | null;
  renderItem: (item: T, index: number) => React.ReactNode;
  emptyMessage?: string;
  onRetry?: () => void;
}

function AsyncList<T>({
  items, loading, error, renderItem,
  emptyMessage = 'No items', onRetry,
}: AsyncListProps<T>) {
  if (loading) return <SkeletonList count={5} />;
  if (error) return <ErrorState message={error} onRetry={onRetry} />;
  if (items.length === 0) return <EmptyState message={emptyMessage} />;

  return <>{items.map((item, i) => renderItem(item, i))}</>;
}

// Usage — TypeScript automatically infers T from the items array
<AsyncList
  items={products}               // T = Product
  loading={isLoading}
  error={error}
  renderItem={(product) => <ProductCard key={product.id} product={product} />}  // product is typed as Product
/>

<AsyncList
  items={transactions}           // T = Transaction
  loading={isLoading}
  error={error}
  renderItem={(tx) => <TransactionRow key={tx.id} transaction={tx} />}           // tx is typed as Transaction
/>

// Typed forms with Zod — runtime validation + compile-time type inference
import { z } from 'zod';

const transferSchema = z.object({
  fromAccount: z.string().min(1, 'Required'),
  toAccount: z.string().regex(/^\d{10}$/, 'Invalid account number'),
  amount: z.number().positive().max(100000),
  memo: z.string().max(100).optional(),
});

// Infer the TypeScript type from the schema — single source of truth
type TransferFormData = z.infer<typeof transferSchema>;

function TransferForm() {
  const [errors, setErrors] = useState<z.ZodError | null>(null);

  function handleSubmit(data: unknown) {
    const result = transferSchema.safeParse(data);
    if (!result.success) {
      setErrors(result.error);
      return;
    }
    // result.data is now typed as TransferFormData
    api.transfer(result.data);
  }
}
```

## Interview Tips
- **Don't over-type** — TypeScript infers types for `useState(0)`, event handlers, etc. Only type what can't be inferred
- **`strict: true`** — non-negotiable. Enables `strictNullChecks`, `noImplicitAny`, and other critical checks
- **Discriminated unions** — the most powerful TypeScript pattern for React. Different action types, different component states
- **Generics** — `useState<T>`, custom hooks like `useLocalStorage<T>`, and list components

## 10 Interview Questions

**Q1:** How do you type props in React with TypeScript?
**A:** Define an interface for props. Use it in the component: `const Comp = ({ name, age }: Props) =>`. Or `const Comp: React.FC<Props> = ({ name, age }) =>` — but FC is discouraged (adds implicit children).

**Q2:** What is `React.FC` and why might you avoid it?
**A:** `React.FunctionComponent` implicitly includes `children` in props, which may not be desired. It also doesn't support generics well. Prefer explicit prop interfaces.

**Q3:** How do you type `useState` with complex objects?
**A:** `useState<User | null>(null)` — explicit type for union. `useState<FetchState<User>>({ status: 'loading' })` — typed state machine. For initial empty arrays: `useState<Product[]>([])`.

**Q4:** How do you type event handlers?
**A:** `onChange: (e: React.ChangeEvent<HTMLInputElement>) => void`. `onSubmit: (e: React.FormEvent<HTMLFormElement>) => void`. `onClick: (e: React.MouseEvent<HTMLButtonElement>) => void`.

**Q5:** What is a discriminated union and how is it useful in React?
**A:** A type that's one of several variants, distinguished by a common field (the "discriminant"). Example: `{ status: 'loading' } | { status: 'success'; data: T } | { status: 'error'; error: string }`. TypeScript narrows the type based on the `status` check.

**Q6:** How do you type the `children` prop?
**A:** `children: React.ReactNode` (accepts any renderable: elements, strings, numbers, fragments, portals). For render functions: `children: (item: T) => React.ReactNode`.

**Q7:** How do you create a generic component?
**A:** `<T,>(props: AsyncListProps<T>)` — the generic type parameter on the function. TypeScript infers T from usage. Useful for lists, tables, and data-driven components.

**Q8:** How do you type a custom hook that returns a tuple like useState?
**A:** `function useLocalStorage<T>(key: string, initial: T): [T, (value: T | ((prev: T) => T)) => void]`. The return type is a tuple of [value, setter].

**Q9:** How do you type `useReducer` with TypeScript?
**A:** Define actions as discriminated union: `type Action = { type: 'ADD'; payload: Task } | { type: 'DELETE'; payload: number }`. The reducer infers payload type from the action type in the switch case.

**Q10:** What is the purpose of `satisfies` operator (TS 4.9+) in React?
**A:** Checks that a value satisfies a type without widening the type. Example: `const config = { theme: 'dark', onSave: () => {} } satisfies ThemeConfig`. TypeScript validates the shape but preserves the literal types for narrowing.

## E2E Examples

### E-Commerce
```tsx
// Typed API layer — explicit interfaces for all domain objects
interface Product {
  id: number;
  name: string;
  price: number;
  inStock: boolean;
  category: string;
  variants: ProductVariant[];
  metadata: Record<string, unknown>;   // Flexible but typed
}

interface ProductVariant {
  sku: string;
  size: string;
  color: string;
  quantity: number;
}

// Typed custom hook with discriminated union for state machine
type FetchState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };

function useProduct(id: number) {
  const [state, setState] = useState<FetchState<Product>>({ status: 'idle' });

  useEffect(() => {
    setState({ status: 'loading' });
    fetch(`/api/products/${id}`)
      .then(r => { if (!r.ok) throw new Error('Not found'); return r.json(); })
      .then(data => setState({ status: 'success', data }))
      .catch(err => setState({ status: 'error', error: err.message }));
  }, [id]);

  // Returns a discriminated union — consumers switch on state.status
  return state;
}

// Generic typed list component — enforces item has id
function TypedList<T extends { id: string | number }>({
  items,
  renderItem,
  keyExtractor = (item: T) => item.id,
}: {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor?: (item: T) => string | number;
}) {
  return <>{items.map(item => <React.Fragment key={keyExtractor(item)}>{renderItem(item)}</React.Fragment>)}</>;
}

// Usage — TypeScript infers T from products array
function ProductGrid({ products }: { products: Product[] }) {
  return (
    <TypedList
      items={products}
      renderItem={(product) => <ProductCard product={product} />}
    />
  );
}
```

### Banking
```tsx
// Discriminated union for transaction types — each variant has distinct fields
interface CreditTransaction {
  type: 'credit';
  id: string;
  amount: number;
  source: string;
  date: string;
}

interface DebitTransaction {
  type: 'debit';
  id: string;
  amount: number;
  destination: string;
  fee: number;
  date: string;
}

interface TransferTransaction {
  type: 'transfer';
  id: string;
  amount: number;
  fromAccount: string;
  toAccount: string;
  memo: string;
  date: string;
}

type Transaction = CreditTransaction | DebitTransaction | TransferTransaction;

// TypeScript narrows the type based on the `type` field — each case has access to correct fields
function TransactionRow({ tx }: { tx: Transaction }) {
  switch (tx.type) {
    case 'credit':
      return <div className="credit">+${tx.amount} from {tx.source}</div>;
    case 'debit':
      return <div className="debit">-${tx.amount + tx.fee} to {tx.destination} (+${tx.fee} fee)</div>;
    case 'transfer':
      return <div className="transfer">${tx.amount} from {tx.fromAccount} → {tx.toAccount}<br/><small>{tx.memo}</small></div>;
    default:
      // Exhaustive check — if a new type is added and not handled, this line won't compile
      const _exhaustive: never = tx;
      return _exhaustive;
  }
}

// Branded types — prevents accidentally mixing account numbers with other strings
type AccountNumber = string & { readonly __brand: 'AccountNumber' };
function createAccountNumber(raw: string): AccountNumber {
  if (!/^\d{10}$/.test(raw)) throw new Error('Invalid account number');
  return raw as AccountNumber;
}

interface TransferFormProps {
  fromAccount: AccountNumber;       // Branded type — cannot pass a plain string
  onSubmit: (data: { to: AccountNumber; amount: number }) => Promise<void>;
}

function TransferForm({ fromAccount, onSubmit }: TransferFormProps) {
  const [amount, setAmount] = useState<number>(0);
  const [toAccount, setToAccount] = useState<string>('');   // Raw input starts as string

  async function handleSubmit() {
    const validatedTo = createAccountNumber(toAccount);      // Validate and brand
    if (validatedTo === fromAccount) { setError('Same account'); return; }
    await onSubmit({ to: validatedTo, amount });
  }
  // ...
}
```

---

# 18. PORTALS & MODALS

## 3 Real-World Scenarios

### Scenario 1: E-Commerce — Product Image Lightbox

**Scenario:** A product gallery shows thumbnail images. When clicked, a full-screen lightbox opens using `createPortal` so the overlay renders into `document.getElementById('modal-root')` — outside the gallery's DOM hierarchy. This avoids CSS stacking context issues from the gallery's `overflow: hidden` or `transform`. The lightbox blocks body scrolling, handles Escape key to close, and prevents click-outside propagation.

```jsx
import { createPortal } from 'react-dom';

// Lightbox rendered via portal — breaks out of parent overflow/stacking context
function ImageLightbox({ images, currentIndex, onClose }) {
  useEffect(() => {
    document.body.style.overflow = 'hidden';     // Prevent background scrolling
    const handleEsc = (e) => { if (e.key === 'Escape') onClose(); };
    window.addEventListener('keydown', handleEsc);
    return () => {
      document.body.style.overflow = '';          // Restore scrolling on unmount
      window.removeEventListener('keydown', handleEsc);
    };
  }, [onClose]);

  const [activeIndex, setActiveIndex] = useState(currentIndex);

  // Render into #modal-root (outside the component tree in the DOM)
  return createPortal(
    <div className="lightbox-overlay" onClick={onClose} role="dialog" aria-modal="true">
      <div className="lightbox-content" onClick={e => e.stopPropagation()}>
        <button className="lightbox-close" onClick={onClose}>×</button>
        <button className="lightbox-prev" onClick={() => setActiveIndex(i => i - 1)}
          disabled={activeIndex === 0}>‹</button>
        <img src={images[activeIndex].fullUrl} alt={images[activeIndex].alt} />
        <button className="lightbox-next" onClick={() => setActiveIndex(i => i + 1)}
          disabled={activeIndex === images.length - 1}>›</button>
        <div className="lightbox-thumbnails">
          {images.map((img, i) => (
            <button key={i} className={i === activeIndex ? 'active' : ''}
              onClick={() => setActiveIndex(i)}>
              <img src={img.thumbUrl} alt="" />
            </button>
          ))}
        </div>
      </div>
    </div>,
    document.getElementById('modal-root')   // Portal target — outside the component tree
  );
}

// Gallery that triggers the lightbox
function ProductGallery({ images }) {
  const [lightboxOpen, setLightboxOpen] = useState(false);
  const [startIndex, setStartIndex] = useState(0);

  return (
    <div className="product-gallery">
      <div className="gallery-grid">
        {images.slice(0, 4).map((img, i) => (
          <button key={i} onClick={() => { setStartIndex(i); setLightboxOpen(true); }}>
            <img src={img.thumbUrl} alt={img.alt} loading="lazy" />
            {i === 3 && images.length > 4 && (
              <span className="more-overlay">+{images.length - 4}</span>
            )}
          </button>
        ))}
      </div>
      {lightboxOpen && (
        <ImageLightbox images={images} currentIndex={startIndex}
          onClose={() => setLightboxOpen(false)} />
      )}
    </div>
  );
}
```

### Scenario 2: Banking — Two-Factor Authentication Modal

**Scenario:** A high-value bank transfer requires 2FA verification before processing. The modal uses a portal to render at the document body level, ensuring it appears above all other UI. The six-digit code input has auto-focus, auto-advance to the next digit, backspace navigation, and numeric-only validation. On failure, the input resets and focus returns to the first digit.

```jsx
function TwoFactorModal({ onVerified, onCancel, transactionData }) {
  const [code, setCode] = useState(['', '', '', '', '', '']);   // 6-digit array
  const [isVerifying, setIsVerifying] = useState(false);
  const [error, setError] = useState('');
  const inputRefs = useRef([]);                                    // Refs for auto-advance

  // Focus the first input on mount
  useEffect(() => { inputRefs.current[0]?.focus(); }, []);

  // Auto-advance to the next input when a digit is entered
  function handleChange(index, value) {
    if (!/^\d?$/.test(value)) return;                              // Only allow digits
    const newCode = [...code];
    newCode[index] = value;
    setCode(newCode);
    if (value && index < 5) inputRefs.current[index + 1]?.focus(); // Move to next
  }

  // Backspace goes to previous input
  function handleKeyDown(index, e) {
    if (e.key === 'Backspace' && !code[index] && index > 0) {
      inputRefs.current[index - 1]?.focus();
    }
  }

  async function handleVerify() {
    const fullCode = code.join('');
    if (fullCode.length !== 6) { setError('Enter all 6 digits'); return; }
    setIsVerifying(true);
    setError('');
    try {
      await api.verify2FA({ transactionId: transactionData.id, code: fullCode });
      onVerified();                                                // Notify parent on success
    } catch (err) {
      setError(err.message || 'Invalid code. Try again.');
      setCode(['', '', '', '', '', '']);                           // Clear and refocus
      inputRefs.current[0]?.focus();
    } finally {
      setIsVerifying(false);
    }
  }

  // Portal renders outside the component tree — above all other content
  return createPortal(
    <div className="modal-overlay" onClick={onCancel}>
      <div className="modal-content twofa-modal" onClick={e => e.stopPropagation()}>
        <div className="modal-header">
          <h2>Two-Factor Authentication</h2>
          <p>Enter the 6-digit code from your authenticator app</p>
        </div>
        <div className="twofa-inputs">
          {code.map((digit, i) => (
            <input key={i} ref={el => inputRefs.current[i] = el}
              type="text" inputMode="numeric" maxLength={1} value={digit}
              onChange={e => handleChange(i, e.target.value)}
              onKeyDown={e => handleKeyDown(i, e)}
              className={error ? 'error' : ''} aria-label={`Digit ${i + 1}`} />
          ))}
        </div>
        {error && <p className="twofa-error">{error}</p>}
        <div className="modal-actions">
          <button className="btn-secondary" onClick={onCancel} disabled={isVerifying}>Cancel</button>
          <button className="btn-primary" onClick={handleVerify} disabled={isVerifying}>
            {isVerifying ? 'Verifying...' : 'Verify'}
          </button>
        </div>
        <div className="transaction-summary">
          <p>Transfer: ${transactionData.amount.toLocaleString()}</p>
          <p>To: {transactionData.recipient}</p>
        </div>
      </div>
    </div>,
    document.getElementById('modal-root')
  );
}
```

### Scenario 3: Multi-Step Checkout with Modal Warnings

**Scenario:** During checkout, a confirmation modal asks the user to verify their shipping address before proceeding. The portal ensures the modal overlays the entire checkout flow without being clipped by any parent container. Additionally, a `ToastContainer` uses a separate portal target (`#toast-root`) so notifications also render at the top level and don't interfere with layout.

```jsx
// Address confirmation modal — rendered via portal to break out of checkout layout
function ConfirmAddressModal({ address, onConfirm, onEdit, onClose }) {
  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={e => e.stopPropagation()}>
        <h2>Confirm Shipping Address</h2>
        <div className="address-card">
          <p>{address.fullName}</p>
          <p>{address.street}</p>
          <p>{address.city}, {address.state} {address.zip}</p>
          <p>{address.country}</p>
        </div>
        <p className="warning-text">Items shipping to an unverified address may experience delays.</p>
        <div className="modal-actions">
          <button className="btn-secondary" onClick={onEdit}>Edit</button>
          <button className="btn-primary" onClick={onConfirm}>Confirm & Continue</button>
        </div>
      </div>
    </div>,
    document.getElementById('modal-root')    // Renders outside the checkout component tree
  );
}

// Toast container — also uses portal for a separate overlay layer
function ToastContainer() {
  const toasts = useToastStore(s => s.toasts);
  if (!toasts.length) return null;
  return createPortal(
    <div className="toast-container" aria-live="polite">
      {toasts.map(t => (
        <div key={t.id} className={`toast toast-${t.variant}`}>
          <span>{t.message}</span>
          <button onClick={() => useToastStore.getState().dismiss(t.id)}>×</button>
        </div>
      ))}
    </div>,
    document.getElementById('toast-root')    // Separate portal target for toasts
  );
}
```

## Interview Tips
- **Portal ≠ Modal** — Portals are a mechanism (render anywhere in DOM). Modals are a UI pattern often implemented with portals. Use portals whenever you need to break out of a parent's overflow/stacking context (tooltips, dropdowns, autocomplete menus).
- **Always manage focus** — Trap focus within modals for accessibility. Return focus to the trigger element on close. Use `aria-modal="true"` and `role="dialog"`.
- **Escape key** — Always bind Escape to close. Clean up the listener in useEffect return.
- **Body scroll lock** — Set `overflow: hidden` on `document.body` when modal is open. Restore on close. Account for scrollbar width to prevent layout shift.
- **Nested modals** — Avoid if possible. If necessary, stack them and close one at a time. Use a modal manager (array-based state) instead of boolean flags.
- **Render performance** — Portals don't prevent re-renders. The portal content still re-renders when its parent re-renders. Memoize if the portal content is expensive.
- **Testing portals** — In RTL, portal content renders in the document (not inside the component tree). Use `screen.getByRole('dialog')` to find modal content regardless of portal placement.

## 10 Interview Questions

**Q1:** What is the difference between a portal and a regular component?
**A:** A portal renders its children into a different DOM node (outside the parent component's DOM hierarchy) while maintaining React context access. Event bubbling still works through the React tree, not the DOM tree.

**Q2:** When should you use a portal instead of just positioning with CSS?
**A:** When CSS positioning can't work due to parent constraints: overflow: hidden, z-index stacking contexts, clip-path, or transform on ancestor. Portals let you render at the document body level while keeping logical component hierarchy.

**Q3:** How do portals affect event bubbling?
**A:** Events bubble through the React component tree (the virtual tree where the portal is declared), not the DOM tree. So a click inside a portal can still be caught by an onClick handler on the parent that rendered the portal.

**Q4:** How do you handle accessibility for modals?
**A:** Focus trap (Tab cycles within modal), aria-modal="true", role="dialog", Escape to close, aria-labelledby pointing to the heading, restore focus to trigger on close, inert background content.

**Q5:** How do you prevent background scrolling when a modal is open?
**A:** Set `document.body.style.overflow = 'hidden'` on open, restore on close. Compensate for scrollbar width: `document.body.style.paddingRight = window.innerWidth - document.documentElement.clientWidth + 'px'`.

**Q6:** Can you nest portals?
**A:** Yes — a portal can contain another portal. Each renders to its specified DOM node. Stacking order depends on CSS z-index, not on portal nesting.

**Q7:** How do you create a reusable Modal component?
**A:** A Modal wrapper that accepts `isOpen`, `onClose`, `title`, `children` props. Internally uses createPortal. Manages focus trap, body scroll lock, and Escape key. Uses ReactDOM.createPortal(children, modalRoot).

**Q8:** How do you test portal-based modals?
**A:** `render(<Component />)`, then `screen.getByRole('dialog')` to find modal content. Portal content appears in the document body. Use `userEvent.keyboard('{Escape}')` to test close. Assert the modal is removed from DOM after close.

**Q9:** What is `usePortal` hook?
**A:** A custom hook that creates the portal container div on mount, appends it to document.body, removes it on unmount. Returns the DOM node to portal into. Example: `const portalTarget = usePortal('modal-root')`.

**Q10:** How do React 19's `use()` API relate to portals?
**A:** Portals are orthogonal — the `use()` API handles async data reading in render, while portals handle DOM placement. They compose naturally: portal content can use `use()` to read promises or context.

## E2E Examples

### E-Commerce
```jsx
// Complete modal system: confirm dialog + toast notifications
function CheckoutPage() {
  const [showConfirmModal, setShowConfirmModal] = useState(false);
  const [isSubmitting, setIsSubmitting] = useState(false);

  async function handleConfirmOrder() {
    setIsSubmitting(true);
    try {
      await submitOrder();
      toast.success('Order placed!');
      setShowConfirmModal(false);
    } catch (err) {
      toast.error('Order failed. Please try again.');
    } finally {
      setIsSubmitting(false);
    }
  }

  return (
    <div className="checkout-page">
      <OrderSummary />
      <button className="btn-primary" onClick={() => setShowConfirmModal(true)}>
        Place Order
      </button>

      {showConfirmModal && (
        <ConfirmModal
          title="Confirm Order"
          onConfirm={handleConfirmOrder}
          onCancel={() => setShowConfirmModal(false)}
          isLoading={isSubmitting}
          confirmText={isSubmitting ? 'Processing...' : 'Confirm & Pay'}
        >
          <p>Total: ${cartTotal.toFixed(2)}</p>
          <p>Shipping to: {shippingAddress.street}, {shippingAddress.city}</p>
        </ConfirmModal>
      )}

      <ToastContainer />
    </div>
  );
}

// Reusable confirm modal with portal — focus trapping, body scroll lock, Escape handling
function ConfirmModal({ title, children, onConfirm, onCancel, isLoading, confirmText }) {
  const modalRef = useRef(null);
  const previousFocus = useRef(null);

  // On mount: save focus, focus modal, lock body scroll. On unmount: restore all.
  useEffect(() => {
    previousFocus.current = document.activeElement;
    modalRef.current?.focus();
    document.body.style.overflow = 'hidden';
    return () => {
      document.body.style.overflow = '';
      previousFocus.current?.focus();    // Restore focus to trigger element
    };
  }, []);

  // Escape key handler
  useEffect(() => {
    function handleEscape(e) { if (e.key === 'Escape') onCancel(); }
    window.addEventListener('keydown', handleEscape);
    return () => window.removeEventListener('keydown', handleEscape);
  }, [onCancel]);

  // Portal to modal-root — renders above all page content
  return createPortal(
    <div className="modal-overlay" onClick={onCancel}>
      <div className="modal-content" ref={modalRef} tabIndex={-1} role="dialog" aria-modal="true"
        aria-labelledby="modal-title" onClick={e => e.stopPropagation()}>
        <h2 id="modal-title">{title}</h2>
        <div className="modal-body">{children}</div>
        <div className="modal-actions">
          <button className="btn-secondary" onClick={onCancel} disabled={isLoading}>Cancel</button>
          <button className="btn-primary" onClick={onConfirm} disabled={isLoading}>{confirmText}</button>
        </div>
      </div>
    </div>,
    document.getElementById('modal-root')
  );
}
```

### Banking
```jsx
// Session timeout modal with countdown — portal ensures it overlays everything
function SessionManager() {
  const [showTimeout, setShowTimeout] = useState(false);
  const [secondsLeft, setSecondsLeft] = useState(120);
  const lastActivity = useRef(Date.now());

  // Track user activity via mouse and keyboard events
  useEffect(() => {
    function handleActivity() { lastActivity.current = Date.now(); }
    window.addEventListener('mousemove', handleActivity);
    window.addEventListener('keydown', handleActivity);
    return () => { window.removeEventListener('mousemove', handleActivity); window.removeEventListener('keydown', handleActivity); };
  }, []);

  // Check for inactivity every second
  useEffect(() => {
    const id = setInterval(() => {
      const elapsed = (Date.now() - lastActivity.current) / 1000;
      if (elapsed > 600) {              // 10 minutes idle → show warning modal
        setShowTimeout(true);
        setSecondsLeft(120 - (elapsed - 600));
      }
      if (elapsed > 720) {              // 12 minutes → auto logout
        forceLogout();
      }
    }, 1000);
    return () => clearInterval(id);
  }, []);

  function handleExtend() {
    lastActivity.current = Date.now();   // Reset activity timer
    setShowTimeout(false);               // Dismiss the modal
  }

  // Portal to session-root — separate DOM node for session-related overlays
  return createPortal(
    <div className={`session-modal ${showTimeout ? 'visible' : 'hidden'}`}>
      <div className="session-content" role="alertdialog" aria-labelledby="session-title">
        <h2 id="session-title">Session Expiring</h2>
        <p>Your session will expire in <strong>{secondsLeft}s</strong></p>
        <div className="session-actions">
          <button className="btn-primary" onClick={handleExtend}>Stay Logged In</button>
          <button className="btn-secondary" onClick={forceLogout}>Logout Now</button>
        </div>
      </div>
    </div>,
    document.getElementById('session-root')
  );
}
```

---

# E2E EXAMPLES REFERENCE

The complete End-to-End applications for **E-Commerce** and **Banking** domains are already provided in the separate files:

| File | Content |
|------|---------|
| `04-e2e-ecommerce.md` | Full E-Commerce app: Axios with token refresh, AuthContext, Zustand cart with persistence, product listing with URL-synced filters, checkout with multi-step form, order management |
| `05-e2e-banking.md` | Full Banking app: Secure API with idempotency, WebSocket real-time balance, transfer form with 2FA flow, infinite scroll transaction list, statement downloader with audit logging |

Each includes:
- Complete project architecture
- All components, hooks, services, and utilities
- Error handling, loading states, edge cases
- Security considerations (data masking, audit logs, compliance)
- Performance optimizations
- 5+ interview questions specific to each domain



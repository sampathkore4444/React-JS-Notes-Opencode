# React Mastery Guide — Part 6: Interview Mastery & Roadmap
## From Junior to Principal Engineer

---

# 1. REACT ROUTER V6 (Deep Dive)

```jsx
// App.jsx — Complete routing architecture
import { createBrowserRouter, RouterProvider, Navigate, Outlet } from 'react-router-dom';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, isAuthenticated, loading } = useAuth();

  if (loading) return <FullPageSpinner />;
  if (!isAuthenticated) return <Navigate to="/login" state={{ from: location }} replace />;
  if (requiredRole && user.role !== requiredRole) return <Navigate to="/unauthorized" replace />;

  return children || <Outlet />;
};

const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    errorElement: <ErrorPage />,
    children: [
      { index: true, element: <HomePage /> },
      { path: 'login', element: <LoginPage /> },
      { path: 'register', element: <RegisterPage /> },
      { path: 'products', element: <ProductListing /> },
      { path: 'products/:slug', element: <ProductDetail /> },

      // Protected routes
      {
        element: <ProtectedRoute />,
        children: [
          { path: 'cart', element: <CartPage /> },
          { path: 'checkout', element: <CheckoutPage /> },
          { path: 'orders', element: <OrderHistory /> },
          { path: 'orders/:id', element: <OrderDetail /> },
          { path: 'profile', element: <ProfilePage /> },
        ],
      },

      // Admin only
      {
        element: <ProtectedRoute requiredRole="admin" />,
        children: [
          { path: 'admin', element: <AdminDashboard /> },
          { path: 'admin/products', element: <AdminProducts /> },
          { path: 'admin/orders', element: <AdminOrders /> },
        ],
      },
    ],
  },
]);

// Data loading with loaders
const router = createBrowserRouter([
  {
    path: '/products/:slug',
    element: <ProductDetail />,
    loader: async ({ params }) => {
      const { data } = await api.get(`/products/${params.slug}`);
      return data;
    },
    // Error handling per route
    errorElement: <ProductNotFound />,
  },
]);

// Using loader data
const ProductDetail = () => {
  const product = useLoaderData(); // Type-safe!
  return <ProductView product={product} />;
};
```

# 2. TESTING STRATEGY

## Unit Tests (Vitest + Testing Library)

```jsx
// __tests__/TransferForm.test.jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TransferForm } from '../features/transfers/components/TransferForm';
import { bankingApi } from '../services/bankingApi';

jest.mock('../services/bankingApi');

const mockAccounts = [
  { id: '1', name: 'Checking', number: '1234567890', balance: 5000, displayNumber: '****7890' },
  { id: '2', name: 'Savings', number: '0987654321', balance: 15000, displayNumber: '****4321' },
];

describe('TransferForm', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('renders all form fields', () => {
    render(<TransferForm accounts={mockAccounts} onSuccess={jest.fn()} />);

    expect(screen.getByLabelText(/from account/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/amount/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /review transfer/i })).toBeInTheDocument();
  });

  it('shows validation errors for empty form', async () => {
    render(<TransferForm accounts={mockAccounts} onSuccess={jest.fn()} />);

    const user = userEvent.setup();
    await user.click(screen.getByRole('button', { name: /review transfer/i }));

    expect(screen.getByText(/select source account/i)).toBeInTheDocument();
    expect(screen.getByText(/enter valid amount/i)).toBeInTheDocument();
  });

  it('shows insufficient funds warning', async () => {
    render(<TransferForm accounts={mockAccounts} onSuccess={jest.fn()} />);

    const user = userEvent.setup();
    await user.selectOptions(screen.getByLabelText(/from account/i), '1');
    await user.type(screen.getByLabelText(/amount/i), '6000');

    expect(screen.getByText(/insufficient funds/i)).toBeInTheDocument();
  });

  it('submits transfer successfully', async () => {
    bankingApi.initiateTransfer.mockResolvedValue({ transactionId: 'tx_123' });

    const onSuccess = jest.fn();
    render(<TransferForm accounts={mockAccounts} onSuccess={onSuccess} />);

    const user = userEvent.setup();

    // Fill form
    await user.selectOptions(screen.getByLabelText(/from account/i), '1');
    await user.type(screen.getByLabelText(/amount/i), '100');
    await user.selectOptions(screen.getByLabelText(/to account/i), '2');

    // Review
    await user.click(screen.getByRole('button', { name: /review transfer/i }));
    expect(screen.getByText(/review transfer/i)).toBeInTheDocument();

    // Confirm
    await user.click(screen.getByRole('button', { name: /confirm transfer/i }));

    await waitFor(() => {
      expect(onSuccess).toHaveBeenCalledWith({ transactionId: 'tx_123' });
    });
  });
});
```

## Custom Hook Tests

```jsx
// __tests__/useDebounce.test.js
import { renderHook, act } from '@testing-library/react';
import { useDebounce } from '../hooks/useDebounce';

jest.useFakeTimers();

describe('useDebounce', () => {
  it('returns initial value immediately', () => {
    const { result } = renderHook(() => useDebounce('hello', 500));
    expect(result.current).toBe('hello');
  });

  it('debounces value changes', () => {
    const { result, rerender } = renderHook(
      ({ value, delay }) => useDebounce(value, delay),
      { initialProps: { value: 'hello', delay: 500 } }
    );

    // Change value
    rerender({ value: 'world', delay: 500 });

    // Value should still be 'hello' before timeout
    expect(result.current).toBe('hello');

    // Fast-forward time
    act(() => { jest.advanceTimersByTime(500); });

    expect(result.current).toBe('world');
  });
});
```

## E2E Tests (Cypress)

```jsx
// cypress/e2e/checkout.cy.js
describe('Checkout Flow', () => {
  beforeEach(() => {
    cy.login('user@example.com', 'password123');
    cy.addProductToCart('product-1');
    cy.visit('/checkout');
  });

  it('completes full checkout', () => {
    // Step 1: Shipping
    cy.get('[data-testid="first-name"]').type('John');
    cy.get('[data-testid="last-name"]').type('Doe');
    cy.get('[data-testid="address"]').type('123 Main St');
    cy.get('[data-testid="city"]').type('New York');
    cy.get('[data-testid="state"]').type('NY');
    cy.get('[data-testid="zip"]').type('10001');
    cy.get('[data-testid="phone"]').type('555-0100');
    cy.get('[data-testid="continue-btn"]').click();

    // Step 2: Payment
    cy.get('[data-testid="card-number"]').type('4242424242424242');
    cy.get('[data-testid="expiry-month"]').select('12');
    cy.get('[data-testid="expiry-year"]').select('2027');
    cy.get('[data-testid="cvv"]').type('123');
    cy.get('[data-testid="continue-btn"]').click();

    // Step 3: Review
    cy.get('[data-testid="place-order-btn"]').click();

    // Step 4: Confirmation
    cy.get('[data-testid="order-confirmation"]').should('be.visible');
    cy.get('[data-testid="order-number"]').should('exist');
  });
});
```

# 3. TOP 50 INTERVIEW QUESTIONS & ANSWERS

## Fundamentals (10)

| # | Question | Key Answer Points |
|---|----------|-------------------|
| 1 | Virtual DOM — how does it work? | Reconciliation, diffing algorithm (O(n) assumptions), batching |
| 2 | What is JSX? | Syntax sugar for createElement, not HTML, transpiled by Babel |
| 3 | Props vs State | Immutable vs mutable, parent controls props, component controls state |
| 4 | Component lifecycle with hooks | useState, useEffect (mount/update/unmount), useLayoutEffect |
| 5 | Controlled vs uncontrolled | React controls value vs DOM controls it, refs vs state |
| 6 | Key prop importance | Reconciliation, stable identity, never use index if reordering |
| 7 | Event handling in React | SyntheticEvent, delegation, pooling (<17), persist() |
| 8 | Conditional rendering patterns | ternary, &&, IIFE, enum components |
| 9 | Lifting state up | Common ancestor, callback props, when to use vs context |
| 10 | Composition vs Inheritance | Composition wins — children prop, compound components |

## Hooks (10)

| # | Question | Key Answer Points |
|---|----------|-------------------|
| 11 | Rules of hooks | Top level only, same order, only from React functions |
| 12 | useEffect cleanup | Why needed (subscriptions, timers, abort), runs on unmount and before re-run |
| 13 | useRef use cases | DOM refs, mutable values, previous values, callback refs |
| 14 | useMemo vs useCallback | Memoize value vs memoize function, performance optimization |
| 15 | Custom hooks pattern | Extract reusable logic, compose hooks, share stateful logic |
| 16 | Stale closure problem | Captured old values in callbacks, use refs or functional setState |
| 17 | useReducer when to use | Complex state, interdependent state, easier testing |
| 18 | useEffect dependency array | Empty = mount, omitted = every render, values = when changed |
| 19 | useLayoutEffect vs useEffect | useLayoutEffect fires before paint (measurements), useEffect after paint |
| 20 | useTransition for what? | Mark non-urgent updates, keep UI responsive during heavy renders |

## Advanced (10)

| # | Question | Key Answer Points |
|---|----------|-------------------|
| 21 | React.memo vs useMemo | Component memoization vs value memoization |
| 22 | Error Boundaries | Class component requirement, componentDidCatch, fallback UI |
| 23 | Portals | renderOutside DOM parent, maintain context, modals/tooltips |
| 24 | Context API limitations | All consumers re-render, split contexts, use for low-frequency updates |
| 25 | Code splitting | React.lazy + Suspense, route-based splitting, chunk naming |
| 26 | Render props pattern | Function as child, sharing code between components |
| 27 | Higher Order Components | Factory function, wraps component, adds behavior (legacy pattern) |
| 28 | Compound components | Implicit state via context, flexible API, e.g., Accordion, Tabs |
| 29 | Concurrent React | useTransition, useDeferredValue, Suspense, non-blocking rendering |
| 30 | Server components (RSC) | Zero-bundle-size components, run on server, mix with client components |

## Performance (5)

| # | Question | Key Answer Points |
|---|----------|-------------------|
| 31 | How to identify bottlenecks | React DevTools Profiler, why-did-you-render, Chrome Performance tab |
| 32 | When to optimize | Always profile first, don't optimize prematurely, measure impact |
| 33 | Large list performance | Virtualization (react-window), pagination, lazy loading images |
| 34 | Preventing unnecessary re-renders | React.memo, useMemo, useCallback, state colocation, context splitting |
| 35 | Bundle size optimization | Code splitting, tree shaking, analyze bundle, dynamic imports |

## State Management (5)

| # | Question | Key Answer Points |
|---|----------|-------------------|
| 36 | Context vs Redux vs Zustand | Context for low-frequency, Zustand for medium, Redux for large/complex |
| 37 | State normalization | Flatten nested data, use IDs as keys, avoid duplication |
| 38 | Immutability importance | Reference equality for change detection, never mutate state directly |
| 39 | Optimistic updates | Update UI first, API call in background, rollback on error |
| 40 | Server state management | React Query/SWR handle caching, refetching, stale-while-revalidate |

## Testing (3)

| # | Question | Key Answer Points |
|---|----------|-------------------|
| 41 | Testing component behavior | RTL: render, fireEvent, userEvent, waitFor — test as user would |
| 42 | Mocking API calls | jest.mock, MSW (Mock Service Worker) for integration tests |
| 43 | Test coverage philosophy | Test behavior, not implementation. Don't test internal state, test rendered output |

## Security (2)

| # | Question | Key Answer Points |
|---|----------|-------------------|
| 44 | XSS prevention | React escapes by default, dangerouslySetInnerHTML warning, sanitize HTML |
| 45 | CSRF protection | SameSite cookies, CSRF tokens, CORS configuration |

## Architecture (5)

| # | Question | Key Answer Points |
|---|----------|-------------------|
| 46 | Project structure | Feature-based vs type-based, domain modules, shared components |
| 47 | Folder-by-feature | Each feature has components/hooks/api, colocated concerns |
| 48 | Error handling strategy | Error boundaries per section, global error handler, error logging service |
| 49 | API layer design | Axios instance, interceptors for auth/refresh, request/response transforms |
| 50 | Form handling approach | Custom useForm hook, validation library (Zod), controlled inputs |

# 4. THE MASTERY ROADMAP

```
Phase 1: Foundations (Week 1-2)
├── JS fundamentals (closures, promises, async/await, ES6+)
├── React core: components, JSX, props, state, events
├── Build: Todo App, Counter, Simple Form
└── Test: Explain Virtual DOM, reconciliation

Phase 2: Hooks Mastery (Week 3-4)
├── useState, useEffect, useContext, useRef
├── useReducer, useMemo, useCallback, custom hooks
├── Build: Shopping Cart, Multi-step Form, Dashboard
└── Test: Build 5 custom hooks from scratch

Phase 3: Advanced Patterns (Week 5-6)
├── Compound components, Context API, Error Boundaries
├── React.memo, code splitting, portals
├── Build: E-Commerce product configurator, Modal system
└── Test: Implement a compound component library

Phase 4: Ecosystem (Week 7-8)
├── React Router v6 (loaders, actions, protected routes)
├── State management (Zustand/Redux Toolkit)
├── Styling (Tailwind, CSS Modules, Styled Components)
├── Testing (Vitest, RTL, Cypress)
└── Build: Full E-Commerce app with auth, cart, checkout

Phase 5: Production Readiness (Week 9-10)
├── Performance optimization & profiling
├── Error monitoring (Sentry), analytics
├── CI/CD, Docker, deployment
├── Accessibility (a11y), i18n
├── TypeScript with React
└── Build: Banking dashboard with real-time updates

Phase 6: Senior Level (Week 11-12)
├── Custom architecture patterns
├── Monorepo (Nx/Turborepo)
├── Micro-frontends (Module Federation)
├── Server Components (Next.js App Router)
├── Contributing to React ecosystem
└── Teach others, mentor, write articles
```

# 5. MENTAL MODELS OF A SENIOR REACT ENGINEER

```
┌──────────────────────────────────────────────────┐
│          The Senior React Mental Model            │
├──────────────────────────────────────────────────┤
│                                                    │
│  1. "Component as a Function of State"            │
│     UI = f(state)                                  │
│     Given the same state, component renders the    │
│     same output. Side effects are separate.       │
│                                                    │
│  2. "Think in Effects, not Lifecycles"            │
│     useEffect synchronizes with external systems. │
│     Don't ask "when does this run?"               │
│     Ask "what is this synchronizing with?"        │
│                                                    │
│  3. "Data flows down, events flow up"             │
│     One-way data flow makes debugging trivial.    │
│     State lives high enough for all who need it.  │
│                                                    │
│  4. "Colocate state where it's needed"            │
│     Don't put everything in global state.         │
│     Start local, lift up only when necessary.     │
│                                                    │
│  5. "Profile before optimizing"                   │
│     Premature optimization is the root of all evil.│
│     Measure, identify bottleneck, optimize.       │
│                                                    │
│  6. "Composition over configuration"              │
│     Prefer composing small pieces over big        │
│     configuration objects.                        │
│                                                    │
│  7. "The Platform is Your Friend"                 │
│     Before adding a library, ask:                 │
│     "Can I do this with plain React + Browser?"   │
│                                                    │
└──────────────────────────────────────────────────┘
```

# 6. REACT ECOSYSTEM — WHAT A PRO KNOWS

| Category | Pro Tools | Why |
|----------|-----------|-----|
| Framework | Next.js, Remix | SSR, SSG, RSC, file-based routing, server actions |
| Styling | Tailwind CSS, CSS Modules | Utility-first, colocated, no runtime cost |
| Forms | React Hook Form + Zod | Performant, minimal re-renders, schema validation |
| Server State | TanStack Query (React Query) | Caching, deduplication, background refetch |
| Client State | Zustand, Jotai | Simple, performant, minimal boilerplate |
| Testing | Vitest, RTL, Playwright | Fast, user-centric, reliable E2E |
| Type Safety | TypeScript | Non-negotiable in production React |
| Build | Vite | Fast HMR, tree-shaking, optimized builds |
| Monorepo | Turborepo, Nx | Shared configs, incremental builds, caching |
| Quality | ESLint, Prettier, Husky | Consistent code, lint-staged, pre-commit hooks |

---

## FINAL ADVICE FROM A 20-YEAR VETERAN

1. **Understand the "Why"** — Don't just memorize syntax. Know why React chose one-way data flow, why the Virtual DOM exists, why hooks replaced classes.

2. **Read the Source** — React's source code is well-documented. Read `ReactDOM.createRoot`, `useState`, `useEffect` implementations. You'll understand React deeply.

3. **Build Real Things** — Theory without practice is useless. Build: E-Commerce site, Banking dashboard, Real-time chat, Social media feed, CMS.

4. **Break Things** — Try to create bugs, memory leaks, infinite loops. Understanding failure modes makes you better at preventing them.

5. **Teach Others** — You don't truly know something until you can teach it. Write articles, do code reviews, mentor juniors.

6. **Stay Current** — React evolves. Follow RFCs, read React blog, attend conferences (at least virtually).

7. **The Best Code is No Code** — Every line you don't write has zero bugs. Keep it simple. Don't over-abstract. Don't over-engineer.

8. **You Never Stop Learning** — The day you think you know everything is the day you become obsolete. Stay curious.

---

**You now have every concept, pattern, and real-world example needed to work at a senior level. Start building. Start breaking. Start teaching. 🚀**

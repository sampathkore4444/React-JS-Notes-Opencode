# Lesson 20: Error Boundaries — Never Let Your App Crash

## 1. CONCEPT — Catch Errors Gracefully

> **Real-world analogy:** Error Boundaries are like fire doors in a building. If a fire breaks out in one room (component crash), the fire door closes, containing the fire to that room. The rest of the building (other components) keeps working normally. Without fire doors, the entire building burns down (white screen of death).

**The problem:** A JavaScript error in one component can crash the **entire** React app. The user sees a white screen with nothing.

**Error Boundaries fix this:** They catch errors during rendering, in lifecycle methods, and in constructors — then render a **fallback UI** instead of crashing.

```jsx
// Without Error Boundary: ONE error crashes EVERYTHING
<App>
  <Header />
  <Sidebar />    ← This component throws error
  <MainContent /> ← Also gone! White screen!
  <Footer />
</App>

// With Error Boundary: isolated crash
<App>
  <Header />
  <ErrorBoundary fallback={<SidebarError />}>
    <Sidebar />    ← Catches error, shows fallback
  </ErrorBoundary>
  <MainContent /> ← Still works!
  <Footer />
</ErrorBoundary>
```

**Important limitation:** Error Boundaries only catch errors during:
- Rendering (JSX)
- Lifecycle methods
- Constructors

They do NOT catch errors in:
- Event handlers (use try-catch)
- Async code (use try-catch)
- Server-side rendering
- The error boundary itself

## 2. CODE ALONG — Create an Error Boundary

### Step 1: Create the component

`src/components/ErrorBoundary.jsx`:
\`\`\`jsx
import { Component } from 'react';

// Error Boundaries MUST be class components (React < 19 limitation)
class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false, error: null };  // track error state
  }

  // Called when a child throws during render — must return updated state
  static getDerivedStateFromError(error) {
    return { hasError: true, error };  // triggers re-render with error state
  }

  // Called after error is caught — for side effects (logging)
  componentDidCatch(error, errorInfo) {
    console.error('Error caught by boundary:', error);
    console.error('Component stack:', errorInfo?.componentStack);
    // In production: send to Sentry / LogRocket / your error monitor
    if (this.props.onError) {
      this.props.onError(error, errorInfo);
    }
  }

  render() {
    if (this.state.hasError) {
      // Show fallback UI instead of the crashed component
      if (this.props.fallback) {
        return this.props.fallback;  // custom fallback from parent
      }
      // Default fallback
      return (
        <div style={{
          padding: '20px',
          margin: '16px',
          border: '2px solid #fecaca',
          borderRadius: '8px',
          backgroundColor: '#fef2f2',
          color: '#dc2626',
        }}>
          <h3>Something went wrong</h3>
          <p style={{ fontSize: '14px', color: '#666' }}>
            {this.state.error?.message || 'An unexpected error occurred'}
          </p>
          <button
            onClick={() => {
              this.setState({ hasError: false, error: null });
              // Refresh might not fix it — but let user try
            }}
            style={{
              padding: '8px 16px',
              cursor: 'pointer',
              backgroundColor: '#dc2626',
              color: 'white',
              border: 'none',
              borderRadius: '4px',
              marginTop: '8px',
            }}
          >
            Try Again  {/* Resets error state, re-renders children */}
          </button>
        </div>
      );
    }
    return this.props.children;  // no error — render normally
  }
}

export default ErrorBoundary;
\`\`\`

### Step 2: Wrap strategic parts of TaskFlow

`src/App.jsx`:
```jsx
import ErrorBoundary from './components/ErrorBoundary';
import { useToast } from './components/Toast';
import BoardPage from './pages/BoardPage';

// Error logging function
function logError(error, errorInfo) {
  // In production: send to Sentry, LogRocket, etc.
  console.log('[ErrorBoundary]', error.name, error.message);
  // Could also call: fetch('/api/log-error', { method: 'POST', body: JSON.stringify({...}) })
}

function AppLayout() {
  const addToast = useToast();

  return (
    <div>
      {/* Top-level boundary — catches any unhandled errors */}
      <ErrorBoundary
        fallback={<FullPageError />}
        onError={logError}
      >
        <Header />

        {/* Specific boundary for the board — if Board crashes, Header/Footer still work */}
        <ErrorBoundary
          fallback={<BoardError />}
          onError={logError}
        >
          <BoardPage />
        </ErrorBoundary>

        {/* Specific boundary for each column */}
        {columnIds.map(id => (
          <ErrorBoundary
            key={id}
            fallback={<ColumnError name={columns[id].title} />}
            onError={logError}
          >
            <Column ... />
          </ErrorBoundary>
        ))}

        <Footer />
      </ErrorBoundary>
    </div>
  );
}

// Full page error (for catastrophic failures)
function FullPageError() {
  return (
    <div style={{
      display: 'flex', flexDirection: 'column',
      alignItems: 'center', justifyContent: 'center',
      height: '100vh', textAlign: 'center', padding: '20px',
    }}>
      <h1>💥</h1>
      <h2>Unexpected Error</h2>
      <p>Something went wrong. Please refresh the page.</p>
      <button onClick={() => window.location.reload()}
        style={{ padding: '12px 24px', cursor: 'pointer' }}>
        Refresh Page
      </button>
    </div>
  );
}

// Board-level error (less severe)
function BoardError() {
  const { onRetry } = useBoardContext();
  return (
    <div style={{ padding: '40px', textAlign: 'center' }}>
      <h3>Board Error</h3>
      <p>The board failed to load. Your other data is safe.</p>
      <button onClick={onRetry}>Reload Board</button>
    </div>
  );
}
```

### Step 3: Create a helper hook for event handler errors

Error Boundaries don't catch event handler errors. Use this pattern:

```jsx
// hooks/useAsyncAction.js
import { useState, useCallback } from 'react';

export function useAsyncAction(fn, onError) {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  const execute = useCallback(async (...args) => {
    try {
      setLoading(true);
      setError(null);
      return await fn(...args);
    } catch (err) {
      setError(err.message);
      if (onError) onError(err);
      throw err;
    } finally {
      setLoading(false);
    }
  }, [fn, onError]);

  return { execute, loading, error };
}

// Usage:
const { execute: handleDelete, loading } = useAsyncAction(
  async (id) => {
    await taskApi.delete(id);
    refetch();
  },
  (err) => showToast(err.message, 'error')
);

// In JSX:
<button onClick={() => handleDelete(task.id)} disabled={loading}>
  Delete
</button>
{/* If async fails, error state is set locally — component stays alive */}
```

## 3. THE ERROR HANDLING STRATEGY

```
┌────────────────────────────────────────────────────┐
│              Error Handling Strategy                │
├────────────────────────────────────────────────────┤
│                                                     │
│  Type                    Where to Catch             │
│                                                     │
│  Rendering error         Error Boundary              │
│  (component crashes)     (wraps component tree)      │
│                                                     │
│  Event handler error     try-catch in handler        │
│  (button click)          + local error state          │
│                                                     │
│  API error               try-catch in fetch           │
│  (network failure)       + error state in hook        │
│                                                     │
│  Async error             useAsyncAction hook          │
│  (promise rejection)     (wraps async functions)      │
│                                                     │
│  Unhandled error         window.onerror fallback     │
│  (everything else)       + full page refresh          │
│                                                     │
└────────────────────────────────────────────────────┘
```

## 4. EXERCISE

Add a `window.onerror` handler at the app level as a last-resort catch:

```jsx
// In main.jsx or a useEffect in App:
useEffect(() => {
  function onGlobalError(event) {
    console.error('Unhandled error:', event.error);
    // Show generic error UI
    const root = document.getElementById('root');
    if (root) {
      root.innerHTML = `
        <div style="text-align:center;padding:40px;font-family:sans-serif">
          <h1>Something went wrong</h1>
          <p>Please refresh the page.</p>
          <button onclick="location.reload()" style="padding:12px 24px;cursor:pointer">
            Refresh
          </button>
        </div>
      `;
    }
  }

  window.addEventListener('error', onGlobalError);
  return () => window.removeEventListener('error', onGlobalError);
}, []);
```

### CHALLENGE EXERCISE

Create a `withErrorBoundary` higher-order component (HOC) that wraps any component with an Error Boundary automatically. Add a "Report Error" button that logs the error details to the console and shows a toast.

**Solution:**
\`\`\`jsx
// src/components/withErrorBoundary.jsx
import ErrorBoundary from './ErrorBoundary';

export function withErrorBoundary(Component, fallback) {
  function WrappedComponent(props) {
    return (
      <ErrorBoundary
        fallback={fallback || (
          <div style={{
            padding: '20px', margin: '16px',
            border: '2px solid #fecaca', borderRadius: '8px',
            backgroundColor: '#fef2f2', textAlign: 'center',
          }}>
            <h3>⚠️ Component Error</h3>
            <p>This section encountered an error.</p>
            <button
              onClick={() => {
                navigator.clipboard?.writeText(
                  \`Error at \${new Date().toISOString()}\`
                );
                window.location.reload();
              }}
              style={{
                padding: '8px 16px', cursor: 'pointer',
                backgroundColor: '#dc2626', color: 'white',
                border: 'none', borderRadius: '4px',
              }}
            >
              Report & Reload
            </button>
          </div>
        )}
        onError={(error) => {
          console.group('🔴 Error Report');
          console.error('Message:', error.message);
          console.error('Stack:', error.stack);
          console.groupEnd();
          // Could send to server:
          // fetch('/api/log-error', { method: 'POST', body: JSON.stringify({ message: error.message }) });
        }}
      >
        <Component {...props} />
      </ErrorBoundary>
    );
  }
  return WrappedComponent;
}

// Usage:
const SafeBoardPage = withErrorBoundary(BoardPage);
const SafeColumn = withErrorBoundary(Column);
\`\`\`

## 5. COMMON MISTAKES

**Mistake: Wrapping everything in one Error Boundary**
```jsx
// ❌ One boundary wraps entire app — if ANYTHING crashes, EVERYTHING is replaced
<ErrorBoundary>
  <App />
</ErrorBoundary>

// ✅ Multiple boundaries, one per section — isolated failures
<ErrorBoundary><Header /></ErrorBoundary>
<ErrorBoundary><Sidebar /></ErrorBoundary>
<ErrorBoundary><Main /></ErrorBoundary>
```

**Mistake: Not handling async errors in event handlers**
```jsx
// ❌ Error in async handler crashes the app (not caught by Error Boundary)
async function handleClick() {
  throw new Error('Oops'); // This crashes the app!
}

// ✅ Catch it
async function handleClick() {
  try {
    await operation();
  } catch (err) {
    setError(err.message); // Component stays alive
  }
}
```

### 🐛 DEBUGGING WITH DEVTOOLS

To test Error Boundaries, temporarily throw an error in a component: `throw new Error('Test crash')` inside any component's render. Your Error Boundary should catch it and show the fallback. Without the boundary, you'll see the full React crash overlay (development) or a blank white page (production). Add error logging inside `componentDidCatch` and check the console for the error details.

### ⚙️ UNDER THE HOOD

Error Boundaries work via React's componentDidCatch lifecycle, available only in class components. When a child throws during render, React checks if any ancestor has `getDerivedStateFromError` — if yes, it calls it to set error state, then calls `componentDidCatch` for side effects. React prevents the crashed subtree from rendering any further. In React 19+, Error Boundaries will work with function components via an error boundary hook proposal.

### 🏭 PRODUCTION PATTERN

Always add Error Boundaries at strategic levels: one at the app root (catches everything, shows "Something went wrong" with a refresh button), and individual boundaries per major section (sidebar, main content, each widget). This way, a crash in the sidebar doesn't kill the entire app. Production apps also integrate Sentry or LogRocket in `componentDidCatch` to capture error logs.

### 💼 INTERVIEW READY

**Q:** What types of errors does Error Boundary NOT catch, and how do you handle them?
**A:** Error Boundaries do NOT catch: (1) Event handler errors — use try-catch in the handler. (2) Async code errors (setTimeout, Promise rejections) — use .catch() or try-catch in async functions. (3) SSR errors. (4) Errors in the error boundary itself. For event handler/async errors, wrap the operation in a try-catch and set local error state. For truly unhandled errors, add `window.addEventListener('error', handler)` as a last resort.

### 📝 MINI QUIZ

**1.** Must Error Boundaries be class components?
- A) Yes — they use componentDidCatch which only exists in class components
- B) No — they work with hooks
- C) Only in production
- D) Only in React 18+

**2.** What does `getDerivedStateFromError` do?
- A) Logs the error to console
- B) Updates state to show the fallback UI
- C) Refreshes the component
- D) Sends error to server

**3.** Where should you place Error Boundaries in a production app?
- A) Only at the root level
- B) At multiple levels (app root + per major section)
- C) Around every single component
- D) Error Boundaries are optional

**Answers:** 1. A 2. B 3. B

## 6. CHECKPOINT

> What types of errors does Error Boundary NOT catch?

**Answer:** Event handlers (use try-catch), async code (use try-catch), server-side rendering, and errors in the error boundary itself.

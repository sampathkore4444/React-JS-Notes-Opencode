# Lesson 10: Context API — Avoiding Prop Drilling

## 1. CONCEPT — The Prop Drilling Problem

> **Real-world analogy:** Prop drilling is like having a message passed from the CEO down through 10 layers of management to reach an employee. Each manager (intermediate component) has to receive and pass along the message even though it's not for them. Context is like posting the message on a company-wide bulletin board — the CEO posts it once, and any employee can read it directly without going through managers.

**Problem:** As your app grows, you pass props through components that don't need them:

```
App ──theme──→ Board ──theme──→ Column ──theme──→ TaskCard
                                           ↑ only this needs theme!
```

This is **prop drilling** — passing data through intermediate components.

**Solution:** Context lets you **skip levels**:

```
App ──────────→ ThemeContext.Provider ──────→ TaskCard (uses useContext)
  Board (doesn't see it)    Column (doesn't see it)  ↑ direct!
```

## 2. CODE ALONG — Add Theme Toggle

Create `src/contexts/ThemeContext.jsx`:
```jsx
import { createContext, useContext, useState, useCallback } from 'react';

// Step 1: Create the context object (like a bulletin board)
const ThemeContext = createContext();
// createContext() returns an object with Provider and Consumer properties

// Step 2: Create the Provider component (wraps the app)
export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState(() => {
    return localStorage.getItem('taskflow-theme') || 'light';
    // Initialize from localStorage or default to light
  });

  const toggleTheme = useCallback(() => {
    setTheme(prev => {
      const next = prev === 'light' ? 'dark' : 'light';  // flip the value
      localStorage.setItem('taskflow-theme', next);       // persist preference
      return next;
    });
  }, []);  // empty deps: function never changes (stable reference)

  // Theme values: different colors for light and dark modes
  const themeStyles = {
    light: {
      bg: '#ffffff', text: '#1e293b', card: '#ffffff',
      columnBg: '#f1f5f9', border: '#e2e8f0',
    },
    dark: {
      bg: '#0f172a', text: '#e2e8f0', card: '#1e293b',
      columnBg: '#1e293b', border: '#334155',
    },
  };

  // Value passed to all consumers — changes when theme changes
  const value = { theme, toggleTheme, styles: themeStyles[theme] };

  return (
    <ThemeContext.Provider value={value}>
      {children}  {/* render children — they can now useTheme() */}
    </ThemeContext.Provider>
  );
}

// Step 3: Consumer hook (the convenient way to read context)
export function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
    // Helpful error: developer forgot to wrap the app
  }
  return context;  // { theme, toggleTheme, styles }
}
```

Wrap your app in `src/main.jsx`:
```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';
import { ThemeProvider } from './contexts/ThemeContext.jsx';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <ThemeProvider>
      <App />
    </ThemeProvider>
  </StrictMode>
);
```

Now use the theme in any component:

**Header.jsx**:
```jsx
import { useTheme } from '../contexts/ThemeContext';

function Header({ totalTasks }) {
  const { theme, toggleTheme, styles } = useTheme();

  return (
    <header style={{
      borderBottom: `2px solid ${styles.border}`,
      paddingBottom: '16px',
      marginBottom: '20px',
    }}>
      <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
        <div>
          <h1 style={{ margin: 0, color: styles.text }}>📋 TaskFlow</h1>
          <p style={{ color: '#64748b', margin: '4px 0 0' }}>
            {totalTasks} total tasks
          </p>
        </div>
        <button onClick={toggleTheme} style={{
          padding: '8px 16px', cursor: 'pointer', fontSize: '16px',
          borderRadius: '6px', border: `1px solid ${styles.border}`,
          backgroundColor: styles.card, color: styles.text,
        }}>
          {theme === 'light' ? '🌙 Dark' : '☀️ Light'}
        </button>
      </div>
    </header>
  );
}
```

**Board.jsx** (applies theme to background):
```jsx
import { useTheme } from '../contexts/ThemeContext';
import Column from './Column';

function Board({ columns, onAddTask, onMoveTask, onDeleteTask }) {
  const { styles } = useTheme();
  const columnIds = Object.keys(columns);

  return (
    <div style={{
      display: 'grid',
      gridTemplateColumns: `repeat(${columnIds.length}, 1fr)`,
      gap: '16px',
      backgroundColor: styles.bg,
      borderRadius: '8px',
      padding: '16px',
      minHeight: '400px',
    }}>
      {columnIds.map(columnId => (
        <Column key={columnId} column={columns[columnId]}
          allColumns={columns}
          onAddTask={onAddTask} onMoveTask={onMoveTask}
          onDeleteTask={onDeleteTask}
        />
      ))}
    </div>
  );
}
```

Likewise update `Column.jsx`, `TaskCard.jsx`, `Footer.jsx` to use `useTheme()` for their colors.

## 3. WHEN TO USE CONTEXT (AND WHEN NOT TO)

| Use Context For | Don't Use Context For |
|----------------|----------------------|
| Theme | High-frequency updates (every keystroke) |
| Auth user | Data that only 2 components share (use props) |
| Locale/language | Form state (keep local) |
| Feature flags | API cache (use React Query/SWR) |

**Performance warning:** Every consumer re-renders when context value changes. For frequently changing data, split contexts or use state management libraries.

## 4. EXERCISE

Create a `BoardContext` that holds all board data and actions, so `Board`, `Column`, and `TaskCard` can access them without prop drilling.

**Solution:**
```jsx
// src/contexts/BoardContext.jsx
import { createContext, useContext } from 'react';
import { useBoard } from '../hooks/useBoard';

const BoardContext = createContext(null);

export function BoardProvider({ children }) {
  const board = useBoard();
  return (
    <BoardContext.Provider value={board}>
      {children}
    </BoardContext.Provider>
  );
}

export function useBoardContext() {
  const context = useContext(BoardContext);
  if (!context) throw new Error('useBoardContext must be inside BoardProvider');
  return context;
}

// Now App.jsx uses BoardProvider, and any descendant can call useBoardContext()
```

### CHALLENGE EXERCISE

Create a `UserContext` that stores the current user's name and avatar. Use it in the Header to show "Welcome, {name}" and the avatar image. Include a "Logout" button that resets the user to null and shows a login form instead.

**Solution:**
```jsx
// src/contexts/UserContext.jsx
import { createContext, useContext, useState } from 'react';

const UserContext = createContext(null);

export function UserProvider({ children }) {
  const [user, setUser] = useState(null);

  function login(name, avatar) {
    setUser({ name, avatar: avatar || 'https://via.placeholder.com/40' });
  }

  function logout() {
    setUser(null);
  }

  return (
    <UserContext.Provider value={{ user, login, logout }}>
      {children}
    </UserContext.Provider>
  );
}

export function useUser() {
  const context = useContext(UserContext);
  if (!context) throw new Error('useUser must be used within UserProvider');
  return context;
}
```

In App.jsx, wrap with UserProvider:
```jsx
<UserProvider>
  <ThemeProvider>
    <BoardProvider>
      <AppLayout />
    </BoardProvider>
  </ThemeProvider>
</UserProvider>
```

In Header, show user info or login button:
```jsx
import { useUser } from '../contexts/UserContext';

function Header() {
  const { user, login, logout } = useUser();

  return (
    <header style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'center' }}>
      <h1>📋 TaskFlow</h1>
      <div style={{ display: 'flex', alignItems: 'center', gap: '12px' }}>
        {user ? (
          <>
            <img src={user.avatar} alt="" style={{ width: '32px', height: '32px', borderRadius: '50%' }} />
            <span>Welcome, {user.name}</span>
            <button onClick={logout} style={{ padding: '4px 12px', cursor: 'pointer' }}>Logout</button>
          </>
        ) : (
          <button onClick={() => login('Alex', '')} style={{ padding: '4px 12px', cursor: 'pointer' }}>
            Login as Alex
          </button>
        )}
      </div>
    </header>
  );
}
```

## 5. COMMON MISTAKES

**Mistake: Value not memoized (causes all consumers to re-render)**
```jsx
// ❌ New object every render — all consumers re-render
<ThemeContext.Provider value={{ theme, toggleTheme }}>

// ✅ Memoize with useMemo
const value = useMemo(() => ({ theme, toggleTheme }), [theme, toggleTheme]);
<ThemeContext.Provider value={value}>
```

**Mistake: Too many things in one context**
```jsx
// ❌ Any change → all consumers re-render
<AppContext.Provider value={{ user, theme, tasks, notifications }}>

// ✅ Split by concern
<UserContext.Provider value={user}>
  <ThemeContext.Provider value={theme}>
    <TasksContext.Provider value={tasks}>
```

### 🐛 DEBUGGING WITH DEVTOOLS

React DevTools shows Context providers and consumers. In the Components tab, you'll see `<ThemeProvider>` wrapping the app. Click it — the right panel shows the context value (theme, toggleTheme, styles). Below the tree, a "Context" section shows which components consume this context. If a component isn't re-rendering when context changes, check it's wrapped inside the provider.

### ⚙️ UNDER THE HOOD

Context uses React's built-in subscription model. When the provider's `value` prop changes, React walks the component tree and re-renders every consumer — regardless of whether they use `React.memo`. This is why Context value should be memoized with `useMemo`, and frequently-changing data should use a separate state management solution. There's no bailout for context consumers.

### 🏭 PRODUCTION PATTERN

Use Context for truly global, infrequently-changing data: theme, auth user, locale, feature flags. For frequently-changing data that many components need, pair Context with `useReducer` or use Zustand/Redux. Context is NOT a performance tool — it's an architectural tool to avoid prop drilling.

### 💼 INTERVIEW READY

**Q:** What are the downsides of Context API for state management?
**A:** (1) All consumers re-render when the context value changes, regardless of which piece they read. (2) No way to bail out a child from re-rendering. (3) Debugging is harder than Redux DevTools. (4) Can't efficiently handle high-frequency updates. (5) Mixing contexts reduces re-renders but increases boilerplate. Context is best for low-frequency, globally-needed data like theme/auth.

### 📝 MINI QUIZ

**1.** What problem does Context solve?
- A) Slow renders
- B) Prop drilling
- C) State management
- D) Code organization

**2.** What happens to all context consumers when the context value changes?
- A) Only the nearest consumer re-renders
- B) All consumers re-render
- C) Nothing — they must call forceUpdate
- D) Only consumers that use React.memo re-render

**3.** When should you NOT use Context?
- A) Theme data
- B) Data that changes frequently (every keystroke)
- C) Auth user info
- D) Locale/language preference

**Answers:** 1. B 2. B 3. B

---

## 6. CHECKPOINT

> What problem does Context API solve, and what's its main performance caveat?

**Answer:** Solves prop drilling (passing props through intermediate components). Caveat: all consumers re-render when context value changes, so split contexts to minimize unnecessary re-renders.

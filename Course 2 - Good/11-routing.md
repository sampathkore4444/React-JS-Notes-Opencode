# Lesson 11: React Router — Multi-Page App

## 1. CONCEPT — Client-Side Routing

> **Real-world analogy:** Traditional websites are like a physical book — turning a page (clicking a link) throws away the current page and loads an entirely new one (full page refresh). An SPA with client-side routing is like a tablet app — tapping a tab instantly shows new content without closing the app. React Router is the "tab switching" mechanism for web apps.

Traditional websites: clicking a link → browser fetches new HTML → **full page refresh**.

SPA (Single Page Application): clicking a link → JavaScript intercepts → **no refresh** → React renders new component → URL changes.

React Router is the standard routing library.

```
User clicks /about
       ↓
React Router updates URL without refresh
       ↓
React renders <AboutPage /> component
       ↓
No network request for HTML (only data if needed)
```

## 2. CODE ALONG — Add Pages to TaskFlow

### Install React Router
```bash
npm install react-router-dom
```

### Create pages directory
```bash
mkdir src/pages
```

### Create page components

`src/pages/BoardPage.jsx` — moves current board code here:
```jsx
import Board from '../components/Board';
import { useBoardContext } from '../contexts/BoardContext';
import { useTheme } from '../contexts/ThemeContext';

function BoardPage() {
  const { columns, addTask, moveTask, deleteTask } = useBoardContext();
  const { styles } = useTheme();

  return (
    <div style={{ backgroundColor: styles.bg, minHeight: 'calc(100vh - 200px)' }}>
      <Board
        columns={columns}
        onAddTask={addTask}
        onMoveTask={moveTask}
        onDeleteTask={deleteTask}
      />
    </div>
  );
}
export default BoardPage;
```

`src/pages/AboutPage.jsx`:
```jsx
import { useTheme } from '../contexts/ThemeContext';

function AboutPage() {
  const { styles } = useTheme();

  return (
    <div style={{ color: styles.text, maxWidth: '600px', margin: '0 auto' }}>
      <h2>About TaskFlow</h2>
      <p>A project management app built with React to learn the library from scratch.</p>
      <p>Features:</p>
      <ul>
        <li>Kanban-style board with columns</li>
        <li>Add, move, and delete tasks</li>
        <li>Dark/light theme</li>
        <li>LocalStorage persistence</li>
        <li>Priority labels</li>
      </ul>
    </div>
  );
}
export default AboutPage;
```

`src/pages/SettingsPage.jsx`:
```jsx
import { useState } from 'react';
import { useTheme } from '../contexts/ThemeContext';

function SettingsPage() {
  const { styles, theme, toggleTheme } = useTheme();
  const [boardSize, setBoardSize] = useLocalStorage('board-size', 'medium');

  return (
    <div style={{ color: styles.text, maxWidth: '600px', margin: '0 auto' }}>
      <h2>Settings</h2>
      <div style={{ marginBottom: '20px' }}>
        <label style={{ display: 'block', marginBottom: '8px' }}>Theme</label>
        <button onClick={toggleTheme} style={{ padding: '8px 16px', cursor: 'pointer' }}>
          Current: {theme}
        </button>
      </div>
      <div>
        <label style={{ display: 'block', marginBottom: '8px' }}>Board Size</label>
        <select value={boardSize} onChange={e => setBoardSize(e.target.value)}
          style={{ padding: '8px', fontSize: '16px' }}>
          <option value="small">Small</option>
          <option value="medium">Medium</option>
          <option value="large">Large</option>
        </select>
      </div>
    </div>
  );
}
export default SettingsPage;
```

### Set up router in App.jsx

```jsx
import { BrowserRouter, Routes, Route, NavLink, Navigate } from 'react-router-dom';
import { ThemeProvider, useTheme } from './contexts/ThemeContext';
import { BoardProvider } from './contexts/BoardContext';
import BoardPage from './pages/BoardPage';
import AboutPage from './pages/AboutPage';
import SettingsPage from './pages/SettingsPage';
import Footer from './components/Footer';
import { useOnlineStatus } from './hooks/useOnlineStatus';

function AppLayout() {
  const { styles, theme, toggleTheme } = useTheme();
  const isOnline = useOnlineStatus();

  return (
    <div style={{
      backgroundColor: styles.bg, color: styles.text,
      minHeight: '100vh', padding: '20px',
      transition: 'background-color 0.3s, color 0.3s',
    }}>
      {!isOnline && (
        <div style={{ background: '#fef2f2', color: '#dc2626', padding: '8px 20px',
          textAlign: 'center', borderRadius: '8px', marginBottom: '16px' }}>
          🔴 You are offline
        </div>
      )}

      {/* Navigation */}
      <nav style={{
        display: 'flex', gap: '16px', marginBottom: '24px',
        borderBottom: `2px solid ${styles.border}`, paddingBottom: '12px',
      }}>
        <NavLink to="/board" style={({ isActive }) => ({
          color: isActive ? '#3b82f6' : styles.text,   // blue if current page
          fontWeight: isActive ? 'bold' : 'normal',      // bold if current page
          textDecoration: 'none',
        })}>Board</NavLink>
        <NavLink to="/about" style={({ isActive }) => ({
          color: isActive ? '#3b82f6' : styles.text,
          fontWeight: isActive ? 'bold' : 'normal',
          textDecoration: 'none',
        })}>About</NavLink>
        <NavLink to="/settings" style={({ isActive }) => ({
          color: isActive ? '#3b82f6' : styles.text,
          fontWeight: isActive ? 'bold' : 'normal',
          textDecoration: 'none',
        })}>Settings</NavLink>
        <div style={{ flex: 1 }} />
        <button onClick={toggleTheme} style={{
          cursor: 'pointer', padding: '4px 12px',
          border: `1px solid ${styles.border}`, borderRadius: '4px',
          background: styles.card, color: styles.text,
        }}>
          {theme === 'light' ? '🌙' : '☀️'}
        </button>
      </nav>

      {/* Routes container: only ONE route renders at a time */}
      <Routes>
        {/* path="/board" → http://localhost:5173/board renders BoardPage */}
        <Route path="/board" element={<BoardPage />} />
        <Route path="/about" element={<AboutPage />} />
        <Route path="/settings" element={<SettingsPage />} />
        {/* Catch-all: any unknown path redirects to /board */}
        <Route path="*" element={<Navigate to="/board" replace />} />
      </Routes>

      <Footer />
    </div>
  );
}

function App() {
  return (
    <BrowserRouter>
      <ThemeProvider>
        <BoardProvider>
          <AppLayout />
        </BoardProvider>
      </ThemeProvider>
    </BrowserRouter>
  );
}

export default App;
```

**Update `main.jsx`** — remove ThemeProvider (it's now inside App):
```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import App from './App.jsx';

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

**Test your routes:**
- `http://localhost:5173/board` — your Kanban board
- `http://localhost:5173/about` — About page
- `http://localhost:5173/settings` — Settings page
- `http://localhost:5173/anything` — Redirects to /board

Click the nav links — notice **no page refresh**. That's SPA routing.

## 3. KEY ROUTER CONCEPTS

| Concept | What It Does |
|---------|--------------|
| `<BrowserRouter>` | Wraps app, syncs UI with URL |
| `<Routes>` | Container for all routes |
| `<Route path="..." element={...} />` | Defines a URL → Component mapping |
| `<NavLink>` | Link with `isActive` styling |
| `<Navigate>` | Redirect programmatically |
| `useParams()` | Get URL params (`:id`) |
| `useNavigate()` | Programmatic navigation |

## 4. EXERCISE

Add a Task Detail page at `/task/:taskId` that shows a single task's full info. Create a link from each TaskCard to this page.

**Solution:**
```jsx
// src/pages/TaskDetailPage.jsx
import { useParams, useNavigate } from 'react-router-dom';
import { useBoardContext } from '../contexts/BoardContext';

function TaskDetailPage() {
  const { taskId } = useParams();
  const navigate = useNavigate();
  const { columns } = useBoardContext();

  // Find task across all columns
  let foundTask = null;
  let foundColumn = null;
  for (const col of Object.values(columns)) {
    const task = col.tasks.find(t => t.id === Number(taskId));
    if (task) { foundTask = task; foundColumn = col; break; }
  }

  if (!foundTask) return <h2>Task not found</h2>;

  return (
    <div>
      <button onClick={() => navigate(-1)}>← Back</button>
      <h2>{foundTask.title}</h2>
      <p>Column: {foundColumn.title}</p>
      <p>Priority: {foundTask.priority}</p>
      {foundTask.completedAt && <p>Completed: {foundTask.completedAt}</p>}
    </div>
  );
}
export default TaskDetailPage;

// Add route in AppLayout:
<Route path="/task/:taskId" element={<TaskDetailPage />} />

// In TaskCard, make title a NavLink:
<NavLink to={`/task/${task.id}`} style={{ textDecoration: 'none', color: 'inherit' }}>
  {task.title}
</NavLink>
```

### CHALLENGE EXERCISE

Add a "History" page at `/history` that shows a log of all task movements (which task moved from which column to which column, with timestamps). Add a link to it in the navigation. Store the history in a context or a Zustand store.

**Solution:**
```jsx
// src/pages/HistoryPage.jsx
import { useTheme } from '../contexts/ThemeContext';

function HistoryPage() {
  const { styles } = useTheme();
  const history = useBoardStore((state) => state.history) || [];

  return (
    <div style={{ color: styles.text, maxWidth: '600px', margin: '0 auto' }}>
      <h2>Activity History</h2>
      {history.length === 0 ? (
        <p>No activity yet.</p>
      ) : (
        <div>
          {history.map((entry, index) => (
            <div key={index} style={{
              padding: '12px', marginBottom: '8px',
              borderLeft: '3px solid #3b82f6',
              backgroundColor: styles.card,
              borderRadius: '4px',
            }}>
              <strong>{entry.taskTitle}</strong> moved from{' '}
              <em>{entry.fromColumn}</em> to <em>{entry.toColumn}</em>
              <div style={{ fontSize: '12px', color: '#94a3b8' }}>
                {new Date(entry.timestamp).toLocaleString()}
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}
export default HistoryPage;
```

Add route in AppLayout:
```jsx
<Route path="/history" element={<HistoryPage />} />
```

Add nav link:
```jsx
<NavLink to="/history" style={({ isActive }) => ({
  color: isActive ? '#3b82f6' : styles.text,
  fontWeight: isActive ? 'bold' : 'normal',
  textDecoration: 'none',
})}>History</NavLink>
```

Add to boardStore:
```jsx
// In moveTask action, add to history:
set((state) => ({
  ...state,
  history: [
    ...state.history,
    {
      taskTitle: task.title,
      fromColumn: state.columns[fromColumnId].title,
      toColumn: state.columns[toColumnId].title,
      timestamp: Date.now(),
    },
  ],
  // ... existing move logic
}));
```

## 5. COMMON MISTAKES

**Mistake: Router not at the top**
```jsx
// ❌ useNavigate/useParams won't work here
function App() {
  return (
    <>
      <SomeComponent />  {/* Can't use router hooks here */}
      <BrowserRouter>
        <Routes>...</Routes>
      </BrowserRouter>
    </>
  );
}

// ✅ Router must wrap everything that needs routing
function App() {
  return (
    <BrowserRouter>
      <SomeComponent />  {/* Now can use router hooks */}
      <Routes>...</Routes>
    </BrowserRouter>
  );
}
```

**Mistake: Missing `*` catch-all route for 404s**
```jsx
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/about" element={<About />} />
  {/* No catch-all — navigating to /xyz shows blank page */}
</Routes>
```

### 🐛 DEBUGGING WITH DEVTOOLS

In React DevTools, the component tree now shows `<BrowserRouter>` wrapping everything. Inside it, you'll see `<Routes>` which renders ONLY the active route's component. Click different nav links and watch the tree change — the previous route's component unmounts and the new one mounts. Use this to verify that route transitions are clean (no memory leaks from unmounted components).

### ⚙️ UNDER THE HOOD

React Router uses the browser's History API (`pushState`, `popState`) to change the URL without a full page reload. When a `<NavLink>` is clicked, React Router intercepts the event, calls `history.pushState()`, and renders the matching `<Route>` component. The old component unmounts (cleanup runs) and the new one mounts. This is why SPA navigation feels instant — no network request for HTML.

### 🏭 PRODUCTION PATTERN

React Router v6.4+ introduced "data routers" (`createBrowserRouter`) with built-in data loading via `loader` functions. This enables loading data BEFORE rendering the route (no "flash of loading"). Also use `<Suspense>` + `React.lazy()` for code splitting routes — each page loads only when navigated to, reducing initial bundle size.

### 💼 INTERVIEW READY

**Q:** What's the difference between `<a href>` and React Router's `<Link>`?
**A:** `<a href>` triggers a full browser page navigation — the current page unloads, all React state is lost, and the server sends new HTML. `<Link>` (or `<NavLink>`) prevents the default navigation, uses the History API to update the URL, and renders the new route component — preserving all React state and avoiding a full page reload.

### 📝 MINI QUIZ

**1.** What does `<Routes>` do?
- A) Defines individual URL paths
- B) Contains all `<Route>` definitions and renders one at a time
- C) Creates navigation links
- D) Manages browser history

**2.** What happens to the previous page's state when you navigate to a new route?
- A) It's preserved in memory
- B) The component unmounts (state is lost) unless lifted up
- C) It's saved to localStorage automatically
- D) React Router keeps it in cache

**3.** What does `<Navigate to="/board" replace />` do?
- A) Shows a confirmation dialog before navigating
- B) Redirects to /board, replacing the current history entry
- C) Loads /board in a new tab
- D) Prevents navigation to /board

**Answers:** 1. B 2. B 3. B

---

## 6. CHECKPOINT

> What's the difference between a traditional link (`<a href="...">`) and `<NavLink>` in React Router?

**Answer:** `<a>` causes a full page reload (loses React state). `<NavLink>` intercepts the click and renders the component without a page refresh, preserving all state.

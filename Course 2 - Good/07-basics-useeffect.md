# Lesson 7: useEffect — Talking to the Outside World

## 1. CONCEPT — Synchronizing With External Systems

> **Real-world analogy:** `useEffect` is like having a butler who watches for specific events and reacts automatically. When the mail arrives (a dependency changes), the butler picks it up (runs the effect). When you move houses (component unmounts), the butler cancels the newspaper subscription (runs cleanup). You don't check for mail yourself — the butler handles it.

`useState` handles **internal** component state. But what about things outside React?
- Fetching data from an API
- Reading/saving to localStorage
- Setting a timer
- Listening to keyboard events
- Connecting to a WebSocket

These are **side effects**. `useEffect` runs code **after** the component renders.

```jsx
import { useEffect } from 'react';

useEffect(() => {
  // This runs after the component renders
  console.log('Component mounted or dependencies changed');

  return () => {
    // This cleanup runs before the component unmounts
    // or before the effect runs again
    console.log('Cleaning up');
  };
}, [/* dependencies */]);
```

**Three variations:**

| Dependency Array | When It Runs | Common Use Case |
|-----------------|--------------|-----------------|
| `useEffect(fn)` | After EVERY render | Logging, analytics (use with caution) |
| `useEffect(fn, [])` | Only on mount (first render) | API fetch, event listeners |
| `useEffect(fn, [a, b])` | On mount + when `a` or `b` change | Syncing state to localStorage |

```jsx
// Variation 1: No dependency array — runs after EVERY render
useEffect(() => { document.title = 'Count: ' + count; });
// Use with CAUTION: often causes infinite loops

// Variation 2: Empty array — runs ONCE after first render (mount)
useEffect(() => { fetch('/api/data').then(r => r.json()).then(setData); }, []);
// Cleanup runs when component unmounts

// Variation 3: With dependencies — runs on mount AND when deps change
useEffect(() => { localStorage.setItem('tasks', JSON.stringify(tasks)); }, [tasks]);
// Only re-runs when `tasks` reference changes
```

## 2. CODE ALONG — Save Tasks to localStorage

Let's make tasks persist even after page refresh. Update `src/App.jsx`:

```jsx
import { useState, useEffect } from 'react';
import Header from './components/Header';
import TaskForm from './components/TaskForm';
import TaskList from './components/TaskList';
import Footer from './components/Footer';

function App() {
  // Lazy initializer: runs ONLY on first render to load saved data
  const [tasks, setTasks] = useState(() => {
    const saved = localStorage.getItem('taskflow-tasks');  // read from browser storage
    if (saved) {
      try { return JSON.parse(saved); }    // parse JSON string back to array
      catch { return []; }                 // if corrupted JSON, start fresh
    }
    // nothing saved → use defaults
    return [
      { id: 1, title: 'Learn React', done: false, priority: 'High' },
      { id: 2, title: 'Build TaskFlow', done: false, priority: 'Medium' },
      { id: 3, title: 'Deploy app', done: false, priority: 'Low' },
    ];
  });

  // Side effect: syncs state changes to localStorage automatically
  useEffect(() => {
    localStorage.setItem('taskflow-tasks', JSON.stringify(tasks));  // save as JSON string
    console.log('Saved', tasks.length, 'tasks to localStorage');
  }, [tasks]);  // re-run this effect whenever `tasks` array changes

  function addTask(taskData) {
    setTasks(prev => [
      ...prev,
      { id: Date.now(), ...taskData, done: false },
    ]);
  }

  function toggleTask(id) {
    setTasks(tasks.map(t =>
      t.id === id ? { ...t, done: !t.done } : t
    ));
  }

  function deleteTask(id) {
    setTasks(tasks.filter(t => t.id !== id));
  }

  function clearCompleted() {
    setTasks(tasks.filter(t => !t.done));
  }

  return (
    <div style={{ padding: '20px', maxWidth: '600px', margin: '0 auto' }}>
      <Header />
      <TaskForm onAddTask={addTask} />
      <TaskList tasks={tasks} onToggle={toggleTask} onDelete={deleteTask} />
      {tasks.some(t => t.done) && (
        <button onClick={clearCompleted} style={{
          padding: '8px 16px', marginTop: '12px', cursor: 'pointer',
          backgroundColor: '#dc3545', color: 'white', border: 'none', borderRadius: '4px',
        }}>
          Clear Completed
        </button>
      )}
      <Footer />
    </div>
  );
}

export default App;
```

**Test it:** Add tasks, refresh the page — they're still there!

## 3. CODE ALONG — Keyboard Shortcut

Let's add a "Press Escape to clear search" feature:

```jsx
// At the top of App.jsx
function App() {
  // ... existing state
  const [search, setSearch] = useState('');

  // Keyboard shortcut: Escape clears search
  useEffect(() => {
    function handleKeyDown(e) {
      if (e.key === 'Escape') {
        setSearch('');
        console.log('Search cleared via Escape');
      }
    }
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, []); // Empty array = only on mount/unmount

  // ... rest of component
```

## 4. WHY THE CLEANUP FUNCTION MATTERS

Without cleanup, if this component unmounts, the event listener still exists — causing a **memory leak**.

```jsx
// ❌ Without cleanup: event listener stays forever
useEffect(() => {
  window.addEventListener('keydown', handleKey);
  // Missing: return () => window.removeEventListener(...)
}, []);

// ✅ With cleanup: removed when component unmounts
useEffect(() => {
  window.addEventListener('keydown', handleKey);
  return () => window.removeEventListener('keydown', handleKey);
}, []);
```

**Always clean up:** event listeners, timers, subscriptions, WebSockets.

## 5. EXERCISE

Add a document title that shows the task count, e.g., "TaskFlow (3 remaining)".

**Solution:**
```jsx
// In App.jsx
useEffect(() => {
  const remaining = tasks.filter(t => !t.done).length;
  document.title = `TaskFlow (${remaining} remaining)`;
}, [tasks]);
```

### CHALLENGE EXERCISE

Add an "auto-save" indicator. Show a small "Saving..." text when the effect runs, then "Saved" after 500ms, then nothing after 2 seconds. Use a combination of `useEffect` and a separate state variable.

**Solution:**
```jsx
// Add state for save status
const [saveStatus, setSaveStatus] = useState('');

// Modify the localStorage effect:
useEffect(() => {
  setSaveStatus('Saving...');
  const timer = setTimeout(() => {
    localStorage.setItem('taskflow-tasks', JSON.stringify(tasks));
    setSaveStatus('Saved ✓');
  }, 300);  // small delay to show the saving state

  return () => {
    clearTimeout(timer);  // cleanup: cancel if tasks change again quickly
  };
}, [tasks]);

// Auto-hide "Saved" after 2 seconds
useEffect(() => {
  if (saveStatus === 'Saved ✓') {
    const timer = setTimeout(() => setSaveStatus(''), 2000);
    return () => clearTimeout(timer);
  }
}, [saveStatus]);

// Display in JSX:
<div style={{ textAlign: 'right', fontSize: '12px', color: '#888', minHeight: '16px' }}>
  {saveStatus}
</div>
```

## 6. COMMON MISTAKES

**Mistake: Infinite loop — updating state in useEffect without dependency**
```jsx
// ❌ INFINITE LOOP!
useEffect(() => {
  setCount(count + 1);
});  // Missing dependency array: runs after EVERY render
// setCount causes re-render → effect runs again → setCount → ...

// ✅ Fix: add dependency array
useEffect(() => {
  setCount(count + 1);
}, [count]);  // But still runs every time count changes
// Better:
useEffect(() => {
  setCount(prev => prev + 1);
}, []);  // Only once
```

**Mistake: async directly in useEffect**
```jsx
// ❌ Can't do this (useEffect expects a function, not a promise)
useEffect(async () => {
  const data = await fetchData();
  setData(data);
}, []);

// ✅ Define async function inside
useEffect(() => {
  async function loadData() {
    const data = await fetchData();
    setData(data);
  }
  loadData();
}, []);
```

**Mistake: Stale closure — using old values**
```jsx
const [count, setCount] = useState(0);

// ❌ count inside effect is always 0 (captured at mount)
useEffect(() => {
  const timer = setInterval(() => {
    setCount(count + 1); // count is always 0!
  }, 1000);
  return () => clearInterval(timer);
}, []);

// ✅ Fix: use functional state update
useEffect(() => {
  const timer = setInterval(() => {
    setCount(prev => prev + 1);
  }, 1000);
  return () => clearInterval(timer);
}, []);

// ✅ Or: add count to deps (but timer resets each second)
useEffect(() => {
  const timer = setInterval(() => {
    setCount(count + 1);
  }, 1000);
  return () => clearInterval(timer);
}, [count]);
```

### 🐛 DEBUGGING WITH DEVTOOLS

Add `console.log('Effect ran, tasks:', tasks.length)` inside your localStorage effect. Watch the console: the effect runs on mount, then runs every time tasks change. If it runs in an infinite loop, you're either missing deps or updating state in the effect that's in the dep array. The Profiler tab also shows when effects fire — look for the "Commits" timeline.

### ⚙️ UNDER THE HOOD

useEffect runs AFTER the browser paints (asynchronously). React defers effects to avoid blocking visual updates. The cleanup function from the previous render runs BEFORE the next effect — this ensures stale subscriptions are always cleaned up. React compares dependency arrays using `Object.is` — if any value differs, the effect re-runs.

### 🏭 PRODUCTION PATTERN

You'll rarely write raw `useEffect` for data fetching in production. Libraries like TanStack Query (React Query) and SWR wrap useEffect internally, adding caching, deduplication, background refetching, and loading state management. But understanding useEffect is essential for custom side effects like WebSocket connections, analytics, and scroll listeners.

### 💼 INTERVIEW READY

**Q:** What's the difference between `useEffect` and `useLayoutEffect`?
**A:** `useEffect` runs asynchronously after the browser paints (non-blocking). `useLayoutEffect` runs synchronously BEFORE the browser paints (blocking). Use `useLayoutEffect` when you need to read DOM measurements or make visual changes that the user should see immediately. Prefer `useEffect` by default — it doesn't block painting.

### 📝 MINI QUIZ

**1.** When does `useEffect(() => {...}, [])` run?
- A) After every render
- B) Only on the first mount
- C) When any state changes
- D) Every time the component updates

**2.** What does the cleanup function in useEffect do?
- A) Deletes the component from the DOM
- B) Runs before the effect re-runs or the component unmounts
- C) Clears all state
- D) Resets props to defaults

**3.** What causes an infinite loop in useEffect?
- A) Using an empty dependency array
- B) Updating state inside the effect that's listed in the dependencies
- C) Not returning a cleanup function
- D) Calling setState outside useEffect

**Answers:** 1. B 2. B 3. B

---

## 7. CHECKPOINT

> If a component unmounts but its useEffect set up an interval, what happens and how do you fix it?

**Answer:** The interval continues running (memory leak + bugs). Fix by returning a cleanup function that calls `clearInterval()`.

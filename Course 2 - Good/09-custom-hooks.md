# Lesson 9: Custom Hooks — Reusable Logic

## 1. CONCEPT — Extract Logic, Not Just UI

> **Real-world analogy:** A custom hook is like a recipe card. Instead of explaining "how to make dough" every time you bake, you write the recipe once on a card and reuse it whenever needed. The recipe card (custom hook) encapsulates the logic, and each baker (component) can follow it without rewriting the steps.

Components should handle **rendering**. Hooks should handle **logic**.

**Problem:** You're writing the same localStorage save pattern in multiple components.

**Solution:** Extract it into a custom hook:

```jsx
// Before: logic mixed with UI
function App() {
  const [tasks, setTasks] = useState(() => {
    const saved = localStorage.getItem('tasks');
    return saved ? JSON.parse(saved) : [];
  });
  useEffect(() => {
    localStorage.setItem('tasks', JSON.stringify(tasks));
  }, [tasks]);
  // ... UI rendering
}

// After: clean hook
function useLocalStorage(key, initialValue) {
  // Inside useLocalStorage:
  const [value, setValue] = useState(() => {     // lazy initialization: runs once
    const saved = localStorage.getItem(key);     // read from browser storage
    return saved ? JSON.parse(saved) : initialValue;  // parse or use default
  });
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));  // sync to storage on change
  }, [key, value]);  // re-run if key OR value changes

  return [value, setValue];  // same API as useState! drop-in replacement
}

// App is now cleaner
function App() {
  const [tasks, setTasks] = useLocalStorage('tasks', []);
  // ... UI only
}
```

**A custom hook is just a function that starts with `use` and calls other hooks.**

## 2. CODE ALONG — Refactor TaskFlow

Create `src/hooks/useLocalStorage.js`:
```jsx
import { useState, useEffect } from 'react';

export function useLocalStorage(key, initialValue) {
  // Lazy initialization: only runs once
  const [value, setValue] = useState(() => {
    try {
      const saved = localStorage.getItem(key);
      return saved ? JSON.parse(saved) : initialValue;
    } catch {
      return initialValue;
    }
  });

  // Save on change
  useEffect(() => {
    try {
      localStorage.setItem(key, JSON.stringify(value));
    } catch (e) {
      console.warn('Failed to save to localStorage:', e);
    }
  }, [key, value]);

  return [value, setValue];
}
```

Create `src/hooks/useBoard.js`:
```jsx
import { useCallback } from 'react';
import { useLocalStorage } from './useLocalStorage';

const defaultBoard = {
  'todo': {
    id: 'todo', title: 'To Do',
    tasks: [
      { id: 1, title: 'Learn React', priority: 'High' },
      { id: 2, title: 'Design components', priority: 'Medium' },
    ],
  },
  'in-progress': {
    id: 'in-progress', title: 'In Progress',
    tasks: [
      { id: 3, title: 'Build TaskFlow', priority: 'Medium' },
    ],
  },
  'done': {
    id: 'done', title: 'Done',
    tasks: [
      { id: 4, title: 'Set up project', priority: 'Low' },
    ],
  },
};

export function useBoard() {
  const [columns, setColumns] = useLocalStorage('taskflow-board', defaultBoard);

  const addTask = useCallback((columnId, taskData) => {
    setColumns(prev => ({
      ...prev,
      [columnId]: {
        ...prev[columnId],
        tasks: [...prev[columnId].tasks, { id: Date.now(), ...taskData }],
      },
    }));
  }, [setColumns]);

  const moveTask = useCallback((taskId, fromColumnId, toColumnId) => {
    setColumns(prev => {
      const task = prev[fromColumnId].tasks.find(t => t.id === taskId);
      if (!task) return prev;
      const updatedTask = toColumnId === 'done'
        ? { ...task, completedAt: new Date().toLocaleDateString() }
        : { ...task, completedAt: undefined };

      return {
        ...prev,
        [fromColumnId]: {
          ...prev[fromColumnId],
          tasks: prev[fromColumnId].tasks.filter(t => t.id !== taskId),
        },
        [toColumnId]: {
          ...prev[toColumnId],
          tasks: [...prev[toColumnId].tasks, updatedTask],
        },
      };
    });
  }, [setColumns]);

  const deleteTask = useCallback((columnId, taskId) => {
    setColumns(prev => ({
      ...prev,
      [columnId]: {
        ...prev[columnId],
        tasks: prev[columnId].tasks.filter(t => t.id !== taskId),
      },
    }));
  }, [setColumns]);

  // Derived data: computed from state, not stored separately
  const totalTasks = Object.values(columns)
    .reduce((sum, col) => sum + col.tasks.length, 0);
  // .reduce iterates over each column, adding its task count to the running sum

  return { columns, addTask, moveTask, deleteTask, totalTasks };
}
```

Now **App.jsx** becomes extremely clean:
```jsx
import Header from './components/Header';
import Board from './components/Board';
import Footer from './components/Footer';
import { useBoard } from './hooks/useBoard';

function App() {
  const { columns, addTask, moveTask, deleteTask, totalTasks } = useBoard();

  return (
    <div style={{ padding: '20px', maxWidth: '1200px', margin: '0 auto' }}>
      <Header totalTasks={totalTasks} />
      <Board
        columns={columns}
        onAddTask={addTask}
        onMoveTask={moveTask}
        onDeleteTask={deleteTask}
      />
      <Footer />
    </div>
  );
}

export default App;
```

## 3. WHY THIS MATTERS

Custom hooks are the most powerful pattern in React. They let you:
- **Reuse logic** across components (not just UI)
- **Separate concerns** (UI in components, logic in hooks)
- **Test logic independently** (hooks are just functions)
- **Compose hooks** (one hook can use another)

## 4. EXERCISE

Create a `useOnlineStatus` hook that returns `true/false` and shows a banner when the user goes offline.

**Solution:**
```jsx
// src/hooks/useOnlineStatus.js
import { useState, useEffect } from 'react';

export function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    function goOnline() { setIsOnline(true); }
    function goOffline() { setIsOnline(false); }

    window.addEventListener('online', goOnline);
    window.addEventListener('offline', goOffline);
    return () => {
      window.removeEventListener('online', goOnline);
      window.removeEventListener('offline', goOffline);
    };
  }, []);

  return isOnline;
}

// Use in App.jsx:
import { useOnlineStatus } from './hooks/useOnlineStatus';

function App() {
  const isOnline = useOnlineStatus();
  // ...
  return (
    <>
      {!isOnline && (
        <div style={{ background: '#fef2f2', color: '#dc2626', padding: '8px 20px', textAlign: 'center' }}>
          🔴 You are offline. Changes will sync when reconnected.
        </div>
      )}
      {/* ... rest of app */}
    </>
  );
}
```

### CHALLENGE EXERCISE

Create a `useDocumentTitle` hook that sets the document title and restores the previous title when the component unmounts.

**Solution:**
```jsx
// src/hooks/useDocumentTitle.js
import { useEffect } from 'react';

export function useDocumentTitle(title) {
  useEffect(() => {
    const previousTitle = document.title;  // save current title
    document.title = title;                // set new title

    return () => {
      document.title = previousTitle;      // restore on unmount
    };
  }, [title]);  // re-run if title changes
}

// Usage in any component:
function BoardPage() {
  useDocumentTitle('TaskFlow - Board');
  // ... component code
  // When BoardPage unmounts, title goes back to whatever it was before
}

// With task count:
function App() {
  const { totalTasks } = useBoard();
  useDocumentTitle(`TaskFlow (${totalTasks} tasks)`);
  // Browser tab shows: "TaskFlow (5 tasks)"
}
```

## 5. COMMON MISTAKES

**Mistake: Custom hook without `use` prefix**
```jsx
// ❌ React won't detect hook rules violations
function localStorage(key, initialValue) { ... }

// ✅ `use` prefix enables React's hook lint rules
function useLocalStorage(key, initialValue) { ... }
```

**Mistake: Calling hooks conditionally inside custom hook**
```jsx
// ❌ Violates rules of hooks
function useToggle(initial) {
  if (initial === undefined) {
    useState(false); // Conditional! Breaks ordering
  }
}

// ✅ Condition inside the hook call
function useToggle(initial) {
  const [on, setOn] = useState(initial ?? false);
}
```

### 🐛 DEBUGGING WITH DEVTOOLS

Custom hooks don't appear directly in React DevTools (they're not components). But the state they manage does — if your hook uses `useState`, that state appears under the component that calls the hook. To debug hooks, add `console.log` inside them or use the `useDebugValue` hook: `useDebugValue(value)` shows a label in DevTools next to the component that uses the hook.

### ⚙️ UNDER THE HOOD

A custom hook is just a function that starts with `use` and calls other hooks. React's hook rules still apply: hooks must be called at the top level, in the same order, in every render. The custom hook pattern extracts logic without violating these rules. Behind the scenes, the hooks inside run in the context of the calling component — they share the same hook state list.

### 🏭 PRODUCTION PATTERN

Custom hooks are the primary code reuse pattern in modern React. Check any production React codebase and you'll find dozens: `useAuth`, `useDebounce`, `useMediaQuery`, `useIntersectionObserver`. They isolate complex logic, make components readable, and are independently testable with `renderHook` from Testing Library.

### 💼 INTERVIEW READY

**Q:** What rules must a custom hook follow?
**A:** (1) The function name must start with "use" — this enables React's lint rules and DevTools. (2) It must call other hooks (useState, useEffect, etc.) at the top level — no conditions, loops, or nested functions. (3) It follows the same rules of hooks as any React component because the hooks execute in the calling component's context.

### 📝 MINI QUIZ

**1.** What makes a function a custom hook?
- A) It returns JSX
- B) Its name starts with "use" and it calls other hooks
- C) It's exported from a hooks folder
- D) It accepts props

**2.** Can a custom hook conditionally call useState?
- A) Yes, if wrapped in an if statement
- B) No — hooks must be called unconditionally at the top level
- C) Yes, with useConditional flag
- D) Only in development mode

**3.** What's the main benefit of extracting logic into a custom hook?
- A) Smaller bundle size
- B) Reusable logic across multiple components
- C) Automatic performance optimization
- D) Server-side rendering support

**Answers:** 1. B 2. B 3. B

---

## 6. CHECKPOINT

> What two rules must a custom hook follow?

**Answer:** (1) Function name must start with "use". (2) Must call other hooks at the top level (no conditions, loops).

# Lesson 21: Zustand — Simple Global State

## 1. CONCEPT — Context Without the Boilerplate

> **Real-world analogy:** Context API is like a company-wide PA system — when anyone uses it, EVERYONE hears it (all consumers re-render). Zustand is like individual Slack channels — you subscribe only to the channels you care about. If the "kitchen" channel posts a message, only the kitchen staff get notified, not the entire company.

**Context API problem:** Every consumer re-renders when context value changes, no matter what data they read. Splitting contexts helps but adds boilerplate.

**Zustand solution:** Components subscribe to **specific slices** of state. Changing one field only re-renders components that use that field.

```jsx
// Context: change theme → ALL theme consumers re-render
// Zustand: change user name → only components that USE name re-render
// Components that only use email do NOT re-render
```

**What makes Zustand special:**
- ~1KB bundle (tiny)
- No Provider wrapper needed
- Subscribe to specific fields (fine-grained re-renders)
- Works outside React (for redirects, service workers, etc.)
- Built-in persist middleware (localStorage)
- DevTools support

## 2. CODE ALONG — Replace Board Context with Zustand

### Install
```bash
npm install zustand
```

### Step 1: Create a store

`src/stores/boardStore.js`:
\`\`\`jsx
import { create } from 'zustand';
import { persist, devtools } from 'zustand/middleware';
import { taskApi } from '../services/api';

const initialColumns = {
  'todo': { id: 'todo', title: 'To Do', tasks: [] },
  'in-progress': { id: 'in-progress', title: 'In Progress', tasks: [] },
  'done': { id: 'done', title: 'Done', tasks: [] },
};

export const useBoardStore = create(
  devtools(
    persist(
      (set, get) => ({
        // State
        columns: initialColumns,      // board data
        loading: false,               // API loading state
        error: null,                  // API error state

        // Actions — use set() to update state
        addTask: (columnId, title, priority) =>
          set((state) => {            // set() accepts function for immutable updates
            const newTask = {
              id: Date.now(), title,
              priority: priority || 'Medium',
              createdAt: new Date().toISOString(),
            };
            return {
              columns: {              // must return NEW objects (immutability)
                ...state.columns,
                [columnId]: {
                  ...state.columns[columnId],
                  tasks: [...state.columns[columnId].tasks, newTask],
                },
              },
            };
          }),

        moveTask: (taskId, fromColumnId, toColumnId) =>
          set((state) => {
            const task = state.columns[fromColumnId].tasks.find(t => t.id === taskId);
            if (!task) return state;
            return {
              columns: {
                ...state.columns,
                [fromColumnId]: {
                  ...state.columns[fromColumnId],
                  tasks: state.columns[fromColumnId].tasks.filter(t => t.id !== taskId),
                },
                [toColumnId]: {
                  ...state.columns[toColumnId],
                  tasks: [...state.columns[toColumnId].tasks, task],
                },
              },
            };
          }),

        deleteTask: (columnId, taskId) =>
          set((state) => ({
            columns: {
              ...state.columns,
              [columnId]: {
                ...state.columns[columnId],
                tasks: state.columns[columnId].tasks.filter(t => t.id !== taskId),
              },
            },
          })),

        // Async action — Zustand supports async/await directly
        fetchTasks: async () => {
          set({ loading: true, error: null });    // start loading
          try {
            const tasks = await taskApi.getAll();
            set({
              columns: {
                'todo': { id: 'todo', title: 'To Do', tasks: tasks.filter(t => !t.done) },
                'in-progress': { id: 'in-progress', title: 'In Progress', tasks: [] },
                'done': { id: 'done', title: 'Done', tasks: tasks.filter(t => t.done) },
              },
              loading: false,                       // success
            });
          } catch (err) {
            set({ error: err.message, loading: false });  // error
          }
        },

        // Derived value — uses get() to read current state
        get totalTasks() {
          return Object.values(get().columns)
            .reduce((sum, col) => sum + col.tasks.length, 0);
        },
      }),
      {
        name: 'taskflow-board',     // persist middleware: localStorage key
        partialize: (state) => ({ columns: state.columns }), // Only persist columns
      }
    ),
    { name: 'BoardStore' }          // devtools middleware: debug name
  )
);

// Selector hooks — fine-grained subscriptions
export const useColumns = () => useBoardStore((state) => state.columns);
export const useLoading = () => useBoardStore((state) => state.loading);
// Components using useColumns() only re-render when columns changes
// Components using useLoading() only re-render when loading changes
export const useError = () => useBoardStore((state) => state.error);
export const useTotalTasks = () => useBoardStore((state) => state.totalTasks);
\`\`\`

### Step 2: Use in components — no Provider needed!

`src/pages/BoardPage.jsx`:
```jsx
import { useEffect } from 'react';
import { useColumns, useLoading, useError } from '../stores/boardStore';
import Board from '../components/Board';
import { useTheme } from '../contexts/ThemeContext';

function BoardPage() {
  const columns = useColumns();
  const loading = useLoading();
  const error = useError();
  const { styles } = useTheme();

  // Only runs once — fetch data
  useEffect(() => { useBoardStore.getState().fetchTasks(); }, []);

  if (loading) return <div style={{ textAlign: 'center', padding: '40px' }}>Loading board...</div>;
  if (error) return <div style={{ color: 'red', padding: '20px' }}>Error: {error}</div>;

  return (
    <div style={{ backgroundColor: styles.bg, minHeight: 'calc(100vh - 200px)' }}>
      <Board columns={columns} />
    </div>
  );
}
```

`src/components/Column.jsx`:
```jsx
import { useState } from 'react';
import { useBoardStore } from '../stores/boardStore';

function Column({ columnId }) {
  // Only subscribe to THIS column's data — fine-grained!
  const column = useBoardStore((state) => state.columns[columnId]);
  const addTask = useBoardStore((state) => state.addTask);
  const deleteTask = useBoardStore((state) => state.deleteTask);
  const moveTask = useBoardStore((state) => state.moveTask);

  // ... rest of component
}
```

`src/components/Header.jsx`:
```jsx
import { useTotalTasks } from '../stores/boardStore';

function Header({ toggleTheme }) {
  // Only subscribes to totalTasks — adding a task re-renders this
  // but it does NOT re-render when a different unrelated field changes
  const totalTasks = useTotalTasks();

  return (
    <header>
      <h1>📋 TaskFlow</h1>
      <p>{totalTasks} total tasks</p>
      <button onClick={toggleTheme}>Toggle Theme</button>
    </header>
  );
}
```

### Step 3: Compare Context vs Zustand re-renders

Open React DevTools Profiler:
- **Context:** Toggle theme → ALL Board components re-render (even though theme is the only change)
- **Zustand:** Toggle theme → ONLY Header re-renders (because it's the only component subscribing to theme)

## 3. ZUSTAND VS CONTEXT VS REDUX

| Feature | Context | Zustand | Redux Toolkit |
|---------|---------|---------|--------------|
| Bundle size | 0kb (built-in) | ~1kb | ~12kb |
| Provider needed | Yes | No | Yes |
| Fine-grained subs | No (all re-render) | Yes | Yes |
| Boilerplate | Medium | Low | Medium |
| DevTools | No | Yes (via middleware) | Yes |
| Async actions | Manual | Built-in | createAsyncThunk |
| Persistence | Manual | Built-in middleware | Manual |

**When to use Zustand:** Medium-complexity apps. You need global state but Redux feels like overkill.

## 4. EXERCISE

Create a `useNotificationsStore` in Zustand that holds toast notifications. Components can call `addNotification('Task moved')` from anywhere (even outside React).

**Solution:**
```jsx
import { create } from 'zustand';

export const useNotificationStore = create((set) => ({
  notifications: [],

  addNotification: (message, type = 'info') => {
    const id = Date.now();
    set((state) => ({
      notifications: [...state.notifications, { id, message, type }],
    }));
    // Auto-dismiss after 3s
    setTimeout(() => {
      set((state) => ({
        notifications: state.notifications.filter(n => n.id !== id),
      }));
    }, 3000);
  },

  clearNotifications: () => set({ notifications: [] }),
}));

// Usage anywhere:
import { useNotificationStore } from '../stores/notificationStore';

// In component:
const addNotification = useNotificationStore(state => state.addNotification);
addNotification('Task created!', 'success');

// Outside React (e.g., in api.js):
// import { useNotificationStore } from '../stores/notificationStore';
// But Zustand works outside React:
// useNotificationStore.getState().addNotification('Error!', 'error');
```

### CHALLENGE EXERCISE

Create a Zustand store for the theme (replacing the Context API ThemeProvider). Compare the re-renders using React DevTools.

**Solution:**
\`\`\`jsx
// src/stores/themeStore.js
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useThemeStore = create(
  persist(
    (set, get) => ({
      theme: 'light',

      toggleTheme: () => set((state) => ({
        theme: state.theme === 'light' ? 'dark' : 'light',
      })),

      setTheme: (theme) => set({ theme }),

      // Derived styles
      get styles() {
        const theme = get().theme;
        return theme === 'light'
          ? { bg: '#ffffff', text: '#1e293b', card: '#ffffff', columnBg: '#f1f5f9', border: '#e2e8f0' }
          : { bg: '#0f172a', text: '#e2e8f0', card: '#1e293b', columnBg: '#1e293b', border: '#334155' };
      },
    }),
    { name: 'taskflow-theme' }
  )
);

// Selectors for fine-grained subscriptions
export const useTheme = () => useThemeStore((s) => s.theme);
export const useStyles = () => useThemeStore((s) => s.styles);
export const useToggleTheme = () => useThemeStore((s) => s.toggleTheme);

// Usage in Header:
function Header() {
  const theme = useTheme();           // only re-renders when theme changes
  const styles = useStyles();         // only re-renders when styles change
  const toggleTheme = useToggleTheme();

  return (
    <header style={{ color: styles.text }}>
      <h1>📋 TaskFlow</h1>
      <button onClick={toggleTheme}>
        {theme === 'light' ? '🌙 Dark' : '☀️ Light'}
      </button>
    </header>
  );
}

// No Provider needed — just use the hooks directly!
\`\`\`

## 5. COMMON MISTAKES

**Mistake: Subscribing to the entire store**
```jsx
// ❌ Re-renders on ANY state change
const state = useBoardStore();

// ✅ Subscribe only to what you need
const columns = useBoardStore((state) => state.columns);
const addTask = useBoardStore((state) => state.addTask);
```

**Mistake: Mutating state directly in Zustand**
```jsx
// ❌ Zustand detects changes via reference equality
const handleClick = () => {
  const state = useBoardStore.getState();
  state.columns.todo.tasks.push(newTask); // Mutation!
  // React won't re-render!
};

// ✅ Always use set() or return new objects
const handleClick = () => {
  useBoardStore.getState().addTask('todo', 'New task');
};
```

### 🐛 DEBUGGING WITH DEVTOOLS

Zustand's devtools middleware integrates with Redux DevTools. Open Redux DevTools extension → you'll see each `set()` call as a dispatched action with a diff of what changed. You can time-travel by clicking previous states. If a component isn't re-rendering when a specific field changes, check which selector it uses — it might be subscribing to the wrong slice.

### ⚙️ UNDER THE HOOD

Zustand uses a simple publish-subscribe pattern under the hood. The store is a plain JavaScript object with methods. `create()` returns a hook that's also a store: `useStore.getState()` reads without subscribing, `useStore.subscribe(listener)` watches for changes. The selector function determines which slice each component subscribes to — Zustand uses `Object.is` to compare previous and current selector output, notifying only when the selected value changes.

### 🏭 PRODUCTION PATTERN

Zustand is the most popular lightweight state manager in the React ecosystem (2025). Its ~1KB size, zero boilerplate, and ability to work outside React make it ideal for medium-complexity apps. For very large apps with complex data, Redux Toolkit with RTK Query provides more structure. For simple apps, React Context + useState is sufficient. Choose the simplest tool that fits.

### 💼 INTERVIEW READY

**Q:** Compare Zustand, Context API, and Redux Toolkit. When would you use each?
**A:** Context: zero dependencies, good for low-frequency global data (theme, auth). Re-renders all consumers on any change. Zustand: ~1KB, selector-based subscriptions (fine-grained re-renders), works outside React, minimal boilerplate. Redux Toolkit: ~12KB, structured with slices and thunks, best for large apps with complex state logic, async flows, and team conventions. Choose Context for 1-2 global values. Choose Zustand for medium complexity. Choose RTK for enterprise-scale apps.

### 📝 MINI QUIZ

**1.** How does Zustand prevent unnecessary re-renders?
- A) All components re-render on every change
- B) Components subscribe to specific slices via selectors
- C) It uses React.memo internally
- D) It batches all updates

**2.** Does Zustand require a Provider wrapper?
- A) Yes, like Context
- B) No — the store is accessed directly anywhere
- C) Only for persistence
- D) Only in development

**3.** How do you persist Zustand state to localStorage?
- A) Manual useEffect
- B) Built-in `persist` middleware
- C) External library
- D) It's not possible

**Answers:** 1. B 2. B 3. B

## 6. CHECKPOINT

> How does Zustand know which components to re-render when state changes?

**Answer:** Each component subscribes via a selector function (`useBoardStore(s => s.columns)`). Zustand tracks which selectors are used and only notifies components whose selected values change (reference comparison).

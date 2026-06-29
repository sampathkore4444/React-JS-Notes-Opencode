# Lesson 15: React.memo & useMemo — Performance Optimization

## 1. CONCEPT — When Components Re-render

> **Real-world analogy:** `React.memo` is like a security guard at an office building who checks IDs before letting people in. Without the guard, everyone entering the building (parent re-render) forces every office (child component) to prepare for a visitor (re-render). With the guard (`React.memo`), only people with new badges (changed props) get through. Everyone else stays focused on their work.

**By default, React re-renders a component when its parent re-renders.** Not just when its props change.

```
App re-renders (e.g., theme toggles)
  ├── Header re-renders ← unnecessary: theme changed, but tasks didn't!
  ├── Board re-renders  ← unnecessary
  │   ├── Column re-renders ← unnecessary
  │   │   └── TaskCard re-renders ← unnecessary times 20!
  │   │   └── TaskCard re-renders
```

**React.memo prevents this.** It says: "Only re-render if props actually changed."

```jsx
// Without React.memo: re-renders EVERY TIME parent re-renders
function TaskCard({ task }) { ... }  // no protection

// With React.memo: re-renders ONLY when props change
const TaskCard = React.memo(function TaskCard({ task }) { ... });
// React.memo performs a SHALLOW comparison of props
// If task reference is the same, skip re-render
```

**useMemo** is the same idea but for **values** instead of components:

```jsx
// Without useMemo: recalculates on EVERY render
const sortedTasks = [...tasks].sort(sortFn);  // O(n log n) each render!

// With useMemo: recalculates ONLY when dependencies change
const sortedTasks = useMemo(
  () => [...tasks].sort(sortFn),  // expensive calculation
  [tasks, sortFn]                 // only recalculate if tasks or sortFn change
);
```

## 2. CODE ALONG — Optimize TaskFlow

### Step 1: Memoize TaskCard

`src/components/TaskCard.jsx`:
```jsx
import { memo } from 'react';

const TaskCard = memo(function TaskCard({ task, columnId, onMoveTask, onDeleteTask }) {
  console.log(`Rendering: ${task.title}`); // Check console to see when it renders

  const priorityColors = {
    High: { bg: '#fee2e2', text: '#dc2626' },
    Medium: { bg: '#fef3c7', text: '#d97706' },
    Low: { bg: '#dcfce7', text: '#16a34a' },
  };

  const colors = priorityColors[task.priority] || priorityColors.Medium;

  return (
    <div style={{
      backgroundColor: 'white',
      padding: '10px',
      marginBottom: '8px',
      borderRadius: '6px',
      boxShadow: '0 1px 3px rgba(0,0,0,0.1)',
    }}>
      <div>{task.title}</div>
      <span style={{
        display: 'inline-block', fontSize: '11px', padding: '2px 6px',
        borderRadius: '4px', marginTop: '6px',
        backgroundColor: colors.bg, color: colors.text,
      }}>
        {task.priority}
      </span>
    </div>
  );
});

export default TaskCard;
```

### Step 2: Memoize Column

`src/components/Column.jsx`:
```jsx
import { memo } from 'react';
import TaskCard from './TaskCard';

const Column = memo(function Column({ column, allColumns, onAddTask, onMoveTask, onDeleteTask }) {
  // ... same content as before
});

export default Column;
```

### Step 3: Test the difference

Add this to Board.jsx and check the console:
```jsx
// Before memo: type in ANY input → ALL TaskCards log "Rendering..."
// After memo: type in input → only that Column re-renders, TaskCards inside it don't!
```

### Step 4: useMemo for expensive calculations

In `src/hooks/useBoard.js`:
```jsx
// Expensive: counting across all columns
const totalTasks = useMemo(
  () => Object.values(columns).reduce((sum, col) => sum + col.tasks.length, 0),
  [columns]
);

// Expensive: sorting tasks by priority
const sortedColumns = useMemo(() => {
  return Object.entries(columns).map(([id, col]) => ({
    ...col,
    tasks: [...col.tasks].sort((a, b) => {
      const order = { High: 0, Medium: 1, Low: 2 };
      return order[a.priority] - order[b.priority];
    }),
  }));
}, [columns]);
```

### Step 5: useCallback for stable function references

Every time `Board` re-renders, it creates new `onAddTask`, `onMoveTask` etc. functions. These new references **defeat React.memo** because the props "changed" (new function reference).

Fix by already wrapping in `useCallback` in the hook (we did this in lesson 9). Verify your `useBoard` hook has `useCallback`:

```jsx
// If we forgot useCallback:
function useBoard() {
  const [columns, dispatch] = useReducer(boardReducer, initialColumns);

  // ❌ Without useCallback: NEW function EVERY render
  function addTask(columnId, title, priority) {
    dispatch({ type: ACTIONS.ADD_TASK, payload: { columnId, title, priority } });
  }
  // Each render creates a new function reference → React.memo sees "props changed"!

  // ✅ With useCallback: SAME function reference across renders
  const addTask = useCallback((columnId, title, priority) => {
    dispatch({ type: ACTIONS.ADD_TASK, payload: { columnId, title, priority } });
  }, []);  // empty deps: dispatch is stable, function never changes
  // React.memo now sees the same function prop → can safely skip re-render
}
```

## 3. THE PERFORMANCE RULES

| Technique | When to Use | When NOT to Use |
|-----------|-------------|----------------|
| `React.memo` | Child renders often with same props | Props are always different (e.g., random values) |
| `useMemo` | Expensive calculation (>1ms) | Simple math (a + b) |
| `useCallback` | Passing function to memoized child | Passing function to HTML element (`onClick`) |

**The golden rule: Profile first. Don't optimize blindly.**

```jsx
// How to profile:
// 1. Open React DevTools → Profiler tab
// 2. Click record → interact with your app → stop
// 3. Look for components that rendered unnecessarily
// 4. Add React.memo/useMemo only where it helps
```

## 4. EXERCISE

Without looking at the code, add `React.memo` to your Board and Column components. Then add console logs to verify they only re-render when their actual data changes.

**Check your work:**
```jsx
// Each component should:
// 1. Be wrapped with memo()
// 2. Accept stable callback props (from useCallback)
// 3. Not re-render when sibling components update

// Test: toggle theme → Board should NOT re-render (it doesn't use theme)
// Test: add a task → only the column that received the task should re-render
// Test: type in search → only the component with the input should re-render
```

### CHALLENGE EXERCISE

Use React Profiler to identify the slowest component. Then add `React.memo` to it and verify the improvement using console logs. Create a "Task Counter" badge in the Header that only updates when the total task count changes.

**Solution:**

Open DevTools → Components → Profiler tab. Click record, add a task, stop recording.

Now optimize the Header component:
```jsx
import { memo } from 'react';

const Header = memo(function Header({ totalTasks, toggleTheme }) {
  console.log('Header rendered');  // Check: should only render when totalTasks changes

  return (
    <header>
      <h1>📋 TaskFlow</h1>
      <div style={{ display: 'flex', gap: '12px', alignItems: 'center' }}>
        <span style={{
          backgroundColor: '#3b82f6', color: 'white',
          padding: '2px 10px', borderRadius: '12px', fontSize: '14px',
        }}>
          {totalTasks} tasks
        </span>
        <button onClick={toggleTheme}>Toggle Theme</button>
      </div>
    </header>
  );
});
```

To verify:
1. Open console
2. Toggle theme — Header should NOT re-render (if totalTasks hasn't changed)
3. Add a task — Header SHOULD re-render (totalTasks changed)
4. Before memo: both toggle and add cause re-render. After memo: only add does.

## 5. COMMON MISTAKES

**Mistake: Memoizing everything (waste of memory)**
```jsx
// ❌ Over-optimization — more overhead than benefit
const five = useMemo(() => 5, []); // 5 is a primitive, always stable
const add = useCallback((a, b) => a + b, []); // Creating function is cheap
```

**Mistake: Not memoizing context value**
```jsx
// ❌ New object every render → all consumers re-render
<ThemeContext.Provider value={{ theme, toggleTheme }}>
  {children}
</ThemeContext.Provider>

// ✅ Memoize the value object
const value = useMemo(() => ({ theme, toggleTheme }), [theme, toggleTheme]);
<ThemeContext.Provider value={value}>
```

**Mistake: Object/array props defeat React.memo**
```jsx
// ❌ New style object every render → React.memo sees "changed" props
<TaskCard task={task} style={{ marginTop: '8px' }} />

// ✅ Extract to constant or use useMemo
const cardStyle = useMemo(() => ({ marginTop: '8px' }), []);
<TaskCard task={task} style={cardStyle} />
```

### 🐛 DEBUGGING WITH DEVTOOLS

React DevTools Profiler is your best friend for performance. Click record, interact with your app (toggle theme, add a task), stop recording. The flamegraph shows which components re-rendered and how long each took. Components that re-rendered without prop changes are optimization candidates. Green = skipped (React.memo worked). Yellow/red = rendered. This data-driven approach beats guessing.

### ⚙️ UNDER THE HOOD

React.memo is a higher-order component that implements `shouldComponentUpdate` under the hood. It stores the previous props and does a shallow comparison with the new props. If all props are the same (by reference), it skips rendering entirely — reusing the previous virtual DOM subtree. This is called "bailing out of rendering." But: if ANY prop is a new reference (new object, new function), the bailout fails.

### 🏭 PRODUCTION PATTERN

Performance optimization rule: DON'T optimize until you've profiled. Adding React.memo everywhere adds comparison overhead. `useMemo` and `useCallback` have their own costs (memory for cached values, garbage collection pressure). Profile first, identify the bottleneck, then optimize only that. The vast majority of React apps don't need optimization until they hit thousands of components.

### 💼 INTERVIEW READY

**Q:** Explain the relationship between React.memo, useMemo, and useCallback.
**A:** All three prevent unnecessary work by caching. `React.memo` caches the component's rendered output (skip re-render if props unchanged). `useMemo` caches a computed value: `const sorted = useMemo(() => sort(arr), [arr])`. `useCallback` caches a function reference: `const fn = useCallback(() => doSomething(a), [a])`. useCallback(fn, deps) is equivalent to useMemo(() => fn, deps). Each addresses a different kind of waste.

### 📝 MINI QUIZ

**1.** What does React.memo do?
- A) Memorizes the last render output
- B) Skips re-rendering if props haven't changed (shallow comparison)
- C) Caches function results
- D) Prevents state updates

**2.** If a parent re-renders, does a React.memo child always re-render too?
- A) Yes, always
- B) No — only if its props actually changed
- C) Only in development mode
- D) Only if the child has state

**3.** When is useMemo appropriate?
- A) For every variable in the component
- B) Only for expensive calculations (sorting, filtering large arrays)
- C) For primitive values like numbers
- D) Only for JSX elements

**Answers:** 1. B 2. B 3. B

---

## 6. CHECKPOINT

> If a parent component re-renders, does a `React.memo` child always re-render too?

**Answer:** No. `React.memo` compares new props to previous props (shallow comparison). If they're the same (same references for objects/functions, same values for primitives), the child skips rendering.

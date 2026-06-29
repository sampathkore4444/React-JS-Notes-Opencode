# Lesson 14: useRef — Beyond State

## 1. CONCEPT — Values That Don't Trigger Re-renders

> **Real-world analogy:** `useRef` is like a notepad you keep on your desk. You can jot down any information (timer ID, previous value, DOM element), read it anytime, and change it freely. But unlike a digital display (state), scribbling on the notepad doesn't trigger any alarms or updates — it's just for your own reference. The screen (UI) only shows what's in state, not what's on your notepad.

`useState` causes re-renders when changed. `useRef` persists values without causing re-renders.

**Three main use cases:**

| Use Case | Example |
|----------|---------|
| **DOM access** | Focus input, measure element size, scroll position |
| **Mutable values** | Timer IDs, animation frames, debounce timers |
| **Stable references** | Store callbacks to avoid stale closures |

```jsx
const ref = useRef(initialValue);
// ref.current = initialValue
// ref.current = newValue  ← doesn't trigger re-render!
// ref.current.someMethod() ← direct DOM manipulation
```

## 2. CODE ALONG — 3 Practical Examples in TaskFlow

### Example 1: Auto-focus the input when adding a task

Update `src/components/Column.jsx` — when the "Add Task" form appears, focus the input automatically:

```jsx
import { useRef, useEffect } from 'react';

function Column({ column, ... }) {
  const inputRef = useRef(null);    // Step 1: Create ref — persists across renders
  // inputRef.current starts as null, becomes the DOM element after mount
  const [showForm, setShowForm] = useState(false);
  const [newTitle, setNewTitle] = useState('');

  // Step 2: Use useEffect to interact with the DOM element
  useEffect(() => {
    if (showForm && inputRef.current) {    // check: is form visible AND element exists?
      inputRef.current.focus();             // call native DOM .focus() method
    }
  }, [showForm]);  // re-run when showForm changes

  // Step 3: Attach ref to JSX element
  {showForm ? (
    <form onSubmit={handleAdd}>
      <input
        ref={inputRef}   // React assigns the DOM element to inputRef.current
        type="text"
        value={newTitle}
        onChange={e => setNewTitle(e.target.value)}
        placeholder="Task title"
        style={{ width: '100%', padding: '6px', marginBottom: '8px', boxSizing: 'border-box' }}
      />
      {/* ... */}
    </form>
  ) : /* ... */}
}
```

### Example 2: Track double-click on column header to rename

Add column renaming. Create `src/hooks/useDoubleClick.js`:

```jsx
import { useRef, useCallback } from 'react';

export function useDoubleClick(handler, timeout = 300) {
  const timerRef = useRef(null);   // stores timer ID across renders (no re-render on change)

  return useCallback(() => {
    if (timerRef.current) {
      // Timer already set → this is the second click within timeout!
      clearTimeout(timerRef.current);    // cancel the pending timer
      timerRef.current = null;           // reset ref
      handler();                         // call the double-click handler
    } else {
      // First click: start a timer
      timerRef.current = setTimeout(() => {
        timerRef.current = null;         // clear after timeout expires (single click)
      }, timeout);
    }
  }, [handler, timeout]);
}
```

Use in Column:
```jsx
function Column({ column, onRenameColumn }) {
  const handleDoubleClick = useDoubleClick(() => {
    const newName = prompt('Rename column:', column.title);
    if (newName?.trim()) onRenameColumn(column.id, newName.trim());
  });

  return (
    <div>
      <h3 onClick={handleDoubleClick} style={{ cursor: 'pointer' }}>
        {column.title}
      </h3>
      {/* ... */}
    </div>
  );
}
```

### Example 3: Measure task card height for animation

`src/hooks/useDimensions.js`:

```jsx
import { useRef, useState, useEffect } from 'react';

export function useDimensions() {
  const ref = useRef(null);
  const [dimensions, setDimensions] = useState({ width: 0, height: 0 });

  useEffect(() => {
    if (!ref.current) return;

    const observer = new ResizeObserver(entries => {
      const { width, height } = entries[0].contentRect;
      setDimensions({ width, height });
    });

    observer.observe(ref.current);
    return () => observer.disconnect();
  }, []);

  return [ref, dimensions];
}

// Use in TaskCard:
function TaskCard({ task }) {
  const [ref, dims] = useDimensions();

  return (
    <div ref={ref}>
      {/* ... */}
      <span style={{ fontSize: '10px', color: '#999' }}>
        {dims.width}×{dims.height}
      </span>
    </div>
  );
}
```

### Example 4: Stable callback reference (fixes useEffect bugs)

```jsx
// ❌ BUG: onMoveTask in deps causes infinite re-renders
useEffect(() => {
  const handler = (e) => onMoveTask(...);
  window.addEventListener('custom-event', handler);
  return () => window.removeEventListener('custom-event', handler);
}, [onMoveTask]); // onMoveTask is new every render → effect runs every render

// ✅ FIX: Store callback in ref, use stable ref in effect
function useStableCallback(callback) {
  const ref = useRef(callback);
  useEffect(() => { ref.current = callback; });
  return useCallback((...args) => ref.current(...args), []);
}

// Now the effect dependency is stable
useEffect(() => {
  const handler = (e) => stableMoveTask(e.detail);
  window.addEventListener('custom-event', handler);
  return () => window.removeEventListener('custom-event', handler);
}, []); // Empty deps — never re-runs
```

## 3. WHY THIS MATTERS

The most common interview question about useRef is: **"What's the difference between useRef and useState?"**

| useState | useRef |
|----------|--------|
| Causes re-render when changed | Does NOT cause re-render |
| Has setter function | Just `ref.current = value` |
| Use for UI data | Use for imperative/technical data |
| Reading immediately after set gives old value | `ref.current` is always current |

## 4. EXERCISE

Add a keyboard shortcut: when the user presses `Ctrl+K`, focus the "Add Task" input of the first column, and scroll to it.

**Solution:**
```jsx
// In App.jsx or a new hook useGlobalShortcuts.js
import { useEffect, useRef } from 'react';

export function useGlobalShortcut(key, ctrlKey, handler) {
  useEffect(() => {
    function onKeyDown(e) {
      if (e.key === key && e.ctrlKey === ctrlKey) {
        e.preventDefault();
        handler();
      }
    }
    window.addEventListener('keydown', onKeyDown);
    return () => window.removeEventListener('keydown', onKeyDown);
  }, [key, ctrlKey, handler]);
}

// Usage:
const focusFirstInput = useCallback(() => {
  const firstInput = document.querySelector('.column:first-child input');
  firstInput?.focus();
  firstInput?.scrollIntoView({ behavior: 'smooth' });
}, []);

useGlobalShortcut('k', true, focusFirstInput);
```

### CHALLENGE EXERCISE

Create a `usePrevious` hook that tracks the previous value of a state or prop. Show it in a tooltip: "Task count changed from 3 to 4".

**Solution:**
```jsx
// src/hooks/usePrevious.js
import { useRef, useEffect } from 'react';

export function usePrevious(value) {
  const ref = useRef();              // store previous value
  useEffect(() => {
    ref.current = value;             // update AFTER every render with current value
  });
  return ref.current;                // return the value from the PREVIOUS render
}

// Usage in App.jsx:
function App() {
  const { totalTasks } = useBoard();
  const prevTotal = usePrevious(totalTasks);

  useEffect(() => {
    if (prevTotal !== undefined && prevTotal !== totalTasks) {
      console.log(`Task count changed from ${prevTotal} to ${totalTasks}`);
    }
  }, [totalTasks, prevTotal]);

  return (
    <div>
      <Header />
      <div style={{ fontSize: '12px', color: '#94a3b8', textAlign: 'center' }}>
        {prevTotal !== undefined && prevTotal !== totalTasks && (
          <span>Changed from {prevTotal} to {totalTasks}</span>
        )}
      </div>
      <Board ... />
    </div>
  );
}

// Alternative: just export the hook and show in any component
function ColumnHeader({ count }) {
  const prevCount = usePrevious(count);

  return (
    <h3>
      {count} tasks
      {prevCount !== undefined && prevCount !== count && (
        <span style={{ fontSize: '11px', color: '#3b82f6', marginLeft: '8px' }}>
          ({count > prevCount ? '+' : ''}{count - prevCount})
        </span>
      )}
    </h3>
  );
}
```

## 5. COMMON MISTAKES

**Mistake: Using ref where state is needed**
```jsx
// ❌ Ref doesn't cause re-render — UI won't update!
const count = useRef(0);
function increment() {
  count.current += 1;
  console.log(count.current); // ✅ value updates
  // ❌ But component doesn't re-render — UI stays same
}

// ✅ Use state when UI needs to reflect the value
const [count, setCount] = useState(0);
```

**Mistake: Reading ref.current in render (before assignment)**
```jsx
function MyComponent() {
  const ref = useRef(null);

  // ❌ ref.current is null on first render
  // (DOM element doesn't exist yet)
  ref.current?.focus();

  useEffect(() => {
    ref.current?.focus(); // ✅ ref.current exists AFTER mount
  }, []);
```

### 🐛 DEBUGGING WITH DEVTOOLS

Refs don't appear in React DevTools (they don't trigger re-renders). To debug them, log `ref.current` in a useEffect. The most common ref bug: accessing `ref.current` during render when it's still null. Remember: DOM refs are only populated AFTER the component mounts (the first useEffect). If you need the value during render, use state, not a ref.

### ⚙️ UNDER THE HOOD

useRef is implemented as `useState({ current: initialValue })[0]` — except React never re-renders when you change `.current`. The ref object is stored in the fiber's hook list (like state), but changing it doesn't queue a re-render. This makes refs ideal for values that must persist across renders but shouldn't trigger UI updates — timer IDs, previous values, DOM references.

### 🏭 PRODUCTION PATTERN

The most common production use of useRef: integrating with imperative third-party libraries (charts, maps, animations). Libraries like D3, Chart.js, and Leaflet need a DOM reference to mount into. useRef gives you that reference without re-rendering the component every time the library updates the DOM internally.

### 💼 INTERVIEW READY

**Q:** What's the difference between useRef and useState? When would you use each?
**A:** `useState` causes re-renders; `useRef` does not. Use state for data that should appear in the UI (form inputs, fetched data, toggles). Use refs for data that persists across renders but shouldn't trigger updates (timer IDs, DOM element references, previous values, animation handles). The rule: if changing it should update the screen → state. If not → ref.

### 📝 MINI QUIZ

**1.** Does changing `ref.current` cause a re-render?
- A) Yes
- B) No
- C) Only in StrictMode
- D) Only if ref was created with useRef

**2.** When is `ref.current` populated for a DOM element ref?
- A) Immediately in the component body
- B) After the component mounts (first useEffect)
- C) After every render
- D) Only when explicitly called

**3.** What's a valid use case for useRef?
- A) Storing form input values for display
- B) Storing a timer ID for setInterval
- C) Storing user data from an API
- D) Toggling a class name

**Answers:** 1. B 2. B 3. B

---

## 6. CHECKPOINT

> What happens to `ref.current` between re-renders, and does changing it cause a re-render?

**Answer:** `ref.current` persists across renders (same object). Changing it does NOT cause a re-render. This makes refs useful for values that need to survive renders but shouldn't trigger UI updates.

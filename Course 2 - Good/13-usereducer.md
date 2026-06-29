# Lesson 13: useReducer — Complex State Logic

## 1. CONCEPT — When useState Isn't Enough

> **Real-world analogy:** `useReducer` is like having a dedicated switchboard operator instead of multiple direct phone lines. With `useState`, each function calls state directly (like dialing a phone yourself). With `useReducer`, you call the operator (dispatch an action), and the operator routes the call correctly (the reducer handles all state logic). It's more organized when you have many connected operations.

`useState` is great for independent values. But when state updates depend on each other in complex ways, `useReducer` is cleaner.

**The pattern:** You dispatch an **action** (an object describing what happened). A **reducer function** receives the current state + the action, and returns the new state.

```jsx
// useState version — multiple functions, scattered logic
function Board() {
  const [columns, setColumns] = useState({});

  function addTask(columnId, task) { /* complex update */ }
  function moveTask(id, from, to) { /* complex update */ }
  function deleteTask(columnId, id) { /* complex update */ }
  function reorderTasks(columnId, fromIndex, toIndex) { /* ... */ }
  function editTask(columnId, id, changes) { /* ... */ }
}

// useReducer version — all logic in ONE place
function Board() {
  const [columns, dispatch] = useReducer(boardReducer, initialState);

  // Every action is a simple dispatch call:
  dispatch({ type: 'ADD_TASK', columnId, task });
  dispatch({ type: 'MOVE_TASK', id, from, to });
  dispatch({ type: 'DELETE_TASK', columnId, id });
}
```

**Why this matters for TaskFlow:** Our board has 5 operations (add, move, delete, reorder, edit). With `useReducer`, all the logic lives in one pure function that's easy to read, test, and debug.

## 2. CODE ALONG — Refactor Board to useReducer

First, let's understand the pattern with a mini example, then apply it.

### Mini Example: Counter with useReducer

```jsx
import { useReducer } from 'react';

// 1. Define reducer — just a function that takes state + action, returns new state
function counterReducer(state, action) {
  // action = { type: 'INCREMENT' } or { type: 'SET', payload: 100 }
  switch (action.type) {
    case 'INCREMENT':
      return { count: state.count + 1 };   // return NEW state object
    case 'DECREMENT':
      return { count: state.count - 1 };
    case 'SET':
      return { count: action.payload };    // payload carries the new value
    case 'RESET':
      return { count: 0 };
    default:
      return state;   // CRITICAL: always return state for unknown actions
  }
}

function Counter() {
  // 2. useReducer(reducer, initialState) → [state, dispatch]
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });

  // 3. dispatch instead of setState
  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>    // no payload needed
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
      <button onClick={() => dispatch({ type: 'RESET' })}>Reset</button>
      <button onClick={() => dispatch({ type: 'SET', payload: 100 })}>Set to 100</button>  // with payload
    </div>
  );
}
```

### Now: Refactor TaskFlow Board

Create `src/reducers/boardReducer.js`:

```jsx
// Action types as constants (prevents typos)
export const ACTIONS = {
  ADD_TASK: 'ADD_TASK',
  MOVE_TASK: 'MOVE_TASK',
  DELETE_TASK: 'DELETE_TASK',
  EDIT_TASK: 'EDIT_TASK',
  ADD_COLUMN: 'ADD_COLUMN',
  REORDER_TASKS: 'REORDER_TASKS',
};

export const initialColumns = {
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

export function boardReducer(state, action) {
  switch (action.type) {
    case ACTIONS.ADD_TASK: {
      const { columnId, title, priority } = action.payload;  // destructure payload
      const newTask = {
        id: Date.now(),               // unique ID from timestamp
        title,
        priority: priority || 'Medium',
        createdAt: new Date().toISOString(),  // ISO string for consistent formatting
      };
      return {
        ...state,                     // spread existing columns
        [columnId]: {                 // update only the target column
          ...state[columnId],         // preserve other column properties
          tasks: [...state[columnId].tasks, newTask],  // append new task
        },
      };
    }

    case ACTIONS.MOVE_TASK: {
      const { taskId, fromColumnId, toColumnId } = action.payload;
      const task = state[fromColumnId].tasks.find(t => t.id === taskId);
      if (!task) return state;

      const updatedTask = toColumnId === 'done'
        ? { ...task, completedAt: new Date().toISOString() }
        : { ...task, completedAt: undefined };

      return {
        ...state,
        [fromColumnId]: {
          ...state[fromColumnId],
          tasks: state[fromColumnId].tasks.filter(t => t.id !== taskId),
        },
        [toColumnId]: {
          ...state[toColumnId],
          tasks: [...state[toColumnId].tasks, updatedTask],
        },
      };
    }

    case ACTIONS.DELETE_TASK: {
      const { columnId, taskId } = action.payload;
      return {
        ...state,
        [columnId]: {
          ...state[columnId],
          tasks: state[columnId].tasks.filter(t => t.id !== taskId),
        },
      };
    }

    case ACTIONS.EDIT_TASK: {
      const { columnId, taskId, changes } = action.payload;
      return {
        ...state,
        [columnId]: {
          ...state[columnId],
          tasks: state[columnId].tasks.map(t =>
            t.id === taskId ? { ...t, ...changes } : t
          ),
        },
      };
    }

    case ACTIONS.ADD_COLUMN: {
      const { id, title } = action.payload;
      return {
        ...state,
        [id]: { id, title, tasks: [] },
      };
    }

    default:
      return state;
  }
}
```

### Now use it in `hooks/useBoard.js`:

```jsx
import { useReducer, useCallback, useEffect } from 'react';
import { boardReducer, initialColumns, ACTIONS } from '../reducers/boardReducer';

export function useBoard() {
  // Load from localStorage
  const [columns, dispatch] = useReducer(
    boardReducer,
    initialColumns,
    // Third arg: initializer function (runs once)
    (initial) => {
      try {
        const saved = localStorage.getItem('taskflow-board');
        return saved ? JSON.parse(saved) : initial;
      } catch {
        return initial;
      }
    }
  );

  // Persist to localStorage
  useEffect(() => {
    localStorage.setItem('taskflow-board', JSON.stringify(columns));
  }, [columns]);

  // Each action is a one-liner
  const addTask = useCallback((columnId, title, priority) => {
    dispatch({ type: ACTIONS.ADD_TASK, payload: { columnId, title, priority } });
  }, []);

  const moveTask = useCallback((taskId, fromColumnId, toColumnId) => {
    dispatch({ type: ACTIONS.MOVE_TASK, payload: { taskId, fromColumnId, toColumnId } });
  }, []);

  const deleteTask = useCallback((columnId, taskId) => {
    dispatch({ type: ACTIONS.DELETE_TASK, payload: { columnId, taskId } });
  }, []);

  const editTask = useCallback((columnId, taskId, changes) => {
    dispatch({ type: ACTIONS.EDIT_TASK, payload: { columnId, taskId, changes } });
  }, []);

  // Derived data
  const totalTasks = Object.values(columns)
    .reduce((sum, col) => sum + col.tasks.length, 0);

  return { columns, addTask, moveTask, deleteTask, editTask, totalTasks, dispatch };
}
```

### Update Column.jsx to pass the new structure:

```jsx
// In Column.jsx, the add form handler:
function handleAdd(e) {
  e.preventDefault();
  if (!newTitle.trim()) return;
  onAddTask(column.id, newTitle.trim(), newPriority);
  setNewTitle('');
  setNewPriority('Medium');
  setShowForm(false);
}
```

## 3. WHY THIS MATTERS

| Aspect | useState | useReducer |
|--------|----------|------------|
| Readability | Logic scattered across handlers | All logic in one function |
| Testability | Hard to test (coupled to component) | Pure function — easy to test |
| Debugging | No structured updates | Each action has a type + payload |
| Complex state | Multiple setState calls get messy | One dispatch per operation |

**Real-world:** E-Commerce checkout state (shipping, payment, review, confirmation) is a classic useReducer use case. So is a banking transaction flow.

## 4. EXERCISE

Write a unit test for the `boardReducer` to verify `MOVE_TASK` works:

```jsx
// __tests__/boardReducer.test.js
import { boardReducer, initialColumns, ACTIONS } from '../reducers/boardReducer';

describe('boardReducer', () => {
  test('moves task between columns', () => {
    const state = structuredClone(initialColumns);
    const todoCount = state['todo'].tasks.length;
    const inProgressCount = state['in-progress'].tasks.length;

    const nextState = boardReducer(state, {
      type: ACTIONS.MOVE_TASK,
      payload: { taskId: 1, fromColumnId: 'todo', toColumnId: 'in-progress' },
    });

    // Task removed from todo
    expect(nextState['todo'].tasks).toHaveLength(todoCount - 1);
    // Task added to in-progress
    expect(nextState['in-progress'].tasks).toHaveLength(inProgressCount + 1);
    // The moved task has the right id
    expect(nextState['in-progress'].tasks.at(-1).id).toBe(1);
  });

  test('does nothing if task not found', () => {
    const state = structuredClone(initialColumns);
    const nextState = boardReducer(state, {
      type: ACTIONS.MOVE_TASK,
      payload: { taskId: 999, fromColumnId: 'todo', toColumnId: 'done' },
    });
    expect(nextState).toEqual(state);
  });

  test('returns state unchanged for unknown action', () => {
    const state = structuredClone(initialColumns);
    const nextState = boardReducer(state, { type: 'UNKNOWN' });
    expect(nextState).toEqual(state);
  });
});
```

Run with:
```bash
npx vitest run
```

### CHALLENGE EXERCISE

Add an `UNDO` action to the boardReducer. Store a `previousState` copy before every state mutation, and when `UNDO` is dispatched, restore it. Add an "Undo" button in the UI that dispatches the undo action.

**Solution:**

In boardReducer.js, add:
```jsx
export const ACTIONS = {
  // ... existing actions
  UNDO: 'UNDO',
};

export function boardReducer(state, action) {
  switch (action.type) {
    // Wrap every mutation to save previous state
    case ACTIONS.ADD_TASK:
    case ACTIONS.MOVE_TASK:
    case ACTIONS.DELETE_TASK:
    case ACTIONS.EDIT_TASK: {
      // Save current state for undo BEFORE applying the action
      const result = boardReducerInner(state, action);
      return { ...result, _previousState: state };
    }

    case ACTIONS.UNDO:
      if (!state._previousState) return state;
      return { ...state._previousState, _previousState: undefined };

    // ... other cases
  }
}

// In useBoard hook, add:
const undo = useCallback(() => {
  dispatch({ type: ACTIONS.UNDO });
}, []);

// Add undo button in App:
<button
  onClick={undo}
  disabled={!columns._previousState}
  style={{ padding: '8px 16px', cursor: 'pointer' }}
>
  ↩ Undo
</button>
```

## 5. COMMON MISTAKES

**Mistake: Mutating state directly in reducer**
```jsx
// ❌ WRONG — mutates existing state
case 'DELETE_TASK':
  state[columnId].tasks = state[columnId].tasks.filter(...);
  return state; // React sees same reference — no re-render!

// ✅ RIGHT — returns new state object
case 'DELETE_TASK':
  return {
    ...state,
    [columnId]: {
      ...state[columnId],
      tasks: state[columnId].tasks.filter(...),
    },
  };
```

**Mistake: Not handling the default case**
```jsx
// ❌ Missing default — returns undefined for unknown actions
switch (action.type) {
  case 'ADD': return ...;
  // No default: function returns undefined → breaks app

// ✅ Always return state for unknown actions
switch (action.type) {
  case 'ADD': return ...;
  default: return state;
}
```

**Mistake: Doing side effects inside reducer**
```jsx
// ❌ Reducer must be PURE — no side effects!
case 'ADD_TASK':
  localStorage.setItem('tasks', JSON.stringify(newState)); // Side effect!
  saveToAnalytics(action); // Side effect!
  return newState;

// ✅ Side effects happen OUTSIDE (in event handler or useEffect)
```

### 🐛 DEBUGGING WITH DEVTOOLS

Use the "Actions" tab in Redux DevTools (also works with useReducer!). After installing the Redux DevTools extension, dispatched actions appear with their type and payload. You can time-travel: click an earlier action to see the state at that point. This is invaluable for debugging complex state transitions. If you use the `devtools` middleware in Zustand, the same DevTools work.

### ⚙️ UNDER THE HOOD

useReducer is built on the same hook mechanism as useState — in fact, `useState` is implemented using `useReducer` internally! `useState(0)` is equivalent to `useReducer((prev, action) => action, 0)`. The reducer pattern guarantees state transitions are pure and predictable. Because reducers are pure functions, you can test them in isolation without rendering any components.

### 🏭 PRODUCTION PATTERN

useReducer + Context is a common lightweight state management pattern: define actions in a reducer, expose state + dispatch via Context, and let any descendant dispatch actions. This avoids prop drilling without adding a library. For truly complex state, pair useReducer with Zustand or Immer (which lets you write mutable syntax in reducers).

### 💼 INTERVIEW READY

**Q:** When would you choose useReducer over useState?
**A:** (1) When state logic is complex with multiple sub-values or interdependent updates. (2) When the next state depends on the previous state in non-trivial ways. (3) When you want to centralize state transitions for testability. (4) When you need time-travel debugging. For simple independent values (a toggle, a counter), useState is simpler and perfectly adequate.

### 📝 MINI QUIZ

**1.** What does dispatch do?
- A) Updates state directly
- B) Sends an action object to the reducer
- C) Creates a new reducer
- D) Logs state to console

**2.** Why must reducers be pure functions?
- A) For performance
- B) For predictable state updates and time-travel debugging
- C) To reduce bundle size
- D) To enable TypeScript inference

**3.** What should a reducer return for an unknown action type?
- A) null
- B) The current state unchanged
- C) A default state
- D) An error

**Answers:** 1. B 2. B 3. B

---

## 6. CHECKPOINT

> What makes a reducer function "pure," and why must it be pure?

**Answer:** A pure function returns the same output for the same input and has no side effects (no API calls, no localStorage, no random values). React relies on this for features like time-travel debugging and re-render optimization.

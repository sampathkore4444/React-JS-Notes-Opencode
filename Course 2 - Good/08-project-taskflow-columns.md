# Lesson 8: Project Day — Building TaskFlow Columns

## 1. TODAY'S GOAL

> **Real-world analogy:** A Kanban board is like a restaurant kitchen with three stations: "Order Tickets" (To Do), "Being Cooked" (In Progress), and "Ready to Serve" (Done). Each order (task) moves through the stations. The board gives everyone instant visibility into what needs to be done, what's being worked on, and what's finished.

Transform the flat task list into a **Kanban board** with columns: **To Do → In Progress → Done**. Each column has its own task list. You can move tasks between columns.

## 2. PLAN FIRST

```
App (state: all board data)
├── Header
├── Board
│   ├── Column ("To Do")
│   │   └── TaskCard × N
│   ├── Column ("In Progress")
│   │   └── TaskCard × N
│   └── Column ("Done")
│       └── TaskCard × N
└── Footer
```

Data structure:
```js
const [columns, setColumns] = useState({
  'todo': { id: 'todo', title: 'To Do', tasks: [] },
  'in-progress': { id: 'in-progress', title: 'In Progress', tasks: [] },
  'done': { id: 'done', title: 'Done', tasks: [] },
});
```

## 3. CODE ALONG — Step by Step

### Step 1: Update the data structure in App.jsx

```jsx
import { useState, useEffect } from 'react';
import Header from './components/Header';
import Board from './components/Board';
import Footer from './components/Footer';

function App() {
  const [columns, setColumns] = useState(() => {
    const saved = localStorage.getItem('taskflow-board');  // check for saved data
    if (saved) {
      try { return JSON.parse(saved); }
      catch { /* corrupted data → start fresh */ }
    }
    // Default board with 3 columns and sample tasks
    return {
      'todo': { id: 'todo', title: 'To Do',
        tasks: [
          { id: 1, title: 'Learn React', priority: 'High' },
          { id: 2, title: 'Design components', priority: 'Medium' },
        ],
      },
      'in-progress': { id: 'in-progress', title: 'In Progress',
        tasks: [ { id: 3, title: 'Build TaskFlow', priority: 'Medium' } ],
      },
      'done': { id: 'done', title: 'Done',
        tasks: [ { id: 4, title: 'Set up project', priority: 'Low' } ],
      },
    };
  });

  // Persist to localStorage
  useEffect(() => {
    localStorage.setItem('taskflow-board', JSON.stringify(columns));
  }, [columns]);

  function addTask(columnId, taskData) {
    setColumns(prev => ({                    // must use function form because we read prev
      ...prev,                               // spread existing columns (immutable update)
      [columnId]: {                          // computed property name: columnId is dynamic
        ...prev[columnId],                   // preserve existing column properties
        tasks: [
          ...prev[columnId].tasks,           // keep existing tasks in that column
          { id: Date.now(), ...taskData },   // append new task with unique ID
        ],
      },
    }));
  }

  function moveTask(taskId, fromColumnId, toColumnId) {
    setColumns(prev => {
      const task = prev[fromColumnId].tasks.find(t => t.id === taskId);  // find the task
      if (!task) return prev;  // guard: task not found, return unchanged

      return {
        ...prev,
        [fromColumnId]: {
          ...prev[fromColumnId],
          tasks: prev[fromColumnId].tasks.filter(t => t.id !== taskId),  // remove from source
        },
        [toColumnId]: {
          ...prev[toColumnId],
          tasks: [...prev[toColumnId].tasks, task],  // add to destination
        },
      };
    });
  }

  function deleteTask(columnId, taskId) {
    setColumns(prev => ({
      ...prev,
      [columnId]: {
        ...prev[columnId],
        tasks: prev[columnId].tasks.filter(t => t.id !== taskId),
      },
    }));
  }

  return (
    <div style={{ padding: '20px', maxWidth: '1200px', margin: '0 auto' }}>
      <Header />
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

### Step 2: Create Board component

`src/components/Board.jsx`:
```jsx
import Column from './Column';

function Board({ columns, onAddTask, onMoveTask, onDeleteTask }) {
  const columnIds = Object.keys(columns);

  return (
    <div style={{
      display: 'grid',
      gridTemplateColumns: `repeat(${columnIds.length}, 1fr)`,
      gap: '16px',
      marginTop: '20px',
      minHeight: '400px',
    }}>
      {columnIds.map(columnId => (
        <Column
          key={columnId}
          column={columns[columnId]}
          allColumns={columns}
          onAddTask={onAddTask}
          onMoveTask={onMoveTask}
          onDeleteTask={onDeleteTask}
        />
      ))}
    </div>
  );
}

export default Board;
```

### Step 3: Create Column component

`src/components/Column.jsx`:
```jsx
import { useState } from 'react';
import TaskCard from './TaskCard';

function Column({ column, allColumns, onAddTask, onMoveTask, onDeleteTask }) {
  const [showForm, setShowForm] = useState(false);
  const [newTitle, setNewTitle] = useState('');
  const [newPriority, setNewPriority] = useState('Medium');

  function handleAdd(e) {
    e.preventDefault();
    if (!newTitle.trim()) return;
    onAddTask(column.id, { title: newTitle.trim(), priority: newPriority });
    setNewTitle('');
    setNewPriority('Medium');
    setShowForm(false);
  }

  const otherColumns = Object.values(allColumns).filter(c => c.id !== column.id);

  return (
    <div style={{
      backgroundColor: '#f1f5f9',
      borderRadius: '8px',
      padding: '12px',
      minHeight: '200px',
    }}>
      <div style={{
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
        marginBottom: '12px',
      }}>
        <h3 style={{ margin: 0 }}>{column.title}</h3>
        <span style={{
          backgroundColor: '#cbd5e1',
          padding: '2px 8px',
          borderRadius: '12px',
          fontSize: '14px',
        }}>
          {column.tasks.length}
        </span>
      </div>

      {/* Tasks */}
      <div style={{ minHeight: '100px' }}>
        {column.tasks.map(task => (
          <TaskCard
            key={task.id}
            task={task}
            columnId={column.id}
            otherColumns={otherColumns}
            onMoveTask={onMoveTask}
            onDeleteTask={onDeleteTask}
          />
        ))}
        {column.tasks.length === 0 && (
          <p style={{ color: '#94a3b8', textAlign: 'center', fontSize: '14px' }}>
            No tasks
          </p>
        )}
      </div>

      {/* Add task form or button */}
      {showForm ? (
        <form onSubmit={handleAdd} style={{ marginTop: '12px' }}>
          <input
            type="text"
            value={newTitle}
            onChange={e => setNewTitle(e.target.value)}
            placeholder="Task title"
            style={{ width: '100%', padding: '6px', marginBottom: '8px', boxSizing: 'border-box' }}
            autoFocus
          />
          <select
            value={newPriority}
            onChange={e => setNewPriority(e.target.value)}
            style={{ width: '100%', padding: '6px', marginBottom: '8px', boxSizing: 'border-box' }}
          >
            <option>High</option>
            <option>Medium</option>
            <option>Low</option>
          </select>
          <div style={{ display: 'flex', gap: '4px' }}>
            <button type="submit" style={{ flex: 1, padding: '6px', cursor: 'pointer' }}>
              Add
            </button>
            <button type="button" onClick={() => setShowForm(false)}
              style={{ padding: '6px', cursor: 'pointer' }}>
              ✕
            </button>
          </div>
        </form>
      ) : (
        <button
          onClick={() => setShowForm(true)}
          style={{
            width: '100%', padding: '8px', marginTop: '8px',
            cursor: 'pointer', border: '2px dashed #cbd5e1',
            background: 'none', borderRadius: '4px', color: '#64748b',
          }}
        >
          + Add Task
        </button>
      )}
    </div>
  );
}

export default Column;
```

### Step 4: Create TaskCard component

`src/components/TaskCard.jsx`:
```jsx
import { useState } from 'react';

function TaskCard({ task, columnId, otherColumns, onMoveTask, onDeleteTask }) {
  const [menuOpen, setMenuOpen] = useState(false);

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
      position: 'relative',
    }}>
      <div style={{ display: 'flex', justifyContent: 'space-between', alignItems: 'start' }}>
        <span style={{ fontSize: '14px', wordBreak: 'break-word', flex: 1 }}>
          {task.title}
        </span>
        <div style={{ position: 'relative' }}>
          <button
            onClick={() => setMenuOpen(!menuOpen)}
            style={{ border: 'none', background: 'none', cursor: 'pointer', fontSize: '16px' }}
          >
            ⋯
          </button>

          {menuOpen && (
            <div style={{
              position: 'absolute', right: 0, top: '100%',
              backgroundColor: 'white', border: '1px solid #ddd',
              borderRadius: '4px', boxShadow: '0 2px 8px rgba(0,0,0,0.15)',
              zIndex: 10, minWidth: '150px',
            }}>
              {/* Move to other columns */}
              {otherColumns.map(col => (
                <button
                  key={col.id}
                  onClick={() => {
                    onMoveTask(task.id, columnId, col.id);
                    setMenuOpen(false);
                  }}
                  style={{
                    display: 'block', width: '100%', padding: '8px 12px',
                    border: 'none', background: 'none', cursor: 'pointer',
                    textAlign: 'left', fontSize: '14px',
                  }}
                >
                  Move to {col.title}
                </button>
              ))}
              <hr style={{ margin: '4px 0' }} />
              <button
                onClick={() => {
                  onDeleteTask(columnId, task.id);
                  setMenuOpen(false);
                }}
                style={{
                  display: 'block', width: '100%', padding: '8px 12px',
                  border: 'none', background: 'none', cursor: 'pointer',
                  textAlign: 'left', fontSize: '14px', color: '#dc2626',
                }}
              >
                Delete
              </button>
            </div>
          )}
        </div>
      </div>

      <span style={{
        display: 'inline-block', fontSize: '11px', padding: '2px 6px',
        borderRadius: '4px', marginTop: '6px',
        backgroundColor: colors.bg, color: colors.text,
      }}>
        {task.priority}
      </span>
    </div>
  );
}

export default TaskCard;
```

## 4. TEST IT

```bash
npm run dev
```

You should see a **3-column Kanban board**:
- Each column has a title and task count
- Click "+ Add Task" to add tasks to any column
- Click "⋯" on any task to move it to another column or delete it
- Data persists in localStorage

## 5. EXERCISE

Add a "completedAt" timestamp to tasks when they move to the "Done" column, and show it on the TaskCard.

**Solution:**
```jsx
// In App.jsx moveTask function, add timestamp:
function moveTask(taskId, fromColumnId, toColumnId) {
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
}

// In TaskCard, below the priority badge:
{task.completedAt && (
  <div style={{ fontSize: '11px', color: '#888', marginTop: '4px' }}>
    ✅ Completed: {task.completedAt}
  </div>
)}
```

### CHALLENGE EXERCISE

Add column colors. Each column should have a distinct background color (e.g., To Do = pink, In Progress = yellow, Done = green). Store the color with each column and apply it to the column header.

**Solution:**

In App.jsx, update the default columns to include a `color` property:
```jsx
'todo': {
  id: 'todo', title: 'To Do', color: '#fce4ec',
  tasks: [...]
},
'in-progress': {
  id: 'in-progress', title: 'In Progress', color: '#fff9c4',
  tasks: [...]
},
'done': {
  id: 'done', title: 'Done', color: '#c8e6c9',
  tasks: [...]
},
```

In Column.jsx, update the column header to use the color:
```jsx
<div style={{
  backgroundColor: column.color || '#f1f5f9',  // fallback color
  borderRadius: '8px', padding: '12px', minHeight: '200px',
}}>
  <div style={{
    display: 'flex', justifyContent: 'space-between', alignItems: 'center',
    marginBottom: '12px', padding: '8px', borderRadius: '6px',
    backgroundColor: column.color ? column.color + '77' : '#e2e8f0',  // semi-transparent
  }}>
    <h3 style={{ margin: 0 }}>{column.title}</h3>
    ...
  </div>
  ...
</div>
```

Also add the color to the addColumn handler if you have one:
```jsx
case 'ADD_COLUMN':
  return {
    ...state,
    [id]: { id, title, tasks: [], color: '#e8f5e9' },
  };
```

## 6. WHAT YOU LEARNED TODAY

- **Objects as state** — updating nested objects correctly with spread
- **Component composition** — 4 levels deep (App → Board → Column → TaskCard)
- **Props drilling** — passing data and callbacks through multiple levels
- **Local vs lifted state** — form state in Column, board data in App

### 🐛 DEBUGGING WITH DEVTOOLS

With the Kanban board, select `<Board>` in React DevTools. Expand the `columns` state — you'll see three column objects, each with tasks. Move a task between columns and watch the state update live. Pay attention to the column object references — if they don't change when you move a task, your immutable update is broken (mutating state directly).

### ⚙️ UNDER THE HOOD

The spread operator (`...`) creates a shallow copy — nested objects still share references. When updating nested state like `columns[columnId].tasks`, every level must create a new object: new columns object → new column object → new tasks array. Skipping any level means that level still references the old object, and React won't detect the change.

### 🏭 PRODUCTION PATTERN

For deeply nested state, consider normalizing data (like a database): store tasks as a flat object `{ '1': task, '2': task }` and columns with just task ID arrays. This avoids three-level spread chains. Libraries like Zustand and Immer make nested updates cleaner — Immer lets you write mutable syntax that compiles to immutable updates.

### 💼 INTERVIEW READY

**Q:** How do you update nested state immutably in React?
**A:** Each nesting level requires a new object: `setColumns(prev => ({ ...prev, [columnId]: { ...prev[columnId], tasks: prev[columnId].tasks.map(t => t.id === id ? { ...t, done: true } : t) } }))`. Every level gets a spread. Libraries like Immer (`produce(state, draft => { draft[columnId].tasks[0].done = true })`) eliminate this boilerplate.

### 📝 MINI QUIZ

**1.** What happens if you only spread one level deep when updating nested state?
- A) It works normally
- B) React may not detect changes in nested objects
- C) An error is thrown
- D) Performance improves

**2.** Why use `setColumns(prev => ...)` instead of `setColumns({...columns, ...})`?
- A) It's required by React
- B) It's shorter to write
- C) It guarantees access to the latest state in concurrent mode
- D) It prevents TypeScript errors

**3.** What is `Date.now()` used for in this lesson?
- A) Displaying the current date
- B) Generating unique task IDs
- C) Timestamping edits
- D) Calculating task duration

**Answers:** 1. B 2. C 3. B

---

## 7. CHECKPOINT

> Why does `addTask` use `setColumns(prev => {...})` instead of modifying `columns` directly?

**Answer:** Because state is immutable. We must create a new object with spread to tell React to re-render.

# Lesson 6: Lifting State Up — Sharing Data Between Components

## 1. CONCEPT — State Lives at the Right Level

> **Real-world analogy:** Lifting state up is like having a family group chat. Instead of each family member calling others individually to share news, one person posts in the group chat and everyone sees it. The "group chat" (common parent component) holds the message (state) and distributes it to everyone who needs it.

**Problem:** You have a `TaskForm` that creates tasks and a `TaskList` that displays them. Both need access to the same `tasks` array.

**Rule:** State should live at the **lowest common ancestor** of all components that need it.

```
App (state: tasks lives here) ← Both children need this
├── TaskForm  (needs to add tasks)
└── TaskList  (needs to display tasks)
```

The parent (`App`) holds the state and passes it down as **props**. The parent also passes **callback functions** so children can modify the state.

## 2. CODE ALONG — Extract TaskList Into Its Own Component

Create `src/components/TaskList.jsx`:

```jsx
function TaskList({ tasks, onToggle, onDelete }) {
  const pendingTasks = tasks.filter(t => !t.done);    // derived data: counts pending
  const doneTasks = tasks.filter(t => t.done);          // derived data: counts done

  return (
    <div>
      <h2>Tasks ({pendingTasks.length} pending)</h2>

      {tasks.length === 0 ? (                           // conditional rendering: empty state
        <p style={{ color: '#888' }}>No tasks yet. Add one above!</p>
      ) : (
        <>
          {tasks.map(task => (
            <div key={task.id} style={{
              display: 'flex',
              alignItems: 'center',
              gap: '8px',
              padding: '12px',
              marginBottom: '8px',
              border: '1px solid #ddd',
              borderRadius: '8px',
              backgroundColor: task.done ? '#f9f9f9' : 'white',
              opacity: task.done ? 0.7 : 1,
            }}>
              <input
                type="checkbox"
                checked={task.done}
                onChange={() => onToggle(task.id)}     // callback UP to parent
              />
              <span style={{
                flex: 1,
                textDecoration: task.done ? 'line-through' : 'none',
              }}>
                {task.title}
              </span>
              <span style={{
                fontSize: '12px',
                padding: '2px 8px',
                borderRadius: '4px',
                color: 'white',
                backgroundColor: task.priority === 'High' ? '#ff4444'
                  : task.priority === 'Medium' ? '#ff8800' : '#00aa00',
              }}>
                {task.priority}
              </span>
              <button onClick={() => onDelete(task.id)} style={{  // callback UP to parent
                cursor: 'pointer', border: 'none', background: 'none', fontSize: '18px',
              }}>
                🗑️
              </button>
            </div>
          ))}

          {/* Summary */}
          <p style={{ marginTop: '16px', color: '#888', fontSize: '14px' }}>
            {doneTasks.length} of {tasks.length} tasks completed
          </p>
        </>
      )}
    </div>
  );
}

export default TaskList;
```

Now your `App.jsx` becomes cleaner:

```jsx
import { useState } from 'react';
import Header from './components/Header';
import TaskForm from './components/TaskForm';
import TaskList from './components/TaskList';
import Footer from './components/Footer';

function App() {
  const [tasks, setTasks] = useState([  // state lives HERE (common ancestor)
    { id: 1, title: 'Learn React', done: false, priority: 'High' },
    { id: 2, title: 'Build TaskFlow', done: false, priority: 'Medium' },
    { id: 3, title: 'Deploy app', done: false, priority: 'Low' },
  ]);

  function addTask(taskData) {                 // callback: child calls this
    setTasks(prev => [
      ...prev,
      { id: Date.now(), ...taskData, done: false },
    ]);
  }

  function toggleTask(id) {                     // another callback for child
    setTasks(tasks.map(t =>
      t.id === id ? { ...t, done: !t.done } : t
    ));
  }

  function deleteTask(id) {                     // another callback for child
    setTasks(tasks.filter(t => t.id !== id));
  }

  return (
    <div style={{ padding: '20px', maxWidth: '600px', margin: '0 auto' }}>
      <Header />
      <TaskForm onAddTask={addTask} />         // pass callback DOWN as prop
      <TaskList                                // pass data + callbacks DOWN
        tasks={tasks}                          // data flows down ↓
        onToggle={toggleTask}                  // events flow up ↑
        onDelete={deleteTask}
      />
      <Footer />
    </div>
  );
}

export default App;
```

**Check your browser.** Everything works the same, but now your code is organized into 4 small, focused components instead of one big file.

## 3. THE DATA FLOW VISUALIZED

```
User types in form
       ↓
TaskForm: onChange → setTitle (local state just for the form)
       ↓
User clicks "Add"
       ↓
TaskForm: onAddTask({ title, priority })  → calls function from App
       ↓
App: addTask() → setTasks(prev => [...prev, newTask])
       ↓
React re-renders App, TaskForm, TaskList
       ↓
TaskList receives new tasks as props → displays new task
```

**Data flows down (via props), events flow up (via callbacks).**

## 4. EXERCISE

Add a "Clear Completed" button to the App that removes all done tasks.

**Solution:**
```jsx
// In App.jsx, add this function:
function clearCompleted() {
  setTasks(tasks.filter(t => !t.done));
}

// Add this button between TaskList and Footer:
<button
  onClick={clearCompleted}
  style={{
    padding: '8px 16px',
    marginTop: '12px',
    cursor: 'pointer',
    backgroundColor: tasks.some(t => t.done) ? '#dc3545' : '#ccc',
    color: 'white',
    border: 'none',
    borderRadius: '4px',
  }}
  disabled={!tasks.some(t => t.done)}
>
  Clear Completed
</button>
```

### CHALLENGE EXERCISE

Add an "Edit Task" feature where clicking a task title turns it into an editable input. The parent (App) holds the editing state. Pass `onEdit` and `editingId` props to TaskList.

**Solution:**

In App.jsx, add:
```jsx
const [editingId, setEditingId] = useState(null);
const [editText, setEditText] = useState('');

function startEdit(task) {
  setEditingId(task.id);
  setEditText(task.title);
}

function saveEdit(id) {
  setTasks(tasks.map(t => t.id === id ? { ...t, title: editText } : t));
  setEditingId(null);
}

function cancelEdit() {
  setEditingId(null);
}
```

Pass to TaskList:
```jsx
<TaskList
  tasks={tasks}
  onToggle={toggleTask}
  onDelete={deleteTask}
  onEdit={startEdit}
  onSaveEdit={saveEdit}
  onCancelEdit={cancelEdit}
  editingId={editingId}
  editText={editText}
  onEditTextChange={setEditText}
/>
```

In TaskList, replace the title span with:
```jsx
{editingId === task.id ? (
  <input
    value={editText}
    onChange={(e) => onEditTextChange(e.target.value)}
    autoFocus
    onKeyDown={e => {
      if (e.key === 'Enter') onSaveEdit(task.id);
      if (e.key === 'Escape') onCancelEdit();
    }}
    style={{ flex: 1, padding: '4px' }}
  />
) : (
  <span style={{
    flex: 1, cursor: 'pointer',
    textDecoration: task.done ? 'line-through' : 'none',
  }} onClick={() => onEdit(task)}>
    {task.title}
  </span>
)}
```

## 5. COMMON MISTAKES

**Mistake: Storing the same data in multiple places**
```jsx
// ❌ Duplicated state — they'll go out of sync!
function App() {
  const [tasks, setTasks] = useState([]);
  const [completedCount, setCompletedCount] = useState(0);
  // Now you have to update BOTH when a task is toggled
}

// ✅ Derive data from the single source of truth
function App() {
  const [tasks, setTasks] = useState([]);
  const completedCount = tasks.filter(t => t.done).length; // derived!
}
```

**Mistake: Lifting state too high (everything in App)**
As your app grows, put state in the nearest common ancestor, not always in the top-level App.

### 🐛 DEBUGGING WITH DEVTOOLS

In React DevTools, select `<App>`. You'll see the `tasks` state. Click the "eye" to expand each task object. When you click "Add" in TaskForm, watch the App's state update in real-time — this confirms data is flowing from TaskForm → App (via callback) → TaskList (via props). If TaskList doesn't update, the callback likely isn't wired correctly.

### ⚙️ UNDER THE HOOD

"Lifting state up" follows React's unidirectional data flow: state lives at the lowest common ancestor and flows down via props. Children communicate UP by calling callback functions passed as props. This ensures predictable, debuggable data flow. The alternative (duplicating state in multiple components) causes synchronization bugs — the same data in two places will inevitably diverge.

### 🏭 PRODUCTION PATTERN

As apps grow, lifting state to App.jsx creates "prop drilling" — passing props through intermediate components. Recognize when prop drilling becomes painful (3+ levels deep) and extract state into Context or a state management library. The right tool depends on how often the state changes.

### 💼 INTERVIEW READY

**Q:** Why does React enforce one-way data flow instead of two-way binding like Angular?
**A:** One-way data flow is predictable — data enters a component through props, and changes go up through callbacks. There's only one path to understand. Two-way binding means any component can modify any data at any time, making debugging harder as the app grows. One-way flow scales better for complex UIs.

### 📝 MINI QUIZ

**1.** Where should state live when two sibling components need the same data?
- A) In each sibling (duplicated)
- B) In their closest common parent
- C) In a global variable
- D) In localStorage

**2.** How does a child component communicate "up" to its parent?
- A) By modifying props directly
- B) By calling a callback function passed as a prop
- C) By using window.postMessage
- D) By dispatching a DOM event

**3.** What problem does lifting state up solve?
- A) Slow performance
- B) Multiple components needing access to the same data
- C) CSS styling issues
- D) Bundle size

**Answers:** 1. B 2. B 3. B

---

## 6. CHECKPOINT

> If `TaskForm` needs to add a task and `TaskList` needs to display tasks, where should the tasks state live?

**Answer:** In their closest common parent component (currently `App`), passed via props and callbacks.

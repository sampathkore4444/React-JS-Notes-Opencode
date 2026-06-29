# Lesson 5: Forms & Events — User Input

## 1. CONCEPT — Controlled Components

> **Real-world analogy:** A controlled component is like a remote control car with a tether. The controller (React) sends signals through the tether, and the car (input) responds exactly as told. Without the tether (uncontrolled), the car moves on its own and you lose track of where it is. React "controls" what the input displays, so there's never a mismatch.

In React, form inputs should be **controlled** — meaning React controls the input value, not the DOM.

```jsx
// Controlled input: React owns the value
const [email, setEmail] = useState('');

<input
  value={email}                          // React controls the value
  onChange={(e) => setEmail(e.target.value)}  // React updates on type
/>
```

**Why?** Because React needs to be the single source of truth for form data. This lets you validate, transform, and react to input changes instantly.

## 2. CODE ALONG — A Proper Task Form

Create `src/components/TaskForm.jsx`:

```jsx
// Import useState for local form state (title and priority)
import { useState } from 'react';

// TaskForm receives onAddTask as a prop — a function to call when submitting
function TaskForm({ onAddTask }) {
  const [title, setTitle] = useState('');       // controlled: React owns the input value
  const [priority, setPriority] = useState('Medium');

  function handleSubmit(e) {
    e.preventDefault();                          // stop browser from refreshing the page
    if (!title.trim()) return;                   // ignore empty submissions
    onAddTask({ title: title.trim(), priority }); // send data UP to parent via callback
    setTitle('');                                // reset form to empty state
    setPriority('Medium');
  }

  return (
    // onSubmit is triggered when the form is submitted (button click or Enter key)
    <form onSubmit={handleSubmit} style={{
      background: '#f8f9fa',
      padding: '20px',
      borderRadius: '8px',
      marginBottom: '20px',
    }}>
      <h3>Add New Task</h3>

      <div style={{ marginBottom: '12px' }}>
        <label style={{ display: 'block', marginBottom: '4px' }}>Title:</label>
        <input
          type="text"
          value={title}                          // React controls what's in the box
          onChange={(e) => setTitle(e.target.value)}  // every keystroke updates state
          placeholder="What needs to be done?"
          style={{ width: '100%', padding: '8px', fontSize: '16px', boxSizing: 'border-box' }}
          autoFocus
        />
      </div>

      <div style={{ marginBottom: '12px' }}>
        <label style={{ display: 'block', marginBottom: '4px' }}>Priority:</label>
        {/* Controlled select: value bound to priority state */}
        <select
          value={priority}
          // onChange fires when the user picks a different option
          onChange={(e) => setPriority(e.target.value)}
          style={{ width: '100%', padding: '8px', fontSize: '16px' }}
        >
          {/* value attribute is what gets stored in state; text between tags is what the user sees */}
          <option value="High">High</option>
          <option value="Medium">Medium</option>
          <option value="Low">Low</option>
        </select>
      </div>

      {/* type="submit" triggers the form's onSubmit handler when clicked */}
      <button type="submit" style={{
        padding: '10px 24px',
        fontSize: '16px',
        backgroundColor: '#0066cc',
        color: 'white',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
      }}>
        Add Task
      </button>
    </form>
  );
}

export default TaskForm;
```

Now update `src/App.jsx` to use the form:

```jsx
import { useState } from 'react';
import Header from './components/Header';
import TaskForm from './components/TaskForm';
import Footer from './components/Footer';

function App() {
  // State holds the full list of tasks, each with id, title, done, and priority
  const [tasks, setTasks] = useState([
    { id: 1, title: 'Learn React', done: false, priority: 'High' },
    { id: 2, title: 'Build TaskFlow', done: false, priority: 'Medium' },
    { id: 3, title: 'Deploy app', done: false, priority: 'Low' },
  ]);

  // addTask receives the new task object from TaskForm and appends it to the array
  // Using a functional updater (prev => ...) ensures we always have the latest state
  function addTask(newTask) {
    setTasks(prev => [
      ...prev,                          // Spread existing tasks into the new array
      { id: Date.now(), ...newTask, done: false },  // Add the new task with a unique id
    ]);
  }

  // toggleTask flips the done property of the task matching the given id
  function toggleTask(id) {
    setTasks(tasks.map(t =>
      t.id === id ? { ...t, done: !t.done } : t
    ));
  }

  // deleteTask removes the task with the given id using filter
  function deleteTask(id) {
    setTasks(tasks.filter(t => t.id !== id));
  }

  return (
    <div style={{ padding: '20px', maxWidth: '600px', margin: '0 auto' }}>
      <Header />
      <TaskForm onAddTask={addTask} />

      <div>
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
          }}>
            <input
              type="checkbox"
              checked={task.done}
              onChange={() => toggleTask(task.id)}
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
            <button onClick={() => deleteTask(task.id)} style={{
              cursor: 'pointer', border: 'none', background: 'none', fontSize: '18px',
            }}>
              🗑️
            </button>
          </div>
        ))}
      </div>

      <Footer />
    </div>
  );
}

export default App;
```

## 3. WHY FORMS ARE SPECIAL

Forms are the most common way users interact with apps. React's controlled component pattern gives you:
- **Instant validation** (show error as user types)
- **Conditional fields** (show/hide based on other inputs)
- **Formatting** (auto-format phone numbers, credit cards)
- **Submit handling** (prevent default, validate, send)

## 4. EXERCISE

Add a "Search" input above the task list that filters tasks in real-time as you type.

**Solution:**
```jsx
// Add this state in App:
const [search, setSearch] = useState('');

// Add this JSX before the task list:
<div style={{ marginBottom: '12px' }}>
  <input
    type="text"
    placeholder="Search tasks..."
    value={search}
    onChange={(e) => setSearch(e.target.value)}
    style={{ width: '100%', padding: '8px', fontSize: '16px', boxSizing: 'border-box' }}
  />
</div>

// Change tasks.map to:
{tasks
  .filter(task =>
    task.title.toLowerCase().includes(search.toLowerCase())
  )
  .map(task => (
    // ... existing task rendering
  ))
}
```

## 5. EXTRA EXERCISE

Add a 'priority filter' dropdown that filters the task list. When the user selects a priority, only tasks with that priority display. Use controlled component pattern for the select element.

**Solution:**
```jsx
// Add this state in App:
const [filter, setFilter] = useState('All');

// Add this JSX before the task list, after '<TaskForm onAddTask={addTask} />':
<div style={{ marginBottom: '12px' }}>
  <label style={{ display: 'block', marginBottom: '4px' }}>Filter by priority:</label>
  <select
    value={filter}
    onChange={(e) => setFilter(e.target.value)}
    style={{ width: '100%', padding: '8px', fontSize: '16px' }}
  >
    <option value="All">All</option>
    <option value="High">High</option>
    <option value="Medium">Medium</option>
    <option value="Low">Low</option>
  </select>
</div>

// Replace {tasks.map(task => (} with:
{tasks
  .filter(task => filter === 'All' || task.priority === filter)
  .map(task => (
    // ... existing task rendering
  ))
}
```

### CHALLENGE EXERCISE

Add a "Due Date" field to the TaskForm. Include a date input that lets the user set a due date for each task. Display the due date in each task card, with overdue tasks highlighted in red.

**Solution:**

Add due date state in TaskForm:
```jsx
const [dueDate, setDueDate] = useState('');
```

Add the input (before priority select):
```jsx
<div style={{ marginBottom: '12px' }}>
  <label style={{ display: 'block', marginBottom: '4px' }}>Due Date:</label>
  <input
    type="date"
    value={dueDate}
    onChange={(e) => setDueDate(e.target.value)}
    style={{ width: '100%', padding: '8px', fontSize: '16px', boxSizing: 'border-box' }}
  />
</div>
```

Update handleSubmit:
```jsx
onAddTask({ title: title.trim(), priority, dueDate });
setDueDate('');
```

In App.jsx, update initial tasks with due dates and display:
```jsx
// Initial tasks:
{ id: 1, title: 'Learn React', done: false, priority: 'High', dueDate: '2025-12-31' },

// In task rendering, below priority badge:
<span style={{
  fontSize: '12px',
  padding: '2px 8px',
  borderRadius: '4px',
  color: 'white',
  backgroundColor: task.dueDate && new Date(task.dueDate) < new Date()
    ? '#dc2626' : '#6b7280',
}}>
  {task.dueDate ? `Due: ${task.dueDate}` : 'No due date'}
</span>
```

## 6. COMMON MISTAKES

**Mistake: Forgetting `e.preventDefault()`**
```jsx
function handleSubmit(e) {
  // ❌ Without this, the page refreshes when form submits
  // You lose all React state!

  // ✅ ALWAYS call this first in form handlers
  e.preventDefault();
  // ... your logic
}
```

**Mistake: Checkbox value instead of checked**
```jsx
// ❌ Wrong for checkboxes
<input type="checkbox" value={agreed} />

// ✅ Checkboxes use `checked`, not `value`
<input type="checkbox" checked={agreed} onChange={(e) => setAgreed(e.target.checked)} />
```

### 🐛 DEBUGGING WITH DEVTOOLS

Add a `console.log({ title, priority, dueDate })` inside `handleSubmit` before `onAddTask`. Watch the console every time you submit — this verifies your controlled component values are correct. In DevTools Components tab, select `<TaskForm>` to watch its local state change on every keystroke. If the input value doesn't update, check that `onChange` calls `setTitle`.

### ⚙️ UNDER THE HOOD

In controlled components, React stores the input value in state. Each keystroke triggers `onChange` → `setState` → re-render → input gets new `value` prop. This creates a single source of truth. In uncontrolled components (using `ref`), the DOM stores the value — React doesn't know it until you read the ref. Controlled is preferred because React always knows the current value.

### 🏭 PRODUCTION PATTERN

In real apps, you'd use a form library like React Hook Form or Formik for validation and performance. But the controlled component pattern (value + onChange) is the foundation all libraries build on. Understanding it deeply prevents 90% of form bugs.

### 💼 INTERVIEW READY

**Q:** What's the difference between controlled and uncontrolled components?
**A:** Controlled: React manages the input value via state (`value={val} onChange={...}`). Uncontrolled: the DOM manages the value, and you read it via a ref when needed. Controlled gives you instant validation, transformation, and conditional rendering. Uncontrolled is simpler for non-interactive forms but React is "blind" to the current value until you read the ref.

### 📝 MINI QUIZ

**1.** What does `e.preventDefault()` do in a form submit handler?
- A) Stops the form from sending data
- B) Prevents the browser's default form submission (page refresh)
- C) Clears all input fields
- D) Validates the form data

**2.** In a controlled input, what prop connects the input display to React state?
- A) `name`
- B) `value`
- C) `defaultValue`
- D) `ref`

**3.** What happens to React state if you forget `e.preventDefault()`?
- A) Nothing — React handles it
- B) The page refreshes and all React state is lost
- C) Only the form resets
- D) An error is thrown

**Answers:** 1. B 2. B 3. B

## 7. CHECKPOINT

> Why do we use `value={title} onChange={...}` on inputs instead of letting the DOM manage the value?

**Answer:** So React is the single source of truth for form data, enabling instant validation, transformation, and controlled behavior.

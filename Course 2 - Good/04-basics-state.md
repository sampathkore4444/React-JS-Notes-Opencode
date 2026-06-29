# Lesson 4: State — Making Your App Interactive

## 1. CONCEPT — State Is Component Memory

> **Real-world analogy:** State is like a light switch in your room. The switch has two positions — on and off. Flipping it changes the room's state. React state works the same way: you declare the possible states, and when state changes, React automatically updates the UI (the room) to match.

Props are data passed **into** a component. State is data that lives **inside** a component and can change over time.

When state changes, **the component re-renders** (updates the UI automatically).

```jsx
import { useState } from 'react';  // Step 1: import the useState hook

function Counter() {
  // Step 2: declare state - returns array [currentValue, setterFunction]
  const [count, setCount] = useState(0);
  //  count = current value (starts at 0)
  //  setCount = function to update it
  //  useState(0) = initial value is 0

  return (
    <div>
      <p>Count: {count}</p>          {/* count is displayed here */}
      <button onClick={() => setCount(count + 1)}>  {/* Step 3: update on click */}
        Increment
      </button>
    </div>
  );
}
```

**The flow:** user clicks → `setCount` runs → React re-renders component → new `count` value displayed.

## 2. CODE ALONG — Add State to TaskFlow

We'll make the task list dynamic. Update `src/App.jsx`:

```jsx
import { useState } from 'react';
import Header from './components/Header';
import Footer from './components/Footer';

function App() {
  // State: array of tasks
  // Each task is an object with id, title, and done status
  const [tasks, setTasks] = useState([
    { id: 1, title: 'Learn React', done: false },
    { id: 2, title: 'Build TaskFlow', done: false },
    { id: 3, title: 'Deploy app', done: false },
  ]);

  // State: new task input — holds whatever the user is currently typing
  const [newTaskTitle, setNewTaskTitle] = useState('');

  function addTask() {
    if (!newTaskTitle.trim()) return;        // guard: skip empty input
    const newTask = {
      id: Date.now(),                         // unique ID from timestamp
      title: newTaskTitle,
      done: false,
    };
    setTasks([...tasks, newTask]);            // spread existing + add new (immutable!)
    setNewTaskTitle('');                      // clear input after adding
  }

  function toggleTask(id) {
    setTasks(
      tasks.map(task =>                       // map over array
        task.id === id ? { ...task, done: !task.done } : task
      // if match: spread task + flip done  // if no match: keep unchanged
      )
    );
  }

  return (
    <div style={{ padding: '20px', maxWidth: '600px', margin: '0 auto' }}>
      <Header />

      {/* Add task form */}
      <div style={{ display: 'flex', gap: '8px', marginBottom: '20px' }}>
        <input
          style={{ flex: 1, padding: '8px', fontSize: '16px' }}
          value={newTaskTitle}
          onChange={(e) => setNewTaskTitle(e.target.value)}
          placeholder="Add a new task..."
          onKeyDown={(e) => e.key === 'Enter' && addTask()}
        />
        <button
          style={{ padding: '8px 16px', fontSize: '16px', cursor: 'pointer' }}
          onClick={addTask}
        >
          Add
        </button>
      </div>

      {/* Task list */}
      <div>
        {tasks.map(task => (
          <div
            key={task.id}
            onClick={() => toggleTask(task.id)}
            style={{
              padding: '12px',
              marginBottom: '8px',
              border: '1px solid #ddd',
              borderRadius: '8px',
              cursor: 'pointer',
              textDecoration: task.done ? 'line-through' : 'none',
              backgroundColor: task.done ? '#f0f0f0' : 'white',
              userSelect: 'none',
            }}
          >
            <input type="checkbox" checked={task.done} readOnly />
            {' '}
            {task.title}
          </div>
        ))}
      </div>

      <Footer />
    </div>
  );
}

export default App;
```

Run `npm run dev` if not already running. **Test it:**
- Type a task and click "Add" or press Enter
- Click a task to toggle it as done
- Notice the UI updates automatically

## 3. THE THREE RULES OF STATE

1. **Don't modify state directly** — use the setter function
   ```jsx
   // ❌ Wrong — React won't re-render
   tasks.push(newTask);
   // ✅ Correct
   setTasks([...tasks, newTask]);
   ```

2. **State updates are async** — you can't read new state immediately after setting
   ```jsx
   setCount(5);
   console.log(count);  // ❌ Still old value!
   ```

3. **Use functional update when new state depends on old**
   ```jsx
   // ❌ Bug: if setCount runs twice, both use same old value
   setCount(count + 1);
   setCount(count + 1);  // Result: count+1, not count+2
   // ✅ Correct: uses previous state
   setCount(prev => prev + 1);
   setCount(prev => prev + 1);  // Result: count+2
   ```

## 4. WHY THIS MATTERS

State is the heart of React. Every interactive app needs state: form inputs, toggle buttons, fetched data, user preferences. Understanding `useState` deeply (including the async behavior and immutability) prevents 90% of React bugs beginners face.

## 5. EXERCISE

Add a "Delete" button to each task. When clicked, remove that task from the array.

**Hints:**
- Add a button next to each task title
- Use `filter()` to create a new array without the deleted item
- Use `e.stopPropagation()` to prevent the click from also toggling the task

**Solution:**
```jsx
// Inside the map, next to {task.title}:
<button
  onClick={(e) => {
    e.stopPropagation();
    setTasks(tasks.filter(t => t.id !== task.id));
  }}
  style={{ marginLeft: 'auto', cursor: 'pointer' }}
>
  ❌
</button>

// Also make the task row a flex container:
style={{
  display: 'flex',
  alignItems: 'center',
  gap: '8px',
  // ... existing styles
}}
```

## 6. EXTRA EXERCISE

Add a 'priority filter' — three buttons (All, High, Medium, Low) that filter the displayed tasks. When 'High' is selected, only high-priority tasks show. Use a new state variable called `filter`.

**Solution:**
```jsx
// Add this state near the other useState calls:
const [filter, setFilter] = useState('All');

// Add this JSX between the input form and the task list:
<div style={{ marginBottom: '12px', display: 'flex', gap: '8px' }}>
  {['All', 'High', 'Medium', 'Low'].map(f => (
    <button
      key={f}
      onClick={() => setFilter(f)}
      style={{
        padding: '6px 16px',
        cursor: 'pointer',
        fontWeight: filter === f ? 'bold' : 'normal',
        backgroundColor: filter === f ? '#0066cc' : '#eee',
        color: filter === f ? 'white' : '#333',
        border: 'none',
        borderRadius: '4px',
      }}
    >
      {f}
    </button>
  ))}
</div>

// Replace {tasks.map(...)} with:
{tasks
  .filter(task => filter === 'All' || task.priority === filter)
  .map(task => (
    // ... existing task rendering
  ))
}

// But wait — our current tasks don't have a priority field. Add it:
const [tasks, setTasks] = useState([
  { id: 1, title: 'Learn React', done: false, priority: 'High' },
  { id: 2, title: 'Build TaskFlow', done: false, priority: 'Medium' },
  { id: 3, title: 'Deploy app', done: false, priority: 'Low' },
]);
```

### CHALLENGE EXERCISE

Add an "Edit Task" feature. Each task gets an "Edit" button. When clicked, the task title becomes an input field pre-filled with the current title. A "Save" button confirms the edit, and "Cancel" reverts it. Use state to track which task is being edited.

**Solution:**

Add this state in App:
```jsx
const [editingId, setEditingId] = useState(null);
const [editText, setEditText] = useState('');
```

Add these functions:
```jsx
function startEdit(task) {
  setEditingId(task.id);
  setEditText(task.title);
}

function saveEdit(id) {
  setTasks(tasks.map(task =>
    task.id === id ? { ...task, title: editText } : task
  ));
  setEditingId(null);
}

function cancelEdit() {
  setEditingId(null);
}
```

In the task rendering, replace `{task.title}` with:
```jsx
{editingId === task.id ? (
  <span style={{ display: 'flex', gap: '4px', flex: 1 }}>
    <input
      value={editText}
      onChange={e => setEditText(e.target.value)}
      style={{ flex: 1, padding: '4px' }}
      autoFocus
      onKeyDown={e => {
        if (e.key === 'Enter') saveEdit(task.id);
        if (e.key === 'Escape') cancelEdit();
      }}
    />
    <button onClick={() => saveEdit(task.id)} style={{ cursor: 'pointer' }}>💾</button>
    <button onClick={cancelEdit} style={{ cursor: 'pointer' }}>✕</button>
  </span>
) : (
  <span style={{ flex: 1, textDecoration: task.done ? 'line-through' : 'none' }}>
    {task.title}
    <button onClick={() => startEdit(task)} style={{ marginLeft: '8px', cursor: 'pointer', fontSize: '12px' }}>
      ✏️
    </button>
  </span>
)}
```

## 7. COMMON MISTAKES

**Mistake: Direct state mutation**
```jsx
// ❌ Mutating array directly
tasks[0].done = true;
setTasks(tasks);
// React sees same array reference — no re-render!

// ✅ Create new array with updated item
setTasks(tasks.map(task =>
  task.id === id ? { ...task, done: true } : task
));
```

**Mistake: Forgetting spread `...` for objects**
```jsx
const [user, setUser] = useState({ name: 'Alex', age: 25 });
// ❌ This replaces the entire object (loses name)
setUser({ age: 26 });
// ✅ This preserves other properties
setUser(prev => ({ ...prev, age: 26 }));
```

### 🐛 DEBUGGING WITH DEVTOOLS

In React DevTools → Components tab, select `<App>`. The right panel shows "hooks" section: `State: 3` (tasks array), `State: ''` (newTaskTitle). Click the "eye" icon to inspect complex state objects. Type a new task and watch the state values update live. If state doesn't update when expected, check if you called the setter function.

### ⚙️ UNDER THE HOOD

React stores hook state in a linked list attached to the fiber node (its internal component representation). Each `useState` call gets an index in this list — that's why hooks must be called in the same order every render. When you call `setState`, React schedules a re-render (batched in React 18+), compares new/old virtual DOM, and applies DOM mutations.

### 🏭 PRODUCTION PATTERN

Use functional updates when new state depends on old: `setCount(prev => prev + 1)`. This avoids stale closure bugs, especially in effects and callbacks. For objects, always spread: `setUser(prev => ({ ...prev, name: 'new' }))`.

### 💼 INTERVIEW READY

**Q:** What happens if you call `setState` with the same value as the current state?
**A:** React uses `Object.is` comparison. If the value is the same (same reference for objects/arrays), React bails out without re-rendering the component or its children. This is an optimization — but be careful: mutating an object then calling `setState(obj)` passes the same reference, so React skips the re-render.

### 📝 MINI QUIZ

**1.** What triggers a component re-render?
- A) Changing a local variable
- B) Calling the state setter function (setCount)
- C) Modifying props directly
- D) Calling a regular function

**2.** Why should you NOT mutate state directly (e.g., `tasks.push(newTask)`)?
- A) It throws a runtime error
- B) React won't detect the change and won't re-render
- C) It's slower than spreading
- D) Mutations are not allowed in JavaScript

**3.** What does `useState(0)` return?
- A) A number (0)
- B) An array `[count, setCount]`
- C) An object `{ value: 0, setValue: fn }`
- D) A tuple `(0, fn)`

**Answers:** 1. B 2. B 3. B

## 8. CHECKPOINT

> What happens to a component when its state changes?

**Answer:** React re-renders the component, and the UI updates to match the new state.

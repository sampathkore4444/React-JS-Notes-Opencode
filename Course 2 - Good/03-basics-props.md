# Lesson 3: Props — Making Components Reusable

## 1. CONCEPT — Props Are Function Arguments for Components

> **Real-world analogy:** Props are like filling out a wedding invitation template. The template (component) stays the same, but each invitation gets different names, dates, and locations (props). One component renders infinite variations based on what data you pass in.

Props let you pass data **from parent to child**. They're like function parameters — and they are **read-only**.

```jsx
// Template: propName={value}
<Greeting name="Alex" />   // name is a prop

// Inside the component:
function Greeting(props) {
  return <h1>Hello, {props.name}!</h1>;
}
// Or destructured:
function Greeting({ name }) {
  return <h1>Hello, {name}!</h1>;
}
```

**You cannot change props inside the child component.** The parent owns the data.

## 2. CODE ALONG

Update `src/components/TaskList.jsx` to accept tasks as props:
```jsx
function TaskList({ tasks, title }) {   // { tasks, title } = destructuring props object
  return (
    <div>
      <h2>{title}</h2>                  {/* title comes from props */}
      <ul>
        {tasks.map(task => (             // .map() loops through array, returns JSX for each
          <li key={task}>{task}</li>     // key prop helps React track list items
        ))}
      </ul>
    </div>
  );
}
export default TaskList;
```

Now update `src/App.jsx` to pass props:
```jsx
import Header from './components/Header';
import TaskList from './components/TaskList';
import Footer from './components/Footer';

function App() {
  const workTasks = ['Learn React', 'Build TaskFlow', 'Deploy app'];       // array for work
  const personalTasks = ['Buy groceries', 'Read a book', 'Exercise'];      // array for personal

  return (
    <div style={{ padding: '20px', maxWidth: '600px', margin: '0 auto' }}>
      <Header />
      {/* Pass tasks array as "tasks" prop, string as "title" prop */}
      <TaskList tasks={workTasks} title="Work Tasks" />       {/* same component, different data */}
      <TaskList tasks={personalTasks} title="Personal Tasks" /> {/* reusing TaskList! */}
      <Footer />
    </div>
  );
}
export default App;
```

**Check your browser.** You now reuse `TaskList` twice with different data. One component, two different outputs.

## 3. WHY THIS MATTERS

Props make components **reusable**. A `UserCard` component can render 100 different users by receiving different props. A `Button` component can render primary, danger, or ghost variants based on a `variant` prop.

Without props, every component would be hardcoded and useless for real apps.

## 4. EXERCISE

1. Create a `TaskCard` component that accepts `title` and `priority` props
2. The component should show the task title and color-code the priority (High=red, Medium=orange, Low=green)
3. Use it in App.jsx with at least 3 tasks of different priorities

**Solution:**
```jsx
// src/components/TaskCard.jsx
function TaskCard({ title, priority }) {
  const colorMap = { High: '#ff4444', Medium: '#ff8800', Low: '#00aa00' };
  const colors = {
    High: { bg: '#ffe0e0', text: '#cc0000' },
    Medium: { bg: '#fff3e0', text: '#cc7700' },
    Low: { bg: '#e0f5e0', text: '#007700' },
  };

  return (
    <div style={{
      border: '1px solid #ddd',
      borderRadius: '8px',
      padding: '12px',
      marginBottom: '8px',
      backgroundColor: colors[priority]?.bg || '#fff',
    }}>
      <div style={{ fontWeight: 'bold', marginBottom: '4px' }}>{title}</div>
      <span style={{
        backgroundColor: colors[priority]?.text || '#666',
        color: 'white',
        padding: '2px 8px',
        borderRadius: '4px',
        fontSize: '12px',
      }}>
        {priority}
      </span>
    </div>
  );
}

export default TaskCard;
```

```jsx
// App.jsx
import TaskCard from './components/TaskCard';
// ... other imports

// Inside App component:
<div style={{ marginTop: '20px' }}>
  <h2>My Tasks</h2>
  <TaskCard title="Fix login bug" priority="High" />
  <TaskCard title="Update README" priority="Medium" />
  <TaskCard title="Buy domain" priority="Low" />
</div>
```

## 5. EXTRA EXERCISE

Create a `UserProfile` component that accepts `name`, `age`, `avatar` (URL string), and `role` props. Display them in a styled card. Use it in App.jsx with at least 3 different users.

**Solution:**
```jsx
// src/components/UserProfile.jsx
function UserProfile({ name, age, avatar, role }) {
  return (
    <div style={{
      border: '1px solid #ddd',
      borderRadius: '12px',
      padding: '16px',
      marginBottom: '12px',
      display: 'flex',
      alignItems: 'center',
      gap: '16px',
      backgroundColor: '#fafafa',
    }}>
      <img
        src={avatar}
        alt={name}
        style={{ width: '60px', height: '60px', borderRadius: '50%' }}
      />
      <div>
        <h3 style={{ margin: 0 }}>{name}</h3>
        <p style={{ margin: '4px 0', color: '#666' }}>{role} &middot; Age {age}</p>
      </div>
    </div>
  );
}

export default UserProfile;
```

```jsx
// App.jsx
import UserProfile from './components/UserProfile';

function App() {
  return (
    <div style={{ padding: '20px', maxWidth: '600px', margin: '0 auto' }}>
      <h1>Team Members</h1>
      <UserProfile
        name="Alice Johnson"
        age={28}
        avatar="https://i.pravatar.cc/150?u=alice"
        role="Frontend Developer"
      />
      <UserProfile
        name="Bob Smith"
        age={35}
        avatar="https://i.pravatar.cc/150?u=bob"
        role="Backend Developer"
      />
      <UserProfile
        name="Carol Lee"
        age={24}
        avatar="https://i.pravatar.cc/150?u=carol"
        role="Designer"
      />
    </div>
  );
}
export default App;
```

### CHALLENGE EXERCISE

Create a `StatusBadge` component that accepts `status` prop with values 'online', 'away', or 'busy'. The badge shows a colored circle (green/orange/red) and the status text. Then create a `UserCard` component that uses `StatusBadge` and accepts `name`, `email`, and `status` props. Render 3 users.

**Solution:**
```jsx
// src/components/StatusBadge.jsx
function StatusBadge({ status }) {
  const colors = {
    online: { bg: '#dcfce7', dot: '#22c55e' },
    away: { bg: '#fef3c7', dot: '#f59e0b' },
    busy: { bg: '#fee2e2', dot: '#ef4444' },
  };
  const style = colors[status] || colors.online;

  return (
    <span style={{
      display: 'inline-flex', alignItems: 'center', gap: '6px',
      padding: '4px 12px', borderRadius: '20px', fontSize: '14px',
      backgroundColor: style.bg,
    }}>
      <span style={{
        width: '8px', height: '8px', borderRadius: '50%',
        backgroundColor: style.dot, display: 'inline-block',
      }} />
      {status}
    </span>
  );
}
export default StatusBadge;

// src/components/UserCard.jsx
import StatusBadge from './StatusBadge';
function UserCard({ name, email, status }) {
  return (
    <div style={{
      border: '1px solid #e2e8f0', borderRadius: '8px',
      padding: '16px', marginBottom: '12px',
      display: 'flex', alignItems: 'center', gap: '12px',
    }}>
      <div style={{
        width: '48px', height: '48px', borderRadius: '50%',
        backgroundColor: '#3b82f6', color: 'white',
        display: 'flex', alignItems: 'center', justifyContent: 'center',
        fontSize: '20px', fontWeight: 'bold',
      }}>
        {name[0]}
      </div>
      <div style={{ flex: 1 }}>
        <div style={{ fontWeight: 'bold' }}>{name}</div>
        <div style={{ color: '#64748b', fontSize: '14px' }}>{email}</div>
      </div>
      <StatusBadge status={status} />
    </div>
  );
}
export default UserCard;
```

In App.jsx:
```jsx
import UserCard from './components/UserCard';

// Inside App return, add:
<h2>Team Members</h2>
<UserCard name="Alice Chen" email="alice@example.com" status="online" />
<UserCard name="Bob Smith" email="bob@example.com" status="away" />
<UserCard name="Carol Davis" email="carol@example.com" status="busy" />
```

## 6. COMMON MISTAKES

**Mistake: Mutating props**
```jsx
// ❌ NEVER do this — props are read-only
function BadComponent({ name }) {
  name = 'Changed';  // Mutation!
  return <p>{name}</p>;
}

// ✅ If you need to change data, use state (next lesson)
```

**Mistake: Forgetting curly braces {} for dynamic values**
```jsx
// ❌ String literal, not variable
<TaskList title="title" />     // renders "title" as text
<TaskList tasks="workTasks" /> // renders "workTasks" as text

// ✅ Dynamic value from JavaScript
<TaskList title={title} />     // renders variable value
<TaskList tasks={workTasks} /> // renders variable value
```

**Mistake: Missing `key` prop on mapped elements**
```jsx
// ❌ React warns: "Each child in a list should have a unique key"
{tasks.map(task => <li>{task}</li>)}

// ✅ Unique key helps React track items
{tasks.map(task => <li key={task}>{task}</li>)}
```

### 🐛 DEBUGGING WITH DEVTOOLS

Select any `<TaskList>` in the Components tab. The right panel shows "props" with the current values: `tasks: Array(3)` and `title: "Work Tasks"`. If a prop shows as `undefined`, you forgot to pass it. If it shows wrong data, check the parent that's passing it. This is the fastest way to trace data flow bugs.

### ⚙️ UNDER THE HOOD

Props are a single JavaScript object passed as the first argument. The destructuring syntax `function Card({ title })` is syntactic sugar — it still receives one argument. React uses `Object.is` (like `===` but for NaN and -0) for shallow prop comparison, which determines whether `React.memo` wrapped components should skip re-rendering.

### 🏭 PRODUCTION PATTERN

For team projects, validate props with TypeScript interfaces. Without TypeScript, use `Component.propTypes = { ... }` as documentation. Prefer default parameter values: `function Button({ variant = 'primary', size = 'md' })` over defaultProps.

### 💼 INTERVIEW READY

**Q:** Can you modify a prop inside a child component? What happens if you try?
**A:** Technically you can (it's a JS object), but it's a React anti-pattern. Mutating props creates inconsistencies — the parent still references the original value, so the app state becomes unpredictable. React won't re-render the parent, and other children receiving the same prop won't see the change. If a child needs to modify data, call a callback passed from the parent.

### 📝 MINI QUIZ

**1.** What data direction do props flow?
- A) Child to parent (upward)
- B) Parent to child (downward)
- C) Between siblings (horizontal)
- D) Any direction

**2.** `function Card({ title })` — what's this called?
- A) Prop spreading
- B) Destructuring props
- C) State initialization
- D) Event binding

**3.** What is the `key` prop used for in lists?
- A) Styling list items
- B) Helping React identify which items changed
- C) Sorting the list
- D) Filtering items

**Answers:** 1. B 2. B 3. B

## 7. CHECKPOINT

> What happens if you try to change a prop value inside a component?

**Answer:** It will change the variable, but React won't re-render and other components won't see the change. Props are read-only — state is for mutable data.

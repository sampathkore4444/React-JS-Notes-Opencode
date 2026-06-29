# Lesson 23: TypeScript with React — Catch Bugs Before Runtime

## 1. CONCEPT — Why TypeScript?

> **Real-world analogy:** TypeScript is like having a spell-checker for your code. JavaScript is like writing a novel without spell-check — you won't know about typos until someone reads it (runtime errors). TypeScript underlines mistakes in RED as you type, catching bugs before anyone else sees them. It's not extra work — it's a safety net that saves debugging time.

**JavaScript:** `const user = getUser();` → you don't know what `user` has. You guess. You check the API. You break production.

**TypeScript:** `const user: User = getUser();` → you know exactly that `user` has `.name` (string), `.age` (number), `.email` (string | undefined). Your editor tells you. No guessing.

```typescript
// JavaScript — you discover bugs at runtime
function formatName(user) {
  return user.firstName.toUpperCase(); // ❌ TypeError if firstName is undefined
}

// TypeScript — you catch it while typing
function formatName(user: { firstName: string; lastName: string }): string {
  return `${user.firstName.toUpperCase()} ${user.lastName}`;
  // ✅ TypeScript warns if you try: formatName({}) — missing firstName!
}
```

### For React specifically, TypeScript catches:

| Bug | JS (caught when?) | TS (caught when?) |
|-----|-------------------|-------------------|
| Passing wrong prop type | Runtime — app breaks | Compile time — red underline |
| Accessing undefined property | Runtime — `Cannot read of undefined` | Compile time — optional chaining suggested |
| Calling function with wrong args | Runtime — unexpected behavior | Compile time — parameter mismatch |
| Typo in state update | Runtime — no error, silent bug | Compile time — property doesn't exist |

## 2. CODE ALONG — Migrate TaskFlow to TypeScript

### Step 1: Install TypeScript

```bash
npm install -D typescript @types/react @types/react-dom
```

Create `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx",
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true
  },
  "include": ["src"]
}
```

### Step 2: Define your types

`src/types/index.ts`:
\`\`\`typescript
// --- Domain Types ---

export interface Task {
  id: number;                          // number, always required
  title: string;                       // string, always required
  priority: 'High' | 'Medium' | 'Low'; // UNION type: only these three strings allowed
  done: boolean;
  createdAt?: string;                  // optional: might be undefined (marked with ?)
  completedAt?: string;                // optional
}

export interface Column {
  id: string;
  title: string;
  tasks: Task[];                       // array of Task objects
}

export type ColumnId = 'todo' | 'in-progress' | 'done';

// Record<K, V> is a utility type: object with keys K and values V
export type Columns = Record<ColumnId, Column>;
// Equivalent to: { 'todo': Column, 'in-progress': Column, 'done': Column }

// --- State Types ---

export interface BoardState {
  columns: Columns;
  loading: boolean;
  error: string | null;
}

export interface ThemeState {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

// --- Action Types ---

export interface AddTaskPayload {
  columnId: string;
  title: string;
  priority: Task['priority'];
}

export interface MoveTaskPayload {
  taskId: number;
  fromColumnId: string;
  toColumnId: string;
}

// --- Component Props ---

export interface TaskCardProps {
  task: Task;                          // typed as the Task interface
  columnId: string;
}

export interface ColumnProps {
  columnId: string;
}

export interface BoardProps {
  columns: Columns;
}

export interface HeaderProps {
  totalTasks: number;
}

// --- Event Handlers ---

export type ChangeHandler = (e: React.ChangeEvent<HTMLInputElement>) => void;
export type SubmitHandler = (e: React.FormEvent<HTMLFormElement>) => void;
\`\`\`

### Step 3: Typed components

`src/components/TaskCard.tsx`:
\`\`\`tsx
import { memo, useState } from 'react';
import type { TaskCardProps } from '../types';

const priorityColors: Record<string, { bg: string; text: string }> = {
  // TypeScript enforces: every key maps to { bg: string, text: string }
  High: { bg: '#fee2e2', text: '#dc2626' },
  Medium: { bg: '#fef3c7', text: '#d97706' },
  Low: { bg: '#dcfce7', text: '#16a34a' },
};

const TaskCard = memo(function TaskCard({ task, columnId }: TaskCardProps) {
  // TypeScript knows: task.title is string, task.priority is 'High'|'Medium'|'Low'
  // If you type task.piriority (typo), TypeScript flags it red!
  const colors = priorityColors[task.priority] ?? priorityColors.Medium;

  // TypeScript flags: <button onClick={task.title}> — Error! onClick expects function
  // TypeScript flags: {task.prioritye} — Error! property doesn't exist on Task

  return (
    <div style={{
      backgroundColor: 'white',
      padding: '10px',
      marginBottom: '8px',
      borderRadius: '6px',
      boxShadow: '0 1px 3px rgba(0,0,0,0.1)',
      cursor: 'pointer',
    }}>
      <div>{task.title}</div>       {/* ✅ Safe — TypeScript knows this is a string */}
      <span style={{
        display: 'inline-block',
        fontSize: '11px',
        padding: '2px 6px',
        borderRadius: '4px',
        marginTop: '6px',
        backgroundColor: colors.bg,
        color: colors.text,
      }}>
        {task.priority}
      </span>
    </div>
  );
});

export default TaskCard;
\`\`\`

`src/components/Column.tsx`:
```tsx
import { useState, useRef, useEffect } from 'react';
import { useBoardStore } from '../stores/boardStore';
import TaskCard from './TaskCard';
import type { ColumnProps } from '../types';

const Column = ({ columnId }: ColumnProps) => {
  // TypeScript infers the return type of this selector
  const column = useBoardStore((state) => state.columns[columnId]);
  const addTask = useBoardStore((state) => state.addTask);
  const moveTask = useBoardStore((state) => state.moveTask);
  const deleteTask = useBoardStore((state) => state.deleteTask);

  const [showForm, setShowForm] = useState(false);
  const [newTitle, setNewTitle] = useState('');
  const [newPriority, setNewPriority] = useState<'High' | 'Medium' | 'Low'>('Medium');
  const inputRef = useRef<HTMLInputElement | null>(null);

  // Auto-focus
  useEffect(() => {
    if (showForm && inputRef.current) {
      inputRef.current.focus();
    }
  }, [showForm]);

  // TypeScript enforces: e is a React.FormEvent
  function handleAdd(e: React.FormEvent) {
    e.preventDefault();
    if (!newTitle.trim()) return;
    addTask(columnId, newTitle.trim(), newPriority);
    setNewTitle('');
    setNewPriority('Medium');
    setShowForm(false);
  }

  // TypeScript shows error if we try: deleteTask('todo', 'not-a-number')
  // because id is typed as number
  function handleDelete(taskId: number) {
    deleteTask(columnId, taskId);
  }

  return (
    <div style={{
      backgroundColor: '#f1f5f9',
      borderRadius: '8px',
      padding: '12px',
      minHeight: '200px',
    }}>
      <h3 style={{ margin: 0 }}>{column?.title}</h3>
      <span>{column?.tasks.length ?? 0}</span>

      {column?.tasks.map(task => (
        <TaskCard key={task.id} task={task} columnId={columnId} />
      ))}

      {showForm && (
        <form onSubmit={handleAdd}>
          <input
            ref={inputRef}
            value={newTitle}
            onChange={(e: React.ChangeEvent<HTMLInputElement>) =>
              setNewTitle(e.target.value)
            }
            placeholder="Task title"
          />
          <select
            value={newPriority}
            onChange={(e: React.ChangeEvent<HTMLSelectElement>) =>
              setNewPriority(e.target.value as 'High' | 'Medium' | 'Low')
            }
          >
            <option value="High">High</option>
            <option value="Medium">Medium</option>
            <option value="Low">Low</option>
          </select>
          <button type="submit">Add</button>
        </form>
      )}
    </div>
  );
};

export default Column;
```

`src/stores/boardStore.ts`:
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import type { Columns, Task } from '../types';

interface BoardStore {
  columns: Columns;
  addTask: (columnId: string, title: string, priority: Task['priority']) => void;
  moveTask: (taskId: number, fromColumnId: string, toColumnId: string) => void;
  deleteTask: (columnId: string, taskId: number) => void;
}

export const useBoardStore = create<BoardStore>()(
  persist(
    (set) => ({
      columns: {
        'todo': { id: 'todo', title: 'To Do', tasks: [] },
        'in-progress': { id: 'in-progress', title: 'In Progress', tasks: [] },
        'done': { id: 'done', title: 'Done', tasks: [] },
      },

      addTask: (columnId, title, priority) =>
        set((state) => {
          const newTask: Task = {
            id: Date.now(),
            title,
            priority,
            done: false,
            createdAt: new Date().toISOString(),
          };
          return {
            columns: {
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
          const task = state.columns[fromColumnId]?.tasks.find(t => t.id === taskId);
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
    }),
    { name: 'taskflow-board' }
  )
);
```

### Step 4: Typed hooks

`src/hooks/useLocalStorage.ts`:
```typescript
import { useState, useEffect } from 'react';

export function useLocalStorage<T>(key: string, initialValue: T): [T, (value: T | ((prev: T) => T)) => void] {
  const [value, setValue] = useState<T>(() => {
    try {
      const saved = localStorage.getItem(key);
      return saved ? JSON.parse(saved) : initialValue;
    } catch {
      return initialValue;
    }
  });

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

## 3. COMMON TYPESCRIPT PATTERNS IN REACT

```typescript
// 1. Generic component props
interface ListProps<T> {
  items: T[];
  renderItem: (item: T, index: number) => React.ReactNode;
}

function List<T>({ items, renderItem }: ListProps<T>) {
  return <div>{items.map(renderItem)}</div>;
}

// Usage: TypeScript infers T from the items array
<List items={tasks} renderItem={(task) => <div>{task.title}</div>} />

// 2. Children prop
interface LayoutProps {
  children: React.ReactNode;
  className?: string;
}

// 3. Event handlers
interface FormProps {
  onSubmit: (data: { email: string; password: string }) => void;
}

// 4. Optional props with defaults
interface ButtonProps {
  variant?: 'primary' | 'secondary';  // optional, defaults handled internally
  children: React.ReactNode;
  onClick?: () => void;
}

// 5. State with discriminated union
type FetchState<T> =
  | { status: 'loading' }
  | { status: 'error'; error: string }
  | { status: 'success'; data: T };

// Usage — TypeScript narrows the type based on status check:
function UserProfile({ userId }: { userId: number }) {
  const [state, setState] = useState<FetchState<User>>({ status: 'loading' });

  if (state.status === 'loading') return <Spinner />;
  if (state.status === 'error') return <Error message={state.error} />;
  // TypeScript knows: state.status === 'success' → state.data exists
  return <div>{state.data.name}</div>;
}
```

## 4. EXERCISE

Convert one of your existing `.jsx` files to `.tsx` and fix all TypeScript errors. Start with TaskCard.

**Steps:**
1. Rename `TaskCard.jsx` → `TaskCard.tsx`
2. Add type imports from `../types`
3. Add prop types to the function
4. Run `npx tsc --noEmit` — fix all red errors
5. Your app should still work (TypeScript compiles away, no runtime overhead)

### CHALLENGE EXERCISE

Define a `FetchState<T>` discriminated union type for API data, and create a generic `DataLoader` component that handles loading, error, and success states based on the type.

**Solution:**
\`\`\`typescript
// src/types/index.ts (add these types)
export type FetchState<T> =
  | { status: 'loading' }
  | { status: 'error'; error: string }
  | { status: 'success'; data: T };

// Generic component that works with any data type
interface DataLoaderProps<T> {
  state: FetchState<T>;
  children: (data: T) => React.ReactNode;
  loadingFallback?: React.ReactNode;
  errorFallback?: (error: string) => React.ReactNode;
}

// src/components/DataLoader.tsx
function DataLoader<T>({
  state,
  children,
  loadingFallback,
  errorFallback,
}: DataLoaderProps<T>) {
  // TypeScript narrows the type based on status check
  switch (state.status) {
    case 'loading':
      return loadingFallback || <div style={{
        padding: '20px', textAlign: 'center', color: '#94a3b8',
      }}>Loading...</div>;

    case 'error':
      return errorFallback ? errorFallback(state.error) : (
        <div style={{
          padding: '20px', textAlign: 'center', color: '#dc2626',
        }}>
          Error: {state.error}
        </div>
      );

    case 'success':
      // TypeScript knows: state.data exists here (not in other cases)
      return <>{children(state.data)}</>;
  }
}

// Usage:
function TaskList() {
  const [state, setState] = useState<FetchState<Task[]>>({ status: 'loading' });

  useEffect(() => {
    taskApi.getAll()
      .then(data => setState({ status: 'success', data }))
      .catch(err => setState({ status: 'error', error: err.message }));
  }, []);

  return (
    <DataLoader state={state} loadingFallback={<div>⏳ Fetching tasks...</div>}>
      {(tasks) => (
        <ul>
          {tasks.map(task => <li key={task.id}>{task.title}</li>)}
        </ul>
      )}
    </DataLoader>
  );
}
\`\`\`

## 5. COMMON MISTAKES

**Mistake: Using `any` everywhere (defeats TypeScript's purpose)**
```typescript
// ❌ Might as well use JavaScript
function process(data: any) {
  return data.name.toLowerCase(); // No type checking!
}

// ✅ Be specific
function process(data: { name: string }) {
  return data.name.toLowerCase();
}

// ✅ Or use generics if truly dynamic
function process<T extends { name: string }>(data: T) {
  return data.name.toLowerCase();
}
```

**Mistake: Not typing useState for complex objects**
```typescript
// ❌ TypeScript infers: useState<undefined>
const [user, setUser] = useState();
// Later: user.name → Error: Object is possibly 'undefined'

// ✅ Be explicit
const [user, setUser] = useState<User | null>(null);
// Now: user?.name → safe, forces null check
```

**Mistake: Over-typing**
```typescript
// ❌ TypeScript can infer this — unnecessary annotation
const [count, setCount] = useState<number>(0);

// ✅ TypeScript knows useState(0) returns number
const [count, setCount] = useState(0);
```

### 🐛 DEBUGGING WITH DEVTOOLS

TypeScript errors appear in the terminal (or VS Code Problems panel) BEFORE you run the app. Common React TS errors: `Type 'undefined' is not assignable to type 'string'` → you forgot optional chaining. `Property 'id' does not exist on type 'never'` → TypeScript inferred a narrow type from a filter. Always run `npx tsc --noEmit` in CI to catch type errors before deployment.

### ⚙️ UNDER THE HOOD

TypeScript's type system is erased at runtime — it compiles to plain JavaScript. This means TypeScript adds zero runtime overhead. All the interfaces, type annotations, and generics are gone in the final bundle. The compiler (`tsc`) checks types during development and build, but the JavaScript that runs is identical to what you'd write in plain JS. This is why TypeScript is "just JavaScript with types."

### 🏭 PRODUCTION PATTERN

All major React codebases use TypeScript (2025). The most common patterns: (1) Define types alongside components in `.tsx` files. (2) Use `interface` for objects/props, `type` for unions/utility types. (3) Prefer `type` for props when using `typeof` — `type Props = typeof Component.defaultProps`. (4) Use `satisfies` operator (TS 4.9+) for type-safe object literals without widening. (5) Always enable `strict: true` in tsconfig.

### 💼 INTERVIEW READY

**Q:** What does `strict: true` in tsconfig.json enable, and why should you always use it?
**A:** `strict: true` enables all strict type-checking options: `strictNullChecks` (catches undefined access), `noImplicitAny` (forces types everywhere), `strictFunctionTypes` (function parameter bivariance check), `strictPropertyInitialization` (class properties must be initialized), `noImplicitThis` (catch `this` reference errors), `alwaysStrict` (use strict mode). Without it, TypeScript lets through too many real bugs. For React projects, always use `strict: true`.

### 📝 MINI QUIZ

**1.** What does `strict: true` in tsconfig prevent?
- A) Slow compilation
- B) Accessing undefined properties (strictNullChecks)
- C) Using JSX
- D) Importing CSS

**2.** What's the TypeScript syntax for an optional property?
- A) `name: string?`
- B) `name?: string`
- C) `name: string | null`
- D) `name: Optional<string>`

**3.** What does `Record<string, number>` represent?
- A) An array of numbers
- B) An object where all values are numbers and keys are strings
- C) A function that returns a number
- D) A tuple of string and number

**Answers:** 1. B 2. B 3. B

## 6. CHECKPOINT

> What does `strict: true` in tsconfig.json enable, and why should you use it?

**Answer:** Enables all strict type-checking options including `strictNullChecks` (catches `undefined` access), `noImplicitAny` (forces types everywhere), and `strictFunctionTypes`. Without it, TypeScript lets through many bugs. **Always use `strict: true` for React projects.**

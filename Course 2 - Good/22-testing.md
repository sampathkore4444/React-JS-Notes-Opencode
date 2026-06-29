# Lesson 22: Testing — Make Your Code Reliable

## 1. CONCEPT — Why Test?

> **Real-world analogy:** Tests are like a safety net for a tightrope walker. You hope you never fall, but if you do, the net catches you. Every time you refactor (walk the rope), the tests (net) give you confidence that you won't crash. Without tests, you're walking without a net — one wrong step and your app breaks in production.

**Without tests:** You manually click through the app after every change. You miss edge cases. You ship bugs.

**With tests:** You run one command. The computer clicks through everything. You catch regressions instantly.

### Three types of testing:

```
              ┌─────────────────┐
              │   E2E Tests     │ ← User clicks through the app
              │  (Playwright)   │   Slow, broad, high confidence
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │ Integration     │ ← Component + hooks together
              │  (Testing Lib)  │   Medium speed, good coverage
              └────────┬────────┘
                       │
              ┌────────▼────────┐
              │  Unit Tests     │ ← One function in isolation
              │  (Vitest)       │   Fast, narrow, great for logic
              └─────────────────┘
```

**The Testing Trophy** (not pyramid):
Most of your tests should be **integration tests** — testing components the way users use them.

## 2. CODE ALONG — Test TaskFlow Components

### Install testing tools
```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

### Configure vitest

`vite.config.js` or `vitest.config.js`:
```js
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',           // Simulates browser DOM
    globals: true,                  // describe, it, expect available globally
    setupFiles: './src/test-setup.js',
  },
});
```

`src/test-setup.js`:
```js
import '@testing-library/jest-dom/vitest';
```

Add to `package.json` scripts:
```json
"scripts": {
  "test": "vitest",
  "test:run": "vitest run",
  "test:coverage": "vitest run --coverage"
}
```

### Test 1: Unit test — pure reducer function

`src/__tests__/boardReducer.test.js`:
```js
import { describe, it, expect } from 'vitest';
import { boardReducer, initialColumns, ACTIONS } from '../reducers/boardReducer';

describe('boardReducer', () => {
  it('adds a task to the specified column', () => {
    const state = structuredClone(initialColumns);     // start with fresh state

    const next = boardReducer(state, {                  // call reducer directly
      type: ACTIONS.ADD_TASK,
      payload: { columnId: 'todo', title: 'New task', priority: 'High' },
    });

    // Assertions — what should be true after the action
    expect(next['todo'].tasks).toHaveLength(            // one more task than before
      initialColumns['todo'].tasks.length + 1
    );
    expect(next['todo'].tasks.at(-1).title).toBe('New task');  // correct title
    expect(next['todo'].tasks.at(-1).priority).toBe('High');   // correct priority
  });

  it('moves a task between columns', () => {
    const state = structuredClone(initialColumns);
    const taskId = state['todo'].tasks[0].id;

    const next = boardReducer(state, {
      type: ACTIONS.MOVE_TASK,
      payload: { taskId, fromColumnId: 'todo', toColumnId: 'done' },
    });

    expect(next['todo'].tasks.find(t => t.id === taskId)).toBeUndefined();
    expect(next['done'].tasks.find(t => t.id === taskId)).toBeDefined();
  });

  it('deletes a task', () => {
    const state = structuredClone(initialColumns);
    const taskId = state['todo'].tasks[0].id;

    const next = boardReducer(state, {
      type: ACTIONS.DELETE_TASK,
      payload: { columnId: 'todo', taskId },
    });

    expect(next['todo'].tasks.find(t => t.id === taskId)).toBeUndefined();
  });

  it('returns unchanged state for unknown action', () => {
    const state = structuredClone(initialColumns);
    const next = boardReducer(state, { type: 'UNKNOWN' });
    expect(next).toEqual(state);
  });

  it('does not mutate the original state', () => {
    const state = structuredClone(initialColumns);
    const stateSnapshot = structuredClone(initialColumns);

    boardReducer(state, {
      type: ACTIONS.ADD_TASK,
      payload: { columnId: 'todo', title: 'Test', priority: 'Low' },
    });

    // Original state should be unchanged
    expect(state).toEqual(stateSnapshot);
  });
});
```

### Test 2: Integration test — component + user interaction

`src/__tests__/TaskForm.test.jsx`:
```jsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import TaskForm from '../components/TaskForm';

describe('TaskForm', () => {
  it('renders input fields and submit button', () => {
    render(<TaskForm onAddTask={() => {}} />);

    expect(screen.getByPlaceholderText('Task title')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /add/i })).toBeInTheDocument();
  });

  it('calls onAddTask with correct data when submitted', async () => {
    const onAddTask = vi.fn();                          // mock function
    const user = userEvent.setup();                      // simulate real user

    render(<TaskForm onAddTask={onAddTask} />);

    await user.type(screen.getByPlaceholderText('Task title'), 'Write tests');
    await user.click(screen.getByRole('button', { name: /add/i }));

    expect(onAddTask).toHaveBeenCalledTimes(1);          // called exactly once
    expect(onAddTask).toHaveBeenCalledWith({             // called with right data
      title: 'Write tests',
      priority: 'Medium',
    });
  });

  it('does not call onAddTask if title is empty', async () => {
    const onAddTask = vi.fn();
    const user = userEvent.setup();

    render(<TaskForm onAddTask={onAddTask} />);

    await user.click(screen.getByRole('button', { name: /add/i }));

    expect(onAddTask).not.toHaveBeenCalled();
  });

  it('clears input after submission', async () => {
    const onAddTask = vi.fn();
    const user = userEvent.setup();

    render(<TaskForm onAddTask={onAddTask} />);

    const input = screen.getByPlaceholderText('Task title');
    await user.type(input, 'New task');
    await user.click(screen.getByRole('button', { name: /add/i }));

    expect(input).toHaveValue('');
  });
});
```

### Test 3: Hook test

`src/__tests__/useLocalStorage.test.jsx`:
```jsx
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { renderHook, act } from '@testing-library/react';
import { useLocalStorage } from '../hooks/useLocalStorage';

describe('useLocalStorage', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  it('returns initial value when nothing is stored', () => {
    const { result } = renderHook(() => useLocalStorage('test-key', 'default'));
    expect(result.current[0]).toBe('default');
  });

  it('stores and retrieves values', () => {
    const { result } = renderHook(() => useLocalStorage('test-key', 'default'));

    act(() => {
      result.current[1]('new value');
    });

    expect(result.current[0]).toBe('new value');
    expect(localStorage.getItem('test-key')).toBe(JSON.stringify('new value'));
  });

  it('uses stored value from localStorage', () => {
    localStorage.setItem('test-key', JSON.stringify('stored'));

    const { result } = renderHook(() => useLocalStorage('test-key', 'default'));

    expect(result.current[0]).toBe('stored');
  });

  it('handles JSON parse errors', () => {
    localStorage.setItem('test-key', 'invalid-json');

    const { result } = renderHook(() => useLocalStorage('test-key', 'default'));

    expect(result.current[0]).toBe('default');
  });
});
```

## 3. TESTING BEST PRACTICES

| Do | Don't |
|----|-------|
| Test behavior, not implementation | Test internal state values |
| Use `getByRole` over `getByTestId` | Use `getByTestId` as first resort |
| Use `userEvent` over `fireEvent` | Simulate clicks with `fireEvent` |
| Write tests that read like user actions | Write tests that know component internals |
| Test error states and edge cases | Only test the happy path |

## 4. EXERCISE

Write a test for `TaskList` that verifies:
1. Shows "No tasks" when array is empty
2. Renders correct number of tasks
3. Calls onToggle when checkbox is clicked

**Solution:**
```jsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import TaskList from '../components/TaskList';

describe('TaskList', () => {
  const mockTasks = [
    { id: 1, title: 'Task 1', priority: 'High', done: false },
    { id: 2, title: 'Task 2', priority: 'Low', done: true },
  ];

  it('shows empty state when no tasks', () => {
    render(<TaskList tasks={[]} onToggle={() => {}} />);
    expect(screen.getByText(/no tasks/i)).toBeInTheDocument();
  });

  it('renders all tasks', () => {
    render(<TaskList tasks={mockTasks} onToggle={() => {}} />);
    expect(screen.getByText('Task 1')).toBeInTheDocument();
    expect(screen.getByText('Task 2')).toBeInTheDocument();
  });

  it('calls onToggle when clicking a task checkbox', async () => {
    const onToggle = vi.fn();
    const user = userEvent.setup();
    render(<TaskList tasks={mockTasks} onToggle={onToggle} />);

    const checkboxes = screen.getAllByRole('checkbox');
    await user.click(checkboxes[0]);

    expect(onToggle).toHaveBeenCalledWith(1);
  });

  it('shows correct completion count', () => {
    render(<TaskList tasks={mockTasks} onToggle={() => {}} />);
    expect(screen.getByText(/1 of 2 tasks completed/)).toBeInTheDocument();
  });
});
```

### CHALLENGE EXERCISE

Write a test for the Column component that verifies: clicking "+ Add Task" shows the form, filling it and clicking "Add" calls onAddTask, and the form disappears after submission.

**Solution:**
\`\`\`jsx
// src/__tests__/Column.test.jsx
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import Column from '../components/Column';

const mockColumn = {
  id: 'todo',
  title: 'To Do',
  tasks: [
    { id: 1, title: 'Test task', priority: 'High' },
  ],
};

describe('Column', () => {
  it('renders column title and task count', () => {
    render(<Column column={mockColumn} allColumns={{ todo: mockColumn }}
      onAddTask={() => {}} onMoveTask={() => {}} onDeleteTask={() => {}} />);

    expect(screen.getByText('To Do')).toBeInTheDocument();
    expect(screen.getByText('1')).toBeInTheDocument();  // task count
  });

  it('shows add form when clicking "+ Add Task"', async () => {
    const user = userEvent.setup();
    render(<Column column={mockColumn} allColumns={{ todo: mockColumn }}
      onAddTask={() => {}} onMoveTask={() => {}} onDeleteTask={() => {}} />);

    await user.click(screen.getByText('+ Add Task'));

    expect(screen.getByPlaceholderText('Task title')).toBeInTheDocument();
    expect(screen.getByRole('button', { name: 'Add' })).toBeInTheDocument();
  });

  it('calls onAddTask when form is submitted', async () => {
    const onAddTask = vi.fn();
    const user = userEvent.setup();
    render(<Column column={mockColumn} allColumns={{ todo: mockColumn }}
      onAddTask={onAddTask} onMoveTask={() => {}} onDeleteTask={() => {}} />);

    // Open form
    await user.click(screen.getByText('+ Add Task'));
    // Fill and submit
    await user.type(screen.getByPlaceholderText('Task title'), 'New task');
    await user.click(screen.getByRole('button', { name: 'Add' }));

    expect(onAddTask).toHaveBeenCalledWith('todo', {
      title: 'New task',
      priority: 'Medium',
    });
  });

  it('shows empty state when no tasks', () => {
    const emptyColumn = { ...mockColumn, tasks: [] };
    render(<Column column={emptyColumn} allColumns={{ todo: emptyColumn }}
      onAddTask={() => {}} onMoveTask={() => {}} onDeleteTask={() => {}} />);

    expect(screen.getByText('No tasks')).toBeInTheDocument();
  });
});
\`\`\`

## 5. COMMON MISTAKES

**Mistake: Testing implementation details**
```jsx
// ❌ Fragile — breaks if you rename state or refactor
expect(result.current[0]).toBe(0); // Testing useState value

// ✅ Robust — tests what the user sees
expect(screen.getByText(/count: 0/i)).toBeInTheDocument();
```

**Mistake: Not cleaning up between tests**
```jsx
// ❌ Test A modifies localStorage → Test B sees modified value
// Tests depend on each other's order!

// ✅ Vitest/RTL auto-cleanup, localStorage needs manual cleanup
beforeEach(() => localStorage.clear());
```

### 🐛 DEBUGGING WITH DEVTOOLS

When a test fails, read the error output carefully. Vitest shows: (1) Expected vs received values (with color diff), (2) The assertion that failed, (3) Stack trace pointing to the exact line. Use `screen.debug()` in your test to print the current DOM state to console — this shows what Testing Library "sees" and helps debug "element not found" errors.

### ⚙️ UNDER THE HOOD

Testing Library works by rendering components into a JSDOM environment (a browser simulation in Node). `render(<MyComponent />)` creates the virtual DOM, and `screen.getByText('Submit')` queries it. `userEvent` simulates real browser events: keydown → keypress → keyup for typing, mousedown → mouseup → click for clicking. This is more realistic than `fireEvent` which dispatches a single event.

### 🏭 PRODUCTION PATTERN

The industry standard testing stack (2025): Vitest (runner) + Testing Library (component tests) + Playwright (E2E tests). Write integration tests for component behavior (what the user sees and does), unit tests for pure logic (reducers, utilities), and E2E tests for critical user flows (signup, checkout). Aim for the "testing trophy": mostly integration tests, fewer unit and E2E tests.

### 💼 INTERVIEW READY

**Q:** What's the difference between `fireEvent` and `userEvent`, and when should you use each?
**A:** `fireEvent` directly dispatches a DOM event (e.g., `fireEvent.click(button)`). It's simpler but less realistic — it doesn't simulate the full event chain. `userEvent` from `@testing-library/user-event` simulates real interactions: `userEvent.click(button)` triggers mousedown, mouseup, click in sequence. Always prefer `userEvent` — it catches more edge cases (clicking disabled buttons, typing with modifiers). Use `fireEvent` only for edge cases userEvent doesn't support.

### 📝 MINI QUIZ

**1.** What's the purpose of Testing Library's `screen.getByRole`?
- A) To query by CSS class
- B) To query by ARIA role (accessible to screen readers)
- C) To query by component name
- D) To query by test ID

**2.** What's the recommended approach for testing component behavior?
- A) Test internal state values
- B) Test what the user sees and does (behavior)
- C) Test implementation details
- D) Test CSS styles

**3.** What does `vi.fn()` create in Vitest?
- A) A real function
- B) A mock function that records calls
- C) A fake timer
- D) A spy on an existing function

**Answers:** 1. B 2. B 3. B

## 6. CHECKPOINT

> What is the difference between `fireEvent` and `userEvent`?

**Answer:** `fireEvent` dispatches a DOM event directly (simpler, less realistic). `userEvent` simulates full user interactions (typing character by character, clicking with full event chain — more realistic, recommended by Testing Library).

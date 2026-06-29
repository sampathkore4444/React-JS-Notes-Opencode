# Lesson 19: Optimistic Updates — Instant UI

## 1. CONCEPT — Show Success Before the Server Responds

> **Real-world analogy:** Optimistic updates are like a magician who makes a coin disappear — the audience sees it vanish instantly (UI update), while the magician secretly drops it into their pocket (API call). If the trick fails (API error), the magician reveals the coin was never gone (rollback). The audience never waits — they see the result immediately.

**Normal update flow:**
```
User clicks "Move task" → Show spinner → API call → Response → Update UI
                              ↑ User waits here (300ms - 2s)!
```

**Optimistic update flow:**
```
User clicks "Move task" → Update UI instantly → API call in background
                              ↑ Zero wait for user!
                              If API fails → Rollback UI + show error
```

**When to use:**
- **High-confidence actions** (moving a task, toggling a checkbox, adding a like)
- **Non-critical accuracy** (a second of stale data is acceptable)
- **When UX matters more than perfect consistency**

**When NOT to use:**
- Payment transactions (must wait for confirmation)
- User registration (need server validation)
- Any action where showing wrong data causes harm

## 2. CODE ALONG — Optimistic Task Move

We'll implement optimistic moves in the board. First, the **non-optimistic** version (what you're used to):

```jsx
// ❌ User waits for server
async function handleMoveTask(id, from, to) {
  setMoving(true); // Show spinner
  await api.moveTask(id, to);
  refetch(); // Wait for server response
  setMoving(false);
}
```

### The optimistic pattern with React Query:

Update `src/hooks/useTasks.js`:
```jsx
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { taskApi } from '../services/api';

export function useMoveTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ taskId, toColumn }) =>
      taskApi.update(taskId, { column: toColumn }),

    // Step 1: OPTIMISTIC UPDATE — runs BEFORE mutationFn
    onMutate: async ({ taskId, fromColumn, toColumn }) => {
      // Cancel in-flight refetches so they don't overwrite our optimistic data
      await queryClient.cancelQueries({ queryKey: ['tasks'] });

      // Snapshot current state for rollback
      const previousTasks = queryClient.getQueryData(['tasks']);

      // Optimistically update the cache — UI updates INSTANTLY
      queryClient.setQueryData(['tasks'], (old) =>
        old.map(t =>
          t.id === taskId ? { ...t, column: toColumn } : t
        )
      );

      return { previousTasks };  // pass to onError for rollback
    },

    // Step 2: If API FAILS — restore snapshot
    onError: (err, variables, context) => {
      if (context?.previousTasks) {
        queryClient.setQueryData(['tasks'], context.previousTasks);
      }
      console.error('Move failed — rolled back', err.message);
    },

    // Step 3: After success OR failure — sync with server truth
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['tasks'] });
    },
  });
}
```

### Without React Query (raw state approach):

If you're not using React Query, here's the same pattern with just `useState`:

```jsx
// hooks/useOptimisticBoard.js
import { useState, useCallback, useRef } from 'react';

export function useOptimisticBoard() {
  const [columns, setColumns] = useState(initialColumns);
  const [moveError, setMoveError] = useState(null);
  const prevColumnsRef = useRef(null); // For rollback

  const moveTask = useCallback(async (taskId, fromColumnId, toColumnId) => {
    // 1. Save current state for rollback
    prevColumnsRef.current = columns;

    // 2. Find the task
    const task = columns[fromColumnId].tasks.find(t => t.id === taskId);
    if (!task) return;

    // 3. Optimistically update UI
    setColumns(prev => ({
      ...prev,
      [fromColumnId]: {
        ...prev[fromColumnId],
        tasks: prev[fromColumnId].tasks.filter(t => t.id !== taskId),
      },
      [toColumnId]: {
        ...prev[toColumnId],
        tasks: [...prev[toColumnId].tasks, task],
      },
    }));
    setMoveError(null);

    // 4. Call server in background
    try {
      await api.moveTask(taskId, toColumnId);
    } catch (err) {
      // 5. Rollback on failure
      if (prevColumnsRef.current) {
        setColumns(prevColumnsRef.current);
      }
      setMoveError('Move failed. Please try again.');
    }
  }, [columns]);

  return { columns, moveTask, moveError };
}
```

### Visual feedback: Show pending state

Add a visual indicator while the optimistic update is being confirmed:

```jsx
// In TaskCard.jsx — add a "syncing" state
function TaskCard({ task, columnId }) {
  const [syncing, setSyncing] = useState(false);

  // When moveTask is called, show syncing briefly
  async function handleMove(toColumnId) {
    setSyncing(true);
    // The optimistic update already moved the task visually
    // But let's show a tiny indicator
    setTimeout(() => setSyncing(false), 500);
    await onMoveTask(task.id, columnId, toColumnId);
  }

  return (
    <div style={{
      opacity: syncing ? 0.7 : 1,
      transition: 'opacity 0.2s',
      backgroundColor: 'white',
      padding: '10px',
      marginBottom: '8px',
      borderRadius: '6px',
      boxShadow: '0 1px 3px rgba(0,0,0,0.1)',
    }}>
      {syncing && <span style={{ fontSize: '12px', color: '#3b82f6' }}>↻ syncing...</span>}
      <div>{task.title}</div>
    </div>
  );
}
```

## 3. THE OPTIMISTIC UI PATTERN (Memorize This)

```
1. Save snapshot of current state
2. Update UI to expected final state
3. Send API request
4. On success: do nothing (UI already correct), optionally invalidate cache
5. On failure: restore snapshot from step 1
6. Show error notification
```

**React Query does steps 1-5 for you with `onMutate` and `onError`.**

## 4. EXERCISE

Implement optimistic delete: when user clicks delete, remove the task instantly from UI. If the server call fails, add it back and show a toast.

**Solution:**
```jsx
export function useOptimisticDelete() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (taskId) => taskApi.delete(taskId),

    onMutate: async (taskId) => {
      await queryClient.cancelQueries({ queryKey: ['tasks'] });
      const previous = queryClient.getQueryData(['tasks']);
      queryClient.setQueryData(['tasks'], (old) =>
        old.filter(t => t.id !== taskId)
      );
      return { previous };
    },

    onError: (err, taskId, context) => {
      // Rollback
      queryClient.setQueryData(['tasks'], context.previous);
      // Show error toast
      showToast('Failed to delete task. It has been restored.', 'error');
    },

    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['tasks'] });
    },
  });
}
```

### CHALLENGE EXERCISE

Implement optimistic task creation. When a user adds a task, show it immediately in the UI with a temporary ID (negative number) and a "syncing" indicator. When the server confirms, replace the temp ID with the real one.

**Solution:**
```jsx
export function useCreateTaskOptimistic() {
  const queryClient = useQueryClient();
  let tempIdCounter = -1;  // negative IDs are temporary

  return useMutation({
    mutationFn: ({ title, priority }) => taskApi.create(title, priority),

    onMutate: async ({ title, priority }) => {
      await queryClient.cancelQueries({ queryKey: ['tasks'] });
      const previous = queryClient.getQueryData(['tasks']);
      const tempId = tempIdCounter--;

      // Add task immediately with temp ID
      queryClient.setQueryData(['tasks'], (old) => [
        {
          id: tempId,
          title,
          priority: priority || 'Medium',
          createdAt: new Date().toISOString(),
          _syncing: true,  // flag for pending state
        },
        ...(old || []),
      ]);

      return { previous, tempId };
    },

    onError: (err, variables, context) => {
      // Rollback: remove the temp task
      queryClient.setQueryData(['tasks'], (old) =>
        old.filter(t => t.id !== context.tempId)
      );
    },

    onSuccess: (serverTask, variables, context) => {
      // Replace temp task with server task
      queryClient.setQueryData(['tasks'], (old) =>
        old.map(t =>
          t.id === context.tempId ? { ...serverTask, _syncing: false } : t
        )
      );
    },
  });
}

// In component, show syncing indicator:
{tasks.map(task => (
  <div key={task.id} style={{
    opacity: task._syncing ? 0.6 : 1,
    display: 'flex', alignItems: 'center', gap: '8px',
  }}>
    {task._syncing && <span style={{ fontSize: '12px', color: '#3b82f6' }}>⟳</span>}
    <span>{task.title}</span>
  </div>
))}
```

## 5. COMMON MISTAKES

**Mistake: Optimistic updates for destructive irreversible actions**
```jsx
// ❌ Don't do optimistic for: payment, account deletion, sending emails
// User might think action succeeded when it didn't

// ✅ Good for: todos, likes, sorting, reordering, theme changes
```

**Mistake: Forgetting error rollback**
```jsx
// ❌ If API fails, UI is now out of sync with server
// User sees task moved → but server never got the update
// Next page refresh: task is back in old column (confusing!)

// ✅ Always implement rollback for optimistic updates
```

### 🐛 DEBUGGING WITH DEVTOOLS

Simulate a network failure to test your rollback. Open DevTools → Network tab → "Offline" checkbox (or throttle to "Slow 3G"). Perform an action and verify: (1) UI updates instantly (optimistic), (2) error toast appears, (3) UI reverts to the previous state (rollback). If rollback doesn't work, check that `onError` receives the correct `context` from `onMutate`.

### ⚙️ UNDER THE HOOD

Optimistic updates follow a pattern: snapshot → mutate → rollback on error. In React Query, `onMutate` runs BEFORE the mutation function. You save the current cache state (`queryClient.getQueryData`), optimistically update the cache (`queryClient.setQueryData`), and return the snapshot. If the mutation fails, `onError` restores the snapshot. If it succeeds, `onSettled` invalidates to sync with server truth.

### 🏭 PRODUCTION PATTERN

Use optimistic updates for: toggles (likes, follows), drag-and-drop reordering, adding items to lists, applying filters. Do NOT use for: payments, form submissions, account deletion, email sending — where showing "success" before confirmation is harmful. Always show a subtle indicator (opacity change, small spinner) that the update is pending server confirmation.

### 💼 INTERVIEW READY

**Q:** Walk through the lifecycle of an optimistic update with React Query.
**A:** (1) User triggers action → React Query calls `onMutate` which cancels in-flight refetches, snapshots current cache, and optimistically updates the cache (UI updates instantly). (2) `mutationFn` runs — the actual API call. (3) On success: `onSettled` invalidates queries to sync with server. (4) On error: `onError` restores the snapshot from onMutate, and UI reverts to the original state. The user never sees a loading spinner — the update appears instant and rolls back gracefully on failure.

### 📝 MINI QUIZ

**1.** What's the purpose of `onMutate` in an optimistic update?
- A) To show a loading spinner
- B) To update the cache immediately before the API call
- C) To validate the input data
- D) To log the action

**2.** What happens if the API call fails during an optimistic update?
- A) The UI stays in the optimistic state
- B) onError restores the previous state (rollback)
- C) The page refreshes
- D) React Query retries forever

**3.** When should you NOT use optimistic updates?
- A) Moving a task between columns
- B) Processing a payment
- C) Toggling a checkbox
- D) Adding a like to a post

**Answers:** 1. B 2. B 3. B

## 6. CHECKPOINT

> What is the minimum server response time needed to make optimistic updates worthwhile?

**Answer:** Any response time > 100ms. Even 300ms feels slow to users. Optimistic updates make the app feel instant (0ms perceived wait) by showing the result before the server confirms.

# Lesson 18: TanStack Query (React Query) — Server State Management

## 1. CONCEPT — Stop Writing Fetch Logic Yourself

> **Real-world analogy:** React Query is like having a personal assistant who handles all your data-fetching chores. Instead of you manually calling the restaurant (fetch), waiting by the phone (loading), and remembering past orders (caching), your assistant does it all — and even brings you fresh food automatically when it gets stale (background refetching). You just say "I need tasks" and the assistant handles the rest.

Every fetch needs: loading, error, success, caching, refetching, deduplication, background updates, pagination. You'll write this 100 times.

**TanStack Query handles ALL of it:**
```jsx
// Without React Query: ~20 lines, repetitive, easy to mess up
const [data, setData] = useState(null);     // manual loading/data/error
const [loading, setLoading] = useState(true);
const [error, setError] = useState(null);
useEffect(() => {
  let mounted = true;                        // cleanup flag for race conditions
  setLoading(true);
  fetch('/api/tasks')
    .then(r => r.json())
    .then(d => { if (mounted) setData(d); })   // only set if still mounted
    .catch(e => { if (mounted) setError(e); })
    .finally(() => { if (mounted) setLoading(false); });
  return () => { mounted = false; };
}, []);

// With React Query: 8 lines, built-in caching, refetching, loading states
const { data, isLoading, error } = useQuery({
  queryKey: ['tasks'],                       // unique key for caching
  queryFn: () => fetch('/api/tasks').then(r => r.json()),  // just the fetch function
});
// React Query manages: loading, error, caching, refetch, dedup
```

**What you get for free:**
- Auto-caching (don't refetch same data twice)
- Background refetching (data is always fresh)
- Stale-while-revalidate (show cached data, fetch new in background)
- Loading/error states
- Refetch on window focus (user comes back → fresh data)
- Deduplication (same query called 3 times → 1 network request)
- Pagination & infinite scroll support

## 2. CODE ALONG — Replace useFetch with React Query

### Install
```bash
npm install @tanstack/react-query
```

### Step 1: Set up QueryClient

`src/main.jsx`:
```jsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import App from './App.jsx';

// Create a client
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,    // 5 min: data considered "fresh" (no refetch needed)
      gcTime: 30 * 60 * 1000,       // 30 min: keep in cache after no subscribers
      retry: 2,                      // auto-retry twice on failure
      refetchOnWindowFocus: true,    // refetch when user returns to tab
    },
  },
});

createRoot(document.getElementById('root')).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <App />
    </QueryClientProvider>
  </StrictMode>
);
```

### Step 2: Replace manual fetch with useQuery

`src/hooks/useTasks.js`:
```jsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { taskApi } from '../services/api';

// All tasks share the same query key
const TASKS_KEY = ['tasks'];

// Hook: get all tasks
export function useTasks() {
  return useQuery({
    queryKey: TASKS_KEY,
    queryFn: () => taskApi.getAll(),
    // select: (data) => data.filter(t => !t.done), // Transform data
  });
}

// Hook: get single task
export function useTask(id) {
  return useQuery({
    queryKey: ['task', id],
    queryFn: () => taskApi.getById(id),
    enabled: !!id,  // Only run when id is truthy
  });
}

// Hook: create task (mutation, not query)
export function useCreateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ title, priority }) => taskApi.create(title, priority),
    // On success: refetch the tasks list
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: TASKS_KEY });
    },
  });
}

// Hook: delete task
export function useDeleteTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (id) => taskApi.delete(id),
    // Optimistic update: remove instantly, rollback on error
    onMutate: async (id) => {
      // Cancel outgoing refetches (so they don't overwrite our update)
      await queryClient.cancelQueries({ queryKey: TASKS_KEY });
      // Snapshot previous value
      const previous = queryClient.getQueryData(TASKS_KEY);
      // Optimistically update cache
      queryClient.setQueryData(TASKS_KEY, old =>
        old.filter(t => t.id !== id)
      );
      return { previous }; // Pass to onError for rollback
    },
    onError: (err, id, context) => {
      // Rollback on error
      queryClient.setQueryData(TASKS_KEY, context.previous);
    },
  });
}
```

### Step 3: Use in components

`src/components/TaskListView.jsx`:
```jsx
import { useTasks, useCreateTask, useDeleteTask } from '../hooks/useTasks';

function TaskListView() {
  const { data: tasks, isLoading, error, refetch } = useTasks();
  const createTask = useCreateTask();
  const deleteTask = useDeleteTask();
  const [newTitle, setNewTitle] = useState('');

  if (isLoading) return <div className="skeleton">Loading tasks...</div>;
  if (error) return (
    <div style={{ color: 'red' }}>
      Error: {error.message}
      <button onClick={() => refetch()}>Retry</button>
    </div>
  );

  function handleAdd(e) {
    e.preventDefault();
    if (!newTitle.trim()) return;
    createTask.mutate({ title: newTitle.trim(), priority: 'Medium' });
    setNewTitle('');
  }

  return (
    <div>
      <form onSubmit={handleAdd}>
        <input value={newTitle} onChange={e => setNewTitle(e.target.value)} />
        <button type="submit" disabled={createTask.isPending}>
          {createTask.isPending ? 'Adding...' : 'Add'}
        </button>
      </form>

      {tasks?.length === 0 ? (
        <p>No tasks yet. Create one!</p>
      ) : (
        tasks.map(task => (
          <div key={task.id}>
            {task.title}
            <button
              onClick={() => deleteTask.mutate(task.id)}
              disabled={deleteTask.isPending}
            >
              Delete
            </button>
          </div>
        ))
      )}
    </div>
  );
}
```

## 3. KEY CONCEPTS

| Concept | Meaning |
|---------|---------|
| **queryKey** | Unique identifier for cached data. `['tasks']`, `['task', id]` |
| **staleTime** | How long data is considered fresh (no refetch needed) |
| **gcTime** | How long inactive data stays in cache |
| **invalidateQueries** | Mark cached data as stale → triggers refetch |
| **useMutation** | For create/update/delete operations (not fetching) |
| **optimistic update** | Update cache immediately, rollback on error |

## 4. EXERCISE

Add loading indicators for each mutation (not just the list). When deleting a task, the button should show "Deleting..." and be disabled.

**Solution:**
```jsx
// The mutation already exposes isPending — use it!
<button
  onClick={() => deleteTask.mutate(task.id)}
  disabled={deleteTask.isPending}
>
  {deleteTask.isPending ? 'Deleting...' : 'Delete'}
</button>

// For better UX, track which task is being deleted:
const [deletingId, setDeletingId] = useState(null);

async function handleDelete(id) {
  setDeletingId(id);
  await deleteTask.mutateAsync(id);
  setDeletingId(null);
}
// Then: disabled={deletingId === task.id}
```

### CHALLENGE EXERCISE

Add a "Refresh" button that manually refetches tasks, and show the "last updated" timestamp. Use `useQuery`'s built-in `refetch` function and `dataUpdatedAt` timestamp.

**Solution:**
```jsx
import { useTasks } from '../hooks/useTasks';
import { useState } from 'react';

function TaskListView() {
  const { data: tasks, isLoading, error, refetch, dataUpdatedAt } = useTasks();
  const [refreshing, setRefreshing] = useState(false);

  async function handleRefresh() {
    setRefreshing(true);
    await refetch();           // manually trigger refetch
    setRefreshing(false);
  }

  return (
    <div>
      <div style={{
        display: 'flex', justifyContent: 'space-between',
        alignItems: 'center', marginBottom: '12px',
      }}>
        <h2 style={{ margin: 0 }}>Tasks</h2>
        <div style={{ display: 'flex', alignItems: 'center', gap: '8px' }}>
          {dataUpdatedAt && (
            <span style={{ fontSize: '12px', color: '#94a3b8' }}>
              Updated: {new Date(dataUpdatedAt).toLocaleTimeString()}
            </span>
          )}
          <button
            onClick={handleRefresh}
            disabled={refreshing}
            style={{
              padding: '6px 12px', cursor: 'pointer',
              border: '1px solid #ddd', borderRadius: '4px',
              backgroundColor: refreshing ? '#e2e8f0' : 'white',
            }}
          >
            {refreshing ? '⟳ Refreshing...' : '⟳ Refresh'}
          </button>
        </div>
      </div>

      {isLoading && <div>Loading...</div>}
      {error && <div>Error: {error.message}</div>}

      {tasks?.map(task => (
        <div key={task.id}>{task.title}</div>
      ))}
    </div>
  );
}
```

## 5. COMMON MISTAKES

**Mistake: Not splitting query keys correctly**
```jsx
// ❌ Both share same key — different data overwrites each other
useQuery({ queryKey: ['tasks'], queryFn: () => api.getTasks(filter) });
// If filter changes, same key returns stale data!

// ✅ Include filter in the key
useQuery({ queryKey: ['tasks', filter], queryFn: () => api.getTasks(filter) });
```

**Mistake: Missing `enabled` for conditional queries**
```jsx
// ❌ Runs on mount with userId=undefined → API call fails
useQuery({
  queryKey: ['user', userId],
  queryFn: () => api.getUser(userId),
});

// ✅ Don't run until userId exists
useQuery({
  queryKey: ['user', userId],
  queryFn: () => api.getUser(userId),
  enabled: !!userId,
});
```

### 🐛 DEBUGGING WITH DEVTOOLS

TanStack Query DevTools is a separate DevTools panel. Install `@tanstack/react-query-devtools` and add `<ReactQueryDevtools />` in your app. It shows: all cached queries, their status (fresh/stale/loading/error), last fetched time, stale time remaining, and refetch count. You can manually refetch, invalidate, or remove queries from the panel — invaluable for debugging cache issues.

### ⚙️ UNDER THE HOOD

React Query maintains a query cache as a JavaScript Map outside the component tree. Each query key maps to a query object with data, status, and metadata. When a component uses `useQuery`, it subscribes to the query store. When data changes, only subscribers of that specific query key re-render. This is why query keys must be unique and specific — they're the cache key and the subscription channel.

### 🏭 PRODUCTION PATTERN

React Query is the standard for server state in React apps (2025). Pair it with the `@tanstack/react-query-devtools` in development. Configure sensible defaults: `staleTime: 5 * 60 * 1000` (5 min) for data that doesn't change often, `staleTime: 0` for live data. Use `gcTime` (formerly `cacheTime`) to control how long data persists after the last subscriber unmounts.

### 💼 INTERVIEW READY

**Q:** What's the difference between `staleTime` and `gcTime` (formerly `cacheTime`)?
**A:** `staleTime` controls how long data is considered "fresh" after fetching. While fresh, React Query won't refetch it on window focus or mount. `gcTime` (garbage collection time) controls how long inactive/unused data stays in the cache before being garbage collected. Example: staleTime=5min means no refetch for 5min. gcTime=30min means if no component uses the query for 30min, the data is removed from cache. staleTime should always be less than gcTime.

### 📝 MINI QUIZ

**1.** What does `useQuery` give you for free?
- A) Only the data
- B) Loading, error, caching, refetching, deduplication
- C) WebSocket connections
- D) TypeScript types

**2.** What is a `queryKey` used for?
- A) Identifying cached data and triggering refetches
- B) Sorting the results
- C) Authenticating the user
- D) Pagination

**3.** What's the difference between `useQuery` and `useMutation`?
- A) useQuery is for GET requests, useMutation is for POST/PUT/DELETE
- B) useQuery auto-retries, useMutation doesn't
- C) useQuery caches data, useMutation is for creating/updating/deleting
- D) All of the above

**Answers:** 1. B 2. A 3. D

## 6. CHECKPOINT

> What is the difference between `staleTime` and `gcTime` (formerly `cacheTime`)?

**Answer:** `staleTime` controls how long data is considered fresh (no background refetch needed). `gcTime` controls how long inactive/unused data stays in memory before being garbage collected. staleTime < gcTime typically.

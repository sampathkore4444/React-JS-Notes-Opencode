# Lesson 17: API Calls with Fetch — Real Data

## 1. CONCEPT — Talking to a Server

So far, all data is in localStorage. Real apps get data from servers.

> **Real-world analogy:** Making an API call is like ordering food at a restaurant. You place an order (send a request), the kitchen prepares it (server processes), and the waiter brings it back (response). Sometimes the kitchen is slow (loading state), sometimes they're out of ingredients (error state), and sometimes you get exactly what you ordered (success state). A good React component handles all three possibilities.

**The fetch pattern in React:**
```jsx
useEffect(() => {
  // State 1: Loading — show spinner
  setLoading(true);
  setError(null);

  // State 2: Fetch — talk to the server
  fetch('/api/tasks')                              // send GET request
    .then(res => {
      if (!res.ok) throw new Error(`HTTP ${res.status}`);  // catch 404/500!
      return res.json();                           // parse JSON body
    })
    .then(data => setData(data))                   // State 3: Success — render data
    .catch(err => setError(err.message))           // State 4: Error — show message
    .finally(() => setLoading(false));             // always: hide spinner
}, []);
```

**Three states every fetch needs:**
```
┌──────────────────┐
│  loading: true   │ → Show spinner
│  data: null      │
│  error: null     │
└──────────────────┘
       ↓ (response received)
┌──────────────────┐
│  loading: false  │ → Render data
│  data: {...}     │
│  error: null     │
└──────────────────┘
       ↓ (network failure)
┌──────────────────┐
│  loading: false  │ → Show error message
│  data: null      │
│  error: "..."    │
└──────────────────┘
```

**Race condition problem:**
```jsx
useEffect(() => {
  fetch(`/api/tasks?filter=${filter}`)
    .then(res => res.json())
    .then(data => setData(data));
    // ❌ If filter changes quickly, response 2 may arrive AFTER response 3
    // Data shows wrong filter's results!
}, [filter]);
```

## 2. CODE ALONG — Add a Task API

### Step 1: Create a mock API

`src/services/api.js`:
```jsx
// Simulates network delay and storage
const delay = (ms) => new Promise(r => setTimeout(r, ms));  // simulate network latency

// In-memory storage (simulates a database)
let tasks = [];  // In-memory "database" (resets on page refresh)

export const taskApi = {
  async getAll() {
    await delay(500);     // simulate 500ms network delay
    return [...tasks];    // return a COPY (immutability principle)
  },

  async create(title, priority = 'Medium') {
    await delay(300);
    const task = { id: Date.now(), title, priority, createdAt: new Date().toISOString() };
    tasks.unshift(task);  // add to beginning of array
    return task;
  },

  async update(id, changes) {
    await delay(300);
    tasks = tasks.map(t => t.id === id ? { ...t, ...changes } : t);  // immutable update
    return tasks.find(t => t.id === id);
  },

  async delete(id) {
    await delay(200);
    tasks = tasks.filter(t => t.id !== id);  // remove by filtering
    return { success: true };
  },
};
```

### Step 2: Custom hook for fetching

`src/hooks/useFetch.js` — handles loading, error, race conditions, and cleanup:

```jsx
import { useState, useEffect, useCallback, useRef } from 'react';

export function useFetch(fetchFn, deps = []) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  // Track latest request — prevents race conditions
  const requestIdRef = useRef(0);

  const execute = useCallback(async (...args) => {
    const requestId = ++requestIdRef.current;
    setLoading(true);
    setError(null);

    try {
      const result = await fetchFn(...args);
      // Only apply if this is still the latest request
      if (requestId === requestIdRef.current) {
        setData(result);
        setError(null);
      }
      return result;
    } catch (err) {
      if (requestId === requestIdRef.current) {
        setError(err.message || 'An error occurred');
      }
      throw err;
    } finally {
      if (requestId === requestIdRef.current) {
        setLoading(false);
      }
    }
  }, deps);

  const refetch = useCallback(() => execute(), [execute]);

  return { data, loading, error, execute: refetch, refetch };
}

// Usage:
// const { data: tasks, loading, error, refetch } = useFetch(taskApi.getAll, []);
```

### Step 3: Use it in a component

`src/components/TaskApiDemo.jsx`:
```jsx
import { useEffect } from 'react';
import { useFetch } from '../hooks/useFetch';
import { taskApi } from '../services/api';

function TaskListFromApi() {
  const { data: tasks, loading, error, execute } = useFetch(taskApi.getAll, []);

  // Fetch on mount
  useEffect(() => { execute(); }, [execute]);

  async function handleAdd() {
    const title = prompt('Task title:');
    if (!title) return;
    await taskApi.create(title);
    execute(); // Re-fetch
  }

  if (loading && !tasks) return <div style={{ textAlign: 'center', padding: '40px' }}>Loading...</div>;
  if (error) return <div style={{ color: 'red', padding: '20px' }}>Error: {error}</div>;

  return (
    <div>
      <button onClick={handleAdd}>Add Task</button>
      <ul>
        {tasks?.map(task => (
          <li key={task.id}>{task.title} — {task.priority}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Step 4: Handle race conditions properly

The `useFetch` hook above handles race conditions via `requestIdRef`. But there's a simpler approach — **AbortController**:

```jsx
// Alternative: AbortController approach
export function useFetchWithAbort(fetchFn, deps = []) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  const execute = useCallback(async (...args) => {
    const abortController = new AbortController();
    // Store controller so we can abort on cleanup
    window.__currentController = abortController;

    setLoading(true);
    setError(null);

    try {
      const result = await fetchFn(...args, { signal: abortController.signal });
      // If request was aborted, this code won't run (throw AbortError)
      setData(result);
    } catch (err) {
      if (err.name !== 'AbortError') {
        setError(err.message);
      }
    } finally {
      setLoading(false);
    }
  }, deps);

  // Abort on unmount
  useEffect(() => {
    return () => {
      window.__currentController?.abort();
    };
  }, []);

  return { data, loading, error, execute };
}
```

## 3. WHY THIS MATTERS

Fetching data is **the most common side effect** in React apps. Getting it wrong causes:
- **Race conditions** — showing data for wrong query
- **Memory leaks** — setting state on unmounted component
- **Poor UX** — no loading/error states
- **Unnecessary requests** — refetching when data hasn't changed

## 4. EXERCISE

Swap TaskFlow from localStorage to this mock API. Use `useFetch` in `useBoard` hook. When you add a task, call `taskApi.create()` then refetch.

**Solution:**
```jsx
// hooks/useBoard.js — API version
export function useBoard() {
  const { data: serverTasks, loading, error, execute: fetchTasks } = useFetch(taskApi.getAll, []);

  useEffect(() => { fetchTasks(); }, [fetchTasks]);

  // Convert server tasks to column format
  const columns = useMemo(() => {
    if (!serverTasks) return {};
    return {
      'todo': { id: 'todo', title: 'To Do', tasks: serverTasks.filter(t => !t.done) },
      'done': { id: 'done', title: 'Done', tasks: serverTasks.filter(t => t.done) },
    };
  }, [serverTasks]);

  const addTask = useCallback(async (title, priority) => {
    await taskApi.create(title, priority);
    await fetchTasks(); // Refetch from server
  }, [fetchTasks]);

  const deleteTask = useCallback(async (id) => {
    // Optimistic: remove from UI immediately
    // Then re-fetch to get server truth
    await taskApi.delete(id);
    await fetchTasks();
  }, [fetchTasks]);

  return { columns, addTask, deleteTask, loading, error };
}
```

### CHALLENGE EXERCISE

Add a "Retry" button when an API call fails. Also show a "Loading..." skeleton (placeholder UI) instead of a text spinner. Use `setTimeout` to simulate a flaky network that fails 50% of the time.

**Solution:**

Modify the mock API to randomly fail:
```jsx
// In api.js, add a flaky version:
async getAll() {
  await delay(500);
  if (Math.random() < 0.5) throw new Error('Network error: server unreachable');
  return [...tasks];
},
```

Add a retry hook:
```jsx
// In TaskApiDemo, add retry button and skeleton:
import { useState, useCallback } from 'react';

function TaskListFromApi() {
  const { data: tasks, loading, error, execute } = useFetch(taskApi.getAll, []);
  const [retryCount, setRetryCount] = useState(0);

  useEffect(() => { execute(); }, [execute]);

  const handleRetry = useCallback(() => {
    setRetryCount(c => c + 1);
    execute();
  }, [execute]);

  // Skeleton loader (placeholder UI)
  if (loading && !tasks) return (
    <div>
      {[1, 2, 3].map(i => (
        <div key={i} style={{
          height: '40px', marginBottom: '8px',
          backgroundColor: '#e2e8f0', borderRadius: '4px',
          animation: 'pulse 1.5s ease-in-out infinite',
        }} />
      ))}
    </div>
  );

  if (error) return (
    <div style={{ textAlign: 'center', padding: '20px' }}>
      <p style={{ color: '#dc2626' }}>Error: {error}</p>
      <p style={{ color: '#64748b', fontSize: '14px' }}>
        Retry #{retryCount}
      </p>
      <button onClick={handleRetry} style={{
        padding: '8px 16px', cursor: 'pointer',
        backgroundColor: '#3b82f6', color: 'white',
        border: 'none', borderRadius: '4px',
      }}>
        🔄 Retry
      </button>
    </div>
  );

  return (
    // ... normal render
  );
}
```

## 5. COMMON MISTAKES

**Mistake: Setting state after component unmounts**
```jsx
useEffect(() => {
  fetch('/api/data')
    .then(res => res.json())
    .then(data => setData(data)); // ❌ If component unmounts: memory leak!
}, []);

// ✅ Check if component is still mounted
useEffect(() => {
  let mounted = true;
  fetch('/api/data')
    .then(res => res.json())
    .then(data => { if (mounted) setData(data); });
  return () => { mounted = false; };
}, []);
```

**Mistake: Not handling HTTP errors properly**
```jsx
// ❌ fetch doesn't reject on 404/500 — only on network failure
fetch('/api/tasks')
  .then(res => res.json()) // Still runs even if 404!
  .then(data => setData(data));

// ✅ Check response.ok first
fetch('/api/tasks')
  .then(res => {
    if (!res.ok) throw new Error(`HTTP ${res.status}: ${res.statusText}`);
    return res.json();
  })
  .then(data => setData(data))
  .catch(err => setError(err.message));
```

### 🐛 DEBUGGING WITH DEVTOOLS

Open the Network tab in DevTools (F12). Filter by "Fetch/XHR" to see only API calls. Click a request to see headers, response body, and timing. If your fetch returns a 404 or 500, the response tab shows the error. If your component shows "loading" forever, check that the request appears here — if not, the fetch never fired. The Network tab is your first stop for API debugging.

### ⚙️ UNDER THE HOOD

The `fetch` API returns a Promise that resolves when the response HEADERS arrive (not the full body). `res.json()` returns another Promise that resolves when the BODY is fully downloaded. This two-step process lets you handle HTTP errors (4xx/5xx) before reading the body. Important: fetch does NOT reject on HTTP errors — only on network failures. Check `res.ok` manually.

### 🏭 PRODUCTION PATTERN

In production, don't use raw fetch with useEffect. Use TanStack Query, SWR, or RTK Query — they handle caching, deduplication, background refetching, loading states, and error handling. But understanding the raw pattern is essential because every abstraction is built on it, and you'll need it for WebSocket connections, upload progress, and cases where query libraries don't fit.

### 💼 INTERVIEW READY

**Q:** Why doesn't `fetch` reject on HTTP 404 or 500 errors, and how do you handle them?
**A:** `fetch` only rejects on network failures (offline, DNS failure, CORS error). HTTP 4xx/5xx are considered "successful HTTP transactions" — the server responded. Handle them by checking `response.ok` or `response.status` after the fetch resolves: `if (!res.ok) throw new Error(HTTP ${res.status})`. Then `.catch()` catches both network errors and your custom HTTP errors.

### 📝 MINI QUIZ

**1.** Does `fetch` reject on HTTP 404 errors?
- A) Yes
- B) No — only on network failures
- C) Only if configured
- D) Only in production

**2.** What does `res.json()` do?
- A) Returns the response as a JavaScript object
- B) Returns a Promise that resolves with the parsed JSON body
- C) Converts the request body to JSON
- D) Sets the Content-Type header

**3.** What are the three states every data-fetching component should handle?
- A) Loading, Success, Error
- B) Start, Middle, End
- C) Request, Response, Timeout
- D) Pending, Fulfilled, Rejected

**Answers:** 1. B 2. B 3. A

## 6. CHECKPOINT

> What three states should every data-fetching component handle, and why?

**Answer:** Loading (show spinner/placeholder), Error (show error message with retry option), Success (render data). Users need feedback at every stage — an empty screen with no loading indicator is confusing.

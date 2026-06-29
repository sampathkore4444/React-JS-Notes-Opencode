# Your Next Steps — The Full Mastery Roadmap

> **Real-world analogy:** Learning React is like learning to cook. You've made your first few meals (built a Kanban app). Now you have a complete cookbook (this roadmap) with recipes for every skill level — from weeknight dinners (hooks, context) to Michelin-star dishes (Next.js, performance). Follow the recipes in order, and don't skip the practice.

You've completed 11 lessons. You've built a working Kanban app with:
- ✅ Components & JSX
- ✅ Props & State
- ✅ Forms & Events
- ✅ Lifting state up
- ✅ useEffect & localStorage
- ✅ Custom hooks
- ✅ Context API
- ✅ React Router

**You are now at a Junior React Developer level.** Here's how to reach Senior.

---

## Phase 2: Deepen (Week 3-4)

### Topics to Learn Next

| Topic | Why | Project to Build |
|-------|-----|-----------------|
| **useReducer** | Complex state logic | Refactor board state to useReducer |
| **useRef** | Direct DOM access, timer refs | Auto-focus input, drag preview |
| **useMemo/useCallback** | Performance optimization | Profile & optimize TaskFlow |
| **React.memo** | Prevent child re-renders | Memoize TaskCard, Column |
| **Portals** | Render outside DOM tree | Modal for task details |

### Exercise: Refactor Board to useReducer

```jsx
// Check your 02-hooks-deep-dive.md for the useReducer pattern
// Goal: Replace useState + individual functions with a single reducer
```

### Watch/Read
- [ ] Dan Abramov's "Just Enough React" talk
- [ ] React docs: "You might not need an effect"

---

## Phase 3: Connect (Week 5-6)

### Topics

| Topic | Why | Project |
|-------|-----|---------|
| **API calls** | Real data | Fetch tasks from a JSON server |
| **React Query** | Cache, refetch, loading states | Replace manual fetch logic |
| **Error handling** | Crash-free apps | Error boundaries, try-catch |
| **Loading states** | UX | Skeleton screens, spinners |
| **Optimistic updates** | Instant UI | Move tasks instantly, rollback on error |

### Exercise: Add a Backend

```bash
# Install JSON server
npm install -D json-server

# Create db.json
echo '{"tasks": []}' > db.json

# Add script to package.json:
# "server": "json-server --watch db.json --port 3001"
```

Replace localStorage with API calls.

---

## Phase 4: Scale (Week 7-8)

### Topics

| Topic | Why | Project |
|-------|-----|---------|
| **State management** | Global state beyond Context | Add Zustand for board state |
| **Testing** | Reliability | Write tests for components & hooks |
| **TypeScript** | Type safety | Migrate TaskFlow to TypeScript |
| **Accessibility** | Inclusive apps | Add aria-labels, keyboard nav |
| **Code splitting** | Performance | Lazy load routes |

### Exercise: Add Drag & Drop

```bash
npm install @dnd-kit/core @dnd-kit/sortable
```

Implement drag-and-drop to move tasks between columns. See `dnd-kit` documentation.

---

## Phase 5: Polish (Week 9-10)

### Topics

| Topic | Why |
|-------|-----|
| **Bundle analysis** | Reduce bundle size |
| **Lighthouse audit** | Performance, accessibility, SEO |
| **Error monitoring** | Sentry integration |
| **Analytics** | Track user actions |
| **CI/CD** | Auto-deploy on push |
| **Storybook** | Component library documentation |

### Deploy Your App

```bash
npm run build   # creates dist/ folder
npm install -g serve
serve -s dist   # test production build locally

# Deploy to:
# - Vercel (vercel.com) — easiest
# - Netlify (netlify.com)
# - GitHub Pages
```

---

## Phase 6: Next.js & The Future (Week 11-12)

### Why Next.js

| React Problem | Next.js Solution |
|--------------|------------------|
| No SSR/SEO | Server-side rendering, static generation |
| Client-only data fetch | Server components, server actions |
| Manual routing | File-based routing (`app/` folder) |
| No image optimization | `<Image>` component |
| No middleware | Edge middleware, auth guards |

### Learn in Order

1. **Pages Router basics** (why Next.js exists)
2. **App Router** (new paradigm)
3. **Server Components** (zero JS sent to client)
4. **Server Actions** (mutate DB directly from component)
5. **Middleware** (auth, redirects)
6. **Deployment** (Vercel)

---

### CHALLENGE EXERCISE

Create a personal learning tracker. Based on this roadmap, build a small React app (or a single component) that lists all the topics across all phases. Each topic should have a checkbox to mark it as completed, and a progress bar showing overall completion percentage. Save the progress to localStorage.

**Solution (single component approach):**
```jsx
import { useState, useEffect } from 'react';

const phases = [
  { name: 'Phase 2: Deepen', topics: ['useReducer', 'useRef', 'useMemo/useCallback', 'React.memo', 'Portals'] },
  { name: 'Phase 3: Connect', topics: ['API calls', 'React Query', 'Error handling', 'Loading states', 'Optimistic updates'] },
  { name: 'Phase 4: Scale', topics: ['State management', 'Testing', 'TypeScript', 'Accessibility', 'Code splitting'] },
  { name: 'Phase 5: Polish', topics: ['Bundle analysis', 'Lighthouse audit', 'Error monitoring', 'Analytics', 'CI/CD', 'Storybook'] },
  { name: 'Phase 6: Next.js', topics: ['Pages Router', 'App Router', 'Server Components', 'Server Actions', 'Middleware', 'Deployment'] },
];

function LearningTracker() {
  const [completed, setCompleted] = useState(() => {
    const saved = localStorage.getItem('learning-tracker');
    return saved ? JSON.parse(saved) : {};
  });

  useEffect(() => {
    localStorage.setItem('learning-tracker', JSON.stringify(completed));
  }, [completed]);

  function toggle(topic) {
    setCompleted(prev => ({ ...prev, [topic]: !prev[topic] }));
  }

  const allTopics = phases.flatMap(p => p.topics);
  const doneCount = allTopics.filter(t => completed[t]).length;
  const percent = Math.round((doneCount / allTopics.length) * 100);

  return (
    <div style={{ maxWidth: '600px', margin: '20px auto', fontFamily: 'system-ui, sans-serif' }}>
      <h2>📊 Learning Progress</h2>
      <div style={{
        height: '24px', backgroundColor: '#e2e8f0', borderRadius: '12px',
        overflow: 'hidden', marginBottom: '24px',
      }}>
        <div style={{
          height: '100%', width: `${percent}%`,
          backgroundColor: percent === 100 ? '#22c55e' : '#3b82f6',
          borderRadius: '12px', transition: 'width 0.3s ease',
          display: 'flex', alignItems: 'center', justifyContent: 'center',
          color: 'white', fontSize: '12px', fontWeight: 'bold',
        }}>
          {percent}%
        </div>
      </div>

      {phases.map(phase => (
        <div key={phase.name} style={{ marginBottom: '16px' }}>
          <h3 style={{ margin: '0 0 8px', fontSize: '16px' }}>{phase.name}</h3>
          {phase.topics.map(topic => (
            <label key={topic} style={{
              display: 'flex', alignItems: 'center', gap: '8px',
              padding: '8px', cursor: 'pointer',
              backgroundColor: completed[topic] ? '#f0fdf4' : '#f8fafc',
              borderRadius: '4px', marginBottom: '4px',
            }}>
              <input
                type="checkbox"
                checked={!!completed[topic]}
                onChange={() => toggle(topic)}
              />
              <span style={{
                textDecoration: completed[topic] ? 'line-through' : 'none',
                color: completed[topic] ? '#22c55e' : '#334155',
              }}>
                {topic}
              </span>
            </label>
          ))}
        </div>
      ))}

      {percent === 100 && (
        <div style={{ textAlign: 'center', padding: '20px', color: '#22c55e', fontWeight: 'bold' }}>
          🎉 All topics completed! You're a React Senior!
        </div>
      )}
    </div>
  );
}
```

---

## Daily Practice Routine

```
Every day (30 min minimum):

Day 1:   Read one React concept from the guide → Build it
Day 2:   Solve one problem from a previous lesson without looking
Day 3:   Add one feature to TaskFlow
Day 4:   Read one real-world codebase (GitHub: vercel/next.js, facebook/react)
Day 5:   Explain a concept out loud (to a rubber duck)
Day 6:   Fix one bug intentionally (break something, then fix it)
Day 7:   Rest
```

---

## Resources (Curated — Not Everything)

### Must Read
- [React.dev documentation](https://react.dev) — official, interactive, excellent
- [React.dev "Learn React" course](https://react.dev/learn) — free, project-based

### Must Watch
- [ ] "The Beginner's Guide to React" — Kent C. Dodds
- [ ] "React Hooks" — Dan Abramov
- [ ] "Building a Full-Stack App with Next.js" — Lee Robinson

### Must Practice
- [ ] Build a Todo App from scratch (no tutorial)
- [ ] Build TaskFlow from scratch (no code to reference)
- [ ] Take an existing app and add one feature
- [ ] Refactor a class component to hooks
- [ ] Deploy something to production

---

### 🐛 DEBUGGING WITH DEVTOOLS

By now you've used React DevTools extensively. Master these three tabs: (1) Components — inspect live state and props, (2) Profiler — record interactions and find unnecessary re-renders, (3) Settings → Highlight updates — see which components re-render on every interaction. These tools separate junior from senior React devs.

### ⚙️ UNDER THE HOOD

By this point you should understand that React's entire model is built on three concepts: (1) Props flow down (unidirectional data flow), (2) State triggers re-renders (via setState/dispatch), (3) Effects synchronize with external systems. Everything else — Context, hooks, portals, error boundaries — is built on this foundation. Master these three and you understand React.

### 🏭 PRODUCTION PATTERN

Senior engineers don't just write code — they decide what NOT to write. Before adding a library, ask: "Does React already solve this?" Before optimizing, profile. Before abstracting, wait until the pattern appears 3 times. The best code is the code you don't write.

### 💼 INTERVIEW READY

**Q:** How would you design a real-time collaborative board like Trello?
**A:** Start with the data model (boards → columns → tasks), then state management (Zustand or useReducer for local state, React Query for server state), then real-time sync (WebSockets with optimistic updates), then conflict resolution (CRDT or operational transforms), then performance (virtualization for large boards, memo for task cards). Break it into clear layers.

### 📝 MINI QUIZ

**1.** What's the primary benefit of Next.js over plain React?
- A) Smaller bundle size
- B) Server-side rendering for SEO and performance
- C) Built-in state management
- D) No JavaScript required

**2.** Which testing type gives the highest confidence?
- A) Unit tests
- B) Integration tests (Testing Library)
- C) E2E tests (Playwright)
- D) Snapshot tests

**3.** What should you do BEFORE optimizing performance?
- A) Add React.memo everywhere
- B) Profile with React DevTools Profiler
- C) Rewrite in TypeScript
- D) Add a CDN

**Answers:** 1. B 2. C 3. B

---

## Interview Preparation — The 3-Week Sprint

### Week 1: Fundamentals
- Explain Virtual DOM in 2 minutes
- Explain reconciliation
- Difference between props and state
- Explain useEffect lifecycle (mount/update/unmount)
- Explain the rules of hooks

### Week 2: Advanced
- Explain useMemo vs useCallback with examples
- Explain Context API limitations
- Explain error boundaries
- Explain code splitting
- Explain React.memo

### Week 3: System Design
How would you build:
- A real-time collaborative board (like Trello)?
- An E-Commerce checkout flow?
- A banking dashboard with live updates?
- A social media feed with infinite scroll?
- A form builder with drag-and-drop?

### Final Interview Prep
- Review your own TaskFlow code — be ready to explain every line
- Be ready to code a Counter, Todo List, or Fetch Data component from scratch
- Be ready to explain a bug you fixed and how you debugged it

---

## The Senior Engineer Mindset

```
┌──────────────────────────────────────────────┐
│                                              │
│  Junior: "How do I do X?"                    │
│  Mid:    "Let me try Y approach"             │
│  Senior: "Should we do X? Let's consider:    │
│           - effort vs impact                 │
│           - maintenance cost                 │
│           - team skill level                 │
│           - alternatives                     │
│           - trade-offs"                      │
│                                              │
└──────────────────────────────────────────────┘
```

### What Seniors Do Differently

1. **Read more than they write** — Understand code before changing it
2. **Ask "why" 3 times** — Before any implementation, understand the real requirement
3. **Write less code** — Remove code that doesn't add value
4. **Think in systems** — Not just the component, but how it affects the whole app
5. **Document decisions** — Why you chose A over B (for future you and team)
6. **Review ruthlessly** — Code reviews catch bugs, but also teach
7. **Say "I don't know"** — Then go find out. Seniors are comfortable with uncertainty.

---

**You now have the complete roadmap. The guide files contain all the reference code. The lessons contain the hands-on exercises. The path is laid out.**

**The only thing left is to write code. Every day. Start now.**

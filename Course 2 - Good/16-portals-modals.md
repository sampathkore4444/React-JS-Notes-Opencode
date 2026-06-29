# Lesson 16: Portals — Render Outside the Parent

## 1. CONCEPT — Breaking Out of the DOM Tree

> **Real-world analogy:** Portals are like having a separate VIP entrance at a concert. The VIP guest (modal/tooltip) is still part of the concert (React tree) — they bought a ticket, they're on the guest list (have access to Context), they follow the same rules. But they enter through a different door (render at a different DOM location) to avoid blocking the main entrance (overflow:hidden, z-index issues).

Normally, a component renders inside its parent's DOM node. But sometimes you need to render **elsewhere in the DOM** — while keeping the React context.

**Use cases:**
- Modals / dialogs (need to overlay everything, avoid z-index issues)
- Tooltips / popovers (need to overflow parent boundaries)
- Toast notifications (fixed position, outside main content)

```
React tree:              DOM tree:
App                      body
├── Header               ├── #header
├── Board                ├── #board
│   └── Column           │   └── #column
│       └── TaskCard     │       └── #taskcard
│           └── Modal    │           └── #modal  ← nested deep!
└── Footer               └── #footer

With Portal:             body
├── #header              ├── #header
├── #board               ├── #board
│   └── ...              │   └── ...
├── #footer              ├── #footer
└── Modal (portal)       └── #modal  ← at root level!
```

**Portal code:**
```jsx
import { createPortal } from 'react-dom';

function Modal({ children }) {
  return createPortal(
    children,                                           // React node to render
    document.getElementById('modal-root')               // DOM target (outside #root!)
  );
  // createPortal takes TWO arguments:
  // 1. What to render (JSX)
  // 2. Where to render it (DOM element)
}

// Usage — still inside React tree, but renders in different DOM location
function TaskCard({ task }) {
  const [showModal, setShowModal] = useState(false);

  return (
    <div>
      <h3>{task.title}</h3>
      <button onClick={() => setShowModal(true)}>Details</button>

      {showModal && (
        <Modal>
          <div className="modal-backdrop">
            <div className="modal-content">
              <h2>{task.title}</h2>
              <button onClick={() => setShowModal(false)}>Close</button>
            </div>
          </div>
        </Modal>
      )}
    </div>
  );
}
```

## 2. CODE ALONG — Task Detail Modal

### Step 1: Add modal-root to index.html

Open `index.html` and add this inside `<body>`:
```html
<body>
  <div id="root"></div>
  <div id="modal-root"></div>  <!-- ← Add this -->
</body>
```

### Step 2: Create Modal component

`src/components/Modal.jsx`:
```jsx
import { useEffect, useRef } from 'react';
import { createPortal } from 'react-dom';
import { useTheme } from '../contexts/ThemeContext';

function Modal({ isOpen, onClose, title, children }) {
  const { styles } = useTheme();
  const overlayRef = useRef(null);

  // Escape key handler
  useEffect(() => {
    if (!isOpen) return;                           // skip if modal is closed
    function onKeyDown(e) {
      if (e.key === 'Escape') onClose();           // Escape = close modal
    }
    document.addEventListener('keydown', onKeyDown);
    return () => document.removeEventListener('keydown', onKeyDown);
    // Cleanup: remove listener when modal closes or component unmounts
  }, [isOpen, onClose]);

  // Prevent body scroll when modal is open
  useEffect(() => {
    if (isOpen) {
      document.body.style.overflow = 'hidden';     // lock background scrolling
    }
    return () => {
      document.body.style.overflow = '';           // restore scrolling
    };
  }, [isOpen]);

  if (!isOpen) return null;

  return createPortal(
    <div
      ref={overlayRef}
      onClick={(e) => {
        // Close only if clicking the overlay (backdrop), not the modal content
        if (e.target === overlayRef.current) onClose();
      }}
      style={{
        position: 'fixed',
        top: 0, left: 0, right: 0, bottom: 0,
        backgroundColor: 'rgba(0,0,0,0.5)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 1000,
      }}
    >
      <div style={{
        backgroundColor: styles.card,
        color: styles.text,
        borderRadius: '12px',
        padding: '24px',
        maxWidth: '500px',
        width: '90%',
        maxHeight: '80vh',
        overflow: 'auto',
        boxShadow: '0 20px 60px rgba(0,0,0,0.3)',
      }}>
        <div style={{
          display: 'flex',
          justifyContent: 'space-between',
          alignItems: 'center',
          marginBottom: '16px',
        }}>
          <h2 style={{ margin: 0 }}>{title}</h2>
          <button
            onClick={onClose}
            style={{
              border: 'none', background: 'none', cursor: 'pointer',
              fontSize: '24px', color: styles.text, padding: '4px',
            }}
          >
            ✕
          </button>
        </div>
        {children}
      </div>
    </div>,
    document.getElementById('modal-root')
  );
}

export default Modal;
```

### Step 3: Use Modal for task details

Update `src/components/TaskCard.jsx`:
```jsx
import { useState, memo } from 'react';
import Modal from './Modal';
import { useBoardContext } from '../contexts/BoardContext';

const TaskCard = memo(function TaskCard({ task, columnId }) {
  const [showModal, setShowModal] = useState(false);
  const { deleteTask, moveTask, columns } = useBoardContext();
  const otherColumns = Object.values(columns).filter(c => c.id !== columnId);

  const priorityColors = {
    High: { bg: '#fee2e2', text: '#dc2626' },
    Medium: { bg: '#fef3c7', text: '#d97706' },
    Low: { bg: '#dcfce7', text: '#16a34a' },
  };
  const colors = priorityColors[task.priority] || priorityColors.Medium;

  return (
    <>
      <div
        onClick={() => setShowModal(true)}
        style={{
          backgroundColor: 'white',
          padding: '10px',
          marginBottom: '8px',
          borderRadius: '6px',
          boxShadow: '0 1px 3px rgba(0,0,0,0.1)',
          cursor: 'pointer',
        }}
      >
        <div>{task.title}</div>
        <span style={{
          display: 'inline-block', fontSize: '11px', padding: '2px 6px',
          borderRadius: '4px', marginTop: '6px',
          backgroundColor: colors.bg, color: colors.text,
        }}>
          {task.priority}
        </span>
      </div>

      {/* Modal rendered via Portal — appears at document root */}
      <Modal
        isOpen={showModal}
        onClose={() => setShowModal(false)}
        title={task.title}
      >
        <div style={{ marginBottom: '16px' }}>
          <p><strong>Priority:</strong> {task.priority}</p>
          {task.createdAt && (
            <p><strong>Created:</strong> {new Date(task.createdAt).toLocaleString()}</p>
          )}
          {task.completedAt && (
            <p><strong>Completed:</strong> {new Date(task.completedAt).toLocaleString()}</p>
          )}
        </div>

        <div style={{ display: 'flex', gap: '8px', flexWrap: 'wrap' }}>
          {otherColumns.map(col => (
            <button
              key={col.id}
              onClick={() => {
                moveTask(task.id, columnId, col.id);
                setShowModal(false);
              }}
              style={{
                padding: '8px 16px', cursor: 'pointer',
                backgroundColor: '#3b82f6', color: 'white',
                border: 'none', borderRadius: '4px',
              }}
            >
              Move to {col.title}
            </button>
          ))}
          <button
            onClick={() => { deleteTask(columnId, task.id); setShowModal(false); }}
            style={{
              padding: '8px 16px', cursor: 'pointer',
              backgroundColor: '#dc2626', color: 'white',
              border: 'none', borderRadius: '4px',
            }}
          >
            Delete
          </button>
        </div>
      </Modal>
    </>
  );
});

export default TaskCard;
```

### Step 4: Verify it works

Click any task card → a modal opens at the **document root level**. Open DevTools and inspect the DOM — the modal is in `#modal-root`, not nested inside your board. Notice the Escape key closes it.

## 3. WHY PORTALS VS NESTED MODAL

```jsx
// ❌ Modal nested deep in DOM — z-index issues, overflow:hidden problems
<div style={{ overflow: 'hidden' }}>
  <div style={{ position: 'relative', zIndex: 1 }}>
    <div style={{ zIndex: 2 }}>
      <Modal />  {/* Can't escape parent constraints */}
    </div>
  </div>
</div>

// ✅ Portal — no parent constraints, appears at root level
<div id="modal-root">
  <Modal />  {/* z-index: 1000, position: fixed — works reliably */}
</div>
```

## 4. EXERCISE

Create a `Toast` notification system using a portal. When a task is added/moved/deleted, show a small notification at the bottom-right corner that auto-dismisses after 3 seconds.

**Solution:**
```jsx
// src/components/Toast.jsx
import { createPortal } from 'react-dom';
import { useState, useEffect, useCallback } from 'react';
import { createContext, useContext } from 'react';

const ToastContext = createContext(null);

export function ToastProvider({ children }) {
  const [toasts, setToasts] = useState([]);

  const addToast = useCallback((message, type = 'info') => {
    const id = Date.now();
    setToasts(prev => [...prev, { id, message, type }]);
    setTimeout(() => {
      setToasts(prev => prev.filter(t => t.id !== id));
    }, 3000);
  }, []);

  return (
    <ToastContext.Provider value={addToast}>
      {children}
      {createPortal(
        <div style={{
          position: 'fixed', bottom: '20px', right: '20px', zIndex: 2000,
          display: 'flex', flexDirection: 'column', gap: '8px',
        }}>
          {toasts.map(t => (
            <div key={t.id} style={{
              padding: '12px 20px', borderRadius: '8px',
              backgroundColor: t.type === 'success' ? '#dcfce7' : '#fef3c7',
              color: t.type === 'success' ? '#16a34a' : '#d97706',
              boxShadow: '0 4px 12px rgba(0,0,0,0.15)',
              animation: 'slideIn 0.3s ease',
            }}>
              {t.message}
            </div>
          ))}
        </div>,
        document.body
      )}
    </ToastContext.Provider>
  );
}

export const useToast = () => useContext(ToastContext);
```

### CHALLENGE EXERCISE

Create a "Confirm Dialog" component using a portal. It should show a message and two buttons (Confirm/Cancel). Use it when the user tries to delete a task — show the confirm dialog first, and only delete if confirmed.

**Solution:**
```jsx
// src/components/ConfirmDialog.jsx
import { createPortal } from 'react-dom';

function ConfirmDialog({ isOpen, message, onConfirm, onCancel }) {
  if (!isOpen) return null;

  return createPortal(
    <div style={{
      position: 'fixed', top: 0, left: 0, right: 0, bottom: 0,
      backgroundColor: 'rgba(0,0,0,0.5)',
      display: 'flex', alignItems: 'center', justifyContent: 'center',
      zIndex: 2000,
    }}>
      <div style={{
        backgroundColor: 'white', borderRadius: '12px',
        padding: '24px', maxWidth: '400px', width: '90%',
        boxShadow: '0 20px 60px rgba(0,0,0,0.3)',
      }}>
        <h3 style={{ margin: '0 0 12px' }}>Confirm</h3>
        <p style={{ margin: '0 0 20px', color: '#64748b' }}>{message}</p>
        <div style={{ display: 'flex', gap: '8px', justifyContent: 'flex-end' }}>
          <button onClick={onCancel} style={{
            padding: '8px 16px', cursor: 'pointer',
            border: '1px solid #ddd', borderRadius: '4px',
            backgroundColor: 'white',
          }}>
            Cancel
          </button>
          <button onClick={onConfirm} style={{
            padding: '8px 16px', cursor: 'pointer',
            border: 'none', borderRadius: '4px',
            backgroundColor: '#dc2626', color: 'white',
          }}>
            Delete
          </button>
        </div>
      </div>
    </div>,
    document.getElementById('modal-root')
  );
}

export default ConfirmDialog;
```

Use in TaskCard:
```jsx
import { useState } from 'react';
import ConfirmDialog from './ConfirmDialog';

function TaskCard({ task, columnId, onDeleteTask }) {
  const [showConfirm, setShowConfirm] = useState(false);

  function handleDelete() {
    onDeleteTask(columnId, task.id);
    setShowConfirm(false);
  }

  return (
    <div>
      {/* ... task content ... */}
      <button onClick={() => setShowConfirm(true)} style={{ color: '#dc2626' }}>
        Delete
      </button>

      <ConfirmDialog
        isOpen={showConfirm}
        message={`Are you sure you want to delete "${task.title}"?`}
        onConfirm={handleDelete}
        onCancel={() => setShowConfirm(false)}
      />
    </div>
  );
}
```

## 5. COMMON MISTAKES

**Mistake: Missing portal target element**
```jsx
// ❌ Error if #modal-root doesn't exist in HTML
createPortal(children, document.getElementById('modal-root'));

// ✅ Create it if missing
const ModalPortal = ({ children }) => {
  const [container] = useState(() => {
    const el = document.getElementById('modal-root');
    if (el) return el;
    const div = document.createElement('div');
    div.id = 'modal-root';
    document.body.appendChild(div);
    return div;
  });
  return createPortal(children, container);
};
```

### 🐛 DEBUGGING WITH DEVTOOLS

Open the Elements tab (not Components) in DevTools. You'll see the modal rendered inside `<div id="modal-root">`, separate from the main `<div id="root">`. This confirms the portal is working. In React DevTools Components tab, the Modal component still appears under TaskCard in the tree — even though its DOM location is different. This is the key insight: React tree ≠ DOM tree.

### ⚙️ UNDER THE HOOD

`createPortal` uses React's `hydrate` mechanism internally. The portal children are rendered into a different DOM container, but they're still part of the same React tree — they receive context from the nearest provider, events bubble up through the React tree (not the DOM tree), and React manages their lifecycle as if they were rendered in place. Only the DOM output location changes.

### 🏭 PRODUCTION PATTERN

Portals are the standard pattern for modals, tooltips, dropdowns, and toast notifications. Always use a portal for overlays — positioning and z-index bugs are nearly impossible to solve without one. Libraries like `react-hot-toast` and headless UI components use portals internally. The `#modal-root` div pattern is recommended by the React docs.

### 💼 INTERVIEW READY

**Q:** If a Portal renders a component outside its parent's DOM node, does it still have access to context from the parent?
**A:** Yes — absolutely. Portals only change where the DOM output appears. The React component tree maintains its original hierarchy, so Context from ancestors is still available, event bubbling through the React tree works normally, and React manages the component's lifecycle as if it rendered in place. This is the most important portal concept to understand.

### 📝 MINI QUIZ

**1.** What does `createPortal(children, domNode)` do?
- A) Creates a new React component
- B) Renders `children` into `domNode` instead of the parent's DOM location
- C) Creates a new DOM node
- D) Moves the parent component to a new location

**2.** Does a portal component still receive context from its React parent?
- A) No — it's disconnected from the React tree
- B) Yes — portals maintain React tree hierarchy, only the DOM location changes
- C) Only if explicitly passed
- D) Only in development mode

**3.** Why use a portal for a modal instead of a nested div?
- A) Better performance
- B) Avoids z-index and overflow:hidden issues from parent containers
- C) Smaller bundle size
- D) Required by React

**Answers:** 1. B 2. B 3. B

## 6. CHECKPOINT

> Does a component rendered via Portal still have access to React Context from its parent?

**Answer:** Yes! Portals only change where the DOM node appears. The React tree maintains its original hierarchy, so Context, props, and event bubbling still work as if the component was rendered in place.

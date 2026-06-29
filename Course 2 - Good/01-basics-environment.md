# Lesson 1: Your First React App

## 1. CONCEPT — What is React?

> **Real-world analogy:** Setting up a React project is like setting up a kitchen before cooking. Vite is your modern kitchen appliance — it handles all the prep work (bundling, hot reloading, configuration) so you can focus on cooking (building the app). Without it, you'd have to chop every vegetable by hand (configure Webpack, Babel, etc.).

React is a JavaScript library for building **user interfaces**.

Before React, developers wrote code like this:
```js
// Old way: manually create and update DOM
const button = document.createElement('button');
button.textContent = 'Click me';
button.onclick = () => {
  button.textContent = 'Clicked!';
};
document.body.appendChild(button);
```

With React, you write this:
```jsx
// React way: declare what you want
const [text, setText] = useState('Click me');
return <button onClick={() => setText('Clicked!')}>{text}</button>;
```

**Key insight:** You describe *what* the UI should look like. React figures out *how* to update the DOM efficiently. This is called **declarative programming**.

## 2. CODE ALONG — Set Up Your Project

We'll use **Vite** (the modern React build tool). Open your terminal:

```bash
# Create a new React project
# @latest ensures the newest Vite version; --template react scaffolds a React project
npm create vite@latest taskflow -- --template react

# Move into the project folder
# All subsequent commands run inside this directory
cd taskflow

# Install dependencies
# Reads package.json and downloads packages into node_modules/
npm install

# Start the dev server
# Launches Vite with Hot Module Replacement for instant updates
npm run dev
```

You'll see output like:
```
  VITE v5.x.x  ready in 300ms
  ➜  Local:   http://localhost:5173/
```

Open http://localhost:5173/ in your browser. You should see a Vite + React page.

## 3. EXPLORE WHAT WAS CREATED

Open the project in VS Code:
```bash
code .
```

Look at the file structure:
```
taskflow/
├── index.html          # HTML entry point
├── src/
│   ├── main.jsx        # React entry point
│   ├── App.jsx         # Main component
│   └── App.css         # Styles
├── package.json        # Dependencies
└── vite.config.js      # Vite configuration
```

Open **src/App.jsx**. This is your first React component:

```jsx
// App.jsx — A React component (function that returns JSX)

// A React component is just a JavaScript function that returns JSX
function App() {
  return (
    // JSX must have a single root element — this &lt;div&gt; wraps everything
    <div>
      {/* Text content is plain text by default (no curly braces needed) */}
      <h1>Vite + React</h1>
    </div>
  );
}

// Export so other files can import and reuse this component
export default App;
```

**Every React component is just a JavaScript function that returns HTML-like code.**

## 4. YOUR FIRST EDIT

Replace everything in `src/App.jsx` with:

```jsx
function App() {
  return (
    {/* style prop takes a JavaScript object with camelCase keys */}
    <div style={{ padding: '20px' }}>
      {/* JSX lets you mix HTML-like tags with JavaScript expressions */}
      <h1>TaskFlow</h1>           {/* renders as an h1 HTML element */}
      <p>My project management app</p>  {/* paragraph with text */}
    </div>
  );
}
export default App;  // makes this component available for import
```

Save the file. Look at your browser — it updates automatically (this is **Hot Module Replacement** — HMR).

## 5. WHY THIS MATTERS

React lets you think in **components**. A component is a reusable piece of UI. Everything in React is a component. Your entire app will be built by composing small components together.

```
App (component)
├── Header (component)
├── Board (component)
│   ├── Column (component)
│   │   ├── TaskCard (component)
│   │   └── TaskCard (component)
│   └── Column (component)
└── Footer (component)
```

This is called **component composition** — and it's the foundation of React.

## 6. EXERCISE

Try these changes on your own:

1. Change the heading to your name
2. Add a second paragraph describing what you want to build
3. Add an image: `<img src="https://via.placeholder.com/150" alt="placeholder" />`
4. Make the background color light blue using the `style` attribute

**Solution:**
```jsx
function App() {
  return (
    <div style={{ padding: '20px', backgroundColor: '#e0f7fa' }}>
      <h1>Alex's TaskFlow</h1>
      <p>I'm building a project management app to learn React.</p>
      <img src="https://via.placeholder.com/150" alt="placeholder" />
    </div>
  );
}
```

## 7. EXTRA EXERCISE

Create a component that displays your name, a short bio, and a profile picture. Style it with inline styles (backgroundColor, color, borderRadius, padding).

**Solution:**
```jsx
function App() {
  return (
    <div style={{
      backgroundColor: '#f0f8ff',
      color: '#333',
      borderRadius: '12px',
      padding: '24px',
      maxWidth: '400px',
      margin: '40px auto',
      fontFamily: 'Arial, sans-serif',
    }}>
      <img
        src="https://via.placeholder.com/100"
        alt="Profile"
        style={{
          borderRadius: '50%',
          width: '100px',
          height: '100px',
        }}
      />
      <h1>Alex</h1>
      <p>I am a software developer learning React. I enjoy building web applications and solving problems.</p>
    </div>
  );
}

export default App;
```

### CHALLENGE EXERCISE

Create a personal portfolio intro component. Display your name in an `<h1>`, a short bio in a `<p>`, and a profile picture using `<img>`. Use inline styles: `backgroundColor`, `color`, `borderRadius: '50%'`, `padding`, and `textAlign: 'center'`.

**Solution:**
```jsx
function PortfolioIntro() {
  return (
    <div style={{
      backgroundColor: '#f0f8ff',
      color: '#333',
      padding: '30px',
      borderRadius: '12px',
      textAlign: 'center',
      maxWidth: '400px',
      margin: '20px auto',
    }}>
      <img
        src="https://via.placeholder.com/100"
        alt="Profile"
        style={{ borderRadius: '50%', width: '100px', height: '100px' }}
      />
      <h1 style={{ margin: '10px 0' }}>Alex Developer</h1>
      <p>I'm learning React to build awesome web applications.</p>
    </div>
  );
}
```

## 8. COMMON MISTAKES

**Mistake 1:** Multiple root elements
```jsx
// ❌ This will error: "Adjacent JSX elements must be wrapped in an enclosing tag"
return (
  <h1>Title</h1>
  <p>Description</p>
);

// ✅ Fix: Wrap in a single element (or Fragment)
return (
  <div>
    <h1>Title</h1>
    <p>Description</p>
  </div>
);
```

**Mistake 2:** `class` instead of `className`
```jsx
// ❌ Wrong (HTML habit)
<h1 class="title">TaskFlow</h1>

// ✅ Correct (React uses className)
<h1 className="title">TaskFlow</h1>
```

**Make these mistakes intentionally right now** — see the error messages in your browser console. This trains you to recognize them later.

### 🐛 DEBUGGING WITH DEVTOOLS

Install the React DevTools browser extension (Chrome/Firefox). Open DevTools (F12) → "Components" tab. You'll see your entire component tree — currently just `<App>`. Click on it to inspect its props and state on the right panel. As you build more components, this tab becomes your cockpit for debugging.

### ⚙️ UNDER THE HOOD

Vite uses native ES modules during development — no bundling needed. Each file is served as a separate module. In production, Vite bundles with Rollup. `createRoot(document.getElementById('root'))` creates a React "root" that manages a separate component tree outside the DOM. React compares the new virtual DOM with the previous one (reconciliation) and applies only differences to the real DOM.

### 🏭 PRODUCTION PATTERN

Real projects use `npm create vite@latest` or `create-next-app` to scaffold. Manual configuration is rare in 2025 — the community has converged on Vite and Next.js as standards.

### 💼 INTERVIEW READY

**Q:** Explain the Virtual DOM and how React uses it.
**A:** The Virtual DOM is a lightweight JavaScript object tree that mirrors the real DOM. When state changes, React creates a new virtual tree, diffs it against the previous one using a O(n) heuristic, calculates the minimum set of DOM mutations, and applies them in a batch. This is faster than directly manipulating the real DOM because JS object comparisons are cheaper than browser layout recalculations.

### 📝 MINI QUIZ

**1.** What is Vite's primary role in a React project?
- A) State management
- B) Development server and bundler
- C) Database management
- D) CSS framework

**2.** JSX is best described as:
- A) A template language like Handlebars
- B) A syntax extension that compiles to React.createElement calls
- C) A separate file format with .jsx extension
- D) A CSS-in-JS solution

**3.** Why must React components return a single root element?
- A) JavaScript functions can only return one value
- B) JSX compiles to a single function call
- C) React needs one root for the virtual DOM tree
- D) Both A and B

**Answers:** 1. B 2. B 3. D

## 9. CHECKPOINT

Before moving on, answer this:
> What is a React component, and what syntax does it return?

**Answer:** A component is a JavaScript function that returns JSX (HTML-like syntax).

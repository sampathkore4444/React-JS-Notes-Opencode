# Lesson 2: Components & JSX — Building Blocks

## 1. CONCEPT — Components Are Functions That Return UI

> **Real-world analogy:** Components are like LEGO bricks. Each LEGO brick has a specific shape and purpose — one connects, another holds a wheel, another is a window. You snap them together to build something complex. React components work the same way: each is a reusable piece of UI that you compose with others to build your app.

A component takes **input (props)** and returns **output (JSX)**. Just like a function takes arguments and returns a value.

```jsx
// This is a component
function Greeting() {
  return <h1>Hello!</h1>;
}
// This is also a component (arrow function)
const Greeting = () => <h1>Hello!</h1>;
```

You use components like HTML tags:
```jsx
<Greeting />   {/* renders the Greeting component */}
```

## 2. CODE ALONG — Create Your First Custom Components

In your `src/` folder, create a new folder called `components/`:
```bash
mkdir src/components
```

Create `src/components/Header.jsx`:
```jsx
function Header() {
  return (
    // style prop: double curly braces = JS expression + JS object
    <header style={{ borderBottom: '2px solid #333', marginBottom: '20px' }}>
      <h1 style={{ margin: 0 }}>📋 TaskFlow</h1>
      <p>Manage your projects</p>
    </header>
  );
}
export default Header;  // Export so other files can import it
```

Create `src/components/TaskList.jsx`:
```jsx
// TaskList component — renders a hardcoded list of tasks (will become dynamic later)
function TaskList() {
  return (
    <div>
      <h2>Tasks</h2>
      {/* &lt;ul&gt; is the standard HTML unordered list element */}
      <ul>
        {/* Each &lt;li&gt; is a list item — in Lesson 3 these will come from props */}
        <li>Learn React basics</li>
        <li>Build TaskFlow app</li>
        <li>Deploy to production</li>
      </ul>
    </div>
  );
}

export default TaskList;
```

Now update `src/App.jsx` to use these components:
```jsx
// Import the components we created so we can use them here
import Header from './components/Header';
import TaskList from './components/TaskList';

// App is the root component — it composes smaller components together
function App() {
  return (
    // Center the app content with max-width and auto margins
    <div style={{ padding: '20px', maxWidth: '600px', margin: '0 auto' }}>
      {/* Self-closing tag — Header needs no children */}
      <Header />
      {/* TaskList renders below Header */}
      <TaskList />
    </div>
  );
}

export default App;
```

**Check your browser** — you should see the Header and TaskList rendered together.

## 3. WHY THIS MATTERS

Components let you **split the UI into independent, reusable pieces**. Each piece manages its own thing. This is how you build complex apps — not one giant file, but many small components composed together.

Real-world analogy: LEGO blocks. Each block does one thing. You combine them to make something complex.

## 4. EXERCISE

1. Create a new component called `Footer.jsx` with your name and current year
2. Import and use it in `App.jsx` below the `TaskList`
3. Create a `WelcomeMessage.jsx` component that shows "Welcome to TaskFlow" in an `<h2>`

**Solution:**
```jsx
// src/components/Footer.jsx
function Footer() {
  return (
    <footer style={{ marginTop: '40px', color: '#666' }}>
      <p>Built by Alex &copy; {new Date().getFullYear()}</p>
    </footer>
  );
}
export default Footer;

// src/components/WelcomeMessage.jsx
function WelcomeMessage() {
  return <h2>Welcome to TaskFlow</h2>;
}
export default WelcomeMessage;

// App.jsx
import Header from './components/Header';
import WelcomeMessage from './components/WelcomeMessage';
import TaskList from './components/TaskList';
import Footer from './components/Footer';

function App() {
  return (
    <div style={{ padding: '20px', maxWidth: '600px', margin: '0 auto' }}>
      <Header />
      <WelcomeMessage />
      <TaskList />
      <Footer />
    </div>
  );
}
export default App;
```

## 5. EXTRA EXERCISE

Create a `Navigation` component with links (Home, About, Contact rendered as styled divs). Create a `HeroSection` component with a title and subtitle. Compose them together in App.jsx.

**Solution:**
```jsx
// src/components/Navigation.jsx
function Navigation() {
  const linkStyle = {
    padding: '8px 16px',
    marginRight: '8px',
    backgroundColor: '#0066cc',
    color: 'white',
    borderRadius: '4px',
    cursor: 'pointer',
    display: 'inline-block',
  };

  return (
    <nav style={{ marginBottom: '20px' }}>
      <div style={linkStyle}>Home</div>
      <div style={linkStyle}>About</div>
      <div style={linkStyle}>Contact</div>
    </nav>
  );
}

export default Navigation;
```

```jsx
// src/components/HeroSection.jsx
function HeroSection() {
  return (
    <div style={{
      backgroundColor: '#e8f4fd',
      padding: '40px',
      borderRadius: '8px',
      textAlign: 'center',
      marginBottom: '20px',
    }}>
      <h1>Welcome to My App</h1>
      <p style={{ color: '#666', fontSize: '18px' }}>Building great things with React</p>
    </div>
  );
}

export default HeroSection;
```

```jsx
// App.jsx
import Navigation from './components/Navigation';
import HeroSection from './components/HeroSection';

function App() {
  return (
    <div style={{ padding: '20px', maxWidth: '600px', margin: '0 auto' }}>
      <Navigation />
      <HeroSection />
    </div>
  );
}

export default App;
```

### CHALLENGE EXERCISE

Create a `ProductCard` component that shows a product image, name, price, and an "Add to Cart" button. Then create a `ProductGrid` component that arranges three `ProductCard` components in a horizontal row using flexbox.

**Solution:**
```jsx
// src/components/ProductCard.jsx
function ProductCard() {
  return (
    <div style={{
      border: '1px solid #ddd',
      borderRadius: '8px',
      padding: '16px',
      textAlign: 'center',
      width: '200px',
    }}>
      <div style={{ fontSize: '48px', marginBottom: '8px' }}>📱</div>
      <h3 style={{ margin: '8px 0' }}>Smartphone</h3>
      <p style={{ color: '#0066cc', fontWeight: 'bold' }}>$599</p>
      <button style={{
        padding: '8px 16px',
        backgroundColor: '#0066cc',
        color: 'white',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
      }}>
        Add to Cart
      </button>
    </div>
  );
}
export default ProductCard;

// src/components/ProductGrid.jsx
import ProductCard from './ProductCard';
function ProductGrid() {
  return (
    <div style={{
      display: 'flex',
      gap: '20px',
      justifyContent: 'center',
      padding: '20px',
    }}>
      <ProductCard />
      <ProductCard />
      <ProductCard />
    </div>
  );
}
export default ProductGrid;
```

## 6. COMMON MISTAKES

**Mistake: Forgetting to export the component**
```jsx
// ❌ Other files can't import this
function Header() {
  return <h1>TaskFlow</h1>;
}
// Missing: export default Header;

// ✅ Always export
export default Header;
```

**Mistake: Importing from wrong path**
```jsx
// ❌ Wrong path
import Header from './Header';

// ✅ Correct path (relative to current file)
import Header from './components/Header';
```

**Mistake: Component name not capitalized**
```jsx
// ❌ React treats lowercase tags as HTML
function header() { ... }
<header />  {/* React renders <header> HTML element */}

// ✅ Capitalized = React component
function Header() { ... }
<Header />  {/* React renders your component */}
```

### 🐛 DEBUGGING WITH DEVTOOLS

Open the Components tab in React DevTools. You'll see `<App> → <Header>, <TaskList>`. This mirrors your component hierarchy. Click any component to see its source location. If a component doesn't appear, it either isn't rendered or returned null. Missing components = first place to check.

### ⚙️ UNDER THE HOOD

React distinguishes components from HTML elements by the first character of the tag name. Uppercase (`<Header>`) → lookup in scope as a component function. Lowercase (`<header>`) → pass to `document.createElement('header')`. This is why component names MUST be capitalized — React literally checks `typeof tag === 'string' && tag[0] === tag[0].toLowerCase()`.

### 🏭 PRODUCTION PATTERN

Teams often use barrel exports: create `src/components/index.js` that re-exports all components (`export { Header } from './Header'`). Then import from `'../components'` instead of `'../components/Header'`. This centralizes imports and hides file reorganization.

### 💼 INTERVIEW READY

**Q:** What's the difference between a React component and a regular JavaScript function?
**A:** A component returns JSX (or null/fragment), is invoked as `<Component />`, and React manages its lifecycle — mounting, updating, and unmounting. A regular function returns any value and has no lifecycle. Components also receive props and can use hooks.

### 📝 MINI QUIZ

**1.** What determines whether `<box />` is an HTML element or a React component?
- A) The file extension (.jsx vs .js)
- B) Capitalization — `<Box />` is a component, `<box />` is HTML
- C) Whether it's imported
- D) The className attribute

**2.** If you forget `export default Header`, what happens?
- A) The app crashes
- B) Other files can't import Header — `import Header from './Header'` returns undefined
- C) React automatically exports it
- D) The component renders but with a warning

**3.** What does component composition mean?
- A) One component renders another component inside it
- B) Combining CSS classes
- C) Merging props from multiple sources
- D) Calling functions in sequence

**Answers:** 1. B 2. B 3. A

## 7. CHECKPOINT

> What rule determines if `<header />` is an HTML tag or a React component?

**Answer:** Capitalization — `<Header />` is a React component, `<header />` is an HTML element.

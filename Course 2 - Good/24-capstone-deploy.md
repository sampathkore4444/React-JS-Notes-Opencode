# Lesson 24: Capstone — Build Something Real & Deploy

## 1. YOUR FINAL CHALLENGE

You've learned 23 lessons. Now it's time to **build without a tutorial** — the only way to go from "I followed along" to "I can build anything."

### The Challenge: Build a "Recipe Box" App

Build a recipe management app from scratch using everything you've learned. No step-by-step guide. Only requirements.

### Requirements

```
Feature 1: Recipe List
  ├── Show all recipes in a grid
  ├── Search by name
  ├── Filter by category (breakfast, lunch, dinner, dessert)
  └── Sort by name or date added

Feature 2: Recipe Detail
  ├── Title, image, category, prep time
  ├── Ingredients list (with checkboxes to mark as "have it")
  ├── Instructions (numbered steps)
  └── Delete button with confirmation

Feature 3: Add/Edit Recipe
  ├── Form with all fields
  ├── Add ingredients dynamically (+ button to add rows)
  ├── Validation (title required, min 1 ingredient, min 1 step)
  └── Save to localStorage + sync (optional: save to API)

Feature 4: Favorites
  ├── Star/unstar recipes
  ├── Filter to show only favorites
  └── Favorites persist

Feature 5: Shopping List
  ├── Auto-generate from recipe ingredients
  ├── Manual add/remove items
  └── Check off items as purchased
```

### What You Must Use

| Concept | Where to Apply |
|---------|---------------|
| React Router | Recipe list page, recipe detail page (dynamic route), add/edit page |
| useState + useReducer | Recipe form state, shopping list |
| useEffect + fetch | Load recipes from localStorage (or API if you made one) |
| Custom hooks | `useRecipes`, `useShoppingList`, `useLocalStorage` |
| Context or Zustand | Theme, favorites, or global recipe state |
| React.memo/useMemo | Recipe card list, filtered/sorted recipes |
| Portals | Confirmation modal for delete |
| Error Boundary | Wrap each main section |
| Optimistic update | Toggle favorite instantly |
| TypeScript (optional) | Define Recipe, Ingredient, Step types |

## 2. THE PROCESS — NOT THE CODE

This section teaches you **how to approach any React project**.

### Step 1: Plan before coding

Write down your component tree and data flow:
```
App
├── Header
│   ├── SearchInput
│   └── NavLinks
├── Routes
│   ├── RecipeListPage
│   │   ├── FilterBar
│   │   └── RecipeGrid
│   │       └── RecipeCard (×N)
│   ├── RecipeDetailPage
│   │   ├── RecipeInfo
│   │   ├── IngredientList
│   │   │   └── IngredientItem (×N)
│   │   └── InstructionList
│   └── RecipeFormPage (add + edit)
└── Footer
```

### Step 2: Data modeling

```typescript
interface Recipe {
  id: string;
  title: string;
  image: string;
  category: 'breakfast' | 'lunch' | 'dinner' | 'dessert';
  prepTime: number; // minutes
  ingredients: { name: string; amount: string; have: boolean }[];
  instructions: string[];
  favorite: boolean;
  createdAt: string;
}
```

### Step 3: Start with state management (no UI)

```jsx
// hooks/useRecipes.js — build and test BEFORE creating any components
export function useRecipes() {
  const [recipes, setRecipes] = useLocalStorage('recipes', []);

  const addRecipe = (recipe) => { ... };
  const updateRecipe = (id, changes) => { ... };
  const deleteRecipe = (id) => { ... };
  const toggleFavorite = (id) => { ... };

  const getRecipeById = (id) => recipes.find(r => r.id === id);

  // Derived
  const favoriteRecipes = recipes.filter(r => r.favorite);

  return { recipes, addRecipe, updateRecipe, deleteRecipe, toggleFavorite, getRecipeById, favoriteRecipes };
}
```

**Test the hook** by temporarily logging in console, then remove logs and build UI.

### Step 4: Build from leaf components up

1. `IngredientItem` — one ingredient row (just a checkbox + text)
2. `IngredientList` — list of IngredientItems
3. `RecipeCard` — one card in the grid
4. `RecipeGrid` — grid of RecipeCards
5. `RecipeListPage` — page with Search + Filter + Sort + Grid

**Each component tests in isolation** before composing.

### Step 5: Connect pieces

```
useRecipes hook → RecipeListPage → RecipeGrid → RecipeCard
               → RecipeDetailPage
               → RecipeFormPage
```

### Step 6: Polish

- Loading states (even from localStorage, simulate delay)
- Error states (what if localStorage is corrupt?)
- Empty states (no recipes yet, no search results)
- Responsive design
- Keyboard navigation
- Accessibility (labels, roles, focus management)

## 3. DEPLOYMENT — SHOW THE WORLD

### Option 1: Vercel (easiest)

```bash
# 1. Push code to GitHub
git init
git add .
git commit -m "Recipe Box app"
# Create repo on github.com, then:
git remote add origin https://github.com/YOUR_USERNAME/recipe-box.git
git push -u origin main

# 2. Go to vercel.com
# 3. Import your GitHub repo
# 4. Click "Deploy"
# Done! Your app is live at recipe-box.vercel.app
```

### Option 2: Netlify

```bash
npm run build  # creates dist/ folder
npx netlify deploy --prod --dir=dist
```

### Option 3: GitHub Pages

```bash
npm install -D gh-pages
# Add to package.json: "predeploy": "npm run build", "deploy": "gh-pages -d dist"
npm run deploy
```

## 4. WHAT YOU'VE LEARNED — RECAP

Go through this checklist. If you can explain **and** code each one, you're ready for senior interviews:

```
✅ JSX & Components
✅ Props & Props drilling
✅ useState & State immutability
✅ Events & Forms (Controlled components)
✅ Lists & Keys
✅ Conditional rendering
✅ Lifting state up
✅ useEffect & Cleanup
✅ Custom hooks
✅ useReducer
✅ useRef (DOM + mutable values)
✅ Context API
✅ React.memo / useMemo / useCallback
✅ Portals
✅ Error Boundaries
✅ React Router (routes, params, navigation)
✅ API calls with fetch (loading, error, race conditions)
✅ React Query (caching, mutations, optimistic updates)
✅ Zustand (or other state management)
✅ Optimistic updates
✅ Testing (Vitest + React Testing Library)
✅ TypeScript with React
✅ Project planning & architecture
✅ Deployment
```

## 5. THE FINAL EXERCISE — TEACH SOMEONE

The best way to prove mastery is to **teach**. Pick any concept from this course and:

1. Explain it verbally (record yourself, listen back — is it clear?)
2. Write a blog post about it
3. Build a tiny example project for it
4. Answer someone's question on StackOverflow or Discord

**If you can't explain it simply, you don't understand it well enough.**

## 6. THE ROAD AHEAD

```
You Are Here → Junior React Developer
                    │
                    ▼
        Build 3 more projects from scratch
        (no tutorials, no copy-paste)
                    │
                    ▼
        Read real codebases (Next.js, shadcn/ui, Radix UI)
                    │
                    ▼
        Contribute to open source (fix a bug in a React library)
                    │
                    ▼
        Teach others → blog posts, talks, mentoring
                    │
                    ▼
        Senior React Developer
```

**You now know everything I know. The difference between us is one thing: I've built 100 projects. You've built 1. Build 99 more.**

---

**End of course. You have what it takes. Now go build. 🚀**

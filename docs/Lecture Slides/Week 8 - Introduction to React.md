---
share_cis4004: "true"
site-folder: docs/Lecture Slides
theme: ucf-knights.css
height: "1080"
width: "1920"
---

# React Introduction
## Building UIs with Components & TypeScript

---

## What We'll Cover Today

- Why React exists and what problem it solves
- Components: the building block of React
- JSX/TSX: writing HTML inside TypeScript
- Props: passing data into components
- State: making components dynamic
- Handling events
- Putting it all together

> *50 min lecture → 60 min lab: you'll build a working UI*

---

### The Problem React Solves

Without React: Manual DOM updates

```html
<!-- index.html -->
<div id="count">0</div>
<button onclick="increment()">+</button>
```

```javascript
// app.js
let count = 0;
function increment() {
  count++;
  // We must manually find & update
  document.getElementById('count').textContent = count;
}
```

> Every change = find the element, update it, remember to do it everywhere. **It scales badly.**

---

### The Problem React Solves
With React: Describe what it *should* look like

```tsx
// Counter.tsx
function Counter() {
  // React tracks this value
  const [count, setCount] = useState(0);

  // Just describe the UI — React handles the DOM
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>
        +
      </button>
    </div>
  );
}
```
---

> **Declarative:** you say *what* to show; React figures out *how* to update the page.

---

## What Is React?

- A **JavaScript/TypeScript library** for building user interfaces
- Created by Facebook/Meta in 2013
- The most widely-used front-end library today
- Core idea: **UI = f(state)**

> The UI is a *function* of your data. Change the data → React updates the screen.

### What React is NOT:

- Not a full framework (no router, no HTTP client built in)
- Not a templating engine
- Not a replacement for HTML/CSS — it *uses* them

---

## Setting Up a React + TypeScript Project

```bash
# Vite is the recommended modern
# way to scaffold a React project
npm create vite@latest my-app \
  -- --template react-ts

cd my-app
npm install
npm run dev
```
---

### What you get:

```
my-app/
├── src/
│   ├── App.tsx        ← main component
│   ├── main.tsx       ← entry point
│   └── index.css
├── index.html
└── package.json
```

---

## Entry Point: `main.tsx`

```tsx
// main.tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import App from './App'

// Find the <div id="root"> in index.html
// and hand it to React
createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <App />
  </StrictMode>
)
```

> React takes over **one** `<div>` in your HTML. Everything inside is React's world.

---

## Components: The Core Idea

> A **component** is a reusable piece of UI — like a custom HTML tag you design yourself.

### Think of UI as a tree of components:

```
<App>
  ├── <Header />
  ├── <ProductList>
  │     ├── <ProductCard />
  │     ├── <ProductCard />
  │     └── <ProductCard />
  └── <Footer />
```

Each box = one `.tsx` file, one function, one job.

---

## Your First Component

```tsx
// Greeting.tsx

// A component is just a TypeScript
// function that returns JSX
function Greeting() {
  return (
    <div>
      <h1>Hello, World!</h1>
      <p>Welcome to React.</p>
    </div>
  );
}

// Always export your component
export default Greeting;
```
---
## Rules:
1. Function name **must** start with a capital letter
2. Must return **one** root element (wrap in `<div>` or `<>`)
3. File extension is `.tsx` (TypeScript + JSX)

---

## What Is JSX / TSX?

> **JSX** = JavaScript XML. A syntax extension that lets you write *HTML-like* markup inside TypeScript.

### It looks like HTML — but it's not:

| HTML               | TSX                     |
| ------------------ | ----------------------- |
| `class="btn"`      | `className="btn"`       |
| `for="name"`       | `htmlFor="name"`        |
| `<br>`             | `<br />` (self-closing) |
| `onclick="fn()"`   | `onClick={fn}`          |
| `<!-- comment -->` | `{/* comment */}`       |

> JSX compiles down to `React.createElement()` calls. It's just a nicer syntax.

---

<style>
.reveal pre {
	font-size: 0.8em;
}
</style>
### Embedding Expressions in TSX

Use **`{ }`** (curly braces) to drop any TypeScript expression into your markup:

```tsx
function UserCard() {
  const name = "Alex";
  const score = 42;
  const isAdmin = true;

  return (
    <div>
      {/* Variables */}
      <h2>{name}</h2>
      
      {/* Math & expressions */}
      <p>Score: {score * 2}</p>
      
      {/* Ternary (if/else inline) */}
      <p>{isAdmin ? "Admin" : "User"}</p>
    </div>
  );
}
```

> Anything valid as a TypeScript *expression* can go inside `{ }`.

---

## Rendering Lists

```tsx
function FruitList() {
  const fruits = [
    "Apple",
    "Banana",
    "Cherry"
  ];

  return (
    <ul>
      {/* .map() turns array → JSX */}
      {fruits.map((fruit) => (
        // key helps React track items
        <li key={fruit}>
          {fruit}
        </li>
      ))}
    </ul>
  );
}
```

> ⚠️ Always add a **`key`** prop when rendering lists. Use a unique ID, not the array index if you can avoid it.

---

## Props: Passing Data In

> **Props** (properties) let you pass data *into* a component — like attributes on an HTML tag.

```tsx
// 1. Define the shape with a type
type GreetingProps = {
  name: string;
  age: number;
};

// 2. Receive props as a parameter
function Greeting(props: GreetingProps) {
  return (
    <p>
      Hi {props.name},
      you are {props.age} years old.
    </p>
  );
}

// 3. Use it like an HTML tag
// <Greeting name="Sam" age={21} />
```

---

#### Props: Destructuring (Cleaner Syntax)

```tsx
type CardProps = {
  title: string;
  description: string;
  imageUrl: string;
};

// Destructure props directly
function Card({
  title,
  description,
  imageUrl,
}: CardProps) {
  return (
    <div className="card">
      <img src={imageUrl} alt={title} />
      <h3>{title}</h3>
      <p>{description}</p>
    </div>
  );
}
```

> TypeScript will catch mismatched prop types *before* you run the code. This is why we use `.tsx`.

---

### Props: Using a Component

```tsx
// App.tsx
import Card from './Card';

function App() {
  return (
    <div>
      <Card
        title="React Basics"
        description="Learn React step by step"
        imageUrl="/react.png"
      />
      <Card
        title="TypeScript"
        description="Types save you from bugs"
        imageUrl="/ts.png"
      />
    </div>
  );
}
```

> One component definition → used many times with different data. **This is the power of components.**

---

## Props Are Read-Only

```tsx
type LabelProps = {
  text: string;
};

function Label({ text }: LabelProps) {
  // ✅ Reading props is fine
  return <span>{text}</span>;

  // ❌ NEVER mutate props
  // text = "different"; // Error!
}
```

> Props flow **down** from parent → child.
> A child component never changes its own props.
>
> If data needs to change, that's a job for **State**.

---

## State: Making Components Dynamic

> **State** is data that belongs to a component and can change over time. When state changes → React re-renders the component.

```tsx
import { useState } from 'react';

function Counter() {
  // useState returns [value, setter]
  // TypeScript infers type as number
  const [count, setCount] = useState(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={
        // Call the setter to update
        () => setCount(count + 1)
      }>
        Increment
      </button>
    </div>
  );
}
```

---

## `useState` Anatomy

```tsx
const [count, setCount] = useState(0);
//     ^       ^                   ^
//     |       |                   |
//     |       |          initial value
//     |       |
//     |    function to UPDATE the value
//     |    (triggers a re-render)
//     |
//   current value (read-only here)
```

---
### TypeScript explicit type (when needed):

```tsx
// Inferred (TypeScript figures it out)
const [name, setName] = useState('');

// Explicit (for complex types)
const [user, setUser] =
  useState<User | null>(null);
```

---

## State with Strings & Booleans

```tsx
import { useState } from 'react';

function ToggleMessage() {
  const [visible, setVisible] =
    useState(false);

  return (
    <div>
      <button onClick={
        () => setVisible(!visible)
      }>
        {visible ? 'Hide' : 'Show'}
      </button>

      {/* Conditional rendering */}
      {visible && (
        <p>Now you see me!</p>
      )}
    </div>
  );
}
```

---

## Props vs State — The Key Distinction

|                          | Props             | State              |
| ------------------------ | ----------------- | ------------------ |
| Who owns it?             | Parent            | Component itself   |
| Can it change?           | No (read-only)    | Yes (via setter)   |
| Where does it come from? | Passed in         | `useState()`       |
| What triggers re-render? | Parent re-renders | Calling the setter |

---

## Mental model:
- **Props** = settings passed to you by someone else
- **State** = your own internal memory

---

### Handling Events

```tsx
function EventDemo() {
  // Button click
  function handleClick() {
    console.log('Button clicked!');
  }

  // Input change
  function handleChange(
    // TypeScript needs the event type
    e: React.ChangeEvent<HTMLInputElement>
  ) {
    console.log(e.target.value);
  }

  return (
    <div>
      <button onClick={handleClick}>
        Click me
      </button>
      <input onChange={handleChange} />
    </div>
  );
}
```

---

### Controlled Inputs

> A **controlled input** ties an `<input>` to a state variable. React is the *single source of truth*.

```tsx
import { useState } from 'react';

function NameInput() {
  const [name, setName] = useState('');

  return (
    <div>
      <input
        // value comes FROM state
        value={name}
        // changes go BACK to state
        onChange={(e) => setName(e.target.value)}
        placeholder="Enter your name"
      />
      {/* Live preview */}
      <p>Hello, {name || 'stranger'}!</p>
    </div>
  );
}
```

---

### Putting It All Together

```tsx
// A small, complete React component
import { useState } from 'react';

type Item = { id: number; text: string };

function TodoList() {
  const [items, setItems] =
    useState<Item[]>([]);
  const [input, setInput] = useState('');

  function addItem() {
    if (!input.trim()) return;
    setItems([
      ...items,
      { id: Date.now(), text: input }
    ]);
    setInput('');
  }

  return ( /* next slide */ );
}
```

---

### Putting It All Together (cont.)

```tsx
  // ...continued from previous slide
  return (
    <div>
      <input
        value={input}
        onChange={(e) =>
          setInput(e.target.value)
        }
        placeholder="Add a task..."
      />
      <button onClick={addItem}>
        Add
      </button>
      <ul>
        {items.map((item) => (
          <li key={item.id}>
            {item.text}
          </li>
        ))}
      </ul>
    </div>
  );
```

---

## The React Mental Model — Summary



```
      DATA FLOWS DOWN (props)
            │
     ┌──────▼───────┐
     │    App.tsx   │  ← owns state
     └──────┬───────┘
            │ props
     ┌──────▼───────┐
     │  Component   │  ← displays data
     └──────┬───────┘
            │ events bubble UP
     ┌──────▼───────┐
     │  User Action │  (click, type…)
     └─────────────-┘
            │ calls setter
            └──── state updates
                  → re-render
```

> **Data down, events up.** This is the React data flow.

---

## Key Concepts Recap

| Concept     | What it is           | Syntax                             |
| ----------- | -------------------- | ---------------------------------- |
| Component   | Reusable UI function | `function Foo() { return <div/> }` |
| JSX/TSX     | HTML-in-TypeScript   | `<h1>{value}</h1>`                 |
| Props       | Data passed in       | `function Foo({ name }: Props)`    |
| State       | Component's own data | `const [x, setX] = useState(0)`    |
| Event       | User interaction     | `onClick={handler}`                |
| Conditional | Show/hide UI         | `{flag && <div/>}`                 |
| Lists       | Render arrays        | `arr.map(i => <li key={i.id}>)`    |

---
<style>
.reveal pre {
	font-size: 1.2em;
}
</style>
## Common Mistakes to Avoid

❌ **Mutating state directly**
```tsx
// WRONG — React won't see the change
items.push(newItem);

// RIGHT — create a new array
setItems([...items, newItem]);
```
---
## Common Mistakes to Avoid

❌ **Missing `key` on list items**
```tsx
// WRONG
{names.map(n => <li>{n}</li>)}

// RIGHT
{names.map(n => <li key={n}>{n}</li>)}
```
---
## Common Mistakes to Avoid

❌ **Component name starts lowercase**
```tsx
// WRONG — React treats this as HTML
function card() { ... }

// RIGHT
function Card() { ... }
```

---

## What's Coming Next

### Demo (60 min): Profile Card Builder
You'll build a multi-component app with:
- A form component with controlled inputs
- A live-preview `ProfileCard` component
- State lifted between components



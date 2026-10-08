# React Essentials -- A Crash Course for Angular Developers

## Table of Contents
1. [React Philosophy vs Angular Philosophy](#react-philosophy-vs-angular-philosophy)
2. [JSX -- Your New Template Language](#jsx----your-new-template-language)
3. [Components: Functional vs Class](#components-functional-vs-class)
4. [Props -- Angular @Input/@Output Equivalent](#props----angular-inputoutput-equivalent)
5. [State with useState](#state-with-usestate)
6. [useEffect -- Lifecycle Hooks Reimagined](#useeffect----lifecycle-hooks-reimagined)
7. [useContext -- Angular Services/DI Equivalent](#usecontext----angular-servicesdi-equivalent)
8. [useRef, useMemo, useCallback](#useref-usememo-usecallback)
9. [Custom Hooks -- Angular Services Reimagined](#custom-hooks----angular-services-reimagined)
10. [Event Handling Differences](#event-handling-differences)
11. [Conditional Rendering](#conditional-rendering)
12. [Lists and Keys](#lists-and-keys)
13. [Component Tree and Data Flow](#component-tree-and-data-flow)
14. [Interview Questions](#interview-questions)

---

## React Philosophy vs Angular Philosophy

| Aspect | Angular | React |
|---|---|---|
| **Type** | Full framework (batteries included) | UI library (bring your own batteries) |
| **Language** | TypeScript (required) | JavaScript or TypeScript (optional) |
| **Templates** | HTML with directives (`*ngIf`, `*ngFor`) | JSX (JavaScript + HTML mixed) |
| **Data Binding** | Two-way binding `[(ngModel)]` | One-way data flow (top-down) |
| **State Management** | Services + RxJS, NgRx | useState, useReducer, Context, Zustand, Redux |
| **Dependency Injection** | Built-in DI system | No DI -- use imports, Context, or props |
| **Routing** | `@angular/router` (built-in) | React Router (third-party) or Next.js file routing |
| **Forms** | Reactive Forms / Template Forms (built-in) | React Hook Form, Formik (third-party) |
| **HTTP** | HttpClient (built-in) | fetch, Axios, SWR, React Query (third-party) |
| **Change Detection** | Zone.js + dirty checking | Virtual DOM diffing |
| **CLI** | Angular CLI (`ng`) | Create React App, Vite, Next.js |
| **Learning Curve** | Steeper (many built-in concepts) | Gentler start, complexity grows with ecosystem |
| **Opinionation** | Highly opinionated | Unopinionated -- you choose the patterns |

### The Core Mental Shift

```
Angular Thinking:                    React Thinking:
+----------------------------+       +----------------------------+
| "Everything is a class"    |       | "Everything is a function" |
| Decorators define behavior |       | Hooks add behavior         |
| Templates are separate     |       | UI is just JavaScript      |
| DI wires things together   |       | Import what you need       |
| RxJS for async             |       | Promises/async-await       |
| Modules organize code      |       | Files/folders organize     |
+----------------------------+       +----------------------------+
```

**Key insight:** In Angular, you declare what should happen (declarative templates with imperative class logic). In React, your component IS a function that returns UI. The function runs, returns JSX, and React figures out what changed.

---

## JSX -- Your New Template Language

JSX is JavaScript XML. It looks like HTML but lives inside your JavaScript/TypeScript files.

### Angular Template vs JSX

**Angular:**
```html
<!-- app.component.html (separate file) -->
<div class="container">
  <h1>{{ title }}</h1>
  <p *ngIf="isVisible">Hello, {{ user.name }}!</p>
  <ul>
    <li *ngFor="let item of items">{{ item.label }}</li>
  </ul>
  <button (click)="handleClick()" [disabled]="isLoading">
    Click Me
  </button>
</div>
```

```typescript
// app.component.ts
@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  title = 'My App';
  isVisible = true;
  isLoading = false;
  user = { name: 'John' };
  items = [{ label: 'One' }, { label: 'Two' }];

  handleClick() {
    console.log('clicked');
  }
}
```

**React (equivalent):**
```tsx
// App.tsx (template + logic in ONE file)
function App() {
  const title = 'My App';
  const isVisible = true;
  const isLoading = false;
  const user = { name: 'John' };
  const items = [{ label: 'One' }, { label: 'Two' }];

  const handleClick = () => {
    console.log('clicked');
  };

  return (
    <div className="container">
      <h1>{title}</h1>
      {isVisible && <p>Hello, {user.name}!</p>}
      <ul>
        {items.map((item, index) => (
          <li key={index}>{item.label}</li>
        ))}
      </ul>
      <button onClick={handleClick} disabled={isLoading}>
        Click Me
      </button>
    </div>
  );
}
```

### JSX Rules to Remember

| Rule | Angular | React JSX |
|---|---|---|
| CSS class | `class="foo"` | `className="foo"` |
| Inline style | `[style.color]="'red'"` | `style={{ color: 'red' }}` |
| Event binding | `(click)="fn()"` | `onClick={fn}` |
| Property binding | `[disabled]="val"` | `disabled={val}` |
| Interpolation | `{{ value }}` | `{value}` |
| for attribute | `for="id"` | `htmlFor="id"` |
| Self-closing tags | Optional | Required for void elements `<img />` |
| Must return one root | N/A (template) | Yes -- use `<>...</>` (Fragment) |

### JSX Expressions

```tsx
// You can put ANY JavaScript expression inside {}
function Example() {
  const score = 85;

  return (
    <div>
      {/* Math */}
      <p>Double: {score * 2}</p>

      {/* Ternary (like Angular's ternary in templates) */}
      <p>{score > 80 ? 'Great!' : 'Keep trying'}</p>

      {/* Function calls */}
      <p>{score.toFixed(2)}</p>

      {/* Template literals */}
      <p>{`Your score is ${score}`}</p>

      {/* Comments in JSX look like this */}
      {/* NOT like <!-- HTML comments --> */}
    </div>
  );
}
```

---

## Components: Functional vs Class

### Angular Component vs React Component

```
Angular Component:                    React Functional Component:
+---------------------------+         +---------------------------+
| @Component({              |         | function MyComponent() {  |
|   selector: 'app-my',     |         |   // hooks go here        |
|   template: '...',        |         |   return <div>...</div>;  |
|   styles: ['...']         |         | }                         |
| })                        |         +---------------------------+
| export class MyComponent  |
|   implements OnInit {     |         React Class Component (legacy):
|   @Input() data: string;  |         +---------------------------+
|   ngOnInit() { ... }      |         | class MyComponent         |
| }                         |         |   extends React.Component |
+---------------------------+         | {                         |
                                      |   render() {              |
                                      |     return <div>...</div>;|
                                      |   }                       |
                                      | }                         |
                                      +---------------------------+
```

**Important:** Modern React uses ONLY functional components. Class components are legacy. If you see them in interviews, know they exist but always write functional components.

### Functional Component (Modern -- USE THIS)

```tsx
// Greeting.tsx
interface GreetingProps {
  name: string;
  age?: number;  // optional prop
}

function Greeting({ name, age }: GreetingProps) {
  return (
    <div>
      <h1>Hello, {name}!</h1>
      {age && <p>You are {age} years old</p>}
    </div>
  );
}

export default Greeting;

// Usage:
// <Greeting name="Alice" age={30} />
```

### Class Component (Legacy -- KNOW IT, DON'T USE IT)

```tsx
// Same component as a class (legacy style)
import React, { Component } from 'react';

interface GreetingProps {
  name: string;
  age?: number;
}

class Greeting extends Component<GreetingProps> {
  render() {
    const { name, age } = this.props;
    return (
      <div>
        <h1>Hello, {name}!</h1>
        {age && <p>You are {age} years old</p>}
      </div>
    );
  }
}
```

### Angular vs React Component Comparison

| Feature | Angular | React |
|---|---|---|
| Define component | `@Component` decorator on a class | A function returning JSX |
| Template | Separate HTML file or inline `template` | JSX returned from function |
| Styles | `styleUrls` or `styles` array | CSS Modules, Tailwind, styled-components |
| Selector/Usage | `<app-greeting>` | `<Greeting />` (PascalCase) |
| Inputs | `@Input() name: string` | Props: `function Greeting({ name })` |
| Outputs | `@Output() clicked = new EventEmitter()` | Callback props: `onClick={() => {}}` |
| Lifecycle | `ngOnInit`, `ngOnDestroy`, etc. | `useEffect` hook |
| Local state | Class properties | `useState` hook |

---

## Props -- Angular @Input/@Output Equivalent

Props (properties) are how React passes data DOWN the component tree. They are READ-ONLY.

### @Input equivalent -- Passing Data Down

**Angular:**
```typescript
// child.component.ts
@Component({ selector: 'app-child', template: '<p>{{ message }}</p>' })
export class ChildComponent {
  @Input() message: string = '';
}

// parent template:
// <app-child [message]="'Hello'"></app-child>
```

**React:**
```tsx
// Child.tsx
interface ChildProps {
  message: string;
}

function Child({ message }: ChildProps) {
  return <p>{message}</p>;
}

// Parent.tsx
function Parent() {
  return <Child message="Hello" />;
}
```

### @Output equivalent -- Passing Callbacks Up

**Angular:**
```typescript
// child.component.ts
@Component({
  selector: 'app-child',
  template: '<button (click)="onClick()">Click</button>'
})
export class ChildComponent {
  @Output() clicked = new EventEmitter<string>();

  onClick() {
    this.clicked.emit('child was clicked');
  }
}

// parent template:
// <app-child (clicked)="handleChildClick($event)"></app-child>
```

**React:**
```tsx
// Child.tsx
interface ChildProps {
  onClicked: (message: string) => void;
}

function Child({ onClicked }: ChildProps) {
  return (
    <button onClick={() => onClicked('child was clicked')}>
      Click
    </button>
  );
}

// Parent.tsx
function Parent() {
  const handleChildClick = (message: string) => {
    console.log(message);
  };

  return <Child onClicked={handleChildClick} />;
}
```

### Children Props (like Angular `<ng-content>`)

**Angular (content projection):**
```html
<!-- card.component.html -->
<div class="card">
  <ng-content></ng-content>
</div>

<!-- usage -->
<app-card>
  <p>This goes inside the card</p>
</app-card>
```

**React:**
```tsx
// Card.tsx
function Card({ children }: { children: React.ReactNode }) {
  return <div className="card">{children}</div>;
}

// Usage
function App() {
  return (
    <Card>
      <p>This goes inside the card</p>
    </Card>
  );
}
```

---

## State with useState

In Angular, state is just class properties. In React, state must be managed with `useState` (or other hooks) because React needs to know WHEN to re-render.

### Why Can't We Just Use Variables?

```tsx
// THIS DOES NOT WORK
function Counter() {
  let count = 0; // React does NOT watch this

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => count++}>Increment</button>
      {/* count changes but UI does NOT update! */}
    </div>
  );
}
```

### useState -- The Correct Way

```tsx
import { useState } from 'react';

function Counter() {
  // Destructure: [currentValue, setterFunction]
  const [count, setCount] = useState(0); // 0 is initial value

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>Increment</button>
      <button onClick={() => setCount(prev => prev + 1)}>
        Increment (functional update -- PREFERRED)
      </button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

### Angular vs React State Comparison

**Angular:**
```typescript
@Component({
  selector: 'app-counter',
  template: `
    <p>{{ count }}</p>
    <button (click)="increment()">Increment</button>
  `
})
export class CounterComponent {
  count = 0;  // Just a class property

  increment() {
    this.count++;  // Angular's Zone.js detects the change
  }
}
```

**React:**
```tsx
function Counter() {
  const [count, setCount] = useState(0);

  const increment = () => {
    setCount(prev => prev + 1); // Must use setter to trigger re-render
  };

  return (
    <div>
      <p>{count}</p>
      <button onClick={increment}>Increment</button>
    </div>
  );
}
```

### State with Objects and Arrays

```tsx
function UserForm() {
  // Object state
  const [user, setUser] = useState({ name: '', email: '' });

  // WRONG: mutating state directly
  // user.name = 'Alice'; // NEVER do this

  // RIGHT: create new object (immutability!)
  const updateName = (name: string) => {
    setUser(prev => ({ ...prev, name })); // spread + override
  };

  // Array state
  const [items, setItems] = useState<string[]>([]);

  const addItem = (item: string) => {
    setItems(prev => [...prev, item]); // spread + append
  };

  const removeItem = (index: number) => {
    setItems(prev => prev.filter((_, i) => i !== index));
  };

  return (
    <div>
      <input
        value={user.name}
        onChange={(e) => updateName(e.target.value)}
      />
    </div>
  );
}
```

**Critical Rule:** React state must be treated as IMMUTABLE. Never mutate state directly. Always create new objects/arrays. This is similar to NgRx's immutability pattern but applies to ALL state in React.

---

## useEffect -- Lifecycle Hooks Reimagined

`useEffect` replaces most Angular lifecycle hooks. It runs side effects after render.

### Angular Lifecycle Hooks vs useEffect

```
Angular Lifecycle:              React useEffect Equivalents:
+---------------------+        +----------------------------------+
| constructor()       |  --->  | Function body (runs every render)|
| ngOnInit()          |  --->  | useEffect(() => {}, [])          |
| ngOnChanges()       |  --->  | useEffect(() => {}, [dep])       |
| ngDoCheck()         |  --->  | No direct equivalent             |
| ngOnDestroy()       |  --->  | useEffect cleanup function       |
| ngAfterViewInit()   |  --->  | useEffect(() => {}, []) + ref    |
+---------------------+        +----------------------------------+
```

### useEffect Syntax

```tsx
import { useState, useEffect } from 'react';

function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState(null);

  // 1. Run on EVERY render (like ngDoCheck -- rarely needed)
  useEffect(() => {
    console.log('Component rendered');
  });

  // 2. Run ONCE on mount (like ngOnInit)
  useEffect(() => {
    console.log('Component mounted');
  }, []); // Empty dependency array = run once

  // 3. Run when dependency changes (like ngOnChanges for specific input)
  useEffect(() => {
    console.log('userId changed, fetching user...');
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => setUser(data));
  }, [userId]); // Runs when userId changes

  // 4. Cleanup on unmount (like ngOnDestroy)
  useEffect(() => {
    const subscription = someObservable.subscribe();

    return () => {
      // This cleanup runs on unmount (like ngOnDestroy)
      subscription.unsubscribe();
    };
  }, []);

  // 5. Cleanup + re-run (like ngOnChanges + ngOnDestroy combined)
  useEffect(() => {
    const ws = new WebSocket(`/ws/user/${userId}`);

    return () => {
      ws.close(); // Cleanup BEFORE re-running with new userId
    };
  }, [userId]);

  return <div>{user ? user.name : 'Loading...'}</div>;
}
```

### Common useEffect Patterns

```tsx
// Fetching data (compare with Angular's HttpClient in ngOnInit)
useEffect(() => {
  let cancelled = false; // prevent state update after unmount

  async function fetchData() {
    const res = await fetch('/api/data');
    const data = await res.json();
    if (!cancelled) {
      setData(data);
    }
  }

  fetchData();
  return () => { cancelled = true; };
}, []);

// Event listener (compare with Angular's @HostListener)
useEffect(() => {
  const handleResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', handleResize);
  return () => window.removeEventListener('resize', handleResize);
}, []);

// Document title (compare with Angular Title service)
useEffect(() => {
  document.title = `${count} items`;
}, [count]);
```

### Dependency Array Rules

```
useEffect(() => { ... });           // Runs after EVERY render
useEffect(() => { ... }, []);       // Runs ONCE (mount only)
useEffect(() => { ... }, [a, b]);   // Runs when a OR b changes

RULES:
- Include ALL values from component scope used inside the effect
- Functions, objects created inside render are NEW every render
  (wrap in useCallback/useMemo or move inside effect)
- ESLint plugin react-hooks/exhaustive-deps catches mistakes
```

---

## useContext -- Angular Services/DI Equivalent

In Angular, you inject services via the constructor. In React, you use Context to share data without prop drilling.

### Angular Service vs React Context

**Angular (service injected everywhere):**
```typescript
@Injectable({ providedIn: 'root' })
export class ThemeService {
  private theme = 'light';
  getTheme() { return this.theme; }
  setTheme(t: string) { this.theme = t; }
}

@Component({ ... })
export class SomeComponent {
  constructor(private themeService: ThemeService) {}
  get theme() { return this.themeService.getTheme(); }
}
```

**React (Context + Provider):**
```tsx
import { createContext, useContext, useState, ReactNode } from 'react';

// 1. Create context (like defining the service)
interface ThemeContextType {
  theme: string;
  setTheme: (t: string) => void;
}

const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

// 2. Create provider component (like providedIn: 'root')
function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState('light');

  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

// 3. Custom hook for convenience (like injecting the service)
function useTheme() {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within ThemeProvider');
  }
  return context;
}

// 4. Use in any component (like constructor injection)
function SomeComponent() {
  const { theme, setTheme } = useTheme();
  return <p>Current theme: {theme}</p>;
}

// 5. Wrap app with provider
function App() {
  return (
    <ThemeProvider>
      <SomeComponent />
    </ThemeProvider>
  );
}
```

### When to Use Context vs Props

```
Use Props when:                    Use Context when:
- Data flows 1-2 levels down       - Data needed by MANY components
- Parent directly renders child     - "Global" data (theme, auth, locale)
- Data is specific to child         - Avoids "prop drilling"

Prop Drilling Problem:
  App
   |-- passes theme -->  Layout
                           |-- passes theme -->  Sidebar
                                                   |-- passes theme -->  Button
                                                                          (finally uses it!)

Context Solution:
  <ThemeProvider>          <-- provides theme
    <App>
      <Layout>
        <Sidebar>
          <Button />       <-- useTheme() -- gets it directly
        </Sidebar>
      </Layout>
    </App>
  </ThemeProvider>
```

---

## useRef, useMemo, useCallback

### useRef -- Like Angular's @ViewChild

`useRef` holds a mutable value that does NOT cause re-renders when changed.

```tsx
import { useRef, useEffect } from 'react';

function TextInput() {
  // Like @ViewChild('inputRef') inputRef: ElementRef
  const inputRef = useRef<HTMLInputElement>(null);

  useEffect(() => {
    // Focus input on mount (like ngAfterViewInit)
    inputRef.current?.focus();
  }, []);

  return <input ref={inputRef} />;
}

// Also used to persist values across renders without causing re-renders
function Timer() {
  const intervalIdRef = useRef<number | null>(null);

  const start = () => {
    intervalIdRef.current = window.setInterval(() => {
      console.log('tick');
    }, 1000);
  };

  const stop = () => {
    if (intervalIdRef.current) {
      clearInterval(intervalIdRef.current);
    }
  };

  return (
    <div>
      <button onClick={start}>Start</button>
      <button onClick={stop}>Stop</button>
    </div>
  );
}
```

### useMemo -- Like Angular's Pure Pipe

`useMemo` caches a computed value and only recomputes when dependencies change.

```tsx
import { useMemo, useState } from 'react';

function ExpensiveList({ items, filter }: { items: Item[]; filter: string }) {
  // Only recomputes when items or filter change
  // Like an Angular pure pipe that caches output
  const filteredItems = useMemo(() => {
    console.log('Filtering... (expensive)');
    return items.filter(item =>
      item.name.toLowerCase().includes(filter.toLowerCase())
    );
  }, [items, filter]);

  return (
    <ul>
      {filteredItems.map(item => <li key={item.id}>{item.name}</li>)}
    </ul>
  );
}
```

### useCallback -- Memoize Functions

`useCallback` caches a function reference. Useful to prevent unnecessary re-renders of child components.

```tsx
import { useCallback, useState, memo } from 'react';

// memo = like ChangeDetectionStrategy.OnPush in Angular
const ExpensiveChild = memo(function ExpensiveChild({
  onClick
}: {
  onClick: () => void;
}) {
  console.log('ExpensiveChild rendered');
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  // Without useCallback: new function every render
  // -> ExpensiveChild re-renders every time Parent re-renders
  // const handleClick = () => console.log('clicked');

  // With useCallback: same function reference across renders
  // -> ExpensiveChild skips re-render (like OnPush)
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []); // [] = never changes

  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <ExpensiveChild onClick={handleClick} />
    </div>
  );
}
```

### Quick Reference

```
+------------------+-----------------------------------+------------------------+
| Hook             | Purpose                           | Angular Equivalent     |
+------------------+-----------------------------------+------------------------+
| useRef           | DOM access, mutable value         | @ViewChild, class prop |
| useMemo          | Cache computed values             | Pure pipe              |
| useCallback      | Cache function references         | (no direct equivalent) |
| React.memo       | Skip re-render if props unchanged | OnPush change detection|
+------------------+-----------------------------------+------------------------+
```

---

## Custom Hooks -- Angular Services Reimagined

Custom hooks extract reusable stateful logic. They are the React equivalent of Angular services but with a twist: each component using a custom hook gets its OWN instance of the state.

### Angular Service vs Custom Hook

**Angular Service (shared singleton):**
```typescript
@Injectable({ providedIn: 'root' })
export class WindowSizeService {
  width$ = new BehaviorSubject(window.innerWidth);

  constructor() {
    window.addEventListener('resize', () => {
      this.width$.next(window.innerWidth);
    });
  }
}
```

**React Custom Hook (per-component instance):**
```tsx
// hooks/useWindowSize.ts
import { useState, useEffect } from 'react';

function useWindowSize() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, []);

  return width;
}

// Usage in ANY component
function Sidebar() {
  const width = useWindowSize(); // Each component gets its own state
  return <div>{width > 768 ? 'Desktop' : 'Mobile'}</div>;
}
```

### Common Custom Hooks

```tsx
// useLocalStorage -- persist state (like Angular service with localStorage)
function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    const stored = localStorage.getItem(key);
    return stored ? JSON.parse(stored) : initialValue;
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue] as const;
}

// Usage:
const [theme, setTheme] = useLocalStorage('theme', 'light');
```

```tsx
// useFetch -- data fetching (like Angular HttpClient wrapper)
function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;
    setLoading(true);

    fetch(url)
      .then(res => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      })
      .then(data => {
        if (!cancelled) {
          setData(data);
          setLoading(false);
        }
      })
      .catch(err => {
        if (!cancelled) {
          setError(err.message);
          setLoading(false);
        }
      });

    return () => { cancelled = true; };
  }, [url]);

  return { data, loading, error };
}

// Usage:
function UserList() {
  const { data: users, loading, error } = useFetch<User[]>('/api/users');

  if (loading) return <p>Loading...</p>;
  if (error) return <p>Error: {error}</p>;
  return <ul>{users?.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

### Rules of Hooks

```
+---------------------------------------------------------------+
|                     RULES OF HOOKS                            |
+---------------------------------------------------------------+
| 1. Only call hooks at the TOP LEVEL                           |
|    - NOT inside loops, conditions, or nested functions         |
|                                                               |
| 2. Only call hooks from React functions                       |
|    - React functional components                              |
|    - Custom hooks (functions starting with "use")             |
|                                                               |
| BAD:                                                          |
|   if (loggedIn) {                                             |
|     const [user, setUser] = useState(null); // WRONG!         |
|   }                                                           |
|                                                               |
| GOOD:                                                         |
|   const [user, setUser] = useState(null);                     |
|   // use conditional logic with the state, not around it      |
+---------------------------------------------------------------+
```

---

## Event Handling Differences

| Feature | Angular | React |
|---|---|---|
| Click | `(click)="fn()"` | `onClick={fn}` |
| Input | `(input)="fn($event)"` | `onChange={fn}` |
| Submit | `(ngSubmit)="fn()"` | `onSubmit={fn}` |
| Key press | `(keydown.enter)="fn()"` | `onKeyDown={fn}` (check `e.key`) |
| Event object | `$event` | First parameter of handler |
| Prevent default | `$event.preventDefault()` | `e.preventDefault()` |
| Two-way binding | `[(ngModel)]="value"` | `value={val} onChange={...}` |

### Controlled Inputs (React's version of ngModel)

```tsx
function SearchForm() {
  const [query, setQuery] = useState('');

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault(); // Like $event.preventDefault()
    console.log('Searching for:', query);
  };

  return (
    <form onSubmit={handleSubmit}>
      {/* This is a "controlled input" -- React controls the value */}
      <input
        type="text"
        value={query}                           // Bind value (like [ngModel])
        onChange={(e) => setQuery(e.target.value)} // Update on change
        placeholder="Search..."
      />
      <button type="submit">Search</button>
    </form>
  );
}
```

---

## Conditional Rendering

### Angular *ngIf vs React

**Angular:**
```html
<div *ngIf="isLoggedIn; else loginTemplate">
  Welcome, {{ user.name }}
</div>
<ng-template #loginTemplate>
  <p>Please log in</p>
</ng-template>
```

**React (multiple patterns):**
```tsx
function AuthStatus({ isLoggedIn, user }) {
  // Pattern 1: Ternary (most common, like *ngIf; else)
  return (
    <div>
      {isLoggedIn ? (
        <p>Welcome, {user.name}</p>
      ) : (
        <p>Please log in</p>
      )}
    </div>
  );

  // Pattern 2: Logical AND (like *ngIf without else)
  return (
    <div>
      {isLoggedIn && <p>Welcome, {user.name}</p>}
    </div>
  );

  // Pattern 3: Early return (for whole component)
  if (!isLoggedIn) {
    return <p>Please log in</p>;
  }
  return <p>Welcome, {user.name}</p>;
}
```

### Angular *ngSwitch vs React

**Angular:**
```html
<div [ngSwitch]="status">
  <p *ngSwitchCase="'loading'">Loading...</p>
  <p *ngSwitchCase="'error'">Error!</p>
  <p *ngSwitchDefault>Content here</p>
</div>
```

**React:**
```tsx
function StatusDisplay({ status }: { status: string }) {
  // Object map pattern (cleaner than switch)
  const statusViews: Record<string, JSX.Element> = {
    loading: <p>Loading...</p>,
    error: <p>Error!</p>,
  };

  return statusViews[status] ?? <p>Content here</p>;
}
```

---

## Lists and Keys

### Angular *ngFor vs React map()

**Angular:**
```html
<ul>
  <li *ngFor="let user of users; let i = index; trackBy: trackById">
    {{ i + 1 }}. {{ user.name }}
  </li>
</ul>
```
```typescript
trackById(index: number, user: User) {
  return user.id;
}
```

**React:**
```tsx
function UserList({ users }: { users: User[] }) {
  return (
    <ul>
      {users.map((user, index) => (
        // key = like trackBy -- helps React identify which items changed
        <li key={user.id}>
          {index + 1}. {user.name}
        </li>
      ))}
    </ul>
  );
}
```

### Key Rules

```
+---------------------------------------------------------------+
|                     KEY RULES                                 |
+---------------------------------------------------------------+
| 1. Keys must be UNIQUE among siblings                         |
| 2. Keys must be STABLE (don't use Math.random())              |
| 3. Use the item's ID from data, NOT the array index           |
|    (index is OK only for static lists that never reorder)     |
| 4. Keys help React minimize DOM updates (like trackBy)        |
+---------------------------------------------------------------+

Why NOT index as key?
  Before: [A(0), B(1), C(2)]
  After:  [X(0), A(1), B(2), C(3)]   <-- inserted X at start
  React thinks item 0 changed from A to X (wrong!)
  With ID keys, React knows X is new and A,B,C just shifted
```

---

## Component Tree and Data Flow

```
                         React Component Tree
                         ====================

                           +----------+
                           |   App    |  <-- Top-level component
                           +----+-----+
                                |
               +----------------+----------------+
               |                                 |
         +-----+------+                   +------+------+
         |   Header   |                   |    Main     |
         |            |                   |             |
         | props:     |                   | state:      |
         | - user     |                   | - products  |
         | - onLogout |                   | - filter    |
         +-----+------+                   +------+------+
               |                                 |
        +------+------+              +-----------+-----------+
        |             |              |                       |
   +----+----+  +-----+---+   +-----+------+         +-----+------+
   |  Logo   |  | NavMenu |   | FilterBar  |         | ProductList|
   |         |  |         |   |            |         |            |
   | (leaf)  |  | props:  |   | props:     |         | props:     |
   +---------+  | - items |   | - filter   |         | - products |
                | - user  |   | - onChange  |         | - onAdd    |
                +---------+   +------------+         +------+-----+
                                                            |
                                                     +------+------+
                                                     | ProductCard |
                                                     |             |
                                                     | props:      |
                                                     | - product   |
                                                     | - onAdd     |
                                                     +-------------+

    DATA FLOWS DOWN (via props) ------->  vvvvv
    EVENTS FLOW UP (via callback props) ---->  ^^^^^

    Key Principles:
    1. Data flows ONE WAY: parent -> child (via props)
    2. Child communicates UP via callback functions passed as props
    3. State lives in the LOWEST COMMON ANCESTOR that needs it
    4. If many components need same data, LIFT STATE UP
       or use Context
```

---

## Interview Questions

### Q1: What is JSX and how does it differ from HTML?

**A:** JSX (JavaScript XML) is a syntax extension that lets you write HTML-like code inside JavaScript. It is NOT HTML -- it gets compiled to `React.createElement()` calls by Babel/SWC. Key differences from HTML: uses `className` instead of `class`, `htmlFor` instead of `for`, uses camelCase for attributes (`onClick`, `onChange`), requires all tags to be closed, expressions go inside `{}`, and inline styles use objects with camelCase properties (`style={{ backgroundColor: 'red' }}`). Unlike Angular's separate template files, JSX lives directly in the component function.

### Q2: What are the rules of hooks?

**A:** Two rules: (1) Only call hooks at the top level of a function component or custom hook -- never inside conditions, loops, or nested functions. This is because React relies on the ORDER of hook calls to maintain state between renders. (2) Only call hooks from React functional components or custom hooks (functions prefixed with `use`). These rules ensure React can correctly track which state belongs to which `useState`/`useEffect` call across re-renders.

### Q3: Explain the difference between useState and useRef.

**A:** `useState` stores state that, when updated via its setter, triggers a re-render of the component. `useRef` stores a mutable value in `.current` that persists across renders but does NOT trigger re-renders when changed. Use `useState` for data that should be reflected in the UI. Use `useRef` for DOM references (like Angular's `@ViewChild`), storing interval/timeout IDs, or tracking previous values without causing re-renders.

### Q4: What is the virtual DOM and how does it work?

**A:** The virtual DOM is a lightweight JavaScript representation of the actual DOM. When state changes, React creates a new virtual DOM tree, diffs it against the previous one (reconciliation), and applies only the minimal set of actual DOM changes needed. This is different from Angular's change detection (Zone.js + dirty checking). The virtual DOM makes React's declarative model efficient -- you describe WHAT the UI should look like, and React figures out HOW to update the real DOM.

### Q5: How does React's one-way data flow differ from Angular's two-way binding?

**A:** Angular supports two-way binding with `[(ngModel)]` where changes in the UI automatically update the model and vice versa. React enforces one-way data flow: data goes down via props, events go up via callbacks. For form inputs, React uses "controlled components" where you explicitly set `value` and handle `onChange`. This makes data flow more predictable and debugging easier, though it requires more boilerplate. In React: `<input value={name} onChange={(e) => setName(e.target.value)} />` vs Angular: `<input [(ngModel)]="name">`.

### Q6: What is useEffect's dependency array and what happens if you get it wrong?

**A:** The dependency array tells React when to re-run the effect. `[]` means run once on mount. `[a, b]` means re-run when `a` or `b` change. Omitting the array runs on every render. Common mistakes: (1) Missing dependencies cause stale closures -- the effect references outdated values. (2) Including objects/arrays/functions creates infinite loops because they're new references every render (use `useMemo`/`useCallback`). (3) Over-fetching by not including all dependencies. The `react-hooks/exhaustive-deps` ESLint rule catches these issues.

### Q7: When should you use useMemo and useCallback?

**A:** `useMemo` memoizes a computed value -- use it when a calculation is expensive and you want to avoid recomputing on every render. `useCallback` memoizes a function reference -- use it when passing callbacks to memoized child components (`React.memo`) to prevent unnecessary re-renders. Do NOT overuse them -- they have their own overhead (storing and comparing previous values). Use them when: (1) you have measurable performance issues, (2) a child wrapped in `React.memo` receives the value/function, or (3) the value is a dependency of another hook.

### Q8: How do custom hooks compare to Angular services?

**A:** Both encapsulate reusable logic. Key differences: (1) Angular services are singletons by default (shared state) -- custom hooks create independent state per component. (2) Angular uses DI to inject services -- React hooks are just imported functions. (3) Angular services can hold persistent state -- custom hooks' state is tied to component lifecycle. To share state across components like an Angular service, combine a custom hook with Context. Custom hooks must follow the "use" prefix convention (e.g., `useAuth`, `useFetch`).

### Q9: What is React.memo and how does it compare to Angular's OnPush?

**A:** `React.memo` is a higher-order component that skips re-rendering if props haven't changed (shallow comparison). It's similar to Angular's `ChangeDetectionStrategy.OnPush` which only re-renders when `@Input` references change. Usage: `const MyComponent = memo(function MyComponent(props) { ... })`. Important: it uses shallow comparison, so objects/arrays/functions that are recreated each render will still cause re-renders unless you also use `useMemo`/`useCallback` in the parent.

### Q10: Explain the concept of "lifting state up" in React.

**A:** When two sibling components need to share state, the state should be moved (lifted) to their closest common ancestor. The ancestor holds the state and passes it down as props to both children, along with callback functions to update it. This is React's answer to Angular's shared service pattern. For example, if a `SearchBar` and `ResultList` both need search results, the parent component holds the state and passes filtered results down. For deeply shared state, Context or external state managers (Zustand, Redux) are better alternatives.

### Q11: What happens when you call a state setter multiple times in the same event handler?

**A:** React batches state updates within event handlers and lifecycle methods. Multiple `setState` calls in the same synchronous handler result in a single re-render. However, if you call `setCount(count + 1)` three times, you get only +1 because each call uses the same stale `count` value. Use the functional form `setCount(prev => prev + 1)` to get +3, as each call receives the latest pending state. Since React 18, batching also applies to async handlers (promises, setTimeout).

### Q12: What is the difference between controlled and uncontrolled components?

**A:** Controlled components have their form values managed by React state (`value` + `onChange`). The React state is the "single source of truth." Uncontrolled components let the DOM handle form state, and you read values via `useRef` when needed. Controlled is preferred because it enables instant validation, conditional disabling, and enforced formatting. Uncontrolled is simpler for basic forms or when integrating non-React libraries. In Angular terms, controlled is like Reactive Forms (form state in code) and uncontrolled is like Template-driven Forms (DOM manages state).

### Q13: How do you handle errors in React components?

**A:** React uses Error Boundaries -- class components with `componentDidCatch` and `static getDerivedStateFromError` methods. They catch errors during rendering, in lifecycle methods, and in constructors of child components. They do NOT catch errors in event handlers (use try/catch), async code, or server-side rendering. There is no hook equivalent yet, so error boundaries must be class components. Libraries like `react-error-boundary` provide a convenient wrapper. In Angular, you use `ErrorHandler` for global errors and the `catchError` RxJS operator.

### Q14: What is the difference between React elements, components, and instances?

**A:** A React **element** is a plain object describing what to render: `{ type: 'div', props: { children: 'Hello' } }`. JSX like `<div>Hello</div>` creates elements. A **component** is a function (or class) that accepts props and returns elements. `function Button(props) { return <button>{props.label}</button>; }` is a component. An **instance** is what React creates internally to track a component in the tree -- you don't create instances directly. When you write `<Button label="Click" />`, you're creating an element whose type is the Button component.

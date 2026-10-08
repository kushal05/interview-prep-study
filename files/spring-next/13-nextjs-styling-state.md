# Next.js Styling and State Management

## Table of Contents
1. [CSS Modules](#css-modules)
2. [Tailwind CSS in Next.js](#tailwind-css-in-nextjs)
3. [CSS-in-JS / styled-components](#css-in-js--styled-components)
4. [Global State Management](#global-state-management)
5. [Form Handling](#form-handling)
6. [Interview Questions](#interview-questions)

---

## CSS Modules

CSS Modules scope CSS to a single component -- class names are automatically made unique. This is the closest to Angular's component-scoped styles.

### Angular Component Styles vs CSS Modules

**Angular:**
```typescript
@Component({
  selector: 'app-card',
  template: `<div class="card"><h2 class="title">{{ title }}</h2></div>`,
  styles: [`
    .card { border: 1px solid #ccc; padding: 16px; }
    .title { color: blue; }
  `]
  // encapsulation: ViewEncapsulation.Emulated (default -- scoped)
})
export class CardComponent {
  @Input() title = '';
}
```

**Next.js with CSS Modules:**
```css
/* components/Card.module.css */
.card {
  border: 1px solid #ccc;
  padding: 16px;
}

.title {
  color: blue;
}

/* Compiled output: .Card_card__x7f2k, .Card_title__a3b1c */
/* Unique per component -- no class name collisions */
```

```tsx
// components/Card.tsx
import styles from './Card.module.css';

export default function Card({ title }: { title: string }) {
  return (
    <div className={styles.card}>
      <h2 className={styles.title}>{title}</h2>
    </div>
  );
}
```

### CSS Modules Features

```tsx
// Composition (like Sass @extend)
/* base.module.css */
.btn {
  padding: 8px 16px;
  border-radius: 4px;
}

/* Button.module.css */
.primary {
  composes: btn from './base.module.css';
  background: blue;
  color: white;
}

// Dynamic class names
import styles from './Alert.module.css';
import clsx from 'clsx'; // or classnames library

function Alert({ type, message }: { type: 'error' | 'success'; message: string }) {
  return (
    <div className={clsx(styles.alert, {
      [styles.error]: type === 'error',
      [styles.success]: type === 'success',
    })}>
      {message}
    </div>
  );
}
```

### Comparison Table

| Feature | Angular Styles | CSS Modules |
|---|---|---|
| Scoping | `ViewEncapsulation.Emulated` (attribute selector) | Unique class names (hash) |
| File location | Same folder or inline | Same folder, `.module.css` extension |
| Global styles | `styles.css` in angular.json | `globals.css` imported in layout |
| Sass/SCSS | Built-in support | Built-in support (`.module.scss`) |
| Dynamic classes | `[ngClass]` directive | Template literals or `clsx` library |
| `:host` selector | Yes | No equivalent (style the root element) |
| `::ng-deep` | Deprecated but exists | Not needed (scoped by default) |

---

## Tailwind CSS in Next.js

Tailwind is utility-first CSS and is the most popular styling choice for Next.js. Next.js has first-class Tailwind support.

### Setup (Built into create-next-app)

```
npx create-next-app@latest my-app
# Select "Yes" when asked about Tailwind CSS
# That's it -- zero configuration needed
```

### Basic Usage

```tsx
// Angular equivalent:
// <div class="container" style="display: flex; gap: 16px; padding: 24px;">
//   <div class="card" style="border: 1px solid #ccc; border-radius: 8px; padding: 16px;">
//     <h2 style="font-size: 1.5rem; font-weight: bold; color: #1a1a1a;">Title</h2>
//     <p style="color: #666;">Description</p>
//   </div>
// </div>

// Next.js with Tailwind:
function Card({ title, description }: { title: string; description: string }) {
  return (
    <div className="flex gap-4 p-6">
      <div className="border border-gray-300 rounded-lg p-4">
        <h2 className="text-2xl font-bold text-gray-900">{title}</h2>
        <p className="text-gray-600">{description}</p>
      </div>
    </div>
  );
}
```

### Responsive Design

```tsx
// Angular: @media queries in component CSS
// Tailwind: prefix classes with breakpoint

function ResponsiveGrid() {
  return (
    <div className="
      grid
      grid-cols-1       /* Mobile: 1 column */
      md:grid-cols-2    /* Tablet (768px+): 2 columns */
      lg:grid-cols-3    /* Desktop (1024px+): 3 columns */
      gap-4
    ">
      <Card />
      <Card />
      <Card />
    </div>
  );
}
```

### Tailwind Cheat Sheet for Angular Developers

```
Angular CSS Property        Tailwind Class
----------------------------------------------
display: flex               flex
flex-direction: column      flex-col
justify-content: center     justify-center
align-items: center         items-center
gap: 16px                   gap-4  (4 = 1rem = 16px)
padding: 16px               p-4
margin: 8px                 m-2
margin-top: 16px            mt-4
border-radius: 8px          rounded-lg
font-size: 1.5rem           text-2xl
font-weight: bold           font-bold
color: blue                 text-blue-500
background: #f3f4f6         bg-gray-100
width: 100%                 w-full
max-width: 768px            max-w-3xl
position: fixed             fixed
display: none               hidden
display: block              block
cursor: pointer             cursor-pointer
transition                  transition-all duration-300
hover: background           hover:bg-blue-600
focus: ring                 focus:ring-2 focus:ring-blue-500
disabled: opacity           disabled:opacity-50
```

### Conditional Styling

```tsx
// Angular: [ngClass]="{ 'active': isActive, 'disabled': isDisabled }"

// React with Tailwind + clsx:
import clsx from 'clsx';

function Button({
  variant = 'primary',
  disabled = false,
  children,
}: {
  variant?: 'primary' | 'secondary' | 'danger';
  disabled?: boolean;
  children: React.ReactNode;
}) {
  return (
    <button
      disabled={disabled}
      className={clsx(
        // Base styles (always applied)
        'px-4 py-2 rounded-lg font-medium transition-colors',
        // Variant styles
        {
          'bg-blue-600 text-white hover:bg-blue-700': variant === 'primary',
          'bg-gray-200 text-gray-800 hover:bg-gray-300': variant === 'secondary',
          'bg-red-600 text-white hover:bg-red-700': variant === 'danger',
        },
        // Disabled state
        disabled && 'opacity-50 cursor-not-allowed',
      )}
    >
      {children}
    </button>
  );
}
```

---

## CSS-in-JS / styled-components

CSS-in-JS libraries let you write CSS in JavaScript. They are less common in Next.js (Tailwind and CSS Modules are preferred) but you should know them for interviews.

### styled-components

```tsx
'use client'; // CSS-in-JS requires client components

import styled from 'styled-components';

const StyledCard = styled.div`
  border: 1px solid #ccc;
  border-radius: 8px;
  padding: 16px;
  background: ${(props) => props.$variant === 'dark' ? '#1a1a1a' : '#fff'};
`;

const Title = styled.h2`
  font-size: 1.5rem;
  color: ${(props) => props.theme.colors.primary};
`;

function Card({ title }: { title: string }) {
  return (
    <StyledCard $variant="light">
      <Title>{title}</Title>
    </StyledCard>
  );
}
```

### Important: Server Component Compatibility

```
+------------------+-------------------+-------------------+
| Styling Method   | Server Components | Client Components |
+------------------+-------------------+-------------------+
| CSS Modules      | YES               | YES               |
| Tailwind CSS     | YES               | YES               |
| Global CSS       | YES               | YES               |
| Inline styles    | YES               | YES               |
| styled-components| NO (needs 'use    | YES               |
|                  |  client')         |                   |
| Emotion          | NO                | YES               |
+------------------+-------------------+-------------------+

Recommendation: Use Tailwind or CSS Modules for Server Components.
               They work everywhere and have zero runtime overhead.
```

---

## Global State Management

### The State Management Landscape

```
                    State Management Options in Next.js
                    ====================================

  Simple/Local                                      Complex/Global
  <------------------------------------------------------>

  useState    useReducer    Context API    Zustand    Redux Toolkit
  (per-comp)  (complex      (shared,       (simple    (enterprise,
              local state)  small apps)    global)    large apps)

  Angular Equivalents:
  Component   Component     Service with   Service    NgRx Store
  property    property      BehaviorSubject  singleton

  RECOMMENDATION FOR NEXT.JS:
  - Start with useState + Server Components (often enough!)
  - Add Context for theme/auth (small shared state)
  - Use Zustand for client-side global state (simple, popular)
  - Use Redux Toolkit only for very complex state needs
```

### Context API (Like Angular Services for Small State)

```tsx
// context/CartContext.tsx
'use client';

import { createContext, useContext, useReducer, ReactNode } from 'react';

// Types
interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartState {
  items: CartItem[];
  total: number;
}

type CartAction =
  | { type: 'ADD_ITEM'; payload: CartItem }
  | { type: 'REMOVE_ITEM'; payload: string }
  | { type: 'CLEAR_CART' };

// Reducer (like NgRx reducer)
function cartReducer(state: CartState, action: CartAction): CartState {
  switch (action.type) {
    case 'ADD_ITEM': {
      const existing = state.items.find(i => i.id === action.payload.id);
      if (existing) {
        return {
          ...state,
          items: state.items.map(i =>
            i.id === action.payload.id
              ? { ...i, quantity: i.quantity + 1 }
              : i
          ),
          total: state.total + action.payload.price,
        };
      }
      return {
        items: [...state.items, { ...action.payload, quantity: 1 }],
        total: state.total + action.payload.price,
      };
    }
    case 'REMOVE_ITEM':
      const item = state.items.find(i => i.id === action.payload);
      return {
        items: state.items.filter(i => i.id !== action.payload),
        total: state.total - (item ? item.price * item.quantity : 0),
      };
    case 'CLEAR_CART':
      return { items: [], total: 0 };
    default:
      return state;
  }
}

// Context
const CartContext = createContext<{
  state: CartState;
  dispatch: React.Dispatch<CartAction>;
} | null>(null);

// Provider
export function CartProvider({ children }: { children: ReactNode }) {
  const [state, dispatch] = useReducer(cartReducer, { items: [], total: 0 });

  return (
    <CartContext.Provider value={{ state, dispatch }}>
      {children}
    </CartContext.Provider>
  );
}

// Hook (like injecting the service)
export function useCart() {
  const context = useContext(CartContext);
  if (!context) throw new Error('useCart must be used within CartProvider');
  return context;
}
```

```tsx
// Usage in components
'use client';
import { useCart } from '@/context/CartContext';

function AddToCartButton({ product }: { product: Product }) {
  const { dispatch } = useCart();

  return (
    <button onClick={() => dispatch({
      type: 'ADD_ITEM',
      payload: { id: product.id, name: product.name, price: product.price, quantity: 1 },
    })}>
      Add to Cart
    </button>
  );
}

function CartSummary() {
  const { state } = useCart();

  return (
    <div>
      <p>Items: {state.items.length}</p>
      <p>Total: ${state.total.toFixed(2)}</p>
    </div>
  );
}
```

### Zustand (Recommended for Next.js)

Zustand is the most popular state management for Next.js. It's simple, fast, and works well with Server Components.

```tsx
// store/useCartStore.ts
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

interface CartStore {
  items: CartItem[];
  addItem: (item: Omit<CartItem, 'quantity'>) => void;
  removeItem: (id: string) => void;
  clearCart: () => void;
  total: () => number;
}

export const useCartStore = create<CartStore>()(
  persist(
    (set, get) => ({
      items: [],

      addItem: (item) => set((state) => {
        const existing = state.items.find(i => i.id === item.id);
        if (existing) {
          return {
            items: state.items.map(i =>
              i.id === item.id ? { ...i, quantity: i.quantity + 1 } : i
            ),
          };
        }
        return { items: [...state.items, { ...item, quantity: 1 }] };
      }),

      removeItem: (id) => set((state) => ({
        items: state.items.filter(i => i.id !== id),
      })),

      clearCart: () => set({ items: [] }),

      total: () => get().items.reduce(
        (sum, item) => sum + item.price * item.quantity, 0
      ),
    }),
    {
      name: 'cart-storage', // localStorage key
    }
  )
);
```

```tsx
// Usage -- no Provider needed!
'use client';
import { useCartStore } from '@/store/useCartStore';

function AddToCartButton({ product }: { product: Product }) {
  const addItem = useCartStore((state) => state.addItem);

  return (
    <button onClick={() => addItem({
      id: product.id,
      name: product.name,
      price: product.price,
    })}>
      Add to Cart
    </button>
  );
}

function CartSummary() {
  const items = useCartStore((state) => state.items);
  const total = useCartStore((state) => state.total);

  return (
    <div>
      <p>Items: {items.length}</p>
      <p>Total: ${total().toFixed(2)}</p>
    </div>
  );
}
```

### Comparison: NgRx vs Context+useReducer vs Zustand vs Redux Toolkit

```
+-------------------+------------------+------------------+------------------+
| Feature           | NgRx             | Context+Reducer  | Zustand          |
+-------------------+------------------+------------------+------------------+
| Boilerplate       | High             | Medium           | Low              |
| Bundle size       | Large            | 0 (built-in)    | ~1KB             |
| DevTools          | Yes              | React DevTools   | Yes              |
| Persistence       | Manual           | Manual           | Built-in         |
| Provider needed   | StoreModule      | Yes              | No               |
| Learning curve    | Steep            | Low              | Low              |
| Middleware        | Effects          | No               | Middleware       |
| Selectors         | createSelector   | Custom hooks     | Selector fn      |
| Async             | Effects/Thunks   | useEffect        | Built-in         |
| Server Components | N/A              | No (client only) | No (client only) |
| Best for          | Large Angular    | Small React apps | Most Next.js     |
+-------------------+------------------+------------------+------------------+
```

### State Flow Diagram

```
  Zustand Store (Global Client State)
  ====================================

  +------------------+       +-----------------------+
  | useCartStore     |       | Zustand Store         |
  | (hook)           |<----->| {                     |
  |                  |       |   items: [...],       |
  | Used in any      |       |   addItem: fn,        |
  | Client Component |       |   removeItem: fn,     |
  +------------------+       |   total: fn,          |
         |                   | }                     |
         |                   +----------+------------+
         |                              |
         v                              v
  +------+-------+              +-------+---------+
  | Component A  |              | localStorage    |
  | (re-renders  |              | (persist        |
  | on change)   |              |  middleware)     |
  +--------------+              +-----------------+

  Server State vs Client State in Next.js:
  =========================================

  +---------------------+           +---------------------+
  | Server State        |           | Client State        |
  | (Server Components) |           | (Client Components) |
  +---------------------+           +---------------------+
  | - Database data     |           | - UI state          |
  | - CMS content       |           | - Form inputs       |
  | - API responses     |           | - Cart items        |
  | - User sessions     |           | - Theme preference  |
  | - Environment vars  |           | - Modal open/close  |
  +---------------------+           +---------------------+
  | Managed by:         |           | Managed by:         |
  | - fetch + cache     |           | - useState          |
  | - Server Actions    |           | - Zustand           |
  | - revalidatePath    |           | - Context API       |
  +---------------------+           +---------------------+

  KEY INSIGHT: In Next.js, much of what you'd put in NgRx
  becomes SERVER state (fetched in Server Components).
  Client state is only for UI interactivity.
```

---

## Form Handling

### Simple Forms with Server Actions

```tsx
// app/contact/page.tsx (Server Component)
import { submitContact } from './actions';

export default function ContactPage() {
  return (
    <form action={submitContact}>
      <label htmlFor="name">Name</label>
      <input id="name" name="name" required />

      <label htmlFor="email">Email</label>
      <input id="email" name="email" type="email" required />

      <label htmlFor="message">Message</label>
      <textarea id="message" name="message" required />

      <button type="submit">Send</button>
    </form>
  );
}
```

```tsx
// app/contact/actions.ts
'use server';

import { z } from 'zod';
import { redirect } from 'next/navigation';

const ContactSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email'),
  message: z.string().min(10, 'Message must be at least 10 characters'),
});

export async function submitContact(formData: FormData) {
  const result = ContactSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
    message: formData.get('message'),
  });

  if (!result.success) {
    // Return errors -- handle in client
    return { errors: result.error.flatten().fieldErrors };
  }

  await db.contact.create({ data: result.data });
  redirect('/contact/thank-you');
}
```

### React Hook Form (Complex Forms)

React Hook Form is the React equivalent of Angular Reactive Forms.

```tsx
// components/RegistrationForm.tsx
'use client';

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const registrationSchema = z.object({
  username: z.string().min(3, 'Username must be at least 3 characters'),
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword'],
});

type RegistrationForm = z.infer<typeof registrationSchema>;

export default function RegistrationForm() {
  const {
    register,    // Like Angular formControlName
    handleSubmit, // Like Angular (ngSubmit)
    formState: { errors, isSubmitting }, // Like form.errors, form.pending
    reset,
  } = useForm<RegistrationForm>({
    resolver: zodResolver(registrationSchema),
    defaultValues: {
      username: '',
      email: '',
      password: '',
      confirmPassword: '',
    },
  });

  const onSubmit = async (data: RegistrationForm) => {
    const res = await fetch('/api/register', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    });

    if (res.ok) {
      reset();
      alert('Registration successful!');
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)} noValidate>
      <div>
        <label htmlFor="username">Username</label>
        <input id="username" {...register('username')} />
        {errors.username && (
          <p className="text-red-500 text-sm">{errors.username.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input id="email" type="email" {...register('email')} />
        {errors.email && (
          <p className="text-red-500 text-sm">{errors.email.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input id="password" type="password" {...register('password')} />
        {errors.password && (
          <p className="text-red-500 text-sm">{errors.password.message}</p>
        )}
      </div>

      <div>
        <label htmlFor="confirmPassword">Confirm Password</label>
        <input
          id="confirmPassword"
          type="password"
          {...register('confirmPassword')}
        />
        {errors.confirmPassword && (
          <p className="text-red-500 text-sm">{errors.confirmPassword.message}</p>
        )}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Registering...' : 'Register'}
      </button>
    </form>
  );
}
```

### Angular Reactive Forms vs React Hook Form

```
Angular Reactive Forms:                React Hook Form:
+--------------------------------+     +----------------------------------+
| FormGroup, FormControl,        |     | useForm() hook                   |
| FormArray                      |     |                                  |
|                                |     | register('fieldName')            |
| this.form = new FormGroup({   |     | = formControlName="fieldName"    |
|   email: new FormControl('',  |     |                                  |
|     [Validators.required,     |     | zodResolver(schema)              |
|      Validators.email])       |     | = Validators in schema           |
| });                           |     |                                  |
|                                |     | handleSubmit(onSubmit)           |
| (ngSubmit)="onSubmit()"      |     | = (ngSubmit)                     |
|                                |     |                                  |
| form.get('email')?.errors    |     | errors.email?.message            |
| form.valid                    |     | formState.isValid                |
| form.pending                  |     | formState.isSubmitting           |
+--------------------------------+     +----------------------------------+

Key Difference:
  Angular: Form state managed by Angular (FormControl instances)
  React:   Form state managed by DOM (uncontrolled by default)
           React Hook Form reads DOM values -- minimal re-renders
```

---

## Interview Questions

### Q1: What styling approaches are available in Next.js and which do you recommend?

**A:** Next.js supports CSS Modules (scoped CSS like Angular component styles), Tailwind CSS (utility-first, most popular), CSS-in-JS libraries (styled-components, Emotion), global CSS, Sass/SCSS, and inline styles. I recommend Tailwind CSS for most projects -- it works with Server Components (zero runtime), reduces CSS bundle size through purging, and integrates seamlessly with Next.js. CSS Modules are a good alternative for teams preferring traditional CSS. Avoid CSS-in-JS for Server Components as it requires `'use client'` and adds runtime overhead.

### Q2: How do CSS Modules work and how do they compare to Angular's style encapsulation?

**A:** CSS Modules automatically scope class names by appending a unique hash (e.g., `.card` becomes `.Card_card__x7f2k`). This prevents name collisions across components, similar to Angular's `ViewEncapsulation.Emulated` which uses attribute selectors (`_ngcontent-xxx`). Usage: create `Component.module.css`, import as `styles`, then use `className={styles.card}`. Differences: Angular scoping uses data attributes on DOM elements; CSS Modules transform class names at build time. Both achieve the same goal of component-scoped styles without global side effects.

### Q3: How does Tailwind CSS work with Next.js Server Components?

**A:** Tailwind CSS works perfectly with Server Components because it's processed at build time -- classes are compiled to plain CSS with no JavaScript runtime. The Next.js build process scans all files for Tailwind class names and generates optimized CSS. This contrasts with CSS-in-JS solutions (styled-components, Emotion) which require JavaScript runtime and therefore need `'use client'`. Tailwind's utility classes are also tree-shaken -- unused classes are removed from the production bundle, resulting in very small CSS files.

### Q4: When would you use Context API vs Zustand vs Redux Toolkit in Next.js?

**A:** **Context API:** For small, simple shared state (theme, locale, auth status). Built-in, no extra dependency, but can cause unnecessary re-renders in large trees. **Zustand:** Recommended for most Next.js apps. Minimal boilerplate, no Provider needed, built-in persistence, excellent performance with selector-based subscriptions, tiny bundle (~1KB). **Redux Toolkit:** For very complex state with many interrelated slices, middleware needs, or teams already experienced with Redux. More boilerplate but powerful with RTK Query, devtools, and ecosystem. In Next.js, much state that would be in Redux lives in Server Components, so client state is often simpler.

### Q5: How does state management in Next.js differ from Angular/NgRx?

**A:** The biggest difference: Next.js Server Components handle most "server state" (data from APIs/databases) without any client-side state management. In Angular, you fetch data client-side and often store it in NgRx. In Next.js, data is fetched server-side and rendered directly. Client state is only for interactivity (form inputs, UI toggles, cart items). This means you typically need less state management infrastructure. A Next.js app might use Zustand for a small cart store where an Angular app would need NgRx with effects, reducers, selectors, and entity adapters.

### Q6: How do you handle forms in Next.js? Compare with Angular Reactive Forms.

**A:** Two approaches: (1) **Server Actions** for simple forms -- attach a server function as the form's `action`. Works without JavaScript (progressive enhancement), validates server-side. Use `useActionState` for form state feedback. (2) **React Hook Form** for complex forms -- similar to Angular Reactive Forms but hooks-based. Uses Zod for validation schemas (replacing Angular Validators). Key difference: Angular Reactive Forms track every field change in real-time; React Hook Form uses uncontrolled inputs and only validates on submit by default, resulting in fewer re-renders.

### Q7: Explain the concept of server state vs client state in Next.js.

**A:** **Server state** is data from external sources (APIs, databases) -- managed by Server Components, fetch caching, and revalidation. It's the "source of truth" and never needs a client-side store. **Client state** is ephemeral UI state -- form inputs, toggles, modals, theme preference. It exists only in the browser. In Angular, ALL data flows through client-side state (services, NgRx). In Next.js, server state stays on the server, and client state is minimal. This separation simplifies architecture: you don't need complex client-side caching, normalization, or synchronization for server data.

### Q8: How do you prevent unnecessary re-renders when using Context API?

**A:** Context re-renders ALL consumers when the context value changes. Strategies: (1) **Split contexts** -- separate frequently-changing state (e.g., `CartItemsContext`) from stable state (e.g., `CartActionsContext`). (2) **Memoize the value** with `useMemo`. (3) **Use selectors** -- Zustand supports selecting specific state slices. (4) **React.memo** on child components. (5) **useReducer** instead of multiple `useState` to batch updates. For complex apps, Zustand is preferred because its selector pattern only triggers re-renders when the selected slice changes, unlike Context which re-renders on any change.

### Q9: How would you implement a theme toggle in Next.js?

**A:** Use Context + `next-themes` library (most common): Create a ThemeProvider with `next-themes`, wrap your layout, and use `useTheme()` hook in client components. Tailwind integration: configure `darkMode: 'class'` in `tailwind.config.ts`, then use `dark:` prefix for dark styles (`className="bg-white dark:bg-gray-900"`). For SSR, `next-themes` handles the flash-of-wrong-theme problem by injecting a script that sets the theme class before React hydrates. This is similar to Angular's approach of using a service with `Renderer2` to toggle classes.

### Q10: What is the `clsx` or `classnames` library and why is it used?

**A:** `clsx` (and the older `classnames`) is a utility for conditionally joining class names. Angular has `[ngClass]` built-in for conditional classes. React has no equivalent, so `clsx` fills that gap. Usage: `className={clsx('base', isActive && 'active', { 'disabled': isDisabled })}`. It handles strings, objects, arrays, and falsy values. It's especially useful with Tailwind CSS where you compose many utility classes conditionally. It's tiny (~300 bytes) and used in nearly every React/Next.js project.

### Q11: How do you set up Zustand with persistence in Next.js?

**A:** Use Zustand's `persist` middleware: `create()(persist((set) => ({ ... }), { name: 'store-key' }))`. This automatically saves state to `localStorage` and restores it on page load. For SSR compatibility, handle the hydration mismatch: the server doesn't have `localStorage`, so the initial server render won't have persisted state. Solutions: (1) Use `skipHydration` option and manually hydrate. (2) Use a client-only wrapper with `useEffect`. (3) Accept a brief flash and suppress hydration warnings for non-critical UI. Zustand's persist middleware also supports custom storage (sessionStorage, AsyncStorage for React Native).

### Q12: How does React Hook Form compare to Angular Reactive Forms in terms of performance?

**A:** React Hook Form is generally more performant for large forms. Angular Reactive Forms use a "push" model -- every keypress triggers change detection across the form tree. React Hook Form uses an "uncontrolled" approach -- inputs manage their own DOM state, and RHF only reads values during validation/submission. This means typing in one field doesn't re-render other fields. Angular's approach gives real-time access to all values but at the cost of more change detection cycles. For forms with 50+ fields, React Hook Form's approach results in noticeably smoother interaction.

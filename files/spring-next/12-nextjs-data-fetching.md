# Next.js Data Fetching Patterns

## Table of Contents
1. [Server-Side Data Fetching](#server-side-data-fetching)
2. [Caching and Revalidation](#caching-and-revalidation)
3. [Server Actions](#server-actions)
4. [Route Handlers (API Routes)](#route-handlers-api-routes)
5. [Client-Side Data Fetching](#client-side-data-fetching)
6. [Data Flow Diagram](#data-flow-diagram)
7. [Angular HttpClient vs Next.js Comparison](#angular-httpclient-vs-nextjs-comparison)
8. [Interview Questions](#interview-questions)

---

## Server-Side Data Fetching

In Next.js App Router, the primary way to fetch data is directly in Server Components using `fetch` or any async operation. No special functions needed -- just use `async/await`.

### Basic Pattern

```tsx
// app/products/page.tsx (Server Component -- default)
export default async function ProductsPage() {
  // This fetch runs on the SERVER, never on the client
  const res = await fetch('https://api.example.com/products');

  if (!res.ok) {
    throw new Error('Failed to fetch products');
  }

  const products: Product[] = await res.json();

  return (
    <ul>
      {products.map(product => (
        <li key={product.id}>
          {product.name} - ${product.price}
        </li>
      ))}
    </ul>
  );
}
```

### Direct Database Access (No API Needed!)

Unlike Angular (which always goes through HTTP), Server Components can access the database directly:

```tsx
// app/users/page.tsx
import { prisma } from '@/lib/prisma'; // Or any ORM

export default async function UsersPage() {
  // Direct database query -- no HTTP call, no API route needed
  const users = await prisma.user.findMany({
    where: { active: true },
    orderBy: { createdAt: 'desc' },
    take: 20,
  });

  return (
    <div>
      {users.map(user => (
        <div key={user.id}>
          <h3>{user.name}</h3>
          <p>{user.email}</p>
        </div>
      ))}
    </div>
  );
}
```

### Angular vs Next.js: Fetching Data

**Angular:**
```typescript
// user.service.ts
@Injectable({ providedIn: 'root' })
export class UserService {
  constructor(private http: HttpClient) {}

  getUsers(): Observable<User[]> {
    return this.http.get<User[]>('/api/users');
  }
}

// users.component.ts
@Component({
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngFor="let user of users$ | async">
      {{ user.name }}
    </div>
  `
})
export class UsersComponent implements OnInit {
  users$!: Observable<User[]>;
  loading = true;

  constructor(private userService: UserService) {}

  ngOnInit() {
    this.users$ = this.userService.getUsers().pipe(
      tap(() => this.loading = false)
    );
  }
}
```

**Next.js (same result, much simpler):**
```tsx
// app/users/page.tsx
async function getUsers(): Promise<User[]> {
  const res = await fetch('https://api.example.com/users');
  if (!res.ok) throw new Error('Failed to fetch');
  return res.json();
}

export default async function UsersPage() {
  const users = await getUsers();
  return (
    <div>
      {users.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
    </div>
  );
}
// No services, no DI, no Observables, no manual loading state
// loading.tsx handles loading UI automatically
```

### Parallel Data Fetching

```tsx
// WRONG: Sequential (waterfall) -- each awaits the previous
export default async function Dashboard() {
  const user = await getUser();      // 200ms
  const posts = await getPosts();    // 300ms
  const stats = await getStats();    // 150ms
  // Total: 650ms (sequential)

  return <DashboardView user={user} posts={posts} stats={stats} />;
}

// RIGHT: Parallel -- all fetches start simultaneously
export default async function Dashboard() {
  const [user, posts, stats] = await Promise.all([
    getUser(),      // 200ms
    getPosts(),     // 300ms
    getStats(),     // 150ms
  ]);
  // Total: 300ms (parallel -- limited by slowest)

  return <DashboardView user={user} posts={posts} stats={stats} />;
}

// BEST: Streaming -- each section loads independently
export default function Dashboard() {
  return (
    <div>
      <Suspense fallback={<UserSkeleton />}>
        <UserSection />   {/* Streams in after 200ms */}
      </Suspense>
      <Suspense fallback={<PostsSkeleton />}>
        <PostsSection />  {/* Streams in after 300ms */}
      </Suspense>
      <Suspense fallback={<StatsSkeleton />}>
        <StatsSection />  {/* Streams in after 150ms */}
      </Suspense>
    </div>
  );
}
```

### Request Memoization

Next.js automatically deduplicates identical `fetch` calls within the same render:

```tsx
// Both of these result in only ONE actual HTTP request!
// layout.tsx
async function Layout({ children }) {
  const user = await fetch('/api/user'); // Request 1
  return <div><Nav user={user} />{children}</div>;
}

// page.tsx (rendered inside Layout)
async function Page() {
  const user = await fetch('/api/user'); // Same URL = DEDUPLICATED
  return <div>Hello {user.name}</div>;
}
// Only 1 HTTP request is made, result is shared
```

---

## Caching and Revalidation

Next.js extends the native `fetch` API with caching options.

### Caching Behavior

```
fetch('https://api.example.com/data')
  // Default: cached indefinitely (like SSG)

fetch('https://api.example.com/data', { cache: 'no-store' })
  // No caching: fresh data every request (like SSR)

fetch('https://api.example.com/data', {
  next: { revalidate: 3600 }  // Cache for 1 hour, then revalidate (ISR)
})

fetch('https://api.example.com/data', {
  next: { tags: ['products'] }  // Tag for on-demand revalidation
})
```

### Caching Layers in Next.js

```
Request Flow Through Cache Layers:

  Client Request
       |
       v
  +-------------------+
  | Router Cache      |  Client-side cache of visited routes
  | (Client)          |  Duration: Session (auto), 30s (dynamic)
  +-------------------+
       |  MISS
       v
  +-------------------+
  | Full Route Cache  |  Cached HTML + RSC payload for static routes
  | (Server)          |  Duration: Until revalidation/redeployment
  +-------------------+
       |  MISS
       v
  +-------------------+
  | Data Cache        |  Cached fetch() responses
  | (Server)          |  Duration: Based on revalidate setting
  +-------------------+
       |  MISS
       v
  +-------------------+
  | Request Memoize   |  Dedup identical fetches in same render
  | (Server, per req) |  Duration: Single server request
  +-------------------+
       |
       v
  Actual Data Source (API, DB, etc.)
```

### Time-Based Revalidation

```tsx
// Entire route: revalidate every 60 seconds
export const revalidate = 60;

export default async function ProductsPage() {
  const products = await fetch('https://api.example.com/products');
  // Cached for 60s, then background refresh
  return <ProductList products={await products.json()} />;
}
```

### On-Demand Revalidation

```tsx
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';

export async function POST(request: Request) {
  const body = await request.json();

  // Option 1: Revalidate a specific page
  revalidatePath('/products');

  // Option 2: Revalidate a layout and all its children
  revalidatePath('/dashboard', 'layout');

  // Option 3: Revalidate by tag (best for granular control)
  revalidateTag('products');

  return Response.json({ revalidated: true, now: Date.now() });
}

// In your data fetching:
async function getProducts() {
  const res = await fetch('https://api.example.com/products', {
    next: { tags: ['products'] },
  });
  return res.json();
}
```

### When to Use Each Caching Strategy

```
+---------------------+------------------------------+-------------------+
| Strategy            | Use Case                     | Config            |
+---------------------+------------------------------+-------------------+
| Default (cached)    | Content that never changes   | (no config)       |
|                     | Blog posts, docs             |                   |
+---------------------+------------------------------+-------------------+
| Time-based (ISR)    | Content updated periodically | revalidate: 3600  |
|                     | Product catalog, news feed   |                   |
+---------------------+------------------------------+-------------------+
| On-demand           | CMS content after publish    | tags + revalidate |
|                     | After admin updates          | Tag()             |
+---------------------+------------------------------+-------------------+
| No cache (SSR)      | User-specific data           | cache: 'no-store' |
|                     | Real-time data, search       | or cookies()      |
+---------------------+------------------------------+-------------------+
```

---

## Server Actions

Server Actions are functions that run on the server, called directly from client or server components. They replace API routes for mutations (like Angular's HttpClient POST/PUT/DELETE calls).

### Basic Server Action

```tsx
// app/actions.ts
'use server'; // This directive marks ALL exports as Server Actions

export async function createTodo(formData: FormData) {
  const title = formData.get('title') as string;

  // Direct database access -- runs on server
  await db.todo.create({
    data: { title, completed: false },
  });

  // Revalidate the page to show new data
  revalidatePath('/todos');
}
```

### Using Server Actions in Forms

```tsx
// app/todos/page.tsx (Server Component)
import { createTodo } from '../actions';

export default async function TodosPage() {
  const todos = await db.todo.findMany();

  return (
    <div>
      {/* Form directly calls server action -- NO API route needed */}
      <form action={createTodo}>
        <input name="title" placeholder="New todo..." required />
        <button type="submit">Add Todo</button>
      </form>

      <ul>
        {todos.map(todo => (
          <li key={todo.id}>{todo.title}</li>
        ))}
      </ul>
    </div>
  );
}
```

### Server Actions in Client Components

```tsx
// components/TodoForm.tsx
'use client';

import { useTransition } from 'react';
import { createTodo } from '@/app/actions';

export default function TodoForm() {
  const [isPending, startTransition] = useTransition();

  const handleSubmit = async (formData: FormData) => {
    startTransition(async () => {
      await createTodo(formData);
    });
  };

  return (
    <form action={handleSubmit}>
      <input name="title" placeholder="New todo..." required />
      <button type="submit" disabled={isPending}>
        {isPending ? 'Adding...' : 'Add Todo'}
      </button>
    </form>
  );
}
```

### Server Actions with Validation

```tsx
// app/actions.ts
'use server';

import { z } from 'zod';
import { revalidatePath } from 'next/cache';

const CreateUserSchema = z.object({
  name: z.string().min(2),
  email: z.string().email(),
});

export type ActionState = {
  errors?: Record<string, string[]>;
  message?: string;
  success?: boolean;
};

export async function createUser(
  prevState: ActionState,
  formData: FormData
): Promise<ActionState> {
  // Validate
  const result = CreateUserSchema.safeParse({
    name: formData.get('name'),
    email: formData.get('email'),
  });

  if (!result.success) {
    return {
      errors: result.error.flatten().fieldErrors,
      message: 'Validation failed',
      success: false,
    };
  }

  // Save to database
  try {
    await db.user.create({ data: result.data });
    revalidatePath('/users');
    return { message: 'User created!', success: true };
  } catch (error) {
    return { message: 'Database error', success: false };
  }
}
```

```tsx
// components/UserForm.tsx
'use client';

import { useActionState } from 'react';
import { createUser, ActionState } from '@/app/actions';

export default function UserForm() {
  const initialState: ActionState = {};
  const [state, formAction, isPending] = useActionState(createUser, initialState);

  return (
    <form action={formAction}>
      <div>
        <label htmlFor="name">Name</label>
        <input id="name" name="name" />
        {state.errors?.name && (
          <p className="text-red-500">{state.errors.name[0]}</p>
        )}
      </div>

      <div>
        <label htmlFor="email">Email</label>
        <input id="email" name="email" type="email" />
        {state.errors?.email && (
          <p className="text-red-500">{state.errors.email[0]}</p>
        )}
      </div>

      <button type="submit" disabled={isPending}>
        {isPending ? 'Creating...' : 'Create User'}
      </button>

      {state.message && (
        <p className={state.success ? 'text-green-500' : 'text-red-500'}>
          {state.message}
        </p>
      )}
    </form>
  );
}
```

### Angular HttpClient vs Server Actions Comparison

```
Angular (Client -> API -> DB):           Next.js Server Actions (Client -> Server -> DB):

  Browser                                  Browser
    |                                        |
    | this.http.post('/api/users', data)     | <form action={createUser}>
    |                                        |
    v                                        v
  Angular HttpClient                       Server Action Function
    |                                        |
    | HTTP POST request                      | Direct function call
    |                                        | (serialized over network)
    v                                        v
  Express/Spring Controller                Server-side function body
    |                                        |
    | Database query                         | Database query
    v                                        v
  Database                                 Database

  With Angular, you NEED a separate backend.
  With Server Actions, the "backend" is the same Next.js app.
```

---

## Route Handlers (API Routes)

Route Handlers are Next.js API endpoints, like Express routes or Spring Boot controllers.

### Basic Route Handler

```tsx
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';

// GET /api/users
export async function GET(request: NextRequest) {
  const searchParams = request.nextUrl.searchParams;
  const page = searchParams.get('page') ?? '1';

  const users = await db.user.findMany({
    skip: (parseInt(page) - 1) * 10,
    take: 10,
  });

  return NextResponse.json(users);
}

// POST /api/users
export async function POST(request: NextRequest) {
  const body = await request.json();

  const user = await db.user.create({
    data: {
      name: body.name,
      email: body.email,
    },
  });

  return NextResponse.json(user, { status: 201 });
}
```

### Dynamic Route Handler

```tsx
// app/api/users/[id]/route.ts
import { NextRequest, NextResponse } from 'next/server';

// GET /api/users/123
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const user = await db.user.findUnique({ where: { id } });

  if (!user) {
    return NextResponse.json(
      { error: 'User not found' },
      { status: 404 }
    );
  }

  return NextResponse.json(user);
}

// PUT /api/users/123
export async function PUT(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  const body = await request.json();

  const user = await db.user.update({
    where: { id },
    data: body,
  });

  return NextResponse.json(user);
}

// DELETE /api/users/123
export async function DELETE(
  request: NextRequest,
  { params }: { params: Promise<{ id: string }> }
) {
  const { id } = await params;
  await db.user.delete({ where: { id } });
  return new NextResponse(null, { status: 204 });
}
```

### Route Handler with Headers and Cookies

```tsx
// app/api/auth/route.ts
import { cookies, headers } from 'next/headers';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const { username, password } = await request.json();

  // Read headers
  const headersList = await headers();
  const userAgent = headersList.get('user-agent');

  // Authenticate
  const token = await authenticate(username, password);

  if (!token) {
    return NextResponse.json({ error: 'Invalid credentials' }, { status: 401 });
  }

  // Set cookie
  const cookieStore = await cookies();
  cookieStore.set('auth-token', token, {
    httpOnly: true,
    secure: true,
    sameSite: 'lax',
    maxAge: 60 * 60 * 24 * 7, // 1 week
  });

  return NextResponse.json({ message: 'Logged in' });
}
```

### When to Use Route Handlers vs Server Actions

```
+-------------------+----------------------------------+----------------------------+
| Feature           | Server Actions                   | Route Handlers             |
+-------------------+----------------------------------+----------------------------+
| Purpose           | Mutations (create, update, del.) | Full REST API              |
| Invocation        | Form action or direct call       | HTTP request (any client)  |
| Used by           | Your Next.js app                 | Any client (mobile, ext.)  |
| HTTP methods      | POST only (under the hood)       | GET, POST, PUT, DELETE     |
| Caching           | No caching                       | GET can be cached          |
| Best for          | Form submissions, data mutations | Public APIs, webhooks      |
+-------------------+----------------------------------+----------------------------+
```

---

## Client-Side Data Fetching

Sometimes you need to fetch data on the client (e.g., after user interaction, real-time updates, infinite scroll).

### Using SWR (Stale-While-Revalidate)

SWR is Vercel's data fetching library. It provides caching, revalidation, and optimistic updates.

```tsx
// components/UserProfile.tsx
'use client';

import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(res => res.json());

export default function UserProfile({ userId }: { userId: string }) {
  const { data, error, isLoading, mutate } = useSWR(
    `/api/users/${userId}`,
    fetcher,
    {
      revalidateOnFocus: true,     // Refetch when user returns to tab
      revalidateOnReconnect: true, // Refetch when internet reconnects
      refreshInterval: 30000,      // Poll every 30 seconds
    }
  );

  if (isLoading) return <div>Loading...</div>;
  if (error) return <div>Error: {error.message}</div>;

  return (
    <div>
      <h2>{data.name}</h2>
      <p>{data.email}</p>
      <button onClick={() => mutate()}>Refresh</button>
    </div>
  );
}
```

### Using React Query (TanStack Query)

React Query is more feature-rich than SWR for complex data fetching scenarios.

```tsx
// providers/QueryProvider.tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState, ReactNode } from 'react';

export default function QueryProvider({ children }: { children: ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,  // Data fresh for 60 seconds
        refetchOnWindowFocus: false,
      },
    },
  }));

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}

// app/layout.tsx
import QueryProvider from '@/providers/QueryProvider';

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  );
}
```

```tsx
// hooks/useUsers.ts
'use client';

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export function useUsers() {
  return useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const res = await fetch('/api/users');
      if (!res.ok) throw new Error('Failed to fetch');
      return res.json() as Promise<User[]>;
    },
  });
}

export function useCreateUser() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (newUser: CreateUserInput) => {
      const res = await fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newUser),
      });
      if (!res.ok) throw new Error('Failed to create');
      return res.json();
    },
    onSuccess: () => {
      // Invalidate and refetch users list
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
}

// Usage in component
function UserList() {
  const { data: users, isLoading, error } = useUsers();
  const createUser = useCreateUser();

  if (isLoading) return <p>Loading...</p>;
  if (error) return <p>Error: {error.message}</p>;

  return (
    <div>
      <button
        onClick={() => createUser.mutate({ name: 'New User', email: 'new@example.com' })}
        disabled={createUser.isPending}
      >
        Add User
      </button>
      {users?.map(user => <p key={user.id}>{user.name}</p>)}
    </div>
  );
}
```

### Comparison: Angular HttpClient vs SWR vs React Query

```
+------------------+-------------------+------------------+------------------+
| Feature          | Angular HttpClient| SWR              | React Query      |
+------------------+-------------------+------------------+------------------+
| Paradigm         | Observables/RxJS  | Hooks            | Hooks            |
| Caching          | Manual / NgRx     | Built-in SWR     | Built-in         |
| Auto revalidate  | No                | Yes              | Yes              |
| Optimistic update| Manual            | mutate()         | onMutate()       |
| Deduplication    | No                | Yes              | Yes              |
| Retry            | retry() operator  | Built-in         | Built-in         |
| Pagination       | Manual            | useSWRInfinite   | useInfiniteQuery |
| DevTools         | No                | Yes              | Yes (excellent)  |
| Server-side      | Angular Universal | SWRConfig        | Hydration        |
| Cancellation     | unsubscribe()     | Automatic        | Automatic        |
| Bundle size      | Built-in          | ~4KB             | ~13KB            |
+------------------+-------------------+------------------+------------------+
```

---

## Data Flow Diagram

```
+=====================================================================+
|                    Next.js Data Flow Architecture                    |
+=====================================================================+

  1. SERVER-SIDE FETCHING (Preferred)
  ===================================

  +------------------+          +------------------+
  |  Server          |  fetch   |  External API /  |
  |  Component       |--------->|  Database         |
  |  (page.tsx)      |          |  (Spring Boot)   |
  |                  |<---------|                   |
  |  async function  |  JSON    |                   |
  +--------+---------+          +------------------+
           |
           | Props (serialized)
           v
  +------------------+
  |  Client          |
  |  Component       |  Interactive parts only
  |  ('use client')  |  (buttons, forms, etc.)
  +------------------+


  2. SERVER ACTIONS (Mutations)
  =============================

  +------------------+    POST (auto)    +------------------+
  |  Client          |----------------->|  Server Action    |
  |  Component       |                  |  ('use server')   |
  |  <form action=>  |                  |                   |
  |                  |<----- revalidate |  DB write +       |
  |  Updated UI      |     + new data   |  revalidatePath() |
  +------------------+                  +------------------+


  3. ROUTE HANDLERS (API Endpoints)
  =================================

  +------------------+    HTTP Request   +------------------+
  |  Any Client      |----------------->|  Route Handler    |
  |  (Browser,       |                  |  /api/xxx/route.ts|
  |   Mobile App,    |                  |                   |
  |   External)      |<----- JSON ------|  GET, POST, PUT,  |
  +------------------+                  |  DELETE           |
                                        +------------------+


  4. CLIENT-SIDE FETCHING (When Needed)
  ======================================

  +------------------+    SWR/React     +------------------+
  |  Client          |    Query         |  Route Handler   |
  |  Component       |----------------->|  or External API |
  |  ('use client')  |                  |                  |
  |                  |<----- JSON ------|                  |
  |  useSWR() or     |  + caching      |                  |
  |  useQuery()      |  + revalidation |                  |
  +------------------+                  +------------------+


  DECISION GUIDE:
  +-----------------------------------------------------------------+
  | Need                    | Solution                              |
  +-------------------------+---------------------------------------+
  | Initial page data       | Server Component (async fetch/DB)     |
  | Form submission         | Server Action                         |
  | Public REST API         | Route Handler                         |
  | Real-time/polling data  | Client-side (SWR/React Query)         |
  | After user interaction  | Client-side or Server Action          |
  | Search with params      | Server Component (searchParams)       |
  +-----------------------------------------------------------------+
```

---

## Angular HttpClient vs Next.js Comparison

| Aspect | Angular HttpClient | Next.js Data Fetching |
|---|---|---|
| **Where it runs** | Always in browser | Server or Client |
| **Import** | `HttpClientModule` in module | Just `fetch` or ORM |
| **Return type** | `Observable<T>` | `Promise<T>` |
| **Interceptors** | `HttpInterceptor` class | `middleware.ts` or wrapper functions |
| **Error handling** | `catchError` RxJS operator | try/catch or error.tsx |
| **Loading state** | Manual boolean flag | `loading.tsx` (auto) or `isPending` |
| **Caching** | Manual or NgRx | Built-in fetch cache |
| **Cancellation** | `unsubscribe()` | AbortController or automatic |
| **Base URL** | `HttpClient` with base URL | Environment variables |
| **Auth headers** | Interceptor adds token | middleware.ts or fetch wrapper |
| **Type safety** | Generic `get<T>()` | TypeScript + Zod validation |
| **Mutations** | `post()`, `put()`, `delete()` | Server Actions or Route Handlers |
| **Transforms** | `pipe(map(...))` | Plain JS transforms |

### Typical Angular HTTP Flow vs Next.js

```
Angular Flow:
  Component -> Service (HttpClient) -> Interceptor -> Backend API -> Response
  (all in browser)

Next.js Flow (Server Component):
  Server Component -> fetch() or DB query -> Response
  (on the server, no round-trip to browser)

Next.js Flow (Client Component):
  Client Component -> SWR/React Query -> Route Handler or API -> Response
  (similar to Angular but with better caching)

Next.js Flow (Server Action):
  Client Component -> Server Action -> DB mutation -> revalidate -> Updated UI
  (no API route needed for mutations)
```

---

## Interview Questions

### Q1: What are the different ways to fetch data in Next.js?

**A:** (1) **Server Components** -- async components that fetch data directly using `fetch` or database queries. This is the primary method. (2) **Server Actions** -- server-side functions for mutations (create/update/delete), invoked from forms or client components. (3) **Route Handlers** -- REST API endpoints for external clients or webhooks. (4) **Client-side fetching** -- using SWR, React Query, or plain `fetch` in client components for real-time data, polling, or post-interaction fetching. The recommendation is to prefer server-side fetching and push client-side fetching to the leaves of the component tree.

### Q2: How does Next.js caching work with fetch?

**A:** Next.js extends `fetch` with caching options. By default, fetch responses are cached indefinitely (static behavior). Use `cache: 'no-store'` to skip caching (dynamic/SSR). Use `next: { revalidate: N }` for time-based revalidation (ISR) where cached data serves for N seconds, then regenerates in the background. Use `next: { tags: ['tag'] }` with `revalidateTag('tag')` for on-demand revalidation. There are four cache layers: Request Memoization (dedup same-render fetches), Data Cache (persistent fetch cache), Full Route Cache (static route HTML), and Router Cache (client-side navigation cache).

### Q3: What are Server Actions and when would you use them?

**A:** Server Actions are async functions marked with `'use server'` that execute on the server. They're used for data mutations -- creating, updating, or deleting data. They can be used directly as form `action` attributes or called from client components. They replace the need for API routes for mutations. Benefits: type-safe, work without JavaScript (progressive enhancement in forms), can directly access databases, and trigger revalidation. Think of them as replacing Angular's `HttpClient.post()` + Spring Boot controller for internal mutations.

### Q4: How do you prevent waterfall data fetching in Next.js?

**A:** Three approaches: (1) **Promise.all** -- start multiple fetches simultaneously: `const [a, b] = await Promise.all([fetchA(), fetchB()])`. (2) **Streaming with Suspense** -- wrap each async section in `<Suspense>` boundaries so they load and display independently. (3) **Preload pattern** -- call a preload function early in the component tree that initiates the fetch, which is then deduplicated by Request Memoization when actually awaited later. The worst pattern is sequential awaits where each fetch waits for the previous to complete.

### Q5: When would you use Route Handlers vs Server Actions?

**A:** Use **Server Actions** for mutations from your own Next.js UI (form submissions, button clicks). They provide the simplest developer experience with type safety and automatic revalidation. Use **Route Handlers** when: (1) you need a public API consumed by other clients (mobile apps, third-party services), (2) you need to handle webhooks, (3) you need specific HTTP methods and headers control, or (4) you need cacheable GET endpoints. Server Actions only support POST internally; Route Handlers support all HTTP methods.

### Q6: How does Request Memoization work?

**A:** During a single server render, if multiple components call `fetch` with the same URL and options, Next.js makes only one actual HTTP request and shares the result. This is useful because in the App Router, parent layouts and child pages render in the same request, and they might both need the same data (e.g., user info). Unlike a global cache, memoization only lasts for the duration of one server render pass. It only works with `fetch` -- database queries or third-party clients aren't memoized automatically (use React's `cache()` function for those).

### Q7: Compare Angular HttpClient with Next.js data fetching approaches.

**A:** Angular HttpClient always runs in the browser, returns Observables, requires manual loading/error state, and needs interceptors for auth. Next.js Server Components fetch on the server (no client-side network call), use Promises, get automatic loading UI with `loading.tsx`, and handle auth in middleware. Angular needs a separate backend API; Next.js can access databases directly. For client-side fetching, Angular uses RxJS operators for caching/retry; Next.js uses SWR or React Query which provide declarative caching, automatic revalidation, and deduplication without manual Observable management.

### Q8: How do you handle form submissions in Next.js?

**A:** The modern approach uses Server Actions. Define an action with `'use server'`, then use it as a form's `action` prop. For client-side UX, use `useActionState` (for form state + validation errors) or `useTransition` (for pending state). The form works without JavaScript (progressive enhancement) but is enhanced with JavaScript for better UX. For complex forms, use React Hook Form combined with Server Actions. Validation should happen on both client (instant feedback) and server (security) -- use Zod schemas shared between both.

### Q9: What is the difference between `revalidatePath` and `revalidateTag`?

**A:** `revalidatePath('/products')` invalidates the cache for a specific URL path -- everything on that page is refetched. `revalidateTag('products')` invalidates all fetch requests tagged with `'products'` across any page. Tags are more granular: if products appear on three different pages, one `revalidateTag('products')` refreshes the product data on all three pages. `revalidatePath` is URL-based; `revalidateTag` is data-based. Use tags when the same data appears on multiple pages; use path when you want to refresh a specific page entirely.

### Q10: How would you implement real-time data in Next.js?

**A:** Several approaches: (1) **SWR with polling** -- set `refreshInterval` for automatic periodic refetching. Simple and works well for near-real-time needs. (2) **Server-Sent Events (SSE)** -- create a Route Handler that streams events, consume in a client component with EventSource. (3) **WebSockets** -- use libraries like Socket.io with a custom server or separate WebSocket service. (4) **React Query with refetchInterval** -- similar to SWR polling. For most interview scenarios, SWR polling is the simplest answer. True real-time (chat, collaborative editing) needs WebSockets.

### Q11: What is the `unstable_cache` function and when would you use it?

**A:** `unstable_cache` (from `next/cache`) caches the results of non-fetch async operations, like direct database queries using Prisma or Drizzle. Since Next.js only automatically caches `fetch` calls, `unstable_cache` lets you apply similar caching to ORM queries. Usage: `const getCachedUser = unstable_cache(async (id) => db.user.findUnique({ where: { id } }), ['user'], { revalidate: 3600, tags: ['users'] })`. It accepts a function, cache keys, and options for revalidation and tags.

### Q12: How do you handle errors in data fetching?

**A:** Multiple layers: (1) **error.tsx** -- catches rendering errors at the route level, shows error UI with retry. (2) **try/catch in Server Components** -- handle fetch failures gracefully, show fallback content. (3) **notFound()** function -- call from Server Components to trigger `not-found.tsx`. (4) **Zod validation** -- validate API responses to catch unexpected data shapes. (5) **Server Action returns** -- return error objects instead of throwing (for form validation). (6) **SWR/React Query** -- expose `error` state for client-side fetch failures. This layered approach is more granular than Angular's single `ErrorHandler`.

### Q13: How do you set up authentication headers for server-side data fetching?

**A:** Create a wrapper function that reads auth tokens and adds headers: `async function authFetch(url: string) { const cookieStore = await cookies(); const token = cookieStore.get('auth-token')?.value; return fetch(url, { headers: { Authorization: \`Bearer ${token}\` } }); }`. For external APIs, use environment variables for API keys (only available on server). For middleware-level auth, use `middleware.ts` to validate tokens before the request reaches page components. This replaces Angular's `HttpInterceptor` pattern.

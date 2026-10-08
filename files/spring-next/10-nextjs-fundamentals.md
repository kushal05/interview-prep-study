# Next.js Fundamentals

## Table of Contents
1. [What is Next.js?](#what-is-nextjs)
2. [Why Next.js Over Plain React?](#why-nextjs-over-plain-react)
3. [App Router vs Pages Router](#app-router-vs-pages-router)
4. [Project Structure](#project-structure)
5. [File-Based Routing](#file-based-routing)
6. [Layouts and Templates](#layouts-and-templates)
7. [Loading UI and Suspense](#loading-ui-and-suspense)
8. [Error Handling](#error-handling)
9. [Metadata API](#metadata-api)
10. [Angular vs Next.js Project Comparison](#angular-vs-nextjs-project-comparison)
11. [Interview Questions](#interview-questions)

---

## What is Next.js?

Next.js is a **React framework** built by Vercel that adds structure, optimization, and server-side capabilities on top of React. Think of it as the Angular CLI + Angular Universal + Angular Router all rolled into one, but for React.

```
Plain React:                          Next.js:
+-------------------------+           +----------------------------+
| Just a UI library       |           | Full-stack framework       |
| No routing              |           | File-based routing         |
| No SSR                  |           | SSR, SSG, ISR built-in     |
| No API layer            |           | API routes / Server Actions|
| You choose everything   |           | Opinionated + flexible     |
| Manual optimization     |           | Auto code-splitting        |
| CRA/Vite for setup      |           | Built-in bundler (Turbopack)|
+-------------------------+           +----------------------------+

Angular Comparison:
  Angular     ~=  Next.js  (both are opinionated frameworks)
  React       ~=  Angular's Component/Template system only
  Next.js     =   React + Router + SSR + API layer + Optimizations
```

---

## Why Next.js Over Plain React?

| Feature | Plain React | Next.js |
|---|---|---|
| Routing | Manual (React Router) | File-based (automatic) |
| Server rendering | Manual setup | Built-in SSR/SSG/ISR |
| Code splitting | Manual with `lazy()` | Automatic per route |
| API routes | Need Express/separate server | Built-in Route Handlers |
| Image optimization | Manual | `<Image>` component |
| Font optimization | Manual | `next/font` |
| SEO | Poor (client-rendered) | Excellent (server-rendered) |
| Caching | Manual | Built-in fetch caching |
| TypeScript | Manual config | Zero-config |
| Deployment | Complex | Vercel one-click or self-host |

**The pitch:** Next.js gives React the structure and tooling that Angular developers already expect from a framework.

---

## App Router vs Pages Router

Next.js has two routing systems. The **App Router** (introduced in Next.js 13) is the modern standard. The **Pages Router** is legacy but still supported.

```
Pages Router (legacy):               App Router (modern):
/pages                                /app
  /index.tsx      --> /                /page.tsx         --> /
  /about.tsx      --> /about           /about/page.tsx   --> /about
  /blog/[id].tsx  --> /blog/:id        /blog/[id]/page.tsx -> /blog/:id
  /_app.tsx       --> root layout      /layout.tsx       --> root layout
  /api/hello.ts   --> /api/hello       /api/hello/route.ts -> /api/hello

Key Differences:
+-------------------+----------------------+------------------------+
| Feature           | Pages Router         | App Router             |
+-------------------+----------------------+------------------------+
| Components        | Client by default    | Server by default      |
| Data fetching     | getServerSideProps   | async Server Components|
| Layouts           | _app.tsx workaround  | Native layout.tsx      |
| Loading states    | Manual               | loading.tsx built-in   |
| Error handling    | _error.tsx           | error.tsx per route    |
| Streaming         | Not supported        | Built-in Suspense      |
| Server Actions    | Not supported        | Built-in               |
+-------------------+----------------------+------------------------+
```

**Focus on App Router for interviews.** The Pages Router might come up in "legacy codebase" discussions.

---

## Project Structure

### Next.js App Router Project Structure

```
my-nextjs-app/
|
+-- app/                          # App Router directory
|   +-- layout.tsx                # Root layout (like Angular's app.component.html)
|   +-- page.tsx                  # Home page (/)
|   +-- globals.css               # Global styles
|   +-- loading.tsx               # Global loading UI
|   +-- error.tsx                 # Global error UI
|   +-- not-found.tsx             # 404 page
|   |
|   +-- about/
|   |   +-- page.tsx              # /about page
|   |
|   +-- blog/
|   |   +-- page.tsx              # /blog page (list)
|   |   +-- [slug]/
|   |   |   +-- page.tsx          # /blog/:slug (detail)
|   |   |   +-- loading.tsx       # Loading UI for this route
|   |   +-- layout.tsx            # Shared layout for /blog/*
|   |
|   +-- dashboard/
|   |   +-- layout.tsx            # Dashboard layout (sidebar etc.)
|   |   +-- page.tsx              # /dashboard
|   |   +-- settings/
|   |       +-- page.tsx          # /dashboard/settings
|   |
|   +-- api/                      # API Route Handlers
|       +-- users/
|           +-- route.ts          # GET/POST /api/users
|           +-- [id]/
|               +-- route.ts      # GET/PUT/DELETE /api/users/:id
|
+-- components/                   # Shared components
|   +-- ui/                       # UI primitives (Button, Card, etc.)
|   +-- forms/                    # Form components
|   +-- layouts/                  # Layout components
|
+-- lib/                          # Utility functions, configs
|   +-- utils.ts
|   +-- db.ts
|   +-- auth.ts
|
+-- hooks/                        # Custom React hooks
|   +-- useAuth.ts
|   +-- useFetch.ts
|
+-- types/                        # TypeScript type definitions
|   +-- index.ts
|
+-- public/                       # Static files (like Angular's assets/)
|   +-- images/
|   +-- favicon.ico
|
+-- middleware.ts                  # Edge middleware (like Angular interceptors)
+-- next.config.js                # Next.js configuration
+-- tailwind.config.ts            # Tailwind CSS config
+-- tsconfig.json                 # TypeScript config
+-- package.json
+-- .env.local                    # Environment variables
```

### File Conventions in the App Router

| File | Purpose | Angular Equivalent |
|---|---|---|
| `page.tsx` | Route component (makes folder a route) | Routed component |
| `layout.tsx` | Shared wrapper (persists across navigation) | Component with `<router-outlet>` |
| `template.tsx` | Like layout but re-mounts on navigation | N/A |
| `loading.tsx` | Loading UI (auto-wrapped in Suspense) | Route resolver loading state |
| `error.tsx` | Error boundary for the route | ErrorHandler |
| `not-found.tsx` | 404 page | Wildcard route component |
| `route.ts` | API endpoint (GET, POST, etc.) | Express/Spring controller |
| `middleware.ts` | Request interceptor | HTTP Interceptor / Guard |
| `default.tsx` | Fallback for parallel routes | N/A |

---

## File-Based Routing

### Basic Routes

In Angular, you define routes in a routing module. In Next.js, the file system IS the router.

```
Angular Routing:                      Next.js File Routing:

// app-routing.module.ts              // Just create files!
const routes: Routes = [
  { path: '', component: Home },      app/page.tsx           --> /
  { path: 'about', component: About },app/about/page.tsx     --> /about
  { path: 'contact', component: ... },app/contact/page.tsx   --> /contact
];                                     app/blog/page.tsx      --> /blog
```

### Dynamic Routes (like Angular's `:id` params)

**Angular:**
```typescript
{ path: 'blog/:slug', component: BlogPostComponent }

// In component:
this.route.params.subscribe(params => {
  this.slug = params['slug'];
});
```

**Next.js:**
```
app/blog/[slug]/page.tsx  --> /blog/hello-world, /blog/my-post
```

```tsx
// app/blog/[slug]/page.tsx
interface Props {
  params: Promise<{ slug: string }>;
}

export default async function BlogPost({ params }: Props) {
  const { slug } = await params;
  // Fetch blog post by slug
  const post = await getPost(slug);

  return (
    <article>
      <h1>{post.title}</h1>
      <p>{post.content}</p>
    </article>
  );
}
```

### Catch-All Routes (like Angular's `**` wildcard)

```
app/docs/[...slug]/page.tsx

Matches:
  /docs/a           --> params.slug = ['a']
  /docs/a/b         --> params.slug = ['a', 'b']
  /docs/a/b/c       --> params.slug = ['a', 'b', 'c']
```

```
app/docs/[[...slug]]/page.tsx    (Optional catch-all with double brackets)

Matches ALL of the above PLUS:
  /docs              --> params.slug = undefined
```

### Route Groups (organize without affecting URL)

Route groups use `(groupName)` syntax. They let you organize routes and apply layouts without adding URL segments.

```
app/
  (marketing)/           # Group -- does NOT appear in URL
    about/page.tsx       # /about  (not /marketing/about)
    blog/page.tsx        # /blog
    layout.tsx           # Shared marketing layout
  (dashboard)/           # Another group
    settings/page.tsx    # /settings
    profile/page.tsx     # /profile
    layout.tsx           # Shared dashboard layout (sidebar etc.)
```

**Angular equivalent:** This is like having separate routing modules with different layouts, but in Next.js it's just folder organization.

### Comparison Table

| Pattern | Angular Router | Next.js App Router |
|---|---|---|
| Static route | `{ path: 'about' }` | `app/about/page.tsx` |
| Dynamic param | `{ path: 'blog/:id' }` | `app/blog/[id]/page.tsx` |
| Wildcard/catch-all | `{ path: '**' }` | `app/[...slug]/page.tsx` |
| Optional param | Not built-in | `app/[[...slug]]/page.tsx` |
| Nested routes | `children: [...]` | Nested folders |
| Named outlets | `outlet: 'sidebar'` | Parallel routes `@sidebar` |
| Guards | `canActivate` | `middleware.ts` |
| Lazy loading | `loadChildren` | Automatic per route |
| Route data | `data: { title: 'X' }` | `generateMetadata()` |
| Query params | `queryParams` | `searchParams` prop |

### Accessing Route and Query Parameters

```tsx
// app/products/[category]/page.tsx
// URL: /products/electronics?sort=price&page=2

interface Props {
  params: Promise<{ category: string }>;
  searchParams: Promise<{ sort?: string; page?: string }>;
}

export default async function ProductsPage({ params, searchParams }: Props) {
  const { category } = await params;       // 'electronics'
  const { sort, page } = await searchParams; // 'price', '2'

  return (
    <div>
      <h1>Category: {category}</h1>
      <p>Sort: {sort}, Page: {page}</p>
    </div>
  );
}
```

### Programmatic Navigation

**Angular:**
```typescript
constructor(private router: Router) {}
navigate() {
  this.router.navigate(['/blog', this.slug]);
}
```

**Next.js:**
```tsx
'use client'; // Navigation hooks require client component

import { useRouter, usePathname, useSearchParams } from 'next/navigation';
import Link from 'next/link';

function Navigation() {
  const router = useRouter();
  const pathname = usePathname();         // Current path
  const searchParams = useSearchParams(); // Query params

  return (
    <nav>
      {/* Declarative (like Angular routerLink) */}
      <Link href="/about">About</Link>
      <Link href={`/blog/${slug}`}>Blog Post</Link>

      {/* Programmatic */}
      <button onClick={() => router.push('/dashboard')}>
        Go to Dashboard
      </button>
      <button onClick={() => router.back()}>Back</button>
      <button onClick={() => router.refresh()}>Refresh</button>
    </nav>
  );
}
```

---

## Layouts and Templates

### Layouts (Persist Across Navigation)

Layouts wrap pages and PERSIST when navigating between child routes. They do NOT re-mount. This is like Angular's component with `<router-outlet>`.

```tsx
// app/layout.tsx -- ROOT layout (required, like Angular's app.component)
import { ReactNode } from 'react';

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html lang="en">
      <body>
        <header>
          <nav>My App Navigation</nav>
        </header>
        <main>{children}</main>  {/* Like <router-outlet> */}
        <footer>Footer</footer>
      </body>
    </html>
  );
}
```

### Nested Layouts

```
Angular:                               Next.js:

AppComponent                           app/layout.tsx
  <router-outlet>                        {children}
    DashboardComponent                   app/dashboard/layout.tsx
      <router-outlet>                      {children}
        SettingsComponent                  app/dashboard/settings/page.tsx
```

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({ children }: { children: ReactNode }) {
  return (
    <div className="flex">
      <aside className="w-64">
        <nav>
          <Link href="/dashboard">Overview</Link>
          <Link href="/dashboard/settings">Settings</Link>
          <Link href="/dashboard/analytics">Analytics</Link>
        </nav>
      </aside>
      <div className="flex-1">
        {children}  {/* Dashboard pages render here */}
      </div>
    </div>
  );
}
```

### How Layout Nesting Works

```
URL: /dashboard/settings

+-- RootLayout (app/layout.tsx) -------------------------+
|  <html><body>                                          |
|  +-- Header / Nav -----------------------------------+ |
|  |  My App Navigation                                | |
|  +---------------------------------------------------+ |
|                                                        |
|  +-- DashboardLayout (app/dashboard/layout.tsx) -----+ |
|  |  +-- Sidebar --+  +-- Content -----------------+  | |
|  |  | Overview    |  |                             |  | |
|  |  | Settings *  |  |  SettingsPage               |  | |
|  |  | Analytics   |  |  (app/dashboard/settings/   |  | |
|  |  |             |  |   page.tsx)                  |  | |
|  |  +-------------+  +-----------------------------+  | |
|  +----------------------------------------------------+ |
|                                                        |
|  +-- Footer -----------------------------------------+ |
|  +---------------------------------------------------+ |
|  </body></html>                                        |
+--------------------------------------------------------+
```

### Templates (Re-Mount on Navigation)

Templates are like layouts but re-mount on every navigation. Use them when you need fresh state on each navigation.

```tsx
// app/dashboard/template.tsx
// This component re-mounts when navigating between dashboard pages
export default function DashboardTemplate({ children }: { children: ReactNode }) {
  // State resets on each navigation (unlike layout)
  return (
    <div>
      <AnimationWrapper>{children}</AnimationWrapper>
    </div>
  );
}
```

---

## Loading UI and Suspense

### Angular vs Next.js Loading States

**Angular (manual):**
```typescript
@Component({
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="!loading">{{ data }}</div>
  `
})
export class MyComponent implements OnInit {
  loading = true;
  data: any;

  ngOnInit() {
    this.http.get('/api/data').subscribe(d => {
      this.data = d;
      this.loading = false;
    });
  }
}
```

**Next.js (automatic with loading.tsx):**

```tsx
// app/dashboard/loading.tsx
// This file automatically shows while page.tsx is loading
export default function Loading() {
  return (
    <div className="flex items-center justify-center">
      <div className="animate-spin h-8 w-8 border-4 border-blue-500
                      rounded-full border-t-transparent" />
      <p>Loading dashboard...</p>
    </div>
  );
}

// app/dashboard/page.tsx
// This is an async Server Component -- loading.tsx shows while it loads
export default async function Dashboard() {
  const data = await fetch('https://api.example.com/dashboard');
  const stats = await data.json();

  return <DashboardView stats={stats} />;
}
```

### Manual Suspense Boundaries

```tsx
import { Suspense } from 'react';

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>

      {/* Each section loads independently */}
      <Suspense fallback={<p>Loading stats...</p>}>
        <StatsSection />   {/* async server component */}
      </Suspense>

      <Suspense fallback={<p>Loading chart...</p>}>
        <ChartSection />   {/* async server component */}
      </Suspense>

      <Suspense fallback={<p>Loading recent...</p>}>
        <RecentActivity />  {/* async server component */}
      </Suspense>
    </div>
  );
}
```

---

## Error Handling

### error.tsx -- Route-Level Error Boundary

```tsx
// app/dashboard/error.tsx
'use client'; // Error components MUST be client components

import { useEffect } from 'react';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  useEffect(() => {
    // Log error to reporting service
    console.error(error);
  }, [error]);

  return (
    <div>
      <h2>Something went wrong!</h2>
      <p>{error.message}</p>
      <button onClick={reset}>Try Again</button>
    </div>
  );
}
```

### not-found.tsx -- Custom 404 Page

```tsx
// app/not-found.tsx
import Link from 'next/link';

export default function NotFound() {
  return (
    <div>
      <h2>404 - Page Not Found</h2>
      <p>The page you are looking for does not exist.</p>
      <Link href="/">Go Home</Link>
    </div>
  );
}
```

### Error Boundary Hierarchy

```
app/
  error.tsx          <-- Catches errors from ALL routes (except layout.tsx)
  layout.tsx
  page.tsx

  dashboard/
    error.tsx        <-- Catches errors in /dashboard/* routes
    layout.tsx       <-- Errors HERE bubble up to app/error.tsx
    page.tsx         <-- Errors here caught by dashboard/error.tsx

    settings/
      error.tsx      <-- Most specific: catches /dashboard/settings errors
      page.tsx

  global-error.tsx   <-- Catches errors in the ROOT layout.tsx
                         (replaces the entire HTML shell)
```

**Angular comparison:** This is like having granular `ErrorHandler` per route segment, whereas Angular has one global `ErrorHandler`.

---

## Metadata API

Next.js has a built-in system for managing `<head>` tags (title, description, Open Graph, etc.).

### Static Metadata

```tsx
// app/about/page.tsx
import { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'About Us',
  description: 'Learn about our company',
  openGraph: {
    title: 'About Us',
    description: 'Learn about our company',
    images: ['/images/about-og.jpg'],
  },
};

export default function AboutPage() {
  return <h1>About Us</h1>;
}
```

### Dynamic Metadata

```tsx
// app/blog/[slug]/page.tsx
import { Metadata } from 'next';

interface Props {
  params: Promise<{ slug: string }>;
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { slug } = await params;
  const post = await getPost(slug);

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      images: [post.coverImage],
    },
  };
}

export default async function BlogPost({ params }: Props) {
  const { slug } = await params;
  const post = await getPost(slug);
  return <article>{post.content}</article>;
}
```

**Angular comparison:** In Angular you use the `Title` and `Meta` services injected into components. Next.js makes it declarative -- just export a `metadata` object or `generateMetadata` function.

---

## Angular vs Next.js Project Comparison

```
Angular Project:                      Next.js Project:
================                      =================

angular.json                          next.config.js
tsconfig.json                         tsconfig.json
package.json                          package.json
.env                                  .env.local

src/                                  app/
  app/                                  layout.tsx         (AppComponent)
    app.component.ts                    page.tsx           (HomeComponent)
    app.component.html                  globals.css
    app.module.ts                       (no modules needed)
    app-routing.module.ts               (file-based routing)

    pages/                              about/
      home/                               page.tsx
        home.component.ts
        home.component.html
      about/
        about.component.ts              blog/
        about.component.html              page.tsx
      blog/                               [slug]/
        blog.component.ts                   page.tsx
        blog-detail.component.ts
        blog-routing.module.ts           dashboard/
      dashboard/                           layout.tsx
        dashboard.component.ts             page.tsx
        dashboard.module.ts                settings/
        settings/                            page.tsx
          settings.component.ts

    shared/                           components/
      components/                       ui/
        button/                           Button.tsx
          button.component.ts             Card.tsx
          button.component.html
      services/                       lib/
        auth.service.ts                 auth.ts
        api.service.ts                  api.ts
      guards/                         middleware.ts
        auth.guard.ts
      interceptors/                   (middleware.ts handles this too)
        auth.interceptor.ts
      pipes/                          (use plain functions or useMemo)
        date-format.pipe.ts
      models/                         types/
        user.model.ts                   user.ts

  assets/                             public/
    images/                             images/
  environments/                       .env.local, .env.production
    environment.ts
    environment.prod.ts
```

Key differences:
- **No modules:** Next.js has no equivalent of `NgModule`. Components are just files you import.
- **No services/DI:** Use plain functions, custom hooks, or Context.
- **No pipes:** Use regular JavaScript functions or `useMemo`.
- **No decorators:** No `@Component`, `@Injectable`, etc.
- **Routing is the file system:** No routing configuration files needed.

---

## Interview Questions

### Q1: What is Next.js and why would you choose it over plain React?

**A:** Next.js is a React framework by Vercel that provides server-side rendering (SSR), static site generation (SSG), file-based routing, API routes, and automatic code splitting out of the box. You'd choose it over plain React when you need: SEO (server-rendered HTML), fast initial page loads, image/font optimization, a structured project architecture, or full-stack capabilities (API routes, Server Actions). It's similar to how Angular provides a complete framework vs. just using a templating library.

### Q2: Explain the difference between the App Router and Pages Router.

**A:** The Pages Router (legacy) uses the `/pages` directory where each file becomes a route, components are client-side by default, and data fetching uses special functions like `getServerSideProps` and `getStaticProps`. The App Router (modern, Next.js 13+) uses the `/app` directory, components are Server Components by default, supports native layouts via `layout.tsx`, has built-in loading/error states via special files, supports streaming with Suspense, and uses async Server Components for data fetching. New projects should use the App Router.

### Q3: How does file-based routing work in Next.js?

**A:** The folder structure inside `/app` determines the URL structure. A `page.tsx` file inside a folder makes that folder a route. `app/page.tsx` is `/`, `app/about/page.tsx` is `/about`. Dynamic segments use brackets: `app/blog/[slug]/page.tsx` matches `/blog/anything`. Catch-all routes use `[...slug]`, optional catch-all uses `[[...slug]]`. Route groups with `(name)` organize without affecting URLs. This replaces Angular's routing module configuration entirely.

### Q4: What are the special files in the Next.js App Router?

**A:** `page.tsx` defines a route's UI, `layout.tsx` creates shared persistent wrappers (like `<router-outlet>`), `template.tsx` is like layout but re-mounts on navigation, `loading.tsx` shows automatic loading UI (wrapped in Suspense), `error.tsx` creates error boundaries per route, `not-found.tsx` handles 404s, `route.ts` defines API endpoints, and `default.tsx` provides fallback UI for parallel routes. Each file has a specific role in the rendering hierarchy.

### Q5: How do layouts work in Next.js? How do they compare to Angular?

**A:** Layouts in Next.js wrap child pages/layouts and persist across navigation -- they don't re-mount when navigating between child routes. The root `layout.tsx` must exist and wraps the entire app (like Angular's `AppComponent` with `<router-outlet>`). Nested layouts compose automatically. `app/dashboard/layout.tsx` wraps all pages under `/dashboard/*`. This is equivalent to Angular's nested `<router-outlet>` pattern but without explicit route configuration. Layouts receive `children` prop where child content renders.

### Q6: How does Next.js handle loading states?

**A:** Place a `loading.tsx` file alongside `page.tsx` in any route folder. Next.js automatically wraps the page in a React `<Suspense>` boundary with the loading component as fallback. While the page (especially async Server Components) is loading, users see the loading UI. You can also use `<Suspense>` manually for more granular control. This replaces the manual `loading` boolean pattern common in Angular and makes the loading state part of the routing infrastructure.

### Q7: How does error handling work in the App Router?

**A:** `error.tsx` files create React Error Boundaries at the route level. They must be Client Components (`'use client'`). They receive `error` and `reset` props -- `reset` lets users retry. Errors bubble up to the nearest parent error boundary. `global-error.tsx` catches errors in the root layout (it must render its own `<html>` and `<body>` tags). `not-found.tsx` handles 404 errors specifically. This is more granular than Angular's single `ErrorHandler`.

### Q8: What is the Metadata API and why is it important?

**A:** The Metadata API lets you define `<head>` content (title, description, Open Graph tags, etc.) per route. Export a static `metadata` object or an async `generateMetadata` function from `page.tsx` or `layout.tsx`. Metadata merges and overrides from parent to child layouts. This is critical for SEO -- server-rendered metadata means search engines see the correct title/description immediately. In Angular, you'd inject `Title` and `Meta` services and set them imperatively in `ngOnInit`.

### Q9: How do you handle programmatic navigation in Next.js?

**A:** Use the `useRouter()` hook from `next/navigation` in Client Components. Call `router.push('/path')` for navigation, `router.replace('/path')` to replace history, `router.back()` to go back, and `router.refresh()` to re-fetch server component data. For declarative navigation, use the `<Link>` component which also prefetches linked pages automatically. This is similar to Angular's `Router.navigate()` and `routerLink` directive.

### Q10: What are route groups and when would you use them?

**A:** Route groups are folders wrapped in parentheses like `(marketing)` that organize routes without affecting the URL path. Use cases: (1) Apply different layouts to different sections -- `(marketing)` pages get a public layout, `(dashboard)` pages get an admin layout. (2) Organize related routes logically. (3) Create multiple root layouts for different sections of the app. The folder name inside parentheses is purely for developer organization.

### Q11: How do dynamic routes work and how do you access the parameters?

**A:** Dynamic routes use square brackets in folder names: `[id]`, `[slug]`. The page component receives `params` as a prop containing the dynamic values. For `app/blog/[slug]/page.tsx` with URL `/blog/hello`, `params.slug` is `'hello'`. Catch-all `[...slug]` captures multiple segments as an array. Optional catch-all `[[...slug]]` also matches the parent path. In Angular terms, this replaces `ActivatedRoute.params` with a simpler prop-based approach.

### Q12: Compare Next.js Link component with Angular's routerLink.

**A:** Next.js `<Link href="/path">` is equivalent to Angular's `<a routerLink="/path">`. Key difference: Next.js Link automatically prefetches linked pages in the viewport for faster navigation (can be disabled with `prefetch={false}`). Both support dynamic paths. Active link styling in Next.js requires checking `usePathname()` manually, whereas Angular has `routerLinkActive` directive. Next.js Link renders an `<a>` tag and supports all anchor attributes.

### Q13: What is the difference between layout.tsx and template.tsx?

**A:** `layout.tsx` persists across navigations -- its state is preserved and it doesn't re-mount when navigating between child routes. `template.tsx` creates a new instance on every navigation, resetting all state. Use layouts for persistent UI like navigation bars and sidebars. Use templates when you need enter/exit animations, per-page state reset, or effects that should run on every navigation (e.g., page view tracking).

### Q14: How does Next.js handle code splitting?

**A:** Next.js automatically code-splits at the route level -- each page only loads the JavaScript needed for that page. This happens without any configuration. In Angular, you achieve this with lazy-loaded modules (`loadChildren`). Additionally, you can use `next/dynamic` for component-level code splitting (equivalent to React's `lazy()` but with SSR support and loading state options). This is one of the major performance advantages over plain React applications.

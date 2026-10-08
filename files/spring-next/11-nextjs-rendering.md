# Next.js Rendering Strategies

> This is one of the MOST important interview topics for Next.js.
> Understanding rendering strategies is what separates junior from senior Next.js developers.

## Table of Contents
1. [Server Components vs Client Components](#server-components-vs-client-components)
2. [The "use client" Directive](#the-use-client-directive)
3. [Server-Side Rendering (SSR)](#server-side-rendering-ssr)
4. [Static Site Generation (SSG)](#static-site-generation-ssg)
5. [Incremental Static Regeneration (ISR)](#incremental-static-regeneration-isr)
6. [Streaming with Suspense](#streaming-with-suspense)
7. [Rendering Decision Flowchart](#rendering-decision-flowchart)
8. [Hydration Explained](#hydration-explained)
9. [Interview Questions](#interview-questions)

---

## Server Components vs Client Components

This is the most fundamental concept in modern Next.js. In the App Router, components are **Server Components by default**.

### What Are Server Components?

Server Components render ONLY on the server. Their code never ships to the browser. They can directly access databases, file systems, and environment variables.

### What Are Client Components?

Client Components render on the server (for initial HTML) AND hydrate on the client. They can use browser APIs, state, effects, and event handlers.

### Comparison Table

| Feature | Server Component | Client Component |
|---|---|---|
| **Default in App Router** | YES | No (must opt in) |
| **Directive needed** | None | `'use client'` at top of file |
| **Can use useState/useEffect** | NO | YES |
| **Can use event handlers** | NO | YES |
| **Can use browser APIs** | NO | YES |
| **Can access backend directly** | YES (DB, filesystem) | NO |
| **Can use async/await** | YES (component itself) | NO (component function) |
| **JavaScript sent to browser** | NO | YES |
| **Can import Server Components** | YES | NO (passed as children) |
| **Can import Client Components** | YES | YES |

### ASCII Diagram: The Server/Client Boundary

```
                    SERVER                    |              CLIENT (Browser)
                                             |
  +------------------------------------------+------------------------------------+
  |                                          |                                    |
  |  Server Components                       |  Client Components                 |
  |  (No JS sent to browser)                 |  ('use client')                    |
  |                                          |                                    |
  |  +----------------+                      |  +-------------------+             |
  |  | RootLayout     |                      |  | InteractiveNav    |             |
  |  | (layout.tsx)   |                      |  | - useState        |             |
  |  |                |   can render --->     |  | - onClick         |             |
  |  | +------------+ |                      |  | - animations      |             |
  |  | | Header     | |                      |  +-------------------+             |
  |  | | (server)   | |                      |                                    |
  |  | +------------+ |                      |  +-------------------+             |
  |  |                |                      |  | SearchBar         |             |
  |  | +------------+ |                      |  | - useState        |             |
  |  | | BlogPost   | |   can render --->    |  | - onChange         |             |
  |  | | (server)   | |                      |  | - form input      |             |
  |  | | - fetches  | |                      |  +-------------------+             |
  |  | |   data     | |                      |                                    |
  |  | | - no JS    | |                      |  +-------------------+             |
  |  | +------------+ |                      |  | AddToCart         |             |
  |  |                |   can render --->    |  | - onClick         |             |
  |  | +------------+ |                      |  | - useContext      |             |
  |  | | Footer     | |                      |  +-------------------+             |
  |  | | (server)   | |                      |                                    |
  |  | +------------+ |                      |                                    |
  |  +----------------+                      |                                    |
  |                                          |                                    |
  +------------------------------------------+------------------------------------+

  RULE: Push 'use client' boundary as far DOWN the tree as possible.
        Keep most components as Server Components.
```

### Server Component Example

```tsx
// app/blog/[slug]/page.tsx
// This is a Server Component (default) -- NO 'use client' directive

import { db } from '@/lib/db'; // Direct database access!

export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;

  // Direct database query -- this code NEVER runs in the browser
  const post = await db.post.findUnique({
    where: { slug },
    include: { author: true, comments: true },
  });

  if (!post) return notFound();

  return (
    <article>
      <h1>{post.title}</h1>
      <p>By {post.author.name}</p>
      <div>{post.content}</div>

      {/* Client Component for interactivity */}
      <LikeButton postId={post.id} initialLikes={post.likes} />
      <CommentSection comments={post.comments} postId={post.id} />
    </article>
  );
}
```

### Client Component Example

```tsx
// components/LikeButton.tsx
'use client'; // This directive marks the client boundary

import { useState } from 'react';

export default function LikeButton({
  postId,
  initialLikes,
}: {
  postId: string;
  initialLikes: number;
}) {
  const [likes, setLikes] = useState(initialLikes);
  const [hasLiked, setHasLiked] = useState(false);

  const handleLike = async () => {
    setLikes(prev => prev + 1);
    setHasLiked(true);
    await fetch(`/api/posts/${postId}/like`, { method: 'POST' });
  };

  return (
    <button onClick={handleLike} disabled={hasLiked}>
      {hasLiked ? 'Liked' : 'Like'} ({likes})
    </button>
  );
}
```

### Composition Pattern: Server Components Wrapping Client Components

```tsx
// app/dashboard/page.tsx (Server Component)
import { db } from '@/lib/db';
import InteractiveDashboard from './InteractiveDashboard';

export default async function DashboardPage() {
  // Fetch data on the server
  const stats = await db.stats.findMany();
  const users = await db.users.findMany({ take: 10 });

  // Pass SERVER data to CLIENT component as props
  return (
    <InteractiveDashboard
      initialStats={stats}    // Serialized and sent to client
      initialUsers={users}    // Must be serializable (no functions, Dates, etc.)
    />
  );
}
```

```tsx
// app/dashboard/InteractiveDashboard.tsx
'use client';

import { useState } from 'react';

export default function InteractiveDashboard({ initialStats, initialUsers }) {
  const [filter, setFilter] = useState('all');
  // Now we can use interactivity with server-fetched data
  // ...
}
```

### Passing Server Components as Children to Client Components

```tsx
// This is the KEY pattern for mixing server and client components

// ClientWrapper.tsx
'use client';
export default function ClientWrapper({ children }: { children: React.ReactNode }) {
  const [isOpen, setIsOpen] = useState(true);
  return (
    <div>
      <button onClick={() => setIsOpen(!isOpen)}>Toggle</button>
      {isOpen && children}  {/* children can be Server Components! */}
    </div>
  );
}

// page.tsx (Server Component)
import ClientWrapper from './ClientWrapper';
import ServerContent from './ServerContent'; // Server Component

export default function Page() {
  return (
    <ClientWrapper>
      <ServerContent />  {/* Rendered on server, passed as children */}
    </ClientWrapper>
  );
}
```

---

## The "use client" Directive

```
'use client' marks the BOUNDARY, not just the file.

When you add 'use client' to a file:
- That component and ALL components it IMPORTS become Client Components
- But components passed as children/props remain Server Components

                   +-- 'use client' --+
                   |                  |
  Server           |  Client          |  Can still receive
  Component ------>|  Component       |  Server Components
  (page.tsx)       |  (Nav.tsx)       |  as {children}
                   |    |             |
                   |    +-> MenuItem  |  MenuItem is also
                   |    +-> Logo     |  client (imported by Nav)
                   |                  |
                   +------------------+
```

### When to Use 'use client'

```
USE 'use client' WHEN you need:        KEEP as Server Component WHEN:
- useState, useEffect, useRef          - Fetching data
- onClick, onChange, onSubmit           - Accessing backend resources
- Browser APIs (window, document)      - Keeping secrets server-side
- Third-party client libraries         - Rendering static/non-interactive UI
- React Context (useContext)            - Large dependencies (don't send to client)
- Custom hooks that use state/effects
```

---

## Server-Side Rendering (SSR)

SSR generates HTML on the server for EACH request. The HTML is sent to the browser, then JavaScript hydrates it to make it interactive.

### Angular Universal Comparison

```
Angular Universal (SSR):              Next.js SSR:
+---------------------------+         +---------------------------+
| Separate server setup     |         | Built-in, zero config     |
| Express.js server         |         | Automatic                 |
| TransferState for data    |         | Server Components         |
| Complex configuration     |         | Just use async components |
| renderModule()            |         | "Just works"              |
+---------------------------+         +---------------------------+
```

### SSR in App Router

In the App Router, SSR happens automatically for Server Components. For dynamic data:

```tsx
// This page is SSR'd on every request
// because it uses dynamic data (cookies, headers, searchParams)
import { cookies } from 'next/headers';

export default async function ProfilePage() {
  const cookieStore = await cookies();
  const token = cookieStore.get('auth-token');

  const user = await fetch('https://api.example.com/me', {
    headers: { Authorization: `Bearer ${token?.value}` },
    cache: 'no-store', // Force fresh data every request = SSR
  });

  const userData = await user.json();

  return <div>Hello, {userData.name}</div>;
}
```

### SSR in Pages Router (Legacy -- Know This)

```tsx
// pages/profile.tsx (Pages Router)
export async function getServerSideProps(context) {
  const { req, res, params, query } = context;

  const data = await fetch('https://api.example.com/profile');
  const profile = await data.json();

  return {
    props: { profile }, // Passed to component
  };
}

export default function Profile({ profile }) {
  return <div>{profile.name}</div>;
}
```

---

## Static Site Generation (SSG)

SSG generates HTML at **build time**. Pages are pre-rendered and served as static files. Fastest possible delivery.

### When to Use SSG

- Blog posts, documentation, marketing pages
- Content that doesn't change per-request
- Pages where stale data is acceptable

### SSG in App Router

```tsx
// app/blog/[slug]/page.tsx

// Tell Next.js which dynamic paths to pre-render at build time
export async function generateStaticParams() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  return posts.map((post: Post) => ({
    slug: post.slug,  // Each object becomes a set of params
  }));
}

// This runs at BUILD TIME for each set of params
export default async function BlogPost({
  params,
}: {
  params: Promise<{ slug: string }>;
}) {
  const { slug } = await params;
  const post = await fetch(`https://api.example.com/posts/${slug}`).then(r =>
    r.json()
  );

  return (
    <article>
      <h1>{post.title}</h1>
      <div>{post.content}</div>
    </article>
  );
}
```

### SSG in Pages Router (Legacy)

```tsx
// pages/blog/[slug].tsx
export async function getStaticPaths() {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  return {
    paths: posts.map(post => ({ params: { slug: post.slug } })),
    fallback: 'blocking', // or false, or true
  };
}

export async function getStaticProps({ params }) {
  const post = await fetch(`https://api.example.com/posts/${params.slug}`)
    .then(r => r.json());

  return {
    props: { post },
    revalidate: 60, // ISR: regenerate every 60 seconds
  };
}
```

---

## Incremental Static Regeneration (ISR)

ISR is the best of both worlds: static speed with fresh data. Pages are served statically but regenerated in the background after a specified time.

```
Request Timeline with ISR (revalidate: 60):

Time 0s:    Build --> HTML generated (static)
Time 1s:    User A visits --> Serves static HTML (FAST)
Time 30s:   User B visits --> Serves same static HTML
Time 61s:   User C visits --> Serves stale HTML BUT triggers
                              background regeneration
Time 62s:   Background: New HTML generated with fresh data
Time 63s:   User D visits --> Serves NEW static HTML

            +--------+     +--------+     +--------+
            | Build  |     | Stale  |     | Fresh  |
  Request:  | v1     | ... | v1     | ... | v2     |
            +--------+     +--------+     +--------+
  Time:     0              0-60s          After regen
            (generated)    (served from   (regenerated
                            cache)        in background)
```

### ISR in App Router

```tsx
// Option 1: Time-based revalidation
// app/products/page.tsx
export const revalidate = 60; // Revalidate every 60 seconds

export default async function Products() {
  const products = await fetch('https://api.example.com/products');
  const data = await products.json();
  return <ProductList products={data} />;
}

// Option 2: Per-fetch revalidation
export default async function Products() {
  const products = await fetch('https://api.example.com/products', {
    next: { revalidate: 3600 }, // This specific fetch: revalidate hourly
  });
  const data = await products.json();
  return <ProductList products={data} />;
}
```

### On-Demand Revalidation

```tsx
// app/api/revalidate/route.ts
import { revalidatePath, revalidateTag } from 'next/cache';
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const { path, tag, secret } = await request.json();

  // Verify secret to prevent unauthorized revalidation
  if (secret !== process.env.REVALIDATION_SECRET) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  if (path) {
    revalidatePath(path);  // Revalidate a specific path
  }
  if (tag) {
    revalidateTag(tag);    // Revalidate all fetches with this tag
  }

  return NextResponse.json({ revalidated: true });
}

// Usage in fetch:
const data = await fetch('https://api.example.com/products', {
  next: { tags: ['products'] }, // Tag this fetch
});

// Then call: POST /api/revalidate { tag: 'products' }
// to refresh all product data
```

---

## Streaming with Suspense

Streaming allows the server to send HTML in chunks as each part becomes ready, instead of waiting for the entire page.

```
Traditional SSR:
  Server: [====== Render entire page ======] --> Send complete HTML
  Client: ............waiting.................  [Receive + Display]

Streaming SSR:
  Server: [= Shell =] --> [= Section A =] --> [= Section B =]
  Client: [Display]       [Display]            [Display]
          Shell appears    A fills in           B fills in
          immediately!     as ready             as ready

  +---------------------------------------------------+
  |  Shell (instant)                                   |
  |  +-- Header (instant) ---------------------------+ |
  |  | Navigation  Logo  Search                      | |
  |  +----------------------------------------------+ |
  |                                                    |
  |  +-- Stats Section (loading...) -----+             |
  |  | [Skeleton / Spinner]              |  <-- Streams|
  |  | (waiting for DB query)            |      in when|
  |  +-----------------------------------+      ready  |
  |                                                    |
  |  +-- Recent Orders (loading...) -----+             |
  |  | [Skeleton / Spinner]              |  <-- Streams|
  |  | (waiting for API call)            |      in when|
  |  +-----------------------------------+      ready  |
  |                                                    |
  |  +-- Footer (instant) --------------------------+ |
  |  +----------------------------------------------+ |
  +---------------------------------------------------+
```

### Implementing Streaming

```tsx
// app/dashboard/page.tsx
import { Suspense } from 'react';
import StatsSection from './StatsSection';
import RecentOrders from './RecentOrders';
import RevenueChart from './RevenueChart';

export default function Dashboard() {
  return (
    <div>
      <h1>Dashboard</h1>

      {/* Each Suspense boundary streams independently */}
      <Suspense fallback={<StatsSkeleton />}>
        <StatsSection />  {/* Async -- might take 200ms */}
      </Suspense>

      <div className="grid grid-cols-2 gap-4">
        <Suspense fallback={<ChartSkeleton />}>
          <RevenueChart />  {/* Async -- might take 500ms */}
        </Suspense>

        <Suspense fallback={<OrdersSkeleton />}>
          <RecentOrders />  {/* Async -- might take 300ms */}
        </Suspense>
      </div>
    </div>
  );
}

// These are async Server Components
async function StatsSection() {
  const stats = await fetchStats(); // 200ms
  return <div>{/* render stats */}</div>;
}

async function RevenueChart() {
  const data = await fetchRevenue(); // 500ms
  return <div>{/* render chart */}</div>;
}

async function RecentOrders() {
  const orders = await fetchOrders(); // 300ms
  return <div>{/* render orders */}</div>;
}
```

---

## Rendering Decision Flowchart

```
START: What kind of page/component are you building?
|
+-- Is it interactive? (clicks, forms, state, animations)
|   |
|   +-- YES --> Does it need server data?
|   |           |
|   |           +-- YES --> Server Component fetches data,
|   |           |           passes to Client Component as props
|   |           |           (Composition pattern)
|   |           |
|   |           +-- NO  --> Client Component ('use client')
|   |                       with client-side data fetching
|   |
|   +-- NO  --> Server Component (default)
|               |
|               +-- Does data change per request? (user-specific, cookies)
|               |   |
|               |   +-- YES --> Dynamic rendering (SSR)
|               |   |           Use: cookies(), headers(), searchParams
|               |   |           or fetch with cache: 'no-store'
|               |   |
|               |   +-- NO  --> Is data updated occasionally?
|               |               |
|               |               +-- YES --> ISR
|               |               |           Use: revalidate = N
|               |               |           or next: { revalidate: N }
|               |               |
|               |               +-- NO  --> SSG (Static)
|               |                           Data fetched at build time
|               |                           Use: generateStaticParams()
|               |
|               +-- Is it a large page with multiple data sources?
|                   |
|                   +-- YES --> Streaming with Suspense
|                               Wrap each async section in <Suspense>
|                   |
|                   +-- NO  --> Simple Server Component
```

### Quick Decision Table

| Scenario | Strategy | How |
|---|---|---|
| Marketing page, blog post | SSG | `generateStaticParams()` |
| Product page (prices change) | ISR | `revalidate = 3600` |
| User dashboard | SSR | `cookies()` or `cache: 'no-store'` |
| Search results | SSR | `searchParams` |
| Interactive form | Client | `'use client'` |
| Dashboard with many sections | Streaming | `<Suspense>` boundaries |
| Static header/footer | Server Component | Default (no directive) |
| Button with click handler | Client | `'use client'` |
| Data display (no interaction) | Server Component | Default |

---

## Hydration Explained

Hydration is the process where the browser takes the server-rendered HTML and attaches JavaScript event handlers to make it interactive.

```
SERVER RENDERING:
  Server generates HTML string:
  "<button class='btn'>Click Me (0)</button>"
                    |
                    v
  HTML sent to browser (user sees the button IMMEDIATELY)
                    |
                    v
HYDRATION:
  React JavaScript loads in browser
                    |
                    v
  React "walks" the existing DOM and attaches:
  - onClick handler to button
  - useState(0) for the counter
  - Any useEffect callbacks
                    |
                    v
  Button is now INTERACTIVE (was just static HTML before)


TIMELINE:
  |-- HTML received --|-- JS loads --|-- Hydration --|-- Interactive --|
  |                   |              |               |                 |
  User sees           Nothing       React attaches  User can now
  content             changes       event handlers  click buttons
  (fast!)             visually
```

### Hydration Mismatch Errors

A hydration mismatch occurs when the server-rendered HTML doesn't match what React expects to render on the client.

```tsx
// BAD: Causes hydration mismatch
function Timestamp() {
  return <p>Current time: {new Date().toISOString()}</p>;
  // Server: "2024-01-01T12:00:00Z"
  // Client: "2024-01-01T12:00:01Z" (different!)
  // React: "MISMATCH ERROR!"
}

// GOOD: Use useEffect for client-only values
function Timestamp() {
  const [time, setTime] = useState<string>('');

  useEffect(() => {
    setTime(new Date().toISOString());
  }, []);

  return <p>Current time: {time || 'Loading...'}</p>;
}

// GOOD: Or suppress hydration warning when intentional
function Timestamp() {
  return (
    <p suppressHydrationWarning>
      Current time: {new Date().toISOString()}
    </p>
  );
}
```

### Common Hydration Mismatch Causes

```
1. Date/time differences between server and client
2. Browser-only APIs (window.innerWidth) used during render
3. Browser extensions modifying the DOM
4. Invalid HTML nesting (<p> inside <p>, <div> inside <p>)
5. Random values (Math.random(), crypto.randomUUID())
6. Locale-dependent formatting differences
```

### Angular Universal vs Next.js Hydration

| Aspect | Angular Universal | Next.js |
|---|---|---|
| Setup | Complex (separate server module) | Zero config |
| Data transfer | `TransferState` API manually | Automatic (RSC payload) |
| Hydration | Full page hydration | Selective hydration (Suspense) |
| Mismatch handling | Silent failures often | Clear error messages |
| Partial hydration | Not built-in | Server Components = zero JS |

---

## Summary: All Rendering Strategies at a Glance

```
+----------+------------------+------------------+------------------+
|          |     SSG          |     ISR          |     SSR          |
+----------+------------------+------------------+------------------+
| When     | Build time       | Build + BG regen | Every request    |
| Speed    | Fastest (CDN)    | Fast (cached)    | Slower (compute) |
| Freshness| Stale til deploy | Periodically     | Always fresh     |
| Use for  | Blog, docs,      | Products, listing| Dashboards, user |
|          | marketing        | pages            | specific content |
| Cost     | Cheapest (CDN)   | Low (CDN + regen)| Higher (compute) |
+----------+------------------+------------------+------------------+

+-------------------+--------------------------------------------------+
| Server Components | Render on server, zero client JS, direct DB/API  |
| Client Components | Render on both, hydrated on client, interactive  |
| Streaming         | Progressive HTML delivery with Suspense           |
+-------------------+--------------------------------------------------+
```

---

## Interview Questions

### Q1: What is the difference between Server Components and Client Components?

**A:** Server Components (default in App Router) render only on the server -- their JavaScript is never sent to the browser. They can directly access databases, filesystems, and secrets. Client Components (marked with `'use client'`) render on both server (for initial HTML) and client (for interactivity). Client Components can use React hooks (useState, useEffect), event handlers, and browser APIs. The key insight is to keep the client boundary as low as possible in the component tree -- only the interactive leaf components should be Client Components.

### Q2: Explain SSR, SSG, and ISR. When would you use each?

**A:** **SSG (Static Site Generation):** HTML generated at build time, served from CDN. Use for content that rarely changes (blogs, docs, marketing pages). Fastest delivery, cheapest to host. **SSR (Server-Side Rendering):** HTML generated on each request. Use for personalized/dynamic content (dashboards, search results, authenticated pages). Always fresh but slower. **ISR (Incremental Static Regeneration):** Static pages that regenerate in the background after a specified time. Use for content that updates periodically but doesn't need real-time freshness (product catalogs, news feeds). Combines SSG speed with SSR freshness.

### Q3: What is hydration and what problems can occur?

**A:** Hydration is the process where React attaches event handlers and state management to server-rendered HTML in the browser, making it interactive. Problems include: (1) Hydration mismatches when server and client HTML differ (e.g., using `Date.now()` or `Math.random()` during render), causing errors and potential UI bugs. (2) Hydration overhead -- large pages require downloading and executing significant JavaScript before becoming interactive (Time to Interactive penalty). (3) The entire page must hydrate before any part is interactive (solved by selective hydration with Suspense in React 18+).

### Q4: How do you decide between Server and Client Components?

**A:** Default to Server Components. Use Client Components only when you need: interactivity (onClick, onChange), React hooks (useState, useEffect, useContext), browser APIs (window, document, localStorage), or third-party libraries that use these. Pattern: Fetch data in Server Components, pass to Client Components as props. For example, a product page's layout, images, and text are Server Components; only the "Add to Cart" button and quantity selector are Client Components.

### Q5: What is streaming and how does it work with Suspense?

**A:** Streaming allows the server to send HTML progressively in chunks. Instead of waiting for the entire page to render, the server sends the page shell immediately, then streams in each section as its data becomes available. This is implemented with React's `<Suspense>` component -- wrap async Server Components in Suspense with a fallback (skeleton/spinner). The fallback shows immediately; the actual content replaces it when ready. This dramatically improves perceived performance because users see content faster. It's especially valuable for dashboards with multiple independent data sources.

### Q6: How does the `'use client'` directive work?

**A:** `'use client'` at the top of a file marks the **boundary** between server and client modules. That file and everything it imports become part of the client bundle. However, Server Components can be passed to Client Components as `children` or other props -- they remain on the server. The directive does NOT mean the component only renders on the client; it still renders on the server for initial HTML, then hydrates on the client. Think of it as saying "this component needs JavaScript in the browser."

### Q7: How does Next.js handle caching and revalidation?

**A:** Next.js has multiple caching layers: (1) **Request Memoization:** Duplicate `fetch` calls in the same render are automatically deduplicated. (2) **Data Cache:** `fetch` responses are cached on the server and persisted across requests. Control with `cache: 'no-store'` (skip cache), `next: { revalidate: N }` (time-based), or `next: { tags: ['tag'] }` (on-demand with `revalidateTag`). (3) **Full Route Cache:** Static routes are cached as HTML. (4) **Router Cache:** Client-side cache of visited routes for instant back/forward navigation. Use `revalidatePath()` or `revalidateTag()` for on-demand revalidation.

### Q8: Compare Next.js SSR with Angular Universal.

**A:** Both serve pre-rendered HTML for SEO and performance. Differences: Angular Universal requires separate server setup (Express module, server.ts), manual `TransferState` to avoid double-fetching, and complex configuration. Next.js SSR is zero-config -- Server Components fetch data directly, and the framework handles data transfer automatically. Next.js also offers SSG and ISR which Angular Universal doesn't have built-in. Angular Universal does full-page hydration; Next.js with Server Components sends zero JavaScript for non-interactive parts, reducing the hydration cost significantly.

### Q9: What is selective hydration?

**A:** Selective hydration (React 18+) allows React to hydrate different parts of the page independently and prioritize based on user interaction. If a user clicks on a not-yet-hydrated component, React prioritizes hydrating that component first. Combined with Suspense boundaries, different sections of the page can hydrate independently as their JavaScript loads. This means users don't have to wait for the entire page's JavaScript to load before any part is interactive. Server Components take this further by eliminating hydration entirely for non-interactive parts.

### Q10: What happens when you use `cache: 'no-store'` in a fetch call?

**A:** `cache: 'no-store'` opts out of Next.js's data cache, making the fetch execute fresh on every request. This effectively makes the route dynamically rendered (SSR behavior). The page cannot be statically generated because the data must be fetched at request time. Use this for user-specific data, real-time data, or when you need the latest information on every request. Without `cache: 'no-store'`, fetch results are cached indefinitely by default (SSG behavior).

### Q11: How does `generateStaticParams` work?

**A:** `generateStaticParams` is the App Router equivalent of Pages Router's `getStaticPaths`. It tells Next.js which dynamic route segments to pre-render at build time. It returns an array of parameter objects. For `app/blog/[slug]/page.tsx`, returning `[{ slug: 'post-1' }, { slug: 'post-2' }]` generates static pages for `/blog/post-1` and `/blog/post-2`. For unspecified params, Next.js can generate them on first request and cache them (equivalent to `fallback: 'blocking'` in Pages Router). It can be used with nested dynamic segments.

### Q12: What is the RSC (React Server Components) payload?

**A:** The RSC payload is a special binary format that React uses to stream Server Component render results from server to client. It contains the rendered output of Server Components, placeholders for Client Components with their props, and references to Client Component JavaScript bundles. The client uses this payload to construct the component tree without needing the Server Component code. It's also used for client-side navigation -- when navigating, Next.js fetches the RSC payload instead of full HTML, enabling SPA-like transitions while keeping Server Components on the server.

### Q13: Can a Client Component import a Server Component?

**A:** No, a Client Component cannot directly import a Server Component because once you cross the `'use client'` boundary, everything imported is part of the client bundle. However, you CAN pass a Server Component to a Client Component as `children` or any React node prop. This is the "composition pattern": the parent Server Component renders both and passes the Server Component as children to the Client Component. This is a critical pattern for keeping server logic server-side while using client interactivity.

### Q14: What are the performance benefits of Server Components?

**A:** (1) Zero client JavaScript for non-interactive parts -- reduces bundle size significantly. (2) Direct backend access eliminates API roundtrips (no fetch waterfall from client). (3) Sensitive logic (DB queries, API keys) stays on the server. (4) Large dependencies used only for rendering (markdown parsers, syntax highlighters) never ship to the client. (5) Automatic code splitting at the component level. (6) Better streaming support since server controls the rendering. These benefits compound -- a typical page with 80% static content sends dramatically less JavaScript to the browser compared to a traditional React SPA.

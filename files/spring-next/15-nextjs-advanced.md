# Next.js Advanced Topics

## Table of Contents
1. [Parallel Routes and Intercepting Routes](#parallel-routes-and-intercepting-routes)
2. [Internationalization (i18n)](#internationalization-i18n)
3. [Testing](#testing)
4. [Middleware Patterns](#middleware-patterns)
5. [Edge Runtime vs Node.js Runtime](#edge-runtime-vs-nodejs-runtime)
6. [Monorepo Setup (Turborepo)](#monorepo-setup-turborepo)
7. [Next.js + Spring Boot Integration](#nextjs--spring-boot-integration)
8. [Interview Questions](#interview-questions)

---

## Parallel Routes and Intercepting Routes

### Parallel Routes

Parallel routes render multiple pages simultaneously in the same layout, in named "slots." Think of them as Angular's named `<router-outlet>` but more powerful.

```
Angular Named Outlets:                Next.js Parallel Routes:

<router-outlet></router-outlet>       {children}    (default slot)
<router-outlet name="sidebar">        {sidebar}     (@sidebar slot)
</router-outlet>                      {modal}       (@modal slot)
```

### Folder Structure for Parallel Routes

```
app/
  dashboard/
    layout.tsx          <-- Receives @analytics, @team, and children
    page.tsx            <-- Default children slot
    @analytics/
      page.tsx          <-- /dashboard shows analytics panel
      loading.tsx       <-- Independent loading state
      default.tsx       <-- Fallback when no match
    @team/
      page.tsx          <-- /dashboard shows team panel
      loading.tsx       <-- Independent loading state
      default.tsx
```

```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children,    // Default slot (page.tsx)
  analytics,   // @analytics slot
  team,        // @team slot
}: {
  children: React.ReactNode;
  analytics: React.ReactNode;
  team: React.ReactNode;
}) {
  return (
    <div>
      <div className="main">{children}</div>
      <div className="grid grid-cols-2 gap-4">
        <div>{analytics}</div>  {/* Loads independently */}
        <div>{team}</div>       {/* Loads independently */}
      </div>
    </div>
  );
}
```

### Benefits of Parallel Routes

```
Without Parallel Routes:          With Parallel Routes:
(Sequential loading)              (Independent loading)

+----------------------------+    +----------------------------+
| Dashboard Page             |    | Dashboard Page             |
|                            |    |                            |
| Loading everything...      |    | Main Content (loaded)      |
|                            |    |                            |
|                            |    | +----------+ +-----------+ |
|                            |    | |Analytics | |Team       | |
|                            |    | |Loading.. | |  Alice    | |
|                            |    | |          | |  Bob      | |
|                            |    | +----------+ +-----------+ |
+----------------------------+    +----------------------------+

Each slot has its own loading.tsx, error.tsx,
and can stream independently.
```

### Intercepting Routes

Intercepting routes let you load a route within the current layout while keeping the context. The classic use case: clicking a photo in a feed opens a modal, but the direct URL shows the full page.

```
Intercepting Convention:
(.) -- intercept same level
(..) -- intercept one level up
(..)(..) -- intercept two levels up
(...) -- intercept from root
```

```
Example: Photo Feed with Modal

app/
  feed/
    page.tsx               <-- Photo grid
    @modal/
      (..)photo/[id]/      <-- Intercepts /photo/[id] as modal
        page.tsx            <-- Shows photo in modal
      default.tsx           <-- No modal shown by default
    layout.tsx              <-- Receives {children} and {modal}
  photo/
    [id]/
      page.tsx              <-- Full photo page (direct URL)
```

```tsx
// app/feed/layout.tsx
export default function FeedLayout({
  children,
  modal,
}: {
  children: React.ReactNode;
  modal: React.ReactNode;
}) {
  return (
    <>
      {children}
      {modal}  {/* Modal overlay when intercepted */}
    </>
  );
}

// app/feed/@modal/(..)photo/[id]/page.tsx
// This renders when clicking a photo LINK from the feed
export default function PhotoModal({ params }: { params: { id: string } }) {
  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center">
      <div className="bg-white rounded-lg p-4">
        <img src={`/photos/${params.id}.jpg`} alt="Photo" />
      </div>
    </div>
  );
}

// app/photo/[id]/page.tsx
// This renders when navigating DIRECTLY to /photo/123
export default function PhotoPage({ params }: { params: { id: string } }) {
  return (
    <div className="full-page-photo">
      <img src={`/photos/${params.id}.jpg`} alt="Photo" />
      <div>Comments, details, etc.</div>
    </div>
  );
}
```

```
User clicks photo in feed:     User visits /photo/123 directly:

+------ Feed Page ------+       +---- Full Photo Page ----+
| Photo1  Photo2  Photo3|       |                         |
|                       |       |    [Large Photo]        |
| +------- Modal -----+ |       |                         |
| |                    | |       |    Comments:            |
| |   [Photo Detail]  | |       |    - "Great shot!"      |
| |                    | |       |    - "Beautiful"        |
| +--------------------+ |       |                         |
| Photo4  Photo5  Photo6|       +-------------------------+
+------------------------+
```

---

## Internationalization (i18n)

### next-intl (Most Popular i18n Library)

```
app/
  [locale]/               <-- Dynamic segment for language
    layout.tsx
    page.tsx
    about/
      page.tsx
  messages/
    en.json               <-- English translations
    es.json               <-- Spanish translations
    fr.json               <-- French translations
```

```json
// messages/en.json
{
  "Home": {
    "title": "Welcome to our site",
    "description": "The best platform for developers"
  },
  "Navigation": {
    "home": "Home",
    "about": "About",
    "contact": "Contact"
  }
}
```

```json
// messages/es.json
{
  "Home": {
    "title": "Bienvenido a nuestro sitio",
    "description": "La mejor plataforma para desarrolladores"
  },
  "Navigation": {
    "home": "Inicio",
    "about": "Acerca de",
    "contact": "Contacto"
  }
}
```

```tsx
// app/[locale]/page.tsx
import { useTranslations } from 'next-intl';

export default function HomePage() {
  const t = useTranslations('Home');

  return (
    <div>
      <h1>{t('title')}</h1>
      <p>{t('description')}</p>
    </div>
  );
}
```

```tsx
// middleware.ts -- detect and redirect to correct locale
import createMiddleware from 'next-intl/middleware';

export default createMiddleware({
  locales: ['en', 'es', 'fr'],
  defaultLocale: 'en',
});

export const config = {
  matcher: ['/((?!api|_next|.*\\..*).*)'],
};
```

### Angular i18n vs Next.js i18n

```
+-------------------+---------------------------+----------------------------+
| Feature           | Angular i18n              | Next.js (next-intl)        |
+-------------------+---------------------------+----------------------------+
| Approach          | Build-time (separate      | Runtime (single build,     |
|                   | build per locale)         | dynamic locale)            |
| Marker            | i18n attribute in HTML    | t('key') function          |
| Extraction        | xi18n CLI tool            | JSON files manually        |
| URL pattern       | /en/about, /es/about      | /en/about, /es/about       |
| Bundle            | One per locale            | One for all locales        |
| Switching locale  | Page reload (different    | Client navigation          |
|                   | build)                    | (same build)               |
| SSR support       | Complex                   | Built-in                   |
+-------------------+---------------------------+----------------------------+
```

---

## Testing

### Testing Strategy in Next.js

```
+--------------------------------------------------------+
|  Testing Pyramid for Next.js                           |
+--------------------------------------------------------+
|                                                        |
|               /\                                       |
|              /  \    E2E Tests (Playwright/Cypress)     |
|             /    \   - Full user flows                  |
|            /------\  - Navigation, forms, auth          |
|           /        \                                    |
|          / Integrat-\ Integration Tests                |
|         /  ion Tests \ - Server Components              |
|        /              \ - API routes                    |
|       /----------------\ - Server Actions               |
|      /                  \                               |
|     /    Unit Tests      \ Unit Tests (Jest + RTL)      |
|    /                      \ - Client Components          |
|   /                        \ - Hooks, utilities           |
|  /                          \ - Pure functions             |
| /----------------------------\                            |
+-----------------------------------------------------------+
```

### Jest + React Testing Library Setup

```tsx
// jest.config.ts
import type { Config } from 'jest';
import nextJest from 'next/jest';

const createJestConfig = nextJest({
  dir: './', // Path to Next.js app
});

const config: Config = {
  coverageProvider: 'v8',
  testEnvironment: 'jsdom',
  setupFilesAfterSetup: ['<rootDir>/jest.setup.ts'],
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/$1', // Path aliases
  },
};

export default createJestConfig(config);
```

```tsx
// jest.setup.ts
import '@testing-library/jest-dom';
```

### Unit Testing Client Components

```tsx
// components/Counter.tsx
'use client';
import { useState } from 'react';

export default function Counter({ initialCount = 0 }: { initialCount?: number }) {
  const [count, setCount] = useState(initialCount);

  return (
    <div>
      <p data-testid="count">Count: {count}</p>
      <button onClick={() => setCount(c => c + 1)}>Increment</button>
      <button onClick={() => setCount(c => c - 1)}>Decrement</button>
      <button onClick={() => setCount(0)}>Reset</button>
    </div>
  );
}
```

```tsx
// __tests__/Counter.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import Counter from '@/components/Counter';

describe('Counter', () => {
  it('renders with initial count', () => {
    render(<Counter initialCount={5} />);
    expect(screen.getByTestId('count')).toHaveTextContent('Count: 5');
  });

  it('increments count', () => {
    render(<Counter />);
    fireEvent.click(screen.getByText('Increment'));
    expect(screen.getByTestId('count')).toHaveTextContent('Count: 1');
  });

  it('decrements count', () => {
    render(<Counter initialCount={5} />);
    fireEvent.click(screen.getByText('Decrement'));
    expect(screen.getByTestId('count')).toHaveTextContent('Count: 4');
  });

  it('resets count', () => {
    render(<Counter initialCount={10} />);
    fireEvent.click(screen.getByText('Increment'));
    fireEvent.click(screen.getByText('Reset'));
    expect(screen.getByTestId('count')).toHaveTextContent('Count: 0');
  });
});
```

### Testing Async Components (Server Components)

```tsx
// Testing server components requires rendering them as async
// __tests__/ProductList.test.tsx
import { render, screen } from '@testing-library/react';
import ProductList from '@/app/products/page';

// Mock the fetch
global.fetch = jest.fn(() =>
  Promise.resolve({
    ok: true,
    json: () =>
      Promise.resolve([
        { id: '1', name: 'Widget', price: 9.99 },
        { id: '2', name: 'Gadget', price: 19.99 },
      ]),
  })
) as jest.Mock;

describe('ProductList', () => {
  it('renders products from API', async () => {
    const Component = await ProductList();
    render(Component);

    expect(screen.getByText('Widget')).toBeInTheDocument();
    expect(screen.getByText('Gadget')).toBeInTheDocument();
  });
});
```

### E2E Testing with Playwright

```tsx
// e2e/home.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Home Page', () => {
  test('should display the heading', async ({ page }) => {
    await page.goto('/');
    await expect(page.locator('h1')).toContainText('Welcome');
  });

  test('should navigate to about page', async ({ page }) => {
    await page.goto('/');
    await page.click('text=About');
    await expect(page).toHaveURL('/about');
    await expect(page.locator('h1')).toContainText('About Us');
  });

  test('should submit contact form', async ({ page }) => {
    await page.goto('/contact');
    await page.fill('input[name="name"]', 'John Doe');
    await page.fill('input[name="email"]', 'john@example.com');
    await page.fill('textarea[name="message"]', 'Hello there!');
    await page.click('button[type="submit"]');
    await expect(page.locator('.success-message')).toBeVisible();
  });
});
```

### Angular Testing vs React/Next.js Testing

```
+-------------------+---------------------------+----------------------------+
| Aspect            | Angular (Jasmine/Karma)   | Next.js (Jest/RTL)         |
+-------------------+---------------------------+----------------------------+
| Test runner       | Karma (browser-based)     | Jest (Node.js, jsdom)      |
|                   | or Jest                   |                            |
| Assertion         | Jasmine expect            | Jest expect + jest-dom     |
| Component render  | TestBed.createComponent() | render(<Component />)      |
| Query elements    | fixture.debugElement      | screen.getByText(),        |
|                   | .query(By.css('...'))     | screen.getByRole(), etc.   |
| User events       | triggerEventHandler()     | fireEvent / userEvent      |
| Async             | fakeAsync + tick          | waitFor + findBy queries   |
| DI mocking        | TestBed.overrideProvider  | Jest mocks + imports       |
| HTTP mocking      | HttpTestingController     | jest.fn() on fetch/axios   |
| E2E               | Protractor (deprecated),  | Playwright (recommended)   |
|                   | Cypress                   | or Cypress                 |
| Component testing | TestBed setup boilerplate | Simple render() call       |
| Philosophy        | Test implementation       | Test behavior (user-centric)|
+-------------------+---------------------------+----------------------------+

Key shift: Angular tests often test internal state and method calls.
React Testing Library philosophy: "Test what the user sees and does."
Query by visible text, roles, labels -- not by CSS class or component internals.
```

---

## Middleware Patterns

### Rate Limiting

```tsx
// middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

const rateLimitMap = new Map<string, { count: number; timestamp: number }>();

function rateLimit(ip: string, limit: number = 100, windowMs: number = 60000): boolean {
  const now = Date.now();
  const record = rateLimitMap.get(ip);

  if (!record || now - record.timestamp > windowMs) {
    rateLimitMap.set(ip, { count: 1, timestamp: now });
    return false; // Not limited
  }

  if (record.count >= limit) {
    return true; // Rate limited
  }

  record.count++;
  return false;
}

export function middleware(request: NextRequest) {
  const ip = request.headers.get('x-forwarded-for') ?? 'unknown';

  if (request.nextUrl.pathname.startsWith('/api/')) {
    if (rateLimit(ip)) {
      return NextResponse.json(
        { error: 'Too many requests' },
        { status: 429 }
      );
    }
  }

  return NextResponse.next();
}
```

### A/B Testing

```tsx
// middleware.ts
export function middleware(request: NextRequest) {
  const bucket = request.cookies.get('ab-bucket')?.value;

  if (!bucket) {
    // Assign user to a bucket
    const newBucket = Math.random() < 0.5 ? 'control' : 'variant';
    const response = NextResponse.next();
    response.cookies.set('ab-bucket', newBucket, { maxAge: 60 * 60 * 24 * 30 });
    return response;
  }

  // Rewrite to different page version
  if (request.nextUrl.pathname === '/pricing') {
    if (bucket === 'variant') {
      return NextResponse.rewrite(new URL('/pricing-v2', request.url));
    }
  }

  return NextResponse.next();
}
```

### Geolocation-Based Routing

```tsx
// middleware.ts
export function middleware(request: NextRequest) {
  const country = request.geo?.country ?? 'US';
  const pathname = request.nextUrl.pathname;

  // Redirect to country-specific content
  if (pathname === '/' && country === 'DE') {
    return NextResponse.redirect(new URL('/de', request.url));
  }

  // Add country header for downstream use
  const response = NextResponse.next();
  response.headers.set('x-user-country', country);
  return response;
}
```

---

## Edge Runtime vs Node.js Runtime

```
+---------------------+----------------------------------+----------------------------+
| Feature             | Edge Runtime                     | Node.js Runtime            |
+---------------------+----------------------------------+----------------------------+
| Startup time        | ~0ms (cold start)                | ~250ms+ (cold start)       |
| Location            | CDN edge (close to users)        | Origin server              |
| Max execution       | 30s (Vercel)                     | 300s (Vercel), unlimited   |
| Memory              | 128MB (Vercel)                   | 1-3GB+                     |
| APIs available      | Web APIs subset                  | Full Node.js               |
|                     | (fetch, crypto.subtle,           | (fs, path, child_process,  |
|                     |  TextEncoder, URL, etc.)         |  native modules, etc.)     |
| Database            | HTTP-based only (Prisma Edge,    | Any (Prisma, Drizzle,      |
|                     |  Planetscale, Neon)              |  direct TCP connections)   |
| npm packages        | Limited (no native deps)         | All packages               |
| Best for            | Auth, redirects, A/B tests,      | Database ops, file ops,    |
|                     | geolocation, simple transforms   | heavy computation, SSR     |
+---------------------+----------------------------------+----------------------------+
```

```tsx
// Opt into Edge Runtime for a Route Handler
// app/api/hello/route.ts
export const runtime = 'edge'; // Use Edge Runtime

export async function GET() {
  return Response.json({ hello: 'world' });
}

// Opt into Edge Runtime for a page
// app/fast-page/page.tsx
export const runtime = 'edge';

export default function FastPage() {
  return <div>This page renders on the edge!</div>;
}
```

---

## Monorepo Setup (Turborepo)

Turborepo (by Vercel) is a build system for JavaScript monorepos. It's the recommended way to manage large Next.js projects.

```
my-monorepo/
|
+-- apps/
|   +-- web/                     # Next.js frontend
|   |   +-- app/
|   |   +-- package.json
|   |   +-- next.config.js
|   |
|   +-- admin/                   # Next.js admin panel
|   |   +-- app/
|   |   +-- package.json
|   |
|   +-- docs/                    # Documentation site
|       +-- app/
|       +-- package.json
|
+-- packages/
|   +-- ui/                      # Shared UI components
|   |   +-- src/
|   |   |   +-- Button.tsx
|   |   |   +-- Card.tsx
|   |   +-- package.json
|   |
|   +-- config/                  # Shared configs
|   |   +-- eslint/
|   |   +-- tsconfig/
|   |   +-- tailwind/
|   |
|   +-- database/                # Shared database schema/client
|   |   +-- prisma/
|   |   +-- src/client.ts
|   |   +-- package.json
|   |
|   +-- types/                   # Shared TypeScript types
|       +-- src/
|       +-- package.json
|
+-- turbo.json                   # Turborepo config
+-- package.json                 # Root package.json
+-- pnpm-workspace.yaml          # Workspace config
```

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {},
    "test": {}
  }
}
```

```bash
# Commands
npx turbo dev        # Start all apps in dev mode
npx turbo build      # Build all apps (parallelized + cached)
npx turbo lint       # Lint all packages
npx turbo test       # Test all packages

# Turborepo caches build outputs
# Second build is MUCH faster (only rebuilds changed packages)
```

---

## Next.js + Spring Boot Integration

### Architecture Overview

```
+=====================================================================+
|                Next.js + Spring Boot Architecture                    |
+=====================================================================+

                         Internet
                            |
                            v
                    +---------------+
                    |  CDN / Edge   |   Static assets, cached pages
                    |  (Vercel/CF)  |
                    +-------+-------+
                            |
                            v
          +-----------------+-----------------+
          |                                   |
          v                                   v
  +-------+--------+                 +--------+--------+
  |   Next.js      |    REST/gRPC   |  Spring Boot    |
  |   Server       |<-------------->|  API Server     |
  |                |                 |                 |
  | - SSR/SSG/ISR  |   Fetch from   | - Business logic|
  | - React UI     |   Server       | - Data access   |
  | - Middleware    |   Components   | - Auth (JWT)    |
  | - Server       |                | - Validation    |
  |   Actions      |   Route        | - File storage  |
  | - BFF (Backend |   Handlers     | - Email service |
  |   for Frontend)|   as proxy     | - Scheduled jobs|
  +-------+--------+                 +--------+--------+
          |                                   |
          |                                   |
          v                                   v
  +-------+--------+                 +--------+--------+
  |   Browser      |                 |  PostgreSQL /   |
  |   (Client)     |                 |  MySQL / Redis  |
  |                |                 |                 |
  | - Hydrated UI  |                 | - Data storage  |
  | - Client state |                 | - Caching       |
  +----------------+                 +-----------------+


  Data Flow:
  ==========

  1. Browser requests /products
  2. Next.js Server Component fetches from Spring Boot API
  3. Spring Boot queries database, returns JSON
  4. Next.js renders HTML with data, sends to browser
  5. Browser hydrates interactive parts

  For mutations:
  1. User submits form in browser
  2. Server Action runs on Next.js server
  3. Server Action calls Spring Boot API
  4. Spring Boot validates + saves to database
  5. Next.js revalidates cache, sends updated HTML
```

### API Communication Pattern

```tsx
// lib/api.ts -- Spring Boot API client
const API_BASE = process.env.API_URL; // e.g., http://localhost:8080

class ApiClient {
  private async request<T>(
    endpoint: string,
    options: RequestInit = {}
  ): Promise<T> {
    const url = `${API_BASE}${endpoint}`;

    const res = await fetch(url, {
      ...options,
      headers: {
        'Content-Type': 'application/json',
        ...options.headers,
      },
    });

    if (!res.ok) {
      const error = await res.json().catch(() => ({}));
      throw new ApiError(res.status, error.message || 'API Error');
    }

    return res.json();
  }

  // Products
  async getProducts(params?: { page?: number; size?: number }) {
    const query = new URLSearchParams(params as any).toString();
    return this.request<PaginatedResponse<Product>>(
      `/api/products?${query}`,
      { next: { tags: ['products'], revalidate: 300 } }
    );
  }

  async getProduct(id: string) {
    return this.request<Product>(`/api/products/${id}`, {
      next: { tags: [`product-${id}`] },
    });
  }

  async createProduct(data: CreateProductDTO, token: string) {
    return this.request<Product>('/api/products', {
      method: 'POST',
      headers: { Authorization: `Bearer ${token}` },
      body: JSON.stringify(data),
    });
  }

  // Auth
  async login(credentials: LoginDTO) {
    return this.request<AuthResponse>('/api/auth/login', {
      method: 'POST',
      body: JSON.stringify(credentials),
    });
  }
}

export const api = new ApiClient();
```

```tsx
// app/products/page.tsx -- Using the API client
import { api } from '@/lib/api';

export default async function ProductsPage({
  searchParams,
}: {
  searchParams: Promise<{ page?: string }>;
}) {
  const { page } = await searchParams;
  const products = await api.getProducts({
    page: parseInt(page ?? '1'),
    size: 20,
  });

  return (
    <div>
      <h1>Products</h1>
      {products.content.map(product => (
        <ProductCard key={product.id} product={product} />
      ))}
      <Pagination totalPages={products.totalPages} />
    </div>
  );
}
```

### Spring Boot CORS Configuration

```java
// Spring Boot: CorsConfig.java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "http://localhost:3000",  // Next.js dev
                "https://myapp.vercel.app" // Next.js production
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

### BFF (Backend for Frontend) Pattern

```
  Without BFF:                      With Next.js as BFF:

  Browser --> Spring Boot           Browser --> Next.js --> Spring Boot
  (direct API calls,                (Next.js aggregates, transforms,
   CORS issues, multiple            caches, and serves what the
   round-trips, leaked              frontend actually needs)
   API structure)

  BFF benefits:
  - No CORS issues (same-origin)
  - API aggregation (combine multiple backend calls)
  - Response shaping (only send what frontend needs)
  - Caching layer
  - Auth token handling (httpOnly cookies)
  - Rate limiting at the BFF level
```

```tsx
// app/api/dashboard/route.ts -- BFF example
// Aggregates multiple Spring Boot API calls into one response

export async function GET() {
  const token = await getAccessToken();

  // Parallel calls to Spring Boot
  const [stats, recentOrders, topProducts] = await Promise.all([
    fetch(`${API_URL}/api/stats`, { headers: { Authorization: `Bearer ${token}` } }),
    fetch(`${API_URL}/api/orders?limit=5`, { headers: { Authorization: `Bearer ${token}` } }),
    fetch(`${API_URL}/api/products/top?limit=10`, { headers: { Authorization: `Bearer ${token}` } }),
  ]);

  return Response.json({
    stats: await stats.json(),
    recentOrders: await recentOrders.json(),
    topProducts: await topProducts.json(),
  });
}
```

---

## Interview Questions

### Q1: What are parallel routes and when would you use them?

**A:** Parallel routes render multiple pages simultaneously in the same layout using named slots (e.g., `@analytics`, `@team`). Each slot has its own `loading.tsx` and `error.tsx`, so they load and fail independently. Use cases: (1) Dashboards with multiple independently-loading sections. (2) Modals that overlay the current page (using intercepting routes). (3) Split-view layouts where different panes show different routes. They're defined by `@folder` convention and received as props in the parent layout.

### Q2: How do intercepting routes work?

**A:** Intercepting routes let you load a route within the current layout context while preserving the background page. Convention: `(.)` for same level, `(..)` for parent level, `(...)` for root. Classic example: clicking a photo in a feed shows a modal (intercepted route), but navigating directly to `/photo/123` shows the full page. The soft navigation (Link click) triggers the intercepted version; hard navigation (direct URL, refresh) loads the full page. Combined with parallel routes (`@modal` slot) for modal patterns.

### Q3: How would you implement i18n in Next.js?

**A:** Use `next-intl` or similar library with the App Router. Structure: `app/[locale]/` for locale-based routing, `messages/en.json` for translations. Middleware detects locale from Accept-Language header or cookies and redirects. Components use `useTranslations('namespace')` hook. Unlike Angular's i18n (separate build per locale), Next.js i18n is runtime -- one build serves all locales. For SSG, use `generateStaticParams` to pre-render all locale variants. URL structure: `/en/about`, `/es/about`.

### Q4: Compare testing approaches between Angular and Next.js.

**A:** Angular uses TestBed for component testing with dependency injection mocking, Jasmine/Karma as default runner, and tests implementation details. Next.js uses Jest + React Testing Library (RTL) which focuses on testing user behavior -- query by role, text, label rather than CSS selectors. RTL philosophy: "The more your tests resemble the way software is used, the more confidence they give you." Server Components are tested by rendering them (async) and checking output. E2E: both can use Playwright; Angular historically used Protractor (deprecated). Next.js testing is generally less boilerplate.

### Q5: What are the differences between Edge Runtime and Node.js Runtime?

**A:** Edge Runtime runs on CDN edge nodes with near-zero cold starts but limited APIs (Web APIs only, no `fs`, no native modules, limited memory). Node.js Runtime runs on origin servers with full Node.js API access but slower cold starts. Edge is ideal for middleware (auth, redirects, geolocation), simple API responses, and latency-sensitive operations. Node.js is needed for database connections (TCP), file system access, heavy computation, and packages with native dependencies. Choose based on the operation's requirements and where it benefits most from running close to the user.

### Q6: What is Turborepo and how does it help with Next.js projects?

**A:** Turborepo is a build system for JavaScript/TypeScript monorepos that optimizes task execution. Benefits: (1) Caching -- caches build outputs locally and remotely, so unchanged packages aren't rebuilt. (2) Parallelism -- runs independent tasks simultaneously. (3) Dependency graph -- understands package dependencies and builds in correct order. Structure: `apps/` for deployable applications (Next.js, admin panels), `packages/` for shared code (UI components, types, configs). In a Next.js + Spring Boot project, you'd have `apps/web` (Next.js) and `packages/ui` (shared components).

### Q7: How would you integrate Next.js with a Spring Boot backend?

**A:** Architecture: Next.js as the frontend (SSR/SSG) + BFF (Backend for Frontend), Spring Boot as the core API. Communication: Server Components fetch from Spring Boot REST API using fetch with caching. Auth: Spring Boot issues JWTs, Next.js stores them in httpOnly cookies, middleware/Server Components attach to API calls. CORS: Configure Spring Boot to allow Next.js origins. Server Actions call Spring Boot for mutations and revalidate Next.js cache. Docker Compose runs both services together. Next.js acts as BFF, aggregating multiple Spring Boot calls and shaping responses for the frontend.

### Q8: What is the BFF (Backend for Frontend) pattern and how does Next.js implement it?

**A:** BFF is a pattern where a dedicated backend serves a specific frontend's needs. Next.js naturally acts as a BFF via Route Handlers and Server Components. Instead of the browser calling Spring Boot directly (CORS issues, multiple round-trips, leaked API structure), the browser calls Next.js, which aggregates multiple backend calls, transforms responses, handles auth, and caches results. Benefits: no CORS issues (same-origin), API aggregation (combine multiple calls into one), response shaping (only send what the UI needs), and a caching layer between frontend and backend.

### Q9: How would you test Server Actions?

**A:** Test Server Actions by: (1) **Unit testing the function** -- call it directly with mock FormData, verify it calls the right database operations and returns expected results. Mock the database client and `revalidatePath`/`revalidateTag`. (2) **Integration testing** -- render the form component, fill in fields, submit, and verify the action was called with correct data. (3) **E2E testing** -- use Playwright to fill in and submit the form in a real browser, verify the page updates. Most teams focus on E2E tests for Server Actions since they involve both client and server.

### Q10: How do you handle rate limiting in Next.js?

**A:** For basic rate limiting, implement in middleware using an in-memory Map (for single-server) or Redis (for distributed). Track request count per IP within a time window. For production, use: (1) Vercel's built-in rate limiting (Edge Config). (2) `@upstash/ratelimit` with Redis for distributed rate limiting. (3) Cloudflare rate limiting at CDN level. Apply different limits to different paths (stricter for auth endpoints, lenient for static pages). Return 429 status with `Retry-After` header when limit is hit.

### Q11: How would you implement A/B testing in Next.js?

**A:** Use middleware for A/B testing: (1) Check if user has an existing bucket cookie. (2) If not, assign randomly and set cookie. (3) Use `NextResponse.rewrite()` to serve different page variants without changing the URL. The user always sees `/pricing` but middleware rewrites to `/pricing-v2` for the variant group. For analytics, pass the bucket value as a header or cookie to the client for tracking. Libraries like `@vercel/edge-config` or Statsig provide more sophisticated feature flagging with server-side configuration.

### Q12: What is the recommended testing strategy for a Next.js application?

**A:** Follow the testing pyramid: (1) **Unit tests (most):** Test utility functions, custom hooks (with `renderHook`), and isolated client components with Jest + React Testing Library. (2) **Integration tests (some):** Test page components that fetch data, Server Actions with mocked APIs, and Route Handlers with mock requests. (3) **E2E tests (few, critical paths):** Test complete user flows (auth, checkout, navigation) with Playwright. Key principle: test behavior, not implementation. For Server Components, test the rendered output. For Client Components, test user interactions. Aim for 80%+ coverage on business logic.

### Q13: How do parallel routes handle errors independently?

**A:** Each parallel route slot (`@analytics`, `@team`) can have its own `error.tsx`. If the analytics data fetch fails, only the analytics slot shows an error -- the team slot and main content continue working normally. This is a significant improvement over traditional SPAs where one failed API call can break the entire page. The error boundary for each slot includes a `reset` function to retry just that section. This granular error handling is similar to having independent error boundaries per micro-frontend.

### Q14: How do you handle WebSocket connections in a Next.js application?

**A:** Next.js doesn't natively support persistent WebSocket connections in Route Handlers (they're designed for request-response). Options: (1) **Separate WebSocket server** -- run a standalone Node.js/Spring Boot WebSocket server alongside Next.js. Client components connect directly. (2) **Socket.io with custom server** -- create a custom Next.js server (`server.ts`) that adds Socket.io. (3) **Third-party services** -- use Pusher, Ably, or Supabase Realtime. (4) **Server-Sent Events** -- Route Handlers can stream SSE for one-way real-time updates. For most cases, SSE or a third-party service is simpler than managing WebSocket infrastructure.

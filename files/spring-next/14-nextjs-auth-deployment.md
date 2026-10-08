# Next.js Authentication and Deployment

## Table of Contents
1. [NextAuth.js / Auth.js](#nextauthjs--authjs)
2. [Middleware](#middleware)
3. [Protected Routes Pattern](#protected-routes-pattern)
4. [JWT Handling](#jwt-handling)
5. [Deployment](#deployment)
6. [Environment Variables](#environment-variables)
7. [Performance Optimization](#performance-optimization)
8. [SEO Best Practices](#seo-best-practices)
9. [Interview Questions](#interview-questions)

---

## NextAuth.js / Auth.js

NextAuth.js (now called Auth.js) is the most popular authentication library for Next.js. It handles OAuth, credentials, JWT, sessions, and more.

### Angular Auth vs Next.js Auth

```
Angular Auth Flow:                        Next.js Auth Flow:
+--------------------------+              +--------------------------+
| Login Component          |              | Login Page               |
| -> AuthService.login()   |              | -> signIn('provider')    |
| -> HttpClient.post()     |              | -> NextAuth handles      |
| -> Store token in memory |              | -> Cookie set auto       |
| -> AuthInterceptor adds  |              | -> Middleware validates   |
|    token to all requests |              |    on every request      |
| -> AuthGuard checks      |              | -> getServerSession()    |
|    route access          |              |    in Server Components  |
+--------------------------+              +--------------------------+
```

### Setting Up NextAuth.js (App Router)

```tsx
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import { authOptions } from '@/lib/auth';

const handler = NextAuth(authOptions);
export { handler as GET, handler as POST };
```

```tsx
// lib/auth.ts
import { NextAuthOptions } from 'next-auth';
import CredentialsProvider from 'next-auth/providers/credentials';
import GoogleProvider from 'next-auth/providers/google';
import GitHubProvider from 'next-auth/providers/github';

export const authOptions: NextAuthOptions = {
  providers: [
    // OAuth Providers
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),

    GitHubProvider({
      clientId: process.env.GITHUB_ID!,
      clientSecret: process.env.GITHUB_SECRET!,
    }),

    // Credentials (username/password) -- like Angular login form
    CredentialsProvider({
      name: 'credentials',
      credentials: {
        email: { label: 'Email', type: 'email' },
        password: { label: 'Password', type: 'password' },
      },
      async authorize(credentials) {
        // Validate against your database or Spring Boot backend
        const res = await fetch(`${process.env.API_URL}/auth/login`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            email: credentials?.email,
            password: credentials?.password,
          }),
        });

        const user = await res.json();

        if (res.ok && user) {
          return {
            id: user.id,
            name: user.name,
            email: user.email,
            accessToken: user.token,
          };
        }

        return null; // Return null = authentication failed
      },
    }),
  ],

  callbacks: {
    // Add custom data to the JWT token
    async jwt({ token, user }) {
      if (user) {
        token.accessToken = user.accessToken;
        token.role = user.role;
      }
      return token;
    },

    // Add custom data to the session
    async session({ session, token }) {
      session.accessToken = token.accessToken as string;
      session.user.role = token.role as string;
      return session;
    },
  },

  pages: {
    signIn: '/login',       // Custom login page
    signOut: '/logout',
    error: '/auth/error',
    newUser: '/onboarding',
  },

  session: {
    strategy: 'jwt',        // JWT (stateless) or 'database' (stateful)
    maxAge: 30 * 24 * 60 * 60, // 30 days
  },
};
```

### Session Provider (Client-Side Access)

```tsx
// app/providers.tsx
'use client';

import { SessionProvider } from 'next-auth/react';
import { ReactNode } from 'react';

export default function Providers({ children }: { children: ReactNode }) {
  return <SessionProvider>{children}</SessionProvider>;
}

// app/layout.tsx
import Providers from './providers';

export default function RootLayout({ children }: { children: ReactNode }) {
  return (
    <html>
      <body>
        <Providers>{children}</Providers>
      </body>
    </html>
  );
}
```

### Using Auth in Components

```tsx
// SERVER Component -- use getServerSession
import { getServerSession } from 'next-auth';
import { authOptions } from '@/lib/auth';

export default async function DashboardPage() {
  const session = await getServerSession(authOptions);

  if (!session) {
    redirect('/login');
  }

  return <div>Welcome, {session.user.name}!</div>;
}

// CLIENT Component -- use useSession
'use client';
import { useSession, signIn, signOut } from 'next-auth/react';

export default function AuthButton() {
  const { data: session, status } = useSession();

  if (status === 'loading') return <p>Loading...</p>;

  if (session) {
    return (
      <div>
        <p>Signed in as {session.user?.email}</p>
        <button onClick={() => signOut()}>Sign Out</button>
      </div>
    );
  }

  return (
    <div>
      <button onClick={() => signIn('google')}>Sign in with Google</button>
      <button onClick={() => signIn('github')}>Sign in with GitHub</button>
      <button onClick={() => signIn('credentials')}>Sign in with Email</button>
    </div>
  );
}
```

---

## Middleware

Next.js middleware runs BEFORE a request is completed. It's like Angular's `HttpInterceptor` + route guards combined, but runs on the server (Edge runtime).

### Angular Guards/Interceptors vs Next.js Middleware

```
Angular:                               Next.js:
+---------------------------+          +----------------------------+
| AuthGuard (canActivate)   |          | middleware.ts               |
| - Checks auth state       |  --->    | - Checks cookies/tokens    |
| - Redirects if not authed |          | - Redirects if not authed  |
+---------------------------+          +----------------------------+
| AuthInterceptor           |          | (Also in middleware.ts)     |
| - Adds token to headers   |  --->    | - Modify request headers   |
| - Handles 401 responses   |          | - Rewrite/redirect         |
+---------------------------+          +----------------------------+
```

### Basic Middleware

```tsx
// middleware.ts (at project ROOT, not inside app/)
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;

  // Get auth token from cookies
  const token = request.cookies.get('auth-token')?.value;

  // Protected routes
  const protectedPaths = ['/dashboard', '/profile', '/settings'];
  const isProtected = protectedPaths.some(path => pathname.startsWith(path));

  if (isProtected && !token) {
    // Redirect to login (like Angular AuthGuard returning UrlTree)
    const loginUrl = new URL('/login', request.url);
    loginUrl.searchParams.set('callbackUrl', pathname);
    return NextResponse.redirect(loginUrl);
  }

  // Already logged in? Redirect away from login page
  if (pathname === '/login' && token) {
    return NextResponse.redirect(new URL('/dashboard', request.url));
  }

  // Add custom headers (like Angular HttpInterceptor)
  const response = NextResponse.next();
  response.headers.set('x-custom-header', 'my-value');

  return response;
}

// Configure which paths middleware runs on
export const config = {
  matcher: [
    // Match all paths except static files and API
    '/((?!api|_next/static|_next/image|favicon.ico).*)',
  ],
};
```

### Middleware with NextAuth

```tsx
// middleware.ts
import { withAuth } from 'next-auth/middleware';

export default withAuth(
  function middleware(req) {
    // This runs after auth check passes
    const { pathname } = req.nextUrl;
    const token = req.nextauth.token;

    // Role-based access (like Angular guard with role check)
    if (pathname.startsWith('/admin') && token?.role !== 'admin') {
      return Response.redirect(new URL('/unauthorized', req.url));
    }
  },
  {
    callbacks: {
      authorized: ({ token }) => !!token, // Must have valid token
    },
  }
);

export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*', '/profile/:path*'],
};
```

### Middleware Capabilities

```
What middleware CAN do:               What middleware CANNOT do:
- Read/modify cookies                 - Access databases directly
- Read/modify headers                 - Use Node.js APIs (fs, crypto)
- Redirect requests                   - Use large npm packages
- Rewrite URLs                        - Access React state
- Return responses                    - Run long computations
- Rate limiting                       (Runs on Edge Runtime with
- Geolocation-based routing            limited API surface)
- A/B testing
- Bot detection
- Logging
```

---

## Protected Routes Pattern

### Pattern 1: Middleware-Level Protection (Recommended)

```tsx
// middleware.ts -- protects ALL routes matching the config
export { default } from 'next-auth/middleware';

export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*'],
};
```

### Pattern 2: Page-Level Protection

```tsx
// app/dashboard/page.tsx
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';
import { authOptions } from '@/lib/auth';

export default async function DashboardPage() {
  const session = await getServerSession(authOptions);

  if (!session) {
    redirect('/login');
  }

  // Only reaches here if authenticated
  return <Dashboard user={session.user} />;
}
```

### Pattern 3: Layout-Level Protection (Protects All Children)

```tsx
// app/(protected)/layout.tsx
import { getServerSession } from 'next-auth';
import { redirect } from 'next/navigation';
import { authOptions } from '@/lib/auth';

export default async function ProtectedLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  const session = await getServerSession(authOptions);

  if (!session) {
    redirect('/login');
  }

  return (
    <div>
      <nav>Welcome, {session.user.name}</nav>
      {children}
    </div>
  );
}
```

### Protection Strategy Comparison

```
+--------------------+-------------+------------------+------------------+
| Strategy           | Middleware   | Page-level       | Layout-level     |
+--------------------+-------------+------------------+------------------+
| Runs where         | Edge (fast) | Server           | Server           |
| Protects           | URL pattern | Single page      | All child pages  |
| Can access DB      | No          | Yes              | Yes              |
| Shows loading      | No (instant)| Yes (loading.tsx)| Yes (loading.tsx)|
| Angular equivalent | AuthGuard   | ngOnInit check   | Parent guard     |
| Best for           | Simple auth | Role-based logic | Section auth     |
+--------------------+-------------+------------------+------------------+
```

---

## JWT Handling

### JWT Flow with Spring Boot Backend

```
  Login Flow:
  ==========
  Browser           Next.js Server          Spring Boot API
  |                 |                       |
  | POST /login     |                       |
  |---------------->| POST /api/auth/login  |
  |                 |---------------------->|
  |                 |                       | Validate credentials
  |                 |   { accessToken,      | Generate JWT
  |                 |     refreshToken }    |
  |                 |<----------------------|
  |                 |                       |
  |  Set httpOnly   | Store in cookies:     |
  |  cookies        | - access_token        |
  |<----------------|  (httpOnly, secure)   |
  |                 | - refresh_token       |
  |                 |  (httpOnly, secure)   |
  |                 |                       |

  Authenticated Request Flow:
  ===========================
  Browser           Next.js Server          Spring Boot API
  |                 |                       |
  | GET /dashboard  |                       |
  | (cookies sent   |                       |
  |  automatically) |                       |
  |---------------->| Read access_token     |
  |                 | from cookie           |
  |                 |                       |
  |                 | GET /api/data         |
  |                 | Authorization: Bearer |
  |                 |   <access_token>      |
  |                 |---------------------->|
  |                 |                       | Validate JWT
  |                 |  { data }             | Return data
  |                 |<----------------------|
  |  HTML response  |                       |
  |<----------------|                       |
```

### JWT Implementation

```tsx
// lib/auth-utils.ts
import { cookies } from 'next/headers';
import { jwtDecode } from 'jwt-decode';

interface TokenPayload {
  sub: string;
  email: string;
  role: string;
  exp: number;
}

export async function getAccessToken(): Promise<string | null> {
  const cookieStore = await cookies();
  return cookieStore.get('access_token')?.value ?? null;
}

export async function isTokenExpired(): Promise<boolean> {
  const token = await getAccessToken();
  if (!token) return true;

  const decoded = jwtDecode<TokenPayload>(token);
  return decoded.exp * 1000 < Date.now();
}

export async function refreshAccessToken(): Promise<string | null> {
  const cookieStore = await cookies();
  const refreshToken = cookieStore.get('refresh_token')?.value;

  if (!refreshToken) return null;

  const res = await fetch(`${process.env.API_URL}/auth/refresh`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ refreshToken }),
  });

  if (!res.ok) return null;

  const { accessToken } = await res.json();
  // Set new cookie via Server Action or Route Handler
  return accessToken;
}

// Authenticated fetch wrapper
export async function authFetch(url: string, options: RequestInit = {}) {
  let token = await getAccessToken();

  if (!token || (await isTokenExpired())) {
    token = await refreshAccessToken();
    if (!token) throw new Error('Authentication required');
  }

  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      Authorization: `Bearer ${token}`,
    },
  });
}
```

---

## Deployment

### Deployment Options

```
+------------------+----------------------------------------------------------+
| Platform         | Description                                              |
+------------------+----------------------------------------------------------+
| Vercel           | Built by Next.js creators. Zero-config, best DX.        |
|                  | Auto-deploys from Git. Edge network. Free tier.          |
+------------------+----------------------------------------------------------+
| Self-hosted      | node server.js on your own infrastructure.               |
| (Node.js)        | Full control. Need to handle scaling, CDN, etc.          |
+------------------+----------------------------------------------------------+
| Docker           | Containerized deployment. Works with any container       |
|                  | orchestrator (K8s, ECS, Cloud Run).                      |
+------------------+----------------------------------------------------------+
| Static Export    | next export -- pure static HTML. No server features.     |
|                  | Host on any CDN (S3, CloudFront, Nginx).                 |
+------------------+----------------------------------------------------------+
```

### Docker Deployment

```dockerfile
# Dockerfile (multi-stage build)
# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Stage 2: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Stage 3: Production
FROM node:20-alpine AS runner
WORKDIR /app

ENV NODE_ENV production

# Create non-root user
RUN addgroup --system --gid 1001 nodejs
RUN adduser --system --uid 1001 nextjs

# Copy built assets
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

USER nextjs

EXPOSE 3000
ENV PORT 3000
ENV HOSTNAME "0.0.0.0"

CMD ["node", "server.js"]
```

```javascript
// next.config.js -- enable standalone output for Docker
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: 'standalone', // Creates a minimal node server
};

module.exports = nextConfig;
```

### Vercel Deployment

```bash
# Install Vercel CLI
npm i -g vercel

# Deploy (that's it!)
vercel

# Deploy to production
vercel --prod

# Or just push to Git -- Vercel auto-deploys
```

---

## Environment Variables

### Environment Variable Conventions

```
+-----------------------------+---------------------------------------+
| File                        | Purpose                               |
+-----------------------------+---------------------------------------+
| .env                        | All environments (committed)          |
| .env.local                  | Local overrides (NOT committed)       |
| .env.development            | Development only                      |
| .env.production             | Production only                       |
| .env.test                   | Test only                             |
+-----------------------------+---------------------------------------+

CRITICAL RULE:
  Variables starting with NEXT_PUBLIC_ are exposed to the browser.
  All other variables are SERVER-ONLY.

  NEXT_PUBLIC_API_URL=https://api.example.com   <-- Visible in browser JS
  DATABASE_URL=postgresql://...                  <-- Server only (SAFE)
  API_SECRET_KEY=sk_...                          <-- Server only (SAFE)
```

```tsx
// Server Component -- can access ALL env vars
export default async function Page() {
  const dbUrl = process.env.DATABASE_URL;       // Works (server only)
  const apiKey = process.env.API_SECRET_KEY;    // Works (server only)
  const publicUrl = process.env.NEXT_PUBLIC_API_URL; // Works
  // ...
}

// Client Component -- only NEXT_PUBLIC_ vars
'use client';
export default function ClientComp() {
  const apiUrl = process.env.NEXT_PUBLIC_API_URL;  // Works
  const dbUrl = process.env.DATABASE_URL;           // undefined! (server only)
  // ...
}
```

### Angular environments vs Next.js env files

```
Angular:                               Next.js:
environment.ts (dev)                   .env.local (dev)
environment.prod.ts (prod)             .env.production (prod)

Angular replaces files at build        Next.js reads .env files at
time (fileReplacements in              build/runtime. Server vars
angular.json). ALL values end          are truly server-only.
up in the client bundle.               Only NEXT_PUBLIC_ vars are
                                       in the client bundle.
```

---

## Performance Optimization

### Image Component

```tsx
// Angular: <img src="photo.jpg" loading="lazy">
// Next.js: <Image> component with automatic optimization

import Image from 'next/image';

function ProductCard({ product }: { product: Product }) {
  return (
    <div>
      <Image
        src={product.imageUrl}
        alt={product.name}
        width={400}
        height={300}
        placeholder="blur"            // Show blur while loading
        blurDataURL={product.blurUrl}  // Base64 tiny image
        priority={false}               // true for above-the-fold images
        sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      />
    </div>
  );
}

// Benefits over plain <img>:
// - Automatic WebP/AVIF conversion
// - Automatic resizing for different devices
// - Lazy loading by default
// - Prevents layout shift (CLS)
// - Serves from optimized cache
```

### Font Optimization

```tsx
// app/layout.tsx
import { Inter, Roboto_Mono } from 'next/font/google';

// Fonts are downloaded at BUILD time and self-hosted
// No external network requests at runtime
const inter = Inter({
  subsets: ['latin'],
  display: 'swap', // Prevent FOIT (Flash of Invisible Text)
  variable: '--font-inter',
});

const robotoMono = Roboto_Mono({
  subsets: ['latin'],
  variable: '--font-roboto-mono',
});

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" className={`${inter.variable} ${robotoMono.variable}`}>
      <body className={inter.className}>{children}</body>
    </html>
  );
}
```

### Script Component

```tsx
import Script from 'next/script';

export default function Layout({ children }) {
  return (
    <>
      {children}

      {/* Load analytics after page is interactive */}
      <Script
        src="https://www.googletagmanager.com/gtag/js?id=GA_ID"
        strategy="afterInteractive"  // Load after hydration
      />

      {/* Load chat widget when browser is idle */}
      <Script
        src="https://widget.intercom.io/widget/abc123"
        strategy="lazyOnload"  // Load during idle time
      />

      {/* Load critical script before hydration */}
      <Script
        src="/critical.js"
        strategy="beforeInteractive"
      />
    </>
  );
}
```

### Performance Checklist

```
+--------------------------------------------------+
| Next.js Performance Optimization Checklist       |
+--------------------------------------------------+
| [x] Use <Image> for all images                   |
| [x] Use next/font for fonts (self-hosted)        |
| [x] Use <Script> with appropriate strategy       |
| [x] Keep Client Components small and at leaves   |
| [x] Use Server Components for data fetching      |
| [x] Implement Suspense for streaming             |
| [x] Use dynamic() for code splitting components  |
| [x] Set proper cache/revalidation on fetches     |
| [x] Use ISR where possible over SSR              |
| [x] Minimize client-side JavaScript              |
| [x] Use route groups to split layouts             |
| [x] Lazy load below-the-fold components          |
+--------------------------------------------------+
```

---

## SEO Best Practices

### Metadata API

```tsx
// app/layout.tsx -- site-wide metadata
import { Metadata } from 'next';

export const metadata: Metadata = {
  metadataBase: new URL('https://example.com'),
  title: {
    default: 'My App',
    template: '%s | My App', // Child pages: "About | My App"
  },
  description: 'My awesome Next.js application',
  openGraph: {
    type: 'website',
    locale: 'en_US',
    url: 'https://example.com',
    siteName: 'My App',
    images: [{ url: '/og-image.jpg', width: 1200, height: 630 }],
  },
  twitter: {
    card: 'summary_large_image',
    creator: '@myhandle',
  },
  robots: {
    index: true,
    follow: true,
  },
};
```

### Sitemap and Robots

```tsx
// app/sitemap.ts -- generates /sitemap.xml
import { MetadataRoute } from 'next';

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const posts = await fetch('https://api.example.com/posts').then(r => r.json());

  const postUrls = posts.map((post: Post) => ({
    url: `https://example.com/blog/${post.slug}`,
    lastModified: new Date(post.updatedAt),
    changeFrequency: 'weekly' as const,
    priority: 0.8,
  }));

  return [
    { url: 'https://example.com', lastModified: new Date(), priority: 1.0 },
    { url: 'https://example.com/about', lastModified: new Date(), priority: 0.5 },
    ...postUrls,
  ];
}

// app/robots.ts -- generates /robots.txt
import { MetadataRoute } from 'next';

export default function robots(): MetadataRoute.Robots {
  return {
    rules: {
      userAgent: '*',
      allow: '/',
      disallow: ['/admin/', '/api/'],
    },
    sitemap: 'https://example.com/sitemap.xml',
  };
}
```

### Structured Data (JSON-LD)

```tsx
// app/blog/[slug]/page.tsx
export default async function BlogPost({ params }) {
  const { slug } = await params;
  const post = await getPost(slug);

  const jsonLd = {
    '@context': 'https://schema.org',
    '@type': 'BlogPosting',
    headline: post.title,
    datePublished: post.createdAt,
    author: { '@type': 'Person', name: post.author.name },
    image: post.coverImage,
  };

  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
      />
      <article>
        <h1>{post.title}</h1>
        {/* ... */}
      </article>
    </>
  );
}
```

---

## Interview Questions

### Q1: How does authentication work in Next.js? Compare with Angular.

**A:** Next.js uses NextAuth.js (Auth.js) which provides OAuth, credentials, and JWT authentication. Unlike Angular where auth state lives in a service and tokens are managed by interceptors, Next.js stores session data in httpOnly cookies managed by the library. Server Components access sessions via `getServerSession()`, Client Components via `useSession()` hook. Middleware (`middleware.ts`) replaces Angular's route guards for protecting routes. The key advantage: in Next.js, auth checks happen server-side before any HTML is sent, while Angular guards run client-side after JavaScript loads.

### Q2: How does Next.js middleware compare to Angular guards and interceptors?

**A:** Next.js middleware runs on the Edge Runtime before a request is completed, combining the roles of Angular's `canActivate` guards and `HttpInterceptor`. It can redirect unauthenticated users (like guards), add/modify headers (like interceptors), rewrite URLs, and set cookies. Key differences: (1) It runs on the server, not the client. (2) It's a single file (`middleware.ts`) with path matching, not separate classes per route. (3) It runs on the Edge Runtime with limited API access (no Node.js fs, no database connections). (4) It processes ALL matching requests, including static file requests unless excluded.

### Q3: How do you protect routes in Next.js?

**A:** Three levels: (1) **Middleware** -- fastest, runs before any rendering. Check auth tokens in cookies, redirect if missing. Best for simple auth checks. (2) **Layout-level** -- use `getServerSession()` in a layout to protect all child routes. Can access database for role checks. (3) **Page-level** -- protect individual pages with `getServerSession()` and `redirect()`. Most granular, can check specific permissions. Middleware is recommended as the first line of defense because it's fastest and prevents any server rendering for unauthorized requests.

### Q4: How do you handle JWT tokens in Next.js with a Spring Boot backend?

**A:** Store JWT tokens in httpOnly, secure cookies (not localStorage -- XSS vulnerability). Login: call Spring Boot's auth endpoint from a Server Action or Route Handler, set the JWT in a cookie. Requests: read the cookie in Server Components or middleware, attach as `Authorization: Bearer` header when calling Spring Boot APIs. Refresh: implement token refresh logic in middleware or a utility function -- check expiry, call refresh endpoint, update cookie. This is more secure than Angular's typical localStorage approach because httpOnly cookies are inaccessible to JavaScript.

### Q5: What are the different deployment options for Next.js?

**A:** (1) **Vercel** -- zero-config, built by Next.js team. Best DX with preview deployments, analytics, edge functions. (2) **Self-hosted Node.js** -- run `next build && next start` on any server. Need to handle reverse proxy, SSL, CDN yourself. (3) **Docker** -- use multi-stage Dockerfile with `output: 'standalone'` config. Works with K8s, ECS, Cloud Run. (4) **Static export** -- `output: 'export'` generates pure HTML/CSS/JS. No SSR/ISR, but works on any CDN/S3. Choose based on needs: Vercel for simplicity, Docker for infrastructure control, static for simple sites.

### Q6: How do environment variables work in Next.js?

**A:** Two types: server-only (default, e.g., `DATABASE_URL`) and public (prefixed `NEXT_PUBLIC_`, e.g., `NEXT_PUBLIC_API_URL`). Server-only variables are never exposed to the browser -- safe for secrets, database URLs, API keys. Public variables are inlined into the client JavaScript bundle at build time. Files: `.env.local` for local development (gitignored), `.env.production` for production. Unlike Angular's `environment.ts` where all values end up in the bundle, Next.js server variables are truly secret.

### Q7: How does the Next.js Image component improve performance?

**A:** The `<Image>` component: (1) Automatically serves WebP/AVIF formats (smaller files). (2) Resizes images based on device viewport and `sizes` prop. (3) Lazy loads images by default (only loads when visible). (4) Prevents Cumulative Layout Shift by requiring width/height (reserves space). (5) Serves from an optimization cache. (6) Supports blur-up placeholder for perceived performance. Compared to a plain `<img>` tag, it can reduce image payload by 30-50% with zero extra effort.

### Q8: What is the difference between `beforeInteractive`, `afterInteractive`, and `lazyOnload` Script strategies?

**A:** `beforeInteractive` loads the script before any Next.js code -- use for critical scripts needed for page functionality (polyfills, cookie consent). `afterInteractive` (default) loads after the page becomes interactive -- use for analytics, tag managers. `lazyOnload` loads during browser idle time -- use for low-priority scripts (chat widgets, social embeds). The strategy determines when the script downloads and executes, directly impacting Core Web Vitals. Angular has no equivalent -- scripts are typically just in `index.html` or loaded via service.

### Q9: How do you implement role-based access control (RBAC) in Next.js?

**A:** Multi-layered: (1) **Middleware** for route-level RBAC: check the JWT token's role claim, redirect if insufficient. (2) **Server Components** for fine-grained access: `const session = await getServerSession(); if (session.user.role !== 'admin') redirect('/unauthorized')`. (3) **Client Components** for conditional UI: `const { data: session } = useSession(); {session?.user.role === 'admin' && <AdminPanel />}`. Always enforce on the server side (middleware or Server Components) -- client-side checks are for UX only. Store roles in the JWT/session via NextAuth callbacks.

### Q10: How do you implement CSRF protection in Next.js?

**A:** Next.js Server Actions have built-in CSRF protection -- they generate and validate CSRF tokens automatically. For Route Handlers, use: (1) SameSite cookies (`sameSite: 'lax'` or `'strict'`). (2) Check `Origin` and `Referer` headers in middleware. (3) Use the `csrf` npm package for explicit token-based CSRF protection. (4) For API routes consumed by external clients, use CORS headers and API keys instead of cookies. NextAuth.js also includes CSRF protection for its auth endpoints. In Angular, `HttpClientXsrfModule` handles this; in Next.js, it's largely handled by the framework for Server Actions.

### Q11: How do you handle SEO in Next.js compared to Angular?

**A:** Next.js has significant SEO advantages: (1) Server-rendered HTML by default -- search engines see full content. (2) Built-in Metadata API for title, description, OG tags per page. (3) Automatic sitemap.xml and robots.txt generation via file conventions. (4) Structured data (JSON-LD) support. (5) `<Image>` prevents CLS, improving Core Web Vitals. Angular Universal provides SSR but requires manual setup. Angular has no built-in metadata API (uses `Title`/`Meta` services imperatively), no sitemap generation, and no image optimization. Next.js treats SEO as a first-class concern.

### Q12: What is the Edge Runtime and when is it used in Next.js?

**A:** The Edge Runtime is a lightweight JavaScript runtime that runs closer to users (CDN edge locations). It starts faster than Node.js but has a limited API surface (no `fs`, no native modules, limited `crypto`). Middleware always runs on Edge. Route Handlers and Server Components can opt into Edge: `export const runtime = 'edge'`. Use Edge for: auth checks, redirects, A/B testing, geolocation, and simple API responses. Use Node.js for: database connections, file operations, CPU-intensive work. Edge provides lower latency (runs near the user) at the cost of API limitations.

### Q13: How do you set up monitoring and error tracking in production Next.js?

**A:** (1) **Error boundaries** (`error.tsx`) for route-level error UI. (2) **Sentry** (most popular) -- install `@sentry/nextjs`, wrap `next.config.js` with Sentry config, create `sentry.server.config.ts` and `sentry.client.config.ts`. Captures errors from Server Components, Client Components, middleware, and Route Handlers. (3) **Vercel Analytics** -- zero-config performance monitoring on Vercel. (4) **OpenTelemetry** -- Next.js has experimental support for distributed tracing. (5) Custom logging with `console.log` in Server Components (appears in server logs). Combine Sentry for errors with Vercel Analytics for performance.

# Spring Boot + Next.js Fullstack Patterns

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [API Communication Patterns](#api-communication-patterns)
3. [Authentication Flow](#authentication-flow)
4. [CORS Setup](#cors-setup)
5. [Error Handling Across the Stack](#error-handling-across-the-stack)
6. [Environment Management](#environment-management)
7. [Docker Compose Setup](#docker-compose-setup)
8. [Deployment Strategies](#deployment-strategies)
9. [Interview Questions](#interview-questions)

---

## Architecture Overview

```
+=====================================================================+
|           Spring Boot + Next.js Production Architecture              |
+=====================================================================+

  Users (Browser / Mobile)
        |
        v
  +-----+------+
  |   CDN      |  Static assets (images, CSS, JS bundles)
  |  (CloudFront|  Cached SSG/ISR pages
  |   / Vercel) |
  +-----+------+
        |
        v
  +-----+------+     +-------------------------------------------------+
  |  Next.js   |     |              Internal Network                    |
  |  Server    |     |                                                  |
  |            |     |  +-----------+    +-----------+   +----------+   |
  | - SSR/SSG  +---->|  | Spring    |    | Spring    |   | Redis    |   |
  | - BFF      |  |  |  | Boot API  |    | Boot      |   | Cache    |   |
  | - Auth     |  |  |  | Service   |    | Auth      |   |          |   |
  | - Caching  |  |  |  |           |    | Service   |   |          |   |
  | - API agg. |  |  |  | /api/**   |    | /auth/**  |   |          |   |
  +-----+------+  |  |  +-----+-----+    +-----+-----+   +----+-----+   |
        |          |  |        |                |              |         |
        |          |  |        v                v              |         |
        |          |  |  +-----+----------------+-----+        |         |
        |          |  |  |      PostgreSQL             |<------+         |
        |          |  |  |      (Primary DB)           |                 |
        |          |  |  +-----------------------------+                 |
        |          |  |                                                  |
        |          |  +-------------------------------------------------+
        |          |
        |          +----> External Services (Stripe, SendGrid, S3)
        |
        v
  +-----+------+
  |  Browser   |
  |  (React    |
  |   Hydrated)|
  +------------+
```

### Request Flow Types

```
TYPE 1: Static Page (SSG/ISR) -- Fastest
========================================
  Browser --> CDN --> Cached HTML (no server hit)

TYPE 2: Server-Rendered Page (SSR)
===================================
  Browser --> Next.js --> Spring Boot API --> Database
                     <-- JSON
             <-- HTML
  <-- HTML

TYPE 3: Client-Side Interaction
================================
  Browser --> Next.js Server Action --> Spring Boot API --> Database
                                  <-- JSON
         <-- Updated HTML / Redirect

TYPE 4: Client-Side Data Fetch (Real-time)
==========================================
  Browser --> Next.js Route Handler --> Spring Boot API --> Database
                                   <-- JSON
         <-- JSON (via SWR/React Query)
```

---

## API Communication Patterns

### Pattern 1: REST API

The most common pattern. Spring Boot exposes REST endpoints, Next.js consumes them.

```java
// Spring Boot: ProductController.java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    @Autowired
    private ProductService productService;

    @GetMapping
    public ResponseEntity<Page<ProductDTO>> getProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String category) {
        return ResponseEntity.ok(
            productService.getProducts(page, size, category)
        );
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductDTO> getProduct(@PathVariable Long id) {
        return ResponseEntity.ok(productService.getProduct(id));
    }

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductDTO> createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        return ResponseEntity.status(201)
            .body(productService.createProduct(request));
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductDTO> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody UpdateProductRequest request) {
        return ResponseEntity.ok(productService.updateProduct(id, request));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        productService.deleteProduct(id);
        return ResponseEntity.noContent().build();
    }
}
```

```tsx
// Next.js: lib/api/products.ts
const API_URL = process.env.API_URL; // http://spring-boot:8080

export async function getProducts(params: {
  page?: number;
  size?: number;
  category?: string;
}) {
  const searchParams = new URLSearchParams();
  if (params.page) searchParams.set('page', String(params.page));
  if (params.size) searchParams.set('size', String(params.size));
  if (params.category) searchParams.set('category', params.category);

  const res = await fetch(
    `${API_URL}/api/products?${searchParams}`,
    {
      next: { tags: ['products'], revalidate: 300 },
    }
  );

  if (!res.ok) throw new ApiError(res.status, 'Failed to fetch products');
  return res.json() as Promise<PaginatedResponse<Product>>;
}

export async function getProduct(id: string) {
  const res = await fetch(`${API_URL}/api/products/${id}`, {
    next: { tags: [`product-${id}`] },
  });

  if (!res.ok) {
    if (res.status === 404) return null;
    throw new ApiError(res.status, 'Failed to fetch product');
  }

  return res.json() as Promise<Product>;
}

// Server Action for mutations
'use server';
import { revalidateTag } from 'next/cache';
import { cookies } from 'next/headers';

export async function createProduct(formData: FormData) {
  const cookieStore = await cookies();
  const token = cookieStore.get('access_token')?.value;

  const res = await fetch(`${API_URL}/api/products`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${token}`,
    },
    body: JSON.stringify({
      name: formData.get('name'),
      price: Number(formData.get('price')),
      category: formData.get('category'),
    }),
  });

  if (!res.ok) {
    const error = await res.json();
    return { error: error.message };
  }

  revalidateTag('products');
  return { success: true };
}
```

### Pattern 2: GraphQL

```tsx
// Next.js: lib/graphql.ts
const GRAPHQL_URL = `${process.env.API_URL}/graphql`;

export async function graphqlFetch<T>(
  query: string,
  variables?: Record<string, any>,
  options?: { tags?: string[]; revalidate?: number }
): Promise<T> {
  const res = await fetch(GRAPHQL_URL, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ query, variables }),
    next: {
      tags: options?.tags,
      revalidate: options?.revalidate,
    },
  });

  const json = await res.json();
  if (json.errors) {
    throw new Error(json.errors[0].message);
  }

  return json.data;
}

// Usage in Server Component
export default async function ProductsPage() {
  const data = await graphqlFetch<{ products: Product[] }>(
    `
    query GetProducts($page: Int!, $size: Int!) {
      products(page: $page, size: $size) {
        id
        name
        price
        category { name }
        images { url }
      }
    }
    `,
    { page: 1, size: 20 },
    { tags: ['products'], revalidate: 300 }
  );

  return <ProductGrid products={data.products} />;
}
```

### Pattern Comparison

```
+------------------+----------------------------+----------------------------+
| Aspect           | REST                       | GraphQL                    |
+------------------+----------------------------+----------------------------+
| Endpoint         | Multiple (/products,       | Single (/graphql)          |
|                  |  /products/:id, etc.)      |                            |
| Data fetching    | Fixed shape per endpoint   | Client specifies fields    |
| Over-fetching    | Common                     | Avoided (request only what |
|                  |                            |  you need)                 |
| Under-fetching   | Common (need multiple      | Avoided (get everything    |
|                  |  calls)                    |  in one query)             |
| Caching          | Easy (URL-based)           | Complex (POST requests)    |
| Next.js support  | Native fetch               | fetch with POST body       |
| Spring Boot      | Spring MVC                 | Spring GraphQL             |
| Complexity       | Lower                      | Higher (schema, resolvers) |
| Best for         | CRUD, simple data          | Complex, nested data       |
|                  |                            | Multiple frontends         |
+------------------+----------------------------+----------------------------+
```

---

## Authentication Flow

### Complete JWT Auth Flow

```
  REGISTRATION FLOW:
  ==================

  Browser              Next.js Server           Spring Boot
  |                    |                         |
  | Submit reg form    |                         |
  |--- POST ---------->| Server Action:          |
  |                    | POST /api/auth/register  |
  |                    |------------------------>|
  |                    |                         | Validate input
  |                    |                         | Hash password
  |                    |                         | Save user to DB
  |                    |   201 { user }          | Return user
  |                    |<------------------------|
  |  Redirect /login   |                         |
  |<-------------------|                         |


  LOGIN FLOW:
  ===========

  Browser              Next.js Server           Spring Boot
  |                    |                         |
  | Submit login form  |                         |
  |--- POST ---------->| Server Action:          |
  |                    | POST /api/auth/login     |
  |                    |------------------------>|
  |                    |                         | Validate credentials
  |                    |                         | Generate JWT tokens:
  |                    |  {                      |   accessToken (15min)
  |                    |    accessToken,          |   refreshToken (7d)
  |                    |    refreshToken          |
  |                    |  }                      |
  |                    |<------------------------|
  |                    |                         |
  |                    | Set httpOnly cookies:    |
  |                    | - access_token           |
  |                    | - refresh_token          |
  |  Set-Cookie        |                         |
  |  Redirect /dash    |                         |
  |<-------------------|                         |


  AUTHENTICATED REQUEST FLOW:
  ============================

  Browser              Next.js Server           Spring Boot
  |                    |                         |
  | GET /dashboard     |                         |
  | (cookies auto-sent)|                         |
  |--- GET ----------->| Middleware:              |
  |                    | 1. Read access_token     |
  |                    | 2. Verify not expired    |
  |                    | 3. Allow request         |
  |                    |                         |
  |                    | Server Component:        |
  |                    | GET /api/dashboard       |
  |                    | Auth: Bearer <token>     |
  |                    |------------------------>|
  |                    |                         | Validate JWT
  |                    |   { dashboard data }    | Return data
  |                    |<------------------------|
  |                    |                         |
  |  HTML (rendered    | Render with data        |
  |   with data)       |                         |
  |<-------------------|                         |


  TOKEN REFRESH FLOW:
  ====================

  Browser              Next.js Server           Spring Boot
  |                    |                         |
  | GET /dashboard     |                         |
  |--- GET ----------->| Middleware:              |
  |                    | 1. Read access_token     |
  |                    | 2. EXPIRED!              |
  |                    | 3. Read refresh_token    |
  |                    |                         |
  |                    | POST /api/auth/refresh   |
  |                    | { refreshToken }         |
  |                    |------------------------>|
  |                    |                         | Validate refresh
  |                    |  { newAccessToken,      | Generate new
  |                    |    newRefreshToken }     | tokens
  |                    |<------------------------|
  |                    |                         |
  |                    | Update cookies           |
  |                    | Continue to page...      |
  |  Set-Cookie +      |                         |
  |  HTML response     |                         |
  |<-------------------|                         |
```

### Implementation

```java
// Spring Boot: AuthController.java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired private AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<UserDTO> register(
            @Valid @RequestBody RegisterRequest request) {
        UserDTO user = authService.register(request);
        return ResponseEntity.status(201).body(user);
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(
            @Valid @RequestBody LoginRequest request) {
        AuthResponse response = authService.login(request);
        return ResponseEntity.ok(response);
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(
            @Valid @RequestBody RefreshRequest request) {
        AuthResponse response = authService.refresh(request.getRefreshToken());
        return ResponseEntity.ok(response);
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(
            @RequestHeader("Authorization") String token) {
        authService.logout(token);
        return ResponseEntity.noContent().build();
    }
}
```

```tsx
// Next.js: app/actions/auth.ts
'use server';

import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

const API_URL = process.env.API_URL;

export async function login(formData: FormData) {
  const res = await fetch(`${API_URL}/api/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      email: formData.get('email'),
      password: formData.get('password'),
    }),
  });

  if (!res.ok) {
    const error = await res.json();
    return { error: error.message || 'Invalid credentials' };
  }

  const { accessToken, refreshToken } = await res.json();

  const cookieStore = await cookies();

  cookieStore.set('access_token', accessToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 15 * 60, // 15 minutes
    path: '/',
  });

  cookieStore.set('refresh_token', refreshToken, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    maxAge: 7 * 24 * 60 * 60, // 7 days
    path: '/',
  });

  redirect('/dashboard');
}

export async function logout() {
  const cookieStore = await cookies();
  const token = cookieStore.get('access_token')?.value;

  if (token) {
    await fetch(`${API_URL}/api/auth/logout`, {
      method: 'POST',
      headers: { Authorization: `Bearer ${token}` },
    });
  }

  cookieStore.delete('access_token');
  cookieStore.delete('refresh_token');

  redirect('/login');
}
```

```tsx
// Next.js: middleware.ts
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';
import { jwtDecode } from 'jwt-decode';

const protectedPaths = ['/dashboard', '/profile', '/settings', '/admin'];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const isProtected = protectedPaths.some(p => pathname.startsWith(p));

  if (!isProtected) return NextResponse.next();

  const accessToken = request.cookies.get('access_token')?.value;
  const refreshToken = request.cookies.get('refresh_token')?.value;

  // No tokens at all
  if (!accessToken && !refreshToken) {
    return redirectToLogin(request);
  }

  // Try access token
  if (accessToken) {
    try {
      const decoded = jwtDecode(accessToken);
      if (decoded.exp && decoded.exp * 1000 > Date.now()) {
        // Token valid -- check role for admin routes
        if (pathname.startsWith('/admin') && decoded.role !== 'ADMIN') {
          return NextResponse.redirect(new URL('/unauthorized', request.url));
        }
        return NextResponse.next();
      }
    } catch {
      // Token invalid, try refresh
    }
  }

  // Try refresh
  if (refreshToken) {
    try {
      const res = await fetch(`${process.env.API_URL}/api/auth/refresh`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ refreshToken }),
      });

      if (res.ok) {
        const { accessToken: newToken, refreshToken: newRefresh } = await res.json();
        const response = NextResponse.next();
        response.cookies.set('access_token', newToken, {
          httpOnly: true,
          secure: true,
          sameSite: 'lax',
          maxAge: 15 * 60,
        });
        response.cookies.set('refresh_token', newRefresh, {
          httpOnly: true,
          secure: true,
          sameSite: 'lax',
          maxAge: 7 * 24 * 60 * 60,
        });
        return response;
      }
    } catch {
      // Refresh failed
    }
  }

  return redirectToLogin(request);
}

function redirectToLogin(request: NextRequest) {
  const loginUrl = new URL('/login', request.url);
  loginUrl.searchParams.set('callbackUrl', request.nextUrl.pathname);
  return NextResponse.redirect(loginUrl);
}

export const config = {
  matcher: ['/dashboard/:path*', '/profile/:path*', '/settings/:path*', '/admin/:path*'],
};
```

---

## CORS Setup

### Spring Boot CORS Configuration

```java
// Method 1: Global CORS config
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins(
                "http://localhost:3000",         // Next.js dev
                "https://myapp.com",             // Production
                "https://preview-*.vercel.app"   // Preview deployments
            )
            .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS")
            .allowedHeaders("*")
            .exposedHeaders("X-Total-Count", "X-Page-Count")
            .allowCredentials(true) // Required for cookies
            .maxAge(3600);
    }
}

// Method 2: Spring Security CORS (when using Spring Security)
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigSource()))
            .csrf(csrf -> csrf.disable()) // Disable CSRF for API
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    CorsConfigurationSource corsConfigSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of(
            "http://localhost:3000",
            "https://myapp.com"
        ));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}
```

### CORS Is Not Needed When Next.js Is the BFF

```
  Scenario 1: Browser calls Spring Boot directly (CORS NEEDED)
  ============================================================
  Browser (localhost:3000) ---> Spring Boot (localhost:8080)
  Different origins! CORS required.

  Scenario 2: Next.js as BFF (NO CORS NEEDED for browser)
  ========================================================
  Browser (myapp.com) ---> Next.js (myapp.com) ---> Spring Boot (internal)
  Same origin!            Server-to-server (no CORS)

  CORS is still needed for:
  - Direct browser-to-Spring Boot calls (client-side fetching)
  - Swagger/OpenAPI testing
  - Mobile app direct API access
```

---

## Error Handling Across the Stack

### Error Handling Architecture

```
  Browser (Client)        Next.js Server         Spring Boot
  ================        ==============         ===========

  error.tsx               Server Action          @ControllerAdvice
  (route error boundary)  try/catch              GlobalExceptionHandler

  Client Component        Route Handler          @ExceptionHandler
  try/catch               try/catch              Custom exceptions

  React Error Boundary    middleware              ResponseStatusException
  (global fallback)       error handling

  Error Flow:
  ===========
  Spring Boot throws BusinessException("Product not found")
       |
       v
  @ControllerAdvice catches, returns { status: 404, message: "..." }
       |
       v
  Next.js fetch gets 404 response
       |
       v
  Option A: Server Component throws --> error.tsx shows error UI
  Option B: Server Action returns error --> Client shows inline error
  Option C: Route Handler returns error JSON --> Client handles
```

### Spring Boot Error Handling

```java
// Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(
            new ErrorResponse("NOT_FOUND", ex.getMessage())
        );
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException ex) {
        return ResponseEntity.status(400).body(
            new ErrorResponse("VALIDATION_ERROR", ex.getMessage(), ex.getErrors())
        );
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleMethodArgNotValid(
            MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
        );
        return ResponseEntity.status(400).body(
            new ErrorResponse("VALIDATION_ERROR", "Validation failed", errors)
        );
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        return ResponseEntity.status(403).body(
            new ErrorResponse("FORBIDDEN", "Access denied")
        );
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        log.error("Unexpected error", ex);
        return ResponseEntity.status(500).body(
            new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred")
        );
    }
}
```

### Next.js Error Handling

```tsx
// lib/api-error.ts
export class ApiError extends Error {
  constructor(
    public status: number,
    message: string,
    public errors?: Record<string, string>
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

// lib/api/base.ts -- fetch wrapper with error handling
export async function apiFetch<T>(
  endpoint: string,
  options: RequestInit & { next?: NextFetchRequestConfig } = {}
): Promise<T> {
  const token = await getAccessToken();

  const res = await fetch(`${process.env.API_URL}${endpoint}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.headers,
    },
  });

  if (!res.ok) {
    const errorBody = await res.json().catch(() => ({}));

    // Log on server for debugging
    console.error(`API Error: ${res.status} ${endpoint}`, errorBody);

    throw new ApiError(
      res.status,
      errorBody.message || `API request failed: ${res.status}`,
      errorBody.errors
    );
  }

  return res.json();
}
```

```tsx
// app/products/[id]/page.tsx -- Page-level error handling
import { notFound } from 'next/navigation';
import { apiFetch, ApiError } from '@/lib/api';

export default async function ProductPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;

  try {
    const product = await apiFetch<Product>(`/api/products/${id}`);
    return <ProductDetail product={product} />;
  } catch (error) {
    if (error instanceof ApiError && error.status === 404) {
      notFound(); // Triggers not-found.tsx
    }
    throw error; // Re-throw for error.tsx to catch
  }
}

// app/products/[id]/error.tsx
'use client';

export default function ProductError({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div className="text-center py-10">
      <h2 className="text-2xl font-bold text-red-600">
        Failed to load product
      </h2>
      <p className="text-gray-600 mt-2">{error.message}</p>
      <button
        onClick={reset}
        className="mt-4 px-4 py-2 bg-blue-600 text-white rounded"
      >
        Try Again
      </button>
    </div>
  );
}

// app/products/[id]/not-found.tsx
import Link from 'next/link';

export default function ProductNotFound() {
  return (
    <div className="text-center py-10">
      <h2 className="text-2xl font-bold">Product Not Found</h2>
      <p className="text-gray-600 mt-2">
        The product you are looking for does not exist.
      </p>
      <Link href="/products" className="text-blue-600 mt-4 inline-block">
        Browse all products
      </Link>
    </div>
  );
}
```

---

## Environment Management

### Environment Variables Across the Stack

```
+=====================================================================+
|                    Environment Configuration                         |
+=====================================================================+

Spring Boot:                          Next.js:
============                          =======

application.yml                       .env.local (development)
application-dev.yml                   .env.production (production)
application-prod.yml                  .env.test (testing)

Properties:                           Variables:

spring:                               # Server-only (safe for secrets)
  datasource:                         DATABASE_URL=postgresql://...
    url: jdbc:postgresql://...        API_URL=http://localhost:8080
    username: ${DB_USER}              JWT_SECRET=super-secret-key
    password: ${DB_PASS}              GOOGLE_CLIENT_SECRET=...
  jpa:
    hibernate:                        # Public (exposed to browser)
      ddl-auto: validate              NEXT_PUBLIC_APP_URL=https://myapp.com
                                      NEXT_PUBLIC_STRIPE_PK=pk_live_...
server:
  port: 8080                          # Docker/CI overrides via env vars
                                      # (no files needed)
jwt:
  secret: ${JWT_SECRET}
  expiration: 900000


Development:              Staging:                Production:
============              ========                ===========
Spring: 8080              Spring: 8080            Spring: 8080 (internal)
Next.js: 3000             Next.js: 3000           Next.js: 3000 (behind LB)
DB: localhost:5432         DB: staging-db:5432     DB: prod-db:5432
```

### Shared Type Definitions

When Spring Boot and Next.js need to agree on data shapes, define types that mirror the API contracts.

```tsx
// Next.js: types/api.ts -- mirrors Spring Boot DTOs

export interface Product {
  id: number;
  name: string;
  description: string;
  price: number;
  category: Category;
  images: ProductImage[];
  createdAt: string; // ISO date string (Java Instant -> String)
  updatedAt: string;
}

export interface Category {
  id: number;
  name: string;
  slug: string;
}

export interface PaginatedResponse<T> {
  content: T[];
  totalElements: number;
  totalPages: number;
  size: number;
  number: number; // Current page (0-based from Spring)
  first: boolean;
  last: boolean;
}

export interface ErrorResponse {
  code: string;
  message: string;
  errors?: Record<string, string>;
  timestamp: string;
}

export interface AuthResponse {
  accessToken: string;
  refreshToken: string;
  expiresIn: number;
  user: UserDTO;
}
```

---

## Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:16-alpine
    container_name: app-postgres
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: app-redis
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Spring Boot API
  spring-api:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: app-spring
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/myapp
      SPRING_DATASOURCE_USERNAME: postgres
      SPRING_DATASOURCE_PASSWORD: postgres
      SPRING_REDIS_HOST: redis
      SPRING_REDIS_PORT: 6379
      JWT_SECRET: your-jwt-secret-here
      SPRING_PROFILES_ACTIVE: docker
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Next.js Frontend
  nextjs:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: app-nextjs
    environment:
      API_URL: http://spring-api:8080
      NEXT_PUBLIC_APP_URL: http://localhost:3000
      NEXTAUTH_URL: http://localhost:3000
      NEXTAUTH_SECRET: your-nextauth-secret
    ports:
      - "3000:3000"
    depends_on:
      spring-api:
        condition: service_healthy

volumes:
  postgres_data:
```

```dockerfile
# backend/Dockerfile (Spring Boot)
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY . .
RUN ./gradlew bootJar --no-daemon

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```dockerfile
# frontend/Dockerfile (Next.js)
FROM node:20-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-alpine
WORKDIR /app
ENV NODE_ENV production
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nextjs
COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

### Development Workflow

```bash
# Start everything
docker compose up -d

# View logs
docker compose logs -f spring-api
docker compose logs -f nextjs

# Rebuild after code changes
docker compose up --build spring-api  # Rebuild only Spring Boot
docker compose up --build nextjs      # Rebuild only Next.js

# Stop everything
docker compose down

# Stop and remove volumes (reset database)
docker compose down -v
```

---

## Deployment Strategies

### Strategy 1: Vercel + Cloud (Recommended for Most)

```
+------------------------------------+
|  Vercel (Next.js)                  |
|  - Auto-deploy from Git            |
|  - Preview deployments per PR      |
|  - Edge network                    |
|  - Serverless functions            |
+------------------------------------+
        |
        | Internal API calls
        v
+------------------------------------+
|  AWS/GCP/Azure (Spring Boot)       |
|  - ECS/Cloud Run/App Engine        |
|  - Auto-scaling                    |
|  - RDS/Cloud SQL (PostgreSQL)      |
|  - ElastiCache (Redis)             |
+------------------------------------+
```

### Strategy 2: Kubernetes (Enterprise)

```
+------------------------------------+
|  Kubernetes Cluster                |
|                                    |
|  +-- Ingress Controller ----------+|
|  |  (Nginx / Traefik)            ||
|  +--------------------------------+|
|       |                |           |
|       v                v           |
|  +----------+    +----------+     ||
|  |Next.js   |    |Spring    |     ||
|  |Pods (3)  |    |Boot      |     ||
|  |          |    |Pods (3)  |     ||
|  +----------+    +----------+     ||
|                       |            |
|                       v            |
|               +----------+        ||
|               |PostgreSQL|        ||
|               |StatefulSet|       ||
|               +----------+        ||
+------------------------------------+
```

### Strategy 3: Single Server (Budget)

```
+------------------------------------+
|  Single VPS (DigitalOcean/Hetzner) |
|                                    |
|  Nginx (reverse proxy)             |
|  |                                 |
|  +-> :3000 Next.js (PM2)          |
|  +-> :8080 Spring Boot (systemd)   |
|  +-> :5432 PostgreSQL              |
|  +-> :6379 Redis                   |
|                                    |
|  + Certbot (SSL)                   |
|  + Fail2ban (security)             |
+------------------------------------+
```

### Deployment Checklist

```
PRE-DEPLOYMENT:
[ ] Environment variables set for production
[ ] Database migrations prepared
[ ] API URL pointing to production Spring Boot
[ ] CORS configured for production domain
[ ] SSL/TLS certificates configured
[ ] Health checks configured
[ ] Logging configured (structured JSON)

SECURITY:
[ ] httpOnly cookies for auth tokens
[ ] CSRF protection enabled
[ ] Rate limiting on auth endpoints
[ ] Input validation (Spring: @Valid, Next.js: Zod)
[ ] SQL injection prevention (parameterized queries)
[ ] XSS prevention (React auto-escapes, CSP headers)
[ ] Environment secrets not in code/git
[ ] HTTPS enforced

PERFORMANCE:
[ ] Next.js Image optimization configured
[ ] Static pages use SSG/ISR
[ ] Database indexes on query fields
[ ] Redis caching for hot data
[ ] CDN for static assets
[ ] Gzip/Brotli compression
[ ] Connection pooling (HikariCP)

MONITORING:
[ ] Error tracking (Sentry)
[ ] Application metrics (Prometheus/Grafana)
[ ] Uptime monitoring
[ ] Log aggregation
[ ] Alerting rules set
```

---

## Interview Questions

### Q1: Describe the architecture of a fullstack application using Spring Boot and Next.js.

**A:** The architecture follows a BFF pattern: Next.js serves as both the frontend and a Backend-for-Frontend layer. Next.js handles SSR/SSG, routing, and acts as a proxy to Spring Boot. Spring Boot handles business logic, data access, authentication (JWT), and serves as the core API. Communication is REST (or GraphQL) over HTTP. Auth tokens are stored in httpOnly cookies on Next.js and forwarded as Bearer tokens to Spring Boot. A CDN sits in front for static assets and cached pages. Database (PostgreSQL) and cache (Redis) back the Spring Boot layer. Docker Compose ties everything together for development.

### Q2: How does authentication work across Next.js and Spring Boot?

**A:** Login flow: (1) User submits credentials to a Next.js Server Action. (2) Server Action calls Spring Boot's auth endpoint. (3) Spring Boot validates credentials, generates JWT access + refresh tokens. (4) Next.js stores tokens in httpOnly, secure cookies (not accessible to JavaScript). (5) For subsequent requests, Next.js middleware reads the cookie, checks expiry, and refreshes if needed. (6) Server Components forward the token as `Authorization: Bearer` header when calling Spring Boot. This approach prevents XSS token theft (httpOnly cookies) and keeps auth logic server-side.

### Q3: How do you handle CORS between Next.js and Spring Boot?

**A:** Two approaches: (1) **BFF pattern (preferred)** -- Next.js server makes API calls to Spring Boot (server-to-server), eliminating CORS entirely for the browser. The browser only talks to Next.js (same origin). (2) **Direct API calls** -- when the browser needs to call Spring Boot directly (e.g., file uploads, WebSockets), configure Spring Boot's `CorsConfigurationSource` to allow the Next.js origin, specific methods, and credentials. In production, CORS origins should be explicit (not `*`) when `allowCredentials` is true. Use Spring Security's CORS configuration when security is enabled.

### Q4: How do you handle errors across Spring Boot and Next.js?

**A:** Layered approach: Spring Boot uses `@ControllerAdvice` with `@ExceptionHandler` to convert exceptions to structured JSON error responses (status code, error code, message, field errors). Next.js wraps API calls in a fetch utility that converts non-2xx responses to typed `ApiError` objects. Server Components can throw to trigger `error.tsx` or call `notFound()` for 404s. Server Actions return error objects for inline form validation. Each route can have its own `error.tsx` and `not-found.tsx` for granular error UI. Errors are logged server-side (Spring: SLF4J, Next.js: console + Sentry).

### Q5: How would you set up Docker Compose for this stack?

**A:** Define four services: (1) **postgres** -- PostgreSQL with health check, persistent volume, and environment credentials. (2) **redis** -- Redis for caching with health check. (3) **spring-api** -- multi-stage Dockerfile (build with Gradle/Maven, run with JRE), depends on postgres and redis, environment variables for database URL and JWT secret. (4) **nextjs** -- multi-stage Dockerfile (deps, build, standalone runner), depends on spring-api, environment variables for API_URL pointing to `http://spring-api:8080`. Services communicate via Docker's internal network using service names as hostnames.

### Q6: How would you deploy this stack to production?

**A:** Recommended approach for most teams: Deploy Next.js on Vercel (zero-config, preview deployments, edge network), Spring Boot on AWS ECS/GCP Cloud Run with auto-scaling, PostgreSQL on managed service (RDS/Cloud SQL), Redis on managed service (ElastiCache/Memorystore). Connect via private networking where possible. Alternative: Docker Compose on a single VPS for budget-conscious startups, or Kubernetes for enterprise scale. Regardless of platform: use CI/CD (GitHub Actions), staging environment, database migrations, health checks, and monitoring.

### Q7: How do you handle database migrations with this stack?

**A:** Spring Boot side: Use Flyway or Liquibase for database schema migrations. Migrations run automatically on application startup. Version migrations in source control (e.g., `V1__create_users.sql`, `V2__add_products.sql`). For production: run migrations as a separate step before deploying new code (migration job in CI/CD). Next.js side: If using Prisma for Next.js direct database access (some architectures), use `prisma migrate`. Best practice: let Spring Boot own the database schema since it's the core backend.

### Q8: What is the BFF (Backend for Frontend) pattern and why use it with Next.js?

**A:** BFF is a server-side layer that sits between the frontend and core backend services, tailored to the frontend's needs. Next.js naturally acts as a BFF: Server Components and Route Handlers aggregate multiple Spring Boot API calls into a single response, shape data for the UI (eliminating over-fetching), handle auth token management, add caching layers, and eliminate CORS issues. Benefits: (1) Browser makes fewer requests (to Next.js only). (2) Sensitive API structure hidden from client. (3) Can optimize responses per page needs. (4) Single point for auth/caching/error handling.

### Q9: How would you implement a real-time feature (like notifications) in this stack?

**A:** Spring Boot: Use WebSocket with STOMP protocol (`spring-boot-starter-websocket`) or Server-Sent Events (SSE) for one-way push. Next.js: Create a client component that connects to Spring Boot's WebSocket/SSE endpoint directly (bypassing the BFF for real-time). For a simpler approach: use polling with SWR (`refreshInterval: 5000`) through a Next.js Route Handler. For enterprise: use a message broker (RabbitMQ/Kafka) between Spring Boot services and a dedicated WebSocket gateway (Socket.io server). Redis Pub/Sub can synchronize WebSocket connections across multiple Spring Boot instances.

### Q10: How do you share type definitions between Spring Boot and Next.js?

**A:** Several approaches: (1) **Manual mirroring** -- define TypeScript interfaces in Next.js that match Spring Boot DTOs. Simple but can drift. (2) **OpenAPI/Swagger codegen** -- Spring Boot generates OpenAPI spec, use `openapi-generator-cli` to generate TypeScript types and API client for Next.js. Automated and always in sync. (3) **Shared JSON Schema** -- define schemas in a shared location, generate both Java POJOs and TypeScript types. (4) **GraphQL** -- schema is the contract, both sides generate types from it. OpenAPI codegen is the most practical for REST APIs.

### Q11: How do you handle file uploads in this architecture?

**A:** Pattern: (1) Client uploads to Next.js Route Handler (avoids CORS). (2) Route Handler streams the file to Spring Boot or directly to object storage (S3). (3) Spring Boot stores metadata in the database, returns URL. For large files: use presigned URLs -- client requests a presigned S3 URL from the API, then uploads directly to S3 (bypassing both servers). The presigned URL is time-limited and scoped to a specific key. Spring Boot generates the presigned URL; Next.js passes it to the client component.

### Q12: How do you implement caching across the stack?

**A:** Multi-layer caching: (1) **CDN layer** -- Next.js static/ISR pages cached at the CDN edge. (2) **Next.js data cache** -- fetch responses cached with `revalidate` or tags. (3) **Next.js Router Cache** -- client-side cache of visited routes. (4) **Spring Boot application cache** -- `@Cacheable` with Redis for frequently accessed data (product catalogs, user profiles). (5) **Database query cache** -- PostgreSQL's built-in query cache. Cache invalidation: Next.js uses `revalidateTag`/`revalidatePath`; Spring Boot uses `@CacheEvict` on mutations. Webhooks from Spring Boot to Next.js for cross-layer invalidation.

### Q13: How do you handle distributed logging and tracing?

**A:** Generate a correlation ID (request ID) in Next.js middleware and pass it as a header (`X-Request-ID`) to all Spring Boot API calls. Spring Boot logs include this ID. Both services use structured JSON logging (Spring: Logback JSON encoder, Next.js: structured console.log). Aggregate logs in a central system (ELK Stack, Datadog, or CloudWatch). For distributed tracing: use OpenTelemetry -- Next.js has experimental support, Spring Boot has `micrometer-tracing`. Traces connect the full request path: Browser -> Next.js -> Spring Boot -> Database. Sentry for error tracking on both sides.

### Q14: What security best practices would you implement for this stack?

**A:** **Transport:** HTTPS everywhere, HSTS headers. **Auth:** httpOnly/secure cookies for JWTs, short-lived access tokens (15min), refresh token rotation. **Input:** Server-side validation on both layers (Zod in Next.js, @Valid in Spring Boot). **Headers:** CSP, X-Frame-Options, X-Content-Type-Options via Next.js middleware. **API:** Rate limiting on auth endpoints, request size limits. **Database:** Parameterized queries (JPA/Prisma prevent SQL injection), principle of least privilege for DB users. **Secrets:** Environment variables (never in code), secrets manager in production. **Dependencies:** Regular security audits (`npm audit`, OWASP dependency check). **CORS:** Explicit allowed origins, never `*` with credentials.

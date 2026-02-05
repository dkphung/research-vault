---
tags: [tanstack, react, full-stack, architecture, patterns]
date: 2026-01-07
status: complete
---

# TanStack Start Best Practices for Production Applications

## Executive Summary

TanStack Start is a full-stack React framework built on TanStack Router and Vite, offering type-safe routing, server functions, streaming SSR, and universal deployment capabilities. Unlike Next.js's server-first approach with React Server Components, TanStack Start follows a **client-first, isomorphic architecture** where code runs in both environments by default unless explicitly constrained. For production applications requiring auth, CRUD, and API integrations, TanStack Start provides robust patterns through server functions for secure server-side logic, TanStack Query integration for data management, and flexible deployment to Node.js, Edge, or serverless platforms.

## Core Architecture Principles

TanStack Start follows an **isomorphic-first** design philosophy:

- All code is isomorphic by default (runs on both server and client)
- Server-only code must be explicitly constrained using `createServerFn()` or `createServerOnlyFn()`
- Route loaders execute on the server during SSR AND on the client during navigation
- This differs fundamentally from Next.js where Server Components are server-only by default

## Project Structure & Architecture

### Recommended Folder Organization

```
src/
├── routes/
│   ├── __root.tsx           # Root layout (document shell)
│   ├── index.tsx            # Home route (/)
│   ├── about.tsx            # Static route (/about)
│   ├── _authed.tsx          # Layout route (auth wrapper, no URL segment)
│   ├── _authed/
│   │   ├── dashboard.tsx    # /dashboard (requires auth)
│   │   └── settings.tsx     # /settings (requires auth)
│   ├── posts.tsx            # /posts layout
│   ├── posts/
│   │   ├── index.tsx        # /posts (list)
│   │   └── $postId.tsx      # /posts/:postId (dynamic)
│   └── api/
│       └── health.ts        # API route: /api/health
├── server/
│   ├── auth.ts              # Auth server functions
│   ├── posts.ts             # Posts server functions
│   └── middleware.ts        # Global middleware
├── components/
│   ├── ui/                  # Shared UI components
│   └── features/            # Feature-specific components
├── lib/
│   ├── query-client.ts      # TanStack Query configuration
│   └── utils.ts             # Shared utilities
├── router.tsx               # Router configuration
├── start.ts                 # Start configuration
└── entry-client.tsx         # Client entry point
```

### File-Based Routing Patterns

| Pattern | File | URL |
|---------|------|-----|
| Index | `index.tsx` | `/` |
| Static | `about.tsx` | `/about` |
| Dynamic | `$postId.tsx` | `/posts/:postId` |
| Nested | `posts.$postId.tsx` | `/posts/:postId` (flat file) |
| Layout | `_authed.tsx` | No URL segment, wraps children |
| Wildcard | `$.tsx` | Catch-all |
| API | `api/health.ts` | `/api/health` |

## Data Fetching Patterns

### Server Functions

Server functions are the primary data fetching mechanism in TanStack Start:

```tsx
// server/posts.ts
import { createServerFn } from '@tanstack/react-start'
import { z } from 'zod'

// GET request (default)
export const getPosts = createServerFn().handler(async () => {
  const posts = await db.posts.findMany()
  return posts
})

// POST request with validation
const CreatePostSchema = z.object({
  title: z.string().min(1),
  content: z.string().min(10),
})

export const createPost = createServerFn({ method: 'POST' })
  .validator(CreatePostSchema)
  .handler(async ({ data }) => {
    const post = await db.posts.create({ data })
    return post
  })
```

### Route Loader with Server Function

```tsx
// routes/posts/$postId.tsx
import { createFileRoute } from '@tanstack/react-router'
import { getPost } from '../../server/posts'

export const Route = createFileRoute('/posts/$postId')({
  loader: ({ params }) => getPost({ data: { id: params.postId } }),
  component: PostComponent,
})

function PostComponent() {
  const post = Route.useLoaderData()
  return <article>{post.title}</article>
}
```

### beforeLoad for Authentication

```tsx
export const Route = createFileRoute('/_authed')({
  beforeLoad: async ({ location }) => {
    const user = await getCurrentUser()
    if (!user) {
      throw redirect({
        to: '/login',
        search: { redirect: location.href },
      })
    }
    return { user }
  },
})
```

### TanStack Query Integration

```tsx
// lib/query-client.ts
import { QueryClient } from '@tanstack/react-query'

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // 1 minute
        gcTime: 5 * 60 * 1000, // 5 minutes
      },
    },
  })
}

let browserQueryClient: QueryClient | undefined

export function getQueryClient() {
  if (typeof window === 'undefined') {
    return makeQueryClient() // New instance per SSR request
  }
  if (!browserQueryClient) {
    browserQueryClient = makeQueryClient()
  }
  return browserQueryClient
}
```

### Parallel Data Loading

```tsx
export const Route = createFileRoute('/dashboard')({
  loader: async () => {
    const [stats, recentActivity] = await Promise.all([
      getStats(),
      getRecentActivity(),
    ])
    return { stats, recentActivity }
  },
})
```

### Deferred Data Loading (Streaming)

```tsx
import { defer } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  loader: async ({ params }) => {
    // Critical data - awaited
    const post = await getPost({ data: { id: params.postId } })

    // Non-critical data - deferred (streamed later)
    const relatedPosts = getRelatedPosts({ data: { postId: params.postId } })

    return {
      post,
      relatedPosts: defer(relatedPosts),
    }
  },
})

function PostComponent() {
  const { post, relatedPosts } = Route.useLoaderData()

  return (
    <article>
      <h1>{post.title}</h1>
      <Suspense fallback={<div>Loading related...</div>}>
        <Await promise={relatedPosts}>
          {(related) => <RelatedPosts posts={related} />}
        </Await>
      </Suspense>
    </article>
  )
}
```

## Server Components & Client Components

### TanStack Start's Approach: Composite Components

**Key Differences from Next.js:**

| Aspect | Next.js | TanStack Start |
|--------|---------|----------------|
| Default execution | Server-only | Isomorphic (both) |
| Tree ownership | Server decides | Client composes |
| RSC status | Core feature | Experimental |
| Composition direction | Server → Client | Client fetches, composes |
| Directive | `'use client'` to opt-in | `createServerFn` to constrain |

### Code Execution Patterns

```tsx
import {
  createServerFn,
  createServerOnlyFn,
  createClientOnlyFn,
  createIsomorphicFn,
} from '@tanstack/react-start'

// Server function (RPC - callable from anywhere)
const getSecret = createServerFn().handler(async () => {
  return process.env.API_SECRET
})

// Server-only (crashes if called on client)
const serverOnlyUtil = createServerOnlyFn(() => {
  return process.env.DATABASE_URL
})

// Client-only (crashes if called on server)
const saveToStorage = createClientOnlyFn((data: unknown) => {
  localStorage.setItem('data', JSON.stringify(data))
})

// Isomorphic (different implementations per environment)
const logger = createIsomorphicFn()
  .server((msg: string) => console.log(`[SERVER]: ${msg}`))
  .client((msg: string) => console.log(`[CLIENT]: ${msg}`))
```

## State Management

### Server State vs Client State Separation

**TanStack Query for Server State:**
- Data fetched from APIs/database
- Caching, background updates, stale management
- Synchronization across components

**Local State for Client State:**
- UI state (modals, dropdowns)
- Form drafts
- User preferences (if not persisted)

### Optimistic Updates

```tsx
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // Cancel outgoing refetches
    await queryClient.cancelQueries({ queryKey: ['todos'] })

    // Snapshot previous value
    const previousTodos = queryClient.getQueryData(['todos'])

    // Optimistically update cache
    queryClient.setQueryData(['todos'], (old: Todo[]) =>
      old.map((t) => (t.id === newTodo.id ? newTodo : t))
    )

    // Return snapshot for rollback
    return { previousTodos }
  },
  onError: (err, newTodo, context) => {
    // Rollback on error
    queryClient.setQueryData(['todos'], context?.previousTodos)
  },
  onSettled: () => {
    // Refetch after mutation settles
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  },
})
```

## Performance Optimization

### Code Splitting Strategies

```tsx
// vite.config.ts
import { tanstackRouter } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [
    tanstackRouter({
      autoCodeSplitting: true,
    }),
  ],
})
```

**What Gets Split:**
- Route components
- Error components
- Pending components

**What Stays in Main Bundle:**
- Loaders (recommended)
- Path parsing
- Search param validation

### Prefetching Patterns

```tsx
const router = createRouter({
  routeTree,
  defaultPreload: 'intent', // Prefetch on hover/focus
})

// Link-level control
<Link to="/posts" preload="intent">Posts</Link>
<Link to="/about" preload={false}>About</Link>
```

## Production Considerations

### Error Handling Patterns

```tsx
const router = createRouter({
  routeTree,
  defaultErrorComponent: ({ error, reset }) => (
    <div className="error-boundary">
      <h1>Something went wrong</h1>
      <p>{error.message}</p>
      <button onClick={reset}>Try again</button>
    </div>
  ),
})
```

### Authentication Patterns

```tsx
// server/auth.ts
import { useSession } from '@tanstack/react-start'

interface SessionData {
  userId?: string
}

export function useAppSession() {
  return useSession<SessionData>({
    name: 'app-session',
    password: process.env.SESSION_SECRET!, // 32+ characters
    cookie: {
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      httpOnly: true,
      maxAge: 60 * 60 * 24 * 7, // 7 days
    },
  })
}

export const getCurrentUser = createServerFn().handler(async () => {
  const session = await useAppSession()
  if (!session.data.userId) return null
  return db.users.findUnique({ where: { id: session.data.userId } })
})
```

### SEO Optimization

```tsx
export const Route = createFileRoute('/posts/$postId')({
  loader: getPost,
  head: ({ loaderData }) => ({
    meta: [
      { title: loaderData.title },
      { name: 'description', content: loaderData.excerpt },
      { property: 'og:title', content: loaderData.title },
      { property: 'og:description', content: loaderData.excerpt },
      { property: 'og:image', content: loaderData.coverImage },
      { property: 'og:type', content: 'article' },
      { name: 'twitter:card', content: 'summary_large_image' },
    ],
    links: [
      { rel: 'canonical', href: `https://myapp.com/posts/${loaderData.id}` },
    ],
  }),
})
```

### Deployment Configurations

**Cloudflare Workers:**
```tsx
// vite.config.ts
import { cloudflare } from '@cloudflare/vite-plugin'

export default defineConfig({
  plugins: [
    cloudflare({ viteEnvironment: { name: 'ssr' } }),
    tanstackStart(),
    viteReact(),
  ],
})
```

**Vercel/Node.js (via Nitro):**
```tsx
import { nitro } from 'nitro/vite'

export default defineConfig({
  plugins: [
    tanstackStart(),
    nitro({ preset: 'vercel' }), // or 'node-server'
    viteReact(),
  ],
})
```

## Best Practices Summary

1. **Keep loaders in the main bundle** - Don't code-split loaders
2. **Use beforeLoad sparingly** - It blocks all child routes
3. **Validate inputs server-side** - Use Zod with server functions
4. **Delegate caching to Query** - Set `defaultPreloadStaleTime: 0`
5. **Use layout routes for auth** - Single auth check for protected routes
6. **Use server functions for secrets** - Loaders are isomorphic!
7. **Parallel fetch with Promise.all** - Avoid request waterfalls
8. **Use defer for non-critical data** - Streaming improves perceived performance

## Sources

1. [TanStack Start Overview](https://tanstack.com/start/latest/docs/framework/react/overview)
2. [Server Functions Guide](https://tanstack.com/start/latest/docs/framework/react/guide/server-functions)
3. [Code Execution Patterns](https://tanstack.com/start/latest/docs/framework/react/guide/code-execution-patterns)
4. [Routing Guide](https://tanstack.com/start/latest/docs/framework/react/guide/routing)
5. [Authentication Guide](https://tanstack.com/start/latest/docs/framework/react/guide/authentication)
6. [Hosting Guide](https://tanstack.com/start/latest/docs/framework/react/guide/hosting)
7. [SEO Guide](https://tanstack.com/start/latest/docs/framework/react/guide/seo)
8. [TanStack Router Data Loading](https://tanstack.com/router/v1/docs/framework/react/guide/data-loading)
9. [Code Splitting Guide](https://tanstack.com/router/v1/docs/framework/react/guide/code-splitting)
10. [Advanced SSR with TanStack Query](https://tanstack.com/query/latest/docs/framework/react/guides/advanced-ssr)

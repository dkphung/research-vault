---
tags: [tanstack, react, anti-patterns, mistakes, security]
date: 2026-01-07
status: complete
---

# TanStack Start Anti-Patterns and Common Mistakes

## Executive Summary

TanStack Start is a client-first full-stack React framework built on TanStack Router and Vite. Coming from Next.js, developers face significant paradigm shifts around data fetching, routing conventions, and server/client boundaries. The most critical anti-patterns involve: assuming loaders are server-only (they're isomorphic), creating request waterfalls through sequential awaits, misunderstanding cache invalidation, and exposing secrets through improper code boundaries.

## Architecture Anti-Patterns

### Anti-Pattern 1: Over-Engineering File Structure

**Wrong Way:**
```
src/
  routes/
    users/
      components/
        UserCard.tsx
      hooks/
        useUser.ts
      page.tsx
```

**Right Way:**
```
src/
  routes/
    users/
      index.tsx
  components/
    users/
      UserCard.tsx
  lib/
    formatUser.ts
```

### Anti-Pattern 2: Misusing Route Context

**Wrong Way:**
```typescript
// Trying to use hooks in router configuration
const router = createRouter({
  context: {
    user: useAuth()  // Won't work - hooks can't be called here
  }
})
```

**Right Way:**
```typescript
function App() {
  const auth = useAuth()
  return (
    <RouterProvider router={router} context={{ auth }} />
  )
}
```

## Data Fetching Anti-Patterns

### Anti-Pattern 3: Creating Request Waterfalls

**Wrong Way:**
```typescript
export const Route = createFileRoute('/dashboard')({
  loader: async () => {
    // Sequential - creates waterfall
    const users = await fetchUsers()
    const posts = await fetchPosts()
    return { users, posts }
  }
})
```

**Right Way:**
```typescript
export const Route = createFileRoute('/dashboard')({
  loader: async () => {
    // Parallel - no waterfall
    const [users, posts] = await Promise.allSettled([
      fetchUsers(),
      fetchPosts()
    ])
    return {
      users: users.status === 'fulfilled' ? users.value : [],
      posts: posts.status === 'fulfilled' ? posts.value : []
    }
  }
})
```

### Anti-Pattern 4: Manual Refetching Instead of Cache Invalidation

**Wrong Way:**
```typescript
const mutation = useMutation({
  mutationFn: createTodo,
  onSuccess: () => {
    refetchTodos()  // Manual refetch
    refetchStats()
  }
})
```

**Right Way:**
```typescript
const mutation = useMutation({
  mutationFn: createTodo,
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  }
})
```

### Anti-Pattern 5: Missing Parameters in Query Keys

**Wrong Way:**
```typescript
const useTodos = (page: number) => {
  return useQuery({
    queryKey: ['todos'],  // Missing page!
    queryFn: () => apiService.getTodos(page)
  })
}
```

**Right Way:**
```typescript
const useTodos = (page: number) => {
  return useQuery({
    queryKey: ['todos', page],  // Include all params
    queryFn: () => apiService.getTodos(page)
  })
}
```

### Anti-Pattern 6: Confusing staleTime and gcTime

**Common Misconception:** `gcTime` is "how long data is cached."

**Reality:** `gcTime` only affects inactive queries (no subscribers). `staleTime` controls when data is considered stale.

**Configuration Guide:**
```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,  // 5 min: data considered fresh
      gcTime: 1000 * 60 * 30,    // 30 min: keep inactive data
    }
  }
})
```

## Component Anti-Patterns

### Anti-Pattern 7: Assuming Loaders Are Server-Only

**CRITICAL for Next.js developers:** TanStack Start loaders are isomorphic - they run on BOTH server AND client.

**Wrong Way:**
```typescript
export const Route = createFileRoute('/dashboard')({
  loader: async () => {
    // DANGER: This secret leaks to the client bundle!
    const apiKey = process.env.SECRET_API_KEY
    const data = await fetch(`/api?key=${apiKey}`)
    return data
  }
})
```

**Right Way:**
```typescript
// Use server functions for sensitive operations
const fetchSecureData = createServerFn()
  .handler(async () => {
    const apiKey = process.env.SECRET_API_KEY
    return fetch(`/api?key=${apiKey}`)
  })

export const Route = createFileRoute('/dashboard')({
  loader: async () => {
    return fetchSecureData()
  }
})
```

### Anti-Pattern 8: Hydration Mismatches

**Wrong Way:**
```tsx
function TimeDisplay() {
  // Different on server vs client = hydration mismatch
  return <span>{new Date().toLocaleString()}</span>
}
```

**Right Way:**
```tsx
function TimeDisplay() {
  const [time, setTime] = useState<string>()

  useEffect(() => {
    setTime(new Date().toLocaleString())
  }, [])

  return <span>{time ?? 'Loading...'}</span>
}
```

### Anti-Pattern 9: Wrong Server/Client Component Boundaries

**Wrong Way:**
```tsx
'use client'  // Everything below is now a client component

function ProductPage() {
  const [quantity, setQuantity] = useState(1)

  return (
    <div>
      <ProductImage />      {/* Could be server */}
      <ProductDetails />    {/* Could be server */}
      <QuantityPicker value={quantity} onChange={setQuantity} />
    </div>
  )
}
```

**Right Way:**
```tsx
// ProductPage.tsx (Server Component)
function ProductPage() {
  return (
    <div>
      <ProductImage />
      <ProductDetails />
      <QuantityPickerClient />  {/* Only this is client */}
    </div>
  )
}

// QuantityPickerClient.tsx
'use client'
function QuantityPickerClient() {
  const [quantity, setQuantity] = useState(1)
  return <QuantityPicker value={quantity} onChange={setQuantity} />
}
```

## Performance Anti-Patterns

### Anti-Pattern 10: Not Using Code Splitting

**Right Way:**
```typescript
// vite.config.ts
export default defineConfig({
  plugins: [
    tanstackStart({
      autoCodeSplitting: true
    })
  ]
})
```

### Anti-Pattern 11: SSR Memory Issues with TanStack Query

**Wrong Way:**
```typescript
// Same config for server and client
const queryClient = new QueryClient({
  defaultOptions: {
    queries: { gcTime: 1000 * 60 * 5 }
  }
})
```

**Right Way:**
```typescript
// Server: immediate cleanup
const serverQueryClient = new QueryClient({
  defaultOptions: {
    queries: { gcTime: 0 }  // Clean up immediately on server
  }
})

// Client: normal caching
const clientQueryClient = new QueryClient({
  defaultOptions: {
    queries: { gcTime: 1000 * 60 * 5 }
  }
})
```

## State Management Anti-Patterns

### Anti-Pattern 12: Using TanStack Query for UI State

**Wrong Way:**
```typescript
const { data: isModalOpen } = useQuery({
  queryKey: ['modalState'],
  queryFn: () => false,
  staleTime: Infinity
})
```

**Right Way:**
```typescript
// Use React state for UI state
const [isModalOpen, setIsModalOpen] = useState(false)

// Use Query for server state
const { data: todos } = useQuery({
  queryKey: ['todos'],
  queryFn: fetchTodos
})
```

### Anti-Pattern 13: Optimistic Updates Without Rollback

**Wrong Way:**
```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    queryClient.setQueryData(['todos'], (old) =>
      old.map(t => t.id === newTodo.id ? newTodo : t)
    )
    // No rollback if mutation fails!
  }
})
```

**Right Way:**
```typescript
const mutation = useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    await queryClient.cancelQueries({ queryKey: ['todos'] })
    const previousTodos = queryClient.getQueryData(['todos'])

    queryClient.setQueryData(['todos'], (old) =>
      old.map(t => t.id === newTodo.id ? newTodo : t)
    )

    return { previousTodos }  // Return for rollback
  },
  onError: (err, newTodo, context) => {
    queryClient.setQueryData(['todos'], context.previousTodos)
  },
  onSettled: () => {
    queryClient.invalidateQueries({ queryKey: ['todos'] })
  }
})
```

## Security Anti-Patterns

### Anti-Pattern 14: Missing CSRF Protection

TanStack Start doesn't provide built-in CSRF protection for mutations.

**Required Mitigations:**
```typescript
const session = useSession({
  password: process.env.SESSION_SECRET!,  // 32+ characters
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',  // Basic CSRF protection
    httpOnly: true,   // XSS protection
    maxAge: 7 * 24 * 60 * 60
  }
})
```

### Anti-Pattern 15: Missing Input Validation

**Wrong Way:**
```typescript
const createUser = createServerFn()
  .handler(async ({ data }) => {
    // No validation!
    return db.users.create(data)
  })
```

**Right Way:**
```typescript
const createUserSchema = z.object({
  email: z.string().email().max(255),
  password: z.string().min(8).max(100),
  name: z.string().min(1).max(100)
})

const createUser = createServerFn()
  .validator(createUserSchema)
  .handler(async ({ data }) => {
    const hashedPassword = await bcrypt.hash(data.password, 12)
    return db.users.create({
      ...data,
      password: hashedPassword
    })
  })
```

## Next.js Migration Pitfalls

### Pitfall 1: Expecting layout.tsx to Work

**Next.js (won't work):**
```
app/
  layout.tsx
  dashboard/
    layout.tsx
    page.tsx
```

**TanStack Start:**
```
routes/
  __root.tsx        // Only root layout
  dashboard/
    index.tsx
```

### Pitfall 2: Using Async Components

**Next.js Pattern:**
```tsx
async function Page() {
  const data = await fetchData()
  return <div>{data}</div>
}
```

**TanStack Start Pattern:**
```tsx
export const Route = createFileRoute('/page')({
  loader: async () => fetchData(),
  component: Page
})

function Page() {
  const data = Route.useLoaderData()
  return <div>{data}</div>
}
```

### Pitfall 3: Dynamic Route Syntax

| Next.js | TanStack Start |
|---------|----------------|
| `[slug]` | `$slug.tsx` |
| `[...slug]` | `$.tsx` |
| `[[...slug]]` | Not supported |

### Pitfall 4: Link Component Props

**Next.js:**
```tsx
<Link href="/dashboard">Dashboard</Link>
```

**TanStack Start:**
```tsx
<Link to="/dashboard">Dashboard</Link>
```

### Pitfall 5: Server Actions Syntax

**Next.js:**
```tsx
'use server'

export async function createTodo(formData: FormData) {
  // ...
}
```

**TanStack Start:**
```tsx
const createTodo = createServerFn()
  .validator(todoSchema)
  .handler(async ({ data }) => {
    // ...
  })
```

### Pitfall 6: Metadata/Head Configuration

**Next.js:**
```tsx
export const metadata = {
  title: 'My Page',
  description: 'Description'
}
```

**TanStack Start:**
```tsx
export const Route = createFileRoute('/page')({
  head: () => ({
    meta: [
      { title: 'My Page' },
      { name: 'description', content: 'Description' }
    ]
  }),
  component: Page
})
```

## Quick Reference Checklist

### Before Starting Development

- [ ] Define team conventions for routing, data loading, error handling
- [ ] Set up proper QueryClient configuration with appropriate staleTime/gcTime
- [ ] Configure code splitting (autoCodeSplitting: true)
- [ ] Set up proper session configuration with security flags

### Data Fetching

- [ ] Use `Promise.allSettled()` for parallel requests in loaders
- [ ] Include all dynamic parameters in query keys
- [ ] Use `invalidateQueries` instead of manual `refetch()`
- [ ] Don't map Query data to Redux/Context
- [ ] Don't use Query for UI state (use useState)
- [ ] Implement proper optimistic update rollbacks

### Security

- [ ] Never access secrets directly in loaders (use server functions)
- [ ] Validate all server function inputs with Zod
- [ ] Configure session cookies with httpOnly, secure, sameSite
- [ ] Implement CSRF protection for mutations
- [ ] Add rate limiting for authentication endpoints

### Performance

- [ ] Enable automatic code splitting
- [ ] Don't code-split loaders
- [ ] Use different gcTime for server (0) vs client
- [ ] Virtualize long lists of Link components
- [ ] Monitor memory usage in SSR production

### Components

- [ ] Use `<ClientOnly>` for browser-only APIs
- [ ] Avoid hydration mismatches (no Date.now(), Math.random() in SSR)
- [ ] Push client boundaries as low as possible
- [ ] Pass Server Components as children, not imports

### Next.js Migration

- [ ] Replace `layout.tsx` with `__root.tsx`
- [ ] Replace `page.tsx` with `index.tsx`
- [ ] Update dynamic routes from `[slug]` to `$slug.tsx`
- [ ] Replace `href` with `to` on Links
- [ ] Move async component logic to loaders
- [ ] Replace `'use server'` with `createServerFn()`
- [ ] Update metadata exports to `head` configuration

## Sources

1. [Tips from 8 months of TanStack/Router in production](https://swizec.com/blog/tips-from-8-months-of-tan-stack-router-in-production/)
2. [Performance & Request Waterfalls | TanStack Query](https://tanstack.com/query/latest/docs/framework/react/guides/request-waterfalls)
3. [Code Execution Patterns | TanStack Start](https://tanstack.com/start/latest/docs/framework/react/guide/code-execution-patterns)
4. [Server Functions | TanStack Start](https://tanstack.com/start/latest/docs/framework/react/guide/server-functions)
5. [Authentication | TanStack Start](https://tanstack.com/start/latest/docs/framework/react/guide/authentication)
6. [Migrate from Next.js | TanStack Start](https://tanstack.com/start/latest/docs/framework/react/migrate-from-next-js)
7. [Code Splitting | TanStack Router](https://tanstack.com/router/v1/docs/framework/react/guide/code-splitting)
8. [Optimistic Updates | TanStack Query](https://tanstack.com/query/v4/docs/framework/react/guides/optimistic-updates)

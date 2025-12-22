---
tags: [state-management]
date: 2024-12-22
status: complete
---

# Zustand with React Server Components: State Management Research

**Date:** October 20, 2025
**Author:** Claude Code
**Status:** Complete

## Context

While optimizing the client breadcrumb UX, we discovered that client names are already loaded and visible on the home page before navigation. Currently, when navigating from home to a client details page, the breadcrumb shows a skeleton loader while re-fetching the client name - data we already had.

This led to exploring whether Zustand could cache client data client-side to eliminate this redundant fetch and provide instant breadcrumb display. This research investigates how Zustand integrates with React Server Components (RSC) in Next.js 15 and whether it's the right solution for this use case.

### Current Architecture

**Breadcrumb Component** (`src/components/client-breadcrumb.tsx`):
- Server Component
- Fetches client data via `getClient(id)`
- Cached with `React.cache()` for deduplication within a request
- Shows skeleton during navigation from home page

**Home Page** (`src/app/_components/clients-list.tsx`):
- Server Component
- Already displays client names in table
- User sees client name before clicking

**Goal:** Eliminate skeleton flash when navigating from home page by reusing already-visible client data.

---

## Research Findings

### The Fundamental Constraint

**Zustand cannot be used in React Server Components.** This is a hard architectural limitation.

React Server Components (RSC):
- Execute once per request on the Node.js server
- Cannot use React hooks (`useState`, `useEffect`, custom hooks)
- Cannot access browser APIs or client-side state
- Cannot use React Context
- Return serialized component payload (not HTML)

Zustand:
- Requires `"use client"` directive
- Uses React hooks internally (`useSyncExternalStore`)
- Lives in browser JavaScript memory
- Requires Context Provider pattern for Next.js App Router
- State cleared on page refresh (unless persisted)

**Implication:** Using Zustand for the breadcrumb requires converting it from a Server Component to a Client Component.

### Official Zustand Pattern for Next.js App Router

Based on official Zustand documentation and community best practices, the recommended pattern is:

#### 1. Per-Request Store with Context Provider

```typescript
// Store definition (vanilla Zustand)
import { createStore } from 'zustand/vanilla'

export type ClientStore = {
  clients: Map<string, { id: string; name: string }>
  setClient: (id: string, data: { name: string }) => void
}

export const createClientStore = (initProps?: Partial<ClientStore>) => {
  return createStore<ClientStore>()((set) => ({
    clients: initProps?.clients ?? new Map(),
    setClient: (id, data) =>
      set((state) => ({
        clients: new Map(state.clients).set(id, data),
      })),
  }))
}
```

#### 2. Context Provider (Client Component)

```typescript
'use client'

import { createContext, useRef, useContext } from 'react'
import { useStore } from 'zustand'

const ClientStoreContext = createContext<ClientStoreApi | undefined>(undefined)

export function ClientStoreProvider({ children }: { children: ReactNode }) {
  const storeRef = useRef<ClientStoreApi>()
  if (!storeRef.current) {
    storeRef.current = createClientStore()
  }

  return (
    <ClientStoreContext.Provider value={storeRef.current}>
      {children}
    </ClientStoreContext.Provider>
  )
}

export const useClientStore = <T,>(selector: (store: ClientStore) => T): T => {
  const context = useContext(ClientStoreContext)
  if (!context) throw new Error('useClientStore must be used within Provider')
  return useStore(context, selector)
}
```

#### 3. Hydration Pattern

```typescript
// Server Component fetches data
export default async function HomePage() {
  const clients = await getClients()
  return <ClientsListWrapper initialClients={clients} />
}

// Client Component hydrates store
'use client'
export function ClientsListWrapper({ initialClients }: Props) {
  const setClient = useClientStore((state) => state.setClient)

  useEffect(() => {
    initialClients.forEach((client) => {
      setClient(client.id, { name: client.name })
    })
  }, [initialClients, setClient])

  return <ClientsListUI clients={initialClients} />
}
```

#### 4. Consumer Components

```typescript
'use client'
export function ClientBreadcrumb({ clientId }: Props) {
  const client = useClientStore((state) => state.clients.get(clientId))

  return (
    <Breadcrumb>
      <BreadcrumbItem>{client?.name || <Skeleton />}</BreadcrumbItem>
    </Breadcrumb>
  )
}
```

### Key Architecture Principles

1. **Store Per Request:** Next.js servers handle multiple requests concurrently. A global Zustand store would leak data between users. Must create store per request via Context.

2. **Client Component Boundary:** Once `"use client"` is added to a component, all its descendants become client components (unless passed via `children` prop).

3. **Hydration Required:** Server fetches data → passes to client → client populates store → components consume. Two-phase process.

4. **No SSR for Store Consumers:** Components reading from Zustand store cannot be server-rendered. Their content only appears after client hydration.

---

## Options Evaluated

### Option 1: Keep Server Component + React.cache() (Current)

**Architecture:**
```
Server Component (breadcrumb)
  ↓
await getClient(id)
  ↓
React.cache() deduplication
  ↓
Returns client.name
```

**Pros:**
✅ SEO-friendly (breadcrumb in initial HTML)
✅ Works on direct URL access
✅ Works on page refresh
✅ Simple implementation
✅ Smaller bundle (stays server-side)
✅ Deduplication via React.cache() within request

**Cons:**
❌ Shows skeleton when navigating from home page
❌ Re-fetches data already visible to user

**Code Impact:** Zero (current state)

---

### Option 2: Zustand Client Component Store

**Architecture:**
```
Server Component (page)
  ↓
Fetches clients
  ↓
Client Component Wrapper
  ↓
Populates Zustand store
  ↓
Client Breadcrumb reads from store
```

**Pros:**
✅ Instant display when navigating from home (no skeleton)
✅ Can cache multiple clients
✅ State persists during session
✅ Shared state across components

**Cons:**
❌ Breadcrumb not in initial HTML (SEO impact)
❌ Always shows skeleton on direct URL access
❌ Always shows skeleton on page refresh
❌ Converts layout to client component
❌ Forces all layout children to be client components
❌ Complex setup (provider, context, hydration)
❌ Larger bundle size
❌ More moving parts (store, provider, hooks)

**Code Impact:**
- Create: `src/stores/client-store.ts`
- Create: `src/providers/client-store-provider.tsx`
- Modify: `src/app/layout.tsx` (add provider)
- Modify: `src/app/client/layout.tsx` (convert to client)
- Modify: `src/components/client-breadcrumb.tsx` (convert to client)
- Modify: `src/app/_components/clients-list.tsx` (add hydration)

**Estimated:** ~200-250 lines of new code, 6 files modified

---

### Option 3: URL Search Params (Optimistic Display)

**Architecture:**
```
Home page link: /client/123?name=UCB
  ↓
Breadcrumb reads searchParams
  ↓
Shows name immediately (optimistic)
  ↓
Server validates via getClient() in background
```

**Pros:**
✅ Instant display when navigating from home
✅ SEO-friendly (server-rendered)
✅ Works on direct URL (falls back to fetch)
✅ Works on page refresh
✅ Simple implementation (~15 lines)
✅ Stays server component
✅ No hydration complexity

**Cons:**
⚠️ Query param visible in URL (`?name=UCB`)
⚠️ Name could be stale (mitigated by background fetch)

**Code Impact:**
- Modify: `src/app/_components/clients-list.tsx` (add `?name=` to links)
- Modify: `src/components/client-breadcrumb.tsx` (accept searchParams)
- Modify: `src/app/client/layout.tsx` (pass searchParams)

**Estimated:** ~15-20 lines modified, 3 files

---

## Detailed Comparison

| Criterion | Current (Server + cache) | Zustand (Client Store) | URL Params (Optimistic) |
|-----------|-------------------------|------------------------|------------------------|
| **UX: Navigate from home** | ❌ Skeleton flash | ✅ Instant | ✅ Instant |
| **UX: Direct URL access** | ✅ Instant | ❌ Skeleton | ✅ Instant |
| **UX: Page refresh** | ✅ Instant | ❌ Skeleton | ✅ Instant |
| **SEO: Breadcrumb in HTML** | ✅ Yes | ❌ No (client-only) | ✅ Yes |
| **Bundle size** | ✅ Smaller | ❌ Larger (+Zustand) | ✅ Smaller |
| **Code complexity** | ✅ Simple | ❌ Complex | ✅ Simple |
| **Server vs Client** | Server Component | Client Component | Server Component |
| **Implementation time** | 0 (current) | 2-3 hours | 15 minutes |
| **Maintenance burden** | ✅ Low | ❌ High | ✅ Low |
| **Testing complexity** | ✅ Simple | ❌ Complex (mock store) | ✅ Simple |
| **Works offline/cached** | ⚠️ Needs fetch | ✅ Yes (in session) | ⚠️ Needs fetch |

### Architecture Impact Assessment

**Zustand Impact on Current Architecture:**

1. **Layout Conversion:**
   - `src/app/client/layout.tsx` currently: Server Component
   - After Zustand: Must be Client Component
   - **Impact:** All children lose server component benefits

2. **Client Component Pollution:**
   ```
   layout.tsx ("use client")
     ├─ breadcrumb.tsx (forced client)
     ├─ navigation.tsx (forced client)
     └─ page.tsx (forced client unless via children)
   ```

3. **Lost Server Benefits:**
   - Cannot use Server Actions directly in layout
   - Cannot access `headers()`, `cookies()` in layout
   - Layout re-renders on client navigation (current: static)
   - Larger JavaScript bundle sent to client

---

## Best Practices for Mixing Server + Client Components

### Rule 1: Server by Default
Start with Server Components. Only add `"use client"` when absolutely necessary:
- Need interactivity (onClick, onChange, etc.)
- Need React hooks (useState, useEffect, etc.)
- Need browser APIs (localStorage, window, etc.)
- Need third-party libraries requiring client-side

### Rule 2: Server Components Can Wrap Client Components
```typescript
// ✅ GOOD: Server wraps Client
export default async function ServerPage() {
  const data = await fetchData() // Server-side
  return <ClientComponent data={data} />
}
```

### Rule 3: Client Components CANNOT Wrap Server Components
```typescript
// ❌ BAD: Client tries to render Server
'use client'
export function ClientWrapper() {
  return <ServerComponent /> // ERROR!
}

// ✅ GOOD: Use children pattern
'use client'
export function ClientWrapper({ children }: Props) {
  return <div>{children}</div>
}

// Then in Server Component:
<ClientWrapper>
  <ServerComponent /> {/* Works! */}
</ClientWrapper>
```

### Rule 4: Beware Client Component Boundary
Once `"use client"` is added, ALL descendants become client components (unless passed via `children`).

**Example of Pollution:**
```typescript
// app/layout.tsx
'use client' // ⚠️ ENTIRE APP becomes client-side!

export default function RootLayout({ children }) {
  return <html><body>{children}</body></html>
}
```

### Rule 5: Use Composition for Hybrid Patterns
```typescript
// Server Component (layout)
export default async function Layout({ children }) {
  const data = await fetch()

  return (
    <div>
      <ServerNav data={data} />
      <ClientSidebar>
        {children} {/* Server components can go here */}
      </ClientSidebar>
    </div>
  )
}
```

---

## When to Use Server vs Client Components

### Use Server Components When:
- ✅ Fetching data from databases/APIs
- ✅ Accessing backend resources directly
- ✅ Keeping sensitive information server-side (API keys, tokens)
- ✅ Reducing client JavaScript bundle
- ✅ Improving SEO and initial page load
- ✅ Static content that doesn't change client-side

### Use Client Components When:
- ✅ Need interactivity (event handlers)
- ✅ Need React hooks (state, effects, refs)
- ✅ Need browser APIs (localStorage, geolocation, etc.)
- ✅ Need third-party libraries requiring client-side execution
- ✅ Need to subscribe to real-time data

### Use Zustand Specifically When:
- ✅ **Complex shared state** - Multiple unrelated components need same data
- ✅ **Frequent updates** - State changes often during user session
- ✅ **Cross-route persistence** - State survives navigation between different routes
- ✅ **Forms/UI state** - Managing complex multi-step forms, UI toggles
- ✅ **Client-side caching** - Avoid refetching data already loaded
- ❌ **NOT for simple prop passing** - Use props/context instead
- ❌ **NOT for server data** - Use Server Components + React.cache()
- ❌ **NOT for layouts** - Causes client component pollution

**Good Zustand Use Cases:**
- Shopping cart state
- Multi-step form wizard
- Filter/search UI state
- Theme/settings preferences
- Real-time notifications state

**Bad Zustand Use Cases:**
- Breadcrumb optimization (this case!)
- Simple parent → child data flow
- Server-fetched data display
- Layout-level state

---

## Recommendations

### For Breadcrumb Optimization: Use URL Search Params

**Recommended Approach:** Option 3 (URL Search Params)

**Rationale:**
1. **Solves the problem** - Eliminates skeleton when navigating from home
2. **Preserves SSR benefits** - Breadcrumb remains in initial HTML
3. **Works everywhere** - Direct URLs, page refresh, navigation all work
4. **Simple implementation** - 15 lines of code vs 200+ for Zustand
5. **Low maintenance** - No new patterns, providers, or state management
6. **Aligns with Next.js philosophy** - Server-first, progressive enhancement

**When to Reconsider Zustand:**

If requirements change to include:
- Caching multiple clients for fast switching
- Complex client-side state beyond just name
- Need to persist state across page refreshes
- Building a SPA-style experience

Then Zustand becomes more appropriate. But for current requirements, it's overkill.

### General Guidance

**Use Zustand when:**
- You have genuine client-side state management needs
- State is complex and shared across many components
- You're building highly interactive, SPA-style features
- You need fine-grained reactivity and performance optimization

**Avoid Zustand when:**
- Server Components can solve the problem
- Data comes from server and is mostly static
- Simple prop passing or URL params would work
- You're early in a feature and complexity isn't justified yet

### Implementation Priority

For breadcrumb optimization:

1. **Immediate:** Implement URL Search Params (15 min)
2. **Monitor:** Track user experience and performance
3. **Evaluate:** After shipping, assess if more optimization needed
4. **Consider Zustand:** Only if data shows significant UX issues with direct URLs

---

## Alternative Patterns Considered

### Pattern: Next.js Link Prefetching + Streaming

Next.js Link prefetches pages on hover (default behavior). Combined with streaming, this could reduce skeleton time.

**Pros:** Uses Next.js built-in features
**Cons:** Still shows skeleton briefly, doesn't eliminate the fetch

**Verdict:** Helpful but doesn't fully solve the problem.

### Pattern: SWR/React Query + Client Components

Similar to Zustand but with built-in data fetching.

**Pros:** Handles fetching + caching together
**Cons:** Same client component pollution as Zustand, more complex

**Verdict:** No better than Zustand for this use case.

### Pattern: Cookies/Headers for Client Hints

Server reads cookie/header set by client with recently viewed clients.

**Pros:** Server-side, no client components
**Cons:** Cookie management complexity, stale data issues, privacy concerns

**Verdict:** Overcomplicated for minimal benefit.

---

## Conclusion

While Zustand is a powerful state management library, it's not the right solution for optimizing breadcrumb display in this Next.js 15 application with React Server Components.

**The architectural mismatch is clear:**
- Breadcrumb lives in Server Component layout
- Zustand requires Client Components
- Converting layout to client loses significant benefits
- Use case doesn't justify the complexity

**URL Search Params emerges as the optimal solution:**
- Solves the UX problem (instant display from home)
- Preserves SSR benefits (works on direct URLs)
- Minimal code changes (15 lines)
- Follows Next.js best practices (server-first)

**Zustand should be reserved for:**
- Complex client-side state (shopping cart, filters, etc.)
- Highly interactive features requiring frequent updates
- Scenarios where client component trade-offs are justified

For this breadcrumb optimization, simplicity wins.

---

## References

- [Zustand Official Next.js Guide](https://zustand.docs.pmnd.rs/guides/nextjs)
- [Next.js Server Components Documentation](https://nextjs.org/docs/app/building-your-application/rendering/server-components)
- [React Server Components RFC](https://github.com/reactjs/rfcs/blob/main/text/0188-server-components.md)
- GitHub Discussion: [Using Zustand in React Server Components #2200](https://github.com/pmndrs/zustand/discussions/2200)
- GitHub Discussion: [NextJS and zustand #2326](https://github.com/pmndrs/zustand/discussions/2326)

---

**Next Steps:**
- Create spec document for URL Search Params implementation
- Implement optimistic breadcrumb display
- Consider Zustand for future features where it's appropriate (cart, filters, etc.)

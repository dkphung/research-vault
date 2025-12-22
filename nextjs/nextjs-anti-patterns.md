# Next.js App Router Anti-Patterns - Research

**Date**: 2025-12-19
**Status**: Research Complete
**Scope**: Next.js 15 & 16 with App Router

## Table of Contents

- [Executive Summary](#executive-summary)
- [React Server Components & Client Components](#react-server-components--client-components)
- [Data Fetching](#data-fetching)
- [Caching](#caching)
- [Routing](#routing)
- [Performance](#performance)
- [Security](#security)
- [TypeScript & Typing](#typescript--typing)
- [State Management](#state-management)
- [Form Handling & Server Actions](#form-handling--server-actions)
- [Build & Deployment](#build--deployment)
- [Middleware](#middleware)
- [Image Optimization](#image-optimization)
- [Common Misconceptions](#common-misconceptions)
- [Quick Reference Checklist](#quick-reference-checklist)
- [Sources](#sources)

## Executive Summary

This document catalogs anti-patterns for Next.js App Router (versions 15 and 16), based on official documentation, Vercel guidance, and community best practices. The most critical anti-patterns involve: unnecessary `"use client"` directives that inflate bundle sizes, improper caching strategies after Next.js 15's default behavior changes, security vulnerabilities from exposed server-only code, and misunderstanding the async nature of `params` and `searchParams` in Next.js 15+.

---

## React Server Components & Client Components

### Anti-Pattern 1: Adding `"use client"` Unnecessarily

**Description**: Marking components with `"use client"` when they don't need client-side features (useState, useEffect, event handlers, browser APIs).

**Why It's Bad**:
- Increases JavaScript bundle size sent to the client
- Removes benefits of Server Components (direct data access, reduced client JS)
- Client Components still pre-render on the server, so there's no SSR benefit

**Correct Pattern**:

```tsx
// ❌ BAD: Static content marked as client
'use client'
export default function StaticHeader() {
  return <h1>Welcome to My Site</h1>
}

// ✅ GOOD: Only use 'use client' when you need interactivity
export default function StaticHeader() {
  return <h1>Welcome to My Site</h1>
}

// ✅ GOOD: Split interactive parts into separate components
// app/layout.tsx (Server Component)
import Search from './search'
import Logo from './logo'

export default function Layout({ children }) {
  return (
    <>
      <nav>
        <Logo />     {/* Server Component - no JS sent */}
        <Search />   {/* Client Component - only this ships JS */}
      </nav>
      <main>{children}</main>
    </>
  )
}
```

### Anti-Pattern 2: Adding `"use client"` Too High in the Component Tree

**Description**: Placing `"use client"` at the top of a layout or parent component, making all children Client Components.

**Why It's Bad**:
- All imports and child components become part of the client bundle
- Loses the ability to have Server Components nested within

**Correct Pattern**:

```tsx
// ❌ BAD: Entire layout is a Client Component
'use client'
export default function DashboardLayout({ children }) {
  const [theme, setTheme] = useState('light')
  return (
    <div className={theme}>
      <Sidebar />      {/* Now a Client Component */}
      <UserProfile />  {/* Now a Client Component */}
      {children}
    </div>
  )
}

// ✅ GOOD: Push client boundary down to smallest needed scope
// app/dashboard/layout.tsx (Server Component)
import ThemeWrapper from './theme-wrapper'
import Sidebar from './sidebar'
import UserProfile from './user-profile'

export default function DashboardLayout({ children }) {
  return (
    <ThemeWrapper>
      <Sidebar />      {/* Can be Server Component */}
      <UserProfile />  {/* Can be Server Component */}
      {children}
    </ThemeWrapper>
  )
}

// app/dashboard/theme-wrapper.tsx (Client Component)
'use client'
export default function ThemeWrapper({ children }) {
  const [theme, setTheme] = useState('light')
  return <div className={theme}>{children}</div>
}
```

### Anti-Pattern 3: Using React Context Directly in Server Components

**Description**: Attempting to use `useContext` or create context providers in Server Components.

**Why It's Bad**:
- React Context is a client-only API
- Server Components cannot use hooks

**Correct Pattern**:

```tsx
// ❌ BAD: Context in Server Component
// app/layout.tsx
import { ThemeContext } from './context'

export default function RootLayout({ children }) {
  return (
    <ThemeContext.Provider value="dark">  {/* Error! */}
      {children}
    </ThemeContext.Provider>
  )
}

// ✅ GOOD: Wrap provider in Client Component, use in layout
// app/theme-provider.tsx
'use client'
import { createContext } from 'react'

export const ThemeContext = createContext({})

export default function ThemeProvider({ children }) {
  return (
    <ThemeContext.Provider value="dark">
      {children}
    </ThemeContext.Provider>
  )
}

// app/layout.tsx (Server Component)
import ThemeProvider from './theme-provider'

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  )
}
```

### Anti-Pattern 4: Not Wrapping Third-Party Components

**Description**: Using third-party components that use client-only features directly in Server Components without a wrapper.

**Why It's Bad**:
- Many npm packages don't have the `"use client"` directive yet
- Causes build/runtime errors

**Correct Pattern**:

```tsx
// ❌ BAD: Using third-party component directly in Server Component
// app/page.tsx
import { Carousel } from 'acme-carousel'  // Uses useState internally

export default function Page() {
  return <Carousel />  // Error!
}

// ✅ GOOD: Wrap in a Client Component
// app/carousel.tsx
'use client'
import { Carousel } from 'acme-carousel'
export default Carousel

// app/page.tsx (Server Component)
import Carousel from './carousel'

export default function Page() {
  return <Carousel />  // Works!
}
```

### Anti-Pattern 5: Placing Context Providers Too High in the Tree

**Description**: Wrapping the entire `<html>` document in providers instead of just the parts that need them.

**Why It's Bad**:
- Prevents Next.js from optimizing static parts of Server Components
- Unnecessary re-renders

**Correct Pattern**:

```tsx
// ❌ BAD: Provider wraps everything
export default function RootLayout({ children }) {
  return (
    <ThemeProvider>
      <html>
        <body>{children}</body>
      </html>
    </ThemeProvider>
  )
}

// ✅ GOOD: Provider wraps only children
export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        <ThemeProvider>{children}</ThemeProvider>
      </body>
    </html>
  )
}
```

### Anti-Pattern 6: Interleaving Without Using Children Pattern

**Description**: Trying to import and render Server Components inside Client Components directly.

**Why It's Bad**:
- Server Components cannot be imported directly into Client Components
- Loses server-side benefits

**Correct Pattern**:

```tsx
// ❌ BAD: Importing Server Component in Client Component
'use client'
import ServerComponent from './server-component'  // Won't work as expected

export default function ClientComponent() {
  return <ServerComponent />
}

// ✅ GOOD: Use children or slots pattern
// app/ui/modal.tsx (Client Component)
'use client'
export default function Modal({ children }) {
  return <div className="modal">{children}</div>
}

// app/page.tsx (Server Component)
import Modal from './ui/modal'
import ServerContent from './server-content'

export default function Page() {
  return (
    <Modal>
      <ServerContent />  {/* Server Component passed as children */}
    </Modal>
  )
}
```

---

## Data Fetching

### Anti-Pattern 7: Fetching from Route Handlers in Server Components

**Description**: Making a network request from a Server Component to your own Route Handler.

**Why It's Bad**:
- Creates unnecessary network overhead
- Both run securely on the server - no need for the extra hop
- Adds latency

**Correct Pattern**:

```tsx
// ❌ BAD: Server Component fetching from Route Handler
export default async function Page() {
  const res = await fetch('http://localhost:3000/api/data')
  const data = await res.json()
  return <div>{data.title}</div>
}

// ✅ GOOD: Call the logic directly
import { getData } from '@/lib/data'

export default async function Page() {
  const data = await getData()  // Direct database/API call
  return <div>{data.title}</div>
}

// ✅ ALSO GOOD: Fetch external APIs directly
export default async function Page() {
  const res = await fetch('https://api.example.com/data')
  const data = await res.json()
  return <div>{data.title}</div>
}
```

### Anti-Pattern 8: Client-Side Data Fetching When Server Fetching is Possible

**Description**: Using `useEffect` + `fetch` in Client Components when the data could be fetched in a Server Component.

**Why It's Bad**:
- Adds network latency (server → client → API → client)
- Shows loading states unnecessarily
- Increases bundle size with fetching logic

**Correct Pattern**:

```tsx
// ❌ BAD: Client-side fetching
'use client'
import { useEffect, useState } from 'react'

export default function Posts() {
  const [posts, setPosts] = useState([])

  useEffect(() => {
    fetch('/api/posts')
      .then(res => res.json())
      .then(setPosts)
  }, [])

  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}

// ✅ GOOD: Server Component fetches directly
import { getPosts } from '@/lib/data'

export default async function Posts() {
  const posts = await getPosts()
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

### Anti-Pattern 9: Not Using Request Memoization

**Description**: Manually caching or prop-drilling data that's fetched multiple times in the component tree.

**Why It's Bad**:
- Unnecessary complexity
- Next.js automatically memoizes `fetch` requests with the same URL and options

**Correct Pattern**:

```tsx
// ❌ BAD: Prop drilling to avoid duplicate fetches
export default async function Page() {
  const user = await getUser()
  return (
    <>
      <Header user={user} />
      <Sidebar user={user} />
      <Content user={user} />
    </>
  )
}

// ✅ GOOD: Let each component fetch what it needs
// Requests are automatically deduplicated
export default async function Page() {
  return (
    <>
      <Header />
      <Sidebar />
      <Content />
    </>
  )
}

// components/header.tsx
export default async function Header() {
  const user = await getUser()  // Memoized!
  return <header>Welcome, {user.name}</header>
}

// components/sidebar.tsx
export default async function Sidebar() {
  const user = await getUser()  // Same request, reused!
  return <aside>{user.role}</aside>
}
```

### Anti-Pattern 10: Creating Request Waterfalls

**Description**: Sequential data fetching that could be parallelized.

**Why It's Bad**:
- Increases total loading time
- Blocks rendering unnecessarily

**Correct Pattern**:

```tsx
// ❌ BAD: Sequential fetching (waterfall)
export default async function Page() {
  const user = await getUser()           // 100ms
  const posts = await getPosts(user.id)  // 200ms - waits for user
  const comments = await getComments()   // 150ms - waits for posts
  // Total: 450ms

  return <div>...</div>
}

// ✅ GOOD: Parallel fetching
export default async function Page() {
  const userPromise = getUser()
  const postsPromise = getPosts()
  const commentsPromise = getComments()

  const [user, posts, comments] = await Promise.all([
    userPromise,
    postsPromise,
    commentsPromise
  ])
  // Total: 200ms (longest request)

  return <div>...</div>
}
```

### Anti-Pattern 11: Not Using Streaming for Slow Data

**Description**: Blocking the entire page render while waiting for slow data.

**Why It's Bad**:
- Users see nothing until all data loads
- Poor perceived performance

**Correct Pattern**:

```tsx
// ❌ BAD: Entire page blocked by slow data
export default async function Page() {
  const fastData = await getFastData()    // 50ms
  const slowData = await getSlowData()    // 2000ms - blocks everything!

  return (
    <div>
      <FastSection data={fastData} />
      <SlowSection data={slowData} />
    </div>
  )
}

// ✅ GOOD: Stream slow content with Suspense
import { Suspense } from 'react'

export default async function Page() {
  const fastData = await getFastData()

  return (
    <div>
      <FastSection data={fastData} />
      <Suspense fallback={<SlowSectionSkeleton />}>
        <SlowSection />  {/* Async component that fetches own data */}
      </Suspense>
    </div>
  )
}
```

---

## Caching

### Anti-Pattern 12: Assuming `fetch` is Cached by Default (Next.js 15+)

**Description**: Relying on automatic caching behavior from Next.js 14 that changed in Next.js 15.

**Why It's Bad**:
- In Next.js 15, `fetch` requests are NOT cached by default
- Can cause unexpected re-fetching on every request
- Performance degradation

**Correct Pattern**:

```tsx
// ❌ BAD: Assuming this is cached (it's not in Next.js 15+)
export default async function Page() {
  const data = await fetch('https://api.example.com/posts')
  return <div>{data}</div>
}

// ✅ GOOD: Explicitly opt into caching
export default async function Page() {
  const data = await fetch('https://api.example.com/posts', {
    cache: 'force-cache'  // Explicitly cache
  })
  return <div>{data}</div>
}

// ✅ GOOD: Time-based revalidation
export default async function Page() {
  const data = await fetch('https://api.example.com/posts', {
    next: { revalidate: 3600 }  // Revalidate every hour
  })
  return <div>{data}</div>
}
```

### Anti-Pattern 13: Forgetting to Revalidate After Mutations

**Description**: Updating data in a Server Action without calling revalidation functions.

**Why It's Bad**:
- UI displays stale data
- Users don't see their changes

**Correct Pattern**:

```tsx
// ❌ BAD: No revalidation after mutation
'use server'
export async function createPost(formData: FormData) {
  const title = formData.get('title')
  await db.posts.create({ data: { title } })
  // UI shows stale list!
}

// ✅ GOOD: Revalidate after mutation
'use server'
import { revalidatePath, revalidateTag } from 'next/cache'

export async function createPost(formData: FormData) {
  const title = formData.get('title')
  await db.posts.create({ data: { title } })

  revalidatePath('/posts')        // Path-based
  // or
  revalidateTag('posts')          // Tag-based
}
```

### Anti-Pattern 14: Mixing Cached and Uncached Fetch in Same Route

**Description**: Having one uncached fetch makes the entire route dynamic.

**Why It's Bad**:
- Invalidates Full Route Cache for the entire page
- Static optimization benefits lost

**Correct Pattern**:

```tsx
// ❌ BAD: One no-store makes entire route dynamic
export default async function Page() {
  const staticData = await fetch('https://...', { cache: 'force-cache' })
  const dynamicData = await fetch('https://...', { cache: 'no-store' })
  // Entire page is now dynamic!

  return <div>...</div>
}

// ✅ GOOD: Isolate dynamic content in separate components
// app/page.tsx (static)
import { Suspense } from 'react'

export default async function Page() {
  const staticData = await fetch('https://...', { cache: 'force-cache' })

  return (
    <div>
      <StaticSection data={staticData} />
      <Suspense fallback={<Loading />}>
        <DynamicSection />
      </Suspense>
    </div>
  )
}

// components/dynamic-section.tsx
export default async function DynamicSection() {
  const data = await fetch('https://...', { cache: 'no-store' })
  return <div>{data}</div>
}
```

### Anti-Pattern 15: Using `router.refresh()` Expecting Data Revalidation

**Description**: Calling `router.refresh()` and expecting Data Cache to be invalidated.

**Why It's Bad**:
- `router.refresh()` only refreshes the Router Cache (client-side)
- Does NOT invalidate the Data Cache or Full Route Cache

**Correct Pattern**:

```tsx
// ❌ BAD: router.refresh() doesn't revalidate data
'use client'
export default function RefreshButton() {
  const router = useRouter()

  const handleRefresh = () => {
    // Update something...
    router.refresh()  // Data is still stale!
  }

  return <button onClick={handleRefresh}>Refresh</button>
}

// ✅ GOOD: Use Server Action with revalidation
'use client'
import { refreshData } from './actions'

export default function RefreshButton() {
  return (
    <form action={refreshData}>
      <button type="submit">Refresh</button>
    </form>
  )
}

// actions.ts
'use server'
import { revalidatePath } from 'next/cache'

export async function refreshData() {
  revalidatePath('/')  // Invalidates all caches
}
```

### Anti-Pattern 16: Conflicting Cache Options

**Description**: Using contradictory cache options on the same fetch.

**Why It's Bad**:
- Both options are ignored
- Development mode shows warnings

**Correct Pattern**:

```tsx
// ❌ BAD: Conflicting options (both ignored!)
const data = await fetch('https://...', {
  revalidate: 3600,
  cache: 'no-store'  // Conflict!
})

// ✅ GOOD: Use one strategy
const data = await fetch('https://...', {
  next: { revalidate: 3600 }
})
// or
const data = await fetch('https://...', {
  cache: 'no-store'
})
```

---

## Routing

### Anti-Pattern 17: Placing Suspense Boundary Inside Async Components

**Description**: Putting `<Suspense>` inside the async component that does the data fetching.

**Why It's Bad**:
- Suspense boundary must be higher in the tree than the async work
- Won't show fallback while data loads

**Correct Pattern**:

```tsx
// ❌ BAD: Suspense inside async component
async function BlogPosts() {
  const posts = await getPosts()

  return (
    <Suspense fallback={<Loading />}>  {/* Won't work! */}
      <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
    </Suspense>
  )
}

// ✅ GOOD: Suspense wraps the async component
import { Suspense } from 'react'

export default function Page() {
  return (
    <Suspense fallback={<Loading />}>
      <BlogPosts />
    </Suspense>
  )
}

async function BlogPosts() {
  const posts = await getPosts()
  return <ul>{posts.map(p => <li key={p.id}>{p.title}</li>)}</ul>
}
```

### Anti-Pattern 18: Using Client Hooks for Request Data in Server Components

**Description**: Using `useSearchParams` or `useParams` in Server Components.

**Why It's Bad**:
- Hooks are client-only
- Server Components have dedicated APIs

**Correct Pattern**:

```tsx
// ❌ BAD: Using hook in Server Component
import { useSearchParams } from 'next/navigation'

export default function Page() {
  const searchParams = useSearchParams()  // Error!
  return <div>{searchParams.get('query')}</div>
}

// ✅ GOOD: Use props in Server Components
export default function Page({
  params,
  searchParams
}: {
  params: Promise<{ slug: string }>
  searchParams: Promise<{ query?: string }>
}) {
  const { query } = await searchParams
  return <div>{query}</div>
}
```

### Anti-Pattern 19: Using searchParams in Layouts

**Description**: Expecting `searchParams` prop in layout components.

**Why It's Bad**:
- Layouts don't receive `searchParams` prop
- Layouts aren't re-rendered on navigation (could be stale)

**Correct Pattern**:

```tsx
// ❌ BAD: searchParams in layout
export default function Layout({
  children,
  searchParams  // This doesn't exist!
}: {
  children: React.ReactNode
  searchParams: { [key: string]: string }
}) {
  return <div>{searchParams.query}</div>  // Error!
}

// ✅ GOOD: Use searchParams in Page components only
// app/layout.tsx
export default function Layout({ children }) {
  return <div>{children}</div>
}

// app/page.tsx
export default async function Page({
  searchParams
}: {
  searchParams: Promise<{ query?: string }>
}) {
  const { query } = await searchParams
  return <div>{query}</div>
}
```

### Anti-Pattern 20: Not Handling Parallel Route Slot Mismatches

**Description**: Forgetting to handle when a parallel route slot doesn't match the current URL.

**Why It's Bad**:
- Slot remains visible with stale content
- Can cause confusing UI states

**Correct Pattern**:

```tsx
// ❌ BAD: No default for parallel slot
// app/layout.tsx
export default function Layout({
  children,
  modal
}: {
  children: React.ReactNode
  modal: React.ReactNode
}) {
  return (
    <>
      {children}
      {modal}  {/* What happens when no modal route matches? */}
    </>
  )
}

// ✅ GOOD: Add default.tsx to handle non-matching routes
// app/@modal/default.tsx
export default function Default() {
  return null  // Renders nothing when no modal route matches
}
```

---

## Performance

### Anti-Pattern 21: Hydration Mismatches

**Description**: Server and client rendering different content (using `Date.now()`, `Math.random()`, or browser-only APIs during render).

**Why It's Bad**:
- Causes hydration errors
- In production, entire page falls back to client rendering
- Flash of content, slower Time to Interactive

**Correct Pattern**:

```tsx
// ❌ BAD: Different content on server vs client
export default function Page() {
  return <div>Current time: {Date.now()}</div>  // Mismatch!
}

// ❌ BAD: Browser API during render
export default function Page() {
  const width = window.innerWidth  // Error on server!
  return <div>Width: {width}</div>
}

// ✅ GOOD: Use useEffect for client-only values
'use client'
import { useState, useEffect } from 'react'

export default function Page() {
  const [time, setTime] = useState<number | null>(null)

  useEffect(() => {
    setTime(Date.now())
  }, [])

  return <div>Current time: {time ?? 'Loading...'}</div>
}

// ✅ GOOD: Suppress hydration warning for intentional mismatches
export default function Page() {
  return (
    <time suppressHydrationWarning>
      {new Date().toLocaleDateString()}
    </time>
  )
}
```

### Anti-Pattern 22: Not Using `@next/bundle-analyzer`

**Description**: Not analyzing bundle size to identify large dependencies.

**Why It's Bad**:
- Large bundles slow down page loads
- May include unused code
- Hard to identify problems without visualization

**Correct Pattern**:

```bash
# Install analyzer
npm install @next/bundle-analyzer

# next.config.js
const withBundleAnalyzer = require('@next/bundle-analyzer')({
  enabled: process.env.ANALYZE === 'true',
})
module.exports = withBundleAnalyzer({})

# Run analysis
ANALYZE=true npm run build
```

### Anti-Pattern 23: Not Using Dynamic Imports for Large Libraries

**Description**: Importing large client-side libraries at the top level.

**Why It's Bad**:
- Increases initial bundle size
- Loads code that may not be needed immediately

**Correct Pattern**:

```tsx
// ❌ BAD: Top-level import of large library
import { Chart } from 'heavy-charting-library'  // 500KB!

export default function Dashboard() {
  return <Chart data={data} />
}

// ✅ GOOD: Dynamic import
import dynamic from 'next/dynamic'

const Chart = dynamic(() => import('heavy-charting-library').then(m => m.Chart), {
  loading: () => <ChartSkeleton />,
  ssr: false  // If library doesn't support SSR
})

export default function Dashboard() {
  return <Chart data={data} />
}
```

### Anti-Pattern 24: Not Using the `priority` Prop for Above-the-Fold Images

**Description**: Letting LCP images lazy-load.

**Why It's Bad**:
- Delays Largest Contentful Paint (LCP)
- Hurts Core Web Vitals

**Correct Pattern**:

```tsx
// ❌ BAD: Hero image lazy loads by default
import Image from 'next/image'

export default function Hero() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero"
      width={1200}
      height={600}
    />
  )
}

// ✅ GOOD: Priority loading for above-the-fold images
import Image from 'next/image'

export default function Hero() {
  return (
    <Image
      src="/hero.jpg"
      alt="Hero"
      width={1200}
      height={600}
      priority  // Loads immediately, no lazy loading
    />
  )
}
```

---

## Security

### Anti-Pattern 25: Hardcoding Secrets in Server Actions

**Description**: Defining secrets directly in code rather than using environment variables.

**Why It's Bad**:
- Secrets may be inlined in compiled function output
- Recent CVE-2025-55183 showed compiled source could leak to clients
- Vulnerable to code inspection

**Correct Pattern**:

```tsx
// ❌ BAD: Hardcoded secrets
'use server'
export async function callAPI() {
  const response = await fetch('https://api.example.com', {
    headers: {
      Authorization: 'Bearer sk_live_abc123'  // NEVER do this!
    }
  })
}

// ✅ GOOD: Use environment variables
'use server'
export async function callAPI() {
  const response = await fetch('https://api.example.com', {
    headers: {
      Authorization: `Bearer ${process.env.API_SECRET}`
    }
  })
}
```

### Anti-Pattern 26: Exposing Server-Only Code to Client

**Description**: Not protecting server-only modules from being imported in Client Components.

**Why It's Bad**:
- Secrets and sensitive logic can leak to client bundle
- Only `NEXT_PUBLIC_` prefixed env vars are safe for client

**Correct Pattern**:

```tsx
// ❌ BAD: No protection on server-only code
// lib/db.ts
export async function getSecretData() {
  const apiKey = process.env.API_SECRET  // Could leak!
  return fetch(`https://api.example.com?key=${apiKey}`)
}

// ✅ GOOD: Use server-only package
// lib/db.ts
import 'server-only'  // Build error if imported in client!

export async function getSecretData() {
  const apiKey = process.env.API_SECRET
  return fetch(`https://api.example.com?key=${apiKey}`)
}
```

Install with: `npm install server-only`

### Anti-Pattern 27: Not Validating Server Action Inputs

**Description**: Trusting arguments passed to Server Actions without validation.

**Why It's Bad**:
- Server Actions are public API endpoints
- Malicious users can call them directly with arbitrary data
- SQL injection, XSS, business logic bypasses

**Correct Pattern**:

```tsx
// ❌ BAD: Trusting client input
'use server'
export async function updateUser(userId: string, data: unknown) {
  await db.users.update({
    where: { id: userId },  // Could be any user!
    data: data as UserData  // Could be malicious!
  })
}

// ✅ GOOD: Validate everything
'use server'
import { z } from 'zod'
import { auth } from '@/lib/auth'

const updateSchema = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email()
})

export async function updateUser(userId: string, rawData: unknown) {
  // Verify authorization
  const session = await auth()
  if (!session || session.user.id !== userId) {
    throw new Error('Unauthorized')
  }

  // Validate input
  const data = updateSchema.parse(rawData)

  await db.users.update({
    where: { id: userId },
    data
  })
}
```

### Anti-Pattern 28: Passing Too Much Data to Client Components

**Description**: Sending entire database objects to Client Components instead of minimal DTOs.

**Why It's Bad**:
- Exposes private fields (password hashes, internal IDs, etc.)
- Larger data transfer than necessary
- Security risk

**Correct Pattern**:

```tsx
// ❌ BAD: Passing entire user object
export default async function Page() {
  const user = await db.users.findUnique({ where: { id } })
  return <UserProfile user={user} />  // Includes passwordHash, apiKeys, etc!
}

// ✅ GOOD: Create minimal DTOs
// lib/dto.ts
import 'server-only'

export async function getUserDTO(id: string) {
  const user = await db.users.findUnique({ where: { id } })
  return {
    name: user.name,
    avatar: user.avatar,
    // Only public fields
  }
}

// app/page.tsx
export default async function Page() {
  const user = await getUserDTO(id)
  return <UserProfile user={user} />  // Only safe data
}
```

### Anti-Pattern 29: Missing CSRF Protection for Custom API Routes

**Description**: Not validating request origin for sensitive endpoints.

**Why It's Bad**:
- Cross-site request forgery attacks possible
- Server Actions have built-in protection, but Route Handlers don't

**Correct Pattern**:

```tsx
// ❌ BAD: No origin validation
export async function POST(request: Request) {
  const data = await request.json()
  await updateDatabase(data)  // Vulnerable to CSRF!
}

// ✅ GOOD: Validate origin
export async function POST(request: Request) {
  const origin = request.headers.get('origin')
  const host = request.headers.get('host')

  // Validate origin matches host
  if (!origin || new URL(origin).host !== host) {
    return new Response('Forbidden', { status: 403 })
  }

  const data = await request.json()
  await updateDatabase(data)
}

// ✅ BETTER: Use Server Actions (CSRF protection built-in)
'use server'
export async function updateData(formData: FormData) {
  // Automatically protected
}
```

---

## TypeScript & Typing

### Anti-Pattern 30: Not Awaiting Async `params` and `searchParams` (Next.js 15+)

**Description**: Accessing `params` or `searchParams` synchronously in Next.js 15+.

**Why It's Bad**:
- Breaking change in Next.js 15: these are now Promises
- TypeScript errors during build
- Will be deprecated in future versions

**Correct Pattern**:

```tsx
// ❌ BAD: Synchronous access (works in 14, breaks in 15)
export default function Page({
  params
}: {
  params: { slug: string }  // Wrong type!
}) {
  return <div>{params.slug}</div>  // Error!
}

// ✅ GOOD: Await the Promise (Next.js 15+)
export default async function Page({
  params
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = await params
  return <div>{slug}</div>
}

// ✅ GOOD: In Client Components, use React.use()
'use client'
import { use } from 'react'

export default function Page({
  params
}: {
  params: Promise<{ slug: string }>
}) {
  const { slug } = use(params)
  return <div>{slug}</div>
}
```

### Anti-Pattern 31: Not Using Route-Aware Type Helpers

**Description**: Manually typing page props instead of using Next.js provided types.

**Why It's Bad**:
- Prone to errors
- Doesn't stay in sync with actual route segments
- More maintenance

**Correct Pattern**:

```tsx
// ❌ BAD: Manual typing
export default async function Page({
  params,
  searchParams
}: {
  params: Promise<{ id: string }>
  searchParams: Promise<{ query?: string }>
}) {
  // ...
}

// ✅ GOOD: Use Next.js type helpers
import type { PageProps } from '@/types/next'
// Or use the built-in helpers:
// PageProps<'/route'>, LayoutProps<'/route'>, RouteContext<'/route'>

export default async function Page({ params, searchParams }: PageProps<'/products/[id]'>) {
  const { id } = await params
  // TypeScript knows id is string
}
```

### Anti-Pattern 32: Type Assertions Instead of Parsing

**Description**: Using `as` to assert param types instead of properly parsing.

**Why It's Bad**:
- Params are always strings from the URL
- Type assertions don't convert values
- Runtime errors when expecting numbers

**Correct Pattern**:

```tsx
// ❌ BAD: Type assertion
export default async function Page({
  params
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const numericId = id as number  // Still a string!
  await getProduct(numericId)     // Type error at runtime
}

// ✅ GOOD: Parse the value
export default async function Page({
  params
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const numericId = Number(id)

  if (isNaN(numericId)) {
    notFound()
  }

  await getProduct(numericId)
}
```

---

## State Management

### Anti-Pattern 33: Using Global Redux/Zustand Stores

**Description**: Defining stores as global variables in App Router.

**Why It's Bad**:
- Server renders are shared across requests
- User A's data could leak to User B
- Stale state between requests

**Correct Pattern**:

```tsx
// ❌ BAD: Global store
// store.ts
export const store = createStore()  // Shared across requests!

// ✅ GOOD: Create store per request
// store.ts
import { createStore } from 'zustand'

export const createAppStore = () => createStore((set) => ({
  // ...state
}))

// providers.tsx
'use client'
import { createContext, useContext, useRef } from 'react'

const StoreContext = createContext(null)

export function StoreProvider({ children }) {
  const storeRef = useRef()
  if (!storeRef.current) {
    storeRef.current = createAppStore()
  }

  return (
    <StoreContext.Provider value={storeRef.current}>
      {children}
    </StoreContext.Provider>
  )
}
```

### Anti-Pattern 34: Reading/Writing Redux Store in Server Components

**Description**: Trying to access client-side state stores in Server Components.

**Why It's Bad**:
- Server Components cannot use hooks or context
- Violates the architecture of App Router

**Correct Pattern**:

```tsx
// ❌ BAD: Using store in Server Component
// app/page.tsx (Server Component)
import { useStore } from '@/store'

export default function Page() {
  const user = useStore(state => state.user)  // Error!
  return <div>{user.name}</div>
}

// ✅ GOOD: Fetch data on server, hydrate store on client
// app/page.tsx
export default async function Page() {
  const user = await getUser()  // Server-side data fetch

  return (
    <StoreHydrator user={user}>
      <UserProfile />
    </StoreHydrator>
  )
}

// StoreHydrator.tsx
'use client'
export function StoreHydrator({ user, children }) {
  const setUser = useStore(state => state.setUser)

  useEffect(() => {
    setUser(user)
  }, [user])

  return children
}
```

### Anti-Pattern 35: Stacking Too Many Context Providers

**Description**: Nesting 5+ context providers for different concerns.

**Why It's Bad**:
- Each provider can cause re-renders
- Hard to maintain
- Performance overhead

**Correct Pattern**:

```tsx
// ❌ BAD: Provider hell
export default function App({ children }) {
  return (
    <AuthProvider>
      <ThemeProvider>
        <UserProvider>
          <CartProvider>
            <NotificationProvider>
              <ModalProvider>
                {children}
              </ModalProvider>
            </NotificationProvider>
          </CartProvider>
        </UserProvider>
      </ThemeProvider>
    </AuthProvider>
  )
}

// ✅ GOOD: Use Zustand or similar for global state
// store.ts
export const useStore = create((set) => ({
  user: null,
  theme: 'light',
  cart: [],
  notifications: [],
  // All state in one place
}))

// ✅ OR: Combine related providers
export default function App({ children }) {
  return (
    <AuthProvider>
      <UIProvider>  {/* Combines theme, modals, notifications */}
        {children}
      </UIProvider>
    </AuthProvider>
  )
}
```

---

## Form Handling & Server Actions

### Anti-Pattern 36: Creating Route Handlers for Form Submissions

**Description**: Using Route Handlers to handle form submissions from Client Components instead of Server Actions.

**Why It's Bad**:
- Adds unnecessary complexity
- No progressive enhancement
- More boilerplate code

**Correct Pattern**:

```tsx
// ❌ BAD: Route Handler for form
// app/api/submit/route.ts
export async function POST(request: Request) {
  const data = await request.json()
  await saveData(data)
  return Response.json({ success: true })
}

// component.tsx
'use client'
export function Form() {
  const handleSubmit = async (e) => {
    e.preventDefault()
    await fetch('/api/submit', { method: 'POST', body: JSON.stringify(data) })
  }
  return <form onSubmit={handleSubmit}>...</form>
}

// ✅ GOOD: Server Action
// actions.ts
'use server'
export async function submitForm(formData: FormData) {
  const name = formData.get('name')
  await saveData({ name })
  revalidatePath('/')
}

// component.tsx
import { submitForm } from './actions'

export function Form() {
  return (
    <form action={submitForm}>
      <input name="name" />
      <button type="submit">Submit</button>
    </form>
  )
}
```

### Anti-Pattern 37: Using `redirect` Inside Try/Catch

**Description**: Wrapping `redirect()` calls in try/catch blocks.

**Why It's Bad**:
- `redirect()` throws a special Next.js error to trigger navigation
- Try/catch catches this "error" and prevents the redirect

**Correct Pattern**:

```tsx
// ❌ BAD: Redirect in try/catch
'use server'
import { redirect } from 'next/navigation'

export async function createPost(formData: FormData) {
  try {
    await db.posts.create({ data: { title: formData.get('title') } })
    redirect('/posts')  // Caught by catch block!
  } catch (error) {
    console.error(error)  // Logs redirect "error"
    // Redirect never happens
  }
}

// ✅ GOOD: Redirect outside try/catch
'use server'
import { redirect } from 'next/navigation'

export async function createPost(formData: FormData) {
  let shouldRedirect = false

  try {
    await db.posts.create({ data: { title: formData.get('title') } })
    shouldRedirect = true
  } catch (error) {
    console.error(error)
    return { error: 'Failed to create post' }
  }

  if (shouldRedirect) {
    redirect('/posts')
  }
}
```

### Anti-Pattern 38: Returning JSX from Server Actions

**Description**: Embedding HTML/JSX inside Server Actions.

**Why It's Bad**:
- Server Actions should handle data logic only
- Violates separation of concerns
- Harder to test and maintain

**Correct Pattern**:

```tsx
// ❌ BAD: JSX in Server Action
'use server'
export async function getProducts() {
  const products = await db.products.findMany()
  return (
    <ul>
      {products.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  )
}

// ✅ GOOD: Return data, render in component
'use server'
export async function getProducts() {
  return db.products.findMany()
}

// component.tsx
export default async function ProductList() {
  const products = await getProducts()
  return (
    <ul>
      {products.map(p => <li key={p.id}>{p.name}</li>)}
    </ul>
  )
}
```

### Anti-Pattern 39: Using Server Actions for Parallel Data Fetching

**Description**: Calling multiple Server Actions in parallel for data fetching.

**Why It's Bad**:
- Server Actions are dispatched and awaited one at a time (current implementation)
- Causes waterfalls
- Server Components are better for parallel data fetching

**Correct Pattern**:

```tsx
// ❌ BAD: Parallel Server Action calls
'use client'
export function Dashboard() {
  const [data, setData] = useState({ users: [], posts: [] })

  useEffect(() => {
    // These run sequentially!
    Promise.all([
      getUsers(),    // Server Action
      getPosts()     // Server Action
    ]).then(([users, posts]) => setData({ users, posts }))
  }, [])
}

// ✅ GOOD: Use Server Component for parallel fetching
export default async function Dashboard() {
  // These run in parallel
  const [users, posts] = await Promise.all([
    getUsers(),
    getPosts()
  ])

  return (
    <div>
      <UserList users={users} />
      <PostList posts={posts} />
    </div>
  )
}

// ✅ ALSO GOOD: Single Server Action with parallel work inside
'use server'
export async function getDashboardData() {
  const [users, posts] = await Promise.all([
    db.users.findMany(),
    db.posts.findMany()
  ])
  return { users, posts }
}
```

---

## Build & Deployment

### Anti-Pattern 40: Using SSR Without Caching

**Description**: Dynamic rendering on every request without any caching layer.

**Why It's Bad**:
- Every request hits the server
- Higher latency (TTFB)
- More server costs

**Correct Pattern**:

```tsx
// ❌ BAD: Always dynamic, no caching
export const dynamic = 'force-dynamic'

export default async function Page() {
  const data = await getData()  // Fetched on every request
  return <div>{data}</div>
}

// ✅ GOOD: Use ISR or time-based revalidation
export const revalidate = 60  // Revalidate every 60 seconds

export default async function Page() {
  const data = await getData()
  return <div>{data}</div>
}

// ✅ GOOD: Use on-demand revalidation
export default async function Page() {
  const data = await fetch('https://...', {
    next: { tags: ['products'] }
  })
  return <div>{data}</div>
}

// In Server Action after mutation:
revalidateTag('products')
```

### Anti-Pattern 41: Static Export with Dynamic Features

**Description**: Using `output: 'export'` while relying on dynamic features.

**Why It's Bad**:
- Static export doesn't support ISR, API routes, or dynamic rendering
- Build fails or features silently don't work

**Correct Pattern**:

```js
// ❌ BAD: Static export with dynamic features
// next.config.js
module.exports = {
  output: 'export'  // Then using Server Actions, ISR, etc.
}

// ✅ GOOD: Match output mode to features used
// For static sites:
module.exports = {
  output: 'export'
  // Only use: Static pages, client-side fetch, no Server Actions
}

// For dynamic apps:
module.exports = {
  // Default output, supports all features
}
```

### Anti-Pattern 42: Skipping `generateStaticParams` for Known Dynamic Routes

**Description**: Not pre-generating static pages for known dynamic segments.

**Why It's Bad**:
- Pages generated on-demand instead of at build time
- Slower first visits

**Correct Pattern**:

```tsx
// ❌ BAD: Dynamic route without static generation
// app/products/[id]/page.tsx
export default async function Page({ params }) {
  const { id } = await params
  const product = await getProduct(id)
  return <div>{product.name}</div>
}
// Every product page generated on first request

// ✅ GOOD: Pre-generate known pages
export async function generateStaticParams() {
  const products = await getProducts()
  return products.map((product) => ({
    id: product.id
  }))
}

export default async function Page({ params }) {
  const { id } = await params
  const product = await getProduct(id)
  return <div>{product.name}</div>
}
// Known products pre-rendered at build time
```

---

## Middleware

### Anti-Pattern 43: Running Middleware on Every Route

**Description**: Not using matcher to scope middleware execution.

**Why It's Bad**:
- Adds latency to every request
- Unnecessary processing for static assets
- Higher costs

**Correct Pattern**:

```tsx
// ❌ BAD: Middleware runs on everything
// middleware.ts
export function middleware(request: NextRequest) {
  // Runs on every request including _next/static, images, etc.
}

// ✅ GOOD: Scope with matcher
export function middleware(request: NextRequest) {
  // Only runs on matched routes
}

export const config = {
  matcher: [
    // Match all routes except static files
    '/((?!_next/static|_next/image|favicon.ico).*)',
    // Or be specific:
    '/dashboard/:path*',
    '/api/:path*'
  ]
}
```

### Anti-Pattern 44: Heavy Computation in Middleware

**Description**: Performing expensive operations in middleware.

**Why It's Bad**:
- Blocks every matched request
- Middleware should be fast
- Edge runtime has CPU limits

**Correct Pattern**:

```tsx
// ❌ BAD: Heavy work in middleware
export async function middleware(request: NextRequest) {
  const data = await fetchLargeDataset()  // Slow!
  const result = expensiveCalculation(data)  // Blocks
  // ...
}

// ✅ GOOD: Keep middleware lightweight
export async function middleware(request: NextRequest) {
  // Quick checks only
  const token = request.cookies.get('token')

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Defer heavy work to the route
  return NextResponse.next()
}
```

### Anti-Pattern 45: Using Node.js APIs in Edge Middleware

**Description**: Using Node.js-specific APIs in middleware (which runs on Edge by default).

**Why It's Bad**:
- Edge runtime doesn't support all Node.js APIs
- Build/runtime errors

**Correct Pattern**:

```tsx
// ❌ BAD: Node.js API in Edge middleware
import { readFileSync } from 'fs'  // Not available on Edge!
import jwt from 'jsonwebtoken'      // Uses Node crypto!

export function middleware(request: NextRequest) {
  const token = jwt.verify(...)  // Error!
}

// ✅ GOOD: Use Edge-compatible alternatives
import { jwtVerify } from 'jose'  // Edge-compatible!

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('token')
  const verified = await jwtVerify(token, secret)
  // ...
}

// ✅ OR: Use Node.js runtime (Next.js 15.5+)
export const config = {
  runtime: 'nodejs'  // Opt into Node.js runtime
}
```

---

## Image Optimization

### Anti-Pattern 46: Overly Permissive Remote Patterns

**Description**: Using wildcard patterns that allow any image URL.

**Why It's Bad**:
- Security risk - malicious actors could abuse your image optimization
- Unexpected costs from optimizing arbitrary images

**Correct Pattern**:

```js
// ❌ BAD: Too permissive
// next.config.js
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: '**'  // Allows any host!
      }
    ]
  }
}

// ✅ GOOD: Specific patterns
module.exports = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'images.example.com',
        pathname: '/uploads/**'
      },
      {
        protocol: 'https',
        hostname: 'cdn.trusted-source.com'
      }
    ]
  }
}
```

### Anti-Pattern 47: Not Specifying Image Dimensions

**Description**: Using `next/image` without width/height or fill.

**Why It's Bad**:
- Causes layout shift (CLS)
- Browser can't reserve space before image loads

**Correct Pattern**:

```tsx
// ❌ BAD: No dimensions
<Image
  src="/photo.jpg"
  alt="Photo"
/>  // Error: width and height required

// ✅ GOOD: Explicit dimensions
<Image
  src="/photo.jpg"
  alt="Photo"
  width={800}
  height={600}
/>

// ✅ GOOD: Fill mode for responsive images
<div className="relative h-64">
  <Image
    src="/photo.jpg"
    alt="Photo"
    fill
    className="object-cover"
  />
</div>
```

---

## Common Misconceptions

### Misconception 1: "Client Components Don't Render on the Server"

**Reality**: Client Components ARE pre-rendered on the server (like Pages Router). The `"use client"` directive doesn't mean "client-only" - it means "include in client bundle and hydrate."

### Misconception 2: "Server Components Can't Have Interactive Children"

**Reality**: Server Components can render Client Components. Use the children pattern to compose them.

### Misconception 3: "Route Handlers are the Same as API Routes"

**Reality**: Route Handlers are statically cached by default (unlike Pages Router API routes). They share route segment configuration options.

### Misconception 4: "fetch is Always Cached"

**Reality**: In Next.js 15+, fetch is NOT cached by default. You must explicitly opt-in with `cache: 'force-cache'` or `next: { revalidate: N }`.

### Misconception 5: "Layouts Re-render on Navigation"

**Reality**: Layouts persist across navigations. Only the page component changes. This is a feature for performance but can cause unexpected state persistence.

### Misconception 6: "router.refresh() Updates All Data"

**Reality**: `router.refresh()` only clears the Router Cache (client-side). It does NOT invalidate the Data Cache or Full Route Cache. Use `revalidatePath` or `revalidateTag` in Server Actions for full cache invalidation.

### Misconception 7: "Server Actions are Only for Forms"

**Reality**: Server Actions can be called from event handlers, useEffect, or anywhere in Client Components. They're not limited to form submissions.

---

## Quick Reference Checklist

### Before Starting a New Component

- [ ] Does it need interactivity? Only then add `"use client"`
- [ ] Can data be fetched on the server? Prefer Server Components
- [ ] Is this a third-party component? Wrap in Client Component if needed

### Before Fetching Data

- [ ] Am I in a Server Component? Fetch directly, no Route Handler needed
- [ ] Do I need caching? Explicitly set `cache` or `revalidate` options
- [ ] Are requests independent? Use `Promise.all` for parallel fetching
- [ ] Is any data slow? Use Suspense boundaries for streaming

### Before Creating a Server Action

- [ ] Am I validating ALL inputs with a schema library (Zod)?
- [ ] Am I verifying user authorization?
- [ ] Am I calling `revalidatePath` or `revalidateTag` after mutations?
- [ ] Is my `redirect()` outside of try/catch?

### Before Deploying

- [ ] Have I run `npm run build` to check for errors?
- [ ] Have I analyzed bundle size with `@next/bundle-analyzer`?
- [ ] Are all secrets in environment variables (not hardcoded)?
- [ ] Have I tested caching behavior in production mode?
- [ ] Am I on a patched Next.js version (check CVE-2025-55182, CVE-2025-66478)?

### Security Checklist

- [ ] Using `server-only` package for sensitive modules?
- [ ] Only passing minimal DTOs to Client Components?
- [ ] Validating Server Action inputs with Zod or similar?
- [ ] Re-authorizing users in every Server Action?
- [ ] Using `NEXT_PUBLIC_` prefix only for truly public env vars?
- [ ] Scoped `remotePatterns` for `next/image`?

---

## Sources

1. [Common mistakes with the Next.js App Router and how to fix them - Vercel](https://vercel.com/blog/common-mistakes-with-the-next-js-app-router-and-how-to-fix-them)
2. [Getting Started: Server and Client Components | Next.js](https://nextjs.org/docs/app/getting-started/server-and-client-components)
3. [Getting Started: Fetching Data | Next.js](https://nextjs.org/docs/app/getting-started/fetching-data)
4. [Data Fetching: Data Fetching Patterns and Best Practices | Next.js](https://nextjs.org/docs/app/building-your-application/data-fetching/patterns)
5. [Server Actions and Mutations | Next.js](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations)
6. [Guides: Caching | Next.js](https://nextjs.org/docs/app/guides/caching)
7. [Getting Started: Caching and Revalidating | Next.js](https://nextjs.org/docs/app/getting-started/caching-and-revalidating)
8. [Guides: Data Security | Next.js](https://nextjs.org/docs/app/guides/data-security)
9. [How to Think About Security in Next.js | Next.js Blog](https://nextjs.org/blog/security-nextjs-server-components-actions)
10. [Dynamic APIs are Asynchronous | Next.js](https://nextjs.org/docs/messages/sync-dynamic-apis)
11. [File-system conventions: Dynamic Segments | Next.js](https://nextjs.org/docs/app/api-reference/file-conventions/dynamic-routes)
12. [Routing: Middleware | Next.js](https://nextjs.org/docs/14/app/building-your-application/routing/middleware)
13. [File-system conventions: Parallel Routes | Next.js](https://nextjs.org/docs/app/building-your-application/routing/parallel-routes)
14. [Getting Started: Layouts and Pages | Next.js](https://nextjs.org/docs/app/getting-started/layouts-and-pages)
15. [Getting Started: Image Optimization | Next.js](https://nextjs.org/docs/app/getting-started/images)
16. [Next.js 15 | Next.js Blog](https://nextjs.org/blog/next-15)
17. [Next.js Security Update: December 11, 2025 | Next.js Blog](https://nextjs.org/blog/security-update-2025-12-11)
18. [Security Advisory: CVE-2025-66478 | Next.js Blog](https://nextjs.org/blog/CVE-2025-66478)
19. [10 Next.js Anti Patterns to Avoid | JavaScript in Plain English](https://javascript.plainenglish.io/10-next-js-anti-patterns-to-avoid-as-a-next-js-developer-f7828bf569d4)
20. [Redux Toolkit Setup with Next.js | Redux](https://redux.js.org/usage/nextjs)
21. [Mastering State Management with Zustand in Next.js | DEV Community](https://dev.to/mrsupercraft/mastering-state-management-with-zustand-in-nextjs-and-react-1g26)
22. [Building APIs with Next.js | Next.js Blog](https://nextjs.org/blog/building-apis-with-nextjs)
23. [Mastering Next.js 15 Caching: dynamicIO, "use cache" & More | Strapi](https://strapi.io/blog/mastering-nextjs-15-caching-dynamic-io-and-the-use-cache)
24. [Fix over-caching with Dynamic IO caching in Next.js 15 | LogRocket](https://blog.logrocket.com/dynamic-io-caching-next-js-15/)
25. [Next.js Middleware Explained: Best Practices and Examples | Pagepro](https://pagepro.co/blog/next-js-middleware-what-is-it-and-when-to-use-it/)

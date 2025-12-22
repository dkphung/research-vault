# GraphQL Client Comparison: fetch() vs graphql-request

**Date**: 2025-10-27
**Context**: Evaluating GraphQL client options for Next.js 16 server actions

## Overview

Comparing native `fetch()` API versus `graphql-request` library for calling GraphQL endpoints from Next.js server actions.

---

## Native `fetch()` API

### Advantages

**Zero Dependencies**
- No additional packages required
- Smaller bundle size
- No version compatibility concerns
- Already available in Next.js runtime

**Full Control**
- Complete control over HTTP request configuration
- Easy to add custom headers, retries, timeouts
- Transparent about what's happening under the hood
- Simple to debug with browser DevTools

**Flexibility**
- Works with any GraphQL server
- Can handle non-standard GraphQL implementations
- Easy to add middleware/interceptors
- No library-specific abstractions to learn

**Performance**
- No library overhead
- Direct HTTP request without abstraction layers
- Minimal memory footprint

### Disadvantages

**Verbose Boilerplate**
```typescript
const response = await fetch('https://api.example.com/graphql', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${token}`
  },
  body: JSON.stringify({
    query: `query GetUser($id: ID!) { user(id: $id) { name email } }`,
    variables: { id }
  })
})

const { data, errors } = await response.json()
if (errors) throw new Error(errors[0].message)
if (!response.ok) throw new Error(`HTTP error: ${response.status}`)
return data.user
```

**Manual Error Handling**
- Must manually check both `response.ok` AND `errors` field
- GraphQL returns 200 even with errors
- Easy to miss error cases
- No built-in error normalization

**No Query Syntax Highlighting**
- Queries are just strings
- No syntax highlighting in most editors
- No compile-time validation
- Typos caught at runtime

**Repetitive Code**
- Same fetch configuration repeated across actions
- Authentication headers duplicated everywhere
- Error handling logic duplicated

---

## graphql-request

### Advantages

**Minimal Boilerplate**
```typescript
import { GraphQLClient, gql } from 'graphql-request'

const client = new GraphQLClient('https://api.example.com/graphql', {
  headers: { authorization: `Bearer ${token}` }
})

export async function getUser(id: string) {
  const query = gql`
    query GetUser($id: ID!) {
      user(id: $id) { name email }
    }
  `
  return await client.request(query, { id })
}
```

**Better Developer Experience**
- `gql` template tag provides syntax highlighting (with editor extensions)
- Cleaner, more readable code
- Less boilerplate to maintain
- Centralized client configuration

**Automatic Error Handling**
- Throws on GraphQL errors automatically
- Throws on HTTP errors automatically
- Consistent error handling behavior
- No need to check `response.ok` and `errors` separately

**Lightweight**
- Only ~5KB gzipped
- Minimal performance overhead
- Simple API surface
- No complex features you won't use

**TypeScript Support**
- Works well with GraphQL Code Generator
- Type-safe queries and responses
- Better IDE autocomplete

### Disadvantages

**Additional Dependency**
- Need to install and maintain `graphql-request` package
- Another package to keep updated
- Adds to bundle size (though minimal)

**Less Transparent**
- Abstracts away HTTP details
- Harder to debug if something goes wrong
- Must understand library's error handling

**Less Flexible for Edge Cases**
- Harder to customize request behavior
- May need to drop down to `fetch()` for special cases
- Limited control over response parsing

---

## Side-by-Side Code Comparison

### Simple Query

**fetch()**
```typescript
'use server'

export async function getUser(id: string) {
  const response = await fetch('https://api.example.com/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query: `query GetUser($id: ID!) {
        user(id: $id) {
          name
          email
        }
      }`,
      variables: { id }
    })
  })

  if (!response.ok) {
    throw new Error(`HTTP error: ${response.status}`)
  }

  const { data, errors } = await response.json()

  if (errors) {
    throw new Error(errors[0].message)
  }

  return data.user
}
```

**graphql-request**
```typescript
'use server'

import { GraphQLClient, gql } from 'graphql-request'

const client = new GraphQLClient('https://api.example.com/graphql')

export async function getUser(id: string) {
  const query = gql`
    query GetUser($id: ID!) {
      user(id: $id) {
        name
        email
      }
    }
  `
  return await client.request(query, { id })
}
```

### With Authentication

**fetch()**
```typescript
'use server'

import { cookies } from 'next/headers'

export async function getProtectedData() {
  const token = (await cookies()).get('auth-token')?.value

  const response = await fetch('https://api.example.com/graphql', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`
    },
    body: JSON.stringify({
      query: `query { protectedData { id value } }`
    })
  })

  if (!response.ok) throw new Error(`HTTP error: ${response.status}`)
  const { data, errors } = await response.json()
  if (errors) throw new Error(errors[0].message)
  return data.protectedData
}
```

**graphql-request**
```typescript
'use server'

import { GraphQLClient, gql } from 'graphql-request'
import { cookies } from 'next/headers'

async function getClient() {
  const token = (await cookies()).get('auth-token')?.value
  return new GraphQLClient('https://api.example.com/graphql', {
    headers: { authorization: `Bearer ${token}` }
  })
}

export async function getProtectedData() {
  const client = await getClient()
  return await client.request(gql`
    query {
      protectedData { id value }
    }
  `)
}
```

### Multiple Queries in One File

**fetch()** - Lots of repetition
```typescript
'use server'

export async function getUser(id: string) {
  const response = await fetch('https://api.example.com/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query: `query GetUser($id: ID!) { user(id: $id) { name } }`,
      variables: { id }
    })
  })
  const { data, errors } = await response.json()
  if (errors) throw new Error(errors[0].message)
  return data.user
}

export async function getPost(id: string) {
  const response = await fetch('https://api.example.com/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query: `query GetPost($id: ID!) { post(id: $id) { title } }`,
      variables: { id }
    })
  })
  const { data, errors } = await response.json()
  if (errors) throw new Error(errors[0].message)
  return data.post
}
```

**graphql-request** - DRY
```typescript
'use server'

import { GraphQLClient, gql } from 'graphql-request'

const client = new GraphQLClient('https://api.example.com/graphql')

export async function getUser(id: string) {
  return await client.request(gql`
    query GetUser($id: ID!) { user(id: $id) { name } }
  `, { id })
}

export async function getPost(id: string) {
  return await client.request(gql`
    query GetPost($id: ID!) { post(id: $id) { title } }
  `, { id })
}
```

---

## Performance Comparison

### Bundle Size Impact
- **fetch()**: 0 KB (built-in)
- **graphql-request**: ~5 KB gzipped

### Runtime Performance
- **fetch()**: Direct HTTP request
- **graphql-request**: Minimal overhead (~1-2ms per request for serialization)

**Verdict**: Performance difference is negligible for most applications.

---

## Maintainability Comparison

### Code Volume
For a typical app with 20 GraphQL queries:
- **fetch()**: ~800-1000 lines (with error handling)
- **graphql-request**: ~400-500 lines

### Refactoring
**fetch()**: Changing endpoint URL or adding headers requires updating every function

**graphql-request**: Change client configuration in one place:
```typescript
const client = new GraphQLClient(process.env.GRAPHQL_URL, {
  headers: { 'x-api-key': process.env.API_KEY }
})
```

### Testing
**fetch()**: Need to mock `fetch` global
**graphql-request**: Need to mock `graphql-request` module or use MSW for both

**Verdict**: Both are equally testable with MSW (recommended approach)

---

## Recommendations

### Use `fetch()` if:
- You want zero dependencies
- You have 1-3 GraphQL queries total
- You need fine-grained control over HTTP behavior
- You're calling non-standard GraphQL endpoints
- You're already familiar with fetch patterns
- Bundle size is critical (<5KB matters)

### Use `graphql-request` if:
- You have 5+ GraphQL queries
- You value code readability and maintainability
- You want automatic error handling
- You want syntax highlighting for queries
- You're building a long-term project with multiple developers
- You want to pair with GraphQL Code Generator for type safety

### Middle Ground
Start with `fetch()` and migrate to `graphql-request` when:
- You have more than 3-4 GraphQL calls
- Error handling becomes repetitive
- Code maintenance becomes tedious

---

## TypeScript Type Safety (Both Options)

### With GraphQL Code Generator

Both approaches work well with `@graphql-codegen/cli`:

**fetch() + codegen**
```typescript
import type { GetUserQuery, GetUserQueryVariables } from '@/generated/graphql'

export async function getUser(id: string) {
  const response = await fetch('https://api.example.com/graphql', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      query: GetUserDocument,
      variables: { id } satisfies GetUserQueryVariables
    })
  })
  const { data, errors } = await response.json()
  if (errors) throw new Error(errors[0].message)
  return data as GetUserQuery
}
```

**graphql-request + codegen**
```typescript
import { getSdk } from '@/generated/graphql'
import { GraphQLClient } from 'graphql-request'

const client = new GraphQLClient('https://api.example.com/graphql')
const sdk = getSdk(client)

export async function getUser(id: string) {
  return await sdk.GetUser({ id })
}
```

With codegen, `graphql-request` wins on ergonomics - you get a fully typed SDK.

---

## Final Verdict

**For this Next.js 16 project**: Use **graphql-request**

**Reasoning**:
1. Better developer experience with minimal overhead
2. Scales well as GraphQL usage grows
3. Cleaner codebase with less boilerplate
4. Automatic error handling reduces bugs
5. Only 5KB - acceptable for modern web apps
6. Works seamlessly with GraphQL Code Generator

**Exception**: If you only have 1-2 GraphQL calls and want to avoid any dependency, use `fetch()`.

---

## Example Implementation for This Project

```typescript
// src/lib/graphql-client.ts
'use server'

import { GraphQLClient } from 'graphql-request'
import { cookies } from 'next/headers'
import { env } from '@/env.mjs'

export async function getGraphQLClient() {
  const token = (await cookies()).get('auth-token')?.value

  return new GraphQLClient(env.GRAPHQL_ENDPOINT, {
    headers: token ? { authorization: `Bearer ${token}` } : undefined
  })
}
```

```typescript
// src/app/actions/user.ts
'use server'

import { gql } from 'graphql-request'
import { getGraphQLClient } from '@/lib/graphql-client'

export async function getUser(id: string) {
  const client = await getGraphQLClient()

  return await client.request(gql`
    query GetUser($id: ID!) {
      user(id: $id) {
        id
        name
        email
      }
    }
  `, { id })
}
```

Don't forget to add to `src/env.mjs`:
```typescript
server: {
  GRAPHQL_ENDPOINT: z.string().url(),
  // ... other vars
}
```

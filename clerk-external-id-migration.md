# Clerk Authentication Migration: External ID Workflow for Preserving Existing User IDs

**Date:** 2026-01-16
**Context:** folder-server migration from RSS OAuth to Clerk
**Tags:** auth, clerk, migration, openFGA

---

## Executive Summary

This research explores Clerk's `external_id` feature for migrating from a custom authentication system while preserving existing user IDs. This is critical for folder-server where user IDs are embedded in OpenFGA authorization tuples and MongoDB audit metadata.

**Key Finding**: Clerk's `external_id` field combined with JWT customization (`{{user.external_id || user.id}}`) provides a clean migration path:
1. Existing users retain their legacy IDs via bulk import or client-side initialization
2. New users receive custom-generated IDs via a dedicated Identity Service
3. A single `userId` claim in the JWT works for both scenarios
4. **No OpenFGA tuple migration required** if legacy IDs are preserved

**Critical Constraint**: Clerk does NOT support synchronous hooks that block token generation. All webhooks are async. The solution must work with Clerk's async architecture.

**Recommended Approach**: Client-side initialization gate with a dedicated Identity Service - see "Recommended Approach" section below.

---

## Current System Analysis

### How User IDs Flow Today

```
HTTP Request
  ↓
authMiddleware extracts userId (from Clerk or RSS OAuth)
  ↓
ServiceContext.auth = AuthUser { userId, fullName, email, tenantCode, campusCode }
  ↓
getUserMetadata(ctx.auth) → { id: userId, fullName, email }
  ↓
Passed to:
  - Repository layer: stored in audit metadata
  - Service layer: used for authorization checks
  - OpenFGA: user entity references (format: user:{userId})
```

### Where User IDs Are Stored

| Location | Field | Impact of ID Change |
|----------|-------|---------------------|
| OpenFGA tuples | `user:{userId}` | Orphaned tuples, broken authorization |
| MongoDB role assignments | `userId` | Broken role lookups |
| MongoDB audit metadata | `meta.createdBy.id`, `meta.lastUpdatedBy.id` | Audit trail inconsistency |

### Critical Constraint

The system does **not** store users in MongoDB - only user ID references. This means we cannot perform a database migration of user IDs; they must come correctly from the auth system.

---

## Clerk's External ID Feature

### What is `external_id`?

A field on the Clerk User object that stores your legacy system's user ID:

```typescript
interface BackendUser {
  id: string;                    // Clerk's internal ID (e.g., "user_2abc123...")
  externalId: string | null;     // Your legacy system's user ID
  // ... other fields
}
```

**Characteristics:**
- Must be unique across your Clerk instance
- Optional (can be null for new users)
- Searchable via Backend API
- Accessible in JWT templates

### JWT Customization

Configure in Clerk Dashboard → Sessions → Customize session token:

```json
{
  "userId": "{{user.external_id || user.id}}"
}
```

This conditional expression:
- Returns `external_id` if set (migrated/existing users)
- Falls back to Clerk's native `id` if not set (edge case during JIT)

**Size Limit:** Custom claims limited to ~1.2KB (browser cookie constraint)

---

## Clerk's Async Architecture (Critical Constraint)

### No Synchronous Pre-Token Hooks

Clerk does **not** provide any mechanism to run custom code synchronously before a token is issued. From the official docs:

> "Webhooks are best used for things like sending a notification or updating a database, but **not for synchronous flows** where you need to know the webhook was delivered before moving on to the next step."

### What Clerk Offers

| Feature | What It Does | Blocks Token? |
|---------|--------------|---------------|
| **Webhooks** | Async notifications after events | No |
| **Session Tasks** | Built-in tasks like "choose organization" | Partial (pending state, but only for Clerk's built-in tasks) |
| **Waitlist/Allowlist** | Block sign-up entirely | Yes, but blocks ALL sign-ups |
| **clerkMiddleware** | Route protection after auth | No (token already issued) |

### Implications for Our Design

Since we cannot intercept token generation:
1. The first token a user receives will have `userId = Clerk ID` (since `external_id` is null)
2. We must set `external_id` via the Backend API after the user exists
3. The user must refresh their token to get the updated `userId`

This means **the blocking must happen in our application**, not in Clerk.

---

## Migration Approaches (Evaluated)

### Approach A: Bulk Import (Recommended for Existing Users)

Use Clerk's official migration script for importing users with their legacy IDs.

**Repository:** [github.com/clerk/migration-script](https://github.com/clerk/migration-script)

**User Schema:**
```json
[
  {
    "userId": "legacy-user-123",
    "email": "user@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "password": "$2b$10$...",
    "passwordHasher": "bcrypt"
  }
]
```

**Key Fields:**
- `userId` (required): Maps to Clerk's `externalId`
- `email` (required): Primary email address
- `password` + `passwordHasher`: For migrating password hashes (supports bcrypt, argon2, pbkdf2, scrypt)

**Rate Limits:**
- Development: 100 requests / 10 seconds
- Production: 1000 requests / 10 seconds

**Pros:**
- All existing users have `external_id` set immediately
- No race conditions with authorization
- Supports password hash migration

**Cons:**
- Requires exporting all users and passwords
- May not scale for millions of users
- Doesn't handle users created between export and go-live

---

### Approach B: Just-in-Time (JIT) Provisioning

Set `external_id` when users first log in via Clerk webhook.

**Webhook Handler:**

```typescript
import { verifyWebhook } from '@clerk/nextjs/server';
import { clerkClient } from '@clerk/clerk-sdk-node';

export async function handleUserCreated(req: Request) {
  const evt = await verifyWebhook(req);

  if (evt.type !== 'user.created') return;

  const { id: clerkUserId, email_addresses, external_id } = evt.data;

  // Skip if external_id already set (imported user)
  if (external_id) return;

  const primaryEmail = email_addresses.find(
    e => e.id === evt.data.primary_email_address_id
  )?.email_address;

  // Check legacy system for existing user
  const legacyUser = await findLegacyUserByEmail(primaryEmail);

  let externalId: string;
  if (legacyUser) {
    // Existing user: use their legacy ID
    externalId = legacyUser.id;
  } else {
    // New user: generate new ID
    externalId = generateUserId(); // e.g., nanoid() or uuid()
  }

  // Update Clerk user with externalId
  await clerkClient.users.updateUser(clerkUserId, { externalId });
}
```

**Pros:**
- No bulk data export required
- Scales to any user count
- Handles users created after migration starts

**Cons:**
- Race condition: User may make requests before webhook completes
- Requires legacy system lookup capability (by email)

---

### Approach C: Hybrid (Recommended)

Combine bulk import for known users with JIT for stragglers.

**Implementation Steps:**

1. **JWT Configuration** (Before migration):
   ```json
   { "userId": "{{user.external_id || user.id}}" }
   ```

2. **Deploy Webhook Handler** (Before migration):
   - Handles new users and missed users
   - Generates new IDs for genuinely new users
   - Looks up legacy IDs for existing users by email

3. **Bulk Import** (During maintenance window):
   - Export known users with their IDs
   - Import via Clerk migration script
   - All imported users have `external_id` set immediately

4. **Go Live**:
   - Switch auth middleware to Clerk
   - Webhook handles any users missed during import

**Handling the Race Condition:**

Between `user.created` and webhook processing, a user might make requests with Clerk's native ID. Options:

| Strategy | Complexity | User Experience |
|----------|------------|-----------------|
| Accept both ID formats temporarily | High | Seamless |
| Block actions until webhook completes | Medium | Slight delay on first action |
| Rely on webhook being fast (~<1s) | Low | Rare edge case failures |

---

## Recommended Approach: Client-Side Initialization Gate + Dedicated Identity Service

After evaluating the architectural concerns with per-service middleware (access sprawl, single point of failure, code duplication), the recommended solution is a **client-side initialization gate** combined with a **dedicated Identity Service**.

### Why Not Per-Service Middleware?

The per-service middleware approach requires every microservice to have:
- Access to the legacy user database (to look up existing users by email)
- Access to Clerk's Backend API (to set `external_id`)
- The same middleware code duplicated everywhere

This creates:
- **Security sprawl** - Every service needs credentials for legacy DB and Clerk API
- **Single point of failure** - If legacy DB is down, ALL services fail for new users
- **Code duplication** - Same middleware logic in every service
- **Tight coupling** - All services depend on user initialization logic

### The Better Architecture

Centralize the `external_id` assignment in a **single Identity Service**:

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT                                   │
│                                                                  │
│  1. User logs in via Clerk                                       │
│  2. Check: does token.userId start with "user_"?                │
│     ├─ NO  → User is ready, proceed to app                      │
│     └─ YES → BLOCK: Show loading/initializing screen            │
│              ↓                                                   │
│  3. Call Identity Service: POST /auth/initialize                │
│  4. Identity Service sets external_id in Clerk                  │
│  5. Force token refresh: getToken({ skipCache: true })          │
│  6. Now userId is correct → proceed to app                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              ALL OTHER SERVICES (folder-server, etc.)           │
│                                                                  │
│  - No special middleware needed                                  │
│  - No access to legacy DB needed                                │
│  - No access to Clerk Backend API needed                        │
│  - Just validate JWT - userId is already correct                │
└─────────────────────────────────────────────────────────────────┘
```

### JWT Template Configuration

Configure in Clerk Dashboard → Sessions → Customize session token:

```json
{
  "userId": "{{user.external_id || user.id}}",
  "clerkUserId": "{{user.id}}"
}
```

This provides:
- `userId` - Your legacy ID (or Clerk ID as fallback before mapping)
- `clerkUserId` - Always the Clerk ID (needed for `getUser`/`updateUser` API calls)

### Identity Service Implementation

This is the **only** service that needs access to the legacy user DB and Clerk Backend API:

```typescript
// identity-service/src/index.ts
import { Hono } from 'hono'
import { createClerkClient } from '@clerk/backend'
import { authMiddleware } from '@risk-and-safety/hono-rss-auth'

const app = new Hono()
const clerkClient = createClerkClient({ secretKey: process.env.CLERK_SECRET_KEY! })

app.use('*', authMiddleware({ /* config */ }))

app.post('/auth/initialize', async (ctx) => {
  const auth = ctx.var.user

  // Already has a mapped ID (doesn't start with "user_")
  if (!auth.userId.startsWith('user_')) {
    return ctx.json({ status: 'ready', userId: auth.userId })
  }

  const clerkUserId = auth.clerkUserId ?? auth.userId
  const clerkUser = await clerkClient.users.getUser(clerkUserId)

  // Already has external_id - just needs token refresh
  if (clerkUser.externalId) {
    return ctx.json({ status: 'ready', userId: clerkUser.externalId })
  }

  // Look up in legacy system by email
  const primaryEmail = clerkUser.emailAddresses.find(
    e => e.id === clerkUser.primaryEmailAddressId
  )?.emailAddress

  if (!primaryEmail) {
    return ctx.json({ error: 'User has no primary email' }, 400)
  }

  // Check if user exists in legacy system
  const legacyUser = await legacyUserDb.findByEmail(primaryEmail)
  const externalId = legacyUser?.id ?? generateNewUserId()

  // Set in Clerk
  await clerkClient.users.updateUser(clerkUserId, { externalId })

  return ctx.json({
    status: 'ready',
    userId: externalId,
    refreshRequired: true
  })
})

// Legacy user lookup - implement based on your system
async function legacyUserDb.findByEmail(email: string): Promise<{ id: string } | null> {
  // Query Neptune/relationship-v2 or wherever users are stored
  return null
}

function generateNewUserId(): string {
  return crypto.randomUUID()
}

export default app
```

### Client-Side Initialization Gate

The client blocks the user until initialization is complete:

```typescript
// app/providers/AuthInitializer.tsx
'use client'

import { useAuth } from '@clerk/nextjs'
import { useEffect, useState, ReactNode } from 'react'

const IDENTITY_SERVICE_URL = process.env.NEXT_PUBLIC_IDENTITY_SERVICE_URL

export function AuthInitializer({ children }: { children: ReactNode }) {
  const { getToken, userId, isLoaded, isSignedIn } = useAuth()
  const [isInitialized, setIsInitialized] = useState(false)
  const [isInitializing, setIsInitializing] = useState(false)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    if (!isLoaded || !isSignedIn || !userId) {
      setIsInitialized(false)
      return
    }

    // Already has a mapped ID (doesn't start with "user_")
    if (!userId.startsWith('user_')) {
      setIsInitialized(true)
      return
    }

    // Need to initialize
    const initialize = async () => {
      setIsInitializing(true)
      setError(null)

      try {
        const token = await getToken()
        const response = await fetch(`${IDENTITY_SERVICE_URL}/auth/initialize`, {
          method: 'POST',
          headers: { Authorization: `Bearer ${token}` }
        })

        if (!response.ok) {
          throw new Error('Failed to initialize session')
        }

        const data = await response.json()

        if (data.refreshRequired) {
          // Force token refresh to pick up new external_id
          await getToken({ skipCache: true })
        }

        setIsInitialized(true)
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error')
      } finally {
        setIsInitializing(false)
      }
    }

    initialize()
  }, [isLoaded, isSignedIn, userId, getToken])

  // Not signed in - render children (they'll handle auth redirect)
  if (!isSignedIn) {
    return <>{children}</>
  }

  // Still loading Clerk
  if (!isLoaded) {
    return <div>Loading...</div>
  }

  // Initializing session
  if (isInitializing) {
    return <div>Setting up your account...</div>
  }

  // Error during initialization
  if (error) {
    return (
      <div>
        <p>Failed to initialize session: {error}</p>
        <button onClick={() => window.location.reload()}>Retry</button>
      </div>
    )
  }

  // Not yet initialized (shouldn't happen, but safety check)
  if (!isInitialized && userId?.startsWith('user_')) {
    return <div>Initializing...</div>
  }

  // Ready - render the app
  return <>{children}</>
}
```

### Wrap Your App

```typescript
// app/layout.tsx
import { ClerkProvider } from '@clerk/nextjs'
import { AuthInitializer } from './providers/AuthInitializer'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <ClerkProvider>
      <AuthInitializer>
        {children}
      </AuthInitializer>
    </ClerkProvider>
  )
}
```

### Other Microservices - Simple JWT Validation Only

All other services (folder-server, relationship-v2, etc.) just validate JWTs:

```typescript
// folder-server/src/app.ts
import { Hono } from 'hono'
import { authMiddleware } from '@risk-and-safety/hono-rss-auth'

const app = new Hono()

// Just validate JWT - no special external_id logic needed
// userId will already be the legacy ID (client initialized before calling us)
app.use('*', authMiddleware({ /* config */ }))

// Business logic uses ctx.auth.userId directly
app.route('/graphql', graphqlHandler)
```

### Advantages of This Approach

| Aspect | Per-Service Middleware | Identity Service + Client Gate |
|--------|------------------------|-------------------------------|
| **Access to legacy DB** | Every service | Only Identity Service |
| **Access to Clerk API** | Every service | Only Identity Service |
| **Code duplication** | Middleware in every service | Single implementation |
| **Single point of failure** | All services fail if legacy DB down | Only initialization fails; existing users unaffected |
| **Complexity in microservices** | Each service has init logic | Services are simple JWT validators |

### Edge Cases Handled

| Scenario | Behavior |
|----------|----------|
| New user, first login | Client calls Identity Service → generates new ID → sets external_id → refresh token |
| Existing user, first Clerk login | Client calls Identity Service → looks up legacy ID → sets external_id → refresh token |
| User with external_id set | `userId` doesn't start with "user_" → client proceeds immediately |
| Bulk-imported user | external_id already set → client proceeds immediately |
| Identity Service down | Client shows error, user can retry; existing initialized users unaffected |

---

## Backend API Operations

### Creating Users with External ID

```typescript
const user = await clerkClient.users.createUser({
  emailAddress: ['user@example.com'],
  firstName: 'John',
  lastName: 'Doe',
  externalId: 'legacy-user-123',
});
```

### Looking Up Users by External ID

```typescript
const { data } = await clerkClient.users.getUserList({
  externalId: ['legacy-user-123'],
  limit: 1,
});
const user = data[0];
```

### Updating External ID

```typescript
await clerkClient.users.updateUser('user_clerk123', {
  externalId: 'new-legacy-id',
});
```

---

## Webhook Events

| Event | Trigger | Use Case |
|-------|---------|----------|
| `user.created` | New user registration | Set `external_id` via JIT |
| `user.updated` | User info changed | Sync cached user info |
| `user.deleted` | User deleted | Clean up role assignments |

**Webhook Payload:**
```typescript
interface WebhookEvent {
  data: {
    id: string;                    // Clerk user ID
    external_id: string | null;    // Your legacy ID
    email_addresses: Array<{ email_address: string }>;
    // ... other fields
  };
  type: 'user.created' | 'user.updated' | 'user.deleted';
}
```

---

## Implementation Plan for folder-server

### Phase 1: Infrastructure

1. **Configure JWT template** in Clerk Dashboard:
   ```json
   {
     "userId": "{{user.external_id || user.id}}",
     "clerkUserId": "{{user.id}}"
   }
   ```

2. **Update `@risk-and-safety/hono-rss-auth`** to extract both `userId` and `clerkUserId` claims

3. **Create Identity Service** (new microservice):
   - Single endpoint: `POST /auth/initialize`
   - Access to legacy user DB (for email → ID lookup)
   - Access to Clerk Backend API (for setting `external_id`)
   - Deploy with appropriate secrets/credentials

4. **Implement client-side AuthInitializer** component:
   - Wraps the app at the root level
   - Blocks rendering until user is initialized
   - Calls Identity Service for unmapped users
   - Forces token refresh after initialization

### Phase 2: Migration (Optional - for faster first-login)

5. **Export users from legacy system** (email → ID mapping)

6. **Bulk import** using Clerk migration script with `userId` → `externalId`

7. **Verify import** - check a sample of users have correct `external_id`

Note: Bulk import is optional. Without it, the Identity Service handles all users on first login. Bulk import just avoids the initialization step for existing users.

### Phase 3: Cutover

8. **Deploy Identity Service** to production

9. **Deploy client-side AuthInitializer** to all frontend apps

10. **Switch authentication** from RSS OAuth to Clerk

11. **Monitor** for any issues with the initialization flow

12. **Decommission legacy auth** after validation period

### What Doesn't Need to Change

- **OpenFGA tuples**: `user:{legacyId}` remains valid
- **MongoDB references**: `userId` fields unchanged
- **Service layer**: `ctx.auth.userId` works the same way
- **Authorization checks**: No code changes needed
- **folder-server, relationship-v2, etc.**: No middleware changes needed - just validate JWTs

---

## Open Questions

1. **ID Generation for New Users**: Should new users get UUIDs, sequential IDs, or match legacy format?

2. **Legacy User Lookup**: How do we look up existing users by email? Options:
   - Query Neptune/relationship-v2 directly
   - Create a dedicated user lookup service
   - Export to a simple lookup table

3. **Identity Service Hosting**: Where should the Identity Service live?
   - New standalone service
   - Endpoint in an existing service (e.g., relationship-v2)
   - Serverless function (AWS Lambda, Cloudflare Worker)

4. **Bulk Import Decision**: Do we want to bulk import existing users, or let the Identity Service handle everyone on first login?

---

## Sources

### Clerk Documentation
- [Migrate to Clerk from another platform](https://clerk.com/docs/deployments/migrate-overview)
- [Customize your session token](https://clerk.com/docs/backend-requests/custom-session-token)
- [Session tokens](https://clerk.com/docs/backend-requests/resources/session-tokens) - 60-second token lifetime
- [Force a session token refresh](https://clerk.com/docs/guides/sessions/force-token-refresh) - `getToken({ skipCache: true })`
- [Webhooks Overview](https://clerk.com/docs/guides/development/webhooks/overview) - Confirms webhooks are async
- [Sync Clerk data with webhooks](https://clerk.com/docs/guides/development/webhooks/syncing) - "not for synchronous flows"
- [Session Tasks](https://clerk.com/docs/guides/configure/session-tasks) - Built-in post-auth tasks

### Clerk Backend API
- [createUser() API Reference](https://clerk.com/docs/references/backend/user/create-user)
- [getUser() API Reference](https://clerk.com/docs/references/backend/user/get-user)
- [getUserList() API Reference](https://clerk.com/docs/references/backend/user/get-user-list)
- [updateUser() API Reference](https://clerk.com/docs/references/backend/user/update-user) - `externalId` parameter
- [JS Backend SDK - createClerkClient](https://clerk.com/docs/getting-started/quickstart.js-backend)

### Migration Tools
- [Clerk Migration Script (GitHub)](https://github.com/clerk/migration-script)

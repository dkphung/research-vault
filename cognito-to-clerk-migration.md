---
tags: [authentication]
date: 2024-12-22
status: complete
---

# Cognito to Clerk Migration Research

**Date:** 2025-10-30

**Status:** Research Complete

## Executive Summary

This document provides comprehensive research on migrating from AWS Cognito to Clerk for a multi-account setup with password-based users and enterprise SSO connections. The key findings recommend a **gradual "trickle" migration** approach that allows both systems to run in parallel, minimizing risk and user disruption.

## Current Environment

### Infrastructure
- **3 AWS Accounts**: Each with its own Cognito user pool
- **Authentication Types**:
  - Password-based users (Cognito native)
  - Enterprise SSO connections (external IdPs like UC Davis)
- **Use Cases**: User authentication + M2M token exchange

### Migration Goals
1. Consolidate to a single Clerk instance
2. Preserve existing user passwords (no forced resets)
3. Map enterprise connections to Clerk organizations (e.g., UC Davis users → UC Davis organization)
4. Support M2M authentication for service-to-service communication
5. Minimize user disruption and risk

---

## 1. Password Migration Strategy

### Clerk's Cognito Password Migration

Clerk provides **seamless password migration** without requiring users to reset credentials using a special password hasher mechanism.

#### How It Works

When creating users in Clerk from Cognito, set special fields:
- **`password_hasher`**: Set to `awscognito`
- **`password_digest`**: Format is `awscognito#<COGNITO_USER_POOL_ID>#<COGNITO_CLIENT_ID>#<identifier>`

This allows existing Cognito passwords to work immediately upon migration. When users first sign in through Clerk, their passwords are automatically re-hashed using Clerk's secure hashing algorithm.

#### Prerequisites

**CRITICAL REQUIREMENT**: AWS Cognito must have a **public client** with `ALLOW_USER_PASSWORD_AUTH` flow enabled. Without this, the migration will not work.

#### Implementation Process

Clerk provides a Node.js batch upload script that:
1. Lists all users from each Cognito user pool
2. Calls Clerk's `CreateUser` Backend API for each user
3. Sets the special password hasher and digest fields
4. Skips unconfirmed users or external provider accounts (only migrates `CONFIRMED` status)

#### Validation Strategy

**Recommended approach**:
1. Test migration with a single known user first
2. Validate in development instance
3. Test in production with a small subset
4. Monitor for authentication failures
5. Roll out to remaining users

#### Password Re-hashing

As users successfully sign in to Clerk, their passwords are automatically re-hashed with Clerk's secure algorithm, gradually improving security over time.

---

## 2. Migration Approach: Big Bang vs. Gradual

### Recommended: Trickle Migration (Gradual)

Clerk explicitly supports a **"trickle migration"** approach where both systems run in parallel for a controlled transition period.

#### How Trickle Migration Works

**Phase 1: Parallel Operation**
- Keep both Cognito and Clerk running simultaneously
- New users created in both systems (or Clerk only)
- Existing users authenticate through Cognito
- Gradually migrate active users to Clerk as they sign in

**Phase 2: Gradual Transition**
- Handle authentication routing behind the scenes
- Active users naturally migrate when they sign in
- Inactive users remain in Cognito until needed

**Phase 3: Final Cutover**
- Once sufficient users have migrated, perform bulk import of remaining users
- Decommission Cognito user pools
- Clerk becomes the single source of truth

#### Advantages of Trickle Migration

✅ **Lower Risk**: No single point of failure; can roll back if issues arise
✅ **Controlled Rollout**: Migrate users gradually, monitor for issues
✅ **No Forced Disruption**: Active users migrate naturally without password resets
✅ **Flexible Timeline**: No pressure for a single coordinated event
✅ **Better Testing**: Validate migration with real users before full rollout

#### Trade-offs

**Cost Implications**
- Both systems must run concurrently (short-term increased cost)
- **However**: Clerk charges only by Monthly Active Users (MAU), not total users
- During migration, you only pay for users who create active sessions in Clerk
- Cognito costs remain for users still authenticating there

**Inactive Users**
- Trickle migration works great for active users
- Inactive users who don't sign in during the migration window will need bulk import
- Typically handled via Basic Import/Export at the end of the migration period

#### Implementation Strategy for Parallel Systems

**User ID Management**:
- Store Cognito user IDs as `external_id` in Clerk
- Use Clerk's JWT customization to include `external_id` in tokens
- Application logic can work with either Clerk ID or `external_id`
- New users get Clerk IDs only (no `external_id`)

**Authentication Routing**:
```
User Sign-In Attempt
    ↓
Check if user exists in Clerk
    ↓
  Yes → Authenticate with Clerk
    ↓
  No → Authenticate with Cognito
       ↓
       Create user in Clerk
       ↓
       Future logins use Clerk
```

**Session Management**:
- Gradually transition frontend to use Clerk session tokens
- Backend can validate either Cognito or Clerk tokens during transition
- Use `external_id` to correlate sessions between systems

#### Detailed Trickle Migration Flow (Step-by-Step)

This section provides a concrete example of how trickle migration works in practice, clarifying the authentication flow during the parallel operation phase.

**Visual Flow:**

```
┌─────────────────────────────────────────────────────────────┐
│ User enters email/password on login page                    │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│ Frontend checks: Does this user exist in Clerk?             │
│ (Call backend API: /api/auth/check-user?email=user@example) │
└─────────────────────────────────────────────────────────────┘
                         ↓
                    ┌────┴────┐
                    │         │
           YES ←────┤ Exists? │────→ NO
            ↓       │ in      │      ↓
            │       │ Clerk?  │      │
            │       └─────────┘      │
            │                        │
    ┌───────┴────────┐      ┌───────┴────────────────┐
    │ Authenticate   │      │ Authenticate with      │
    │ with Clerk     │      │ Cognito (old system)   │
    │ (new system)   │      │                        │
    └───────┬────────┘      └───────┬────────────────┘
            │                        │
            │                        ↓
            │               ┌────────────────────────┐
            │               │ Cognito auth succeeds  │
            │               └────────┬───────────────┘
            │                        │
            │                        ↓
            │               ┌─────────────────────────────────┐
            │               │ Backend: Create user in Clerk   │
            │               │ using Cognito password hasher   │
            │               │ await clerkClient.users.create({│
            │               │   emailAddress: user.email,     │
            │               │   password_hasher: 'awscognito',│
            │               │   password_digest: '...',       │
            │               │   externalId: cognito_sub       │
            │               │ })                              │
            │               └────────┬────────────────────────┘
            │                        │
            │                        ↓
            │               ┌─────────────────────────────────┐
            │               │ Return Cognito token to user    │
            │               │ (User is logged in!)            │
            │               └────────┬────────────────────────┘
            │                        │
            │                        ↓
            │               ┌─────────────────────────────────┐
            │               │ NEXT LOGIN: User now exists in  │
            │               │ Clerk, so they use Clerk path   │
            │               └─────────────────────────────────┘
            │                        │
            └────────────────────────┘
                         ↓
            ┌────────────────────────┐
            │ User gets Clerk token  │
            │ and is fully migrated  │
            └────────────────────────┘
```

**Implementation Example:**

**Frontend Login Component:**
```typescript
// app/_components/login-form.tsx
"use client";

import { useState } from 'react';

export function LoginForm() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  async function handleLogin(e: FormEvent) {
    e.preventDefault();

    // Step 1: Check if user exists in Clerk
    const checkResponse = await fetch('/api/auth/check-user', {
      method: 'POST',
      body: JSON.stringify({ email }),
    });

    const { existsInClerk } = await checkResponse.json();

    if (existsInClerk) {
      // Step 2a: User already migrated - use Clerk
      const response = await fetch('/api/auth/clerk-login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      });
      const { token } = await response.json();
      // Store token, redirect to dashboard

    } else {
      // Step 2b: User NOT migrated yet - use Cognito
      const response = await fetch('/api/auth/cognito-login', {
        method: 'POST',
        body: JSON.stringify({ email, password }),
      });
      const { token } = await response.json();
      // Store token, redirect to dashboard
      // Behind the scenes, user was just created in Clerk!
    }
  }

  return <form onSubmit={handleLogin}>...</form>;
}
```

**Backend Check User Route:**
```typescript
// app/api/auth/check-user/route.ts
import { clerkClient } from '@clerk/nextjs/server';

export async function POST(request: Request) {
  const { email } = await request.json();

  // Check if user exists in Clerk
  const users = await clerkClient.users.getUserList({
    emailAddress: [email],
  });

  return Response.json({
    existsInClerk: users.data.length > 0
  });
}
```

**Backend Cognito Login Route (with Clerk creation):**
```typescript
// app/api/auth/cognito-login/route.ts
import { CognitoIdentityProviderClient, InitiateAuthCommand } from '@aws-sdk/client-cognito-identity-provider';
import { clerkClient } from '@clerk/nextjs/server';

export async function POST(request: Request) {
  const { email, password } = await request.json();

  // Step 1: Authenticate with Cognito
  const cognitoClient = new CognitoIdentityProviderClient({});
  const authResult = await cognitoClient.send(new InitiateAuthCommand({
    AuthFlow: 'USER_PASSWORD_AUTH',
    ClientId: COGNITO_CLIENT_ID,
    AuthParameters: {
      USERNAME: email,
      PASSWORD: password,
    },
  }));

  // Step 2: Auth succeeded! Now create user in Clerk in the background
  const cognitoUserId = authResult.AuthenticationResult.IdToken; // decode to get sub

  try {
    await clerkClient.users.createUser({
      emailAddress: email,
      passwordHasher: 'awscognito',
      passwordDigest: `awscognito#${COGNITO_USER_POOL_ID}#${COGNITO_CLIENT_ID}#${email}`,
      externalId: cognitoUserId,
    });
  } catch (error) {
    // User might already exist (race condition), ignore
  }

  // Step 3: Return Cognito token to user (they're logged in!)
  return Response.json({
    token: authResult.AuthenticationResult.AccessToken,
  });
}
```

**Backend Token Validation (Dual Support):**
```typescript
// middleware.ts or auth validation utility
async function validateAuthToken(token: string) {
  try {
    // Parse the JWT to inspect claims without verification
    const decoded = parseJwtWithoutVerify(token);

    // Check the issuer to determine token source
    if (decoded.iss?.includes('cognito')) {
      // Validate using Cognito's verification
      return await validateCognitoToken(token);
    } else if (decoded.iss?.includes('clerk')) {
      // Validate using Clerk's verification
      return await validateClerkToken(token);
    } else {
      throw new Error('Unknown token issuer');
    }
  } catch (error) {
    // Token is invalid or malformed
    return null;
  }
}

async function validateCognitoToken(token: string) {
  // Use AWS Cognito JWT verification
  const verifier = CognitoJwtVerifier.create({
    userPoolId: COGNITO_USER_POOL_ID,
    clientId: COGNITO_CLIENT_ID,
    tokenUse: 'access'
  });

  const payload = await verifier.verify(token);
  return {
    userId: payload.sub,
    source: 'cognito'
  };
}

async function validateClerkToken(token: string) {
  // Use Clerk's verification
  const verified = await clerkClient.verifyToken(token, {
    secretKey: CLERK_SECRET_KEY
  });

  return {
    userId: verified.sub,
    source: 'clerk',
    externalId: verified.external_id // Original Cognito ID if migrated
  };
}
```

**Key Clarifications:**

1. **First Login During Migration**: When a user who exists in Cognito (but not yet in Clerk) signs in:
   - They authenticate with Cognito (because they don't exist in Clerk yet)
   - Backend receives Cognito authentication success
   - Backend **immediately creates the user in Clerk** using the special password hasher
   - User receives a Cognito token and is logged in
   - User experience: seamless, no disruption

2. **Subsequent Logins**: When the same user signs in again:
   - Frontend checks if user exists in Clerk → **YES** (created during first login)
   - User authenticates directly with Clerk
   - User receives a Clerk token
   - User is now fully migrated

3. **Backend Token Validation**: During the migration period, the backend must support BOTH token types:
   - Users who logged in once during migration → Clerk tokens
   - Users who haven't logged in yet during migration → Cognito tokens (if they somehow get a session)
   - Distinguish tokens by inspecting the `iss` (issuer) claim in the JWT

4. **User ID Correlation**: Use `external_id` in Clerk to store the original Cognito user ID, allowing correlation across systems during the transition.

5. **Migration Progression**:
   - Week 1: 10% of active users migrated (logged in during week 1)
   - Week 2: 30% of active users migrated (logged in during weeks 1-2)
   - Week 8: 95% of active users migrated
   - Week 12: Bulk import remaining 5% inactive users, decommission Cognito

**Alternative: Simpler Bulk Import Approach**

If the trickle approach seems too complex, you can also:

1. **Bulk import all users** from Cognito to Clerk using the migration script (one-time operation)
2. **Switch frontend** to Clerk authentication immediately
3. Users' passwords work immediately due to Clerk's password hasher support
4. Higher risk but simpler implementation (no dual authentication logic)

The trickle approach provides a safety net and gradual validation, while bulk import is faster to implement but requires more confidence in the migration process.

### Alternative: Big Bang Migration

A big bang migration involves importing all users at once and immediately switching to Clerk.

#### Advantages
✅ Clean cutover date
✅ No parallel systems to maintain
✅ Simpler long-term architecture

#### Disadvantages
❌ Higher risk of widespread issues
❌ Requires extensive pre-migration testing
❌ Pressure on a single coordinated event
❌ Harder to roll back if problems arise
❌ May require user communication and training

#### When to Consider Big Bang
- Small user base (< 1,000 users)
- Highly engaged users who sign in frequently
- Strong confidence in migration process
- Ability to schedule maintenance window

---

## 3. Multi-Account Cognito Consolidation

### Challenge

Migrating from 3 separate Cognito user pools across different AWS accounts to a single Clerk instance presents unique challenges:

**Cognito Limitations**:
- No native way to export password hashes between user pools
- User `sub` (unique identifier) doesn't carry over between pools
- Many settings are immutable after pool creation

**Identity Conflicts**:
- Same email address may exist in multiple Cognito pools
- Need strategy to handle duplicate identities
- Must preserve user associations with their respective accounts/organizations

### Recommended Strategy

#### Step 1: Identify and Map Users to Organizations

Before migration, create a mapping of users to their intended Clerk organizations:

```
AWS Account 1 Users → Organization A
AWS Account 2 Users → Organization B (e.g., UC Davis)
AWS Account 3 Users → Organization C
```

For enterprise SSO users (like UC Davis), map them to organization-specific SSO connections.

#### Step 2: Sequential Pool Migration

Migrate one Cognito pool at a time to reduce complexity:

**Pool 1 Migration** (e.g., AWS Account 1)
1. Run Clerk migration script for Pool 1
2. Map users to Organization A in Clerk
3. Test authentication for Pool 1 users
4. Monitor for issues before proceeding

**Pool 2 Migration** (e.g., AWS Account 2 - UC Davis)
1. Set up UC Davis enterprise SSO in Clerk
2. Migrate password-based users from Pool 2
3. Map users to UC Davis organization
4. Test both password and SSO authentication
5. Monitor before proceeding

**Pool 3 Migration** (e.g., AWS Account 3)
1. Repeat process for remaining pool
2. Map to Organization C

#### Step 3: Handle Identity Conflicts

**Duplicate Email Addresses**:

If the same email exists in multiple pools:
- Clerk requires unique email addresses
- Options:
  1. Contact users to use different emails
  2. Use organization-scoped usernames instead of email
  3. Merge duplicate identities (if they represent the same person)

**Recommended Approach**: During migration, detect duplicates and flag for manual review. Use `external_id` to track original Cognito user IDs from each pool.

#### Step 4: External ID Tracking

Store original Cognito identifiers in Clerk's `external_id` field:

```
Format: cognito:{pool_id}:{user_sub}
Example: cognito:us-east-1_ABC123:a1b2c3d4-5678-90ab-cdef-EXAMPLE11111
```

This allows:
- Correlating users across systems during parallel operation
- Troubleshooting migration issues
- Maintaining audit trail
- Supporting rollback if needed

### Rate Limiting Considerations

**Clerk Rate Limits**:
- Development instances: 100 requests per 10 seconds
- Production instances: 1,000 requests per 10 seconds

**For 3 User Pools**: Batch import must respect these limits. The migration script should include rate limiting logic to avoid hitting API limits.

---

## 4. Enterprise SSO and Organization Mapping

### Organization-Level Enterprise SSO

Clerk supports **organization-level enterprise SSO**, which is perfect for the UC Davis use case.

#### How It Works

**Configuration**:
1. Create an organization in Clerk (e.g., "UC Davis")
2. Configure enterprise SSO connection for that organization
3. Set the email domain (e.g., `@ucdavis.edu`)
4. Configure SAML 2.0 or OIDC connection to UC Davis IdP

**Automatic User Provisioning**:
- When users sign in via the organization's SSO, they are **automatically added as members** of that organization
- Users are assigned a default role (member or admin)
- No manual user-to-organization mapping required

**Just-in-Time (JIT) Provisioning**:
- Clerk supports JIT provisioning during SAML SSO
- User attributes from IdP are automatically mapped to Clerk user properties
- Customizable attribute mapping in Clerk Dashboard

#### Migrating Existing Enterprise SSO Users

**Current Setup**: UC Davis users authenticate via Cognito enterprise connection

**Migration Steps**:

1. **Configure UC Davis SSO in Clerk**:
   - Set up SAML connection with UC Davis IdP
   - Map to "UC Davis" organization
   - Configure attribute mapping (name, email, groups)
   - Enable subdomain support if needed

2. **Migrate Password-Based Users (if any)**:
   - Use Clerk migration script for UC Davis users who have passwords
   - Associate them with UC Davis organization during import
   - These users can still use passwords or switch to SSO

3. **Test Enterprise Connection**:
   - Test authentication flow with UC Davis IdP
   - Verify automatic organization membership
   - Confirm attribute mapping works correctly

4. **Cutover Strategy**:
   - Update application to use Clerk SSO endpoint
   - Users continue authenticating via UC Davis IdP seamlessly
   - No user disruption (same IdP, different broker)

#### Supported Protocols

**SAML 2.0**:
- Microsoft Azure AD
- Google Workspace
- Okta Workforce
- Custom SAML providers

**OIDC**:
- EASIE: Multi-tenant OpenID (Google Workspace, Microsoft Entra ID)
- Custom OIDC providers

**Recommendation**: If UC Davis supports it, EASIE SSO provides a simpler alternative to SAML with automatic deprovisioning support.

#### Attribute Mapping

Clerk provides flexible attribute mapping in the Dashboard:

**Standard Attributes**:
- Email
- First name
- Last name
- Username

**Custom Attributes**:
- Department
- Employee ID
- Group memberships
- Any other SAML attributes from IdP

**Mapping Interface**: Clerks Dashboard shows what User object properties map to IdP claims, with easy customization if they don't match.

### Organization Architecture

**User-to-Organization Relationship**:
- A user can belong to multiple organizations (no limit)
- A user has one "active organization" at a time (workspace context)
- Users switch between organizations via `<OrganizationSwitcher />` component

**Membership Limits**:
- Free plan: 5 members per organization
- Pro plan: Unlimited members per organization

**Management**:
- Organizations can be created via Dashboard or programmatically
- Users can create up to 100 organizations per application
- Organization admins can manage members and settings

---

## 5. Machine-to-Machine (M2M) Authentication

### Clerk M2M Tokens

Clerk launched M2M tokens (now in General Availability as of October 2025) for service-to-service authentication.

#### Key Features

**Purpose**: Authenticate requests between backend services within your infrastructure:
- Microservices
- Background workers
- Distributed systems
- Internal APIs

**Configuration**:
1. Create "machines" in Clerk Dashboard
2. Define which machines can communicate with each other (scopes)
3. Generate M2M tokens for each machine
4. Use tokens as Bearer tokens in Authorization headers

**Token Lifecycle**:
- **Generation**: `clerkClient.m2m.createToken()`
- **Validation**: `verifyToken()` on receiving service
- **Revocation**: `revokeToken()` to invalidate immediately

**Optional Parameters**:
- `secondsUntilExpiration`: Control token lifespan (default: no expiration)
- `claims`: Custom metadata stored in token

#### Implementation Pattern

**Service A (Caller)**:
```javascript
const m2mToken = await clerkClient.m2m.createToken({
  secondsUntilExpiration: 3600, // 1 hour
  claims: { serviceId: 'service-a' }
});

// Make request to Service B
await fetch('https://service-b/api/endpoint', {
  headers: {
    'Authorization': `Bearer ${m2mToken.secret}`
  }
});
```

**Service B (Receiver)**:
```javascript
const token = request.headers.authorization.replace('Bearer ', '');
const verified = await clerkClient.m2m.verifyToken(token);

if (verified) {
  // Process request
} else {
  // Reject unauthorized request
}
```

#### Migration from Cognito Client Credentials

**Cognito Setup**: Currently using Cognito client credentials flow for M2M authentication

**Migration Path**:

1. **Parallel Operation**:
   - Keep Cognito client credentials active
   - Create machines in Clerk for each service
   - Update services to support both Cognito and Clerk tokens
   - Gradually transition services to use Clerk M2M tokens

2. **Token Validation**:
   ```javascript
   // Pseudo-code for parallel validation
   async function validateToken(token) {
     if (token.startsWith('cognito_')) {
       return validateCognitoToken(token);
     } else {
       return clerkClient.m2m.verifyToken(token);
     }
   }
   ```

3. **Service-by-Service Migration**:
   - Migrate caller services one at a time
   - Update to generate Clerk M2M tokens
   - Receiver services support both token types during transition
   - Decommission Cognito clients after all services migrated

#### Pricing

**Current Model** (as of October 2025):
- Token Creation: $0.001 per token
- Token Verification: $0.0001 per request

**Future Enhancement**: JWT-based M2M tokens (in development)
- Local verification (no network calls)
- Only charged for token creation
- Faster and more flexible

#### Scope Management

**Important**: When modifying machine scopes, changes only apply to newly created tokens. Existing tokens retain their original scopes.

**Best Practice**: Use short-lived tokens (1-24 hours) to ensure scope changes take effect quickly.

---

## 6. Implementation Roadmap

### Phase 1: Preparation (Week 1-2)

**Setup Clerk Instance**:
- [ ] Create Clerk production instance
- [ ] Configure production settings
- [ ] Set up organizations structure
- [ ] Configure rate limiting and monitoring

**Configure Enterprise SSO**:
- [ ] Set up UC Davis SAML connection
- [ ] Configure other enterprise connections
- [ ] Test attribute mapping
- [ ] Verify automatic organization membership

**Prepare Migration Scripts**:
- [ ] Customize Clerk's Cognito migration script
- [ ] Add organization mapping logic
- [ ] Implement rate limiting
- [ ] Add conflict detection for duplicate emails
- [ ] Test with sample data in development

**Verify Cognito Configuration**:
- [ ] Confirm all user pools have public clients with `ALLOW_USER_PASSWORD_AUTH`
- [ ] Document current app client configurations
- [ ] Export user lists from each pool for planning

### Phase 2: Development Environment Migration (Week 2-3)

**Migrate Dev Environment**:
- [ ] Run migration script for one Cognito pool (dev)
- [ ] Validate users in Clerk
- [ ] Test password authentication
- [ ] Test enterprise SSO flows
- [ ] Verify organization membership

**Test Application Changes**:
- [ ] Update frontend to support Clerk SDK
- [ ] Implement authentication routing logic
- [ ] Test session management
- [ ] Verify user profile access
- [ ] Test organization switching

**Validate M2M Setup**:
- [ ] Create test machines in Clerk
- [ ] Generate M2M tokens
- [ ] Test service-to-service authentication
- [ ] Verify token validation
- [ ] Test parallel Cognito/Clerk M2M support

### Phase 3: Production Pilot (Week 4-5)

**Migrate First User Pool**:
- [ ] Choose lowest-risk pool (smallest, least critical)
- [ ] Run migration script for production
- [ ] Map users to appropriate organizations
- [ ] Monitor for errors

**Pilot with Small User Group**:
- [ ] Select 10-20 test users
- [ ] Communicate changes to pilot group
- [ ] Monitor their authentication flows
- [ ] Collect feedback
- [ ] Fix any issues before broader rollout

**Parallel System Operation**:
- [ ] Deploy updated application with dual authentication support
- [ ] Route existing users to Cognito
- [ ] Route new users to Clerk
- [ ] Monitor both systems
- [ ] Track migration progress

### Phase 4: Gradual User Migration (Week 6-10)

**Trickle Migration**:
- [ ] Deploy authentication routing logic
- [ ] Users authenticate through Cognito initially
- [ ] On successful auth, create user in Clerk
- [ ] Subsequent logins use Clerk
- [ ] Monitor migration rate

**Migrate Remaining Pools**:
- [ ] Repeat pilot process for Pool 2
- [ ] Test UC Davis enterprise SSO thoroughly
- [ ] Migrate Pool 3
- [ ] Address any pool-specific issues

**M2M Token Transition**:
- [ ] Update service A to generate Clerk tokens
- [ ] Update service B to validate both token types
- [ ] Monitor M2M authentication success rates
- [ ] Gradually migrate remaining services

**Monitor and Adjust**:
- [ ] Track daily active users in each system
- [ ] Monitor authentication failure rates
- [ ] Address user-reported issues promptly
- [ ] Communicate progress to stakeholders

### Phase 5: Bulk Import and Cutover (Week 11-12)

**Import Inactive Users**:
- [ ] Identify users who haven't migrated naturally
- [ ] Run bulk import for remaining users
- [ ] Send email notifications about migration
- [ ] Provide support documentation

**Final Validation**:
- [ ] Verify all critical users migrated successfully
- [ ] Test all authentication flows end-to-end
- [ ] Confirm enterprise SSO connections working
- [ ] Validate M2M token exchange for all services

**Decommission Cognito**:
- [ ] Switch application to Clerk-only mode
- [ ] Remove Cognito SDK and authentication code
- [ ] Archive Cognito user pools (don't delete immediately)
- [ ] Monitor for any rollback needs (1-2 week safety window)

**Post-Migration Cleanup**:
- [ ] Delete archived Cognito resources after safety period
- [ ] Remove parallel authentication logic
- [ ] Clean up `external_id` references if no longer needed
- [ ] Update documentation

---

## 7. Risk Mitigation

### Potential Risks and Mitigation Strategies

#### Risk 1: Password Migration Failure
**Impact**: Users unable to sign in with existing passwords

**Mitigation**:
- Test with single user first before bulk import
- Verify `ALLOW_USER_PASSWORD_AUTH` is enabled in Cognito
- Implement password reset fallback
- Monitor authentication failure rates
- Provide clear user support documentation

#### Risk 2: Duplicate Email Addresses
**Impact**: Migration script fails or users locked out

**Mitigation**:
- Pre-scan all pools for duplicate emails before migration
- Flag duplicates for manual review
- Contact affected users to resolve before migration
- Consider using username instead of email for some users
- Implement conflict resolution strategy

#### Risk 3: Enterprise SSO Configuration Issues
**Impact**: All users from enterprise connection unable to sign in

**Mitigation**:
- Test enterprise SSO thoroughly in development
- Migrate enterprise connections during low-traffic periods
- Keep Cognito SSO active during parallel operation
- Have rollback plan ready
- Communicate with IdP administrators (e.g., UC Davis IT)

#### Risk 4: M2M Token Transition Issues
**Impact**: Service-to-service communication failures

**Mitigation**:
- Support both token types during transition
- Implement robust error handling and retries
- Monitor M2M authentication metrics closely
- Test all service integrations thoroughly
- Have fallback to Cognito tokens ready

#### Risk 5: Session Management Conflicts
**Impact**: Users with invalid or conflicting sessions

**Mitigation**:
- Implement clear session handling logic
- Use `external_id` to correlate sessions
- Handle token validation errors gracefully
- Log session-related issues for debugging
- Force re-authentication if necessary

#### Risk 6: Data Loss or Corruption
**Impact**: User data lost or incorrectly migrated

**Mitigation**:
- Create complete backups of Cognito data before migration
- Validate migrated data matches source data
- Implement verification scripts to check data integrity
- Keep Cognito pools archived for 30+ days post-migration
- Have rollback process documented and tested

### Rollback Strategy

**Triggers for Rollback**:
- Authentication failure rate > 5%
- Critical enterprise SSO connection fails
- Data corruption detected
- Widespread user complaints
- Service disruption > 1 hour

**Rollback Process**:
1. Switch application back to Cognito authentication
2. Disable Clerk authentication endpoints
3. Restore any lost data from backups
4. Investigate and fix root cause
5. Plan new migration attempt with fixes

**Rollback Window**: Maintain ability to rollback for 1-2 weeks post-cutover

---

## 8. Testing Strategy

### Unit Testing

**Authentication Module**:
- Test Cognito token validation
- Test Clerk token validation
- Test authentication routing logic
- Test `external_id` mapping
- Test session management

**M2M Token Handling**:
- Test token generation
- Test token verification
- Test token expiration handling
- Test parallel token support (Cognito + Clerk)

### Integration Testing

**Authentication Flows**:
- Password-based sign-in (Cognito users migrated to Clerk)
- Enterprise SSO sign-in (UC Davis)
- User registration (new users to Clerk)
- Password reset
- MFA flows

**Organization Membership**:
- Automatic membership via SSO
- Manual user-to-organization assignment
- Organization switching
- Permission enforcement

**M2M Communication**:
- Service A → Service B with Cognito tokens
- Service A → Service B with Clerk tokens
- Token validation in receiving services
- Error handling for invalid tokens

### End-to-End Testing

**User Journeys**:
1. Existing user (Cognito) → Sign in via Clerk → Access resources
2. Enterprise user (UC Davis SSO) → Sign in → Auto-join organization → Access resources
3. New user → Register in Clerk → Join organization → Access resources
4. User with multiple organizations → Switch between organizations

**Service Communication**:
1. Service A generates Clerk M2M token → Calls Service B → Service B validates token → Success
2. Service A generates Cognito token (during parallel operation) → Calls Service B → Service B validates → Success

### Performance Testing

**Load Testing**:
- Simulate concurrent sign-ins (100+ users)
- Test authentication endpoint response times
- Verify rate limiting handles high load
- Test organization switching under load

**M2M Token Load**:
- Simulate high-volume token generation
- Test token verification performance
- Verify Clerk pricing model impact

### Security Testing

**Authentication Security**:
- Test password complexity enforcement
- Verify MFA configuration
- Test session timeout handling
- Validate JWT signature verification

**M2M Security**:
- Test token revocation
- Verify scope enforcement
- Test expired token handling
- Validate machine-to-machine permissions

---

## 9. Cost Analysis

### Clerk Pricing Model

**Monthly Active Users (MAU)**:
- Free: Up to 10,000 MAUs
- Pro: $25/month base + $0.02 per MAU above 10,000
- Enterprise: Custom pricing

**M2M Tokens** (paid feature):
- Token Creation: $0.001 per token
- Token Verification: $0.0001 per request

**Enterprise SSO**:
- Free: Up to 25 connections in development
- Production: Requires Pro plan + Enhanced Authentication Add-on

**Organizations**:
- Free: 5 members per organization
- Pro: Unlimited members per organization

### Cost Comparison: Cognito vs. Clerk

**Cognito Costs** (3 user pools):
- Monthly Active Users: $0.00550 per MAU (first 50,000)
- Advanced Security Features: Additional cost
- Enterprise SSO: Included in pricing

**Clerk Costs** (1 instance):
- Monthly Active Users: $0.02 per MAU above 10,000
- M2M Tokens: $0.001 per creation + $0.0001 per verification
- Enterprise SSO: Included with Pro plan + add-on

**Example Scenario** (10,000 active users):
- Cognito: ~$55/month for MAUs + potential advanced security costs
- Clerk: $25/month (within free tier for MAUs) + M2M costs + SSO add-on

### Transition Period Costs

**Parallel Operation**:
- Pay for both Cognito and Clerk during migration
- Clerk only charges for active users in Clerk (not total users)
- Gradual cost shift from Cognito to Clerk as users migrate

**Estimated Timeline**: 2-3 months of parallel operation

**Budget for**:
- Cognito costs for users not yet migrated
- Clerk costs for users actively using Clerk
- M2M token costs for parallel token support
- Potential engineering time for migration implementation

---

## 10. Key Decisions and Recommendations

### Primary Recommendations

1. **Migration Approach**: ✅ **Gradual Trickle Migration**
   - Lower risk than big bang
   - Allows real-world validation during rollout
   - Clerk explicitly supports this approach
   - Better user experience (no forced password resets)

2. **User Pool Consolidation**: ✅ **Sequential Pool Migration**
   - Migrate one Cognito pool at a time
   - Reduce complexity and risk
   - Easier to troubleshoot issues
   - Map each pool to appropriate Clerk organizations

3. **Enterprise SSO**: ✅ **Organization-Level SSO in Clerk**
   - Perfect fit for UC Davis use case
   - Automatic user provisioning to organizations
   - Maintains seamless SSO experience
   - Easy attribute mapping

4. **M2M Authentication**: ✅ **Parallel Token Support → Gradual Migration**
   - Support both Cognito and Clerk tokens during transition
   - Migrate services one at a time
   - Monitor M2M authentication closely
   - Consider short-lived tokens for faster scope updates

5. **Password Migration**: ✅ **Clerk's Native Cognito Password Migration**
   - No user password resets required
   - Seamless authentication experience
   - Automatic re-hashing for improved security
   - Well-documented and supported by Clerk

### Decision Points Requiring Further Analysis

1. **Duplicate Email Handling**: Requires pre-migration scan and resolution strategy
2. **Timeline**: Balance between speed and risk tolerance (recommended: 2-3 months)
3. **Pilot Group Selection**: Choose representative users for initial testing
4. **Rollback Criteria**: Define specific thresholds for triggering rollback
5. **M2M Token Lifespan**: Determine optimal token expiration for security vs. performance

### Success Criteria

**Technical Success**:
- [ ] 100% of users migrated to Clerk
- [ ] Authentication failure rate < 1%
- [ ] All enterprise SSO connections working
- [ ] All M2M token exchanges successful
- [ ] Zero data loss or corruption

**Business Success**:
- [ ] No user disruption or complaints
- [ ] Reduced authentication infrastructure complexity (1 system vs. 4)
- [ ] Improved security posture
- [ ] Cost-neutral or cost-savings achieved
- [ ] Foundation for future features (e.g., advanced MFA, passwordless)

**User Experience Success**:
- [ ] Existing passwords continue to work
- [ ] Enterprise SSO remains seamless
- [ ] No forced password resets
- [ ] Clear communication throughout migration
- [ ] Support documentation available

---

## 11. Additional Resources

### Clerk Documentation
- [Cognito Migration Guide](https://clerk.com/docs/guides/development/migrating/cognito)
- [Organization-Level Enterprise SSO](https://clerk.com/docs/organizations/manage-sso)
- [M2M Tokens Documentation](https://clerk.com/docs/machine-auth/m2m-tokens)
- [Trickle Migration Strategy](https://clerk.com/docs/deployments/migrate-overview)
- [JIT Provisioning](https://clerk.com/docs/authentication/enterprise-connections/jit-provisioning)

### AWS Cognito Resources
- [Cognito User Pool Migration](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-import-users.html)
- [Cross-Account Migration Strategies](https://aws.amazon.com/blogs/security/approaches-for-migrating-users-to-amazon-cognito-user-pools/)

### Migration Tools
- [Clerk Migration Script (GitHub)](https://github.com/clerk/migration-script)
- Clerk Backend API Documentation

### Community Resources
- [Novu Case Study: Migrating to Clerk](https://novu.co/blog/migrating-user-management-to-clerk-with-one-developer/)
- [Turso: Why We Chose Clerk](https://turso.tech/blog/why-we-transitioned-to-clerk-for-authentication)

---

## Next Steps

1. **Review this research with stakeholders** and get alignment on migration approach
2. **Estimate engineering resources** needed for 2-3 month migration timeline
3. **Set up Clerk development instance** and test password migration with sample data
4. **Configure UC Davis enterprise SSO** in Clerk development instance and test
5. **Develop detailed project plan** with specific dates and milestones
6. **Prepare communication plan** for users (especially enterprise SSO users)
7. **Create rollback procedures** and document thoroughly
8. **Begin Phase 1 preparation work** per implementation roadmap

---

## Conclusion

Migrating from a multi-account Cognito setup to a single Clerk instance is achievable with careful planning and a gradual approach. The key findings are:

- **Password migration is seamless** using Clerk's native Cognito password support
- **Trickle migration minimizes risk** by allowing parallel operation and gradual user adoption
- **Organization-level SSO** perfectly addresses the UC Davis (and similar) use case
- **M2M token migration** is straightforward with parallel token support
- **Timeline of 2-3 months** balances thoroughness with reasonable time-to-completion

The recommended gradual approach, sequential pool migration, and organization-based architecture provide a low-risk path to consolidating authentication into a single, modern platform while maintaining a seamless user experience.

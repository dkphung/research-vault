# SimpleSAMLphp + Clerk Integration for InCommon Federation - Research

**Date**: 2025-10-21
**Status**: Research Complete

## Executive Summary

Integrating SimpleSAMLphp with Clerk for InCommon federation support presents a **feasible but architecturally complex challenge**. The recommended approach is to use **SimpleSAMLphp as a SAML Service Provider (SP)** that handles SAML authentication with InCommon IdPs, then bridges to Clerk by creating Clerk sessions via the Backend API. SimpleSAMLphp handles all SAML protocol complexity, while Clerk handles session management and user lifecycle.

**Critical Architecture Clarification**:
- SimpleSAMLphp acts as **BOTH** an Identity Provider (to Clerk) and Service Provider (to InCommon)
- This creates a "SAML proxy" that bridges Clerk's Enterprise SAML to InCommon's multilateral federation
- The flow is: `App → Clerk Enterprise SAML → SimpleSAMLphp IdP → SimpleSAMLphp SP → InCommon Federation → University IdP`
- SimpleSAMLphp handles all SAML complexity (metadata, discovery, validation)
- Clerk handles all user/session management using its native Enterprise SAML feature
- Users interact with your app using standard Clerk components (`<SignIn />`)

**Key Recommendation**:
- **Option 1 (SAML Proxy - RECOMMENDED)**: Use SimpleSAMLphp as a SAML proxy IdP that Clerk connects to via Enterprise SAML. This provides seamless Clerk integration with InCommon multilateral federation.
- **Option 2 (Clerk Native)**: Use Clerk's native Enterprise SAML if organizations can configure direct SAML connections (bilateral, not multilateral).
- **Option 3 (Custom Integration)**: Build custom Next.js API routes if you need full control but don't want the SAML proxy approach.

## Context

This research was conducted to determine how to enable InCommon federation authentication for a Next.js application that currently uses Clerk for authentication and user management. The goal is to allow university users to authenticate via their institutional InCommon Identity Providers (IdPs) while maintaining Clerk as the primary session and user management system.

InCommon is a multilateral SAML federation serving higher education institutions in the United States. It allows Service Providers (SPs) to trust hundreds of university IdPs through shared metadata without individual integration agreements.

## Authentication Flow Architecture

### The Complete Flow (SimpleSAMLphp + Clerk)

```
┌─────────────────────────────────────────────────────────────────┐
│                        User's Browser                            │
└───────┬─────────────────────────────────────────────────────┬───┘
        │                                                     │
        │ 1. Click "Login with University"                   │
        ▼                                                     │
┌────────────────────────────────┐                           │
│   Next.js App (Server)         │                           │
│   /login route                 │                           │
└───────┬────────────────────────┘                           │
        │ 2. Redirect to SAML login                          │
        ▼                                                     │
┌────────────────────────────────┐                           │
│   SimpleSAMLphp (SP)           │                           │
│   - Initiates SAML AuthnRequest│                           │
│   - Redirects to IdP           │                           │
└───────┬────────────────────────┘                           │
        │ 3. SAML AuthnRequest                               │
        ▼                                                     │
┌────────────────────────────────┐                           │
│   InCommon Federation          │                           │
│   (Metadata Registry)          │                           │
└───────┬────────────────────────┘                           │
        │ 4. Routes to correct IdP                           │
        ▼                                                     │
┌────────────────────────────────┐                           │
│   University IdP               │                           │
│   (e.g., UC Davis Shibboleth)  │                           │
│   - User enters credentials    │                           │
│   - Authenticates user         │                           │
│   - Generates SAML assertion   │                           │
└───────┬────────────────────────┘                           │
        │ 5. SAML Response with assertion                    │
        ▼                                                     │
┌────────────────────────────────┐                           │
│   SimpleSAMLphp (SP)           │                           │
│   - Validates SAML assertion   │                           │
│   - Extracts user attributes   │                           │
│   - POST to ACS callback       │                           │
└───────┬────────────────────────┘                           │
        │ 6. POST to /api/auth/saml/acs                      │
        ▼                                                     │
┌────────────────────────────────┐                           │
│   Next.js API Route            │                           │
│   /api/auth/saml/acs           │                           │
│   - Parse SAML attributes      │                           │
│   - Map to Clerk user schema   │                           │
└───────┬────────────────────────┘                           │
        │ 7. Create/update user via Clerk Backend API        │
        ▼                                                     │
┌────────────────────────────────┐                           │
│   Clerk Backend API            │                           │
│   - Create or update user      │                           │
│   - Generate session token     │                           │
└───────┬────────────────────────┘                           │
        │ 8. Return session token                            │
        ▼                                                     │
┌────────────────────────────────┐                           │
│   Next.js API Route            │                           │
│   - Set __session cookie       │                           │
│   - Redirect to app            │────────────────────────────┘
└───────┬────────────────────────┘          9. Redirect with session cookie
        │
        ▼
┌────────────────────────────────┐
│   Next.js App                  │
│   User is authenticated        │
│   with Clerk session           │
└────────────────────────────────┘
```

### Key Points

1. **SimpleSAMLphp is ONLY a Service Provider (SP)**: It does NOT act as an IdP. It communicates directly with university IdPs via the SAML protocol.

2. **No SimpleSAMLphp IdP Required**: The common misconception is that you need to run SimpleSAMLphp as both an SP and an IdP. This is incorrect. SimpleSAMLphp's SP functionality is sufficient.

3. **Clerk is NOT involved in SAML**: Clerk doesn't speak SAML in this flow. It only:
   - Stores user accounts
   - Manages sessions via JWT tokens
   - Provides user management APIs

4. **The Bridge Point is the ACS Callback**: The Next.js API route at `/api/auth/saml/acs` is where SAML world meets Clerk world. This is where you:
   - Receive validated SAML attributes from SimpleSAMLphp
   - Create/update Clerk user accounts
   - Establish Clerk sessions

5. **End Result**: Users authenticate via university SAML, but once authenticated, they have a standard Clerk session and the app works exactly like any other Clerk-authenticated user.

### Login User Experience

**Important**: Clerk does NOT automatically redirect to SimpleSAMLphp. Your application must provide a separate login path for university users:

**Two Login Paths**:

1. **Regular Users**: Use Clerk's `<SignIn />` component for email/password, OAuth, etc.
2. **University Users**: Custom "Login with University" button that:
   - Prompts for university email OR shows university picker
   - Redirects to your `/api/auth/saml/login` route
   - Which redirects to SimpleSAMLphp SP
   - SAML flow completes
   - Returns to `/api/auth/saml/acs` which creates Clerk session

**Example Login Page**:
```typescript
// Two separate login options on your sign-in page
<div>
  <SignIn /> {/* Clerk's standard login */}
  <div>OR</div>
  <Link href="/login/university">Login with University Account</Link>
</div>
```

**After Authentication**: Once a university user has a Clerk session, they interact with your app identically to regular users. Clerk hooks (`useUser()`, `useAuth()`, etc.) work the same. The only difference is stored in `user.publicMetadata.authMethod === 'saml'`.

## Research Goals

- [x] Understand SimpleSAMLphp architecture and InCommon federation patterns
- [x] Analyze Clerk's authentication extensibility and Enterprise SSO capabilities
- [x] Identify SAML-to-modern-auth integration patterns
- [x] Evaluate technical implementation approaches
- [x] Research session synchronization and user provisioning strategies
- [x] Investigate security considerations and best practices

## Technical Deep Dive

### SimpleSAMLphp Overview

SimpleSAMLphp is a native PHP SAML 2.0 implementation that can function as either an Identity Provider (IdP), Service Provider (SP), or authentication proxy. It's widely used in higher education for InCommon federation integration.

#### Core Architecture

**Authentication Sources**: SimpleSAMLphp uses a modular authentication system defined in `config/authsources.php`. Each authentication source is a class implementing different authentication methods (LDAP, SQL, SAML, etc.).

**Entity Configuration**: SimpleSAMLphp operates as an SP through configuration entries that specify:
- Entity ID (must be a stable URI)
- Single Sign-On Service endpoints
- Assertion Consumer Service (ACS) URLs
- Certificate-based signing and encryption

**Metadata Management**: Identity Providers are configured in `metadata/saml20-idp-remote.php` with SSO endpoints and certificates.

#### Authentication Flow (SP-Initiated)

1. Application code calls `$as->requireAuth()`
2. User is redirected to the configured IdP (or discovery service for multiple IdPs)
3. User authenticates at IdP with institutional credentials
4. IdP generates SAML assertion with user attributes
5. IdP redirects user back to SP's Assertion Consumer Service (ACS)
6. SimpleSAMLphp validates assertion, signature, and conditions
7. Attributes are extracted and made available via `$as->getAttributes()`
8. Session is established in SimpleSAMLphp

#### InCommon-Specific Features

**Metadata Consumption**: SimpleSAMLphp can consume InCommon metadata via:
- Traditional metarefresh module (for metadata aggregates)
- Modern MDQ (Metadata Query) service (recommended)

MDQ Configuration:
```php
'metadata.sources' => [
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => ['/path/to/inc-md-cert-mdq.pem'],
        'cachedir' => '/path/to/cache/mdq',
        'cachelength' => 86400
    ],
]
```

**Important**: SimpleSAMLphp versions earlier than 1.13.2 have performance issues with large metadata files.

**IdP Discovery**: With multilateral federation, SimpleSAMLphp typically integrates a discovery service to allow users to select their home institution. However, with MDQ there's no local metadata for discovery services to process, requiring separate metadata provision.

#### Attribute Processing

SAML assertions contain user attributes released by the IdP. InCommon IdPs commonly release:

- **eduPersonPrincipalName** (ePPN): Scoped identifier in format `username@institution.edu`
  - OID: `urn:oid:1.3.6.1.4.1.5923.1.1.1.6`
  - Non-reassignable, persistent identifier
  - Includes scope validation against IdP's registered domain

- **eduPersonTargetedID**: Anonymous persistent identifier

- **mail**: Email address

- **displayName**: User's full name

- **eduPersonScopedAffiliation**: User's relationship to institution (student, faculty, staff, etc.)

Attributes are accessed in PHP as:
```php
$attributes = $as->getAttributes();
// Returns: ['eduPersonPrincipalName' => ['jsmith@ucdavis.edu'], 'mail' => ['jsmith@ucdavis.edu']]
// Note: Every attribute value is an array
```

#### Session Management

SimpleSAMLphp maintains its own session layer separate from PHP's native sessions. This creates integration challenges:

**Session Conflicts**: PHP doesn't allow two sessions open simultaneously. Applications must:
1. Call SimpleSAMLphp authentication methods
2. Call `\SimpleSAML\Session::getSessionFromRequest()->cleanup()`
3. Resume application session

**Session Storage**: SimpleSAMLphp supports multiple session stores:
- `phpsession`: Built-in PHP sessions (not suitable for load balancing)
- `memcache`: Distributed caching with replication
- `redis`: Distributed session storage
- `sql`: Database-backed sessions

**Single Logout (SLO)**: SimpleSAMLphp supports SAML Single Logout, which propagates logout to all connected SPs. Implementation requires:
- Calling `$auth->logout()` terminates all sessions (local, SP, IdP, and other SPs)
- Registering logout handlers for application-specific cleanup
- Handling partial logout scenarios

### Clerk Authentication Architecture

Clerk is a modern authentication and user management platform designed for React/Next.js applications. It provides prebuilt UI components, session management, and user provisioning.

#### Core Authentication Flow

Clerk manages authentication through two primary objects:
- **SignUp**: Progressive user registration with field validation
- **SignIn**: Multi-factor authentication support

The flow involves:
1. User initiates sign-in through Clerk components
2. First factor verification (password, email link, OTP, OAuth, or Enterprise SSO)
3. Optional second factor verification (MFA)
4. Session creation and token generation
5. Optional session tasks (e.g., organization selection)

#### Session Token Architecture

**JWT Session Tokens**: Clerk generates short-lived JWTs containing:
- Standard claims: `azp`, `exp`, `iat`, `iss`, `jti`, `nbf`, `sub`
- User identification: user ID, session ID
- Custom claims via JWT templates (up to 1.2KB after default claims)

**Cookie Storage**: Session tokens are stored in `__session` cookie (4KB browser limit)

**Validation**: Tokens are signed with instance private key, verified with public key. Middleware automatically validates, or manual verification is available.

**Custom Claims**: You can add custom claims using shortcodes:
```json
{
  "org_id": "{{org.id}}",
  "org_slug": "{{org.slug}}",
  "metadata": {{user.public_metadata}}
}
```

#### Enterprise SSO (SAML) Support

Clerk offers **native Enterprise SSO** with SAML protocol support:

**Supported Providers**: Azure AD, Google Workspace, Okta Workforce, and custom SAML 2.0 IdPs

**Organization-Level SSO**: Enterprise connections are scoped to Clerk Organizations:
- Users signing in with enterprise connection are auto-added to the organization
- Domain-based enforcement (email domain must match configured SAML domain)
- Optional subdomain support (eTLD+1 matching)

**User Provisioning**:
- Automatic user creation on first sign-in (JIT provisioning)
- Account linking if user exists with matching email
- Automatic deprovisioning checks for EASIE connections (checks IdP for deleted/suspended users)

**Flows Supported**:
- SP-initiated flow (default)
- IdP-initiated flow

**Pricing**: Enterprise SSO requires Pro plan + Enhanced Authentication Add-on (free for development, max 25 connections)

**Important Limitation**: Clerk's Enterprise SAML is designed for **bilateral agreements** (one organization, one IdP). It's **not designed for multilateral federations** like InCommon where any member IdP should be able to authenticate users.

#### User Metadata System

Clerk provides three metadata types:

1. **Public Metadata**: Accessible by frontend and backend, set only on backend. Suitable for non-sensitive user attributes.

2. **Private Metadata**: Backend-only access. For sensitive data.

3. **Unsafe Metadata**: Set by frontend (untrusted). Not recommended for SAML attributes.

Metadata can be included in session tokens using shortcodes.

#### Webhook System

Clerk emits webhook events for user lifecycle:

**Key Events**:
- `user.created`: New user registered
- `user.updated`: User information changed
- `user.deleted`: User account removed

**Use Cases**:
- Syncing user data to external database
- Triggering provisioning workflows
- Maintaining audit logs

**Security**: Webhooks are signed (Svix) and should be verified using `verifyWebhook()` helper.

**Important Consideration**: Webhooks provide **eventual consistency** with potential delays causing race conditions.

### SAML-to-Modern-Auth Integration Patterns

Several architectural patterns exist for bridging legacy SAML authentication with modern OAuth/JWT-based systems:

#### 1. Identity Broker / Federation Proxy Pattern

**Description**: A middle layer accepts SAML requests from users, proxies to SAML IdP, then translates the SAML response into OAuth/OIDC tokens.

**Flow**:
1. User requests access to application
2. Application (relying party) sends OAuth/OIDC request to broker
3. Broker transforms request to SAML and proxies to SAML IdP
4. IdP authenticates user and returns SAML assertion to broker
5. Broker validates SAML assertion
6. Broker generates OAuth/OIDC tokens with claims from SAML attributes
7. Application receives OAuth/OIDC tokens

**Examples**: Auth0, Okta, FoxIDs SAML bridge, Cirrus Identity SAML Bridge

**Pros**:
- Clean separation of concerns
- Protocol translation is transparent to application
- Centralized policy enforcement

**Cons**:
- Additional infrastructure component
- Single point of failure
- Complexity in session lifecycle management

#### 2. SAML Bearer Assertion Flow (OAuth 2.0 Extension)

**Description**: OAuth 2.0 token endpoint accepts SAML assertion as a grant type (RFC 7522).

**Flow**:
1. Client obtains SAML bearer assertion from SAML IdP
2. Client requests OAuth access token from Authorization Server using SAML assertion as proof of identity
3. Authorization Server validates SAML assertion
4. Authorization Server issues OAuth access token

**Pros**:
- Standards-based approach
- No custom proxy needed
- Direct token exchange

**Cons**:
- Requires Authorization Server support for SAML bearer grants
- Limited support in modern platforms
- Client must handle SAML and OAuth

#### 3. Protocol Translation / Bridging Layer

**Description**: Authentication middleware intercepts SAML assertions and automatically converts to JWT claims.

**Flow**:
1. Login request routed as SAML authentication
2. SAML response received from external IdP
3. Bridge maps SAML claims to JWT claims automatically
4. Application receives JWT with OpenID Connect response

**Examples**: Serverless SAML-to-OIDC bridges, custom middleware

**Pros**:
- Automatic claim mapping
- Bi-directional support (SAML→OIDC and OIDC→SAML)
- Application agnostic

**Cons**:
- Custom development required
- Claims mapping complexity
- Session synchronization challenges

#### 4. Custom Authentication Backend

**Description**: Application implements custom authentication logic that accepts SAML assertions and creates native sessions.

**Flow**:
1. SimpleSAMLphp handles SAML authentication
2. Custom code extracts SAML attributes
3. Custom code creates or updates user in application database
4. Application generates its own session/JWT tokens
5. User accesses application with native session

**Pros**:
- Full control over authentication flow
- Direct integration with application logic
- No external dependencies

**Cons**:
- Highest development effort
- Maintenance burden
- Must implement security best practices manually

### How SAML-to-Clerk Bridge Works

For SimpleSAMLphp + Clerk integration, we combine elements of patterns #3 and #4:

**Conceptual Architecture**:

```
User → SimpleSAMLphp SP → InCommon IdP
                ↓
        SAML Assertion
                ↓
    Next.js API Route (ACS)
                ↓
        Extract Attributes
                ↓
    Map to Clerk User Schema
                ↓
    Create/Update Clerk User
        (Backend API)
                ↓
    Generate Clerk Session Token
                ↓
        Set Session Cookie
                ↓
    Redirect to Application
```

**Key Integration Points**:

1. **SAML Assertion Consumer Service (ACS)**: Next.js API route receives SAML response
2. **Attribute Extraction**: Parse SAML assertion for user attributes
3. **User Provisioning**: Create or update Clerk user via Backend API
4. **Session Creation**: Generate Clerk session token
5. **Session Handoff**: Set Clerk session cookie and redirect

## Implementation Feasibility

### Benefits

- **Unified User Management**: All users (SAML and non-SAML) managed in Clerk with consistent API
- **Leverages Existing Infrastructure**: Uses current Next.js + Clerk stack
- **Flexible Authentication**: Supports multiple authentication methods (password, OAuth, SAML)
- **Organization Isolation**: Clerk Organizations provide natural tenant boundaries
- **Rich User Interface**: Clerk's prebuilt components for user management, profile editing
- **Webhook Integration**: Real-time user lifecycle events for external system synchronization
- **Session Token Customization**: Include organization, SAML attributes in JWT claims

### Trade-offs & Challenges

- **Complexity**: Significant architectural complexity bridging two authentication systems

- **Session Synchronization**: Two session layers (SimpleSAMLphp and Clerk) must remain in sync
  - SimpleSAMLphp session must be cleaned up after Clerk session creation
  - Single Logout across both systems requires custom implementation

- **Attribute Mapping Maintenance**: SAML attributes vary by IdP, requiring flexible mapping logic

- **SimpleSAMLphp Hosting**: Additional infrastructure required
  - PHP runtime environment
  - Session storage (Redis/Memcache for production)
  - Certificate management
  - Metadata refresh automation

- **Cost**: Clerk Enterprise SSO requires Pro plan + Enhanced Authentication Add-on

- **Metadata Management**: InCommon metadata must be refreshed periodically (daily recommended)

- **IdP Discovery**: Users must select home institution; discovery service integration needed

- **Performance**: Additional latency from SAML flow then Clerk user creation/lookup

- **Error Handling**: Complex failure modes (SAML validation, Clerk API failures, attribute mapping errors)

- **Clerk's Enterprise SAML Limitation**: Not designed for multilateral federations; each organization requires separate SAML configuration

### When to Use

- **Multi-University Platform**: Your application serves multiple universities, each with InCommon membership
- **Mixed Authentication Requirements**: Some users need SAML, others use email/password or social login
- **Existing Clerk Investment**: You've already built significant functionality on Clerk and want to preserve it
- **Organization-Scoped Access**: Users from different institutions should be isolated into separate organizations
- **JIT Provisioning Acceptable**: Users are created on first login rather than pre-provisioned
- **Moderate User Scale**: Hundreds to thousands of users (not millions; double authentication overhead)

### When to Avoid

- **SimpleSAMLphp-Only Requirements**: If all users authenticate via InCommon, consider pure SimpleSAMLphp implementation
- **High Performance Requirements**: Double authentication flow adds latency; consider native SAML-only solution
- **Limited Development Resources**: Integration complexity requires experienced developers
- **Budget Constraints**: Clerk Enterprise SSO pricing may not be justified
- **Pre-Provisioning Required**: If users must exist before first login, additional automation needed
- **Single Organization**: If you only support one university, Clerk's native Enterprise SAML is simpler

## Implementation Options

### Option 1: SimpleSAMLphp as SAML Proxy IdP for Clerk (RECOMMENDED)

**Description**: SimpleSAMLphp acts as **both** a SAML Identity Provider (to Clerk) and a SAML Service Provider (to InCommon). This creates a "SAML proxy" that bridges Clerk's Enterprise SAML to InCommon's multilateral federation.

**Architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│                     User's Browser                           │
└────────┬────────────────────────────────────────────────┬───┘
         │                                                │
         │ 1. Click "Sign In" (Clerk component)          │
         ▼                                                │
┌────────────────────────────────┐                        │
│   Clerk (via Enterprise SAML)  │                        │
│   - Initiates SAML AuthnRequest│                        │
│   - to SimpleSAMLphp IdP       │                        │
└────────┬───────────────────────┘                        │
         │ 2. SAML AuthnRequest                           │
         ▼                                                │
┌────────────────────────────────┐                        │
│   SimpleSAMLphp (IdP mode)     │                        │
│   - Receives request from Clerk│                        │
│   - Acts as SP to InCommon     │                        │
│   - Initiates auth to Univ IdP │                        │
└────────┬───────────────────────┘                        │
         │ 3. SAML AuthnRequest                           │
         ▼                                                │
┌────────────────────────────────┐                        │
│   University IdP               │                        │
│   (via InCommon)               │                        │
│   - User authenticates         │                        │
└────────┬───────────────────────┘                        │
         │ 4. SAML Response                               │
         ▼                                                │
┌────────────────────────────────┐                        │
│   SimpleSAMLphp (IdP mode)     │                        │
│   - Validates assertion        │                        │
│   - Transforms attributes      │                        │
│   - Generates new SAML Response│                        │
│   - for Clerk                  │                        │
└────────┬───────────────────────┘                        │
         │ 5. SAML Response to Clerk                      │
         ▼                                                │
┌────────────────────────────────┐                        │
│   Clerk                        │                        │
│   - Creates/updates user       │                        │
│   - Creates session            │                        │
│   - Redirects to app           │────────────────────────┘
└────────┬───────────────────────┘   6. Redirect with session
         │
         ▼
┌────────────────────────────────┐
│   Next.js App                  │
│   User authenticated via Clerk │
└────────────────────────────────┘
```

**Pros**:
- **Seamless Clerk integration** - Uses Clerk's native Enterprise SAML
- **Single login flow** - No separate "University login" button needed
- **Clerk handles everything** - User creation, session management, UI components all work normally
- **SimpleSAMLphp does the heavy lifting** - Handles InCommon metadata, IdP discovery, SAML complexity
- **Cleanest architecture** - Each component does what it does best
- **No custom API routes** - No need to manually create Clerk users or sessions
- **Organization-based** - Can configure per Clerk Organization

**Cons**:
- **SimpleSAMLphp runs in dual mode** - More complex SimpleSAMLphp configuration (both IdP and SP)
- **Still requires SimpleSAMLphp hosting** - Need PHP infrastructure
- **Attribute mapping in two places** - SimpleSAMLphp transforms attributes, then Clerk maps them again
- **Per-organization setup in Clerk** - Each Clerk Organization needs its own SAML connection to SimpleSAMLphp

**Complexity**: Medium-High (SimpleSAMLphp configuration is complex, but Clerk integration is simple)

**Time Estimate**: 1-2 weeks

**When to Use**:
- You want to keep Clerk as the primary authentication system
- You want users to use Clerk's standard `<SignIn />` component
- You're willing to run SimpleSAMLphp in both IdP and SP modes
- You want InCommon multilateral federation support
- You want Clerk to handle all user/session management

**How It Works**:

1. **Configure SimpleSAMLphp as IdP**: SimpleSAMLphp can act as an IdP using the `saml:SP` authentication source as a backend
2. **Configure SimpleSAMLphp as SP**: Same SimpleSAMLphp instance also acts as SP to InCommon
3. **Configure Clerk Enterprise SAML**: Point Clerk to SimpleSAMLphp's IdP metadata
4. **User Flow**: Clerk → SimpleSAMLphp IdP → InCommon → University IdP → back through the chain

**SimpleSAMLphp Configuration** (simplified):

```php
// config/authsources.php

// SimpleSAMLphp as SP (to InCommon)
'incommon-sp' => [
    'saml:SP',
    'entityID' => 'https://yourapp.com/saml/sp',
    'idp' => null, // Use discovery service
],

// SimpleSAMLphp as IdP (to Clerk)
// Uses the SP above as its authentication source
'hosted-idp' => [
    'auth' => 'incommon-sp', // Use InCommon SP for authentication
    'attributes.NameFormat' => 'urn:oasis:names:tc:SAML:2.0:attrname-format:uri',
    'authproc' => [
        // Transform InCommon attributes to Clerk-friendly format
        100 => [
            'class' => 'core:AttributeMap',
            'oid2name',
        ],
    ],
],
```

**Clerk Configuration**:

1. Create Enterprise Connection in Clerk Dashboard
2. Select "SAML" as connection type
3. Upload SimpleSAMLphp IdP metadata from: `https://your-simplesaml.example.com/saml2/idp/metadata.php`
4. Configure attribute mapping (email, firstName, lastName)
5. Assign to Organization(s)

**User Experience**:

```typescript
// app/sign-in/page.tsx
import { SignIn } from '@clerk/nextjs';

export default function SignInPage() {
  return (
    <SignIn
      // Clerk automatically shows configured SAML connections
      // Users see "Continue with [Your Organization]" button
    />
  );
}
```

### Option 2: Clerk Native Enterprise SAML (Bilateral)

**Description**: Use Clerk's built-in Enterprise SAML without SimpleSAMLphp. Each client organization registers their own SAML connection in Clerk.

**Pros**:
- Simplest implementation (no custom code)
- Fully managed by Clerk
- Native Clerk UI for SSO configuration
- Automatic user provisioning
- Built-in session management
- No additional infrastructure

**Cons**:
- Not true multilateral federation (each organization configures separately)
- Requires each client to expose their IdP metadata
- Client must configure your SP in their IdP
- No InCommon metadata consumption
- Limited to organizations with SAML expertise

**Complexity**: Low

**Time Estimate**: 1-2 days for first organization, < 1 hour per additional organization

**Reuses Patterns**: Yes (Clerk's native features)

**When to Use**:
- Small number of client organizations (< 10)
- Clients have IT staff capable of SAML configuration
- Bilateral SAML agreements are acceptable
- No requirement for "any InCommon member" access

**Example/Reference**: https://clerk.com/docs/authentication/enterprise-connections/saml/custom-provider

### Option 3: SimpleSAMLphp SP with Custom Clerk Integration

**Description**: SimpleSAMLphp acts **ONLY** as a SAML Service Provider (SP) that communicates directly with InCommon university IdPs. After SAML authentication succeeds, Next.js API routes receive the SAML assertion, extract user attributes, and create Clerk users via Backend API. SimpleSAMLphp handles all SAML protocol complexity; Clerk handles all session management.

**Important**: SimpleSAMLphp does NOT act as an IdP in this architecture. It's purely an SP that bridges SAML authentication to Clerk sessions via custom code.

**Architecture**:

```
┌─────────────────────────────────────────────────────────────┐
│                     InCommon Federation                      │
│  (UC Davis IdP, UCLA IdP, Stanford IdP, ...)                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         │ SAML AuthnRequest / Response
                         │
                         ▼
        ┌────────────────────────────────┐
        │      SimpleSAMLphp SP          │
        │  - Consumes InCommon metadata  │
        │  - Validates SAML assertions   │
        │  - Extracts attributes         │
        └────────────────┬───────────────┘
                         │
                         │ POST SAML Response
                         │
                         ▼
        ┌────────────────────────────────┐
        │   Next.js API Route (ACS)      │
        │   /api/auth/saml/callback      │
        │  - Parse SAML assertion        │
        │  - Map attributes to user data │
        └────────────────┬───────────────┘
                         │
                         │ Clerk Backend API
                         │
                         ▼
        ┌────────────────────────────────┐
        │         Clerk Backend          │
        │  - Create/update user          │
        │  - Add to organization         │
        │  - Generate session token      │
        └────────────────┬───────────────┘
                         │
                         │ Session token
                         │
                         ▼
        ┌────────────────────────────────┐
        │      Next.js Application       │
        │  - Set __session cookie        │
        │  - Render authenticated UI     │
        └────────────────────────────────┘
```

**Pros**:
- True InCommon multilateral federation support
- Single SP registration in InCommon (not per-organization)
- Automatic IdP discovery for users
- Centralized SAML configuration
- Full control over attribute mapping
- Clerk handles session management and user lifecycle

**Cons**:
- Requires SimpleSAMLphp hosting (PHP environment)
- Custom code for SAML callback handling
- Two authentication layers to maintain
- Session synchronization complexity
- Additional infrastructure (PHP, Redis/Memcache)

**Complexity**: High

**Time Estimate**: 2-3 weeks

**Reuses Patterns**: Partial (uses Clerk for sessions, custom SAML integration)

**When to Use**:
- Need true InCommon multilateral federation
- Want "any InCommon member can authenticate" functionality
- Have PHP hosting capability
- Willing to invest in custom integration

**Technical Implementation Details**:

**1. SimpleSAMLphp Configuration**

`config/authsources.php`:
```php
<?php
$config = [
    'default-sp' => [
        'saml:SP',
        'entityID' => 'https://yourapp.com/saml/metadata',
        'metadata.sign.enable' => true,
        'metadata.sign.privatekey' => 'saml.pem',
        'metadata.sign.certificate' => 'saml.crt',
        'AssertionConsumerService' => 'https://yourapp.com/api/auth/saml/acs',
        'SingleLogoutService' => 'https://yourapp.com/api/auth/saml/slo',
    ],
];
```

`config/config.php`:
```php
'metadata.sources' => [
    [
        'type' => 'mdq',
        'server' => 'https://mdq.incommon.org',
        'validateCertificate' => ['/path/to/inc-md-cert-mdq.pem'],
        'cachedir' => '/var/cache/simplesamlphp/mdq',
        'cachelength' => 86400,
    ],
],
'store.type' => 'redis',
'store.redis.host' => 'localhost',
'store.redis.port' => 6379,
```

**2. Next.js API Route for ACS**

`app/api/auth/saml/acs/route.ts`:
```typescript
import { NextRequest, NextResponse } from 'next/server';
import { clerkClient } from '@clerk/nextjs/server';
import { parseSAMLResponse } from '@/lib/saml';

export async function POST(request: NextRequest) {
  try {
    // Parse SAML response from SimpleSAMLphp
    const formData = await request.formData();
    const samlResponse = formData.get('SAMLResponse') as string;
    const relayState = formData.get('RelayState') as string;

    // Validate and extract attributes
    const assertion = await parseSAMLResponse(samlResponse);
    const attributes = assertion.attributes;

    // Extract key attributes
    const eppn = attributes['urn:oid:1.3.6.1.4.1.5923.1.1.1.6']?.[0]; // eduPersonPrincipalName
    const email = attributes['urn:oid:0.9.2342.19200300.100.1.3']?.[0]; // mail
    const displayName = attributes['urn:oid:2.16.840.1.113730.3.1.241']?.[0];
    const idpEntityId = assertion.issuer; // IdP entity ID

    if (!eppn || !email) {
      throw new Error('Missing required SAML attributes');
    }

    // Determine organization from IdP entity ID
    const orgId = await getOrganizationFromIdP(idpEntityId);

    // Create or update user in Clerk
    let user = await clerkClient.users.getUserList({
      emailAddress: [email],
    }).then(users => users[0]);

    if (!user) {
      user = await clerkClient.users.createUser({
        emailAddress: [email],
        firstName: displayName?.split(' ')[0],
        lastName: displayName?.split(' ').slice(1).join(' '),
        publicMetadata: {
          saml_eppn: eppn,
          saml_idp: idpEntityId,
          auth_method: 'saml',
        },
      });

      // Add user to organization
      if (orgId) {
        await clerkClient.organizations.createOrganizationMembership({
          organizationId: orgId,
          userId: user.id,
          role: 'member',
        });
      }
    } else {
      // Update user metadata
      await clerkClient.users.updateUser(user.id, {
        publicMetadata: {
          ...user.publicMetadata,
          saml_eppn: eppn,
          saml_idp: idpEntityId,
          saml_last_login: new Date().toISOString(),
        },
      });
    }

    // Create Clerk session
    const session = await clerkClient.sessions.createSession({
      userId: user.id,
    });

    // Generate session token
    const token = await session.getToken();

    // Set Clerk session cookie
    const response = NextResponse.redirect(relayState || '/dashboard');
    response.cookies.set('__session', token, {
      httpOnly: true,
      secure: true,
      sameSite: 'lax',
      path: '/',
      maxAge: 60 * 60 * 24 * 7, // 7 days
    });

    return response;
  } catch (error) {
    console.error('SAML ACS error:', error);
    return NextResponse.redirect('/auth/error?type=saml_error');
  }
}
```

**3. Organization Mapping**

```typescript
// lib/saml-organization-mapping.ts
const IDP_TO_ORG_MAP: Record<string, string> = {
  'urn:mace:incommon:ucdavis.edu': 'org_ucdavis',
  'urn:mace:incommon:ucla.edu': 'org_ucla',
  'urn:mace:incommon:stanford.edu': 'org_stanford',
  // ... more mappings
};

export async function getOrganizationFromIdP(idpEntityId: string): Promise<string | null> {
  const orgId = IDP_TO_ORG_MAP[idpEntityId];

  if (!orgId) {
    // Create new organization for unmapped IdP
    const institution = extractInstitutionName(idpEntityId);
    const org = await clerkClient.organizations.createOrganization({
      name: institution,
      publicMetadata: {
        saml_idp_entity_id: idpEntityId,
      },
    });

    IDP_TO_ORG_MAP[idpEntityId] = org.id;
    return org.id;
  }

  return orgId;
}
```

### Option 4: Passport-SAML with Next.js (No SimpleSAMLphp)

**Description**: Use node-saml/passport-saml library in Next.js API routes to handle SAML directly, then create Clerk users.

**Pros**:
- No PHP infrastructure needed
- Full JavaScript/TypeScript stack
- Direct control over SAML flow
- Integrates naturally with Next.js
- Passport.js middleware ecosystem

**Cons**:
- Must implement SAML SP logic in JavaScript
- InCommon metadata management complexity
- No built-in IdP discovery service
- More code to maintain vs SimpleSAMLphp
- passport-saml less mature than SimpleSAMLphp for federation

**Complexity**: Medium-High

**Time Estimate**: 2-4 weeks

**Reuses Patterns**: Partial (similar to Option 2, but SAML handling in Node.js)

**When to Use**:
- Want to avoid PHP infrastructure
- Comfortable with JavaScript SAML libraries
- Need deep customization of SAML flow
- Prefer single-language stack

**Example/Reference**: https://github.com/node-saml/passport-saml

**Technical Implementation Details**:

**1. Passport-SAML Configuration**

```typescript
// lib/saml-config.ts
import { Strategy as SamlStrategy, VerifyWithoutRequest } from 'passport-saml';
import { Profile } from 'passport';

const samlStrategy = new SamlStrategy(
  {
    callbackUrl: 'https://yourapp.com/api/auth/saml/callback',
    entryPoint: 'https://idp.example.com/sso',
    issuer: 'https://yourapp.com',
    cert: process.env.SAML_CERT, // IdP certificate
    privateKey: process.env.SAML_PRIVATE_KEY,
    decryptionPvk: process.env.SAML_DECRYPTION_KEY,
    signatureAlgorithm: 'sha256',
    identifierFormat: 'urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress',
    wantAssertionsSigned: true,
    wantAuthnResponseSigned: true,
  },
  async (profile: Profile, done: VerifyWithoutRequest) => {
    // Verification callback
    const user = {
      id: profile.nameID,
      email: profile.email,
      displayName: profile.displayName,
      eppn: profile['urn:oid:1.3.6.1.4.1.5923.1.1.1.6'],
    };

    done(null, user);
  }
);
```

**2. API Routes**

```typescript
// app/api/auth/saml/login/route.ts
import { samlStrategy } from '@/lib/saml-config';

export async function GET(request: NextRequest) {
  return new Promise((resolve, reject) => {
    samlStrategy.authenticate(request as any, {
      successRedirect: '/dashboard',
      failureRedirect: '/login',
    });
  });
}

// app/api/auth/saml/callback/route.ts
export async function POST(request: NextRequest) {
  const formData = await request.formData();

  // Validate SAML response
  const profile = await samlStrategy.validateResponse(formData);

  // Create Clerk user (same as Option 2)
  // ...
}
```

**3. InCommon Metadata Handling**

```typescript
// lib/incommon-metadata.ts
import fetch from 'node-fetch';
import { parseStringPromise } from 'xml2js';

export async function fetchInCommonMetadata(): Promise<any> {
  const response = await fetch('https://mdq.incommon.org/entities/{entityID}');
  const xml = await response.text();
  return parseStringPromise(xml);
}

export async function getIdPCertificate(entityId: string): Promise<string> {
  const metadata = await fetchInCommonMetadata();
  // Parse and extract certificate
  // ...
}
```

## Comparison Matrix

| Criteria | Option 1: SAML Proxy | Option 2: Clerk Native | Option 3: Custom SP Bridge | Option 4: Passport-SAML |
|----------|---------------------|------------------------|---------------------------|------------------------|
| Complexity | Medium-High | Low | High | Medium-High |
| Clerk Integration | Seamless (native) | Seamless (native) | Custom API routes | Custom API routes |
| Login UX | Standard Clerk | Standard Clerk | Separate login button | Separate login button |
| Maintainability | Medium | High | Medium | Medium |
| Development Time | 1-2 weeks | 1-2 days | 2-3 weeks | 2-4 weeks |
| Infrastructure | PHP (IdP+SP) | None (Clerk SaaS) | PHP (SP only) | Node.js only |
| InCommon Federation | Yes (multilateral) | No (bilateral only) | Yes (multilateral) | Yes (with work) |
| Learning Curve | Medium | Low | Medium | Medium-High |
| Community Support | Strong | Strong (Clerk) | Strong | Moderate |
| IdP Discovery | Available | Not needed | Available | Must implement |
| Metadata Management | Automated (MDQ) | Per-organization | Automated (MDQ) | Manual |
| Custom Code Needed | Minimal | None | Significant | Significant |
| Cost | Infrastructure + Clerk Pro | Clerk Pro + Add-on | Infrastructure + Clerk Pro | Infrastructure + Clerk Pro |
| Scalability | Good (needs caching) | High (Clerk managed) | Good (needs caching) | Good (stateless) |
| Testing Complexity | Medium (multi-component) | Low | High (multi-component) | Medium |
| Protocol Support | SAML 2.0 + SLO | SAML 2.0 | SAML 2.0 + SLO | SAML 2.0 |
| Multi-Language Stack | Yes (PHP + JS) | No | Yes (PHP + JS) | No (JS only) |

## Implementation Approach

### Recommended: Option 1 (SimpleSAMLphp as SAML Proxy IdP)

Based on research, **Option 1 provides the best balance** for true InCommon multilateral federation while fully leveraging Clerk's native features. This approach:

- ✅ Uses Clerk's standard `<SignIn />` component (seamless UX)
- ✅ Leverages Clerk's Enterprise SAML (no custom auth code)
- ✅ Supports InCommon multilateral federation (any university IdP)
- ✅ SimpleSAMLphp handles all SAML complexity
- ✅ Clerk handles all user/session management

**Key Advantage over Option 3**: Option 1 uses Clerk's native Enterprise SAML, so you don't need custom API routes to create users/sessions. Clerk does this automatically when it receives the SAML assertion from SimpleSAMLphp's IdP.

### Prerequisites & Requirements

**Infrastructure**:
- PHP 7.4+ runtime environment
- Redis or Memcache for session storage (production)
- Web server (Apache/Nginx) for SimpleSAMLphp
- SSL/TLS certificates for SAML endpoints
- Next.js 13+ with App Router
- Clerk account with Pro plan + Enhanced Authentication Add-on

**Skills Required**:
- PHP configuration (SimpleSAMLphp setup)
- Next.js API routes development
- SAML protocol knowledge
- Certificate management
- Redis/Memcache operations
- Clerk Backend API usage

**InCommon Membership**:
- Organization must be InCommon participant
- Entity ID registration with InCommon Federation
- SAML metadata publication

### Getting Started

**Phase 1: SimpleSAMLphp Setup (Week 1)**

1. **Install SimpleSAMLphp**
   ```bash
   composer require simplesamlphp/simplesamlphp
   ```

2. **Configure Entity**
   - Generate certificate and private key
   - Set entity ID: `https://yourapp.com/saml`
   - Configure ACS URL: `https://yourapp.com/api/auth/saml/acs`
   - Configure SLO URL: `https://yourapp.com/api/auth/saml/slo`

3. **Configure InCommon Metadata**
   - Download InCommon MDQ certificate
   - Configure MDQ source in `config.php`
   - Test metadata retrieval

4. **Test Authentication**
   - Use SimpleSAMLphp test interface
   - Authenticate against test IdP
   - Verify attribute release

5. **Register with InCommon**
   - Submit metadata to InCommon Federation
   - Configure entity categories and attribute requirements
   - Wait for approval (typically 1-2 business days)

**Phase 2: Next.js Integration (Week 2)**

1. **Create API Routes**
   - `/api/auth/saml/login` - Initiate SAML flow
   - `/api/auth/saml/acs` - Assertion Consumer Service
   - `/api/auth/saml/metadata` - Proxy SimpleSAMLphp metadata
   - `/api/auth/saml/slo` - Single Logout handler

2. **Implement SAML Response Parsing**
   - Call SimpleSAMLphp API from Node.js (via HTTP or shared filesystem)
   - Extract attributes from SAML assertion
   - Validate assertion conditions and signatures

3. **Implement Clerk Integration**
   - Create/update user via Clerk Backend API
   - Map SAML attributes to Clerk metadata
   - Handle organization membership

4. **Session Management**
   - Create Clerk session after SAML authentication
   - Set `__session` cookie
   - Clean up SimpleSAMLphp session

**Phase 3: Organization Mapping (Week 2-3)**

1. **Build IdP-to-Organization Mapping**
   - Create mapping table (IdP entity ID → Clerk organization ID)
   - Implement auto-organization creation for new IdPs
   - Add admin UI for mapping management

2. **Implement Organization Auto-Provisioning**
   - Extract institution name from IdP metadata
   - Create Clerk organization on first user login
   - Set organization metadata (IdP entity ID, institution name)

**Phase 4: Testing & Refinement (Week 3)**

1. **Integration Testing**
   - Test with multiple InCommon IdPs
   - Verify attribute mapping for different IdPs
   - Test organization isolation
   - Test error scenarios (invalid assertions, missing attributes)

2. **Security Testing**
   - Verify signature validation
   - Test session timeouts
   - Test replay attack protection
   - Verify CSRF protection on API routes

3. **User Experience Testing**
   - Test IdP discovery flow
   - Test first-time login (user creation)
   - Test returning user login
   - Test logout (local and SLO)

### Architecture & Design Considerations

**Session Lifecycle**:
1. User initiates login → Redirected to SimpleSAMLphp
2. SimpleSAMLphp creates SAML session
3. User authenticates at IdP
4. SimpleSAMLphp validates assertion, stores session
5. Next.js API route processes callback
6. Clerk user created/updated, Clerk session created
7. SimpleSAMLphp session cleaned up (optional, or retained for SLO)
8. User redirected to application with Clerk session

**Data Flow**:
```
InCommon IdP → SAML Assertion → SimpleSAMLphp → SAML Attributes
                                                        ↓
                                            Next.js API Route
                                                        ↓
                                        Attribute Mapping Logic
                                                        ↓
                                    Clerk User Object + Metadata
                                                        ↓
                                            Clerk Session Token
                                                        ↓
                                            Next.js Application
```

**Error Handling Strategy**:
- **SAML Validation Errors**: Redirect to error page with message
- **Missing Required Attributes**: Log error, show user-friendly message
- **Clerk API Errors**: Retry with exponential backoff, fallback to error page
- **Organization Mapping Errors**: Create default organization or show admin notification

**Attribute Mapping**:
```typescript
interface SAMLAttributes {
  eduPersonPrincipalName?: string[];
  mail?: string[];
  displayName?: string[];
  eduPersonScopedAffiliation?: string[];
  eduPersonTargetedID?: string[];
  givenName?: string[];
  sn?: string[]; // surname
}

interface ClerkUserData {
  emailAddress: string[];
  firstName?: string;
  lastName?: string;
  publicMetadata: {
    saml_eppn: string;
    saml_idp: string;
    saml_affiliation?: string;
    auth_method: 'saml';
  };
}

function mapSAMLToClerk(attributes: SAMLAttributes, idpEntityId: string): ClerkUserData {
  return {
    emailAddress: [attributes.mail?.[0] || attributes.eduPersonPrincipalName?.[0] || ''],
    firstName: attributes.givenName?.[0] || attributes.displayName?.[0]?.split(' ')[0],
    lastName: attributes.sn?.[0] || attributes.displayName?.[0]?.split(' ').slice(1).join(' '),
    publicMetadata: {
      saml_eppn: attributes.eduPersonPrincipalName?.[0] || '',
      saml_idp: idpEntityId,
      saml_affiliation: attributes.eduPersonScopedAffiliation?.[0],
      auth_method: 'saml',
    },
  };
}
```

### Best Practices

**SAML Security**:
- Always validate SAML assertion signatures [InCommon Federation Security Requirements]
- Implement replay detection (track assertion IDs with expiration)
- Set short assertion lifetime (5 minutes default is appropriate) [OWASP SAML Security]
- Use strong signing algorithms (SHA-256 minimum)
- Validate NotBefore and NotOnOrAfter conditions
- Verify Audience matches your entity ID
- Use encrypted assertions for sensitive attributes
- Validate IdP certificate against InCommon metadata

**Session Management**:
- Keep SAML assertion lifetime short (5 minutes) for clock skew tolerance
- Set application session timeout based on risk assessment (not tied to assertion lifetime)
- Implement idle timeout for inactive sessions
- Provide explicit logout functionality
- Support SAML Single Logout for security-sensitive applications

**Performance Optimizations**:
- Cache InCommon metadata (24-hour cache lifetime recommended)
- Use Redis/Memcache for SimpleSAMLphp sessions in production
- Implement lazy organization creation (only when users login)
- Batch Clerk API calls when possible
- Add monitoring for SAML flow latency

**InCommon Best Practices**:
- Request minimal necessary attributes (ePPN + mail typically sufficient)
- Support eduPersonPrincipalName as primary identifier [InCommon Attribute Recommendations]
- Validate attribute scope matches IdP's registered domain
- Publish metadata to InCommon with correct entity categories
- Use MDQ service instead of full metadata aggregate [InCommon MDQ Service]
- Monitor InCommon announcements for metadata changes

**Testing Strategies**:

*Unit Tests*:
- Test attribute mapping logic with various IdP attribute combinations
- Test organization lookup/creation logic
- Test SAML response parsing
- Mock Clerk API responses

*Integration Tests*:
- Test full authentication flow with InCommon Test IdP
- Test with multiple production IdPs (UC Davis, UCLA, Stanford)
- Test error scenarios (malformed SAML, missing attributes)
- Test organization isolation (users from different IdPs)
- Avoid mocking SAML validation (use real SimpleSAMLphp validation)

*Security Tests*:
- Test with expired assertions
- Test with replayed assertions
- Test with tampered assertions
- Test with invalid signatures
- Test cross-organization access attempts

### Common Pitfalls & How to Avoid Them

1. **Session Conflicts Between SimpleSAMLphp and Application**
   - **Problem**: SimpleSAMLphp closes active PHP sessions
   - **Solution**: Call `\SimpleSAML\Session::getSessionFromRequest()->cleanup()` after authentication, before accessing application session
   - **Reference**: SimpleSAMLphp SP API documentation

2. **Attribute Name Variations Across IdPs**
   - **Problem**: Different IdPs use different attribute names (OID vs friendly names)
   - **Solution**: Implement flexible attribute mapping supporting both OID and friendly names
   - **Example**:
     ```typescript
     const eppn = attributes['urn:oid:1.3.6.1.4.1.5923.1.1.1.6']?.[0]
                  || attributes['eduPersonPrincipalName']?.[0];
     ```

3. **Large Metadata Files Causing Performance Issues**
   - **Problem**: SimpleSAMLphp < 1.13.2 struggles with InCommon's full metadata
   - **Solution**: Use MDQ service instead of full metadata aggregate, upgrade to SimpleSAMLphp 1.14+
   - **Reference**: InCommon Federation Software Guidelines

4. **Forgotten SimpleSAMLphp Session Cleanup Causing SLO Failures**
   - **Problem**: SimpleSAMLphp sessions remain active after creating Clerk sessions, preventing proper logout
   - **Solution**: Decide on SLO strategy upfront:
     - Option A: Preserve SimpleSAMLphp session for SLO support
     - Option B: Clean up SimpleSAMLphp session, implement custom SLO via Clerk webhooks
   - **Reference**: SimpleSAMLphp SLO documentation

5. **Clerk Session Token Size Limit (4KB Cookie)**
   - **Problem**: Adding too many SAML attributes to session token exceeds cookie limit
   - **Solution**: Store only essential attributes in session token (user ID, org ID), fetch additional attributes from Clerk metadata via API
   - **Reference**: Clerk Session Token documentation

6. **Missing IdP Discovery Service**
   - **Problem**: With MDQ, no local metadata for discovery service
   - **Solution**: Implement custom IdP discovery:
     - Provide searchable list of InCommon members
     - Remember user's last IdP (cookie)
     - Domain-based auto-detection (email suffix → IdP)
   - **Reference**: InCommon MDQ configuration

7. **Race Conditions with Clerk Webhooks**
   - **Problem**: Eventual consistency of webhooks causes data sync issues
   - **Solution**: Don't rely on webhooks for critical path; use synchronous Clerk Backend API calls during authentication flow
   - **Reference**: Clerk Webhooks documentation on eventual consistency

### Migration/Adoption Strategy

**For New Deployment**:

Phase 1: Foundation (Weeks 1-2)
- Set up SimpleSAMLphp in development environment
- Register test entity with InCommon Federation
- Implement basic SAML authentication flow
- Create initial Next.js API routes

Phase 2: Clerk Integration (Weeks 2-3)
- Implement user provisioning via Clerk Backend API
- Build organization mapping system
- Test with InCommon Test IdP

Phase 3: Production Readiness (Week 4)
- Production SimpleSAMLphp deployment (with Redis)
- Security testing and penetration testing
- Performance testing and optimization
- Documentation and runbooks

Phase 4: Onboarding (Week 5+)
- Pilot with first client organization
- Gather feedback and iterate
- Gradual rollout to additional organizations

**For Existing Clerk Deployment**:

Phase 1: Parallel Authentication (Weeks 1-2)
- Deploy SimpleSAMLphp without disrupting existing auth
- Create SAML login route separate from existing flows
- Test with select pilot users

Phase 2: Organization Migration (Weeks 3-4)
- Migrate one organization to SAML authentication
- Provide users with both login options during transition
- Monitor for issues

Phase 3: Full Rollout (Week 5+)
- Migrate remaining organizations
- Deprecate old authentication methods as appropriate
- Maintain backward compatibility

**Rollback Strategy**:
- Keep existing authentication methods active during SAML rollout
- Feature flag SAML authentication at organization level
- Maintain ability to disable SAML globally via environment variable
- Database backups before major migrations
- Monitoring and alerting for SAML authentication failures

## Alternatives Considered

### Alternative 1: Pure SimpleSAMLphp (No Clerk)

**Description**: Use SimpleSAMLphp for both authentication and session management, ditching Clerk entirely.

**Why Not Chosen**:
- Loses Clerk's rich user management UI
- Loses Clerk's prebuilt components
- Loses Clerk's webhook system
- Requires building custom user database
- Requires custom session management
- Doesn't leverage existing Clerk investment

**When It Might Be Better**:
- Starting from scratch with no existing auth system
- All users authenticate via SAML (no mixed auth methods)
- Don't need Clerk's features (prebuilt UI, webhooks, etc.)
- Want to minimize vendor lock-in

### Alternative 2: Auth0 or Okta

**Description**: Use commercial identity broker (Auth0, Okta) that supports both InCommon SAML and modern protocols.

**Why Not Chosen**:
- Already invested in Clerk
- Additional vendor and cost
- Auth0/Okta have similar limitations with multilateral federation
- Still requires integration work

**When It Might Be Better**:
- Starting from scratch
- Need enterprise features (advanced MFA, risk detection)
- Want fully managed SAML bridge
- Budget allows for enterprise IdP pricing

### Alternative 3: Shibboleth Service Provider

**Description**: Use Shibboleth SP (the original InCommon standard) instead of SimpleSAMLphp.

**Why Not Chosen**:
- More complex to configure than SimpleSAMLphp
- Heavier infrastructure requirements (Apache module)
- SimpleSAMLphp provides equivalent functionality with simpler deployment
- Harder to integrate with Next.js (designed for Apache/PHP apps)

**When It Might Be Better**:
- Existing Shibboleth expertise
- Running traditional Apache/PHP application
- Need advanced Shibboleth features (attribute filtering, caching)

## Debates & Open Questions

**Debate: Should SimpleSAMLphp Sessions Be Retained for SLO?**

- **Retain Sessions**: Enables proper SAML Single Logout across all SPs in federation
  - Pros: Standards-compliant, better security (logout propagates)
  - Cons: Two session layers, complexity, SimpleSAMLphp sessions must be maintained

- **Discard Sessions**: Clean up SimpleSAMLphp session after Clerk session creation
  - Pros: Simpler architecture, single session layer (Clerk)
  - Cons: SAML SLO doesn't work, must implement custom logout via Clerk

**Current Recommendation**: Retain SimpleSAMLphp sessions for SLO support unless application risk assessment determines SLO is not required.

**Open Question: How to Handle IdP Metadata Changes?**

InCommon IdPs occasionally rotate certificates and update metadata. Options:
- Poll InCommon metadata hourly/daily (adds latency to lookups)
- Subscribe to InCommon metadata change notifications (requires webhook handling)
- Use MDQ with long cache (risk of stale data if IdP changes suddenly)

**Current Recommendation**: Use MDQ with 24-hour cache and implement monitoring for authentication failures that may indicate stale metadata.

**Open Question: Where Should SAML Attribute Mapping Rules Live?**

Options:
- Hardcoded in application (simple, fast, requires deployment to change)
- Database table (flexible, admin-editable, requires migration for schema changes)
- Configuration file (version-controlled, requires restart to change)
- Admin UI (most flexible, most complex)

**Current Recommendation**: Start with hardcoded mapping, migrate to database table if mapping complexity grows.

**Debate: Should We Support Non-InCommon SAML IdPs?**

The architecture supports any SAML 2.0 IdP, not just InCommon members.

- **InCommon-Only**: Simpler metadata management, consistent attribute release
- **Any SAML IdP**: More flexible, supports non-InCommon institutions

**Current Recommendation**: Support any SAML 2.0 IdP, but document InCommon as primary use case.

## Recommendations

### Preferred Approach: Option 1 (SimpleSAMLphp as SAML Proxy IdP for Clerk)

**Should This Be Implemented?**: **Conditional - Yes if true InCommon multilateral federation is required; No if bilateral SAML agreements are acceptable**

**Rationale**:

1. **InCommon Multilateral Federation**: Option 1 provides true multilateral federation where any InCommon member IdP can authenticate users without bilateral agreements. This is the defining feature of InCommon.

2. **Seamless Clerk Integration**: Uses Clerk's native Enterprise SAML feature, so all of Clerk's components, hooks, and features work out of the box. No custom authentication code needed in your Next.js app.

3. **Clean Architecture**: SimpleSAMLphp acts as a "SAML proxy" - it's an IdP to Clerk, but an SP to InCommon. This creates a clean separation of concerns:
   - SimpleSAMLphp: Handles SAML protocol, InCommon metadata, IdP discovery
   - Clerk: Handles users, sessions, authentication UI, webhooks
   - Your App: Just uses standard Clerk components

4. **Proven Technology**: SimpleSAMLphp is the de-facto standard for InCommon federation in higher education, with extensive documentation and community support.

5. **Standard User Experience**: Users see Clerk's standard login UI with a "Continue with [Organization]" button. No separate "University login" flow needed.

**Alternative**: If only working with a small number of known institutions (< 5-10), **Option 2 (Clerk Native Enterprise SAML)** is significantly simpler and should be preferred. Each organization configures their SAML connection directly in Clerk, bypassing SimpleSAMLphp entirely.

**Why**:

**Pros**:
- True multilateral federation (any InCommon member can authenticate)
- Single SP registration with InCommon
- Mature, well-tested SAML implementation
- Clerk handles session management and user lifecycle
- Flexible attribute mapping
- Organization-based multi-tenancy

**Cons**:
- High implementation complexity (2-3 weeks)
- Requires PHP infrastructure
- Two authentication layers to maintain
- Additional infrastructure costs

**Key Considerations**:

1. **Infrastructure Requirements**: Must provision PHP environment, Redis/Memcache, and manage SimpleSAMLphp deployment. This adds operational complexity.

2. **Development Expertise**: Requires developers with SAML protocol knowledge, SimpleSAMLphp configuration experience, and Next.js API development skills.

3. **InCommon Membership**: Organization must be InCommon participant or sponsored affiliate. Entity registration takes 1-2 business days.

4. **Clerk Pricing**: Requires Clerk Pro plan + Enhanced Authentication Add-on. Evaluate cost vs. benefits.

5. **Metadata Management**: Must implement automated InCommon metadata refresh (daily recommended).

6. **Session Synchronization**: Two session systems (SimpleSAMLphp and Clerk) must be carefully managed to prevent logout/security issues.

**Potential Challenges**:

1. **SimpleSAMLphp Hosting and Maintenance**
   - Mitigation: Use Docker containerization for consistent deployment; implement automated metadata refresh; monitor SimpleSAMLphp logs
   - Fallback: If PHP hosting is not feasible, consider Option 3 (Passport-SAML) despite higher complexity

2. **Session Lifecycle Complexity**
   - Mitigation: Document session flow thoroughly; implement comprehensive testing for login/logout scenarios; decide on SLO strategy upfront
   - Fallback: If SLO is critical and complexity is too high, consider commercial identity broker (Auth0/Okta)

3. **Attribute Mapping Variability**
   - Mitigation: Implement flexible mapping supporting both OID and friendly names; provide admin UI for mapping overrides; extensive testing with multiple IdPs
   - Fallback: Document required attributes in InCommon metadata; work with IdPs to ensure consistent attribute release

4. **Performance Overhead**
   - Mitigation: Use Redis for SimpleSAMLphp sessions; implement MDQ caching; monitor authentication latency; optimize Clerk API calls
   - Fallback: If latency is unacceptable, consider pure SimpleSAMLphp without Clerk (loses Clerk benefits)

**Success Criteria**:

- Users from any InCommon member institution can successfully authenticate
- Authentication completes in < 3 seconds (90th percentile)
- Users are correctly assigned to organizations based on IdP
- SAML attributes are accurately mapped to Clerk user metadata
- Session tokens include necessary organization and user context
- Logout (local and SLO) functions correctly
- Zero authentication-related security incidents in first 90 days
- < 1% authentication error rate
- InCommon metadata refreshes successfully daily

## Additional Notes

**Clerk Enterprise SAML Limitation**: Clerk's Enterprise SAML is designed for B2B SaaS where each customer organization brings their own IdP. It's not designed for multilateral federations where many IdPs should all be trusted by a single SP. This is the fundamental architectural reason SimpleSAMLphp or similar SAML broker is needed for true InCommon integration.

**InCommon Entity Categories**: When registering with InCommon, you'll be asked to select entity categories that describe your service. Common categories:
- `http://refeds.org/category/research-and-scholarship`: Research and scholarship services
- `http://id.incommon.org/category/registered-by-incommon`: InCommon registration authority

These affect what attributes IdPs will release to your SP.

**Attribute Release Variation**: Not all InCommon IdPs release the same attributes. Some IdPs have restrictive default policies. Your SP metadata should include `<md:RequestedAttribute>` elements to signal which attributes you need, but this doesn't guarantee release. Plan for graceful degradation when optional attributes are missing.

**Testing Resources**:
- InCommon Test Federation: https://spaces.at.internet2.edu/display/federation/Testing
- InCommon Test IdP: Available for testing SAML flows before production
- SimpleSAMLphp test interface: Built-in testing tools at `/simplesaml/module.php/core/authenticate.php`

**Monitoring Recommendations**:
- Track SAML authentication success/failure rates
- Monitor SAML response time (IdP → ACS callback)
- Alert on InCommon metadata refresh failures
- Alert on SimpleSAMLphp session store connectivity issues
- Track Clerk API error rates and latency
- Monitor organization creation (unexpected new organizations may indicate misconfiguration)

**Future Enhancements**:
- Admin UI for IdP-to-Organization mapping management
- Support for SAML attribute-based authorization (beyond authentication)
- Integration with Clerk's Team/Organization features for InCommon groups
- Advanced IdP discovery with predictive search
- SAML authentication analytics dashboard

## Next Steps

1. [ ] **Clarify Requirements**: Determine if true InCommon multilateral federation is required, or if bilateral SAML agreements (Clerk native) are acceptable

2. [ ] **Evaluate Alternatives**:
   - If < 10 organizations: Consider Option 1 (Clerk Native Enterprise SAML)
   - If multilateral federation required: Proceed with Option 2 (SimpleSAMLphp Bridge)

3. [ ] **Infrastructure Planning**:
   - [ ] Provision PHP hosting environment (if Option 2)
   - [ ] Set up Redis/Memcache for session storage
   - [ ] Obtain SSL certificates for SAML endpoints

4. [ ] **InCommon Membership**:
   - [ ] Verify organization's InCommon membership status
   - [ ] Identify InCommon representative/administrator
   - [ ] Plan entity ID and metadata registration

5. [ ] **Proof of Concept** (if Option 2):
   - [ ] Set up SimpleSAMLphp in development environment
   - [ ] Configure InCommon Test IdP
   - [ ] Implement basic ACS API route
   - [ ] Test end-to-end authentication flow

6. [ ] **Create Specification Document**: If moving forward with implementation, create detailed specification in `docs/spec/` with:
   - Detailed architecture diagrams
   - API endpoint specifications
   - Database schema for organization mapping
   - Security requirements checklist
   - Testing plan

7. [ ] **Team Review**: Present research findings to team for feedback and go/no-go decision

8. [ ] **Budget Approval**: Confirm budget for Clerk Pro + Enhanced Authentication Add-on and infrastructure costs

## Sources

1. SimpleSAMLphp Service Provider QuickStart - https://simplesamlphp.org/docs/stable/simplesamlphp-sp.html - Accessed 2025-10-21
2. InCommon Federation Software Guidelines - https://spaces.at.internet2.edu/display/federation/InCommon+Federation+Software+Guidelines - Accessed 2025-10-21
3. InCommon Metadata Query (MDQ) Service - https://spaces.at.internet2.edu/display/MDQ/how-to-configure-other-software-to-use-mdq - Accessed 2025-10-21
4. InCommon Federation Attribute Overview - https://incommon.org/federation/attributes/ - Accessed 2025-10-21
5. Clerk Enterprise SSO Documentation - https://clerk.com/docs/authentication/enterprise-connections/overview - Accessed 2025-10-21
6. Clerk Custom Authentication Flows - https://clerk.com/docs/custom-flows/overview - Accessed 2025-10-21
7. Clerk Session Tokens Documentation - https://clerk.com/docs/backend-requests/resources/session-tokens - Accessed 2025-10-21
8. Clerk Webhooks Overview - https://clerk.com/docs/webhooks/overview - Accessed 2025-10-21
9. Clerk JWT Templates - https://clerk.com/docs/backend-requests/jwt-templates - Accessed 2025-10-21
10. Bridging the OAuth2 / SAML2 Divide - https://optimalidm.com/resources/blog/bridging-oauth2-saml2-divide/ - Accessed 2025-10-21
11. How to convert SAML 2.0 assertions to OAuth 2.0 access tokens - https://connect2id.com/blog/how-to-bridge-saml-and-oauth - Accessed 2025-10-21
12. OWASP SAML Security Cheat Sheet - https://cheatsheetseries.owasp.org/cheatsheets/SAML_Security_Cheat_Sheet.html - Accessed 2025-10-21
13. InCommon Baseline Expectations for Trust in Federation - https://incommon.org/federation/baseline-expectations-for-trust-in-federation/ - Accessed 2025-10-21
14. node-saml/passport-saml GitHub Repository - https://github.com/node-saml/passport-saml - Accessed 2025-10-21
15. SimpleSAMLphp SP API Reference - https://simplesamlphp.readthedocs.io/en/stable/simplesamlphp-sp-api/ - Accessed 2025-10-21
16. Multi-Tenant SaaS and Single Sign-On - https://ssojet.com/blog/multi-tenant-saas-and-single-sign-on - Accessed 2025-10-21
17. Use SAML with Amazon Cognito for Multi-Tenant Applications - https://aws.amazon.com/blogs/security/use-saml-with-amazon-cognito-to-support-a-multi-tenant-application-with-a-single-user-pool/ - Accessed 2025-10-21

# SAML Authentication Guide

This document provides a comprehensive overview of SAML authentication, how it works, and how to implement it with Clerk in this application.

## Table of Contents

1. [What is SAML Authentication?](#what-is-saml-authentication)
2. [How SAML Authentication Flow Works](#how-saml-authentication-flow-works)
3. [What You Need to Establish a SAML Connection](#what-you-need-to-establish-a-saml-connection)
4. [Clerk SAML Implementation](#clerk-saml-implementation)
5. [Implementation Plan](#implementation-plan)

---

## What is SAML Authentication?

**SAML (Security Assertion Markup Language)** is an XML-based protocol for exchanging authentication and authorization data between parties. It enables **Single Sign-On (SSO)**, allowing users to log in once and access multiple applications without re-entering credentials.

### Key Players in SAML

1. **Identity Provider (IdP)**: The system that authenticates users and stores their credentials
   - Examples: Okta, OneLogin, Azure AD, Google Workspace, Shibboleth, InCommon
   - Manages user identities and credentials
   - Generates SAML assertions after successful authentication

2. **Service Provider (SP)**: Your application that relies on the IdP for authentication
   - In this case, **Clerk acts as the Service Provider**
   - Receives and validates SAML assertions
   - Creates user sessions based on validated assertions

3. **User/Principal**: The person trying to authenticate
   - Typically a student, faculty member, or employee
   - Has credentials stored in the IdP

---

## How SAML Authentication Flow Works

### High-Level Overview

```
┌─────────┐                    ┌──────────────┐                    ┌─────────────┐
│  User   │                    │  Your App    │                    │     IdP     │
│ (Browser)│                   │  (Clerk SP)  │                    │  (Okta)     │
└────┬────┘                    └──────┬───────┘                    └──────┬──────┘
     │                                │                                   │
     │ 1. Visit app.acme.com          │                                   │
     ├───────────────────────────────>│                                   │
     │                                │                                   │
     │ 2. Enter email                 │                                   │
     │    john@university.edu         │                                   │
     ├───────────────────────────────>│                                   │
     │                                │                                   │
     │                                │ 3. Domain recognized              │
     │                                │    Initiate SAML flow             │
     │                                │                                   │
     │ 4. Redirect to IdP             │                                   │
     │<───────────────────────────────┤                                   │
     │                                │                                   │
     │ 5. Navigate to IdP login       │                                   │
     ├───────────────────────────────────────────────────────────────────>│
     │                                │                                   │
     │                                │ 6. User enters credentials        │
     │                                │    IdP validates                  │
     │                                │                                   │
     │ 7. SAML Response (POST)        │                                   │
     │<───────────────────────────────────────────────────────────────────┤
     │    with signed assertion       │                                   │
     │                                │                                   │
     │ 8. POST to ACS URL             │                                   │
     ├───────────────────────────────>│                                   │
     │                                │                                   │
     │                                │ 9. Validate signature             │
     │                                │    Extract user attributes        │
     │                                │    Create/update user             │
     │                                │    Create session                 │
     │                                │                                   │
     │ 10. Redirect with session      │                                   │
     │<───────────────────────────────┤                                   │
     │                                │                                   │
     │ 11. User is authenticated!     │                                   │
     │                                │                                   │
```

### Detailed Step-by-Step Flow

#### Step 1: User Initiates Login (Service Provider Initiated)

1. User visits your application (e.g., `app.acme.com`)
2. User enters email address in the sign-in form (e.g., `john@university.edu`)
3. Your app/Clerk detects that the email domain (`university.edu`) matches a configured SAML connection
4. Clerk generates a **SAML Authentication Request** (XML document)
5. User's browser is redirected to the IdP's SSO URL with this request

**Example SAML Authentication Request:**
```xml
<samlp:AuthnRequest
    xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
    ID="REQUEST_ID"
    Version="2.0"
    IssueInstant="2025-01-15T10:00:00Z"
    Destination="https://idp.university.edu/sso"
    AssertionConsumerServiceURL="https://clerk.accounts.dev/v1/saml/acs/samlc_123">
  <saml:Issuer>https://clerk.accounts.dev/saml/samlc_123</saml:Issuer>
</samlp:AuthnRequest>
```

#### Step 2: User Authenticates with Identity Provider

1. User arrives at the IdP's login page (e.g., `idp.university.edu/login`)
2. User enters their credentials (username/password)
3. IdP may require additional authentication (MFA, security questions, etc.)
4. IdP validates the credentials against its user directory

#### Step 3: IdP Generates SAML Assertion

Upon successful authentication, the IdP creates a **SAML Response** containing:

**Assertion Components:**
- **Subject**: Who the assertion is about (the user)
- **Conditions**: Validity timeframe, audience restrictions
- **Attributes**: User information (email, name, ID, custom attributes)
- **Signature**: Cryptographic signature using the IdP's private key

**Example SAML Assertion (simplified):**
```xml
<saml:Assertion ID="ASSERTION_ID" Version="2.0">
  <saml:Issuer>https://idp.university.edu</saml:Issuer>
  <ds:Signature>
    <!-- Digital signature for validation -->
  </ds:Signature>
  <saml:Subject>
    <saml:NameID>john.doe@university.edu</saml:NameID>
  </saml:Subject>
  <saml:Conditions NotBefore="2025-01-15T10:00:00Z" NotOnOrAfter="2025-01-15T10:05:00Z">
    <saml:AudienceRestriction>
      <saml:Audience>https://clerk.accounts.dev/saml/samlc_123</saml:Audience>
    </saml:AudienceRestriction>
  </saml:Conditions>
  <saml:AttributeStatement>
    <saml:Attribute Name="email">
      <saml:AttributeValue>john.doe@university.edu</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="firstName">
      <saml:AttributeValue>John</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="lastName">
      <saml:AttributeValue>Doe</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="id">
      <saml:AttributeValue>123456789</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

#### Step 4: IdP Redirects Back to Service Provider

1. IdP creates an HTML form with the SAML Response
2. The response is Base64-encoded
3. Browser automatically POSTs this form to the **ACS URL** (Assertion Consumer Service URL)

**Example POST:**
```html
<form method="POST" action="https://clerk.accounts.dev/v1/saml/acs/samlc_123">
  <input type="hidden" name="SAMLResponse" value="BASE64_ENCODED_SAML_RESPONSE" />
  <input type="hidden" name="RelayState" value="ORIGINAL_DESTINATION" />
</form>
```

#### Step 5: Service Provider Validates Assertion

Clerk receives the SAML Response and performs validation:

1. **Signature Validation**:
   - Extracts the signature from the assertion
   - Uses the IdP's public certificate (X.509) to verify the signature
   - Ensures the assertion hasn't been tampered with

2. **Condition Checks**:
   - Verifies the assertion hasn't expired (`NotOnOrAfter`)
   - Confirms the assertion is for this application (`Audience`)
   - Checks that the assertion isn't being replayed

3. **Attribute Extraction**:
   - Uses configured attribute mappings to extract user data
   - Maps IdP attributes to Clerk user fields:
     - `email` → `emailAddress`
     - `firstName` → `firstName`
     - `lastName` → `lastName`
     - `id` → `userId`

#### Step 6: User Session Created

1. Clerk creates or updates the user account with the extracted attributes
2. A new session is created for the user
3. User is redirected to the application (or the original destination via RelayState)
4. User is now authenticated and can access the application

---

## What You Need to Establish a SAML Connection

### Information Required from the Identity Provider (IdP)

When configuring a SAML connection, you need the following information from your IdP:

#### Option 1: IdP Metadata URL (Easiest)

- **IdP Metadata URL**: A URL that serves an XML file containing all the necessary configuration
- Example: `https://trial-000000.okta.com/app/exkabc123/sso/saml/metadata`
- **Advantage**: Clerk can automatically fetch and parse this to extract all required fields

#### Option 2: Manual Configuration (If metadata URL isn't available)

1. **IdP Entity ID** (Required)
   - Unique identifier for the Identity Provider
   - Format: Usually a URI
   - Example: `http://www.okta.com/exkabc123`
   - Where to find: IdP's SAML configuration page

2. **IdP SSO URL** (Required)
   - The Single Sign-On endpoint where authentication requests are sent
   - Format: HTTPS URL
   - Example: `https://trial-000000.okta.com/app/exkabc123/sso/saml`
   - Where to find: IdP's SAML configuration page

3. **IdP Certificate** (Required)
   - X.509 public certificate used to verify SAML assertion signatures
   - Format: PEM format, including `-----BEGIN CERTIFICATE-----` and `-----END CERTIFICATE-----`
   - Example:
     ```
     -----BEGIN CERTIFICATE-----
     MIIDuDCCAqCgAwIBAgIJAKz...
     ...
     -----END CERTIFICATE-----
     ```
   - Where to find: IdP provides this for download or copy-paste

#### Option 3: IdP Metadata XML (Alternative to URL)

- Raw XML content containing all the above information
- Can be uploaded or pasted directly
- Useful when metadata URL isn't publicly accessible

### Information to Provide to the Identity Provider (IdP)

The IdP administrator needs these details from Clerk to configure their side:

1. **ACS URL** (Assertion Consumer Service URL)
   - Where the IdP should POST SAML responses
   - Format: `https://{clerk-instance}.clerk.accounts.dev/v1/saml/acs/{connection-id}`
   - Example: `https://prompt-gazelle-54.clerk.accounts.dev/v1/saml/acs/samlc_349i3g0fra443V8ZCBYnFAzstDQ`
   - **Note**: This is generated by Clerk after creating the connection

2. **SP Entity ID** (Service Provider Entity ID)
   - Unique identifier for your application in Clerk
   - Format: `https://{clerk-instance}.clerk.accounts.dev/saml/{connection-id}`
   - Example: `https://prompt-gazelle-54.clerk.accounts.dev/saml/samlc_349i3g0fra443V8ZCBYnFAzstDQ`
   - **Note**: This is generated by Clerk after creating the connection

3. **SP Metadata URL** (Optional but recommended)
   - XML metadata file describing your Service Provider configuration
   - Format: `https://{clerk-instance}.clerk.accounts.dev/v1/saml/metadata/{connection-id}.xml`
   - Example: `https://prompt-gazelle-54.clerk.accounts.dev/v1/saml/metadata/samlc_349i3g0fra443V8ZCBYnFAzstDQ.xml`
   - **Advantage**: IdP can auto-configure using this URL

### Additional Configuration Details

#### Connection Settings

1. **Connection Name**
   - A friendly label for this SAML connection
   - Example: "University SSO", "Acme Corp InCommon"
   - Used for internal identification

2. **Domain(s)**
   - Email domain(s) that will use this SAML connection
   - Example: `university.edu`, `acme.com`
   - When a user enters an email with this domain, the SAML flow is triggered
   - Can specify multiple domains

3. **Provider Type**
   - The type of IdP being used
   - Options:
     - `saml_custom`: Generic SAML provider
     - `saml_okta`: Okta-specific configuration
     - `saml_google`: Google Workspace
     - `saml_microsoft`: Azure AD / Microsoft Entra ID

4. **Organization ID** (Optional)
   - Link this SAML connection to a specific Clerk organization
   - Users authenticating via this connection will be added to this organization

#### Attribute Mapping

Map IdP SAML attributes to Clerk user properties:

| Clerk Field | Description | Common IdP Attribute Names |
|-------------|-------------|----------------------------|
| User ID | Unique identifier for the user | `id`, `uid`, `nameID`, `userName` |
| Email Address | User's email address | `email`, `mail`, `emailAddress` |
| First Name | User's first name | `firstName`, `givenName`, `given_name` |
| Last Name | User's last name | `lastName`, `surname`, `family_name`, `sn` |

**Example Configuration:**
```json
{
  "attributeMapping": {
    "userId": "id",
    "emailAddress": "email",
    "firstName": "firstName",
    "lastName": "lastName"
  }
}
```

#### Advanced Settings

1. **Sync user attributes during Sign in**
   - When enabled: User attributes are updated from the IdP on every login
   - When disabled: Attributes are only set during initial account creation
   - **Use case**: Keep user data in sync if it changes in the IdP

2. **Allow subdomains**
   - When enabled: Users with subdomain emails can authenticate
   - Example: If domain is `university.edu`, allow `john@engineering.university.edu`
   - **Use case**: Organizations with multiple subdomains

3. **Allow IdP-Initiated flow**
   - When enabled: Users can start the login from the IdP portal
   - When disabled: Only SP-initiated flows are allowed (user starts from your app)
   - **Security note**: Some organizations disable this for security reasons

4. **Allow additional identifiers**
   - Allows users to link additional email addresses or accounts to their profile
   - **Use case**: Users with multiple email addresses

5. **Force authentication**
   - Requires users to re-authenticate with the IdP even if they have an active session
   - **Use case**: High-security applications requiring fresh authentication

---

## Clerk SAML Implementation

### Creating a SAML Connection via Clerk Backend API

#### Using IdP Metadata URL (Recommended)

```typescript
import { clerkClient } from '@clerk/nextjs/server'

const client = await clerkClient()

const connection = await client.samlConnections.createSamlConnection({
  name: 'University SSO',
  provider: 'saml_custom',
  domain: 'university.edu',
  idpMetadataUrl: 'https://idp.university.edu/metadata.xml',
  attributeMapping: {
    userId: 'id',
    emailAddress: 'email',
    firstName: 'firstName',
    lastName: 'lastName',
  },
  active: true,
  syncUserAttributes: true,
  allowSubdomains: false,
  allowIdpInitiated: false,
})

// Response includes:
// - connection.id (e.g., "samlc_123")
// - connection.acsUrl (provide to IdP)
// - connection.entityId (provide to IdP)
// - connection.metadataUrl (provide to IdP)
```

#### Using Manual Configuration

```typescript
const connection = await client.samlConnections.createSamlConnection({
  name: 'University SSO',
  provider: 'saml_custom',
  domain: 'university.edu',
  idpEntityId: 'https://idp.university.edu',
  idpSsoUrl: 'https://idp.university.edu/sso/saml',
  idpCertificate: `-----BEGIN CERTIFICATE-----
MIIDuDCCAqCgAwIBAgIJAKz...
-----END CERTIFICATE-----`,
  attributeMapping: {
    userId: 'id',
    emailAddress: 'email',
    firstName: 'firstName',
    lastName: 'lastName',
  },
  active: true,
})
```

### Updating a SAML Connection

```typescript
const updated = await client.samlConnections.updateSamlConnection(
  'samlc_123',
  {
    name: 'Updated Connection Name',
    active: true,
    syncUserAttributes: true,
    allowSubdomains: true,
  }
)
```

### Retrieving a SAML Connection

```typescript
const connection = await client.samlConnections.getSamlConnection('samlc_123')
```

### Deleting a SAML Connection

```typescript
await client.samlConnections.deleteSamlConnection('samlc_123')
```

### Error Handling

Common errors when working with SAML connections:

1. **`saml_failed_to_fetch_idp_metadata`** (400)
   - Unable to fetch metadata from the provided URL
   - Solution: Verify URL is accessible, or use manual configuration

2. **`saml_failed_to_parse_idp_metadata`** (422)
   - Metadata XML is malformed or invalid
   - Solution: Validate XML structure, or use manual configuration

3. **`saml_connection_cant_be_activated`** (422)
   - Missing required fields (Entity ID, SSO URL, or Certificate)
   - Solution: Provide all required IdP configuration details

4. **`saml_email_address_domain_reserved`** (422)
   - Domain is already used by another SAML connection
   - Solution: Use a different domain or update the existing connection

5. **`saml_email_address_domain_mismatch`** (400)
   - User's email domain doesn't match the connection's configured domain
   - Solution: Verify domain configuration or user email

---

## Implementation Plan

### Phase 1: Type Definitions & Data Structures

#### 1.1 Create SAML Type Definitions

**File**: `src/lib/types/saml.ts`

```typescript
export type SAMLProvider = 'saml_custom' | 'saml_okta' | 'saml_google' | 'saml_microsoft'

export type SAMLAttributeMapping = {
  userId?: string
  emailAddress?: string
  firstName?: string
  lastName?: string
}

export type SAMLConnection = {
  id: string
  name: string
  provider: SAMLProvider
  domain: string
  organizationId?: string

  // IdP Configuration
  idpEntityId?: string
  idpSsoUrl?: string
  idpCertificate?: string
  idpMetadataUrl?: string
  idpMetadata?: string

  // SP Configuration (read-only from Clerk)
  acsUrl: string
  entityId: string
  metadataUrl: string

  // Attribute Mapping
  attributeMapping: SAMLAttributeMapping

  // Settings
  active: boolean
  syncUserAttributes: boolean
  allowSubdomains: boolean
  allowIdpInitiated: boolean

  // Metadata
  status: 'active' | 'inactive' | 'pending'
  createdAt: number
  updatedAt: number
}

export type CreateSAMLConnectionRequest = {
  name: string
  provider: SAMLProvider
  domain: string
  organizationId?: string
  idpEntityId?: string
  idpSsoUrl?: string
  idpCertificate?: string
  idpMetadataUrl?: string
  idpMetadata?: string
  attributeMapping?: SAMLAttributeMapping
  active?: boolean
  syncUserAttributes?: boolean
  allowSubdomains?: boolean
  allowIdpInitiated?: boolean
}

export type UpdateSAMLConnectionRequest = Partial<CreateSAMLConnectionRequest>
```

#### 1.2 Update Client Type

**File**: `src/lib/types/client.ts`

Add SAML connections to the Client type:

```typescript
export type Client = {
  // ... existing fields
  samlConnections?: SAMLConnection[]
}
```

### Phase 2: Backend API Routes (TDD Required)

#### 2.1 Create SAML Connection API

**Test File**: `src/app/api/saml-connections/route.spec.ts` (WRITE FIRST)

**Implementation File**: `src/app/api/saml-connections/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { clerkClient } from '@clerk/nextjs/server'
import type { ApiResponse } from '@/lib/types/api'
import type { SAMLConnection, CreateSAMLConnectionRequest } from '@/lib/types/saml'

export async function POST(request: NextRequest): Promise<NextResponse<ApiResponse<SAMLConnection>>> {
  try {
    const body: CreateSAMLConnectionRequest = await request.json()

    // Validate required fields
    if (!body.name || !body.provider || !body.domain) {
      return NextResponse.json(
        { success: false, error: 'Missing required fields: name, provider, domain' },
        { status: 400 }
      )
    }

    const client = await clerkClient()
    const connection = await client.samlConnections.createSamlConnection(body)

    return NextResponse.json({ success: true, data: connection })
  } catch (error) {
    console.error('Error creating SAML connection:', error)
    return NextResponse.json(
      { success: false, error: 'Failed to create SAML connection' },
      { status: 500 }
    )
  }
}
```

#### 2.2 Get/Update/Delete SAML Connection API

**Test File**: `src/app/api/saml-connections/[id]/route.spec.ts` (WRITE FIRST)

**Implementation File**: `src/app/api/saml-connections/[id]/route.ts`

```typescript
import { NextRequest, NextResponse } from 'next/server'
import { clerkClient } from '@clerk/nextjs/server'

export async function GET(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const client = await clerkClient()
    const connection = await client.samlConnections.getSamlConnection(params.id)
    return NextResponse.json({ success: true, data: connection })
  } catch (error) {
    return NextResponse.json(
      { success: false, error: 'SAML connection not found' },
      { status: 404 }
    )
  }
}

export async function PATCH(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const body = await request.json()
    const client = await clerkClient()
    const connection = await client.samlConnections.updateSamlConnection(params.id, body)
    return NextResponse.json({ success: true, data: connection })
  } catch (error) {
    return NextResponse.json(
      { success: false, error: 'Failed to update SAML connection' },
      { status: 500 }
    )
  }
}

export async function DELETE(
  request: NextRequest,
  { params }: { params: { id: string } }
) {
  try {
    const client = await clerkClient()
    await client.samlConnections.deleteSamlConnection(params.id)
    return NextResponse.json({ success: true, data: null })
  } catch (error) {
    return NextResponse.json(
      { success: false, error: 'Failed to delete SAML connection' },
      { status: 500 }
    )
  }
}
```

### Phase 3: UI Components (TDD Required)

#### 3.1 SAML Connection Form Component

**Test File**: `src/components/saml-connection-form.spec.tsx` (WRITE FIRST)

**Implementation File**: `src/components/saml-connection-form.tsx`

The form should match the Clerk UI with three main sections:

1. **Basic Configuration**
   - Connection name input
   - Domain input(s) with ability to add multiple
   - Provider selection (Custom, Okta, Google, Microsoft)
   - Optional organization selector

2. **Identity Provider Configuration**
   - Choice between "Metadata URL" or "Manual Configuration"
   - If Metadata URL: Single input field
   - If Manual:
     - SSO URL input
     - Entity ID input
     - Certificate textarea/upload

3. **Attribute Mapping**
   - User ID mapping
   - Email address mapping
   - First name mapping
   - Last name mapping

4. **Advanced Settings**
   - Sync user attributes (toggle)
   - Allow subdomains (toggle)
   - Allow IdP-initiated flow (toggle)
   - Allow additional identifiers (toggle)
   - Force authentication (toggle)

#### 3.2 SAML Configuration Section

**Test File**: `src/components/sections/saml-config-section.spec.tsx` (WRITE FIRST)

**Implementation File**: `src/components/sections/saml-config-section.tsx`

This section displays:
- List of existing SAML connections
- Connection status indicators
- Add new connection button
- Edit/Delete actions for each connection
- Service Provider details (ACS URL, Entity ID, Metadata URL) with copy buttons

#### 3.3 Service Provider Details Component

Display Clerk's SP details in a read-only format with copy-to-clipboard functionality:

```tsx
<div className="space-y-4">
  <h3>Service Provider Configuration</h3>
  <p className="text-sm text-muted-foreground">
    Provide these values to your Identity Provider administrator
  </p>

  <div className="space-y-2">
    <Label>Assertion Consumer Service (ACS) URL</Label>
    <div className="flex gap-2">
      <Input value={connection.acsUrl} readOnly />
      <Button onClick={() => copyToClipboard(connection.acsUrl)}>
        <CopyIcon />
      </Button>
    </div>
  </div>

  <div className="space-y-2">
    <Label>Entity ID</Label>
    <div className="flex gap-2">
      <Input value={connection.entityId} readOnly />
      <Button onClick={() => copyToClipboard(connection.entityId)}>
        <CopyIcon />
      </Button>
    </div>
  </div>

  <div className="space-y-2">
    <Label>Metadata URL</Label>
    <div className="flex gap-2">
      <Input value={connection.metadataUrl} readOnly />
      <Button onClick={() => copyToClipboard(connection.metadataUrl)}>
        <CopyIcon />
      </Button>
    </div>
  </div>
</div>
```

### Phase 4: Integration with Client Settings

Update the client settings navigation to include SAML configuration:

**File**: `src/components/client-settings-form.tsx`

```typescript
const sections = [
  // ... existing sections
  {
    id: 'saml',
    label: 'SSO Configuration',
    icon: ShieldIcon,
    component: SAMLConfigSection,
  },
]
```

### Phase 5: Testing Requirements

**All tests MUST be written BEFORE implementation (TDD)**

1. **Unit Tests**:
   - API route handlers (success, validation errors, server errors)
   - Form validation logic
   - Attribute mapping utilities
   - Helper functions

2. **Component Tests**:
   - SAML connection form rendering
   - Form submission with valid data
   - Form submission with invalid data
   - Error state handling
   - Loading state handling

3. **Integration Tests**:
   - Create → Read → Update → Delete flow
   - Multiple domain handling
   - Metadata URL vs. manual configuration

4. **Coverage Requirements**:
   - Minimum 60% overall (enforced)
   - Target 80%+ for new SAML code
   - 100% for critical paths

### Phase 6: Implementation Workflow

**MANDATORY ORDER** (Following TDD):

1. Write type definitions (no tests needed)
2. Write API route tests → Implement routes → Run tests
3. Write service tests (if needed) → Implement services → Run tests
4. Write component tests → Implement components → Run tests
5. Run `pnpm run lint` after EVERY code change
6. Run `pnpm run test:ci` before committing
7. Ensure coverage requirements are met

---

## Common SAML Providers

### Okta

- Provider: `saml_okta`
- Default attribute names: `user.email`, `user.firstName`, `user.lastName`
- Metadata URL format: `https://{domain}.okta.com/app/{app-id}/sso/saml/metadata`

### Google Workspace

- Provider: `saml_google`
- Default attribute names: `email`, `given_name`, `family_name`
- Requires Google Workspace admin configuration

### Microsoft Azure AD / Entra ID

- Provider: `saml_microsoft`
- Default attribute names: `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress`, `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname`, `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname`

### InCommon (Universities)

- Provider: `saml_custom`
- Attribute names vary by institution
- Common: `eduPersonPrincipalName`, `mail`, `givenName`, `sn`

---

## Troubleshooting

### Common Issues

1. **"Failed to fetch IdP metadata"**
   - Check that the metadata URL is publicly accessible
   - Verify the URL doesn't require authentication
   - Try using manual configuration instead

2. **"Failed to parse IdP metadata"**
   - Validate the XML structure
   - Ensure the metadata follows SAML 2.0 specification
   - Use manual configuration as fallback

3. **"SAML connection can't be activated"**
   - Ensure all required fields are provided (Entity ID, SSO URL, Certificate)
   - Verify certificate format is correct (PEM with headers)

4. **"Email address domain mismatch"**
   - User's email domain doesn't match configured domain
   - Check domain configuration in SAML connection
   - Verify `allowSubdomains` setting if using subdomains

5. **Signature validation fails**
   - Certificate doesn't match the one used by IdP
   - Certificate has expired
   - IdP rotated certificates - update the connection

---

## Security Considerations

1. **Certificate Management**
   - Store IdP certificates securely
   - Monitor certificate expiration dates
   - Have a process for certificate rotation

2. **Assertion Validation**
   - Always validate signatures
   - Check assertion expiration times
   - Verify audience restrictions
   - Prevent replay attacks

3. **IdP-Initiated Flows**
   - Consider disabling if not needed (security best practice)
   - Can be vulnerable to unsolicited response attacks
   - Enable only if users need to start from IdP portal

4. **Force Authentication**
   - Enable for sensitive applications
   - Ensures fresh authentication even with active IdP session
   - Prevents session riding attacks

---

## Resources

- [SAML 2.0 Specification](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf)
- [Clerk SAML Documentation](https://clerk.com/docs/authentication/saml)
- [Clerk Backend API Reference](https://clerk.com/docs/reference/backend-api)
- [SAML Security Best Practices](https://docs.oasis-open.org/security/saml/v2.0/saml-sec-consider-2.0-os.pdf)

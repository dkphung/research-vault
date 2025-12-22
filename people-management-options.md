---
tags: [architecture]
date: 2024-12-22
status: complete
---

# People Management Options Research

**Date**: 2025-10-18

## Context

The application uses Clerk Organizations to represent "clients" (universities/institutions). We need to implement comprehensive people/member management for these organizations, replicating the functionality shown in Clerk's Dashboard.

The user has provided screenshots showing Clerk's organization member management UI with three approaches:
1. **Existing User** - Add a user who already exists in Clerk
2. **Create User** - Create a new user account directly
3. **Invite User** - Send an email invitation

Additionally, a bonus feature is requested: **Bulk CSV Upload** to invite multiple users at once.

## Current State

### Existing Code
- **Architecture**: Clients are Clerk Organizations (see `src/server/client/client.actions.ts`)
- **People Types**: Defined in `src/lib/types/person.ts`
- **UI Component**: `src/app/client/_components/sections/people-management-section.tsx`
- **Clerk Service**: `src/server/services/clerk.service.ts` (app-level invitations, not organization-specific)
- **Mock Data**: `src/server/data/mock-people.ts`

### Gaps
- No organization-specific member management
- No bulk invitation capability
- No UI for the three add-user approaches (existing/create/invite)
- API routes exist (`/api/people`) but use mock data

## Research Findings

### 1. Add Existing User to Organization

**Clerk API**: `POST /organizations/{organization_id}/memberships`

**Method**: `clerkClient.organizations.createOrganizationMembership()`

**Use Case**: Add a user who already has a Clerk account to an organization

**Key Parameters**:
- `organizationId` (string, required)
- `userId` (string, required)
- `role` (string, required) - e.g., "org:admin", "org:member"

**Advantages**:
- Immediate access (no invitation email needed)
- User already exists in system
- Direct membership creation

**Limitations**:
- Requires knowing the user's Clerk user ID
- User must already exist in the Clerk instance
- Cannot be used for new users

**Implementation Notes**:
- Need a user search/lookup UI to find existing users
- Could search by email using `clerkClient.users.getUserList({ emailAddress: [...] })`
- Assign role during addition

### 2. Create User Directly

**Clerk API**: `POST /users`

**Method**: `clerkClient.users.createUser()`

**Use Case**: Create a new Clerk user account and optionally add to organization

**Key Parameters**:
- `emailAddress` (array of strings, required)
- `firstName` (string, optional)
- `lastName` (string, optional)
- `password` (string, optional)
- `publicMetadata` (object, optional)

**After Creation**: Use the created user's ID with `createOrganizationMembership()` to add to organization

**Advantages**:
- Full control over user creation
- Can set initial password
- Can set metadata immediately
- User can log in right away

**Limitations**:
- Requires password management
- Password policies must be respected
- Need to handle password delivery securely
- Email verification considerations

**Implementation Notes**:
- Already have `createClerkUser()` in `clerk.service.ts` but it's app-level
- Need two-step process: create user, then add to organization
- Consider option to "Ignore password policies" checkbox (shown in screenshot)
- Should validate email doesn't already exist

### 3. Invite User

**Clerk API**: `POST /organizations/{organization_id}/invitations`

**Method**: `clerkClient.organizations.createOrganizationInvitation()`

**Use Case**: Send an email invitation to join the organization

**Key Parameters**:
- `organizationId` (string, required)
- `inviterUserId` (string, required)
- `emailAddress` (string, required)
- `role` (string, required)
- `redirectUrl` (string, optional) - Where to redirect after accepting
- `publicMetadata` (object, optional) - Transferred to membership on acceptance

**Invitation Flow**:
1. Admin creates invitation
2. Email sent to recipient with unique link
3. Recipient clicks link (may need to sign up first)
4. Recipient accepts invitation
5. User becomes organization member

**Advantages**:
- Email-based workflow
- User creates their own password
- Email auto-verified on acceptance
- Can include custom metadata
- Can set redirect URL after acceptance

**Limitations**:
- Requires email delivery
- User must complete signup flow
- Delay until user accepts
- Invitation can expire (configurable: 30 days default in screenshot)

**Revocation**:
- `clerkClient.organizations.revokeOrganizationInvitation()`
- Prevents invitation link from being used

**Implementation Notes**:
- Track invitation status (pending/accepted/revoked)
- Allow resending invitations
- Allow revoking pending invitations
- Set invitation expiry (screenshot shows "30 Days")

### 4. Bulk Invitation (Bonus Feature)

**Clerk API**: `POST /organizations/{organization_id}/invitations/bulk`

**Method**: `clerkClient.organizations.createOrganizationInvitations()` (note: plural)

**Use Case**: Invite multiple users at once from a CSV file

**Key Parameters**:
- `organizationId` (string, required)
- Array of invitation objects, each with:
  - `inviterUserId` (string, required)
  - `emailAddress` (string, required)
  - `role` (string, required)
  - `redirectUrl` (string, optional)
  - `publicMetadata` (object, optional)

**CSV Format** (proposed):
```csv
firstName,lastName,email,role
John,Doe,john.doe@example.com,org:member
Jane,Smith,jane.smith@example.com,org:admin
```

**Implementation Approach**:
1. Upload CSV file
2. Parse CSV on server
3. Validate all rows (email format, required fields)
4. Preview parsed data to user
5. Confirm and send bulk invitations
6. Show results (success/failure per row)

**Advantages**:
- Efficient for onboarding many users
- Single API call for multiple invitations
- Clerk handles email delivery

**Limitations**:
- Need CSV parsing library
- Need validation for each row
- Error handling per invitation
- File size limits

**Implementation Notes**:
- Use a CSV parsing library (e.g., `papaparse`, `csv-parse`)
- Validate CSV structure and data
- Show preview table before confirming
- Handle partial failures gracefully
- Provide downloadable results report

## Options Comparison

| Feature | Existing User | Create User | Invite User | Bulk Invite |
|---------|--------------|-------------|-------------|-------------|
| User must exist | Yes | No | No | No |
| Immediate access | Yes | Yes | No (pending) | No (pending) |
| Email sent | No | Optional | Yes | Yes |
| Password required | N/A | Yes | No (user sets) | No (user sets) |
| Email verified | N/A | No | Yes (auto) | Yes (auto) |
| Metadata support | Yes | Yes | Yes | Yes |
| Best for | Adding colleagues | Quick access | Standard onboarding | Mass onboarding |

## Roles

Clerk organizations support custom roles. Common roles:
- `org:admin` - Full organization admin
- `org:member` - Regular member

Custom roles can be defined in Clerk Dashboard. The app should:
1. Fetch available roles from organization
2. Present as dropdown in UI
3. Default to `org:member`

## API Endpoints to Create

### Organization Members
- `GET /api/organizations/{orgId}/members` - List members
- `POST /api/organizations/{orgId}/members` - Add existing user
- `PATCH /api/organizations/{orgId}/members/{userId}` - Update member role
- `DELETE /api/organizations/{orgId}/members/{userId}` - Remove member

### Organization Users (Create)
- `POST /api/organizations/{orgId}/users` - Create user and add to org

### Organization Invitations
- `GET /api/organizations/{orgId}/invitations` - List pending invitations
- `POST /api/organizations/{orgId}/invitations` - Create single invitation
- `POST /api/organizations/{orgId}/invitations/bulk` - Create bulk invitations
- `POST /api/organizations/{orgId}/invitations/{invId}/revoke` - Revoke invitation
- `POST /api/organizations/{orgId}/invitations/{invId}/resend` - Resend invitation email

### CSV Upload
- `POST /api/organizations/{orgId}/invitations/csv` - Upload and preview CSV
- `POST /api/organizations/{orgId}/invitations/csv/confirm` - Confirm and send bulk

## Recommendations

### Approach
1. **Implement all three add-user methods** (existing/create/invite) as shown in Clerk's UI
2. **Prioritize invite flow** as it's the most common use case
3. **Add bulk CSV upload** as a power-user feature
4. **Use tabbed interface** like Clerk's screenshots show

### Technical Decisions
1. **Create organization-specific services** in `src/server/services/organization.service.ts`
2. **Separate API routes** under `/api/organizations/{orgId}/...`
3. **Use Zod schemas** for validation (CSV rows, invitation params)
4. **Implement preview step** for bulk operations
5. **Add comprehensive error handling** per invitation
6. **Store invitation metadata** linking to internal person records

### Libraries Needed
- **CSV Parsing**: `papaparse` (MIT license, 39KB, widely used)
  - Alternative: `csv-parse` from csv project (similar features, slightly larger)
  - Recommendation: `papaparse` for simplicity and TypeScript support

### Security Considerations
1. **Verify organization access** before any member operations
2. **Check user permissions** (only org admins can invite)
3. **Rate limit** bulk operations
4. **Validate email addresses** before sending invitations
5. **Sanitize CSV input** to prevent injection attacks
6. **Log all member changes** for audit trail

## Next Steps

1. Create detailed specification document
2. Design API routes and request/response types
3. Design UI components and flows
4. Plan testing strategy
5. Implement in phases (invite → create → existing → bulk)

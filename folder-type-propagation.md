---
tags: [architecture]
date: 2024-12-22
status: complete
---

# Folder Type Propagation Research

**Date**: 2024-12-18
**Status**: Complete
**Author**: Brainstorming session

## Context

Folder Types serve as blueprints when creating folders. At folder creation time, we instantiate:
- Roles based on the folder type's role templates
- Permissions based on those role templates
- Custom properties defined by the folder type's property schema

The core challenge: **What happens when a folder type's blueprint changes after folders have been created from it?**

## Current Architecture

### FolderType Structure
- `id` - UUID identifier
- `scopeFolderId` - where type is defined (inherits down to descendants)
- `key` - unique identifier within scope
- `roleTemplates` - array of role definitions (key, label, permissions)
- `propertySchema` - JSON Schema (draft-07) for folder properties
- `uiSchema` - RJSF UI Schema for form rendering

### Folder Structure
- `folderTypeKey` - references a FolderType.key
- `properties` - JSON data validated against folderType.propertySchema

### Current Relationship
- Loose binding: folders reference folder types by key
- No version tracking
- No distinction between "inherited from type" vs "added locally"
- Property validation happens on create/update against current schema

---

## Problem 1: Propagating Blueprint Changes

### Option A: Snapshot at Creation (Current Implied Model)

When a folder is created, it gets a snapshot of the folder type's state. Changes to the folder type don't affect existing folders.

| Pros | Cons |
|------|------|
| Simple to implement | No consistency - folders drift from type |
| No cascade failures | Hard to answer "which folders use role X?" |
| Predictable behavior | Updates require touching each folder |

### Option B: Live Binding with Versioning

Add a `folderTypeVersion` field to folders. Folder types are immutable - updates create new versions.

```graphql
type FolderType {
  id: ID!
  key: String!
  version: Int!  # incrementing version
  # ... rest of fields
}

type Folder {
  folderTypeKey: String!
  folderTypeVersion: Int!  # which version folder was created with
}
```

**Migration Strategy Options:**
1. **Manual migration**: Admin triggers "upgrade folders to latest version"
2. **Lazy migration**: On next folder access, prompt/auto-upgrade
3. **Gradual rollout**: Background job migrates in batches

| Pros | Cons |
|------|------|
| Full audit trail | Storage overhead for versions |
| Can rollback if needed | Complexity in resolving "current" type |
| Explicit upgrade path | Need migration tooling |

### Option C: Live Binding with Change Types

Folder types always resolve to latest, but changes are categorized:

```typescript
type ChangeType =
  | 'additive'      // new optional property, new role - auto-applies
  | 'breaking'      // new required property, removed role - needs migration
  | 'permission'    // permission changes - immediate effect
```

**Breaking changes** create a "pending migration" state on affected folders until resolved.

| Pros | Cons |
|------|------|
| Smart about what needs action | Complex change detection |
| Minimal friction for safe changes | Edge cases in classification |

### Option D: Inheritance with Override Tracking

Folders inherit from folder types live, but track local overrides:

```graphql
type Folder {
  # Live from folder type (not stored, resolved at query time)
  effectiveRoles: [Role!]!
  effectiveProperties: JSON!

  # Local additions (stored)
  localRoles: [Role!]!
  localPropertyValues: JSON!

  # What was overridden (stored)
  suppressedRoles: [String!]!  # role keys hidden locally
}
```

| Pros | Cons |
|------|------|
| Always current | Query complexity |
| Clear provenance | "Suppressed" is confusing UX |

---

## Problem 2: Adding Required Properties

This is the hardest edge case when modifying folder types.

### Option 2A: Required = Required on Create Only

New required properties only apply to folders created after the change. Existing folders keep their data as-is but show validation warnings.

```typescript
propertyRequirements: {
  "newField": {
    required: true,
    requiredAfter: "2024-01-15T00:00:00Z"
  }
}
```

### Option 2B: Required with Default Value

Required properties must specify a default value that auto-applies to existing folders.

```typescript
{
  "newField": {
    "type": "string",
    "required": true,
    "default": "Unknown",
    "migrateExisting": true
  }
}
```

### Option 2C: Pending Compliance State

Folders that don't meet new requirements enter a "pending compliance" state:

```graphql
enum FolderComplianceStatus {
  COMPLIANT
  PENDING_MIGRATION
  NON_COMPLIANT
}

type Folder {
  complianceStatus: FolderComplianceStatus!
  complianceIssues: [ComplianceIssue!]!
}
```

Surfaces in UI as "X folders need attention" with a migration wizard.

### Option 2D: Property Optionality Tiers

```typescript
type PropertyRequirement =
  | "required"           // Must exist, validated on create AND update
  | "required-new"       // Must exist for new folders only
  | "recommended"        // Shown prominently, but optional
  | "optional"           // Fully optional
```

---

## Problem 3: Locking Folder-Type-Defined Items

Preventing folders from modifying inherited roles/permissions while allowing full control over locally-created roles.

**Key Principle**: Folders have complete freedom to create new local roles with any permissions. The lock only applies to roles that originated from the folder type.

### Option 3A: Source Tagging (Selected)

Every role tracks its origin:

```graphql
type Role {
  key: String!
  source: RoleSource!
  permissions: [Permission!]!
}

enum RoleSource {
  FOLDER_TYPE  # Immutable, inherited from type
  LOCAL        # Created on this folder, fully editable
}
```

**Enforcement**: Mutations check `source` before allowing edits.

**Local Role Capabilities**:
- Create new roles with any key (except collisions with inherited keys)
- Assign any permissions available at the folder's scope
- Modify label, description, and permissions freely
- Delete local roles at any time

**Inherited Role Restrictions**:
- Cannot modify key, label, description, or permissions
- Cannot delete (would be restored on next migration anyway)

### Option 3B: Separate Collections

Store inherited and local items separately:

```graphql
type Folder {
  # Read-only, resolved from folder type
  inheritedRoles: [Role!]!

  # Fully editable, stored on folder
  localRoles: [Role!]!

  # Combined view
  allRoles: [Role!]!
}
```

### Option 3C: Permission Override Layer

Local folders can't modify folder-type roles, but can add permissions:

```graphql
type Folder {
  # From folder type (immutable)
  roleTemplates: [RoleTemplate!]!

  # Local additions to existing roles (additive only)
  rolePermissionAdditions: [{
    roleKey: String!
    additionalPermissions: [Permission!]!
  }]

  # New roles defined locally (fully editable)
  localRoles: [Role!]!
}
```

Like CSS cascade - you can add, but not remove.

### Option 3D: Lock Flags with Admin Override

```graphql
type RoleTemplate {
  key: String!
  locked: Boolean!
  lockReason: String
}
```

Tenant admins could override locks in exceptional cases.

---

## Recommended Approach: Layered Inheritance Model

Combines versioning (Option B) with source tagging (Option 3A) and compliance tracking (Option 2C).

### Proposed Schema

```graphql
type FolderType {
  id: ID!
  key: String!
  version: Int!

  roleTemplates: [RoleTemplate!]!
  propertySchema: JSON!

  changeLog: [FolderTypeChange!]!
}

type FolderTypeChange {
  version: Int!
  timestamp: DateTime!
  changeType: ChangeType!  # additive, breaking, permission
  description: String!
  affectedFields: [String!]!
}

type Folder {
  folderTypeKey: String!
  folderTypeVersion: Int!  # Pinned version

  properties: JSON!

  # Local additions (fully editable)
  localRoles: [Role!]!
  localPermissionGrants: [PermissionGrant!]!

  migrationStatus: MigrationStatus
}

enum MigrationStatus {
  CURRENT
  UPDATE_AVAILABLE
  MIGRATION_REQUIRED
}
```

### How It Works

1. **On folder type update**: Version increments, change is logged
2. **Automatic rolling migration starts** as a background job
3. **Dashboard shows**: Real-time progress "Migrating 15 folders to v3... (8/15 complete)"
4. **Breaking changes** require resolution config before migration starts
5. **Roles from folder type** are read-only; folders can create local roles freely
6. **Local roles** have full CRUD - any permissions, any configuration

### Automatic Rolling Migration

When a folder type version changes, migration happens automatically via a background job:

```typescript
type MigrationJob {
  id: string
  folderTypeId: string
  fromVersion: number
  toVersion: number
  status: 'pending' | 'running' | 'completed' | 'failed' | 'paused'

  // Progress tracking
  totalFolders: number
  migratedCount: number
  failedCount: number

  // Configuration for breaking changes
  resolutions: MigrationResolution[]

  // Timing
  startedAt: DateTime
  completedAt?: DateTime
  estimatedCompletion?: DateTime
}
```

**Flow**:
1. Admin updates folder type → version increments
2. System detects change type (additive, breaking, permission)
3. If **breaking**: Admin must provide resolutions (default values, role mappings) before job starts
4. If **additive/permission**: Job starts immediately
5. Job processes folders in batches (e.g., 50 at a time) with rate limiting
6. UI shows real-time progress via subscription/polling
7. Failed folders are logged and can be retried

**Rate Limiting**: Prevents overwhelming the database during large migrations. Configurable batch size and delay between batches.

---

## Edge Cases

| Edge Case | Handling |
|-----------|----------|
| Remove a role template | See "Role Deletion" section below |
| Rename a role template | Old key deprecated, new key added; migration maps old→new |
| Add required property | Must provide default value for migration |
| Remove a property | See "Property Deletion" section below |
| Change property type | Breaking change requiring migration with transform |
| Nested folder types | Each folder pins its own version independently |
| Role key collision | Folder-type wins, local must use different key |

---

## Property Deletion

When a property is removed from a folder type's schema:

### Behavior
1. **Change classified as**: `BREAKING` (data loss potential)
2. **Migration resolution required**: Admin must choose what happens to existing data

### Resolution Options

| Option | Description | Use Case |
|--------|-------------|----------|
| `DELETE_DATA` | Remove the property value from all folders | Data is obsolete, not needed |
| `KEEP_AS_UNMANAGED` | Keep data in folder but no longer validate/display | Preserve data for export/audit |
| `MIGRATE_TO_PROPERTY` | Move value to a different property | Renamed or restructured |

### Migration Job Handling

```typescript
interface PropertyDeletionResolution {
  deletedPropertyKey: string;
  action: 'DELETE_DATA' | 'KEEP_AS_UNMANAGED' | 'MIGRATE_TO_PROPERTY';
  targetPropertyKey?: string; // Required if MIGRATE_TO_PROPERTY
  transform?: string; // Optional JS expression for value transformation
}
```

### Example: Delete "legacyCode" Property

```typescript
{
  deletedPropertyKey: "legacyCode",
  action: "KEEP_AS_UNMANAGED"
}
// Result: Property value stays in folder.properties but is not shown in UI
// Can be accessed via API for migration/export purposes
```

---

## Role Deletion

When a role template is removed from a folder type:

### Behavior
1. **Change classified as**: `BREAKING`
2. **Migration resolution required**: Admin must choose what happens to:
   - The role definition on existing folders
   - Users currently assigned to that role

### Resolution Options

| Option | Description | User Impact |
|--------|-------------|-------------|
| `DELETE_ROLE` | Remove role from all folders | Users lose the role assignment |
| `CONVERT_TO_LOCAL` | Role becomes a local role on each folder | Users keep role, but it's now folder-managed |
| `REASSIGN_TO_ROLE` | Reassign all users to a different role | Users get new role with potentially different permissions |

### Migration Job Handling

```typescript
interface RoleDeletionResolution {
  deletedRoleKey: string;
  action: 'DELETE_ROLE' | 'CONVERT_TO_LOCAL' | 'REASSIGN_TO_ROLE';
  targetRoleKey?: string; // Required if REASSIGN_TO_ROLE
}
```

### Example: Delete "contributor" Role, Reassign to "viewer"

```typescript
{
  deletedRoleKey: "contributor",
  action: "REASSIGN_TO_ROLE",
  targetRoleKey: "viewer"
}
// Result:
// 1. Role definition removed from folder
// 2. All users with "contributor" role now have "viewer" role
// 3. FGA tuples updated: delete contributor, add viewer
```

### Example: Convert "specialist" to Local Role

```typescript
{
  deletedRoleKey: "specialist",
  action: "CONVERT_TO_LOCAL"
}
// Result:
// 1. Role moves from inheritedRoles to localRoles
// 2. source changes from FOLDER_TYPE to LOCAL
// 3. User assignments preserved
// 4. Folder admin can now modify the role freely
```

---

## Resolved Decisions

1. ✅ **Local roles**: Folders can create new roles with full control (any permissions, full CRUD). Only inherited roles from folder type are locked.

2. ✅ **Versioning activation**: Versioning only triggers when folders exist that use the folder type. New folder types can be edited freely until the first folder is created from them.

3. ✅ **Automatic migration**: All folder type updates trigger automatic rolling migration via background job. No manual "migrate" button needed.

4. ✅ **Breaking changes**: Admin must provide resolutions (defaults, mappings) before the migration job can start. Job waits in "pending" until configured.

5. ✅ **Property deletion**: Three options - delete data, keep as unmanaged, or migrate to another property.

6. ✅ **Role deletion**: Three options - delete role, convert to local, or reassign users to different role.

7. ✅ **Version storage**: Separate `FolderTypeVersion` collection with full snapshots. Pre-compute `changeType` for quick queries, compute detailed changes on-demand from snapshots.

8. ✅ **Version retention**: Keep last 10 versions per folder type. Cleanup on insert - delete older versions beyond the limit.

9. ✅ **Draft mode**: No drafts for v1. Changes save immediately, but migration jobs are scheduled for **10pm server timezone**. Admin can manually trigger immediate execution if needed.

10. ✅ **Batch processing**: 100 folders per batch, 100ms delay between batches, parallel processing within each batch (Promise.allSettled).

11. ✅ **Failed migrations**: Continue always - process all folders, collect errors, allow retry of failed folders afterward.

12. ✅ **Unmanaged property data**: Keep forever. Never auto-delete - admin explicitly chose to preserve data.

---

## References

- Current folder type schema: `/Users/little/Projects/folder/src/graphql/folder.schema.graphql`
- Folder type service: `/Users/little/Projects/folder/src/features/folder-type/folder-type.service.ts`
- Permission registry: `/Users/little/Projects/folder-client-rsc/src/lib/permissions/permission-registry.ts`

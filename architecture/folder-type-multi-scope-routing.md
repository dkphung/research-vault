# Folder Type Multi-Scope Routing Research

**Date:** 2025-12-14

## Context

Folder types can now be scoped to any folder in the hierarchy via `scopeFolderId`. This enables defining folder types at:
- **Global** scope (`scopeFolderId: "GLOBAL"`) - available everywhere
- **Tenant** scope (`scopeFolderId: tenantId`) - available to tenant and descendants
- **Campus** scope (`scopeFolderId: campusId`) - available to campus and descendants
- **Any folder** - for deep customization

The current UI only supports tenant-scoped folder types at `/tenants/[tenantId]/folder-types/` and uses mock data. We need to:
1. Migrate to real GraphQL API
2. Support create/update at global, tenant, and campus levels
3. Decide on routing and component architecture

## Current Implementation Analysis

### GraphQL API (from folder service)

**Queries:**
```graphql
# Get folder types defined at a specific scope (not inherited)
folderTypesAtScope(scopeFolderId: ID!): [FolderType!]!

# Get all folder types available at a folder (including inherited)
availableFolderTypes(folderId: ID!): [FolderType!]!

# Get single folder type
folderType(id: ID!): FolderType
```

**Mutations:**
```graphql
createFolderType(input: CreateFolderTypeInput!): FolderType!
updateFolderType(id: ID!, input: UpdateFolderTypeInput!): FolderType!
deleteFolderType(id: ID!): Boolean!
archiveFolderType(id: ID!): FolderType!
unarchiveFolderType(id: ID!): FolderType!
copyFolderType(input: CopyFolderTypeInput!): FolderType!
```

**CreateFolderTypeInput:**
```graphql
input CreateFolderTypeInput {
  id: ID!                          # UUID
  scopeFolderId: ID!               # Where type is defined
  key: String!                     # Unique identifier
  label: String!                   # Display name
  description: String
  scopeTier: Int!                  # 0-10
  allowedSubFolderTypes: [String!]
  roleTemplates: [RoleTemplateInput!]
  propertySchema: JSON             # JSON Schema draft-07
  uiSchema: JSON                   # RJSF UI schema
}
```

### Current Form Pattern (TenantDetailsSection)

The tenant create/edit form demonstrates the established pattern:

1. **RSC page** fetches data, passes to client component
2. **Client component** uses TanStack React Form
3. **Server actions** handle mutations with Zod validation
4. **Schema** lives in domain module (`tenant.schema.ts`)

```typescript
// page.tsx (RSC)
export default async function NewTenantPage() {
  const folderTypes = await getFolderTypesAtScope("GLOBAL");
  return <TenantDetailsSection availableFolderTypes={folderTypes} />;
}

// tenant-details-section.tsx (Client)
const form = useForm({
  defaultValues: { ... },
  onSubmit: async ({ value }) => {
    startTransition(async () => {
      const validated = createTenantInputSchema.parse(value);
      await createTenant(validated);
      router.push(`/tenants/${created.id}`);
    });
  },
});
```

### Current Folder Type Pages

| Route | Current State |
|-------|---------------|
| `/folder-types/` | Placeholder "Coming Soon" |
| `/tenants/[tenantId]/folder-types/` | Client component, mock data |
| `/tenants/[tenantId]/folder-types/[typeId]` | Client component, mock data, complex tabs |

The detail page at `[typeId]/page.tsx` is 633 lines with:
- General tab (name, slug, description, properties)
- Hierarchy tab (scope tier, allowed parent/child types)
- Roles tab (role template editor)
- Properties tab (property schema builder)

## Routing Options

### Option A: Scope-Specific Routes (Recommended)

Keep folder types under their parent context:

```
/folder-types/                          # Global list
/folder-types/new                       # Create at global scope
/folder-types/[typeId]                  # Edit global type

/tenants/[tenantId]/folder-types/       # Tenant list (inherited + defined here)
/tenants/[tenantId]/folder-types/new    # Create at tenant scope
/tenants/[tenantId]/folder-types/[typeId]  # Edit tenant type

/tenants/[tenantId]/campuses/[campusId]/folder-types/      # Campus list
/tenants/[tenantId]/campuses/[campusId]/folder-types/new   # Create at campus
/tenants/[tenantId]/campuses/[campusId]/folder-types/[typeId]  # Edit campus type
```

**Pros:**
- URL clearly indicates scope
- Consistent with existing URL hierarchy (Global → Tenant → Campus)
- Navigation context preserved (breadcrumbs, sidebar)
- Easy to understand permissions model

**Cons:**
- Requires route files at each level
- Form component needs to handle different scopes

### Option B: Unified Route with Query Param

Single route with scope parameter:

```
/folder-types?scope=GLOBAL
/folder-types?scope=tenant-123
/folder-types?scope=campus-456
/folder-types/new?scope=tenant-123
```

**Pros:**
- Single set of route files
- Less code duplication

**Cons:**
- Loses URL hierarchy context
- Breaks existing navigation patterns
- Query params feel less RESTful
- Harder to bookmark/share specific views

### Option C: Global-Only Route with Inline Scope Selector

Single `/folder-types/` route that lets you select scope inline:

```
/folder-types/                    # Shows all scopes with filter
/folder-types/new                 # Select scope in form
/folder-types/[typeId]            # Edit (scope shown, read-only)
```

**Pros:**
- Simplest routing structure
- Single source of truth for folder type UI

**Cons:**
- Loses context when drilling down from tenant/campus
- Navigation complexity for selecting scope
- Doesn't fit current app navigation patterns

## Recommended Approach: Option A with Shared Components

### Route Structure

```
src/app/(auth)/
├── folder-types/
│   ├── page.tsx                 # Global list
│   ├── new/
│   │   └── page.tsx            # Create global type
│   ├── [typeId]/
│   │   └── page.tsx            # Edit global type
│   └── _components/             # Shared folder-type components
│       ├── folder-type-list.tsx
│       ├── folder-type-form.tsx
│       ├── property-schema-builder.tsx
│       └── role-template-editor.tsx
├── tenants/[tenantId]/
│   └── folder-types/
│       ├── page.tsx             # Tenant list
│       ├── new/
│       │   └── page.tsx        # Create tenant type
│       └── [typeId]/
│           └── page.tsx        # Edit tenant type
└── tenants/[tenantId]/campuses/[campusId]/
    └── folder-types/
        ├── page.tsx             # Campus list
        ├── new/
        │   └── page.tsx        # Create campus type
        └── [typeId]/
            └── page.tsx        # Edit campus type
```

### Component Architecture

**Shared components** (in `/folder-types/_components/`):
- `folder-type-list.tsx` - List view with grid/table toggle
- `folder-type-form.tsx` - Create/edit form (tabbed interface)
- `property-schema-builder.tsx` - Property schema builder (already exists)
- `role-template-editor.tsx` - Role template editor (already exists)

**Route-specific pages** (thin wrappers):
- Fetch scope-specific data
- Pass scopeFolderId to shared components
- Handle route-specific navigation (back links, breadcrumbs)

### Server Module Structure

```
src/server/folder-type/
├── folder-type.schema.ts        # Types and validation schemas
├── folder-type.actions.ts       # Server actions (queries)
├── folder-type.mutations.ts     # Server actions (mutations) - NEW
└── folder-type.graphql          # GraphQL operations
```

### Form Design

The form component should accept:
```typescript
interface FolderTypeFormProps {
  // Scope context
  scopeFolderId: string;           // "GLOBAL" | tenantId | campusId
  scopeLabel: string;              // "Global" | "Acme Corp" | "Main Campus"

  // Data
  folderType?: FolderType;         // Existing type for edit mode
  availableFolderTypes: FolderType[]; // For allowed parent/child selection

  // Navigation
  backUrl: string;                 // Where to go on cancel/save
}
```

### List Design

The list component should show:
```typescript
interface FolderTypeListProps {
  // Scope context
  scopeFolderId: string;
  scopeLabel: string;

  // Data
  folderTypes: FolderType[];       // Types defined at this scope
  inheritedTypes?: FolderType[];   // Types inherited from ancestors

  // Navigation
  createUrl: string;               // URL for "Create Type" button
  editUrl: (typeId: string) => string; // URL builder for edit
}
```

## Migration Strategy

### Phase 1: Server Module Migration
1. Add mutations to `folder-type.mutations.ts`
2. Add GraphQL operations for create/update/delete
3. Create Zod schemas for inputs

### Phase 2: Shared Components
1. Move `property-schema-builder.tsx` and `role-template-editor.tsx` to `/folder-types/_components/`
2. Create `folder-type-form.tsx` (extract from current 633-line page)
3. Create `folder-type-list.tsx` (extract from current list page)

### Phase 3: Global Routes
1. Implement `/folder-types/` list page (replace placeholder)
2. Implement `/folder-types/new` create page
3. Implement `/folder-types/[typeId]` edit page

### Phase 4: Tenant Routes
1. Refactor `/tenants/[tenantId]/folder-types/` to use shared components
2. Add `/tenants/[tenantId]/folder-types/new`
3. Refactor `/tenants/[tenantId]/folder-types/[typeId]`

### Phase 5: Campus Routes
1. Create `/tenants/[tenantId]/campuses/[campusId]/folder-types/`
2. Create `/tenants/[tenantId]/campuses/[campusId]/folder-types/new`
3. Create `/tenants/[tenantId]/campuses/[campusId]/folder-types/[typeId]`

## Form Simplifications

The current 633-line form has some issues to address:

1. **Complex tab structure** - Consider if all tabs are needed for MVP
2. **Mock data types** - Need to align with GraphQL schema types
3. **UI Builder** and **Preview** tabs - Nice-to-have, not MVP
4. **Allowed parent/child types** - GraphQL uses `allowedSubFolderTypes` keys, not IDs

### Suggested Tab Structure for MVP

1. **Details** - Key, label, description, scope tier
2. **Properties** - Property schema builder (JSON Schema output)
3. **Roles** - Role template editor
4. **Children** - Allowed subfolder type keys

The Hierarchy/parent types concept may not apply with the new scope model since types are inherited down the tree automatically.

## Key Decisions Needed

1. **Should we support editing scopeFolderId?** - Probably not, it's an identity field
2. **Should we show inherited types?** - Yes, with visual distinction
3. **Can types be copied across scopes?** - Yes, GraphQL has `copyFolderType`
4. **What about archiving?** - Should support archive/unarchive in UI

## References

- GraphQL Schema: `/Users/little/Projects/folder/src/graphql/folder.schema.graphql`
- Current tenant form: `src/app/(auth)/tenants/_components/tenant-details-section.tsx`
- Current folder type detail: `src/app/(auth)/tenants/[tenantId]/folder-types/[typeId]/page.tsx`
- Prototype mapping: `docs/prototype-mapping.md`

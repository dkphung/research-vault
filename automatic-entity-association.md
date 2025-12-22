---
tags: [architecture]
date: 2024-12-22
status: complete
---

---
layout: default
title: Automatic Entity Association Research
---

[← Back to Index](../index.md)

# Automatic Entity Association Research

**Date:** 2024-12-09
**Status:** Draft

## Problem Statement

When a user creates a form under a folder (e.g., ProfessorXLab), they may fill in form fields that reference other entities — such as "Room 101" or "Department X". Currently, if the user wants the form to appear in the Room 101 folder or Department folder, they must explicitly add those as parents.

**Goal:** Automatically associate forms (and potentially folders) with other folders/entities based on shared property values, enabling discovery without explicit parent assignment.

## Context

### Current Multi-Parent Model

The system is being refactored to support multi-parent folders (DAG structure). This allows a folder to appear under multiple parents. However, parents are still explicitly defined by the user.

### Entity Landscape

The system has a heterogeneous entity model:

- **Some entities are folders** — e.g., Room 101 might be a folder of type "Room"
- **Some entities are external** — e.g., departments might live in a separate system
- **Both folders and forms can reference entities** — via property values (structured references or picklists)

### Relationship Types

Two ways entities become related:

1. **Direct reference** — A form field explicitly selects a folder (e.g., "Room" picklist selects Room 101 folder)
2. **Shared property value** — A form and folder both have properties referencing the same external entity ID

### User Need

The primary use case is **discovery**: "Show me everything related to Room 101" — regardless of where forms/folders were originally created or who owns them.

---

## Approach A: Entity Index with Reverse Lookups (Recommended)

### Concept

Create a lightweight index that tracks "what references what." When a form or folder saves a property value that references an entity, write an entry to an index. Discovery becomes a simple query against this index.

### Data Model

```typescript
// Index collection: entity_references
{
  entityType: "room",           // Type of referenced entity
  entityId: "room-101",         // ID of referenced entity
  referencedBy: {
    type: "form",               // What type of thing references it
    id: "form-abc",             // ID of the referencing thing
    field: "roomNumber",        // Which field contains the reference
    tenantId: "tenant-123"      // For multi-tenancy
  },
  createdAt: Date
}
```

### How It Works

1. **On form/folder save:** Extract entity references from properties, write index entries
2. **On form/folder delete:** Remove corresponding index entries
3. **On query "what's related to X":** Query index for `entityId: X`

### Example Query

```typescript
// "Show me everything related to Room 101"
const references = await db.collection('entity_references').find({
  entityType: 'room',
  entityId: 'room-101',
  tenantId: currentTenant
}).toArray();

// Returns: [{ referencedBy: { type: 'form', id: 'form-abc' }}, { referencedBy: { type: 'folder', id: 'folder-xyz' }}]
```

### Schema Metadata Required

To know which fields contain entity references, the property schema needs metadata:

```typescript
// In folder type's propertySchema
{
  roomNumber: {
    type: "string",
    format: "entity-reference",    // Marker that this is a reference
    entityType: "room",            // What type of entity it references
    entitySource: "folder:room"    // Where entities come from (folder type or external)
  }
}
```

### Pros

- **Fast read-time lookups** — Index is pre-computed, queries are O(1)
- **Works uniformly** — Same pattern for folders, forms, external entities
- **Decoupled from hierarchy** — Doesn't pollute parent/child model
- **Flexible** — Can add new entity types without schema changes
- **Auditable** — Index entries can track when relationships were created

### Cons

- **Write-time overhead** — Must update index on every save
- **Sync complexity** — Index must stay consistent with source data
- **Schema dependency** — Requires property schema to declare what fields are references
- **Migration needed** — Existing data needs backfill

### Authorization Considerations

The index enables discovery, but viewing the referenced items still requires authorization:

```typescript
// Pseudo-code for "related items" query
const references = await getEntityReferences('room-101');
const visibleItems = await filterByAuthorization(references, currentUser);
```

---

## Approach B: Virtual Folder Membership (Computed at Read Time)

### Concept

Don't store associations explicitly. Instead, when viewing a folder that represents an entity (e.g., Room 101), dynamically query for all forms/folders whose properties reference that entity.

### How It Works

1. **Folder has an "entity identity"** — Room 101 folder knows it represents entity `room:101`
2. **On "view folder contents":** Query forms/folders where any property matches `room:101`
3. **No write-time indexing** — Relationships are computed on demand

### Example Query

```typescript
// Room 101 folder viewing its "related items"
const roomFolder = await getFolderById('room-101-folder');
const entityId = roomFolder.properties.entityId; // "room:101"

// Find all forms/folders that reference this entity
const relatedForms = await db.collection('forms').find({
  tenantId: currentTenant,
  'properties.roomNumber': entityId
}).toArray();

const relatedFolders = await db.collection('folders').find({
  tenantId: currentTenant,
  'properties.referenceRoom': entityId
}).toArray();
```

### Pros

- **No write overhead** — No index to maintain
- **Always consistent** — No sync issues, queries source of truth directly
- **Simpler implementation** — Fewer moving parts

### Cons

- **Slow at scale** — Must scan potentially large collections on every view
- **Query complexity** — Need to know which fields to search across
- **No unified query** — Must query each collection separately
- **Performance unpredictable** — Depends on data volume and field distribution

### When This Works Well

- Small datasets
- Infrequent access to "related items" view
- Prototyping before committing to index approach

---

## Approach C: Implicit Parents via Property Rules

### Concept

Extend the multi-parent model to support "computed parents." A folder or form can have explicit parents (user-defined) plus implicit parents (derived from property values via rules).

### Data Model

```typescript
// Folder type definition includes parent derivation rules
{
  typeKey: "lab-form",
  propertySchema: {
    roomNumber: { type: "string", format: "entity-reference", entityType: "room" }
  },
  parentDerivationRules: [
    {
      field: "roomNumber",
      targetFolderType: "room",       // Look for folders of this type
      matchProperty: "entityId"        // Where folder.properties.entityId === form.properties.roomNumber
    }
  ]
}
```

### How It Works

1. **On form/folder save:** Evaluate derivation rules, compute implicit parents
2. **Store as separate field:** `implicitParents: [{ id, type, derivedFrom: "roomNumber" }]`
3. **On folder "view children":** Query for items where `parents` OR `implicitParents` includes this folder

### Example

```typescript
// Form saved with roomNumber: "room-101"
// System evaluates rules, finds Room 101 folder, adds implicit parent:
{
  id: "form-abc",
  parents: [{ id: "professor-x-lab", type: "FOLDER" }],           // Explicit
  implicitParents: [{ id: "room-101-folder", type: "FOLDER", derivedFrom: "roomNumber" }]  // Computed
}
```

### Pros

- **Uses existing parent model** — Leverages multi-parent infrastructure
- **Unified query** — "Children of X" query works for both explicit and implicit
- **Authorization reuse** — Can inherit `can_view` through implicit parents (if desired)
- **Visible to user** — UI can show "this form appears here because of Room 101 reference"

### Cons

- **Blurs parent semantics** — Are implicit parents "real" parents? Affects moves, deletes, etc.
- **Write-time computation** — Still need to compute and store on save
- **Rule complexity** — Derivation rules add configuration overhead
- **Cascade effects** — If Room 101 folder is deleted, what happens to implicit parent references?
- **Authorization ambiguity** — Does `can_view` inherit through implicit parents?

### Key Design Questions

If pursuing this approach:

1. **Do implicit parents grant view permission?** (Probably not — could leak data)
2. **Can users remove implicit parents?** (Probably not — they're derived)
3. **What happens when source property changes?** (Must recompute)
4. **Are implicit parents visible in the folder's parent list?** (Probably yes, with visual distinction)

---

## Comparison Matrix

| Aspect | A: Entity Index | B: Virtual Membership | C: Implicit Parents |
|--------|-----------------|----------------------|---------------------|
| Write overhead | Medium (index writes) | None | Medium (compute + store) |
| Read performance | Fast (indexed) | Slow (scan) | Fast (indexed) |
| Consistency | Requires sync | Always consistent | Requires sync |
| Implementation complexity | Medium | Low | High |
| Works for external entities | Yes | Yes | No (needs folder target) |
| Reuses parent model | No | No | Yes |
| Authorization model | Separate | Separate | Could inherit |
| UI clarity | Clear (separate "related" section) | Clear | Potentially confusing |

---

## Recommendation

**Start with Approach A (Entity Index)** for these reasons:

1. **Clean separation of concerns** — Discovery is a separate concept from hierarchy
2. **Works universally** — Handles folders, forms, and external entities uniformly
3. **Scales well** — Read-time performance is predictable
4. **No semantic confusion** — "Related items" is clearly different from "children"
5. **Authorization is explicit** — Doesn't accidentally grant permissions through implicit relationships

### Suggested Implementation Path

1. **Define reference metadata in property schemas** — Mark which fields are entity references
2. **Create `entity_references` collection** — Store reverse lookup index
3. **Add indexing on form/folder save** — Extract references, write index entries
4. **Add cleanup on delete** — Remove index entries when source is deleted
5. **Add GraphQL query** — `relatedItems(entityType, entityId)` returns referenced forms/folders
6. **UI integration** — Folder detail page shows "Related Items" section for entity-type folders

---

## Open Questions

1. **Should "related items" be a separate tab/section, or mixed with children?**
   - Recommendation: Separate section to avoid confusion

2. **Should related items show in both directions?**
   - e.g., Form shows "Related to: Room 101, Department X"
   - Room 101 shows "Referenced by: Form ABC"

3. **How to handle property schema changes?**
   - If a field is changed from regular string to entity-reference, need to backfill index

4. **Should external entities have "landing pages"?**
   - If Department X isn't a folder, where does the user go to see "everything related to Department X"?

5. **Performance at scale?**
   - What if an entity (e.g., "Main Campus") is referenced by 10,000 forms?
   - May need pagination, filtering, or lazy loading

---

## Next Steps

- [ ] Validate approach with stakeholders
- [ ] Decide on open questions above
- [ ] Create specification document if proceeding

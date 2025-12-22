---
tags: [architecture]
date: 2024-12-22
status: complete
---

---
layout: default
title: Multi-Parent Folders Research
---

[← Back to Index](../index.md)

# Multi-Parent Folders Research

**Date:** 2024-12-09
**Status:** Final

## Problem Statement

Currently, folders have a single parent (tree structure). We want to explore allowing folders to have multiple parents (DAG structure) to enable folders to appear in multiple locations.

## Current Model

```fga
type folder
  relations
    define parent: [campus, folder]
    define configuration_manager: configuration_manager from parent
    define can_view_folder: [role#member] or configuration_manager or can_view_folder from parent
    # ... other permissions inherit or use configuration_manager
```

**Key characteristics:**
- Single `parent` relation (tree)
- `configuration_manager` inherited through entire parent chain
- `same_campus_policy` inherited through parent chain
- All permissions reference `configuration_manager` or inherit via `from parent`

## Key Insight: FGA Already Supports Multi-Parent

OpenFGA's `from parent` uses **union semantics** by default. If a folder has multiple parents, permissions resolve as the union of all parent paths.

**Example:**
```
Campus A
  ├── Safety Program (config_manager: Alice)
  │     └── Lab X ←──┐
  │                  │
  └── Radiation Program (config_manager: Bob)
        └───────────────┘ (Lab X has both as parents)
```

With `configuration_manager from parent`, Lab X's config_managers = **Alice OR Bob** (union).

## Analysis: Is Union Semantics Acceptable?

| Scenario | Union Behavior | Acceptable? |
|----------|----------------|-------------|
| `can_view_folder` | Any parent path grants view | ✅ Yes - additive, no conflict |
| `configuration_manager` | Any ancestor's config_manager can configure | ✅ Yes - if multiple authorities are acceptable |
| `can_update_folder` | Direct grants only (no inheritance) | ✅ N/A |

**Decision:** Union semantics are acceptable for our use case. Both program managers being able to configure a shared lab is the desired behavior.

## Implications

### No Direct `campus` Relation Needed in FGA

The original proposal added a direct `campus` relation to resolve `configuration_manager` deterministically. Since union semantics are acceptable, this is unnecessary.

- `configuration_manager from parent` works correctly with multi-parent
- No FGA model changes required (already supports multi-parent)

### Tier Validation is Application Logic

Folders have a "scope tier" (e.g., tier 1 = programs, tier 2 = labs). Only higher tiers can be parents of lower tiers.

| Concern | Responsibility |
|---------|----------------|
| "Can user X add a parent?" | FGA (authorization) |
| "Is parent Y a valid tier for folder Z?" | Application (business rule) |

Tier validation belongs in the service layer, not FGA.

## Final Model

**No changes to FGA model required.** The existing model already supports multi-parent:

```fga
type folder
  relations
    # Multiple parents allowed (DAG) - permissions inherit through all parents (union semantics)
    define parent: [campus, folder]

    # Inherits through parent chain - with multi-parent, union of all ancestors' config_managers
    define configuration_manager: [role#member] or configuration_manager from parent

    # View inherits through all parent paths
    define can_view_folder: [role#member] or configuration_manager or can_view_folder from parent

    # Direct grants only
    define can_update_folder: [role#member] or configuration_manager
    define can_delete_folder: [role#member] or configuration_manager
    define can_add_child_folder: [role#member] or configuration_manager
    define can_remove_child_folder: [role#member] or configuration_manager

    # Role management
    define can_view_roles: [role#member] or configuration_manager or can_view_roles from parent
    define can_create_role: [role#member] or configuration_manager
    define can_update_role: [role#member] or configuration_manager
    define can_delete_role: [role#member] or configuration_manager
    define can_add_role_member: [role#member] or configuration_manager
    define can_remove_role_member: [role#member] or configuration_manager

    # Resource creation
    define can_create_form_template: [role#member] or configuration_manager
    define can_create_workflow: [role#member] or configuration_manager
    define can_create_dashboard: [role#member] or configuration_manager
    define can_create_metric: [role#member] or configuration_manager
    define can_link_inventory: [role#member] or configuration_manager

    # Membership
    define member: can_view_folder or member from parent
```

## Implementation Impact

### FGA Model Changes
- **None required** - existing model already supports multi-parent via union semantics

### Service Layer Changes

**New operations:**
```typescript
addParent(folderId: string, parentId: string, parentType: ParentType)
removeParent(folderId: string, parentId: string, parentType: ParentType)
```

**Validation required:**
- Tier validation: parent must be a valid tier for the child folder (application rule)
- Cycle detection: adding parent must not create cycle in DAG
- Minimum parent: folder must have at least one parent

**Cycle detection algorithm:**
```typescript
function wouldCreateCycle(folderId: string, newParentId: string): boolean {
  // DFS from newParentId upward through all its parents
  // If we reach folderId, adding this edge creates a cycle
  const visited = new Set<string>();
  const stack = [newParentId];

  while (stack.length > 0) {
    const current = stack.pop();
    if (current === folderId) return true;
    if (visited.has(current)) continue;
    visited.add(current);

    const folder = getFolderById(current);
    for (const parent of folder.parents) {
      if (parent.type === 'FOLDER') {
        stack.push(parent.id);
      }
    }
  }
  return false;
}
```

### Data Model Changes

**Before:**
```typescript
{
  parentId: string | null
  parentType: ParentType | null
  ancestors: Ancestor[]
  depth: number
}
```

**After:**
```typescript
{
  parents: Array<{ id: string; type: ParentType }>
  // ancestors: removed (not needed without breadcrumbs)
  // depth: removed (not meaningful in DAG)
}
```

Note: `campusId` is optional in the data model. It's useful for queries ("show all folders in Campus A") but not required for FGA authorization.

### Tuple Changes

**On folder creation:**
```typescript
// Write parent tuple(s)
fga.write(tuple(campus:X, "parent", folder:Y))
fga.write(tuple(folder:A, "parent", folder:Y))  // if multiple parents
```

**On add parent:**
```typescript
fga.write(tuple(folder:newParent, "parent", folder:target))
```

**On remove parent:**
```typescript
fga.delete(tuple(folder:oldParent, "parent", folder:target))
```

## Complexity Comparison

| Component | Original Proposal | Final Approach |
|-----------|-------------------|----------------|
| FGA Model | 🟡 Add campus relation | 🟢 No changes needed |
| Permission Logic | 🟢 Union works naturally | 🟢 Union works naturally |
| Cycle Detection | 🟠 Required | 🟠 Required |
| Service Layer | 🟡 Moderate changes | 🟡 Moderate changes |
| Migration | 🟡 Add campus tuples | 🟢 Convert parent to array |
| **Estimated Effort** | 2-3 days | 2-3 days |

## Use Case: Lab Safety Programs

**Scenario:** A campus has multiple safety programs (Lab Safety, Radiation Safety) that need visibility into the same labs.

```
Campus: Chemistry
├── Lab Safety Program (folder)
│   ├── Form Templates: Safety Checklist, Hazard Assessment
│   └── needs visibility into Labs A, B
│
├── Radiation Safety Program (folder)
│   ├── Form Templates: Dosimetry Log, Waste Tracking
│   └── needs visibility into Labs A, C
│
├── Lab A (folder) ← needs to be in BOTH programs
│     parents: [Lab Safety Program, Radiation Safety Program]
├── Lab B (folder)
│     parents: [Lab Safety Program]
├── Lab C (folder)
│     parents: [Radiation Safety Program]
```

**Why multi-parent works:**
- Lab A inherits `can_view_folder` from both programs (union semantics)
- Lab Safety admin can view Lab A via their program
- Radiation Safety admin can also view Lab A via their program
- Forms work via audience mechanism (who can fill forms)
- All entities stay within same campus (constraint satisfied)

**Key tuples for Lab A:**
```
folder:lab-a#parent@folder:lab-safety-program
folder:lab-a#parent@folder:radiation-safety-program
```

Note: No `campus` tuple needed - `configuration_manager` resolves through parent chain.

## Decision: Union Semantics for All Inherited Permissions

**Chosen approach:**
- All `from parent` permissions use union semantics (FGA default)
- `configuration_manager` inherits through parent chain (union of all ancestors)
- Tier validation enforced in application code, not FGA
- No direct `campus` relation needed in FGA model

## Resolved Questions

1. **Root folders:** Can a folder have zero parents? Or must it have at least one (campus or folder)?
   - **Decision:** No. Every folder must have at least one parent. Campus can serve as the minimum parent.

2. **Primary parent:** For UI breadcrumbs, should we designate one parent as "primary" for display purposes?
   - **Decision:** No breadcrumbs. Show all parents as chips/tags instead of a linear path.

3. **Move operation:** Keep `moveFolder` (replaces all parents) or only `addParent`/`removeParent`?
   - **Decision:** Replace with `addParent`/`removeParent`, deprecate `moveFolder`.

4. **Descendant queries:** How to efficiently query all descendants when paths branch?
   - **Decision:** Recursive query at read time. Simpler to maintain, descendant queries are infrequent (delete validation, not hot path). Can migrate to materialized list if performance becomes an issue.

5. **Tier validation:** Where to enforce that only valid tiers can be parents?
   - **Decision:** Application code (service layer). FGA handles authorization ("can user do this?"), application handles business rules ("is this a valid operation?").

## Next Steps

- ✅ Decide on approach (union semantics)
- ✅ Document use case (Lab Safety Programs)
- ✅ Confirm FGA model supports multi-parent (no changes needed)
- [ ] Create specification document for implementation
- [ ] Design database migration strategy
- [ ] Update GraphQL schema for multi-parent operations

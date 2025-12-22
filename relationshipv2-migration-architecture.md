---
tags: [architecture]
date: 2024-12-22
status: complete
---

# RelationshipV2 to Folder-Server Migration Architecture

**Date**: 2025-12-11
**Status**: Research / Planning
**Related**: [Migration Spec](../spec/06-relationshipv2-migration-spec.md)

## Executive Summary

This document describes the architecture for migrating from relationship-v2 (Neptune graph database) to folder-server (MongoDB) using Apollo Federation as the unified routing layer. The key changes are:

1. **Apollo Router** becomes the single entry point for all GraphQL requests
2. **Queries** for tenant, campus, program entities route to folder-server
3. **Mutations** for tenant, campus, program are removed from relationship-v2 and implemented in folder-server
4. **Dual-write** to Neptune ensures backward compatibility during transition
5. **OpenFGA** remains the authorization layer, with both services writing tuples

---

## Current State

```mermaid
flowchart TB
    subgraph Clients["Client Applications"]
        Shell["Platform Shell"]
    end

    subgraph GraphQL["GraphQL Services (Direct Access)"]
        RelV2["relationship-v2"]
        OtherSvc["Other Services..."]
    end

    subgraph Databases["Data Stores"]
        Neptune[("Neptune<br/>Graph DB")]
        FGA[("OpenFGA")]
    end

    Shell --> RelV2
    Shell --> OtherSvc

    RelV2 --> Neptune
    RelV2 --> FGA

    style RelV2 fill:#ff9999
    style Neptune fill:#ff9999
```

**Current Issues:**
- No unified API gateway - clients connect directly to services
- Neptune is expensive and complex to maintain
- Schema is rigid; adding new entity types requires code changes
- Dual database sync (Neptune + FGA) is error-prone

---

## Target State

```mermaid
flowchart TB
    subgraph Clients["Client Applications"]
        Shell["Platform Shell"]
    end

    subgraph Gateway["Apollo Gateway"]
        Router["Apollo Router"]
    end

    subgraph Subgraphs["Federated Subgraphs"]
        FolderSvc["folder-server"]
        RelV2["relationship-v2"]
        OtherSvc["Other Services..."]
    end

    subgraph Databases["Data Stores"]
        MongoDB[("MongoDB")]
        Neptune[("Neptune")]
        FGA[("OpenFGA")]
    end

    Shell --> Router
    Router --> FolderSvc
    Router --> RelV2
    Router --> OtherSvc

    FolderSvc --> MongoDB
    FolderSvc --> Neptune
    FolderSvc --> FGA
    RelV2 --> Neptune
    RelV2 --> FGA

    style FolderSvc fill:#99ff99
    style MongoDB fill:#99ff99
```

**Benefits:**
- Apollo Router provides single entry point for all clients
- Unified folder model for all hierarchical entities
- MongoDB is simpler, cheaper, and scales horizontally
- Dynamic folder types - no code changes for new entity types
- folder-server dual-writes to Neptune for backward compatibility

---

## Query Routing

Apollo Router uses the federated supergraph schema to route queries to the appropriate subgraph based on which service owns the type.

```mermaid
flowchart TB
    subgraph Client["Platform Shell"]
        Query["GraphQL Query"]
    end

    subgraph Router["Apollo Router"]
        Route["Route by Type Owner"]
    end

    subgraph FolderServer["folder-server"]
        TQ["tenants"]
        CQ["campuses"]
        PQ["programs"]
        FQ["folders"]
        RQ["getRoles"]
        PEQ["getPermissions"]
    end

    subgraph RelV2["relationship-v2"]
        OQ["Other queries..."]
    end

    subgraph Databases["Data Stores"]
        MongoDB[("MongoDB")]
        Neptune[("Neptune")]
        FGA[("OpenFGA")]
    end

    Query --> Route
    Route -->|"Tenant, Campus, Program, Roles"| FolderServer
    Route -->|"Other"| RelV2

    FolderServer --> MongoDB
    FolderServer --> FGA
    RelV2 --> Neptune

    style FolderServer fill:#99ff99
    style MongoDB fill:#99ff99
```

### Query Routing Table

| Query | Current Owner | Target Owner |
|-------|---------------|--------------|
| `tenants` | relationship-v2 | **folder-server** |
| `campus(id)` | relationship-v2 | **folder-server** |
| `campuses` | relationship-v2 | **folder-server** |
| `program(id)` | relationship-v2 | **folder-server** |
| `searchPrograms` | relationship-v2 | **folder-server** |
| `programsByUser` | relationship-v2 | **folder-server** |
| `getRoles` | relationship-v2 | **folder-server** |
| `getPermissions` | relationship-v2 | **folder-server** |
| `programRole` | relationship-v2 | **folder-server** |
| `programRolesForProgram` | relationship-v2 | **folder-server** |

### Query Flow Example

```mermaid
sequenceDiagram
    participant Client as Platform Shell
    participant Router as Apollo Router
    participant Folder as folder-server
    participant Mongo as MongoDB
    participant FGA as OpenFGA

    Client->>Router: query { campuses { id, name } }
    Router->>Folder: campuses (routed by schema)

    Folder->>FGA: check(user, can_view, tenant)
    FGA-->>Folder: ✅ allowed

    Folder->>Mongo: find({ folderTypeKey: 'campus' })
    Mongo-->>Folder: [{ id, name, ... }]

    Folder-->>Router: [Campus!]
    Router-->>Client: { campuses: [...] }
```

---

## Mutation Changes

**All tenant, campus, and program mutations are removed from relationship-v2** and implemented in folder-server. The new mutations use the folder-server API with dual-write to Neptune.

```mermaid
flowchart TB
    subgraph Client["Platform Shell"]
        Mutation["GraphQL Mutation"]
    end

    subgraph Router["Apollo Router"]
        Route["Route by Type Owner"]
    end

    subgraph FolderServer["folder-server"]
        CM["createFolder / upsertSystemFolder"]
        UM["updateFolder"]
        DM["archiveFolder"]
        RM["assignRole"]
        RMR["removeRole"]
    end

    subgraph RelV2["relationship-v2"]
        OMut["Other mutations..."]
    end

    subgraph Databases["Data Stores"]
        MongoDB[("MongoDB")]
        Neptune[("Neptune")]
        FGA[("OpenFGA")]
    end

    Mutation --> Route
    Route -->|"Folder & Role operations"| FolderServer
    Route -->|"Other"| RelV2

    FolderServer --> MongoDB
    FolderServer --> Neptune
    FolderServer --> FGA
    RelV2 --> Neptune

    style FolderServer fill:#99ff99
    style MongoDB fill:#99ff99
```

### Mutation Changes Table

| Old Mutation (relationship-v2) | New Mutation (folder-server) | Notes |
|-------------------------------|------------------------------|-------|
| `createTenant` | `upsertSystemFolder` | folderTypeKey: "tenant" |
| `createCampus` | `upsertSystemFolder` | folderTypeKey: "campus" |
| `createProgram` | `createFolder` | folderTypeKey: "program" |
| `updateProgram` | `updateFolder` | |
| `createRole` | `assignRole` | |
| `addRoleMember` | `assignRole` | |
| `removeRoleMember` | `removeRole` | |
| `createProgramRole` | `assignRole` | scopeType: FOLDER |
| `addMemberToProgramRole` | `assignRole` | |
| `removeMemberFromProgramRole` | `removeRole` | |
| ~~`createDomain`~~ | *removed* | Domain entity deprecated |

### Mutation Flow with Dual-Write

When folder-server receives a mutation for tenant/campus/program, it writes to both MongoDB and Neptune to maintain backward compatibility.

```mermaid
sequenceDiagram
    participant Client as Platform Shell
    participant Router as Apollo Router
    participant Folder as folder-server
    participant Mongo as MongoDB
    participant Neptune as Neptune
    participant FGA as OpenFGA

    Client->>Router: mutation { upsertSystemFolder(input: { folderTypeKey: "campus", ... }) }
    Router->>Folder: upsertSystemFolder (routed)

    Folder->>FGA: authorize(user, configuration_manager, tenant)
    FGA-->>Folder: ✅ allowed

    Folder->>Mongo: upsert({ folderTypeKey: "campus", ... })
    Mongo-->>Folder: success

    Folder->>Neptune: CREATE (:CAMPUS {...})
    Neptune-->>Folder: success

    Folder->>FGA: write([folder:parent → parent → folder:new])
    FGA-->>Folder: success

    Folder-->>Router: Folder
    Router-->>Client: { upsertSystemFolder: { ... } }
```

---

## Dual-Write Strategy

folder-server writes to both MongoDB (primary) and Neptune (backward compatibility) for all tenant, campus, and program operations.

```mermaid
flowchart LR
    subgraph FolderServer["folder-server"]
        Service["Folder Service"]
    end

    subgraph Primary["Primary Store"]
        MongoDB[("MongoDB")]
    end

    subgraph Legacy["Legacy Store (Dual-Write)"]
        Neptune[("Neptune")]
    end

    subgraph Auth["Authorization"]
        FGA[("OpenFGA")]
    end

    Service -->|"1. Write"| MongoDB
    Service -->|"2. Dual-Write"| Neptune
    Service -->|"3. Write Tuples"| FGA

    style MongoDB fill:#99ff99
    style Neptune fill:#ffff99
```

**Why Dual-Write:**
- Other services may still query Neptune directly
- Gradual migration - can disable Neptune writes once all consumers migrate
- Rollback capability - Neptune remains in sync

**Dual-Write Rules:**
1. MongoDB write happens first (primary)
2. Neptune write is best-effort (failure logged but doesn't fail the operation)
3. FGA tuples written after both stores succeed

---

## OpenFGA Authorization

Since all entity mutations (tenant, campus, program) and role mutations move to folder-server, **folder-server becomes the sole writer to OpenFGA**. relationship-v2 no longer writes any FGA tuples.

```mermaid
flowchart TB
    subgraph Services["Services"]
        Folder["folder-server"]
        RelV2["relationship-v2"]
    end

    subgraph FGA["OpenFGA"]
        Tuples["Authorization Tuples"]
    end

    subgraph TupleTypes["Tuple Types"]
        Hierarchy["Hierarchy: folder:x → parent → folder:y"]
        Roles["Roles: user:x → member → folder_role:y"]
    end

    Folder -->|"all tuples"| Tuples
    RelV2 -.->|"no writes"| Tuples

    Tuples --> Hierarchy
    Tuples --> Roles

    style Folder fill:#99ff99
    style RelV2 fill:#ffcccc,stroke-dasharray: 5 5
```

### FGA Tuple Examples

| Operation | Service | FGA Tuple Written |
|-----------|---------|-------------------|
| Create tenant folder | folder-server | `folder:global-id` → `parent` → `folder:tenant-id` |
| Create campus folder | folder-server | `folder:tenant-id` → `parent` → `folder:campus-id` |
| Create program folder | folder-server | `folder:campus-id` → `parent` → `folder:program-id` |
| Assign role to user | folder-server | `user:123` → `member` → `folder_role:xyz` |

---

## Entity Mapping

System folder types map to legacy Neptune entities:

| Neptune Entity | Folder Type | Notes |
|----------------|-------------|-------|
| `:GLOBAL` | `global` | Root of hierarchy, empty parents |
| `:TENANT` | `tenant` | Child of global |
| `:CAMPUS` | `campus` | Child of tenant |
| `:PROGRAM` | `program` | Child of campus (or domain) |
| `:DOMAIN` | *deprecated* | Not migrated |

### Campus Example

```mermaid
flowchart LR
    subgraph Neptune["Neptune"]
        CN["(:CAMPUS {<br/>id: 'usc',<br/>name: 'UC Santa Cruz',<br/>shortName: 'UCSC'<br/>})"]
    end

    subgraph MongoDB["MongoDB"]
        CM["{<br/>id: 'usc',<br/>folderTypeKey: 'campus',<br/>name: 'UC Santa Cruz',<br/>parents: ['uc'],<br/>properties: { shortName: 'UCSC' }<br/>}"]
    end

    CN <-->|"Dual-Write Sync"| CM
```

---

## System Folder Hierarchy

```mermaid
flowchart TB
    subgraph SystemTypes["System Folder Types"]
        Global["global<br/>(root)"]
        Tenant["tenant"]
        Campus["campus"]
    end

    subgraph UserTypes["User-Defined Types"]
        Program["program"]
        Lab["lab"]
        Custom["..."]
    end

    Global --> Tenant
    Tenant --> Campus
    Campus --> Program
    Campus --> Lab
    Program --> Custom

    style Global fill:#ffcccc
    style Tenant fill:#ffcccc
    style Campus fill:#ffcccc
```

**Rules:**
- System folders use `upsertSystemFolder` mutation
- Only `global` can have empty `parents` array
- System folders cannot be archived/deleted via folder-server

---

## Open Questions

1. **Neptune Deprecation Timeline**: When can we stop dual-writing to Neptune?
2. **Direct Neptune Queries**: Which services query Neptune directly and need migration?
3. **FGA Model Unification**: Should legacy tuples (`tenant:x`) migrate to `folder:x` format?

---

## References

- [Migration Spec](../spec/06-relationshipv2-migration-spec.md)
- [System Folder Types Spec](../spec/archive/system-folder-types-spec.md)
- [OpenFGA Documentation](https://openfga.dev/docs)
- [Apollo Federation Documentation](https://www.apollographql.com/docs/federation/)

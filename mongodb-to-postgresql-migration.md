---
tags: [databases]
date: 2024-12-22
status: complete
---

# MongoDB to PostgreSQL Migration with Drizzle ORM - Research

**Date**: 2025-12-17
**Status**: Research Complete

## Table of Contents

- [Executive Summary](#executive-summary)
- [Technical Deep Dive](#technical-deep-dive)
- [Technology Stack / Ecosystem](#technology-stack--ecosystem)
- [Codebase Analysis](#codebase-analysis)
- [Implementation Feasibility](#implementation-feasibility)
- [Implementation Options](#implementation-options)
- [Comparison Matrix](#comparison-matrix)
- [Implementation Approach](#implementation-approach)
- [Alternatives Considered](#alternatives-considered)
- [Debates & Open Questions](#debates--open-questions)
- [Recommendations](#recommendations)
- [Additional Notes](#additional-notes)
- [Sources](#sources)

## Executive Summary

This research evaluates migrating folder-server from MongoDB to PostgreSQL using Drizzle ORM, analyzing two approaches: **Direct Replacement** (complete switchover) and **Dual Database Support** (feature-flag controlled gradual migration). Given the relatively small codebase (3 repositories, ~500 lines of repository code), the **Direct Replacement** approach is recommended as the simpler, lower-maintenance option that avoids the ongoing complexity of maintaining two database implementations.

## Technical Deep Dive

### Overview

The folder-server uses MongoDB with the native driver for persistence, storing three entity types: folders, folder_types, and roles. The migration to PostgreSQL involves schema transformation from document-oriented to relational design, replacing MongoDB operations with Drizzle ORM queries, and adapting transaction patterns.

### Current MongoDB Architecture

The existing implementation uses:

- **MongoDB Native Driver v6.12.0** with `@fastify/mongodb` plugin
- **Zod schemas** generated from GraphQL with MongoDB `_id` transformation
- **ClientSession** for transaction management
- **Document-oriented storage** with nested objects and arrays

```mermaid
flowchart TB
    subgraph GraphQL["GraphQL Layer"]
        Resolvers[Resolvers]
    end

    subgraph Service["Service Layer"]
        FolderService[folder.service.ts]
        FolderTypeService[folder-type.service.ts]
        RoleService[role.service.ts]
    end

    subgraph Repository["Repository Layer"]
        FolderRepo[folder.repository.ts]
        FolderTypeRepo[folder-type.repository.ts]
        RoleRepo[role.repository.ts]
    end

    subgraph MongoDB["MongoDB"]
        FoldersCollection[(folders)]
        FolderTypesCollection[(folder_types)]
        RolesCollection[(roles)]
    end

    Resolvers --> FolderService & FolderTypeService & RoleService
    FolderService --> FolderRepo
    FolderTypeService --> FolderTypeRepo
    RoleService --> RoleRepo
    FolderRepo --> FoldersCollection
    FolderTypeRepo --> FolderTypesCollection
    RoleRepo --> RolesCollection
```

### MongoDB Operations to Migrate

| MongoDB Operation | PostgreSQL/Drizzle Equivalent |
|-------------------|-------------------------------|
| `findOne({ _id: id })` | `db.select().from(table).where(eq(table.id, id))` |
| `find({ field: value })` | `db.select().from(table).where(eq(table.field, value))` |
| `insertOne(document)` | `db.insert(table).values(data).returning()` |
| `findOneAndUpdate()` | `db.update(table).set(data).where(...).returning()` |
| `deleteOne()` | `db.delete(table).where(...)` |
| `$addToSet: { array: item }` | Custom SQL with `array_append()` or update full array |
| `$pull: { array: item }` | Custom SQL with `array_remove()` or update full array |
| `{ parents: folderId }` (array contains) | `arrayContains(table.parents, [folderId])` |
| `{ $in: ancestorIds }` | `inArray(table.scopeFolderId, ancestorIds)` |
| `countDocuments()` | `db.select({ count: count() }).from(table).where(...)` |

### Transaction Pattern Migration

**MongoDB Pattern:**
```typescript
const session = ctx.mongoClient.startSession();
try {
  session.startTransaction();
  // operations with session parameter
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  await session.endSession();
}
```

**Drizzle Pattern:**
```typescript
await db.transaction(async (tx) => {
  // operations using tx instead of db
  // automatic rollback on error
});
```

```mermaid
sequenceDiagram
    participant Service as Service Layer
    participant DB as Drizzle ORM
    participant PG as PostgreSQL
    participant FGA as OpenFGA

    Service->>DB: db.transaction(async (tx) => {...})
    activate DB
    DB->>PG: BEGIN

    Service->>DB: tx.insert(folders).values(...)
    DB->>PG: INSERT INTO folders...
    PG-->>DB: Folder record

    Service->>FGA: openFga.write(tuples)
    FGA-->>Service: Success

    alt Success
        DB->>PG: COMMIT
        PG-->>DB: OK
        DB-->>Service: Return result
    else Error
        DB->>PG: ROLLBACK
        PG-->>DB: OK
        DB-->>Service: Throw error
    end
    deactivate DB
```

### Schema Transformation

**Document Model (MongoDB):**
```typescript
// Folder document structure
{
  _id: "uuid",
  tenantId: "uuid",
  parents: ["uuid1", "uuid2"],        // Array field
  folderTypeKey: "string",
  name: "string",
  description: "string",
  properties: { /* dynamic JSON */ }, // Flexible schema
  status: "ACTIVE" | "ARCHIVED",
  meta: {                             // Nested document
    createdBy: { id, name, email },
    createdDate: Date,
    lastUpdatedBy: { id, name, email },
    lastUpdatedDate: Date
  }
}
```

**Relational Model (PostgreSQL with Drizzle):**
```typescript
export const folders = pgTable('folders', {
  id: uuid('id').primaryKey(),
  tenantId: uuid('tenant_id').notNull(),
  parents: uuid('parents').array().notNull().default(sql`'{}'::uuid[]`),
  folderTypeKey: varchar('folder_type_key', { length: 255 }).notNull(),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  properties: jsonb('properties').$type<Record<string, unknown>>().default({}),
  status: folderStatusEnum('status').notNull().default('ACTIVE'),
  // Flattened metadata
  createdById: varchar('created_by_id', { length: 255 }).notNull(),
  createdByName: varchar('created_by_name', { length: 255 }).notNull(),
  createdByEmail: varchar('created_by_email', { length: 255 }).notNull(),
  createdDate: timestamp('created_date', { withTimezone: true }).notNull(),
  lastUpdatedById: varchar('last_updated_by_id', { length: 255 }).notNull(),
  lastUpdatedByName: varchar('last_updated_by_name', { length: 255 }).notNull(),
  lastUpdatedByEmail: varchar('last_updated_by_email', { length: 255 }).notNull(),
  lastUpdatedDate: timestamp('last_updated_date', { withTimezone: true }).notNull(),
}, (table) => [
  index('folders_tenant_id_idx').on(table.tenantId),
  index('folders_folder_type_key_idx').on(table.folderTypeKey),
  index('folders_parents_idx').using('gin', table.parents),
]);
```

### JSONB Handling for Dynamic Fields

The `properties` and `propertySchema` fields require JSONB columns:

```typescript
// Type-safe JSONB with Drizzle
properties: jsonb('properties').$type<Record<string, unknown>>().default({}),
propertySchema: jsonb('property_schema').$type<JsonSchema>().notNull().default({
  type: 'object',
  properties: {}
}),
uiSchema: jsonb('ui_schema').$type<UiSchema | null>(),
```

**Querying JSONB:**
```typescript
// Check if JSONB field exists
db.select().from(folderTypes)
  .where(sql`${folderTypes.propertySchema}->>'type' = 'object'`);

// GIN index for JSONB queries
index('folder_types_property_schema_idx')
  .using('gin', table.propertySchema);
```

### Array Column Handling

PostgreSQL arrays replace MongoDB arrays:

```typescript
// Schema definition
parents: uuid('parents').array().notNull().default(sql`'{}'::uuid[]`),
allowedSubFolderTypes: text('allowed_sub_folder_types').array().notNull().default(sql`'{}'::text[]`),

// Query: find folders where parents array contains a specific ID
db.select().from(folders)
  .where(arrayContains(folders.parents, [parentId]));

// Update: append to array
db.update(folders)
  .set({
    parents: sql`array_append(${folders.parents}, ${parentId}::uuid)`,
    lastUpdatedDate: new Date()
  })
  .where(eq(folders.id, folderId));

// Update: remove from array
db.update(folders)
  .set({
    parents: sql`array_remove(${folders.parents}, ${parentId}::uuid)`,
    lastUpdatedDate: new Date()
  })
  .where(eq(folders.id, folderId));
```

## Technology Stack / Ecosystem

### Required Dependencies

**Add:**
```json
{
  "drizzle-orm": "^0.40.0",
  "postgres": "^3.4.7"
}
```

**Dev Dependencies:**
```json
{
  "drizzle-kit": "^0.30.0",
  "@testcontainers/postgresql": "^10.23.0"
}
```

**Remove:**
```json
{
  "@fastify/mongodb": "^9.0.0",
  "mongodb": "^6.12.0",
  "mongodb-memory-server": "^10.0.0"
}
```

### Bun SQL Native Support

Drizzle ORM natively supports Bun's SQL module for PostgreSQL:

```typescript
import { drizzle } from 'drizzle-orm/bun-sql';
import { SQL } from 'bun';

const client = new SQL(process.env.DATABASE_URL!);
const db = drizzle({ client });
```

This provides optimal performance with Bun's native PostgreSQL bindings.

### Testing Infrastructure

Replace `mongodb-memory-server` with `@testcontainers/postgresql`:

```typescript
import { PostgreSqlContainer } from '@testcontainers/postgresql';

let container: StartedTestContainer;

beforeAll(async () => {
  container = await new PostgreSqlContainer('postgres:16')
    .withDatabase('test_db')
    .start();

  const db = drizzle({
    connection: container.getConnectionUri()
  });

  await migrate(db, { migrationsFolder: './drizzle' });
});

afterAll(async () => {
  await container.stop();
});
```

## Codebase Analysis

### Repository Files to Migrate

| File | Lines | Complexity | Key Challenges |
|------|-------|------------|----------------|
| `folder.repository.ts` | ~260 | High | Array operations, DAG traversal, transactions |
| `folder-type.repository.ts` | ~165 | Medium | JSONB fields, `$in` queries |
| `role.repository.ts` | ~78 | Low | Standard CRUD |

### Key MongoDB Patterns Found

1. **Array Contains Query** (`folder.repository.ts:46-47`)
   ```typescript
   const filter = { parents: folderId };  // MongoDB implicit array contains
   ```

2. **$addToSet Operation** (`folder.repository.ts:194-204`)
   ```typescript
   { $addToSet: { parents: parentId } }
   ```

3. **$pull Operation** (`folder.repository.ts:224-227`)
   ```typescript
   { $pull: { parents: parentId } }
   ```

4. **$in Query** (`folder-type.repository.ts:89`)
   ```typescript
   { scopeFolderId: { $in: ancestorIds } }
   ```

5. **Nested Document Update** (`folder.repository.ts:168-172`)
   ```typescript
   { "meta.lastUpdatedBy": user, "meta.lastUpdatedDate": new Date() }
   ```

### Service Layer Transaction Usage

Transactions are used in:
- `createFolder()` - folder creation + OpenFGA writes
- `deleteFolder()` - folder deletion + OpenFGA deletes
- `addFolderParent()` - parent addition + OpenFGA writes
- `removeFolderParent()` - parent removal + OpenFGA deletes

The service layer pattern is clean and would work identically with Drizzle transactions.

### Schema Files Impact

| File | Changes Required |
|------|------------------|
| `folder.schema.ts` | Remove `transformMongoId`, adapt to Drizzle types |
| `folder-type.schema.ts` | Remove `transformMongoId`, adapt to Drizzle types |
| `role.schema.ts` | Remove `transformMongoId`, adapt to Drizzle types |

The `_id` to `id` transformation layer becomes unnecessary with PostgreSQL since we'd use `id` directly.

### Critical Files to Review

1. `src/graphql/context.ts` - Update `ServiceContext` to use Drizzle db instead of MongoDB
2. `src/server.ts` - Replace `@fastify/mongodb` plugin with Drizzle setup
3. `src/lib/mongodb/transform-id.ts` - Can be deleted
4. All `*.spec.ts` files - Replace mongodb-memory-server with testcontainers

## Implementation Feasibility

### Benefits

**Direct Replacement:**
- Single codebase to maintain
- No interface abstraction overhead
- Simpler mental model for developers
- Cleaner architecture long-term
- ~30% less code than dual-database approach
- Drizzle's type safety catches errors at compile time

**PostgreSQL Advantages:**
- ACID compliance with robust transaction support
- Mature JSON/JSONB support with indexing
- Better query optimization and explain plans
- Rich ecosystem of monitoring tools
- Horizontal read scaling with replicas
- GIN indexes for array/JSONB operations

### Trade-offs & Challenges

**Migration Complexity:**
- One-time data migration required
- Schema flattening for nested documents (meta object)
- Array operation syntax changes
- DAG traversal queries need optimization

**JSONB Limitations:**
- Runtime validation still needed (Ajv for propertySchema)
- JSONB queries less elegant than MongoDB document queries
- Type inference is compile-time only, not runtime

**Testing Infrastructure:**
- Docker required for testcontainers (vs in-memory MongoDB)
- Slightly slower test startup (~2-3s for container vs ~1s for mongodb-memory-server)
- CI/CD needs Docker-in-Docker or privileged mode

### When to Use

- Production-ready relational database needed
- ACID transactions are critical (already using them)
- Team has SQL/PostgreSQL experience
- Need for complex queries and joins in future
- Want to consolidate on a single database technology

### When to Avoid

- Schema changes extremely frequently and unpredictably
- Deep nesting of documents is core to the model
- Need for horizontal write scaling (though read replicas work)
- No SQL expertise on team

## Implementation Options

### Option 1: Direct Replacement (Recommended)

**Description**: Create a new branch, completely replace MongoDB with PostgreSQL/Drizzle, migrate data, and deploy.

**Pros:**
- Single codebase - no abstraction layer overhead
- Simpler architecture going forward
- No ongoing dual-maintenance burden
- Clean separation of concerns
- Type-safe from edge to database
- Fastest path to production stability

**Cons:**
- Requires data migration downtime (or blue-green deployment)
- All-or-nothing deployment
- Rollback requires database restore

**Complexity**: Medium

**Time Estimate**: 5-7 days

**Reuses Patterns**: Partial - service layer unchanged, repository layer rewritten

**When to Use:**
- Small dataset (< 1M records)
- Can tolerate maintenance window
- Team prefers simpler codebase

**Implementation Outline:**

```mermaid
flowchart TD
    A[Create feature branch] --> B[Define Drizzle schemas]
    B --> C[Rewrite repositories with Drizzle]
    C --> D[Update service layer context]
    D --> E[Migrate tests to testcontainers]
    E --> F[Write data migration script]
    F --> G[Test in staging]
    G --> H{Tests pass?}
    H -->|Yes| I[Deploy with maintenance window]
    H -->|No| J[Fix issues]
    J --> G
    I --> K[Run data migration]
    K --> L[Verify data integrity]
    L --> M[Switch traffic]
```

### Option 2: Dual Database Support (Feature Flag)

**Description**: Support both MongoDB and PostgreSQL simultaneously with an environment variable controlling which is active. Migrate gradually using the six-stage migration pattern.

**Pros:**
- Zero-downtime migration possible
- Gradual rollout with instant rollback
- Can run shadow traffic to new database
- Lower risk per deployment
- Validates data consistency continuously

**Cons:**
- Double the repository code to maintain
- Abstraction layer adds complexity
- Longer total project duration
- Higher cognitive load for developers
- Feature flag complexity in testing

**Complexity**: High

**Time Estimate**: 2-3 weeks

**Reuses Patterns**: Yes - service layer unchanged

**When to Use:**
- Large dataset requiring zero downtime
- High-traffic system requiring shadow testing
- Team resources available for extended migration

**Architecture:**

```mermaid
flowchart TB
    subgraph Service["Service Layer (Unchanged)"]
        FolderService[folder.service.ts]
    end

    subgraph Interface["Repository Interface"]
        IFolderRepo[IFolderRepository]
    end

    subgraph MongoDB["MongoDB Implementation"]
        MongoFolderRepo[MongoFolderRepository]
        MongoDb[(MongoDB)]
    end

    subgraph PostgreSQL["PostgreSQL Implementation"]
        PgFolderRepo[PgFolderRepository]
        PgDb[(PostgreSQL)]
    end

    subgraph Factory["Repository Factory"]
        RepoFactory{DB_TYPE env var}
    end

    FolderService --> IFolderRepo
    IFolderRepo --> RepoFactory
    RepoFactory -->|mongodb| MongoFolderRepo
    RepoFactory -->|postgresql| PgFolderRepo
    MongoFolderRepo --> MongoDb
    PgFolderRepo --> PgDb
```

**Six-Stage Migration Flow:**

```mermaid
stateDiagram-v2
    [*] --> Off: Initial State
    Off --> Dualwrite: Begin Migration
    Dualwrite --> Shadow: Data Sync Complete
    Shadow --> Live: New DB Validated
    Live --> Rampdown: Production Stable
    Rampdown --> Complete: Legacy Deprecated
    Complete --> [*]: Migration Done

    Off: Read/Write MongoDB only
    Dualwrite: Read MongoDB, Write Both
    Shadow: Read/Write Both, Return MongoDB
    Live: Read/Write Both, Return PostgreSQL
    Rampdown: Read PostgreSQL, Write Both
    Complete: Read/Write PostgreSQL only
```

### Option 3: Hybrid - Repository Abstraction Without Dual Support

**Description**: Create repository interfaces but only implement PostgreSQL. This provides future flexibility without current dual-maintenance cost.

**Pros:**
- Clean interfaces for testing/mocking
- Future database changes easier
- No dual implementation burden

**Cons:**
- More abstraction than strictly needed
- Slightly more boilerplate code
- YAGNI concern

**Complexity**: Medium

**Time Estimate**: 6-8 days

**Reuses Patterns**: Partial

## Comparison Matrix

| Criteria | Option 1: Direct | Option 2: Dual DB | Option 3: Hybrid |
|----------|------------------|-------------------|------------------|
| **Complexity** | Medium | High | Medium |
| **Implementation Time** | 5-7 days | 2-3 weeks | 6-8 days |
| **Maintenance Burden** | Low | High | Low |
| **Migration Risk** | Medium | Low | Medium |
| **Rollback Capability** | DB Restore | Instant (feature flag) | DB Restore |
| **Testing Complexity** | Low | High (both paths) | Low |
| **Code to Write** | ~800 lines | ~1600 lines | ~900 lines |
| **Future Flexibility** | Low | Medium | High |
| **Team Learning Curve** | Low | High | Medium |
| **Downtime Required** | 15-30 min | Zero | 15-30 min |

## Implementation Approach

### Prerequisites & Requirements

**Technical Requirements:**
- PostgreSQL 16+ (for latest JSONB features)
- Docker (for testcontainers)
- Bun 1.2+ (for native SQL support)

**Knowledge Requirements:**
- Basic SQL and PostgreSQL understanding
- Drizzle ORM fundamentals
- Understanding of current codebase patterns

**Environment Setup:**
```bash
# Add PostgreSQL for local development
docker run -d \
  --name folder-postgres \
  -e POSTGRES_USER=folder \
  -e POSTGRES_PASSWORD=folder \
  -e POSTGRES_DB=folder \
  -p 5432:5432 \
  postgres:16

# Install dependencies
bun add drizzle-orm postgres
bun add -D drizzle-kit @testcontainers/postgresql
```

### Getting Started

1. **Create Drizzle Config** (`drizzle.config.ts`):
```typescript
import type { Config } from 'drizzle-kit';

export default {
  schema: './src/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: {
    url: process.env.DATABASE_URL!,
  },
} satisfies Config;
```

2. **Define Schema** (`src/db/schema.ts`):
```typescript
import { pgTable, uuid, varchar, text, timestamp, jsonb, pgEnum, index } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const folderStatusEnum = pgEnum('folder_status', ['ACTIVE', 'ARCHIVED']);
export const scopeTypeEnum = pgEnum('scope_type', ['FOLDER', 'LIBRARY', 'INVENTORY']);

export const folders = pgTable('folders', {
  id: uuid('id').primaryKey(),
  tenantId: uuid('tenant_id').notNull(),
  parents: uuid('parents').array().notNull().default(sql`'{}'::uuid[]`),
  folderTypeKey: varchar('folder_type_key', { length: 255 }).notNull(),
  name: varchar('name', { length: 255 }).notNull(),
  description: text('description'),
  properties: jsonb('properties').$type<Record<string, unknown>>().default({}),
  status: folderStatusEnum('status').notNull().default('ACTIVE'),
  createdById: varchar('created_by_id', { length: 255 }).notNull(),
  createdByName: varchar('created_by_name', { length: 255 }).notNull(),
  createdByEmail: varchar('created_by_email', { length: 255 }).notNull(),
  createdDate: timestamp('created_date', { withTimezone: true }).notNull(),
  lastUpdatedById: varchar('last_updated_by_id', { length: 255 }).notNull(),
  lastUpdatedByName: varchar('last_updated_by_name', { length: 255 }).notNull(),
  lastUpdatedByEmail: varchar('last_updated_by_email', { length: 255 }).notNull(),
  lastUpdatedDate: timestamp('last_updated_date', { withTimezone: true }).notNull(),
}, (table) => [
  index('folders_tenant_id_idx').on(table.tenantId),
  index('folders_folder_type_key_idx').on(table.folderTypeKey),
  index('folders_status_idx').on(table.status),
  index('folders_parents_idx').using('gin', table.parents),
]);

export const folderTypes = pgTable('folder_types', {
  id: uuid('id').primaryKey(),
  scopeFolderId: varchar('scope_folder_id', { length: 255 }).notNull(),
  key: varchar('key', { length: 255 }).notNull(),
  label: varchar('label', { length: 255 }).notNull(),
  description: text('description'),
  allowedSubFolderTypes: text('allowed_sub_folder_types').array().notNull().default(sql`'{"*"}'::text[]`),
  roleTemplates: jsonb('role_templates').$type<RoleTemplate[]>().default([]),
  propertySchema: jsonb('property_schema').$type<JsonSchema>().notNull().default({ type: 'object', properties: {} }),
  uiSchema: jsonb('ui_schema').$type<UiSchema | null>(),
  isArchived: boolean('is_archived').notNull().default(false),
  // Metadata fields (flattened)
  createdById: varchar('created_by_id', { length: 255 }).notNull(),
  createdByName: varchar('created_by_name', { length: 255 }).notNull(),
  createdByEmail: varchar('created_by_email', { length: 255 }).notNull(),
  createdDate: timestamp('created_date', { withTimezone: true }).notNull(),
  lastUpdatedById: varchar('last_updated_by_id', { length: 255 }).notNull(),
  lastUpdatedByName: varchar('last_updated_by_name', { length: 255 }).notNull(),
  lastUpdatedByEmail: varchar('last_updated_by_email', { length: 255 }).notNull(),
  lastUpdatedDate: timestamp('last_updated_date', { withTimezone: true }).notNull(),
}, (table) => [
  index('folder_types_scope_folder_id_idx').on(table.scopeFolderId),
  index('folder_types_key_idx').on(table.key),
  uniqueIndex('folder_types_scope_key_unique').on(table.scopeFolderId, table.key),
]);

export const roles = pgTable('roles', {
  id: uuid('id').primaryKey(),
  scopeType: scopeTypeEnum('scope_type').notNull(),
  scopeId: uuid('scope_id').notNull(),
  userId: varchar('user_id', { length: 255 }).notNull(),
  key: varchar('key', { length: 255 }).notNull(),
  label: varchar('label', { length: 255 }).notNull(),
  permissions: text('permissions').array().notNull().default(sql`'{}'::text[]`),
  // Metadata fields
  createdById: varchar('created_by_id', { length: 255 }).notNull(),
  createdByName: varchar('created_by_name', { length: 255 }).notNull(),
  createdByEmail: varchar('created_by_email', { length: 255 }).notNull(),
  createdDate: timestamp('created_date', { withTimezone: true }).notNull(),
  lastUpdatedById: varchar('last_updated_by_id', { length: 255 }).notNull(),
  lastUpdatedByName: varchar('last_updated_by_name', { length: 255 }).notNull(),
  lastUpdatedByEmail: varchar('last_updated_by_email', { length: 255 }).notNull(),
  lastUpdatedDate: timestamp('last_updated_date', { withTimezone: true }).notNull(),
}, (table) => [
  index('roles_scope_idx').on(table.scopeType, table.scopeId),
  index('roles_user_id_idx').on(table.userId),
]);
```

3. **Generate Initial Migration:**
```bash
bunx drizzle-kit generate
```

### Architecture & Design Considerations

**Database Connection:**
```typescript
// src/db/index.ts
import { drizzle } from 'drizzle-orm/bun-sql';
import { SQL } from 'bun';
import * as schema from './schema';

export function createDb(connectionString: string) {
  const client = new SQL(connectionString);
  return drizzle({ client, schema });
}

export type Database = ReturnType<typeof createDb>;
```

**Context Update:**
```typescript
// src/graphql/context.ts
export interface ServiceContext {
  auth: AuthUser;
  db: Database;        // Changed from Db (MongoDB)
  openFga: OpenFgaClient;
  // mongoClient removed
}
```

**Repository Example (folder.repository.ts):**
```typescript
import { eq, and, inArray, arrayContains, sql, count } from 'drizzle-orm';
import type { Database } from '../../db';
import { folders } from '../../db/schema';

export async function getFolderById(id: string, db: Database): Promise<Folder | null> {
  const results = await db
    .select()
    .from(folders)
    .where(eq(folders.id, id));

  return results[0] ?? null;
}

export async function getSubFolders(
  folderId: string,
  db: Database,
  options: GetFoldersOptions = {},
): Promise<Folder[]> {
  let query = db
    .select()
    .from(folders)
    .where(arrayContains(folders.parents, [folderId]));

  if (options.folderTypeKey) {
    query = query.where(eq(folders.folderTypeKey, options.folderTypeKey));
  }
  if (options.status) {
    query = query.where(eq(folders.status, options.status));
  }

  return query.orderBy(folders.name);
}

export async function addParentToFolder(
  id: string,
  parentId: string,
  auth: AuthUser,
  db: Database,
): Promise<Folder | null> {
  const user = getUserMetadata(auth);

  const results = await db
    .update(folders)
    .set({
      parents: sql`array_append(${folders.parents}, ${parentId}::uuid)`,
      lastUpdatedById: user.id,
      lastUpdatedByName: user.name,
      lastUpdatedByEmail: user.email,
      lastUpdatedDate: new Date(),
    })
    .where(eq(folders.id, id))
    .returning();

  return results[0] ?? null;
}
```

### Best Practices

**1. Use Transactions for Multi-Step Operations:**
```typescript
await db.transaction(async (tx) => {
  const folder = await createFolderInDb(input, auth, tenantId, tx);

  // If FGA write fails, DB changes are rolled back
  await ctx.openFga.write(tuples);

  return folder;
});
```

**2. Index Strategy:**
- B-tree indexes for equality/range queries (default)
- GIN indexes for array and JSONB columns
- Composite indexes for multi-column filters

**3. JSONB Querying:**
```typescript
// Prefer indexed paths over deep nesting
db.select().from(folderTypes)
  .where(sql`${folderTypes.propertySchema}->>'type' = 'object'`);
```

**4. Array Operations:**
```typescript
// Use PostgreSQL array functions
sql`array_append(${folders.parents}, ${id}::uuid)`
sql`array_remove(${folders.parents}, ${id}::uuid)`
sql`array_length(${folders.parents}, 1)`
```

### Common Pitfalls & How to Avoid Them

1. **N+1 Queries in DAG Traversal**
   - The `getAncestorChain` and `getAllDescendants` functions make multiple queries
   - Consider using PostgreSQL recursive CTEs for better performance:
   ```sql
   WITH RECURSIVE ancestors AS (
     SELECT id, parents FROM folders WHERE id = $1
     UNION ALL
     SELECT f.id, f.parents FROM folders f
     INNER JOIN ancestors a ON f.id = ANY(a.parents)
   )
   SELECT id FROM ancestors;
   ```

2. **Type Mismatch with JSONB**
   - `.$type<T>()` is compile-time only
   - Continue using Zod/Ajv for runtime validation of `properties`

3. **Array Null vs Empty**
   - Default arrays to `'{}'::type[]` not NULL
   - Use `coalesce()` when aggregating to avoid null

4. **Transaction Isolation**
   - Default is READ COMMITTED
   - For critical operations, consider SERIALIZABLE:
   ```typescript
   await db.transaction(async (tx) => { ... }, {
     isolationLevel: 'serializable'
   });
   ```

### Testing Strategy

**Unit Tests:**
- Use the same patterns, just with Drizzle's in-memory mock (limited) or testcontainers

**Integration Tests:**
```typescript
import { PostgreSqlContainer } from '@testcontainers/postgresql';
import { drizzle } from 'drizzle-orm/bun-sql';
import { migrate } from 'drizzle-orm/bun-sql/migrator';

describe('Folder Repository', () => {
  let container: StartedTestContainer;
  let db: Database;

  beforeAll(async () => {
    container = await new PostgreSqlContainer('postgres:16')
      .withDatabase('test')
      .start();

    db = drizzle({ connection: container.getConnectionUri() });
    await migrate(db, { migrationsFolder: './drizzle' });
  });

  afterAll(async () => {
    await container.stop();
  });

  beforeEach(async () => {
    // Clean tables between tests
    await db.delete(folders);
    await db.delete(folderTypes);
    await db.delete(roles);
  });
});
```

### Data Migration Script

```typescript
// scripts/migrate-mongodb-to-postgres.ts
import { MongoClient } from 'mongodb';
import { drizzle } from 'drizzle-orm/bun-sql';
import { SQL } from 'bun';
import { folders, folderTypes, roles } from '../src/db/schema';

async function migrate() {
  const mongoClient = await MongoClient.connect(process.env.MONGODB_URL!);
  const mongodb = mongoClient.db('folder');

  const pgClient = new SQL(process.env.DATABASE_URL!);
  const db = drizzle({ client: pgClient });

  // Migrate folder_types
  console.log('Migrating folder_types...');
  const folderTypeDocs = await mongodb.collection('folder_types').find().toArray();
  for (const doc of folderTypeDocs) {
    await db.insert(folderTypes).values({
      id: doc._id,
      scopeFolderId: doc.scopeFolderId,
      key: doc.key,
      label: doc.label,
      description: doc.description,
      allowedSubFolderTypes: doc.allowedSubFolderTypes,
      roleTemplates: doc.roleTemplates,
      propertySchema: doc.propertySchema,
      uiSchema: doc.uiSchema,
      isArchived: doc.isArchived,
      createdById: doc.meta.createdBy.id,
      createdByName: doc.meta.createdBy.name,
      createdByEmail: doc.meta.createdBy.email,
      createdDate: doc.meta.createdDate,
      lastUpdatedById: doc.meta.lastUpdatedBy.id,
      lastUpdatedByName: doc.meta.lastUpdatedBy.name,
      lastUpdatedByEmail: doc.meta.lastUpdatedBy.email,
      lastUpdatedDate: doc.meta.lastUpdatedDate,
    });
  }
  console.log(`Migrated ${folderTypeDocs.length} folder types`);

  // Migrate folders
  console.log('Migrating folders...');
  const folderDocs = await mongodb.collection('folders').find().toArray();
  for (const doc of folderDocs) {
    await db.insert(folders).values({
      id: doc._id,
      tenantId: doc.tenantId,
      parents: doc.parents,
      folderTypeKey: doc.folderTypeKey,
      name: doc.name,
      description: doc.description,
      properties: doc.properties,
      status: doc.status,
      createdById: doc.meta.createdBy.id,
      createdByName: doc.meta.createdBy.name,
      createdByEmail: doc.meta.createdBy.email,
      createdDate: doc.meta.createdDate,
      lastUpdatedById: doc.meta.lastUpdatedBy.id,
      lastUpdatedByName: doc.meta.lastUpdatedBy.name,
      lastUpdatedByEmail: doc.meta.lastUpdatedBy.email,
      lastUpdatedDate: doc.meta.lastUpdatedDate,
    });
  }
  console.log(`Migrated ${folderDocs.length} folders`);

  // Migrate roles
  console.log('Migrating roles...');
  const roleDocs = await mongodb.collection('roles').find().toArray();
  for (const doc of roleDocs) {
    await db.insert(roles).values({
      id: doc._id,
      scopeType: doc.scopeType,
      scopeId: doc.scopeId,
      userId: doc.userId,
      key: doc.key,
      label: doc.label,
      permissions: doc.permissions,
      createdById: doc.meta.createdBy.id,
      createdByName: doc.meta.createdBy.name,
      createdByEmail: doc.meta.createdBy.email,
      createdDate: doc.meta.createdDate,
      lastUpdatedById: doc.meta.lastUpdatedBy.id,
      lastUpdatedByName: doc.meta.lastUpdatedBy.name,
      lastUpdatedByEmail: doc.meta.lastUpdatedBy.email,
      lastUpdatedDate: doc.meta.lastUpdatedDate,
    });
  }
  console.log(`Migrated ${roleDocs.length} roles`);

  await mongoClient.close();
  console.log('Migration complete!');
}

migrate().catch(console.error);
```

## Alternatives Considered

### Alternative 1: Prisma ORM

- **Brief description**: Full-featured ORM with schema-first approach and migrations
- **Why it wasn't chosen**: Heavier runtime (~2MB), less SQL-like API, Drizzle better fits project's "thin abstraction" philosophy
- **When it might be better**: If team prefers declarative schema and stronger ORM conventions

### Alternative 2: Kysely

- **Brief description**: Type-safe SQL query builder without schema abstraction
- **Why it wasn't chosen**: No built-in migrations, requires more manual schema management
- **When it might be better**: If team wants maximum SQL control without ORM conventions

### Alternative 3: Raw pg/postgres.js

- **Brief description**: Direct PostgreSQL client usage
- **Why it wasn't chosen**: No type safety, manual SQL strings, more error-prone
- **When it might be better**: Simple queries without complex type requirements

### Alternative 4: Keep MongoDB

- **Brief description**: Remain on current stack
- **Why it wasn't chosen**: Assumes migration is desired; MongoDB works fine for current use case
- **When it might be better**: If there's no compelling reason to migrate

## Debates & Open Questions

### Schema Design Decisions

1. **Flatten metadata vs. separate table?**
   - Current recommendation: Flatten into main table (simpler queries)
   - Alternative: `metadata` table with foreign keys (more normalized)
   - Trade-off: Query simplicity vs. schema normalization

2. **Parents as array vs. junction table?**
   - Current recommendation: UUID array with GIN index
   - Alternative: `folder_parents` junction table
   - Trade-off: Query simplicity vs. relational purity
   - Note: Array approach works well for small-to-medium arrays (<100 parents)

3. **JSONB vs. separate columns for propertySchema?**
   - Must remain JSONB - schema is user-defined and varies per folder type
   - Runtime validation (Ajv) still required

### Performance Considerations

1. **DAG traversal performance**
   - Recursive CTEs vs. iterative queries
   - May need to benchmark with realistic data volumes
   - Consider materialized path pattern if performance is critical

2. **Array vs. junction table performance**
   - Arrays faster for reads and existence checks
   - Junction tables better for large arrays and complex joins
   - Current parents arrays are typically < 10 elements

### Migration Strategy

1. **Blue-green deployment vs. maintenance window?**
   - Blue-green: More complex, zero downtime
   - Maintenance window: Simpler, brief downtime acceptable for this service
   - Recommend: Maintenance window for simplicity (15-30 minutes)

## Recommendations

### Preferred Approach: Option 1 - Direct Replacement

**Should This Be Implemented?**: Yes

**Rationale:**
- Codebase is small enough (3 repositories, ~500 lines) for complete rewrite
- No compelling need for zero-downtime migration
- Dual-database approach doubles maintenance burden without proportional benefit
- Drizzle ORM provides excellent type safety with minimal abstraction

**Why:**
- **Simplicity**: Single implementation to maintain, test, and debug
- **Type Safety**: Drizzle's compile-time checks catch schema mismatches early
- **Performance**: Native Bun SQL support with Drizzle is highly optimized
- **Long-term**: Clean codebase without abstraction layer overhead

**Key Considerations:**
- Schedule 30-minute maintenance window for production migration
- Test migration script thoroughly in staging first
- Have MongoDB backup ready for rollback if needed
- Update documentation after migration

**Potential Challenges:**
1. **Test infrastructure change** - Docker required for testcontainers
   - Mitigation: Verify CI/CD supports Docker; consider GitHub Actions with services
2. **DAG traversal performance** - May need optimization
   - Mitigation: Benchmark with realistic data; implement recursive CTE if needed
3. **Data migration errors** - Schema mismatch possible
   - Mitigation: Run migration against staging MongoDB copy first; validate counts

**Success Criteria:**
- All existing tests pass with PostgreSQL backend
- Response times remain within acceptable range (< 100ms p95)
- Data integrity verified (row counts match, spot check records)
- Zero data loss during migration

## Additional Notes

### Environment Variables

```bash
# Before (MongoDB)
MONGODB_URL=mongodb://localhost:27017/folder

# After (PostgreSQL)
DATABASE_URL=postgresql://folder:folder@localhost:5432/folder
```

### Files to Create/Modify

**Create:**
- `src/db/schema.ts` - Drizzle schema definitions
- `src/db/index.ts` - Database connection factory
- `drizzle.config.ts` - Drizzle Kit configuration
- `scripts/migrate-mongodb-to-postgres.ts` - Data migration script

**Modify:**
- `src/features/folder/folder.repository.ts` - Rewrite with Drizzle
- `src/features/folder-type/folder-type.repository.ts` - Rewrite with Drizzle
- `src/features/role/role.repository.ts` - Rewrite with Drizzle
- `src/features/*/schema.ts` - Remove MongoDB transformations
- `src/graphql/context.ts` - Update ServiceContext type
- `src/server.ts` - Replace MongoDB plugin with Drizzle setup
- `package.json` - Update dependencies
- All `*.spec.ts` files - Update to use testcontainers

**Delete:**
- `src/lib/mongodb/transform-id.ts` - No longer needed

### Estimated Timeline (Direct Replacement)

| Phase | Duration | Activities |
|-------|----------|------------|
| Setup | 0.5 day | Add dependencies, create Drizzle config, define schemas |
| Repository Migration | 2 days | Rewrite all 3 repositories |
| Test Migration | 1 day | Update all tests to testcontainers |
| Service Layer | 0.5 day | Update context, remove MongoDB references |
| Data Migration Script | 0.5 day | Write and test migration script |
| Staging Validation | 1 day | Deploy to staging, run migration, validate |
| Production Deployment | 0.5 day | Schedule maintenance, deploy, migrate |

**Total: 5-7 days**

## Sources

1. [Drizzle ORM - PostgreSQL Column Types](https://orm.drizzle.team/docs/column-types/pg) - Official documentation for JSONB, arrays, and UUID columns
2. [Drizzle ORM - Bun SQL](https://orm.drizzle.team/docs/connect-bun-sql) - Native Bun SQL support for PostgreSQL
3. [Drizzle ORM - Transactions](https://orm.drizzle.team/docs/transactions) - Transaction API and patterns
4. [Drizzle ORM - Migrations](https://orm.drizzle.team/docs/migrations) - Migration strategies and drizzle-kit commands
5. [Drizzle ORM PostgreSQL Best Practices Guide (2025)](https://gist.github.com/productdevbook/7c9ce3bbeb96b3fabc3c7c2aa2abc717) - Community best practices for schema design
6. [PostgreSQL Module - Testcontainers for NodeJS](https://node.testcontainers.org/modules/postgresql/) - Testing with PostgreSQL containers
7. [Database Migration: MongoDB to PostgreSQL - Nearform](https://nearform.com/digital-community/database-migration-a-typescript-guided-journey-from-mongodb-to-postgresql/) - Real-world migration patterns with TypeScript interfaces
8. [Repository Pattern with TypeScript and Node.js](https://dev.to/fyapy/repository-pattern-with-typescript-and-nodejs-25da) - Repository abstraction patterns
9. [Database Migrations with Feature Flags - Harness](https://www.harness.io/blog/database-migration-with-feature-flags) - Six-stage migration pattern
10. [LaunchDarkly - Multi-stage Migrations](https://docs.launchdarkly.com/guides/infrastructure/infrastructure-migration) - Feature flag controlled database migrations
11. [API with NestJS - Arrays with PostgreSQL and Drizzle](https://wanago.io/2024/07/08/api-nestjs-postgresql-arrays-drizzle-orm/) - PostgreSQL array handling patterns
12. [API with NestJS - JSON with PostgreSQL and Drizzle](https://wanago.io/2024/07/15/api-nestjs-json-drizzle-postgresql/) - JSONB handling patterns
13. [MongoDB to Postgres Migration - Entrans](https://www.entrans.ai/blog/mongodb-to-postgres-migration) - Migration strategies and JSONB patterns
14. [Drizzle ORM Schema Declaration](https://orm.drizzle.team/docs/sql-schema-declaration) - Table definitions and constraints

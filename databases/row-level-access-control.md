# Row-Level Access Control for Inspect BI Views

**Date:** 2025-12-03
**Status:** Research Complete
**Author:** Engineering Team

---

## Table of Contents

- [Context](#context)
  - [Data Hierarchy](#data-hierarchy)
  - [Access Patterns](#access-patterns)
  - [Current Authorization System](#current-authorization-system)
  - [Affected Views](#affected-views)
- [Options Evaluated](#options-evaluated)
  - [Option 1: Snowflake Row Access Policies with Mapping Table](#option-1-snowflake-row-access-policies-with-mapping-table)
  - [Option 2: Application-Layer Filtering with OpenFGA](#option-2-application-layer-filtering-with-openfga)
  - [Option 3: Hybrid - OpenFGA + Snowflake Session Variables](#option-3-hybrid---openfga--snowflake-session-variables)
  - [Option 4: Materialized Permissions Table (Recommended)](#option-4-materialized-permissions-table-recommended)
  - [Option 5: Secure Views with External Functions](#option-5-secure-views-with-external-functions)
- [Comparison Matrix](#comparison-matrix)
- [Recommendation](#recommendation)
  - [Primary Recommendation: Option 4 (Materialized Permissions)](#primary-recommendation-option-4-materialized-permissions)
  - [When to Choose Hybrid (Option 3) Instead](#when-to-choose-hybrid-option-3-instead)
  - [Implementation Roadmap](#implementation-roadmap)
- [References](#references)

---

## Context

We need to provide end users access to Inspect BI views through a web application (Google Sheets-like interface). Users should only see data they are authorized to access based on a hierarchical permission model.

### Data Hierarchy

```
TENANT → CAMPUS → PROGRAM
```

- **TENANT**: Top-level organization (e.g., "USC", "UCLA")
- **CAMPUS**: Belongs to one tenant (e.g., "USC-Main", "USC-Health")
- **PROGRAM**: Belongs to one campus (e.g., "Fire Safety 2024")

### Access Patterns

| Access Level | User Sees |
|--------------|-----------|
| Single Program | Data for one specific program |
| Multiple Programs | Data for selected programs |
| Campus-wide | All programs within a campus |
| Tenant-wide | All campuses and programs within a tenant |

### Current Authorization System

- **OpenFGA** - Relationship-based access control (ReBAC)
- Permissions managed externally, not in Snowflake

### Affected Views

All views contain `PROGRAM_ID`, `CAMPUS`, and `TENANT` columns:

- `INSPECT.BI_FINDINGS`
- `INSPECT.BI_RESPONSE_DETAIL`
- `INSPECT.BI_CHECKLIST_ITEMS_SUMMARY`
- `INSPECT.BI_TOP_RANK_BY_ISSUES`
- `INSPECT.BI_INSPECTION_STATUS`
- `INSPECT.BI_ADMIN_CHECKLIST`
- `INSPECT.BI_ADMIN_LOCATION_LISTS`
- `INSPECT.BI_HISTORIC_COMPARISONS()` (table function)

---

## Options Evaluated

### Option 1: Snowflake Row Access Policies with Mapping Table

#### Description

Use Snowflake's native Row Access Policies (RAP) with a mapping table that stores user-to-resource permissions. The mapping table must be synchronized from OpenFGA.

#### Implementation

```sql
-- Mapping table for user permissions
CREATE TABLE INSPECT.USER_ACCESS_MAPPING (
    USER_EMAIL VARCHAR,
    ACCESS_TYPE VARCHAR,  -- 'TENANT', 'CAMPUS', 'PROGRAM'
    ACCESS_VALUE VARCHAR, -- tenant name, campus code, or program ID
    CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Row Access Policy
CREATE OR REPLACE ROW ACCESS POLICY INSPECT.HIERARCHICAL_ACCESS
AS (program_id VARCHAR, campus VARCHAR, tenant VARCHAR) RETURNS BOOLEAN ->
    EXISTS (
        SELECT 1 FROM INSPECT.USER_ACCESS_MAPPING m
        WHERE m.USER_EMAIL = CURRENT_USER()
        AND (
            (m.ACCESS_TYPE = 'TENANT' AND m.ACCESS_VALUE = tenant)
            OR (m.ACCESS_TYPE = 'CAMPUS' AND m.ACCESS_VALUE = campus)
            OR (m.ACCESS_TYPE = 'PROGRAM' AND m.ACCESS_VALUE = program_id)
        )
    )
    OR IS_ROLE_IN_SESSION('ADMIN_ROLE');

-- Apply to views
ALTER VIEW INSPECT.BI_FINDINGS ADD ROW ACCESS POLICY INSPECT.HIERARCHICAL_ACCESS
    ON (PROGRAM_ID, CAMPUS, TENANT);
```

#### Pros

- Database-level enforcement (cannot be bypassed)
- Transparent to queries (same SQL for all users)
- Works across all tools (BI tools, SQL clients, APIs)
- Single enforcement point for auditing

#### Cons

- Requires Snowflake Enterprise Edition
- Must sync permissions from OpenFGA (separate source of truth)
- Performance impact with complex policies or large mapping tables
- One policy per table/view limitation
- Web apps typically use service accounts, not per-user connections

#### Scalability

| Scenario | Impact |
|----------|--------|
| Simple policies | Negligible |
| Mapping table < 10k rows | Minimal |
| Mapping table 10k-100k rows | Moderate (use memoizable functions) |
| Mapping table > 100k rows | Consider partitioning/clustering |

---

### Option 2: Application-Layer Filtering with OpenFGA

#### Description

Query OpenFGA at request time to determine accessible resources, then inject WHERE clauses into SQL queries.

#### Implementation

```typescript
// Pattern: List Objects then Filter
async function getFindings(userId: string) {
  // Ask OpenFGA what this user can access
  const [programs, campuses, tenants] = await Promise.all([
    openfga.listObjects({ user: `user:${userId}`, relation: 'can_view', type: 'program' }),
    openfga.listObjects({ user: `user:${userId}`, relation: 'can_view', type: 'campus' }),
    openfga.listObjects({ user: `user:${userId}`, relation: 'can_view', type: 'tenant' }),
  ]);

  // Build parameterized query
  const sql = `
    SELECT * FROM INSPECT.BI_FINDINGS
    WHERE PROGRAM_ID IN (?)
       OR CAMPUS IN (?)
       OR TENANT IN (?)
  `;

  return snowflake.execute(sql, [programs, campuses, tenants]);
}
```

#### OpenFGA Authorization Model

```yaml
model
  schema 1.1

type user

type tenant
  relations
    define viewer: [user] or admin
    define admin: [user]

type campus
  relations
    define parent: [tenant]
    define viewer: [user] or viewer from parent
    define admin: [user] or admin from parent

type program
  relations
    define parent: [campus]
    define viewer: [user] or viewer from parent
    define admin: [user] or admin from parent
```

#### Pros

- OpenFGA remains single source of truth
- Rich permission model (inheritance, groups, conditions)
- No Snowflake edition requirements
- Supports fine-grained permissions (`can_edit`, `can_delete`, etc.)

#### Cons

- Application must enforce filtering on every query path
- ListObjects has 1,000 result limit by default
- Complex models can cause ListObjects to timeout
- Extra network latency (OpenFGA call before each query)
- SQL injection risk if not using parameterized queries
- Direct database access bypasses controls

#### Scalability

| Operation | Latency |
|-----------|---------|
| Check (single permission) | 1-10ms |
| ListObjects (simple model) | 50-200ms |
| ListObjects (complex model with `but not`) | Can timeout at 3s+ |

**Recommendation:** Cache OpenFGA results with 1-5 minute TTL.

---

### Option 3: Hybrid - OpenFGA + Snowflake Session Variables

#### Description

Use OpenFGA for authorization decisions, pass results to Snowflake via session variables, enforce with Row Access Policies.

#### Implementation

```typescript
// Application code - set session context before queries
async function executeQuery(userId: string, sql: string) {
  const permissions = await getPermissionsFromOpenFGA(userId);

  // Set session variables
  await snowflake.execute(`
    ALTER SESSION SET
      APP_USER_ID = '${userId}',
      APP_ALLOWED_TENANTS = '${permissions.tenants.join(",")}',
      APP_ALLOWED_CAMPUSES = '${permissions.campuses.join(",")}',
      APP_ALLOWED_PROGRAMS = '${permissions.programs.join(",")}'
  `);

  return snowflake.execute(sql);
}
```

```sql
-- Row Access Policy using session variables
CREATE OR REPLACE ROW ACCESS POLICY INSPECT.SESSION_BASED_ACCESS
AS (program_id VARCHAR, campus VARCHAR, tenant VARCHAR) RETURNS BOOLEAN ->
    ARRAY_CONTAINS(program_id::VARIANT, SPLIT(GETVARIABLE('APP_ALLOWED_PROGRAMS'), ','))
    OR ARRAY_CONTAINS(campus::VARIANT, SPLIT(GETVARIABLE('APP_ALLOWED_CAMPUSES'), ','))
    OR ARRAY_CONTAINS(tenant::VARIANT, SPLIT(GETVARIABLE('APP_ALLOWED_TENANTS'), ','))
    OR GETVARIABLE('APP_ALLOWED_TENANTS') = '*';  -- Admin override
```

#### Pros

- OpenFGA remains authoritative (no sync)
- Database-level enforcement
- No mapping table to maintain
- Can cache OpenFGA results in application

#### Cons

- Session setup overhead on each request
- Connection pooling complexity (must reset variables between users)
- Session variable size limits
- Still requires Enterprise Edition for RAP

#### Scalability

- Session variable setup: < 10ms
- Policy evaluation: same as Option 1
- OpenFGA calls: cacheable (1-5 minute TTL recommended)

---

### Option 4: Materialized Permissions Table (Recommended)

#### Description

Periodically sync permissions from OpenFGA to a Snowflake table. Use Row Access Policies against the materialized table. Best query performance with eventual consistency trade-off.

#### Implementation

```sql
-- Materialized permissions table
CREATE TABLE INSPECT.USER_PERMISSIONS (
    USER_ID VARCHAR NOT NULL,
    PERMISSION_TYPE VARCHAR NOT NULL,  -- 'program', 'campus', 'tenant'
    RESOURCE_ID VARCHAR NOT NULL,
    GRANTED_AT TIMESTAMP_NTZ,
    SYNC_BATCH_ID VARCHAR NOT NULL,
    PRIMARY KEY (USER_ID, PERMISSION_TYPE, RESOURCE_ID)
);

-- Index for fast lookups
CREATE INDEX IDX_USER_PERMISSIONS_USER ON INSPECT.USER_PERMISSIONS(USER_ID);

-- Row Access Policy
CREATE OR REPLACE ROW ACCESS POLICY INSPECT.MATERIALIZED_ACCESS
AS (program_id VARCHAR, campus VARCHAR, tenant VARCHAR) RETURNS BOOLEAN ->
    EXISTS (
        SELECT 1 FROM INSPECT.USER_PERMISSIONS p
        WHERE p.USER_ID = GETVARIABLE('APP_USER_ID')
        AND (
            (p.PERMISSION_TYPE = 'tenant' AND p.RESOURCE_ID = tenant)
            OR (p.PERMISSION_TYPE = 'campus' AND p.RESOURCE_ID = campus)
            OR (p.PERMISSION_TYPE = 'program' AND p.RESOURCE_ID = program_id)
        )
    )
    OR GETVARIABLE('APP_USER_ID') = 'SYSTEM';  -- Admin bypass
```

#### Sync Job

```typescript
// Runs on schedule (hourly) or triggered by permission changes
async function syncPermissions() {
  const allUsers = await getActiveUsers();
  const batchId = crypto.randomUUID();
  const records: PermissionRecord[] = [];

  for (const user of allUsers) {
    const [programs, campuses, tenants] = await Promise.all([
      openfga.listObjects({ user: `user:${user.id}`, relation: 'viewer', type: 'program' }),
      openfga.listObjects({ user: `user:${user.id}`, relation: 'viewer', type: 'campus' }),
      openfga.listObjects({ user: `user:${user.id}`, relation: 'viewer', type: 'tenant' }),
    ]);

    for (const p of programs) {
      records.push({ userId: user.id, type: 'program', resourceId: p, batchId });
    }
    for (const c of campuses) {
      records.push({ userId: user.id, type: 'campus', resourceId: c, batchId });
    }
    for (const t of tenants) {
      records.push({ userId: user.id, type: 'tenant', resourceId: t, batchId });
    }
  }

  // Batch insert
  await snowflake.bulkInsert('INSPECT.USER_PERMISSIONS', records);

  // Atomic cleanup: remove old permissions
  await snowflake.execute(`
    DELETE FROM INSPECT.USER_PERMISSIONS
    WHERE SYNC_BATCH_ID != '${batchId}'
  `);

  console.log(`Synced ${records.length} permissions for ${allUsers.length} users`);
}
```

#### Pros

- Best query performance (no external calls during query)
- Database-level enforcement
- Works with BI tools without modification
- Debuggable (can query permissions table directly)
- Consistent across all access paths

#### Cons

- Eventually consistent (permission changes not immediate)
- Sync job complexity (failures, retries, partial updates)
- Storage cost for denormalized permissions
- Full sync can be slow with many users

#### Scalability

| Metric | Expectation |
|--------|-------------|
| Query performance | Excellent (indexed joins) |
| Storage | ~100 bytes per permission row |
| Sync time (100 users) | < 1 minute |
| Sync time (1,000 users) | 5-15 minutes |
| Sync time (10,000 users) | Consider incremental sync |

#### Incremental Sync (For Scale)

For large user bases, implement incremental sync triggered by OpenFGA webhooks:

```typescript
// Webhook handler for permission changes
async function handlePermissionChange(event: OpenFGAEvent) {
  const { user, relation, object } = event;

  if (event.type === 'tuple_added') {
    await snowflake.execute(`
      INSERT INTO INSPECT.USER_PERMISSIONS (USER_ID, PERMISSION_TYPE, RESOURCE_ID, SYNC_BATCH_ID)
      VALUES (?, ?, ?, 'webhook')
    `, [user, getType(object), getId(object)]);
  } else if (event.type === 'tuple_removed') {
    await snowflake.execute(`
      DELETE FROM INSPECT.USER_PERMISSIONS
      WHERE USER_ID = ? AND PERMISSION_TYPE = ? AND RESOURCE_ID = ?
    `, [user, getType(object), getId(object)]);
  }
}
```

---

### Option 5: Secure Views with External Functions

#### Description

Create external functions that call OpenFGA, use them in secure views for real-time authorization.

#### Implementation

```sql
-- External function (requires API integration setup)
CREATE OR REPLACE EXTERNAL FUNCTION INSPECT.CHECK_ACCESS(
    user_id VARCHAR,
    resource_type VARCHAR,
    resource_id VARCHAR
)
RETURNS BOOLEAN
API_INTEGRATION = openfga_api_integration
AS 'https://auth-api.example.com/check';

-- Secure view
CREATE OR REPLACE SECURE VIEW INSPECT.BI_FINDINGS_SECURED AS
SELECT * FROM INSPECT.BI_FINDINGS
WHERE INSPECT.CHECK_ACCESS(GETVARIABLE('APP_USER_ID'), 'program', PROGRAM_ID)
   OR INSPECT.CHECK_ACCESS(GETVARIABLE('APP_USER_ID'), 'campus', CAMPUS)
   OR INSPECT.CHECK_ACCESS(GETVARIABLE('APP_USER_ID'), 'tenant', TENANT);
```

#### Pros

- Real-time authorization (always current)
- OpenFGA authoritative
- No sync needed

#### Cons

- **Severe performance impact** - External call per row
- External function setup complexity and cost
- Rate limiting concerns
- Not practical for result sets > 100 rows

#### Scalability

**Not recommended** for production use with realistic data volumes.

---

## Comparison Matrix

| Criteria | RAP + Mapping | App-Layer | Hybrid | Materialized | External UDF |
|----------|---------------|-----------|--------|--------------|--------------|
| OpenFGA authoritative | No (sync) | Yes | Yes | No (sync) | Yes |
| Database enforcement | Yes | No | Yes | Yes | Yes |
| Real-time permissions | No | Yes | Yes | No | Yes |
| Query performance | Good | Good | Good | Excellent | Poor |
| Implementation complexity | Medium | Medium | High | Medium | Medium |
| Works with BI tools | Yes | No | Yes | Yes | Yes |
| Snowflake Edition | Enterprise | Any | Enterprise | Enterprise | Enterprise |
| Operational overhead | Low | Low | Medium | Medium | High |

---

## Recommendation

### Primary Recommendation: Option 4 (Materialized Permissions)

For a web application with Google Sheets-like interface, **Materialized Permissions** provides the best balance of:

1. **Query Performance** - Users expect instant responses when filtering/sorting data
2. **Simplicity** - Single enforcement point, easy to debug
3. **Flexibility** - Works if you later add Excel export, BI dashboards, direct SQL access
4. **Reliability** - No runtime dependencies on external services during queries

### When to Choose Hybrid (Option 3) Instead

- Permissions change very frequently (multiple times per hour)
- Real-time permission changes are a hard requirement
- You have a small, stable user base (< 100 users)

### Implementation Roadmap

1. **Phase 1: OpenFGA Model**
   - Define tenant/campus/program type hierarchy
   - Implement viewer/admin relations with inheritance
   - Test with sample permission tuples

2. **Phase 2: Snowflake Infrastructure**
   - Create `INSPECT.USER_PERMISSIONS` table
   - Create Row Access Policy
   - Apply policy to all BI views

3. **Phase 3: Sync Service**
   - Implement full sync job (run hourly)
   - Add monitoring and alerting
   - Test with production-like data volume

4. **Phase 4: Application Integration**
   - Set `APP_USER_ID` session variable on each request
   - Add "permissions syncing" indicator in UI if needed
   - Implement permission change webhook for near-real-time updates

5. **Phase 5: Optimization (if needed)**
   - Implement incremental sync via webhooks
   - Add caching layer for OpenFGA calls
   - Optimize permission table indexes

---

## References

- [Snowflake Row Access Policies Documentation](https://docs.snowflake.com/en/user-guide/security-row-intro)
- [Snowflake Row Access Policies Demystified (InterWorks)](https://interworks.com/blog/2024/01/30/snowflake-row-access-policies-demystified/)
- [Snowflake Row Access Policy with Hierarchical Roles](https://ericheilman.com/2024/01/03/snowflake-row-access-policy-with-hierarchical-roles/)
- [OpenFGA Documentation](https://openfga.dev/)
- [OpenFGA Search with Permissions](https://openfga.dev/docs/interacting/search-with-permissions)
- [OpenFGA ListObjects Performance](https://openfga.dev/docs/getting-started/perform-list-objects)
- [OpenFGA Production Best Practices](https://openfga.dev/docs/best-practices/running-in-production)
- [AuthZed Materialize (similar pattern)](https://authzed.com/docs/authzed/concepts/authzed-materialize)
- [Permit.io Data Filtering Patterns](https://docs.permit.io/how-to/enforce-permissions/data-filtering/)

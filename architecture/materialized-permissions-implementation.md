# Materialized Permissions Implementation Guide

**Date:** 2025-12-03
**Status:** Research Complete
**Author:** Engineering Team

---

## Table of Contents

- [Executive Summary](#executive-summary)
- [Architecture Overview](#architecture-overview)
  - [System Components](#system-components)
  - [Data Flow](#data-flow)
- [OpenFGA Authorization Model](#openfga-authorization-model)
  - [Type Definitions](#type-definitions)
  - [Relation Inheritance](#relation-inheritance)
  - [Future Extensibility](#future-extensibility)
  - [Example Tuples](#example-tuples)
- [Snowflake Implementation](#snowflake-implementation)
  - [Permissions Table Design](#permissions-table-design)
  - [Memoizable Function for Performance](#memoizable-function-for-performance)
  - [Row Access Policy](#row-access-policy)
  - [Applying Policies to Views](#applying-policies-to-views)
- [Sync Service Architecture](#sync-service-architecture)
  - [Deployment Options](#deployment-options)
  - [Recommended: Dedicated Microservice](#recommended-dedicated-microservice)
  - [Full Sync Implementation](#full-sync-implementation)
  - [Incremental Sync via ReadChanges API](#incremental-sync-via-readchanges-api)
  - [Webhook Trigger Pattern](#webhook-trigger-pattern)
- [Application Integration](#application-integration)
  - [Setting User Context](#setting-user-context)
  - [Connection Pooling Considerations](#connection-pooling-considerations)
  - [Query Execution Pattern](#query-execution-pattern)
- [View Metadata and Discovery](#view-metadata-and-discovery)
  - [Business Requirement](#business-requirement)
  - [Approach: Snowflake Tags (Enterprise Edition)](#approach-snowflake-tags-enterprise-edition)
  - [Tag Definition](#tag-definition)
  - [Applying Tags to Views](#applying-tags-to-views)
  - [Querying View Metadata](#querying-view-metadata)
  - [Optimized Query Function](#optimized-query-function)
  - [Next.js Integration](#nextjs-integration)
  - [React Component Example](#react-component-example)
  - [Schema-Level Default Tags](#schema-level-default-tags)
  - [Governance and Audit](#governance-and-audit)
  - [Alternative: JSON Comment (Non-Enterprise)](#alternative-json-comment-non-enterprise)
- [Permission-Based Report Discovery](#permission-based-report-discovery)
  - [The Core Challenge](#the-core-challenge)
  - [Options Analysis](#options-analysis)
  - [Recommended Approach: Phased Implementation](#recommended-approach-phased-implementation)
  - [Data Points Required](#data-points-required)
  - [Complete Flow](#complete-flow)
  - [Implementation: GET_USER_REPORTS Function](#implementation-get_user_reports-function)
  - [Next.js API Integration](#nextjs-api-integration-1)
  - [React Hook for Report Picker](#react-hook-for-report-picker)
  - [Phase 2: Category-Gated Implementation](#phase-2-category-gated-implementation)
  - [Sync Service Update for Phase 2](#sync-service-update-for-phase-2)
- [Scalability Analysis](#scalability-analysis)
  - [User Scale Projections](#user-scale-projections)
  - [Query Performance](#query-performance)
  - [Sync Performance](#sync-performance)
  - [Storage Requirements](#storage-requirements)
- [Operational Considerations](#operational-considerations)
  - [Monitoring](#monitoring)
  - [Alerting](#alerting)
  - [Consistency Guarantees](#consistency-guarantees)
- [Security Considerations](#security-considerations)
- [Tenant Data Export (Service Accounts)](#tenant-data-export-service-accounts)
  - [Business Case](#business-case)
  - [Options Analysis](#options-analysis)
  - [Option 5: Unified OpenFGA Approach (Recommended)](#option-5-unified-openfga-approach-recommended)
  - [Security Considerations (Service Accounts)](#security-considerations-1)
  - [When to Use Reader Accounts Instead](#when-to-use-reader-accounts-instead)
- [Implementation Roadmap](#implementation-roadmap)
- [Implementation Notes](#implementation-notes)
- [References](#references)

---

## Executive Summary

This document provides a comprehensive implementation guide for **Option 4: Materialized Permissions Table** - the recommended approach for implementing row-level access control on Snowflake BI views using OpenFGA as the authorization system.

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Authorization Model | Extends existing FGA model | Adds `can_view_data_explorer` relation to existing tenant/campus/domain/program types |
| Data Hierarchy | tenant → campus → domain → program | Matches existing FGA model; domain is intermediate level between campus and program |
| Cross-Service Access | Unified for now | Program access grants all data; designed for future service-specific permissions |
| Deny Rules | Not supported | Simpler model; only positive grants |
| Sync Runtime | Dedicated microservice | Separation of concerns; independent scaling |
| Sync Triggers | Scheduled + webhook + ReadChanges polling | Near-real-time updates with reliable fallback |
| User Scale | 100-1,000 users | Medium scale; full sync viable |

### Architecture Summary

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Microservices │────▶│    OpenFGA      │◀────│  Sync Service   │
│ (write tuples)  │     │   (source of    │     │ (polls changes) │
└─────────────────┘     │     truth)      │     └────────┬────────┘
                        └─────────────────┘              │
                                                         ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Web App       │────▶│   Snowflake     │◀────│ USER_PERMISSIONS│
│ (sets context)  │     │  (RAP enforces) │     │    (table)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

---

## Architecture Overview

### System Components

| Component | Responsibility |
|-----------|----------------|
| **OpenFGA Server** | Single source of truth for all authorization decisions |
| **Microservices** | Write permission tuples when business events occur (user assigned to program, etc.) |
| **Sync Service** | Materializes OpenFGA permissions into Snowflake table |
| **Snowflake USER_PERMISSIONS** | Denormalized permissions table for fast query-time filtering |
| **Row Access Policy** | Enforces access control on all BI views |
| **Web Application** | Sets user context before executing queries |

### Data Flow

```
1. PERMISSION GRANT
   ┌──────────────┐    ┌──────────┐
   │ Admin grants │───▶│ OpenFGA  │  Tuple: user:alice#can_view_data_explorer@program:fire-safety-2024
   │ access in UI │    │          │
   └──────────────┘    └──────────┘

2. SYNC (Scheduled or Triggered)
   ┌──────────────┐    ┌──────────┐    ┌───────────────────┐
   │ Sync Service │───▶│ OpenFGA  │───▶│ Snowflake         │
   │              │    │ ListObj  │    │ USER_PERMISSIONS  │
   └──────────────┘    └──────────┘    └───────────────────┘

3. QUERY EXECUTION
   ┌──────────────┐    ┌──────────┐    ┌───────────────────┐
   │ Web App      │───▶│Snowflake │───▶│ Row Access Policy │
   │ SET context  │    │ Query    │    │ filters rows      │
   └──────────────┘    └──────────┘    └───────────────────┘
```

---

## OpenFGA Authorization Model

### Existing Model Context

Your current OpenFGA model has the following hierarchy with existing relations:

```
global → tenant → campus → domain → program
```

Key existing relations:
- `dashboard_viewer` - For internal application dashboards (leave unchanged)
- `configuration_manager` - Admin access inherited down the hierarchy
- `admin` - Role-based admin access at each level

### Adding `can_view_data_explorer` Relation

We add a new relation `can_view_data_explorer` to enable Snowflake data explorer access. This relation:
- Is separate from `dashboard_viewer` (different feature)
- Inherits down the hierarchy like other relations
- Can be granted at tenant, campus, domain, or program level

**Additions to existing types:**

```yaml
type tenant
  relations
    # ... existing relations ...

    # NEW: Data explorer access (Snowflake BI views)
    define can_view_data_explorer: [user] or configuration_manager

type campus
  relations
    # ... existing relations ...

    # NEW: Data explorer - direct grant OR inherited from tenant
    define can_view_data_explorer: [user] or can_view_data_explorer from tenant

type domain
  relations
    # ... existing relations ...

    # NEW: Data explorer - direct grant OR inherited from campus
    define can_view_data_explorer: [user] or can_view_data_explorer from campus

type program
  relations
    # ... existing relations ...

    # NEW: Data explorer - direct grant OR inherited from domain
    define can_view_data_explorer: [user] or can_view_data_explorer from domain
```

### Complete Type Additions (Copy-Paste Ready)

Add these relations to your existing model:

```yaml
# In type tenant, add:
    define can_view_data_explorer: [user] or configuration_manager

# In type campus, add:
    define can_view_data_explorer: [user] or can_view_data_explorer from tenant

# In type domain, add:
    define can_view_data_explorer: [user] or can_view_data_explorer from campus

# In type program, add:
    define can_view_data_explorer: [user] or can_view_data_explorer from domain
```

### Relation Inheritance

The model implements **permission inheritance** through the 4-level hierarchy:

```
tenant:uc
  └── can_view_data_explorer: user:alice
      │
      ▼ (inherits down via: can_view_data_explorer from tenant)
campus:USC (tenant: tenant:uc)
  └── can_view_data_explorer: [inherited from tenant]
      │
      ▼ (inherits down via: can_view_data_explorer from campus)
domain:safety (campus: campus:USC)
  └── can_view_data_explorer: [inherited from campus]
      │
      ▼ (inherits down via: can_view_data_explorer from domain)
program:fire-safety-2024 (domain: domain:safety)
  └── can_view_data_explorer: [inherited from domain]
```

**Result:** Granting `can_view_data_explorer` at the tenant level automatically grants access to all campuses, domains, and programs within that tenant.

### Inheritance from `configuration_manager`

Note that `configuration_manager` users automatically get `can_view_data_explorer` at the tenant level:

```yaml
define can_view_data_explorer: [user] or configuration_manager
```

This means:
- Global admins (who are `configuration_manager` at tenant level) automatically have data explorer access
- This aligns with your existing permission model

### Future Extensibility

The model can support **service-specific permissions** in the future:

```yaml
# Future addition for service-specific access
type program
  relations
    # ... existing relations ...

    # Domain-specific data explorer (future)
    define can_view_inspect_data: [user] or can_view_inspect_data from domain
    define can_view_forms_data: [user] or can_view_forms_data from domain
    define can_view_inventory_data: [user] or can_view_inventory_data from domain

    # General data explorer inherits from all service-specific viewers
    define can_view_data_explorer: [user] or can_view_inspect_data or can_view_forms_data or can_view_inventory_data or can_view_data_explorer from domain
```

This allows gradual migration:
1. Start with `can_view_data_explorer` for all data access
2. Later, introduce `can_view_inspect_data`, `can_view_forms_data`, etc.
3. Update sync service to include data source in permissions table
4. Update Row Access Policy to filter by data source

### Example Tuples

```bash
# Tenant-level access (sees all UC system data)
fga tuple write user:alice can_view_data_explorer tenant:uc

# Campus-level access (sees all USC data)
fga tuple write user:bob can_view_data_explorer campus:USC

# Domain-level access (sees all Safety domain data)
fga tuple write user:charlie can_view_data_explorer domain:safety

# Program-level access (sees only Fire Safety 2024)
fga tuple write user:dana can_view_data_explorer program:fire-safety-2024

# Note: Hierarchy tuples already exist in your model via:
#   tenant: [tenant] on campus
#   campus: [campus] on domain
#   domain: [domain] on program
```

---

## Snowflake Implementation

### Permissions Table Design

The permissions table lives in a dedicated `AUTH` schema, shared across all data schemas (INSPECT, INVENTORY, FORMS, etc.).

```sql
-- ============================================================================
-- AUTH Schema: Shared authorization infrastructure
-- ============================================================================

CREATE SCHEMA IF NOT EXISTS AUTH;

GRANT USAGE ON SCHEMA AUTH TO ROLE APP_DASHBOARD_ROLE;
GRANT USAGE ON SCHEMA AUTH TO ROLE MSK_CONNECTOR_ROLE;

-- ============================================================================
-- USER_PERMISSIONS: Materialized permissions from OpenFGA
-- ============================================================================

CREATE TABLE IF NOT EXISTS AUTH.USER_PERMISSIONS (
    -- User identifier (matches OAuth subject claim)
    USER_ID VARCHAR(255) NOT NULL,

    -- Permission scope: 'tenant', 'campus', 'domain', 'program'
    PERMISSION_SCOPE VARCHAR(20) NOT NULL,

    -- Resource identifier (tenant name, campus code, domain ID, or program ID)
    RESOURCE_ID VARCHAR(255) NOT NULL,

    -- Audit fields
    SYNCED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    SYNC_BATCH_ID VARCHAR(36) NOT NULL,

    -- Primary key prevents duplicates
    PRIMARY KEY (USER_ID, PERMISSION_SCOPE, RESOURCE_ID)
);

-- Index for fast user lookups during policy evaluation
CREATE INDEX IF NOT EXISTS IDX_USER_PERMISSIONS_LOOKUP
    ON AUTH.USER_PERMISSIONS (USER_ID, PERMISSION_SCOPE);

-- Index for sync operations (finding stale records)
CREATE INDEX IF NOT EXISTS IDX_USER_PERMISSIONS_BATCH
    ON AUTH.USER_PERMISSIONS (SYNC_BATCH_ID);

-- Grant sync service access
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE AUTH.USER_PERMISSIONS
    TO ROLE MSK_CONNECTOR_ROLE;

-- Grant read access for Row Access Policy evaluation
GRANT SELECT ON TABLE AUTH.USER_PERMISSIONS TO ROLE APP_DASHBOARD_ROLE;

-- Table comment for documentation
COMMENT ON TABLE AUTH.USER_PERMISSIONS IS
    'Materialized permissions from OpenFGA. Synced by permission-sync-service.
     Shared across all data schemas (INSPECT, INVENTORY, FORMS, etc.).';
```

### Memoizable Function for Performance

```sql
-- ============================================================================
-- GET_USER_PERMISSIONS: Memoizable function for Row Access Policy
-- Lives in AUTH schema, used by Row Access Policies across all data schemas
-- ============================================================================

CREATE OR REPLACE FUNCTION AUTH.GET_USER_PERMISSIONS(p_user_id VARCHAR)
RETURNS TABLE (
    PERMISSION_SCOPE VARCHAR,
    RESOURCE_ID VARCHAR
)
MEMOIZABLE
AS
$$
    SELECT PERMISSION_SCOPE, RESOURCE_ID
    FROM AUTH.USER_PERMISSIONS
    WHERE USER_ID = p_user_id
$$;

-- Grant usage to roles that need RAP evaluation
GRANT USAGE ON FUNCTION AUTH.GET_USER_PERMISSIONS(VARCHAR) TO ROLE APP_DASHBOARD_ROLE;

COMMENT ON FUNCTION AUTH.GET_USER_PERMISSIONS(VARCHAR) IS
    'Memoizable function returning user permissions for RAP evaluation.
     Cache invalidates automatically when USER_PERMISSIONS table changes.
     Used by Row Access Policies across all data schemas.';
```

**Why Memoizable?**

Per [Snowflake documentation](https://docs.snowflake.com/en/developer-guide/udf/sql/udf-sql-scalar-functions):
- Memoizable functions cache results per session
- Cache automatically invalidates when underlying table changes
- Reduces repeated subquery execution in Row Access Policy
- Can improve policy evaluation by 10x or more

**Limitations:**
- 10 KB result set limit per session
- Does not cache VARIANT or OBJECT types
- Cannot reference other memoizable functions

### Row Access Policy

The Row Access Policy is also defined in the `AUTH` schema and can be applied to views in any data schema.

```sql
-- ============================================================================
-- DATA_ACCESS: Row Access Policy for BI views
-- Lives in AUTH schema, applied to views across all data schemas
-- Evaluates user permissions at tenant, campus, domain, and program levels
-- ============================================================================

CREATE OR REPLACE ROW ACCESS POLICY AUTH.DATA_ACCESS
AS (program_id VARCHAR, domain_id VARCHAR, campus VARCHAR, tenant VARCHAR)
RETURNS BOOLEAN ->
    -- Check if user has any matching permission
    EXISTS (
        SELECT 1
        FROM TABLE(AUTH.GET_USER_PERMISSIONS(GETVARIABLE('APP_USER_ID'))) p
        WHERE
            -- Tenant-level access
            (p.PERMISSION_SCOPE = 'tenant' AND p.RESOURCE_ID = tenant)
            -- Campus-level access
            OR (p.PERMISSION_SCOPE = 'campus' AND p.RESOURCE_ID = campus)
            -- Domain-level access
            OR (p.PERMISSION_SCOPE = 'domain' AND p.RESOURCE_ID = domain_id)
            -- Program-level access
            OR (p.PERMISSION_SCOPE = 'program' AND p.RESOURCE_ID = program_id)
    )
    -- Admin bypass (for service accounts or super admins)
    OR GETVARIABLE('APP_USER_ID') IS NULL  -- Allows internal queries
    OR IS_ROLE_IN_SESSION('ACCOUNTADMIN');

COMMENT ON ROW ACCESS POLICY AUTH.DATA_ACCESS IS
    'Enforces hierarchical data access based on materialized OpenFGA permissions.
     Requires APP_USER_ID session variable to be set.
     Can be applied to views in any schema (INSPECT, INVENTORY, FORMS, etc.).';
```

**Note:** The BI views need a `DOMAIN_ID` column for domain-level filtering. If views don't have this column, you can:
1. Add it to the views by joining with program metadata
2. Skip domain-level permissions and only use tenant/campus/program
3. Use program-level permissions as a proxy (since programs belong to domains)

### Applying Policies to Views

**Option A: If views have DOMAIN_ID column:**

```sql
-- ============================================================================
-- Apply Row Access Policy to BI views (with DOMAIN_ID)
-- The AUTH.DATA_ACCESS policy can be applied to any schema
-- ============================================================================

-- INSPECT schema views
ALTER VIEW INSPECT.BI_FINDINGS
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS
    ON (PROGRAM_ID, DOMAIN_ID, CAMPUS, TENANT);

ALTER VIEW INSPECT.BI_RESPONSE_DETAIL
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS
    ON (PROGRAM_ID, DOMAIN_ID, CAMPUS, TENANT);

-- INVENTORY schema views (future)
ALTER VIEW INVENTORY.BI_ITEMS
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS
    ON (PROGRAM_ID, DOMAIN_ID, CAMPUS, TENANT);

-- FORMS schema views (future)
ALTER VIEW FORMS.BI_SUBMISSIONS
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS
    ON (PROGRAM_ID, DOMAIN_ID, CAMPUS, TENANT);
```

**Option B: If views don't have DOMAIN_ID (simpler, skip domain-level):**

```sql
-- ============================================================================
-- Simplified Row Access Policy (without domain level)
-- ============================================================================

CREATE OR REPLACE ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE
AS (program_id VARCHAR, campus VARCHAR, tenant VARCHAR)
RETURNS BOOLEAN ->
    EXISTS (
        SELECT 1
        FROM TABLE(AUTH.GET_USER_PERMISSIONS(GETVARIABLE('APP_USER_ID'))) p
        WHERE
            (p.PERMISSION_SCOPE = 'tenant' AND p.RESOURCE_ID = tenant)
            OR (p.PERMISSION_SCOPE = 'campus' AND p.RESOURCE_ID = campus)
            OR (p.PERMISSION_SCOPE = 'program' AND p.RESOURCE_ID = program_id)
    )
    OR GETVARIABLE('APP_USER_ID') IS NULL
    OR IS_ROLE_IN_SESSION('ACCOUNTADMIN');

COMMENT ON ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE IS
    'Simplified policy for views without DOMAIN_ID column.
     Use AUTH.DATA_ACCESS for views with domain support.';

-- Apply to INSPECT views
ALTER VIEW INSPECT.BI_FINDINGS
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE
    ON (PROGRAM_ID, CAMPUS, TENANT);

ALTER VIEW INSPECT.BI_RESPONSE_DETAIL
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE
    ON (PROGRAM_ID, CAMPUS, TENANT);

ALTER VIEW INSPECT.BI_CHECKLIST_ITEMS_SUMMARY
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE
    ON (PROGRAM_ID, CAMPUS, TENANT);

ALTER VIEW INSPECT.BI_TOP_RANK_BY_ISSUES
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE
    ON (PROGRAM_ID, CAMPUS, TENANT);

ALTER VIEW INSPECT.BI_INSPECTION_STATUS
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE
    ON (PROGRAM_ID, CAMPUS, TENANT);

ALTER VIEW INSPECT.BI_ADMIN_CHECKLIST
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE
    ON (PROGRAM_ID, CAMPUS, TENANT);

ALTER VIEW INSPECT.BI_ADMIN_LOCATION_LISTS
    ADD ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE
    ON (PROGRAM_ID, CAMPUS, TENANT);
```

**Note:** The table function `BI_HISTORIC_COMPARISONS` cannot have a Row Access Policy directly attached. Options:
1. Wrap it in a view that applies filtering
2. Filter in the application layer
3. Create a secure UDF wrapper

---

## Sync Service Architecture

### Deployment Options

| Option | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| **Same service as web app** | Simple deployment | Couples concerns; sync failures affect web app | Not recommended |
| **Dedicated microservice** | Separation of concerns; independent scaling; clearer monitoring | Additional deployment | **Recommended** |
| **Serverless (Lambda)** | Cost-efficient for low frequency; auto-scaling | Cold start latency; execution time limits | Good for scheduled-only |

### Recommended: Dedicated Microservice

```
┌───────────────────────────────────────────────────────────┐
│                  Permission Sync Service                  │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  Scheduler   │  │   Webhook    │  │  ReadChanges │     │
│  │    (cron)    │  │   Handler    │  │    Poller    │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                 │                 │             │
│         ▼                 ▼                 ▼             │
│  ┌───────────────────────────────────────────────────┐    │
│  │                 Sync Orchestrator                 │    │
│  │  - Determines sync strategy (full vs incremental) │    │
│  │  - Manages batch IDs                              │    │
│  │  - Handles retries and failures                   │    │
│  └───────────────────────────────────────────────────┘    │
│                           │                               │
│        ┌──────────────────┼──────────────────┐            │
│        ▼                  ▼                  ▼            │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐       │
│  │  OpenFGA   │    │ Permission │    │ Snowflake  │       │
│  │   Client   │    │  Resolver  │    │   Writer   │       │
│  └────────────┘    └────────────┘    └────────────┘       │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### Full Sync Implementation

```typescript
// src/sync/full-sync.ts

import { OpenFgaClient } from '@openfga/sdk';
import { Snowflake } from 'snowflake-sdk';
import { v4 as uuid } from 'uuid';

interface Permission {
  userId: string;
  scope: 'tenant' | 'campus' | 'domain' | 'program';
  resourceId: string;
}

interface SyncResult {
  batchId: string;
  usersProcessed: number;
  permissionsWritten: number;
  durationMs: number;
  errors: string[];
}

export class FullSyncService {
  constructor(
    private openfga: OpenFgaClient,
    private snowflake: Snowflake,
    private config: {
      storeId: string;
      pageSize: number;
      maxConcurrency: number;
    }
  ) {}

  async sync(userIds: string[]): Promise<SyncResult> {
    const startTime = Date.now();
    const batchId = uuid();
    const errors: string[] = [];
    const allPermissions: Permission[] = [];

    // Process users in batches to control concurrency
    const batches = this.chunk(userIds, this.config.maxConcurrency);

    for (const batch of batches) {
      const results = await Promise.allSettled(
        batch.map(userId => this.getUserPermissions(userId))
      );

      results.forEach((result, index) => {
        if (result.status === 'fulfilled') {
          allPermissions.push(...result.value);
        } else {
          errors.push(`User ${batch[index]}: ${result.reason}`);
        }
      });
    }

    // Write to Snowflake
    await this.writePermissions(allPermissions, batchId);

    // Clean up old permissions (atomic swap)
    await this.cleanupOldPermissions(batchId);

    return {
      batchId,
      usersProcessed: userIds.length,
      permissionsWritten: allPermissions.length,
      durationMs: Date.now() - startTime,
      errors,
    };
  }

  private async getUserPermissions(userId: string): Promise<Permission[]> {
    const permissions: Permission[] = [];

    // Query each resource type in the hierarchy
    for (const type of ['tenant', 'campus', 'domain', 'program'] as const) {
      let continuationToken: string | undefined;

      do {
        const response = await this.openfga.listObjects({
          user: `user:${userId}`,
          relation: 'can_view_data_explorer',  // The new relation for data explorer access
          type,
          pageSize: this.config.pageSize,
          continuationToken,
        });

        for (const object of response.objects) {
          // Extract resource ID from "type:id" format
          const resourceId = object.split(':')[1];
          permissions.push({
            userId,
            scope: type,
            resourceId,
          });
        }

        continuationToken = response.continuation_token;
      } while (continuationToken);
    }

    return permissions;
  }

  private async writePermissions(
    permissions: Permission[],
    batchId: string
  ): Promise<void> {
    if (permissions.length === 0) return;

    // Use MERGE for upsert behavior
    const sql = `
      MERGE INTO AUTH.USER_PERMISSIONS AS target
      USING (
        SELECT
          column1 AS USER_ID,
          column2 AS PERMISSION_SCOPE,
          column3 AS RESOURCE_ID,
          column4 AS SYNC_BATCH_ID
        FROM VALUES ${permissions.map(() => '(?, ?, ?, ?)').join(', ')}
      ) AS source
      ON target.USER_ID = source.USER_ID
         AND target.PERMISSION_SCOPE = source.PERMISSION_SCOPE
         AND target.RESOURCE_ID = source.RESOURCE_ID
      WHEN MATCHED THEN UPDATE SET
        SYNCED_AT = CURRENT_TIMESTAMP(),
        SYNC_BATCH_ID = source.SYNC_BATCH_ID
      WHEN NOT MATCHED THEN INSERT (
        USER_ID, PERMISSION_SCOPE, RESOURCE_ID, SYNC_BATCH_ID
      ) VALUES (
        source.USER_ID, source.PERMISSION_SCOPE, source.RESOURCE_ID, source.SYNC_BATCH_ID
      )
    `;

    const binds = permissions.flatMap(p => [
      p.userId,
      p.scope,
      p.resourceId,
      batchId,
    ]);

    await this.snowflake.execute({ sqlText: sql, binds });
  }

  private async cleanupOldPermissions(batchId: string): Promise<void> {
    // Remove permissions not in current batch
    await this.snowflake.execute({
      sqlText: `
        DELETE FROM AUTH.USER_PERMISSIONS
        WHERE SYNC_BATCH_ID != ?
      `,
      binds: [batchId],
    });
  }

  private chunk<T>(array: T[], size: number): T[][] {
    return Array.from(
      { length: Math.ceil(array.length / size) },
      (_, i) => array.slice(i * size, i * size + size)
    );
  }
}
```

### Incremental Sync via ReadChanges API

```typescript
// src/sync/incremental-sync.ts

import { OpenFgaClient } from '@openfga/sdk';

interface TupleChange {
  tuple_key: {
    user: string;
    relation: string;
    object: string;
  };
  operation: 'TUPLE_OPERATION_WRITE' | 'TUPLE_OPERATION_DELETE';
  timestamp: string;
}

export class IncrementalSyncService {
  private continuationToken: string | null = null;

  constructor(
    private openfga: OpenFgaClient,
    private fullSync: FullSyncService,
    private storage: TokenStorage,
  ) {}

  async initialize(): Promise<void> {
    // Load saved continuation token from persistent storage
    this.continuationToken = await this.storage.getToken('openfga_changes');
  }

  async pollChanges(): Promise<void> {
    let hasMoreChanges = true;

    while (hasMoreChanges) {
      const response = await this.openfga.readChanges({
        type: 'program', // Filter to relevant types
        pageSize: 100,
        continuationToken: this.continuationToken || undefined,
      });

      const changes = response.changes || [];

      if (changes.length === 0) {
        hasMoreChanges = false;
        continue;
      }

      // Process changes
      await this.processChanges(changes);

      // Save continuation token
      if (response.continuation_token !== this.continuationToken) {
        this.continuationToken = response.continuation_token;
        await this.storage.saveToken('openfga_changes', this.continuationToken);
      } else {
        // Same token returned = no more changes
        hasMoreChanges = false;
      }
    }
  }

  private async processChanges(changes: TupleChange[]): Promise<void> {
    // Group changes by user for efficient processing
    const userChanges = new Map<string, TupleChange[]>();

    for (const change of changes) {
      // Only process can_view_data_explorer relation changes
      if (change.tuple_key.relation !== 'can_view_data_explorer') continue;

      const userId = this.extractUserId(change.tuple_key.user);
      if (!userId) continue;

      if (!userChanges.has(userId)) {
        userChanges.set(userId, []);
      }
      userChanges.get(userId)!.push(change);
    }

    // Re-sync affected users
    const affectedUsers = Array.from(userChanges.keys());
    if (affectedUsers.length > 0) {
      await this.fullSync.sync(affectedUsers);
    }
  }

  private extractUserId(user: string): string | null {
    // Parse "user:alice" format
    const match = user.match(/^user:(.+)$/);
    return match ? match[1] : null;
  }
}

// Polling scheduler
export function startPolling(
  incrementalSync: IncrementalSyncService,
  intervalMs: number = 30000 // 30 seconds
): NodeJS.Timeout {
  return setInterval(async () => {
    try {
      await incrementalSync.pollChanges();
    } catch (error) {
      console.error('Polling error:', error);
      // Don't throw - let next interval try again
    }
  }, intervalMs);
}
```

### Webhook Trigger Pattern

Since OpenFGA doesn't have native webhooks, implement at the application layer:

```typescript
// src/api/webhook-handler.ts

import { Router } from 'express';
import { FullSyncService } from '../sync/full-sync';

export function createWebhookRouter(syncService: FullSyncService): Router {
  const router = Router();

  // Called by microservices after writing tuples to OpenFGA
  router.post('/sync/user/:userId', async (req, res) => {
    const { userId } = req.params;

    try {
      const result = await syncService.sync([userId]);
      res.json({ success: true, result });
    } catch (error) {
      res.status(500).json({ success: false, error: error.message });
    }
  });

  // Called to trigger full sync (admin endpoint)
  router.post('/sync/full', async (req, res) => {
    const { userIds } = req.body; // Optional: specific users

    try {
      const allUsers = userIds || await getAllActiveUsers();
      const result = await syncService.sync(allUsers);
      res.json({ success: true, result });
    } catch (error) {
      res.status(500).json({ success: false, error: error.message });
    }
  });

  return router;
}
```

**Microservice Integration:**

```typescript
// In your microservice that grants data explorer access
async function grantDataExplorerAccess(
  userId: string,
  resourceType: 'tenant' | 'campus' | 'domain' | 'program',
  resourceId: string
): Promise<void> {
  // 1. Write tuple to OpenFGA
  await openfga.write({
    writes: [{
      user: `user:${userId}`,
      relation: 'can_view_data_explorer',
      object: `${resourceType}:${resourceId}`,
    }],
  });

  // 2. Trigger sync (fire-and-forget or await)
  fetch(`${SYNC_SERVICE_URL}/sync/user/${userId}`, {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${SERVICE_TOKEN}` },
  }).catch(err => console.error('Sync trigger failed:', err));
}

// Examples:
// Grant tenant-wide access
await grantDataExplorerAccess('alice', 'tenant', 'uc');

// Grant campus-level access
await grantDataExplorerAccess('bob', 'campus', 'USC');

// Grant domain-level access
await grantDataExplorerAccess('charlie', 'domain', 'safety');

// Grant program-level access
await grantDataExplorerAccess('dana', 'program', 'fire-safety-2024');
```

---

## Application Integration

### Setting User Context

```typescript
// src/snowflake/context.ts

import { Connection } from 'snowflake-sdk';

export async function setUserContext(
  connection: Connection,
  userId: string
): Promise<void> {
  // Set session variable for Row Access Policy
  await connection.execute({
    sqlText: `SET APP_USER_ID = ?`,
    binds: [userId],
  });
}

export async function clearUserContext(
  connection: Connection
): Promise<void> {
  // Clear session variable when returning to pool
  await connection.execute({
    sqlText: `UNSET APP_USER_ID`,
  });
}
```

### Connection Pooling Considerations

```typescript
// src/snowflake/pool.ts

import { createPool, Pool } from 'generic-pool';
import snowflake from 'snowflake-sdk';

interface PooledConnection {
  connection: snowflake.Connection;
  userId: string | null;
}

export function createSnowflakePool(): Pool<PooledConnection> {
  return createPool({
    create: async () => {
      const connection = snowflake.createConnection({
        account: process.env.SNOWFLAKE_ACCOUNT,
        username: process.env.SNOWFLAKE_USER,
        privateKey: process.env.SNOWFLAKE_PRIVATE_KEY,
        database: 'RSS_DEV',
        schema: 'INSPECT',
        role: 'APP_DASHBOARD_ROLE',
      });

      await new Promise<void>((resolve, reject) => {
        connection.connect((err) => {
          if (err) reject(err);
          else resolve();
        });
      });

      return { connection, userId: null };
    },

    destroy: async (pooled) => {
      await new Promise<void>((resolve) => {
        pooled.connection.destroy((err) => resolve());
      });
    },

    validate: async (pooled) => {
      // Ensure connection is still valid
      try {
        await pooled.connection.execute({ sqlText: 'SELECT 1' });
        return true;
      } catch {
        return false;
      }
    },
  }, {
    min: 2,
    max: 10,
    acquireTimeoutMillis: 30000,
    idleTimeoutMillis: 300000,
  });
}
```

### Query Execution Pattern

```typescript
// src/snowflake/query-executor.ts

import { Pool } from 'generic-pool';
import { setUserContext, clearUserContext } from './context';

export class QueryExecutor {
  constructor(private pool: Pool<PooledConnection>) {}

  async executeAsUser<T>(
    userId: string,
    sqlText: string,
    binds?: any[]
  ): Promise<T[]> {
    const pooled = await this.pool.acquire();

    try {
      // Set user context for Row Access Policy
      await setUserContext(pooled.connection, userId);
      pooled.userId = userId;

      // Execute query
      const result = await new Promise<T[]>((resolve, reject) => {
        pooled.connection.execute({
          sqlText,
          binds,
          complete: (err, stmt, rows) => {
            if (err) reject(err);
            else resolve(rows as T[]);
          },
        });
      });

      return result;
    } finally {
      // Clear context before returning to pool
      await clearUserContext(pooled.connection);
      pooled.userId = null;
      await this.pool.release(pooled);
    }
  }
}

// Usage example
const executor = new QueryExecutor(pool);

// In your API handler
app.get('/api/findings', async (req, res) => {
  const userId = req.user.sub; // From OAuth JWT

  const findings = await executor.executeAsUser(
    userId,
    `SELECT * FROM INSPECT.BI_FINDINGS WHERE RESPONSE_STATUS = ?`,
    ['NOT_RESOLVED']
  );

  res.json(findings);
});
```

---

## View Metadata and Discovery

### Business Requirement

The Next.js application needs to dynamically discover available BI views and display them with rich metadata (display name, description, category, icon, sort order). This enables:

- Dynamic report dropdown menus with descriptions
- Categorization and grouping of reports
- User-friendly display names (vs technical view names)
- Consistent UI without hardcoding view lists

### Approach: Snowflake Tags (Enterprise Edition)

With Enterprise Edition, Snowflake Tags are the recommended approach for view metadata. Tags provide:

- Native key-value metadata on database objects
- Full audit trail of metadata changes
- Queryable via `INFORMATION_SCHEMA` functions
- Validation via `ALLOWED_VALUES` constraints
- Inheritance (can set defaults at schema level)

### Tag Definition

```sql
-- ============================================================================
-- View Metadata Tags
-- Used by Next.js application to discover and display BI views
-- ============================================================================

-- Display name shown in UI (e.g., "Incident Reports")
CREATE TAG IF NOT EXISTS ADMIN.DISPLAY_NAME;

-- Description shown in dropdowns/tooltips
CREATE TAG IF NOT EXISTS ADMIN.DESCRIPTION;

-- Category for grouping reports
CREATE TAG IF NOT EXISTS ADMIN.CATEGORY
    ALLOWED_VALUES ('Safety', 'Training', 'Compliance', 'Admin', 'Inventory', 'Forms');

-- Sort order within category (as string, cast to INT when querying)
CREATE TAG IF NOT EXISTS ADMIN.SORT_ORDER;

-- Icon identifier for UI (e.g., "alert-triangle", "clipboard-check")
CREATE TAG IF NOT EXISTS ADMIN.ICON;

-- Whether view should be visible in report picker (default true)
CREATE TAG IF NOT EXISTS ADMIN.IS_VISIBLE
    ALLOWED_VALUES ('true', 'false');

-- Grant usage to roles that need to query metadata
GRANT USAGE ON TAG ADMIN.DISPLAY_NAME TO ROLE APP_DASHBOARD_ROLE;
GRANT USAGE ON TAG ADMIN.DESCRIPTION TO ROLE APP_DASHBOARD_ROLE;
GRANT USAGE ON TAG ADMIN.CATEGORY TO ROLE APP_DASHBOARD_ROLE;
GRANT USAGE ON TAG ADMIN.SORT_ORDER TO ROLE APP_DASHBOARD_ROLE;
GRANT USAGE ON TAG ADMIN.ICON TO ROLE APP_DASHBOARD_ROLE;
GRANT USAGE ON TAG ADMIN.IS_VISIBLE TO ROLE APP_DASHBOARD_ROLE;
```

### Applying Tags to Views

```sql
-- ============================================================================
-- Tag INSPECT schema views with metadata
-- ============================================================================

ALTER VIEW INSPECT.BI_FINDINGS SET TAG
    ADMIN.DISPLAY_NAME = 'Unresolved Findings',
    ADMIN.DESCRIPTION = 'Track and manage unresolved inspection findings requiring corrective action',
    ADMIN.CATEGORY = 'Safety',
    ADMIN.SORT_ORDER = '1',
    ADMIN.ICON = 'alert-circle',
    ADMIN.IS_VISIBLE = 'true';

ALTER VIEW INSPECT.BI_RESPONSE_DETAIL SET TAG
    ADMIN.DISPLAY_NAME = 'Response Detail',
    ADMIN.DESCRIPTION = 'Detailed view of all inspection responses with item-level data',
    ADMIN.CATEGORY = 'Safety',
    ADMIN.SORT_ORDER = '2',
    ADMIN.ICON = 'file-text',
    ADMIN.IS_VISIBLE = 'true';

ALTER VIEW INSPECT.BI_INSPECTION_STATUS SET TAG
    ADMIN.DISPLAY_NAME = 'Inspection Status',
    ADMIN.DESCRIPTION = 'Overview of inspection completion status by location and program',
    ADMIN.CATEGORY = 'Safety',
    ADMIN.SORT_ORDER = '3',
    ADMIN.ICON = 'check-square',
    ADMIN.IS_VISIBLE = 'true';

ALTER VIEW INSPECT.BI_CHECKLIST_ITEMS_SUMMARY SET TAG
    ADMIN.DISPLAY_NAME = 'Checklist Items Summary',
    ADMIN.DESCRIPTION = 'Aggregated checklist item responses across all inspections',
    ADMIN.CATEGORY = 'Safety',
    ADMIN.SORT_ORDER = '4',
    ADMIN.ICON = 'list-checks',
    ADMIN.IS_VISIBLE = 'true';

ALTER VIEW INSPECT.BI_TOP_RANK_BY_ISSUES SET TAG
    ADMIN.DISPLAY_NAME = 'Top Locations by Issues',
    ADMIN.DESCRIPTION = 'Locations ranked by number of inspection issues identified',
    ADMIN.CATEGORY = 'Safety',
    ADMIN.SORT_ORDER = '5',
    ADMIN.ICON = 'bar-chart-2',
    ADMIN.IS_VISIBLE = 'true';

ALTER VIEW INSPECT.BI_ADMIN_CHECKLIST SET TAG
    ADMIN.DISPLAY_NAME = 'Admin Checklist',
    ADMIN.DESCRIPTION = 'Administrative view of checklist configurations and usage',
    ADMIN.CATEGORY = 'Admin',
    ADMIN.SORT_ORDER = '1',
    ADMIN.ICON = 'settings',
    ADMIN.IS_VISIBLE = 'true';

ALTER VIEW INSPECT.BI_ADMIN_LOCATION_LISTS SET TAG
    ADMIN.DISPLAY_NAME = 'Location Lists Manager',
    ADMIN.DESCRIPTION = 'Manage and review location list assignments for inspection programs',
    ADMIN.CATEGORY = 'Admin',
    ADMIN.SORT_ORDER = '2',
    ADMIN.ICON = 'map-pin',
    ADMIN.IS_VISIBLE = 'true';
```

### Querying View Metadata

```sql
-- ============================================================================
-- Query to retrieve all visible views with metadata for Next.js app
-- Returns views the current user has access to with their metadata tags
-- ============================================================================

SELECT
    v.TABLE_SCHEMA AS schema_name,
    v.TABLE_NAME AS view_name,
    MAX(CASE WHEN t.TAG_NAME = 'DISPLAY_NAME' THEN t.TAG_VALUE END) AS display_name,
    MAX(CASE WHEN t.TAG_NAME = 'DESCRIPTION' THEN t.TAG_VALUE END) AS description,
    MAX(CASE WHEN t.TAG_NAME = 'CATEGORY' THEN t.TAG_VALUE END) AS category,
    MAX(CASE WHEN t.TAG_NAME = 'SORT_ORDER' THEN t.TAG_VALUE END)::INT AS sort_order,
    MAX(CASE WHEN t.TAG_NAME = 'ICON' THEN t.TAG_VALUE END) AS icon
FROM INFORMATION_SCHEMA.VIEWS v
LEFT JOIN TABLE(
    INFORMATION_SCHEMA.TAG_REFERENCES_ALL_COLUMNS(
        v.TABLE_SCHEMA || '.' || v.TABLE_NAME,
        'VIEW'
    )
) t ON TRUE
WHERE v.TABLE_SCHEMA IN ('INSPECT', 'INVENTORY', 'FORMS')
  AND (
      MAX(CASE WHEN t.TAG_NAME = 'IS_VISIBLE' THEN t.TAG_VALUE END) IS NULL
      OR MAX(CASE WHEN t.TAG_NAME = 'IS_VISIBLE' THEN t.TAG_VALUE END) = 'true'
  )
GROUP BY v.TABLE_SCHEMA, v.TABLE_NAME
ORDER BY category, sort_order, display_name;
```

### Optimized Query Function

For better performance, create a table function:

```sql
-- ============================================================================
-- GET_AVAILABLE_REPORTS: Returns all visible reports with metadata
-- Called by Next.js API to populate report picker
-- ============================================================================

CREATE OR REPLACE FUNCTION AUTH.GET_AVAILABLE_REPORTS(p_schemas ARRAY)
RETURNS TABLE (
    SCHEMA_NAME VARCHAR,
    VIEW_NAME VARCHAR,
    DISPLAY_NAME VARCHAR,
    DESCRIPTION VARCHAR,
    CATEGORY VARCHAR,
    SORT_ORDER INT,
    ICON VARCHAR
)
AS
$$
    WITH tagged_views AS (
        SELECT
            obj.VALUE::VARCHAR AS full_name,
            SPLIT_PART(obj.VALUE::VARCHAR, '.', 1) AS schema_name,
            SPLIT_PART(obj.VALUE::VARCHAR, '.', 2) AS view_name
        FROM TABLE(FLATTEN(
            SELECT ARRAY_AGG(TABLE_SCHEMA || '.' || TABLE_NAME)
            FROM INFORMATION_SCHEMA.VIEWS
            WHERE ARRAY_CONTAINS(TABLE_SCHEMA::VARIANT, p_schemas)
        )) obj
    ),
    view_tags AS (
        SELECT
            tv.schema_name,
            tv.view_name,
            t.TAG_NAME,
            t.TAG_VALUE
        FROM tagged_views tv,
        TABLE(INFORMATION_SCHEMA.TAG_REFERENCES(tv.full_name, 'VIEW')) t
    )
    SELECT
        schema_name,
        view_name,
        MAX(CASE WHEN TAG_NAME = 'DISPLAY_NAME' THEN TAG_VALUE END) AS display_name,
        MAX(CASE WHEN TAG_NAME = 'DESCRIPTION' THEN TAG_VALUE END) AS description,
        MAX(CASE WHEN TAG_NAME = 'CATEGORY' THEN TAG_VALUE END) AS category,
        MAX(CASE WHEN TAG_NAME = 'SORT_ORDER' THEN TAG_VALUE END)::INT AS sort_order,
        MAX(CASE WHEN TAG_NAME = 'ICON' THEN TAG_VALUE END) AS icon
    FROM view_tags
    GROUP BY schema_name, view_name
    HAVING COALESCE(MAX(CASE WHEN TAG_NAME = 'IS_VISIBLE' THEN TAG_VALUE END), 'true') = 'true'
    ORDER BY category, sort_order, display_name
$$;

GRANT USAGE ON FUNCTION AUTH.GET_AVAILABLE_REPORTS(ARRAY) TO ROLE APP_DASHBOARD_ROLE;

-- Usage:
-- SELECT * FROM TABLE(AUTH.GET_AVAILABLE_REPORTS(ARRAY_CONSTRUCT('INSPECT', 'INVENTORY', 'FORMS')));
```

### Next.js Integration

```typescript
// src/lib/snowflake/reports.ts

interface ReportMetadata {
  schemaName: string;
  viewName: string;
  displayName: string;
  description: string;
  category: string;
  sortOrder: number;
  icon: string;
}

interface ReportsByCategory {
  [category: string]: ReportMetadata[];
}

export async function getAvailableReports(
  userId: string,
  schemas: string[] = ['INSPECT', 'INVENTORY', 'FORMS']
): Promise<ReportsByCategory> {
  const executor = new QueryExecutor(pool);

  const reports = await executor.executeAsUser<ReportMetadata>(
    userId,
    `SELECT * FROM TABLE(AUTH.GET_AVAILABLE_REPORTS(?))`,
    [schemas]
  );

  // Group by category for UI
  return reports.reduce((acc, report) => {
    const category = report.category || 'Other';
    if (!acc[category]) {
      acc[category] = [];
    }
    acc[category].push(report);
    return acc;
  }, {} as ReportsByCategory);
}

// API route: /api/reports
export async function GET(request: Request) {
  const userId = await getCurrentUserId(request);
  const reports = await getAvailableReports(userId);

  return Response.json(reports);
}
```

### React Component Example

```tsx
// components/ReportPicker.tsx

interface ReportPickerProps {
  onSelect: (schemaName: string, viewName: string) => void;
}

export function ReportPicker({ onSelect }: ReportPickerProps) {
  const { data: reportsByCategory, isLoading } = useQuery({
    queryKey: ['reports'],
    queryFn: () => fetch('/api/reports').then(r => r.json()),
  });

  if (isLoading) return <Skeleton />;

  return (
    <DropdownMenu>
      <DropdownMenuTrigger>
        <Button variant="outline">
          <FileText className="mr-2 h-4 w-4" />
          Select Report
          <ChevronDown className="ml-2 h-4 w-4" />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent className="w-80">
        {Object.entries(reportsByCategory).map(([category, reports]) => (
          <DropdownMenuGroup key={category}>
            <DropdownMenuLabel>{category}</DropdownMenuLabel>
            {reports.map((report) => (
              <DropdownMenuItem
                key={`${report.schemaName}.${report.viewName}`}
                onClick={() => onSelect(report.schemaName, report.viewName)}
              >
                <div className="flex flex-col">
                  <div className="flex items-center">
                    <Icon name={report.icon} className="mr-2 h-4 w-4" />
                    <span className="font-medium">{report.displayName}</span>
                  </div>
                  <span className="text-sm text-muted-foreground">
                    {report.description}
                  </span>
                </div>
              </DropdownMenuItem>
            ))}
            <DropdownMenuSeparator />
          </DropdownMenuGroup>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

### Schema-Level Default Tags

Set default category for all views in a schema:

```sql
-- All views in INSPECT schema default to 'Safety' category
ALTER SCHEMA INSPECT SET TAG ADMIN.CATEGORY = 'Safety';

-- All views in INVENTORY schema default to 'Inventory' category
ALTER SCHEMA INVENTORY SET TAG ADMIN.CATEGORY = 'Inventory';

-- Individual views can override the schema-level default
ALTER VIEW INSPECT.BI_ADMIN_CHECKLIST SET TAG ADMIN.CATEGORY = 'Admin';
```

### Governance and Audit

```sql
-- Query tag change history (Enterprise Edition)
SELECT
    TAG_NAME,
    TAG_VALUE,
    OBJECT_NAME,
    DOMAIN,  -- 'VIEW', 'TABLE', etc.
    CREATED,
    DELETED
FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES_HISTORY
WHERE TAG_SCHEMA = 'ADMIN'
ORDER BY CREATED DESC
LIMIT 100;

-- Find views missing required tags
SELECT
    v.TABLE_SCHEMA,
    v.TABLE_NAME
FROM INFORMATION_SCHEMA.VIEWS v
WHERE v.TABLE_SCHEMA IN ('INSPECT', 'INVENTORY', 'FORMS')
  AND NOT EXISTS (
      SELECT 1
      FROM TABLE(INFORMATION_SCHEMA.TAG_REFERENCES(
          v.TABLE_SCHEMA || '.' || v.TABLE_NAME, 'VIEW'
      ))
      WHERE TAG_NAME = 'DISPLAY_NAME'
  );
```

### Alternative: JSON Comment (Non-Enterprise)

For Standard Edition without Tags, use structured JSON in view comments:

```sql
-- Set structured metadata in comment
COMMENT ON VIEW INSPECT.BI_FINDINGS IS '{
  "displayName": "Unresolved Findings",
  "description": "Track and manage unresolved inspection findings requiring corrective action",
  "category": "Safety",
  "sortOrder": 1,
  "icon": "alert-circle"
}';

-- Query with JSON parsing
SELECT
    TABLE_SCHEMA,
    TABLE_NAME,
    TRY_PARSE_JSON(COMMENT):displayName::VARCHAR AS display_name,
    TRY_PARSE_JSON(COMMENT):description::VARCHAR AS description,
    TRY_PARSE_JSON(COMMENT):category::VARCHAR AS category,
    TRY_PARSE_JSON(COMMENT):sortOrder::INT AS sort_order,
    TRY_PARSE_JSON(COMMENT):icon::VARCHAR AS icon
FROM INFORMATION_SCHEMA.VIEWS
WHERE TABLE_SCHEMA IN ('INSPECT', 'INVENTORY', 'FORMS')
  AND TRY_PARSE_JSON(COMMENT) IS NOT NULL
ORDER BY category, sort_order;
```

### Comparison: Tags vs JSON Comment

| Aspect | Snowflake Tags | JSON Comment |
|--------|----------------|--------------|
| Edition Required | Enterprise | Standard |
| Structure | Native key-value | Ad-hoc JSON |
| Validation | `ALLOWED_VALUES` | None |
| Audit Trail | Full history | None |
| Querying | Native functions | JSON parsing |
| Inheritance | Schema → View | None |
| Maintenance | More setup | Simpler |

**Recommendation:** Use Tags with Enterprise Edition for production. JSON Comment is acceptable for prototyping or Standard Edition deployments.

---

## Permission-Based Report Discovery

### The Core Challenge

The architecture solves two distinct concerns:

1. **Data access** (solved by RAP): Row Access Policy filters data based on user's tenant/campus/domain/program permissions
2. **Report visibility** (this section): Which reports appear in the dropdown for a given user

The question: **What determines if a user can "see" a report in the dropdown?**

### Options Analysis

#### Option 1: Universal Access (Recommended for MVP)

**Principle:** If a user has `can_view_data_explorer` at ANY level, they see ALL reports. RAP handles data filtering.

```
User has permission at ANY level → Show ALL tagged views → RAP filters data
```

**Why this makes sense:**
- All INSPECT views share the same hierarchical columns (TENANT, CAMPUS, PROGRAM_ID)
- A user with program-level access sees the same *reports* as a tenant admin, just less *data*
- Empty results are informative ("no findings yet" is useful information)
- Zero additional infrastructure required

**Pros:** Simple, uses existing sync + tags, no new infrastructure
**Cons:** Can't hide specific reports from certain users

#### Option 2: Category-Gated Access

**Principle:** Certain report categories require elevated permissions. Map categories to OpenFGA relations.

```
Category "Admin" → requires configuration_manager
Category "Safety" → requires can_view_data_explorer
```

**Implementation requires:**
1. New `REQUIRED_PERMISSION` tag on views
2. Syncing additional relations (e.g., `configuration_manager`) to `AUTH.USER_PERMISSIONS`
3. Adding `PERMISSION_TYPE` column to permissions table

```sql
-- New tag for permission requirement
CREATE TAG IF NOT EXISTS ADMIN.REQUIRED_PERMISSION
    ALLOWED_VALUES ('can_view_data_explorer', 'configuration_manager', 'admin');

-- Tag admin views with higher requirement
ALTER VIEW INSPECT.BI_ADMIN_CHECKLIST SET TAG
    ADMIN.REQUIRED_PERMISSION = 'configuration_manager';

-- Safety views use default
ALTER VIEW INSPECT.BI_FINDINGS SET TAG
    ADMIN.REQUIRED_PERMISSION = 'can_view_data_explorer';
```

**Extended permissions table:**
```sql
-- Add PERMISSION_TYPE column to track which relation was granted
ALTER TABLE AUTH.USER_PERMISSIONS
    ADD COLUMN PERMISSION_TYPE VARCHAR(50) DEFAULT 'can_view_data_explorer';

-- Example data after sync:
-- USER_ID | PERMISSION_SCOPE | RESOURCE_ID | PERMISSION_TYPE
-- alice   | tenant           | uc          | can_view_data_explorer
-- alice   | tenant           | uc          | configuration_manager  ← admin also synced
-- bob     | campus           | USC         | can_view_data_explorer
```

**Pros:** Fine-grained control by category
**Cons:** Requires syncing additional relations, more complexity

#### Option 3: Schema-Based Access

**Principle:** Permissions are granted per data domain (schema). Users only see reports in schemas they have access to.

**New OpenFGA relations:**
```yaml
type tenant
  relations
    define can_view_data_explorer: [user, client] or configuration_manager
    # Schema-level grants
    define can_view_inspect: [user] or can_view_data_explorer
    define can_view_inventory: [user] or can_view_data_explorer
    define can_view_forms: [user] or can_view_data_explorer
```

**Usage:**
```bash
# Grant access to all schemas (default behavior)
fga tuple write user:alice can_view_data_explorer tenant:uc

# Or grant access to specific schema only
fga tuple write user:bob can_view_inspect tenant:uc
# Bob can only see INSPECT reports, not INVENTORY or FORMS
```

**Pros:** Clean separation by data domain
**Cons:** More relations to manage, most users probably want all schemas

#### Option 4: Data-Aware Visibility

**Principle:** Only show reports that would return at least one row for this user.

**Approach A: Background Pre-computation**
```sql
-- Cache which users can see which views
CREATE TABLE AUTH.USER_VIEW_ACCESS (
    USER_ID VARCHAR,
    SCHEMA_NAME VARCHAR,
    VIEW_NAME VARCHAR,
    HAS_DATA BOOLEAN,
    COMPUTED_AT TIMESTAMP_NTZ,
    PRIMARY KEY (USER_ID, SCHEMA_NAME, VIEW_NAME)
);

-- Background job checks each user/view combination periodically
```

**Approach B: Lazy Evaluation with Caching**
```typescript
// First request: return all views
// After user queries a view: cache whether it returned data
// Subsequent requests: filter based on cache

async function getReportsForUser(userId: string): Promise<Report[]> {
  const allReports = await getTaggedReports();
  const cache = await getViewAccessCache(userId);

  return allReports.map(report => ({
    ...report,
    hasData: cache[report.viewName] ?? null, // null = unknown
  }));
}
```

**Pros:** Best UX - users only see relevant reports
**Cons:** Computationally expensive, cache staleness issues

### Recommended Approach: Phased Implementation

| Phase | Approach | Trigger |
|-------|----------|---------|
| **Phase 1 (MVP)** | Option 1 - Universal Access | Initial launch |
| **Phase 2** | Option 2 - Category-Gated | Need to restrict admin views |
| **Phase 3** | Option 4 - Data-Aware | Users complain about empty reports |

### Data Points Required

| Data Point | Source | Status |
|------------|--------|--------|
| User identity | OAuth JWT `sub` claim | Existing |
| User's permissions | `AUTH.USER_PERMISSIONS` (synced from OpenFGA) | Existing |
| View metadata | Snowflake Tags on views | Existing |
| View list | `INFORMATION_SCHEMA.VIEWS` | Existing |

**No additional infrastructure needed for Phase 1!**

### Complete Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Report Picker Flow                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. User logs in                                                             │
│     └─▶ OAuth provides user ID (e.g., "alice")                              │
│                                                                              │
│  2. Background: Sync service has already materialized permissions           │
│     └─▶ AUTH.USER_PERMISSIONS contains:                                     │
│         | USER_ID | PERMISSION_SCOPE | RESOURCE_ID |                        │
│         | alice   | tenant           | uc          |                        │
│                                                                              │
│  3. Next.js calls GET /api/reports                                          │
│     └─▶ Sets APP_USER_ID = 'alice'                                          │
│     └─▶ Calls AUTH.GET_USER_REPORTS('alice')                                │
│                                                                              │
│  4. Function checks: Does alice have ANY permission?                         │
│     └─▶ Yes (tenant:uc) → Return ALL tagged views with metadata             │
│                                                                              │
│  5. Returns to UI:                                                           │
│     ┌──────────────────────────────────────────────────────────┐            │
│     │ Category: Safety                                          │            │
│     │   • Unresolved Findings - Track inspection findings...    │            │
│     │   • Inspection Status - Overview of completion...         │            │
│     │   • Response Detail - Detailed view of responses...       │            │
│     │ Category: Admin                                           │            │
│     │   • Admin Checklist - Administrative view of...           │            │
│     │   • Location Lists Manager - Manage location...           │            │
│     └──────────────────────────────────────────────────────────┘            │
│                                                                              │
│  6. User selects "Unresolved Findings"                                       │
│     └─▶ Query: SELECT * FROM INSPECT.BI_FINDINGS                            │
│     └─▶ RAP filters to UC data only                                         │
│     └─▶ User sees their filtered data                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation: GET_USER_REPORTS Function

```sql
-- ============================================================================
-- GET_USER_REPORTS: Ties together permissions and view metadata
-- Returns reports a user can access based on their OpenFGA permissions
-- ============================================================================

CREATE OR REPLACE FUNCTION AUTH.GET_USER_REPORTS(p_user_id VARCHAR)
RETURNS TABLE (
    SCHEMA_NAME VARCHAR,
    VIEW_NAME VARCHAR,
    DISPLAY_NAME VARCHAR,
    DESCRIPTION VARCHAR,
    CATEGORY VARCHAR,
    SORT_ORDER INT,
    ICON VARCHAR
)
MEMOIZABLE  -- Cache results per session for performance
AS
$$
    SELECT
        v.TABLE_SCHEMA AS SCHEMA_NAME,
        v.TABLE_NAME AS VIEW_NAME,
        MAX(CASE WHEN t.TAG_NAME = 'DISPLAY_NAME' THEN t.TAG_VALUE END) AS DISPLAY_NAME,
        MAX(CASE WHEN t.TAG_NAME = 'DESCRIPTION' THEN t.TAG_VALUE END) AS DESCRIPTION,
        MAX(CASE WHEN t.TAG_NAME = 'CATEGORY' THEN t.TAG_VALUE END) AS CATEGORY,
        MAX(CASE WHEN t.TAG_NAME = 'SORT_ORDER' THEN t.TAG_VALUE END)::INT AS SORT_ORDER,
        MAX(CASE WHEN t.TAG_NAME = 'ICON' THEN t.TAG_VALUE END) AS ICON
    FROM INFORMATION_SCHEMA.VIEWS v
    LEFT JOIN TABLE(
        INFORMATION_SCHEMA.TAG_REFERENCES(v.TABLE_SCHEMA || '.' || v.TABLE_NAME, 'VIEW')
    ) t ON TRUE
    WHERE v.TABLE_SCHEMA IN ('INSPECT', 'INVENTORY', 'FORMS')
      -- Only return if user has at least one permission
      AND EXISTS (
          SELECT 1 FROM AUTH.USER_PERMISSIONS
          WHERE USER_ID = p_user_id
      )
    GROUP BY v.TABLE_SCHEMA, v.TABLE_NAME
    -- Filter to visible views only
    HAVING COALESCE(
        MAX(CASE WHEN t.TAG_NAME = 'IS_VISIBLE' THEN t.TAG_VALUE END),
        'true'
    ) = 'true'
    ORDER BY CATEGORY, SORT_ORDER, DISPLAY_NAME
$$;

GRANT USAGE ON FUNCTION AUTH.GET_USER_REPORTS(VARCHAR) TO ROLE APP_DASHBOARD_ROLE;

COMMENT ON FUNCTION AUTH.GET_USER_REPORTS(VARCHAR) IS
    'Returns all reports a user can access based on their OpenFGA permissions.
     Combines AUTH.USER_PERMISSIONS check with view metadata from Snowflake Tags.
     Memoizable for session-level caching.';
```

### Next.js API Integration

```typescript
// src/app/api/reports/route.ts

import { NextResponse } from 'next/server';
import { getCurrentUserId } from '@/lib/auth';
import { snowflake } from '@/lib/snowflake';

interface Report {
  schemaName: string;
  viewName: string;
  displayName: string;
  description: string;
  category: string;
  sortOrder: number;
  icon: string;
}

interface ReportsByCategory {
  [category: string]: Report[];
}

export async function GET(request: Request) {
  const userId = await getCurrentUserId(request);

  if (!userId) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const connection = await snowflake.getConnection();

  try {
    // Set user context
    await connection.execute({ sqlText: `SET APP_USER_ID = ?`, binds: [userId] });

    // Get reports user can access
    const reports = await connection.execute<Report>({
      sqlText: `SELECT * FROM TABLE(AUTH.GET_USER_REPORTS(?))`,
      binds: [userId],
    });

    // Group by category for UI
    const grouped = reports.reduce((acc, report) => {
      const category = report.category || 'Other';
      if (!acc[category]) {
        acc[category] = [];
      }
      acc[category].push({
        schemaName: report.SCHEMA_NAME,
        viewName: report.VIEW_NAME,
        displayName: report.DISPLAY_NAME,
        description: report.DESCRIPTION,
        category: report.CATEGORY,
        sortOrder: report.SORT_ORDER,
        icon: report.ICON,
      });
      return acc;
    }, {} as ReportsByCategory);

    return NextResponse.json(grouped);
  } finally {
    await connection.execute({ sqlText: `UNSET APP_USER_ID` });
    connection.release();
  }
}
```

### React Hook for Report Picker

```typescript
// src/hooks/useReports.ts

import { useQuery } from '@tanstack/react-query';

interface Report {
  schemaName: string;
  viewName: string;
  displayName: string;
  description: string;
  category: string;
  sortOrder: number;
  icon: string;
}

interface ReportsByCategory {
  [category: string]: Report[];
}

export function useReports() {
  return useQuery<ReportsByCategory>({
    queryKey: ['reports'],
    queryFn: async () => {
      const response = await fetch('/api/reports');
      if (!response.ok) {
        throw new Error('Failed to fetch reports');
      }
      return response.json();
    },
    staleTime: 5 * 60 * 1000, // Cache for 5 minutes
  });
}

// Usage in component
export function ReportPicker({ onSelect }: { onSelect: (report: Report) => void }) {
  const { data: reportsByCategory, isLoading, error } = useReports();

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  if (!reportsByCategory || Object.keys(reportsByCategory).length === 0) {
    return <EmptyState message="No reports available" />;
  }

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="outline">
          Select Report
          <ChevronDown className="ml-2 h-4 w-4" />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent className="w-80">
        {Object.entries(reportsByCategory).map(([category, reports]) => (
          <DropdownMenuGroup key={category}>
            <DropdownMenuLabel>{category}</DropdownMenuLabel>
            {reports.map((report) => (
              <DropdownMenuItem
                key={`${report.schemaName}.${report.viewName}`}
                onClick={() => onSelect(report)}
              >
                <div className="flex flex-col gap-1">
                  <span className="font-medium">{report.displayName}</span>
                  <span className="text-xs text-muted-foreground line-clamp-2">
                    {report.description}
                  </span>
                </div>
              </DropdownMenuItem>
            ))}
            <DropdownMenuSeparator />
          </DropdownMenuGroup>
        ))}
      </DropdownMenuContent>
    </DropdownMenu>
  );
}
```

### Phase 2: Category-Gated Implementation

If you need to restrict certain reports to elevated permissions:

```sql
-- ============================================================================
-- GET_USER_REPORTS_GATED: Category-gated version
-- Filters reports based on required permission level
-- ============================================================================

CREATE OR REPLACE FUNCTION AUTH.GET_USER_REPORTS_GATED(p_user_id VARCHAR)
RETURNS TABLE (
    SCHEMA_NAME VARCHAR,
    VIEW_NAME VARCHAR,
    DISPLAY_NAME VARCHAR,
    DESCRIPTION VARCHAR,
    CATEGORY VARCHAR,
    SORT_ORDER INT,
    ICON VARCHAR
)
AS
$$
    WITH user_permissions AS (
        -- Get all permission types this user has
        SELECT DISTINCT PERMISSION_TYPE
        FROM AUTH.USER_PERMISSIONS
        WHERE USER_ID = p_user_id
    ),
    tagged_views AS (
        SELECT
            v.TABLE_SCHEMA AS SCHEMA_NAME,
            v.TABLE_NAME AS VIEW_NAME,
            MAX(CASE WHEN t.TAG_NAME = 'DISPLAY_NAME' THEN t.TAG_VALUE END) AS DISPLAY_NAME,
            MAX(CASE WHEN t.TAG_NAME = 'DESCRIPTION' THEN t.TAG_VALUE END) AS DESCRIPTION,
            MAX(CASE WHEN t.TAG_NAME = 'CATEGORY' THEN t.TAG_VALUE END) AS CATEGORY,
            MAX(CASE WHEN t.TAG_NAME = 'SORT_ORDER' THEN t.TAG_VALUE END)::INT AS SORT_ORDER,
            MAX(CASE WHEN t.TAG_NAME = 'ICON' THEN t.TAG_VALUE END) AS ICON,
            COALESCE(
                MAX(CASE WHEN t.TAG_NAME = 'REQUIRED_PERMISSION' THEN t.TAG_VALUE END),
                'can_view_data_explorer'  -- Default permission level
            ) AS REQUIRED_PERMISSION,
            COALESCE(
                MAX(CASE WHEN t.TAG_NAME = 'IS_VISIBLE' THEN t.TAG_VALUE END),
                'true'
            ) AS IS_VISIBLE
        FROM INFORMATION_SCHEMA.VIEWS v
        LEFT JOIN TABLE(
            INFORMATION_SCHEMA.TAG_REFERENCES(v.TABLE_SCHEMA || '.' || v.TABLE_NAME, 'VIEW')
        ) t ON TRUE
        WHERE v.TABLE_SCHEMA IN ('INSPECT', 'INVENTORY', 'FORMS')
        GROUP BY v.TABLE_SCHEMA, v.TABLE_NAME
    )
    SELECT
        SCHEMA_NAME,
        VIEW_NAME,
        DISPLAY_NAME,
        DESCRIPTION,
        CATEGORY,
        SORT_ORDER,
        ICON
    FROM tagged_views
    WHERE IS_VISIBLE = 'true'
      AND REQUIRED_PERMISSION IN (SELECT PERMISSION_TYPE FROM user_permissions)
    ORDER BY CATEGORY, SORT_ORDER, DISPLAY_NAME
$$;

COMMENT ON FUNCTION AUTH.GET_USER_REPORTS_GATED(VARCHAR) IS
    'Returns reports filtered by required permission level.
     Views tagged with REQUIRED_PERMISSION are only shown if user has that permission type.
     Requires PERMISSION_TYPE column in AUTH.USER_PERMISSIONS.';
```

### Sync Service Update for Phase 2

```typescript
// src/sync/full-sync.ts - Extended for multiple permission types

const PERMISSION_TYPES = ['can_view_data_explorer', 'configuration_manager'] as const;

async function getUserPermissions(userId: string): Promise<Permission[]> {
  const permissions: Permission[] = [];

  // Query each permission type
  for (const permissionType of PERMISSION_TYPES) {
    for (const resourceType of ['tenant', 'campus', 'domain', 'program'] as const) {
      const response = await openfga.listObjects({
        user: `user:${userId}`,
        relation: permissionType,
        type: resourceType,
      });

      for (const object of response.objects) {
        permissions.push({
          userId,
          scope: resourceType,
          resourceId: object.split(':')[1],
          permissionType,  // NEW: track which relation
        });
      }
    }
  }

  return permissions;
}
```

---

## Scalability Analysis

### User Scale Projections

| Users | Full Sync Time | Incremental Sync | Permissions Table Size |
|-------|----------------|------------------|------------------------|
| 100 | ~30 seconds | < 5 seconds | ~10 KB |
| 500 | ~2 minutes | < 10 seconds | ~50 KB |
| 1,000 | ~5 minutes | < 15 seconds | ~100 KB |
| 5,000 | ~20 minutes | < 30 seconds | ~500 KB |
| 10,000 | ~45 minutes | < 1 minute | ~1 MB |

**Assumptions:**
- Average 10 permissions per user
- OpenFGA ListObjects: ~100ms per call
- Snowflake write: ~50ms per batch of 100 records

### Query Performance

| Scenario | Without Memoization | With Memoization |
|----------|---------------------|------------------|
| First query | 50-100ms | 50-100ms |
| Subsequent queries (same session) | 50-100ms | 5-15ms |
| Large result set (10k rows) | 200-500ms | 100-200ms |

**Optimization Tips:**
1. Use memoizable function (implemented above)
2. Cluster permissions table by USER_ID
3. Consider search optimization service for very large tables

### Sync Performance

```typescript
// Performance tuning options

const syncConfig = {
  // OpenFGA
  pageSize: 100,           // Max allowed by OpenFGA
  maxConcurrency: 10,      // Parallel user processing

  // Snowflake
  batchSize: 1000,         // Records per INSERT

  // Polling
  pollIntervalMs: 30000,   // 30 seconds

  // Retry
  maxRetries: 3,
  retryDelayMs: 1000,
};
```

### Storage Requirements

```
Per permission record:
- USER_ID: 255 bytes (VARCHAR)
- PERMISSION_SCOPE: 20 bytes (VARCHAR)
- RESOURCE_ID: 255 bytes (VARCHAR)
- SYNCED_AT: 8 bytes (TIMESTAMP)
- SYNC_BATCH_ID: 36 bytes (UUID)
- Overhead: ~50 bytes

Total: ~624 bytes per record (compressed: ~100-150 bytes)

Estimates:
- 1,000 users × 10 permissions = 10,000 records ≈ 1-1.5 MB
- 10,000 users × 10 permissions = 100,000 records ≈ 10-15 MB
```

---

## Operational Considerations

### Monitoring

```typescript
// src/monitoring/metrics.ts

import { Counter, Histogram, Gauge } from 'prom-client';

export const metrics = {
  // Sync metrics
  syncDuration: new Histogram({
    name: 'permission_sync_duration_seconds',
    help: 'Duration of permission sync operations',
    labelNames: ['type'], // 'full' or 'incremental'
  }),

  syncErrors: new Counter({
    name: 'permission_sync_errors_total',
    help: 'Total number of sync errors',
    labelNames: ['type', 'error_type'],
  }),

  permissionsTotal: new Gauge({
    name: 'permissions_total',
    help: 'Total number of permissions in Snowflake',
  }),

  lastSyncTimestamp: new Gauge({
    name: 'permission_sync_last_timestamp',
    help: 'Timestamp of last successful sync',
    labelNames: ['type'],
  }),

  // OpenFGA metrics
  openfgaCallDuration: new Histogram({
    name: 'openfga_call_duration_seconds',
    help: 'Duration of OpenFGA API calls',
    labelNames: ['operation'],
  }),
};
```

### Alerting

| Alert | Condition | Severity |
|-------|-----------|----------|
| Sync Failure | Full sync failed 3 consecutive times | Critical |
| Sync Lag | No successful sync in 2 hours | Warning |
| OpenFGA Unavailable | Connection failures > 5/minute | Critical |
| Permission Count Drop | Permissions decreased by > 50% | Warning |
| Snowflake Write Failure | Insert/merge failures | Critical |

### Consistency Guarantees

**Eventually Consistent Model:**

```
Timeline:
  T0: Admin grants access in UI
  T1: Microservice writes tuple to OpenFGA (~10ms)
  T2: Webhook triggers sync service (~50ms)
  T3: Sync service queries OpenFGA (~100ms)
  T4: Sync service writes to Snowflake (~100ms)
  T5: User can see data (~260ms total)

Worst case (scheduled sync only):
  T0: Admin grants access
  ... (up to 1 hour) ...
  T1: Scheduled sync runs
  T2: User can see data
```

**Handling Stale Permissions:**

```typescript
// Show "permissions updating" in UI
async function checkPermissionFreshness(userId: string): Promise<boolean> {
  const result = await snowflake.execute({
    sqlText: `
      SELECT MAX(SYNCED_AT) as last_sync
      FROM INSPECT.USER_PERMISSIONS
      WHERE USER_ID = ?
    `,
    binds: [userId],
  });

  const lastSync = result[0]?.last_sync;
  const maxAge = 5 * 60 * 1000; // 5 minutes

  return lastSync && (Date.now() - new Date(lastSync).getTime()) < maxAge;
}
```

---

## Security Considerations

### Service Authentication

```typescript
// Sync service should use service account
const snowflakeConfig = {
  account: process.env.SNOWFLAKE_ACCOUNT,
  username: 'PERMISSION_SYNC_SERVICE', // Dedicated service user
  privateKey: process.env.SNOWFLAKE_PRIVATE_KEY,
  role: 'MSK_CONNECTOR_ROLE', // Role with write access to USER_PERMISSIONS
};
```

### Least Privilege

```sql
-- Create dedicated role for sync service
CREATE ROLE IF NOT EXISTS PERMISSION_SYNC_ROLE;

-- Grant only necessary permissions
GRANT USAGE ON DATABASE RSS_DEV TO ROLE PERMISSION_SYNC_ROLE;
GRANT USAGE ON SCHEMA AUTH TO ROLE PERMISSION_SYNC_ROLE;
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE AUTH.USER_PERMISSIONS
    TO ROLE PERMISSION_SYNC_ROLE;

-- Create service user
CREATE USER IF NOT EXISTS PERMISSION_SYNC_SERVICE
    TYPE = SERVICE
    DEFAULT_ROLE = PERMISSION_SYNC_ROLE
    RSA_PUBLIC_KEY = '...';

GRANT ROLE PERMISSION_SYNC_ROLE TO USER PERMISSION_SYNC_SERVICE;
```

### Audit Logging

```sql
-- Enable access history for compliance
-- (Requires Enterprise Edition)
ALTER ACCOUNT SET ENABLE_ACCOUNT_USAGE_HISTORY = TRUE;

-- Query access history
SELECT *
FROM SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY
WHERE BASE_OBJECTS_ACCESSED LIKE '%USER_PERMISSIONS%'
ORDER BY QUERY_START_TIME DESC;
```

---

## Tenant Data Export (Service Accounts)

### Business Case

Beyond end-user access through the web application, there's a need to provide **direct Snowflake access to tenants** for bulk data export. This enables clients to:

- Perform full data dumps into their own systems/databases
- Run custom analytics and reporting outside the web application
- Integrate with their internal ETL pipelines
- Maintain their own data warehouse replica

**Key Differences from User Access:**

| Aspect | User Access (Web App) | Tenant Export (Service Account) |
|--------|----------------------|--------------------------------|
| Authentication | OAuth → Web App → Snowflake | Direct Snowflake (key-pair) |
| Identity | Individual user ID | Tenant service account |
| Access Pattern | Interactive queries | Bulk export / ETL |
| Scope | User's permitted resources | All tenant data |
| Context Setting | Web app sets `APP_USER_ID` | Role-based or `APP_TENANT_ID` |

### Options Analysis

#### Option 1: Snowflake Reader Accounts with OpenFGA Provisioning

**Description**: Create a separate Snowflake Reader Account for each tenant. Share data to that account via Secure Data Sharing. Use OpenFGA to authorize and track which tenants have reader accounts.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Your Main Account                              │
│                                                                          │
│  ┌─────────────┐    ┌──────────────────────────────────────────────┐   │
│  │  OpenFGA    │───▶│  Provisioning Service                        │   │
│  │             │    │  - Creates secure views per tenant           │   │
│  │ client:uc   │    │  - Creates shares                            │   │
│  │ has access  │    │  - Creates reader accounts                   │   │
│  │ to tenant:uc│    │  - Grants shares to reader accounts          │   │
│  └─────────────┘    └──────────────────────────────────────────────┘   │
│                                    │                                     │
│                     ┌──────────────┴──────────────┐                     │
│                     ▼                              ▼                     │
│         ┌─────────────────────┐      ┌─────────────────────┐           │
│         │ SHARE: UC_DATA      │      │ SHARE: CSU_DATA     │           │
│         │                     │      │                     │           │
│         │ UC_BI_FINDINGS      │      │ CSU_BI_FINDINGS     │           │
│         │ UC_BI_INSPECTIONS   │      │ CSU_BI_INSPECTIONS  │           │
│         │ (filtered views)    │      │ (filtered views)    │           │
│         └──────────┬──────────┘      └──────────┬──────────┘           │
└──────────────────────────────────────────────────────────────────────────┘
                     │                              │
                     ▼                              ▼
         ┌─────────────────────┐      ┌─────────────────────┐
         │ UC_READER Account   │      │ CSU_READER Account  │
         │ (client manages     │      │ (client manages     │
         │  their own users)   │      │  their own users)   │
         └─────────────────────┘      └─────────────────────┘
```

**Pros:**
- Complete isolation between tenants
- Clients manage their own users within their reader account
- No risk of cross-tenant data access
- Native Snowflake feature, well-supported
- Can meter and bill per reader account
- OpenFGA provides audit trail of provisioned access

**Cons:**
- More complex setup per tenant
- Requires creating shares with filtered views per tenant
- Additional Snowflake costs (reader accounts consume credits)
- Must maintain share definitions when schema changes
- Higher operational overhead

**OpenFGA Model for Reader Account Provisioning:**

```yaml
type client
  relations
    # Which tenant this client belongs to
    define tenant: [tenant]
    # Track if client has been provisioned a reader account
    define has_reader_account: [user]  # Admin who provisioned it

type tenant
  relations
    # ... existing relations ...

    # Authorization to provision reader account
    define can_provision_reader_account: [client] or admin
```

**Provisioning Workflow:**

```bash
# 1. Authorize UC to have a reader account
fga tuple write client:uc can_provision_reader_account tenant:uc

# 2. Provisioning service checks authorization and creates infrastructure
# 3. Record that reader account was provisioned
fga tuple write user:admin has_reader_account client:uc
```

**Implementation - Filtered Views:**

```sql
-- ============================================================================
-- SHARES Schema: Contains tenant-filtered secure views for data sharing
-- ============================================================================

CREATE SCHEMA IF NOT EXISTS SHARES;

-- Create filtered views for tenant UC
-- These views pre-filter data - no RAP needed in reader account
CREATE OR REPLACE SECURE VIEW SHARES.UC_BI_FINDINGS AS
SELECT * FROM INSPECT.BI_FINDINGS WHERE TENANT = 'uc';

CREATE OR REPLACE SECURE VIEW SHARES.UC_BI_RESPONSE_DETAIL AS
SELECT * FROM INSPECT.BI_RESPONSE_DETAIL WHERE TENANT = 'uc';

CREATE OR REPLACE SECURE VIEW SHARES.UC_BI_INSPECTION_STATUS AS
SELECT * FROM INSPECT.BI_INSPECTION_STATUS WHERE TENANT = 'uc';

CREATE OR REPLACE SECURE VIEW SHARES.UC_BI_CHECKLIST_ITEMS_SUMMARY AS
SELECT * FROM INSPECT.BI_CHECKLIST_ITEMS_SUMMARY WHERE TENANT = 'uc';

-- Repeat for INVENTORY, FORMS schemas...
CREATE OR REPLACE SECURE VIEW SHARES.UC_INVENTORY_ITEMS AS
SELECT * FROM INVENTORY.BI_ITEMS WHERE TENANT = 'uc';
```

**Implementation - Share and Reader Account:**

```sql
-- ============================================================================
-- Create share and reader account for tenant UC
-- ============================================================================

-- 1. Create share
CREATE SHARE UC_DATA_SHARE;
COMMENT ON SHARE UC_DATA_SHARE IS 'Data share for UC tenant - all schemas';

-- 2. Grant database and schema access to share
GRANT USAGE ON DATABASE RSS_DEV TO SHARE UC_DATA_SHARE;
GRANT USAGE ON SCHEMA SHARES TO SHARE UC_DATA_SHARE;

-- 3. Add filtered views to share
GRANT SELECT ON VIEW SHARES.UC_BI_FINDINGS TO SHARE UC_DATA_SHARE;
GRANT SELECT ON VIEW SHARES.UC_BI_RESPONSE_DETAIL TO SHARE UC_DATA_SHARE;
GRANT SELECT ON VIEW SHARES.UC_BI_INSPECTION_STATUS TO SHARE UC_DATA_SHARE;
GRANT SELECT ON VIEW SHARES.UC_BI_CHECKLIST_ITEMS_SUMMARY TO SHARE UC_DATA_SHARE;
GRANT SELECT ON VIEW SHARES.UC_INVENTORY_ITEMS TO SHARE UC_DATA_SHARE;

-- 4. Create reader account
CREATE MANAGED ACCOUNT UC_READER
    ADMIN_NAME = 'uc_admin'
    ADMIN_PASSWORD = '...'  -- Securely provide to client
    TYPE = READER;

-- 5. Grant share to reader account
ALTER SHARE UC_DATA_SHARE ADD ACCOUNTS = UC_READER;

-- 6. Record in tracking table
INSERT INTO AUTH.READER_ACCOUNT_REGISTRY
    (TENANT_ID, READER_ACCOUNT_NAME, SHARE_NAME, PROVISIONED_AT, PROVISIONED_BY)
VALUES
    ('uc', 'UC_READER', 'UC_DATA_SHARE', CURRENT_TIMESTAMP(), CURRENT_USER());
```

**Reader Account Registry Table:**

```sql
-- Track provisioned reader accounts
CREATE TABLE IF NOT EXISTS AUTH.READER_ACCOUNT_REGISTRY (
    TENANT_ID VARCHAR(50) PRIMARY KEY,
    READER_ACCOUNT_NAME VARCHAR(255) NOT NULL,
    SHARE_NAME VARCHAR(255) NOT NULL,
    PROVISIONED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    PROVISIONED_BY VARCHAR(255),
    STATUS VARCHAR(20) DEFAULT 'ACTIVE',  -- ACTIVE, SUSPENDED, TERMINATED
    NOTES VARCHAR(1000)
);
```

**Client Experience (in their Reader Account):**

```sql
-- In UC_READER account (client runs these commands):

-- 1. Create database from share
CREATE DATABASE UC_DATA FROM SHARE <your_account_locator>.UC_DATA_SHARE;

-- 2. Query data - already filtered to UC only!
SELECT * FROM UC_DATA.SHARES.UC_BI_FINDINGS;
SELECT * FROM UC_DATA.SHARES.UC_INVENTORY_ITEMS;

-- 3. Client can create their own users within their reader account
CREATE USER uc_analyst PASSWORD = '...';
GRANT SELECT ON DATABASE UC_DATA TO ROLE PUBLIC;
```

**Automated Provisioning Service:**

```typescript
// services/reader-account-provisioner.ts

interface ReaderAccountConfig {
  tenantId: string;
  adminEmail: string;
  schemas: string[];  // ['INSPECT', 'INVENTORY', 'FORMS']
}

export class ReaderAccountProvisioner {
  constructor(
    private openfga: OpenFgaClient,
    private snowflake: SnowflakeConnection,
  ) {}

  async provision(config: ReaderAccountConfig): Promise<void> {
    const { tenantId, adminEmail, schemas } = config;

    // 1. Verify authorization in OpenFGA
    const canProvision = await this.openfga.check({
      user: `client:${tenantId}`,
      relation: 'can_provision_reader_account',
      object: `tenant:${tenantId}`,
    });

    if (!canProvision.allowed) {
      throw new Error(`Tenant ${tenantId} not authorized for reader account`);
    }

    // 2. Create filtered views for each schema
    await this.createFilteredViews(tenantId, schemas);

    // 3. Create share
    const shareName = `${tenantId.toUpperCase()}_DATA_SHARE`;
    await this.createShare(tenantId, shareName, schemas);

    // 4. Create reader account
    const readerAccountName = `${tenantId.toUpperCase()}_READER`;
    const tempPassword = this.generateSecurePassword();
    await this.createReaderAccount(readerAccountName, tempPassword);

    // 5. Grant share to reader account
    await this.snowflake.execute(`
      ALTER SHARE ${shareName} ADD ACCOUNTS = ${readerAccountName}
    `);

    // 6. Record in OpenFGA
    await this.openfga.write({
      writes: [{
        user: `user:${await this.getCurrentUser()}`,
        relation: 'has_reader_account',
        object: `client:${tenantId}`,
      }],
    });

    // 7. Send credentials to client
    await this.sendCredentials(adminEmail, {
      accountLocator: readerAccountName,
      username: `${tenantId}_admin`,
      tempPassword,
      shareName,
    });
  }

  private async createFilteredViews(
    tenantId: string,
    schemas: string[]
  ): Promise<void> {
    const viewMappings = {
      'INSPECT': [
        'BI_FINDINGS',
        'BI_RESPONSE_DETAIL',
        'BI_INSPECTION_STATUS',
        'BI_CHECKLIST_ITEMS_SUMMARY',
        'BI_TOP_RANK_BY_ISSUES',
        'BI_ADMIN_CHECKLIST',
        'BI_ADMIN_LOCATION_LISTS',
      ],
      'INVENTORY': ['BI_ITEMS', 'BI_LOCATIONS'],
      'FORMS': ['BI_SUBMISSIONS', 'BI_RESPONSES'],
    };

    for (const schema of schemas) {
      const views = viewMappings[schema] || [];
      for (const view of views) {
        await this.snowflake.execute(`
          CREATE OR REPLACE SECURE VIEW SHARES.${tenantId.toUpperCase()}_${view} AS
          SELECT * FROM ${schema}.${view} WHERE TENANT = '${tenantId}'
        `);
      }
    }
  }

  private async createShare(
    tenantId: string,
    shareName: string,
    schemas: string[]
  ): Promise<void> {
    await this.snowflake.execute(`CREATE SHARE IF NOT EXISTS ${shareName}`);
    await this.snowflake.execute(
      `GRANT USAGE ON DATABASE RSS_DEV TO SHARE ${shareName}`
    );
    await this.snowflake.execute(
      `GRANT USAGE ON SCHEMA SHARES TO SHARE ${shareName}`
    );

    // Grant all filtered views for this tenant
    const views = await this.snowflake.execute(`
      SELECT TABLE_NAME
      FROM INFORMATION_SCHEMA.VIEWS
      WHERE TABLE_SCHEMA = 'SHARES'
        AND TABLE_NAME LIKE '${tenantId.toUpperCase()}_%'
    `);

    for (const view of views) {
      await this.snowflake.execute(`
        GRANT SELECT ON VIEW SHARES.${view.TABLE_NAME} TO SHARE ${shareName}
      `);
    }
  }
}
```

**View Maintenance - Schema Change Handler:**

When BI views change, filtered views need updating:

```sql
-- ============================================================================
-- Stored procedure to regenerate all tenant filtered views
-- Run after schema changes to BI views
-- ============================================================================

CREATE OR REPLACE PROCEDURE AUTH.REFRESH_TENANT_VIEWS()
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
DECLARE
    v_tenant VARCHAR;
    v_cursor CURSOR FOR SELECT TENANT_ID FROM AUTH.READER_ACCOUNT_REGISTRY WHERE STATUS = 'ACTIVE';
BEGIN
    OPEN v_cursor;
    FOR record IN v_cursor DO
        v_tenant := record.TENANT_ID;

        -- Recreate INSPECT views
        EXECUTE IMMEDIATE '
            CREATE OR REPLACE SECURE VIEW SHARES.' || UPPER(v_tenant) || '_BI_FINDINGS AS
            SELECT * FROM INSPECT.BI_FINDINGS WHERE TENANT = ''' || v_tenant || '''
        ';
        -- ... repeat for other views

    END FOR;
    CLOSE v_cursor;

    RETURN 'Refreshed views for all active tenants';
END;
$$;
```

#### Option 2: Tenant-Scoped Database Roles

**Description**: Create database roles per tenant that are checked in the Row Access Policy. Service accounts are assigned the appropriate tenant role.

```
┌─────────────────┐
│  RSS_DEV        │
│                 │
│  ┌───────────┐  │     ┌─────────────────┐
│  │ EXT_UC_   │◀─┼─────│  UC Service     │
│  │ DATA_ROLE │  │     │  Account        │
│  └───────────┘  │     └─────────────────┘
│       │         │
│       ▼         │
│  ┌───────────┐  │
│  │ Row Access│  │
│  │ Policy    │  │
│  │ checks    │  │
│  │ role      │  │
│  └───────────┘  │
└─────────────────┘
```

**Pros:**
- Simpler setup than reader accounts
- Leverages existing AUTH infrastructure
- No additional Snowflake account costs
- Single policy handles both user and service account access
- Already have a pattern in `scripts/external-clients/`

**Cons:**
- Less isolation than reader accounts
- Must carefully manage role assignments
- All tenants share the same Snowflake account
- Credential management for service accounts

**Implementation Sketch:**
```sql
-- Create tenant-specific role
CREATE DATABASE ROLE IF NOT EXISTS EXT_UC_DATA_ROLE;

-- Grant read access to BI views
GRANT SELECT ON ALL VIEWS IN SCHEMA INSPECT TO DATABASE ROLE EXT_UC_DATA_ROLE;
GRANT SELECT ON ALL VIEWS IN SCHEMA INVENTORY TO DATABASE ROLE EXT_UC_DATA_ROLE;
GRANT SELECT ON ALL VIEWS IN SCHEMA FORMS TO DATABASE ROLE EXT_UC_DATA_ROLE;

-- Create service account for tenant
CREATE USER IF NOT EXISTS SVC_UC_DATA_EXPORT
    TYPE = SERVICE
    DEFAULT_ROLE = APP_DASHBOARD_ROLE
    RSA_PUBLIC_KEY = '...';

-- Grant the tenant role to service account
GRANT DATABASE ROLE EXT_UC_DATA_ROLE TO USER SVC_UC_DATA_EXPORT;
```

#### Option 3: Session Variable with Service Account

**Description**: Similar to user access, but service accounts set `APP_TENANT_ID` instead of `APP_USER_ID`. The Row Access Policy checks for tenant-level access.

**Pros:**
- Consistent with existing architecture
- No additional roles to manage per tenant
- Simple policy logic

**Cons:**
- Relies on service account correctly setting session variable
- Less secure than role-based approach (variable can be changed)
- Harder to audit which tenant accessed what

**Not Recommended** due to security concerns with session variable manipulation.

#### Option 4: Hybrid Approach (Recommended)

**Description**: Extend the existing AUTH Row Access Policy to check for **both** user permissions (via `AUTH.USER_PERMISSIONS`) **and** tenant database roles.

```sql
CREATE OR REPLACE ROW ACCESS POLICY AUTH.DATA_ACCESS
AS (program_id VARCHAR, domain_id VARCHAR, campus VARCHAR, tenant VARCHAR)
RETURNS BOOLEAN ->
    -- User-based access (web app users via OpenFGA)
    EXISTS (
        SELECT 1
        FROM TABLE(AUTH.GET_USER_PERMISSIONS(GETVARIABLE('APP_USER_ID'))) p
        WHERE
            (p.PERMISSION_SCOPE = 'tenant' AND p.RESOURCE_ID = tenant)
            OR (p.PERMISSION_SCOPE = 'campus' AND p.RESOURCE_ID = campus)
            OR (p.PERMISSION_SCOPE = 'domain' AND p.RESOURCE_ID = domain_id)
            OR (p.PERMISSION_SCOPE = 'program' AND p.RESOURCE_ID = program_id)
    )
    -- Tenant service account access (direct Snowflake, role-based)
    OR IS_DATABASE_ROLE_IN_SESSION('EXT_' || UPPER(tenant) || '_DATA_ROLE')
    -- Admin bypass
    OR GETVARIABLE('APP_USER_ID') IS NULL
    OR IS_ROLE_IN_SESSION('ACCOUNTADMIN');
```

**Pros:**
- Single policy handles both access patterns
- Role-based security for service accounts (no session variable trust)
- Leverages existing AUTH infrastructure
- Easy to add new tenants (create role, create user, grant role)
- Clear audit trail via role membership

**Cons:**
- Dynamic role name construction in policy (tenant naming must be consistent)
- Need to create role per tenant

### Recommended Approach: Hybrid with Tenant Roles

The hybrid approach (Option 4) provides the best balance of security, simplicity, and integration with the existing architecture.

#### Architecture

```
                         ┌─────────────────────────────────────────┐
                         │           AUTH.HIERARCHICAL_            │
                         │           DATA_ACCESS Policy            │
                         │                                         │
   Web App Users         │   ┌─────────────────────────────────┐   │   Service Accounts
   ─────────────────────▶│   │  Check APP_USER_ID              │   │◀─────────────────────
   (APP_USER_ID set)     │   │  against AUTH.USER_PERMISSIONS  │   │   (Tenant role)
                         │   └─────────────────────────────────┘   │
                         │                  OR                      │
                         │   ┌─────────────────────────────────┐   │
                         │   │  Check IS_DATABASE_ROLE_IN_     │   │
                         │   │  SESSION('EXT_<TENANT>_DATA_    │   │
                         │   │  ROLE')                         │   │
                         │   └─────────────────────────────────┘   │
                         │                  OR                      │
                         │   ┌─────────────────────────────────┐   │
                         │   │  Admin bypass                   │   │
                         │   └─────────────────────────────────┘   │
                         └─────────────────────────────────────────┘
```

#### Tenant Onboarding Process

```sql
-- ============================================================================
-- Onboard new tenant for direct data export
-- Example: Tenant 'uc' (University of California)
-- ============================================================================

-- 1. Create tenant-specific database role
CREATE DATABASE ROLE IF NOT EXISTS EXT_UC_DATA_ROLE;

COMMENT ON DATABASE ROLE EXT_UC_DATA_ROLE IS
    'Data export role for tenant UC. Grants read access to all UC data across schemas.';

-- 2. Grant schema usage (no table grants needed - RAP handles filtering)
GRANT USAGE ON SCHEMA AUTH TO DATABASE ROLE EXT_UC_DATA_ROLE;
GRANT USAGE ON SCHEMA INSPECT TO DATABASE ROLE EXT_UC_DATA_ROLE;
GRANT USAGE ON SCHEMA INVENTORY TO DATABASE ROLE EXT_UC_DATA_ROLE;
GRANT USAGE ON SCHEMA FORMS TO DATABASE ROLE EXT_UC_DATA_ROLE;

-- 3. Grant SELECT on views (RAP will filter to tenant's data only)
GRANT SELECT ON ALL VIEWS IN SCHEMA INSPECT TO DATABASE ROLE EXT_UC_DATA_ROLE;
GRANT SELECT ON ALL VIEWS IN SCHEMA INVENTORY TO DATABASE ROLE EXT_UC_DATA_ROLE;
GRANT SELECT ON ALL VIEWS IN SCHEMA FORMS TO DATABASE ROLE EXT_UC_DATA_ROLE;

-- Future-proof: grant on future views
GRANT SELECT ON FUTURE VIEWS IN SCHEMA INSPECT TO DATABASE ROLE EXT_UC_DATA_ROLE;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA INVENTORY TO DATABASE ROLE EXT_UC_DATA_ROLE;
GRANT SELECT ON FUTURE VIEWS IN SCHEMA FORMS TO DATABASE ROLE EXT_UC_DATA_ROLE;

-- 4. Create service account for tenant
CREATE USER IF NOT EXISTS SVC_UC_DATA_EXPORT
    TYPE = SERVICE
    DEFAULT_ROLE = APP_DASHBOARD_ROLE
    DEFAULT_SECONDARY_ROLES = ('ALL')
    RSA_PUBLIC_KEY = '<tenant-provided-public-key>';

COMMENT ON USER SVC_UC_DATA_EXPORT IS
    'Service account for UC tenant data export. Contact: uc-admin@example.edu';

-- 5. Grant roles to service account
GRANT ROLE APP_DASHBOARD_ROLE TO USER SVC_UC_DATA_EXPORT;
GRANT DATABASE ROLE EXT_UC_DATA_ROLE TO USER SVC_UC_DATA_EXPORT;

-- 6. Verify setup
SHOW GRANTS TO USER SVC_UC_DATA_EXPORT;
```

#### Updated Row Access Policy

```sql
-- ============================================================================
-- AUTH.DATA_ACCESS: Extended for Service Account Access
-- Supports both user-based (OpenFGA) and tenant service account access
-- ============================================================================

CREATE OR REPLACE ROW ACCESS POLICY AUTH.DATA_ACCESS
AS (program_id VARCHAR, domain_id VARCHAR, campus VARCHAR, tenant VARCHAR)
RETURNS BOOLEAN ->
    -- Path 1: User-based access (web app users via OpenFGA permissions)
    EXISTS (
        SELECT 1
        FROM TABLE(AUTH.GET_USER_PERMISSIONS(GETVARIABLE('APP_USER_ID'))) p
        WHERE
            (p.PERMISSION_SCOPE = 'tenant' AND p.RESOURCE_ID = tenant)
            OR (p.PERMISSION_SCOPE = 'campus' AND p.RESOURCE_ID = campus)
            OR (p.PERMISSION_SCOPE = 'domain' AND p.RESOURCE_ID = domain_id)
            OR (p.PERMISSION_SCOPE = 'program' AND p.RESOURCE_ID = program_id)
    )
    -- Path 2: Tenant service account access (direct Snowflake connection)
    -- Role naming convention: EXT_<TENANT>_DATA_ROLE (e.g., EXT_UC_DATA_ROLE)
    OR IS_DATABASE_ROLE_IN_SESSION('EXT_' || UPPER(tenant) || '_DATA_ROLE')
    -- Path 3: Internal/admin bypass
    OR GETVARIABLE('APP_USER_ID') IS NULL
    OR IS_ROLE_IN_SESSION('ACCOUNTADMIN');

COMMENT ON ROW ACCESS POLICY AUTH.DATA_ACCESS IS
    'Unified access policy supporting:
     1. Web app users (OpenFGA permissions via AUTH.USER_PERMISSIONS)
     2. Tenant service accounts (database role EXT_<TENANT>_DATA_ROLE)
     3. Admin bypass (ACCOUNTADMIN or no APP_USER_ID set)';
```

#### Simplified Policy (Without Domain)

```sql
CREATE OR REPLACE ROW ACCESS POLICY AUTH.DATA_ACCESS_SIMPLE
AS (program_id VARCHAR, campus VARCHAR, tenant VARCHAR)
RETURNS BOOLEAN ->
    -- User-based access
    EXISTS (
        SELECT 1
        FROM TABLE(AUTH.GET_USER_PERMISSIONS(GETVARIABLE('APP_USER_ID'))) p
        WHERE
            (p.PERMISSION_SCOPE = 'tenant' AND p.RESOURCE_ID = tenant)
            OR (p.PERMISSION_SCOPE = 'campus' AND p.RESOURCE_ID = campus)
            OR (p.PERMISSION_SCOPE = 'program' AND p.RESOURCE_ID = program_id)
    )
    -- Tenant service account access
    OR IS_DATABASE_ROLE_IN_SESSION('EXT_' || UPPER(tenant) || '_DATA_ROLE')
    -- Admin bypass
    OR GETVARIABLE('APP_USER_ID') IS NULL
    OR IS_ROLE_IN_SESSION('ACCOUNTADMIN');
```

#### Tenant Mapping Table (Optional)

For more flexibility in tenant role naming:

```sql
-- Optional: Map tenant IDs to role names for non-standard naming
CREATE TABLE IF NOT EXISTS AUTH.TENANT_ROLE_MAPPING (
    TENANT_ID VARCHAR(50) PRIMARY KEY,
    DATABASE_ROLE_NAME VARCHAR(255) NOT NULL,
    CREATED_AT TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
    NOTES VARCHAR(1000)
);

-- Example entries
INSERT INTO AUTH.TENANT_ROLE_MAPPING (TENANT_ID, DATABASE_ROLE_NAME, NOTES) VALUES
    ('uc', 'EXT_UC_DATA_ROLE', 'University of California system'),
    ('csu', 'EXT_CSU_DATA_ROLE', 'California State University system');
```

This allows for a memoizable function approach if the dynamic role construction becomes a performance concern:

```sql
CREATE OR REPLACE FUNCTION AUTH.GET_TENANT_ROLE(p_tenant VARCHAR)
RETURNS VARCHAR
MEMOIZABLE
AS
$$
    SELECT COALESCE(
        (SELECT DATABASE_ROLE_NAME FROM AUTH.TENANT_ROLE_MAPPING WHERE TENANT_ID = p_tenant),
        'EXT_' || UPPER(p_tenant) || '_DATA_ROLE'
    )
$$;
```

### Client Usage

Once onboarded, the tenant's service account connects directly to Snowflake:

```python
# Example: Python client for tenant data export
import snowflake.connector

conn = snowflake.connector.connect(
    account='your-account',
    user='SVC_UC_DATA_EXPORT',
    private_key_file='/path/to/rsa_key.p8',
    database='RSS_DEV',
    schema='INSPECT',
    role='APP_DASHBOARD_ROLE',  # Default role
)

# The database role EXT_UC_DATA_ROLE is automatically activated
# Row Access Policy filters to UC data only

cursor = conn.cursor()

# Export all findings for UC tenant
cursor.execute("SELECT * FROM BI_FINDINGS")
for row in cursor:
    print(row)  # Only sees UC data

# Export all inventory for UC tenant
cursor.execute("SELECT * FROM INVENTORY.BI_ITEMS")
for row in cursor:
    print(row)  # Only sees UC data
```

### Security Considerations

1. **Key-Pair Authentication**: Service accounts must use RSA key-pair authentication (no passwords)

2. **Role Naming Convention**: Enforce `EXT_<TENANT>_DATA_ROLE` naming to prevent role injection

3. **Audit Trail**: All queries are logged with the service account identity
   ```sql
   SELECT *
   FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
   WHERE USER_NAME = 'SVC_UC_DATA_EXPORT'
   ORDER BY START_TIME DESC;
   ```

4. **Network Policies**: Consider restricting service account access to known IP ranges
   ```sql
   CREATE NETWORK POLICY UC_EXPORT_POLICY
       ALLOWED_IP_LIST = ('203.0.113.0/24');  -- UC's IP range

   ALTER USER SVC_UC_DATA_EXPORT SET NETWORK_POLICY = UC_EXPORT_POLICY;
   ```

5. **Resource Monitors**: Set up resource monitors to prevent runaway queries
   ```sql
   CREATE RESOURCE MONITOR UC_EXPORT_MONITOR
       WITH CREDIT_QUOTA = 100
       TRIGGERS
           ON 75 PERCENT DO NOTIFY
           ON 100 PERCENT DO SUSPEND;

   -- Apply to warehouse used by tenant
   ALTER WAREHOUSE EXPORT_WH SET RESOURCE_MONITOR = UC_EXPORT_MONITOR;
   ```

### Option 5: Unified OpenFGA Approach (Recommended)

**Description**: Instead of using Snowflake database roles for service accounts, treat service accounts as just another identity type in OpenFGA. Grant them `can_view_data_explorer` the same way you grant users, and leverage the existing `AUTH.USER_PERMISSIONS` sync infrastructure.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         OpenFGA                                      │
│                                                                      │
│   user:alice ──can_view_data_explorer──▶ program:fire-safety        │
│   user:bob ────can_view_data_explorer──▶ campus:USC                 │
│   client:uc ───can_view_data_explorer──▶ tenant:uc    ◀── NEW!      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (sync service - same as before)
┌─────────────────────────────────────────────────────────────────────┐
│                    AUTH.USER_PERMISSIONS                             │
│                                                                      │
│   USER_ID      | SCOPE   | RESOURCE_ID                              │
│   alice        | program | fire-safety                              │
│   bob          | campus  | USC                                      │
│   client:uc    | tenant  | uc           ◀── service account!        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (Row Access Policy - unchanged!)
┌─────────────────────────────────────────────────────────────────────┐
│   Web App User:     SET APP_USER_ID = 'alice'                       │
│   Service Account:  SET APP_USER_ID = 'client:uc'                   │
│                                                                      │
│   Same policy, same table, same logic!                              │
└─────────────────────────────────────────────────────────────────────┘
```

**Why This is Better:**

| Aspect | Hybrid (Option 4) | Unified OpenFGA (Option 5) |
|--------|-------------------|---------------------------|
| Source of truth | Split: OpenFGA + Snowflake roles | Single: OpenFGA only |
| Row Access Policy | Complex: checks permissions + roles | Simple: checks permissions only |
| Grant/revoke access | Two systems to update | One system to update |
| Audit trail | Split across systems | Unified in OpenFGA |
| Granularity | Tenant-level only (role per tenant) | Any level (tenant/campus/domain/program) |
| New tenant setup | Create role + user + grants | Just write tuple |

#### OpenFGA Model Extension

Add a new `client` type to represent external service accounts:

```yaml
type client
  # Represents an external client/service account
  # No relations needed - it's just an identity type

type tenant
  relations
    # ... existing relations ...

    # Allow both users AND clients to have data explorer access
    define can_view_data_explorer: [user, client] or configuration_manager

type campus
  relations
    # ... existing relations ...
    define can_view_data_explorer: [user, client] or can_view_data_explorer from tenant

type domain
  relations
    # ... existing relations ...
    define can_view_data_explorer: [user, client] or can_view_data_explorer from campus

type program
  relations
    # ... existing relations ...
    define can_view_data_explorer: [user, client] or can_view_data_explorer from domain
```

#### Granting Access to Service Accounts

```bash
# Grant UC's service account access to all UC data (tenant-level)
fga tuple write client:uc can_view_data_explorer tenant:uc

# Or grant more granular access if needed:
# Campus-level only
fga tuple write client:uc-safety can_view_data_explorer campus:USC

# Program-level only
fga tuple write client:uc-fire-audit can_view_data_explorer program:fire-safety-2024
```

#### Sync Service Update

The sync service needs a minor update to query both `user` and `client` types:

```typescript
interface SyncConfig {
  // Identity types to sync permissions for
  identityTypes: ('user' | 'client')[];
}

async function syncAllPermissions(config: SyncConfig): Promise<void> {
  for (const identityType of config.identityTypes) {
    const identities = await getActiveIdentities(identityType);

    for (const identity of identities) {
      // Query OpenFGA for this identity's permissions
      const permissions = await getUserPermissions(
        `${identityType}:${identity.id}`
      );

      // Write to AUTH.USER_PERMISSIONS
      // USER_ID will be "client:uc" for service accounts
      await writePermissions(identity.id, permissions);
    }
  }
}

// The USER_ID in AUTH.USER_PERMISSIONS can be:
// - "alice" (user)
// - "client:uc" (service account)
// - "client:uc-safety" (scoped service account)
```

#### Service Account Setup

```sql
-- ============================================================================
-- Create service account for tenant UC (Unified OpenFGA Approach)
-- ============================================================================

-- 1. Create Snowflake user (authentication only - no special roles needed)
CREATE USER IF NOT EXISTS SVC_UC_DATA_EXPORT
    TYPE = SERVICE
    DEFAULT_ROLE = APP_DASHBOARD_ROLE
    RSA_PUBLIC_KEY = '<tenant-provided-public-key>';

COMMENT ON USER SVC_UC_DATA_EXPORT IS
    'Service account for UC tenant. APP_USER_ID: client:uc';

-- 2. Grant basic access (RAP handles data filtering)
GRANT ROLE APP_DASHBOARD_ROLE TO USER SVC_UC_DATA_EXPORT;
GRANT USAGE ON SCHEMA AUTH TO ROLE APP_DASHBOARD_ROLE;
GRANT USAGE ON SCHEMA INSPECT TO ROLE APP_DASHBOARD_ROLE;
-- ... other schemas as needed

-- 3. Grant permissions in OpenFGA (not Snowflake!)
-- Run via CLI or API:
-- fga tuple write client:uc can_view_data_explorer tenant:uc
```

#### Client Connection

The service account must set `APP_USER_ID` when connecting:

```python
import snowflake.connector

conn = snowflake.connector.connect(
    account='your-account',
    user='SVC_UC_DATA_EXPORT',
    private_key_file='/path/to/rsa_key.p8',
    database='RSS_DEV',
    schema='INSPECT',
    role='APP_DASHBOARD_ROLE',
)

cursor = conn.cursor()

# REQUIRED: Set identity for Row Access Policy
# This must match the OpenFGA client ID
cursor.execute("SET APP_USER_ID = 'client:uc'")

# Now queries are filtered to UC data only
cursor.execute("SELECT * FROM BI_FINDINGS")
for row in cursor:
    print(row)  # Only sees UC data
```

#### Enforcing APP_USER_ID for Service Accounts

To prevent service accounts from forgetting to set (or manipulating) `APP_USER_ID`, you have several options:

**Option A: Login Script (Recommended)**

Create a stored procedure that service accounts must call:

```sql
CREATE OR REPLACE PROCEDURE AUTH.INIT_CLIENT_SESSION(client_id VARCHAR)
RETURNS VARCHAR
LANGUAGE SQL
AS
$$
BEGIN
    -- Validate client_id format
    IF NOT STARTSWITH(client_id, 'client:') THEN
        RETURN 'ERROR: client_id must start with "client:"';
    END IF;

    -- Verify this client has permissions (optional validation)
    IF NOT EXISTS (
        SELECT 1 FROM AUTH.USER_PERMISSIONS
        WHERE USER_ID = client_id
    ) THEN
        RETURN 'ERROR: No permissions found for ' || client_id;
    END IF;

    -- Set the session variable
    EXECUTE IMMEDIATE 'SET APP_USER_ID = ''' || client_id || '''';

    RETURN 'Session initialized for ' || client_id;
END;
$$;

GRANT USAGE ON PROCEDURE AUTH.INIT_CLIENT_SESSION(VARCHAR) TO ROLE APP_DASHBOARD_ROLE;
```

Client usage:
```python
cursor.execute("CALL AUTH.INIT_CLIENT_SESSION('client:uc')")
```

**Option B: Snowflake Session Policy**

Use a session policy to automatically set `APP_USER_ID` based on the connecting user:

```sql
-- Create mapping table
CREATE TABLE AUTH.USER_CLIENT_MAPPING (
    SNOWFLAKE_USER VARCHAR PRIMARY KEY,
    CLIENT_ID VARCHAR NOT NULL
);

INSERT INTO AUTH.USER_CLIENT_MAPPING VALUES
    ('SVC_UC_DATA_EXPORT', 'client:uc'),
    ('SVC_CSU_DATA_EXPORT', 'client:csu');

-- Create session policy (Enterprise Edition required)
-- This automatically sets APP_USER_ID on login
```

**Option C: Application-Level Wrapper**

Provide clients with a connection wrapper that handles setup:

```python
# Provided to clients as a library
class TenantSnowflakeConnection:
    def __init__(self, client_id: str, **snowflake_config):
        self.client_id = client_id
        self.conn = snowflake.connector.connect(**snowflake_config)
        self._init_session()

    def _init_session(self):
        cursor = self.conn.cursor()
        cursor.execute(f"CALL AUTH.INIT_CLIENT_SESSION('{self.client_id}')")
        result = cursor.fetchone()[0]
        if result.startswith('ERROR'):
            raise ValueError(result)

    def cursor(self):
        return self.conn.cursor()

# Client usage
conn = TenantSnowflakeConnection(
    client_id='client:uc',
    account='your-account',
    user='SVC_UC_DATA_EXPORT',
    private_key_file='/path/to/rsa_key.p8',
    database='RSS_DEV',
)
```

#### Row Access Policy (Simplified!)

With the unified approach, the Row Access Policy is simpler - no database role check needed:

```sql
CREATE OR REPLACE ROW ACCESS POLICY AUTH.DATA_ACCESS
AS (program_id VARCHAR, domain_id VARCHAR, campus VARCHAR, tenant VARCHAR)
RETURNS BOOLEAN ->
    -- Single path for ALL access (users and clients)
    EXISTS (
        SELECT 1
        FROM TABLE(AUTH.GET_USER_PERMISSIONS(GETVARIABLE('APP_USER_ID'))) p
        WHERE
            (p.PERMISSION_SCOPE = 'tenant' AND p.RESOURCE_ID = tenant)
            OR (p.PERMISSION_SCOPE = 'campus' AND p.RESOURCE_ID = campus)
            OR (p.PERMISSION_SCOPE = 'domain' AND p.RESOURCE_ID = domain_id)
            OR (p.PERMISSION_SCOPE = 'program' AND p.RESOURCE_ID = program_id)
    )
    -- Admin bypass
    OR GETVARIABLE('APP_USER_ID') IS NULL
    OR IS_ROLE_IN_SESSION('ACCOUNTADMIN');

COMMENT ON ROW ACCESS POLICY AUTH.DATA_ACCESS IS
    'Unified access policy for users and service accounts.
     All permissions managed in OpenFGA and synced to AUTH.USER_PERMISSIONS.
     APP_USER_ID can be a user ID or client ID (e.g., "client:uc").';
```

#### Comparison: Hybrid vs Unified

| Scenario | Hybrid (Database Roles) | Unified (OpenFGA) |
|----------|------------------------|-------------------|
| Grant tenant access | Create role, grant to user | `fga tuple write client:uc can_view_data_explorer tenant:uc` |
| Grant campus-only access | Not easily possible | `fga tuple write client:uc-safety can_view_data_explorer campus:USC` |
| Revoke access | Drop role or revoke from user | `fga tuple delete ...` |
| Audit "who has access" | Query Snowflake roles | Query OpenFGA |
| Add new schema | Grant to each tenant role | Nothing - RAP handles it |

#### When to Use Which Approach

**Use Unified OpenFGA (Option 5) when:**
- You want single source of truth for all permissions
- Service accounts need granular access (not just tenant-level)
- You already have OpenFGA infrastructure
- You want consistent audit trail

**Use Hybrid Database Roles (Option 4) when:**
- Service accounts don't need to set session variables (simpler client experience)
- You want Snowflake-native access control for service accounts
- Tenant-level access is sufficient (no need for campus/program granularity)

**Use Reader Accounts (Option 1) when:**
- Complete data isolation is required
- Client needs to manage their own users
- Billing separation is required

### Recommendation

**Option 5 (Unified OpenFGA)** is recommended as the primary approach because:

1. **Single source of truth** - All permissions in OpenFGA
2. **Simpler Row Access Policy** - No database role logic
3. **Flexible granularity** - Service accounts can have any access level
4. **Consistent management** - Same workflow for users and clients
5. **Better auditability** - All permission changes tracked in OpenFGA

The only trade-off is that service accounts must set `APP_USER_ID`, but this can be enforced via login scripts or provided client libraries.

### When to Use Reader Accounts Instead

Consider Reader Accounts (Option 1) when:

- **Regulatory requirements** mandate complete data isolation
- **Client needs their own users** - multiple people at the tenant need access with different permissions
- **Billing separation** - tenant pays directly for their Snowflake consumption
- **Schema customization** - tenant wants to create their own views/transformations

---

## Implementation Roadmap

### Phase 1: Foundation (Week 1-2)

- [ ] Create OpenFGA authorization model
- [ ] Set up hierarchy tuples (tenant→campus→program)
- [ ] Create `INSPECT.USER_PERMISSIONS` table
- [ ] Create memoizable function
- [ ] Create Row Access Policy
- [ ] Apply policy to one test view (BI_FINDINGS)
- [ ] Manual testing with hardcoded permissions

### Phase 2: Sync Service (Week 3-4)

- [ ] Set up sync service project structure
- [ ] Implement full sync logic
- [ ] Implement incremental sync (ReadChanges polling)
- [ ] Add API endpoint for webhook triggers
- [ ] Add monitoring and metrics
- [ ] Deploy to staging environment

### Phase 3: Application Integration (Week 5)

- [ ] Update web app to set `APP_USER_ID` session variable
- [ ] Implement connection pool with context management
- [ ] Add "permissions syncing" indicator to UI
- [ ] Test end-to-end flow

### Phase 4: Production Rollout (Week 6)

- [ ] Apply Row Access Policy to all BI views
- [ ] Configure alerting
- [ ] Run full initial sync
- [ ] Enable webhook triggers from microservices
- [ ] Monitor and tune performance

### Phase 5: Optimization (Ongoing)

- [ ] Tune sync intervals based on usage patterns
- [ ] Optimize OpenFGA model if needed
- [ ] Consider search optimization service for large tables
- [ ] Document operational runbooks

---

## Implementation Notes

**Last Updated:** 2025-12-05

The code examples in this research document are conceptual illustrations. The actual implementation uses a **Bun workspace monorepo** structure with Fastify (not the generic TypeScript shown in examples).

### Actual Project Structure

```
dashboard-rsc/
├── apps/
│   ├── client/                    # Next.js 16 BFF
│   │   └── src/app/               # App Router pages
│   └── server/                    # Fastify backend (sync service)
│       └── src/
│           ├── app.ts             # Fastify app factory
│           ├── server.ts          # Entry point
│           ├── env.ts             # @t3-oss/env-core config
│           ├── features/
│           │   └── sync/          # Permission sync feature
│           │       ├── sync.service.ts   # PermissionSyncService
│           │       ├── sync.routes.ts    # Fastify routes
│           │       ├── sync.schema.ts    # Validation schemas
│           │       └── change-poller.ts  # ReadChanges polling
│           └── lib/
│               ├── openfga-client.ts     # OpenFGA wrapper
│               ├── snowflake-client.ts   # Snowflake wrapper
│               └── errors.ts             # Error types
└── scripts/
    └── auth/                      # Snowflake SQL scripts
```

**Note:** Scheduled full sync (hourly reconciliation) is documented in this research but deferred in implementation. Webhook + ChangePoller provide sufficient coverage initially.

### Key Mapping: Research Examples → Actual Files

| Research Example Path | Actual Implementation |
|-----------------------|----------------------|
| `src/sync/full-sync.ts` | `apps/server/src/features/sync/sync.service.ts` |
| `src/sync/incremental-sync.ts` | `apps/server/src/features/sync/change-poller.ts` |
| `src/api/webhook-handler.ts` | `apps/server/src/features/sync/sync.routes.ts` |
| OpenFgaClient | `apps/server/src/lib/openfga-client.ts` |
| Snowflake client | `apps/server/src/lib/snowflake-client.ts` |

### Technology Choices

| Research Assumption | Actual Implementation |
|---------------------|----------------------|
| Generic Node.js | **Bun** runtime |
| Express/generic HTTP | **Fastify** with @risk-and-safety/* plugins |
| Generic config | **@t3-oss/env-core** for typed env validation |
| Hono framework | Migrated to **Fastify** for plugin ecosystem |

---

## References

### Snowflake Documentation
- [Row Access Policies](https://docs.snowflake.com/en/user-guide/security-row-intro)
- [Memoizable Functions](https://docs.snowflake.com/en/developer-guide/udf/sql/udf-sql-scalar-functions)
- [Session Variables](https://docs.snowflake.com/en/sql-reference/session-variables)
- [Connection Pooling (.NET)](https://github.com/snowflakedb/snowflake-connector-net/blob/master/doc/ConnectionPooling.md)

### OpenFGA Documentation
- [OpenFGA Home](https://openfga.dev/)
- [ReadChanges API](https://openfga.dev/docs/interacting/read-tuple-changes)
- [ListObjects](https://openfga.dev/docs/getting-started/perform-list-objects)
- [Search with Permissions](https://openfga.dev/docs/interacting/search-with-permissions)
- [Production Best Practices](https://openfga.dev/docs/best-practices/running-in-production)

### Architecture Patterns
- [Event-Driven Architecture (AWS)](https://aws.amazon.com/event-driven-architecture/)
- [AuthZed Materialize](https://authzed.com/docs/authzed/concepts/authzed-materialize)
- [Permit.io Data Filtering](https://docs.permit.io/how-to/enforce-permissions/data-filtering/)

### Performance Articles
- [Faster Snowflake UDFs with Memoizable](https://hoffa.medium.com/faster-snowflake-udfs-and-policies-with-memoizable-b84544c1bf5)
- [OpenFGA Performance Optimizations](https://deepwiki.com/openfga/openfga/2.3-performance-optimizations)

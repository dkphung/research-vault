# MAQ Snowflake Migration: Snowpark Python Specification

This document provides a comprehensive specification for implementing the MAQ report system in Snowflake using Snowpark Python (Option 3).

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Architecture Overview](#architecture-overview)
3. [Component Inventory](#component-inventory)
4. [Data Pipeline Design](#data-pipeline-design)
5. [Snowflake Objects Specification](#snowflake-objects-specification)
6. [Python Module Structure](#python-module-structure)
7. [UDF Specifications](#udf-specifications)
8. [SQL Views Specification](#sql-views-specification)
9. [Stored Procedures Specification](#stored-procedures-specification)
10. [Deployment Pipeline](#deployment-pipeline)
11. [Testing Strategy](#testing-strategy)
12. [Monitoring & Observability](#monitoring--observability)
13. [Maintenance Considerations](#maintenance-considerations)
14. [Cost Analysis](#cost-analysis)
15. [Risk Assessment](#risk-assessment)

---

## Executive Summary

### What We're Building

A complete Snowflake-based MAQ reporting system that replicates the current MongoDB/Elasticsearch/Node.js implementation using:
- **Snowpark Python UDFs** for complex calculation logic
- **SQL Views** for data joins and aggregation
- **Stored Procedures** for orchestration and caching
- **Scheduled Tasks** for automated refresh

### Scope

| In Scope | Out of Scope |
|----------|--------------|
| MAQ limit calculations | UI rendering |
| Actual quantity aggregation | GraphQL API layer |
| Compliance status determination | User authentication |
| Report data generation | Real-time container updates |
| HMIS export calculations | Building/room management |

### Key Metrics

| Metric | Target |
|--------|--------|
| Data freshness | < 15 minutes (configurable) |
| Query response time | < 5 seconds for single building |
| Calculation accuracy | 100% parity with current system |
| Uptime | 99.9% |

---

## Architecture Overview

### System Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                SNOWFLAKE ACCOUNT                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                           DATABASE: MAQ_DB                                  │ │
│  ├────────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                             │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │ │
│  │  │  SCHEMA: RAW    │  │ SCHEMA: STAGING │  │  SCHEMA: MARTS  │             │ │
│  │  │                 │  │                 │  │                 │             │ │
│  │  │ • fire_codes    │  │ • stg_control_  │  │ • maq_report    │             │ │
│  │  │ • control_areas │  │   areas         │  │ • maq_building_ │             │ │
│  │  │ • containers    │  │ • stg_hazard_   │  │   summary       │             │ │
│  │  │ • buildings     │  │   classes       │  │ • maq_report_   │             │ │
│  │  │ • floors        │  │ • stg_container_│  │   cache         │             │ │
│  │  │ • rooms         │  │   actuals       │  │                 │             │ │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────┘             │ │
│  │                                                                             │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │ │
│  │  │                        SCHEMA: UDFS                                  │   │ │
│  │  │                                                                      │   │ │
│  │  │  Python UDFs:                    SQL UDFs:                           │   │ │
│  │  │  • get_maq_factors()             • format_display_value()            │   │ │
│  │  │  • calculate_limit()             • convert_gram_to_pound()           │   │ │
│  │  │  • get_compliance_status()       • convert_liter_to_gallon()         │   │ │
│  │  │  • get_floor_factors()           • convert_liter_to_cubic_feet()     │   │ │
│  │  │  • evaluate_rule()                                                   │   │ │
│  │  │  • get_open_closed_limits()                                          │   │ │
│  │  └─────────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                             │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │ │
│  │  │                     SCHEMA: PROCEDURES                               │   │ │
│  │  │                                                                      │   │ │
│  │  │  • refresh_maq_cache()           • validate_maq_calculations()       │   │ │
│  │  │  • refresh_building_report()     • sync_source_data()                │   │ │
│  │  └─────────────────────────────────────────────────────────────────────┘   │ │
│  │                                                                             │ │
│  │  ┌─────────────────────────────────────────────────────────────────────┐   │ │
│  │  │                        SCHEMA: STAGES                                │   │ │
│  │  │                                                                      │   │ │
│  │  │  • @python_udfs (internal stage for Python packages)                 │   │ │
│  │  │  • @data_import (for file-based data loads)                          │   │ │
│  │  └─────────────────────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                              WAREHOUSES                                     │ │
│  │                                                                             │ │
│  │  • MAQ_ETL_WH (X-Small) - Data loading                                     │ │
│  │  • MAQ_COMPUTE_WH (Small) - Report queries, UDF execution                  │ │
│  │  • MAQ_TASK_WH (X-Small) - Scheduled task execution                        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                                TASKS                                        │ │
│  │                                                                             │ │
│  │  • TASK_REFRESH_MAQ_CACHE (every 15 min)                                   │ │
│  │  • TASK_SYNC_CONTAINERS (every 5 min via Snowpipe)                         │ │
│  │  • TASK_DAILY_VALIDATION (daily at 2 AM)                                   │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                              EXTERNAL SYSTEMS                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                 │
│  │    MongoDB      │  │  Elasticsearch  │  │ Building Service│                 │
│  │                 │  │                 │  │                 │                 │
│  │ • fireCode      │  │ • container     │  │ • buildings     │                 │
│  │ • controlArea   │  │   index         │  │ • floors        │                 │
│  │                 │  │                 │  │ • rooms         │                 │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘                 │
│           │                    │                    │                          │
│           ▼                    ▼                    ▼                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         DATA PIPELINE                                    │   │
│  │                                                                          │   │
│  │  Option A: Kafka → Snowpipe (real-time)                                 │   │
│  │  Option B: Scheduled ETL jobs (batch)                                   │   │
│  │  Option C: Fivetran/Airbyte connectors (managed)                        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Data Flow

```
Source Systems          Snowflake Raw         Staging Views         Calculation         Output
─────────────────────────────────────────────────────────────────────────────────────────────────

MongoDB: fireCode  ──►  raw.fire_codes  ──►  stg_fire_code_    ──►                  ──►
                                             occupancies            │
MongoDB: controlArea ►  raw.control_    ──►  stg_control_      ──►  │
                        areas                area_base              │
                                                                    │
Elasticsearch:      ──►  raw.containers ──►  stg_container_    ──►  │    Python UDFs     marts.
container                                    actuals                │    ────────────    maq_report
                                                                    ├──► get_maq_       ──►
Building Service:   ──►  raw.buildings  ──►  stg_building_     ──►  │    factors()
                        raw.floors           hierarchy              │
                        raw.rooms                                   │    calculate_
                                                                    │    limit()
                                             stg_hazard_class_ ──►  │
                                             band_mapping           │    get_compliance_
                                                                         status()
```

---

## Component Inventory

### Complete List of Objects to Create

#### Database & Schemas (5 objects)

| Object | Type | Purpose |
|--------|------|---------|
| `MAQ_DB` | Database | Main database |
| `MAQ_DB.RAW` | Schema | Raw data from source systems |
| `MAQ_DB.STAGING` | Schema | Transformed/cleaned data |
| `MAQ_DB.MARTS` | Schema | Final report tables/views |
| `MAQ_DB.UDFS` | Schema | Python and SQL UDFs |
| `MAQ_DB.PROCEDURES` | Schema | Stored procedures |
| `MAQ_DB.STAGES` | Schema | Internal stages |

#### Raw Tables (6 tables)

| Table | Source | Est. Rows | Update Frequency |
|-------|--------|-----------|------------------|
| `raw.fire_codes` | MongoDB | ~10 | Weekly |
| `raw.control_areas` | MongoDB | ~500 | Daily |
| `raw.containers` | Elasticsearch | ~100,000 | Real-time |
| `raw.buildings` | Building Service | ~200 | Weekly |
| `raw.floors` | Building Service | ~2,000 | Weekly |
| `raw.rooms` | Building Service | ~20,000 | Weekly |

#### Staging Views (8 views)

| View | Purpose |
|------|---------|
| `staging.stg_fire_code_occupancies` | Flattened fire code occupancies |
| `staging.stg_hazard_classes` | Flattened hazard classes with rules |
| `staging.stg_hazard_class_band_mapping` | Band to hazard class mapping |
| `staging.stg_control_area_base` | Control areas with building join |
| `staging.stg_building_hierarchy` | Building → Floor → Room hierarchy |
| `staging.stg_container_actuals` | Aggregated container quantities |
| `staging.stg_room_control_area_map` | Room to control area mapping |
| `staging.stg_occupancy_rules` | Floor-level rules by occupancy |

#### Python UDFs (8 UDFs)

| UDF | Inputs | Output | Complexity |
|-----|--------|--------|------------|
| `get_maq_factors` | hazard_class, rules, conditions | ARRAY | High |
| `calculate_limit_from_factors` | factors ARRAY | OBJECT | Medium |
| `get_compliance_status` | limit, actual | STRING | Low |
| `get_floor_maq_factors` | occupancy_rules, floor_level | ARRAY | Medium |
| `evaluate_rule_conditions` | rule, conditions | BOOLEAN | Medium |
| `get_open_closed_limits` | hazard_class, factors | OBJECT | High |
| `is_control_area_sprinklered` | fire_supp_type, floor | BOOLEAN | Low |
| `calculate_control_area_maq` | all inputs | OBJECT | High (orchestrator) |

#### SQL UDFs (4 UDFs)

| UDF | Purpose |
|-----|---------|
| `convert_gram_to_pound` | Unit conversion |
| `convert_liter_to_gallon` | Unit conversion |
| `convert_liter_to_cubic_feet` | Unit conversion |
| `format_display_value` | Significant figure formatting |

#### Mart Views/Tables (4 objects)

| Object | Type | Purpose |
|--------|------|---------|
| `marts.maq_report` | View | Real-time MAQ report |
| `marts.maq_report_cache` | Table | Cached MAQ report |
| `marts.maq_building_summary` | View | Building-level rollup |
| `marts.maq_compliance_dashboard` | View | Compliance statistics |

#### Stored Procedures (5 procedures)

| Procedure | Purpose | Frequency |
|-----------|---------|-----------|
| `refresh_maq_cache` | Full cache refresh | Every 15 min |
| `refresh_building_report` | Single building refresh | On-demand |
| `validate_calculations` | Compare with source | Daily |
| `sync_source_data` | Manual data sync trigger | On-demand |
| `cleanup_old_cache` | Remove stale cache entries | Daily |

#### Tasks (3 tasks)

| Task | Schedule | Warehouse |
|------|----------|-----------|
| `TASK_REFRESH_MAQ_CACHE` | CRON 0/15 * * * * | MAQ_TASK_WH |
| `TASK_DAILY_VALIDATION` | CRON 0 2 * * * | MAQ_TASK_WH |
| `TASK_CLEANUP` | CRON 0 3 * * * | MAQ_TASK_WH |

#### Warehouses (3 warehouses)

| Warehouse | Size | Purpose | Auto-Suspend |
|-----------|------|---------|--------------|
| `MAQ_ETL_WH` | X-Small | Data loading | 60 sec |
| `MAQ_COMPUTE_WH` | Small | Query execution | 120 sec |
| `MAQ_TASK_WH` | X-Small | Scheduled tasks | 60 sec |

#### Stages (2 stages)

| Stage | Type | Purpose |
|-------|------|---------|
| `@stages.python_udfs` | Internal | Python package deployment |
| `@stages.data_import` | Internal/External | File-based data loads |

#### Snowpipes (1-3 pipes)

| Pipe | Source | Target |
|------|--------|--------|
| `PIPE_CONTAINERS` | S3/Kafka | raw.containers |
| `PIPE_CONTROL_AREAS` | S3 | raw.control_areas (optional) |
| `PIPE_FIRE_CODES` | S3 | raw.fire_codes (optional) |

---

## Data Pipeline Design

### Option A: Kafka → Snowpipe (Recommended for Containers)

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Elasticsearch│ ──► │    Kafka     │ ──► │  Kafka       │ ──► │  Snowflake   │
│   Changes    │     │   Connect    │     │  Topic       │     │  Snowpipe    │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                                      │
                                                                      ▼
                                                              ┌──────────────┐
                                                              │ raw.containers│
                                                              └──────────────┘
```

**Setup Requirements:**
- Kafka Connect with Elasticsearch source connector
- Snowflake Kafka Connector
- S3 bucket for staging (or direct Kafka ingestion)

### Option B: Scheduled ETL (For Low-Change Data)

```python
# Example: Python ETL script for fire codes
from pymongo import MongoClient
from snowflake.connector import connect

def sync_fire_codes():
    # Extract from MongoDB
    mongo = MongoClient(MONGO_URI)
    fire_codes = list(mongo.maq.fireCode.find())

    # Transform
    rows = [transform_fire_code(fc) for fc in fire_codes]

    # Load to Snowflake
    sf = connect(**SNOWFLAKE_CREDS)
    cursor = sf.cursor()
    cursor.execute("TRUNCATE TABLE raw.fire_codes")
    cursor.executemany(
        "INSERT INTO raw.fire_codes (id, name, data) VALUES (%s, %s, %s)",
        rows
    )
```

**Schedule:**
| Source | Frequency | Method |
|--------|-----------|--------|
| fire_codes | Daily | ETL script |
| control_areas | Hourly | ETL script |
| containers | Real-time | Snowpipe |
| buildings/floors/rooms | Daily | ETL script |

### Option C: Managed Connectors (Fivetran/Airbyte)

| Connector | Source | Destination |
|-----------|--------|-------------|
| Fivetran MongoDB | MongoDB | raw.fire_codes, raw.control_areas |
| Fivetran Elasticsearch | Elasticsearch | raw.containers |
| Custom API connector | Building Service | raw.buildings, etc. |

**Pros:** Managed, reliable, built-in schema detection
**Cons:** Additional cost ($1-2/MAR per connector)

---

## Snowflake Objects Specification

### Raw Tables DDL

```sql
-- ============================================================================
-- RAW SCHEMA TABLES
-- ============================================================================

-- Fire Codes (from MongoDB)
CREATE OR REPLACE TABLE raw.fire_codes (
    id VARCHAR PRIMARY KEY,
    name VARCHAR,
    hmis_report BOOLEAN,
    occupancies VARIANT,           -- Array of occupancy objects
    hazard_class_mappings VARIANT, -- Array of band mapping objects
    created_date TIMESTAMP_NTZ,
    last_updated_date TIMESTAMP_NTZ,
    _loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Control Areas (from MongoDB)
CREATE OR REPLACE TABLE raw.control_areas (
    id VARCHAR PRIMARY KEY,
    name VARCHAR,
    building_id VARCHAR,
    campus_code VARCHAR,
    fire_code_id VARCHAR,
    occupancy VARCHAR,
    floor_above_ground_plane INT,
    is_outdoor BOOLEAN DEFAULT FALSE,
    approved_storage VARIANT,      -- Array of approved storage objects
    exemption VARIANT,             -- { isExempt, reason, notes }
    fire_suppression_override VARIANT, -- { override, value }
    notes VARCHAR,
    status VARCHAR,
    created_date TIMESTAMP_NTZ,
    last_updated_date TIMESTAMP_NTZ,
    _loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Containers (from Elasticsearch)
CREATE OR REPLACE TABLE raw.containers (
    id VARCHAR PRIMARY KEY,
    room_id VARCHAR,
    normalized_size_gram FLOAT,
    normalized_size_liter FLOAT,
    family_form_at_ntp VARCHAR,    -- 'solid', 'liquid', 'gas'
    family_bands VARIANT,          -- Array of band IDs
    exemption VARCHAR,
    active BOOLEAN,
    _loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Buildings (from Building Service)
CREATE OR REPLACE TABLE raw.buildings (
    id VARCHAR PRIMARY KEY,
    building_key VARCHAR,
    name VARCHAR,
    address VARCHAR,
    campus_code VARCHAR,
    fire_code_id VARCHAR,
    fire_suppression_type VARCHAR, -- 'FULL', 'BASEMENT_ONLY', 'NONE'
    _loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Floors
CREATE OR REPLACE TABLE raw.floors (
    id VARCHAR PRIMARY KEY,
    building_id VARCHAR,
    floor_key VARCHAR,
    name VARCHAR,
    ground_plane INT,              -- -2, -1, 0, 1, 2, etc.
    _loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- Rooms
CREATE OR REPLACE TABLE raw.rooms (
    id VARCHAR PRIMARY KEY,
    floor_id VARCHAR,
    room_key VARCHAR,
    name VARCHAR,
    room_number VARCHAR,
    control_area_id VARCHAR,
    _loaded_at TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
);

-- ============================================================================
-- INDEXES (Clustering Keys for Performance)
-- ============================================================================

ALTER TABLE raw.containers CLUSTER BY (room_id, active);
ALTER TABLE raw.rooms CLUSTER BY (control_area_id);
ALTER TABLE raw.control_areas CLUSTER BY (building_id);
```

### Staging Views DDL

```sql
-- ============================================================================
-- STAGING SCHEMA VIEWS
-- ============================================================================

-- Flattened Fire Code Occupancies
CREATE OR REPLACE VIEW staging.stg_fire_code_occupancies AS
SELECT
    fc.id AS fire_code_id,
    fc.name AS fire_code_name,
    fc.hmis_report,
    occ.value:id::VARCHAR AS occupancy_id,
    occ.value:name::VARCHAR AS occupancy_name,
    occ.value:rules::VARIANT AS occupancy_rules,
    occ.value:hazardClasses::VARIANT AS hazard_classes
FROM raw.fire_codes fc,
LATERAL FLATTEN(input => fc.occupancies) occ;

-- Flattened Hazard Classes with Rules
CREATE OR REPLACE VIEW staging.stg_hazard_classes AS
SELECT
    fco.fire_code_id,
    fco.fire_code_name,
    fco.hmis_report,
    fco.occupancy_id,
    fco.occupancy_name,
    fco.occupancy_rules,
    hc.value:id::VARCHAR AS hazard_class_id,
    hc.value:name::VARCHAR AS hazard_class_name,
    hc.value:isHealthHazard::BOOLEAN AS is_health_hazard,
    -- Solid state
    hc.value:solid:baseline::FLOAT AS baseline_solid,
    hc.value:solid:units::VARCHAR AS units_solid,
    hc.value:solid:isNoLimit::BOOLEAN AS is_no_limit_solid,
    hc.value:solid:isNotApplicable::BOOLEAN AS is_not_applicable_solid,
    -- Liquid state
    hc.value:liquid:baseline::FLOAT AS baseline_liquid,
    hc.value:liquid:units::VARCHAR AS units_liquid,
    hc.value:liquid:isNoLimit::BOOLEAN AS is_no_limit_liquid,
    hc.value:liquid:isNotApplicable::BOOLEAN AS is_not_applicable_liquid,
    hc.value:liquid:reportAsLiquid::BOOLEAN AS report_as_liquid,
    -- Gas state
    hc.value:gas:baseline::FLOAT AS baseline_gas,
    hc.value:gas:units::VARCHAR AS units_gas,
    hc.value:gas:isNoLimit::BOOLEAN AS is_no_limit_gas,
    hc.value:gas:isNotApplicable::BOOLEAN AS is_not_applicable_gas,
    hc.value:gas:reportAsLiquid::BOOLEAN AS gas_report_as_liquid,
    -- Rules
    hc.value:rules::VARIANT AS hazard_class_rules,
    -- Full object for UDF
    hc.value AS hazard_class_data
FROM staging.stg_fire_code_occupancies fco,
LATERAL FLATTEN(input => fco.hazard_classes) hc;

-- Hazard Class Band Mapping
CREATE OR REPLACE VIEW staging.stg_hazard_class_band_mapping AS
SELECT
    fc.id AS fire_code_id,
    mapping.value:name::VARCHAR AS hazard_class_name,
    mapping.value:must::VARIANT AS must_bands,
    mapping.value:should::VARIANT AS should_bands,
    mapping.value:mustNot::VARIANT AS must_not_bands,
    -- Extract band IDs for joining
    ARRAY_AGG(DISTINCT must_band.value:id::VARCHAR) AS must_band_ids,
    ARRAY_AGG(DISTINCT should_band.value:id::VARCHAR) AS should_band_ids,
    ARRAY_AGG(DISTINCT must_not_band.value:id::VARCHAR) AS must_not_band_ids
FROM raw.fire_codes fc,
LATERAL FLATTEN(input => fc.hazard_class_mappings) mapping,
LATERAL FLATTEN(input => mapping.value:must, OUTER => TRUE) must_band,
LATERAL FLATTEN(input => mapping.value:should, OUTER => TRUE) should_band,
LATERAL FLATTEN(input => mapping.value:mustNot, OUTER => TRUE) must_not_band
GROUP BY fc.id, mapping.value:name, mapping.value:must, mapping.value:should, mapping.value:mustNot;

-- Control Area Base (with building join and sprinkler calculation)
CREATE OR REPLACE VIEW staging.stg_control_area_base AS
SELECT
    ca.id AS control_area_id,
    ca.name AS control_area_name,
    ca.building_id,
    ca.fire_code_id,
    ca.occupancy,
    COALESCE(ca.floor_above_ground_plane, 1) AS floor_above_ground_plane,
    COALESCE(ca.is_outdoor, FALSE) AS is_outdoor,
    ca.approved_storage,
    COALESCE(ca.exemption:isExempt::BOOLEAN, FALSE) AS is_exempt,
    ca.exemption:reason::VARCHAR AS exemption_reason,
    ca.exemption:notes::VARCHAR AS exemption_notes,
    ca.fire_suppression_override:override::BOOLEAN AS fire_suppression_override_enabled,
    ca.fire_suppression_override:value::BOOLEAN AS fire_suppression_override_value,
    ca.notes,
    b.name AS building_name,
    b.fire_suppression_type AS building_fire_suppression_type,
    -- Calculate effective sprinkler coverage
    CASE
        WHEN ca.fire_suppression_override:override::BOOLEAN = TRUE
        THEN ca.fire_suppression_override:value::BOOLEAN
        WHEN b.fire_suppression_type = 'FULL' THEN TRUE
        WHEN b.fire_suppression_type = 'BASEMENT_ONLY'
             AND COALESCE(ca.floor_above_ground_plane, 1) < 0 THEN TRUE
        ELSE FALSE
    END AS is_sprinklered,
    -- Effective fire suppression type for rule evaluation
    CASE
        WHEN ca.fire_suppression_override:override::BOOLEAN = TRUE
        THEN CASE WHEN ca.fire_suppression_override:value::BOOLEAN THEN 'FULL' ELSE 'NONE' END
        ELSE b.fire_suppression_type
    END AS effective_fire_suppression_type
FROM raw.control_areas ca
LEFT JOIN raw.buildings b ON ca.building_id = b.id;

-- Building Hierarchy
CREATE OR REPLACE VIEW staging.stg_building_hierarchy AS
SELECT
    b.id AS building_id,
    b.name AS building_name,
    b.fire_suppression_type,
    f.id AS floor_id,
    f.name AS floor_name,
    f.ground_plane,
    r.id AS room_id,
    r.name AS room_name,
    r.room_number,
    r.control_area_id
FROM raw.buildings b
JOIN raw.floors f ON f.building_id = b.id
JOIN raw.rooms r ON r.floor_id = f.id;

-- Room to Control Area Mapping
CREATE OR REPLACE VIEW staging.stg_room_control_area_map AS
SELECT
    r.id AS room_id,
    r.control_area_id,
    f.ground_plane AS floor_ground_plane
FROM raw.rooms r
JOIN raw.floors f ON r.floor_id = f.id
WHERE r.control_area_id IS NOT NULL;

-- Container Actuals (Aggregated)
CREATE OR REPLACE VIEW staging.stg_container_actuals AS
SELECT
    rcam.control_area_id,
    c.family_form_at_ntp AS physical_state,
    -- We need to join to band mapping to get hazard class
    hcbm.hazard_class_name,
    SUM(c.normalized_size_gram) AS total_grams,
    SUM(c.normalized_size_liter) AS total_liters,
    COUNT(*) AS container_count
FROM raw.containers c
JOIN staging.stg_room_control_area_map rcam ON c.room_id = rcam.room_id
JOIN staging.stg_hazard_class_band_mapping hcbm ON (
    -- Must match ALL 'must' bands
    (
        ARRAY_SIZE(hcbm.must_band_ids) = 0
        OR ARRAY_SIZE(ARRAY_INTERSECTION(c.family_bands, hcbm.must_band_ids)) = ARRAY_SIZE(hcbm.must_band_ids)
    )
    -- Must match AT LEAST ONE 'should' band (if any exist)
    AND (
        ARRAY_SIZE(hcbm.should_band_ids) = 0
        OR ARRAY_SIZE(ARRAY_INTERSECTION(c.family_bands, hcbm.should_band_ids)) > 0
    )
    -- Must NOT match ANY 'must_not' bands
    AND ARRAY_SIZE(ARRAY_INTERSECTION(c.family_bands, hcbm.must_not_band_ids)) = 0
)
WHERE c.active = TRUE
  AND (c.exemption IS NULL OR c.exemption = '')
GROUP BY rcam.control_area_id, c.family_form_at_ntp, hcbm.hazard_class_name;

-- Occupancy Rules (Floor-level factors)
CREATE OR REPLACE VIEW staging.stg_occupancy_rules AS
SELECT
    fire_code_id,
    occupancy_name,
    rule.value:operator::VARCHAR AS operator,
    rule.value:appliedToFloors::VARIANT AS applied_to_floors,
    rule.value:percentage::FLOAT AS percentage
FROM staging.stg_fire_code_occupancies,
LATERAL FLATTEN(input => occupancy_rules) rule;
```

---

## Python Module Structure

### Directory Layout

```
maq_snowpark/
├── pyproject.toml                 # Project configuration
├── requirements.txt               # Dependencies
├── setup.py                       # Package setup
│
├── src/
│   └── maq_snowpark/
│       ├── __init__.py
│       │
│       ├── constants.py           # Conversion constants, enums
│       ├── types.py               # Type definitions
│       │
│       ├── udfs/
│       │   ├── __init__.py
│       │   ├── maq_factors.py     # get_maq_factors UDF
│       │   ├── limit_calc.py      # calculate_limit_from_factors UDF
│       │   ├── compliance.py      # get_compliance_status UDF
│       │   ├── floor_factors.py   # get_floor_maq_factors UDF
│       │   ├── rule_eval.py       # evaluate_rule_conditions UDF
│       │   ├── open_closed.py     # get_open_closed_limits UDF
│       │   └── sprinkler.py       # is_control_area_sprinklered UDF
│       │
│       ├── procedures/
│       │   ├── __init__.py
│       │   ├── refresh_cache.py   # refresh_maq_cache procedure
│       │   ├── refresh_building.py # refresh_building_report procedure
│       │   └── validate.py        # validate_calculations procedure
│       │
│       └── deploy/
│           ├── __init__.py
│           ├── register_udfs.py   # UDF registration script
│           └── setup_objects.py   # Schema/table creation
│
├── tests/
│   ├── __init__.py
│   ├── conftest.py                # Pytest fixtures
│   ├── test_maq_factors.py
│   ├── test_limit_calc.py
│   ├── test_compliance.py
│   ├── test_floor_factors.py
│   ├── test_integration.py
│   └── fixtures/
│       ├── fire_codes.json
│       ├── control_areas.json
│       └── expected_results.json
│
└── scripts/
    ├── deploy.py                  # Deployment script
    ├── validate.py                # Validation script
    └── benchmark.py               # Performance benchmarking
```

### Dependencies (requirements.txt)

```
snowflake-snowpark-python>=1.11.0
snowflake-connector-python>=3.0.0
pandas>=2.0.0
pytest>=7.0.0
pytest-cov>=4.0.0
pydantic>=2.0.0
```

### Package Configuration (pyproject.toml)

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "maq-snowpark"
version = "1.0.0"
description = "MAQ Report Calculation UDFs for Snowflake"
requires-python = ">=3.8"
dependencies = [
    "snowflake-snowpark-python>=1.11.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0.0",
    "pytest-cov>=4.0.0",
    "black>=23.0.0",
    "mypy>=1.0.0",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
python_files = "test_*.py"

[tool.black]
line-length = 100
target-version = ["py38", "py39", "py310", "py311"]

[tool.mypy]
python_version = "3.10"
warn_return_any = true
warn_unused_configs = true
```

---

## UDF Specifications

### UDF 1: get_maq_factors

**Purpose:** Calculate all applicable MAQ factors for a hazard class and physical state

**Signature:**
```python
@udf(
    name="udfs.get_maq_factors",
    is_permanent=True,
    stage_location="@stages.python_udfs",
    packages=["snowflake-snowpark-python"],
    replace=True
)
def get_maq_factors(
    hazard_class_data: dict,      # Full hazard class object from fire code
    occupancy_rules: list,         # Floor-level rules for the occupancy
    physical_state: str,           # 'solid', 'liquid', 'gas'
    fire_suppression_type: str,    # 'FULL', 'BASEMENT_ONLY', 'NONE'
    floor_above_ground_plane: int, # Floor level
    is_outdoor: bool,              # Outdoor control area
    approved_storage: list         # List of approved storage objects
) -> list:
    """
    Returns array of factor objects:
    [
        { "label": "Baseline", "value": 30, "operator": "" },
        { "label": "Sprinkler", "value": 2, "operator": "MULTIPLY" },
        { "label": "Floor Level", "value": 0.75, "operator": "MULTIPLY" }
    ]
    """
```

**Implementation:** (See maq_factors.py in Python module)

**Test Cases:**
| Scenario | Inputs | Expected Output |
|----------|--------|-----------------|
| Basic baseline | baseline=30, no rules apply | `[{label: "Baseline", value: 30}]` |
| Sprinkler applies | sprinklered=true, rule exists | `[{Baseline: 30}, {Sprinkler: 2, MULTIPLY}]` |
| Floor reduction | floor=-1, rule exists | `[{Baseline: 30}, {Floor Level: 0.75, MULTIPLY}]` |
| OVERRIDE rule | rule with OVERRIDE | `[{Override Rule: value}]` |
| Exempt | is_exempt=true | `[{Exempt: null, isNoLimit: true}]` |

---

### UDF 2: calculate_limit_from_factors

**Purpose:** Reduce factor array to final limit value

**Signature:**
```python
@udf(name="udfs.calculate_limit_from_factors", ...)
def calculate_limit_from_factors(factors: list) -> dict:
    """
    Returns:
    {
        "value": 60.0,
        "displayLimit": "60.00",
        "isNoLimit": false,
        "isNotApplicable": false
    }
    """
```

**Logic:**
```
result = factors[0].value
for factor in factors[1:]:
    if factor.operator == "MULTIPLY":
        result = result * factor.value
return result
```

---

### UDF 3: get_compliance_status

**Purpose:** Determine compliance status from limit and actual

**Signature:**
```python
@udf(name="udfs.get_compliance_status", ...)
def get_compliance_status(limit: float, actual: float) -> str:
    """
    Returns: 'COMPLIANT' | 'NEAR_THRESHOLD' | 'OVER_THRESHOLD'
    """
```

**Logic:**
```
if limit is None: return 'COMPLIANT'
if limit == 0 and actual == 0: return 'COMPLIANT'
if actual >= limit: return 'OVER_THRESHOLD'
if actual >= limit * 0.8: return 'NEAR_THRESHOLD'
return 'COMPLIANT'
```

---

### UDF 4: get_floor_maq_factors

**Purpose:** Calculate floor-level reduction factors

**Signature:**
```python
@udf(name="udfs.get_floor_maq_factors", ...)
def get_floor_maq_factors(
    occupancy_rules: list,
    floor_above_ground_plane: int
) -> list:
    """
    Returns array of floor-level factors
    """
```

---

### UDF 5: evaluate_rule_conditions

**Purpose:** Check if all conditions in a rule apply

**Signature:**
```python
@udf(name="udfs.evaluate_rule_conditions", ...)
def evaluate_rule_conditions(
    rule: dict,
    is_sprinklered: bool,
    fire_suppression_type: str,
    floor_above_ground_plane: int,
    is_outdoor: bool,
    approved_storage: list,
    hazard_class_name: str,
    physical_state: str
) -> bool:
    """
    Returns True if all conditions in rule apply
    """
```

---

### UDF 6: get_open_closed_limits

**Purpose:** Calculate open/closed MAQ limits for HMIS reports

**Signature:**
```python
@udf(name="udfs.get_open_closed_limits", ...)
def get_open_closed_limits(
    hazard_class_name: str,
    physical_state: str,
    storage_limit: float,
    is_sprinklered: bool
) -> dict:
    """
    Returns:
    {
        "closedLimit": 120,
        "openLimit": 30,
        "displayClosedLimit": "120.00",
        "displayOpenLimit": "30.00"
    }
    """
```

---

### SQL UDFs

```sql
-- Unit Conversions
CREATE OR REPLACE FUNCTION udfs.convert_gram_to_pound(grams FLOAT)
RETURNS FLOAT
LANGUAGE SQL
AS 'COALESCE(grams, 0) * 0.0022046226';

CREATE OR REPLACE FUNCTION udfs.convert_liter_to_gallon(liters FLOAT)
RETURNS FLOAT
LANGUAGE SQL
AS 'COALESCE(liters, 0) * 0.2641720524';

CREATE OR REPLACE FUNCTION udfs.convert_liter_to_cubic_feet(liters FLOAT)
RETURNS FLOAT
LANGUAGE SQL
AS 'COALESCE(liters, 0) * 0.0353146667';

-- Display Value Formatting
CREATE OR REPLACE FUNCTION udfs.format_display_value(value FLOAT)
RETURNS VARCHAR
LANGUAGE SQL
AS $$
    CASE
        WHEN value IS NULL THEN '0.00'
        WHEN value = 0 THEN '0.00'
        WHEN value < 0.0001 THEN '< 0.0001'
        WHEN value < 0.01 THEN TO_VARCHAR(value, '0.0000')
        ELSE TO_VARCHAR(value, '999990.00')
    END
$$;
```

---

## SQL Views Specification

### Main MAQ Report View

```sql
CREATE OR REPLACE VIEW marts.maq_report AS
WITH control_area_hazard_classes AS (
    SELECT
        cab.control_area_id,
        cab.control_area_name,
        cab.building_id,
        cab.building_name,
        cab.occupancy,
        cab.floor_above_ground_plane,
        cab.is_outdoor,
        cab.is_exempt,
        cab.exemption_reason,
        cab.is_sprinklered,
        cab.effective_fire_suppression_type,
        cab.approved_storage,
        hc.hazard_class_id,
        hc.hazard_class_name,
        hc.is_health_hazard,
        hc.hazard_class_data,
        hc.occupancy_rules,
        hc.hmis_report,
        hc.units_solid,
        hc.units_liquid,
        hc.units_gas,
        hc.gas_report_as_liquid
    FROM staging.stg_control_area_base cab
    JOIN staging.stg_hazard_classes hc
        ON hc.occupancy_name = cab.occupancy
        AND hc.fire_code_id = cab.fire_code_id
    WHERE NOT cab.is_exempt
),

calculated_limits AS (
    SELECT
        cahc.*,
        -- Calculate MAQ factors for each state
        udfs.get_maq_factors(
            cahc.hazard_class_data,
            cahc.occupancy_rules,
            'solid',
            cahc.effective_fire_suppression_type,
            cahc.floor_above_ground_plane,
            cahc.is_outdoor,
            cahc.approved_storage
        ) AS maq_factors_solid,
        udfs.get_maq_factors(
            cahc.hazard_class_data,
            cahc.occupancy_rules,
            'liquid',
            cahc.effective_fire_suppression_type,
            cahc.floor_above_ground_plane,
            cahc.is_outdoor,
            cahc.approved_storage
        ) AS maq_factors_liquid,
        udfs.get_maq_factors(
            cahc.hazard_class_data,
            cahc.occupancy_rules,
            'gas',
            cahc.effective_fire_suppression_type,
            cahc.floor_above_ground_plane,
            cahc.is_outdoor,
            cahc.approved_storage
        ) AS maq_factors_gas
    FROM control_area_hazard_classes cahc
),

limits_with_values AS (
    SELECT
        cl.*,
        udfs.calculate_limit_from_factors(cl.maq_factors_solid) AS limit_result_solid,
        udfs.calculate_limit_from_factors(cl.maq_factors_liquid) AS limit_result_liquid,
        udfs.calculate_limit_from_factors(cl.maq_factors_gas) AS limit_result_gas
    FROM calculated_limits cl
),

with_actuals AS (
    SELECT
        lwv.*,
        -- Solid actuals
        COALESCE(act_solid.total_grams, 0) AS actual_grams_solid,
        udfs.convert_gram_to_pound(COALESCE(act_solid.total_grams, 0)) AS actual_solid,
        -- Liquid actuals (may need volume or weight depending on units)
        COALESCE(act_liquid.total_liters, 0) AS actual_liters_liquid,
        COALESCE(act_liquid.total_grams, 0) AS actual_grams_liquid,
        CASE lwv.units_liquid
            WHEN 'gal' THEN udfs.convert_liter_to_gallon(COALESCE(act_liquid.total_liters, 0))
            WHEN 'ft3' THEN udfs.convert_liter_to_cubic_feet(COALESCE(act_liquid.total_liters, 0))
            ELSE udfs.convert_gram_to_pound(COALESCE(act_liquid.total_grams, 0))
        END AS actual_liquid,
        -- Gas actuals (handle reportAsLiquid)
        COALESCE(
            CASE WHEN lwv.gas_report_as_liquid THEN act_liquid.total_liters ELSE act_gas.total_liters END,
            0
        ) AS actual_liters_gas,
        udfs.convert_liter_to_cubic_feet(
            COALESCE(
                CASE WHEN lwv.gas_report_as_liquid THEN act_liquid.total_liters ELSE act_gas.total_liters END,
                0
            )
        ) AS actual_gas
    FROM limits_with_values lwv
    LEFT JOIN staging.stg_container_actuals act_solid
        ON lwv.control_area_id = act_solid.control_area_id
        AND lwv.hazard_class_name = act_solid.hazard_class_name
        AND act_solid.physical_state = 'solid'
    LEFT JOIN staging.stg_container_actuals act_liquid
        ON lwv.control_area_id = act_liquid.control_area_id
        AND lwv.hazard_class_name = act_liquid.hazard_class_name
        AND act_liquid.physical_state = 'liquid'
    LEFT JOIN staging.stg_container_actuals act_gas
        ON lwv.control_area_id = act_gas.control_area_id
        AND lwv.hazard_class_name = act_gas.hazard_class_name
        AND act_gas.physical_state = 'gas'
)

SELECT
    -- Identifiers
    wa.control_area_id,
    wa.control_area_name,
    wa.building_id,
    wa.building_name,
    wa.hazard_class_id,
    wa.hazard_class_name,
    wa.is_health_hazard,
    wa.occupancy,

    -- Solid
    wa.limit_result_solid:value::FLOAT AS limit_solid,
    wa.limit_result_solid:displayLimit::VARCHAR AS display_limit_solid,
    wa.limit_result_solid:isNoLimit::BOOLEAN AS is_no_limit_solid,
    wa.limit_result_solid:isNotApplicable::BOOLEAN AS is_not_applicable_solid,
    wa.actual_solid,
    udfs.format_display_value(wa.actual_solid) AS display_actual_solid,
    wa.units_solid,
    wa.maq_factors_solid,
    udfs.get_compliance_status(
        wa.limit_result_solid:value::FLOAT,
        wa.actual_solid
    ) AS status_solid,

    -- Liquid
    wa.limit_result_liquid:value::FLOAT AS limit_liquid,
    wa.limit_result_liquid:displayLimit::VARCHAR AS display_limit_liquid,
    wa.limit_result_liquid:isNoLimit::BOOLEAN AS is_no_limit_liquid,
    wa.limit_result_liquid:isNotApplicable::BOOLEAN AS is_not_applicable_liquid,
    wa.actual_liquid,
    udfs.format_display_value(wa.actual_liquid) AS display_actual_liquid,
    wa.units_liquid,
    wa.maq_factors_liquid,
    udfs.get_compliance_status(
        wa.limit_result_liquid:value::FLOAT,
        wa.actual_liquid
    ) AS status_liquid,

    -- Gas
    wa.limit_result_gas:value::FLOAT AS limit_gas,
    wa.limit_result_gas:displayLimit::VARCHAR AS display_limit_gas,
    wa.limit_result_gas:isNoLimit::BOOLEAN AS is_no_limit_gas,
    wa.limit_result_gas:isNotApplicable::BOOLEAN AS is_not_applicable_gas,
    wa.actual_gas,
    udfs.format_display_value(wa.actual_gas) AS display_actual_gas,
    wa.units_gas,
    wa.maq_factors_gas,
    udfs.get_compliance_status(
        wa.limit_result_gas:value::FLOAT,
        wa.actual_gas
    ) AS status_gas,

    -- HMIS (if applicable)
    wa.hmis_report,
    CASE WHEN wa.hmis_report THEN
        udfs.get_open_closed_limits(wa.hazard_class_name, 'solid', wa.limit_result_solid:value::FLOAT, wa.is_sprinklered)
    END AS hmis_solid,
    CASE WHEN wa.hmis_report THEN
        udfs.get_open_closed_limits(wa.hazard_class_name, 'liquid', wa.limit_result_liquid:value::FLOAT, wa.is_sprinklered)
    END AS hmis_liquid,

    -- Metadata
    CURRENT_TIMESTAMP() AS calculated_at

FROM with_actuals wa;
```

### Building Summary View

```sql
CREATE OR REPLACE VIEW marts.maq_building_summary AS
SELECT
    building_id,
    building_name,
    COUNT(DISTINCT control_area_id) AS control_area_count,
    COUNT(DISTINCT hazard_class_name) AS hazard_class_count,
    -- Overall status (worst of all)
    CASE
        WHEN SUM(CASE WHEN status_solid = 'OVER_THRESHOLD' THEN 1 ELSE 0 END) > 0
          OR SUM(CASE WHEN status_liquid = 'OVER_THRESHOLD' THEN 1 ELSE 0 END) > 0
          OR SUM(CASE WHEN status_gas = 'OVER_THRESHOLD' THEN 1 ELSE 0 END) > 0
        THEN 'OVER_THRESHOLD'
        WHEN SUM(CASE WHEN status_solid = 'NEAR_THRESHOLD' THEN 1 ELSE 0 END) > 0
          OR SUM(CASE WHEN status_liquid = 'NEAR_THRESHOLD' THEN 1 ELSE 0 END) > 0
          OR SUM(CASE WHEN status_gas = 'NEAR_THRESHOLD' THEN 1 ELSE 0 END) > 0
        THEN 'NEAR_THRESHOLD'
        ELSE 'COMPLIANT'
    END AS overall_status,
    -- Counts by status
    SUM(CASE WHEN status_solid = 'OVER_THRESHOLD' OR status_liquid = 'OVER_THRESHOLD' OR status_gas = 'OVER_THRESHOLD' THEN 1 ELSE 0 END) AS over_threshold_count,
    SUM(CASE WHEN status_solid = 'NEAR_THRESHOLD' OR status_liquid = 'NEAR_THRESHOLD' OR status_gas = 'NEAR_THRESHOLD' THEN 1 ELSE 0 END) AS near_threshold_count,
    MAX(calculated_at) AS last_calculated
FROM marts.maq_report
GROUP BY building_id, building_name;
```

---

## Stored Procedures Specification

### Refresh MAQ Cache

```sql
CREATE OR REPLACE PROCEDURE procedures.refresh_maq_cache()
RETURNS VARCHAR
LANGUAGE SQL
EXECUTE AS CALLER
AS
$$
DECLARE
    start_time TIMESTAMP_NTZ;
    end_time TIMESTAMP_NTZ;
    row_count INT;
BEGIN
    start_time := CURRENT_TIMESTAMP();

    -- Truncate and reload cache
    TRUNCATE TABLE marts.maq_report_cache;

    INSERT INTO marts.maq_report_cache
    SELECT *, CURRENT_TIMESTAMP() AS cached_at
    FROM marts.maq_report;

    SELECT COUNT(*) INTO row_count FROM marts.maq_report_cache;

    end_time := CURRENT_TIMESTAMP();

    -- Log execution
    INSERT INTO procedures.execution_log (procedure_name, start_time, end_time, row_count, status)
    VALUES ('refresh_maq_cache', :start_time, :end_time, :row_count, 'SUCCESS');

    RETURN 'Refreshed ' || :row_count || ' rows in ' || DATEDIFF('second', :start_time, :end_time) || ' seconds';

EXCEPTION
    WHEN OTHER THEN
        INSERT INTO procedures.execution_log (procedure_name, start_time, end_time, status, error_message)
        VALUES ('refresh_maq_cache', :start_time, CURRENT_TIMESTAMP(), 'ERROR', SQLERRM);
        RAISE;
END;
$$;
```

### Validate Calculations

```sql
CREATE OR REPLACE PROCEDURE procedures.validate_calculations()
RETURNS TABLE (
    control_area_id VARCHAR,
    hazard_class_name VARCHAR,
    field VARCHAR,
    snowflake_value FLOAT,
    expected_value FLOAT,
    difference FLOAT,
    status VARCHAR
)
LANGUAGE SQL
AS
$$
    -- Compare Snowflake calculations with expected values from source
    SELECT
        sf.control_area_id,
        sf.hazard_class_name,
        'limit_solid' AS field,
        sf.limit_solid AS snowflake_value,
        exp.limit_solid AS expected_value,
        ABS(sf.limit_solid - exp.limit_solid) AS difference,
        CASE
            WHEN ABS(sf.limit_solid - exp.limit_solid) < 0.01 THEN 'PASS'
            ELSE 'FAIL'
        END AS status
    FROM marts.maq_report sf
    LEFT JOIN raw.expected_values exp  -- Populated from current system
        ON sf.control_area_id = exp.control_area_id
        AND sf.hazard_class_name = exp.hazard_class_name
    WHERE ABS(sf.limit_solid - exp.limit_solid) >= 0.01

    UNION ALL

    -- Similar for liquid, gas, actuals, status...
$$;
```

---

## Deployment Pipeline

### CI/CD Workflow

```yaml
# .github/workflows/deploy-snowpark.yml
name: Deploy MAQ Snowpark

on:
  push:
    branches: [main]
    paths:
      - 'maq_snowpark/**'
  workflow_dispatch:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: |
          pip install -e ".[dev]"
        working-directory: maq_snowpark

      - name: Run tests
        run: pytest --cov=src tests/
        working-directory: maq_snowpark

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'

      - name: Install dependencies
        run: pip install snowflake-snowpark-python

      - name: Deploy UDFs
        env:
          SNOWFLAKE_ACCOUNT: ${{ secrets.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ secrets.SNOWFLAKE_USER }}
          SNOWFLAKE_PASSWORD: ${{ secrets.SNOWFLAKE_PASSWORD }}
          SNOWFLAKE_WAREHOUSE: MAQ_ETL_WH
          SNOWFLAKE_DATABASE: MAQ_DB
        run: python scripts/deploy.py
        working-directory: maq_snowpark

      - name: Run validation
        run: python scripts/validate.py
        working-directory: maq_snowpark
```

### Deployment Script

```python
# scripts/deploy.py
from snowflake.snowpark import Session
import os

def create_session():
    return Session.builder.configs({
        "account": os.environ["SNOWFLAKE_ACCOUNT"],
        "user": os.environ["SNOWFLAKE_USER"],
        "password": os.environ["SNOWFLAKE_PASSWORD"],
        "warehouse": os.environ["SNOWFLAKE_WAREHOUSE"],
        "database": os.environ["SNOWFLAKE_DATABASE"],
    }).create()

def deploy_udfs(session: Session):
    """Deploy all Python UDFs to Snowflake."""
    from src.maq_snowpark.deploy.register_udfs import register_all_udfs
    register_all_udfs(session)

def deploy_sql_objects(session: Session):
    """Deploy SQL views, procedures, tasks."""
    sql_files = [
        "sql/schemas.sql",
        "sql/raw_tables.sql",
        "sql/staging_views.sql",
        "sql/mart_views.sql",
        "sql/procedures.sql",
        "sql/tasks.sql",
    ]
    for sql_file in sql_files:
        with open(sql_file) as f:
            for statement in f.read().split(';'):
                if statement.strip():
                    session.sql(statement).collect()

def main():
    session = create_session()
    print("Deploying UDFs...")
    deploy_udfs(session)
    print("Deploying SQL objects...")
    deploy_sql_objects(session)
    print("Deployment complete!")

if __name__ == "__main__":
    main()
```

---

## Testing Strategy

### Test Levels

| Level | What | How | Coverage Target |
|-------|------|-----|-----------------|
| Unit | Individual UDF functions | pytest with mock data | 100% of functions |
| Integration | UDFs in Snowflake | Snowpark test session | All UDFs |
| Validation | Full calculation accuracy | Compare with current system | 100% match |
| Performance | Query response times | Benchmark scripts | < 5s per building |

### Unit Test Example

```python
# tests/test_maq_factors.py
import pytest
from src.maq_snowpark.udfs.maq_factors import get_maq_factors

class TestGetMaqFactors:
    def test_baseline_only(self):
        """Test basic baseline with no rules applying."""
        hazard_class = {
            "name": "Flammable Liquid: IA",
            "solid": {"baseline": 30, "units": "lbs"},
            "liquid": {"baseline": 30, "units": "gal"},
            "gas": {"baseline": None, "isNotApplicable": True},
            "rules": []
        }

        factors = get_maq_factors(
            hazard_class_data=hazard_class,
            occupancy_rules=[],
            physical_state="liquid",
            fire_suppression_type="NONE",
            floor_above_ground_plane=0,
            is_outdoor=False,
            approved_storage=[]
        )

        assert len(factors) == 1
        assert factors[0]["label"] == "Baseline"
        assert factors[0]["value"] == 30

    def test_sprinkler_multiplier(self):
        """Test that sprinkler rule applies 2x multiplier."""
        hazard_class = {
            "name": "Flammable Liquid: IA",
            "liquid": {"baseline": 30, "units": "gal"},
            "rules": [{
                "conditions": [{"criteria": "SPRINKLER", "operator": "is", "value": "true"}],
                "liquid": {"value": 2, "operator": "MULTIPLY"}
            }]
        }

        factors = get_maq_factors(
            hazard_class_data=hazard_class,
            occupancy_rules=[],
            physical_state="liquid",
            fire_suppression_type="FULL",  # Sprinklered
            floor_above_ground_plane=0,
            is_outdoor=False,
            approved_storage=[]
        )

        assert len(factors) == 2
        assert factors[0]["label"] == "Baseline"
        assert factors[0]["value"] == 30
        assert factors[1]["label"] == "SPRINKLER"
        assert factors[1]["value"] == 2
        assert factors[1]["operator"] == "MULTIPLY"

    def test_floor_level_reduction(self):
        """Test basement floor level reduction."""
        # ... test implementation

    def test_override_rule(self):
        """Test that OVERRIDE replaces entire calculation."""
        # ... test implementation

    def test_approved_storage(self):
        """Test approved storage multiplier."""
        # ... test implementation
```

### Validation Test

```python
# tests/test_integration.py
import pytest
from snowflake.snowpark import Session

@pytest.fixture
def snowflake_session():
    return Session.builder.configs({...}).create()

def test_calculation_parity(snowflake_session):
    """Compare Snowflake calculations with expected values."""
    # Load expected values from current system
    expected = load_expected_values()

    # Query Snowflake
    result = snowflake_session.sql("""
        SELECT control_area_id, hazard_class_name,
               limit_solid, limit_liquid, limit_gas,
               status_solid, status_liquid, status_gas
        FROM marts.maq_report
        WHERE control_area_id IN ({})
    """.format(",".join(f"'{id}'" for id in expected.keys())))

    # Compare
    for row in result.collect():
        exp = expected[row["CONTROL_AREA_ID"]][row["HAZARD_CLASS_NAME"]]
        assert abs(row["LIMIT_SOLID"] - exp["limit_solid"]) < 0.01
        assert row["STATUS_SOLID"] == exp["status_solid"]
        # ... etc
```

---

## Monitoring & Observability

### Metrics to Track

| Metric | Source | Alert Threshold |
|--------|--------|-----------------|
| Cache refresh duration | Procedure log | > 5 minutes |
| Cache refresh failures | Procedure log | Any failure |
| Query latency (P95) | Query history | > 10 seconds |
| Data freshness | Cache timestamp | > 30 minutes stale |
| Calculation mismatches | Validation procedure | Any mismatch |
| Warehouse credit usage | Account usage | > daily budget |

### Execution Log Table

```sql
CREATE TABLE procedures.execution_log (
    id INT AUTOINCREMENT PRIMARY KEY,
    procedure_name VARCHAR,
    start_time TIMESTAMP_NTZ,
    end_time TIMESTAMP_NTZ,
    row_count INT,
    status VARCHAR,
    error_message VARCHAR,
    CONSTRAINT chk_status CHECK (status IN ('SUCCESS', 'ERROR', 'WARNING'))
);
```

### Alerting (via Snowflake Alerts)

```sql
-- Alert on cache refresh failure
CREATE OR REPLACE ALERT alerts.cache_refresh_failed
    WAREHOUSE = MAQ_TASK_WH
    SCHEDULE = 'USING CRON 0/15 * * * *'
    IF (EXISTS (
        SELECT 1 FROM procedures.execution_log
        WHERE procedure_name = 'refresh_maq_cache'
          AND status = 'ERROR'
          AND start_time > DATEADD('minute', -20, CURRENT_TIMESTAMP())
    ))
    THEN
        CALL SYSTEM$SEND_EMAIL(
            'maq-alerts@company.com',
            'MAQ Cache Refresh Failed',
            'The MAQ cache refresh procedure failed. Check execution_log for details.'
        );
```

---

## Maintenance Considerations

### Regular Maintenance Tasks

| Task | Frequency | Effort | Owner |
|------|-----------|--------|-------|
| Monitor cache refresh | Daily | 5 min | Ops |
| Review execution logs | Weekly | 15 min | Ops |
| Validate calculations | Weekly | 30 min | Data team |
| Update fire code rules | As needed | 2-4 hrs | Data team |
| UDF performance tuning | Quarterly | 4-8 hrs | Data eng |
| Snowflake version upgrades | Quarterly | 2-4 hrs | Data eng |
| Security review | Annually | 4-8 hrs | Security |

### Change Management

| Change Type | Impact | Process |
|-------------|--------|---------|
| Fire code rule change | High | Update hazard class rules in MongoDB, verify sync |
| New hazard class | Medium | Add to fire code, update band mappings |
| UDF logic change | High | Update Python, test, deploy via CI/CD |
| New occupancy type | Medium | Update fire code, verify calculations |
| Schema change | High | Migration script, backfill, deploy |

### Runbooks

1. **Cache Refresh Failure**
   - Check `procedures.execution_log` for error
   - Check warehouse availability
   - Check source data availability
   - Manually run `CALL procedures.refresh_maq_cache()`

2. **Calculation Mismatch**
   - Run `CALL procedures.validate_calculations()`
   - Identify affected control areas
   - Check fire code rules for recent changes
   - Compare with source system calculation

3. **Performance Degradation**
   - Check warehouse size and utilization
   - Check query profile for slow steps
   - Consider materialized views for hot paths
   - Review clustering keys

---

## Cost Analysis

### Infrastructure Costs (Monthly Estimates)

| Component | Size/Units | Cost/Month | Notes |
|-----------|------------|------------|-------|
| **Compute** | | | |
| MAQ_ETL_WH | X-Small, ~2 hrs/day | ~$60 | Data loading |
| MAQ_COMPUTE_WH | Small, ~4 hrs/day | ~$240 | Query execution |
| MAQ_TASK_WH | X-Small, ~1 hr/day | ~$30 | Scheduled tasks |
| **Storage** | | | |
| Raw tables | ~10 GB | ~$23 | $2.30/TB |
| Staging views | 0 (views) | $0 | |
| Cache table | ~1 GB | ~$2 | |
| **Data Transfer** | | | |
| Ingestion | ~100 GB/month | ~$5 | $0.05/GB |
| **Total Infrastructure** | | **~$360/month** | |

### Development Costs (One-Time)

| Phase | Effort (Person-Days) | Cost @ $800/day |
|-------|---------------------|-----------------|
| Architecture & Design | 5 | $4,000 |
| Data Pipeline Setup | 10 | $8,000 |
| Python UDF Development | 15 | $12,000 |
| SQL Views Development | 8 | $6,400 |
| Testing & Validation | 10 | $8,000 |
| Documentation | 3 | $2,400 |
| Deployment & Cutover | 5 | $4,000 |
| **Total Development** | **56 days** | **$44,800** |

### Ongoing Maintenance Costs (Monthly)

| Activity | Hours/Month | Cost @ $100/hr |
|----------|-------------|----------------|
| Monitoring & alerting | 4 | $400 |
| Bug fixes & updates | 8 | $800 |
| Performance tuning | 4 | $400 |
| Documentation updates | 2 | $200 |
| **Total Maintenance** | **18 hrs** | **$1,800/month** |

### Total Cost of Ownership (Year 1)

| Category | Cost |
|----------|------|
| Development | $44,800 |
| Infrastructure (12 months) | $4,320 |
| Maintenance (12 months) | $21,600 |
| **Total Year 1** | **$70,720** |

### Total Cost of Ownership (Year 2+)

| Category | Cost |
|----------|------|
| Infrastructure | $4,320 |
| Maintenance | $21,600 |
| **Total Annual** | **$25,920** |

---

## Risk Assessment

### Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Calculation mismatch with current system | Medium | High | Extensive validation testing, parallel run period |
| Python UDF performance issues | Medium | Medium | Benchmark early, consider JS UDFs for hot paths |
| Data sync latency | Low | Medium | Snowpipe for real-time, monitoring for staleness |
| Snowflake outage | Low | High | Multi-region deployment (future), alerting |
| Fire code complexity not fully captured | Medium | High | Thorough review of all edge cases before dev |

### Operational Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Knowledge concentration | Medium | Medium | Documentation, cross-training |
| Cost overrun | Low | Medium | Budget alerts, warehouse size limits |
| Schema drift from source | Medium | Low | Schema validation in pipeline |
| Security vulnerabilities | Low | High | Regular security reviews, least privilege |

### Migration Risks

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Extended parallel run period | Medium | Low | Automate comparison, define acceptance criteria |
| User resistance | Low | Medium | Training, gradual rollout |
| Rollback required | Low | Medium | Keep current system operational during migration |

---

## Implementation Checklist

### Phase 1: Foundation (Week 1-2)
- [ ] Create Snowflake database, schemas, warehouses
- [ ] Create raw tables with correct schemas
- [ ] Set up data pipeline (Snowpipe or ETL)
- [ ] Initial data load and validation
- [ ] Create staging views

### Phase 2: Core Logic (Week 3-5)
- [ ] Develop Python UDFs locally with tests
- [ ] Deploy UDFs to Snowflake
- [ ] Create SQL UDFs for conversions
- [ ] Create main MAQ report view
- [ ] Validate against sample data

### Phase 3: Integration (Week 6-7)
- [ ] Create all staging views
- [ ] Handle edge cases (reportAsLiquid, exemptions, etc.)
- [ ] Create building summary view
- [ ] Create HMIS calculations (if needed)
- [ ] Create cache table and refresh procedure

### Phase 4: Validation (Week 8-9)
- [ ] Load expected values from current system
- [ ] Run full comparison
- [ ] Fix any discrepancies
- [ ] Performance benchmarking
- [ ] Document any intentional differences

### Phase 5: Production Readiness (Week 10-11)
- [ ] Set up scheduled tasks
- [ ] Set up monitoring and alerting
- [ ] Create runbooks
- [ ] Security review
- [ ] Load testing

### Phase 6: Cutover (Week 12)
- [ ] Final validation
- [ ] User training
- [ ] Cutover to Snowflake reporting
- [ ] Monitor closely for 1 week
- [ ] Decommission parallel run

---

*Document Version: 1.0*
*Last Updated: December 2024*

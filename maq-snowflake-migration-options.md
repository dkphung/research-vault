# MAQ Report Snowflake Migration Options

This document analyzes options for reproducing the MAQ (Maximum Allowable Quantity) compliance report using only Snowflake, assuming all raw data can be pushed to Snowflake.

---

## Table of Contents

1. [First Principles Analysis](#first-principles-analysis)
2. [Current System Overview](#current-system-overview)
3. [Data Requirements](#data-requirements)
4. [Migration Options](#migration-options)
   - [Option 1: Pure SQL Views](#option-1-pure-sql-views)
   - [Option 2: SQL + JavaScript UDFs](#option-2-sql--javascript-udfs)
   - [Option 3: Snowpark Python](#option-3-snowpark-python)
   - [Option 4: dbt + Jinja](#option-4-dbt--jinja)
   - [Option 5: Materialized Views + Scheduled Refresh](#option-5-materialized-views--scheduled-refresh)
5. [Recommended Architecture](#recommended-architecture)
6. [Critical Considerations](#critical-considerations)
7. [Implementation Roadmap](#implementation-roadmap)

---

## First Principles Analysis

### What is the fundamental problem?

**Transform raw data → Compliance report showing actual quantities vs calculated limits per hazard class per control area**

### Atomic Operations Required

| Operation | Description |
|-----------|-------------|
| **Join entities** | Building → Floors → Rooms → Control Areas → Containers |
| **Evaluate conditional rules** | Sprinkler status, floor level, approved storage, outdoor |
| **Calculate MAQ limits** | `baseline × factor1 × factor2 × ...` |
| **Aggregate actual quantities** | Sum containers by hazard class + physical state |
| **Convert units** | gram→pound, liter→gallon, liter→ft³ |
| **Determine compliance** | Compare actual vs limit (80% and 100% thresholds) |

---

## Current System Overview

### Data Sources

| Source | Type | Data |
|--------|------|------|
| MongoDB: `fireCode` | Document DB | Fire code rules, baselines, occupancies, hazard class mappings |
| MongoDB: `controlArea` | Document DB | Control area metadata, exemptions, approved storage |
| Elasticsearch: `container` | Search Index | Container quantities, locations, exemption status |
| External Building Service | REST API | Building info, floors, rooms, fire suppression type |

### Core Calculation Logic

The calculation logic lives in `packages/common/src/utils/maq-compliance.helper.ts` (~900 lines):

```
MAQ Limit = Baseline × Sprinkler Factor × Floor Factor × Approved Storage Factor × ...
```

Key functions:
- `getMaqFactors()` - Evaluates all applicable rules and returns factor array
- `calculateLimitByState()` - Reduces factors to final limit value
- `getComplianceStatus()` - Compares actual vs limit for status
- `getControlAreaMaq()` - Orchestrates the full calculation

### Rule Operators

| Operator | Effect |
|----------|--------|
| `MULTIPLY` | Multiply current value by rule value |
| `EQUALS` | Replace baseline with rule value |
| `OVERRIDE` | Replace entire calculation with rule value |
| `REPORT_AS_LIQUID` | Gas values reported under liquid |

### Compliance Thresholds

| Status | Condition |
|--------|-----------|
| `COMPLIANT` | actual < 80% of limit OR limit is null (NL) |
| `NEAR_THRESHOLD` | actual >= 80% of limit AND actual < limit |
| `OVER_THRESHOLD` | actual >= limit |
| `EXEMPT` | control area marked exempt |

---

## Data Requirements

### Snowflake Table Mapping

| Current Source | Snowflake Table | Key Fields |
|----------------|-----------------|------------|
| MongoDB: `fireCode` | `fire_codes` | occupancies (VARIANT), hazard_class_mappings (VARIANT) |
| MongoDB: `controlArea` | `control_areas` | occupancy, is_outdoor, exemption, approved_storage |
| Elasticsearch: `container` | `containers` | normalized_size_gram, normalized_size_liter, room_id, family_form_at_ntp |
| External: Building Service | `buildings`, `floors`, `rooms` | fire_suppression_type, ground_plane, control_area_id |

### Unit Conversion Constants

```sql
-- To be used in calculations
gram_to_pound = 0.0022046226
liter_to_gallon = 0.2641720524
liter_to_cubic_feet = 0.0353146667
```

---

## Migration Options

### Option 1: Pure SQL Views

**Approach**: Replicate calculation logic as a chain of Snowflake views

```
RAW TABLES → JOINED_VIEW → LIMITS_VIEW → ACTUALS_VIEW → REPORT_VIEW
```

#### Implementation Example

```sql
-- Step 1: Joined base view
CREATE OR REPLACE VIEW control_area_base AS
SELECT
    ca.id AS control_area_id,
    ca.name AS control_area_name,
    ca.occupancy,
    ca.is_outdoor,
    ca.floor_above_ground_plane,
    b.id AS building_id,
    b.fire_suppression_type,
    -- Determine effective sprinkler coverage
    CASE
        WHEN ca.fire_suppression_override_enabled = TRUE
        THEN ca.fire_suppression_override_value
        WHEN b.fire_suppression_type = 'FULL' THEN TRUE
        WHEN b.fire_suppression_type = 'BASEMENT_ONLY'
             AND ca.floor_above_ground_plane < 0 THEN TRUE
        ELSE FALSE
    END AS is_sprinklered,
    ca.approved_storage,
    ca.exemption_is_exempt,
    ca.exemption_reason
FROM control_areas ca
JOIN buildings b ON ca.building_id = b.id;

-- Step 2: Calculate limits with CASE statements for rules
CREATE OR REPLACE VIEW hazard_class_limits AS
SELECT
    cab.control_area_id,
    cab.building_id,
    hc.name AS hazard_class_name,
    hc.is_health_hazard,
    -- Solid limit calculation
    hc.baseline_solid *
        -- Sprinkler factor
        CASE WHEN cab.is_sprinklered THEN 2 ELSE 1 END *
        -- Floor level factor (simplified)
        CASE
            WHEN cab.is_outdoor THEN 1
            WHEN cab.floor_above_ground_plane < 0 THEN 0.75
            WHEN cab.floor_above_ground_plane > 2 THEN 0.5
            ELSE 1
        END AS limit_solid,
    hc.units_solid,
    -- Similar for liquid and gas...
    hc.baseline_liquid,
    hc.units_liquid,
    hc.baseline_gas,
    hc.units_gas
FROM control_area_base cab
CROSS JOIN hazard_classes hc
WHERE hc.occupancy = cab.occupancy;

-- Step 3: Aggregate actuals
CREATE OR REPLACE VIEW container_actuals AS
SELECT
    r.control_area_id,
    c.hazard_class_name,
    c.physical_state,
    SUM(c.normalized_size_gram) AS total_grams,
    SUM(c.normalized_size_liter) AS total_liters
FROM containers c
JOIN rooms r ON c.room_id = r.id
WHERE c.active = TRUE
  AND (c.exemption IS NULL OR c.exemption = '')
GROUP BY r.control_area_id, c.hazard_class_name, c.physical_state;

-- Step 4: Final report view
CREATE OR REPLACE VIEW maq_report AS
SELECT
    hcl.control_area_id,
    hcl.building_id,
    hcl.hazard_class_name,
    hcl.is_health_hazard,
    -- Solid
    hcl.limit_solid,
    COALESCE(ca_solid.total_grams * 0.0022046226, 0) AS actual_solid_lbs,
    hcl.units_solid,
    CASE
        WHEN hcl.limit_solid IS NULL THEN 'COMPLIANT'
        WHEN COALESCE(ca_solid.total_grams * 0.0022046226, 0) >= hcl.limit_solid THEN 'OVER_THRESHOLD'
        WHEN COALESCE(ca_solid.total_grams * 0.0022046226, 0) >= hcl.limit_solid * 0.8 THEN 'NEAR_THRESHOLD'
        ELSE 'COMPLIANT'
    END AS status_solid
    -- Similar for liquid and gas...
FROM hazard_class_limits hcl
LEFT JOIN container_actuals ca_solid
    ON hcl.control_area_id = ca_solid.control_area_id
    AND hcl.hazard_class_name = ca_solid.hazard_class_name
    AND ca_solid.physical_state = 'solid';
```

#### Pros
- No external dependencies
- Queryable directly in Snowflake
- Transparent logic
- Easy to debug with intermediate views

#### Cons
- CASE statements become deeply nested (hard to maintain)
- Rule priority/ordering is complex in SQL
- OVERRIDE operator breaks the model
- ~20+ views needed for full implementation
- Difficult to handle dynamic rule evaluation

#### Verdict
**Works for basic cases, breaks on complex rules.** Best suited if you can simplify the rule engine.

---

### Option 2: SQL + JavaScript UDFs

**Approach**: Encapsulate complex rule evaluation in JavaScript UDFs, keep joins in SQL

#### Implementation Example

```sql
-- JavaScript UDF for MAQ factor calculation
CREATE OR REPLACE FUNCTION calculate_maq_factors(
    baseline FLOAT,
    rules VARIANT,
    is_sprinklered BOOLEAN,
    floor_level INT,
    is_outdoor BOOLEAN,
    approved_storage VARIANT,
    physical_state STRING
)
RETURNS VARIANT
LANGUAGE JAVASCRIPT
AS $$
    // Helper functions
    function isTrue(val) {
        return val === true || val === 'true';
    }

    function conditionApplies(condition, isSprinklered, floorLevel, isOutdoor, approvedStorage, physicalState) {
        switch (condition.criteria) {
            case 'SPRINKLER':
                return isTrue(condition.value) === isSprinklered;
            case 'OUTDOOR':
                return isTrue(condition.value) === isOutdoor;
            case 'BASEMENT':
                return isTrue(condition.value) === (floorLevel < 0);
            case 'APPROVED_STORAGE':
                if (!approvedStorage) return !isTrue(condition.value);
                const hasApproved = approvedStorage.some(s =>
                    s.physicalStates && s.physicalStates.includes(physicalState)
                );
                return isTrue(condition.value) === hasApproved;
            default:
                return false;
        }
    }

    function ruleApplies(rule, isSprinklered, floorLevel, isOutdoor, approvedStorage, physicalState) {
        if (!rule.conditions) return false;
        return rule.conditions.every(c =>
            conditionApplies(c, isSprinklered, floorLevel, isOutdoor, approvedStorage, physicalState)
        );
    }

    // Start with baseline
    let factors = [{
        label: 'Baseline',
        value: BASELINE,
        operator: ''
    }];

    if (BASELINE === null) {
        return factors;
    }

    // Apply rules
    if (RULES) {
        for (const rule of RULES) {
            if (ruleApplies(rule, IS_SPRINKLERED, FLOOR_LEVEL, IS_OUTDOOR, APPROVED_STORAGE, PHYSICAL_STATE)) {
                const ruleState = rule[PHYSICAL_STATE];
                if (!ruleState) continue;

                if (ruleState.isNotApplicable || ruleState.operator === 'OVERRIDE' || ruleState.isNoLimit) {
                    // Replace entire calculation
                    factors = [{
                        label: rule.conditions.map(c => c.criteria).join(' & '),
                        value: ruleState.value,
                        operator: ruleState.operator,
                        isNoLimit: ruleState.isNoLimit,
                        isNotApplicable: ruleState.isNotApplicable
                    }];
                    break;
                } else if (ruleState.operator === 'EQUALS') {
                    factors[0] = {
                        label: rule.conditions.map(c => c.criteria).join(' & '),
                        value: ruleState.value,
                        operator: 'EQUALS'
                    };
                } else if (ruleState.operator === 'MULTIPLY') {
                    factors.push({
                        label: rule.conditions.map(c => c.criteria).join(' & '),
                        value: ruleState.value,
                        operator: 'MULTIPLY'
                    });
                }
            }
        }
    }

    // Apply floor level factors (if not outdoor)
    if (!IS_OUTDOOR && FLOOR_LEVEL !== null) {
        // Simplified floor rules - would need to be expanded based on occupancy rules
        if (FLOOR_LEVEL < 0) {
            factors.push({ label: 'Floor Level', value: 0.75, operator: 'MULTIPLY' });
        } else if (FLOOR_LEVEL > 2) {
            factors.push({ label: 'Floor Level', value: 0.5, operator: 'MULTIPLY' });
        }
    }

    return factors;
$$;

-- JavaScript UDF to reduce factors to final limit
CREATE OR REPLACE FUNCTION calculate_limit_from_factors(factors VARIANT)
RETURNS VARIANT
LANGUAGE JAVASCRIPT
AS $$
    if (!FACTORS || FACTORS.length === 0) {
        return { value: null, displayLimit: 'N/A' };
    }

    let result = {
        value: FACTORS[0].value,
        isNoLimit: FACTORS[0].isNoLimit || false,
        isNotApplicable: FACTORS[0].isNotApplicable || false
    };

    for (let i = 1; i < FACTORS.length; i++) {
        const factor = FACTORS[i];
        if (factor.operator === 'MULTIPLY' && result.value !== null) {
            result.value = result.value * factor.value;
        }
    }

    // Format display
    if (result.isNoLimit) {
        result.displayLimit = 'NL';
    } else if (result.isNotApplicable) {
        result.displayLimit = 'N/A';
    } else if (result.value !== null) {
        result.displayLimit = result.value.toFixed(2);
    } else {
        result.displayLimit = 'N/A';
    }

    return result;
$$;

-- JavaScript UDF for compliance status
CREATE OR REPLACE FUNCTION get_compliance_status(limit_value FLOAT, actual_value FLOAT)
RETURNS STRING
LANGUAGE JAVASCRIPT
AS $$
    if (LIMIT_VALUE === null) return 'COMPLIANT';
    if (LIMIT_VALUE === 0 && ACTUAL_VALUE === 0) return 'COMPLIANT';
    if (ACTUAL_VALUE >= LIMIT_VALUE) return 'OVER_THRESHOLD';
    if (ACTUAL_VALUE >= LIMIT_VALUE * 0.8) return 'NEAR_THRESHOLD';
    return 'COMPLIANT';
$$;

-- Main report view using UDFs
CREATE OR REPLACE VIEW maq_report AS
WITH control_area_base AS (
    SELECT
        ca.id AS control_area_id,
        ca.occupancy,
        ca.is_outdoor,
        ca.floor_above_ground_plane,
        ca.approved_storage,
        ca.exemption_is_exempt,
        CASE
            WHEN ca.fire_suppression_override_enabled THEN ca.fire_suppression_override_value
            WHEN b.fire_suppression_type = 'FULL' THEN TRUE
            WHEN b.fire_suppression_type = 'BASEMENT_ONLY' AND ca.floor_above_ground_plane < 0 THEN TRUE
            ELSE FALSE
        END AS is_sprinklered
    FROM control_areas ca
    JOIN buildings b ON ca.building_id = b.id
),
hazard_class_factors AS (
    SELECT
        cab.control_area_id,
        hc.name AS hazard_class_name,
        hc.is_health_hazard,
        -- Calculate factors for each state
        calculate_maq_factors(
            hc.baseline_solid, hc.rules, cab.is_sprinklered,
            cab.floor_above_ground_plane, cab.is_outdoor,
            cab.approved_storage, 'solid'
        ) AS factors_solid,
        calculate_maq_factors(
            hc.baseline_liquid, hc.rules, cab.is_sprinklered,
            cab.floor_above_ground_plane, cab.is_outdoor,
            cab.approved_storage, 'liquid'
        ) AS factors_liquid,
        calculate_maq_factors(
            hc.baseline_gas, hc.rules, cab.is_sprinklered,
            cab.floor_above_ground_plane, cab.is_outdoor,
            cab.approved_storage, 'gas'
        ) AS factors_gas,
        hc.units_solid,
        hc.units_liquid,
        hc.units_gas
    FROM control_area_base cab
    CROSS JOIN hazard_classes hc
    WHERE hc.occupancy = cab.occupancy
      AND NOT cab.exemption_is_exempt
)
SELECT
    hcf.control_area_id,
    hcf.hazard_class_name,
    hcf.is_health_hazard,
    -- Solid
    hcf.factors_solid AS maq_factors_solid,
    calculate_limit_from_factors(hcf.factors_solid) AS limit_solid,
    hcf.units_solid,
    -- Liquid
    hcf.factors_liquid AS maq_factors_liquid,
    calculate_limit_from_factors(hcf.factors_liquid) AS limit_liquid,
    hcf.units_liquid,
    -- Gas
    hcf.factors_gas AS maq_factors_gas,
    calculate_limit_from_factors(hcf.factors_gas) AS limit_gas,
    hcf.units_gas
FROM hazard_class_factors hcf;
```

#### Pros
- JavaScript UDFs can mirror TypeScript logic closely
- Easier to port `maq-compliance.helper.ts` functions
- Cleaner SQL layer
- Full control over rule evaluation logic

#### Cons
- JavaScript UDFs have performance overhead
- Debugging is harder (no breakpoints)
- Logic split between SQL and JS
- Need to handle VARIANT types carefully

#### Verdict
**Good middle ground - manageable complexity, reasonable performance.** Recommended for most cases.

---

### Option 3: Snowpark Python

**Approach**: Full calculation logic in Python via Snowpark

#### Implementation Example

```python
# maq_calculations.py - deployed as Snowflake stored procedure/UDFs
from snowflake.snowpark import Session
from snowflake.snowpark.functions import udf, col, lit
from snowflake.snowpark.types import VariantType, FloatType, StringType, BooleanType
import json

# Constants
GRAM_TO_POUND = 0.0022046226
LITER_TO_GALLON = 0.2641720524
LITER_TO_CUBIC_FEET = 0.0353146667

def is_control_area_sprinklered(fire_suppression_type: str, floor_level: int) -> bool:
    """Determine if control area is sprinklered based on building type and floor."""
    if fire_suppression_type == 'FULL':
        return True
    if fire_suppression_type == 'BASEMENT_ONLY':
        return floor_level is not None and floor_level < 0
    return False

def condition_applies(
    condition: dict,
    is_sprinklered: bool,
    floor_level: int,
    is_outdoor: bool,
    approved_storage: list,
    hazard_class_name: str,
    physical_state: str
) -> bool:
    """Evaluate if a single rule condition applies."""
    criteria = condition.get('criteria')
    value = condition.get('value') in [True, 'true']

    if criteria == 'SPRINKLER':
        return value == is_sprinklered
    elif criteria == 'SPRINKLER_BASEMENT_ONLY':
        return value == (fire_suppression_type == 'BASEMENT_ONLY')
    elif criteria == 'OUTDOOR':
        return value == is_outdoor
    elif criteria == 'BASEMENT':
        return value == (floor_level is not None and floor_level < 0)
    elif criteria == 'APPROVED_STORAGE':
        has_approved = any(
            s.get('hazardClass') == hazard_class_name and
            physical_state in (s.get('physicalStates') or [])
            for s in (approved_storage or [])
        )
        return value == has_approved
    return False

def rule_applies(
    rule: dict,
    is_sprinklered: bool,
    floor_level: int,
    is_outdoor: bool,
    approved_storage: list,
    hazard_class_name: str,
    physical_state: str
) -> bool:
    """Evaluate if all conditions in a rule apply."""
    conditions = rule.get('conditions', [])
    return all(
        condition_applies(c, is_sprinklered, floor_level, is_outdoor,
                         approved_storage, hazard_class_name, physical_state)
        for c in conditions
    )

def get_floor_maq_factors(occupancy_rules: list, floor_level: int) -> list:
    """Calculate floor level MAQ factors based on occupancy rules."""
    factors = []

    if not occupancy_rules or floor_level is None:
        return factors

    # Get floors with 'is equal to' operator for exclusion
    equal_floors = set()
    for rule in occupancy_rules:
        if rule.get('operator') == 'is equal to':
            equal_floors.update(rule.get('appliedToFloors', []))

    for rule in occupancy_rules:
        operator = rule.get('operator')
        applied_floors = rule.get('appliedToFloors', [])
        percentage = rule.get('percentage', 100)

        applies = False
        if operator == 'is equal to' and floor_level in applied_floors:
            applies = True
        elif operator == 'is greater than' and applied_floors and floor_level > applied_floors[0] and floor_level not in equal_floors:
            applies = True
        elif operator == 'is less than' and applied_floors and floor_level < applied_floors[0] and floor_level not in equal_floors:
            applies = True

        if applies:
            factors.append({
                'label': 'Floor Level',
                'value': percentage / 100,
                'operator': 'MULTIPLY'
            })

    return factors

def get_maq_factors(
    hazard_class: dict,
    occupancy_rules: list,
    physical_state: str,
    is_sprinklered: bool,
    approved_storage: list,
    is_outdoor: bool,
    floor_level: int
) -> list:
    """Calculate all MAQ factors for a hazard class and physical state."""
    state_data = hazard_class.get(physical_state, {})
    baseline = state_data.get('baseline')

    # Handle reportAsLiquid for gas
    if physical_state == 'gas' and state_data.get('reportAsLiquid'):
        liquid_data = hazard_class.get('liquid', {})
        baseline = liquid_data.get('baseline')
        factors = [{
            'label': 'Report as Liquid',
            'value': baseline,
            'operator': '',
            'isNoLimit': liquid_data.get('isNoLimit', False),
            'isNotApplicable': liquid_data.get('isNotApplicable', False)
        }]
    else:
        factors = [{
            'label': 'Baseline',
            'value': baseline,
            'operator': '',
            'isNoLimit': state_data.get('isNoLimit', False),
            'isNotApplicable': state_data.get('isNotApplicable', False)
        }]

    if baseline is None:
        return factors

    # Apply hazard class rules
    for rule in hazard_class.get('rules', []):
        if rule_applies(rule, is_sprinklered, floor_level, is_outdoor,
                       approved_storage, hazard_class.get('name'), physical_state):
            rule_state = rule.get(physical_state, {})

            if rule_state.get('isNotApplicable') or rule_state.get('operator') == 'OVERRIDE' or rule_state.get('isNoLimit'):
                label = ' & '.join(c.get('criteria', '') for c in rule.get('conditions', []))
                factors = [{
                    'label': label,
                    'value': rule_state.get('value'),
                    'operator': rule_state.get('operator', ''),
                    'isNoLimit': rule_state.get('isNoLimit', False),
                    'isNotApplicable': rule_state.get('isNotApplicable', False)
                }]
                break
            elif rule_state.get('operator') == 'EQUALS':
                label = ' & '.join(c.get('criteria', '') for c in rule.get('conditions', []))
                factors[0] = {
                    'label': label,
                    'value': rule_state.get('value'),
                    'operator': 'EQUALS'
                }
            elif rule_state.get('operator') == 'MULTIPLY':
                label = ' & '.join(c.get('criteria', '') for c in rule.get('conditions', []))
                factors.append({
                    'label': label,
                    'value': rule_state.get('value'),
                    'operator': 'MULTIPLY'
                })

    # Apply floor level factors (if not outdoor)
    if not is_outdoor:
        factors.extend(get_floor_maq_factors(occupancy_rules, floor_level))

    # Filter out factors with value === 1 (no effect)
    return [f for f in factors if f.get('value') != 1 or f == factors[0]]

def calculate_limit_from_factors(factors: list) -> dict:
    """Reduce factors to final limit value."""
    if not factors:
        return {'value': None, 'displayLimit': 'N/A', 'isNoLimit': False, 'isNotApplicable': True}

    result = {
        'value': factors[0].get('value'),
        'isNoLimit': factors[0].get('isNoLimit', False),
        'isNotApplicable': factors[0].get('isNotApplicable', False)
    }

    for factor in factors[1:]:
        if factor.get('operator') == 'MULTIPLY' and result['value'] is not None:
            result['value'] = result['value'] * factor.get('value', 1)

    # Format display
    if result['isNoLimit']:
        result['displayLimit'] = 'NL'
    elif result['isNotApplicable']:
        result['displayLimit'] = 'N/A'
    elif result['value'] is not None:
        result['displayLimit'] = f"{result['value']:.2f}"
    else:
        result['displayLimit'] = 'N/A'

    return result

def get_compliance_status(limit: float, actual: float) -> str:
    """Determine compliance status based on limit and actual values."""
    if limit is None:
        return 'COMPLIANT'
    if limit == 0 and actual == 0:
        return 'COMPLIANT'
    if actual >= limit:
        return 'OVER_THRESHOLD'
    if actual >= limit * 0.8:
        return 'NEAR_THRESHOLD'
    return 'COMPLIANT'

def convert_units(value: float, conversion: str) -> float:
    """Convert between units."""
    if value is None:
        return 0
    conversions = {
        'gram_to_pound': GRAM_TO_POUND,
        'liter_to_gallon': LITER_TO_GALLON,
        'liter_to_cubic_feet': LITER_TO_CUBIC_FEET
    }
    return value * conversions.get(conversion, 1)

# Register as Snowpark UDFs
def register_udfs(session: Session):
    """Register all UDFs with Snowflake session."""

    @udf(name='calculate_maq_factors_udf', is_permanent=True,
         stage_location='@udfs', replace=True)
    def calculate_maq_factors_udf(
        hazard_class: dict,
        occupancy_rules: list,
        physical_state: str,
        is_sprinklered: bool,
        approved_storage: list,
        is_outdoor: bool,
        floor_level: int
    ) -> list:
        return get_maq_factors(
            hazard_class, occupancy_rules, physical_state,
            is_sprinklered, approved_storage, is_outdoor, floor_level
        )

    @udf(name='calculate_limit_udf', is_permanent=True,
         stage_location='@udfs', replace=True)
    def calculate_limit_udf(factors: list) -> dict:
        return calculate_limit_from_factors(factors)

    @udf(name='get_compliance_status_udf', is_permanent=True,
         stage_location='@udfs', replace=True)
    def get_compliance_status_udf(limit: float, actual: float) -> str:
        return get_compliance_status(limit, actual)
```

#### Pros
- 1:1 port of TypeScript logic possible
- Full Python ecosystem (pandas, numpy if needed)
- Easier testing and validation (can test locally)
- Can version control the Python code
- Better IDE support for development

#### Cons
- Requires Snowpark setup and configuration
- Higher compute costs (Python runtime)
- More complex deployment pipeline
- Cold start latency

#### Verdict
**Best for exact parity with current system.** Recommended if accuracy is critical and you have Snowpark expertise.

---

### Option 4: dbt + Jinja

**Approach**: Use dbt for transformation pipeline with Jinja macros for complex logic

#### Implementation Example

```sql
-- models/staging/stg_control_areas.sql
SELECT
    id,
    name,
    building_id,
    occupancy,
    is_outdoor,
    floor_above_ground_plane,
    approved_storage,
    exemption:isExempt::BOOLEAN AS exemption_is_exempt,
    exemption:reason::STRING AS exemption_reason,
    fire_suppression_override:override::BOOLEAN AS fire_suppression_override_enabled,
    fire_suppression_override:value::BOOLEAN AS fire_suppression_override_value
FROM {{ source('raw', 'control_areas') }}
```

```sql
-- models/intermediate/int_control_area_base.sql
WITH control_areas AS (
    SELECT * FROM {{ ref('stg_control_areas') }}
),

buildings AS (
    SELECT * FROM {{ ref('stg_buildings') }}
)

SELECT
    ca.id AS control_area_id,
    ca.name AS control_area_name,
    ca.building_id,
    ca.occupancy,
    ca.is_outdoor,
    ca.floor_above_ground_plane,
    ca.approved_storage,
    ca.exemption_is_exempt,
    b.fire_suppression_type,
    -- Determine effective sprinkler coverage
    CASE
        WHEN ca.fire_suppression_override_enabled
        THEN ca.fire_suppression_override_value
        WHEN b.fire_suppression_type = 'FULL' THEN TRUE
        WHEN b.fire_suppression_type = 'BASEMENT_ONLY'
             AND ca.floor_above_ground_plane < 0 THEN TRUE
        ELSE FALSE
    END AS is_sprinklered
FROM control_areas ca
JOIN buildings b ON ca.building_id = b.id
```

```jinja
-- macros/maq_calculations.sql

{% macro sprinkler_factor(is_sprinklered) %}
    CASE WHEN {{ is_sprinklered }} THEN 2 ELSE 1 END
{% endmacro %}

{% macro floor_level_factor(floor_level, is_outdoor) %}
    CASE
        WHEN {{ is_outdoor }} THEN 1
        WHEN {{ floor_level }} < 0 THEN 0.75
        WHEN {{ floor_level }} > 2 THEN 0.5
        ELSE 1
    END
{% endmacro %}

{% macro calculate_limit(baseline, is_sprinklered, floor_level, is_outdoor) %}
    {{ baseline }} *
    {{ sprinkler_factor(is_sprinklered) }} *
    {{ floor_level_factor(floor_level, is_outdoor) }}
{% endmacro %}

{% macro compliance_status(limit, actual) %}
    CASE
        WHEN {{ limit }} IS NULL THEN 'COMPLIANT'
        WHEN {{ actual }} >= {{ limit }} THEN 'OVER_THRESHOLD'
        WHEN {{ actual }} >= {{ limit }} * 0.8 THEN 'NEAR_THRESHOLD'
        ELSE 'COMPLIANT'
    END
{% endmacro %}

{% macro convert_gram_to_pound(value) %}
    COALESCE({{ value }}, 0) * 0.0022046226
{% endmacro %}

{% macro convert_liter_to_gallon(value) %}
    COALESCE({{ value }}, 0) * 0.2641720524
{% endmacro %}
```

```sql
-- models/marts/maq_report.sql
{{ config(materialized='table') }}

WITH base AS (
    SELECT * FROM {{ ref('int_control_area_base') }}
),

hazard_classes AS (
    SELECT * FROM {{ ref('stg_hazard_classes') }}
),

actuals AS (
    SELECT * FROM {{ ref('int_container_actuals') }}
),

limits AS (
    SELECT
        b.control_area_id,
        b.building_id,
        hc.name AS hazard_class_name,
        hc.is_health_hazard,
        -- Solid limit
        {{ calculate_limit(
            'hc.baseline_solid',
            'b.is_sprinklered',
            'b.floor_above_ground_plane',
            'b.is_outdoor'
        ) }} AS limit_solid,
        hc.units_solid,
        -- Liquid limit
        {{ calculate_limit(
            'hc.baseline_liquid',
            'b.is_sprinklered',
            'b.floor_above_ground_plane',
            'b.is_outdoor'
        ) }} AS limit_liquid,
        hc.units_liquid,
        -- Gas limit
        {{ calculate_limit(
            'hc.baseline_gas',
            'b.is_sprinklered',
            'b.floor_above_ground_plane',
            'b.is_outdoor'
        ) }} AS limit_gas,
        hc.units_gas
    FROM base b
    CROSS JOIN hazard_classes hc
    WHERE hc.occupancy = b.occupancy
      AND NOT b.exemption_is_exempt
)

SELECT
    l.control_area_id,
    l.building_id,
    l.hazard_class_name,
    l.is_health_hazard,
    -- Solid
    l.limit_solid,
    {{ convert_gram_to_pound('a_solid.total_grams') }} AS actual_solid,
    l.units_solid,
    {{ compliance_status('l.limit_solid', convert_gram_to_pound('a_solid.total_grams')) }} AS status_solid,
    -- Liquid
    l.limit_liquid,
    {{ convert_liter_to_gallon('a_liquid.total_liters') }} AS actual_liquid,
    l.units_liquid,
    {{ compliance_status('l.limit_liquid', convert_liter_to_gallon('a_liquid.total_liters')) }} AS status_liquid,
    -- Gas
    l.limit_gas,
    {{ convert_liter_to_gallon('a_gas.total_liters') }} AS actual_gas,  -- Using cubic feet conversion
    l.units_gas,
    {{ compliance_status('l.limit_gas', convert_liter_to_gallon('a_gas.total_liters')) }} AS status_gas
FROM limits l
LEFT JOIN actuals a_solid
    ON l.control_area_id = a_solid.control_area_id
    AND l.hazard_class_name = a_solid.hazard_class_name
    AND a_solid.physical_state = 'solid'
LEFT JOIN actuals a_liquid
    ON l.control_area_id = a_liquid.control_area_id
    AND l.hazard_class_name = a_liquid.hazard_class_name
    AND a_liquid.physical_state = 'liquid'
LEFT JOIN actuals a_gas
    ON l.control_area_id = a_gas.control_area_id
    AND l.hazard_class_name = a_gas.hazard_class_name
    AND a_gas.physical_state = 'gas'
```

```yaml
# dbt_project.yml
name: 'maq_snowflake'
version: '1.0.0'

models:
  maq_snowflake:
    staging:
      +materialized: view
    intermediate:
      +materialized: view
    marts:
      +materialized: table
```

#### Pros
- Industry-standard data transformation tool
- Version controlled, testable with dbt tests
- Good lineage tracking and documentation
- Incremental builds possible
- Great for pipeline orchestration

#### Cons
- Complex rules still need UDFs or become unwieldy Jinja
- Another tool in the stack to learn/maintain
- Jinja macros can become hard to debug
- May need to combine with Option 2 for complex logic

#### Verdict
**Good for pipeline orchestration.** Best combined with JavaScript UDFs (Option 2) for complex rule evaluation.

---

### Option 5: Materialized Views + Scheduled Refresh

**Approach**: Pre-compute reports on a schedule, store results in cache table

```
RAW DATA → SNOWFLAKE TASKS → MATERIALIZED REPORT TABLE
                ↓
    (run every 15 min or on-demand)
```

#### Implementation Example

```sql
-- Create cache table
CREATE OR REPLACE TABLE maq_report_cache (
    building_id VARCHAR,
    control_area_id VARCHAR,
    control_area_name VARCHAR,
    hazard_class_name VARCHAR,
    is_health_hazard BOOLEAN,
    -- Solid
    limit_solid FLOAT,
    actual_solid FLOAT,
    display_limit_solid VARCHAR,
    display_actual_solid VARCHAR,
    units_solid VARCHAR,
    status_solid VARCHAR,
    maq_factors_solid VARIANT,
    -- Liquid
    limit_liquid FLOAT,
    actual_liquid FLOAT,
    display_limit_liquid VARCHAR,
    display_actual_liquid VARCHAR,
    units_liquid VARCHAR,
    status_liquid VARCHAR,
    maq_factors_liquid VARIANT,
    -- Gas
    limit_gas FLOAT,
    actual_gas FLOAT,
    display_limit_gas VARCHAR,
    display_actual_gas VARCHAR,
    units_gas VARCHAR,
    status_gas VARCHAR,
    maq_factors_gas VARIANT,
    -- Metadata
    computed_at TIMESTAMP_NTZ,
    PRIMARY KEY (control_area_id, hazard_class_name)
);

-- Stored procedure to refresh report
CREATE OR REPLACE PROCEDURE refresh_maq_report()
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
    -- Clear existing cache
    TRUNCATE TABLE maq_report_cache;

    -- Insert fresh calculations
    INSERT INTO maq_report_cache
    SELECT
        cab.building_id,
        cab.control_area_id,
        cab.control_area_name,
        hc.name AS hazard_class_name,
        hc.is_health_hazard,
        -- Solid calculations using UDFs
        calculate_limit_from_factors(
            calculate_maq_factors(hc.data, occ.rules, 'solid',
                cab.is_sprinklered, cab.approved_storage,
                cab.is_outdoor, cab.floor_above_ground_plane)
        ):value::FLOAT AS limit_solid,
        COALESCE(act_solid.total_grams * 0.0022046226, 0) AS actual_solid,
        calculate_limit_from_factors(
            calculate_maq_factors(hc.data, occ.rules, 'solid',
                cab.is_sprinklered, cab.approved_storage,
                cab.is_outdoor, cab.floor_above_ground_plane)
        ):displayLimit::VARCHAR AS display_limit_solid,
        TO_VARCHAR(COALESCE(act_solid.total_grams * 0.0022046226, 0), '999990.00') AS display_actual_solid,
        hc.units_solid,
        get_compliance_status(
            calculate_limit_from_factors(
                calculate_maq_factors(hc.data, occ.rules, 'solid',
                    cab.is_sprinklered, cab.approved_storage,
                    cab.is_outdoor, cab.floor_above_ground_plane)
            ):value::FLOAT,
            COALESCE(act_solid.total_grams * 0.0022046226, 0)
        ) AS status_solid,
        calculate_maq_factors(hc.data, occ.rules, 'solid',
            cab.is_sprinklered, cab.approved_storage,
            cab.is_outdoor, cab.floor_above_ground_plane) AS maq_factors_solid,
        -- Similar for liquid and gas...
        NULL, NULL, NULL, NULL, NULL, NULL, NULL,
        NULL, NULL, NULL, NULL, NULL, NULL, NULL,
        CURRENT_TIMESTAMP()
    FROM control_area_base cab
    JOIN fire_codes fc ON cab.fire_code_id = fc.id
    JOIN LATERAL FLATTEN(input => fc.occupancies) occ
    JOIN LATERAL FLATTEN(input => occ.value:hazardClasses) hc
    LEFT JOIN container_actuals act_solid
        ON cab.control_area_id = act_solid.control_area_id
        AND hc.value:name::VARCHAR = act_solid.hazard_class_name
        AND act_solid.physical_state = 'solid'
    WHERE occ.value:name::VARCHAR = cab.occupancy
      AND NOT cab.exemption_is_exempt;

    RETURN 'Success: ' || (SELECT COUNT(*) FROM maq_report_cache) || ' rows refreshed';
END;
$$;

-- Create scheduled task
CREATE OR REPLACE TASK refresh_maq_task
    WAREHOUSE = COMPUTE_WH
    SCHEDULE = 'USING CRON 0/15 * * * * UTC'  -- Every 15 minutes
AS
    CALL refresh_maq_report();

-- Enable the task
ALTER TASK refresh_maq_task RESUME;

-- Manual refresh (for on-demand)
CREATE OR REPLACE PROCEDURE refresh_maq_report_now()
RETURNS STRING
LANGUAGE SQL
AS
$$
BEGIN
    CALL refresh_maq_report();
    RETURN 'Manual refresh completed at ' || CURRENT_TIMESTAMP();
END;
$$;

-- Query the cached report
CREATE OR REPLACE VIEW maq_report AS
SELECT * FROM maq_report_cache;
```

#### Pros
- Fast reads (pre-computed data)
- Complex logic hidden in stored procedure
- Predictable resource usage (scheduled compute)
- Easy to add manual refresh triggers
- Good for dashboards and reporting

#### Cons
- Data staleness (max 15 min by default)
- Storage costs for cache table
- Need to manage task scheduling
- Full refresh each time (could optimize with incremental)

#### Verdict
**Good for dashboards and reports where real-time isn't critical.** Can be combined with real-time views for hybrid approach.

---

## Recommended Architecture

Based on the analysis, a **layered architecture** provides the best balance of maintainability, performance, and accuracy:

```
┌──────────────────────────────────────────────────────────────┐
│                      LAYER 4: REPORTING                       │
│         Views for BI tools, API queries, exports              │
│         - maq_report (main view)                              │
│         - maq_building_summary (aggregated by building)       │
│         - maq_compliance_dashboard (status counts)            │
└──────────────────────────────────────────────────────────────┘
                              ↑
┌──────────────────────────────────────────────────────────────┐
│                   LAYER 3: MATERIALIZED CACHE                 │
│    Pre-computed reports (optional, for performance)           │
│    - maq_report_cache (refreshed every 15 min)                │
│    - Useful for high-traffic dashboards                       │
└──────────────────────────────────────────────────────────────┘
                              ↑
┌──────────────────────────────────────────────────────────────┐
│                 LAYER 2: CALCULATION LOGIC                    │
│   JavaScript UDFs or Snowpark Python for rule evaluation      │
│   - calculate_maq_factors()                                   │
│   - calculate_limit_from_factors()                            │
│   - get_compliance_status()                                   │
│   - convert_units()                                           │
└──────────────────────────────────────────────────────────────┘
                              ↑
┌──────────────────────────────────────────────────────────────┐
│                  LAYER 1: BASE JOINS/STAGING                  │
│   SQL Views for entity relationships                          │
│   - control_area_base (joined with building, sprinkler calc)  │
│   - container_actuals (aggregated by control area)            │
│   - fire_code_hazard_classes (flattened from VARIANT)         │
└──────────────────────────────────────────────────────────────┘
                              ↑
┌──────────────────────────────────────────────────────────────┐
│                     LAYER 0: RAW DATA                         │
│   Replicated from MongoDB, Elasticsearch, External Service    │
│   - control_areas                                             │
│   - fire_codes                                                │
│   - containers                                                │
│   - buildings, floors, rooms                                  │
└──────────────────────────────────────────────────────────────┘
```

### Recommended Option Combination

| Layer | Recommended Approach |
|-------|---------------------|
| Layer 0 (Raw) | Snowpipe or scheduled ETL |
| Layer 1 (Staging) | SQL Views (Option 1) |
| Layer 2 (Calculation) | JavaScript UDFs (Option 2) or Snowpark Python (Option 3) |
| Layer 3 (Cache) | Scheduled Tasks (Option 5) - optional |
| Layer 4 (Reporting) | SQL Views over cache or direct calculation |

---

## Critical Considerations

### 1. Data Sync Strategy

How will you keep Snowflake in sync with source systems?

| Strategy | Latency | Complexity | Best For |
|----------|---------|------------|----------|
| **Snowpipe** (Kafka/S3) | Near real-time | Medium | Containers (high volume) |
| **Scheduled ETL** | Minutes-hours | Low | Fire codes, buildings (low change) |
| **CDC (Debezium)** | Real-time | High | All sources if real-time critical |

### 2. Hazard Class Band Mapping

The Elasticsearch query uses complex band filtering that needs replication:

```javascript
// Current Elasticsearch filter
filters[hazardClassName] = {
    bool: {
        must: mapping.must.map(m => termQuery(m.id)),
        should: mapping.should.map(m => termQuery(m.id)),
        must_not: mapping.mustNot.map(m => termQuery(m.id))
    }
}
```

**Snowflake equivalent:**
```sql
-- Container to hazard class mapping view
CREATE VIEW container_hazard_class_mapping AS
SELECT
    c.id AS container_id,
    c.room_id,
    hcm.hazard_class_name,
    c.normalized_size_gram,
    c.normalized_size_liter,
    c.physical_state
FROM containers c
JOIN hazard_class_mappings hcm ON (
    -- Must match ALL 'must' bands
    ARRAY_SIZE(ARRAY_INTERSECTION(c.band_ids, hcm.must_band_ids)) = ARRAY_SIZE(hcm.must_band_ids)
    -- Must match AT LEAST ONE 'should' band (if any exist)
    AND (ARRAY_SIZE(hcm.should_band_ids) = 0
         OR ARRAY_SIZE(ARRAY_INTERSECTION(c.band_ids, hcm.should_band_ids)) > 0)
    -- Must NOT match ANY 'must_not' bands
    AND ARRAY_SIZE(ARRAY_INTERSECTION(c.band_ids, hcm.must_not_band_ids)) = 0
)
WHERE c.active = TRUE
  AND (c.exemption IS NULL OR c.exemption = '');
```

### 3. Special Cases to Handle

| Case | Current Behavior | Snowflake Implementation |
|------|------------------|--------------------------|
| `reportAsLiquid` | Gas reported under liquid | Check flag in UDF, redirect aggregation |
| Exempted control areas | All limits = NL | Short-circuit calculation, return NL |
| OVERRIDE rules | Replace entire calculation | Break loop in UDF, return single factor |
| Floor level rules | Complex operator matching | Implement in UDF with operator logic |
| NL/N/A states | Special display values | Handle in display formatting |

### 4. Validation Strategy

Run both systems in parallel and compare outputs:

```sql
-- Validation query
CREATE OR REPLACE VIEW maq_validation AS
SELECT
    sf.control_area_id,
    sf.hazard_class_name,
    sf.limit_solid AS snowflake_limit_solid,
    curr.limit_solid AS current_limit_solid,
    ABS(sf.limit_solid - curr.limit_solid) AS diff_solid,
    sf.actual_solid AS snowflake_actual_solid,
    curr.actual_solid AS current_actual_solid,
    ABS(sf.actual_solid - curr.actual_solid) AS diff_actual_solid,
    sf.status_solid AS snowflake_status,
    curr.status_solid AS current_status,
    CASE WHEN sf.status_solid != curr.status_solid THEN 'MISMATCH' ELSE 'OK' END AS status_match
FROM snowflake_maq_report sf
FULL OUTER JOIN current_maq_report curr
    ON sf.control_area_id = curr.control_area_id
    AND sf.hazard_class_name = curr.hazard_class_name;

-- Find discrepancies
SELECT * FROM maq_validation
WHERE diff_solid > 0.01
   OR diff_actual_solid > 0.01
   OR status_match = 'MISMATCH';
```

---

## Implementation Roadmap

### Phase 1: Foundation (Week 1-2)
- [ ] Define Snowflake table schemas for all source data
- [ ] Set up data replication pipeline (Snowpipe or ETL)
- [ ] Create Layer 0 raw tables
- [ ] Create Layer 1 staging views

### Phase 2: Core Logic (Week 3-4)
- [ ] Port `getMaqFactors()` to JavaScript UDF
- [ ] Port `calculateLimitByState()` to JavaScript UDF
- [ ] Port `getComplianceStatus()` to JavaScript UDF
- [ ] Create unit conversion functions
- [ ] Test UDFs with sample data

### Phase 3: Integration (Week 5-6)
- [ ] Create hazard class band mapping view
- [ ] Create container actuals aggregation view
- [ ] Create main MAQ report view
- [ ] Handle all special cases (reportAsLiquid, exemptions, etc.)

### Phase 4: Validation (Week 7-8)
- [ ] Set up parallel validation pipeline
- [ ] Compare Snowflake vs current system outputs
- [ ] Identify and fix discrepancies
- [ ] Document any intentional differences

### Phase 5: Optimization (Week 9-10)
- [ ] Add materialized cache if needed for performance
- [ ] Set up scheduled refresh tasks
- [ ] Create reporting views for BI tools
- [ ] Performance tuning

### Phase 6: Cutover (Week 11-12)
- [ ] Final validation
- [ ] Documentation
- [ ] Training
- [ ] Cutover to Snowflake-based reporting

---

## Appendix: File References

| Current File | Purpose | Snowflake Equivalent |
|--------------|---------|---------------------|
| `maq-compliance.helper.ts` | All MAQ calculations | JavaScript UDFs or Snowpark Python |
| `MAQReportService.ts` | Server-side orchestration | SQL Views + Stored Procedures |
| `FireCodeService.ts` | Fire code data access | `fire_codes` table |
| `ControlAreaService.ts` | Control area data access | `control_areas` table |
| Elasticsearch queries | Container aggregation | SQL aggregation views |

---

*Document generated: December 2024*

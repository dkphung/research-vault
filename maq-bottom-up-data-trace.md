# MAQ Report Data Lineage: Bottom-Up Trace

This document traces every data point displayed in the MAQ Report from its final rendered value back to its original source. Each section covers a specific data point, showing the complete derivation path.

---

## Table of Contents

1. [Building-Level Data](#1-building-level-data)
2. [Control Area Metadata](#2-control-area-metadata)
3. [Hazard Class Data](#3-hazard-class-data)
4. [The Core MAQ Calculation](#4-the-core-maq-calculation)
5. [Actual Quantity Values](#5-actual-quantity-values)
6. [MAQ Limit Calculation](#6-maq-limit-calculation)
7. [Compliance Status](#7-compliance-status)
8. [Data Source Summary](#8-data-source-summary)

---

## 1. Building-Level Data

### 1.1 Building Name (`buildingMaqReport.name`)

**Rendered in:** `Building.tsx`, `BuildingDetail.tsx`

**Derivation Chain:**
```
UI Display
    ↓
GraphQL Response: buildingMaqReport.name
    ↓
MAQReportService.getMAQReportByBuildingId():224-237
    building.name
    ↓
RelationshipGateway.fetchById()
    ↓
SOURCE: External Building Service (relationship service)
```

**Source:** External service via `AxiosRelationshipGateway.fetchById()` - building record from campus facilities database.

---

### 1.2 Fire Code Name (`buildingMaqReport.fireCodeName`)

**Rendered in:** Building header

**Derivation Chain:**
```
UI Display
    ↓
GraphQL Response: buildingMaqReport.fireCodeName
    ↓
MAQReportService.getMAQReportByBuildingId():229
    fireCode.name
    ↓
FireCodeService.getById(building.fireCodeId):86-88
    ↓
SOURCE: MongoDB Collection 'fireCode', field 'name'
```

---

### 1.3 Fire Suppression Type (`buildingMaqReport.fireSuppressionType`)

**Rendered in:** Building header, passed to MaqTable

**Derivation Chain:**
```
UI Display
    ↓
GraphQL Response: buildingMaqReport.fireSuppressionType
    ↓
MAQReportService.getMAQReportByBuildingId():231
    building.fireSuppressionType
    ↓
RelationshipGateway.fetchById()
    ↓
SOURCE: External Building Service - building.fireSuppressionType
    Enum values: 'NONE' | 'BASEMENT_ONLY' | 'FULL'
```

---

### 1.4 Fire Suppression Coverage (`buildingMaqReport.fireSuppressionCoverage`)

**Rendered in:** Building header

**Derivation Chain:**
```
UI Display
    ↓
GraphQL Response: buildingMaqReport.fireSuppressionCoverage
    ↓
MAQReportService.getMAQReportByBuildingId():222
    this.calculateFireSuppressionCoverage(building.fireSuppressionType)
    ↓
MAQReportService.calculateFireSuppressionCoverage():93-95
    returns: fireSuppressionType === FireSuppressionType.Full
    ↓
CALCULATION: Boolean derived from fireSuppressionType
    FULL → true
    BASEMENT_ONLY → false
    NONE → false
```

---

### 1.5 Overall Building Status (`buildingMaqReport.status`)

**Rendered in:** Building header, campus building list

**Derivation Chain:**
```
UI Display
    ↓
GraphQL Response: buildingMaqReport.status
    ↓
MAQReportService.getMAQReportByBuildingId():236
    getOverallComplianceStatus(controlAreasWHazardClasses)
    ↓
maq-compliance.helper.ts:21-35 getOverallComplianceStatus()
    if (areas.length === 0) → INCOMPLETE
    if (any area.status === OVER_THRESHOLD) → OVER_THRESHOLD
    if (any area.status === NEAR_THRESHOLD) → NEAR_THRESHOLD
    else → COMPLIANT
    ↓
CALCULATION: Derived from control area statuses (worst status wins)
```

---

## 2. Control Area Metadata

### 2.1 Control Area Name (`controlArea.name`)

**Rendered in:** `ControlAreaDetail.tsx`, MaqTable

**Derivation Chain:**
```
UI Display
    ↓
GraphQL Response: buildingMaqReport.controlAreas[].name
    ↓
MAQReportService.calculateMAQByBuilding():335-336
    controlArea.name (from controlAreasWoutHazardClasses)
    ↓
ControlAreaService.getByBuildingId():36-41
    ↓
SOURCE: MongoDB Collection 'controlArea', field 'name'
```

---

### 2.2 Control Area Occupancy (`controlArea.occupancy`)

**Rendered in:** `ControlAreaDetail.tsx` line 52

**Derivation Chain:**
```
UI Display: "Occupancy: {controlArea.occupancy}"
    ↓
GraphQL Response: controlArea.occupancy
    ↓
SOURCE: MongoDB Collection 'controlArea', field 'occupancy'
    Values: 'A', 'B', 'H', etc. (fire code occupancy classification)
```

**Critical Usage:** The occupancy links the control area to specific fire code rules:
- Used in `getControlAreaLimits()`:788 to find matching fire code occupancy
- Determines which hazard classes apply
- Determines which floor-level rules apply

---

### 2.3 Floor Above Ground Plane (`controlArea.floorAboveGroundPlane`)

**Rendered in:** `ControlAreaDetail.tsx` line 54

**Derivation Chain:**
```
UI Display: "Floor Level: {floor.groundPlane}"
    ↓
ControlAreaDetail.tsx:29-33
    floors.reduce((acc, cur) =>
        Math.abs(stringToInt(cur.groundPlane)) > Math.abs(stringToInt(acc.groundPlane))
            ? cur : acc
    )
    ↓
Building floors filtered by rooms with controlAreaId
    ↓
SOURCE: External Building Service - floor.groundPlane
    Integer value: negative = basement, 0 = ground, positive = above ground
```

**Critical Usage:** Floor level affects MAQ limits via floor-level reduction factors:
- `getFloorMaqFactors()`:183-219 applies percentage reductions based on floor
- Basement floors may get different sprinkler treatment

---

### 2.4 Is Outdoor (`controlArea.isOutdoor`)

**Rendered in:** `ControlAreaDetail.tsx` line 53

**Derivation Chain:**
```
UI Display: "Outdoor: Yes/No"
    ↓
GraphQL Response: controlArea.isOutdoor
    ↓
SOURCE: MongoDB Collection 'controlArea', field 'isOutdoor'
    Boolean value
```

**Critical Usage:** Outdoor status affects MAQ calculations:
- `getMaqFactors()`:296,351 - Floor level factors NOT applied if outdoor
- `ruleApplies()`:156 - Outdoor condition in rule evaluation

---

### 2.5 Exemption Status (`controlArea.exemption`)

**Rendered in:** `ControlAreaDetail.tsx` lines 68-78

**Derivation Chain:**
```
UI Display:
    "Exemption Reason: {exemption.reason}"
    "Exemption Notes: {exemption.notes}"
    ↓
GraphQL Response: controlArea.exemption { isExempt, reason, notes }
    ↓
SOURCE: MongoDB Collection 'controlArea', field 'exemption'
    {
        isExempt: Boolean,
        reason: String | null,
        notes: String | null
    }
```

**Critical Usage:** If `isExempt === true`:
- `getControlAreaMaq()`:902 sets status to `EXEMPT`
- `getHazardClassLimits()`:735-737 returns `exemptLimits()` - all limits set to null/NL

---

### 2.6 Approved Storage (`controlArea.approvedStorage`)

**Rendered in:** `ControlAreaDetail.tsx` lines 74-78

**Derivation Chain:**
```
UI Display: displayApprovedStorage(approvedStorage, selectedOccupancy)
    ↓
GraphQL Response: controlArea.approvedStorage[]
    { hazardClass: String, physicalStates: [PhysicalStates] }
    ↓
SOURCE: MongoDB Collection 'controlArea', field 'approvedStorage'
    Array of { hazardClass, physicalStates }
```

**Critical Usage:** Approved storage can double MAQ limits:
- `conditionApplies()`:150-155 checks if hazard class + physical state matches
- When matched, multiplier rule (typically 2x) is applied

---

### 2.7 Fire Suppression Override (`controlArea.fireSuppressionOverride`)

**Rendered in:** `ControlAreaDetail.tsx` lines 55-59

**Derivation Chain:**
```
UI Display: "Fire Suppression Coverage: Yes/No" (only if override === true)
    ↓
GraphQL Response: controlArea.fireSuppressionOverride { override, value }
    ↓
SOURCE: MongoDB Collection 'controlArea', field 'fireSuppressionOverride'
    {
        override: Boolean,  // Whether to use override instead of building value
        value: Boolean      // The overridden value (true = sprinklered)
    }
```

**Critical Usage:**
- `MaqTable.tsx`:96-98 determines effective fire suppression:
  ```javascript
  const fireSuppressionCoverage = controlArea.fireSuppressionOverride?.override
    ? controlArea.fireSuppressionOverride.value
    : isControlAreaSprinklered(buildingFireSuppressionType, floorAboveGroundPlane);
  ```
- `getControlAreaLimits()`:803-811 uses override for fire suppression type determination

---

## 3. Hazard Class Data

### 3.1 Hazard Class Name (`hazardClass.name`)

**Rendered in:** MaqTable Material column

**Derivation Chain:**
```
UI Display: Material column in MaqTable
    ↓
MaqTable.tsx:127-129 accessor: 'name'
    row.original.name
    ↓
GraphQL Response: controlArea.hazardClasses[].name
    ↓
getControlAreaMaq():874-875
    hazardClassLimit.name (copied from fire code)
    ↓
getControlAreaLimits():813
    fireCodeHazardClass.name
    ↓
FireCode.occupancies[].hazardClasses[].name
    ↓
SOURCE: MongoDB Collection 'fireCode', occupancy.hazardClasses[].name
```

---

### 3.2 Is Health Hazard (`hazardClass.isHealthHazard`)

**Rendered in:** Determines which table (Physical vs Health Hazards)

**Derivation Chain:**
```
UI Display: Table header "Physical Hazards" vs "Health Hazards"
    ↓
ControlAreaDetail.tsx:39-40
    physicalHazardClasses = hazardClasses.filter(hc => !hc.isHealthHazard)
    healthHazardClasses = hazardClasses.filter(hc => hc.isHealthHazard)
    ↓
GraphQL Response: hazardClass.isHealthHazard
    ↓
getControlAreaMaq():876
    hazardClassLimit.isHealthHazard
    ↓
getHazardClassLimits():781
    fireCodeHazardClass.isHealthHazard
    ↓
SOURCE: MongoDB Collection 'fireCode', occupancy.hazardClasses[].isHealthHazard
    Boolean value
```

---

## 4. The Core MAQ Calculation

### Overview

The MAQ Table displays three key values per hazard class per physical state:
1. **Actual** - Current quantity in the control area
2. **MAQ** (Limit) - Maximum allowable quantity
3. **Units** - Unit of measurement (lbs, gal, ft³)

Each of these involves complex derivation chains. The following sections trace each in detail.

---

## 5. Actual Quantity Values

### 5.1 Display Value (`hazardClass.solid.displayValue`)

**Rendered in:** MaqTable.tsx:51 - "Actual" column

**Complete Derivation Chain:**

```
UI Display: value.displayValue || '0'
    ↓
MaqTable.tsx:45-52 Cell renderer
    ↓
GraphQL Response: hazardClass.solid.displayValue
    ↓
getControlAreaMaq():881 (for solid)
    sigFigDecimal(solidActual)
    ↓
misc.ts:14-25 sigFigDecimal()
    if (value === 0) → '0.00'
    if (value < 0.0001) → '< 0.0001'
    if (value < 0.01) → value.toFixed(4)
    else → value.toFixed(2)
    ↓
getControlAreaMaq():859
    solidActual = convertNormalizedAmount(actual?.solid?.actualWeight || 0, 'gramToPound')
    ↓
maq-compliance.helper.ts:830-852 convertNormalizedAmount()
    gramToPound = 0.0022046226
    value * gramToPound
    ↓
Elasticsearch Aggregation Result
    actual.solid.actualWeight (in grams)
    ↓
MAQReportService.fetchActuals():455-505
    Elasticsearch aggregation sum: 'normalizedSize.gram'
    ↓
SOURCE: Elasticsearch 'container' index
    Field: normalizedSize.gram (float)
```

### 5.2 Actual Weight in Grams (`hazardClass.solid.actualWeight`)

**Derivation Chain:**

```
GraphQL Response: hazardClass.solid.actualWeight
    ↓
getControlAreaMaq():879
    actual?.solid?.actualWeight || 0
    ↓
MAQReportService.fetchActuals():507-532
    familyBands[hazClass].container_state.buckets.forEach(state => {
        hazardClass[stateKey] = {
            actualWeight: state.containers_sum_gram.value,
            actualVolume: state.containers_sum_liter.value
        }
    })
    ↓
Elasticsearch Aggregation:478-500
    {
        aggs: {
            FamilyBands: {
                filters: { filters },  // Hazard class band mappings
                aggs: {
                    container_state: {
                        terms: { field: 'family.formAtNtp.keyword' },  // solid/liquid/gas
                        aggs: {
                            containers_sum_gram: {
                                sum: { field: 'normalizedSize.gram' }
                            },
                            containers_sum_liter: {
                                sum: { field: 'normalizedSize.liter' }
                            }
                        }
                    }
                }
            }
        }
    }
    ↓
Elasticsearch Query Filter:460-475
    {
        must: [
            { terms: { 'location.roomId': [room IDs in control area] } }
        ],
        must_not: [
            { script: "doc['exemption.keyword'].size() > 0 && ..." },  // Exclude exempted
            { term: { active: false } }  // Exclude inactive
        ]
    }
    ↓
SOURCE: Elasticsearch 'container' index
    - normalizedSize.gram: Pre-computed weight in grams
    - normalizedSize.liter: Pre-computed volume in liters
    - family.formAtNtp: Physical state at NTP (solid/liquid/gas)
    - family.bands._id: Chemical band IDs for hazard class matching
    - location.roomId: Room association
    - exemption: Exemption status (excluded from MAQ)
    - active: Container active status
```

### 5.3 Hazard Class Filter Mapping

**Derivation Chain for Band Matching:**

```
MAQReportService.buildHazardClassFilters():376-401
    fireCode.hazardClassMappings.find(mapping => mapping.name === hazardClass.name)
    ↓
    filters[hazardClassName] = {
        bool: {
            must: mapping.must.map(m => termQuery(m.id)),
            should: mapping.should.map(m => termQuery(m.id)),
            must_not: mapping.mustNot.map(m => termQuery(m.id))
        }
    }
    ↓
SOURCE: MongoDB 'fireCode' collection
    hazardClassMappings[]: {
        name: "Flammable Liquid: IA",
        must: [{ id: "band-uuid-1", displayName: "Flammable Liquid Cat 1" }],
        should: [{ id: "band-uuid-2", displayName: "..." }],
        mustNot: [{ id: "band-uuid-3", displayName: "..." }]
    }
```

### 5.4 Liquid Actual Value (Special Handling)

**Derivation Chain:**

```
getControlAreaMaq():860-872
    const liquidUnits = hazardClassLimit.liquid.units;
    switch (liquidUnits) {
        case 'gal':
            liquidActual = convertNormalizedAmount(actualVolume, 'literToGallon');
            // literToGallon = 0.2641720524
            break;
        case 'ft3':
            liquidActual = convertNormalizedAmount(actualVolume, 'literToCubicFeet');
            // literToCubicFeet = 0.0353146667
            break;
        default:
            liquidActual = convertNormalizedAmount(actualWeight, 'gramToPound');
            // gramToPound = 0.0022046226
            break;
    }
```

**Note:** Liquid units determined by fire code baseline definition.

### 5.5 Gas Actual Value

**Derivation Chain:**

```
getControlAreaMaq():873
    gasActual = convertNormalizedAmount(actual?.gas?.actualVolume || 0, 'literToCubicFeet')
    // literToCubicFeet = 0.0353146667
```

### 5.6 Report As Liquid (Special Case)

**Derivation Chain:**

```
MAQReportService.fetchActuals():513-519
    const reportAsLiquid = fireCodeHazardClass?.gas?.reportAsLiquid || false;
    const targetKey = stateKey === 'gas' && reportAsLiquid ? 'liquid' : stateKey;

    // If gas should be reported as liquid, values are stored under 'liquid' key
    hazardClass[targetKey] = { actualWeight, actualVolume }
    ↓
SOURCE: MongoDB 'fireCode' collection
    occupancy.hazardClasses[].gas.reportAsLiquid: Boolean
```

---

## 6. MAQ Limit Calculation

### 6.1 Display Limit (`hazardClass.solid.displayLimit`)

**Rendered in:** MaqTable.tsx:65-75 via `DisplayMaqFormulae` component

**Complete Derivation Chain:**

```
UI Display: DisplayMaqFormulae shows limit value with hover tooltip
    ↓
DisplayMaqFormulae.tsx:49
    hazardClass[fireCodePhysicalState].limit
    ↓
GraphQL Response: hazardClass.solid.displayLimit
    ↓
getHazardClassLimits():743-750
    let displayLimit = `${limit.value}`;
    if (isNoLimit) displayLimit = 'NL';
    if (isNotApplicable) displayLimit = 'N/A';
    ↓
calculateLimitByState():358-398
    [baseline, ...factors] = getMaqFactors(...)
    return factors.reduce((acc, cur) => {
        if (cur.operator === MULTIPLY) {
            return { ...acc, value: acc.value * cur.value }
        }
        return acc;
    }, { value: baseline.value, ... })
    ↓
CALCULATION: baseline × factor1 × factor2 × ...
```

### 6.2 MAQ Factors Breakdown

**Rendered in:** DisplayMaqFormulae tooltip on hover

**Derivation Chain:**

```
UI Display: Tooltip table showing factor labels and values
    ↓
DisplayMaqFormulae.tsx:55-68
    maqFactors.map(factor => <th>{factor.label}</th>)
    maqFactors.map(factor => <td>{operator} {value}</td>)
    ↓
GraphQL Response: hazardClass.solid.maqFactors[]
    { label, value, operator, isNoLimit, isNotApplicable }
    ↓
getHazardClassLimits():753-769
    stateMaqFactors = getMaqFactors(
        fireCodeHazardClass,
        occupancyRules,
        physicalState,
        fireSuppressionType,
        approvedStorage,
        isOutdoor,
        floorAboveGroundPlane
    )
    ↓
getMaqFactors():221-356
    Returns array of MaqFactor objects
```

### 6.3 Baseline Value (First MAQ Factor)

**Derivation Chain:**

```
getMaqFactors():303-312
    maqFactors = [{
        label: 'Baseline',
        value: fireCodeHazardClass[physicalState].baseline ?? null,
        isNoLimit: fireCodeHazardClass[physicalState].isNoLimit || false,
        isNotApplicable: false,
        operator: '' as RuleOperators
    }]
    ↓
SOURCE: MongoDB 'fireCode' collection
    occupancy.hazardClasses[].solid.baseline: Float
    occupancy.hazardClasses[].solid.units: String
    occupancy.hazardClasses[].solid.isNoLimit: Boolean
    occupancy.hazardClasses[].solid.isNotApplicable: Boolean
```

### 6.4 Conditional Rule Factors

**Derivation Chain:**

```
getMaqFactors():314-349
    for (const rule of fireCodeHazardClass.rules) {
        if (ruleApplies(rule, ...)) {
            const { operator, value, isNoLimit, isNotApplicable } = rule[physicalState];
            const ruleLabel = rule.conditions.map(c => capitalizeFirstLetter(c.criteria)).join(' & ');

            if (isNotApplicable || operator === OVERRIDE || isNoLimit) {
                maqFactors = [{ label: ruleLabel, ...}];  // Replace all factors
                break;
            } else if (operator === EQUALS) {
                maqFactors[0] = { label: ruleLabel, ...};  // Replace baseline
            } else {
                maqFactors.push({ label: ruleLabel, ...});  // Add multiplier
            }
        }
    }
    ↓
ruleApplies():162-181
    rule.conditions.every(condition => conditionApplies(condition, ...))
    ↓
conditionApplies():132-160
    switch (condition.criteria) {
        case SPRINKLER_BASEMENT_ONLY:
            return isTrue(value) === (fireSuppressionType === 'BASEMENT_ONLY');
        case SPRINKLER:
            return isTrue(value) === isControlAreaSprinklered(fireSuppressionType, floor);
        case APPROVED_STORAGE:
            return isTrue(value) === approvedStorage.some(h =>
                h.hazardClass === hazardClass.name && h.physicalStates.includes(physicalState)
            );
        case OUTDOOR:
            return isTrue(value) === isOutdoor;
        case BASEMENT:
            return isTrue(value) === (floor < 0);
    }
    ↓
SOURCE: MongoDB 'fireCode' collection
    occupancy.hazardClasses[].rules[]: {
        id: String,
        conditions: [{ criteria: String, operator: String, value: String }],
        solid: { value: Float, operator: String, isNoLimit: Boolean, isNotApplicable: Boolean },
        liquid: { ... },
        gas: { ... }
    }
```

### 6.5 Floor Level Factor

**Derivation Chain:**

```
getMaqFactors():351-353
    if (!isOutdoor) {
        maqFactors = maqFactors.concat(getFloorMaqFactors(occupancyRules, floorAboveGroundPlane));
    }
    ↓
getFloorMaqFactors():183-219
    fireCodeOccupancyRules.forEach(rule => {
        if (
            (rule.operator === 'is equal to' && rule.appliedToFloors.includes(floor)) ||
            (rule.operator === 'is greater than' && floor > rule.appliedToFloors[0]) ||
            (rule.operator === 'is less than' && floor < rule.appliedToFloors[0])
        ) {
            floorMaqFactors.push({
                label: 'Floor Level',
                value: rule.percentage / 100,  // e.g., 75 → 0.75
                operator: MULTIPLY
            });
        }
    });
    ↓
SOURCE: MongoDB 'fireCode' collection
    occupancy.rules[]: {
        operator: 'is equal to' | 'is greater than' | 'is less than',
        appliedToFloors: [Int],  // e.g., [-1, -2] for basements
        percentage: Float  // e.g., 75 for 75%
    }
```

### 6.6 Sprinkler Coverage Determination

**Critical Helper Function:**

```
isControlAreaSprinklered():122-130
    if (fireSuppressionType === 'FULL') return true;
    if (fireSuppressionType === 'BASEMENT_ONLY') {
        return typeof floorAboveGroundPlane === 'number' && floorAboveGroundPlane < 0;
    }
    return false;
```

**Usage in Rules:**
- FULL suppression → Always sprinklered → 2x multiplier typically applies
- BASEMENT_ONLY → Only sprinklered if floor < 0 (basement)
- NONE → Never sprinklered

### 6.7 Complete MAQ Calculation Example

```
Example: "Flammable Liquid: IA" in Control Area with:
- Occupancy: B
- Fire Suppression: FULL
- Floor: 0 (ground)
- No approved storage
- Not outdoor

Calculation:
    Baseline: 30 gal (from fire code)
    × Sprinkler: 2 (FULL = sprinklered)
    × Floor Level: 1.0 (ground floor = no reduction)
    = 60 gal

MAQ Factors Array:
[
    { label: 'Baseline', value: 30, operator: '' },
    { label: 'Sprinkler', value: 2, operator: 'MULTIPLY' }
]

Display: 60 gal with tooltip showing "Baseline: 30, * Sprinkler: 2"
```

---

## 7. Compliance Status

### 7.1 Physical State Status (`hazardClass.solid.status`)

**Rendered in:** MaqTable cell color (red for OVER_THRESHOLD)

**Derivation Chain:**

```
UI Display: Text color red if OverThreshold
    MaqTable.tsx:49
    value?.status === ComplianceStatus.OverThreshold ? 'text-red-500' : 'text-darkgray-500'
    ↓
GraphQL Response: hazardClass.solid.status
    ↓
getControlAreaMaq():882
    status: getComplianceStatus(hazardClassLimit.solid.limit, solidActual)
    ↓
getComplianceStatus():38-57
    if (limit === null) return COMPLIANT;  // NL = no limit
    if (limit === 0 && actual === 0) return COMPLIANT;
    if (actual >= limit) return OVER_THRESHOLD;
    if (actual >= limit * 0.8) return NEAR_THRESHOLD;  // >= 80%
    return COMPLIANT;
    ↓
CALCULATION: Compare actual vs limit
    actual >= limit → OVER_THRESHOLD
    actual >= 80% of limit → NEAR_THRESHOLD
    else → COMPLIANT
```

### 7.2 Control Area Status (`controlArea.status`)

**Rendered in:** Control area headers, building list

**Derivation Chain:**

```
UI Display: ComplianceIcon component
    ↓
GraphQL Response: controlArea.status
    ↓
getControlAreaMaq():902
    status: controlArea.exemption.isExempt
        ? ComplianceStatus.Exempt
        : getControlAreaCompliance(maq)
    ↓
getControlAreaCompliance():59-80
    controlAreaHazardClasses.every(hazardClass => {
        if (solid/liquid/gas.status === OVER_THRESHOLD) {
            status = OVER_THRESHOLD;
            return false;  // Stop iteration
        }
        if (solid/liquid/gas.status === NEAR_THRESHOLD) {
            status = NEAR_THRESHOLD;
            // Continue to check for worse status
        }
        return true;
    });
    return status;
    ↓
CALCULATION: Worst status from any physical state of any hazard class
    Priority: OVER_THRESHOLD > NEAR_THRESHOLD > COMPLIANT
```

---

## 8. Data Source Summary

### Primary Data Sources

| Source | Type | Data |
|--------|------|------|
| MongoDB `fireCode` | Document DB | Fire code rules, baselines, occupancies, hazard class mappings |
| MongoDB `controlArea` | Document DB | Control area metadata, exemptions, approved storage |
| Elasticsearch `container` | Search Index | Container quantities, locations, exemption status |
| External Building Service | REST API | Building info, floors, rooms, fire suppression type |

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              RENDERED UI (MaqTable)                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Material  │  │    Solid    │  │   Liquid    │  │     Gas     │             │
│  │   Name      │  │ Actual│MAQ  │  │ Actual│MAQ  │  │ Actual│MAQ  │             │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                         ▲
                                         │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            GraphQL Response Layer                                │
│  buildingMaqReport.controlAreas[].hazardClasses[].{solid,liquid,gas}            │
│    - displayValue (formatted actual)                                             │
│    - displayLimit (formatted limit or NL/N/A)                                    │
│    - limit (numeric)                                                             │
│    - actualWeight / actualVolume                                                 │
│    - status (COMPLIANT/NEAR_THRESHOLD/OVER_THRESHOLD)                           │
│    - maqFactors[] (calculation breakdown)                                        │
│    - units                                                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                         ▲
                                         │
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         MAQ Calculation Layer (common)                           │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │ getControlAreaMaq() - Main entry point                                     │  │
│  │   ├── getControlAreaLimits() - Calculate limits for all hazard classes    │  │
│  │   │     ├── getHazardClassLimits() - Per hazard class                     │  │
│  │   │     │     ├── calculateLimitByState() - Per physical state            │  │
│  │   │     │     │     ├── getMaqFactors() - Get all applicable factors      │  │
│  │   │     │     │     │     ├── Baseline from fire code                     │  │
│  │   │     │     │     │     ├── Rules evaluation (ruleApplies)              │  │
│  │   │     │     │     │     └── Floor level factors                         │  │
│  │   │     │     │     └── Reduce factors: base × factor1 × factor2 × ...    │  │
│  │   │     │     └── Unit conversion (gram→lb, liter→gal/ft³)               │  │
│  │   │     └── Return { name, solid, liquid, gas }                           │  │
│  │   └── Merge with actuals, calculate compliance status                      │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                         ▲
                        ┌────────────────┼────────────────┐
                        │                │                │
┌───────────────────────┴───┐ ┌──────────┴─────────┐ ┌────┴────────────────────────┐
│     Fire Code Rules       │ │  Control Area Data │ │     Container Actuals       │
│     (MongoDB)             │ │  (MongoDB)         │ │     (Elasticsearch)         │
├───────────────────────────┤ ├────────────────────┤ ├─────────────────────────────┤
│ • Baseline values         │ │ • occupancy        │ │ • normalizedSize.gram       │
│ • Units (lbs/gal/ft³)     │ │ • isOutdoor        │ │ • normalizedSize.liter      │
│ • isNoLimit flags         │ │ • approvedStorage  │ │ • family.formAtNtp (state)  │
│ • isNotApplicable flags   │ │ • exemption        │ │ • family.bands._id          │
│ • Conditional rules       │ │ • fireSuppr.       │ │ • location.roomId           │
│   - conditions            │ │   Override         │ │ • exemption status          │
│   - multipliers           │ │ • notes            │ │ • active flag               │
│ • Floor level rules       │ └────────────────────┘ └─────────────────────────────┘
│ • Hazard class mappings   │           ▲                         ▲
│   (band → hazard class)   │           │                         │
└───────────────────────────┘ ┌─────────┴─────────┐    ┌──────────┴──────────────┐
            ▲                 │ Control Area      │    │ Container Index         │
            │                 │ Collection        │    │ (container)             │
┌───────────┴───────────────┐ │ (controlArea)     │    │                         │
│ Fire Code Collection      │ │                   │    │ Pre-computed from       │
│ (fireCode)                │ │ User-created,     │    │ chemical inventory      │
│                           │ │ references rooms  │    │ system via Kafka        │
│ Admin-managed fire code   │ │ in building       │    │                         │
│ configurations            │ └───────────────────┘    └─────────────────────────┘
└───────────────────────────┘           ▲                         ▲
                                        │                         │
                            ┌───────────┴─────────────────────────┴───────────────┐
                            │              External Building Service               │
                            │  • Building ID, Name, Address                        │
                            │  • Floors with groundPlane                           │
                            │  • Rooms with controlAreaId assignment               │
                            │  • fireSuppressionType (NONE/BASEMENT_ONLY/FULL)     │
                            └─────────────────────────────────────────────────────┘
```

### Conversion Constants

| Conversion | Constant | Used For |
|------------|----------|----------|
| Gram → Pound | 0.0022046226 | Solid weights |
| Liter → Gallon | 0.2641720524 | Liquid volumes |
| Liter → Cubic Feet | 0.0353146667 | Gas volumes |

### Compliance Thresholds

| Status | Condition |
|--------|-----------|
| COMPLIANT | actual < 80% of limit OR limit is null (NL) |
| NEAR_THRESHOLD | actual >= 80% of limit AND actual < limit |
| OVER_THRESHOLD | actual >= limit |
| EXEMPT | control area marked exempt |
| INCOMPLETE | no control areas in building |

---

## Appendix A: Key File Locations

| File | Purpose |
|------|---------|
| `packages/client/src/app/building-maq/control-area/MaqTable.tsx` | Main table rendering |
| `packages/client/src/app/building-maq/control-area/DisplayMaqFormulae.tsx` | MAQ factor tooltip |
| `packages/client/src/app/building-maq/control-area/ControlAreaDetail.tsx` | Control area page |
| `packages/client/src/app/graphql/fragments.ts` | GraphQL data shapes |
| `packages/common/src/utils/maq-compliance.helper.ts` | All MAQ calculations |
| `packages/common/src/utils/misc.ts` | Formatting utilities |
| `packages/server/src/building/MAQReportService.ts` | Server-side orchestration |
| `packages/server/src/fire-code/FireCodeService.ts` | Fire code data access |
| `packages/server/src/control-area/ControlAreaService.ts` | Control area data access |

---

## Appendix B: Calculation Verification Checklist

When verifying a MAQ calculation, check each step:

- [ ] **Baseline**: Match fire code occupancy → hazard class → physical state → baseline
- [ ] **Sprinkler Factor**: Building type + floor level → isControlAreaSprinklered() → rule match
- [ ] **Approved Storage**: Control area approved storage matches hazard class + physical state
- [ ] **Floor Level**: Occupancy rules with matching floor conditions
- [ ] **Outdoor Override**: If outdoor, floor level factors should NOT apply
- [ ] **Exemption Override**: If control area exempt, all limits should be NL
- [ ] **Override Rule**: If OVERRIDE operator matched, it replaces entire calculation
- [ ] **Report As Liquid**: For gases with reportAsLiquid, values aggregate to liquid
- [ ] **Unit Conversion**: Verify correct conversion constant applied
- [ ] **Actual Aggregation**: Container quantities filtered by room, exemption, active status

---

## Appendix C: HMIS Export Calculations (Open/Closed MAQ)

The HMIS export provides additional open/closed container MAQ values. These are calculated differently from the main MAQ display.

### Open vs Closed MAQ

**Location:** `packages/client/src/utils/MaqExportHelper.tsx`

**Closed MAQ:** `useClosedMaq()` (lines 241-279)
- Most hazard classes: closedLimit = storageMaq × multiplier
- Multipliers vary by hazard class (0.1x to 1.0x)
- Some always return 'NL' or '0'

**Open MAQ:** `useOpenMaq()` (lines 281-341)
- More restrictive than closed
- Many gas types return 'N/A' for open use
- Multipliers typically 0.2x to 0.33x

### Server-Side Open/Closed Calculation

**Location:** `maq-compliance.helper.ts:400-727` - `getOpenAndClosedMaqLimit()`

This function provides hardcoded open/closed limits per hazard class, used when `fireCode.hmis_report === true`.

**Example Calculation:**
```
Flammable Liquid: IB, IC
├── Fire Suppression Factor: 2 (if sprinklered)
├── Closed Limit: 120 × fireSuppressionFactor = 240 gal
└── Open Limit: 30 × fireSuppressionFactor = 60 gal
```

### HMIS Row Structure

```typescript
{
  'CFC 2016 Hazard, based on chemical families': string,  // e.g., "Flammable Liquid"
  'CFC 2016 Hazard Class': string,                        // e.g., "IB, IC"
  'In Storage': number | string,                          // Current actual quantity
  'Storage MAQ': number | string,                         // Main MAQ limit
  'Use-Closed': null,                                     // Currently unused
  'Use-Closed MAQ': number | string,                      // Closed container MAQ
  'Use-Open': null,                                       // Currently unused
  'Use-Open MAQ': number | string,                        // Open container MAQ
  'State of Matter of Containers': string,                // solid/liquid/gas
  'Units': string                                         // Display units
}
```

---

## Appendix D: Special Cases and Edge Conditions

### 1. Report As Liquid Flag

**Condition:** `fireCodeHazardClass.gas.reportAsLiquid === true`

**Effect:**
- Gas containers aggregate to liquid actuals (server-side)
- MAQ factors use liquid baseline instead of gas baseline
- Display shows "Report as Liquid" as first MAQ factor

**Location:**
- Server: `MAQReportService.fetchActuals():513-519`
- Common: `getMaqFactors():245-300`

### 2. Liquefied State (Health Hazards Only)

**Condition:** Health hazard classes with "Liquefied" in name (e.g., "Toxic Liquefied")

**Effect:**
- Fourth column added to MaqTable for liquefied state
- `addLiquefiedState()` matches base hazard to liquefied variant
- Uses liquid values from the liquefied hazard class

**Location:** `MaqExportHelper.tsx:72-91`

### 3. Control Area Exemption Override

**Condition:** `controlArea.exemption.isExempt === true`

**Effect:**
- All limits set to `null` (displayed as "NL")
- Status forced to `EXEMPT`
- `exemptLimits()` returns exempted hazard class structure

**Location:**
- `maq-compliance.helper.ts:100-118` - `exemptLimits()`
- `maq-compliance.helper.ts:735-737` - exemption check in `getHazardClassLimits()`

### 4. Override Rules

**Condition:** Rule with `operator === 'OVERRIDE'`

**Effect:**
- Replaces entire MAQ factor calculation
- Only the override value is used (no baseline, no multipliers)
- Stops rule processing immediately

**Location:** `getMaqFactors():339-341`

### 5. No Limit (NL) Handling

**Condition:**
- `fireCodeHazardClass[state].isNoLimit === true`, OR
- Rule result has `isNoLimit === true`

**Effect:**
- `displayLimit = 'NL'`
- `limit = null`
- Compliance status = COMPLIANT (no limit means compliant)

### 6. Not Applicable (N/A) Handling

**Condition:**
- `fireCodeHazardClass[state].isNotApplicable === true`, OR
- Rule result has `isNotApplicable === true`

**Effect:**
- `displayLimit = 'N/A'`
- Physical state not tracked for this hazard class

### 7. Outdoor Control Areas

**Condition:** `controlArea.isOutdoor === true`

**Effect:**
- Floor level factors are NOT applied
- Outdoor-specific rules may apply/not apply

**Location:** `getMaqFactors():296, 351` - outdoor check before floor factors

### 8. Fire Suppression Type Logic

```
Building.fireSuppressionType
    │
    ├── 'FULL' → All floors sprinklered → Sprinkler rules apply everywhere
    │
    ├── 'BASEMENT_ONLY' → Only floors < 0 sprinklered
    │   ├── Floor < 0 → Sprinkler rules apply
    │   └── Floor >= 0 → Sprinkler rules do NOT apply
    │
    └── 'NONE' → No sprinklers → Sprinkler rules never apply

Control Area can override via fireSuppressionOverride:
    ├── override: true, value: true → Force sprinklered
    └── override: true, value: false → Force not sprinklered
```

### 9. Container Exemption in Elasticsearch

**Condition:** Container has non-empty `exemption` field

**Effect:**
- Container excluded from MAQ aggregation via Elasticsearch script filter
- Filter: `doc['exemption.keyword'].size() > 0 && doc['exemption.keyword'].value.length() > 0`

**Location:** `MAQReportService.fetchActuals():470-473`

### 10. Inactive Containers

**Condition:** `container.active === false`

**Effect:**
- Container excluded from MAQ aggregation
- Filter: `{ term: { active: false } }` in must_not

---

*Document generated: December 2024*
*Last verified against codebase: packages/common/src/utils/maq-compliance.helper.ts*

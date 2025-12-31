# MAQ Complete Guide: Maximum Allowable Quantities for Fire Code Compliance

**Version:** 1.1
**Last Updated:** December 2024
**Audience:** Fire Marshals, Safety Officers, Compliance Administrators, Fire Inspectors, System Implementers

---

## Table of Contents

1. [Introduction: What is MAQ?](#1-introduction-what-is-maq)
2. [Why MAQ Matters](#2-why-maq-matters)
3. [Regulatory Framework](#3-regulatory-framework)
4. [Core Concepts](#4-core-concepts)
5. [Complete Variable Reference](#5-complete-variable-reference)
6. [The MAQ Calculation Engine](#6-the-maq-calculation-engine)
7. [Factor Deep Dives](#7-factor-deep-dives)
8. [Hazard Class Mapping System](#8-hazard-class-mapping-system)
9. [Storage vs. Use Conditions](#9-storage-vs-use-conditions)
10. [Open/Closed MAQ Calculations (HMIS)](#10-openclosed-maq-calculations-hmis)
11. [Special Cases & Edge Conditions](#11-special-cases--edge-conditions)
12. [Compliance Status Determination](#12-compliance-status-determination)
13. [Data Sources & Integration](#13-data-sources--integration)
14. [Practical Scenarios & Worked Examples](#14-practical-scenarios--worked-examples)
15. [Common Mistakes to Avoid](#15-common-mistakes-to-avoid)
16. [Quick Reference Tables](#16-quick-reference-tables)
17. [Manual Calculation Guide](#17-manual-calculation-guide)
18. [Glossary](#18-glossary)
19. [Appendix A: File Locations (For Developers)](#appendix-a-file-locations-for-developers)
20. [Appendix B: Field Inspector Guides](#appendix-b-field-inspector-guides)
    - [B.1 Hazard Class Identification](#b1-hazard-class-identification)
    - [B.2 MAQ Baseline Tables](#b2-maq-baseline-tables)
        - [B.2.6 Complete Use-Open/Use-Closed Tables](#b26-complete-use-openuse-closed-tables)
        - [B.2.7 California Fire Code vs IFC Deviations](#b27-california-fire-code-cfc-vs-ifc-deviations)
        - [B.2.8 Combination Limits (Flammable Liquids)](#b28-combination-limits-flammable-liquids)
        - [B.2.9 Multi-Hazard Chemicals](#b29-multi-hazard-chemicals)
        - [B.2.10 Compressed Gas Cylinder Volume Calculations](#b210-compressed-gas-cylinder-volume-calculations)
        - [B.2.11 MAQ Exemptions](#b211-maq-exemptions)
    - [B.3 Floor Level Factors](#b3-floor-level-factors-ibc-table-41422)
    - [B.4 Storage vs Use Determination](#b4-storage-vs-use-determination-field-guide)
    - [B.5 Approved Storage Verification](#b5-approved-storage-verification-field-guide)
    - [B.6 Field Quantity Measurement](#b6-field-quantity-measurement-guide)
    - [B.7 Control Area Boundary Determination](#b7-control-area-boundary-determination)
    - [B.8 Field Inspection Checklist](#b8-field-inspection-checklist)
    - [B.9 Sprinkler System Field Verification](#b9-sprinkler-system-field-verification)
    - [B.10 Occupancy Classification and Determination](#b10-occupancy-classification-and-determination)
    - [B.11 Ground Plane Determination](#b11-ground-plane-determination)
    - [B.12 Unknown and Unlabeled Chemical Handling](#b12-unknown-and-unlabeled-chemical-handling)
    - [B.13 Outdoor Control Area Verification](#b13-outdoor-control-area-verification)
    - [B.14 Enforcement and Non-Compliance Procedures](#b14-enforcement-and-non-compliance-procedures)

---

## 1. Introduction: What is MAQ?

**Maximum Allowable Quantity (MAQ)** is the maximum amount of a hazardous material permitted in a **control area** before a building must be classified as a **High-Hazard (Group H) occupancy**.

Think of MAQ as a threshold. Below it, you can store and use hazardous materials in a standard building. Above it, you need special construction, fire protection systems, and operational requirements that significantly increase costs and regulatory burden.

### The Fundamental Equations

**Simple threshold test:**
```
If actual quantity < MAQ  →  Standard occupancy allowed
If actual quantity ≥ MAQ  →  High-Hazard occupancy required
```

**The MAQ calculation formula:**
```
MAQ = Baseline × Sprinkler Factor × Floor Factor × Approved Storage Factor × Rule Modifiers
```

### Key Insight for Inspectors

MAQ is not about whether a material is "safe" or "dangerous." It's about **quantity thresholds** that trigger different building code requirements. A small amount of even highly toxic material might be under MAQ, while large quantities of relatively benign materials might exceed it.

### Key Entities

| Entity | Description |
|--------|-------------|
| **Fire Code** | Regulatory framework (e.g., CFC 2016, CFC 2022) defining limits |
| **Building** | Physical structure with fire suppression and floor data |
| **Control Area** | Bounded zone within a building for MAQ calculations |
| **Hazard Class** | Category of hazardous material (e.g., Flammable Liquid IA) |
| **Physical State** | Form of material: Solid, Liquid, Gas, or Liquefied |

---

## 2. Why MAQ Matters

### For Building Owners and Operators

- **Cost**: High-Hazard occupancy classification requires expensive construction features (fire barriers, special ventilation, explosion venting)
- **Insurance**: Higher premiums for H-occupancy buildings
- **Operations**: Stricter operating permits and inspections

### For Fire Departments

- **Pre-planning**: Knowing MAQ status helps with emergency response planning
- **Risk assessment**: Identifies where significant hazardous material quantities exist
- **Code enforcement**: Clear, quantifiable compliance standard

### For Public Safety

- **Fire protection**: Limits hazardous material accumulation in standard buildings
- **Emergency response**: Ensures adequate fire protection when quantities are high
- **Community protection**: Reduces risk of major chemical incidents

---

## 3. Regulatory Framework

### Primary Codes

| Code | Publisher | Notes |
|------|-----------|-------|
| **International Fire Code (IFC)** | ICC | Most widely adopted; basis for most state/local codes |
| **International Building Code (IBC)** | ICC | Defines control areas and occupancy classifications |
| **California Fire Code (CFC)** | CA State | IFC with California amendments |
| **NFPA 1 Fire Code** | NFPA | Alternative to IFC; used in some jurisdictions |

### Supporting Standards

| Standard | Topic |
|----------|-------|
| **NFPA 30** | Flammable and Combustible Liquids |
| **NFPA 400** | Hazardous Materials Code |
| **NFPA 55** | Compressed Gases and Cryogenic Fluids |
| **NFPA 13** | Sprinkler Systems (referenced for MAQ increases) |

### Key Code Sections

For IFC (and CFC), the critical sections are:

- **Section 5003.1.1** - Maximum allowable quantity tables
- **Table 5003.1.1(1)** - Indoor storage/use, physical hazards
- **Table 5003.1.1(2)** - Indoor storage/use, health hazards
- **Table 5003.1.1(3)** - Outdoor storage/use, physical hazards
- **Table 5003.1.1(4)** - Outdoor storage/use, health hazards

For IBC:

- **Table 414.2.2** - Control area design requirements (floor percentages)
- **Section 307.1** - High-hazard group definitions

---

## 4. Core Concepts

### 4.1 Control Areas

A **Control Area** is a bounded space within a building where hazardous materials are stored, used, or handled. Control areas are separated from each other by fire-rated construction.

**Key Characteristics:**
- Contains one or more rooms
- Has a single **occupancy classification** (A, B, E, F, H, etc.)
- May be indoor or outdoor
- May have approved storage for specific hazard classes
- Can be exempted from MAQ calculations

**Why Control Areas Matter:**
- Fire codes limit hazardous material quantities per control area, not per room or per building
- A single building may have multiple control areas with different limits
- Each control area has its own MAQ limits
- Control areas must be separated by 1-hour or 2-hour fire barriers

**Control Areas per Floor (IBC Table 414.2.2):**

| Floor Level | Max Control Areas |
|-------------|-------------------|
| Ground (1st) | 4 |
| 2nd floor | 3 |
| 3rd floor | 2 |
| 4th-9th floor | 2 |
| 10th+ floor | 1 |
| 1st basement | 3 |
| 2nd basement | 2 |
| 3rd+ basement | Not permitted |

### 4.2 Hazard Classes

Hazardous materials are divided into two main categories:

**Physical Hazards** (fire/explosion risk):
- Combustible Liquids (Class II, IIIA, IIIB)
- Flammable Liquids (Class IA, IB, IC)
- Flammable Solids
- Flammable Gases
- Oxidizers (Class 1, 2, 3, 4)
- Organic Peroxides (Class I-V, UD)
- Pyrophoric Materials
- Unstable/Reactive Materials (Class 1-4)
- Water-Reactive Materials (Class 1-3)
- Explosives (Division 1.1-1.6)
- Cryogenic Fluids

**Health Hazards** (toxicity risk):
- Corrosives
- Toxics
- Highly Toxics

### 4.3 Occupancy Classifications

Occupancy determines which fire code rules apply:

| Code | Occupancy Type | Description |
|------|---------------|-------------|
| **A** | Assembly | Gathering places (theaters, churches) |
| **B** | Business | Offices, professional services, laboratories |
| **E** | Educational | Schools, training facilities |
| **F** | Factory | Manufacturing, industrial |
| **H** | High-Hazard | H-1 through H-5 classifications (facilities exceeding MAQ) |
| **I** | Institutional | Hospitals, care facilities |
| **M** | Mercantile | Retail stores, markets |
| **R** | Residential | Homes, apartments, hotels |
| **S** | Storage | Warehouses, parking garages |
| **U** | Utility | Agricultural, miscellaneous |

### 4.4 Physical States

MAQ limits are specified separately for each physical state:

| State | Unit of Measure | Description |
|-------|-----------------|-------------|
| **Solid** | pounds (lbs) | Materials in solid form at NTP (powders, pellets, crystals) |
| **Liquid** | gallons (gal) or pounds (lbs) | Materials in liquid form at NTP (solvents, acids, fuels) |
| **Gas** | cubic feet (ft³) | Materials in gaseous form at NTP (compressed gases, vapors) |
| **Liquefied** | gallons (gal) | Gases stored in liquefied form |

**NTP** = Normal Temperature and Pressure (20°C/68°F, 1 atm)

**Important:** A single hazard class may have different MAQ limits for solid, liquid, and gas forms.

---

## 5. Complete Variable Reference

### 5.1 Fire Code Variables (Foundation)

The fire code establishes the regulatory framework:

| Variable | Description | Example |
|----------|-------------|---------|
| `name` | Fire code identifier | "CFC 2022" |
| `occupancies[]` | List of occupancy configurations | B, H-2, S |
| `occupancies[].hazardClasses[]` | Hazard classes with baselines | Flammable Liquid IA |
| `occupancies[].rules[]` | Floor-level reduction rules | "Floor < 0 → 75%" |
| `hazardClassMappings[]` | Chemical band → hazard class mappings | See Section 8 |
| `hmis_report` | Enable HMIS export with open/closed MAQ | true/false |

### 5.2 Building Variables (External Data)

Building properties come from facilities management:

| Variable | Description | Impact on MAQ |
|----------|-------------|--------------|
| `fireSuppressionType` | Sprinkler coverage type | Determines sprinkler multiplier |
| `floors[].groundPlane` | Floor position relative to ground | Determines floor-level factors |
| `floors[].rooms[]` | Physical spaces in building | Container locations |

**Fire Suppression Types:**

| Type | Meaning | Sprinkler Coverage |
|------|---------|-------------------|
| `FULL` | Entire building sprinklered | All floors = sprinklered |
| `BASEMENT_ONLY` | Only basements sprinklered | Floor < 0 = sprinklered |
| `NONE` | No fire suppression | No floors sprinklered |

### 5.3 Control Area Variables (User-Configurable)

Control area administrators can modify these:

| Variable | Description | Impact on MAQ |
|----------|-------------|--------------|
| `occupancy` | Occupancy classification | Determines which fire code rules apply |
| `isOutdoor` | Outdoor location flag | Skips floor-level reductions if true |
| `floorAboveGroundPlane` | Floor level | Determines floor-level multipliers |
| `approvedStorage[]` | Approved storage designations | Can double limits (2× multiplier) |
| `fireSuppressionOverride` | Override building sprinkler status | Force sprinkler on/off |
| `exemption.isExempt` | Exemption status | All limits become "No Limit" |
| `exemption.reason` | Exemption reason | Documentation only |
| `exemption.notes` | Additional notes | Documentation only |

### 5.4 Chemical/Container Variables (Inventory System)

Container data comes from the chemical inventory system:

| Variable | Description | Storage |
|----------|-------------|---------|
| `normalizedSize.gram` | Weight in grams | Elasticsearch |
| `normalizedSize.liter` | Volume in liters | Elasticsearch |
| `family.formAtNtp` | Physical state at NTP | solid/liquid/gas |
| `family.bands[]._id` | Chemical band identifiers | Links to hazard class |
| `location.roomId` | Room where container is located | Links to control area |
| `exemption` | Container exemption status | Excluded if set |
| `active` | Container active status | Excluded if false |

---

## 6. The MAQ Calculation Engine

### 6.1 The Master Formula

```
Final MAQ = Baseline × Sprinkler Factor × Floor Level Factor × Approved Storage Factor
```

Each factor is determined by specific conditions:

| Factor | Typical Values | Condition |
|--------|---------------|-----------|
| **Baseline** | From code tables | Per hazard class and physical state |
| **Sprinkler Factor** | 1× or 2× | Building sprinklered throughout? |
| **Floor Level Factor** | 0.05× to 1× | What floor is the control area on? |
| **Approved Storage Factor** | 1× or 2× | Materials in approved cabinets? |

### 6.2 Factor Application

Factors are **multiplicative** and **cumulative**. This is explicitly stated in IFC footnotes:

> "Where Note d also applies, the increase for both notes shall be applied accumulatively."

**Example:**
```
Baseline: 30 gallons
Sprinkler Factor: 2× (building fully sprinklered)
Approved Storage Factor: 2× (in approved flammable cabinet)
Floor Level Factor: 1× (ground floor)

Final MAQ = 30 × 2 × 2 × 1 = 120 gallons
```

### 6.3 Calculation Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           MAQ CALCULATION FLOW                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   STEP 1: Get Baseline                                                       │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │  fireCode.occupancies[{occupancy}].hazardClasses[{name}].{state}     │  │
│   │  .baseline → e.g., 30 gal                                             │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│   STEP 2: Apply Conditional Rules (from hazardClass.rules[])                │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │  IF sprinklered → × 2                                                 │  │
│   │  IF approved storage → × 2                                            │  │
│   │  IF outdoor + specific rule → may override                            │  │
│   │  IF basement + specific rule → may override                           │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│   STEP 3: Apply Floor-Level Rules (from occupancy.rules[])                  │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │  Floor < 0 (basement) → × 0.75 (typical)                              │  │
│   │  Floor > 2 (high floor) → × 0.50 (typical)                            │  │
│   │  Skip if outdoor                                                       │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                       │
│                                      ▼                                       │
│   STEP 4: Calculate Final Limit                                             │
│   ┌──────────────────────────────────────────────────────────────────────┐  │
│   │  Final MAQ = baseline × factor1 × factor2 × ...                       │  │
│   │  e.g., 30 × 2 × 0.75 = 45 gal                                         │  │
│   └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.4 MAQ Factor Types

#### Baseline Factor

The starting point from the fire code:

```javascript
{
  label: 'Baseline',
  value: 30,           // e.g., 30 gallons
  operator: '',        // No operator for baseline
  isNoLimit: false,
  isNotApplicable: false
}
```

#### Multiplier Factors (MULTIPLY)

Applied by multiplication:

```javascript
{
  label: 'Sprinkler',
  value: 2,
  operator: 'MULTIPLY'
}
// Result: baseline × 2
```

#### Replacement Factors (EQUALS)

Replaces the baseline:

```javascript
{
  label: 'Outdoor Storage',
  value: 120,
  operator: 'EQUALS'
}
// Result: 120 (baseline ignored)
```

#### Override Factors (OVERRIDE)

Replaces entire calculation:

```javascript
{
  label: 'Special Condition',
  value: 0,
  operator: 'OVERRIDE',
  isNotApplicable: true
}
// Result: N/A (entire calculation overridden)
```

### 6.5 Rule Condition Types

Rules are applied when ALL conditions match:

| Criteria | Description | Example |
|----------|-------------|---------|
| `SPRINKLER` | Control area is sprinklered | `value: 'true'` |
| `SPRINKLER_BASEMENT_ONLY` | Building has BASEMENT_ONLY suppression | `value: 'true'` |
| `APPROVED_STORAGE` | Has approved storage for this hazard class | `value: 'true'` |
| `OUTDOOR` | Control area is outdoor | `value: 'true'` |
| `BASEMENT` | Control area is in basement (floor < 0) | `value: 'true'` |

### 6.6 Sprinkler Coverage Logic

```
Building Fire Suppression Type + Floor Level = Is Control Area Sprinklered?

┌─────────────────────────────────────────────────────────────────────────┐
│  Fire Suppression Type    │    Floor Level    │    Sprinklered?        │
├───────────────────────────┼───────────────────┼────────────────────────┤
│  FULL                     │    Any floor      │    ✅ YES              │
│  BASEMENT_ONLY            │    Floor < 0      │    ✅ YES              │
│  BASEMENT_ONLY            │    Floor >= 0     │    ❌ NO               │
│  NONE                     │    Any floor      │    ❌ NO               │
└─────────────────────────────────────────────────────────────────────────┘

Can be overridden at control area level:
  controlArea.fireSuppressionOverride = { override: true, value: true/false }
```

### 6.7 Floor-Level Reduction Rules

Typical floor-level rules (vary by fire code):

| Condition | Percentage | Meaning |
|-----------|------------|---------|
| Floor < 0 (basement) | 75% | 0.75× multiplier |
| Floor = 0 (ground) | 100% | No reduction |
| Floor = 1 | 100% | No reduction |
| Floor = 2 | 100% | No reduction |
| Floor > 2 | 50% | 0.5× multiplier |

**Important:** Floor-level rules do NOT apply to outdoor control areas.

---

## 7. Factor Deep Dives

### 7.1 Sprinkler Factor Details

The 2× sprinkler increase is one of the most impactful factors, but it has strict requirements.

**What Qualifies:**
- NFPA 13 automatic sprinkler system
- Must protect the **entire building**
- System must be in working order and maintained

**What Does NOT Qualify:**
- NFPA 13R (residential sprinklers)
- NFPA 13D (one/two-family dwellings)
- Partial sprinkler coverage
- Standpipe systems without sprinklers
- Fire suppression systems other than sprinklers (unless specifically approved)

**Critical:** The building must be protected **throughout**. A building with sprinklers only in the basement does NOT qualify for the 2× increase on any floor (strict interpretation).

**Basement-Only Sprinklers:**

Some buildings have sprinklers only in basement levels. In these cases:
- Basement floors may or may not qualify (consult your AHJ)
- Upper floors do NOT get the sprinkler increase
- The strict interpretation is that NO floors qualify since it's not "throughout"

| Condition | Factor |
|-----------|--------|
| Fully sprinklered (NFPA 13) | **2×** (100% increase) |
| Not sprinklered | **1×** (no increase) |
| Partially sprinklered | **1×** (no increase - must be "throughout") |
| NFPA 13R or 13D system | **1×** (residential systems don't qualify) |

### 7.2 Floor Level Factor Details

Floor level reductions reflect increased difficulty of firefighting at height and depth.

**Why Reductions Apply:**
- Higher floors: Harder to reach with fire apparatus
- Basement floors: Limited ventilation, difficult egress
- Deep basements: Especially dangerous - prohibited beyond 2 levels

**Ground Plane Determination:**

The "ground floor" (floor level 1) is determined by:
- The level of exit discharge
- Usually matches exterior grade
- In sloped sites, the AHJ determines the reference level

**Above Grade:**

| Floor | Percentage | Factor |
|-------|------------|--------|
| 1 (Ground) | 100% | 1.00 |
| 2 | 75% | 0.75 |
| 3 | 50% | 0.50 |
| 4-6 | 12.5% | 0.125 |
| 7-9 | 5% | 0.05 |
| 10+ | 5% | 0.05 |

**Below Grade (Basements):**

| Floor | Percentage | Factor |
|-------|------------|--------|
| 1 below | 75% | 0.75 |
| 2 below | 50% | 0.50 |
| 3+ below | **Not Permitted** | — |

**Mixed-Level Control Areas:**

If a control area spans multiple floors:
- Use the floor with the most restrictive (lowest) percentage
- Or calculate based on where materials are actually located

**Important:** Outdoor control areas do NOT get floor level reductions (they use separate outdoor tables).

### 7.3 Approved Storage Factor Details

The 2× increase for approved storage recognizes that proper cabinets contain spills and slow fire spread.

**For Liquids - Approved Flammable Storage Cabinets:**
- Must meet NFPA 30 or be listed to UL 1275
- Maximum 60 gallons per cabinet
- Maximum 3 cabinets per fire area (unless separated)
- Self-closing doors required

**Approved Cabinet Requirements (IFC 5003.9.10):**
- Double-walled construction with 1.5" air space
- Liquid-tight door sill (2" minimum)
- Self-closing, self-latching doors
- Labeled "HAZARDOUS—KEEP FIRE AWAY"

**For Gases - Gas Cabinets:**
- Exhausted enclosures with dedicated ventilation
- Fail-safe exhaust system
- Gas detection recommended

**For Cylinders - Gas Rooms:**
- Separated 1-hour construction
- Dedicated exhaust ventilation
- Sprinkler protection within the room

**Combination:**

The sprinkler and approved storage factors can **both** apply:
```
Baseline 30 gal × 2 (sprinkler) × 2 (cabinet) = 120 gallons
```

| Condition | Factor |
|-----------|--------|
| In approved cabinet | **2×** (100% increase) |
| Not in approved cabinet | **1×** (no increase) |

---

## 8. Hazard Class Mapping System

### 8.1 Overview

The hazard class mapping system connects **chemicals** (via their "bands") to **fire code hazard classes**. This is how the system knows which hazard class limits apply to which chemicals.

### 8.2 Chemical Bands

A **band** is a classification assigned to chemicals based on their hazard properties:

```javascript
// Example bands
{ id: '5c8be8bf7c4d9b28a4500143', name: 'Flammable Liquid Cat 1' }
{ id: '5c8be8bf7c4d9b28a4500144', name: 'Flammable Liquid Cat 2' }
{ id: '5c8be8bf7c4d9b28a4500145', name: 'Flammable Liquid Cat 3' }
{ id: '5c8be8bf7c4d9b28a4500146', name: 'Liquefied Gas' }
```

### 8.3 Hazard Class Mappings

Mappings define which bands belong to which hazard class using boolean logic:

```javascript
{
  name: 'Flammable Liquid: IA',
  must: [                        // ALL of these must be present
    { id: '5c8be8bf7c4d9b28a4500143', displayName: 'Flammable Liquid Cat 1' }
  ],
  should: [],                    // At least one of these (optional)
  mustNot: []                    // NONE of these can be present
}
```

### 8.4 Mapping Logic

```
Chemical matches hazard class IF:
  ALL "must" bands are present
  AND
  (no "should" bands defined OR at least one "should" band is present)
  AND
  NONE of "mustNot" bands are present
```

### 8.5 Complex Mapping Example

**Flammable Gas vs Flammable Gas Liquefied:**

```javascript
// Flammable Gas (non-liquefied only)
{
  name: 'Flammable Gas',
  must: [
    { id: '...', displayName: 'Flammable Gas Category 1' }
  ],
  should: [],
  mustNot: [
    { id: '5c8be8bf7c4d9b28a4500146', displayName: 'Liquefied Gas' }  // Exclude liquefied
  ]
}

// Flammable Gas Liquefied (liquefied only)
{
  name: 'Flammable Gas Liquefied',
  must: [
    { id: '...', displayName: 'Flammable Gas Category 1' },
    { id: '5c8be8bf7c4d9b28a4500146', displayName: 'Liquefied Gas' }  // Require liquefied
  ],
  should: [],
  mustNot: []
}
```

### 8.6 Elasticsearch Query Generation

Hazard class mappings are converted to Elasticsearch queries:

```javascript
// Generated Elasticsearch filter for 'Flammable Gas'
{
  bool: {
    must: [
      { terms: { 'family.bands._id': ['flammable-gas-cat-1-id'] } }
    ],
    should: [],
    must_not: [
      { terms: { 'family.bands._id': ['liquefied-gas-id'] } }
    ]
  }
}
```

---

## 9. Storage vs. Use Conditions

### 9.1 The Three Usage Categories

The fire code distinguishes between three conditions with **different MAQ limits**:

| Category | Description | Vapor Exposure |
|----------|-------------|----------------|
| **Storage** | Sealed containers, not being used | None |
| **Use-Closed** | In use but contained (closed systems, fume hoods) | Minimal |
| **Use-Open** | Open to atmosphere (pouring, mixing, open containers) | Maximum |

### 9.2 Why This Matters

Use-open conditions present higher risk:
- Vapors can accumulate to flammable concentrations
- Spills spread faster from open containers
- Ignition sources more likely to contact vapors

### 9.3 Typical MAQ Ratios

| Material | Storage | Use-Closed | Use-Open | Ratio |
|----------|---------|------------|----------|-------|
| Flammable Liquid IA | 30 gal | 30 gal | 10 gal | 3:1 |
| Flammable Liquid IB/IC | 120 gal | 120 gal | 30 gal | 4:1 |
| Corrosive Liquid | 500 gal | 500 gal | 100 gal | 5:1 |
| Toxic Solid | 500 lb | 500 lb | 125 lb | 4:1 |

### 9.4 The Aggregate Rule

**Critical IFC Requirement:**

> "The aggregate quantity in use and in storage shall not exceed the quantity listed for storage."

This means:
- You cannot have 120 gallons in storage AND 30 gallons in use
- The total of both must stay under the storage MAQ
- Use quantities come out of the storage allowance

**Example:**
```
Storage MAQ for Flammable IB: 120 gallons

Scenario A: 100 gal stored + 15 gal in use = 115 gal total → COMPLIANT
Scenario B: 120 gal stored + 30 gal in use = 150 gal total → VIOLATION
```

### 9.5 Determining Use Condition

When inspecting, look for:

**Storage indicators:**
- Containers in original sealed packaging
- Materials on shelves/racks not being accessed
- Stockroom or warehouse areas

**Use-Closed indicators:**
- Connected to closed processing equipment
- Inside fume hoods with sash down
- Sealed transfer systems
- Safety cans with spring-loaded caps

**Use-Open indicators:**
- Open beakers, flasks, or containers
- Pouring/dispensing operations
- Active mixing or reactions
- Containers with removed caps

---

## 10. Open/Closed MAQ Calculations (HMIS)

### 10.1 What is HMIS?

**HMIS (Hazardous Materials Inventory Statement)** is a reporting format that requires additional MAQ values:

- **Storage MAQ**: Standard MAQ limit (calculated as described above)
- **Use-Closed MAQ**: Limit for containers that are closed during use
- **Use-Open MAQ**: Limit for containers that are open during use

### 10.2 Open vs Closed Containers

| Container State | Description | Example |
|-----------------|-------------|---------|
| **Closed** | Container sealed during use | Pressurized tanks, sealed drums |
| **Open** | Container open during use | Open beakers, dip tanks, spray operations |

**Important:** Open container limits are typically much lower than closed container limits due to increased vapor release and fire risk.

### 10.3 Open/Closed MAQ Formula

For HMIS-enabled fire codes:

```
Closed MAQ = Base Closed Value × Sprinkler Factor × (Approved Storage Factor if applicable)
Open MAQ = Base Open Value × Sprinkler Factor × (Approved Storage Factor if applicable)
```

### 10.4 Open/Closed Limits by Hazard Class

Below are the base open/closed limits (before factors are applied):

#### Physical Hazards - Flammable Liquids

| Hazard Class | Closed (gal) | Open (gal) | Notes |
|--------------|-------------|-----------|-------|
| Flammable Liquid: IA | 30 | 10 | Most volatile |
| Flammable Liquid: IB, IC | 120 | 30 | Moderate volatility |
| Combustible Liquid: II | 120 | 30 | Lower volatility |
| Combustible Liquid: IIIA | 330 | 80 | Least volatile |

#### Physical Hazards - Flammable Gases

| Hazard Class | Physical State | Closed (ft³) | Open (ft³) |
|--------------|---------------|-------------|-----------|
| Flammable Gas | Gas | 1000 | 1000 |
| Flammable Gas Liquefied | Liquid | 150 | — |

#### Physical Hazards - Oxidizers

| Hazard Class | State | Closed | Open | Notes |
|--------------|-------|--------|------|-------|
| Oxidizer: 1 | S/L | NL (if sprinkled) or 4000 | NL or 1000 | |
| Oxidizer: 2 | S/L | 250 | 50 | |
| Oxidizer: 3 | S/L | 2 | 2 | |
| Oxidizer: 4 | S/L | 0.25 (if sprinkled) | 0.25 | Requires sprinklers |

#### Physical Hazards - Explosives

| Hazard Class | Closed (lbs) | Open (lbs) | Notes |
|--------------|-------------|-----------|-------|
| Explosive: Division 1.1, 1.2, 1.5 | 0.25 | 0.25 | Requires sprinklers |
| Explosive: Division 1.3 | 1 | 1 | Requires sprinklers |
| Explosive: Division 1.4 | 50 | — | Requires sprinklers |
| Explosive: Division 1.4G, 1.6 | — | — | No limit specified |

#### Health Hazards - Toxics

| Hazard Class | State | Closed | Open | Units |
|--------------|-------|--------|------|-------|
| Toxic | Solid | 500 | 125 | lbs |
| Toxic | Liquid | 500 | 125 | gal |
| Toxic | Gas | 810 | 810 | ft³ |
| Toxic Liquefied | Gas | 150 | 150 | gal |
| Highly Toxic | Solid | 10 | 3 | lbs |
| Highly Toxic | Liquid | 10 | 3 | gal |
| Highly Toxic | Gas | 20 (requires sprinklers) | 20 | ft³ |
| Highly Toxic Liquefied | Gas | 4 (requires sprinklers) | 4 | gal |

#### Health Hazards - Corrosives

| Hazard Class | State | Closed | Open | Units |
|--------------|-------|--------|------|-------|
| Corrosive | Solid | 5000 | 1000 | lbs |
| Corrosive | Liquid | 500 | 100 | gal |
| Corrosive | Gas | 810 | 810 | ft³ |
| Corrosive Liquefied | Solid | 5000 | 1000 | lbs |
| Corrosive Liquefied | Liquid | 500 | 100 | gal |
| Corrosive Liquefied | Gas | 150 | 150 | ft³ |

### 10.5 HMIS Report Structure

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  CFC 2016 Hazard    │ Class   │ In Storage │ Storage │ Closed │ Open   │ Units │
├─────────────────────┼─────────┼────────────┼─────────┼────────┼────────┼───────┤
│  Flammable Liquid   │ IA      │ 5.23       │ 60      │ 60     │ 20     │ gal   │
│  Flammable Liquid   │ IB, IC  │ 12.50      │ 240     │ 240    │ 60     │ gal   │
│  Oxidizer           │ 1       │ 0.00       │ NL      │ NL     │ NL     │ lbs   │
│  Toxic              │         │ 0.00       │ 1000    │ 1000   │ 250    │ lbs   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 11. Special Cases & Edge Conditions

### 11.1 Outdoor Control Areas

Outdoor areas use different MAQ tables (5003.1.1(3) and (4)) with generally higher limits.

**Key Differences:**
- No floor level reductions
- Higher baseline quantities
- Different separation requirements

**What Qualifies as "Outdoor":**
- Roofless areas
- Open-sided structures (≥75% open)
- Covered but well-ventilated areas (per AHJ)

### 11.2 Report As Liquid (Gas → Liquid Conversion)

Some gaseous materials are tracked as liquids for MAQ purposes. When `reportAsLiquid: true` is set on a gas baseline:

**What Happens:**
1. Gas quantities aggregate to the **liquid** bucket in Elasticsearch
2. MAQ limit uses the **liquid baseline** instead of gas baseline
3. Display shows "Report as Liquid" as first MAQ factor

**Example:**
```javascript
// Fire code definition
{
  name: 'Some Gas',
  liquid: { baseline: 50, units: 'gal' },
  gas: { baseline: 200, units: 'ft3', reportAsLiquid: true }
}

// For gas physical state, calculation uses:
// - Liquid baseline (50 gal) instead of gas baseline (200 ft³)
// - Units become gallons instead of cubic feet
```

### 11.3 Liquefied Gases

Gases that are liquid under pressure require special consideration.

**Examples:**
- Liquefied Petroleum Gas (LPG)
- Liquefied Natural Gas (LNG)
- Refrigerants

**MAQ Determination:**
- Use the gas MAQ limits
- Quantity typically expressed in water capacity (gallons) or gas equivalent (cubic feet)
- Conversion: 1 gallon liquid ≈ 270 cubic feet gas (varies by material)

For health hazards (Toxic, Highly Toxic, Corrosive), gases stored in liquefied form have separate hazard classes:

| Base Class | Liquefied Class |
|------------|-----------------|
| Toxic | Toxic Liquefied |
| Highly Toxic | Highly Toxic Liquefied |
| Corrosive | Corrosive Liquefied |
| Flammable Gas | Flammable Gas Liquefied |
| Oxidizing Gas | Oxidizing Gas Liquefied |

**In the UI:** Health hazards table shows a fourth column "Liquefied" that displays values from the corresponding liquefied hazard class.

### 11.4 Exempt Materials

Certain materials may be exempt from MAQ calculations:

**Consumer Products:**
- In original sealed packaging
- Quantities typical of consumer use
- Not for resale distribution

**Pharmaceuticals:**
- FDA-approved drugs
- In manufacturer packaging
- Under proper storage conditions

**Specific Exemptions:**
- Alcoholic beverages in retail
- Cosmetics and personal care products
- Certain laboratory chemicals under specific conditions

**Always verify exemptions with your AHJ.**

### 11.5 Control Area Exemptions

When a control area is marked exempt (`exemption.isExempt: true`):

- **All limits** become `NL` (No Limit)
- **Status** is forced to `EXEMPT`
- No compliance checking occurs

**Possible Exemption Reasons:**
- H-occupancy classification already applied
- Standalone building with no exposure risk
- Specialized permitted facility
- Historical exemption (grandfathered)

**Documentation Required:**
- Written exemption with basis
- AHJ approval
- Periodic review

### 11.6 Container-Level Exemptions

Individual containers can be exempted from MAQ calculations:

- Container has non-empty `exemption` field
- Container is **excluded** from aggregation
- Does not count toward actual quantities
- Exemption must be documented on container record

### 11.7 Inactive Containers

Containers with `active: false`:

- **Excluded** from MAQ calculations
- Represent disposed or removed containers

### 11.8 Override Rules

When a rule has `operator: 'OVERRIDE'`:

- **Replaces entire calculation** (baseline and all factors)
- Processing stops immediately
- Common use: Special conditions that negate normal limits

### 11.9 No Limit (NL) Handling

When a hazard class has `isNoLimit: true`:

- `displayLimit` shows "NL"
- `limit` is `null`
- **Always compliant** (no limit to exceed)

**Example:** Combustible Liquid Class IIIB often has NL for storage because of its high flash point (200°F+).

### 11.10 Not Applicable (N/A) Handling

When a hazard class has `isNotApplicable: true` for a physical state:

- `displayLimit` shows "N/A"
- Physical state not tracked for this hazard class
- Example: Solid state for Flammable Liquid (liquids can't be solid)
- Example: Flammable gases have N/A for solid and liquid states

### 11.11 Zero Limits

Some hazard classes have zero limits without sprinklers:

```javascript
// Example: Pyrophoric materials
if (physicalState === 'solid') {
  closedLimit = fireSuppressionCoverage ? 1 : 0;  // 0 if no sprinklers
  openLimit = 0;  // Always 0 for open use
}
```

**Meaning:** You cannot store these materials without sprinkler coverage.

### 11.12 Combination of Materials

When multiple materials in the same hazard class are present:

**Rule:** Sum all quantities and compare to the MAQ for that class.

```
Acetone (Flammable IB):     50 gallons
Toluene (Flammable IB):     75 gallons
Ethanol (Flammable IB):     30 gallons
────────────────────────────────────────
Total Flammable IB:         155 gallons
MAQ (with 2× sprinkler):    240 gallons
Status:                     COMPLIANT
```

---

## 12. Compliance Status Determination

### 12.1 Status Values

| Status | Condition | UI Display | Action Required |
|--------|-----------|-----------|-----------------|
| `COMPLIANT` | Actual < 80% of limit | Gray text | None - within limits |
| `NEAR_THRESHOLD` | 80% ≤ Actual < 100% of limit | Warning icon | Warning - reduce quantities or prepare for H-occupancy |
| `OVER_THRESHOLD` | Actual ≥ limit | Red text + Error icon | Violation - immediate reduction required or reclassification |
| `EXEMPT` | Control area is exempted | Gray text | Document exemption basis |
| `INCOMPLETE` | No control areas in building | — | — |

**Note:** The 80% "near threshold" warning is typically an institutional policy, not a code requirement. However, it's an excellent best practice.

### 12.2 Status Hierarchy

Status rolls up from physical state → hazard class → control area → building:

```
Building Status = WORST of all Control Area statuses
Control Area Status = WORST of all Hazard Class statuses
Hazard Class Status = WORST of Solid, Liquid, Gas statuses
Physical State Status = Compare actual vs limit
```

**Priority:** `OVER_THRESHOLD` > `NEAR_THRESHOLD` > `COMPLIANT`

### 12.3 Status Calculation Logic

```javascript
function getComplianceStatus(limit, actual) {
  if (limit === null) return 'COMPLIANT';      // NL = always compliant
  if (limit === 0 && actual === 0) return 'COMPLIANT';
  if (actual >= limit) return 'OVER_THRESHOLD';
  if (actual >= limit * 0.8) return 'NEAR_THRESHOLD';  // 80% threshold
  return 'COMPLIANT';
}
```

### 12.4 Control Area vs. Building Compliance

**Control Area Level:**
- Each control area is evaluated independently
- One non-compliant control area doesn't affect others

**Building Level:**
- If ANY control area exceeds MAQ, the building needs H-occupancy features
- Building status = worst control area status
- "One bad apple" principle applies

---

## 13. Data Sources & Integration

### 13.1 Data Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              DATA SOURCES                                        │
├───────────────────────┬────────────────────────┬────────────────────────────────┤
│    MongoDB            │    Elasticsearch       │    External Services           │
│    (Configuration)    │    (Quantities)        │    (Building/Facilities)       │
├───────────────────────┼────────────────────────┼────────────────────────────────┤
│  • fireCode           │  • container index     │  • Relationship Service        │
│    - occupancies      │    - normalizedSize    │    - building data             │
│    - hazardClasses    │      .gram             │    - floors                    │
│    - rules            │      .liter            │    - rooms                     │
│    - mappings         │    - family.formAtNtp  │    - fire suppression          │
│                       │    - family.bands._id  │                                │
│  • controlArea        │    - location.roomId   │                                │
│    - occupancy        │    - exemption         │                                │
│    - approvedStorage  │    - active            │                                │
│    - exemption        │                        │                                │
│    - override         │                        │                                │
└───────────────────────┴────────────────────────┴────────────────────────────────┘
```

### 13.2 Unit Conversions

Container quantities are stored in metric units and converted for display:

| Source Unit | Display Unit | Conversion Factor |
|-------------|-------------|-------------------|
| Grams | Pounds (lbs) | × 0.0022046226 |
| Liters | Gallons (gal) | × 0.2641720524 |
| Liters | Cubic Feet (ft³) | × 0.0353146667 |

Additional conversions:

| Conversion | Factor |
|------------|--------|
| Kilograms → Pounds | × 2.20462 |
| Cubic Meters → Cubic Feet | × 35.3147 |
| Gallons → Pounds (water) | × 8.34 |
| Gallons → Pounds (typical liquid) | × 10 (code default) |

### 13.3 Data Flow

```
Chemical Inventory System
         │
         │ (Kafka events)
         ▼
    Elasticsearch
    (container index)
         │
         │ (Aggregation query)
         ▼
    MAQ Report Service
         │
         ├──── MongoDB: controlArea (configuration)
         │
         ├──── MongoDB: fireCode (rules & limits)
         │
         └──── External: Building Service (structure)
         │
         ▼
    GraphQL API
         │
         ▼
    React Client (UI)
```

---

## 14. Practical Scenarios & Worked Examples

### Scenario 1: Building Gets New Sprinkler System

**Before:** Building fire suppression = `NONE`
**After:** Building fire suppression = `FULL`

**Impact:**
- All MAQ limits effectively **double** (2× sprinkler factor)
- Control areas that were `OVER_THRESHOLD` may become `COMPLIANT`
- Open/Closed MAQ limits also double for HMIS reports

**Example:**
```
Flammable Liquid IA:
  Before: 30 gal (baseline × 1)
  After:  60 gal (baseline × 2)
```

### Scenario 2: Moving Lab to Basement

**Before:** Control area on floor 0 (ground)
**After:** Control area on floor -1 (basement)

**Impact:**
- Floor-level factor applies (typically 0.75×)
- MAQ limits **decrease** by 25%
- BUT: If building has `BASEMENT_ONLY` suppression, basement is now sprinklered!

**Example (BASEMENT_ONLY suppression):**
```
Before (ground floor, no sprinklers):
  30 × 1.0 (floor) × 1 (no sprinkler) = 30 gal

After (basement, sprinklered):
  30 × 0.75 (floor) × 2 (sprinkler) = 45 gal  ← Actually INCREASED!
```

### Scenario 3: Adding Approved Storage

**Before:** No approved storage
**After:** Approved storage for "Flammable Liquid IA, liquid"

**Impact:**
- 2× multiplier for that specific hazard class + physical state
- Only affects the designated hazard class and state
- Other hazard classes unchanged

### Scenario 4: Control Area Exemption

**Before:** Control area with various limits
**After:** Control area marked exempt (reason: "Research exemption per agreement with Fire Marshal")

**Impact:**
- All limits become `NL` (No Limit)
- Status becomes `EXEMPT`
- No compliance checking
- Documentation preserved for audit

### Scenario 5: Container-Level Exemption

**Situation:** Large drum of flammable liquid has approved exemption

**Impact:**
- That specific container excluded from MAQ aggregation
- Other containers still count
- Exemption must be documented on container record

### Example 6: Basic Calculation

**Scenario:** A chemistry teaching lab on the ground floor of a fully sprinklered building. No approved storage cabinets.

**Materials:**
- Acetone (Flammable Liquid IB): 20 gallons
- Hydrochloric Acid (Corrosive Liquid): 15 gallons

**Calculations:**

*Flammable Liquid IB:*
```
Baseline:           120 gal
× Sprinkler:        2
× Floor Level:      1.0 (ground)
× Approved Storage: 1
────────────────────
MAQ:                240 gallons
Actual:             20 gallons
Status:             COMPLIANT (8% of limit)
```

*Corrosive Liquid:*
```
Baseline:           500 gal
× Sprinkler:        2
× Floor Level:      1.0 (ground)
× Approved Storage: 1
────────────────────
MAQ:                1,000 gallons
Actual:             15 gallons
Status:             COMPLIANT (1.5% of limit)
```

### Example 7: Basement with Partial Sprinklers

**Scenario:** Chemical storage in a first basement level. Building has sprinklers in basement only.

**Materials:**
- Flammable Liquid IA: 25 gallons in approved cabinet

**Strict Interpretation:**
```
Baseline:           30 gal
× Sprinkler:        1 (not "throughout")
× Floor Level:      0.75 (1st basement)
× Approved Storage: 2 (in cabinet)
────────────────────
MAQ:                45 gallons
Actual:             25 gallons
Status:             COMPLIANT (56% of limit)
```

**Alternative Interpretation (if AHJ allows basement sprinkler credit):**
```
Baseline:           30 gal
× Sprinkler:        2 (basement is sprinklered)
× Floor Level:      0.75 (1st basement)
× Approved Storage: 2 (in cabinet)
────────────────────
MAQ:                90 gallons
Actual:             25 gallons
Status:             COMPLIANT (28% of limit)
```

**Inspector Note:** Always confirm with your AHJ which interpretation applies.

### Example 8: High Floor with Reduced MAQ

**Scenario:** Research lab on the 5th floor of a fully sprinklered building.

**Materials:**
- Oxidizer Class 2 (solid): 100 pounds

**Calculation:**
```
Baseline:           250 lb
× Sprinkler:        2
× Floor Level:      0.125 (5th floor = 12.5%)
× Approved Storage: 1
────────────────────
MAQ:                62.5 pounds
Actual:             100 pounds
Status:             OVER THRESHOLD (160% of limit) - VIOLATION
```

**Resolution Options:**
1. Reduce quantity below 62.5 lb
2. Move materials to a lower floor
3. Create additional control areas on the floor
4. Accept H-occupancy classification for that area

### Example 9: Multiple Hazard Classes

**Scenario:** Industrial warehouse with various materials. Ground floor, sprinklered, no approved storage.

**Materials:**
- Flammable Liquid IB: 150 gallons
- Flammable Liquid IA: 40 gallons
- Combustible Liquid II: 200 gallons
- Corrosive Liquid: 100 gallons

**Calculations:**

| Hazard Class | Baseline | × Factors | MAQ | Actual | Status |
|--------------|----------|-----------|-----|--------|--------|
| Flammable IB | 120 gal | × 2 × 1 × 1 | 240 gal | 150 gal | OK (63%) |
| Flammable IA | 30 gal | × 2 × 1 × 1 | 60 gal | 40 gal | OK (67%) |
| Combustible II | 120 gal | × 2 × 1 × 1 | 240 gal | 200 gal | Near (83%) |
| Corrosive | 500 gal | × 2 × 1 × 1 | 1000 gal | 100 gal | OK (10%) |

**Special Check - Flammable Combination:**

IFC has a special rule: Combined Class IA + IB + IC cannot exceed the IB/IC limit.

```
Class IA: 40 gallons
Class IB: 150 gallons
Combined: 190 gallons
Limit:    240 gallons (IB/IC combined limit)
Status:   COMPLIANT
```

But also check that IA doesn't exceed its individual limit within the combined total.

### Example 10: Storage vs. Use-Open

**Scenario:** Research lab actively using chemicals. Ground floor, sprinklered.

**Materials:**
- Flammable Liquid IA: 10 gallons stored, 5 gallons in open use

**Storage MAQ Check:**
```
Baseline:           30 gal
× Sprinkler:        2
────────────────────
Storage MAQ:        60 gallons
Total (stored+use): 15 gallons
Status:             COMPLIANT for storage
```

**Use-Open MAQ Check:**
```
Baseline:           10 gal (use-open baseline)
× Sprinkler:        2
────────────────────
Use-Open MAQ:       20 gallons
In Open Use:        5 gallons
Status:             COMPLIANT for use-open
```

**Both checks must pass.**

### Example 11: Worked Example with Full Factor Chain

**Scenario:** Flammable Liquid IA in a sprinklered basement office

```
Given:
  - Fire Code: CFC 2022
  - Occupancy: B (Business)
  - Hazard Class: Flammable Liquid IA
  - Physical State: Liquid
  - Building Fire Suppression: FULL
  - Floor: -1 (basement)
  - Approved Storage: None
  - Outdoor: No

Calculation:
┌──────────────────────────────────────────────────────────────────────────┐
│  Factor            │  Source                          │  Value          │
├────────────────────┼──────────────────────────────────┼─────────────────┤
│  Baseline          │  CFC 2022, Occupancy B, FL:IA    │  30 gal         │
│  × Sprinkler       │  FULL coverage = sprinklered     │  × 2            │
│  × Floor Level     │  Floor -1 (basement) rule        │  × 0.75         │
├────────────────────┼──────────────────────────────────┼─────────────────┤
│  FINAL MAQ         │  30 × 2 × 0.75                   │  = 45 gal       │
└──────────────────────────────────────────────────────────────────────────┘

Display in UI:
  MAQ: 45 gal
  Tooltip: "Baseline: 30, × Sprinkler: 2, × Floor Level: 0.75"
```

---

## 15. Common Mistakes to Avoid

### Mistake 1: Applying Sprinkler Factor to Partial Systems

**Wrong:** Building has sprinklers in common areas only → Apply 2× factor
**Right:** Must be sprinklered **throughout** per NFPA 13 → Apply 1× factor

### Mistake 2: Forgetting Floor Level Reductions

**Wrong:** 5th floor lab has same MAQ as ground floor
**Right:** 5th floor = 12.5% of ground floor MAQ (0.125 factor)

### Mistake 3: Ignoring Use-Open Limits

**Wrong:** 50 gallons of Flammable IA is under the 60-gallon storage MAQ, so it's compliant
**Right:** If 50 gallons is in open use, the use-open MAQ of 20 gallons is exceeded

### Mistake 4: Counting Materials Twice

**Wrong:** Adding approved-cabinet materials to general storage totals
**Right:** Materials in approved cabinets have their own MAQ calculation

### Mistake 5: Misidentifying Hazard Class

**Wrong:** Classifying diesel fuel as Flammable Liquid
**Right:** Diesel fuel (flash point >100°F) is Combustible Liquid Class II

### Mistake 6: Ignoring Aggregate Rule

**Wrong:** 120 gallons stored + 30 gallons in use = OK (each under its limit)
**Right:** Total 150 gallons exceeds the 120-gallon storage MAQ aggregate limit

### Mistake 7: Not Verifying Approved Storage

**Wrong:** Assuming any metal cabinet is "approved"
**Right:** Must be listed/labeled flammable storage cabinet meeting NFPA 30

### Mistake 8: Mixing Indoor and Outdoor Tables

**Wrong:** Using outdoor MAQ values for a covered loading dock
**Right:** Unless ≥75% open-sided, use indoor tables

---

## 16. Quick Reference Tables

### 16.1 Sprinkler Multiplier Quick Reference

| Building Type | Floor | Sprinklered? | Multiplier |
|--------------|-------|--------------|------------|
| FULL | Any | Yes | 2× |
| BASEMENT_ONLY | < 0 | Yes | 2× |
| BASEMENT_ONLY | ≥ 0 | No | 1× |
| NONE | Any | No | 1× |

### 16.2 Floor Level Multiplier Quick Reference

| Floor | % of MAQ | Factor | Max Control Areas |
|-------|----------|--------|-------------------|
| 10+ | 5% | 0.05 | 1 |
| 7-9 | 5% | 0.05 | 2 |
| 6 | 12.5% | 0.125 | 2 |
| 5 | 12.5% | 0.125 | 2 |
| 4 | 12.5% | 0.125 | 2 |
| 3 | 50% | 0.50 | 2 |
| 2 | 75% | 0.75 | 3 |
| 1 (Ground) | 100% | 1.00 | 4 |
| B1 | 75% | 0.75 | 3 |
| B2 | 50% | 0.50 | 2 |
| B3+ | Not Permitted | — | — |

### 16.3 Common Hazard Class Limits (Occupancy B, Baseline - Unsprinklered)

#### Physical Hazards

| Hazard Class | Solid (lbs) | Liquid (gal) | Gas (ft³) |
|--------------|------------|-------------|----------|
| Flammable Liquid: IA | N/A | 30 | N/A |
| Flammable Liquid: IB | N/A | 120 | N/A |
| Flammable Liquid: IC | N/A | 120 | N/A |
| Combustible Liquid: II | — | 120 | — |
| Combustible Liquid: IIIA | — | 330 | — |
| Combustible Liquid: IIIB | — | NL | — |
| Flammable Gas | N/A | N/A | 1,000 |
| Flammable Solid | 125 | N/A | N/A |
| Oxidizer: 1 | 4,000 | 4,000 | NL |
| Oxidizer: 2 | 250 | 250 | — |
| Oxidizer: 3 | 10 | 10 | — |
| Oxidizer: 4 | 1* | 1* | — |
| Organic Peroxide UD | 1* | 1* | — |
| Pyrophoric | 4* | 4* | 50* |
| Unstable Reactive 3 | 5 | 5 | — |
| Unstable Reactive 4 | 1* | 1* | — |
| Water Reactive 2 | 50 | 50 | — |
| Water Reactive 3 | 5 | 5 | — |

*Sprinkler protection required for any quantity

#### Health Hazards

| Hazard Class | Solid (lbs) | Liquid (gal) | Gas (ft³) |
|--------------|------------|-------------|----------|
| Corrosive | 5,000 | 500 | 810 |
| Toxic | 500 | 500 | 810 |
| Highly Toxic | 10 | 10 | 20* |

*Sprinkler protection required for any quantity

*Note: These are typical baselines before sprinkler/floor/storage factors. Actual limits vary by fire code and occupancy.*

### 16.4 Status Thresholds

| Percentage of Limit | Status |
|--------------------|--------|
| 0% - 79% | COMPLIANT |
| 80% - 99% | NEAR_THRESHOLD |
| 100%+ | OVER_THRESHOLD |

### 16.5 Unit Conversions

| Conversion | Factor |
|------------|--------|
| Grams → Pounds | × 0.00220462 |
| Kilograms → Pounds | × 2.20462 |
| Liters → Gallons | × 0.264172 |
| Liters → Cubic Feet | × 0.0353147 |
| Cubic Meters → Cubic Feet | × 35.3147 |
| Gallons → Pounds (water) | × 8.34 |
| Gallons → Pounds (typical liquid) | × 10 (code default) |

### 16.6 What Can Be Changed (Quick Reference)

| Variable | Who Can Change | Where |
|----------|---------------|-------|
| Fire code baselines | Fire Code Admin | Fire Code settings |
| Fire code rules | Fire Code Admin | Fire Code settings |
| Hazard class mappings | Fire Code Admin | Fire Code settings |
| Control area occupancy | Control Area Admin | Control Area settings |
| Approved storage | Control Area Admin | Control Area settings |
| Fire suppression override | Control Area Admin | Control Area settings |
| Exemption status | Control Area Admin | Control Area settings |
| Room assignments | Control Area Admin | Control Area settings |
| Building sprinkler type | Facilities Management | External system |
| Chemical quantities | Lab Personnel | Inventory system |

---

## 17. Manual Calculation Guide

This section provides step-by-step instructions for manually calculating an MAQ report from scratch. Follow these steps exactly to reproduce what the system calculates.

### 17.1 Overview: The Complete Calculation Pipeline

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE MAQ CALCULATION PIPELINE                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  PHASE 1: GATHER INPUT DATA                                                      │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  A. Building Data (from facilities)                                        │ │
│  │  B. Fire Code Data (from fire code configuration)                          │ │
│  │  C. Control Area Configuration (from MAQ system)                           │ │
│  │  D. Chemical Container Data (from inventory system)                        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  PHASE 2: MAP CHEMICALS TO HAZARD CLASSES                                       │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  For each container, determine which hazard class(es) it belongs to        │ │
│  │  using the must/should/mustNot band matching logic                         │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  PHASE 3: AGGREGATE ACTUAL QUANTITIES                                           │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  Sum container quantities by hazard class and physical state               │ │
│  │  Convert from metric (grams, liters) to display units (lbs, gal, ft³)      │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  PHASE 4: CALCULATE MAQ LIMITS                                                  │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  For each hazard class + physical state:                                   │ │
│  │    1. Get baseline from fire code                                          │ │
│  │    2. Apply hazard class rules (sprinkler, approved storage, outdoor)      │ │
│  │    3. Apply floor-level rules (from occupancy)                             │ │
│  │    4. Multiply all factors together                                        │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                      │                                           │
│                                      ▼                                           │
│  PHASE 5: DETERMINE COMPLIANCE STATUS                                           │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │  Compare actual vs limit for each hazard class + physical state            │ │
│  │  Roll up status to control area and building level                         │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### 17.2 Phase 1: Gather Input Data

#### Step 1A: Collect Building Data

You need the following from the building/facilities system:

| Field | Example | Where to Find |
|-------|---------|---------------|
| Building ID | `bldg-12345` | Building database |
| Building Name | "Chemistry Building" | Building database |
| Fire Suppression Type | `FULL`, `BASEMENT_ONLY`, or `NONE` | Building database |
| Floors | List of floors with groundPlane values | Building database |
| Rooms | List of rooms with their floor and controlAreaId | Building database |

**Floor groundPlane values:**
- `-2` = Second basement level
- `-1` = First basement level
- `0` = Ground floor
- `1` = First floor above ground
- `2` = Second floor above ground
- etc.

#### Step 1B: Collect Fire Code Data

You need the fire code configuration:

| Field | Example | Where to Find |
|-------|---------|---------------|
| Fire Code Name | "CFC 2022" | Fire Code settings |
| Occupancies | List of occupancy configurations (B, H-2, S, etc.) | Fire Code settings |
| Hazard Class Mappings | Chemical band → hazard class mappings | Fire Code settings |
| HMIS Report Flag | `true` or `false` | Fire Code settings |

**For each occupancy, you need:**
- Floor-level rules (percentage reductions by floor)
- List of hazard classes with:
  - Baseline values for solid, liquid, gas
  - Units (lbs, gal, ft³)
  - Conditional rules (sprinkler, approved storage, outdoor)

#### Step 1C: Collect Control Area Configuration

For each control area in the building:

| Field | Example | Where to Find |
|-------|---------|---------------|
| Control Area ID | `ca-67890` | Control Area settings |
| Control Area Name | "Lab Wing A" | Control Area settings |
| Occupancy | `B` | Control Area settings |
| Is Outdoor | `false` | Control Area settings |
| Approved Storage | `[{hazardClass: "Flammable Gas", physicalStates: ["gas"]}]` | Control Area settings |
| Fire Suppression Override | `null` or `{override: true, value: true}` | Control Area settings |
| Exemption | `{isExempt: false}` | Control Area settings |
| Assigned Rooms | List of room IDs | Building database |

#### Step 1D: Collect Chemical Container Data

For each container in the building:

| Field | Example | Where to Find |
|-------|---------|---------------|
| Container ID | `cont-11111` | Inventory system |
| Room ID | `room-22222` | Inventory system |
| Family Bands | `["5c8be8bd7c4d9b28a44fffa1"]` | Inventory system (family.bands._id) |
| Form at NTP | `liquid` | Inventory system (family.formAtNtp) |
| Normalized Size - Grams | `500` | Inventory system (normalizedSize.gram) |
| Normalized Size - Liters | `0.5` | Inventory system (normalizedSize.liter) |
| Exemption | `""` or `"Compliant Storage"` | Inventory system |
| Active | `true` | Inventory system |

**Important:** Only include containers where:
- `active = true`
- `exemption` is empty or null
- Room is assigned to a control area in this building

---

### 17.3 Phase 2: Map Chemicals to Hazard Classes

For each container, determine which hazard class(es) it belongs to.

#### Step 2A: Get Container's Band IDs

```
Container: Acetone (500g / 0.5L)
  family.bands._id = ["5c8be8bd7c4d9b28a44fffa1"]  // Flammable Liquid: IA
  family.formAtNtp = "liquid"
```

#### Step 2B: Apply Hazard Class Mapping Logic

For each hazard class mapping in the fire code, check if the container matches:

```
Hazard Class Mapping: "Flammable Liquid: IA"
  must:    ["5c8be8bd7c4d9b28a44fffa1"]  // Must have this band
  should:  []                             // At least one of these (if not empty)
  mustNot: []                             // Must NOT have these

MATCHING LOGIC:
  1. ALL bands in "must" must be present in container's bands
  2. IF "should" is not empty, at least ONE band in "should" must be present
  3. NONE of the bands in "mustNot" can be present

CHECK:
  ✅ must["5c8be8bd7c4d9b28a44fffa1"] is in container bands
  ✅ should[] is empty (skip check)
  ✅ mustNot[] is empty (skip check)

RESULT: Container matches "Flammable Liquid: IA"
```

#### Step 2C: Example - Flammable Gas vs Flammable Gas Liquefied

```
Container: Propane (liquefied gas cylinder)
  family.bands._id = ["5c8be8bd7c4d9b28a44fff94", "5c8be8bf7c4d9b28a4500146"]
                      ↑ Flammable Gas            ↑ Liquefied Gas

Mapping 1: "Flammable Gas"
  must:    ["5c8be8bd7c4d9b28a44fff94"]  // Flammable Gas
  mustNot: ["5c8be8bf7c4d9b28a4500146"]  // Liquefied Gas

  CHECK: Container HAS "Liquefied Gas" band which is in mustNot
  RESULT: ❌ Does NOT match "Flammable Gas"

Mapping 2: "Flammable Gas Liquefied"
  must:    ["5c8be8bd7c4d9b28a44fff94", "5c8be8bf7c4d9b28a4500146"]
  mustNot: []

  CHECK: Container has BOTH required bands
  RESULT: ✅ Matches "Flammable Gas Liquefied"
```

---

### 17.4 Phase 3: Aggregate Actual Quantities

#### Step 3A: Group Containers by Control Area, Hazard Class, and Physical State

For each control area:
1. Get all rooms assigned to the control area
2. Get all containers in those rooms
3. For each container:
   - Determine which hazard class(es) it matches
   - Get its physical state from `family.formAtNtp`
   - Add its quantities to the appropriate bucket

#### Step 3B: Sum Quantities in Metric Units

```
Control Area: "Lab Wing A"
  Hazard Class: "Flammable Liquid: IA"
    liquid:
      - Container 1: 500g / 0.5L
      - Container 2: 1000g / 1.0L
      - Container 3: 2000g / 2.0L
      ─────────────────────────────
      Total: 3500g / 3.5L
```

#### Step 3C: Convert to Display Units

Use these conversion factors:

| From | To | Multiply By |
|------|----|-------------|
| Grams | Pounds (lbs) | 0.0022046226 |
| Liters | Gallons (gal) | 0.2641720524 |
| Liters | Cubic Feet (ft³) | 0.0353146667 |

**Which conversion to use:**
- **Solid** → Grams to Pounds (weight)
- **Liquid** → Liters to Gallons (usually) OR Liters to Pounds (for some hazard classes)
- **Gas** → Liters to Cubic Feet (volume)

```
Example: 3.5 liters of Flammable Liquid IA

Liquid units = "gal" (from fire code)
Actual = 3.5 L × 0.2641720524 = 0.9246 gal
```

#### Step 3D: Handle "Report As Liquid" Flag

If a hazard class has `gas.reportAsLiquid: true`:
- Gas containers aggregate into the **liquid** bucket, not gas
- The MAQ limit uses the **liquid** baseline, not gas

```
Hazard Class: "Some Special Gas"
  gas: { baseline: 200, units: 'ft3', reportAsLiquid: true }
  liquid: { baseline: 50, units: 'gal' }

When gas containers are found:
  - Add to liquid bucket (actualVolume in liters)
  - Use liquid baseline (50 gal) for limit calculation
```

---

### 17.5 Phase 4: Calculate MAQ Limits

For each hazard class and physical state, calculate the MAQ limit.

#### Step 4A: Determine Fire Suppression Status

```javascript
function isControlAreaSprinklered(fireSuppressionType, floorAboveGroundPlane) {
  // Check for control area override first
  if (controlArea.fireSuppressionOverride?.override) {
    return controlArea.fireSuppressionOverride.value;
  }

  // Otherwise use building configuration
  if (fireSuppressionType === 'FULL') {
    return true;
  }
  if (fireSuppressionType === 'BASEMENT_ONLY') {
    return floorAboveGroundPlane < 0;  // Only basements are sprinklered
  }
  return false;  // NONE
}
```

#### Step 4B: Get Baseline Value

Look up the baseline from the fire code:

```
Fire Code: CFC 2022
Occupancy: B
Hazard Class: Combustible Liquid: II
Physical State: liquid

Baseline = fireCode.occupancies['B'].hazardClasses['Combustible Liquid: II'].liquid.baseline
         = 120 gal
```

#### Step 4C: Build MAQ Factors List

Start with baseline, then add applicable rules:

```javascript
// Initialize with baseline
maqFactors = [
  { label: 'Baseline', value: 120, operator: '' }
]

// Check each rule in hazard class
for (rule of hazardClass.rules) {
  if (allConditionsMatch(rule.conditions)) {
    maqFactors.push({
      label: rule.conditions.map(c => c.criteria).join(' & '),
      value: rule[physicalState].value,
      operator: rule[physicalState].operator
    });
  }
}

// Add floor-level rules (if not outdoor)
if (!controlArea.isOutdoor) {
  floorFactor = getFloorLevelFactor(occupancy.rules, floorAboveGroundPlane);
  if (floorFactor !== 1.0) {
    maqFactors.push({
      label: 'Floor Level',
      value: floorFactor,
      operator: 'MULTIPLY'
    });
  }
}
```

#### Step 4D: Evaluate Rule Conditions

Each rule has conditions that must ALL be true:

| Condition | How to Evaluate |
|-----------|-----------------|
| `SPRINKLER: true` | Is control area sprinklered? (see Step 4A) |
| `SPRINKLER: false` | Is control area NOT sprinklered? |
| `APPROVED_STORAGE: true` | Is there approved storage for this hazard class + physical state? |
| `OUTDOOR: true` | Is `controlArea.isOutdoor === true`? |
| `BASEMENT: true` | Is `floorAboveGroundPlane < 0`? |
| `SPRINKLER_BASEMENT_ONLY: true` | Is `fireSuppressionType === 'BASEMENT_ONLY'`? |

```
Example: Rule with condition { criteria: 'SPRINKLER', value: 'true' }

Control area on floor 0 (ground) in building with FULL sprinkler coverage
→ isControlAreaSprinklered('FULL', 0) = true
→ Condition value 'true' === sprinkler status true
→ ✅ Condition matches, rule applies
```

#### Step 4E: Get Floor-Level Factor

Look up the floor-level rule from the occupancy:

```
Occupancy B Floor Rules:
  { operator: 'is equal to', appliedToFloors: [1], percentage: 100 }
  { operator: 'is equal to', appliedToFloors: [-1, 2], percentage: 75 }
  { operator: 'is equal to', appliedToFloors: [-2, 3], percentage: 50 }
  { operator: 'is equal to', appliedToFloors: [4, 5, 6], percentage: 12.5 }
  { operator: 'is between', appliedToFloors: [7, 9], percentage: 5 }
  { operator: 'is greater than', appliedToFloors: [9], percentage: 5 }
  { operator: 'is less than', appliedToFloors: [-2], percentage: 0 }

For floor = -1:
  Matches rule { appliedToFloors: [-1, 2], percentage: 75 }
  Floor Level Factor = 75 / 100 = 0.75
```

**Important:** Floor-level rules are SKIPPED if `controlArea.isOutdoor === true`.

#### Step 4F: Calculate Final Limit

Apply operators in order:

```javascript
function calculateLimit(maqFactors) {
  let result = maqFactors[0].value;  // Start with baseline

  for (factor of maqFactors.slice(1)) {
    switch (factor.operator) {
      case 'MULTIPLY':
        result = result * factor.value;
        break;
      case 'EQUALS':
        result = factor.value;  // Replace baseline
        break;
      case 'OVERRIDE':
        return factor;  // Stop processing, return this factor
      case 'ADD':
        result = result + factor.value;
        break;
    }

    // Check for special flags
    if (factor.isNoLimit) {
      return { value: null, displayLimit: 'NL' };  // No Limit
    }
    if (factor.isNotApplicable) {
      return { value: null, displayLimit: 'N/A' };  // Not Applicable
    }
  }

  return { value: result, displayLimit: String(result) };
}
```

#### Step 4G: Complete Calculation Example

```
INPUTS:
  Fire Code: CFC 2022
  Occupancy: B
  Hazard Class: Combustible Liquid: II
  Physical State: liquid
  Building Fire Suppression: FULL
  Floor: -1 (basement)
  Is Outdoor: false
  Approved Storage: none

STEP-BY-STEP:

1. Get Baseline:
   Baseline = 120 gal
   maqFactors = [{ label: 'Baseline', value: 120, operator: '' }]

2. Check Sprinkler Rule:
   Is sprinklered? FULL coverage → YES
   Rule condition: { criteria: 'SPRINKLER', value: 'true' } → MATCHES
   Rule effect: { operator: 'MULTIPLY', value: 2 }
   maqFactors = [
     { label: 'Baseline', value: 120, operator: '' },
     { label: 'Sprinkler', value: 2, operator: 'MULTIPLY' }
   ]

3. Check Approved Storage Rule:
   Has approved storage? NO
   Rule condition: { criteria: 'APPROVED_STORAGE', value: 'true' } → DOES NOT MATCH
   (Skip rule)

4. Check Outdoor Rule:
   Is outdoor? NO
   Rule condition: { criteria: 'OUTDOOR', value: 'true' } → DOES NOT MATCH
   (Skip rule)

5. Get Floor-Level Factor:
   Floor = -1
   Matches rule: { appliedToFloors: [-1, 2], percentage: 75 }
   Floor factor = 0.75
   maqFactors = [
     { label: 'Baseline', value: 120, operator: '' },
     { label: 'Sprinkler', value: 2, operator: 'MULTIPLY' },
     { label: 'Floor Level', value: 0.75, operator: 'MULTIPLY' }
   ]

6. Calculate Final Limit:
   Start: 120
   × Sprinkler (2): 120 × 2 = 240
   × Floor Level (0.75): 240 × 0.75 = 180

RESULT: MAQ Limit = 180 gal

DISPLAY:
  Limit: 180 gal
  Tooltip: "Baseline: 120, × Sprinkler: 2, × Floor Level: 0.75"
```

---

### 17.6 Phase 5: Determine Compliance Status

#### Step 5A: Compare Actual vs Limit for Each Physical State

```javascript
function getComplianceStatus(limit, actual) {
  if (limit === null) {
    return 'COMPLIANT';  // NL = No Limit = always compliant
  }

  if (limit === 0 && actual === 0) {
    return 'COMPLIANT';  // Zero limit with zero quantity = compliant
  }

  if (actual >= limit) {
    return 'OVER_THRESHOLD';
  }

  if (actual >= limit * 0.8) {
    return 'NEAR_THRESHOLD';  // 80% threshold
  }

  return 'COMPLIANT';
}
```

**Thresholds:**
| Percentage | Status |
|------------|--------|
| 0% - 79.99% | COMPLIANT |
| 80% - 99.99% | NEAR_THRESHOLD |
| 100%+ | OVER_THRESHOLD |

#### Step 5B: Roll Up to Hazard Class

```javascript
// Hazard class status = worst of solid, liquid, gas
function getHazardClassStatus(hazardClass) {
  const statuses = [
    hazardClass.solid.status,
    hazardClass.liquid.status,
    hazardClass.gas.status
  ];

  if (statuses.includes('OVER_THRESHOLD')) return 'OVER_THRESHOLD';
  if (statuses.includes('NEAR_THRESHOLD')) return 'NEAR_THRESHOLD';
  return 'COMPLIANT';
}
```

#### Step 5C: Roll Up to Control Area

```javascript
// Control area status = worst of all hazard classes
function getControlAreaStatus(controlArea) {
  if (controlArea.exemption.isExempt) {
    return 'EXEMPT';
  }

  const statuses = controlArea.hazardClasses.map(hc => getHazardClassStatus(hc));

  if (statuses.includes('OVER_THRESHOLD')) return 'OVER_THRESHOLD';
  if (statuses.includes('NEAR_THRESHOLD')) return 'NEAR_THRESHOLD';
  return 'COMPLIANT';
}
```

#### Step 5D: Roll Up to Building

```javascript
// Building status = worst of all control areas
function getBuildingStatus(building) {
  const controlAreaStatuses = building.controlAreas.map(ca => ca.status);

  if (controlAreaStatuses.length === 0) {
    return 'INCOMPLETE';
  }

  if (controlAreaStatuses.includes('OVER_THRESHOLD')) return 'OVER_THRESHOLD';
  if (controlAreaStatuses.includes('NEAR_THRESHOLD')) return 'NEAR_THRESHOLD';
  return 'COMPLIANT';
}
```

---

### 17.7 Complete Manual Calculation Worksheet

Use this worksheet to manually calculate an MAQ report:

```
═══════════════════════════════════════════════════════════════════════════════
                        MAQ MANUAL CALCULATION WORKSHEET
═══════════════════════════════════════════════════════════════════════════════

BUILDING INFORMATION
────────────────────
Building Name: ________________________________
Building ID: __________________________________
Fire Suppression Type: [ ] FULL  [ ] BASEMENT_ONLY  [ ] NONE

CONTROL AREA INFORMATION
────────────────────────
Control Area Name: ____________________________
Control Area ID: ______________________________
Occupancy: ____________________________________
Is Outdoor: [ ] Yes  [ ] No
Floor Above Ground Plane: _____________________
Fire Suppression Override: [ ] None  [ ] Force ON  [ ] Force OFF
Is Exempt: [ ] Yes  [ ] No (if Yes, skip to status = EXEMPT)

Assigned Rooms: _______________________________

Approved Storage:
  [ ] Hazard Class: _____________ Physical States: [ ] Solid [ ] Liquid [ ] Gas
  [ ] Hazard Class: _____________ Physical States: [ ] Solid [ ] Liquid [ ] Gas

═══════════════════════════════════════════════════════════════════════════════

HAZARD CLASS CALCULATION (repeat for each hazard class)
────────────────────────────────────────────────────────
Hazard Class Name: ____________________________
Is Health Hazard: [ ] Yes  [ ] No

SOLID:
  Baseline from fire code:         __________ lbs
  Is N/A (Not Applicable)?         [ ] Yes → Skip to status N/A
  Is NL (No Limit)?                [ ] Yes → Skip to status NL

  Applicable Rules:
  [ ] Sprinkler (×2):              __________ (1 or 2)
  [ ] Approved Storage (×2):       __________ (1 or 2)
  [ ] Outdoor Override:            __________
  [ ] Other: _____________         __________

  Floor Level Factor:              __________ (percentage ÷ 100)
  Skip if outdoor?                 [ ] Yes

  CALCULATION:
  _______ × _______ × _______ × _______ = _______ lbs (LIMIT)

  Actual Quantity (from inventory): _______ grams
  Convert to lbs: _______ × 0.0022046226 = _______ lbs (ACTUAL)

  Percentage: (ACTUAL ÷ LIMIT) × 100 = _______%

  Status: [ ] COMPLIANT (<80%)  [ ] NEAR_THRESHOLD (80-99%)  [ ] OVER_THRESHOLD (≥100%)

LIQUID:
  Baseline from fire code:         __________ gal
  Is N/A (Not Applicable)?         [ ] Yes → Skip to status N/A
  Is NL (No Limit)?                [ ] Yes → Skip to status NL

  Applicable Rules:
  [ ] Sprinkler (×2):              __________ (1 or 2)
  [ ] Approved Storage (×2):       __________ (1 or 2)
  [ ] Outdoor Override:            __________
  [ ] Other: _____________         __________

  Floor Level Factor:              __________ (percentage ÷ 100)
  Skip if outdoor?                 [ ] Yes

  CALCULATION:
  _______ × _______ × _______ × _______ = _______ gal (LIMIT)

  Actual Quantity (from inventory): _______ liters
  Convert to gal: _______ × 0.2641720524 = _______ gal (ACTUAL)

  Percentage: (ACTUAL ÷ LIMIT) × 100 = _______%

  Status: [ ] COMPLIANT (<80%)  [ ] NEAR_THRESHOLD (80-99%)  [ ] OVER_THRESHOLD (≥100%)

GAS:
  Baseline from fire code:         __________ ft³
  Is N/A (Not Applicable)?         [ ] Yes → Skip to status N/A
  Is NL (No Limit)?                [ ] Yes → Skip to status NL
  Report As Liquid?                [ ] Yes → Use liquid bucket instead

  Applicable Rules:
  [ ] Sprinkler (×2):              __________ (1 or 2)
  [ ] Approved Storage (×2):       __________ (1 or 2)
  [ ] Outdoor Override:            __________
  [ ] Other: _____________         __________

  Floor Level Factor:              __________ (percentage ÷ 100)
  Skip if outdoor?                 [ ] Yes

  CALCULATION:
  _______ × _______ × _______ × _______ = _______ ft³ (LIMIT)

  Actual Quantity (from inventory): _______ liters
  Convert to ft³: _______ × 0.0353146667 = _______ ft³ (ACTUAL)

  Percentage: (ACTUAL ÷ LIMIT) × 100 = _______%

  Status: [ ] COMPLIANT (<80%)  [ ] NEAR_THRESHOLD (80-99%)  [ ] OVER_THRESHOLD (≥100%)

HAZARD CLASS STATUS: (worst of solid, liquid, gas)
  [ ] COMPLIANT  [ ] NEAR_THRESHOLD  [ ] OVER_THRESHOLD  [ ] N/A

═══════════════════════════════════════════════════════════════════════════════

CONTROL AREA ROLLUP
───────────────────
List all hazard class statuses:

| Hazard Class                | Status           |
|-----------------------------|------------------|
| ___________________________| ________________ |
| ___________________________| ________________ |
| ___________________________| ________________ |
| ___________________________| ________________ |

CONTROL AREA STATUS: (worst of all hazard classes)
  [ ] EXEMPT  [ ] COMPLIANT  [ ] NEAR_THRESHOLD  [ ] OVER_THRESHOLD

═══════════════════════════════════════════════════════════════════════════════

BUILDING ROLLUP
───────────────
List all control area statuses:

| Control Area                | Status           |
|-----------------------------|------------------|
| ___________________________| ________________ |
| ___________________________| ________________ |
| ___________________________| ________________ |

BUILDING STATUS: (worst of all control areas)
  [ ] INCOMPLETE (no control areas)
  [ ] COMPLIANT
  [ ] NEAR_THRESHOLD
  [ ] OVER_THRESHOLD

═══════════════════════════════════════════════════════════════════════════════
```

---

### 17.8 Conversion Quick Reference

#### Unit Conversions

| From | To | Multiply By | Example |
|------|----|-------------|---------|
| Grams | Pounds | 0.0022046226 | 1000g × 0.0022046226 = 2.2046 lbs |
| Liters | Gallons | 0.2641720524 | 10L × 0.2641720524 = 2.6417 gal |
| Liters | Cubic Feet | 0.0353146667 | 100L × 0.0353146667 = 3.5315 ft³ |

#### Common Floor Level Percentages (Occupancy B)

| Floor | Percentage | Factor |
|-------|------------|--------|
| 1 (first above ground) | 100% | 1.0 |
| -1 or 2 | 75% | 0.75 |
| -2 or 3 | 50% | 0.50 |
| 4, 5, 6 | 12.5% | 0.125 |
| 7-9 | 5% | 0.05 |
| 10+ | 5% | 0.05 |
| <-2 | 0% | 0 (prohibited) |

#### Sprinkler Logic

| Building Type | Floor | Sprinklered? | Factor |
|--------------|-------|--------------|--------|
| FULL | Any | Yes | 2 |
| BASEMENT_ONLY | < 0 | Yes | 2 |
| BASEMENT_ONLY | ≥ 0 | No | 1 |
| NONE | Any | No | 1 |

---

## 18. Glossary

| Term | Definition |
|------|------------|
| **AHJ** | Authority Having Jurisdiction - The organization, office, or individual responsible for enforcing fire code requirements |
| **Approved Storage Cabinet** | A cabinet specifically designed for hazardous material storage, meeting NFPA 30 requirements or listed to UL 1275 |
| **Band** | Classification assigned to chemicals based on hazard properties |
| **Baseline** | The maximum allowable quantity as listed in code tables, before any factors are applied |
| **Control Area** | A space within a building where hazardous materials not exceeding the MAQ are stored, dispensed, used, or handled |
| **Cubic Feet (ft³)** | Standard unit of volume for gases at normal temperature and pressure |
| **Factor** | Multiplier or modifier applied to baseline (sprinkler, floor, etc.) |
| **Fire Code** | A set of regulations adopted by a jurisdiction governing fire prevention, fire protection, and life safety |
| **Flash Point** | The minimum temperature at which a liquid gives off vapor sufficient to form an ignitable mixture with air |
| **Gallon (gal)** | Standard unit of volume for liquids in fire codes |
| **Ground Plane** | The finished floor level of a building at the level of exit discharge |
| **Hazard Class** | A category of hazardous material based on its physical or health properties (e.g., Flammable Liquid IA) |
| **High-Hazard Occupancy (Group H)** | A building classification for structures containing quantities of hazardous materials exceeding MAQ limits |
| **HMIS** | Hazardous Materials Inventory Statement - a reporting format requiring storage, use-closed, and use-open MAQ values |
| **MAQ** | Maximum Allowable Quantity - the maximum amount of a hazardous material permitted per control area before High-Hazard classification is required |
| **NFPA 13** | Standard for the Installation of Sprinkler Systems - referenced for the 100% MAQ increase |
| **NL** | No Limit - designation meaning no maximum quantity limit applies |
| **N/A** | Not Applicable - designation meaning the combination doesn't apply or isn't tracked |
| **NTP** | Normal Temperature and Pressure (20°C/68°F, 1 atm) |
| **Occupancy** | Building use classification (A, B, E, F, H, etc.) |
| **Physical State** | Whether a material is solid, liquid, or gas at normal temperature and pressure |
| **Pound (lb)** | Standard unit of weight for solids in fire codes |
| **Sprinkler System** | An automatic fire suppression system with heat-activated sprinkler heads connected to a water supply |
| **Use-Closed** | Using hazardous materials in a system that doesn't normally allow vapors to escape to the atmosphere |
| **Use-Open** | Using hazardous materials in a way that allows vapors to escape to the atmosphere |

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│                     MAQ CALCULATION FORMULA                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Final MAQ = Baseline × Sprinkler × Floor Level × Appr. Storage │
│                                                                  │
├─────────────────────────────────────────────────────────────────┤
│  SPRINKLER FACTOR                                                │
│  • Building sprinklered throughout (NFPA 13): 2×                 │
│  • Not sprinklered or partial: 1×                                │
├─────────────────────────────────────────────────────────────────┤
│  FLOOR LEVEL FACTORS                                             │
│  • Ground: 100%  • 2nd: 75%   • 3rd: 50%                        │
│  • 4-6th: 12.5%  • 7+: 5%     • B1: 75%   • B2: 50%             │
├─────────────────────────────────────────────────────────────────┤
│  APPROVED STORAGE FACTOR                                         │
│  • In approved cabinet: 2×                                       │
│  • Standard storage: 1×                                          │
├─────────────────────────────────────────────────────────────────┤
│  COMPLIANCE STATUS                                               │
│  • Actual < 80% MAQ: COMPLIANT                                   │
│  • Actual 80-99% MAQ: NEAR THRESHOLD (warning)                   │
│  • Actual ≥ MAQ: OVER THRESHOLD (violation)                      │
├─────────────────────────────────────────────────────────────────┤
│  KEY REMINDERS                                                   │
│  • Check Use-Open limits separately (typically 1/3 to 1/5)       │
│  • Storage + Use must not exceed storage MAQ (aggregate rule)    │
│  • Outdoor areas use separate tables, no floor reductions        │
│  • Sum all materials of same hazard class together               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Appendix A: File Locations (For Developers)

| Purpose | File Path |
|---------|-----------|
| MAQ calculations | `packages/common/src/utils/maq-compliance.helper.ts` |
| Server orchestration | `packages/server/src/building/MAQReportService.ts` |
| Fire code service | `packages/server/src/fire-code/FireCodeService.ts` |
| Control area service | `packages/server/src/control-area/ControlAreaService.ts` |
| MAQ table UI | `packages/client/src/app/building-maq/control-area/MaqTable.tsx` |
| MAQ factor tooltip | `packages/client/src/app/building-maq/control-area/DisplayMaqFormulae.tsx` |
| HMIS export | `packages/client/src/utils/MaqExportHelper.tsx` |
| Fire code fixtures | `packages/common/src/fixtures/internal/CFC2016_firecode.ts` |

---

## Appendix B: Field Inspector Guides

This appendix provides comprehensive field guidance for fire inspectors conducting MAQ inspections. These sections address practical field inspection scenarios and provide the detailed technical information needed to perform inspections without requiring additional reference materials.

---

### B.1 Hazard Class Identification

**Authority:** This section reflects hazard class definitions from IFC 2024 Chapter 50 and NFPA 400-2022. Check for amendments in your jurisdiction.

**Currency Date:** Current as of January 2025. Verify classifications against your jurisdiction's adopted fire code edition.

#### B.1.1 Reading Safety Data Sheets (SDS)

The SDS is your primary source for hazard classification. Key sections to review:

**Section 2: Hazard Identification**
- GHS hazard classification
- Signal word (Danger/Warning)
- Hazard statements (H-codes)
- Precautionary statements (P-codes)

**Section 9: Physical and Chemical Properties**
- Flash point (critical for flammable classification)
- Boiling point
- Physical state
- Vapor pressure

**Section 14: Transport Information**
- DOT hazard class
- UN number
- Packing group

#### B.1.2 GHS to Fire Code Hazard Class Mapping

**Physical Hazards - Flammable Liquids**

| GHS Category | H-Code | Fire Code Class | Flash Point | Boiling Point |
|--------------|--------|-----------------|-------------|---------------|
| Category 1 | H224 | Flammable Liquid IA | <73°F (23°C) | <100°F (38°C) |
| Category 2 | H225 | Flammable Liquid IB | <73°F (23°C) | ≥100°F (38°C) |
| Category 3 | H226 | Flammable Liquid IC | ≥73°F and <100°F (23-38°C) | Any |
| Category 4 | H227 | Combustible Liquid II | ≥100°F and <140°F (38-60°C) | Any |
| — | — | Combustible Liquid IIIA | ≥140°F and <200°F (60-93°C) | Any |
| — | — | Combustible Liquid IIIB | ≥200°F (93°C) | Any |

**Physical Hazards - Flammable Gases**

| GHS Category | H-Code | Fire Code Class |
|--------------|--------|-----------------|
| Category 1A | H220 | Flammable Gas |
| Category 1B | H220 | Flammable Gas |
| Category 2 | H221 | Flammable Gas (limited quantities) |

**Physical Hazards - Oxidizers**

| GHS Category | H-Code | Fire Code Class |
|--------------|--------|-----------------|
| Category 1 | H271 | Oxidizer 4 (may cause fire or explosion) |
| Category 2 | H272 | Oxidizer 3 (may intensify fire) |
| Category 3 | H272 | Oxidizer 2 (may intensify fire) |
| — | — | Oxidizer 1 (slight increase in burning rate) |

**Physical Hazards - Organic Peroxides**

| GHS Type | H-Code | Fire Code Class |
|----------|--------|-----------------|
| Type A | H240 | Not permitted (explosive) |
| Type B | H241 | Organic Peroxide I |
| Type C/D | H242 | Organic Peroxide II |
| Type E/F | H242 | Organic Peroxide III |
| Type G | — | Organic Peroxide V |

**Physical Hazards - Pyrophoric Materials**

| GHS Category | H-Code | Fire Code Class |
|--------------|--------|-----------------|
| Pyrophoric Liquid 1 | H250 | Pyrophoric |
| Pyrophoric Solid 1 | H250 | Pyrophoric |

**Physical Hazards - Water-Reactive Materials**

| GHS Category | H-Code | Fire Code Class |
|--------------|--------|-----------------|
| Category 1 | H260 | Water-Reactive 3 |
| Category 2 | H261 | Water-Reactive 2 |
| Category 3 | H261 | Water-Reactive 1 |

**Physical Hazards - Unstable/Reactive Materials**

| GHS Category | H-Code | Fire Code Class |
|--------------|--------|-----------------|
| Category 1 | H200/H201 | Unstable Reactive 4 |
| Category 2 | H202/H203 | Unstable Reactive 3 |
| Category 3 | H204 | Unstable Reactive 2 |
| Category 4 | — | Unstable Reactive 1 |

**Health Hazards - Toxics**

| GHS Category | H-Code | Fire Code Class | LD50 (oral, rat) |
|--------------|--------|-----------------|------------------|
| Category 1 | H300/H310/H330 | Highly Toxic | ≤50 mg/kg |
| Category 2 | H300/H310/H330 | Highly Toxic | 50-200 mg/kg |
| Category 3 | H301/H311/H331 | Toxic | 200-2000 mg/kg |
| Category 4 | H302/H312/H332 | (Not MAQ regulated) | >2000 mg/kg |

**Health Hazards - Corrosives**

| GHS Category | H-Code | Fire Code Class |
|--------------|--------|-----------------|
| Category 1A | H314 | Corrosive |
| Category 1B | H314 | Corrosive |
| Category 1C | H314 | Corrosive |

#### B.1.3 Reading Container Labels

When SDS is not readily available, container labels provide classification information.

**NFPA 704 Diamond**
```
        ┌───┐
        │ 4 │ ← FLAMMABILITY (Red, top)
    ┌───┼───┼───┐
    │ 4 │   │ 4 │
    │   │   │   │
    └───┼───┼───┘
    ↑   │ W │   ↑
HEALTH  └───┘  INSTABILITY
(Blue)    ↑    (Yellow)
       SPECIAL
       (White)
```

**Rating Scale (0-4):**
- 0 = Minimal hazard
- 1 = Slight hazard
- 2 = Moderate hazard
- 3 = Serious hazard
- 4 = Severe hazard

**Special Symbols:**
- W = Water reactive
- OX = Oxidizer
- SA = Simple asphyxiant

**GHS Pictograms on Labels**

| Pictogram | Hazard Types |
|-----------|--------------|
| Flame | Flammable liquids, gases, solids; pyrophorics; self-heating |
| Flame over circle | Oxidizers |
| Exploding bomb | Explosives, self-reactives, organic peroxides |
| Skull and crossbones | Acute toxicity (Categories 1-3) |
| Corrosion | Corrosive to metals, skin corrosion |
| Health hazard | Carcinogen, respiratory sensitizer, target organ toxicity |
| Exclamation mark | Irritant, skin sensitizer, acute toxicity (Category 4) |
| Gas cylinder | Compressed gases |

#### B.1.4 When SDS is Unavailable

If SDS cannot be located during inspection:

1. **Check container label** for GHS pictograms, NFPA 704 diamond, or hazard statements
2. **Look for manufacturer name** and contact for SDS
3. **Search SDS databases:**
   - Manufacturer website
   - OSHA SDS database
   - ChemSpider/PubChem for chemical properties
4. **Document the gap** in your inspection report
5. **Require facility to obtain SDS** within 24-48 hours
6. **Use conservative classification** if immediate determination needed

#### B.1.5 Physical State Determination

Determine physical state at Normal Temperature and Pressure (NTP: 68°F/20°C, 1 atm):

| Observation | Classification |
|-------------|----------------|
| Flows and takes container shape | Liquid |
| Maintains own shape | Solid |
| Expands to fill container | Gas |
| Compressed gas in container that liquefies | Liquefied Gas |

**Common Ambiguities:**
- **Gels/pastes**: Usually classified as liquids
- **Powders/granules**: Classified as solids
- **Aerosols**: Consider both propellant and product hazards
- **Cryogenic liquids**: Gases stored at very low temperatures

#### B.1.6 Chemical Mixtures

For mixtures, apply the **most restrictive** classification:

1. **Consult SDS Section 2** for mixture classification
2. **If not classified as mixture**, identify hazards of each component
3. **Apply concentration thresholds:**
   - Highly Toxic: ≥1% of mixture
   - Toxic: ≥1% of mixture
   - Corrosive: ≥1% of mixture
   - Flammable: Based on overall flash point

**Example:**
```
Mixture: 70% Ethanol (Flammable IB), 30% Water
Flash Point of mixture: ~75°F
Classification: Flammable Liquid IC (flash point 73-100°F)
```

---

### B.2 MAQ Baseline Tables

**Authority:** IFC 2024 Tables 5003.1.1(1) through 5003.1.1(4) and IBC 2024 Table 414.2.2.

**Currency Date:** Current as of January 2025. Values shown are for Occupancy Group B (Business). Other occupancies may have different values.

#### B.2.1 Indoor Physical Hazards - Table 5003.1.1(1)

**Storage and Use-Closed Quantities**

| Hazard Class | Solid (lbs) | Liquid (gal) | Gas (ft³) | Notes |
|--------------|-------------|--------------|-----------|-------|
| **Combustible Liquid II** | — | 120 | — | FP ≥100°F and <140°F |
| **Combustible Liquid IIIA** | — | 330 | — | FP ≥140°F and <200°F |
| **Combustible Liquid IIIB** | — | NL | — | FP ≥200°F |
| **Cryogenic Flammable** | — | 45 | — | |
| **Cryogenic Inert** | — | NL | — | |
| **Cryogenic Oxidizing** | — | 45 | — | |
| **Explosives Div 1.1** | 1ᵃ | 1ᵃ | — | Mass explosion hazard |
| **Explosives Div 1.2** | 1ᵃ | 1ᵃ | — | Projection hazard |
| **Explosives Div 1.3** | 5ᵃ | 5ᵃ | — | Fire hazard |
| **Explosives Div 1.4** | 50ᵃ | 50ᵃ | — | Minor blast hazard |
| **Explosives Div 1.4G** | 125 | 125 | — | Consumer fireworks |
| **Explosives Div 1.5** | 1ᵃ | 1ᵃ | — | Blasting agents |
| **Explosives Div 1.6** | NL | NL | — | Very insensitive |
| **Flammable Gas** | — | — | 1,000 | |
| **Flammable Gas Liquefied** | — | 150 | — | Stored as liquid |
| **Flammable Liquid IA** | — | 30 | — | FP <73°F, BP <100°F |
| **Flammable Liquid IB** | — | 120 | — | FP <73°F, BP ≥100°F |
| **Flammable Liquid IC** | — | 120 | — | FP ≥73°F and <100°F |
| **Flammable Solid** | 125 | — | — | |
| **Inert Gas** | — | — | NL | |
| **Inert Gas Liquefied** | — | NL | — | |
| **Organic Peroxide UD** | 1ᵃ | 1ᵃ | — | Unclassified/Detonable |
| **Organic Peroxide I** | 1ᵃ | 1ᵃ | — | |
| **Organic Peroxide II** | 5ᵃ | 5ᵃ | — | |
| **Organic Peroxide III** | 10 | 10 | — | |
| **Organic Peroxide IV** | 20 | 20 | — | |
| **Organic Peroxide V** | NL | NL | — | |
| **Oxidizer 1** | 4,000 | 4,000 | — | |
| **Oxidizer 2** | 250 | 250 | — | |
| **Oxidizer 3** | 10 | 10 | — | |
| **Oxidizer 4** | 1ᵃ | 1ᵃ | — | |
| **Oxidizing Gas** | — | — | 1,500 | |
| **Oxidizing Gas Liquefied** | — | 150 | — | |
| **Pyrophoric** | 4ᵃ | 4ᵃ | 50ᵃ | |
| **Unstable Reactive 1** | NL | NL | — | |
| **Unstable Reactive 2** | 50 | 50 | — | |
| **Unstable Reactive 3** | 5 | 5 | — | |
| **Unstable Reactive 4** | 1ᵃ | 1ᵃ | — | |
| **Water-Reactive 1** | NL | NL | — | |
| **Water-Reactive 2** | 50 | 50 | — | |
| **Water-Reactive 3** | 5ᵃ | 5ᵃ | — | |

ᵃ Sprinkler protection required for any quantity

#### B.2.2 Indoor Health Hazards - Table 5003.1.1(2)

**Storage and Use-Closed Quantities**

| Hazard Class | Solid (lbs) | Liquid (gal) | Gas (ft³) | Liquefied (gal) |
|--------------|-------------|--------------|-----------|-----------------|
| **Corrosive** | 5,000 | 500 | 810 | 150 |
| **Highly Toxic** | 10 | 10 | 20ᵃ | 4ᵃ |
| **Toxic** | 500 | 500 | 810 | 150 |

ᵃ Sprinkler protection required for any quantity

#### B.2.3 Outdoor Physical Hazards - Table 5003.1.1(3)

**Storage and Use-Closed Quantities**

| Hazard Class | Solid (lbs) | Liquid (gal) | Gas (ft³) | Notes |
|--------------|-------------|--------------|-----------|-------|
| **Combustible Liquid II** | — | NL | — | |
| **Combustible Liquid IIIA** | — | NL | — | |
| **Combustible Liquid IIIB** | — | NL | — | |
| **Cryogenic Flammable** | — | 90 | — | |
| **Cryogenic Inert** | — | NL | — | |
| **Cryogenic Oxidizing** | — | 90 | — | |
| **Explosives** | Per IFC Ch. 56 | — | — | |
| **Flammable Gas** | — | — | NL | |
| **Flammable Gas Liquefied** | — | NL | — | |
| **Flammable Liquid IA** | — | 60 | — | |
| **Flammable Liquid IB** | — | 240 | — | |
| **Flammable Liquid IC** | — | 240 | — | |
| **Flammable Solid** | 250 | — | — | |
| **Inert Gas** | — | — | NL | |
| **Inert Gas Liquefied** | — | NL | — | |
| **Organic Peroxide UD** | 2ᵃ | 2ᵃ | — | |
| **Organic Peroxide I** | 2ᵃ | 2ᵃ | — | |
| **Organic Peroxide II** | 10ᵃ | 10ᵃ | — | |
| **Organic Peroxide III** | 20 | 20 | — | |
| **Organic Peroxide IV** | 40 | 40 | — | |
| **Organic Peroxide V** | NL | NL | — | |
| **Oxidizer 1** | NL | NL | — | |
| **Oxidizer 2** | 500 | 500 | — | |
| **Oxidizer 3** | 20 | 20 | — | |
| **Oxidizer 4** | 2ᵃ | 2ᵃ | — | |
| **Oxidizing Gas** | — | — | NL | |
| **Oxidizing Gas Liquefied** | — | NL | — | |
| **Pyrophoric** | 8ᵃ | 8ᵃ | 100ᵃ | |
| **Unstable Reactive 1** | NL | NL | — | |
| **Unstable Reactive 2** | 100 | 100 | — | |
| **Unstable Reactive 3** | 10 | 10 | — | |
| **Unstable Reactive 4** | 2ᵃ | 2ᵃ | — | |
| **Water-Reactive 1** | NL | NL | — | |
| **Water-Reactive 2** | 100 | 100 | — | |
| **Water-Reactive 3** | 10ᵃ | 10ᵃ | — | |

ᵃ Sprinkler protection required for any quantity

#### B.2.4 Outdoor Health Hazards - Table 5003.1.1(4)

**Storage and Use-Closed Quantities**

| Hazard Class | Solid (lbs) | Liquid (gal) | Gas (ft³) | Liquefied (gal) |
|--------------|-------------|--------------|-----------|-----------------|
| **Corrosive** | NL | NL | NL | NL |
| **Highly Toxic** | 20 | 20 | 40ᵃ | 8ᵃ |
| **Toxic** | 1,000 | 1,000 | 1,620 | 300 |

ᵃ Sprinkler protection required for any quantity

#### B.2.5 Table Notes and Conditions

**Note a - Sprinkler Requirement:**
Quantities shown require building to be equipped with an approved automatic sprinkler system throughout per NFPA 13. Without sprinklers, the MAQ is 0.

**Note b - Sprinkler Increase:**
The MAQ may be increased 100% (2×) in buildings equipped with an approved automatic sprinkler system throughout per NFPA 13.

**Note c - Approved Storage Increase:**
The MAQ may be increased 100% (2×) when stored in approved storage cabinets, gas cabinets, exhausted enclosures, or safety cans.

**Note d - Cumulative Increases:**
When both Note b and Note c apply, the increases are cumulative (baseline × 2 × 2 = 4× baseline).

**NL = No Limit:**
No maximum quantity limit for standard occupancy classification.

**N/A = Not Applicable:**
Physical state not applicable to this hazard class.

#### B.2.6 Complete Use-Open/Use-Closed Tables

**Authority:** IFC 2024 Tables 5003.1.1(1) and 5003.1.1(2)

Use-Open quantities are lower than Storage/Use-Closed because open containers present greater vapor exposure and fire risk. The following tables provide **complete** Use-Open and Use-Closed limits for all hazard classes.

**Important:** All values shown are **baseline** values (unsprinklered). Apply the sprinkler factor (×2) and approved storage factor (×2) where applicable.

##### Physical Hazards - Flammable/Combustible Liquids

| Hazard Class | State | Use-Closed | Use-Open | Units | Ratio |
|--------------|-------|------------|----------|-------|-------|
| Flammable Liquid: IA | Liquid | 30 | 10 | gal | 3:1 |
| Flammable Liquid: IB, IC | Liquid | 120 | 30 | gal | 4:1 |
| Combustible Liquid: II | Liquid | 120 | 30 | gal | 4:1 |
| Combustible Liquid: IIIA | Liquid | 330 | 80 | gal | ~4:1 |
| Combustible Liquid: IIIB | Liquid | NL | NL | gal | — |

##### Physical Hazards - Flammable Gases

| Hazard Class | State | Use-Closed | Use-Open | Units | Notes |
|--------------|-------|------------|----------|-------|-------|
| Flammable Gas | Gas | 1,000 | 1,000 | ft³ | Approved storage ×2 applies |
| Flammable Gas Liquefied | Liquid | 150 | — | gal | No use-open limit defined |

##### Physical Hazards - Flammable Solids

| Hazard Class | State | Use-Closed | Use-Open | Units | Ratio |
|--------------|-------|------------|----------|-------|-------|
| Flammable Solid | Solid | 125 | 25 | lbs | 5:1 |

##### Physical Hazards - Cryogenics

| Hazard Class | State | Use-Closed | Use-Open | Units | Ratio |
|--------------|-------|------------|----------|-------|-------|
| Cryogenic Flammable | Liquid | 45 | 10 | gal | ~4.5:1 |
| Cryogenic Oxidizing | Liquid | 45 | 10 | gal | ~4.5:1 |
| Cryogenic Inert | Liquid | NL | NL | gal | — |

##### Physical Hazards - Explosives

| Hazard Class | State | Use-Closed | Use-Open | Units | Notes |
|--------------|-------|------------|----------|-------|-------|
| Explosive: Division 1.1, 1.2, 1.5 | S/L | 0.25ᵃ | 0.25ᵃ | lbs | Sprinklers required |
| Explosive: Division 1.3 | S/L | 1ᵃ | 1ᵃ | lbs | Sprinklers required |
| Explosive: Division 1.4 | S/L | 50ᵃ | — | lbs | Sprinklers required |
| Explosive: Division 1.4G, 1.6 | S/L | — | — | lbs | No limit specified |

##### Physical Hazards - Organic Peroxides

| Hazard Class | State | Use-Closed | Use-Open | Units | Ratio |
|--------------|-------|------------|----------|-------|-------|
| Organic Peroxide: UD | S/L | 0.25ᵃ | 0.25ᵃ | lbs/gal | Sprinklers required |
| Organic Peroxide: I | S/L | 1 | 1 | lbs/gal | 1:1 |
| Organic Peroxide: II | S/L | 50 | 10 | lbs/gal | 5:1 |
| Organic Peroxide: III | S/L | 125 | 25 | lbs/gal | 5:1 |
| Organic Peroxide: IV | S/L | 20 | — | lbs/gal | — |
| Organic Peroxide: V | S/L | NL | NL | lbs/gal | — |

##### Physical Hazards - Oxidizers

| Hazard Class | State | Use-Closed | Use-Open | Units | Notes |
|--------------|-------|------------|----------|-------|-------|
| Oxidizer: 1 | S/L | NLᵇ or 4,000 | NLᵇ or 1,000 | lbs/gal | NL if sprinklered |
| Oxidizer: 2 | S/L | 250 | 50 | lbs/gal | 5:1 ratio |
| Oxidizer: 3 | S/L | 2 | 2 | lbs/gal | 1:1 ratio |
| Oxidizer: 4 | S/L | 0.25ᵃ | 0.25ᵃ | lbs/gal | Sprinklers required |
| Oxidizing Gas | Gas | 1,500 | 1,500 | ft³ | Approved storage ×2 applies |
| Oxidizing Gas Liquefied | Liquid | 150 | 150 | gal | Approved storage ×2 applies |

##### Physical Hazards - Pyrophoric Materials

| Hazard Class | State | Use-Closed | Use-Open | Units | Notes |
|--------------|-------|------------|----------|-------|-------|
| Pyrophoric | Solid | 1ᵃ | 0 | lbs | **No open use permitted** |
| Pyrophoric | Liquid | 1ᵃ | 0 | gal | **No open use permitted** |
| Pyrophoric | Gas | 1ᵃ | 1ᵃ | ft³ | Sprinklers required |

##### Physical Hazards - Unstable/Reactive Materials

| Hazard Class | State | Use-Closed | Use-Open | Units | Ratio |
|--------------|-------|------------|----------|-------|-------|
| Unstable Reactive: 1 | S/L | NL | NL | lbs/gal | — |
| Unstable Reactive: 2 | S/L | 50 | 10 | lbs/gal | 5:1 |
| Unstable Reactive: 2 | Gas | 750 | 750 | ft³ | 1:1 |
| Unstable Reactive: 3 | S/L | 1 | 1 | lbs/gal | 1:1 |
| Unstable Reactive: 3 | Gas | 10 | 10 | ft³ | 1:1 |
| Unstable Reactive: 4 | S/L | 0.25ᵃ | 0.25ᵃ | lbs/gal | Sprinklers required |
| Unstable Reactive: 4 | Gas | 2ᵃ | 2ᵃ | ft³ | Sprinklers required |

##### Physical Hazards - Water-Reactive Materials

| Hazard Class | State | Use-Closed | Use-Open | Units | Ratio |
|--------------|-------|------------|----------|-------|-------|
| Water-Reactive: 1 | S/L | NL | NL | lbs/gal | — |
| Water-Reactive: 2 | S/L | 50 | 10 | lbs/gal | 5:1 |
| Water-Reactive: 3 | S/L | 5 | 1 | lbs/gal | 5:1 |

##### Health Hazards - Corrosives

| Hazard Class | State | Use-Closed | Use-Open | Units | Ratio |
|--------------|-------|------------|----------|-------|-------|
| Corrosive | Solid | 5,000 | 1,000 | lbs | 5:1 |
| Corrosive | Liquid | 500 | 100 | gal | 5:1 |
| Corrosive | Gas | 810 | 810 | ft³ | 1:1 |
| Corrosive Liquefied | Solid | 5,000 | 1,000 | lbs | 5:1 |
| Corrosive Liquefied | Liquid | 500 | 100 | gal | 5:1 |
| Corrosive Liquefied | Gas | 150 | 150 | ft³ | 1:1 |

##### Health Hazards - Toxics

| Hazard Class | State | Use-Closed | Use-Open | Units | Ratio |
|--------------|-------|------------|----------|-------|-------|
| Toxic | Solid | 500 | 125 | lbs | 4:1 |
| Toxic | Liquid | 500 | 125 | gal | 4:1 |
| Toxic | Gas | 810 | 810 | ft³ | 1:1 |
| Toxic Liquefied | Solid | 500 | 125 | lbs | 4:1 |
| Toxic Liquefied | Liquid | 500 | 125 | gal | 4:1 |
| Toxic Liquefied | Gas | 150 | 150 | ft³ | 1:1 |

##### Health Hazards - Highly Toxics

| Hazard Class | State | Use-Closed | Use-Open | Units | Notes |
|--------------|-------|------------|----------|-------|-------|
| Highly Toxic | Solid | 10 | 3 | lbs | ~3:1 ratio |
| Highly Toxic | Liquid | 10 | 3 | gal | ~3:1 ratio |
| Highly Toxic | Gas | 20ᵃ | 20ᵃ | ft³ | Sprinklers required |
| Highly Toxic Liquefied | Solid | 10 | 3 | lbs | ~3:1 ratio |
| Highly Toxic Liquefied | Liquid | 10 | 3 | gal | ~3:1 ratio |
| Highly Toxic Liquefied | Gas | 4ᵃ | 4ᵃ | ft³ | Sprinklers required |

**Table Notes:**
- ᵃ Sprinkler protection required for any quantity. Without sprinklers, MAQ = 0.
- ᵇ NL (No Limit) applies when building is sprinklered.
- S/L = Solid or Liquid (same limits apply to both states)
- "—" indicates no use-open limit is defined (use-open not permitted or not applicable)
- All values are baseline before applying sprinkler (×2) or approved storage (×2) factors

#### B.2.7 California Fire Code (CFC) vs IFC Deviations

**Key Finding:** The MAQ table values in CFC are **IDENTICAL** to IFC. California has adopted the IFC MAQ tables without amendment to the quantity values themselves.

**California-Specific Additions:**

| Topic | CFC Requirement | IFC Equivalent |
|-------|-----------------|----------------|
| **Seismic** | Enhanced seismic anchoring per CBC Chapter 17 | References ASCE 7 only |
| **HMMP** | Hazardous Materials Management Plan required per H&SC 25500-25545 | No equivalent federal requirement |
| **CUPA Reporting** | Report to local CUPA via CERS | No equivalent |
| **Semiconductor** | Enhanced provisions for HPM (CFC Ch. 27) | Similar IFC Ch. 27 |

**CFC-Specific Considerations for California Inspections:**

1. **Hazardous Materials Business Plan (HMBP):** Required for facilities with hazardous materials above threshold quantities. Verify registration with CERS (California Environmental Reporting System).

2. **Unified Program Inspections:** CUPAs (Certified Unified Program Agencies) conduct multi-media inspections. Coordinate with local CUPA.

3. **Acutely Hazardous Materials:** California maintains a separate list of acutely hazardous materials with lower reporting thresholds.

4. **Seismic Requirements:** California Building Code Chapter 17 requires enhanced anchoring for storage racks, cabinets, and cylinders in seismic zones.

#### B.2.8 Combination Limits (Flammable Liquids)

**Authority:** IFC 2024 Table 5003.1.1(1), Footnote "h"

**Critical Rule:** Flammable liquids are the **ONLY** hazard class with combination limits in the IFC. All other hazard classes are evaluated independently.

##### The Flammable Liquid Combination Rule

Per IFC Table 5003.1.1(1) Footnote "h":

> "The aggregate quantity of Class IA, IB, and IC flammable liquids shall not exceed the quantity listed for Class IB and IC flammable liquids. The quantity of Class IA flammable liquids shall not exceed the quantity listed for Class IA flammable liquids."

**What This Means:**

| Requirement | Limit (Baseline) | Limit (Sprinklered) |
|-------------|------------------|---------------------|
| **Combined** Class IA + IB + IC | 120 gal | 240 gal |
| **Individual** Class IA maximum | 30 gal | 60 gal |

**Both limits must be satisfied simultaneously.**

##### Combination Limit Examples

**Example 1 - COMPLIANT:**
```
Building: Sprinklered throughout
Class IA: 50 gal ✅ (under 60 gal individual limit)
Class IB: 100 gal
Class IC: 80 gal
────────────────
Total: 230 gal ✅ (under 240 gal combined limit)

Both checks pass → COMPLIANT
```

**Example 2 - VIOLATION (Individual Limit):**
```
Building: Sprinklered throughout
Class IA: 70 gal ❌ (exceeds 60 gal individual limit)
Class IB: 50 gal
Class IC: 50 gal
────────────────
Total: 170 gal ✅ (under 240 gal combined limit)

Class IA exceeds individual limit → VIOLATION
```

**Example 3 - VIOLATION (Combined Limit):**
```
Building: Sprinklered throughout
Class IA: 50 gal ✅ (under 60 gal individual limit)
Class IB: 120 gal
Class IC: 100 gal
────────────────
Total: 270 gal ❌ (exceeds 240 gal combined limit)

Combined total exceeds limit → VIOLATION
```

##### No Other Combination Limits

**Important:** Except for flammable liquids, the IFC tables do **NOT** regulate quantities based on combined/aggregate approaches by hazard class.

This means:
- A control area can have 5,000 lbs of corrosive solids AND 500 gal of corrosive liquids AND 810 ft³ of corrosive gases
- Each physical state is evaluated independently
- No "combined corrosive" limit exists
- Same applies to all other hazard classes

#### B.2.9 Multi-Hazard Chemicals

**Authority:** IFC 2024 Section 5003.1.1 and Appendix E

##### The Multi-Hazard Principle

**Rule:** A chemical with multiple hazards must be counted against EACH applicable hazard class MAQ independently.

Many chemicals have both physical AND health hazards. For example:
- Hydrochloric Acid: Corrosive (health) AND may release toxic gas
- Acetone: Flammable Liquid IB (physical) AND may be an irritant
- Nitric Acid: Corrosive (health) AND Oxidizer (physical)

##### Multi-Hazard Calculation Process

**Step 1:** Identify ALL hazard classifications from the SDS

Review SDS Section 2 (Hazard Identification) for all GHS classifications. A chemical may have multiple H-codes indicating multiple hazards.

**Step 2:** Map EACH hazard to fire code hazard class

Use Section B.1 to map each GHS hazard to its fire code equivalent.

**Step 3:** Count quantity against EACH applicable MAQ

The full quantity counts toward EACH hazard class limit.

**Step 4:** Check compliance for ALL hazard classes

If ANY hazard class exceeds its MAQ, the control area is non-compliant.

##### Multi-Hazard Example

**Chemical:** Glacial Acetic Acid (25 gallons)
**SDS Hazards:**
- H226: Flammable Liquid Category 3 → Flammable Liquid IC
- H314: Skin Corrosion 1A → Corrosive

**MAQ Evaluation (Sprinklered Building):**

| Hazard Class | Quantity | MAQ Limit | Status |
|--------------|----------|-----------|--------|
| Flammable Liquid IC | 25 gal | 240 gal | ✅ COMPLIANT (10%) |
| Corrosive (liquid) | 25 gal | 1,000 gal | ✅ COMPLIANT (2.5%) |

**Both checks must pass.** The 25 gallons counts toward BOTH hazard class limits.

##### Multi-Hazard Worked Example: Nitric Acid

**Chemical:** Nitric Acid, Concentrated (10 gallons)
**SDS Hazards:**
- H272: Oxidizer Category 3 → Oxidizer 3
- H314: Skin Corrosion 1A → Corrosive

**MAQ Evaluation (Unsprinklered Building):**

| Hazard Class | Quantity | MAQ Limit | Status |
|--------------|----------|-----------|--------|
| Oxidizer 3 (liquid) | 10 gal | 10 gal | ⚠️ AT THRESHOLD |
| Corrosive (liquid) | 10 gal | 500 gal | ✅ COMPLIANT (2%) |

**Result:** At threshold for Oxidizer 3. Adding any more concentrated nitric acid would trigger High-Hazard classification.

##### Common Multi-Hazard Chemicals

| Chemical | Physical Hazard | Health Hazard |
|----------|-----------------|---------------|
| Acetone | Flammable Liquid IB | — |
| Hydrochloric Acid | — | Corrosive |
| Nitric Acid (conc.) | Oxidizer 3 | Corrosive |
| Sulfuric Acid (conc.) | — | Corrosive |
| Hydrogen Peroxide (>60%) | Oxidizer 4, Organic Peroxide | Corrosive |
| Sodium Hypochlorite | Oxidizer 1 | Corrosive |
| Potassium Permanganate | Oxidizer 2 | Toxic |
| Formaldehyde | Flammable Liquid IC | Toxic |
| Methanol | Flammable Liquid IB | Toxic |
| Phosphorus (white) | Pyrophoric | Toxic |

#### B.2.10 Compressed Gas Cylinder Volume Calculations

**Authority:** IFC 2024 Chapter 53, Ideal Gas Law

##### Why This Matters

Compressed gas MAQ limits are specified in **cubic feet at NTP** (Normal Temperature and Pressure: 68°F/20°C, 1 atm). However, gas cylinders are labeled with:
- Water capacity (gallons or liters)
- Service pressure (psig)

You must calculate the equivalent gas volume at NTP.

##### The Ideal Gas Law Method

**Formula:**
```
V₂ = (P₁ × V₁) / P₂

Where:
  V₂ = Gas volume at atmospheric pressure (what we need)
  P₁ = Cylinder pressure (absolute: gauge + 14.7 psi)
  V₁ = Cylinder internal volume (water capacity)
  P₂ = Atmospheric pressure (14.7 psia)
```

**Simplified Formula:**
```
Gas Volume (ft³) = Cylinder Volume (ft³) × (Gauge Pressure + 14.7) / 14.7
```

##### Standard Cylinder Volumes

| Cylinder Size | Water Capacity | At 2,200 psig | At 2,400 psig |
|---------------|----------------|---------------|---------------|
| Lecture Bottle | 0.5 L (0.13 gal) | ~25 ft³ | ~27 ft³ |
| Size B | 5 L (1.3 gal) | ~260 ft³ | ~285 ft³ |
| Size C | 10 L (2.6 gal) | ~520 ft³ | ~570 ft³ |
| Size D | 20 L (5.3 gal) | ~1,040 ft³ | ~1,140 ft³ |
| Size E (T-cylinder) | 45 L (12 gal) | ~2,340 ft³ | ~2,565 ft³ |
| Size G/H | 80 L (21 gal) | ~4,160 ft³ | ~4,560 ft³ |
| Size K (K-bottle) | 130 L (34 gal) | ~6,760 ft³ | ~7,410 ft³ |

**Note:** Actual volumes vary by gas type. These are approximations for ideal gases.

##### Step-by-Step Cylinder Calculation

**Given:** Nitrogen cylinder, 50L water capacity, 2,200 psig service pressure

**Step 1:** Convert water capacity to cubic feet
```
50 L × 0.0353147 ft³/L = 1.77 ft³
```

**Step 2:** Calculate pressure ratio
```
(2,200 + 14.7) / 14.7 = 150.7
```

**Step 3:** Calculate gas volume at NTP
```
1.77 ft³ × 150.7 = 267 ft³
```

**Result:** This cylinder contains approximately **267 cubic feet** of nitrogen at NTP.

##### Quick Reference: Common Gases

| Gas | Typical Pressure | 50L Cylinder | Hazard Class |
|-----|------------------|--------------|--------------|
| Nitrogen | 2,200 psig | 267 ft³ | Inert Gas |
| Argon | 2,200 psig | 267 ft³ | Inert Gas |
| Helium | 2,400 psig | 292 ft³ | Inert Gas |
| Oxygen | 2,200 psig | 267 ft³ | Oxidizing Gas |
| Hydrogen | 2,400 psig | 292 ft³ | Flammable Gas |
| Acetylene | 250 psig | 18 ft³ | Flammable Gas |
| Propane | 200 psig | 15 ft³ | Flammable Gas Liquefied |

**Special Case - Acetylene:**
Acetylene cylinders contain a porous mass saturated with solvent (acetone). The pressure is much lower (~250 psig) and the calculation is different. Use manufacturer data.

**Special Case - Liquefied Gases:**
For gases stored in liquefied form (propane, ammonia, CO₂), use the liquid volume for MAQ comparison, not the gas expansion volume.

##### Field Shortcut

**For standard high-pressure cylinders (2,000-2,500 psig):**
```
Gas Volume (ft³) ≈ Water Capacity (liters) × 5.3
```

**Example:** 45L cylinder ≈ 45 × 5.3 = 240 ft³ (rough estimate)

#### B.2.11 MAQ Exemptions

**Authority:** IFC 2024 Section 5003.1.1.1 and Table Footnotes

The following materials and quantities are **EXEMPT** from MAQ calculations:

##### Automatic Exemptions (Not Counted Toward MAQ)

| Exemption Category | Conditions | IFC Reference |
|--------------------|------------|---------------|
| **Vehicle fuel tanks** | Liquid or gaseous fuel in fuel tanks on vehicles | 5003.1.1.1(1) |
| **Motorized equipment fuel** | Fuel in tanks on motorized equipment operated per code | 5003.1.1.1(2) |
| **Piped gaseous fuels** | Gaseous fuels in piping systems and fixed appliances per IFGC | 5003.1.1.1(3) |
| **Piped liquid fuels** | Liquid fuels in piping systems and fixed appliances per IMC | 5003.1.1.1(4) |
| **Hand sanitizers** | Alcohol-based hand rubs (Class I/II) in dispensers per 5705.5 | 5003.1.1.1(5) |

##### Retail/Wholesale Exemptions (Group M and S Occupancies)

| Material Type | Container Limit | Additional Conditions |
|---------------|-----------------|----------------------|
| **Alcoholic beverages** | ≤1.3 gallons each | Retail/wholesale sales only |
| **Medicines** | ≤1.3 gallons each | Packaged for consumer use |
| **Foodstuffs** | ≤1.3 gallons each | Packaged for consumer use |
| **Consumer products** | ≤1.3 gallons each | ≤50% water-miscible liquids, remainder non-flammable |
| **Cosmetics** | ≤1.3 gallons each | ≤50% water-miscible liquids, remainder non-flammable |

**Key Limitation:** These exemptions apply ONLY in retail and wholesale sales occupancies (Group M) and storage (Group S). They do NOT apply in laboratories (Group B), manufacturing (Group F), or other occupancies.

##### Exemption Decision Tree

```
Is the material in a vehicle fuel tank?
  → YES: EXEMPT

Is the material in piping/fixed appliances?
  → YES: EXEMPT (if per IMC/IFGC)

Is this a retail/wholesale occupancy (Group M or S)?
  → NO: No retail exemptions apply
  → YES: Continue...

      Is it alcoholic beverages ≤1.3 gal containers?
        → YES: EXEMPT

      Is it medicine/food/cosmetics in ≤1.3 gal consumer packaging?
        → YES: Check if ≤50% water-miscible with non-flammable remainder
           → YES: EXEMPT
           → NO: NOT EXEMPT

Is it alcohol-based hand rub in compliant dispenser?
  → YES: EXEMPT (per Section 5705.5 requirements below)
  → NO: NOT EXEMPT
```

##### Section 5705.5 Alcohol-Based Hand Rub Dispenser Requirements

**Authority:** IFC 2024 Section 5705.5

For alcohol-based hand rub (ABHR) dispensers to qualify for the MAQ exemption, ALL of the following requirements must be met:

**Dispenser Requirements:**

| Requirement | Specification |
|-------------|---------------|
| **Maximum dispenser capacity** | 68 fl oz (2.0 L) for liquid/gel, 18 fl oz (0.5 L) for aerosol |
| **Maximum single dose** | 0.08 fl oz (2.4 mL) per activation |
| **Dispenser construction** | Designed to resist accidental discharge |
| **Mounting** | Securely attached to wall, floor stand, or counter |

**Quantity Limits per Floor:**

| Location | Maximum per Floor | Corridor Spacing |
|----------|-------------------|------------------|
| **Corridors** | 1 per 1,000 ft² of corridor | ≥4 ft between dispensers |
| **Rooms/Suites** | 1 per room or 1 per 2,500 ft² | N/A |
| **Near exits** | Not within 1 ft of ignition sources | N/A |

**Storage Limits:**

| Storage Type | Maximum Quantity |
|--------------|------------------|
| **Aerosol dispensers in storage** | 576 fl oz (17 L) per fire area outside of cabinet |
| **Aerosol in approved storage cabinet** | 1,152 fl oz (34 L) per fire area |
| **Liquid/gel in storage** | 5 gallons per fire area outside of cabinet |
| **Liquid/gel in approved cabinet** | 10 gallons per fire area |

**Prohibited Locations:**

- Within 1 inch of an ignition source (switches, outlets, appliances)
- Above or beside ignition sources
- In means of egress where width is less than 6 feet (unless recessed)
- Where dispensers obstruct the required egress width

**Field Verification Checklist for ABHR Exemption:**

- [ ] Dispenser capacity ≤2.0 L (liquid/gel) or ≤0.5 L (aerosol)
- [ ] Dispenser is securely mounted (not freestanding unless approved)
- [ ] Corridor spacing ≥4 ft between dispensers
- [ ] Not located within 1 inch of electrical outlets/switches
- [ ] Not blocking required egress width
- [ ] Quantity limits per floor not exceeded
- [ ] Storage quantities comply with limits above

**If ANY requirement is not met:** The ABHR does NOT qualify for exemption and MUST be counted toward the Flammable Liquid MAQ.

##### Exemption Examples

**Example 1 - EXEMPT:**
```
Location: Grocery store (Group M)
Material: 100 bottles of wine (750mL each)
Analysis: Alcoholic beverages in containers ≤1.3 gal in retail
Result: EXEMPT - does not count toward MAQ
```

**Example 2 - NOT EXEMPT:**
```
Location: Research laboratory (Group B)
Material: 50 bottles of isopropyl alcohol (1L each) for cleaning
Analysis: Group B occupancy - retail exemptions do not apply
Result: NOT EXEMPT - counts toward Flammable Liquid IB MAQ
```

**Example 3 - EXEMPT:**
```
Location: Office building lobby (Group B)
Material: 20 hand sanitizer dispensers (1.2L each, 70% ethanol)
Analysis: Alcohol-based hand rub in compliant dispensers per 5705.5
Result: EXEMPT - does not count toward MAQ
```

**Example 4 - NOT EXEMPT:**
```
Location: Hardware store (Group M)
Material: 50 gallons of acetone in 5-gallon containers
Analysis: Container size (5 gal) exceeds 1.3 gal limit
Result: NOT EXEMPT - counts toward Flammable Liquid IB MAQ
```

##### Container-Level Exemptions (System Feature)

In the MAQ system, individual containers can be marked as exempt by a Control Area Admin. This is used for:

- Materials with AHJ-approved exemptions
- Pre-existing conditions (grandfathered materials)
- Special permits or variances
- Research exemptions under specific conditions

**Documentation Required:**
- Written exemption reason
- AHJ approval (if applicable)
- Periodic review date

---

### B.3 Floor Level Factors (IBC Table 414.2.2)

**Authority:** IBC 2024 Table 414.2.2

**Authoritative Floor Level Factors:**

| Floor Level | MAQ Percentage | Factor | Max Control Areas | Notes |
|-------------|----------------|--------|-------------------|-------|
| Ground (1st) | 100% | 1.00 | 4 | Primary reference level |
| 2nd floor | 75% | 0.75 | 3 | |
| 3rd floor | 50% | 0.50 | 2 | |
| 4th floor | 12.5% | 0.125 | 2 | |
| 5th floor | 12.5% | 0.125 | 2 | |
| 6th floor | 12.5% | 0.125 | 2 | |
| 7th floor | 5% | 0.05 | 2 | |
| 8th floor | 5% | 0.05 | 2 | |
| 9th floor | 5% | 0.05 | 2 | |
| 10th+ floor | 5% | 0.05 | 1 | Single control area only |
| 1st basement | 75% | 0.75 | 3 | |
| 2nd basement | 50% | 0.50 | 2 | |
| 3rd+ basement | **Not Permitted** | 0 | 0 | No MAQ storage allowed |

**Visual Floor Level Chart:**

```
Floor 10+  ████████░░░░░░░░░░░░  5% (0.05×)   Max 1 CA
Floor 7-9  ████████░░░░░░░░░░░░  5% (0.05×)   Max 2 CA
Floor 6    ████████████░░░░░░░░  12.5% (0.125×)
Floor 5    ████████████░░░░░░░░  12.5% (0.125×)
Floor 4    ████████████░░░░░░░░  12.5% (0.125×)  Max 2 CA
Floor 3    ██████████████████░░  50% (0.50×)   Max 2 CA
Floor 2    ███████████████████░  75% (0.75×)   Max 3 CA
Floor 1    ████████████████████  100% (1.00×)  Max 4 CA
Basement 1 ███████████████████░  75% (0.75×)   Max 3 CA
Basement 2 ██████████████████░░  50% (0.50×)   Max 2 CA
Basement 3 ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓  NOT PERMITTED
```

**Important Notes:**

1. **Outdoor control areas** do not receive floor level reductions—they use outdoor MAQ tables.

2. **Ground floor determination:** The ground floor is defined as the level of exit discharge. In buildings with sloped sites, the AHJ determines the reference level.

3. **Mixed-level control areas:** If a control area spans multiple floors, use the most restrictive (lowest) percentage.

---

### B.4 Storage vs Use Determination (Field Guide)

#### B.4.1 Decision Tree

```
START: Is the container sealed in original packaging?
  │
  ├─ YES → STORAGE
  │
  └─ NO → Is the container connected to equipment or being actively used?
           │
           ├─ NO → Are contents accessed regularly (daily/weekly)?
           │        │
           │        ├─ NO → STORAGE
           │        │
           │        └─ YES → Is the container open to atmosphere when accessed?
           │                  │
           │                  ├─ YES → USE-OPEN
           │                  │
           │                  └─ NO → USE-CLOSED
           │
           └─ YES → Is the system open to atmosphere during operation?
                     │
                     ├─ YES → USE-OPEN
                     │
                     └─ NO → USE-CLOSED
```

#### B.4.2 Common Scenarios

| Scenario | Classification | Reasoning |
|----------|---------------|-----------|
| Chemicals on storage shelves, sealed | Storage | Not being accessed or used |
| 5-gallon can in use at workstation, cap on | Use-Closed | In use but contained |
| 5-gallon can with pour spout, actively dispensing | Use-Open | Open to atmosphere |
| Chemicals in fume hood, beakers with watch glasses | Use-Closed | Contained in ventilated enclosure |
| Open beaker on bench, no cover | Use-Open | Vapors escaping to room |
| Safety can with self-closing spout | Use-Closed | Self-closing = contained |
| Drum with pump, closed system | Use-Closed | Closed transfer system |
| Dip tank for parts cleaning | Use-Open | Open liquid surface |
| Spray booth operations | Use-Open | Atomized material in air |

#### B.4.3 Aggregate Rule Examples

The IFC aggregate rule states: "The aggregate quantity in use and in storage shall not exceed the quantity listed for storage."

**Example 1 - COMPLIANT:**
```
Flammable Liquid IB (sprinklered building):
  Storage MAQ: 240 gal
  In Storage: 180 gal
  In Use-Closed: 30 gal
  Total: 210 gal
  Status: COMPLIANT (210 < 240)
```

**Example 2 - VIOLATION:**
```
Flammable Liquid IB (sprinklered building):
  Storage MAQ: 240 gal
  In Storage: 200 gal
  In Use-Closed: 50 gal
  Total: 250 gal
  Status: VIOLATION (250 > 240)
```

**Example 3 - Use-Open Check:**
```
Flammable Liquid IB (sprinklered building):
  Use-Open MAQ: 60 gal (30 × 2 for sprinkler)
  In Use-Open: 25 gal
  Status: COMPLIANT (25 < 60)

  BUT ALSO CHECK AGGREGATE:
  Storage MAQ: 240 gal
  In Storage: 150 gal
  In Use-Closed: 40 gal
  In Use-Open: 25 gal
  Total: 215 gal
  Status: COMPLIANT (215 < 240)

  Both checks pass = COMPLIANT
```

#### B.4.4 Resolving Ambiguous Scenarios

Real-world inspections often encounter situations not clearly covered by the decision tree. Use this guidance for ambiguous cases.

##### Time-Based Classification Rule

**The "Intent of Use" Principle:** Classification is based on the **operational intent** of the material, not its momentary state at inspection time.

| Scenario | Classification | Reasoning |
|----------|---------------|-----------|
| Container on bench, cap on, last used 1 hour ago, will be used again today | **Use-Closed** | Part of active work process |
| Container on bench, cap on, not used this week, kept "just in case" | **Storage** | No active use intent |
| Container moved from stockroom to lab for today's experiment, not yet opened | **Use-Closed** | Staged for imminent use |
| Container in lab, hasn't been touched in 30+ days | **Storage** | No evidence of active use |

##### Ambiguous Scenario Resolution Table

| Ambiguous Situation | Resolution | Classification |
|---------------------|------------|----------------|
| Container with cap on sitting at workstation | Ask: "Is this being used today?" If yes → **Use-Closed**. If no → **Storage** |
| Drum with pump attached but not currently pumping | If pump is used daily → **Use-Closed**. If rarely used → **Storage** |
| Chemical in fume hood but hood is off | If hood is normally operated → **Use-Closed**. If hood is just storage location → **Storage** |
| Solvent bottle at wash station | If wash station is active (daily use) → **Use-Closed**. If inactive → **Storage** |
| Containers in "staging area" for experiments | **Use-Closed** (staged for use) |
| Safety can that serves as both storage and dispensing | **Use-Closed** (dual purpose defaults to use) |
| Sealed bottle in refrigerator in active lab | **Storage** (sealed = not in use, refrigeration = preservation) |
| Partially full container awaiting disposal | **Storage** (not in active use) |

##### When Classification is Genuinely Unclear

If after applying the decision tree and resolution table, classification remains unclear:

1. **Default to Use-Closed** for containers in active work areas (labs, production floors)
2. **Default to Storage** for containers in dedicated storage areas (stockrooms, chemical storage rooms)
3. **Document your reasoning** in inspection notes
4. **Ask the occupant** about typical use patterns - their answer helps determine intent

##### MAQ System Tracking Note

**Important:** The MAQ inventory system tracks **total quantities** by hazard class, not separate storage vs. use quantities. For compliance:

- The **Storage MAQ** (larger limit) applies to the **total** of all containers
- The **Use-Open MAQ** (smaller limit) applies only to quantities **actively in open use**
- You must mentally separate use-open quantities during inspection to verify both limits

**Field Calculation Example:**
```
Total Flammable IB in control area: 150 gallons
  - 100 gal in stockroom (sealed)      → Storage
  - 40 gal at workstations (caps on)   → Use-Closed
  - 10 gal in open containers          → Use-Open

Check 1: Total (150 gal) vs Storage MAQ (240 gal) → COMPLIANT
Check 2: Use-Open (10 gal) vs Use-Open MAQ (60 gal) → COMPLIANT
```

##### Special Case: Fluctuating Quantities

**Question:** Should I inspect at peak or typical inventory?

**Answer:** Inspect the **actual quantity present** at inspection time, but:
- Ask about typical and peak quantities
- If peak quantities would exceed MAQ, note this as a potential compliance concern
- Recommend inventory management controls if quantities fluctuate near thresholds

---

### B.5 Approved Storage Verification (Field Guide)

#### B.5.1 Flammable Storage Cabinets

**Required Markings:**
- "FLAMMABLE - KEEP FIRE AWAY" or equivalent
- FM Approved, UL Listed, or ULC Listed label

**Visual Identification:**
```
┌─────────────────────────────────────────────┐
│    ⚠️ FLAMMABLE - KEEP FIRE AWAY ⚠️        │
│                                             │
│   ┌─────────────────────────────────────┐   │
│   │                                     │   │
│   │     [FM APPROVED] or [UL LISTED]    │   │
│   │          Label Location             │   │
│   │                                     │   │
│   └─────────────────────────────────────┘   │
│                                             │
│    Self-closing doors with 3-point latch    │
│    2" liquid-tight door sill                │
│    Double-walled construction               │
└─────────────────────────────────────────────┘
```

**Inspection Checklist:**
- [ ] Listed/labeled (FM, UL, or ULC)
- [ ] Self-closing doors operational
- [ ] Door latches engage automatically
- [ ] 2" liquid-tight sill intact
- [ ] No visible damage or corrosion
- [ ] Not more than 60 gallons per cabinet
- [ ] Not more than 3 cabinets in any single fire area (unless separated by 100 ft)
- [ ] Vented only if connected to proper exhaust system

#### B.5.2 Gas Cabinets

**Required Features:**
- Exhausted enclosure
- Self-closing access ports
- Internal sprinkler (if required)
- Gas detection (recommended)

**Inspection Checklist:**
- [ ] Exhaust system operational (verify airflow)
- [ ] Negative pressure maintained
- [ ] Self-closing ports functional
- [ ] Sprinkler head present (if flammable gases)
- [ ] Gas detection connected to alarm (if present)
- [ ] Emergency shutoff accessible

#### B.5.3 Safety Cans (UL 30)

**Identification:**
- FM Approved or UL Listed label
- Spring-loaded, self-closing spout
- Flame arrester in fill opening
- Pressure relief mechanism

**Common Brands:** Justrite, Eagle, Protectoseal

**Inspection Checklist:**
- [ ] Listed/labeled (FM or UL)
- [ ] Self-closing spout operational
- [ ] Flame arrester present and not damaged
- [ ] No leaks or damage
- [ ] Appropriate for contents (Type I or Type II)

#### B.5.4 Corrosive Storage Cabinets

**Required Features:**
- Labeled for corrosive storage
- Corrosion-resistant interior
- Separate from flammable storage

**Inspection Checklist:**
- [ ] Labeled for corrosive materials
- [ ] No mixing of acids and bases in same cabinet
- [ ] No mixing of oxidizers with organics
- [ ] Spill containment intact
- [ ] Ventilation adequate (if required)

---

### B.6 Field Quantity Measurement Guide

#### B.6.1 Reading Container Labels

**Standard Label Information:**
- Net weight or volume
- Gross weight (may include packaging)
- Concentration (for solutions)

**Convert Label Quantities:**
- Percentages refer to concentration, not volume
- Example: "4L of 70% Nitric Acid" = 4L total, not 2.8L

#### B.6.2 Standard Container Sizes

**Laboratory Containers:**

| Container Type | Typical Sizes | Notes |
|----------------|---------------|-------|
| Reagent bottle | 100mL, 250mL, 500mL, 1L, 2.5L, 4L | Glass or plastic |
| Winchester bottle | 2.5L | Brown glass, light-sensitive |
| Carboy | 5 gal (19L), 15 gal (57L) | Plastic, bulk storage |
| Drum | 5 gal, 30 gal, 55 gal | Steel or plastic |

**Compressed Gas Cylinders:**

| Cylinder Size | Water Capacity | Gas Volume (STP) |
|---------------|----------------|------------------|
| Lecture bottle | 0.2-1.0 L | 10-50 ft³ |
| Size A | ~2 L | ~100 ft³ |
| Size B | ~5 L | ~250 ft³ |
| Size C | ~10 L | ~500 ft³ |
| Size D | ~20 L | ~1,000 ft³ |
| Size E (standard) | ~45 L | ~2,200 ft³ |
| Size G/H (large) | ~80 L | ~4,000 ft³ |
| Size K (K-bottle) | ~130 L | ~6,500 ft³ |

#### B.6.3 Estimation Techniques

**Partial Containers:**
- Visual estimation: 1/4, 1/2, 3/4 full
- For drums: sound test (knock on side)
- For transparent containers: measure liquid level

**Conversion Shortcuts:**
```
1 gallon ≈ 3.8 liters
1 liter ≈ 0.26 gallons
1 pound ≈ 454 grams
1 kilogram ≈ 2.2 pounds
1 cubic foot ≈ 28.3 liters
```

**Quick Conversions:**
```
Liters to Gallons: Divide by 4 (rough)
Grams to Pounds: Divide by 500 (rough)
Kilograms to Pounds: Multiply by 2
Cubic feet to Liters: Multiply by 28
```

#### B.6.4 Unit Conversion Table

| From | To | Multiply By | Example |
|------|----|-------------|---------|
| Liters | Gallons | 0.2642 | 10L = 2.64 gal |
| Gallons | Liters | 3.785 | 5 gal = 18.9L |
| Grams | Pounds | 0.0022 | 1000g = 2.2 lbs |
| Pounds | Grams | 453.6 | 5 lbs = 2268g |
| Kilograms | Pounds | 2.205 | 10 kg = 22 lbs |
| mL | Gallons | 0.000264 | 500mL = 0.13 gal |
| Liters | Cubic feet | 0.0353 | 100L = 3.53 ft³ |
| Cubic feet | Liters | 28.32 | 10 ft³ = 283L |

---

### B.7 Control Area Boundary Determination

#### B.7.1 Fire Barrier Requirements by Floor

| Floor Level | Min Fire Barrier Rating | Max Control Areas |
|-------------|------------------------|-------------------|
| Ground (1st) | 1 hour | 4 |
| 2nd floor | 1 hour | 3 |
| 3rd floor | 1 hour | 2 |
| 4th-9th floor | 1 hour | 2 |
| 10th+ floor | 1 hour | 1 |
| 1st basement | 1 hour | 3 |
| 2nd basement | 2 hours | 2 |
| 3rd+ basement | N/A | Not permitted |

#### B.7.2 Field Identification of Fire Barriers

**Look for:**
- Continuous wall from floor to underside of deck above
- Rated construction documentation (labels on doors/dampers)
- Firestopping at all penetrations
- Fire dampers at duct penetrations

**Visual Indicators:**
```
FIRE BARRIER IDENTIFICATION

Wall Construction:
┌─────────────────────────────────────────────┐
│                                             │
│  Look for:                                  │
│  • Continuous to deck (no gaps above)       │
│  • No unprotected openings                  │
│  • Firestop at penetrations                 │
│  • Fire dampers at ducts                    │
│                                             │
│  ════════════════════════════════════════   │ ← Deck
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
│  ░░░░░░░░░░░ FIRE BARRIER ░░░░░░░░░░░░░░░░ │
│  ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
│  ════════════════════════════════════════   │ ← Floor
└─────────────────────────────────────────────┘
```

#### B.7.3 Fire Door Identification

**Required Labels on Fire Doors:**
- Fire protection rating (20-min, 45-min, 60-min, 90-min)
- Listing agency (UL, FM, WHI)
- Temperature rise rating (if applicable)
- Positive pressure tested (if applicable)

**Door Hardware Requirements:**
- Self-closing device
- Latching hardware
- Coordinator (for pairs)
- Astragal (for pairs, if required)

**Common Fire Door Ratings:**

| Opening Type | Barrier Rating | Required Door Rating |
|--------------|----------------|---------------------|
| Standard opening | 1-hour | 3/4-hour (45-min) |
| Standard opening | 2-hour | 1-1/2-hour (90-min) |
| Corridor opening | 1-hour | 20-minute |

#### B.7.4 Penetration and Firestop Requirements

**All penetrations through fire barriers must be:**
- Firestopped with listed system
- Labeled with firestop system identification
- Free of damage or deterioration

**Look for:**
- Through-penetration firestop systems at pipes, conduits, cables
- Fire dampers at HVAC ducts
- Fire-resistant joint systems at construction joints

#### B.7.5 Control Area Boundary Verification Process

**Step 1: Identify Potential Boundaries**
- Review building plans (if available)
- Walk the perimeter of hazmat storage/use areas
- Look for fire-rated construction

**Step 2: Verify Fire Rating**
- Check door labels for rating
- Look for fire damper access panels at ducts
- Verify wall extends to structure above

**Step 3: Document Boundaries**
- Sketch control area boundaries on floor plan
- Note room numbers included
- Identify any questionable boundaries

**Step 4: Address Issues**
- Missing or damaged doors → Compliance issue
- Holes in barriers → Compliance issue
- Propped-open fire doors → Immediate hazard

#### B.7.6 Common Violations

| Violation | Description | Action |
|-----------|-------------|--------|
| Missing door closer | Fire door cannot self-close | Repair/replace closer |
| Door held open | Fire door wedged or propped | Remove obstruction |
| Damaged firestop | Holes around pipes/cables | Repair with listed system |
| Missing fire damper | No damper at duct penetration | Install rated damper |
| Wall not to deck | Gap between wall and ceiling | Extend wall or fire-rate ceiling |
| Unpermitted penetration | New hole without firestop | Install listed firestop |

#### B.7.7 Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              CONTROL AREA BOUNDARY CHECKLIST                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  FIRE BARRIERS                                                   │
│  □ Walls extend floor-to-deck continuously                       │
│  □ Fire rating appropriate for floor level                       │
│  □ All penetrations firestopped                                  │
│  □ Fire dampers at HVAC penetrations                             │
│                                                                  │
│  FIRE DOORS                                                      │
│  □ Rated labels visible and legible                              │
│  □ Self-closers operational                                      │
│  □ Latches engage properly                                       │
│  □ No obstructions to closure                                    │
│  □ Coordinators working (pairs)                                  │
│                                                                  │
│  PENETRATIONS                                                    │
│  □ Firestop systems labeled                                      │
│  □ No gaps or holes                                              │
│  □ Materials in good condition                                   │
│                                                                  │
│  DOCUMENTATION                                                   │
│  □ Control area boundaries clearly defined                       │
│  □ Room assignments match actual use                             │
│  □ MAQ calculations current                                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### B.8 Field Inspection Checklist

Use this checklist for a complete MAQ inspection:

**Pre-Inspection:**
- [ ] Obtain building fire suppression status
- [ ] Review control area documentation
- [ ] Identify fire code edition in effect
- [ ] Obtain chemical inventory list (if available)

**Control Area Boundaries:**
- [ ] Verify fire barrier ratings
- [ ] Check all fire doors
- [ ] Inspect penetration firestops
- [ ] Document room assignments

**Hazard Identification:**
- [ ] Review SDS for each chemical
- [ ] Classify per fire code hazard classes
- [ ] Note physical states (solid/liquid/gas)
- [ ] Identify any exemptions

**Quantity Measurement:**
- [ ] Count all containers
- [ ] Record container sizes
- [ ] Estimate fill levels
- [ ] Convert to code units (lbs, gal, ft³)

**Storage Verification:**
- [ ] Identify approved storage cabinets
- [ ] Verify cabinet ratings and labels
- [ ] Check cabinet condition
- [ ] Confirm proper segregation

**Calculation:**
- [ ] Determine baseline for each hazard class
- [ ] Apply sprinkler factor (if applicable)
- [ ] Apply floor level factor
- [ ] Apply approved storage factor (if applicable)
- [ ] Calculate final MAQ

**Compliance Determination:**
- [ ] Sum actual quantities by hazard class
- [ ] Compare actual vs MAQ limit
- [ ] Check Use-Open limits separately
- [ ] Verify aggregate rule compliance
- [ ] Document status (Compliant/Near/Over)

---

### B.9 Sprinkler System Field Verification

**Authority:** NFPA 13-2022 (Standard for the Installation of Sprinkler Systems), IFC 2024 Section 903

The 2× sprinkler factor is one of the most impactful MAQ multipliers. This section provides field verification guidance to determine if a building qualifies for the sprinkler increase.

#### B.9.1 Qualification Requirements

**To qualify for the 2× MAQ increase, the building must have:**

1. An **automatic sprinkler system** installed throughout
2. System designed and installed per **NFPA 13** (not NFPA 13R or 13D)
3. System in **working order** and properly maintained
4. No areas of the building **unprotected** by sprinklers

**Systems That DO NOT Qualify:**

| System Type | Why It Doesn't Qualify |
|-------------|------------------------|
| NFPA 13R (Residential) | Limited coverage, designed for life safety only |
| NFPA 13D (One/Two-Family) | Limited coverage, designed for life safety only |
| Partial sprinkler systems | Building not protected "throughout" |
| Standpipe only (no sprinklers) | No automatic suppression |
| Fire extinguishers only | Not automatic suppression |
| Foam systems (unless approved) | Different standard, requires AHJ approval |
| Kitchen hood suppression only | Protects equipment, not building |

#### B.9.2 Visual Identification of Sprinkler Systems

**Sprinkler Head Types:**

```
PENDENT (most common)          UPRIGHT                    SIDEWALL
     ┌─┐                          │                         ┌──┐
     │ │ ← Pipe                   │ ← Pipe                  │  │ ← Pipe
   ┌─┴─┴─┐                     ┌──┴──┐                    ──┤  │
   │ ○○○ │ ← Deflector         │ ○○○ │ ← Deflector          │◄─┤ ← Deflector
   └─────┘                     └─────┘                      └──┘
   Points DOWN                 Points UP                   Points OUT
   (below ceiling)             (above pipes)               (from wall)
```

**What to Look For:**

| Component | Location | Appearance |
|-----------|----------|------------|
| **Sprinkler heads** | Ceiling throughout building | Metal disc with deflector, usually chrome or white |
| **Sprinkler pipes** | Above ceilings, in mechanical rooms | Black steel pipe (1"-6" diameter) with red/orange paint |
| **Riser** | Mechanical room, usually near entrance | Vertical pipe (4"-8") with valves and gauges |
| **Fire Department Connection (FDC)** | Exterior wall near entrance | Siamese connection, usually brass or chrome |
| **Flow switches** | On sprinkler pipes | Electronic device clamped to pipe |
| **Control valves** | Riser room, throughout building | OS&Y valve or butterfly valve with supervisor |

**Fire Department Connection (FDC) Identification:**

```
EXTERIOR WALL
─────────────────────────────
         ┌─────────┐
         │  AUTO   │ ← Sign (required)
         │ SPKLR   │
         └────┬────┘
         ┌────┴────┐
         │ ◯   ◯  │ ← Siamese connection
         │ (caps) │    (2.5" inlets)
         └────────┘
```

#### B.9.3 Sprinkler System Verification Checklist

**Step 1: Confirm System Exists**

- [ ] Sprinkler heads visible in all occupied areas
- [ ] Sprinkler pipes visible in mechanical spaces
- [ ] Fire riser present with gauges showing pressure
- [ ] FDC present on exterior of building

**Step 2: Confirm "Throughout" Coverage**

Walk the entire building and verify sprinkler heads in:

- [ ] All offices and work areas
- [ ] All corridors and lobbies
- [ ] All storage rooms and closets (>24 sq ft)
- [ ] All mechanical/electrical rooms
- [ ] All restrooms (may be exempt in some jurisdictions)
- [ ] All stairwells (may be exempt if enclosed)
- [ ] Loading docks and warehouses
- [ ] Basement levels

**Areas That May Lack Sprinklers (Check Carefully):**

| Area | Common Issue |
|------|--------------|
| Electrical rooms | Sometimes exempt - verify with AHJ |
| Small closets (<24 sq ft) | May be exempt per NFPA 13 |
| Elevator machine rooms | May have different suppression |
| Recent additions/renovations | May not have been retrofitted |
| Attic spaces | May lack coverage if non-combustible |

**Step 3: Confirm System is Operational**

- [ ] Main control valve is OPEN (OS&Y stem extended, or indicator shows "OPEN")
- [ ] System pressure gauges show pressure (typically 50-175 psi)
- [ ] No visible damage to sprinkler heads (paint, corrosion, covers missing)
- [ ] Annual inspection tag current (within 12 months)
- [ ] No "IMPAIRED" or "OUT OF SERVICE" signs posted

#### B.9.4 Requesting Documentation

If visual inspection is inconclusive, request:

| Document | What It Shows |
|----------|---------------|
| **Certificate of Occupancy** | May list sprinkler requirement |
| **Fire sprinkler inspection report** | Annual inspection by licensed contractor |
| **NFPA 13 design documents** | Original system design and coverage |
| **Fire alarm monitoring certificate** | Shows system is monitored |
| **Impairment log** | Any periods when system was out of service |

**Questions to Ask Building Management:**

1. "Is this building fully sprinklered per NFPA 13?"
2. "When was the last annual sprinkler inspection?"
3. "Are there any areas not protected by sprinklers?"
4. "Has the system been impaired recently?"

#### B.9.5 Partial Sprinkler Coverage Scenarios

**Scenario: Building has sprinklers in common areas only**

```
Result: Does NOT qualify for 2× increase
Reason: Not protected "throughout"
Action: Use 1× sprinkler factor for ALL control areas
```

**Scenario: Building has BASEMENT_ONLY sprinklers**

```
Result: ONLY basement control areas MAY qualify (consult AHJ)
        Above-grade control areas do NOT qualify
Action: Strict interpretation = no areas qualify (not "throughout")
        Liberal interpretation = basement floors get 2×, others get 1×
        Document which interpretation is used
```

**Scenario: One floor lacks sprinklers due to renovation**

```
Result: NO areas qualify until renovation is complete
Reason: Building not protected "throughout"
Action: Use 1× sprinkler factor for ALL control areas
```

**Scenario: Sprinkler system is temporarily impaired**

```
Result: Document impairment status
        If impairment >4 hours, fire watch should be in place
        MAQ calculations assume system is operational
Action: Note impairment in inspection report
        Verify fire watch is established per IFC 901.7
```

#### B.9.6 Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────┐
│              SPRINKLER VERIFICATION QUICK CHECK                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  QUALIFIES FOR 2× FACTOR:                                        │
│  ✅ NFPA 13 system throughout building                           │
│  ✅ Heads visible in ALL areas                                   │
│  ✅ Control valves OPEN                                          │
│  ✅ Gauges show pressure                                         │
│  ✅ Annual inspection current                                    │
│  ✅ No impairment notices                                        │
│                                                                  │
│  DOES NOT QUALIFY (use 1× factor):                               │
│  ❌ NFPA 13R or 13D system                                       │
│  ❌ Partial coverage (some areas unprotected)                    │
│  ❌ System impaired or out of service                            │
│  ❌ Standpipe only, no sprinklers                                │
│  ❌ Control valves closed                                        │
│  ❌ No pressure on gauges                                        │
│                                                                  │
│  WHEN IN DOUBT:                                                  │
│  → Request documentation                                         │
│  → Default to 1× (conservative)                                  │
│  → Document reasoning in inspection notes                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### B.10 Occupancy Classification and Determination

**Authority:** IBC 2024 Chapter 3, IFC 2024 Chapter 3

#### B.10.1 Why Occupancy Matters for MAQ

**Critical Clarification:** The MAQ baseline tables in this document are for **Occupancy Group B (Business)**. However, the IFC MAQ tables (Tables 5003.1.1(1) through 5003.1.1(4)) apply **uniformly to all occupancy groups except Group H**.

**Key Point:** MAQ baseline values are the **SAME** for occupancy groups A, B, E, F, I, M, R, S, and U. The differences between occupancies relate to:

1. **Permitted uses** of hazardous materials
2. **Additional operational requirements**
3. **High-Hazard (H) occupancy thresholds** (which trigger reclassification)

**When MAQ is exceeded in ANY occupancy → Building/area must be reclassified as Group H**

#### B.10.2 Occupancy Group Definitions

| Group | Name | Description | Examples |
|-------|------|-------------|----------|
| **A** | Assembly | Gathering of people for civic, social, religious purposes | Theaters, churches, restaurants, stadiums |
| **B** | Business | Office, professional, service transactions | Offices, banks, laboratories, clinics |
| **E** | Educational | Education through 12th grade, 6+ persons | Schools, daycare (>5 children) |
| **F-1** | Factory (Moderate) | Manufacturing/fabrication, moderate hazard | Appliance manufacturing, furniture making |
| **F-2** | Factory (Low) | Manufacturing/fabrication, low hazard | Food processing, glass products |
| **H** | High-Hazard | Materials exceeding MAQ quantities | See Section B.10.3 |
| **I** | Institutional | Supervised/restrained care | Hospitals, nursing homes, jails |
| **M** | Mercantile | Display and sale of merchandise | Retail stores, markets, gas stations |
| **R** | Residential | Sleeping accommodations (not I) | Hotels, apartments, dormitories |
| **S-1** | Storage (Moderate) | Storage of moderate hazard materials | Furniture storage, tire storage |
| **S-2** | Storage (Low) | Storage of low hazard/noncombustible | Parking garages, cold storage |
| **U** | Utility | Miscellaneous structures | Barns, greenhouses, sheds |

#### B.10.3 High-Hazard Occupancy Subgroups

When MAQ is exceeded, the specific H occupancy depends on the hazard type:

| Group | Hazard Type | Examples |
|-------|-------------|----------|
| **H-1** | Detonable materials | Explosives, certain organic peroxides |
| **H-2** | Deflagration/accelerated burning hazard | Flammable gases, Class I liquids (bulk), oxidizers |
| **H-3** | Physical hazards (not H-1 or H-2) | Combustible liquids, flammable solids, oxidizers (Class 2-3) |
| **H-4** | Health hazards | Corrosives, highly toxics, toxics |
| **H-5** | Semiconductor fabrication | HPM (hazardous production materials) per IFC Ch. 27 |

#### B.10.4 Determining Occupancy in the Field

**Step 1: Identify Primary Use**

| If the space is primarily used for... | Likely Occupancy |
|--------------------------------------|------------------|
| Office work, administrative tasks | B |
| Teaching/classrooms (K-12) | E |
| Manufacturing/assembly of products | F-1 or F-2 |
| Research/laboratory work | B |
| Retail sales to public | M |
| Warehousing/storage | S-1 or S-2 |
| Sleeping/residential | R |
| Medical treatment (inpatient) | I |
| Public gathering/entertainment | A |

**Step 2: Check for Mixed Occupancy**

Many buildings contain multiple occupancies. For MAQ purposes:

- Each **control area** is evaluated based on its occupancy
- A laboratory (B) in a manufacturing building (F) uses B occupancy rules
- Common areas follow the predominant building occupancy

**Step 3: Verify with Building Records**

| Document | Information Provided |
|----------|---------------------|
| Certificate of Occupancy | Legal occupancy classification |
| Building permit records | Approved use and occupancy |
| Fire inspection records | Historical occupancy determination |
| Lease agreements | Permitted uses |

#### B.10.5 Occupancy-Specific MAQ Considerations

While MAQ baselines are the same, certain occupancies have additional considerations:

**Group A (Assembly):**
- Lower thresholds for some materials due to life safety concerns
- Egress capacity requirements may limit hazmat storage
- Event permits may impose additional restrictions

**Group B (Business/Laboratory):**
- Most common for MAQ-regulated facilities
- Laboratory operations typically qualify as "use" not "storage"
- Teaching labs may have specific exemptions

**Group E (Educational):**
- Additional restrictions for schools
- Science lab exemptions may apply for small quantities
- Parental notification requirements in some jurisdictions

**Group F (Factory):**
- Process quantities may be separately regulated
- Manufacturing exemptions may apply (check IFC 5003.1.1)
- Quantities in process equipment may be exempt

**Group M (Mercantile):**
- Retail exemptions apply (see Section B.2.11)
- Consumer product exemptions in original packaging
- Display quantity limits may apply

**Group S (Storage):**
- Storage occupancy already accounts for bulk storage
- Retail exemptions apply for wholesale
- Rack storage has additional requirements (NFPA 13 Chapter 20)

#### B.10.6 Common Scenarios

**Scenario 1: Research Laboratory in Office Building**
```
Building: Office tower (Group B)
Space: Chemistry research lab on 3rd floor
Determination: Lab is Group B (research/laboratory)
MAQ applies: Yes, use B occupancy baselines
Floor factor: 50% (3rd floor)
```

**Scenario 2: Chemical Storage Room in Manufacturing Plant**
```
Building: Manufacturing facility (Group F-1)
Space: Dedicated chemical storage room
Determination: Storage room is S-1, but within F-1 building
MAQ applies: Yes, use standard baselines
Note: Control area includes the storage room
```

**Scenario 3: Pharmacy in Hospital**
```
Building: Hospital (Group I-2)
Space: Hospital pharmacy
Determination: Pharmacy is M (mercantile) but within I occupancy
MAQ applies: Yes - retail exemptions DO NOT apply (not retail to public)
Special: Pharmaceutical exemptions may apply (FDA-approved drugs)
```

#### B.10.7 Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│                 OCCUPANCY DETERMINATION GUIDE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  STEP 1: What is the PRIMARY activity?                           │
│  • Office/lab/professional → B                                   │
│  • Manufacturing → F                                             │
│  • Storage → S                                                   │
│  • Retail → M                                                    │
│  • School → E                                                    │
│                                                                  │
│  STEP 2: Do MAQ baselines change by occupancy?                   │
│  • NO - baselines are SAME for A, B, E, F, I, M, R, S, U        │
│  • Exemptions vary (retail, educational, process)                │
│                                                                  │
│  STEP 3: What happens if MAQ is exceeded?                        │
│  • Area must be reclassified as Group H                          │
│  • H-1: Detonation hazards                                       │
│  • H-2: Deflagration hazards                                     │
│  • H-3: Physical hazards                                         │
│  • H-4: Health hazards                                           │
│                                                                  │
│  WHEN IN DOUBT:                                                  │
│  → Check Certificate of Occupancy                                │
│  → Consult AHJ                                                   │
│  → Use most restrictive classification                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### B.11 Ground Plane Determination

**Authority:** IBC 2024 Section 502.1, Definition of "Grade Plane"

#### B.11.1 Why Ground Plane Matters

Floor level factors significantly impact MAQ:
- Ground floor (1st): 100% of baseline
- 2nd floor: 75% of baseline
- 5th floor: 12.5% of baseline
- 1st basement: 75% of baseline

**Incorrect ground plane determination can result in:**
- Using wrong floor factors
- Under/over-stating MAQ limits
- Compliance calculation errors

#### B.11.2 Definition

**Grade Plane:** A reference plane representing the average of finished ground level adjoining the building at exterior walls.

**Ground Floor (Floor 1):** The floor level closest to the grade plane, typically the level of exit discharge to a public way.

#### B.11.3 Simple Buildings (Single Grade Level)

For buildings on flat sites with uniform grade:

```
                    EXTERIOR
─────────────────────────────────────────────────
                 ┌────────────────────┐
                 │     Floor 3       │  ← 3rd floor above ground
                 ├────────────────────┤
                 │     Floor 2       │  ← 2nd floor above ground
                 ├────────────────────┤
  GRADE ═══════▶ │     Floor 1       │  ← Ground floor (exit discharge)
─────────────────┼────────────────────┼──────────
                 │    Basement 1     │  ← 1st floor below ground
                 └────────────────────┘
```

**Determination:** Floor with exit doors to exterior at grade = Ground floor (Floor 1)

#### B.11.4 Sloped Site Buildings

For buildings on sloped sites, multiple floors may have exits to grade:

```
                    UPHILL SIDE                 DOWNHILL SIDE
                         │                            │
                    ─────│────────────────────────────│─────
                         │    ┌──────────────────┐    │
                         │    │     Floor 3      │    │
                         │    ├──────────────────┤    │
            GRADE ═══════╪════│     Floor 2      │    │
                    ─────│────┼──────────────────┤    │
                              │     Floor 1      │════╪══════ GRADE
                         ─────┼──────────────────┤────│─────
                              │    Basement      │    │
                              └──────────────────┘
```

**Which floor is "ground"?**

**Rule:** Calculate the average grade at exterior walls, then identify the floor nearest that level.

**Simplified Method:**
1. If >50% of perimeter is at one level → that level is ground
2. If split roughly equally → use the **lower** level as ground (conservative)
3. Document your determination and reasoning

#### B.11.5 Split-Level Buildings

Buildings with offset floor plates:

```
┌─────────────┐
│  Level 3A   │
├─────────────┼─────────────┐
│  Level 2A   │  Level 2B   │
├─────────────┼─────────────┤
│  Level 1A   │  Level 1B   │ ← Both may exit to grade
└─────────────┴─────────────┘
        ▲             ▲
     Section A    Section B
```

**Approach:**
- Treat each section with different grade levels as potentially separate control areas
- Apply floor factors based on each section's relationship to its grade
- Document which grade reference applies to which areas

#### B.11.6 Parking Structures Below Buildings

```
                    ┌────────────────────┐
                    │     Floor 3       │
                    ├────────────────────┤
                    │     Floor 2       │
                    ├────────────────────┤
   GRADE ═══════▶   │     Floor 1       │  ← Main entrance/lobby
────────────────────┼────────────────────┤────────────────────
                    │   Parking P1      │
                    ├────────────────────┤
                    │   Parking P2      │
                    └────────────────────┘
```

**For MAQ purposes:**
- Floor 1 (lobby level) = Ground floor
- Parking P1 = 1st basement (75% factor)
- Parking P2 = 2nd basement (50% factor)
- Below P2 = Not permitted for MAQ storage

#### B.11.7 Field Determination Process

**Step 1: Identify Exit Discharge Level**
- Where do primary exits discharge to the public way?
- This is typically the "ground floor"

**Step 2: Count Floors Above and Below**
- Floors above exit discharge = above grade (+1, +2, etc.)
- Floors below exit discharge = below grade (-1, -2, etc.)

**Step 3: Handle Ambiguous Cases**
- Multiple exit levels: Use level with **most** exits to public way
- Partial basement exposure: Use **average** of exposed perimeter
- When truly unclear: **Consult AHJ** or use most conservative interpretation

**Step 4: Document Determination**
Record in inspection notes:
- Which floor is designated as ground (Floor 1)
- Reasoning for determination
- Any AHJ consultation

#### B.11.8 Floor Numbering Convention

**MAQ System Floor Numbering:**

| Floor Designation | Floor Above Ground Plane | Description |
|-------------------|-------------------------|-------------|
| 3rd basement | -3 | NOT PERMITTED |
| 2nd basement | -2 | 50% factor |
| 1st basement | -1 | 75% factor |
| Ground/1st floor | 0 or 1 | 100% factor |
| 2nd floor | 2 | 75% factor |
| 3rd floor | 3 | 50% factor |
| 4th-6th floor | 4-6 | 12.5% factor |
| 7th-9th floor | 7-9 | 5% factor |
| 10th+ floor | 10+ | 5% factor |

**Note:** Some systems use 0 for ground, others use 1. Verify which convention your jurisdiction/system uses.

#### B.11.9 Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│               GROUND PLANE DETERMINATION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SIMPLE CASE (flat site):                                        │
│  → Ground floor = level of main entrance/exit discharge          │
│                                                                  │
│  SLOPED SITE:                                                    │
│  → Calculate average grade at perimeter                          │
│  → OR use floor where >50% of exits discharge                    │
│  → OR use lower level (conservative)                             │
│                                                                  │
│  SPLIT LEVEL:                                                    │
│  → May need separate ground reference per section                │
│  → Document determination for each control area                  │
│                                                                  │
│  PARKING BELOW:                                                  │
│  → Parking levels below occupied floors = basements              │
│  → Apply basement floor factors                                  │
│                                                                  │
│  WHEN UNCERTAIN:                                                 │
│  → Consult AHJ                                                   │
│  → Use most conservative (lower floor = ground)                  │
│  → Document reasoning                                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### B.12 Unknown and Unlabeled Chemical Handling

**Authority:** IFC 2024 Section 5003.5, OSHA 29 CFR 1910.1200

#### B.12.1 The Problem

During inspections, you may encounter:
- Containers with missing or illegible labels
- Unknown chemicals without SDS
- Deteriorated labels that cannot be read
- Secondary containers without proper labeling

**These situations require a documented response protocol.**

#### B.12.2 Immediate Actions

**Step 1: Do Not Handle**
- Do not open, move, or disturb unknown containers
- Unknown materials may be unstable, reactive, or highly hazardous
- Contact facility personnel before any action

**Step 2: Attempt Identification**

| Method | Action |
|--------|--------|
| **Ask facility personnel** | "What is this material? Where is the SDS?" |
| **Check nearby containers** | Same type/size containers may provide clues |
| **Look for partial labels** | Product name, manufacturer, lot number |
| **Check inventory records** | Facility should have chemical inventory |
| **Contact manufacturer** | If manufacturer is known, request SDS |

**Step 3: If Identification Fails**

| Situation | Action |
|-----------|--------|
| Personnel can identify verbally | Document their statement, require labeling within 24 hours |
| SDS can be obtained within 24-48 hours | Document gap, schedule follow-up |
| Material cannot be identified | Apply conservative classification (see below) |

#### B.12.3 Conservative Classification Protocol

**When a material cannot be identified, apply the MOST RESTRICTIVE classification that could reasonably apply based on available information.**

**Decision Matrix:**

| Observable Clues | Conservative Classification |
|------------------|----------------------------|
| **Liquid in metal container** | Flammable Liquid IA (most restrictive) |
| **Liquid in glass/plastic, no clues** | Flammable Liquid IA + Corrosive + Toxic |
| **Solid powder, no clues** | Oxidizer 3 + Toxic |
| **Compressed gas cylinder** | Flammable Gas + Toxic (dual classification) |
| **Cylinder with green band** | Oxidizing Gas (typically oxygen) |
| **Cylinder with red band** | Flammable Gas |
| **Any material, completely unknown** | Apply ALL applicable hazard classes |

**Conservative Classification Example:**

```
Unknown: 4L bottle of clear liquid, no label

Step 1: Ask personnel → "Don't know, been here for years"
Step 2: Check inventory → Not listed
Step 3: Conservative classification:
  • Flammable Liquid IA (4L = 1.06 gal) → Count toward FL:IA MAQ
  • Corrosive Liquid (4L = 1.06 gal) → Count toward Corrosive MAQ
  • Toxic Liquid (4L = 1.06 gal) → Count toward Toxic MAQ

Document: "Unidentified liquid, 4L, conservative classification applied.
          Facility required to identify and label within 48 hours."
```

#### B.12.4 Documentation Requirements

**For Each Unknown Chemical, Document:**

| Item | Description |
|------|-------------|
| **Location** | Building, room, shelf/cabinet |
| **Container description** | Type, size, condition |
| **Quantity** | Volume or weight estimate |
| **Observable characteristics** | Color, state, any visible markings |
| **Classification applied** | Which hazard class(es) assigned |
| **Required action** | Identification deadline, responsible party |
| **Follow-up date** | When to verify compliance |

**Sample Documentation:**

```
UNKNOWN CHEMICAL REPORT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Date: ____________    Inspector: ____________    Building: ____________

Location: Room 215, Shelf B-3
Container: 1-gallon plastic jug, white, screw cap
Quantity: Approximately 3/4 full (~0.75 gal)
Observable: Clear liquid, no odor detected, slight yellow tint
Markings: None visible, label area shows adhesive residue

Classification Applied:
  [X] Flammable Liquid IA    Quantity: 0.75 gal
  [X] Corrosive Liquid       Quantity: 0.75 gal
  [ ] Toxic Liquid           Quantity: ______
  [ ] Other: ____________    Quantity: ______

Facility Contact: John Smith, Lab Manager
Required Action: Identify material and properly label by [DATE + 48 hrs]
Follow-up Scheduled: [DATE + 7 days]

Notes: Facility states this may be solvent waste awaiting pickup.
       Recommended immediate disposal through hazmat contractor.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### B.12.5 Labeling Violations

**Missing Labels = Code Violation**

IFC 5003.5 requires hazardous materials to be properly labeled. Unlabeled containers are:

1. **Immediate safety hazard** - Emergency responders cannot identify materials
2. **Code violation** - Subject to citation
3. **Potential criminal violation** - If willful (rare)

**Required Label Information:**

- Product identifier (chemical name)
- Hazard warning (pictograms, signal word)
- Manufacturer name and contact

**Enforcement Options:**

| Severity | Action |
|----------|--------|
| **Minor** (1-2 unlabeled containers) | Warning, require labeling within 24-48 hours |
| **Moderate** (multiple unlabeled) | Notice of violation, deadline for compliance |
| **Severe** (systematic, large quantities) | Stop-work order until labeled, potential citation |
| **Willful/repeat** | Citation with fines, potential referral to OSHA |

#### B.12.6 Abandoned Chemical Protocol

**Definition:** Chemicals left by previous occupants or with no responsible party

**Actions:**

1. **Do not assume ownership** - Current occupant may not be responsible
2. **Document thoroughly** - Quantities, locations, apparent condition
3. **Contact property owner** - Responsibility for disposal
4. **Recommend professional disposal** - Hazmat contractor for unknowns
5. **Do NOT count toward MAQ** if no current occupant claims ownership
   - Instead, issue violation for abandoned hazardous materials

#### B.12.7 Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│              UNKNOWN CHEMICAL PROTOCOL                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. DO NOT HANDLE unknown containers                             │
│                                                                  │
│  2. ATTEMPT TO IDENTIFY:                                         │
│     → Ask personnel                                              │
│     → Check inventory/SDS files                                  │
│     → Look for partial labels                                    │
│     → Contact manufacturer                                       │
│                                                                  │
│  3. IF IDENTIFICATION FAILS, apply conservative classification:  │
│     → Unknown liquid → FL:IA + Corrosive + Toxic                │
│     → Unknown solid → Oxidizer 3 + Toxic                        │
│     → Unknown gas → Flammable Gas + Toxic                       │
│                                                                  │
│  4. DOCUMENT:                                                    │
│     → Location, container, quantity                              │
│     → Classification applied                                     │
│     → Required action and deadline                               │
│                                                                  │
│  5. REQUIRE:                                                     │
│     → Identification within 48 hours                             │
│     → Proper labeling                                            │
│     → Or disposal through hazmat contractor                      │
│                                                                  │
│  6. FOLLOW UP:                                                   │
│     → Verify compliance at deadline                              │
│     → Issue citation if not corrected                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### B.13 Outdoor Control Area Verification

**Authority:** IFC 2024 Section 5003.1.1, Tables 5003.1.1(3) and 5003.1.1(4)

#### B.13.1 Why Outdoor Classification Matters

Outdoor control areas:
- Use **different (higher) MAQ baselines** from Tables 5003.1.1(3) and (4)
- Do **NOT receive floor level reductions**
- Have **different separation requirements**

Incorrectly classifying an indoor area as outdoor can result in using inflated MAQ limits.

#### B.13.2 Definition of "Outdoor"

**IBC/IFC Definition:** An outdoor area is one that is **open to the atmosphere** and provides **natural ventilation**.

**Quantitative Test:** Area must be **at least 75% open** on the perimeter.

#### B.13.3 The 75% Open Rule

**Calculation:**

```
Percent Open = (Open Perimeter Length ÷ Total Perimeter Length) × 100

If Percent Open ≥ 75% → OUTDOOR
If Percent Open < 75% → INDOOR
```

**What Counts as "Open":**

| Feature | Counts as Open? |
|---------|-----------------|
| No wall (open air) | ✅ Yes |
| Chain-link fence | ✅ Yes |
| Open louvers (>50% open area) | ✅ Yes |
| Wire mesh screen | ✅ Yes |
| Roll-up door (when open) | ✅ Yes |
| Roll-up door (when closed) | ❌ No |
| Solid wall | ❌ No |
| Glass curtain wall | ❌ No |
| Closed louvers | ❌ No |
| Partial height wall (>42") | Proportional |

#### B.13.4 Field Measurement Process

**Step 1: Measure Total Perimeter**

Walk the perimeter of the storage/use area and measure total length.

```
Example: Covered storage area
┌─────────────────────────────┐
│                             │
│      STORAGE AREA           │ 50 ft
│                             │
└─────────────────────────────┘
           80 ft

Total Perimeter = 50 + 80 + 50 + 80 = 260 ft
```

**Step 2: Measure Open Sections**

Identify and measure sections that qualify as "open":

```
┌─────────────────────────────┐
│                             │
│      STORAGE AREA           │ 50 ft (solid wall)
│                             │
├── OPEN ──┴── ROLL-UP ──┴────┤
     40 ft       40 ft (closed)

Open sections: 40 ft (open) + 50 ft (open side) + 50 ft (fence) = 140 ft
Closed sections: 80 ft (solid) + 40 ft (roll-up closed) = 120 ft
```

**Step 3: Calculate Percentage**

```
Open Percentage = 140 ÷ 260 × 100 = 53.8%

Result: < 75%, therefore INDOOR (use indoor MAQ tables)
```

#### B.13.5 Common Scenarios

**Scenario 1: Loading Dock with Roll-Up Doors**

```
Loading dock with 3 roll-up doors on one side, solid walls on other 3 sides.
Perimeter: 200 ft total
Doors: 30 ft total width

If doors are OPEN: 30 ft open → 30/200 = 15% → INDOOR
If doors are CLOSED: 0 ft open → 0% → INDOOR

Result: Loading docks are typically INDOOR unless substantially open on multiple sides
```

**Scenario 2: Covered Fuel Island (Gas Station)**

```
Canopy over fuel pumps, open on all sides
Perimeter: 160 ft
Open: 160 ft (no walls)

Open Percentage = 160/160 = 100%

Result: OUTDOOR (use outdoor MAQ tables)
```

**Scenario 3: Partially Enclosed Storage Yard**

```
Storage yard with fence on 3 sides, building wall on 1 side
Perimeter: 400 ft
- Building wall: 100 ft (solid)
- Chain-link fence: 300 ft (open)

Open Percentage = 300/400 = 75%

Result: Exactly 75% → OUTDOOR (meets threshold)
```

**Scenario 4: Covered Walkway/Breezeway**

```
Covered walkway between buildings, open on 2 long sides
Perimeter: 200 ft (10 ft × 80 ft)
- Short ends (10 ft each): 20 ft solid (connected to buildings)
- Long sides (80 ft each): 160 ft open

Open Percentage = 160/200 = 80%

Result: OUTDOOR
```

#### B.13.6 Roof/Canopy Considerations

**Question:** Does having a roof make an area "indoor"?

**Answer:** **NO** - the presence of a roof does not automatically make an area indoor. The 75% perimeter rule applies regardless of roof.

| Configuration | Classification |
|---------------|----------------|
| Open-sided carport (roof, no walls) | OUTDOOR |
| Covered loading dock (3 walls, 1 open) | INDOOR (25% open) |
| Canopy over outdoor storage (no walls) | OUTDOOR |
| Tent/fabric structure (can be opened) | Depends on configuration during use |

#### B.13.7 Seasonal/Operational Variations

**Question:** What if doors are open in summer but closed in winter?

**Answer:** Use the **most restrictive** (closed) configuration for MAQ compliance.

**Reasoning:**
- MAQ limits must be maintained at all times
- Cannot assume doors will be open when incident occurs
- Seasonal variation does not change building classification

**Exception:** If operational procedures REQUIRE doors to remain open during hazmat operations, document this requirement and verify compliance.

#### B.13.8 Documentation

When classifying an area as outdoor, document:

```
OUTDOOR CLASSIFICATION DOCUMENTATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Control Area: ____________    Date: ____________

Total Perimeter: ________ ft

Open Sections:
  North side: ________ ft  Description: ________________
  South side: ________ ft  Description: ________________
  East side:  ________ ft  Description: ________________
  West side:  ________ ft  Description: ________________

Total Open: ________ ft
Percentage Open: ________ %

Classification: [ ] OUTDOOR (≥75%)  [ ] INDOOR (<75%)

Notes: ________________________________________________
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### B.13.9 Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│              OUTDOOR CLASSIFICATION GUIDE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  THE 75% RULE:                                                   │
│  Perimeter must be ≥75% open to qualify as OUTDOOR               │
│                                                                  │
│  COUNTS AS OPEN:              COUNTS AS CLOSED:                  │
│  ✅ No wall                   ❌ Solid wall                       │
│  ✅ Chain-link fence          ❌ Glass/windows                    │
│  ✅ Open louvers (>50%)       ❌ Closed louvers                   │
│  ✅ Wire mesh                 ❌ Roll-up doors (closed)           │
│  ✅ Roll-up doors (open)      ❌ Curtain walls                    │
│                                                                  │
│  ROOF DOES NOT MATTER:                                           │
│  A canopy with open sides = OUTDOOR                              │
│                                                                  │
│  SEASONAL VARIATION:                                             │
│  Use most restrictive (closed) configuration                     │
│                                                                  │
│  IF OUTDOOR:                                                     │
│  → Use Tables 5003.1.1(3) and (4)                               │
│  → NO floor level reduction                                      │
│  → Higher baseline MAQ limits                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

### B.14 Enforcement and Non-Compliance Procedures

**Authority:** IFC 2024 Chapter 1 (Administration), local jurisdiction adoption ordinance

#### B.14.1 Enforcement Authority

Fire code enforcement authority is established by:
1. State adoption of IFC (or state fire code based on IFC)
2. Local adoption ordinance
3. Delegation to fire marshal/fire department
4. Specific enforcement procedures per jurisdiction

**Note:** Enforcement procedures vary by jurisdiction. This section provides general guidance that should be adapted to local requirements.

#### B.14.2 Violation Categories

| Category | Description | Typical Response |
|----------|-------------|------------------|
| **Imminent Hazard** | Immediate threat to life/property | Stop operations, evacuate if necessary |
| **Major Violation** | Significant non-compliance, high risk | Notice of violation, short deadline |
| **Minor Violation** | Technical non-compliance, low risk | Warning, reasonable deadline |
| **Repeat Violation** | Same violation previously cited | Escalated enforcement, fines |

**MAQ-Specific Violation Categories:**

| Violation | Category | Response |
|-----------|----------|----------|
| Exceeds MAQ by >25% | Major | 24-48 hour correction or stop operations |
| Exceeds MAQ by 1-25% | Major | 7-14 day correction deadline |
| At 80-99% of MAQ | Minor | Warning, recommend reduction |
| Missing SDS | Minor | 24-48 hour deadline |
| Unlabeled containers | Minor/Major | Depends on quantity and hazard |
| Unapproved storage | Major | Immediate correction required |
| Sprinkler impairment | Major | Fire watch required, expedite repair |

#### B.14.3 Inspection Documentation

**Required Elements for All Inspections:**

| Element | Description |
|---------|-------------|
| **Date and time** | When inspection occurred |
| **Inspector identification** | Name, badge/ID number, contact |
| **Facility information** | Name, address, contact person |
| **Areas inspected** | Buildings, control areas, rooms |
| **Findings** | Compliant items and violations |
| **Photographs** | Document violations (if policy allows) |
| **Signatures** | Inspector and facility representative |

**MAQ-Specific Documentation:**

| Element | Description |
|---------|-------------|
| **Control area identification** | Name/number of each control area |
| **Quantities observed** | By hazard class and physical state |
| **MAQ limits calculated** | Show factors applied |
| **Compliance status** | Compliant, Near, Over for each |
| **Calculation worksheet** | Attach completed worksheet |

#### B.14.4 Non-Compliance Response Procedures

**Step 1: Document the Violation**

```
MAQ VIOLATION DOCUMENTATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Date: ____________    Inspector: ____________    Case #: ____________

Facility: ________________________________
Address: ________________________________
Contact: ________________________________

VIOLATION DESCRIPTION:
Control Area: ____________
Hazard Class: ____________
Physical State: ____________
MAQ Limit: ____________
Actual Quantity: ____________
Overage: ____________ (____%)

Code Section Violated: IFC 5003.1.1

Violation Category: [ ] Imminent Hazard  [ ] Major  [ ] Minor

REQUIRED CORRECTIVE ACTION:
__________________________________________________________________
__________________________________________________________________

DEADLINE: ____________

CONSEQUENCES OF NON-COMPLIANCE:
__________________________________________________________________
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

**Step 2: Issue Notice of Violation**

| Content | Description |
|---------|-------------|
| Violation description | What was found non-compliant |
| Code reference | Specific section violated (e.g., IFC 5003.1.1) |
| Required action | What must be done to correct |
| Deadline | Date by which correction must be complete |
| Consequences | What happens if not corrected |
| Appeal rights | How to contest the violation |

**Step 3: Set Appropriate Deadline**

| Violation Severity | Typical Deadline |
|--------------------|------------------|
| Imminent hazard | Immediate (stop operations) |
| Major MAQ exceedance (>25%) | 24-72 hours |
| Major MAQ exceedance (1-25%) | 7-14 days |
| Minor violation | 30 days |
| Administrative (paperwork) | 30-60 days |

**Step 4: Schedule Re-inspection**

- Re-inspection typically scheduled at or shortly after deadline
- Verify all violations have been corrected
- Document compliance or continued non-compliance

#### B.14.5 Escalation Procedures

**If Violation is Not Corrected by Deadline:**

| Step | Action |
|------|--------|
| **1. Second Notice** | Reissue violation with shorter deadline |
| **2. Citation** | Issue formal citation with fine |
| **3. Administrative Hearing** | Schedule hearing if contested |
| **4. Stop Operations** | Order cessation of hazmat operations |
| **5. Legal Action** | Refer to city/county attorney |

**Fine Schedules (Example - Varies by Jurisdiction):**

| Violation Type | First Offense | Second Offense | Third+ Offense |
|----------------|---------------|----------------|----------------|
| Minor | Warning or $100-250 | $250-500 | $500-1,000 |
| Major | $250-500 | $500-1,000 | $1,000-5,000 |
| Imminent Hazard | $500-2,500 | $2,500-10,000 | $10,000+ |
| Per day continued | +$100-500/day | +$250-1,000/day | +$500-2,500/day |

**Note:** Fine amounts vary significantly by jurisdiction. Verify local fee schedule.

#### B.14.6 Stop Operations Authority

**When Stop Operations May Be Ordered:**

- MAQ exceeded by significant amount (>50%)
- Imminent hazard to occupants or public
- Sprinkler system impaired without fire watch
- Repeated non-compliance
- Refusal to allow inspection

**Stop Operations Procedure:**

1. **Verbal notification** to responsible party on-site
2. **Written order** posted on premises
3. **Notification to owner** if different from operator
4. **Documentation** in inspection file
5. **Verification** that operations have ceased
6. **Clearance inspection** before operations may resume

**Stop Operations Order Content:**

```
STOP OPERATIONS ORDER
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ORDER NUMBER: ____________    DATE: ____________

FACILITY: ________________________________
ADDRESS: ________________________________

OPERATIONS ORDERED STOPPED:
[ ] All hazardous materials operations
[ ] Specific operations: ________________________________

REASON:
__________________________________________________________________
__________________________________________________________________

CODE AUTHORITY: IFC Section 110.1, [Local Code Reference]

REQUIREMENTS TO RESUME OPERATIONS:
__________________________________________________________________
__________________________________________________________________

THIS ORDER IS EFFECTIVE IMMEDIATELY.

Failure to comply may result in additional fines, legal action,
and/or referral for criminal prosecution.

Inspector: ________________    Badge #: ____________
Contact: ________________________________

APPEAL: You may appeal this order to [AHJ contact] within [X] days.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

#### B.14.7 Compliance Assistance Options

Before escalating enforcement, consider offering:

| Option | Description |
|--------|-------------|
| **Compliance plan** | Allow facility to develop phased reduction plan |
| **Alternative methods** | Suggest approved storage, separation into multiple control areas |
| **Variance/modification** | For legitimate operational needs (requires formal application) |
| **Consultation** | Connect with fire prevention bureau for guidance |
| **Additional time** | For complex situations requiring capital improvements |

#### B.14.8 Record Keeping Requirements

**Maintain Records Of:**

| Document | Retention Period |
|----------|------------------|
| Inspection reports | Minimum 3 years |
| Notices of violation | Until resolved + 3 years |
| Citations issued | Permanent |
| Stop operations orders | Permanent |
| Correspondence | 3 years |
| Photographs | 3 years minimum |
| Compliance verifications | 3 years |

#### B.14.9 Quick Reference

```
┌─────────────────────────────────────────────────────────────────┐
│              ENFORCEMENT QUICK REFERENCE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  VIOLATION FOUND:                                                │
│  1. Document - quantities, locations, hazard classes             │
│  2. Photograph - if policy permits                               │
│  3. Calculate - verify MAQ exceedance                            │
│  4. Categorize - imminent/major/minor                            │
│  5. Issue - notice of violation                                  │
│  6. Set deadline - appropriate to severity                       │
│  7. Schedule - re-inspection                                     │
│                                                                  │
│  DEADLINES:                                                      │
│  • Imminent hazard: Immediate                                    │
│  • Major (>25% over): 24-72 hours                               │
│  • Major (1-25% over): 7-14 days                                │
│  • Minor: 30 days                                                │
│                                                                  │
│  ESCALATION:                                                     │
│  Warning → Notice → Citation → Stop Ops → Legal                  │
│                                                                  │
│  ALWAYS DOCUMENT:                                                │
│  • What was found                                                │
│  • Code section violated                                         │
│  • Required correction                                           │
│  • Deadline given                                                │
│  • Facility response                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

*Document Version: 1.1*
*Last Updated: December 2024*
*Based on 2024 International Fire Code (IFC) and International Building Code (IBC)*

*Document maintained by the Risk & Safety MAQ Development Team*

# MAQ Inspector's Guide: A Complete Reference

**Maximum Allowable Quantities for Fire Code Compliance**

*A comprehensive guide for fire safety inspectors and compliance officers*

---

## Table of Contents

1. [Introduction: What is MAQ?](#1-introduction-what-is-maq)
2. [Why MAQ Matters](#2-why-maq-matters)
3. [Regulatory Framework](#3-regulatory-framework)
4. [Core Concepts](#4-core-concepts)
5. [The MAQ Calculation Formula](#5-the-maq-calculation-formula)
6. [Step-by-Step: Calculating MAQ](#6-step-by-step-calculating-maq)
7. [Factor Deep Dives](#7-factor-deep-dives)
8. [Storage vs. Use Conditions](#8-storage-vs-use-conditions)
9. [Determining Compliance](#9-determining-compliance)
10. [Worked Examples](#10-worked-examples)
11. [Special Cases and Exceptions](#11-special-cases-and-exceptions)
12. [Common Mistakes to Avoid](#12-common-mistakes-to-avoid)
13. [Reference Tables](#13-reference-tables)
14. [Glossary](#14-glossary)

---

## 1. Introduction: What is MAQ?

**Maximum Allowable Quantity (MAQ)** is the maximum amount of a hazardous material permitted in a **control area** before a building must be classified as a **High-Hazard (Group H) occupancy**.

Think of MAQ as a threshold. Below it, you can store and use hazardous materials in a standard building. Above it, you need special construction, fire protection systems, and operational requirements that significantly increase costs and regulatory burden.

### The Simple Equation

```
If actual quantity < MAQ  →  Standard occupancy allowed
If actual quantity ≥ MAQ  →  High-Hazard occupancy required
```

### Key Insight for Inspectors

MAQ is not about whether a material is "safe" or "dangerous." It's about **quantity thresholds** that trigger different building code requirements. A small amount of even highly toxic material might be under MAQ, while large quantities of relatively benign materials might exceed it.

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

A **control area** is a space within a building where hazardous materials are stored, used, or handled. Control areas are separated from each other by fire-rated construction.

**Key Points:**
- Each control area has its own MAQ limits
- Buildings can have multiple control areas per floor
- The number of control areas allowed depends on floor level
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

### 4.3 Physical States

MAQ limits are specified separately for each physical state:

| State | Primary Unit | Typical Materials |
|-------|--------------|-------------------|
| **Solid** | Pounds (lb) | Powders, pellets, crystite |
| **Liquid** | Gallons (gal) | Solvents, acids, fuels |
| **Gas** | Cubic feet (ft³) | Compressed gases, vapors |

**Important:** A single hazard class may have different MAQ limits for solid, liquid, and gas forms.

### 4.4 Occupancy Classifications

Buildings are classified by use. The most common for MAQ purposes:

| Group | Type | Example |
|-------|------|---------|
| **B** | Business | Offices, laboratories |
| **F** | Factory | Manufacturing facilities |
| **S** | Storage | Warehouses |
| **E** | Educational | Schools, universities |
| **H** | High-Hazard | Facilities exceeding MAQ |
| **A** | Assembly | Auditoriums |
| **M** | Mercantile | Retail stores |

---

## 5. The MAQ Calculation Formula

### The Master Formula

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

### Factor Application

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

---

## 6. Step-by-Step: Calculating MAQ

### Step 1: Identify the Hazard Class

Determine the hazard classification of the material. This comes from:
- Safety Data Sheet (SDS) Section 2: Hazard Identification
- Chemical inventory system classification
- Fire code hazard class mapping

**Example:** Acetone
- Flammable Liquid, Class IB (flash point < 73°F, boiling point ≥ 100°F)

### Step 2: Identify the Physical State

Determine if the material is solid, liquid, or gas at normal temperature and pressure (NTP: 68°F, 1 atm).

**Example:** Acetone is a liquid at NTP.

### Step 3: Look Up the Baseline MAQ

Find the baseline value in IFC Table 5003.1.1(1) or (2).

**Example:** Flammable Liquid Class IB
- Storage: 120 gallons
- Use-Closed: 120 gallons
- Use-Open: 30 gallons

### Step 4: Determine Sprinkler Factor

**Question:** Is the building equipped with an automatic sprinkler system **throughout** in accordance with NFPA 13?

| Condition | Factor |
|-----------|--------|
| Fully sprinklered (NFPA 13) | **2×** (100% increase) |
| Not sprinklered | **1×** (no increase) |
| Partially sprinklered | **1×** (no increase - must be "throughout") |
| NFPA 13R or 13D system | **1×** (residential systems don't qualify) |

**Critical:** The building must be protected **throughout**. A building with sprinklers only in the basement does NOT qualify for the 2× increase on any floor.

### Step 5: Determine Floor Level Factor

Look up the percentage in IBC Table 414.2.2 based on the control area's floor level.

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

**Important:** Outdoor control areas do NOT get floor level reductions (they use separate outdoor tables).

### Step 6: Determine Approved Storage Factor

**Question:** Are the materials stored in approved storage cabinets, gas cabinets, exhausted enclosures, or listed safety cans?

| Condition | Factor |
|-----------|--------|
| In approved cabinet | **2×** (100% increase) |
| Not in approved cabinet | **1×** (no increase) |

**Approved Cabinet Requirements (IFC 5003.9.10):**
- Double-walled construction with 1.5" air space
- Liquid-tight door sill (2" minimum)
- Self-closing, self-latching doors
- Labeled "HAZARDOUS—KEEP FIRE AWAY"

### Step 7: Calculate Final MAQ

Multiply all factors together.

**Example Calculation:**

```
Material: Flammable Liquid Class IB (e.g., Acetone)
Location: 2nd floor of a fully sprinklered building
Storage: In standard shelving (not approved cabinet)

Baseline:           120 gallons
× Sprinkler:        2 (fully sprinklered)
× Floor Level:      0.75 (2nd floor = 75%)
× Approved Storage: 1 (not in cabinet)
────────────────────────────────────
Final MAQ:          180 gallons
```

### Step 8: Compare to Actual Quantity

Sum all quantities of that hazard class in the control area and compare.

```
Actual quantity in control area: 45 gallons
MAQ:                             180 gallons

45 < 180 → COMPLIANT
```

---

## 7. Factor Deep Dives

### 7.1 Sprinkler Factor Details

The 2× sprinkler increase is one of the most impactful factors, but it has strict requirements:

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

**Basement-Only Sprinklers:**

Some buildings have sprinklers only in basement levels. In these cases:
- Basement floors may or may not qualify (consult your AHJ)
- Upper floors do NOT get the sprinkler increase
- The strict interpretation is that NO floors qualify since it's not "throughout"

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

**Mixed-Level Control Areas:**

If a control area spans multiple floors:
- Use the floor with the most restrictive (lowest) percentage
- Or calculate based on where materials are actually located

### 7.3 Approved Storage Factor Details

The 2× increase for approved storage recognizes that proper cabinets contain spills and slow fire spread.

**For Liquids - Approved Flammable Storage Cabinets:**
- Must meet NFPA 30 or be listed to UL 1275
- Maximum 60 gallons per cabinet
- Maximum 3 cabinets per fire area (unless separated)
- Self-closing doors required

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

---

## 8. Storage vs. Use Conditions

### The Three Usage Categories

The fire code distinguishes between three conditions with **different MAQ limits**:

| Category | Description | Vapor Exposure |
|----------|-------------|----------------|
| **Storage** | Sealed containers, not being used | None |
| **Use-Closed** | In use but contained (closed systems, fume hoods) | Minimal |
| **Use-Open** | Open to atmosphere (pouring, mixing, open containers) | Maximum |

### Why This Matters

Use-open conditions present higher risk:
- Vapors can accumulate to flammable concentrations
- Spills spread faster from open containers
- Ignition sources more likely to contact vapors

### Typical MAQ Ratios

| Material | Storage | Use-Closed | Use-Open | Ratio |
|----------|---------|------------|----------|-------|
| Flammable Liquid IA | 30 gal | 30 gal | 10 gal | 3:1 |
| Flammable Liquid IB/IC | 120 gal | 120 gal | 30 gal | 4:1 |
| Corrosive Liquid | 500 gal | 500 gal | 100 gal | 5:1 |
| Toxic Solid | 500 lb | 500 lb | 125 lb | 4:1 |

### The Aggregate Rule

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

### Determining Use Condition

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

## 9. Determining Compliance

### Compliance Status Levels

| Status | Condition | Action Required |
|--------|-----------|-----------------|
| **COMPLIANT** | Actual < MAQ | None - within limits |
| **NEAR THRESHOLD** | Actual ≥ 80% of MAQ | Warning - reduce quantities or prepare for H-occupancy |
| **OVER THRESHOLD** | Actual ≥ MAQ | Violation - immediate reduction required or reclassification |
| **EXEMPT** | Control area exempt | Document exemption basis |

**Note:** The 80% "near threshold" warning is typically an institutional policy, not a code requirement. However, it's an excellent best practice.

### No Limit (NL) Designation

Some hazard class/physical state combinations show "NL" (No Limit) in the code tables.

**What NL Means:**
- No maximum quantity limit applies
- Material is still a hazard but doesn't drive H-occupancy
- Common for less volatile or less reactive materials

**Example:** Combustible Liquid Class IIIB often has NL for storage because of its high flash point (200°F+).

### Not Applicable (N/A) Designation

**What N/A Means:**
- That physical state doesn't exist or isn't regulated for that hazard class
- Don't track this combination

**Example:** Flammable gases have N/A for solid and liquid states - they're only gases.

### Control Area vs. Building Compliance

**Control Area Level:**
- Each control area is evaluated independently
- One non-compliant control area doesn't affect others

**Building Level:**
- If ANY control area exceeds MAQ, the building needs H-occupancy features
- Building status = worst control area status
- "One bad apple" principle applies

---

## 10. Worked Examples

### Example 1: Basic Calculation

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

### Example 2: Basement with Partial Sprinklers

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

### Example 3: High Floor with Reduced MAQ

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

### Example 4: Multiple Hazard Classes

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

### Example 5: Storage vs. Use-Open

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

---

## 11. Special Cases and Exceptions

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

### 11.2 Exempt Materials

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

### 11.3 Control Area Exemptions

Entire control areas may be exempt from MAQ limits:

**Possible Exemption Reasons:**
- H-occupancy classification already applied
- Standalone building with no exposure risk
- Specialized permitted facility
- Historical exemption (grandfathered)

**Documentation Required:**
- Written exemption with basis
- AHJ approval
- Periodic review

### 11.4 Liquefied Gases

Gases that are liquid under pressure require special consideration:

**Examples:**
- Liquefied Petroleum Gas (LPG)
- Liquefied Natural Gas (LNG)
- Refrigerants

**MAQ Determination:**
- Use the gas MAQ limits
- Quantity typically expressed in water capacity (gallons) or gas equivalent (cubic feet)
- Conversion: 1 gallon liquid ≈ 270 cubic feet gas (varies by material)

### 11.5 Combination of Materials

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

## 12. Common Mistakes to Avoid

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

## 13. Reference Tables

### Quick Reference: Common Baseline MAQs

#### Physical Hazards (Storage - Unsprinklered)

| Hazard Class | Solid (lb) | Liquid (gal) | Gas (ft³) |
|--------------|------------|--------------|-----------|
| Combustible Liquid II | — | 120 | — |
| Combustible Liquid IIIA | — | 330 | — |
| Combustible Liquid IIIB | — | NL | — |
| Flammable Liquid IA | — | 30 | — |
| Flammable Liquid IB | — | 120 | — |
| Flammable Liquid IC | — | 120 | — |
| Flammable Solid | 125 | — | — |
| Flammable Gas | — | — | 1,000 |
| Oxidizer Class 1 | 4,000 | 4,000 | NL |
| Oxidizer Class 2 | 250 | 250 | — |
| Oxidizer Class 3 | 10 | 10 | — |
| Oxidizer Class 4 | 1* | 1* | — |
| Organic Peroxide UD | 1* | 1* | — |
| Pyrophoric | 4* | 4* | 50* |
| Unstable Reactive 3 | 5 | 5 | — |
| Unstable Reactive 4 | 1* | 1* | — |
| Water Reactive 2 | 50 | 50 | — |
| Water Reactive 3 | 5 | 5 | — |

*Sprinkler protection required for any quantity

#### Health Hazards (Storage - Unsprinklered)

| Hazard Class | Solid (lb) | Liquid (gal) | Gas (ft³) |
|--------------|------------|--------------|-----------|
| Corrosive | 5,000 | 500 | 810 |
| Toxic | 500 | 500 | 810 |
| Highly Toxic | 10 | 10 | 20* |

*Sprinkler protection required for any quantity

### Floor Level Factors

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

### Unit Conversions

| Conversion | Factor |
|------------|--------|
| Grams → Pounds | × 0.00220462 |
| Kilograms → Pounds | × 2.20462 |
| Liters → Gallons | × 0.264172 |
| Liters → Cubic Feet | × 0.0353147 |
| Cubic Meters → Cubic Feet | × 35.3147 |
| Gallons → Pounds (water) | × 8.34 |
| Gallons → Pounds (typical liquid) | × 10 (code default) |

---

## 14. Glossary

**AHJ (Authority Having Jurisdiction):** The organization, office, or individual responsible for enforcing fire code requirements.

**Approved Storage Cabinet:** A cabinet specifically designed for hazardous material storage, meeting NFPA 30 requirements or listed to UL 1275.

**Baseline MAQ:** The maximum allowable quantity as listed in code tables, before any factors are applied.

**Control Area:** A space within a building where hazardous materials not exceeding the MAQ are stored, dispensed, used, or handled.

**Cubic Feet (ft³):** Standard unit of volume for gases at normal temperature and pressure.

**Fire Code:** A set of regulations adopted by a jurisdiction governing fire prevention, fire protection, and life safety.

**Flash Point:** The minimum temperature at which a liquid gives off vapor sufficient to form an ignitable mixture with air.

**Gallon (gal):** Standard unit of volume for liquids in fire codes.

**Ground Plane:** The finished floor level of a building at the level of exit discharge.

**Hazard Class:** A category of hazardous material based on its physical or health properties.

**High-Hazard Occupancy (Group H):** A building classification for structures containing quantities of hazardous materials exceeding MAQ limits.

**MAQ (Maximum Allowable Quantity):** The maximum amount of a hazardous material permitted per control area before High-Hazard classification is required.

**NFPA 13:** Standard for the Installation of Sprinkler Systems - referenced for the 100% MAQ increase.

**NL (No Limit):** Designation meaning no maximum quantity limit applies.

**N/A (Not Applicable):** Designation meaning the combination doesn't apply or isn't tracked.

**Physical State:** Whether a material is solid, liquid, or gas at normal temperature and pressure.

**Pound (lb):** Standard unit of weight for solids in fire codes.

**Sprinkler System:** An automatic fire suppression system with heat-activated sprinkler heads connected to a water supply.

**Use-Closed:** Using hazardous materials in a system that doesn't normally allow vapors to escape to the atmosphere.

**Use-Open:** Using hazardous materials in a way that allows vapors to escape to the atmosphere.

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

*Document Version: 1.0*
*Last Updated: December 2024*
*Based on 2024 International Fire Code (IFC) and International Building Code (IBC)*

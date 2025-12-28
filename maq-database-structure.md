---
title: MAQ MongoDB Database Structure
date: 2024-12-24
tags:
  - mongodb
  - maq
  - database
  - schema
  - architecture
---

# MAQ MongoDB Database Structure

This document maps out the MongoDB collections, schemas, and relationships used in the MAQ (Maximum Allowable Quantity) application.

## Collections Overview

| Collection | Purpose | Primary Service |
|------------|---------|-----------------|
| `fireCode` | Fire code regulations with hazard class limits | `FireCodeService.ts` |
| `controlArea` | Building zones with MAQ calculations | `ControlAreaService.ts` |
| `building` | Building/floor/room hierarchy | External data source |
| `buildings` | Building attachments (files) | `BuildingService.ts` |
| `projections` | User-saved fire code projections | `ProjectionService.ts` |
| `fireCodeHistory` | Audit trail for deleted fire codes | `FireCodeService.ts` |

## Entity Relationship Diagram

```mermaid
erDiagram
    FIRE_CODE ||--o{ OCCUPANCY : contains
    OCCUPANCY ||--o{ HAZARD_CLASS_DEF : defines
    HAZARD_CLASS_DEF ||--o{ RULE : has
    FIRE_CODE ||--o{ HAZARD_CLASS_MAPPING : maps

    BUILDING ||--o{ FLOOR : contains
    FLOOR ||--o{ ROOM : contains
    BUILDING ||--o{ BUILDING_ATTACHMENT : has

    CONTROL_AREA }o--|| BUILDING : "references"
    CONTROL_AREA }o--|| FIRE_CODE : "uses"
    CONTROL_AREA ||--o{ HAZARD_CLASS : tracks
    ROOM }o--o| CONTROL_AREA : "assigned to"

    PROJECTION }o--|| FIRE_CODE : "references"
    FIRE_CODE_HISTORY }o--|| FIRE_CODE : "audits"

    FIRE_CODE {
        string _id PK "e.g., CFC2016"
        string name
        boolean hmis_report
        array occupancies
        array hazardClassMappings
        date createdDate
        date lastUpdatedDate
    }

    OCCUPANCY {
        string id PK
        string name
        array rules
        array hazardClasses
    }

    HAZARD_CLASS_DEF {
        string id PK
        string name
        boolean isHealthHazard
        object solid
        object liquid
        object gas
        array rules
    }

    BUILDING {
        string _id PK
        string buildingKey
        string name
        string campusCode
        string fireCodeId FK
        boolean fireSuppressionCoverage
        string fireSuppressionType
        array floors
    }

    FLOOR {
        string id PK
        string floorKey
        string name
        string groundPlane
        array rooms
    }

    ROOM {
        string id PK
        string roomKey
        string name
        string roomNumber
        string controlAreaId FK
    }

    CONTROL_AREA {
        string _id PK
        string name
        string occupancy
        number floorAboveGroundPlane
        boolean isOutdoor
        object building
        array hazardClasses
        string status
    }

    HAZARD_CLASS {
        string id PK
        string name
        object solid
        object liquid
        object gas
        array inventories
    }

    BUILDING_ATTACHMENT {
        ObjectId _id PK
        string buildingId FK
        string attachmentKey
        string name
        boolean deleted
    }

    PROJECTION {
        ObjectId _id PK
        string name
        string fireCode FK
        string occupancy
    }

    FIRE_CODE_HISTORY {
        string _id PK
        string fireCodeId FK
        string fireCodeName
        object deletedBy
    }
```

## Data Flow Diagram

```mermaid
flowchart TB
    subgraph External["External Data Sources"]
        CHEM[Chemical Inventory System]
        BLDG_SRC[Building Data Source]
    end

    subgraph MongoDB["MongoDB Collections"]
        FC[(fireCode)]
        CA[(controlArea)]
        BLD[(building)]
        ATT[(buildings<br/>attachments)]
        PROJ[(projections)]
        HIST[(fireCodeHistory)]
    end

    subgraph Services["Server Services"]
        FCS[FireCodeService]
        CAS[ControlAreaService]
        BLS[BuildingService]
        PRS[ProjectionService]
    end

    subgraph GraphQL["GraphQL Layer"]
        RES[resolvers.ts]
    end

    subgraph Client["React Client"]
        UI[MAQ UI]
    end

    BLDG_SRC --> BLD
    CHEM --> CA

    FCS --> FC
    FCS --> HIST
    CAS --> CA
    BLS --> ATT
    PRS --> PROJ

    RES --> FCS
    RES --> CAS
    RES --> BLS
    RES --> PRS

    UI <--> RES

    CA -.->|references| FC
    CA -.->|references| BLD
    BLD -.->|has| ATT
```

## MAQ Calculation Flow

```mermaid
flowchart LR
    subgraph Inputs
        FC[Fire Code<br/>Hazard Limits]
        INV[Chemical<br/>Inventory]
        CA_CFG[Control Area<br/>Configuration]
    end

    subgraph Factors["MAQ Factors Applied"]
        SPR[Sprinkler<br/>Coverage]
        FLR[Floor<br/>Position]
        OUT[Outdoor<br/>Location]
        APP[Approved<br/>Storage]
    end

    subgraph Calculation
        LIMIT[Calculate<br/>Adjusted Limit]
        ACTUAL[Sum Actual<br/>Quantities]
        COMPARE[Compare<br/>Limit vs Actual]
    end

    subgraph Output
        STATUS[Compliance<br/>Status]
    end

    FC --> LIMIT
    CA_CFG --> LIMIT
    SPR --> LIMIT
    FLR --> LIMIT
    OUT --> LIMIT
    APP --> LIMIT

    INV --> ACTUAL
    LIMIT --> COMPARE
    ACTUAL --> COMPARE
    COMPARE --> STATUS
```

## Collection Schemas

### 1. `fireCode`

```mermaid
classDiagram
    class FireCode {
        +String _id
        +String name
        +Boolean hmis_report
        +FireCodeOccupancy[] occupancies
        +HazardClassMapping[] hazardClassMappings
        +Date createdDate
        +Date lastUpdatedDate
        +Object createdBy
        +Object lastUpdatedBy
    }

    class FireCodeOccupancy {
        +String id
        +String name
        +FireCodeOccupancyRule[] rules
        +FireCodeHazardClass[] hazardClasses
    }

    class FireCodeOccupancyRule {
        +String operator
        +Number[] appliedToFloors
        +Number percentage
    }

    class FireCodeHazardClass {
        +String id
        +String name
        +Boolean isHealthHazard
        +FireCodeHazardState solid
        +FireCodeHazardState liquid
        +FireCodeHazardState gas
        +FireCodeHazardClassRule[] rules
    }

    class FireCodeHazardState {
        +Number baseline
        +String units
        +Boolean isNoLimit
        +Boolean isNotApplicable
        +Boolean reportAsLiquid
    }

    class FireCodeHazardClassRule {
        +String id
        +FireCodeHazardClassRuleCondition[] conditions
        +FireCodeHazardClassRuleState solid
        +FireCodeHazardClassRuleState liquid
        +FireCodeHazardClassRuleState gas
    }

    class FireCodeHazardClassRuleCondition {
        +String criteria
        +String operator
        +String value
    }

    class HazardClassMapping {
        +String name
        +ChemicalBandDisplay[] must
        +ChemicalBandDisplay[] should
        +ChemicalBandDisplay[] mustNot
    }

    FireCode "1" --> "*" FireCodeOccupancy
    FireCode "1" --> "*" HazardClassMapping
    FireCodeOccupancy "1" --> "*" FireCodeOccupancyRule
    FireCodeOccupancy "1" --> "*" FireCodeHazardClass
    FireCodeHazardClass "1" --> "3" FireCodeHazardState
    FireCodeHazardClass "1" --> "*" FireCodeHazardClassRule
    FireCodeHazardClassRule "1" --> "*" FireCodeHazardClassRuleCondition
```

### 2. `controlArea`

```mermaid
classDiagram
    class ControlArea {
        +String _id
        +String name
        +String occupancy
        +Number floorAboveGroundPlane
        +Boolean isOutdoor
        +ApprovedStorage[] approvedStorage
        +FireSuppressionOverride fireSuppressionOverride
        +ControlAreaExempt exemption
        +ControlAreaBuilding building
        +HazardClass[] hazardClasses
        +ComplianceStatus status
        +String notes
    }

    class ControlAreaBuilding {
        +String buildingId
        +String campusCode
        +String fireCodeId
    }

    class ControlAreaExempt {
        +Boolean isExempt
        +String reason
        +String notes
    }

    class FireSuppressionOverride {
        +Boolean override
        +Boolean value
    }

    class ApprovedStorage {
        +String hazardClass
        +PhysicalState[] physicalStates
    }

    class HazardClass {
        +String id
        +String name
        +Boolean isHealthHazard
        +HazardState solid
        +HazardState liquid
        +HazardState gas
        +HazardClassInventory[] inventories
    }

    class HazardState {
        +Number limit
        +Number closedLimit
        +Number openLimit
        +String displayLimit
        +String displayValue
        +Number actualWeight
        +Number actualVolume
        +String units
        +ComplianceStatus status
        +MaqFactor[] maqFactors
    }

    class MaqFactor {
        +String label
        +Number value
        +String operator
        +Boolean isNoLimit
        +Boolean isNotApplicable
    }

    ControlArea "1" --> "1" ControlAreaBuilding
    ControlArea "1" --> "1" ControlAreaExempt
    ControlArea "1" --> "1" FireSuppressionOverride
    ControlArea "1" --> "*" ApprovedStorage
    ControlArea "1" --> "*" HazardClass
    HazardClass "1" --> "3" HazardState
    HazardState "1" --> "*" MaqFactor
```

### 3. `building`

```mermaid
classDiagram
    class Building {
        +String _id
        +String id
        +String buildingKey
        +String name
        +String address
        +String city
        +String county
        +String state
        +String campusCode
        +String fireCodeId
        +Boolean fireSuppressionCoverage
        +FireSuppressionType fireSuppressionType
        +Floor[] floors
    }

    class Floor {
        +String id
        +String floorKey
        +String name
        +String groundPlane
        +Room[] rooms
    }

    class Room {
        +String id
        +String roomKey
        +String name
        +String roomNumber
        +String controlAreaId
    }

    Building "1" --> "*" Floor
    Floor "1" --> "*" Room
```

## Data Access Layer

All collections use `MongoService` abstraction:

**Location:** `packages/server/src/utils/mongo/MongoService.ts`

```mermaid
classDiagram
    class MongoService~T~ {
        +Collection~T~ collection
        +find(query, options) Promise~T[]~
        +findOne(query) Promise~T~
        +insertOne(doc) Promise~T~
        +insertMany(docs) Promise~T[]~
        +updateOne(query, update) Promise~UpdateResult~
        +updateMany(query, update) Promise~UpdateResult~
        +findOneAndUpdate(query, update, options) Promise~T~
        +delete(query) Promise~DeleteResult~
    }

    class FireCodeService {
        +MongoService fireCodeCollection
        +MongoService historyCollection
        +getById(id)
        +fireCodes()
        +cloneFireCode()
        +deleteFireCode()
    }

    class ControlAreaService {
        +MongoService controlAreaCollection
        +getControlAreasById(ids)
        +getByBuildingId(buildingIds)
        +saveControlArea()
        +updateMaqOnControlAreas()
    }

    class BuildingService {
        +MongoService attachmentCollection
        +getBuildingAttachmentsById()
        +addBuildingAttachment()
        +deleteBuildingAttachments()
    }

    class ProjectionService {
        +MongoService projectionCollection
        +saveNewProjection()
        +findByUserId()
    }

    FireCodeService --> MongoService
    ControlAreaService --> MongoService
    BuildingService --> MongoService
    ProjectionService --> MongoService
```

## Auto-Added Fields

All documents automatically receive these fields via MongoService:

| Field | Type | Description |
|-------|------|-------------|
| `createdDate` | Date | Document creation timestamp |
| `lastUpdatedDate` | Date | Last modification timestamp |
| `createdBy` | `{ userId: string }` | User who created the document |
| `lastUpdatedBy` | `{ userId: string }` | User who last modified |
| `meta.testId` | string (optional) | Test identifier for cleanup |

## Seed Data Locations

| Collection | Fixture File |
|------------|--------------|
| fireCode | `packages/common/src/fixtures/internal/firecodes.ts` |
| controlArea | `packages/common/src/fixtures/internal/controlareas.ts` |
| building | `packages/common/src/fixtures/external/buildings.ts` |

## Key Enums

```typescript
// Compliance status for control areas
type ComplianceStatus = 'COMPLIANT' | 'NON_COMPLIANT' | 'UNKNOWN'

// Fire suppression types
type FireSuppressionType = 'FULL' | 'PARTIAL' | 'NONE'

// Physical states for hazard tracking
type PhysicalState = 'SOLID' | 'LIQUID' | 'GAS'

// Rule operators for MAQ calculations
type RuleOperator = 'MULTIPLY' | 'EQUALS' | 'OVERRIDE' | 'REPORT_AS_LIQUID'

// Rule criteria for conditional limits
type RuleCriteria = 'APPROVED_STORAGE' | 'OUTDOOR' | 'SPRINKLER' | ...
```

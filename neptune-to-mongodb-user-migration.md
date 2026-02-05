---
tags:
  - mongodb
  - neptune
  - migration
  - users
  - folder-server
  - atlas-search
date: 2026-01-16
---

# Analysis: Migrating Users from Neptune to MongoDB

## Context

- **Current**: Users sync from IdP → Neptune via webhooks, then Kafka → Elastic for search
- **Pain points**: Neptune CDC is REST polling with 5-second delays, Kafka sync adds complexity
- **Scale**: 100K+ users
- **Critical relationships**: USER → CAMPUS affiliation, USER → PROGRAM_ROLE membership

## Recommendation: Yes, Migrate

The migration makes sense because:

1. **Neptune isn't source of truth** - IdP is. You're replacing a replica, not a primary store
2. **Webhook sync is easy to intercept** - Add MongoDB write alongside Neptune write in handler
3. **12+ month timeline** - Ample time for careful app-by-app migration
4. **Patterns exist** - folder-server already has dual-write (FGA), user_role_assignments, Atlas Search infrastructure
5. **Eliminates complexity** - No more CDC polling, no Kafka → Elastic pipeline

## Architecture Comparison

```
CURRENT:
IdP → [webhook] → Neptune → [REST CDC 5s delay] → Kafka → Elastic (search)
                          ↓
                    Apps query Neptune

PROPOSED:
IdP → [webhook] → MongoDB + Atlas Search (real-time indexing)
              ↘
               Neptune (dual-write, 12mo transition)
                    ↓
              Legacy apps (gradually migrate)
```

## Data Model in MongoDB

```typescript
// users collection
interface User {
  _id: string;              // UUID from IdP
  firstName: string;
  lastName: string;
  email: string;
  status: "ACTIVE" | "INACTIVE";
  campusId: string;         // FK to folders (campus)
  ucnetId?: string;         // For lookups
  idps: string[];           // Identity providers
  meta: Metadata;           // Audit trail
}

// Existing: user_role_assignments handles USER → PROGRAM_ROLE
```

## Migration Strategy

### Phase 1: Infrastructure (Week 1-2)
- Add User collection + schema to folder-server
- Create Atlas Search index for user search
- Add GraphQL types + resolvers

### Phase 2: Dual-Write Setup (Week 3-4)
- Modify webhook handler to write to BOTH MongoDB + Neptune
- New users/updates go to both systems
- Verify data parity

### Phase 3: Backfill (Week 5-8)
- Batch migrate existing 100K+ users
- Nightly batches of ~10K users
- Validate against Neptune after each batch

### Phase 4: App Migration (Months 2-12)
- Move search queries to Atlas Search (replaces Elastic)
- Move user lookups to folder-server GraphQL
- Track which apps still hit Neptune

### Phase 5: Deprecation (Month 12+)
- Stop Neptune writes once all apps migrated
- Decommission Kafka → Elastic sync
- Archive Neptune user data

## Key Decisions Needed

1. **Where does the webhook handler live?**
   - If in relationship-v2: add MongoDB client + dual-write there
   - If separate service: point it at folder-server API

2. **Should campusId be embedded or referenced?**
   - Reference (FK): Simpler, matches current model
   - Embedded: Denormalized campus info for single-query reads

3. **How to handle employee details if needed later?**
   - Option A: Add optional fields to User
   - Option B: Separate Employee collection with userId FK

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Dual-write latency | Async write to Neptune (eventual consistency ok since it's replica) |
| Data drift during migration | Daily reconciliation job comparing counts/checksums |
| App migration takes longer | 12+ month timeline built in, Neptune stays read-only at end |
| Atlas Search learning curve | Infrastructure already in folder-server, same patterns |

## Next Steps

1. Decide on webhook handler approach
2. Design User schema + GraphQL types
3. Implement Phase 1 infrastructure
4. Set up dual-write before backfill

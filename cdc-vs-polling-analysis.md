---
tags: [databases]
date: 2024-12-22
status: complete
---

# CDC vs Polling Analysis for Permission Sync

**Date:** 2025-12-03
**Status:** Research Complete
**Author:** Engineering Team

---

## Table of Contents

- [Executive Summary](#executive-summary)
- [Context](#context)
- [OpenFGA Database Schema](#openfga-database-schema)
- [Option A: ReadChanges API Polling](#option-a-readchanges-api-polling)
- [Option B: Database CDC with Debezium + Kafka](#option-b-database-cdc-with-debezium--kafka)
- [Option C: PostgreSQL LISTEN/NOTIFY](#option-c-postgresql-listennotify)
- [Option D: Hybrid Approach](#option-d-hybrid-approach)
- [Comparison Matrix](#comparison-matrix)
- [Recommendation](#recommendation)
- [Migration Path: MySQL to PostgreSQL](#migration-path-mysql-to-postgresql)
- [References](#references)

---

## Executive Summary

This document analyzes whether to use **database-level Change Data Capture (CDC)** on OpenFGA's backing database versus **polling the ReadChanges API** for permission synchronization.

### TL;DR Recommendation

| Your Situation | Recommendation |
|----------------|----------------|
| **< 1,000 users, simple setup preferred** | **ReadChanges API Polling** |
| **Need sub-second latency, have Kafka already** | Database CDC (Debezium) |
| **PostgreSQL, want simple real-time, can tolerate edge cases** | LISTEN/NOTIFY + Polling fallback |
| **Most teams** | **ReadChanges API Polling** (start here, upgrade if needed) |

**Bottom line:** Start with ReadChanges API polling. It's simpler, works with any database, and OpenFGA has optimized it for this use case. Only add CDC complexity if you have a proven need for sub-second latency.

---

## Context

### Current State
- OpenFGA server backed by **MySQL**
- Considering migration to **PostgreSQL**
- Need to sync permissions to Snowflake `USER_PERMISSIONS` table
- Target latency: < 1 minute for permission changes to take effect
- User scale: 100-1,000 users

### Question
Can we get real-time permission changes by implementing CDC on OpenFGA's database instead of polling the ReadChanges API?

---

## OpenFGA Database Schema

OpenFGA stores relationship tuples in a table with approximately this structure:

```sql
-- Simplified representation of OpenFGA's tuple storage
CREATE TABLE tuple (
    store         VARCHAR(26) NOT NULL,    -- ULID
    object_type   VARCHAR(256) NOT NULL,
    object_id     VARCHAR(256) NOT NULL,
    relation      VARCHAR(50) NOT NULL,
    _user         VARCHAR(512) NOT NULL,   -- Legacy column
    user_type     VARCHAR(256),
    user_id       VARCHAR(256),
    user_relation VARCHAR(50),
    condition_name VARCHAR(256),
    condition_context JSONB,
    ulid          VARCHAR(26) NOT NULL,    -- ULID for ordering
    inserted_at   TIMESTAMP NOT NULL,
    PRIMARY KEY (store, object_type, object_id, relation, _user)
);

-- Changelog for ReadChanges API
CREATE TABLE changelog (
    store         VARCHAR(26) NOT NULL,
    object_type   VARCHAR(256) NOT NULL,
    object_id     VARCHAR(256) NOT NULL,
    relation      VARCHAR(50) NOT NULL,
    _user         VARCHAR(512) NOT NULL,
    operation     INTEGER NOT NULL,        -- 1=write, 2=delete
    ulid          VARCHAR(26) NOT NULL,    -- ULID (orderable)
    inserted_at   TIMESTAMP NOT NULL
);
```

**Key insight:** OpenFGA already maintains a `changelog` table specifically for the ReadChanges API, with ULID-based ordering for efficient pagination.

---

## Option A: ReadChanges API Polling

### How It Works

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Sync Service  │────▶│    OpenFGA      │────▶│    Database     │
│   (polls every  │     │  ReadChanges    │     │   changelog     │
│    30 seconds)  │     │      API        │     │     table       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
         │
         ▼
┌─────────────────┐
│   Snowflake     │
│ USER_PERMISSIONS│
└─────────────────┘
```

### Implementation

```typescript
class ReadChangesPoller {
  private continuationToken: string | null = null;

  async poll(): Promise<TupleChange[]> {
    const response = await this.openfga.readChanges({
      type: 'program', // Filter to relevant types
      pageSize: 100,
      continuationToken: this.continuationToken || undefined,
    });

    // Same token returned = no new changes
    if (response.continuation_token === this.continuationToken) {
      return [];
    }

    this.continuationToken = response.continuation_token;
    await this.persistToken(this.continuationToken);

    return response.changes;
  }
}

// Poll every 30 seconds
setInterval(() => poller.poll(), 30000);
```

### Pros

| Benefit | Description |
|---------|-------------|
| **Simplicity** | No additional infrastructure; uses existing OpenFGA API |
| **Database agnostic** | Works with MySQL, PostgreSQL, or SQLite |
| **Battle-tested** | This is the intended pattern; OpenFGA optimizes for it |
| **Reliable** | Continuation token guarantees no missed changes |
| **No DB access needed** | Don't need direct access to OpenFGA's database |

### Cons

| Drawback | Description |
|----------|-------------|
| **Latency** | 30-second polling = up to 30-second delay |
| **API overhead** | Continuous polling even when no changes |
| **Not truly real-time** | Cannot achieve sub-second latency |

### Latency Analysis

```
Permission granted → OpenFGA writes to DB → ReadChanges available
    T+0                    T+~10ms              T+~10ms

Poller runs (worst case) → Processes change → Writes to Snowflake
    T+30s                    T+30.1s            T+30.2s

Total worst-case latency: ~30 seconds
Total best-case latency: ~200ms (if poll happens immediately after change)
Average latency: ~15 seconds
```

### Cost

- **Infrastructure:** None additional
- **OpenFGA load:** ~2 API calls/minute (minimal)
- **Engineering:** 1-2 days to implement

---

## Option B: Database CDC with Debezium + Kafka

### How It Works

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   OpenFGA DB    │────▶│    Debezium     │────▶│     Kafka       │
│  (MySQL/PG)     │     │   Connector     │     │                 │
│   binlog/WAL    │     │                 │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                                        ▼
                        ┌─────────────────┐     ┌─────────────────┐
                        │   Snowflake     │◀────│  Sync Consumer  │
                        │ USER_PERMISSIONS│     │                 │
                        └─────────────────┘     └─────────────────┘
```

### Infrastructure Required

```yaml
# docker-compose.yml (simplified)
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]

  kafka-connect:
    image: debezium/connect:2.4
    depends_on: [kafka]

  # Your existing services
  openfga:
    image: openfga/openfga:latest

  mysql:  # or postgres
    image: mysql:8.0
    command: --binlog-format=ROW --log-bin=mysql-bin
```

### Debezium Connector Configuration

```json
{
  "name": "openfga-tuple-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "***",
    "database.server.id": "1",
    "database.server.name": "openfga",
    "database.include.list": "openfga",
    "table.include.list": "openfga.tuple,openfga.changelog",
    "include.schema.changes": "false",
    "transforms": "filter",
    "transforms.filter.type": "io.debezium.transforms.Filter",
    "transforms.filter.condition": "value.relation == 'snowflake_viewer'"
  }
}
```

### Consumer Implementation

```typescript
import { Kafka } from 'kafkajs';

const kafka = new Kafka({ brokers: ['kafka:9092'] });
const consumer = kafka.consumer({ groupId: 'permission-sync' });

await consumer.subscribe({ topic: 'openfga.openfga.tuple' });

await consumer.run({
  eachMessage: async ({ message }) => {
    const change = JSON.parse(message.value.toString());

    if (change.op === 'c') { // INSERT
      await syncUserPermission(change.after);
    } else if (change.op === 'd') { // DELETE
      await removeUserPermission(change.before);
    }
  },
});
```

### Pros

| Benefit | Description |
|---------|-------------|
| **True real-time** | Sub-second latency (typically 100-500ms) |
| **No polling overhead** | Only processes actual changes |
| **Scalable** | Kafka handles high throughput |
| **Durable** | Kafka retains events for replay |
| **Rich ecosystem** | Many connectors, transformations available |

### Cons

| Drawback | Description |
|----------|-------------|
| **Massive complexity** | Kafka + ZooKeeper + Connect + Debezium |
| **Operational burden** | [4-6 engineers at Netflix/Robinhood](https://estuary.dev/blog/debezium-cdc-pain-points/) to maintain at scale |
| **Infrastructure cost** | Kafka cluster, storage, network |
| **Expertise required** | Deep Kafka + Java knowledge needed |
| **Failure modes** | Binlog gaps, connector crashes, Kafka outages |
| **Database coupling** | Direct dependency on OpenFGA's internal schema |
| **Schema changes** | OpenFGA upgrades may break your CDC |

### Latency Analysis

```
Permission granted → DB commits → Binlog written → Debezium reads
    T+0               T+~5ms       T+~10ms         T+~50ms

Kafka produces → Consumer reads → Writes to Snowflake
   T+~100ms        T+~150ms           T+~200ms

Total typical latency: 200-500ms
```

### Cost

| Component | Monthly Cost (Estimate) |
|-----------|------------------------|
| Kafka cluster (3 brokers) | $300-1,000 |
| Kafka Connect workers | $100-300 |
| ZooKeeper (if not KRaft) | $50-150 |
| Storage (retention) | $50-200 |
| Engineering time | 2-4 weeks initial, ongoing maintenance |

**Total:** $500-1,500/month + significant engineering investment

---

## Option C: PostgreSQL LISTEN/NOTIFY

### How It Works

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   OpenFGA DB    │────▶│   PG Trigger    │────▶│   NOTIFY        │
│  (PostgreSQL)   │     │  on tuple table │     │   Channel       │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                                                        │
                                                        ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Snowflake     │◀────│  Sync Service   │◀────│   LISTEN        │
│ USER_PERMISSIONS│     │                 │     │   Connection    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

### Implementation

**Step 1: Create trigger on OpenFGA's database**

```sql
-- WARNING: This modifies OpenFGA's internal database
-- May break on OpenFGA upgrades

CREATE OR REPLACE FUNCTION notify_tuple_change()
RETURNS TRIGGER AS $$
DECLARE
  payload JSON;
BEGIN
  -- Only notify for snowflake_viewer relation
  IF (TG_OP = 'INSERT' AND NEW.relation = 'snowflake_viewer') OR
     (TG_OP = 'DELETE' AND OLD.relation = 'snowflake_viewer') THEN

    payload = json_build_object(
      'operation', TG_OP,
      'object_type', COALESCE(NEW.object_type, OLD.object_type),
      'object_id', COALESCE(NEW.object_id, OLD.object_id),
      'user', COALESCE(NEW._user, OLD._user)
    );

    PERFORM pg_notify('permission_changes', payload::text);
  END IF;

  RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tuple_change_trigger
AFTER INSERT OR DELETE ON tuple
FOR EACH ROW EXECUTE FUNCTION notify_tuple_change();
```

**Step 2: Listen in sync service**

```typescript
import { Client } from 'pg';

const client = new Client({
  connectionString: process.env.OPENFGA_DATABASE_URL,
});

await client.connect();
await client.query('LISTEN permission_changes');

client.on('notification', async (msg) => {
  const change = JSON.parse(msg.payload);

  if (change.operation === 'INSERT') {
    await syncUserPermission(change);
  } else if (change.operation === 'DELETE') {
    await removeUserPermission(change);
  }
});

// Keep connection alive
setInterval(() => client.query('SELECT 1'), 30000);
```

### Pros

| Benefit | Description |
|---------|-------------|
| **Near real-time** | Sub-second latency (typically 10-100ms) |
| **Lightweight** | No additional infrastructure |
| **Simple** | Just PostgreSQL + a trigger |
| **Low cost** | No new services to run |

### Cons

| Drawback | Description |
|----------|-------------|
| **PostgreSQL only** | Must migrate from MySQL |
| **Ephemeral** | Notifications lost if listener disconnected |
| **Payload limit** | 8KB max per notification |
| **Trigger maintenance** | Your trigger may break on OpenFGA schema changes |
| **Internal coupling** | Depends on OpenFGA's internal table structure |
| **No replay** | Cannot recover missed notifications |
| **Connection required** | Must maintain persistent connection |

### Critical Limitation: Ephemeral Notifications

```
Timeline:
  T0: Sync service connected, listening
  T1: Permission granted → NOTIFY sent → Sync service receives ✓
  T2: Sync service crashes/restarts
  T3: Permission granted → NOTIFY sent → LOST (no listener)
  T4: Sync service reconnects
  T5: No way to know T3 change happened!
```

**Mitigation:** Combine with periodic ReadChanges polling as fallback.

### Latency Analysis

```
Permission granted → Trigger fires → NOTIFY sent → Listener receives
    T+0               T+~1ms         T+~2ms         T+~10ms

Writes to Snowflake
   T+~50ms

Total typical latency: 50-100ms
```

### Cost

- **Infrastructure:** None (uses existing PostgreSQL)
- **Engineering:** 2-3 days to implement + migration from MySQL
- **Risk:** Maintenance burden when OpenFGA upgrades

---

## Option D: Hybrid Approach

### How It Works

Combine LISTEN/NOTIFY (or webhooks) for immediate notifications with ReadChanges polling as a reliable fallback.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Sync Service                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │  LISTEN/     │     │  App-Level   │     │ ReadChanges  │    │
│  │  NOTIFY      │     │  Webhooks    │     │   Poller     │    │
│  │  (primary)   │     │  (primary)   │     │  (fallback)  │    │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘    │
│         │                    │                    │              │
│         ▼                    ▼                    ▼              │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Event Deduplication Layer                   │   │
│  │         (by ULID or user+resource+timestamp)            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            │                                    │
│                            ▼                                    │
│                   ┌──────────────┐                             │
│                   │  Snowflake   │                             │
│                   │   Writer     │                             │
│                   └──────────────┘                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implementation

```typescript
class HybridSyncService {
  private lastPolledUlid: string | null = null;
  private processedChanges = new Set<string>(); // LRU cache

  constructor(
    private openfga: OpenFgaClient,
    private pgClient: Client,
    private snowflake: SnowflakeClient,
  ) {}

  async start() {
    // Start LISTEN/NOTIFY for real-time
    await this.startListener();

    // Start polling as fallback (every 5 minutes)
    setInterval(() => this.pollChanges(), 5 * 60 * 1000);

    // Initial catch-up
    await this.pollChanges();
  }

  private async startListener() {
    await this.pgClient.query('LISTEN permission_changes');

    this.pgClient.on('notification', async (msg) => {
      const change = JSON.parse(msg.payload);
      await this.processChange(change, 'listen');
    });
  }

  private async pollChanges() {
    let hasMore = true;

    while (hasMore) {
      const response = await this.openfga.readChanges({
        pageSize: 100,
        continuationToken: this.lastPolledUlid || undefined,
      });

      for (const change of response.changes) {
        await this.processChange(change, 'poll');
      }

      if (response.continuation_token === this.lastPolledUlid) {
        hasMore = false;
      } else {
        this.lastPolledUlid = response.continuation_token;
        await this.persistToken(this.lastPolledUlid);
      }
    }
  }

  private async processChange(change: any, source: string) {
    // Deduplicate
    const key = `${change.user}:${change.object_type}:${change.object_id}`;
    if (this.processedChanges.has(key)) {
      return; // Already processed via other source
    }
    this.processedChanges.add(key);

    // Process
    console.log(`Processing change from ${source}:`, change);
    await this.syncToSnowflake(change);
  }
}
```

### Pros

| Benefit | Description |
|---------|-------------|
| **Best of both** | Real-time when possible, reliable always |
| **Self-healing** | Polling catches anything LISTEN misses |
| **Graceful degradation** | Works even if LISTEN fails |

### Cons

| Drawback | Description |
|----------|-------------|
| **Complexity** | Two systems to maintain |
| **Deduplication** | Must handle events from both sources |
| **Still couples to DB** | If using LISTEN/NOTIFY |

---

## Comparison Matrix

| Criteria | ReadChanges Polling | Debezium + Kafka | LISTEN/NOTIFY | Hybrid |
|----------|---------------------|------------------|---------------|--------|
| **Latency** | 15-30s avg | 200-500ms | 50-100ms | 50-100ms (real-time) |
| **Reliability** | Excellent | Good (with tuning) | Poor (ephemeral) | Excellent |
| **Complexity** | Low | Very High | Medium | Medium-High |
| **Infrastructure** | None | Kafka cluster | None | None |
| **Cost** | ~$0 | $500-1,500/mo | ~$0 | ~$0 |
| **Engineering effort** | 1-2 days | 2-4 weeks | 2-3 days | 3-5 days |
| **Database coupling** | None | High | High | Medium |
| **MySQL support** | Yes | Yes | No | No (for LISTEN) |
| **Survives restarts** | Yes | Yes | No | Yes |
| **OpenFGA upgrade safe** | Yes | No | No | Partially |

---

## Recommendation

### For Your Situation (100-1,000 users, wants simplicity + reliability)

**Start with: ReadChanges API Polling**

Rationale:
1. **Your latency requirement is ~1 minute** - 30-second polling easily meets this
2. **You're at 100-1,000 users** - No need for sub-second sync
3. **OpenFGA designed for this** - ReadChanges API is the official pattern
4. **Zero additional infrastructure** - No Kafka, no triggers
5. **Database agnostic** - Works with current MySQL, no need to migrate
6. **Upgrade safe** - Won't break when OpenFGA updates

### When to Consider CDC

| Situation | Consider |
|-----------|----------|
| Need < 1 second latency | Debezium (if you have Kafka) or LISTEN/NOTIFY hybrid |
| Already running Kafka | Debezium (marginal cost to add) |
| 10,000+ users with frequent changes | Debezium (better throughput) |
| Must have real-time for compliance | Debezium or Hybrid |

### When NOT to Use CDC

- You don't already have Kafka infrastructure
- Your latency SLA is > 30 seconds
- You want minimal operational burden
- You have < 1,000 users
- You're not willing to couple to OpenFGA's internal schema

### Complexity vs Latency Trade-off

```
Latency      │
(seconds)    │
             │
    30 ──────┼─────────────────────● ReadChanges Polling
             │                       (Low complexity)
             │
    15 ──────┼
             │
     1 ──────┼───────● Hybrid (LISTEN + Polling)
             │         (Medium complexity)
             │
   0.5 ──────┼─● Debezium + Kafka
             │   (High complexity)
             │
   0.1 ──────┼● LISTEN/NOTIFY only
             │  (Medium complexity, unreliable)
             │
             └────────────────────────────────────────
                   Complexity / Operational Burden →
```

---

## Migration Path: MySQL to PostgreSQL

If you decide to use LISTEN/NOTIFY (now or later), you'll need PostgreSQL.

### Benefits of PostgreSQL for OpenFGA

| Benefit | Description |
|---------|-------------|
| **Read replicas** | OpenFGA supports separate read/write connections |
| **Better CDC** | Logical replication superior to MySQL binlog |
| **LISTEN/NOTIFY** | Built-in pub/sub for real-time |
| **JSON support** | Better JSONB handling for conditions |
| **No field limits** | MySQL has stricter varchar limits |

### Migration Steps

1. **Set up PostgreSQL** alongside MySQL
2. **Run OpenFGA migration** to create schema:
   ```bash
   openfga migrate --datastore-engine postgres \
     --datastore-uri 'postgres://...'
   ```
3. **Export tuples from MySQL OpenFGA** using ReadChanges API
4. **Import tuples to PostgreSQL OpenFGA** using Write API
5. **Update OpenFGA config** to point to PostgreSQL
6. **Validate** with Check/ListObjects calls
7. **Decommission MySQL**

### Estimated Effort

- **Simple migration (< 100k tuples):** 1-2 days
- **With LISTEN/NOTIFY setup:** 3-4 days
- **Testing and validation:** 1-2 days

---

## References

### OpenFGA
- [OpenFGA Configuration](https://openfga.dev/docs/getting-started/setup-openfga/configure-openfga)
- [ReadChanges API](https://openfga.dev/docs/interacting/read-tuple-changes)
- [OpenFGA GitHub](https://github.com/openfga/openfga)

### Debezium & CDC
- [Debezium Pain Points in Production](https://estuary.dev/blog/debezium-cdc-pain-points/)
- [Debezium PostgreSQL Connector](https://debezium.io/documentation/reference/stable/connectors/postgresql.html)
- [MySQL CDC with Debezium](https://medium.com/@harshgajjar7110/debezium-is-a-powerful-open-source-platform-for-change-data-capture-cdc-that-can-be-used-to-77c677f6a018)
- [Debezium Performance Impact](https://reorchestrate.com/posts/debezium-performance-impact/)

### PostgreSQL
- [PostgreSQL LISTEN/NOTIFY](https://tapoueh.org/blog/2018/07/postgresql-listen-notify/)
- [PostgreSQL CDC Complete Guide](https://datacater.io/blog/2021-09-02/postgresql-cdc-complete-guide.html)
- [All Ways to Do CDC in Postgres](https://blog.sequinstream.com/all-the-ways-to-capture-changes-in-postgres/)

### Alternatives
- [Debezium Alternatives](https://estuary.dev/blog/debezium-alternatives/)
- [Query-Based CDC](https://medium.com/@christinataylor0926/less-is-more-a-case-for-query-based-change-data-capture-a3b22349dba6)

---
tags: [graphql]
date: 2024-12-22
status: complete
---

# Schema Management Strategy for Federated GraphQL

**Date:** 2025-11-16
**Status:** Recommended approach for this project

## Context

This project uses GraphQL Federation with WunderGraph Cosmo Router. As we scale from 1 service to 10+ microservices, we need a sustainable way to manage schema files that:

1. Works locally for developers
2. Works in CI/CD pipelines
3. Scales to 10+ services without becoming unwieldy
4. Maintains schema versioning and history

## Two Viable Approaches

After research and analysis, there are **two practical approaches** for schema management:

### 1. File-Based (Current Setup)
Best for: **1-5 services, simple multi-repo setup**

### 2. GitHub Pattern B (Schemas in Router Repo)
Best for: **5-30+ services, production scale**

## Approach 1: File-Based (Current)

### How It Works

```
Your File System:

/repos/
  cosmo/                          # Router repo
    compose.yaml                  # Points to other repos
  comments-server/                # Subgraph repo
    src/graphql/
      comments.schema.graphql     # Schema file
  users-server/                   # Subgraph repo
    schema.graphql
```

**compose.yaml:**
```yaml
subgraphs:
  - name: comments
    routing_url: http://comments-server:3000/graphql
    schema:
      file: ../comments-server/src/graphql/comments.schema.graphql
```

### Local Development

```bash
# 1. Clone all repos in sibling directories
cd /repos
git clone git@github.com:yourorg/cosmo-gateway.git
git clone git@github.com:yourorg/comments-server.git
git clone git@github.com:yourorg/users-server.git

# 2. Schemas are referenced via file paths
cd cosmo-gateway
pnpm compose:check  # Reads ../comments-server/schema.graphql
pnpm dev
```

### CI/CD

```yaml
# .github/workflows/compose-check.yml
steps:
  - uses: actions/checkout@v4

  - uses: actions/checkout@v4
    with:
      repository: yourorg/comments-server
      path: comments-server

  - uses: actions/checkout@v4
    with:
      repository: yourorg/users-server
      path: users-server

  # Add one checkout per subgraph

  - run: pnpm compose:check
```

### Pros
- ✅ Simple - just file paths
- ✅ Fast - no network calls
- ✅ Works offline
- ✅ IDE can navigate between files
- ✅ No additional setup needed

### Cons
- ❌ All repos must be checked out
- ❌ Directory structure must match across dev machines and CI
- ❌ CI needs N+1 checkout steps (1 for router, N for subgraphs)
- ❌ Doesn't scale beyond ~5 repos
- ❌ Schema changes require updating multiple repos

### When to Use
- **1-5 services**
- **Single team** with simple setup
- **Tight coupling** between router and subgraphs is acceptable
- **Quick prototyping** or MVP phase

---

## Approach 2: Pattern B (Schemas in Router Repo) - RECOMMENDED

### How It Works

```
GitHub:

yourorg/cosmo-gateway/                    # Router repo
  schemas/                        # ← Schemas committed HERE
    comments.graphql
    users.graphql
    posts.graphql
  compose.yaml                    # Points to ./schemas/

yourorg/comments-server/          # Subgraph repo
  src/graphql/
    comments.schema.graphql
  .github/workflows/
    publish-schema.yml            # Publishes to cosmo-gateway/schemas/

yourorg/users-server/             # Subgraph repo
  schema.graphql
  .github/workflows/
    publish-schema.yml            # Publishes to cosmo-gateway/schemas/
```

**compose.yaml:**
```yaml
subgraphs:
  - name: comments
    routing_url: http://comments-server:3000/graphql
    schema:
      file: ./schemas/comments.graphql  # Local file in router repo
```

### How Schemas Get There

**Automated Publishing from Subgraphs:**

Each subgraph has a GitHub Action that:
1. Detects schema file changes
2. Clones router repo
3. Copies schema to `cosmo-gateway/schemas/[service].graphql`
4. Commits and pushes to router repo

**Workflow:**
```
Developer updates schema in comments-server
  ↓
Push to GitHub
  ↓
CI detects schema change
  ↓
CI clones cosmo repo
  ↓
CI copies schema to cosmo-gateway/schemas/comments.graphql
  ↓
CI commits to cosmo repo
  ↓
Router repo now has latest schema
```

### Local Development

```bash
# Clone just the router repo
git clone git@github.com:yourorg/cosmo-gateway.git
cd cosmo-gateway

# Schemas are already here (committed to repo)
ls schemas/
# comments.graphql  users.graphql  posts.graphql

# Just compose and run
pnpm compose:check
pnpm dev
```

**Getting schema updates:**
```bash
# Pull latest router repo
git pull

# Schemas update automatically
pnpm compose:check
pnpm dev:restart:router
```

### CI/CD

```yaml
# .github/workflows/compose.yml
steps:
  - uses: actions/checkout@v4
    # That's it! Schemas already in repo

  - run: pnpm install
  - run: pnpm compose:check
```

**One checkout step** instead of N+1!

### Pros
- ✅ **Scales to unlimited services**
- ✅ Simple local dev (just `git clone` and go)
- ✅ Simple CI/CD (single checkout)
- ✅ Git history tracks all schema changes
- ✅ Can review schema changes via `git log schemas/`
- ✅ Atomic commits (schema + compose.yaml together)
- ✅ No directory structure requirements
- ✅ Easy developer onboarding

### Cons
- ⚠️ Initial setup needed (add publish workflows to subgraphs)
- ⚠️ Requires GitHub tokens for cross-repo publishing
- ⚠️ Schema updates have slight delay (CI publishes)
- ⚠️ Router repo commit history includes schema changes

### When to Use
- **5-30+ services**
- **Production systems** that will scale
- **Multiple developers** needing easy onboarding
- **CI/CD pipelines** that need to be fast
- **Schema versioning** via Git is acceptable

---

## Comparison

| Factor | File-Based | Pattern B |
|--------|------------|-----------|
| **Setup Time** | 5 min | 1-2 hours |
| **Local Dev Complexity** | Medium (clone all repos) | Low (clone one repo) |
| **CI/CD Complexity** | High (N+1 checkouts) | Low (1 checkout) |
| **Scales To** | ~5 services | Unlimited |
| **Schema Versioning** | Manual | Git history |
| **Developer Onboarding** | Complex (multiple repos) | Simple (one repo) |
| **Offline Work** | ✅ Yes | ✅ Yes |
| **Best For** | Small projects, MVPs | Production, scaling teams |

## Decision Matrix

```
How many services do you have (or expect)?

1-3 services
  ↓
  Use File-Based
  (Simple, works great at this scale)

4-5 services
  ↓
  Growing soon? → Pattern B
  Staying small? → File-Based

6+ services
  ↓
  Use Pattern B
  (File-based becomes painful)
```

## Migration Path

### Starting New: Use Pattern B

If you're starting fresh or only have 1-2 services, **start with Pattern B**:
- Setup time is only 1-2 hours
- Avoids migration pain later
- Grows with you

### Currently File-Based: When to Migrate

Migrate to Pattern B when you experience:
- ❌ CI taking too long (multiple checkouts)
- ❌ New developers struggling with multi-repo setup
- ❌ Directory structure issues across machines
- ❌ Planning to add 3+ more services

**Migration time:** 2-3 hours for 5 services

### Migration Steps

```bash
# 1. Create schemas/ directory in router repo
mkdir -p schemas

# 2. Copy all existing schemas
cp ../comments-server/src/graphql/comments.schema.graphql schemas/comments.graphql
cp ../users-server/schema.graphql schemas/users.graphql
# ... copy all

# 3. Update compose.yaml paths
# Change: ../comments-server/schema.graphql
# To:     ./schemas/comments.graphql

# 4. Commit to router repo
git add schemas/ compose.yaml
git commit -m "Migrate to Pattern B: schemas in router repo"
git push

# 5. Add publish workflows to each subgraph
# (See implementation guide below)

# 6. Test: Schema updates now automatic via CI
```

## Recommendation for This Project

### Current State: 1 service (comments-server)
### Expected: 10+ services

**Recommended: Start with Pattern B now**

**Rationale:**
- You're planning 10+ services (well beyond file-based scale)
- 1-2 hour setup is worth avoiding future migration
- Clean foundation as services are added
- Better CI/CD from the start

**Alternative:**
Stay with file-based until you have 3-4 services, then migrate.

## Implementation

### For File-Based (Current)

**Nothing to do!** Already working.

To add a new service:
```yaml
# Update compose.yaml
- name: newservice
  routing_url: http://newservice:3010/graphql
  schema:
    file: ../newservice-server/schema.graphql

# Update CI to checkout new repo
- uses: actions/checkout@v4
  with:
    repository: yourorg/newservice-server
    path: newservice-server
```

### For Pattern B (Recommended)

**See detailed implementation:** `docs/PATTERN-B-SETUP.md`

**Quick overview:**

1. **Router repo setup** (5 minutes)
   ```bash
   mkdir -p schemas
   git add schemas/
   git commit -m "Add schemas directory"
   ```

2. **For each subgraph** (15 minutes per service)
   - Add `.github/workflows/publish-schema.yml`
   - Configure `SUBGRAPH_NAME` and `SCHEMA_FILE`
   - Add `ROUTER_PUBLISH_TOKEN` secret
   - Test publish on merge

3. **Developer workflow**
   ```bash
   git clone cosmo-gateway    # Just one repo
   pnpm dev           # Schemas already there
   ```

## Schema Change Workflows

### File-Based

```bash
# In subgraph repo
vim schema.graphql
git commit -m "Add new field"
git push

# In router repo
pnpm compose:check  # Picks up change via file path
pnpm dev:restart:router
```

### Pattern B

```bash
# In subgraph repo
vim schema.graphql
git commit -m "Add new field"
git push
# → CI automatically publishes to router repo

# In router repo
git pull  # Get published schema
pnpm compose:check
pnpm dev:restart:router
```

**Or for immediate testing:**
```bash
# In router repo (test before subgraph publishes)
vim schemas/comments.graphql  # Edit directly
pnpm compose:check
# Test locally, then commit if intentional change
```

## Alternatives Not Recommended

We evaluated but **do not recommend** these approaches for this project:

### ❌ Pattern A (Dedicated Schemas Repo)
**Why not:** Adds complexity with a third repo (`graphql-schemas`) without meaningful benefit for a single-team setup. Only needed for multi-team orgs with complex access control.

### ❌ Introspection-Based
**Why not:** Requires services to be running during composition. Creates chicken-egg problems in CI/CD. No schema versioning.

### ❌ Schema Registry (Cosmo Cloud)
**Why not:** External dependency, potential cost, vendor lock-in. Only justified at enterprise scale (50+ services) or when you need managed infrastructure.

### ❌ NPM Packages
**Why not:** Publishing overhead, package management complexity. No advantage over git-based versioning.

## Next Steps

1. **Review this document** with your team
2. **Choose an approach:**
   - Stay with file-based if you have ≤3 services
   - Migrate to Pattern B if you have 4+ or plan to scale
3. **If Pattern B:** Follow `docs/PATTERN-B-SETUP.md`
4. **Document decision** in team wiki/notion

## References

- **Pattern B Setup Guide:** `docs/PATTERN-B-SETUP.md`
- **Cosmo Router Docs:** https://cosmo-docs.wundergraph.com/
- **Federation Spec:** https://www.apollographql.com/docs/federation/

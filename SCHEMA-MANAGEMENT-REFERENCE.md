---
tags: [graphql]
date: 2024-12-22
status: complete
---

# Schema Management Quick Reference

**How to manage GraphQL schemas across multiple microservices.**

## Choose Your Approach

| Your Situation | Use This |
|----------------|----------|
| 1-3 services | **File-Based** (already working) |
| 4-5 services, planning to grow | **Pattern B** |
| 6+ services | **Pattern B** |

See `docs/SCHEMA-MANAGEMENT-DECISION.md` for detailed decision guide.

---

## File-Based (Current Setup)

### Structure
```
/repos/
  cosmo-gateway/
    compose.yaml  → points to ../comments-server/schema.graphql
  comments-server/
    schema.graphql
  users-server/
    schema.graphql
```

### Local Development
```bash
# Clone all repos in sibling directories
git clone git@github.com:yourorg/cosmo-gateway.git
git clone git@github.com:yourorg/comments-server.git
git clone git@github.com:yourorg/users-server.git

# Compose and run
cd cosmo-gateway
pnpm compose:check
pnpm dev
```

### Adding a Service
```yaml
# 1. Create new service repo with schema.graphql

# 2. Update compose.yaml
subgraphs:
  - name: newservice
    routing_url: http://newservice:3010/graphql
    schema:
      file: ../newservice-server/schema.graphql

# 3. Update CI to checkout new repo
- uses: actions/checkout@v4
  with:
    repository: yourorg/newservice-server
    path: newservice-server

# 4. Clone locally
git clone git@github.com:yourorg/newservice-server.git
```

### Updating a Schema
```bash
# In subgraph repo
cd ../comments-server
vim schema.graphql
git commit -m "Update schema"
git push

# In router repo
cd cosmo-gateway
pnpm compose:check  # Picks up changes via file path
pnpm dev:restart:router
```

**Pros:** Simple, no setup
**Cons:** Doesn't scale, complex CI, directory dependencies

---

## Pattern B (Schemas in Router Repo)

### Structure
```
GitHub:
  yourorg/cosmo-gateway/
    schemas/           ← Committed to router repo!
      comments.graphql
      users.graphql
    compose.yaml       → points to ./schemas/

  yourorg/comments-server/
    schema.graphql
    .github/workflows/
      publish-schema.yml  → publishes to cosmo-gateway/schemas/
```

### Local Development
```bash
# Just clone router repo
git clone git@github.com:yourorg/cosmo-gateway.git
cd cosmo-gateway

# Schemas already inside
ls schemas/

# Compose and run
pnpm compose:check
pnpm dev
```

### Adding a Service
```bash
# 1. Create new service repo with schema.graphql

# 2. Add publish workflow to service
cp docs/PATTERN-B-SETUP.md  # See "Subgraph Setup" section
# Edit workflow:
#   SUBGRAPH_NAME: newservice
#   SCHEMA_FILE: schema.graphql

# 3. Add GitHub token
gh secret set ROUTER_PUBLISH_TOKEN --repo yourorg/newservice-server

# 4. Update router's compose.yaml
subgraphs:
  - name: newservice
    routing_url: http://newservice:3010/graphql
    schema:
      file: ./schemas/newservice.graphql

# 5. Push service code
# → CI automatically publishes schema to cosmo-gateway/schemas/

# 6. Pull router repo
cd cosmo-gateway
git pull  # Get published schema
pnpm compose:check
```

### Updating a Schema
```bash
# In subgraph repo
cd ../comments-server
vim schema.graphql
git commit -m "Update schema"
git push
# → CI automatically publishes to cosmo-gateway/schemas/

# In router repo
cd cosmo-gateway
git pull  # Get published schema
pnpm compose:check
pnpm dev:restart:router
```

**Pros:** Scales infinitely, simple dev, fast CI
**Cons:** Initial setup (1-2 hours)

---

## Commands Cheat Sheet

### Composition
```bash
pnpm compose:check    # Validate without writing
pnpm compose          # Generate router.json
pnpm compose:watch    # Auto-recompose on changes
```

### Development
```bash
pnpm dev              # Start all services
pnpm dev:logs         # View all logs
pnpm dev:logs:router  # View router logs
pnpm dev:restart:router  # Restart after schema change
pnpm dev:down         # Stop all services
```

### Schema Review
```bash
# View schema changes (Pattern B only)
cd schemas
git log comments.graphql
git diff HEAD~1 comments.graphql

# Show schema history
git log --oneline schemas/
```

---

## Common Workflows

### Reviewing Schema Changes (Pattern B)

```bash
# See what schemas changed recently
git log --oneline schemas/

# View specific schema history
git log -p schemas/comments.graphql

# Compare versions
git diff HEAD~1 schemas/comments.graphql
```

### Testing Schema Before Deploy

```bash
# Pattern B: Edit locally to test
vim schemas/comments.graphql
pnpm compose:check  # Test composition
git restore schemas/comments.graphql  # Discard if just testing

# File-Based: Edit in subgraph repo
cd ../comments-server
vim schema.graphql
cd ../cosmo
pnpm compose:check
```

### Rolling Back Schema Change (Pattern B)

```bash
# Revert to previous version
git revert <commit-hash>
git push

# Or restore specific schema
git restore --source=HEAD~1 schemas/comments.graphql
git commit -m "Revert comments schema"
git push
```

---

## Troubleshooting

### Composition fails

```bash
# Run without :check to see detailed errors
pnpm compose

# Common issues:
# - Type conflicts between subgraphs
# - Missing @shareable on shared types
# - Missing @key on entity types
# - Invalid Federation directives
```

### Schema not found (File-Based)

```bash
# Check directory structure
ls ../comments-server/schema.graphql

# Verify path in compose.yaml matches
cat compose.yaml | grep -A 2 comments
```

### Schema not updating (Pattern B)

```bash
# Did subgraph CI run?
gh run list --repo yourorg/comments-server

# Did it push to router repo?
git log schemas/comments.graphql

# Pull latest
git pull
```

---

## CI/CD Examples

### File-Based CI

```yaml
# .github/workflows/compose.yml
name: Compose Schema

on:
  push:
    branches: [main]

jobs:
  compose:
    runs-on: ubuntu-latest
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

      # Add checkout for each service

      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
      - run: pnpm install
      - run: pnpm compose:check
```

### Pattern B CI

```yaml
# .github/workflows/compose.yml
name: Compose Schema

on:
  push:
    branches: [main]
    paths: ['schemas/**', 'compose.yaml']

jobs:
  compose:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        # That's it! Schemas already in repo

      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
      - run: pnpm install
      - run: pnpm compose:check
```

---

## Migration: File-Based → Pattern B

**Time:** 2-3 hours for 5 services

```bash
# 1. Create schemas directory
mkdir -p schemas

# 2. Copy all schemas
cp ../comments-server/schema.graphql schemas/comments.graphql
cp ../users-server/schema.graphql schemas/users.graphql
# ... copy all

# 3. Update compose.yaml paths
# Change: ../comments-server/schema.graphql
# To:     ./schemas/comments.graphql

# 4. Commit to router repo
git add schemas/ compose.yaml
git commit -m "Migrate to Pattern B"
git push

# 5. Add publish workflows to each subgraph
# See: docs/PATTERN-B-SETUP.md

# 6. Test - schema updates now automatic
```

---

## Documentation Index

- **Decision Guide:** `docs/SCHEMA-MANAGEMENT-DECISION.md` - Choose file-based or Pattern B
- **Research:** `docs/research/schema-management-strategy.md` - Detailed analysis
- **Pattern B Setup:** `docs/PATTERN-B-SETUP.md` - Complete implementation guide
- **Project Guide:** `CLAUDE.md` - Project-specific setup instructions

---

## When to Get Help

**File-Based issues:**
- Check directory structure matches
- Verify all repos are cloned in sibling directories
- Look at CI checkout paths

**Pattern B issues:**
- Check subgraph publish workflow ran
- Verify GitHub token has `repo` permission
- Check schema was committed to router repo

**Composition issues:**
- Read full error with `pnpm compose` (not `:check`)
- See Federation 2 guide: `docs/research/graphql-federation-2-cosmos-router-guide.md`
- Check Cosmo docs: https://cosmo-docs.wundergraph.com/

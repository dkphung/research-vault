---
tags: [graphql]
date: 2024-12-22
status: complete
---

# Schema Management: Quick Decision Guide

**Choose how to manage schemas across your federated GraphQL microservices.**

## Two Options

### Option 1: File-Based (Simple)
Schemas live in subgraph repos, referenced by file path.

### Option 2: Pattern B (Scalable)
Schemas published to router repo's `schemas/` directory.

## Decision Tree

```
How many GraphQL services do you have (or plan to have)?

1-3 services
  │
  └─→ Use File-Based
      Simple, no setup needed
      Already working in this repo!

4-5 services
  │
  ├─→ Plan to grow? → Use Pattern B
  │                   (Avoid migration later)
  │
  └─→ Staying small? → Use File-Based
                        (Keep it simple)

6+ services
  │
  └─→ Use Pattern B
      File-based becomes painful at this scale
```

## Quick Comparison

|  | File-Based | Pattern B |
|--|------------|-----------|
| **Setup** | ✅ Already works | ⚠️ 1-2 hours setup |
| **Local dev** | Clone all repos | Clone router only |
| **CI/CD** | N+1 checkouts | 1 checkout |
| **Best for** | 1-5 services | 5+ services |
| **Scalability** | ❌ ~5 max | ✅ Unlimited |
| **Complexity** | Low | Medium |

## File-Based Overview

**How it works:**
```yaml
# compose.yaml points to other repos
subgraphs:
  - name: comments
    schema:
      file: ../comments-server/schema.graphql
```

**Local dev:**
```bash
# Must clone all repos
git clone cosmo-gateway
git clone comments-server
git clone users-server
# Directory structure must match
```

**CI/CD:**
```yaml
# Must checkout all repos
- uses: actions/checkout@v4  # Router
- uses: actions/checkout@v4  # Comments
- uses: actions/checkout@v4  # Users
# Add one per service
```

**Pros:** Simple, fast, already working
**Cons:** Doesn't scale, complex CI, directory dependencies

---

## Pattern B Overview

**How it works:**
```yaml
# compose.yaml points to local schemas/
subgraphs:
  - name: comments
    schema:
      file: ./schemas/comments.graphql
```

**Local dev:**
```bash
# Just clone router repo
git clone cosmo-gateway
cd cosmo-gateway
# Schemas already inside!
ls schemas/
pnpm dev
```

**CI/CD:**
```yaml
# Just one checkout
- uses: actions/checkout@v4
- run: pnpm compose:check
```

**Automated publishing:**
Each subgraph has workflow that pushes its schema to `cosmo-gateway/schemas/`

**Pros:** Scales infinitely, simple dev, fast CI
**Cons:** Initial setup, requires GitHub tokens

---

## When to Migrate

**Stay with file-based if:**
- You have ≤3 services
- No plans to grow significantly
- Team is comfortable with multi-repo workflow

**Migrate to Pattern B when:**
- ✅ Planning to add 3+ more services
- ✅ CI is slow (too many checkouts)
- ✅ New developers struggle with setup
- ✅ Directory structure issues

**Migration time:** 2-3 hours

---

## Current Recommendation

**Your project:** 1 service now, planning 10+ services

**Recommended:** Pattern B

**Why:**
- You're planning to scale to 10+ services
- 1-2 hour setup now saves migration pain later
- Better foundation for adding services
- Simpler for new developers

**Alternative:**
Stay with file-based until you hit 4-5 services, then migrate.

---

## Implementation Guides

**File-Based:** Already working! See `CLAUDE.md` for current setup.

**Pattern B:** See `docs/PATTERN-B-SETUP.md` for complete guide.

---

## What Not to Use

We evaluated but **don't recommend** these alternatives:

- ❌ **Separate schemas repo (Pattern A)** - Adds complexity without benefit for single-team setup
- ❌ **Introspection** - Chicken-egg problems, no versioning
- ❌ **Schema registry** - Overkill, external dependency, cost
- ❌ **NPM packages** - Unnecessary complexity

Stick to file-based or Pattern B.

---

## Next Steps

1. **Count your services** (current + planned)
2. **Choose based on decision tree above**
3. **If Pattern B:** Follow `docs/PATTERN-B-SETUP.md`
4. **If File-Based:** Keep current setup, revisit at 4-5 services

# Pattern B: Schemas in Router Repo (Recommended)

**Best for:** Single team, 10-30 services, simplicity

## Architecture

```
┌─────────────────────────────┐
│ Subgraph Repos              │
│  comments-server/           │
│  users-server/              │
│  posts-server/              │
│  ... (10+ more)             │
└──────────┬──────────────────┘
           │ Each pushes schema
           │ directly to router repo
           ↓
┌─────────────────────────────┐
│ cosmo/ (Router Repo)        │
│   schemas/  ← schemas here! │
│     comments.graphql        │
│     users.graphql           │
│     posts.graphql           │
│   compose.yaml              │
│   router.json               │
└─────────────────────────────┘
```

## Setup (One-Time)

### 1. Create schemas/ directory in router repo

```bash
cd /Users/little/Projects/cosmo

# Create schemas directory
mkdir -p schemas

# Copy existing schema (if you have one)
cp ../comments-server/src/graphql/comments.schema.graphql schemas/comments.graphql

# Commit to router repo
git add schemas/
git commit -m "Add schemas directory"
git push
```

### 2. Update .gitignore

```bash
# Make sure schemas/ is NOT ignored
# Remove this line from .gitignore if present:
# schemas/

# Verify
cat .gitignore | grep -v "^#" | grep schemas
# Should return nothing (schemas/ should be tracked)
```

### 3. Update compose.yaml

```yaml
# compose.yaml - no changes needed if already using ./schemas/
version: 1

subgraphs:
  - name: comments
    routing_url: http://comments-server:3000/graphql
    schema:
      file: ./schemas/comments.graphql  # ← Points to committed file
```

### 4. Update package.json

```json
{
  "scripts": {
    "compose": "wgc router compose -i compose.yaml -o router.json",
    "compose:check": "wgc router compose -i compose.yaml -o router.json",
    "compose:watch": "wgc router compose -i compose.yaml -o router.json --watch"
  }
}
```

**Note:** No `schemas:fetch` script needed! Schemas are committed.

## Subgraph Setup (Repeat for Each Service)

### For Each of Your 10+ Subgraphs:

**1. Create publish workflow**

Create `.github/workflows/publish-schema.yml` in each subgraph repo:

```yaml
name: Publish Schema to Router

on:
  push:
    branches: [main]
    paths:
      - 'schema.graphql'
      - 'src/graphql/**/*.graphql'

env:
  SUBGRAPH_NAME: comments  # ⚠️ CHANGE THIS for each service
  ROUTER_REPO: yourorg/cosmo
  SCHEMA_FILE: src/graphql/comments.schema.graphql  # ⚠️ CHANGE THIS

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout subgraph
        uses: actions/checkout@v4

      - name: Publish to router repo
        env:
          GITHUB_TOKEN: ${{ secrets.ROUTER_PUBLISH_TOKEN || secrets.GITHUB_TOKEN }}
        run: |
          echo "Publishing $SUBGRAPH_NAME schema to $ROUTER_REPO"

          # Clone router repo
          git clone "https://${GITHUB_TOKEN}@github.com/${ROUTER_REPO}.git" /tmp/router
          cd /tmp/router

          # Ensure schemas directory exists
          mkdir -p schemas

          # Copy schema file
          cp "$GITHUB_WORKSPACE/$SCHEMA_FILE" "schemas/${SUBGRAPH_NAME}.graphql"

          # Configure git
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

          # Commit if changed
          git add "schemas/${SUBGRAPH_NAME}.graphql"

          if git diff --staged --quiet; then
            echo "No schema changes detected"
          else
            git commit -m "Update ${SUBGRAPH_NAME} schema

Published from: ${{ github.repository }}@${{ github.sha }}
Triggered by: ${{ github.event.head_commit.message }}"
            git push
            echo "✅ Schema published successfully"
          fi

      - name: Trigger router composition
        if: success()
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const [owner, repo] = process.env.ROUTER_REPO.split('/');

            try {
              await github.rest.actions.createWorkflowDispatch({
                owner,
                repo,
                workflow_id: 'compose.yml',
                ref: 'main'
              });
              console.log('✅ Triggered router composition');
            } catch (error) {
              console.log('Note: Could not trigger router (may need ROUTER_PUBLISH_TOKEN with workflow permissions)');
            }
```

**2. Create GitHub token**

Each subgraph needs permission to push to router repo:

```bash
# Create Personal Access Token with:
# - repo (full)
# - workflow (if triggering workflows)

# Add as secret to EACH subgraph repo:
gh secret set ROUTER_PUBLISH_TOKEN --body "ghp_xxxxx" --repo yourorg/comments-server
gh secret set ROUTER_PUBLISH_TOKEN --body "ghp_xxxxx" --repo yourorg/users-server
# ... repeat for all 10+ repos
```

**Or use GitHub App** (recommended for orgs):
- Create GitHub App with `contents: write` permission
- Install on organization
- Use app token instead

## Local Development Workflow

### Developer Onboarding

```bash
# Just clone router repo - schemas are already inside!
git clone git@github.com:yourorg/cosmo-gateway.git
cd cosmo-gateway

# Install dependencies
pnpm install

# Schemas are already here
ls schemas/
# comments.graphql  users.graphql  posts.graphql  ...

# Compose and run
pnpm compose
pnpm dev
```

**That's it!** No multi-repo cloning, no fetch scripts.

### Daily Development

```bash
# Pull latest (includes schema updates)
git pull

# Check what changed
git log --oneline schemas/

# Compose and test
pnpm compose:check
pnpm dev
```

### Updating a Schema Manually

Sometimes you want to test a schema change before it's deployed:

```bash
# Edit schema locally
vim schemas/users.graphql

# Test composition
pnpm compose:check

# Commit if intentional change
git add schemas/users.graphql
git commit -m "Update users schema: add email field"
git push
```

## CI/CD Configuration

### Router Repo CI/CD

`.github/workflows/compose.yml`:

```yaml
name: Compose and Deploy Router

on:
  push:
    branches: [main]
    paths:
      - 'schemas/**'
      - 'compose.yaml'
      - 'router/**'
  workflow_dispatch:

jobs:
  compose:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        # Schemas are already in the repo - no fetch needed!

      - name: Setup
        uses: pnpm/action-setup@v2

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Validate composition
        run: pnpm compose:check

      - name: Upload router.json
        uses: actions/upload-artifact@v4
        with:
          name: router-config
          path: router.json

  build:
    needs: compose
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Download router.json
        uses: actions/download-artifact@v4
        with:
          name: router-config

      - name: Build and push Docker image
        # ... your Docker build steps
```

**Key point:** No schema fetching - they're already committed!

## Schema Review Process

### Option 1: Auto-Merge (Fast Iteration)

Subgraphs publish schemas automatically → Router auto-composes

**Good for:** Small teams, high trust

### Option 2: PR Review (Governance)

Configure subgraph workflow to create PR instead of direct push:

```yaml
# In subgraph publish workflow
- name: Create PR to router repo
  run: |
    cd /tmp/router
    git checkout -b "update-${SUBGRAPH_NAME}-schema-${{ github.sha }}"
    git add schemas/${SUBGRAPH_NAME}.graphql
    git commit -m "Update ${SUBGRAPH_NAME} schema"
    git push origin "update-${SUBGRAPH_NAME}-schema-${{ github.sha }}"

    gh pr create \
      --title "Update ${SUBGRAPH_NAME} schema" \
      --body "Schema updated from ${{ github.repository }}@${{ github.sha }}" \
      --base main
```

**Good for:** Larger teams, want review of schema changes

## Comparison to Other Patterns

### vs. Pattern A (Dedicated Schemas Repo)

| Aspect | Pattern B (This) | Pattern A |
|--------|------------------|-----------|
| Repos to clone | 1 (just router) | 2 (router + schemas) |
| Local dev | `git pull` | `pnpm schemas:fetch` |
| CI complexity | Simple | Medium (fetch step) |
| Commit history | Mixed | Separated |
| Access control | All-or-nothing | Granular |

**Pattern B is simpler for most teams.**

### vs. File-Based (Current Multi-Repo)

| Aspect | Pattern B | File-Based |
|--------|-----------|------------|
| Repos to checkout | 1 | 11+ (router + all subgraphs) |
| Schema updates | Automatic (CI) | Manual |
| CI steps | 1 checkout | 11+ checkouts |
| Scalability | ✅ Unlimited | ❌ ~5 services max |

**Pattern B scales much better.**

## Monitoring & Notifications

### Schema Change Notifications

Add to router repo's `.github/workflows/compose.yml`:

```yaml
- name: Notify on schema change
  if: github.event_name == 'push'
  run: |
    # Get changed schemas
    CHANGED=$(git diff --name-only HEAD~1 HEAD -- schemas/)

    if [ -n "$CHANGED" ]; then
      echo "📝 Schema changes detected:"
      echo "$CHANGED"

      # Post to Slack (optional)
      curl -X POST ${{ secrets.SLACK_WEBHOOK }} \
        -H 'Content-Type: application/json' \
        -d "{\"text\": \"Schema updated: \n$CHANGED\"}"
    fi
```

## Troubleshooting

### Schema not updating after subgraph push

**Check:**
```bash
# 1. Did subgraph workflow run?
gh run list --repo yourorg/comments-server

# 2. Did it push to router repo?
cd cosmo-gateway
git log schemas/comments.graphql

# 3. Check workflow logs
gh run view --repo yourorg/comments-server
```

### Permission denied when publishing

**Fix:** Ensure `ROUTER_PUBLISH_TOKEN` has `repo` scope

```bash
# Verify token has correct permissions
gh auth status

# Re-create with correct scope
gh secret set ROUTER_PUBLISH_TOKEN --repo yourorg/comments-server
```

## Migration from File-Based

**If you're currently using `../comments-server/schema.graphql`:**

```bash
# 1. Copy all schemas to router repo
mkdir -p schemas
cp ../comments-server/schema.graphql schemas/comments.graphql
cp ../users-server/schema.graphql schemas/users.graphql
# ... copy all

# 2. Update compose.yaml paths
# Change: ../comments-server/schema.graphql
# To:     ./schemas/comments.graphql

# 3. Commit to router repo
git add schemas/ compose.yaml
git commit -m "Migrate to Pattern B: schemas in router repo"
git push

# 4. Add publish workflows to each subgraph
# (so future updates are automatic)
```

## Summary

**Pattern B (Schemas in Router Repo) is recommended because:**

✅ **Simpler** - Just one repo to manage
✅ **Faster** - No fetch step in CI or local dev
✅ **Atomic** - Schema + compose.yaml changes together
✅ **Scales** - Handles 10-30 services easily
✅ **Standard** - Familiar Git workflow

**Use this unless you need:**
- Complex access control (different teams, different permissions)
- Multiple routers sharing schemas
- Very large scale (50+ services)

---
tags: [graphql]
date: 2024-12-22
status: complete
---

# Schema Publishing

This service automatically publishes its GraphQL schema to the router repository.

## How It Works

When you update the schema file and push to `main` or `trunk`:

1. **Edit schema:** `src/graphql/comments.schema.graphql`
2. **Commit and push:**
   ```bash
   git add src/graphql/comments.schema.graphql
   git commit -m "Add new field to Comment type"
   git push
   ```
3. **GitHub Action runs:** `.github/workflows/publish-schema.yml`
4. **Schema published** to `cosmo-gateway/schemas/comments.graphql`
5. **Router repo updated** automatically

## Setup Required

### GitHub Token

The publish workflow needs permission to push to the router repository.

**Option 1: Use Personal Access Token (Recommended for getting started)**

1. Create GitHub PAT with `repo` scope: https://github.com/settings/tokens
2. Add as repository secret:
   ```bash
   gh secret set ROUTER_PUBLISH_TOKEN --body "ghp_xxxxx" --repo yourorg/comments-server
   ```

**Option 2: Use GitHub App (Recommended for production)**

More secure for organizations. See: https://docs.github.com/en/apps

### Workflow Configuration

The workflow is configured in `.github/workflows/publish-schema.yml`:

```yaml
env:
  SUBGRAPH_NAME: comments                           # Service name
  ROUTER_REPO: little/cosmo-gateway                 # Router repo (update this!)
  SCHEMA_FILE: src/graphql/comments.schema.graphql  # Schema file path
```

**Important:** Update `ROUTER_REPO` to match your GitHub username/organization!

## Testing Schema Changes Locally

Before pushing, test that your schema composes correctly:

```bash
# In router repo (cosmo-gateway)
cd /path/to/cosmo-gateway

# Copy schema manually for testing
cp ../comments-server/src/graphql/comments.schema.graphql schemas/comments.graphql

# Test composition
pnpm compose:check

# If successful, push your comments-server changes
# The workflow will publish automatically
```

## Manual Publishing

If you need to manually trigger the workflow:

```bash
# Via GitHub CLI
gh workflow run publish-schema.yml --repo yourorg/comments-server

# Or via GitHub UI
# Go to Actions → Publish GraphQL Schema → Run workflow
```

## Troubleshooting

### Workflow fails with "permission denied"

- Verify `ROUTER_PUBLISH_TOKEN` secret exists and has `repo` scope
- Check token hasn't expired

### Schema not updating in router repo

1. Check workflow ran successfully:
   ```bash
   gh run list --workflow=publish-schema.yml --repo yourorg/comments-server
   ```

2. Check workflow logs:
   ```bash
   gh run view --log --repo yourorg/comments-server
   ```

3. Verify schema file path is correct in workflow

### Schema composition fails in router

- The workflow only publishes the schema
- It doesn't validate composition
- Test locally with `pnpm compose:check` in router repo before pushing

## Related Documentation

- **Router Repo:** See `cosmo-gateway/docs/PATTERN-B-SETUP.md`
- **Schema Management:** See `cosmo-gateway/docs/SCHEMA-MANAGEMENT-DECISION.md`

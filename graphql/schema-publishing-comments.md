# Schema Publishing

This service automatically publishes its GraphQL schema to the federation-router repository.

## How It Works

When you update the schema file and push to `main` or `trunk`:

1. **Edit schema:** `src/graphql/comments.schema.graphql`
2. **Commit and push:**
   ```bash
   git add src/graphql/comments.schema.graphql
   git commit -m "Add new field to Comment type"
   git push
   ```
3. **GitHub Action runs:** `.github/workflows/compose-federation-schema.yml`
4. **Schema composed** using Rover CLI
5. **Schema published** to `federation-router/subgraphs/comments-server.schema.graphql`

## Setup Required

### GitHub Token

The workflow needs permission to push to the federation-router repository.

The secret `RSS_SVC_GITHUB_TOKEN` must be configured with `repo` scope to push to `risk-and-safety/federation-router`.

## Testing Schema Changes Locally

Before pushing, test that your schema composes correctly:

```bash
# Run local composition (uses Rover CLI)
pnpm compose

# This will:
# 1. Copy schema to federation/subgraphs/
# 2. Run Rover supergraph compose
# 3. Report any composition errors
```

## Manual Publishing

If you need to manually trigger the workflow:

```bash
# Via GitHub CLI
gh workflow run compose-federation-schema.yml

# Or via GitHub UI
# Go to Actions → Federation Schema Composition → Run workflow
```

## Troubleshooting

### Workflow fails with "permission denied"

- Verify `RSS_SVC_GITHUB_TOKEN` secret exists and has `repo` scope
- Check token hasn't expired

### Schema not updating in federation-router

1. Check workflow ran successfully:
   ```bash
   gh run list --workflow=compose-federation-schema.yml
   ```

2. Check workflow logs:
   ```bash
   gh run view --log
   ```

3. Verify schema file path matches `src/**/*.graphql` or `graphql/**/*.graphql`

### Schema composition fails

- Run `pnpm compose` locally to debug composition errors
- Check for breaking changes with `pnpm schema:check`
- Show schema differences with `pnpm schema:diff`

## Related Documentation

- **Router Repo:** See `federation-router/README.md`
- **Local Scripts:** See `scripts/compose-federation-schema.ts`

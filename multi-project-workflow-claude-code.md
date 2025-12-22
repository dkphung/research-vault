---
tags: [architecture]
date: 2024-12-22
status: complete
---

# Multi-Project Workflow with Claude Code: Next.js + Fastify Backend

**Date**: 2025-11-07
**Context**: Research how to effectively work with the Next.js client app and a sibling Fastify backend service using Claude Code.

## Project Structure

```
/client-onboarding-poc/     # Next.js app (current working directory)
/comment/                    # Node.js Fastify REST API (sibling directory)
```

## Use Case Requirements

Based on your workflow:
- **Frequency**: 30-50% of features touch both projects
- **Git workflow**: Separate commits/PRs per repository
- **Type sharing**: No shared types currently

## Research Findings

### Option 1: `--add-dir` Flag (RECOMMENDED)

**How it works**: Add the Fastify backend directory to your Claude Code session while keeping the Next.js app as your primary working directory.

**Setup**:
```bash
# At launch
cd /Users/little/Projects/client-onboarding-poc
claude --add-dir ../comment

# Or during an active session
/add-dir ../comment
```

**Advantages**:
- ✅ Both codebases available in single session
- ✅ Can read/edit files in both projects
- ✅ Git operations work correctly in each repo
- ✅ Primary working directory stays in Next.js app
- ✅ Can run dev servers for both projects
- ✅ Simple to use, no configuration required

**Limitations**:
- ⚠️ CLAUDE.md files from added directories are NOT automatically loaded
  - Need to manually reference backend's CLAUDE.md if it exists
  - Consider consolidating project instructions
- ⚠️ Higher token usage if both codebases are large
- ⚠️ Need to restart Claude to remove added directory

**Best for**: Your use case where you sometimes need both, but keep them mostly separate.

### Option 2: Launch from Parent Directory

**How it works**: Start Claude Code from the parent directory containing both projects.

**Setup**:
```bash
cd /Users/little/Projects
claude
# Now both client-onboarding-poc/ and comment/ are accessible
```

**Advantages**:
- ✅ Both projects equally accessible
- ✅ CLAUDE.md files from both projects automatically loaded (hierarchical)
- ✅ Clean, symmetric access

**Limitations**:
- ⚠️ No clear "primary" project - working directory is parent
- ⚠️ May need to use full paths more often
- ⚠️ All projects in parent directory are in scope (could be many)
- ⚠️ Less suitable if you primarily work in one project

**Best for**: True monorepo scenarios or when projects are equally important.

### Option 3: MCP Filesystem Server (ADVANCED)

**How it works**: Configure a Model Context Protocol (MCP) filesystem server to provide controlled access to the backend directory.

**Setup**: Would require configuration in `.claude/settings.local.json` or `.mcp.json`:
```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/little/Projects/comment"]
    }
  }
}
```

**Advantages**:
- ✅ Fine-grained control over directory access
- ✅ Can configure multiple backend services
- ✅ Potentially better for security/isolation
- ✅ Persistent configuration (doesn't need flag every time)

**Limitations**:
- ⚠️ More complex setup
- ⚠️ Additional layer of abstraction
- ⚠️ May have performance overhead
- ⚠️ Overkill for simple sibling directory access

**Best for**: Complex scenarios with many services or security requirements.

### Option 4: Separate Claude Sessions (NOT RECOMMENDED)

**How it works**: Run separate Claude Code sessions for each project, manually coordinate changes.

**Why not**: Defeats the purpose of integrated development, loses context between projects, manual coordination is error-prone.

## Recommended Workflow with `--add-dir`

Based on your requirements, here's the recommended approach:

### 1. Launch Setup

Create a shell alias or script for convenience:

```bash
# In your ~/.zshrc or ~/.bashrc
alias claude-client="cd /Users/little/Projects/client-onboarding-poc && claude --add-dir ../comment"
```

### 2. Typical Feature Development Flow

When adding a feature that touches both projects:

```
1. Start Claude Code with both directories:
   $ claude-client

2. Discuss the feature with Claude, providing context:
   "I need to add a commenting feature. The Next.js app should have a Server Action
   that calls the Fastify /comments API endpoint. Let's start by adding the endpoint
   to the backend."

3. Claude can now:
   - Read the Fastify backend code: src/routes/comments.ts (in comment/)
   - Add the new endpoint in the backend
   - Create TypeScript types for the API contract
   - Switch to the Next.js app
   - Create the Server Action that calls the endpoint
   - Update UI components to use the Server Action

4. Test both services:
   - Run Fastify dev server in background: cd ../comment && pnpm dev
   - Run Next.js dev server: pnpm dev
   - Claude can monitor both dev server outputs

5. Commit to each repo separately:
   - Git operations in client-onboarding-poc/ affect Next.js repo
   - Git operations in ../comment/ affect Fastify repo
   - Use /gc slash command or manual commits as needed
```

### 3. Project Documentation

**In Next.js CLAUDE.md**, add a section about the backend:

```markdown
## Related Projects

This Next.js app communicates with a sibling Fastify backend at `../comment/`.

When working on features that span both projects:
1. Launch Claude with: `claude --add-dir ../comment`
2. Backend API routes are in `../comment/src/routes/`
3. API base URL in development: `http://localhost:3001` (or whatever port)
4. Server Actions that call the backend are in `src/server/*/actions.ts`
```

**Consider creating** `../comment/CLAUDE.md` with:

```markdown
# Comment Service - Fastify Backend

This is the REST API backend for the client onboarding POC.

## Related Projects

This service is consumed by the Next.js client app at `../client-onboarding-poc/`.

When adding endpoints:
1. Define routes in `src/routes/`
2. Export TypeScript types for request/response
3. Update the Next.js Server Actions to consume the new endpoints
```

## Bonus Recommendation: Type Sharing

Since you currently have no shared types, consider establishing a contract between projects:

### Approach A: OpenAPI/Swagger (RECOMMENDED)

In the Fastify backend:

```typescript
// Use @fastify/swagger
import swagger from '@fastify/swagger';

fastify.register(swagger, {
  openapi: {
    info: { title: 'Comment API', version: '1.0.0' }
  }
});

// Generate types for the Next.js app
// Use openapi-typescript to generate TypeScript types from the OpenAPI spec
```

In the Next.js app:

```bash
# Generate types from backend's OpenAPI spec
pnpm add -D openapi-typescript
npx openapi-typescript http://localhost:3001/documentation/json -o src/types/backend-api.ts
```

**Benefits**:
- Single source of truth (backend owns the contract)
- Type safety in Next.js Server Actions
- Auto-generated types stay in sync
- Industry standard approach

### Approach B: Shared Types Package (Monorepo-style)

Create a shared types package that both projects depend on:

```
/client-onboarding-poc/
/comment/
/shared-types/              # New shared package
  package.json
  src/
    comments.ts
    users.ts
```

**Benefits**:
- Shared code between frontend/backend
- Can include validation schemas (Zod)
- More control than generated types

**Drawbacks**:
- Need to publish/link package
- More coordination required
- Versioning complexity

### Approach C: Type Copying (Current State)

Manually maintain types in both projects.

**Drawbacks**:
- Prone to drift
- Manual synchronization
- No single source of truth
- Not recommended long-term

## Implementation Checklist

For your next feature that spans both projects:

- [ ] Launch Claude with `--add-dir`: `claude --add-dir ../comment`
- [ ] Create CLAUDE.md in backend project documenting its purpose
- [ ] Update Next.js CLAUDE.md to reference backend project
- [ ] Consider setting up OpenAPI/Swagger in Fastify backend
- [ ] Create a shell alias for convenient launch
- [ ] Test the workflow with a small feature

## Related Resources

- [Claude Code Multi-Directory Support](https://apidog.com/blog/claude-code-multi-directory-support/)
- [Claude Code --add-dir Guide](https://claudelog.com/faqs/--add-dir/)
- [CLAUDE.md Hierarchical Configuration](https://www.letanure.dev/blog/2025-07-31--claude-code-part-2-claude-md-configuration)
- [Fastify Swagger Plugin](https://github.com/fastify/fastify-swagger)
- [openapi-typescript](https://github.com/drwpow/openapi-typescript)

## Conclusion

**For your use case, the `--add-dir` flag is the best solution.** It's simple, requires no configuration, and provides exactly what you need: occasional access to the backend code while primarily working in the Next.js app.

As you scale or if the backend grows more complex, consider:
1. Setting up OpenAPI/Swagger for type contracts
2. Creating more comprehensive CLAUDE.md files in both projects
3. Potentially moving to a monorepo structure if the projects become tightly coupled

---

**Next Steps**: Try the `--add-dir` approach with your next cross-project feature and refine based on what works best for your workflow.

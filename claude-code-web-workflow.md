---
tags:
  - claude-code
  - ai-coding
  - workflow
  - productivity
  - anthropic
date: 2026-01-07
source: Boris Cherny (Creator of Claude Code)
---

# Claude Code Web Workflow: Boris Cherny's Approach

## Executive Summary

Boris Cherny, the creator of Claude Code at Anthropic, has publicly shared his workflow for maximizing productivity with Claude Code. The key insight is that he runs **15+ parallel Claude sessions** across terminal, web (claude.ai/code), and mobile, treating Claude as a scalable workforce rather than a single assistant. The workflow centers on three pillars: parallel execution, shared memory via CLAUDE.md, and verification loops.

## Key Findings

### 1. Parallel Session Architecture

Boris runs multiple Claude instances simultaneously across platforms:

| Platform | Sessions | Primary Use |
|----------|----------|-------------|
| Terminal | 5 numbered tabs | Active coding, immediate feedback |
| Web (claude.ai/code) | 5-10 sessions | Background tasks, code review, PR generation |
| iOS App | As needed | Quick prompts, task initiation on-the-go |

**Cross-platform synergy**: Terminal handles active coding with immediate feedback, web enables asynchronous background work and code review, and iOS allows task initiation from anywhere.

**Session handoff**: The `&` operator backgrounds tasks to the web, while `--teleport` moves sessions between devices. Example workflow:
```bash
# Background a task to web
claude & "refactor the auth module"

# Later, teleport it back to terminal
claude --teleport <session-id>
```

### 2. Claude Code on the Web (claude.ai/code)

**What it is**: A cloud-based coding agent that runs in Anthropic-managed isolated VMs. Available to Pro, Max, Team, and Enterprise users.

**Key capabilities**:
- Connect GitHub repositories directly
- Run multiple tasks in parallel across different repos
- Automatic PR creation
- Works on repos not on your local machine
- Custom network configuration (no network, trusted domains, or full access)

**Ideal use cases**:
- Architectural questions about projects
- Well-defined bugfixes and routine tasks
- Backend changes with test-driven development
- Tasks where Claude can write tests first, then write code to pass them

**Security architecture**:
- Each session runs in an isolated sandbox
- Network access is configurable and limited by default
- Git credentials are protected via secure proxy
- Claude can only access authorized repositories

### 3. The Teleport Feature

Teleporting allows seamless handoff between web and CLI:

**From Web to CLI**:
1. Complete work or reach a handoff point on claude.ai/code
2. Click "Open in CLI" button
3. Copy the command: `claude --teleport <session-id>`
4. Run in terminal (requires clean git working directory)
5. Full session history loads (~50% of context window)

**Current limitations**:
- Unidirectional only (web → CLI)
- Cannot teleport CLI → web or CLI → remote machine
- Follow-up prompts typed in web may not be fully captured

**When to teleport**:
- When complexity increases beyond well-defined tasks
- When you need local debugging or testing
- When you need full VS Code integration
- For large refactoring requiring file tree visibility

### 4. Shared Memory: CLAUDE.md

The team maintains a single `CLAUDE.md` file in their git repository that serves as shared memory:

**Contents**:
- Common bash commands and aliases
- Code style guidelines
- Repository conventions and patterns
- Anti-patterns to avoid
- Lessons learned from mistakes

**Maintenance workflow**:
- When Claude makes a mistake, add it to CLAUDE.md immediately
- During code review, tag Claude via GitHub Action to update the file
- This creates "compounding engineering" where knowledge accumulates

**Example CLAUDE.md section**:
```markdown
## Don't Do This
- Never use `any` type in TypeScript
- Always await async functions in tests
- Don't commit console.log statements

## Conventions
- Use `pnpm` for package management
- Run `pnpm typecheck` before committing
- Test files live next to source as `*.spec.ts`
```

### 5. Two-Phase Workflow: Plan → Execute

**Phase 1 - Planning (Interactive)**:
- Start sessions in Plan mode: `Shift+Tab` twice
- Iterate back and forth with Claude until the plan is solid
- A good plan is critical—invest time upfront to save time in execution

**Phase 2 - Execution (Automated)**:
- Toggle auto-accept mode: `Shift+Tab` once
- Claude executes the approved plan without interruption
- Can often "one-shot" the implementation with a good plan

**Model choice**: Boris uses Opus 4.5 with extended thinking for everything. Despite being slower than Sonnet, it requires less steering and has better tool use, making it faster overall.

### 6. Automation Components

#### Slash Commands
Pre-configured workflows stored in `.claude/commands/`:
- `/commit-push-pr` - Full git workflow (used dozens of times daily)
- `/quick-commit` - Rapid staging and commit
- `/test-and-fix` - Run tests and resolve failures
- `/review-changes` - Analyze uncommitted changes
- `/first-principles` - Deconstruct problems to fundamentals

#### Subagents
Specialized AI personas with separate context windows:
- `code-simplifier` - Refactors and reduces complexity post-implementation
- `verify-app` - Runs end-to-end tests before shipping
- `code-architect` - Reviews design and architecture
- `build-validator` - Pre-deployment build verification

#### Hooks
- **PostToolUse hooks**: Auto-format code via Prettier after edits
- **Pre-commit hooks**: Enforce verification before pushing
- **Stop hooks**: Deterministic checks that block exit until conditions met

#### Permissions Management
Configure in `.claude/settings.json`:
```json
{
  "permissions": {
    "allow": [
      "Bash(npm:*)",
      "Bash(pnpm:*)",
      "Bash(git:*)",
      "Bash(gh:*)"
    ]
  }
}
```
Use `/permissions` to pre-allow safe commands rather than `--dangerously-skip-permissions`.

### 7. MCP (Model Context Protocol) Integration

External tool connections configured in `.mcp.json`:
- **Slack**: Notifications and updates
- **BigQuery**: Analytics queries
- **Sentry**: Error log access

### 8. Verification: The Critical Success Factor

> "The most important thing to get great results out of Claude Code is to give Claude a way to verify its work. If Claude has that feedback loop, it will 2-3x the quality of the final result."

**Verification methods by domain**:

| Domain | Verification Method |
|--------|---------------------|
| Command-line | `npm test`, typecheck, lint |
| Test suites | Unit, integration, E2E with coverage thresholds |
| Browser/UI | Chrome extension validation |
| API | curl verification of endpoints |

**For claude.ai/code UI work**: Claude tests every change using the Claude Chrome extension. It opens a browser, tests the UI, and iterates until the code works and the UX feels good.

## Practical Implementation Guide

### Getting Started (Priority Order)

1. **Create CLAUDE.md** - Start documenting conventions and anti-patterns
2. **Set up one slash command** - Begin with `/commit-push-pr` or similar daily workflow
3. **Configure verification** - Ensure Claude can run tests, typecheck, and lint
4. **Set up permissions** - Pre-allow safe bash commands
5. **Add formatting hooks** - Auto-format on edit to prevent CI failures

### Scaling Up

6. **Add more slash commands** for repetitive workflows
7. **Create subagents** for specialized tasks
8. **Configure MCP servers** for external tool access
9. **Set up parallel web sessions** for background work
10. **Implement stop hooks** for quality gates

### Recommended File Structure

```
your-project/
├── CLAUDE.md                    # Shared memory file
├── .claude/
│   ├── settings.json            # Permissions and hooks
│   ├── commands/
│   │   ├── commit-push-pr.md    # Slash command definitions
│   │   ├── test-and-fix.md
│   │   └── review-changes.md
│   └── agents/
│       ├── code-simplifier.md   # Subagent definitions
│       └── verify-app.md
└── .mcp.json                    # MCP server configuration
```

## Sources

- [VentureBeat: The creator of Claude Code just revealed his workflow](https://venturebeat.com/technology/the-creator-of-claude-code-just-revealed-his-workflow-and-developers-are)
- [DEV Community: How the Creator of Claude Code Uses Claude Code](https://dev.to/sivarampg/how-the-creator-of-claude-code-uses-claude-code-a-complete-breakdown-4f07)
- [Twitter Thread Reader: Boris Cherny's original thread](https://twitter-thread.com/t/2007179832300581177)
- [GitHub: bcherny-claude configuration repository](https://github.com/0xquinto/bcherny-claude)
- [Claude Code Docs: Claude Code on the web](https://code.claude.com/docs/en/claude-code-on-the-web)
- [Simon Willison: Claude Code for web - a new asynchronous coding agent](https://simonwillison.net/2025/Oct/20/claude-code-for-web/)
- [Anthropic Blog: Claude Code on the web](https://claude.com/blog/claude-code-on-the-web)
- [GitHub Issues: Universal session teleporting discussion](https://github.com/anthropics/claude-code/issues/14666)

## Appendix: Quick Reference

### Essential Commands

```bash
# Start in plan mode
Shift+Tab (twice)

# Toggle auto-accept
Shift+Tab (once)

# Background to web
claude & "your task"

# Teleport from web
claude --teleport <session-id>

# Run slash command
/commit-push-pr
```

### Key Principles

1. **Parallel > Sequential**: Run multiple Claudes simultaneously
2. **Plan > Execute**: Invest in planning to save execution time
3. **Verify Everything**: Give Claude feedback loops to self-correct
4. **Compound Knowledge**: Update CLAUDE.md continuously
5. **Automate Repetition**: Create slash commands for daily workflows

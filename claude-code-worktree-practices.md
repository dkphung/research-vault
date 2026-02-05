---
tags:
  - claude-code
  - git-worktrees
  - workflow
  - parallel-development
  - productivity
date: 2026-01-08
---

# Claude Code Git Worktree Best Practices

Comprehensive research on using git worktrees with Claude Code for parallel development workflows.

## Executive Summary

Git worktrees enable running multiple Claude Code sessions simultaneously without conflicts. This research covers terminal management, session handling, built-in skills, and practical recommendations for maximizing productivity with this workflow.

**Key Findings:**
- Yes, separate terminals are needed for different worktrees (one Claude instance per worktree)
- Claude Code has native worktree skills (`/worktree:init`, `/worktree:parallel`, `/worktree:merge`, `/worktree:cleanup`)
- The `/resume` picker shows sessions from the same git repository, including worktrees
- Third-party tools like Claude Squad and GitButler offer alternative approaches

---

## 1. Terminal Management for Worktrees

### Do You Need Separate Terminals?

**Yes.** Each worktree requires its own terminal session running Claude Code. This is fundamental to the isolation model.

**Why Separate Terminals:**
- Each worktree has its own working directory with isolated files
- Claude Code sessions are bound to the directory they were started in
- Running Claude in a worktree means that instance only sees/modifies that worktree's files
- Changes in one worktree do not affect others

### Terminal Organization Strategies

**iTerm2 Split Panes (macOS):**
- `Cmd+D` for vertical splits
- `Cmd+Shift+D` for horizontal splits
- `Cmd+I` to name each pane by ticket/feature
- Navigate with `Cmd+Option+Arrow`

**tmux Sessions:**
- Create named sessions per worktree: `tmux new -s feature-auth`
- Switch between sessions: `tmux attach -t feature-auth`
- List sessions: `tmux ls`

**VS Code Multi-root Workspaces:**
- Add each worktree as a folder in the workspace
- Use integrated terminals per folder
- Each terminal runs Claude Code in its respective worktree

### Practical Recommendations

1. **Keep terminals open between work sessions** - Claude maintains context about what was being accomplished
2. **Use descriptive naming** - Label terminals/panes by feature or ticket number
3. **Limit concurrent sessions** - Running 5-7 parallel sessions is practical; more becomes difficult to track
4. **Consider resource consumption** - Each Claude instance consumes API credits and system resources

---

## 2. Recommended Workflows

### Basic Worktree Workflow

```bash
# 1. Create worktree with new branch
git worktree add ../project-feature-auth -b feature-auth

# 2. Navigate to worktree
cd ../project-feature-auth

# 3. Install dependencies (worktrees don't share node_modules)
npm install  # or bun install, pnpm install

# 4. Copy environment files
cp ../project/.env .env.local

# 5. Start Claude Code
claude

# 6. When done, merge and cleanup
git checkout main && git merge feature-auth
git worktree remove ../project-feature-auth
```

### Worktree Directory Organization

**Recommended structure:**
```
~/Projects/
├── my-project/                    # Main repository
├── my-project-worktrees/          # Worktree container
│   ├── feature-auth/
│   ├── feature-dashboard/
│   └── bugfix-login/
```

This keeps worktrees as siblings to the main project, avoiding nested repository confusion.

### Parallel Development Pattern

1. **Create multiple worktrees** for independent tasks
2. **Open separate terminals** for each worktree
3. **Run Claude Code** in each terminal with descriptive session names
4. **Use Plan Mode** when leaving Claude running to prevent unauthorized changes
5. **Review and merge** each branch independently

---

## 3. Built-in Claude Code Worktree Skills

Claude Code includes four worktree-related skills at `~/.claude/commands/worktree/`:

### `/worktree:init <feature-name>`

Creates and initializes a new worktree for parallel development.

**What it does:**
1. Sanitizes feature name (lowercase, hyphens)
2. Creates worktree from `trunk` in `../<project>-worktrees/<feature>`
3. Copies `.env.local` and `.env` files
4. Installs dependencies (auto-detects bun/pnpm/yarn/npm)
5. Reports success with worktree location

**Example:**
```
/worktree:init feature-auth-refactor
```

### `/worktree:parallel <spec-file>`

Executes tasks from a specification file in parallel worktrees using subagents.

**Usage:**
```
/worktree:parallel docs/spec/my-feature-spec.md
/worktree:parallel docs/spec/my-feature-spec.md --phase 1
/worktree:parallel docs/spec/my-feature-spec.md --task 2.1
```

**What it does:**
1. Parses specification file for phases/tasks
2. Presents interactive phase selection (or uses flags)
3. Creates worktrees for selected phases
4. Spawns Claude Code subagents in parallel (one per worktree)
5. Each agent implements its assigned work independently

**Key features:**
- True parallel execution with multiple subagents
- Result documents created in each worktree
- Specification file remains unmodified (single source of truth)
- Agents implement but don't commit - user reviews and commits manually

### `/worktree:merge [worktree-name]`

Merges a worktree's work into trunk, updates specs, and cleans up.

**Options:**
- `--merge` - Preserve all commits with merge commit
- `--squash` - Squash without rebase first
- `--no-update` - Skip specification file update
- `--no-cleanup` - Keep worktree after merge

**What it does:**
1. Validates uncommitted changes and branch status
2. Updates trunk in main repository
3. Executes merge (default: rebase + squash for clean history)
4. Updates specification checkboxes from RESULTS.md
5. Calls `/worktree:cleanup` to remove worktree

**Multi-worktree support:**
- Run without arguments for interactive multi-select
- Merges worktrees sequentially (required for conflict-free operation)
- Reports batch completion summary

### `/worktree:cleanup [worktree-name] [--force]`

Removes a worktree after the branch has been merged.

**Safety checks (unless --force):**
- Verifies branch is merged into trunk
- Checks for uncommitted changes
- Confirms before deletion

**What gets deleted:**
- Worktree directory
- Git worktree metadata
- Local branch
- Remote branch (optional, user prompted)

---

## 4. Session Management Across Worktrees

### The `/resume` Picker

Claude Code's session picker is worktree-aware:

- **Sessions from same repo are grouped** - including worktrees
- **Descriptive session names help** - use meaningful names when starting sessions
- **Git branch is shown** - helps identify which worktree a session belongs to

**Commands:**
- `claude --resume` - Opens session picker
- `claude -c` or `claude --continue` - Resume most recent conversation
- `claude -r "session-id"` - Resume specific session
- `/resume` - Switch sessions from within Claude

### Session Naming Best Practices

When starting work in a worktree, give the session a descriptive name:
```
claude --name "auth-refactor"
```

This makes it easy to find later with `/resume`.

### Context Preservation

- **Sessions store locally** - Full message history retained (~30 days by default)
- **Tool state preserved** - Previous tool usage and results maintained
- **Worktree isolation** - Each session only knows about its worktree's files

### Third-party Session Management

**CCManager** ([github.com/kbwo/ccmanager](https://github.com/kbwo/ccmanager)):
- Copies Claude Code session data when creating worktrees
- Maintains context across different branches
- Manages multiple AI coding assistant sessions

---

## 5. Practical Considerations

### Resource Consumption

Running multiple Claude Code instances:
- **API costs** - Each session consumes tokens independently
- **System resources** - Multiple terminal processes, increased memory
- **Context limits** - Each session has its own context window

**Recommendation:** Limit to 5-7 concurrent sessions for practical management.

### Dependency Installation

Each worktree requires independent setup:
```bash
# JavaScript/TypeScript
npm install  # or bun/pnpm/yarn

# Python
pip install -r requirements.txt  # or poetry install

# .NET
dotnet restore
```

Worktrees do not share `node_modules`, `vendor`, or virtual environments.

### Test Execution

Tests can run in parallel across worktrees if:
- No hardcoded ports (use dynamic port allocation)
- No shared resources (separate test databases)
- No global state conflicts

### Environment Files

Always copy environment files to new worktrees:
```bash
cp ../<main-repo>/.env* .
```

The `/worktree:init` skill handles this automatically.

### Commit Hygiene

When working with AI assistance:
- Commit frequently with meaningful messages
- Use conventional commits (feat:, fix:, refactor:)
- Review changes before merging to main

---

## 6. Alternative Approaches

### GitButler Lifecycle Hooks

GitButler offers an alternative that avoids worktree bootstrapping overhead:

- Uses Claude Code lifecycle hooks to detect file changes
- Automatically creates separate branches per session
- All sessions work in the same directory
- Smart file assignment to appropriate branches

**Advantage:** No `npm install` per worktree, no merge conflicts with yourself.

### Claude Squad

Terminal app for managing multiple AI agents ([github.com/smtg-ai/claude-squad](https://github.com/smtg-ai/claude-squad)):

- Manages Claude Code, Aider, Codex, and other agents
- Each task gets isolated git workspace
- Single interface for task orchestration
- Uses tmux for session management

**Features:**
- Create sessions with prompts
- Navigate between active sessions
- Pause/resume work
- Review and approve changes
- Direct GitHub push

### Pending Feature Request

GitHub issue [#4963](https://github.com/anthropics/claude-code/issues/4963) proposes native integration:

**Proposed commands:**
- `/fork "<prompt>"` - Create worktree and start headless agent
- `/tasks list/view/kill` - Manage ongoing work
- `/tasks merge <ID>` - Finalize and integrate

This would reduce cognitive load by abstracting worktree management into simple commands.

---

## 7. Workflow Recommendations

### For Independent Features

1. Create worktree per feature
2. Use `/worktree:init <feature-name>`
3. Start Claude Code with descriptive session name
4. Complete work, run tests
5. Create PR, merge
6. Use `/worktree:cleanup <feature-name>`

### For Specification-Driven Development

1. Write specification document with phases/tasks
2. Run `/worktree:parallel <spec-file>`
3. Select phases to work on
4. Subagents work in parallel
5. Review result documents
6. Use `/worktree:merge` to integrate
7. Update specification with completed tasks

### For Exploration/Comparison

1. Create multiple worktrees for different approaches
2. Run competing implementations in parallel
3. Compare results
4. Keep best approach, delete others
5. Cleanup with `/worktree:cleanup --force`

### For Team Collaboration

1. Use consistent naming conventions (`feature/`, `bugfix/`, `ai/`)
2. Document worktree purposes in commit messages
3. Clean up promptly after merging
4. Use `git worktree prune` regularly

---

## 8. Quick Reference

### Commands

| Task | Command |
|------|---------|
| Create worktree | `git worktree add ../path -b branch` |
| List worktrees | `git worktree list` |
| Remove worktree | `git worktree remove ../path` |
| Prune stale refs | `git worktree prune` |
| Init with skill | `/worktree:init <name>` |
| Parallel work | `/worktree:parallel <spec>` |
| Merge worktree | `/worktree:merge <name>` |
| Cleanup | `/worktree:cleanup <name>` |

### Session Management

| Task | Command |
|------|---------|
| Resume picker | `claude --resume` |
| Continue last | `claude -c` |
| Resume by ID | `claude -r "session-id"` |
| Name session | `claude --name "feature-x"` |
| In-session switch | `/resume` |

---

## Sources

- [Claude Code Common Workflows Documentation](https://code.claude.com/docs/en/common-workflows)
- [Anthropic Engineering: Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- [incident.io: Shipping Faster with Claude Code and Git Worktrees](https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees)
- [DEV.to: Git Worktree + Claude Code: My Secret to 10x Developer Productivity](https://dev.to/kevinz103/git-worktree-claude-code-my-secret-to-10x-developer-productivity-520b)
- [DEV.to: Supercharge Your AI Coding Workflow](https://dev.to/bhaidar/supercharge-your-ai-coding-workflow-a-complete-guide-to-git-worktrees-with-claude-code-60m)
- [GitButler: Parallel Claude Code Without Worktrees](https://blog.gitbutler.com/parallel-claude-code)
- [GitHub: Claude Squad](https://github.com/smtg-ai/claude-squad)
- [GitHub: Feature Request #4963 - Integrated Parallel Task Management](https://github.com/anthropics/claude-code/issues/4963)
- [Geeky Gadgets: Git Worktrees with Claude Code](https://www.geeky-gadgets.com/how-to-use-git-worktrees-with-claude-code-for-seamless-multitasking/)
- [Medium: Mastering Git Worktrees with Claude Code](https://medium.com/@dtunai/mastering-git-worktrees-with-claude-code-for-parallel-development-workflow-41dc91e645fe)

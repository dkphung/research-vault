---
tags: [git-workflows]
date: 2024-12-22
status: complete
---

# Git Worktrees for Parallel Development - Research

**Date Created**: 2025-02-11
**Status**: Research Complete

## Executive Summary

Git worktrees enable **true parallel development** by allowing multiple isolated working directories sharing a single `.git` repository. This approach has gained significant momentum in 2024, particularly with AI-assisted development tools like Claude Code, enabling developers to run multiple concurrent development streams without branch-switching friction.

**Key Finding**: The research reveals three maturity levels for implementing parallel workflows:
1. **Basic**: Manual worktree management for isolated parallel tasks
2. **Intermediate**: Custom commands and automation for orchestrated workflows
3. **Advanced**: Specification-driven orchestration with dependency management and automated task execution

---

## 1. Git Worktree Best Practices

### 1.1 Organization & Directory Structure

**Recommended Pattern** (from incident.io and community best practices):
```
~/Projects/
├── client-onboarding-poc/          # Main repository
│   ├── .git/                       # Shared git metadata
│   └── ...
└── client-onboarding-poc-worktrees/  # Worktrees directory
    ├── feature-saml-import-1/
    ├── feature-saml-import-2/
    ├── refactor-sso-config/
    └── bugfix-auth-flow/
```

**Naming Conventions**:
- Use descriptive names matching branch identifiers
- Include username prefixes for multi-developer teams: `username/feature-name`
- Group by feature/task type: `feature-`, `bugfix-`, `refactor-`
- Add numeric suffixes for parallel variations: `-1`, `-2`, `-3`

### 1.2 Core Workflow Commands

**Creation**:
```bash
# Create new worktree with branch
git worktree add ../client-onboarding-poc-worktrees/feature-name -b feature-name

# Create from existing branch
git worktree add ../client-onboarding-poc-worktrees/feature-name feature-name
```

**Management**:
```bash
# List all worktrees
git worktree list

# Remove worktree
git worktree remove ../client-onboarding-poc-worktrees/feature-name

# Clean stale references
git worktree prune
```

**Navigation** (recommended shell function):
```bash
# Add to .bashrc/.zshrc
function w() {
  local worktree=$1
  cd ~/Projects/client-onboarding-poc-worktrees/${worktree}
}
```

### 1.3 Cleanup & Maintenance

**Critical Practices**:
1. **Regular pruning**: Run `git worktree prune` weekly to remove stale metadata
2. **Immediate cleanup**: Remove worktrees as soon as work is merged
3. **Gitignore setup**: Add worktree parent directories to `.gitignore`
4. **Audit routine**: Periodically run `git worktree list` to identify forgotten worktrees

**Automation Script**:
```bash
#!/bin/bash
# Clean up merged worktrees
git worktree list --porcelain | grep "branch refs/heads/" | while read branch; do
  branch_name=$(echo $branch | sed 's/branch refs\/heads\///')
  if git branch -r --merged origin/main | grep -q "origin/${branch_name}"; then
    echo "Removing merged worktree: $branch_name"
    git worktree remove $branch_name
  fi
done
git worktree prune
```

### 1.4 Dependency Management Between Worktrees

**Key Principle**: Worktrees are **isolated by design**. Dependencies require explicit coordination.

**Strategies**:

1. **Shared Dependencies** (node_modules, etc.):
   - **DO NOT share** dependency directories between worktrees
   - Each worktree should run `pnpm install` independently
   - Use pnpm's workspace linking if in monorepo context
   - Storage cost is offset by isolation benefits

2. **Environment Files**:
   - Copy `.env.local` to new worktrees during initialization
   - Use automation to sync non-secret environment variables
   - Keep secrets in encrypted manager, not in worktrees

3. **Database/External Services**:
   - **Option A**: Shared database (faster, risk of conflicts)
   - **Option B**: Per-worktree database (isolated, requires setup)
   - **Recommended**: Use Docker Compose with worktree-specific ports

4. **Specification Files** (Critical for parallel workflows):
   - **Single source of truth**: Spec file lives in main repo only
   - **Read-only access**: Worktrees reference spec from main repo directory
   - **Per-worktree progress**: Use `RESULTS.md` to track task-specific progress
   - **Checkpoint updates**: Update spec checkboxes only after merge to main
   - **Avoid conflicts**: Never modify spec file simultaneously in multiple worktrees

### 1.4.1 Managing Specification Files Across Worktrees

**Problem**: When running parallel tasks from a single specification, how do you track progress without creating merge conflicts?

**Recommended Pattern**: **Spec in Main + RESULTS.md per Worktree**

```
~/Projects/
├── client-onboarding-poc/                    # Main repository
│   └── docs/spec/07-saml-metadata-import-spec.md  # ← Canonical spec (single source of truth)
│
└── client-onboarding-poc-worktrees/
    ├── phase-2-json-converter/
    │   └── RESULTS.md                        # ← Task-specific progress
    ├── phase-2-fetcher-service/
    │   └── RESULTS.md                        # ← Task-specific progress
    └── phase-2-storage-service/
        └── RESULTS.md                        # ← Task-specific progress
```

**Workflow**:

1. **Starting Work in Worktree**
   ```bash
   # From worktree
   cd ~/Projects/client-onboarding-poc-worktrees/phase-2-json-converter

   # Reference spec from main repo (relative path)
   # Main repo is at: ../client-onboarding-poc/
   # Spec is at: ../client-onboarding-poc/docs/spec/07-saml-metadata-import-spec.md

   # Tell Claude: "Implement Phase 2, Task 2.1 from spec:
   # ../client-onboarding-poc/docs/spec/07-saml-metadata-import-spec.md"
   ```

2. **During Development**
   - **Read** from main repo spec: `../client-onboarding-poc/docs/spec/...`
   - **Write** progress to worktree RESULTS.md
   - **Don't modify** spec file in worktree branch
   - Track completion, tests, coverage in RESULTS.md

3. **After Task Completion**
   - PR includes code changes only (not spec updates)
   - RESULTS.md shows what was accomplished
   - After PR merges to main, update spec checkboxes in main repo

4. **Updating Spec Checkboxes**
   ```bash
   # After worktree PR merges
   cd ~/Projects/client-onboarding-poc  # Main repo

   # Update spec checkboxes for completed tasks
   # Edit: docs/spec/07-saml-metadata-import-spec.md
   #   - [ ] Task 2.1: JSON converter
   # becomes:
   #   - ✅ Task 2.1: JSON converter

   git add docs/spec/07-saml-metadata-import-spec.md
   git commit -m "docs: mark Phase 2 Task 2.1 complete"
   git push
   ```

**Why This Works**:
- ✅ **No merge conflicts**: Spec never modified in worktree branches
- ✅ **Single source of truth**: Main repo has canonical spec
- ✅ **Clear progress tracking**: Each worktree has its own RESULTS.md
- ✅ **Accurate history**: Spec updates in main reflect actual completion
- ✅ **Claude can read spec**: Relative path works from any worktree

**Alternative Patterns** (Not Recommended):

❌ **Copy Spec to Each Worktree**
- Leads to merge conflicts when multiple worktrees update checkboxes
- Hard to track overall progress
- Risk of inconsistent versions

❌ **Update Spec in Worktree**
- Merge conflicts when multiple PRs try to update same spec
- Race conditions on checkbox updates
- Breaks single source of truth principle

**Referencing Spec from Worktree**:

```bash
# Main repo location
~/Projects/client-onboarding-poc/

# Worktree location
~/Projects/client-onboarding-poc-worktrees/phase-2-json-converter/

# Relative path to spec from worktree
../client-onboarding-poc/docs/spec/07-saml-metadata-import-spec.md

# Or use absolute path
~/Projects/client-onboarding-poc/docs/spec/07-saml-metadata-import-spec.md
```

**Claude Code Command Pattern**:

```bash
# From worktree directory
cd ~/Projects/client-onboarding-poc-worktrees/phase-2-json-converter

# Tell Claude to execute task
/worktree-execute ../client-onboarding-poc/docs/spec/07-saml-metadata-import-spec.md phase-2-1

# Claude reads spec from main repo
# Claude implements in current worktree
# Claude writes progress to ./RESULTS.md
# Spec file remains unchanged
```

**RESULTS.md Template** (per worktree):

```markdown
# Task Results: Phase 2, Task 2.1 - SimpleSAML JSON Converter

**Worktree**: phase-2-json-converter
**Branch**: phase-2-json-converter
**Specification**: ../client-onboarding-poc/docs/spec/07-saml-metadata-import-spec.md
**Task**: Phase 2, Task 2.1
**Started**: 2025-02-11 10:00
**Completed**: 2025-02-11 10:45

## Implementation Summary
Implemented `metadataConverterToJSON()` function in simplesaml.service.ts.
Reused existing XML parsing logic while adding JSON output format.

## Test Results
- Tests written: 9
- Tests passing: 9/9
- Coverage: 95%

## Quality Validation
- [x] TypeScript: No errors
- [x] Linting: No warnings
- [x] Tests: All passing
- [x] Coverage: ≥60%

## Files Modified
- src/server/services/simplesaml.service.ts (added metadataConverterToJSON)
- src/server/services/simplesaml.service.spec.ts (added 9 new tests)

## Dependencies
None

## Notes & Learnings
JSON format matches SimpleSAMLphp PDO storage requirements.
All existing PHP converter tests (24) still passing.

## Merge Readiness
- [x] All quality gates passed
- [x] Spec checkboxes will be updated after merge
- [x] PR created: https://github.com/.../pull/123
- [x] Ready for review
```

**Multi-Worktree Progress Tracking**:

```bash
# Check progress across all worktrees
cd ~/Projects/client-onboarding-poc-worktrees

# View all RESULTS.md files
for dir in */; do
  echo "=== $dir ==="
  grep "Completed:" "$dir/RESULTS.md" 2>/dev/null || echo "Not completed"
done

# Or use a script (scripts/worktree/status.sh)
# Shows aggregated progress from all worktrees
```

**Best Practices**:

1. **Always reference spec from main repo** - Use relative or absolute paths
2. **Never modify spec in worktree branch** - Avoids merge conflicts
3. **Update spec only after merge** - In main repo, after PR merges
4. **Use RESULTS.md for progress** - Worktree-specific tracking
5. **One task per worktree** - Clear scope, easy to track
6. **Commit RESULTS.md with code** - PR shows what was accomplished
7. **Clean up after merge** - Remove worktree, update spec in main

**Handling Spec Updates During Development**:

If spec file changes in main while you're working:

```bash
# From worktree
cd ~/Projects/client-onboarding-poc-worktrees/phase-2-json-converter

# Your branch may be behind main
git fetch origin

# Rebase to get latest (including spec updates)
git rebase origin/main

# Spec updates from main are now in your worktree
# Continue working with latest spec
```

**Coordination Between Parallel Worktrees**:

When multiple worktrees work on same spec:

1. **Phase-based isolation**: Each worktree works on different phase
2. **Task-based isolation**: Each worktree works on different task
3. **No spec conflicts**: None modify spec file
4. **Merge order**: Sequential merging updates spec progressively
5. **Communication**: Use PR descriptions to coordinate dependencies

### 1.5 Merge Strategies & Conflict Resolution

**Preventive Strategies**:
1. **Rebase before PR**: Always rebase on latest main before creating PR
   ```bash
   git fetch origin
   git rebase origin/main
   git push --force-with-lease
   ```

2. **Feature isolation**: Keep parallel tasks in separate code areas
3. **Communication**: Use PR labels to indicate parallel work

**Conflict Resolution Workflow**:
```bash
# In worktree with conflict
git fetch origin
git rebase origin/main

# Resolve conflicts in editor
git add .
git rebase --continue

# Verify build and tests
pnpm run build
pnpm test

# Force push with safety
git push --force-with-lease
```

**Convergent Merge Strategy** (for parallel experiments):
1. Run N parallel implementations in separate worktrees
2. Compare results across worktrees
3. Select best implementation
4. Merge winner to main
5. Clean up remaining worktrees

---

## 2. Parallel Development Patterns

### 2.1 Industry Best Practices

**Pattern 1: Redundant Implementation**
- Run N agents implementing the same spec
- Compare quality, performance, and correctness
- Select best result
- Use case: Critical features where quality matters more than speed

**Pattern 2: Task Parallelization**
- Break feature into independent tasks
- Execute tasks in parallel worktrees
- Merge in dependency order
- Use case: Large features with clear task boundaries

**Pattern 3: Experimental Branches**
- Try multiple approaches to same problem
- Evaluate trade-offs in parallel
- Converge on best solution
- Use case: Architecture decisions, performance optimization

**Pattern 4: Hotfix + Feature**
- Urgent bugfix in one worktree
- Continue feature work in another worktree
- No context switching required
- Use case: Production emergencies during development

### 2.2 Isolation Strategies

**Code Isolation**: ✅ Native to worktrees
**Dependency Isolation**: Separate `node_modules` per worktree
**Database Isolation**: Docker Compose with worktree-specific ports
**Service Isolation**: Port-based separation (dev server on 3001, 3002, 3003)

**Example Docker Compose Override**:
```yaml
# docker-compose.worktree-1.yml
version: '3.8'
services:
  mysql:
    ports:
      - "3307:3306"  # Instead of default 3306
  app:
    environment:
      - DB_PORT=3307
      - PORT=3001
```

### 2.3 Coordination Patterns

**Loose Coupling** (recommended for most cases):
- Independent execution
- Manual merge orchestration
- Post-execution comparison
- Minimal inter-worktree communication

**Tight Coupling** (for dependent tasks):
- Explicit dependency chains
- Sequential execution with checkpoints
- Shared progress tracking
- Central orchestration

---

## 3. Claude Code Integration

### 3.1 Current Capabilities

**Native Support**:
- Each Claude Code session maintains context within its working directory
- Running `/init` in worktree establishes proper codebase orientation
- Plan Mode enables safe parallel execution without unexpected changes
- Commit and push capabilities per worktree

**Recommended Workflow**:
```
1. Create worktrees for parallel tasks
2. Open separate Claude Code sessions per worktree
3. Run /init in each session for context
4. Use Plan Mode to review before execution
5. Execute tasks independently
6. Review results and merge selectively
```

### 3.2 Automation Possibilities

**Custom Commands Pattern** (based on agent interviews example):

```markdown
<!-- .claude/commands/parallel-init.md -->
# Initialize Parallel Worktrees

Given a feature name and count:
1. Create N worktree directories under trees/
2. Create N branches: feature-name-1, feature-name-2, etc.
3. Copy .env.local to each worktree
4. Run pnpm install in each worktree
5. Create RESULTS.md template in each
6. Output worktree paths for Claude sessions

Arguments: $FEATURE_NAME $COUNT
```

```markdown
<!-- .claude/commands/parallel-execute.md -->
# Execute Specification in Parallel

Given a spec file path:
1. READ the specification
2. Follow TDD workflow
3. Implement all phases
4. Run tests and verify
5. Document results in RESULTS.md
6. Commit changes

Arguments: $SPEC_PATH
```

### 3.3 Challenges & Solutions

**Challenge 1**: Context Switching Between Worktrees
- **Solution**: Use separate terminal windows/IDE instances per worktree
- **Tool**: tmux/screen sessions or VS Code workspaces

**Challenge 2**: Claude Code can't orchestrate multiple sessions
- **Solution**: Manual coordination or future multi-agent capabilities
- **Workaround**: Use Task tool to spawn sub-agents (experimental)

**Challenge 3**: Progress Tracking Across Worktrees
- **Solution**: Standardized RESULTS.md file per worktree
- **Tool**: Central dashboard or git branch status checks

**Challenge 4**: Dependency Detection
- **Solution**: Manual specification of dependencies in spec files
- **Future**: Automated dependency graph parsing

### 3.4 Multi-Agent Execution (Advanced)

**Pattern from Research**:
```markdown
<!-- .claude/commands/parallel-agents.md -->
# Run Multiple Claude Agents in Parallel

Prerequisites:
- Worktrees already initialized
- Specification file created
- Dependencies installed

Execution:
RUN /parallel-init $FEATURE_NAME 3

For each worktree (1, 2, 3):
  SPAWN_AGENT with working_dir=trees/$FEATURE_NAME-{i}
  EXECUTE /parallel-execute specs/$FEATURE_NAME.md

WAIT for all agents to complete
COMPARE RESULTS.md files
RECOMMEND best implementation
```

---

## 4. Workflow Orchestration

### 4.1 Task Scheduling & Execution

**Dependency Graph Approach** (inspired by Nx/Turborepo):

```typescript
interface Task {
  id: string;
  phase: string;
  description: string;
  dependencies: string[];  // Task IDs that must complete first
  parallelizable: boolean; // Can run in parallel with siblings
  estimatedTime: number;   // Minutes
}

interface SpecificationTasks {
  phases: {
    name: string;
    tasks: Task[];
  }[];
}
```

**Execution Algorithm**:
```
1. Parse specification file
2. Extract phases and tasks
3. Build dependency graph (DAG)
4. Identify parallelizable tasks (no dependencies)
5. Schedule tasks:
   - Phase 1 tasks in parallel (if independent)
   - Wait for phase completion
   - Phase 2 tasks in parallel
   - Continue until all phases complete
6. Merge results in dependency order
```

**Example from SAML Import Spec**:
```
Phase 1: Database Setup (1 worktree, sequential)
  ├─ Task 1.1: Install dependencies
  ├─ Task 1.2: Add env variables
  └─ Task 1.3: Create schema

Phase 2: Services (3 worktrees, parallel)
  ├─ Worktree 1: SimpleSAML JSON converter
  ├─ Worktree 2: Metadata fetcher service
  └─ Worktree 3: Storage service

Phase 3: Integration (1 worktree, sequential)
  └─ Task 3.1: Server Action orchestrator
```

### 4.2 Progress Tracking & Reporting

**Checklist-Based Tracking** (current approach):
- Markdown checkboxes in spec files: `- [ ]` → `- ✅`
- Manual updates as tasks complete
- Simple, version-controlled, human-readable

**Structured Tracking** (advanced):
```markdown
## Progress Dashboard

| Worktree | Phase | Status | Tests | Coverage | Last Updated |
|----------|-------|--------|-------|----------|--------------|
| saml-1   | 2.1   | ✅ Done | 9/9   | 95%      | 2025-01-30   |
| saml-2   | 2.2   | ⏳ Running | 8/11 | 87%   | 2025-01-30   |
| saml-3   | 2.3   | 📝 Planned | 0/7 | -      | -            |
```

**Automation Options**:
```bash
# Update progress dashboard
#!/bin/bash
for worktree in trees/*; do
  cd $worktree
  status=$(git status --short)
  tests=$(pnpm test --reporter=json | jq '.numPassedTests')
  coverage=$(pnpm test:coverage --reporter=json | jq '.coverageMap.total.lines.pct')

  echo "| $(basename $worktree) | $phase | $status | $tests | $coverage% | $(date +%Y-%m-%d) |"
done
```

### 4.3 Integration Points for Review

**Pre-Merge Review Workflow**:
```
1. Task completion in worktree
2. Run quality checks:
   - pnpm tsc --noEmit
   - pnpm run lint
   - pnpm test
3. Create PR from worktree branch
4. Automated CI/CD runs
5. Code review (human or AI)
6. Merge to main
7. Trigger dependent tasks (if any)
8. Clean up worktree
```

**Cross-Worktree Review**:
- Use diff tools to compare implementations
- Run performance benchmarks across worktrees
- Evaluate test coverage differences
- Select best approach based on metrics

---

## 5. Specification-Driven Development

### 5.1 Parsing Specification Files

**Current Spec Structure** (from this project):
```markdown
## Implementation Plan

### Phase 1: Database Setup
- [ ] Task 1.1: Install dependencies
- [ ] Task 1.2: Add environment variables
- ✅ Task 1.3: Create schema (COMPLETED)

### Phase 2: Service Development
- [ ] Task 2.1: SimpleSAML JSON converter
  - [ ] Subtask 2.1.1: Add conversion function
  - [ ] Subtask 2.1.2: Write tests
- [ ] Task 2.2: Metadata fetcher
```

**Parsing Strategy**:
```typescript
interface ParsedSpec {
  context: string;
  requirements: {
    functional: string[];
    nonFunctional: string[];
  };
  phases: {
    id: string;
    name: string;
    tasks: {
      id: string;
      description: string;
      status: 'pending' | 'completed';
      subtasks: string[];
    }[];
  }[];
  successCriteria: string[];
}

function parseSpecificationFile(markdown: string): ParsedSpec {
  // Parse markdown structure
  // Extract phases (## headings)
  // Extract tasks (- [ ] checkboxes)
  // Detect dependencies (mentioned task IDs)
  // Identify parallelizable groups (independent tasks in same phase)
}
```

### 5.2 Dependency Detection

**Automatic Detection Strategies**:

1. **Explicit Dependencies** (recommended):
   ```markdown
   - [ ] Task 2.1: Create service
   - [ ] Task 2.2: Write tests (depends on: 2.1)
   - [ ] Task 2.3: Integration test (depends on: 2.1, 2.2)
   ```

2. **Implicit Dependencies** (AI analysis):
   - Scan task descriptions for references to other tasks
   - Identify file/module dependencies
   - Build dependency graph from code structure

3. **Phase-Based Dependencies**:
   - Default: All tasks in Phase N depend on Phase N-1 completion
   - Override with explicit parallel markers:
     ```markdown
     ### Phase 2: Services (PARALLEL)
     - [ ] Task 2.1: Service A (independent)
     - [ ] Task 2.2: Service B (independent)
     - [ ] Task 2.3: Service C (independent)
     ```

### 5.3 Prioritization & Scheduling

**Priority Factors**:
1. **Dependencies**: Must complete before dependents can start
2. **Critical path**: Tasks that block the most other tasks
3. **Estimated effort**: Balance load across parallel worktrees
4. **Risk level**: High-risk tasks first for early feedback
5. **Resource availability**: CPU, memory, external services

**Scheduling Algorithm**:
```
1. Topological sort of dependency graph
2. Identify "ready" tasks (all dependencies met)
3. Assign to available worktrees (load balancing)
4. Execute batch in parallel
5. Mark completed tasks
6. Repeat until all tasks done
```

**Example Schedule** (SAML Import Spec):
```
Batch 1 (1 worktree, sequential):
  - Phase 1: Database Setup (10 min)

Batch 2 (3 worktrees, parallel):
  - Worktree 1: SimpleSAML JSON (20 min)
  - Worktree 2: Metadata Fetcher (15 min)
  - Worktree 3: Storage Service (18 min)
  Wait for slowest: 20 min

Batch 3 (1 worktree, sequential):
  - Phase 3: Server Action (15 min)

Total: 10 + 20 + 15 = 45 min
Sequential: 10 + 20 + 15 + 18 + 15 = 78 min
Speedup: 42% faster
```

### 5.4 Success Criteria Validation

**Validation Points**:
1. **Per-Task**: Tests pass, linting clean, types valid
2. **Per-Phase**: Integration tests pass, phase objectives met
3. **Per-Specification**: All success criteria checked

**Automated Validation**:
```bash
# validate-task.sh
#!/bin/bash
worktree=$1

cd $worktree

echo "Validating $worktree..."

# Type checking
pnpm tsc --noEmit || exit 1

# Linting
pnpm run lint || exit 1

# Tests
pnpm test || exit 1

# Coverage threshold
pnpm test:coverage --coverage-threshold=60 || exit 1

echo "✅ Validation passed for $worktree"
```

---

## 6. Implementation Tiers

### Tier 1: Basic Manual Workflow

**Description**: Use git worktree commands directly, no automation

**Setup Time**: 1-2 hours
**Per-Use Time**: ~10 minutes
**Speedup**: 2-3x

**Pros**:
- Zero setup time
- Maximum flexibility
- Easy to understand

**Cons**:
- Repetitive manual steps
- Error-prone (forget to cleanup, etc.)
- No progress tracking

**Best For**: Occasional parallel work, 1-2 concurrent tasks

### Tier 2: Script-Based Automation

**Description**: Helper scripts + `.claude/commands/` integration

**Setup Time**: 4-8 hours
**Per-Use Time**: ~2 minutes
**Speedup**: 5-10x

**Pros**:
- Moderate setup effort
- Claude Code native integration
- Reusable across projects
- Version controlled

**Cons**:
- Requires script development
- Manual coordination between worktrees
- Limited cross-worktree awareness

**Best For**: Regular parallel work, 3-5 concurrent tasks

**Components**:
1. Helper scripts in `scripts/worktree/`
2. Custom commands in `.claude/commands/worktree/`
3. Spec template enhancements
4. Documentation and examples

### Tier 3: Full Orchestration Framework

**Description**: Spec parser + task scheduler + multi-agent execution

**Setup Time**: 40-80 hours
**Per-Use Time**: ~30 seconds
**Speedup**: 10-20x

**Pros**:
- Fully automated workflow
- Optimal parallelization
- Minimal manual intervention
- Dependency-aware scheduling

**Cons**:
- Significant development effort
- Complexity overhead
- Maintenance burden
- Requires sophisticated tooling

**Best For**: Large-scale projects, 10+ parallel tasks regularly

**Components**:
1. Spec parser (markdown → dependency graph)
2. Task scheduler (topological sort, load balancing)
3. Multi-agent orchestrator
4. Progress dashboard
5. Automated validation and merging

---

## 7. Challenges & Solutions

### 7.1 Technical Challenges

| Challenge | Impact | Solution | Priority |
|-----------|--------|----------|----------|
| Storage overhead (multiple node_modules) | Disk space | Use pnpm with shared store, clean up regularly | Medium |
| Port conflicts (dev servers) | Can't run parallel | Use PORT env variable, automate port assignment | High |
| Database conflicts | Data corruption | Shared database with read-focus OR Docker per worktree | High |
| Git conflicts during merge | Delays | Frequent rebasing, feature isolation | Medium |
| Context switching overhead | Mental load | Separate terminal/IDE instances, clear naming | Low |

### 7.2 Process Challenges

| Challenge | Impact | Solution | Priority |
|-----------|--------|----------|----------|
| Dependency tracking | Wrong execution order | Explicit dependency markers in specs | High |
| Progress visibility | Lost track of status | Standardized RESULTS.md, git branch checks | Medium |
| Quality consistency | Variable test coverage | Automated validation gates per worktree | High |
| Cleanup discipline | Stale worktrees accumulate | Automated cleanup scripts, weekly audits | Low |
| Learning curve | Slow adoption | Documentation, pair programming, examples | Medium |

### 7.3 Claude Code Limitations

| Limitation | Impact | Workaround | Future |
|-----------|--------|------------|--------|
| Single session per worktree | Manual coordination | Multiple terminal windows, custom commands | Multi-agent support |
| No cross-worktree awareness | Can't compare results | Manual review, diff tools | Cross-session API |
| Limited orchestration | Can't manage dependencies | Explicit sequencing, scripting | Workflow engine |

---

## 8. Recommendations

### 8.1 Immediate Actions

1. **Document workflow** (this document) ✅
2. **Start with Tier 2** for practical parallel development
3. **Design Tier 3** for future roadmap understanding
4. **Test workflow** on next feature specification

### 8.2 Decision Matrix

| Factor | Manual | Scripts | Commands | Framework |
|--------|--------|---------|----------|-----------|
| Setup Time | 0h | 8h | 16h | 80h |
| Per-Use Time | 10min | 5min | 2min | 30sec |
| Learning Curve | Low | Medium | Low | High |
| Flexibility | High | High | Medium | Low |
| Automation | None | Medium | High | Complete |
| **Recommended** | Prototype | **Tier 2** | **Tier 2** | **Tier 3** |

### 8.3 Expected Outcomes

**With Tier 2 Implementation**:
- SAML Import spec: ~78 hours sequential → ~45 hours parallel (42% savings)
- Reduced context switching friction
- Cleaner git history with focused branches
- Better code review process (smaller, focused PRs)
- Ability to run experiments in parallel

**With Tier 3 Implementation**:
- Near-optimal parallelization
- Automated dependency management
- Real-time progress visibility
- Minimal manual intervention
- Scales to 10+ parallel tasks

---

## Appendix: Resources

### Research Sources
- [incident.io: Claude Code + Git Worktrees](https://incident.io/blog/shipping-faster-with-claude-code-and-git-worktrees)
- [Agent Interviews: Parallel AI Coding](https://docs.agentinterviews.com/blog/parallel-ai-coding-with-gitworktrees/)
- [GitHub Gist: Worktree Best Practices](https://gist.github.com/ChristopherA/4643b2f5e024578606b9cd5d2e6815cc)
- [GitHub: claude-code-spec-workflow](https://github.com/Pimzino/claude-code-spec-workflow)
- Community articles on Nx, Turborepo, GitLab CI/CD patterns

### Example Specifications in This Project
- `docs/spec/07-saml-metadata-import-spec.md` - Completed spec with parallel opportunities
- `docs/spec/02-incommon-federation-integration-spec.md` - InCommon integration
- `docs/spec/06-nextjs-16-migration-spec.md` - Next.js migration

### Tools & Technologies
- Git Worktree (built-in to Git 2.5+)
- Claude Code (Anthropic)
- pnpm (package manager with workspace support)
- Docker Compose (service isolation)
- Nx/Turborepo (monorepo orchestration patterns)

---

**Document Status**: Research Complete
**Next Action**: Create implementation specification for parallel worktree workflow
**File Location**: `docs/research/git-worktree-parallel-development.md`

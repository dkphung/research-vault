---
title: TDD Workflows with Claude Code and Opus 4.5
date: 2026-01-07
tags:
  - claude-code
  - tdd
  - testing
  - opus-4.5
  - bun
  - react
  - next.js
  - workflow
  - automation
---

# TDD Workflows with Claude Code and Opus 4.5

## Executive Summary

Getting Claude Code to reliably write tests first without manual skill invocation remains a significant challenge. Despite Anthropic declaring TDD as a "favorite workflow," Claude Code's default behavior is **implementation-first**. The research reveals that **CLAUDE.md instructions alone are insufficient**--enforcement requires architectural interventions through hooks, skills with explicit phase gates, and ideally multi-agent (subagent) workflows that isolate context between test-writing and implementation phases.

Key findings:
1. **Pure instruction-based TDD fails at ~20% activation rate** due to Claude's goal-focused behavior
2. **Hook-based forced evaluation achieves ~84% activation rate**
3. **Subagent isolation is the only way to achieve true test-first development** from LLMs due to context pollution
4. **TDD Guard and Superpowers** are the most mature tools for automated enforcement
5. **Opus 4.5 improves compliance** but doesn't eliminate the need for enforcement infrastructure

---

## Main Findings

### 1. CLAUDE.md Instruction Patterns

#### What Works

**Explicit Phase Gates with Blocking Language:**
```markdown
## Testing Philosophy

**MANDATORY: Always write tests first. No exceptions.**

### Red-Green-Refactor Cycle

Every code change follows this workflow:
1. **Red**: Write a failing test that describes the expected behavior
2. **Green**: Write minimal code to make the test pass
3. **Refactor**: Clean up while keeping tests green

Do NOT proceed to implementation until test failure is confirmed.
```

The "Do NOT proceed" phrasing creates explicit blocking behavior rather than suggestions.

**FIRST Principles Template:**
```markdown
## TDD FIRST Principles
- Fast: Unit tests run in milliseconds
- Independent: Tests don't depend on each other
- Repeatable: Same results in any environment
- Self-Validating: Tests have boolean outcome
- Timely: Tests written BEFORE production code
```

**Anti-Pattern Prevention:**
```markdown
## Never Do These:
- Mark todos complete with failing tests
- Skip lint/typecheck before completion
- Fix bugs without adding tests first
- Write tests after implementation
```

#### What Doesn't Work

**Passive Suggestions:**
```markdown
# Don't use this pattern
- Consider writing tests first when appropriate
- TDD is recommended for complex features
- Tests should generally precede implementation
```

Claude treats these as optional guidance and ignores them when goal-focused.

**Excessive Instructions:**
According to research, Claude Code can follow approximately 150-200 instructions with reasonable consistency, but performance degrades as count increases. Claude Code's system prompt already contains ~50 instructions. Recommended CLAUDE.md length: **less than 300 lines**, ideally under 60 lines with progressive disclosure to separate files.

#### Evidence-Based CLAUDE.md Structure

From HumanLayer research, effective CLAUDE.md files follow WHY/WHAT/HOW structure:

```markdown
# Project Name

## Why (Context)
Brief explanation of project purpose.

## What (Architecture)
- src/features/ - Domain logic
- src/graphql/ - API layer
- tests/ - Colocated with source as *.spec.ts

## How (Workflows)

### Testing (MANDATORY)
bun test                    # Run all tests
bun test src/features/auth  # Run specific module

### TDD Workflow (ENFORCED)
1. Write failing test first
2. Run test, confirm failure
3. Write minimal passing code
4. Refactor if needed
5. Never skip steps 1-2

### Commands
bun run build
bun run lint
```

---

### 2. Prompt Engineering Patterns

#### Explicit TDD Requests

Anthropic's official guidance recommends being **explicit about TDD**:

> "Be explicit that you are doing TDD, which helps Claude avoid creating mock implementations or stubbing out imaginary code prematurely."

**Effective Prompt Pattern:**
```
We are doing TDD. First, write a failing test for [feature].
Do NOT write any implementation code yet.
The test must fail when run.
```

**Multi-Step TDD Prompting:**
1. "Write tests for [feature] based on these input/output pairs: [examples]. Do not implement yet."
2. "Run the tests and confirm they fail."
3. "Now write minimal code to make the tests pass. Do not modify the tests."
4. "Run the tests again and verify they pass."

#### Trigger Phrases That Improve Compliance

Based on Scott Spence's testing (200+ prompts), these patterns improve activation:

```
INSTRUCTION: Use the TDD workflow for this request.
MANDATORY: Write failing test before any implementation.
```

Direct commands receive better compliance than suggestions. The framing matters: "INSTRUCTION:" prefix signals required behavior.

---

### 3. Workflow Patterns

#### Hook-Based Enforcement (84% Success Rate)

The most reliable pattern uses `UserPromptSubmit` hooks to inject explicit skill evaluation:

**.claude/settings.json:**
```json
{
  "hooks": {
    "UserPromptSubmit": [{
      "hooks": [{
        "type": "command",
        "command": "npx tsx .claude/hooks/tdd-eval.ts",
        "timeout": 5
      }]
    }]
  }
}
```

**Forced Eval Hook Pattern:**
The hook requires Claude to explicitly evaluate whether TDD applies before responding, creating a commitment mechanism. Once Claude writes "YES - TDD applies" in its evaluation, it's committed to following through.

#### Subagent Architecture (Context Isolation)

The fundamental limitation of single-context TDD is that the LLM "knows" what implementation it will write, contaminating test design.

**Three-Agent Solution:**

1. **Test Writer Agent** (`.claude/agents/tdd-test-writer.md`)
   - Tools: Read, Glob, Grep, Write, Edit, Bash
   - Constraint: "Test MUST fail when run - verify before returning"
   - Cannot see implementation plans

2. **Implementer Agent** (`.claude/agents/tdd-implementer.md`)
   - Receives only the failing test, not test-writing context
   - Constraint: "Fix implementation, not tests"
   - Principle: Write only what the test requires

3. **Refactorer Agent** (`.claude/agents/tdd-refactorer.md`)
   - Evaluation checklist for code quality
   - Can output "no refactoring needed" with reasoning

This achieves genuine test-first development because the test writer has no implementation context to leak into test design.

#### Plan Mode Integration

Use Plan Mode before TDD execution:

```
/plan
Design the feature structure and identify test cases for [feature].
Break down into testable units.
```

Then execute TDD per test case. Plan mode forces structured thinking without executing code, making it ideal for test case identification.

---

### 4. Tools and Frameworks

#### TDD Guard

**Purpose:** Automated TDD enforcement via hooks that block non-compliant operations.

**Installation:**
```bash
npm install -g tdd-guard
# or
brew install tdd-guard
```

**How it works:**
- Intercepts Write, Edit, MultiEdit operations
- Validates against TDD rules using separate Claude session
- Blocks violations with explanatory feedback
- Supports Jest, Vitest, pytest, PHPUnit, Go, Rust

**Key Insight from Creator:**
> "Purely mechanical rule enforcement achieved compliance without improving code quality... TDD's value comes from the mindset and discipline it instills, not from mechanical rule-following."

TDD Guard catches violations but doesn't guarantee good design.

#### Superpowers (obra/superpowers)

**Purpose:** Complete software development workflow with composable skills including TDD.

**Installation:**
```
/plugin marketplace add obra/superpowers-marketplace
/plugin install superpowers@superpowers-marketplace
```

**TDD Enforcement:**
- "Write failing test, watch it fail, write minimal code, watch it pass, commit"
- "Deletes code written before tests"
- Automatic skill checking before any task

**Included Skills:**
- `test-driven-development` - RED-GREEN-REFACTOR cycles
- `testing-anti-patterns` - Prevents mocking abuse
- `verification-before-completion` - Validates fixes

#### Claude Bootstrap

**Purpose:** Opinionated project initialization with security-first, spec-driven approach.

**TDD Enforcement:**
- "TESTS FIRST, ALWAYS" - Features: Write tests -> Watch fail -> Implement -> Pass
- Pre-commit hooks verify linting, type-checking, security, unit tests
- CI blocks below 80% coverage
- Hard limits: 20 lines/function, 3 params/function, 200 lines/file

---

### 5. Limitations and Challenges

#### Fundamental Architectural Limitation

> "When everything runs in one context window, the LLM cannot truly follow TDD. The test writer's detailed analysis bleeds into the implementer's thinking."

Single-context Claude cannot achieve genuine test-first because it designs tests around the implementation it's already planning.

#### Skill Auto-Activation Unreliability

Despite documentation claiming skills are "model-invoked" automatically:

> "Even when users' queries exactly matched skill descriptions, Claude would just ignore the skill and do the work manually."

Testing shows ~50% passive activation--essentially a coin flip. Hooks are required for reliable activation.

#### Instruction Fatigue

Too many CLAUDE.md instructions degrade compliance. The effective limit is ~150-200 instructions total, and Claude Code already uses ~50 in its system prompt.

#### Quality vs. Compliance

Automated enforcement achieves TDD compliance without design quality. The creator of TDD Guard noted:

> "The resultant code still suffered from tight coupling, duplication, and poor overall design."

Human judgment remains essential for design decisions.

---

### 6. React/Next.js/Bun-Specific Patterns

#### Bun Test Integration

Claude Code itself uses Bun for testing. Configure hooks to run Bun tests automatically:

```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "bun test --filter $(basename $CLAUDE_FILE_PATH .ts).spec"
      }]
    }]
  }
}
```

#### React Testing Library Pattern

For React/Next.js apps, the recommended approach:

**CLAUDE.md Configuration:**
```markdown
## React Testing

### Unit Tests
- Use Bun test + @testing-library/react
- Colocate tests as ComponentName.spec.tsx
- Test behavior, not implementation
- Use userEvent over fireEvent

### E2E Tests (Next.js)
- Use Playwright for page-level testing
- Don't unit test Server Components
- Test via actual rendering, not mocks

### What to Test
- Custom hooks and utilities: TDD with unit tests
- Components: E2E verification of user flows
- API routes: Integration tests

### What NOT to Test
- Framework behavior (Next.js routing)
- Pure type transformations
- Third-party library internals
```

#### Next.js Exception Pattern

For Next.js projects, pure TDD isn't always appropriate:

```markdown
### Next.js Testing Strategy

TDD applies to:
- Business logic utilities
- Custom hooks
- Validators and transformations
- API handlers

E2E verification for:
- Page components
- Server Components
- Layout behavior
- Route transitions

Don't write unit tests for Next.js components just to satisfy TDD.
```

---

## Contradictions and Debates

### 1. Automatic vs. Forced Activation

**Anthropic's Position:** Skills are "model-invoked" and Claude "autonomously decides when to use them."

**Community Experience:** Skills activate ~50% of the time without hooks. Forced evaluation hooks achieve ~84%.

**Resolution:** Use hooks for critical workflows like TDD. Don't rely on autonomous activation for important behaviors.

### 2. Single-Agent vs. Multi-Agent TDD

**Single-Agent Advocates:** Hooks and skills can enforce TDD adequately with proper configuration.

**Multi-Agent Advocates:** True test-first is impossible in single context due to information leakage.

**Resolution:** For strict TDD with clean separation, subagents are necessary. For "good enough" TDD, single-agent with hooks suffices.

### 3. Mechanical Compliance vs. Design Quality

**Enforcement Tools:** Focus on preventing implementation-before-test violations.

**Critics:** Mechanical compliance doesn't improve design quality.

**Resolution:** Tools handle compliance; humans handle design judgment. Use TDD Guard for enforcement, but review for design.

---

## Actionable Recommendations

### Minimal Setup (Quick Start)

1. **Add explicit TDD section to CLAUDE.md:**
```markdown
## Testing (MANDATORY)

Write tests first. Every change follows:
1. Write failing test
2. Confirm test fails
3. Write minimal passing code
4. Never skip steps 1-2

Test command: bun test
```

2. **Use explicit prompting:**
```
We're doing TDD. Write a failing test for [feature] first.
Do NOT write implementation code yet.
```

### Intermediate Setup (Hook-Based)

1. **Install TDD Guard:**
```bash
npm install -g tdd-guard
```

2. **Configure hooks in `.claude/settings.json`:**
```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{"type": "command", "command": "tdd-guard"}]
    }]
  }
}
```

### Advanced Setup (Multi-Agent)

1. **Install Superpowers:**
```
/plugin install superpowers@superpowers-marketplace
```

2. **Create phase-specific agents in `.claude/agents/`:**
   - `tdd-test-writer.md`
   - `tdd-implementer.md`
   - `tdd-refactorer.md`

3. **Configure skill with phase gates in `.claude/skills/tdd/SKILL.md`:**
```markdown
---
name: tdd-workflow
description: Enforce RED-GREEN-REFACTOR. Use when implementing features, fixing bugs, or adding functionality.
---

# TDD Workflow

## Phases (MANDATORY SEQUENCE)

### RED: Write Failing Test
Spawn: @tdd-test-writer
Output: Test file path + failure confirmation
Do NOT proceed until test fails.

### GREEN: Implement
Spawn: @tdd-implementer
Input: Failing test only
Output: Passing implementation
Do NOT modify tests.

### REFACTOR: Improve
Spawn: @tdd-refactorer
Output: Changes OR "no refactoring needed"
```

### React/Next.js/Bun Specific

1. **Configure Bun test in CLAUDE.md:**
```markdown
## Testing

bun test                    # All tests
bun test src/features/auth  # Module tests

Tests colocated as *.spec.ts
Use @testing-library/react for components
```

2. **Add post-write hook for auto-testing:**
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Write|Edit",
      "hooks": [{
        "type": "command",
        "command": "bun test --bail 1"
      }]
    }]
  }
}
```

---

## Bibliography

### Official Documentation

- [Claude Code: Best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices) - Anthropic Engineering
- [Agent Skills - Claude Code Docs](https://code.claude.com/docs/en/skills) - Official skills documentation
- [Get started with Claude Code hooks](https://code.claude.com/docs/en/hooks-guide) - Official hooks guide
- [Introducing Claude Opus 4.5](https://www.anthropic.com/news/claude-opus-4-5) - Anthropic announcement

### Tools and Frameworks

- [TDD Guard - GitHub](https://github.com/nizos/tdd-guard) - Automated TDD enforcement tool
- [Superpowers - GitHub](https://github.com/obra/superpowers) - Core skills library including TDD
- [Claude Bootstrap - GitHub](https://github.com/alinaqi/claude-bootstrap) - Security-first project initialization
- [Claude Code Workflows - GitHub](https://github.com/shinpr/claude-code-workflows) - Production-ready development workflows

### Community Research

- [Forcing Claude Code to TDD: An Agentic Red-Green-Refactor Loop](https://alexop.dev/posts/custom-tdd-workflow-claude-code-vue/) - Alex Op's multi-agent approach
- [How to Make Claude Code Skills Activate Reliably](https://scottspence.com/posts/how-to-make-claude-code-skills-activate-reliably) - Scott Spence's hook testing
- [Claude Code Skills Don't Auto-Activate (a workaround)](https://scottspence.com/posts/claude-code-skills-dont-auto-activate) - Auto-activation challenges
- [TDD Guard for Claude Code](https://nizar.se/tdd-guard-for-claude-code/) - Creator's blog post
- [Writing a good CLAUDE.md](https://www.humanlayer.dev/blog/writing-a-good-claude-md) - HumanLayer guidance
- [CLAUDE MD TDD - Wiki](https://github.com/ruvnet/claude-flow/wiki/CLAUDE-MD-TDD) - Configuration templates

### Tutorials and Courses

- [Test-Driven Development with Claude Code](https://stevekinney.com/courses/ai-development/test-driven-development-with-claude) - Steve Kinney's course
- [Claude Code and the Art of Test-Driven Development](https://thenewstack.io/claude-code-and-the-art-of-test-driven-development/) - The New Stack

### Analysis and Commentary

- [The TDD Paradigm Shift](https://medium.com/@moradikor296/the-tdd-paradigm-shift-why-test-driven-development-is-claude-codes-killer-discipline-9be9616d79f6) - Ali Moradi on Medium
- [What Actually Is Claude Code's Plan Mode?](https://lucumr.pocoo.org/2025/12/17/what-is-plan-mode/) - Armin Ronacher
- [Claude Code + Opus 4.5: When the Model Finally Grows into the Harness](https://adam.holter.com/claude-code-opus-4-5-when-the-model-finally-grows-into-the-harness/) - Adam Holter
- [Claude Agent Skills: A First Principles Deep Dive](https://leehanchung.github.io/blogs/2025/10/26/claude-skills-deep-dive/) - Lee Han Chung

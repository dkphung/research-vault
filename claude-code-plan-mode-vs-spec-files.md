---
title: Claude Code Plan Mode vs Spec Files
date: 2026-01-07
tags:
  - claude-code
  - plan-mode
  - workflows
  - spec-files
  - best-practices
---

# Claude Code: Plan Mode vs Spec Files

## Executive Summary

**Plan mode has evolved significantly and is now sufficient for most development workflows**, according to Boris Cherny (creator of Claude Code) and community best practices. However, spec files still provide value for specific scenarios: multi-session complex features, team collaboration, and architectural decisions requiring stakeholder review.

**Key finding**: Boris Cherny uses Plan Mode for *every* non-trivial task, iterating until the plan is solid before switching to auto-accept mode. He does not mention using traditional spec files in his workflow.

**Recommendation**: Default to Plan Mode for daily work. Reserve persistent spec files for:
- Features spanning multiple sessions (context window limits)
- Projects requiring human review/approval before implementation
- Team handoffs or documentation that must survive session closure
- Complex architectural decisions needing version control

## Boris Cherny's Workflow

Boris Cherny shared his Claude Code setup in January 2026, revealing a surprisingly vanilla approach:

### Core Setup
- **5 parallel terminal sessions** + 5-10 on claude.ai/code
- **Opus 4.5 with thinking** for everything ("best coding model I've ever used")
- Numbered tabs 1-5 with system notifications for when Claude needs input

### Plan Mode Strategy

> "A good plan is really important! Developers who skip planning to save time actually spend more time fixing mistakes."

His workflow:
1. **Start in Plan Mode** (Shift+Tab twice) for every non-trivial task
2. **Iterate on the plan** with Claude until satisfied
3. **Switch to auto-accept mode** for execution
4. Claude implements the entire solution without back-and-forth

Plan Mode functions as a "safety rail" while auto-accept is the "accelerator."

### Supporting Infrastructure
- **CLAUDE.md**: Team maintains a shared file, updated multiple times weekly. When Claude errs, they document it to prevent recurrence
- **Verification**: "Give Claude a way to verify its work" increases quality 2-3x
- **Slash commands**: `.claude/commands/` for repeated workflows like `/commit-push-pr`
- **PostToolUse hooks**: Auto-format code to prevent CI failures
- **MCP integrations**: Slack, BigQuery, Sentry without consuming baseline context

**Notable absence**: Boris does not mention spec files, PRDs, or persistent documentation beyond CLAUDE.md.

## Plan Mode Capabilities

### How Plan Mode Works

Plan Mode creates a markdown file in a dedicated plans folder. When activated:
1. Claude analyzes the codebase with read-only operations
2. Generates a structured plan in `plan.md`
3. User reviews and iterates
4. On exit, Claude reads the plan file and executes

**Available tools** (read-only): Read, LS, Glob, Grep, Task, TodoRead/TodoWrite, WebFetch, WebSearch, NotebookRead

**Blocked tools** (modification-prevented): Edit, Write, Bash, NotebookEdit

### Opus 4.5 Enhancements

Opus 4.5 Plan Mode introduces interactive planning workflows:
- **Clarification phase**: Claude asks disambiguating questions upfront
- **Structured plans**: Task breakdown, dependencies, execution order
- **User-editable**: Review and modify before execution
- **Hybrid execution**: Planning with Opus, execution with Sonnet 4.5

Results: 76% fewer tokens while achieving better results.

### Activation Methods
- **Toggle**: Shift+Tab twice (exit with Shift+Tab again)
- **CLI flag**: `--permission-mode plan`
- **Headless**: `-p` flag
- **Model setting**: `/model` option 4 for Opus 4.5 Plan Mode

## Plan Mode Limitations

### Context Window Constraints

The 200K token context window creates practical limits:
- Performance degrades significantly in the final 20% of context
- Sessions running to 90% utilization produce lower-quality code
- **Recommendation**: Stop at 75% utilization for maintainable output

### Plan Persistence Issues

Plans generated in Plan Mode:
- Are stored in `~/.claude/plans` (hidden from user)
- Disappear after session closure
- Cannot be easily edited without exiting the mode
- Don't survive `/clear` commands

As Armin Ronacher noted: "You get a markdown file, but you never get to see it because it's hidden away in a folder."

### Multi-Session Limitations

Claude Code can autonomously work for approximately 10-20 minutes per session. For features exceeding this:
- Context fills with accumulated history
- Quality degrades as the window approaches limits
- Plans must be manually saved to persist across sessions

## When Spec Files Still Add Value

### Persistent Reference Across Sessions

> "The task planning files are still useful. They can serve as a persistent reference for future features, as it is part of the repo, and can be accessed easily by humans and agents alike."

Spec files shine when:
- Features span multiple days/sessions
- Team members need to pick up where others left off
- Historical decisions need documentation for future reference

### Complex Multi-Phase Features

For large features, recommended workflow:
1. **Research phase**: Save `research.md`
2. **Specification phase**: Save `spec.md`
3. **Planning phase**: Save `plan.md`
4. Clear context between each phase

This approach maintains quality by never exceeding 60% context utilization.

### Human Review Requirements

Spec files enable:
- Stakeholder approval before implementation
- Architectural review by senior engineers
- Documentation for compliance/audit requirements
- Design discussion with non-technical team members

### Team Collaboration

When multiple Claude instances or developers work on the same project:
- Spec files prevent collision
- Persistent documentation ensures consistency
- Version-controlled specs track evolution

## Community Best Practices

### The "Explore, Plan, Code, Commit" Workflow

Anthropic's official guidance emphasizes Steps 1-2: "Without them, Claude tends to jump straight to coding a solution."

### Extended Thinking Triggers

Use specific phrases for complex problems:
- "think" (baseline)
- "think hard" (more computation)
- "think harder" (significant depth)
- "ultrathink" (maximum reasoning)

### Context Management Strategies

1. **Simple restart**: `/clear` + `/catchup` (read changed files)
2. **Document & Clear**: Have Claude dump plan/progress to markdown, `/clear`, restart by reading the file
3. **Avoid `/compact`**: Described as "opaque, error-prone, and not well-optimized"

### CLAUDE.md as Constitution

Use `CLAUDE.md` for persistent context:
- Project architecture and conventions
- Common patterns and anti-patterns
- Tool-specific guidance
- Guardrails and alternatives (not just constraints)

### Subagent Strategy

For complex problems:
- Let Claude spawn subagents via `Task(...)` dynamically
- Each subagent gets its own context window
- Preserves main session context for actual implementation

## Decision Framework

### Use Plan Mode When:
- ✅ Starting a new feature (Boris's default)
- ✅ Making edits across multiple files
- ✅ Exploring codebase before changes
- ✅ Iterating on direction with Claude
- ✅ Single-session implementations
- ✅ Daily development work

### Use Spec Files When:
- ✅ Feature spans multiple sessions/days
- ✅ Team handoffs required
- ✅ Stakeholder approval needed before coding
- ✅ Complex architectural decisions
- ✅ Compliance/audit documentation requirements
- ✅ Projects with 3+ phase complexity

### Hybrid Approach

Many experienced developers combine both:
1. Use Plan Mode for immediate planning
2. Save the plan to `docs/PLAN.md` for persistence
3. Reference the file in subsequent sessions
4. Update as implementation progresses

## Actionable Recommendations

### For Solo Developers

1. **Default to Plan Mode** for all non-trivial work
2. **Iterate until solid** before switching to auto-accept
3. **Give Claude verification methods** (tests, visual inspection)
4. **Use `/clear` liberally** between distinct tasks
5. **Save plans to files** only when features span multiple sessions

### For Teams

1. **Maintain shared CLAUDE.md** with patterns and anti-patterns
2. **Use spec files for features requiring review**
3. **Establish clear session handoff protocols**
4. **Version control all plans and specs**
5. **Create slash commands** for team workflows

### For Complex Features

1. **Break into phases**: Research, Spec, Plan, Implement, Verify
2. **Clear context between phases**
3. **Save artifacts at each phase**
4. **Never exceed 60% context utilization**
5. **Use subagents** for parallel investigation

## Conclusion

Plan Mode has matured to the point where it replaces spec files for most daily development work. Boris Cherny's workflow validates this: Plan Mode for every non-trivial task, no mention of traditional spec documents.

However, spec files remain valuable for scenarios where persistence, team collaboration, or stakeholder review matter. The key insight is understanding when ephemeral planning (Plan Mode) is sufficient versus when persistent documentation (spec files) adds value.

**The future appears to be heading toward more sophisticated planning tools** with Opus 4.5's interactive clarification and structured output, potentially making explicit spec files even less necessary. But for now, the hybrid approach offers the best of both worlds.

## Bibliography

### Primary Sources

- [How the Creator of Claude Code Uses Claude Code](https://paddo.dev/blog/how-boris-uses-claude-code/) - Comprehensive analysis of Boris Cherny's workflow
- [The creator of Claude Code just revealed his workflow](https://venturebeat.com/technology/the-creator-of-claude-code-just-revealed-his-workflow-and-developers-are) - VentureBeat coverage
- [Boris Cherny Twitter Thread](https://twitter-thread.com/t/2007179832300581177) - Original thread reader
- [Claude Code: Best practices for agentic coding](https://www.anthropic.com/engineering/claude-code-best-practices) - Anthropic official guidance

### Technical Deep Dives

- [What Actually Is Claude Code's Plan Mode?](https://lucumr.pocoo.org/2025/12/17/what-is-plan-mode/) - Armin Ronacher's technical analysis
- [ClaudeLog Plan Mode Documentation](https://claudelog.com/mechanics/plan-mode/) - Comprehensive Plan Mode mechanics
- [Claude Code Plan Mode Course](https://stevekinney.com/courses/ai-development/claude-code-plan-mode) - Steve Kinney's educational resource

### Workflow Guides

- [My Claude Code Workflow And Personal Tips](https://thegroundtruth.substack.com/p/my-claude-code-workflow-and-personal-tips) - Zhu Liang's dual-system approach
- [How I Use Every Claude Code Feature](https://blog.sshh.io/p/how-i-use-every-claude-code-feature) - Shrivu Shankar's comprehensive guide
- [Cooking with Claude Code: The Complete Guide](https://www.siddharthbharath.com/claude-code-the-complete-guide/) - Sid Bharath's documentation architecture

### Spec-Driven Development

- [claude-code-spec-workflow GitHub](https://github.com/Pimzino/claude-code-spec-workflow) - Automated spec workflow tool
- [Spec Coding Tips for Claude Code](https://medium.com/@sevakavakians/spec-coding-tips-for-claude-code-641109ca3189) - Sevak Avakians on spec patterns
- [OpenSpec + Claude Code Workflow](https://www.vibesparking.com/en/blog/ai/openspec/2025-10-17-openspec-claude-code-dev-process/) - Lightweight spec-driven approach

### Context Management

- [How Claude Code Got Better by Protecting More Context](https://hyperdev.matsuoka.com/p/how-claude-code-got-better-by-protecting) - Context window optimization
- [Claude Code Limits](https://claudelog.com/claude-code-limits/) - Usage and context constraints
- [A Guide to Claude Code 2.0](https://sankalp.bearblog.dev/my-experience-with-claude-code-20-and-how-to-get-better-at-using-coding-agents/) - Sankalp's practical guide

### Related Tools

- [Conductor: Context-driven development for Gemini CLI](https://developers.googleblog.com/conductor-introducing-context-driven-development-for-gemini-cli/) - Google's spec-driven approach
- [GitHub Spec Kit](https://blog.logrocket.com/github-spec-kit/) - GitHub's spec-driven development tool
- [Factory CLI Specification Mode](https://docs.factory.ai/cli/user-guides/specification-mode) - Factory's plan persistence approach

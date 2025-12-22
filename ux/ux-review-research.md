# UX Review Research

**Date:** 2025-12-19

## Summary

Research on conducting comprehensive UX reviews with Claude Code using a hybrid approach combining Claude in Chrome extension, Playwright MCP, and static code analysis.

---

## Available Tools & Capabilities

### Claude in Chrome Extension

Source: [Getting Started with Claude in Chrome](https://support.claude.com/en/articles/12012173-getting-started-with-claude-in-chrome)

| Capability | Tool | Use Case |
|------------|------|----------|
| Navigate pages | `navigate` | Visit each route |
| Take screenshots | `computer(action: screenshot)` | Capture visual state |
| Read page structure | `read_page` | Get accessibility tree |
| Find elements | `find` | Locate specific components |
| Console errors | `read_console_messages` | Catch runtime issues |
| Network requests | `read_network_requests` | Check API calls |
| Record GIFs | `gif_creator` | Document user flows |
| JavaScript execution | `javascript_tool` | Custom analysis |

**Strengths:**
- Uses logged-in session
- Real browser rendering
- Interactive testing
- Sees exactly what users see

**Limitations:**
- Manual/interactive (not automated)
- Requires Chrome open
- One page at a time

### Playwright MCP

Source: [Claude Code Best Practices Comparison](https://github.com/shanraisshan/claude-code-best-practice/blob/main/reports/claude-in-chrome-v-chrome-devtools-mcp.md)

| Capability | Tool | Use Case |
|------------|------|----------|
| Navigate | `browser_navigate` | Automated page visits |
| Screenshot | `browser_take_screenshot` | Full-page captures |
| Accessibility snapshot | `browser_snapshot` | A11y tree analysis |
| Console messages | `browser_console_messages` | Error detection |
| Network requests | `browser_network_requests` | API monitoring |
| Evaluate JS | `browser_evaluate` | Custom checks |

**Strengths:**
- Automated and repeatable
- Can capture all pages quickly
- Headless operation
- Good for CI/CD integration

**Limitations:**
- May need auth setup
- Less interactive

### Code-Based Analysis

Static analysis of components, styles, and patterns without browser.

**What to analyze:**
- `src/components/ui/` - Base component definitions
- `src/app/(auth)/` - All page layouts
- `src/app/globals.css` - Theme tokens
- Component usage patterns across pages

**Strengths:**
- No browser needed
- Can check entire codebase
- Finds code-level issues

**Limitations:**
- Doesn't see rendered output
- Misses visual bugs
- Can't test interactions

---

## UX Heuristics Framework

Based on [Nielsen's 10 Usability Heuristics](https://www.nngroup.com/articles/ten-usability-heuristics/) and [2025 UX Audit Best Practices](https://codi.pro/blog/ux-audit-checklist-for-2025):

### 1. Visual Consistency

| Check | Description |
|-------|-------------|
| Typography hierarchy | H1→H2→H3 consistent across pages |
| Spacing tokens | Consistent margins/padding usage |
| Color usage | Theme compliance, no arbitrary colors |
| Component variants | Button styles match context |
| Icon sizing | Consistent scale throughout |

### 2. Layout & Structure

| Check | Description |
|-------|-------------|
| Visual hierarchy | Attention flows correctly |
| Page patterns | Similar pages have similar layouts |
| Responsive behavior | Breakpoints work consistently |
| Navigation | Predictable sidebar/header behavior |

### 3. Interaction Patterns

| Check | Description |
|-------|-------------|
| Affordance | Buttons look clickable |
| Loading states | User feedback during async operations |
| Error handling | Clear error messages |
| Success feedback | Confirmation of completed actions |
| Empty states | Helpful guidance when no data |

### 4. Accessibility (WCAG 2.1)

| Check | Requirement |
|-------|-------------|
| Color contrast | 4.5:1 minimum ratio |
| Focus states | Visible keyboard focus |
| Form labels | Associated with inputs |
| Keyboard navigation | Logical tab order |
| ARIA attributes | Present for screen readers |

---

## Recommended Workflow

### Phase 1: Project Discovery

Before auditing, discover project-specific context:

1. **Detect framework** (Next.js App/Pages, React Router, Vue, etc.)
2. **Find routes:**
   - Next.js App Router: `Glob src/app/**/page.tsx`
   - Next.js Pages: `Glob pages/**/*.tsx`
   - React Router: `Grep for <Route> patterns`
3. **Find design system:**
   - Tailwind: `tailwind.config.*` or `globals.css @theme`
   - CSS Variables: `Grep for --color, --spacing`
   - Component library: shadcn, MUI, Chakra, etc.
4. **Find component library:**
   - `Glob components/ui/` or `components/common/`
   - Identify base components (Button, Card, Input)

### Phase 2: Code Analysis

1. Check component library for internal consistency
2. Verify design token usage across pages
3. Find pattern deviations
4. Identify anti-patterns (inline styles, magic numbers)

### Phase 3: Visual Audit (Claude in Chrome)

1. Get Chrome tab context
2. Navigate each priority route
3. Take screenshot of each page
4. Check visual consistency
5. Test interactive elements
6. Capture console errors

### Phase 4: Accessibility Audit

1. Use Playwright accessibility snapshots
2. Check WCAG 2.1 compliance
3. Test keyboard navigation
4. Verify screen reader compatibility

### Phase 5: Report Generation

Create report at `docs/audits/ux-audit-{date}.md` including:
- Executive summary
- Project context
- Findings by category
- Severity ratings (Critical/Major/Minor)
- Screenshots
- Recommendations

---

## Sources

- [Getting Started with Claude in Chrome](https://support.claude.com/en/articles/12012173-getting-started-with-claude-in-chrome)
- [Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Claude in Chrome vs Chrome DevTools MCP](https://github.com/shanraisshan/claude-code-best-practice/blob/main/reports/claude-in-chrome-v-chrome-devtools-mcp.md)
- [Nielsen's 10 Usability Heuristics](https://www.nngroup.com/articles/ten-usability-heuristics/)
- [UX Audit Checklist 2025](https://codi.pro/blog/ux-audit-checklist-for-2025)
- [Accessibility Heuristics](https://abstracta.us/blog/accessibility-testing/accessibility-heuristic/)

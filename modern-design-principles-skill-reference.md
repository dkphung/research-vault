---
title: Modern Design Principles for Building a Design Skill
date: 2025-12-25
tags:
  - design
  - ux
  - visual-design
  - skill-building
  - typography
  - color-theory
  - spacing
  - claude-skills
status: complete
---

# Modern Design Principles for Building a Design Skill

## Purpose

This research compiles practical, codifiable design principles from authoritative sources. The goal is to create a skill that teaches Claude good design intuition for clean, intuitive interfaces. This focuses on **visual design skills** rather than UX patterns (which are covered separately).

---

## 1. Foundational Design Philosophy

### Dieter Rams' 10 Principles of Good Design

Source: [Vitsœ](https://www.vitsoe.com/us/about/good-design), [IxDF](https://www.interaction-design.org/literature/article/dieter-rams-10-timeless-commandments-for-good-design)

1. **Good design is innovative** - Technology enables new possibilities
2. **Good design makes a product useful** - Function over decoration
3. **Good design is aesthetic** - Well-executed design is beautiful
4. **Good design makes a product understandable** - Self-explanatory through structure
5. **Good design is unobtrusive** - Tools, not art pieces
6. **Good design is honest** - No false promises or manipulation
7. **Good design is long-lasting** - Avoids trends, stays relevant
8. **Good design is thorough down to the last detail** - Nothing arbitrary
9. **Good design is environmentally friendly** - Conserves resources
10. **Good design is as little design as possible** - "Less, but better"

These principles directly inspired Apple's design language and remain foundational for digital product design.

### Core Philosophy for Digital Design

- **Clarity over cleverness** - Users should understand immediately
- **Content first** - Design serves content, not the reverse
- **Reduce, don't decorate** - Every element must earn its place
- **Consistency builds trust** - Predictable patterns reduce cognitive load

---

## 2. Visual Hierarchy

Source: [IxDF](https://www.interaction-design.org/literature/topics/visual-hierarchy), [Canva](https://www.canva.com/learn/visual-hierarchy/)

### Definition

Visual hierarchy is the order in which humans process information on a page. It guides users from most important to least important elements.

### Primary Tools for Creating Hierarchy

| Tool | Principle | Application |
|------|-----------|-------------|
| **Size** | Larger = more important | Headlines significantly larger than body |
| **Color/Contrast** | High contrast draws attention | Dark text on light, accent colors for CTAs |
| **Position** | Top-left (LTR) gets most attention | Key info above the fold |
| **Whitespace** | More space = more importance | Generous padding around primary elements |
| **Typography weight** | Bold draws eye first | Bold headlines, regular body |
| **Proximity** | Related items grouped | Form labels near inputs |
| **Depth** | Shadows/elevation = prominence | Modals float above content |

### Hierarchy Levels

1. **Primary**: Single most important element (main CTA, headline)
2. **Secondary**: Supporting information (subheads, secondary actions)
3. **Tertiary**: Body content, metadata, auxiliary info

### Critical Rule

> When everything competes for attention, nothing stands out. A design with clear hierarchy feels "designed"; without it, feels chaotic.

### Common Hierarchy Mistakes

- Multiple elements at same visual weight
- Competing colors/sizes without purpose
- Burying primary actions below secondary content
- Using bold/color for decoration rather than meaning

---

## 3. Spacing Systems

### The 8-Point Grid System

Source: [Spec.fm](https://spec.fm/specifics/8-pt-grid), [Medium](https://medium.com/built-to-adapt/intro-to-the-8-point-grid-system-d2573cde8632)

**Core Principle**: All dimensions, padding, and margins use multiples of 8px.

```
Standard scale: 8, 16, 24, 32, 40, 48, 56, 64, 72, 80, 96, 128...
```

**Why 8?**
- Most screen sizes divisible by 8 on at least one axis
- Scales perfectly for Android (0.75x, 1.5x) and iOS
- Apple and Google both recommend 8-based systems
- Reduces decision fatigue (eliminates 7/8 of possible values)
- Creates predictable, harmonious visual rhythm

**Implementation Rules**:

| Element | Spacing |
|---------|---------|
| Component internal padding | 8, 16, 24px |
| Between related elements | 8, 16px |
| Between sections | 24, 32, 48px |
| Page margins | 16, 24, 32px (mobile), 48, 64px (desktop) |
| Card padding | 16, 24px |
| Button padding | 8-12px vertical, 16-24px horizontal |

**Exception - 4px for Micro-Spacing**:
- Tight text groupings
- Icon-to-label spacing
- Small element internal padding
- Fine-tuning optical alignment

### Whitespace Principles

Source: [Flux Academy](https://www.flux-academy.com/blog/the-importance-of-whitespace-in-design-with-examples), [IxDF](https://www.interaction-design.org/literature/article/the-power-of-white-space)

**Types of Whitespace**:

| Type | Description | Example |
|------|-------------|---------|
| **Macro** | Between major content blocks | Section spacing, margins |
| **Micro** | Within elements | Line-height, letter-spacing, padding |
| **Active** | Intentional for emphasis | Space around hero CTA |
| **Passive** | Natural byproduct | Kerning, paragraph spacing |

**Key Principles**:

1. **Whitespace = Importance**: More space around an element signals higher importance
2. **Whitespace = Premium**: Generous spacing signals quality/luxury (like fine dining plating)
3. **Whitespace = Clarity**: Separates unrelated elements, groups related ones
4. **Whitespace is not wasted space**: It's a design element itself

**Practical Rules**:
- Internal spacing ≤ External spacing (content inside a card has less padding than space between cards)
- Consistent rhythm: If sections have 48px between them, maintain throughout
- More whitespace as screen size increases (responsive scaling)

---

## 4. Typography

### Type Scales

Source: [A List Apart](https://alistapart.com/article/more-meaningful-typography/), [Cieden](https://cieden.com/book/sub-atomic/typography/establishing-a-type-scale)

A modular type scale creates harmonious size relationships using a consistent ratio.

**Common Ratios**:

| Ratio | Value | Character | Best For |
|-------|-------|-----------|----------|
| Minor Second | 1.067 | Very subtle | Dense data tables |
| Major Second | 1.125 | Subtle | Documentation, dashboards |
| Minor Third | 1.200 | Moderate | Most UI applications |
| Major Third | 1.250 | Clear | Marketing, content sites |
| Perfect Fourth | 1.333 | Strong | Editorial, blogs |
| Golden Ratio | 1.618 | Dramatic | Creative, high-impact |

**Example Scale (16px base, 1.25 ratio)**:
```
xs:  10px (0.625rem)
sm:  13px (0.8125rem)
base: 16px (1rem)
lg:  20px (1.25rem)
xl:  25px (1.5625rem)
2xl: 31px (1.9375rem)
3xl: 39px (2.4375rem)
4xl: 49px (3.0625rem)
```

**Responsive Considerations**:
- Desktop: Higher ratio (1.333 or 1.618) for impact
- Mobile: Lower ratio (1.2 or 1.25) to fit content
- Scale base size up on larger screens (16px → 18px)

### Typography Best Practices

Source: [Figma](https://www.figma.com/resource-library/typography-in-design/), [Toptal](https://www.toptal.com/designers/typography/typographic-hierarchy)

**Line Height (Leading)**:
- Body text: 1.4–1.6× font size (140-160%)
- Headlines: 1.1–1.3× font size (tighter)
- Small text: 1.5–1.7× font size (looser for readability)

**Line Length (Measure)**:
- Optimal: 45-75 characters per line
- Ideal: ~66 characters
- Too short: Choppy reading
- Too long: Eye loses track returning to start

**Font Pairing**:
- Maximum 2 font families
- Contrast in purpose: One for headings, one for body
- If using one font: Vary weight, not family

**Hierarchy Structure**:
```
H1: 39-49px, Bold, Primary headlines
H2: 31px, Semibold, Section headers
H3: 25px, Semibold, Subsections
H4: 20px, Medium, Card titles
Body: 16px, Regular, Content
Small: 13-14px, Regular, Captions, metadata
```

**Minimum Sizes**:
- Body text: 16px minimum for screens
- Small/caption text: Never below 12px
- Touch interfaces: 16px prevents zoom on iOS

---

## 5. Color

### The 60-30-10 Rule

Source: [UX Planet](https://uxplanet.org/the-60-30-10-rule-a-foolproof-way-to-choose-colors-for-your-ui-design-d15625e56d25), [Hype4](https://hype4.academy/articles/design/60-30-10-rule-in-ui)

A proportion guideline from interior design that creates balanced color harmony.

| Percentage | Role | Characteristics | Application |
|------------|------|-----------------|-------------|
| **60%** | Dominant | Neutral, calm | Backgrounds, large surfaces |
| **30%** | Secondary | Complementary | Headers, sidebars, cards |
| **10%** | Accent | Vibrant, attention-grabbing | CTAs, links, highlights |

**Real-World Examples**:

| Brand | 60% Dominant | 30% Secondary | 10% Accent |
|-------|--------------|---------------|------------|
| Google | White | Light Gray | Blue |
| Spotify | Black | Dark Gray | Green |
| Slack | White | Purple tints | Purple/Yellow |
| Linear | Dark Gray | Lighter Gray | Purple |

**Implementation Tips**:
- Dominant should be neutral (white, off-white, light/dark gray)
- Secondary provides structure without competing
- Accent reserved for actionable elements only
- Use shades/tints within each tier for depth

### Contrast Requirements

Source: [NN/g](https://www.nngroup.com/articles/color-enhance-design/), [WebAIM](https://webaim.org/resources/contrastchecker/)

**WCAG Standards**:

| Element | Minimum Ratio | Level |
|---------|---------------|-------|
| Body text | 4.5:1 | AA |
| Large text (18pt+) | 3:1 | AA |
| UI components | 3:1 | AA |
| Body text | 7:1 | AAA |

**Common Failures**:
- Light gray text on white (often ~2:1)
- Placeholder text too light
- Disabled states indistinguishable
- Color-only differentiation (accessibility issue)

**83.6%** of top websites fail contrast requirements (WebAIM 2024).

### Color System Structure

For complex UIs, define a complete palette upfront:

```
Primary:    50, 100, 200, 300, 400, 500, 600, 700, 800, 900
Secondary:  50, 100, 200, 300, 400, 500, 600, 700, 800, 900
Neutral:    50, 100, 200, 300, 400, 500, 600, 700, 800, 900
Success:    Light, Default, Dark
Warning:    Light, Default, Dark
Error:      Light, Default, Dark
Info:       Light, Default, Dark
```

---

## 6. Touch Targets & Interactive Elements

Source: [Smashing Magazine](https://www.smashingmagazine.com/2023/04/accessible-tap-target-sizes-rage-taps-clicks/), [accessibility.digital.gov](https://accessibility.digital.gov/ux/touch-targets/)

### Size Guidelines by Platform

| Platform | Minimum | Recommended | Notes |
|----------|---------|-------------|-------|
| **WCAG 2.1 AA** | 24×24px | 44×44px | Legal baseline |
| **WCAG 2.1 AAA** | 44×44px | - | Best practice |
| **Apple iOS** | 44×44pt | - | Human Interface Guidelines |
| **Android** | 48×48dp | 8dp spacing | Material Design |
| **Apple VisionOS** | 60pt | - | Spatial computing |

### Practical Implementation

**Button Sizing**:
```
Small:    32px height (icons only, use sparingly)
Default:  40-44px height (most buttons)
Large:    48-56px height (primary CTAs on mobile)
```

**Padding Inside Buttons**:
- Vertical: 8-12px
- Horizontal: 16-24px (≥ vertical)
- Icon + text: 8px gap between

**Spacing Between Buttons**:
- Minimum: 8px
- Recommended: 12-16px
- Prevents mis-taps

**Target Area vs Visual Size**:
The clickable/tappable area can exceed the visible button. A 24px icon can have a 44px touch target via padding.

### Screen Position Matters

Research by Steven Hoober shows accuracy varies by screen position:

| Position | Recommended Target |
|----------|-------------------|
| Top of screen | 42px (11mm) |
| Center | 27px (7mm) minimum |
| Bottom of screen | 46px (12mm) |

Thumb reach and grip affect accuracy—bottom corners are hardest to tap.

---

## 7. Icon Design

Source: [Font Awesome](https://blog.fontawesome.com/icon-grid-ensures-consistent-design/), [IBM Design](https://www.ibm.com/design/language/iconography/ui-icons/design/)

### Grid System

Icons should be designed on a consistent grid:

**Common Grid Sizes**:
- 16×16px: Small inline icons
- 24×24px: Standard UI icons (most common)
- 32×32px: Feature icons, navigation
- 48×48px: Large feature illustrations

**Grid Components**:
1. **Frame**: Outer boundary
2. **Safe area**: Padding from edge (typically 2px)
3. **Key shapes**: Templates for common forms (circle, square, rectangle)
4. **Live area**: Where icon content lives

### Visual Weight & Consistency

**Stroke Weight**:
- Pick one weight and use consistently (1.5px, 2px common)
- Never mix weights within an icon set
- Thinner = more refined; thicker = more bold/friendly

**Optical Balance**:
- Circles appear smaller than squares at same dimensions
- Expand circles ~4% to appear equal
- Narrow icons can be rotated 45° for more visual weight
- Horizontal icons appear shorter than vertical ones

**Style Consistency Checklist**:
- [ ] Same stroke weight throughout
- [ ] Same corner radius (0, 2px, or fully rounded)
- [ ] Same level of detail/abstraction
- [ ] Same visual weight (no heavy vs light mix)
- [ ] Aligned to pixel grid (no blurry edges)

### Icon Best Practices

- **Recognizability**: Test at smallest size; must be identifiable
- **Simplicity**: Reduce to essential features
- **Metaphor**: Use familiar real-world associations
- **Pairing**: Icons + labels > icons alone for clarity

---

## 8. Gestalt Principles

Source: [IxDF](https://www.interaction-design.org/literature/topics/gestalt-principles), [UXMatters](https://www.uxmatters.com/mt/archives/2024/11/how-gestalt-principles-influence-ux-design.php)

The brain organizes visual information into patterns automatically. Design can leverage these tendencies.

### Core Principles

| Principle | Description | UI Application |
|-----------|-------------|----------------|
| **Proximity** | Nearby objects perceived as related | Group form fields, nav items |
| **Similarity** | Similar objects perceived as related | Card grids, tab groups |
| **Continuity** | Eye follows lines/paths | Progress bars, flows |
| **Closure** | Mind completes incomplete shapes | Icons, logos with gaps |
| **Figure/Ground** | Distinguish foreground from background | Modals, dropdowns, overlays |
| **Common Region** | Elements in bounded area = grouped | Cards, sections, containers |
| **Symmetry** | Symmetric elements seen as unified | Layouts, icon design |

### Application Examples

**Proximity**:
```
Bad:  Label          Input field

      Label          Input field

Good: Label
      Input field

      Label
      Input field
```

**Common Region**:
Wrap related settings in a card/container to indicate they belong together.

**Similarity**:
All navigation items should look the same; the active one uses accent color.

**Figure/Ground**:
Modal dialogs dim background to create clear foreground focus.

---

## 9. Laws of UX

Source: [lawsofux.com](https://lawsofux.com), [Laws of UX Book](https://lawsofux.com/book/) by Jon Yablonski

### Most Actionable Laws for Visual Design

| Law | Psychology | Design Application |
|-----|------------|-------------------|
| **Aesthetic-Usability Effect** | Beautiful things perceived as more usable | Invest in visual polish; it affects perceived quality |
| **Fitts's Law** | Time to target = f(distance, size) | Make important targets large and close to likely cursor position |
| **Hick's Law** | Decision time increases with choices | Limit options; use progressive disclosure |
| **Jakob's Law** | Users expect conventions | Follow platform patterns; don't reinvent navigation |
| **Miller's Law** | Working memory: 7±2 items | Chunk information; don't show >7 items without grouping |
| **Law of Proximity** | Nearby = related | Group related elements; separate unrelated |
| **Law of Similarity** | Similar = related | Consistent styling for similar functions |
| **Von Restorff Effect** | Distinctive items remembered | Make CTAs visually distinct from surroundings |
| **Peak-End Rule** | Experience judged by peak & end | Polish key moments: first impression, success states |
| **Tesler's Law** | Complexity is conserved | Irreducible complexity exists; move it to the right place |
| **Doherty Threshold** | <400ms response feels instant | Optimize performance; use loading states for longer waits |
| **Serial Position Effect** | First and last items remembered best | Put key items at start/end of lists |
| **Postel's Law** | Be liberal accepting, conservative sending | Accept varied input; output clearly formatted results |

### Application Framework

When making design decisions, ask:
1. Does this follow what users expect? (Jakob's Law)
2. Are there too many choices? (Hick's Law)
3. Are important targets easy to hit? (Fitts's Law)
4. Is the CTA visually distinct? (Von Restorff)
5. Is information chunked appropriately? (Miller's Law)

---

## 10. Common Anti-Patterns

Source: [Dev.to](https://dev.to/pixel_mosaic/10-common-ui-design-mistakes-developers-make-and-how-to-fix-them-1mmc), [CareerFoundry](https://careerfoundry.com/en/blog/ui-design/common-ui-design-mistakes/)

### Visual Clutter

**Symptoms**:
- Too many competing elements
- Borders, shadows, colors everywhere
- No clear visual priority
- Users feel overwhelmed

**Solutions**:
- Remove decorative elements that don't serve function
- Use whitespace instead of borders to separate
- Establish clear hierarchy (one primary, limited secondary)
- Progressive disclosure for complex features

### Spacing Inconsistency

**Symptoms**:
- Random padding/margin values (13px, 17px, 22px)
- Uneven spacing between similar elements
- No systematic approach

**Solutions**:
- Adopt 8px grid strictly
- Define spacing scale: 4, 8, 16, 24, 32, 48, 64
- Apply consistently throughout

### Typography Chaos

**Symptoms**:
- Random font sizes
- More than 2 font families
- Inconsistent line-height
- Missing hierarchy

**Solutions**:
- Define type scale upfront
- Stick to 1-2 fonts
- Consistent line-height per text type
- Clear H1 → H2 → Body hierarchy

### Poor Contrast

**Symptoms**:
- Light gray on white text
- Color-only differentiation
- Disabled states invisible

**Solutions**:
- Test all text: minimum 4.5:1 contrast
- Use multiple cues (color + icon + text)
- Disabled: Reduce opacity, don't just gray out

### Button Hierarchy Failure

**Symptoms**:
- All buttons look equally important
- Primary action hard to find
- Too many CTAs competing

**Solutions**:
- One primary button per view
- Secondary buttons: outlined or ghost style
- Tertiary: text-only links
- Visual weight: Primary > Secondary > Tertiary

### Mystery Meat Navigation

**Symptoms**:
- Icons without labels
- Unclear what clicking will do
- Hovering required to understand

**Solutions**:
- Always pair icons with labels (or use tooltips)
- Use conventional, recognizable icons
- Make touch targets clearly interactive

---

## 11. Design Systems & Tokens

Source: [Adobe Spectrum](https://spectrum.adobe.com/page/design-tokens/), [Atlassian](https://atlassian.design/tokens/design-tokens/)

### What Are Design Tokens?

Design tokens are design decisions translated into data—name/value pairs representing small, repeatable design decisions.

```json
{
  "color-background-primary": "#FFFFFF",
  "color-text-primary": "#1A1A1A",
  "spacing-md": "16px",
  "radius-default": "8px",
  "font-size-body": "16px"
}
```

### Token Hierarchy

| Level | Description | Example |
|-------|-------------|---------|
| **Primitive** | Raw values | `blue-500: #3B82F6` |
| **Semantic** | Purpose-based | `color-action-primary: {blue-500}` |
| **Component** | Component-specific | `button-background: {color-action-primary}` |

### Benefits

1. **Single source of truth**: Change once, update everywhere
2. **Consistency**: Enforces design decisions systematically
3. **Scalability**: Easy to maintain across large products
4. **Handoff**: Designers and developers share same language
5. **Theming**: Switch themes by swapping token values

### Core Token Categories

```
Colors:
  - Background (primary, secondary, elevated)
  - Text (primary, secondary, disabled, inverse)
  - Border (default, focus, error)
  - Action (primary, secondary, destructive)
  - Status (success, warning, error, info)

Spacing:
  - xs: 4px
  - sm: 8px
  - md: 16px
  - lg: 24px
  - xl: 32px
  - 2xl: 48px

Typography:
  - Font families
  - Font sizes (scale)
  - Font weights
  - Line heights
  - Letter spacing

Radius:
  - none: 0
  - sm: 4px
  - md: 8px
  - lg: 16px
  - full: 9999px

Shadows:
  - sm, md, lg, xl
  - Focus rings

Motion:
  - Duration (fast, normal, slow)
  - Easing curves
```

---

## 12. Decision-Making Framework

### Quick Design Checklist

Before finalizing any design, verify:

**Spacing**:
- [ ] All values on 8px grid (or 4px for micro)
- [ ] Consistent padding within similar components
- [ ] Adequate whitespace between sections
- [ ] Internal spacing ≤ external spacing

**Typography**:
- [ ] Using defined type scale
- [ ] ≤ 2 font families
- [ ] Line height: 1.4-1.6 for body
- [ ] Clear H1 → H2 → Body hierarchy
- [ ] 45-75 characters per line

**Color**:
- [ ] Following 60-30-10 proportions
- [ ] Accent used sparingly (only actionable items)
- [ ] All text passes 4.5:1 contrast
- [ ] No color-only differentiation

**Hierarchy**:
- [ ] Single clear primary action
- [ ] Secondary actions visually subordinate
- [ ] Eye flows logically through content
- [ ] Most important info most prominent

**Interactive Elements**:
- [ ] Touch targets ≥ 44px
- [ ] Buttons have clear hover/active states
- [ ] Focus states visible for keyboard users
- [ ] Primary vs secondary buttons distinguishable

**Icons**:
- [ ] Consistent stroke weight
- [ ] Same visual weight across set
- [ ] Paired with labels where needed
- [ ] Aligned to pixel grid

### When to Apply Which Principle

| Situation | Primary Principle |
|-----------|------------------|
| Grouping related items | Proximity, Common Region |
| Creating emphasis | Size, Contrast, Whitespace |
| Guiding user flow | Continuity, Visual Hierarchy |
| Making CTAs stand out | Von Restorff, Contrast |
| Reducing overwhelm | Hick's Law, Progressive Disclosure |
| Setting expectations | Jakob's Law, Conventions |
| Optimizing targets | Fitts's Law, Touch Targets |

---

## 13. Key Reference Materials

### Books

| Book | Author | Focus |
|------|--------|-------|
| **Refactoring UI** | Adam Wathan, Steve Schoger | Practical visual design tactics |
| **Laws of UX** | Jon Yablonski | Psychology-based design principles |
| **Don't Make Me Think** | Steve Krug | Usability fundamentals |
| **The Design of Everyday Things** | Don Norman | Foundational design thinking |

### Design Systems

| System | URL | Notable For |
|--------|-----|-------------|
| Apple HIG | [developer.apple.com/design](https://developer.apple.com/design/human-interface-guidelines/) | Platform conventions, clarity |
| Material Design | [m3.material.io](https://m3.material.io/) | Comprehensive, well-documented |
| IBM Carbon | [carbondesignsystem.com](https://carbondesignsystem.com/) | Enterprise, accessibility |
| Atlassian | [atlassian.design](https://atlassian.design/) | Practical token system |
| Linear | [linear.app/design](https://linear.app/) | Modern, minimal aesthetic |

### Tools

| Tool | Purpose | URL |
|------|---------|-----|
| Type Scale | Generate type scales | [type-scale.com](https://type-scale.com/) |
| Contrast Checker | Test accessibility | [webaim.org/resources/contrastchecker](https://webaim.org/resources/contrastchecker/) |
| Coolors | Color palette generation | [coolors.co](https://coolors.co/) |
| Realtime Colors | See colors in context | [realtimecolors.com](https://realtimecolors.com/) |

---

## 14. Skill Structure Recommendation

Based on this research, a Claude design skill should be organized around:

### 1. Detection Layer
Identify design decisions being made:
- Is this a spacing decision?
- Is this a color decision?
- Is this a hierarchy decision?
- Is this a component sizing decision?

### 2. Principle Application
For each decision type, apply relevant rules:
- Spacing → 8px grid
- Color → 60-30-10, contrast ratios
- Hierarchy → Size/weight/position/whitespace
- Sizing → Touch target minimums

### 3. Anti-Pattern Detection
Flag when output would violate principles:
- Random spacing values
- Low contrast text
- Competing visual weights
- Too many fonts/colors

### 4. Quality Verification
Final checklist before output:
- Passes contrast requirements?
- Follows spacing system?
- Clear visual hierarchy?
- Appropriate touch targets?
- Consistent styling?

### 5. Contextual Adaptation
Adjust recommendations based on:
- Platform (iOS vs Android vs Web)
- Screen size (mobile vs desktop)
- Content type (dashboard vs marketing)
- Brand constraints (if provided)

---

## Summary

Good design is not subjective—it follows learnable, codifiable principles:

1. **Less is more**: Remove until it breaks, then add back one thing
2. **Consistency creates trust**: Systematic spacing, colors, typography
3. **Hierarchy guides attention**: One primary, clear secondary/tertiary
4. **Whitespace is a feature**: More space = more importance
5. **Conventions reduce friction**: Follow platform patterns
6. **Accessibility is baseline**: Contrast, targets, not color-only
7. **Details matter**: Pixel-perfect alignment, consistent weights

A designer with good intuition has internalized these principles so deeply that applying them feels automatic. A skill should encode these same principles as explicit checks and recommendations.

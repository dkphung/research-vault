---
tags: [architecture]
date: 2024-12-22
status: complete
---

# Annotation Overlay Refactoring Analysis

**Date:** 2025-11-07
**Component:** `AnnotationOverlayClient.tsx`
**Issue:** Multiple useEffect hooks indicating mixed responsibilities and architectural complexity

---

## Executive Summary

The `AnnotationOverlayClient` component suffers from **God Component** anti-pattern with 6+ useEffect hooks managing:
- Pin lifecycle (creation, orphan detection, anchoring)
- Text selection capture and highlighting
- Highlight persistence and restoration
- Event delegation
- Panel coordination

**Recommended approach:** Extract custom hooks (quick win) → separate presentational/container components → consider state machine for complex flows.

---

## Problem Analysis

### Current Component Structure

```
AnnotationOverlayClient (360 lines)
├── 6+ useEffect hooks (~150 lines of effect logic)
├── 4 useCallback handlers
├── 1 useMemo for pins derivation
├── DOM manipulation in multiple places
└── Tight coupling to global store
```

### Code Smells Identified

#### 1. **Too Many Responsibilities (SRP Violation)**

The component handles:
- ✗ Pin rendering and positioning
- ✗ Pin creation workflow
- ✗ Orphaned pin detection
- ✗ Text selection capture
- ✗ Highlight creation and management
- ✗ Highlight persistence/restoration
- ✗ Click event delegation
- ✗ Panel state coordination
- ✗ Scroll and animation orchestration

**Single Responsibility Principle:** A component should have one reason to change. This component has 9+ reasons to change.

#### 2. **Large useEffect Hooks**

```typescript
// 60+ line effect for text selection
useEffect(() => {
  // Don't handle selection if in pin creation mode
  if (isCreatingPin) return;

  let debounceTimer: NodeJS.Timeout | null = null;

  const handleSelectionChange = () => {
    // Debounce logic
    // Selection capture
    // Highlight creation
    // State updates
    // Error handling
  };

  document.addEventListener("selectionchange", handleSelectionChange);

  return () => {
    // Cleanup logic
  };
}, [isCreatingPin, activeSelection, draftHighlight, setActiveSelection, clearSelection]);
```

**Problems:**
- Hard to test in isolation
- Complex dependency array
- Mixing concerns (debouncing, selection, highlighting, state)
- Potential race conditions

#### 3. **Direct DOM Manipulation**

```typescript
// Line 120: Query and manipulate DOM elements
const commentElement = document.querySelector(`[data-comment-id="${pinId}"]`);
commentElement.scrollIntoView({ behavior: "smooth", block: "center" });

// Line 128-144: Manual animation
const avatar = commentElement.querySelector(".comment-avatar");
avatar.animate(pulseKeyframes, animationOptions);
```

**Problems:**
- Breaks React's declarative model
- Hard to test
- Potential hydration issues
- Scattered across multiple handlers

#### 4. **Complex Dependency Arrays**

```typescript
useEffect(() => {
  // ... complex logic
}, [isCreatingPin, activeSelection, draftHighlight, setActiveSelection, clearSelection]);
```

When you have 5+ dependencies, it's usually a sign that:
- The effect does too much
- State is too fragmented
- The component needs decomposition

#### 5. **Tight Coupling to Global Store**

```typescript
const {
  activePin, isCreatingPin, draftPin,
  activeSelection, draftHighlight,
  setActivePin, setCreatingPin, setDraftPin,
  setActiveSelection, setDraftHighlight,
  setActiveHighlight, clearSelection
} = useCommentsStore();
```

**12 properties from a single store** indicates:
- Store might be too broad
- Component is doing too much
- Hard to test in isolation

---

## Refactoring Strategies

### Strategy 1: Extract Custom Hooks (✅ Implemented)

**Complexity:** Low
**Impact:** Medium
**Risk:** Low

Extract each useEffect into a focused custom hook:

#### Created Hooks:

1. **`useOrphanedPins(pins)`** - Tracks which pins are orphaned
   - Single responsibility: orphan detection
   - Easy to test
   - Clear dependencies

2. **`usePinAnchoring(pins)`** - Applies CSS anchor positioning
   - Single responsibility: anchor lifecycle
   - Cleanup handled automatically
   - Testable with mock DOM

3. **`useTextSelection({ isCreatingPin })`** - Captures text selection and creates highlights
   - Encapsulates complex selection logic
   - Manages debouncing internally
   - Clear contract with store

4. **`useHighlightRestoration(initialComments)`** - Restores persisted highlights
   - Runs once on mount
   - No complex dependencies
   - Easy to test with mock comments

5. **`useHighlightClickHandler({ wrapperRef, onHighlightClick, onBackgroundClick })`** - Event delegation
   - Single responsibility: click routing
   - Testable with mock events
   - Clear callback interface

#### Benefits:

✅ **Reduced component complexity** from 360 → 200 lines
✅ **Easier to test** each hook in isolation
✅ **Better readability** - main component shows high-level flow
✅ **Reusable** - hooks can be used in other components
✅ **Maintains existing architecture** - low risk refactor

#### Comparison:

```typescript
// BEFORE: 360 lines with 6 inline useEffects
export function AnnotationOverlayClient({ ... }) {
  useEffect(() => { /* 20 lines orphan detection */ }, [pins]);
  useEffect(() => { /* 15 lines anchoring */ }, [pins]);
  useEffect(() => { /* 60 lines selection handling */ }, [isCreatingPin, ...]);
  useEffect(() => { /* 30 lines restoration */ }, [initialComments]);
  useEffect(() => { /* 25 lines click delegation */ }, [handleHighlightClick, ...]);

  return <div>...</div>;
}

// AFTER: 200 lines with clear intent
export function AnnotationOverlayClient({ ... }) {
  const orphanedPins = useOrphanedPins(pins);
  usePinAnchoring(pins);
  useTextSelection({ isCreatingPin });
  useHighlightRestoration(initialComments);
  useHighlightClickHandler({ wrapperRef, onHighlightClick, onBackgroundClick });

  return <div>...</div>;
}
```

---

### Strategy 2: Container/Presentational Split (Recommended Next Step)

**Complexity:** Medium
**Impact:** High
**Risk:** Medium

Split into multiple focused components:

```
AnnotationOverlayContainer (logic)
├── PinOverlay (presentational)
│   └── PinMarker[]
├── HighlightManager (logic + rendering)
│   └── SelectionBadge
└── EventHandler (invisible, event delegation)
```

#### Example Structure:

```typescript
// AnnotationOverlayContainer.tsx - Orchestration & state
export function AnnotationOverlayContainer({ targetId, targetType, pageUrl, initialComments, children }) {
  const pins = usePins(initialComments);
  const highlights = useHighlights(initialComments);
  const { handlePinClick, handleHighlightClick } = useAnnotationHandlers();

  return (
    <div ref={wrapperRef}>
      {children}
      <PinOverlay pins={pins} onPinClick={handlePinClick} />
      <HighlightManager highlights={highlights} onHighlightClick={handleHighlightClick} />
    </div>
  );
}

// PinOverlay.tsx - Pure presentation
export function PinOverlay({ pins, onPinClick }) {
  const orphanedPins = useOrphanedPins(pins);
  usePinAnchoring(pins);

  return pins.map(pin => (
    <PinMarker
      key={pin._id}
      pin={pin}
      isActive={activePin === pin._id}
      isOrphaned={orphanedPins.has(pin._id)}
      onClick={() => onPinClick(pin._id)}
    />
  ));
}

// HighlightManager.tsx - Highlight logic
export function HighlightManager({ highlights, onHighlightClick }) {
  const activeSelection = useTextSelection();
  const draftHighlight = useDraftHighlight();
  useHighlightRestoration(highlights);

  return activeSelection && !draftHighlight ? (
    <SelectionBadge
      boundingRect={activeSelection.boundingRect}
      onClick={() => onHighlightClick(activeSelection)}
    />
  ) : null;
}
```

#### Benefits:

✅ **Clear separation of concerns** - each component has one job
✅ **Easier testing** - test presentation vs logic separately
✅ **Better composition** - components can be reused
✅ **Performance** - easier to memoize pure components

#### Tradeoffs:

⚠️ **More files** - 3-4 components instead of 1
⚠️ **Prop drilling** - need to pass callbacks down
⚠️ **Medium refactor** - changes component structure

---

### Strategy 3: State Machine for Complex Flows (Advanced)

**Complexity:** High
**Impact:** Very High
**Risk:** High

For managing complex state transitions (pin creation, highlight drafting):

```typescript
type AnnotationState =
  | { type: 'idle' }
  | { type: 'creating_pin'; position: Coordinates }
  | { type: 'selecting_text'; selection: SelectionData }
  | { type: 'draft_highlight'; highlightId: string }
  | { type: 'draft_pin'; pin: DraftPin };

type AnnotationEvent =
  | { type: 'START_PIN_CREATION' }
  | { type: 'CANCEL_PIN' }
  | { type: 'TEXT_SELECTED'; data: SelectionData }
  | { type: 'CLEAR_SELECTION' }
  | { type: 'CREATE_HIGHLIGHT' };

function annotationReducer(state: AnnotationState, event: AnnotationEvent): AnnotationState {
  // XState or custom reducer
}
```

#### Benefits:

✅ **Explicit state transitions** - no invalid states
✅ **Easier to reason about** - clear state diagram
✅ **Testable** - pure state transitions
✅ **Self-documenting** - state machine is the spec

#### When to Use:

- Complex workflows with many states
- Need to prevent invalid state combinations
- State transitions have business logic
- Team is comfortable with state machines

#### Tradeoffs:

⚠️ **Learning curve** - team needs to learn XState or similar
⚠️ **Overkill for simple flows** - adds complexity unnecessarily
⚠️ **High refactor effort** - complete rewrite of state logic

---

### Strategy 4: Event Bus for Decoupling (Alternative)

**Complexity:** Medium
**Impact:** High
**Risk:** Medium

Use an event emitter to decouple components:

```typescript
// annotationEvents.ts
export const annotationEvents = new EventEmitter();

// In PinOverlay
annotationEvents.emit('pin:clicked', { pinId });

// In AnnotationOverlayContainer
useEffect(() => {
  const handlePinClick = ({ pinId }) => {
    scrollToComment(pinId);
    animateComment(pinId);
  };

  annotationEvents.on('pin:clicked', handlePinClick);
  return () => annotationEvents.off('pin:clicked', handlePinClick);
}, []);
```

#### Benefits:

✅ **Loose coupling** - components don't know about each other
✅ **Easier to extend** - add new listeners without changing emitters
✅ **Testable** - mock event bus

#### Tradeoffs:

⚠️ **Hidden dependencies** - not clear who listens to what
⚠️ **Harder to trace** - events flow is implicit
⚠️ **Potential memory leaks** - must clean up listeners
⚠️ **Not idiomatic React** - React prefers explicit prop passing

---

## Deeper Architectural Issues

Beyond the useEffect code smell, there are systemic issues:

### 1. **Global State Overuse**

The component pulls 12 properties from `useCommentsStore()`. This suggests:

**Problem:** Store is too broad and component is too dependent on it.

**Solution Options:**

a) **Slice the store** - separate pin state from highlight state:
```typescript
// Instead of one big store
const useCommentsStore = create((set) => ({
  // 20+ properties
}));

// Split into focused stores
const usePinStore = create((set) => ({
  activePin, isCreatingPin, draftPin,
  setActivePin, setCreatingPin, setDraftPin
}));

const useHighlightStore = create((set) => ({
  activeSelection, draftHighlight,
  setActiveSelection, setDraftHighlight, clearSelection
}));
```

b) **Use local state** - Keep state in component when possible:
```typescript
// Instead of global activeSelection
const [activeSelection, setActiveSelection] = useState(null);
```

c) **Use context for subtrees** - Scope state to annotation feature:
```typescript
<AnnotationContext.Provider value={{ pins, highlights, handlers }}>
  <AnnotationOverlay />
</AnnotationContext.Provider>
```

### 2. **DOM Manipulation Anti-pattern**

Direct DOM queries and manipulation scattered throughout:

```typescript
// Line 120
const commentElement = document.querySelector(`[data-comment-id="${pinId}"]`);

// Line 128
const avatar = commentElement.querySelector(".comment-avatar");

// Line 138
avatar.animate(pulseKeyframes, animationOptions);
```

**Why this is problematic:**

1. **Breaks React's abstraction** - React doesn't know about these changes
2. **Timing issues** - `requestAnimationFrame` nesting indicates race conditions
3. **Hard to test** - requires real DOM
4. **Hydration risks** - client-only behavior might mismatch SSR
5. **Accessibility** - animations might not respect prefers-reduced-motion

**Better approaches:**

a) **Use refs and React state:**
```typescript
const commentRefs = useRef<Map<string, HTMLElement>>(new Map());

const handlePinClick = (pinId: string) => {
  const element = commentRefs.current.get(pinId);
  if (element) {
    element.scrollIntoView({ behavior: 'smooth', block: 'center' });
  }
};

// In Comment component
<div ref={(el) => el && commentRefs.current.set(comment._id, el)}>
```

b) **Declarative animation with state:**
```typescript
const [animatingCommentId, setAnimatingCommentId] = useState<string | null>(null);

const handlePinClick = (pinId: string) => {
  setAnimatingCommentId(pinId);
  setTimeout(() => setAnimatingCommentId(null), 3600); // 3 iterations × 1200ms
};

// In Comment component
<div className={cn(animatingCommentId === comment._id && "animate-pulse")}>
```

c) **Use Web Animations API declaratively:**
```typescript
// Custom hook for animation
function useCommentAnimation() {
  const [targetId, setTargetId] = useState<string | null>(null);

  useEffect(() => {
    if (!targetId) return;

    const element = document.querySelector(`[data-comment-id="${targetId}"]`);
    if (!element) return;

    const animation = element.animate(
      [{ transform: 'scale(1)' }, { transform: 'scale(1.1)' }, { transform: 'scale(1)' }],
      { duration: 1200, iterations: 3 }
    );

    animation.onfinish = () => setTargetId(null);
  }, [targetId]);

  return { animateComment: setTargetId };
}
```

### 3. **Missing Abstraction Layers**

The component mixes high-level business logic with low-level DOM details:

**Current structure:**
```
AnnotationOverlayClient
├── Business logic (pin creation, highlight drafting)
├── DOM manipulation (querySelector, animate)
├── Event handling (click, selection)
└── Rendering (JSX)
```

**Better layered structure:**
```
AnnotationOverlayClient (orchestration)
├── useAnnotationWorkflow (business logic)
├── useAnnotationEvents (event handling)
├── AnnotationRenderer (presentation)
└── DOMEffects (encapsulated DOM access)
```

### 4. **Testing Challenges**

Current code is hard to test because:

1. **Tightly coupled to DOM** - needs jsdom or real browser
2. **Global store dependency** - must mock entire store
3. **Side effects in component** - effects run during render
4. **No clear boundaries** - business logic mixed with UI

**Better testing structure:**

```typescript
// Business logic (pure functions) - easy to test
export function createHighlightFromSelection(
  selection: SelectionData,
  targetId: string,
  targetType: TargetType,
  pageUrl: string
): DraftHighlight {
  const rangeData = serializeRange(selection.range);
  return {
    highlightId: selection.highlightId,
    text: selection.text,
    rangeData,
    targetId,
    targetType,
    pageUrl,
  };
}

// Component (uses business logic) - easier to test
const handleBadgeClick = () => {
  if (!activeSelection) return;

  try {
    const draft = createHighlightFromSelection(
      activeSelection,
      targetId,
      targetType,
      pageUrl
    );
    setDraftHighlight(draft);
    setRightPanelView("activity");
  } catch (error) {
    // error handling
  }
};
```

---

## Recommended Migration Path

### Phase 1: Extract Custom Hooks (✅ Done)

- [x] Extract `useOrphanedPins`
- [x] Extract `usePinAnchoring`
- [x] Extract `useTextSelection`
- [x] Extract `useHighlightRestoration`
- [x] Extract `useHighlightClickHandler`
- [x] Create refactored component using hooks

**Effort:** 2-3 hours
**Risk:** Low
**Benefit:** Immediate readability improvement

### Phase 2: Test the Extracted Hooks

- [ ] Write tests for `useOrphanedPins`
- [ ] Write tests for `useTextSelection` (complex - priority)
- [ ] Write tests for `useHighlightClickHandler`
- [ ] Verify refactored component works identically to original

**Effort:** 4-6 hours
**Risk:** Low
**Benefit:** Confidence in refactor, prevent regressions

### Phase 3: Extract Business Logic

- [ ] Extract `createHighlightFromSelection` pure function
- [ ] Extract `scrollToComment` utility
- [ ] Extract `animateComment` utility
- [ ] Write tests for pure functions

**Effort:** 3-4 hours
**Risk:** Low
**Benefit:** Testable business logic

### Phase 4: Split Container/Presentational (Optional)

- [ ] Create `PinOverlay` component
- [ ] Create `HighlightManager` component
- [ ] Refactor `AnnotationOverlayClient` to orchestrator
- [ ] Update tests

**Effort:** 6-8 hours
**Risk:** Medium
**Benefit:** Better separation of concerns

### Phase 5: Address DOM Manipulation (Optional)

- [ ] Replace `document.querySelector` with refs
- [ ] Extract animation logic to custom hook
- [ ] Make animations declarative with state
- [ ] Add `prefers-reduced-motion` support

**Effort:** 4-6 hours
**Risk:** Medium
**Benefit:** More React-idiomatic, better a11y

### Phase 6: Store Refactoring (Optional, if needed)

- [ ] Analyze store usage across app
- [ ] Split `useCommentsStore` into smaller stores
- [ ] Use context for scoped state
- [ ] Update components to use new stores

**Effort:** 8-12 hours
**Risk:** High
**Benefit:** Better state management architecture

---

## Decision Matrix

| Strategy | Complexity | Risk | Immediate Value | Long-term Value | Recommended |
|----------|-----------|------|-----------------|-----------------|-------------|
| Extract custom hooks | Low | Low | ⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ **DO NOW** |
| Test extracted hooks | Low | Low | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ **DO NEXT** |
| Extract business logic | Low | Low | ⭐⭐⭐ | ⭐⭐⭐⭐ | ✅ **DO SOON** |
| Container/Presentational split | Medium | Medium | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⚠️ **CONSIDER** |
| Fix DOM manipulation | Medium | Medium | ⭐⭐ | ⭐⭐⭐⭐ | ⚠️ **CONSIDER** |
| State machine | High | High | ⭐⭐ | ⭐⭐⭐⭐⭐ | ❌ **LATER** |
| Event bus | Medium | Medium | ⭐⭐ | ⭐⭐ | ❌ **AVOID** |
| Store refactoring | High | High | ⭐⭐ | ⭐⭐⭐⭐ | ❌ **ONLY IF NEEDED** |

---

## Conclusion

The `AnnotationOverlayClient` component suffers from classic **God Component** anti-pattern with too many responsibilities crammed into one file.

**Your instinct was correct** - the multiple useEffect hooks are a code smell indicating the component is doing too much.

**Recommended action:**
1. ✅ **Use the refactored version** with extracted hooks (done)
2. ✅ **Test the extracted hooks** to ensure they work correctly
3. ⚠️ **Consider** extracting business logic to pure functions
4. ⚠️ **Evaluate** if further splitting (container/presentational) is worth the effort
5. ❌ **Avoid** over-engineering with state machines or event buses unless complexity grows significantly

**Key principle:** Start with the smallest refactor that provides value (custom hooks), then iterate based on needs. Don't refactor for refactoring's sake.

---

## Files Created

1. `hooks/useOrphanedPins.ts` - Orphaned pin detection
2. `hooks/useTextSelection.ts` - Text selection handling
3. `hooks/useHighlightRestoration.ts` - Highlight persistence
4. `hooks/usePinAnchoring.ts` - Pin anchor positioning
5. `hooks/useHighlightClickHandler.ts` - Event delegation
6. `AnnotationOverlayClient.refactored.tsx` - Refactored component using hooks

**Next steps:** Replace `AnnotationOverlayClient.tsx` with refactored version after testing confirms identical behavior.

# Pin/Highlight Split Architecture Comparison

**Date:** 2025-11-07
**Decision:** Split AnnotationOverlayClient into separate Pin and Highlight overlays

---

## Architecture Comparison

### Original: Monolithic Component (360 lines)

```
AnnotationOverlayClient.tsx (360 lines)
├── Pin logic (40%)
│   ├── Pin creation mode
│   ├── Orphan detection
│   ├── Anchor positioning
│   └── Pin click handling
├── Highlight logic (40%)
│   ├── Text selection capture
│   ├── Highlight creation
│   ├── Highlight restoration
│   └── Highlight click handling
└── Shared orchestration (20%)
    ├── Wrapper ref management
    ├── Panel coordination
    └── Cross-feature interactions
```

**Problems:**
- Mixed responsibilities
- Hard to reason about interactions
- Difficult to test features in isolation
- Large file with many dependencies

---

### Refactored v1: Custom Hooks (200 lines)

```
AnnotationOverlayClient.refactored.tsx (200 lines)
├── useOrphanedPins()
├── usePinAnchoring()
├── useTextSelection()
├── useHighlightRestoration()
├── useHighlightClickHandler()
├── Pin event handlers
├── Highlight event handlers
└── JSX rendering
```

**Improvements:**
✅ Reduced lines from 360 → 200
✅ Extracted reusable hooks
✅ Better organization

**Remaining issues:**
❌ Still mixes pin and highlight concerns
❌ Single component still owns both features
❌ Can't easily test pins without highlights

---

### Refactored v2: Split Components (Recommended) ✨

```
AnnotationOverlayClient.split.tsx (50 lines - thin orchestrator)
├── Wrapper ref management
├── Pin creation mode CSS class
└── Renders:
    ├── PinOverlay
    └── HighlightOverlay

PinOverlay.tsx (100 lines - focused on pins)
├── useOrphanedPins()
├── usePinAnchoring()
├── usePinCreationMode()
├── Pin click handler
└── Renders: PinMarker[]

HighlightOverlay.tsx (100 lines - focused on highlights)
├── useTextSelection()
├── useHighlightRestoration()
├── useHighlightClickHandler()
├── Highlight click handler
└── Renders: SelectionBadge
```

**Benefits:**
✅ **Clear separation of concerns** - pins and highlights are completely separate
✅ **Independent testability** - test each feature in isolation
✅ **Smaller files** - 50+100+100 lines vs 360 lines
✅ **Easier to understand** - each file has single focus
✅ **Better maintainability** - changes to pins don't affect highlights
✅ **Reusability** - can use PinOverlay without HighlightOverlay
✅ **Parallel development** - different devs can work on each feature

---

## Feature Coupling Analysis

### Cross-Feature Interactions

Only 2 interactions between pins and highlights:

1. **Pin creation disables text selection**
   ```typescript
   // HighlightOverlay respects pin mode via store
   useTextSelection({ isCreatingPin }); // from global store
   ```

2. **Clicking a pin clears active highlight**
   ```typescript
   // PinOverlay clears highlight via store
   const handlePinClick = (pinId) => {
     setActivePin(pinId);
     setActiveHighlight(null); // Cross-feature coordination
   };
   ```

**Key insight:** Both interactions are **mediated through the global store**, not direct component coupling. This means pins and highlights are already loosely coupled - they just need to be separated!

### Coordination via Global Store

```
Global Store (useCommentsStore)
├── Pin state
│   ├── activePin
│   ├── isCreatingPin
│   └── draftPin
├── Highlight state
│   ├── activeSelection
│   ├── draftHighlight
│   └── activeHighlight
└── Shared actions
    ├── setActivePin()
    ├── setActiveHighlight()
    └── ...

PinOverlay ────────────▶ Store ◀──────────── HighlightOverlay
     Uses pin state              Uses highlight state
     Can clear highlight         Respects isCreatingPin
```

The store acts as a **mediator pattern**, allowing components to coordinate without knowing about each other.

---

## Detailed Component Responsibilities

### AnnotationOverlayClient (Orchestrator)

**Responsibilities:**
- Provide wrapper ref for both overlays
- Apply CSS class for pin creation mode
- Render children with overlay layers

**Size:** ~50 lines

**Dependencies:**
- useCommentsStore (only `isCreatingPin` for CSS class)

**Reasoning:** Thin orchestrator with minimal logic. Just composes the two overlay layers.

---

### PinOverlay (Pin Feature)

**Responsibilities:**
- Derive pin list (server + draft pins)
- Detect orphaned pins
- Apply CSS anchor positioning
- Handle pin creation workflow
- Handle pin click events
- Scroll to and animate comments

**Size:** ~100 lines

**Dependencies:**
- useOrphanedPins
- usePinAnchoring
- usePinCreationMode
- useCommentsStore (pin state only)
- usePanelsStore

**State:**
```typescript
const {
  activePin,
  isCreatingPin,
  draftPin,
  setActivePin,
  setCreatingPin,
  setDraftPin,
  setActiveHighlight  // Only for coordination
} = useCommentsStore();
```

**Cross-feature coordination:**
- Clears `activeHighlight` when pin clicked (loose coupling via store)

---

### HighlightOverlay (Highlight Feature)

**Responsibilities:**
- Capture text selection
- Create temporary highlights
- Restore persisted highlights
- Handle highlight click events
- Show selection badge
- Scroll to comments

**Size:** ~100 lines

**Dependencies:**
- useTextSelection
- useHighlightRestoration
- useHighlightClickHandler
- useCommentsStore (highlight state only)
- usePanelsStore

**State:**
```typescript
const {
  isCreatingPin,       // Only for coordination (disable selection)
  activeSelection,
  draftHighlight,
  setDraftHighlight,
  setActiveHighlight
} = useCommentsStore();
```

**Cross-feature coordination:**
- Respects `isCreatingPin` to disable selection (loose coupling via store)

---

## Testing Strategy Comparison

### Original Component Testing

```typescript
describe('AnnotationOverlayClient', () => {
  it('should handle pin creation AND highlight selection', () => {
    // Need to set up:
    // - Pin mocks
    // - Highlight mocks
    // - Store state
    // - DOM structure
    // - Event listeners
    // Test becomes complex and brittle
  });
});
```

**Problems:**
- Must mock entire feature set
- Hard to isolate what's being tested
- Slow (entire component + all hooks)

---

### Split Component Testing

```typescript
// Test pins in isolation
describe('PinOverlay', () => {
  it('should detect orphaned pins', () => {
    const comments = [createMockPinComment()];
    render(<PinOverlay initialComments={comments} {...props} />);
    // Only test pin behavior
  });

  it('should clear highlight when pin clicked', () => {
    const { result } = renderHook(() => useCommentsStore());
    // Test cross-feature coordination
  });
});

// Test highlights in isolation
describe('HighlightOverlay', () => {
  it('should disable selection during pin creation', () => {
    const { result } = renderHook(() => useCommentsStore());
    act(() => result.current.setCreatingPin(true));
    // Only test highlight behavior
  });

  it('should restore highlights from comments', () => {
    const comments = [createMockHighlightComment()];
    render(<HighlightOverlay initialComments={comments} {...props} />);
    // Only test highlight restoration
  });
});
```

**Benefits:**
✅ **Focused tests** - test one feature at a time
✅ **Faster** - smaller components render faster
✅ **Easier mocking** - only mock what the feature needs
✅ **Better coverage** - easier to test edge cases

---

## Code Organization Comparison

### Before (360 lines in one file)

```
AnnotationOverlayClient.tsx
├── Lines 1-50:   Imports & setup
├── Lines 51-77:  Orphaned pin detection
├── Lines 78-108: Pin anchoring
├── Lines 109-173: Pin click handling
├── Lines 174-240: Text selection (60 lines!)
├── Lines 241-270: Badge click handling
├── Lines 271-300: Highlight restoration
├── Lines 301-327: Highlight click delegation
└── Lines 328-359: JSX rendering
```

**Navigation difficulty:**
- Scrolling through 360 lines to find specific logic
- Hard to see feature boundaries
- Context switching between pin and highlight code

---

### After (3 files, ~250 total lines)

```
AnnotationOverlayClient.split.tsx (50 lines)
└── Simple orchestrator - easy overview

PinOverlay.tsx (100 lines)
├── All pin logic in one place
└── Clear feature boundary

HighlightOverlay.tsx (100 lines)
├── All highlight logic in one place
└── Clear feature boundary
```

**Navigation benefit:**
- Want to work on pins? → Open `PinOverlay.tsx`
- Want to work on highlights? → Open `HighlightOverlay.tsx`
- Want to see big picture? → Open `AnnotationOverlayClient.split.tsx`

---

## Future Extensibility

### Adding a New Annotation Type (e.g., DrawingOverlay)

**With monolithic component:**
```typescript
// AnnotationOverlayClient.tsx becomes even larger
export function AnnotationOverlayClient({ ... }) {
  // Add drawing state
  const { activeDrawing, isDrawing, ... } = useCommentsStore();

  // Add drawing hooks
  useDrawingCapture();
  useDrawingRestoration();

  // Add drawing handlers
  const handleDrawingClick = ...;

  // Already 360 lines + 100 more for drawings = 460 lines!

  return <div>
    {/* Pins */}
    {/* Highlights */}
    {/* Drawings */}
  </div>;
}
```

❌ Component keeps growing
❌ More complex to test
❌ Harder to maintain

---

**With split architecture:**
```typescript
// Create new DrawingOverlay.tsx (100 lines)
export function DrawingOverlay({ ... }) {
  useDrawingCapture();
  useDrawingRestoration();

  return <DrawingCanvas />;
}

// Update orchestrator (60 lines)
export function AnnotationOverlayClient({ ... }) {
  return (
    <div ref={wrapperRef}>
      {children}
      <PinOverlay {...sharedProps} />
      <HighlightOverlay {...sharedProps} />
      <DrawingOverlay {...sharedProps} />  {/* New! */}
    </div>
  );
}
```

✅ Each feature stays ~100 lines
✅ Easy to add/remove features
✅ Clear composition pattern

---

## Performance Considerations

### Re-render Isolation

**Monolithic component:**
```typescript
// Any state change re-renders EVERYTHING
setActivePin(id);  // Re-renders pin AND highlight logic
setActiveHighlight(id);  // Re-renders pin AND highlight logic
```

**Split components:**
```typescript
// PinOverlay only re-renders on pin state changes
const { activePin, draftPin } = useCommentsStore();  // Subset

// HighlightOverlay only re-renders on highlight state changes
const { activeSelection, draftHighlight } = useCommentsStore();  // Subset
```

**With zustand's selector optimization:**
```typescript
// Even better - subscribe to specific state slices
const activePin = useCommentsStore(state => state.activePin);
```

✅ **Better performance** - less unnecessary re-renders
✅ **Easier to optimize** - can memoize components independently

---

## Migration Path

### Step 1: Create new files (✅ Done)
- [x] Create `PinOverlay.tsx`
- [x] Create `HighlightOverlay.tsx`
- [x] Create `AnnotationOverlayClient.split.tsx`

### Step 2: Verify functionality
- [ ] Test pin creation workflow
- [ ] Test highlight creation workflow
- [ ] Test cross-feature interactions (pin click clears highlight)
- [ ] Test text selection disabled during pin creation
- [ ] Verify all existing tests still pass

### Step 3: Rename files
```bash
mv AnnotationOverlayClient.tsx AnnotationOverlayClient.old.tsx
mv AnnotationOverlayClient.split.tsx AnnotationOverlayClient.tsx
```

### Step 4: Clean up
- [ ] Delete old file after confirming everything works
- [ ] Update documentation
- [ ] Write tests for new components

---

## Decision Criteria: When to Split?

Use this framework to decide if splitting is worth it:

### ✅ Split when:
- Component has 2+ distinct feature domains (pins, highlights)
- Features can work independently
- Features have minimal coupling (1-2 interactions)
- File exceeds 200 lines
- Team members work on different features
- Testing becomes difficult due to complexity

### ❌ Don't split when:
- Features are tightly coupled (many interactions)
- Component is already small (<150 lines)
- Features share significant rendering logic
- Splitting would create excessive prop drilling
- Team is unfamiliar with composition patterns

### This case: ✅ CLEAR YES
- ✅ 2 distinct features (pins, highlights)
- ✅ Features work independently
- ✅ Only 2 coupling points (via store)
- ✅ 360 lines → 50+100+100
- ✅ Easier testing
- ✅ Better maintainability

---

## Recommendation

**Use the split architecture (AnnotationOverlayClient.split.tsx).**

### Why?

1. **Separation of Concerns** - Pins and highlights are separate annotation types with different UX patterns
2. **Minimal Coupling** - Only 2 interactions, both mediated through store (loose coupling)
3. **Better Testing** - Test pins and highlights independently
4. **Clearer Code** - Each file has single focus, easier to understand
5. **Future-Proof** - Easy to add new annotation types (drawings, comments, etc.)
6. **Better Performance** - Reduced unnecessary re-renders
7. **Easier Maintenance** - Changes to pins won't break highlights

### File Structure

```
_components/comments/
├── AnnotationOverlayClient.tsx (50 lines - orchestrator)
├── PinOverlay.tsx (100 lines - pin feature)
├── HighlightOverlay.tsx (100 lines - highlight feature)
├── hooks/
│   ├── useOrphanedPins.ts
│   ├── usePinAnchoring.ts
│   ├── usePinCreationMode.ts
│   ├── useTextSelection.ts
│   ├── useHighlightRestoration.ts
│   └── useHighlightClickHandler.ts
├── PinMarker.tsx
└── SelectionBadge.tsx
```

**Total:** 250 lines across 3 components vs 360 lines in one component

**Complexity reduction:** ~30% less code, infinitely better organization

---

## Conclusion

Splitting `AnnotationOverlayClient` into `PinOverlay` and `HighlightOverlay` is a **clear win**:

- ✅ **Less code** (250 vs 360 lines)
- ✅ **Better organization** (3 focused files vs 1 monolith)
- ✅ **Easier to test** (isolated features)
- ✅ **More maintainable** (clear boundaries)
- ✅ **More extensible** (add features easily)
- ✅ **Better performance** (reduced re-renders)

The features are already loosely coupled through the store - this refactor just makes that architectural intent explicit in the code structure.

**Next step:** Replace `AnnotationOverlayClient.tsx` with `AnnotationOverlayClient.split.tsx` after verification testing.

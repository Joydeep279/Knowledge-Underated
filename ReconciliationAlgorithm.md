# Virtual DOM and Reconciliation Algorithm: A Deep Systems Perspective

## Executive Summary: The Problem Space

Before we architect the solution, let's understand the fundamental constraint: **DOM manipulation is orders of magnitude slower than in-memory JavaScript operations**.

When you call `element.appendChild()` or modify `innerHTML`, you're not just updating a JavaScript object — you're triggering a cascade through multiple subsystems:

```
JavaScript Call
    ↓
DOM API Boundary (C++ binding layer)
    ↓
Style Recalculation (CSSOM)
    ↓
Layout/Reflow (Geometry computation)
    ↓
Paint (Rasterization)
    ↓
Composite Layers
    ↓
GPU Upload
```

Each DOM mutation potentially invalidates cached layout information, forcing the browser to recalculate positions, sizes, and visual properties. This is the **fundamental tax** we're trying to minimize.

---

## Part I: Virtual DOM — The In-Memory Shadow Tree

### 1.1 Core Concept: Isomorphic Tree Representation

The Virtual DOM is a **lightweight JavaScript object graph** that mirrors the actual DOM structure but lives entirely in the JavaScript heap. It's a *shadow representation* that can be manipulated at memory speed.

**Memory Layout:**
```javascript
// Physical structure in V8 heap
VirtualNode = {
  type: String,        // 8 bytes (pointer to interned string)
  props: Object,       // 8 bytes (pointer to hidden class)
  children: Array,     // 8 bytes (pointer to array)
  key: String | null,  // 8 bytes
  ref: Object | null,  // 8 bytes
  _owner: Fiber,       // 8 bytes (React 16+)
}
// Approximate size: 48 bytes + props payload + children array
```

Compare this to a real DOM node:
```cpp
// Simplified from Chromium's Node.h
class Node : public EventTarget {
  Document* document_;              // 8 bytes
  Node* parent_;                    // 8 bytes
  Node* previous_sibling_;          // 8 bytes
  Node* next_sibling_;              // 8 bytes
  Node* first_child_;               // 8 bytes
  Node* last_child_;                // 8 bytes
  NodeRareData* rare_data_;         // 8 bytes
  uint32_t node_flags_;             // 4 bytes
  // + StyleResolver data
  // + Layout object pointer
  // + Event listeners map
  // + Accessibility data
  // Approximate size: 500+ bytes per node
}
```

**Key Insight:** A Virtual DOM node is ~10x smaller in memory footprint and requires zero browser subsystem coordination.

### 1.2 The Virtual Node Factory

Let's examine React's `createElement` — the entry point for Virtual DOM construction:

```javascript
// Simplified from react/packages/react/src/ReactElement.js

const RESERVED_PROPS = {
  key: true,
  ref: true,
  __self: true,
  __source: true,
};

function createElement(type, config, children) {
  let propName;
  const props = {};

  let key = null;
  let ref = null;

  // Extract reserved props
  if (config != null) {
    if (hasValidRef(config)) {
      ref = config.ref;
    }
    if (hasValidKey(config)) {
      key = '' + config.key; // Coerce to string
    }

    // Copy props (excluding reserved)
    for (propName in config) {
      if (
        hasOwnProperty.call(config, propName) &&
        !RESERVED_PROPS.hasOwnProperty(propName)
      ) {
        props[propName] = config[propName];
      }
    }
  }

  // Handle children (variadic after position 2)
  const childrenLength = arguments.length - 2;
  if (childrenLength === 1) {
    props.children = children;
  } else if (childrenLength > 1) {
    const childArray = Array(childrenLength);
    for (let i = 0; i < childrenLength; i++) {
      childArray[i] = arguments[i + 2];
    }
    props.children = childArray;
  }

  // Apply default props (for class components)
  if (type && type.defaultProps) {
    const defaultProps = type.defaultProps;
    for (propName in defaultProps) {
      if (props[propName] === undefined) {
        props[propName] = defaultProps[propName];
      }
    }
  }

  return ReactElement(
    type,
    key,
    ref,
    ReactCurrentOwner.current, // Fiber that created this element
    props,
  );
}

function ReactElement(type, key, ref, owner, props) {
  const element = {
    // This tag uniquely identifies this as a React Element
    $$typeof: REACT_ELEMENT_TYPE, // Symbol.for('react.element')

    type: type,      // 'div', Component function/class, or Symbol
    key: key,
    ref: ref,
    props: props,

    _owner: owner,   // The Fiber that owns this element
  };

  // Development-only validation metadata
  if (__DEV__) {
    element._store = {};
    element._self = null;
    element._source = null;
  }

  return Object.freeze(element); // Immutable in production
}
```

**Architecture Notes:**

1. **$$typeof Symbol**: Protects against XSS by ensuring untrusted JSON cannot be deserialized into React elements (Symbols can't be serialized).

2. **key Coercion**: Always converted to string for consistent identity across renders.

3. **Owner Tracking**: The `_owner` field links this element to the Fiber that created it, enabling the reconciliation algorithm to track ownership chains.

4. **Immutability**: Elements are frozen in production, enforcing the principle that Virtual DOM is immutable snapshot data.

### 1.3 JSX Transformation Pipeline

JSX is syntactic sugar that Babel transforms into `createElement` calls:

```javascript
// Source JSX
<div className="container">
  <h1>Hello {name}</h1>
  <p>Count: {count}</p>
</div>

// Babel output (classic transform)
React.createElement(
  "div",
  { className: "container" },
  React.createElement("h1", null, "Hello ", name),
  React.createElement("p", null, "Count: ", count)
);

// New JSX transform (React 17+)
import { jsx as _jsx } from "react/jsx-runtime";
_jsx("div", {
  className: "container",
  children: [
    _jsx("h1", { children: ["Hello ", name] }),
    _jsx("p", { children: ["Count: ", count] })
  ]
});
```

The new transform:
- No longer requires `React` in scope
- Generates slightly more efficient bytecode
- Enables better tree-shaking

---

## Part II: The Reconciliation Problem

### 2.1 Formal Problem Statement

Given:
- **Old Tree** `T₁`: Virtual DOM from previous render
- **New Tree** `T₂`: Virtual DOM from current render

Find:
- **Minimal edit sequence** `E = [e₁, e₂, ..., eₙ]` such that applying `E` to the real DOM transforms it from state `D₁` (corresponding to `T₁`) to state `D₂` (corresponding to `T₂`)

**Constraint:** Minimize both:
1. Number of DOM operations
2. Computational complexity of finding `E`

### 2.2 The NP-Complete Nature

The **tree edit distance** problem (finding the minimum number of operations to transform one tree into another) is **NP-Complete** for unordered trees.

**Proof sketch:**
- Decision version: "Does there exist a transformation with ≤ k operations?"
- Reduction from SET COVER
- Even for ordered trees, optimal solutions require O(n³) time (Zhang-Shasha algorithm)

For a UI with 1,000 nodes:
- O(n³) = 1,000,000,000 operations per frame
- At 60fps, we have ~16ms per frame
- Clearly intractable.

### 2.3 React's Heuristic Trade-off

React makes **two key assumptions** that reduce complexity to **O(n)**:

#### Assumption 1: Cross-Level Movement is Rare
```
Old Tree:          New Tree:
   A                  A
  / \                / \
 B   C      ≠       C   D
             (B moved under C)

React's decision: IGNORE cross-level moves
Instead: DELETE B, CREATE new B under C
```

#### Assumption 2: Element Type Indicates Structural Equivalence
```
Old:  <div><span>Text</span></div>
New:  <div><p>Text</p></div>

React's decision: <span> → <p> is a type change
Action: DELETE span subtree, CREATE p subtree
(Even though we could theoretically just change the tag)
```

These heuristics allow **single-pass linear traversal** but sacrifice optimality. In practice, they're correct >99% of the time for typical UI patterns.

---

## Part III: The Reconciliation Algorithm — Deep Dive

### 3.1 The Fiber Architecture (React 16+)

Before React 16, reconciliation was synchronous and could block the main thread. Fiber introduced **incremental reconciliation**.

**Fiber Node Structure:**
```javascript
// From react-reconciler/src/ReactInternalTypes.js

type Fiber = {
  // Identity
  tag: WorkTag,              // FunctionComponent, ClassComponent, HostComponent, etc.
  key: null | string,
  elementType: any,          // The resolved type (function/class/string)
  type: any,                 // The working type (can differ during lazy loading)

  // Relationships
  return: Fiber | null,      // Parent fiber
  child: Fiber | null,       // First child
  sibling: Fiber | null,     // Next sibling
  index: number,             // Position in parent's children

  // Reconciliation state
  alternate: Fiber | null,   // The fiber from the other tree (work-in-progress ↔ current)

  // Input (props/state)
  pendingProps: any,         // Props from this render
  memoizedProps: any,        // Props from last render
  memoizedState: any,        // State from last render
  updateQueue: mixed,        // Queue of state updates and callbacks

  // Effects
  flags: Flags,              // Side-effect tags (Placement, Update, Deletion, etc.)
  subtreeFlags: Flags,       // Aggregate of descendant flags
  deletions: Array<Fiber> | null,

  // Performance
  lanes: Lanes,              // Priority lanes for scheduling
  childLanes: Lanes,

  // Double buffering
  stateNode: any,            // Reference to the actual DOM node / class instance

  // Profiling (DEV)
  actualDuration: number,
  actualStartTime: number,
  selfBaseDuration: number,
  treeBaseDuration: number,
};
```

**The Double-Buffer Pattern:**

React maintains **two fiber trees**:
1. **current**: The tree currently displayed (committed to DOM)
2. **workInProgress**: The tree being built for the next frame

```
┌─────────────┐         ┌─────────────┐
│   current   │←────────│    root     │
│    tree     │ current │   (host)    │
└─────────────┘         └─────────────┘
                               │ alternate
                               ↓
                        ┌─────────────┐
                        │work-in-     │
                        │ progress    │
                        └─────────────┘
```

After reconciliation completes, we swap pointers:
```javascript
root.current = finishedWork;
finishedWork.alternate = oldCurrent;
```

This is **double buffering** — the same technique used in graphics programming to prevent tearing.

### 3.2 The Reconciliation Loop

**Entry Point:**
```javascript
// Simplified from react-reconciler/src/ReactFiberWorkLoop.js

function performUnitOfWork(unitOfWork: Fiber): Fiber | null {
  const current = unitOfWork.alternate;

  let next;
  if (enableProfilerTimer && (unitOfWork.mode & ProfileMode) !== NoMode) {
    startProfilerTimer(unitOfWork);
    next = beginWork(current, unitOfWork, renderLanes);
    stopProfilerTimerIfRunningAndRecordDelta(unitOfWork);
  } else {
    next = beginWork(current, unitOfWork, renderLanes);
  }

  unitOfWork.memoizedProps = unitOfWork.pendingProps;

  if (next === null) {
    // No child, so we're done with this fiber. Move to completion phase.
    completeUnitOfWork(unitOfWork);
  }

  return next;
}

function workLoopSync() {
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}

function workLoopConcurrent() {
  // Yield to browser if we've run out of time
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}
```

**Key Mechanism:** The loop can be **interrupted** (concurrent mode) by checking `shouldYield()`, which queries the browser's scheduler API to check if the frame deadline is approaching.

### 3.3 The beginWork Phase

This is where reconciliation logic lives:

```javascript
// Simplified from react-reconciler/src/ReactFiberBeginWork.js

function beginWork(
  current: Fiber | null,
  workInProgress: Fiber,
  renderLanes: Lanes,
): Fiber | null {

  // Optimization: If props and state haven't changed, bail out
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      didReceiveUpdate = true;
    } else {
      // Props are equal. Check if there's a pending update.
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
        current,
        renderLanes,
      );
      if (!hasScheduledUpdateOrContext) {
        // No updates. Reuse the existing fiber.
        return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
      }
    }
  }

  // Clear flags from previous render
  workInProgress.lanes = NoLanes;

  // Reconciliation logic based on fiber type
  switch (workInProgress.tag) {
    case FunctionComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps = workInProgress.elementType === Component
        ? unresolvedProps
        : resolveDefaultProps(Component, unresolvedProps);
      return updateFunctionComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case ClassComponent: {
      const Component = workInProgress.type;
      const unresolvedProps = workInProgress.pendingProps;
      const resolvedProps = workInProgress.elementType === Component
        ? unresolvedProps
        : resolveDefaultProps(Component, unresolvedProps);
      return updateClassComponent(
        current,
        workInProgress,
        Component,
        resolvedProps,
        renderLanes,
      );
    }
    case HostComponent: {
      // <div>, <span>, etc.
      return updateHostComponent(current, workInProgress, renderLanes);
    }
    case HostText: {
      // Text nodes
      return updateHostText(current, workInProgress);
    }
    // ... 20+ other cases
  }
}
```

### 3.4 The Core Diffing Algorithm: reconcileChildren

When we encounter a **host component** (like `<div>`), we need to reconcile its children:

```javascript
// From react-reconciler/src/ReactChildFiber.js

function reconcileChildFibers(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChild: any,
  lanes: Lanes,
): Fiber | null {

  // Handle different child types
  const isObject = typeof newChild === 'object' && newChild !== null;

  if (isObject) {
    switch (newChild.$$typeof) {
      case REACT_ELEMENT_TYPE:
        return placeSingleChild(
          reconcileSingleElement(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes,
          ),
        );
      case REACT_PORTAL_TYPE:
        return placeSingleChild(
          reconcileSinglePortal(
            returnFiber,
            currentFirstChild,
            newChild,
            lanes,
          ),
        );
      case REACT_LAZY_TYPE:
        // Lazy component, need to resolve first
        const payload = newChild._payload;
        const init = newChild._init;
        return reconcileChildFibers(
          returnFiber,
          currentFirstChild,
          init(payload),
          lanes,
        );
    }
  }

  if (typeof newChild === 'string' || typeof newChild === 'number') {
    return placeSingleChild(
      reconcileSingleTextNode(
        returnFiber,
        currentFirstChild,
        '' + newChild,
        lanes,
      ),
    );
  }

  if (isArray(newChild)) {
    return reconcileChildrenArray(
      returnFiber,
      currentFirstChild,
      newChild,
      lanes,
    );
  }

  // Unsupported type, delete existing children
  return deleteRemainingChildren(returnFiber, currentFirstChild);
}
```

### 3.5 Single Element Reconciliation

**Scenario:** New child is a single React element.

```javascript
function reconcileSingleElement(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  element: ReactElement,
  lanes: Lanes,
): Fiber {
  const key = element.key;
  let child = currentFirstChild;

  // Scan through existing children
  while (child !== null) {
    // STEP 1: Check if keys match
    if (child.key === key) {
      const elementType = element.type;

      // STEP 2: Check if types match
      if (child.elementType === elementType) {
        // MATCH FOUND: Reuse this fiber
        deleteRemainingChildren(returnFiber, child.sibling);

        const existing = useFiber(child, element.props);
        existing.ref = coerceRef(returnFiber, child, element);
        existing.return = returnFiber;

        return existing;
      }

      // Key matches but type differs: Delete this and all siblings
      deleteRemainingChildren(returnFiber, child);
      break;
    } else {
      // Key mismatch: Delete this child and continue
      deleteChild(returnFiber, child);
    }

    child = child.sibling;
  }

  // NO MATCH: Create new fiber
  const created = createFiberFromElement(element, returnFiber.mode, lanes);
  created.ref = coerceRef(returnFiber, currentFirstChild, element);
  created.return = returnFiber;
  return created;
}
```

**Decision Tree:**
```
Does key match?
├─ NO  → Delete old child, continue to next sibling
└─ YES → Does type match?
         ├─ NO  → Delete this + all siblings, create new
         └─ YES → REUSE fiber, update props
```

### 3.6 Array Reconciliation — The Complex Case

This is where React's O(n) optimization shines. Reconciling arrays is **the most common and performance-critical path**.

```javascript
function reconcileChildrenArray(
  returnFiber: Fiber,
  currentFirstChild: Fiber | null,
  newChildren: Array<any>,
  lanes: Lanes,
): Fiber | null {

  let resultingFirstChild: Fiber | null = null;
  let previousNewFiber: Fiber | null = null;

  let oldFiber = currentFirstChild;
  let lastPlacedIndex = 0;
  let newIdx = 0;
  let nextOldFiber = null;

  // ===================================================================
  // PASS 1: Handle updates in-place (common prefix)
  // ===================================================================
  // Walk both lists in parallel while keys/indices match
  for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
    if (oldFiber.index > newIdx) {
      nextOldFiber = oldFiber;
      oldFiber = null;
    } else {
      nextOldFiber = oldFiber.sibling;
    }

    const newFiber = updateSlot(
      returnFiber,
      oldFiber,
      newChildren[newIdx],
      lanes,
    );

    if (newFiber === null) {
      // Key mismatch. Exit this pass.
      if (oldFiber === null) {
        oldFiber = nextOldFiber;
      }
      break;
    }

    if (shouldTrackSideEffects) {
      if (oldFiber && newFiber.alternate === null) {
        // New fiber didn't reuse old one (type mismatch)
        deleteChild(returnFiber, oldFiber);
      }
    }

    lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);

    // Build linked list
    if (previousNewFiber === null) {
      resultingFirstChild = newFiber;
    } else {
      previousNewFiber.sibling = newFiber;
    }
    previousNewFiber = newFiber;
    oldFiber = nextOldFiber;
  }

  // ===================================================================
  // EARLY EXIT: If new list is exhausted, delete remaining old nodes
  // ===================================================================
  if (newIdx === newChildren.length) {
    deleteRemainingChildren(returnFiber, oldFiber);
    return resultingFirstChild;
  }

  // ===================================================================
  // EARLY EXIT: If old list is exhausted, create remaining new nodes
  // ===================================================================
  if (oldFiber === null) {
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
      if (newFiber === null) continue;

      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);

      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
    return resultingFirstChild;
  }

  // ===================================================================
  // PASS 2: Handle moves/insertions/deletions (divergent lists)
  // ===================================================================
  // Build a map: key -> oldFiber
  const existingChildren = mapRemainingChildren(returnFiber, oldFiber);

  for (; newIdx < newChildren.length; newIdx++) {
    const newFiber = updateFromMap(
      existingChildren,
      returnFiber,
      newIdx,
      newChildren[newIdx],
      lanes,
    );

    if (newFiber !== null) {
      if (shouldTrackSideEffects) {
        if (newFiber.alternate !== null) {
          // Reused an old fiber, remove from map
          existingChildren.delete(
            newFiber.key === null ? newIdx : newFiber.key,
          );
        }
      }

      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);

      if (previousNewFiber === null) {
        resultingFirstChild = newFiber;
      } else {
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
    }
  }

  // ===================================================================
  // CLEANUP: Delete any remaining old fibers not reused
  // ===================================================================
  if (shouldTrackSideEffects) {
    existingChildren.forEach(child => deleteChild(returnFiber, child));
  }

  return resultingFirstChild;
}
```

**Algorithm Analysis:**

**Pass 1: O(n) — Common Prefix**
```
Old: [A, B, C, D]
New: [A, B, X, Y]

Walk in parallel:
- A matches → Update
- B matches → Update
- C vs X: Key mismatch → Break
```

**Pass 2: O(n) — Build HashMap and Reconcile**
```
Remaining old: [C, D] → Map { C: fiberC, D: fiberD }
Remaining new: [X, Y]

For X: Not in map → Create new
For Y: Not in map → Create new
Leftover in map: [C, D] → Delete
```

**Placement Logic:**
```javascript
function placeChild(
  newFiber: Fiber,
  lastPlacedIndex: number,
  newIndex: number,
): number {
  newFiber.index = newIndex;

  if (!shouldTrackSideEffects) {
    // Initial render, no need to track moves
    return lastPlacedIndex;
  }

  const current = newFiber.alternate;
  if (current !== null) {
    const oldIndex = current.index;
    if (oldIndex < lastPlacedIndex) {
      // This fiber moved backwards. Mark for MOVE operation.
      newFiber.flags |= Placement;
      return lastPlacedIndex;
    } else {
      // This fiber is in the correct position or moved forward
      return oldIndex;
    }
  } else {
    // New fiber. Mark for INSERT operation.
    newFiber.flags |= Placement;
    return lastPlacedIndex;
  }
}
```

**The Placement Heuristic:**

React tracks `lastPlacedIndex` — the highest index of a reused fiber. If a reused fiber has an `oldIndex < lastPlacedIndex`, it must have moved backward, so it gets the `Placement` flag.

Example:
```
Old: [A(0), B(1), C(2), D(3)]
New: [D(0), A(1), B(2), C(3)]

Processing:
- D: oldIndex=3, lastPlaced=0 → 3 >= 0, keep position, lastPlaced=3
- A: oldIndex=0, lastPlaced=3 → 0 < 3, MOVE, lastPlaced=3
- B: oldIndex=1, lastPlaced=3 → 1 < 3, MOVE, lastPlaced=3
- C: oldIndex=2, lastPlaced=3 → 2 < 3, MOVE, lastPlaced=3

Result: A, B, C all marked for movement (3 moves)
```

**Limitation:** If the first element moves to the end, React will move all other elements instead of the one:

```
Old: [A, B, C, D]
New: [B, C, D, A]

Better: MOVE A to end (1 operation)
React does: MOVE B, C, D forward (3 operations)
```

This is a known trade-off for the O(n) guarantee. Vue 3 uses a more sophisticated LIS (Longest Increasing Subsequence) algorithm that handles this better but is O(n log n).

---

## Part IV: The Commit Phase

### 4.1 Effect Collection

During reconciliation, fibers are marked with effect flags:

```javascript
// From react-reconciler/src/ReactFiberFlags.js

export const NoFlags = 0b00000000000000000000000;
export const PerformedWork = 0b00000000000000000000001;
export const Placement = 0b00000000000000000000010;  // Insert/move DOM node
export const Update = 0b00000000000000000000100;     // Update props/attributes
export const Deletion = 0b00000000000000000001000;   // Remove DOM node
export const ChildDeletion = 0b00000000000000000010000;
export const ContentReset = 0b00000000000000000100000;
export const Callback = 0b00000000000000001000000;  // Has setState callback
export const DidCapture = 0b00000000000000010000000;
export const Ref = 0b00000000000000100000000;       // Ref needs update
export const Snapshot = 0b00000000000001000000000;  // getSnapshotBeforeUpdate
export const Passive = 0b00000000000010000000000;   // useEffect
export const Hydrating = 0b00000000000100000000000;
export const Visibility = 0b00000000001000000000000;
export const StoreConsistency = 0b00000000010000000000000;
```

These are **bitmask flags** for efficient storage and testing:

```javascript
// Set a flag
fiber.flags |= Placement | Update;

// Check a flag
if (fiber.flags & Placement) {
  // Has placement effect
}

// Clear a flag
fiber.flags &= ~Placement;
```

### 4.2 The Commit Pipeline

Once reconciliation completes, React enters the **commit phase** — this is where DOM mutations occur:

```javascript
// From react-reconciler/src/ReactFiberWorkLoop.js

function commitRoot(root: FiberRoot) {
  const finishedWork = root.finishedWork;
  const lanes = root.finishedLanes;

  root.finishedWork = null;
  root.finishedLanes = NoLanes;

  // ===== PHASE 1: Before Mutation =====
  // Class: getSnapshotBeforeUpdate
  commitBeforeMutationEffects(root, finishedWork);

  // ===== PHASE 2: Mutation =====
  // Actual DOM modifications
  commitMutationEffects(root, finishedWork, lanes);

  // Swap the current tree pointer
  root.current = finishedWork;

  // ===== PHASE 3: Layout =====
  // Class: componentDidMount/Update
  // useLayoutEffect
  commitLayoutEffects(finishedWork, root, lanes);

  // ===== PHASE 4: Passive Effects (Async) =====
  // Schedule useEffect callbacks
  if (
    (finishedWork.subtreeFlags & PassiveMask) !== NoFlags ||
    (finishedWork.flags & PassiveMask) !== NoFlags
  ) {
    scheduleCallback(NormalSchedulerPriority, () => {
      flushPassiveEffects();
      return null;
    });
  }
}
```

### 4.3 commitMutationEffects — The DOM Surgery

This is where Virtual DOM changes manifest as real DOM operations:

```javascript
function commitMutationEffects(
  root: FiberRoot,
  finishedWork: Fiber,
  committedLanes: Lanes,
) {
  commitMutationEffectsOnFiber(finishedWork, root, committedLanes);
}

function commitMutationEffectsOnFiber(
  finishedWork: Fiber,
  root: FiberRoot,
  lanes: Lanes,
) {
  const flags = finishedWork.flags;

  // Handle ContentReset (clear text content)
  if (flags & ContentReset) {
    commitResetTextContent(finishedWork);
  }

  // Handle Ref detachment
  if (flags & Ref) {
    const current = finishedWork.alternate;
    if (current !== null) {
      commitDetachRef(current);
    }
  }

  // Handle Visibility (Suspense boundaries)
  if (flags & Visibility) {
    commitVisibilityEffects(finishedWork);
  }

  // Primary mutation logic
  const primaryFlags = flags & (Placement | Update | ChildDeletion | Hydrating);

  switch (primaryFlags) {
    case Placement: {
      commitPlacement(finishedWork);
      finishedWork.flags &= ~Placement;
      break;
    }
    case PlacementAndUpdate: {
      // Placement
      commitPlacement(finishedWork);
      finishedWork.flags &= ~Placement;

      // Update
      const current = finishedWork.alternate;
      commitWork(current, finishedWork);
      break;
    }
    case Update: {
      const current = finishedWork.alternate;
      commitWork(current, finishedWork);
      break;
    }
    case ChildDeletion: {
      commitDeletions(finishedWork);
      break;
    }
  }

  // Recurse into children
  recursivelyTraverseMutationEffects(finishedWork, root);
}
```

### 4.4 commitPlacement — Inserting Nodes

```javascript
function commitPlacement(finishedWork: Fiber): void {
  // Find the parent DOM node
  const parentFiber = getHostParentFiber(finishedWork);

  let parent;
  let isContainer;
  const parentStateNode = parentFiber.stateNode;

  switch (parentFiber.tag) {
    case HostComponent:
      parent = parentStateNode;
      isContainer = false;
      break;
    case HostRoot:
      parent = parentStateNode.containerInfo;
      isContainer = true;
      break;
    case HostPortal:
      parent = parentStateNode.containerInfo;
      isContainer = true;
      break;
    default:
      throw new Error('Invalid host parent fiber.');
  }

  // Find the correct insertion point
  const before = getHostSibling(finishedWork);

  if (isContainer) {
    insertOrAppendPlacementNodeIntoContainer(finishedWork, before, parent);
  } else {
    insertOrAppendPlacementNode(finishedWork, before, parent);
  }
}

function insertOrAppendPlacementNode(
  node: Fiber,
  before: Instance | null,
  parent: Instance,
): void {
  const tag = node.tag;
  const isHost = tag === HostComponent || tag === HostText;

  if (isHost) {
    const stateNode = node.stateNode;
    if (before) {
      insertBefore(parent, stateNode, before);
    } else {
      appendChild(parent, stateNode);
    }
  } else {
    // Not a host node, recurse into children
    const child = node.child;
    if (child !== null) {
      insertOrAppendPlacementNode(child, before, parent);
      let sibling = child.sibling;
      while (sibling !== null) {
        insertOrAppendPlacementNode(sibling, before, parent);
        sibling = sibling.sibling;
      }
    }
  }
}
```

**DOM API Bridge:**
```javascript
// From react-dom/src/client/ReactDOMHostConfig.js

export function insertBefore(
  parentInstance: Instance,
  child: Instance | TextInstance,
  beforeChild: Instance | TextInstance,
): void {
  parentInstance.insertBefore(child, beforeChild);
}

export function appendChild(
  parentInstance: Instance,
  child: Instance | TextInstance,
): void {
  parentInstance.appendChild(child);
}
```

Finally, the Virtual DOM reaches the DOM. These are the only places where React directly mutates the DOM.

### 4.5 commitWork — Updating Properties

```javascript
function commitWork(current: Fiber | null, finishedWork: Fiber): void {
  switch (finishedWork.tag) {
    case FunctionComponent:
    case ForwardRef:
    case MemoComponent:
    case SimpleMemoComponent: {
      // Handle useLayoutEffect cleanup
      commitHookEffectListUnmount(HookLayout | HookHasEffect, finishedWork);
      return;
    }
    case ClassComponent: {
      return;
    }
    case HostComponent: {
      const instance: Instance = finishedWork.stateNode;
      if (instance != null) {
        const newProps = finishedWork.memoizedProps;
        const oldProps = current !== null ? current.memoizedProps : newProps;
        const type = finishedWork.type;

        // Compute diff and apply
        const updatePayload = prepareUpdate(
          instance,
          type,
          oldProps,
          newProps,
          rootContainerInstance,
          hostContext,
        );

        finishedWork.updateQueue = updatePayload;

        if (updatePayload) {
          commitUpdate(
            instance,
            updatePayload,
            type,
            oldProps,
            newProps,
            finishedWork,
          );
        }
      }
      return;
    }
    case HostText: {
      const textInstance: TextInstance = finishedWork.stateNode;
      const newText: string = finishedWork.memoizedProps;
      const oldText: string = current !== null ? current.memoizedProps : newText;

      commitTextUpdate(textInstance, oldText, newText);
      return;
    }
  }
}
```

**Property Diffing:**
```javascript
// From react-dom/src/client/ReactDOMHostConfig.js

export function prepareUpdate(
  domElement: Instance,
  type: string,
  oldProps: Props,
  newProps: Props,
  rootContainerInstance: Container,
  hostContext: HostContext,
): null | Array<mixed> {
  return diffProperties(
    domElement,
    type,
    oldProps,
    newProps,
    rootContainerInstance,
  );
}

function diffProperties(
  domElement: Element,
  tag: string,
  lastRawProps: Object,
  nextRawProps: Object,
  rootContainerElement: Document | DocumentFragment,
): null | Array<mixed> {

  let updatePayload: null | Array<any> = null;

  let lastProps: Object = lastRawProps;
  let nextProps: Object = nextRawProps;

  let propKey;
  let styleName;
  let styleUpdates = null;

  // Remove props that exist in old but not in new
  for (propKey in lastProps) {
    if (
      nextProps.hasOwnProperty(propKey) ||
      !lastProps.hasOwnProperty(propKey) ||
      lastProps[propKey] == null
    ) {
      continue;
    }

    if (propKey === 'style') {
      const lastStyle = lastProps[propKey];
      for (styleName in lastStyle) {
        if (lastStyle.hasOwnProperty(styleName)) {
          if (!styleUpdates) {
            styleUpdates = {};
          }
          styleUpdates[styleName] = ''; // Clear style
        }
      }
    } else {
      // Mark for removal
      (updatePayload = updatePayload || []).push(propKey, null);
    }
  }

  // Add or update props
  for (propKey in nextProps) {
    const nextProp = nextProps[propKey];
    const lastProp = lastProps != null ? lastProps[propKey] : undefined;

    if (
      !nextProps.hasOwnProperty(propKey) ||
      nextProp === lastProp ||
      (nextProp == null && lastProp == null)
    ) {
      continue;
    }

    if (propKey === 'style') {
      if (lastProp) {
        // Clear old styles not in new
        for (styleName in lastProp) {
          if (
            lastProp.hasOwnProperty(styleName) &&
            (!nextProp || !nextProp.hasOwnProperty(styleName))
          ) {
            if (!styleUpdates) {
              styleUpdates = {};
            }
            styleUpdates[styleName] = '';
          }
        }
        // Add/update new styles
        for (styleName in nextProp) {
          if (
            nextProp.hasOwnProperty(styleName) &&
            lastProp[styleName] !== nextProp[styleName]
          ) {
            if (!styleUpdates) {
              styleUpdates = {};
            }
            styleUpdates[styleName] = nextProp[styleName];
          }
        }
      } else {
        styleUpdates = nextProp;
      }
    } else if (propKey === 'dangerouslySetInnerHTML') {
      const nextHtml = nextProp ? nextProp.__html : undefined;
      const lastHtml = lastProp ? lastProp.__html : undefined;
      if (nextHtml != null && lastHtml !== nextHtml) {
        (updatePayload = updatePayload || []).push(propKey, nextHtml);
      }
    } else if (propKey === 'children') {
      if (typeof nextProp === 'string' || typeof nextProp === 'number') {
        (updatePayload = updatePayload || []).push(propKey, '' + nextProp);
      }
    } else {
      // General property
      (updatePayload = updatePayload || []).push(propKey, nextProp);
    }
  }

  if (styleUpdates) {
    (updatePayload = updatePayload || []).push('style', styleUpdates);
  }

  return updatePayload;
}
```

**Update Format:**
The payload is a flat array: `[key1, value1, key2, value2, ...]`

Example:
```javascript
Old: { className: 'foo', id: 'bar', title: 'old' }
New: { className: 'baz', id: 'bar', disabled: true }

Payload: ['className', 'baz', 'title', null, 'disabled', true]
```

**Applying Updates:**
```javascript
export function commitUpdate(
  domElement: Instance,
  updatePayload: Array<mixed>,
  type: string,
  oldProps: Props,
  newProps: Props,
  internalInstanceHandle: Object,
): void {
  // Apply the diff
  updateProperties(domElement, updatePayload, type, oldProps, newProps);

  // Update internal instance
  updateFiberProps(domElement, newProps);
}

function updateProperties(
  domElement: Element,
  updatePayload: Array<any>,
  tag: string,
  lastRawProps: Object,
  nextRawProps: Object,
): void {
  // Walk the updatePayload array
  for (let i = 0; i < updatePayload.length; i += 2) {
    const propKey = updatePayload[i];
    const propValue = updatePayload[i + 1];

    if (propKey === 'style') {
      setValueForStyles(domElement, propValue);
    } else if (propKey === 'dangerouslySetInnerHTML') {
      setInnerHTML(domElement, propValue);
    } else if (propKey === 'children') {
      setTextContent(domElement, propValue);
    } else {
      setValueForProperty(domElement, propKey, propValue, false);
    }
  }
}
```

---

## Part V: Advanced Optimizations

### 5.1 Keys and List Identity

The `key` prop is React's way to track element identity across renders:

```javascript
// Bad: No keys
{items.map(item => <Item data={item} />)}

// Bad: Index as key (breaks on reorder)
{items.map((item, i) => <Item key={i} data={item} />)}

// Good: Stable unique identifier
{items.map(item => <Item key={item.id} data={item} />)}
```

**Why Index Keys Break:**
```
Initial:
  [A(key:0), B(key:1), C(key:2)]

After deleting B:
  [A(key:0), C(key:1)]

React sees:
- Position 0: A → A (reuse) ✓
- Position 1: B → C (UPDATE B's props to C's props) ✗
- Position 2: C → (delete) ✗

Expected:
- Position 0: A → A (reuse) ✓
- Position 1: B → (delete) ✓
- Position 2: C → C (reuse) ✓
```

With proper keys, React identifies elements across positions.

### 5.2 React.memo and Bailout

```javascript
const MemoizedComponent = React.memo(function MyComponent(props) {
  // Only re-renders if props change
  return <div>{props.value}</div>;
});

// Custom comparison
const MemoizedComponent = React.memo(MyComponent, (prevProps, nextProps) => {
  // Return true if props are equal (skip render)
  // Return false if props differ (re-render)
  return prevProps.value === nextProps.value;
});
```

**Under the Hood:**
```javascript
// Simplified from react-reconciler/src/ReactFiberBeginWork.js

function updateMemoComponent(
  current: Fiber | null,
  workInProgress: Fiber,
  Component: any,
  nextProps: any,
  renderLanes: Lanes,
): Fiber | null {

  if (current !== null) {
    const prevProps = current.memoizedProps;

    // Check if props are equal
    let compare = Component.compare;
    compare = compare !== null ? compare : shallowEqual;

    if (compare(prevProps, nextProps)) {
      // Props equal, bail out
      return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
    }
  }

  // Props differ, proceed with render
  const type = Component.type;
  const resolvedProps = resolveDefaultProps(type, nextProps);

  return updateFunctionComponent(
    current,
    workInProgress,
    type,
    resolvedProps,
    renderLanes,
  );
}
```

### 5.3 Lane-Based Priority

React 18+ uses a **lane model** for scheduling:

```javascript
// From react-reconciler/src/ReactFiberLane.js

export const NoLanes: Lanes = 0b0000000000000000000000000000000;
export const NoLane: Lane = 0b0000000000000000000000000000000;

export const SyncLane: Lane = 0b0000000000000000000000000000001;

// Continuous user input (e.g., scroll, mousemove)
export const InputContinuousLane: Lane = 0b0000000000000000000000000000100;

// Discrete user input (e.g., click, keydown)
export const DefaultLane: Lane = 0b0000000000000000000000000010000;

// Transitions (startTransition)
export const TransitionLanes: Lanes = 0b0000000001111111111111111000000;

// Idle/background work
export const IdleLanes: Lanes = 0b0110000000000000000000000000000;

// Offscreen (Suspense, hidden trees)
export const OffscreenLane: Lane = 0b1000000000000000000000000000000;
```

Lanes are **bitmasks** representing priority levels. Multiple updates can be batched into the same lane:

```javascript
function ensureRootIsScheduled(root: FiberRoot) {
  const nextLanes = getNextLanes(
    root,
    root === workInProgressRoot ? workInProgressRootRenderLanes : NoLanes,
  );

  // Determine priority
  const newCallbackPriority = getHighestPriorityLane(nextLanes);

  // Map lane to scheduler priority
  let schedulerPriorityLevel;
  switch (lanesToEventPriority(nextLanes)) {
    case DiscreteEventPriority:
      schedulerPriorityLevel = ImmediateSchedulerPriority;
      break;
    case ContinuousEventPriority:
      schedulerPriorityLevel = UserBlockingSchedulerPriority;
      break;
    case DefaultEventPriority:
      schedulerPriorityLevel = NormalSchedulerPriority;
      break;
    case IdleEventPriority:
      schedulerPriorityLevel = IdleSchedulerPriority;
      break;
    default:
      schedulerPriorityLevel = NormalSchedulerPriority;
      break;
  }

  // Schedule work
  newCallbackNode = scheduleCallback(
    schedulerPriorityLevel,
    performConcurrentWorkOnRoot.bind(null, root),
  );

  root.callbackPriority = newCallbackPriority;
  root.callbackNode = newCallbackNode;
}
```

**Time Slicing:**

In concurrent mode, React can interrupt work and yield to the browser:

```javascript
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    performUnitOfWork(workInProgress);
  }
}

function shouldYield(): boolean {
  const currentTime = getCurrentTime();
  const deadline = currentTime + yieldInterval; // Default: 5ms

  return (
    currentTime >= deadline ||
    (enableSchedulerDebugging && isSchedulerPaused)
  );
}
```

This allows high-priority work (user input) to interrupt low-priority work (background data fetching).

---

## Part VI: Performance Characteristics

### 6.1 Time Complexity Analysis

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| createElement | O(1) | Object allocation |
| Single element reconciliation | O(1) | Direct comparison |
| Array reconciliation (no keys) | O(n) | Linear scan |
| Array reconciliation (with keys) | O(n) | HashMap lookup |
| Deep tree reconciliation | O(n) | Single-pass traversal |
| Commit phase | O(m) | m = number of changed nodes |

**Worst Case: O(n²)** occurs when:
- Every element in a list changes type
- Keys are randomly reassigned

**Best Case: O(1)** occurs when:
- Props reference-equal (React.memo bailout)
- No children

### 6.2 Memory Overhead

**Per Virtual Node:**
- Object header: 8-16 bytes (V8)
- Type pointer: 8 bytes
- Props object: 16 bytes + property storage
- Children array: 16 bytes + element pointers
- Key/ref/owner: 24 bytes
- **Total: ~80-120 bytes minimum**

**Per Fiber Node:**
- Base object: ~48 bytes
- Relationship pointers: 32 bytes
- State/props: variable
- Effect flags: 8 bytes
- **Total: ~200-300 bytes**

For a tree with 1,000 nodes:
- Virtual DOM: ~100KB
- Fiber tree: ~300KB
- Actual DOM: ~500KB-1MB

**Double Buffering Cost:**
React maintains TWO fiber trees simultaneously, so memory usage is approximately doubled during reconciliation.

### 6.3 GC Pressure

Virtual DOM creates significant garbage:

```javascript
// Every render creates new objects
function App({ count }) {
  return (
    <div>                    // New object
      <span>{count}</span>   // New object
    </div>                   // New array for children
  );
}
```

**At 60fps:** 60 * tree_size objects/second

**Mitigation strategies:**
1. Object pooling (React doesn't do this for VNodes, but does for some internal structures)
2. Persistent data structures (not used in React's VTree)
3. Bailout via memo/PureComponent

### 6.4 Benchmark Comparison

**Naive Approach (innerHTML):**
```javascript
// Recreate entire subtree
element.innerHTML = generateHTML(data);
// Time: ~100ms for 1000 nodes
// Operations: 1 massive reflow
```

**Virtual DOM:**
```javascript
// Diff + targeted updates
const newVTree = render(data);
patch(oldVTree, newVTree);
// Time: ~5ms for 1000 nodes (with 10% changes)
// Operations: ~100 targeted DOM ops
```

**Real DOM Manipulation (optimal):**
```javascript
// Manually computed minimal changes
applyChanges(element, computedDiff);
// Time: ~3ms
// Operations: ~100 targeted DOM ops
```

Virtual DOM is **not faster** than optimal manual updates, but it's:
1. Easier to reason about
2. Prevents accidental O(n²) bugs
3. Enables features like time-travel debugging

---

## Part VII: Alternative Approaches

### 7.1 Vue 3's Compiler-Optimized VDOM

Vue takes a hybrid approach: **compiler hints + runtime VDOM**.

```vue
<template>
  <div>
    <span>Static</span>
    <span>{{ dynamic }}</span>
  </div>
</template>
```

**Compiled Output:**
```javascript
import { createElementVNode as _createElementVNode, toDisplayString as _toDisplayString, openBlock as _openBlock, createElementBlock as _createElementBlock } from "vue"

const _hoisted_1 = /*#__PURE__*/_createElementVNode("span", null, "Static", -1 /* HOISTED */)

export function render(_ctx, _cache) {
  return (_openBlock(), _createElementBlock("div", null, [
    _hoisted_1,  // Hoisted static node
    _createElementVNode("span", null, _toDisplayString(_ctx.dynamic), 1 /* TEXT */)
  ]))
}
```

**Optimizations:**
1. **Static Hoisting:** `_hoisted_1` is created once, never re-created
2. **Patch Flags:** `1 /* TEXT */` tells the runtime "only text content can change"
3. **Tree Flattening:** Compiler produces flat arrays of dynamic nodes

**Block Tree Concept:**
```javascript
// Instead of recursing through entire tree
for (let i = 0; i < tree.length; i++) {
  diff(tree[i])  // O(n)
}

// Vue tracks only dynamic nodes
for (let i = 0; i < dynamicChildren.length; i++) {
  diff(dynamicChildren[i])  // O(d) where d << n
}
```

This makes Vue's VDOM **faster than React's** in many scenarios, but requires a compiler.

### 7.2 Svelte's Compile-Away Approach

Svelte eliminates the Virtual DOM entirely:

```svelte
<script>
  let count = 0;
</script>

<button on:click={() => count++}>
  Clicked {count} times
</button>
```

**Compiled Output:**
```javascript
function create_fragment(ctx) {
  let button;
  let t0;
  let t1;
  let t2;
  let mounted;
  let dispose;

  return {
    c() {  // Create
      button = element("button");
      t0 = text("Clicked ");
      t1 = text(/*count*/ ctx[0]);
      t2 = text(" times");
    },
    m(target, anchor) {  // Mount
      insert(target, button, anchor);
      append(button, t0);
      append(button, t1);
      append(button, t2);

      if (!mounted) {
        dispose = listen(button, "click", /*click_handler*/ ctx[1]);
        mounted = true;
      }
    },
    p(ctx, [dirty]) {  // Patch
      if (dirty & /*count*/ 1) set_data(t1, /*count*/ ctx[0]);
    },
    d(detaching) {  // Destroy
      if (detaching) detach(button);
      mounted = false;
      dispose();
    }
  };
}

function instance($$self, $$props, $$invalidate) {
  let count = 0;

  const click_handler = () => $$invalidate(0, count++, count);

  return [count, click_handler];
}

class Component extends SvelteComponent {
  constructor(options) {
    super();
    init(this, options, instance, create_fragment, safe_not_equal, {});
  }
}
```

**Key Differences:**
1. **Surgical Updates:** `set_data(t1, count)` directly updates the specific text node
2. **Dirty Checking:** `dirty & 1` checks if `count` changed using bitmask
3. **No Diffing:** Compiler knows exactly what can change

**Trade-offs:**
- **Pros:** Smaller runtime, faster updates, no VDOM overhead
- **Cons:** Larger bundle (code per component), less dynamic, harder to debug

### 7.3 Solid.js's Fine-Grained Reactivity

Solid uses a **reactive primitive system** with no VDOM:

```javascript
import { createSignal, createEffect } from 'solid-js';

function Counter() {
  const [count, setCount] = createSignal(0);

  return (
    <button onClick={() => setCount(count() + 1)}>
      Clicked {count()} times
    </button>
  );
}
```

**Runtime Behavior:**
```javascript
// Compiled JSX creates DOM imperatively
const button = document.createElement('button');
const textNode1 = document.createTextNode('Clicked ');
const textNode2 = document.createTextNode('');
const textNode3 = document.createTextNode(' times');

button.appendChild(textNode1);
button.appendChild(textNode2);
button.appendChild(textNode3);

// Setup reactive subscription
createEffect(() => {
  textNode2.data = count();  // Updates ONLY this text node
});

button.addEventListener('click', () => setCount(count() + 1));
```

**Reactivity Graph:**
```
Signal(count)
    ↓ subscription
Effect(() => textNode2.data = count())
```

When `count` changes:
1. Signal notifies subscribers
2. Effect re-runs
3. Text node updates directly

**No diffing, no reconciliation** — just reactive subscriptions.

---

## Part VIII: Practical Implementation

Let's build a **minimal Virtual DOM** from scratch to solidify understanding.

### 8.1 Virtual Node Structure

```javascript
// vnode.js

export function h(type, props, ...children) {
  return {
    type,
    props: props || {},
    children: children.flat(),
  };
}

// Text node helper
export function text(value) {
  return {
    type: 'TEXT',
    props: {},
    children: [],
    text: String(value),
  };
}

// Example usage:
const vnode = h('div', { id: 'container' },
  h('h1', { class: 'title' }, 'Hello'),
  h('p', null, 'World')
);

// Result:
{
  type: 'div',
  props: { id: 'container' },
  children: [
    {
      type: 'h1',
      props: { class: 'title' },
      children: ['Hello']
    },
    {
      type: 'p',
      props: {},
      children: ['World']
    }
  ]
}
```

### 8.2 Rendering Virtual DOM to Real DOM

```javascript
// render.js

export function render(vnode) {
  // Handle text nodes
  if (vnode.type === 'TEXT') {
    return document.createTextNode(vnode.text);
  }

  // Handle primitive values
  if (typeof vnode === 'string' || typeof vnode === 'number') {
    return document.createTextNode(vnode);
  }

  // Create element
  const element = document.createElement(vnode.type);

  // Set properties
  setProps(element, vnode.props);

  // Render children
  vnode.children
    .map(render)  // Recursively render each child
    .forEach(child => element.appendChild(child));

  return element;
}

function setProps(element, props) {
  Object.entries(props).forEach(([key, value]) => {
    setProp(element, key, value);
  });
}

function setProp(element, key, value) {
  if (key === 'className') {
    element.className = value;
  } else if (key === 'style') {
    // Handle style object
    Object.assign(element.style, value);
  } else if (key.startsWith('on')) {
    // Event listener
    const eventType = key.slice(2).toLowerCase();
    element.addEventListener(eventType, value);
  } else {
    // Regular attribute
    element.setAttribute(key, value);
  }
}
```

### 8.3 The Diff Algorithm

```javascript
// diff.js

const PATCH_TYPE = {
  CREATE: 'CREATE',
  REMOVE: 'REMOVE',
  REPLACE: 'REPLACE',
  UPDATE: 'UPDATE',
};

export function diff(oldVNode, newVNode) {
  // Case 1: New node doesn't exist
  if (!newVNode) {
    return { type: PATCH_TYPE.REMOVE };
  }

  // Case 2: Old node doesn't exist
  if (!oldVNode) {
    return { type: PATCH_TYPE.CREATE, newVNode };
  }

  // Case 3: Nodes have different types
  if (typeof oldVNode !== typeof newVNode ||
      (oldVNode.type !== newVNode.type)) {
    return { type: PATCH_TYPE.REPLACE, newVNode };
  }

  // Case 4: Both are text nodes
  if (oldVNode.type === 'TEXT' && newVNode.type === 'TEXT') {
    if (oldVNode.text !== newVNode.text) {
      return { type: PATCH_TYPE.UPDATE, text: newVNode.text };
    }
    return null;  // No change
  }

  // Case 5: Both are elements - check props and children
  const propsDiff = diffProps(oldVNode.props, newVNode.props);
  const childrenDiff = diffChildren(oldVNode.children, newVNode.children);

  if (propsDiff || childrenDiff.length > 0) {
    return {
      type: PATCH_TYPE.UPDATE,
      props: propsDiff,
      children: childrenDiff,
    };
  }

  return null;  // No changes
}

function diffProps(oldProps, newProps) {
  const patches = {};

  // Find changed/removed props
  for (const key in oldProps) {
    if (!(key in newProps)) {
      patches[key] = undefined;  // Remove
    } else if (oldProps[key] !== newProps[key]) {
      patches[key] = newProps[key];  // Update
    }
  }

  // Find added props
  for (const key in newProps) {
    if (!(key in oldProps)) {
      patches[key] = newProps[key];  // Add
    }
  }

  return Object.keys(patches).length > 0 ? patches : null;
}

function diffChildren(oldChildren, newChildren) {
  const patches = [];
  const maxLength = Math.max(oldChildren.length, newChildren.length);

  for (let i = 0; i < maxLength; i++) {
    const patch = diff(oldChildren[i], newChildren[i]);
    patches.push({ index: i, patch });
  }

  return patches.filter(p => p.patch !== null);
}
```

### 8.4 Applying Patches

```javascript
// patch.js

export function patch(parent, patches, index = 0) {
  if (!patches) return;

  const element = parent.childNodes[index];

  switch (patches.type) {
    case 'CREATE': {
      const newElement = render(patches.newVNode);
      parent.appendChild(newElement);
      break;
    }

    case 'REMOVE': {
      parent.removeChild(element);
      break;
    }

    case 'REPLACE': {
      const newElement = render(patches.newVNode);
      parent.replaceChild(newElement, element);
      break;
    }

    case 'UPDATE': {
      // Update text content
      if (patches.text !== undefined) {
        element.textContent = patches.text;
      }

      // Update props
      if (patches.props) {
        updateProps(element, patches.props);
      }

      // Update children
      if (patches.children) {
        patches.children.forEach(childPatch => {
          patch(element, childPatch.patch, childPatch.index);
        });
      }
      break;
    }
  }
}

function updateProps(element, props) {
  Object.entries(props).forEach(([key, value]) => {
    if (value === undefined) {
      // Remove property
      removeProp(element, key);
    } else {
      // Update property
      setProp(element, key, value);
    }
  });
}

function removeProp(element, key) {
  if (key === 'className') {
    element.className = '';
  } else if (key === 'style') {
    element.style = {};
  } else if (key.startsWith('on')) {
    const eventType = key.slice(2).toLowerCase();
    // Note: This is simplified. In real implementation,
    // we'd need to track listeners to remove them properly.
    element.removeEventListener(eventType, null);
  } else {
    element.removeAttribute(key);
  }
}
```

### 8.5 Complete Example

```javascript
// app.js

import { h } from './vnode.js';
import { render } from './render.js';
import { diff } from './diff.js';
import { patch } from './patch.js';

// Initial render
let count = 0;

function view(count) {
  return h('div', { id: 'app' },
    h('h1', null, `Count: ${count}`),
    h('button', {
      onclick: () => update()
    }, 'Increment')
  );
}

let oldVTree = view(count);
let rootElement = render(oldVTree);
document.body.appendChild(rootElement);

function update() {
  count++;

  // Create new virtual tree
  const newVTree = view(count);

  // Compute patches
  const patches = diff(oldVTree, newVTree);

  // Apply patches to real DOM
  patch(document.body, patches, 0);

  // Save for next update
  oldVTree = newVTree;
}
```

**Execution Flow:**
```
User clicks button
    ↓
update() called
    ↓
Create new VTree: { h1: "Count: 1" }
    ↓
Diff old vs new:
  - Container: no change
  - h1: text changed "Count: 0" → "Count: 1"
  - button: no change
    ↓
Patches: [{ type: UPDATE, text: "Count: 1" }]
    ↓
Apply to DOM: h1.textContent = "Count: 1"
    ↓
Save new VTree for next update
```

---

## Part IX: Advanced Patterns and Optimizations

### 9.1 Keyed Reconciliation

Let's enhance our diff algorithm with key support:

```javascript
function diffChildren(oldChildren, newChildren) {
  const patches = [];

  // Build key maps
  const oldKeyed = new Map();
  const newKeyed = new Map();

  oldChildren.forEach((child, i) => {
    const key = child.props?.key;
    if (key !== undefined) {
      oldKeyed.set(key, { child, index: i });
    }
  });

  newChildren.forEach((child, i) => {
    const key = child.props?.key;
    if (key !== undefined) {
      newKeyed.set(key, { child, index: i });
    }
  });

  // Track processed indices
  const processedOld = new Set();
  const processedNew = new Set();

  // Pass 1: Match by key
  for (const [key, { child: newChild, index: newIndex }] of newKeyed) {
    if (oldKeyed.has(key)) {
      const { child: oldChild, index: oldIndex } = oldKeyed.get(key);
      const patch = diff(oldChild, newChild);

      if (oldIndex !== newIndex) {
        // Moved
        patches.push({
          type: 'MOVE',
          from: oldIndex,
          to: newIndex,
          patch,
        });
      } else if (patch) {
        // Updated in place
        patches.push({
          index: newIndex,
          patch,
        });
      }

      processedOld.add(oldIndex);
      processedNew.add(newIndex);
    }
  }

  // Pass 2: Handle unkeyed and new nodes
  const maxLength = Math.max(oldChildren.length, newChildren.length);

  for (let i = 0; i < maxLength; i++) {
    if (processedNew.has(i) || processedOld.has(i)) continue;

    const patch = diff(oldChildren[i], newChildren[i]);
    if (patch) {
      patches.push({ index: i, patch });
    }
  }

  return patches;
}
```

### 9.2 Component Abstraction

```javascript
// Component system
class Component {
  constructor(props) {
    this.props = props;
    this.state = {};
  }

  setState(newState) {
    this.state = { ...this.state, ...newState };
    this.update();
  }

  update() {
    const newVTree = this.render();
    const patches = diff(this._vnode, newVTree);
    patch(this._element.parentNode, patches,
          Array.from(this._element.parentNode.childNodes).indexOf(this._element));
    this._vnode = newVTree;
  }

  mount(parent) {
    this._vnode = this.render();
    this._element = render(this._vnode);
    parent.appendChild(this._element);
  }

  render() {
    throw new Error('render() must be implemented');
  }
}

// Example component
class Counter extends Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
  }

  render() {
    return h('div', null,
      h('h1', null, `Count: ${this.state.count}`),
      h('button', {
        onclick: () => this.setState({ count: this.state.count + 1 })
      }, 'Increment')
    );
  }
}

// Usage
const counter = new Counter();
counter.mount(document.body);
```

### 9.3 Async Rendering and Scheduling

```javascript
// Simple scheduler
class Scheduler {
  constructor() {
    this.tasks = [];
    this.isRunning = false;
  }

  schedule(task, priority = 0) {
    this.tasks.push({ task, priority });
    this.tasks.sort((a, b) => b.priority - a.priority);

    if (!this.isRunning) {
      this.run();
    }
  }

  run() {
    this.isRunning = true;

    const work = (deadline) => {
      while (this.tasks.length > 0 && deadline.timeRemaining() > 0) {
        const { task } = this.tasks.shift();
        task();
      }

      if (this.tasks.length > 0) {
        requestIdleCallback(work);
      } else {
        this.isRunning = false;
      }
    };

    requestIdleCallback(work);
  }
}

const scheduler = new Scheduler();

// Usage: Break rendering into chunks
function renderLargeList(items) {
  const container = document.createElement('div');

  const CHUNK_SIZE = 100;
  let index = 0;

  function renderChunk() {
    const end = Math.min(index + CHUNK_SIZE, items.length);

    for (; index < end; index++) {
      const item = render(h('div', null, items[index]));
      container.appendChild(item);
    }

    if (index < items.length) {
      scheduler.schedule(renderChunk, 1);
    }
  }

  scheduler.schedule(renderChunk, 1);
  return container;
}
```

---

## Part X: Debugging and Performance Tools

### 10.1 Virtual DOM Inspector

```javascript
// Debug utilities
export function inspectVTree(vnode, depth = 0) {
  const indent = '  '.repeat(depth);

  if (vnode.type === 'TEXT') {
    console.log(`${indent}TEXT: "${vnode.text}"`);
    return;
  }

  console.log(`${indent}<${vnode.type}> ${JSON.stringify(vnode.props)}`);
  vnode.children.forEach(child => inspectVTree(child, depth + 1));
}

export function visualizeDiff(oldVNode, newVNode) {
  const patches = diff(oldVNode, newVNode);
  console.log('Diff Result:', JSON.stringify(patches, null, 2));

  // Count operations
  const stats = {
    creates: 0,
    removes: 0,
    updates: 0,
    replaces: 0,
  };

  function count(patch) {
    if (!patch) return;

    switch (patch.type) {
      case 'CREATE': stats.creates++; break;
      case 'REMOVE': stats.removes++; break;
      case 'UPDATE': stats.updates++; break;
      case 'REPLACE': stats.replaces++; break;
    }

    if (patch.children) {
      patch.children.forEach(c => count(c.patch));
    }
  }

  count(patches);
  console.log('Operation Stats:', stats);
}
```

### 10.2 Performance Profiling

```javascript
// Performance instrumentation
class Profiler {
  constructor() {
    this.metrics = {
      renders: 0,
      diffs: 0,
      patches: 0,
      renderTime: 0,
      diffTime: 0,
      patchTime: 0,
    };
  }

  measure(name, fn) {
    const start = performance.now();
    const result = fn();
    const end = performance.now();

    this.metrics[name] = (this.metrics[name] || 0) + (end - start);
    this.metrics[`${name}s`] = (this.metrics[`${name}s`] || 0) + 1;

    return result;
  }

  report() {
    console.table({
      'Render Count': this.metrics.renders,
      'Avg Render Time': `${(this.metrics.renderTime / this.metrics.renders).toFixed(2)}ms`,
      'Diff Count': this.metrics.diffs,
      'Avg Diff Time': `${(this.metrics.diffTime / this.metrics.diffs).toFixed(2)}ms`,
      'Patch Count': this.metrics.patches,
      'Avg Patch Time': `${(this.metrics.patchTime / this.metrics.patches).toFixed(2)}ms`,
    });
  }
}

const profiler = new Profiler();

// Instrumented update function
function update() {
  count++;

  const newVTree = profiler.measure('render', () => view(count));
  const patches = profiler.measure('diff', () => diff(oldVTree, newVTree));
  profiler.measure('patch', () => patch(document.body, patches, 0));

  oldVTree = newVTree;
}
```

---

## Conclusion: The Invisible Infrastructure

The Virtual DOM is not magic — it's a **carefully engineered abstraction** that trades memory and CPU cycles for developer ergonomics and correctness.

### Key Takeaways:

1. **Virtual DOM = Shadow Tree**: An in-memory JavaScript object graph that mirrors DOM structure at ~10% the memory cost.

2. **Reconciliation = Diff + Patch**: React's O(n) heuristic algorithm that identifies minimal DOM operations through key-based identity tracking and type-based structural assumptions.

3. **Fiber = Incremental Reconciliation**: React 16+'s priority-based scheduler that enables interruptible rendering through double-buffering and lane-based work batching.

4. **Trade-offs**: Virtual DOM sacrifices optimal update efficiency for predictable behavior, maintainable code, and features like time-travel debugging.

5. **Alternatives Exist**: Compiler-based approaches (Svelte, Vue) and fine-grained reactivity (Solid) offer different trade-offs, often with better raw performance at the cost of developer flexibility.

The Virtual DOM is **infrastructure** — invisible when working correctly, but fundamental to how modern web applications achieve both performance and simplicity at scale.

**You now understand not just WHAT the Virtual DOM is, but WHY it exists and HOW it operates at the system level.**

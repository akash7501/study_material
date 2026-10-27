# 04 — The Fiber Node, Field By Field

> **Part 2 · Topic 4** — [index](README.md) · prev: [03 — Stack → Fiber](../part-1-what-react-is/03-stack-reconciler-to-fiber.md) · next: 05 — Double buffering

In Topic 03 we established *what a fiber is conceptually*: a reified stack frame.
Now we open one up and look at every field.

This file is the reference you will come back to. Everything in Parts 2, 3 and 4
is "some algorithm reading and writing these fields".

---

## The actual constructor

This is essentially verbatim from `react-reconciler/src/ReactFiber.js`:

```js
function FiberNode(tag, pendingProps, key, mode) {
  // ─── IDENTITY ───────────────────────────────────
  this.tag = tag;                 // number: what KIND of fiber
  this.key = key;                 // your key prop, or null
  this.elementType = null;        // what the element said
  this.type = null;               // the resolved function/class/string
  this.stateNode = null;          // the real thing: DOM node, class instance...

  // ─── TREE STRUCTURE ─────────────────────────────
  this.return = null;             // parent (= return address)
  this.child = null;              // first child
  this.sibling = null;            // next sibling
  this.index = 0;                 // position among siblings

  this.ref = null;                // your ref prop

  // ─── PROPS & STATE ──────────────────────────────
  this.pendingProps = pendingProps;  // incoming props (new)
  this.memoizedProps = null;         // props from the last finished render
  this.updateQueue = null;           // queued state updates / effects
  this.memoizedState = null;         // state from the last finished render
  this.dependencies = null;          // contexts this fiber subscribes to

  this.mode = mode;               // Concurrent / Strict / Profile bitmask

  // ─── EFFECTS ────────────────────────────────────
  this.flags = NoFlags;           // what to DO to this fiber at commit
  this.subtreeFlags = NoFlags;    // OR of all descendants' flags
  this.deletions = null;          // children to delete at commit

  // ─── PRIORITY ───────────────────────────────────
  this.lanes = NoLanes;           // pending work ON this fiber
  this.childLanes = NoLanes;      // pending work IN the subtree

  // ─── DOUBLE BUFFERING ───────────────────────────
  this.alternate = null;          // the other version of this same fiber

  // ─── PROFILER (only with ProfileMode) ───────────
  this.actualDuration = 0;
  this.actualStartTime = -1;
  this.selfBaseDuration = 0;
  this.treeBaseDuration = 0;

  // ─── DEV ONLY ───────────────────────────────────
  this._debugOwner = null;
  this._debugHookTypes = null;
}
```

About 25 meaningful fields. Let's take them in groups.

---

## Group 1 — Identity: `tag`, `key`, `elementType`, `type`, `stateNode`

### `tag` — the single most important field

A **number** telling React what kind of node this is. The whole render phase is
one giant `switch (fiber.tag)`.

```js
switch (workInProgress.tag) {
  case FunctionComponent:  return updateFunctionComponent(...);
  case ClassComponent:     return updateClassComponent(...);
  case HostComponent:      return updateHostComponent(...);
  case HostText:           return updateHostText(...);
  case Fragment:           return updateFragment(...);
  case SuspenseComponent:  return updateSuspenseComponent(...);
  // ...~25 cases
}
```

The tag list (`ReactWorkTags.js`):

| # | Tag | What it is |
|---|---|---|
| 0 | `FunctionComponent` | `function App() {}` |
| 1 | `ClassComponent` | `class App extends Component` |
| 3 | `HostRoot` | the root of a tree — `createRoot()`'s fiber |
| 4 | `HostPortal` | `createPortal()` |
| 5 | `HostComponent` | `<div>`, `<span>` — a real host element |
| 6 | `HostText` | a bare text node |
| 7 | `Fragment` | `<>…</>` |
| 8 | `Mode` | `<StrictMode>` |
| 9 | `ContextConsumer` | `<Ctx.Consumer>` |
| 10 | `ContextProvider` | `<Ctx.Provider>` / `<Ctx>` in 19 |
| 11 | `ForwardRef` | `forwardRef(fn)` |
| 12 | `Profiler` | `<Profiler>` |
| 13 | `SuspenseComponent` | `<Suspense>` |
| 14 | `MemoComponent` | `memo(Comp)` where Comp isn't simple |
| 15 | `SimpleMemoComponent` | `memo(fn)` — optimised path |
| 16 | `LazyComponent` | `lazy(() => …)` |
| 18 | `DehydratedFragment` | SSR, not yet hydrated |
| 19 | `SuspenseListComponent` | `<SuspenseList>` |
| 22 | `OffscreenComponent` | hidden subtrees — powers Suspense & Offscreen |
| 24 | `CacheComponent` | the `cache()` boundary |
| 26 | `HostHoistable` | React 19 — `<title>`, `<meta>`, `<link>` hoisted to `<head>` |
| 27 | `HostSingleton` | React 19 — `<html>`, `<head>`, `<body>` |

> **Why a number and not a string?** This switch runs millions of times.
> Integer comparison and jump tables beat string comparison. React's source is
> full of this kind of micro-optimisation — bitmasks, numeric tags, pre-allocated
> objects. Worth noticing when reading it.

### `key`

Your `key` prop, lifted out of props at element-creation time. Used by **one**
algorithm only: child reconciliation (Topic 06). It answers the question
*"is this new element the same thing as that old fiber?"*

### `elementType` vs `type` — why two fields?

For 95% of fibers they are identical. The difference matters for wrappers:

```js
const Lazy = lazy(() => import('./Chart'));

// before the lazy resolves:
elementType = { $$typeof: Symbol(react.lazy), _payload, _init }
type        = null

// after it resolves:
elementType = { $$typeof: Symbol(react.lazy), ... }   // unchanged
type        = Chart                                   // the real component
```

```js
const Memo = memo(MyComp);

// SimpleMemoComponent:
elementType = { $$typeof: Symbol(react.memo), type: MyComp }
type        = MyComp     // unwrapped, so React can call it directly
```

**Rule:** `elementType` is **what you wrote**, `type` is **what React resolved it to**.
Reconciliation compares `elementType` (identity as the author declared it);
rendering uses `type` (the thing to actually call).

### `stateNode` — the bridge to the real world

Polymorphic. Depends entirely on `tag`:

| tag | `stateNode` holds |
|---|---|
| `HostComponent` | **the real DOM element** |
| `HostText` | the real text node |
| `ClassComponent` | the class **instance** (`this` in your methods) |
| `HostRoot` | the `FiberRootNode` |
| `FunctionComponent` | **`null`** — function components have no instance |

> `stateNode === null` for function components is literally why
> `ref` on a function component doesn't work without `forwardRef`
> (pre-19) — there is no instance to hand you.

### Two different "roots" — a classic confusion

```
   createRoot(container)  returns ──►  FiberRootNode
                                       ┌──────────────────────────┐
                                       │ containerInfo: <div#root>│
                                       │ current: ────────────────┼──┐
                                       │ pendingLanes, finishedWork│ │
                                       └──────────────────────────┘ │
                                                                    │
                                       ┌────────────────────────────▼─┐
                                       │ HostRoot fiber  (tag === 3)  │
                                       │ stateNode: ──► FiberRootNode │  (back-pointer)
                                       │ child: ──► <App/> fiber      │
                                       └──────────────────────────────┘
```

- **`FiberRootNode`** — one per `createRoot`. Not a fiber. Holds container info,
  pending lanes, the current tree pointer, callbacks.
- **`HostRoot` fiber** — the top fiber of the tree. `tag === 3`.

They point at each other. `root.current` is the entry to the tree;
`hostRootFiber.stateNode` points back to the root object.

---

## Group 2 — Tree structure: `return`, `child`, `sibling`, `index`

```
                           ┌──────────┐
                           │   App    │
                           └────┬─────┘
                          child │  ▲
                                ▼  │ return
          ┌──────────┐ sibling ┌──────────┐ sibling ┌──────────┐
          │  Header  │◄────────│   ...    │────────►│  Footer  │
          │ index: 0 │         │ index: 1 │         │ index: 2 │
          └──────────┘         └──────────┘         └──────────┘
                 ▲                                        │
                 └────────── all three have ──────────────┘
                             return → App
```

**Only the first child is reachable from the parent.** The rest are a singly
linked list through `sibling`. Three pointers reach the whole tree.

### `index`

Position among siblings. Used by the diffing algorithm to detect *moves*:
if a fiber's old index is less than the "last placed index" so far, the node
must be moved in the DOM, and React marks `Placement`. (Full algorithm: Topic 06.)

### Why not a `children` array?

| array of children | child/sibling linked list |
|---|---|
| needs recursion or an explicit stack to walk | walk with 3 pointer reads |
| parent must be mutated to add a child | just set one pointer |
| can't resume mid-array without extra bookkeeping | `workInProgress` pointer *is* the bookkeeping |

The linked list is what makes the work loop a plain `while` with a single
resumable cursor.

---

## Group 3 — Props and state

### `pendingProps` vs `memoizedProps`

```
      render starts                      render finishes
            │                                   │
            ▼                                   ▼
   pendingProps = new props          memoizedProps = pendingProps
   memoizedProps = old props         (the new becomes the remembered)
```

This pair powers the **bailout check** at the top of `beginWork`:

```js
if (oldProps === newProps && !includesSomeLane(renderLanes, updateLanes)) {
  // props are referentially identical AND no pending state update
  return bailoutOnAlreadyFinishedWork(current, workInProgress, renderLanes);
}
```

`oldProps === newProps` is a **reference** check. On a normal re-render the
parent creates fresh prop objects, so this is almost always false — which is why
`React.memo` (shallow *value* comparison) exists as a separate opt-in.

### `memoizedState` — the most polymorphic field in React

| tag | `memoizedState` holds |
|---|---|
| `FunctionComponent` | **head of the hooks linked list** ← Topic 09 |
| `ClassComponent` | the state object (`this.state`) |
| `HostRoot` | `{ element }` — the root element |
| `SuspenseComponent` | `null` if showing content, `{ dehydrated, … }` if suspended |
| `HostComponent` | `null` |

For a function component:

```js
fiber.memoizedState
  → { memoizedState: 0,      queue, next } ──► useState(0)
  → { memoizedState: "",     queue, next } ──► useState("")
  → { memoizedState: {…},    queue, next } ──► useEffect
  → { memoizedState: [a,b],  queue, next } ──► useMemo
  → null
```

**A linked list with no names.** Hook #3 is found by walking `next` three times.
Nothing else. That one sentence is the entire explanation for the Rules of Hooks —
we prove it in Topic 09.

### `updateQueue` — also polymorphic

| tag | `updateQueue` holds |
|---|---|
| `ClassComponent` / `HostRoot` | circular linked list of `Update` objects |
| `FunctionComponent` | circular linked list of **effects** to fire at commit |
| `HostComponent` | the **prop diff payload** — a flat array `[key, value, key, value]` |

That last one is interesting: `completeWork` computes
`['className', 'active', 'title', 'hi']` and stores it here. The commit phase
just loops over it in pairs. The *diff* happens in the render phase; the
*apply* happens in commit. Clean separation.

### `dependencies`

A linked list of the contexts this fiber reads. When a Provider's value changes,
React walks the subtree and re-renders only fibers whose `dependencies` include
that context — instead of re-rendering everything. (Topic 13.)

---

## Group 4 — Effects: `flags`, `subtreeFlags`, `deletions`

### `flags` — a bitmask of "what to do at commit"

```js
export const NoFlags      = 0b0000000000000000000000000000;
export const Placement    = 0b0000000000000000000000000010;  // insert/move
export const Update       = 0b0000000000000000000000000100;  // update props
export const ChildDeletion= 0b0000000000000000000000010000;  // has deleted children
export const Ref          = 0b0000000000000000001000000000;  // attach/detach ref
export const Snapshot     = 0b0000000000000000010000000000;  // getSnapshotBeforeUpdate
export const Passive      = 0b0000000000000000100000000000;  // has useEffect
export const LayoutMask   = Update | Callback | Ref | Visibility;
```

Bitmask because a fiber can need several things at once:

```js
fiber.flags |= Placement;           // set
fiber.flags |= Update;              // set another
if (fiber.flags & Placement) {…}    // test — one CPU instruction
```

### `subtreeFlags` — the optimisation that replaced the effect list

React 16/17 built a separate **linked list of fibers with effects** during render,
then commit walked that list. React 18 removed it: `completeWork` now **bubbles**
flags up:

```js
// in completeWork, for every child:
subtreeFlags |= child.subtreeFlags;
subtreeFlags |= child.flags;
workInProgress.subtreeFlags = subtreeFlags;
```

Commit then walks the real tree, and at each node asks:

```js
if ((fiber.subtreeFlags & MutationMask) !== NoFlags) {
   recurse into children     // something below needs work
} else {
   SKIP THE ENTIRE SUBTREE   // provably nothing to do
}
```

```
            App  subtreeFlags: Update
           ╱   ╲
     Header     Main  subtreeFlags: Update
   subtree:0      ╲
   ✂ SKIPPED       Text  flags: Update  ← the only real change
   (whole branch)
```

Why the change? The old flat effect list was a second structure to build and keep
in sync, it allocated on every render, and it lost the tree shape (so you couldn't
skip a branch). Bubbled flags cost nothing extra — `completeWork` was already
visiting every node on the way up.

### `deletions`

Deleted children are **not** in the new tree (that's what deleted means), so they
can't carry their own flag. The parent holds them:

```js
parentFiber.deletions = [removedChildFiber, anotherRemovedFiber];
parentFiber.flags |= ChildDeletion;
```

Commit reads this array to run cleanup functions, detach refs, and remove DOM nodes.

---

## Group 5 — Priority: `lanes`, `childLanes`

- **`lanes`** — pending work **on this fiber** (e.g. a `setState` was queued here).
- **`childLanes`** — OR of all pending lanes **in the subtree below**.

When `setState` fires, React walks `return` pointers to the root, OR-ing the lane
into every ancestor's `childLanes`:

```
root        childLanes |= SyncLane   ▲
  App       childLanes |= SyncLane   │  bubble up
    Main    childLanes |= SyncLane   │
      Counter  lanes   |= SyncLane   ●  setState happened here
```

Then on the way **down**, `beginWork` can ask:

```js
if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
  return null;   // nothing below needs work at this priority — skip subtree
}
```

`childLanes` is to the render phase what `subtreeFlags` is to the commit phase:
**a summary on the parent that lets you skip a whole branch without entering it.**

That symmetry is one of the nicer pieces of design in the codebase.

---

## Group 6 — `mode`

Another bitmask, set at root creation and **inherited by every child**:

```js
NoMode            = 0b000000;
ConcurrentMode    = 0b000001;   // createRoot (not legacy ReactDOM.render)
ProfileMode       = 0b000010;   // inside <Profiler>
StrictLegacyMode  = 0b001000;   // inside <StrictMode>
StrictEffectsMode = 0b010000;   // StrictMode in React 18+: double-invoke effects
```

This is how `<StrictMode>` affects only its subtree, and how React decides whether
to double-invoke your component or time it for the profiler.

---

## Group 7 — `alternate`

The whole of Topic 05, previewed here:

```
        CURRENT TREE                      WORK-IN-PROGRESS TREE
        (on screen)                       (being built)

        ┌──────────┐    alternate     ┌──────────┐
        │   App    │ ◄──────────────► │   App'   │
        └────┬─────┘                  └────┬─────┘
             │                             │
        ┌────▼─────┐    alternate     ┌────▼─────┐
        │  Counter │ ◄──────────────► │ Counter' │
        └──────────┘                  └──────────┘

   root.current ──► App                 (after commit, this flips)
```

Every fiber has at most **two** versions, pointing at each other. React never
allocates a third. It builds the new tree by cloning into the alternate,
and at commit it flips one pointer: `root.current = finishedWork`.

Reusing two objects forever instead of allocating a fresh tree each render is a
deliberate GC optimisation — the fiber tree is the hot structure in React.

---

## A real fiber, filled in

```jsx
function Counter({ step }) {
  const [n, setN] = useState(0);
  return <span className="big">{n}</span>;
}
<Counter step={2} />
```

```js
// the Counter fiber
{
  tag: 0,                              // FunctionComponent
  key: null,
  elementType: Counter,                // the function
  type: Counter,                       // same (no wrapper)
  stateNode: null,                     // ← function components have no instance

  return:  <Main fiber>,
  child:   <span fiber>,
  sibling: null,
  index:   0,

  pendingProps:  { step: 2 },
  memoizedProps: { step: 2 },
  memoizedState: { memoizedState: 0, queue: {…}, next: null },   // ← useState(0)
  updateQueue:   null,

  flags:        0,
  subtreeFlags: 4,                     // Update — the span below changed
  lanes:        0,
  childLanes:   0,

  mode:      0b000001,                 // ConcurrentMode
  alternate: <the other Counter fiber>,
}

// the span fiber
{
  tag: 5,                              // HostComponent
  type: 'span',
  stateNode: <span class="big">,       // ← the REAL DOM node
  memoizedProps: { className: 'big', children: 0 },
  updateQueue: ['children', '1'],      // ← the prop diff payload
  flags: 4,                            // Update
}
```

---

## 🔧 Hands-on: inspect real fibers in your browser

The best reverse-engineering trick in React. Open any React app's devtools console:

```js
// 1. Grab any DOM node
const el = document.querySelector('#root > *');

// 2. React stores a back-pointer on it with a randomised key
const fiberKey = Object.keys(el).find(k => k.startsWith('__reactFiber$'));
const fiber = el[fiberKey];

// 3. You now have a real fiber. Explore it.
fiber.tag
fiber.type
fiber.memoizedProps
fiber.memoizedState        // ← walk .next to see every hook
fiber.return.type          // parent component
fiber.child.sibling.type   // second child
fiber.alternate            // the other tree's version
fiber.lanes
```

Walk a component's hooks:

```js
let hook = fiber.memoizedState;
while (hook) { console.log(hook.memoizedState); hook = hook.next; }
```

Walk up to the root:

```js
let f = fiber;
while (f.return) { console.log(f.tag, f.type?.name ?? f.type); f = f.return; }
console.log('root:', f.stateNode);        // the FiberRootNode
```

Do this once on a real app and fibers stop being abstract.

> Props are also exposed under a `__reactProps$…` key on the same DOM node.
> The random suffix is a per-bundle id, so React copies on the page don't collide.

---

## Field reference card

| Field | Type | Purpose |
|---|---|---|
| `tag` | number | what kind of fiber — drives the main switch |
| `key` | string/null | identity for child reconciliation |
| `elementType` | any | what the author wrote (memo/lazy wrapper included) |
| `type` | any | resolved thing to render |
| `stateNode` | any | DOM node / class instance / FiberRootNode / null |
| `return` | Fiber | parent — the return address |
| `child` | Fiber | first child |
| `sibling` | Fiber | next sibling |
| `index` | number | position among siblings; move detection |
| `ref` | ref | the ref to attach at commit |
| `pendingProps` | object | incoming props |
| `memoizedProps` | object | props from last completed render |
| `memoizedState` | any | hooks list / class state / suspense state |
| `updateQueue` | any | state updates / effects / prop-diff payload |
| `dependencies` | object | contexts this fiber reads |
| `mode` | bitmask | Concurrent / Strict / Profile, inherited |
| `flags` | bitmask | what commit must do to THIS fiber |
| `subtreeFlags` | bitmask | OR of descendants — lets commit skip branches |
| `deletions` | array | children removed this render |
| `lanes` | bitmask | pending work on this fiber |
| `childLanes` | bitmask | pending work below — lets render skip branches |
| `alternate` | Fiber | the other version of this fiber |

---

## Key takeaways

- A fiber is **one plain object** holding identity, position, props, state,
  effects, and priority for one node in the tree.
- `tag` is a number because the render phase is one hot `switch`.
- `stateNode` is the bridge to the real world — and it is `null` for function
  components, which explains a lot of ref behaviour.
- `memoizedState` and `updateQueue` are **polymorphic**: their meaning depends
  entirely on `tag`. For function components, `memoizedState` is the hooks list.
- `subtreeFlags` (commit) and `childLanes` (render) are the same idea applied in
  two phases: **a summary on the parent that lets you skip an entire branch**.
- `alternate` means React only ever keeps **two** versions of any fiber. Next topic.

---

> next: **05 — Double buffering**: how the two trees are built, swapped, and reused.

# 03 — Stack Reconciler → Fiber

> **Part 1 · Topic 3** — [index](README.md) · prev: [02 — JSX and Elements](02-jsx-and-elements.md) · next: [Part 2 — The Fiber Engine](../part-2-fiber-engine/README.md)

---

## React 15: reconciliation *was* the call stack

Up to React 15, rendering was a **recursive function call**:

```js
function mountComponent(element, parentDOM) {
  const instance = instantiate(element);
  const rendered = instance.render();          // call your component

  for (const child of rendered.children) {
    mountComponent(child, dom);                // ← recursion
  }

  parentDOM.appendChild(dom);
}
```

The tree traversal and the JavaScript call stack were **the same thing**.
Hence the name: the **stack reconciler**.

```
        mountComponent(<App/>)          ─┐
          mountComponent(<Header/>)      │  JS call stack
            mountComponent(<Logo/>)      │  grows with tree depth
            mountComponent(<Nav/>)       │
          mountComponent(<Main/>)        │  ← you are 400 frames deep
            mountComponent(<List/>)      │     and cannot stop
              mountComponent(<Row/>) ×500│
        ────────────────────────────────┘
```

It worked. It was fast. And it had one fatal property:

> ### You cannot pause the JavaScript call stack.

There is no `pause()`. No `yield`. No way to say *"I've been working 8ms, let the
browser breathe, I'll resume from this exact node."* Once `mountComponent(root)`
is entered, it runs to completion. Period.

---

## Why that's fatal

The browser main thread is **one thread** doing everything:

```
 ONE FRAME (~16.6ms at 60fps)
┌──────────┬────────┬────────┬───────┬────────────┐
│    JS    │ style  │ layout │ paint │ composite  │
└──────────┴────────┴────────┴───────┴────────────┘
     ↑
   React lives here, together with your event handlers,
   animations, and every other script on the page.
```

If React's recursion takes 250ms:

```
│←──────────────── React rendering (250ms) ────────────────→│ paint │
  ✗ keystroke       ✗ click         ✗ animation frame
    ignored           queued          dropped
```

To the user the page is **frozen**. Not slow — broken.

### The real problem is greed, not speed

React 15 wasn't slow. It was **greedy**: it took the thread and refused to give
it back. And it had no concept of importance — a keystroke in a search box and
an off-screen chart re-render were treated as equally urgent, because once the
recursion began, *everything in it was one indivisible unit of work*.

```
What we had:          ████████████████████████  one atomic blob
What we needed:       ██ ░ ██ ░ ██ ░ ██ ░ ██    slices, with gaps
                          ↑ browser breathes here
```

---

## The insight

> If you cannot pause the native call stack — **build your own stack on the heap.**

What *is* a call stack? A linked list of frames. Each frame holds:

| Stack frame holds | React equivalent |
|---|---|
| which function is executing | `type` |
| its arguments | `pendingProps` |
| its local variables | `memoizedState` (hooks live here) |
| the return address | `return` ← *literally this name* |

The browser keeps that in native memory you can't touch. So React **reified** it:
it turned a stack frame into a **plain JavaScript object**.

### That object is a Fiber.

```js
// A fiber ≈ a stack frame React owns
{
  type:          MyComponent,  // which function is "executing"
  pendingProps:  { id: 7 },    // its arguments
  memoizedState: hookList,     // its local state
  return:        parentFiber,  // where to return to  ← stack semantics
  child:         firstChild,   // where to go next
  sibling:       nextSibling,  // the next frame at this level
  stateNode:     domNode,      // the host instance, if any
}
```

Note: it is `return`, **not `parent`**. Same node either way — but the name tells
you the intent. It is a return address.

### Where the name comes from

In computer science a **fiber** is a lightweight unit of execution that is
**cooperatively scheduled** — it yields control voluntarily — as opposed to a
**thread**, which the OS **preemptively** interrupts.

React's fibers are exactly that: units of rendering work that voluntarily hand
the main thread back to the browser. The name is a precise technical reference.

---

## Recursion → iteration over a linked tree

Because the tree is now a data structure with explicit pointers, you can walk it
with a `while` loop instead of recursion — and a `while` loop can **break**.

### The pointer structure

```
                    ┌──────────┐
                    │   App    │
                    └────┬─────┘
                   child │  ▲ return
                         ▼  │
                    ┌──────────┐  sibling   ┌──────────┐  sibling  ┌──────────┐
                    │  Header  │ ─────────► │   Main   │ ────────► │  Footer  │
                    └────┬─────┘            └────┬─────┘           └──────────┘
                   child │                 child │
                         ▼                       ▼
                    ┌──────────┐            ┌──────────┐
                    │   Logo   │            │   List   │
                    └──────────┘            └──────────┘
```

Only **three pointers** per node — `child`, `sibling`, `return` — and you can
reach every node from any node. A singly-linked tree, traversable without
recursion and without an auxiliary stack array.

### The traversal order

```
      App ──1──► Header ──2──► Logo ──3──┐
                                          │ (no child, no sibling → return)
       ┌──────────◄───────────────────────┘
       ▼
    Header completes ──4──► Main ──5──► List ──6──┐
                                                   │
       ┌───────────────◄───────────────────────────┘
       ▼
    Main completes ──7──► Footer ──8──► (completes)
       │
       └──9──► App completes → DONE
```

Rule: **go down while you can; when you can't, complete and go sideways;
if you can't go sideways, complete and go up.**

### The work loop

```js
let workInProgress = null;   // ← module-level. THE bookmark.

function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {   // ← the entire point
    workInProgress = performUnitOfWork(workInProgress);
  }
}

function performUnitOfWork(fiber) {
  const next = beginWork(fiber);     // "call" the frame: render it, make children
  if (next !== null) {
    return next;                     // descend into child
  }

  // no children → unwind
  let node = fiber;
  while (node !== null) {
    completeWork(node);              // "return" from the frame
    if (node.sibling !== null) {
      return node.sibling;           // go sideways
    }
    node = node.return;              // pop a frame
  }
  return null;                       // whole tree finished
}
```

Three things to see:

**① `shouldYield()`** — checks a clock. React takes roughly a **5ms** slice, then
hands the thread back. The browser paints and handles input, then re-invokes React.

**② `workInProgress` is a single module-level variable.** That one pointer is the
complete "where was I?" bookmark. Pause, resume, continue from exactly that node.
This is precisely what the native call stack could never give you.

**③ Work can be thrown away.** If a keystroke arrives mid-render, React can discard
the entire half-built tree and restart at the higher priority. Nothing was committed,
so the user never sees the abandoned work.

---

## Yielding, visualised

```
 React 15 (stack reconciler)
 ├────────────────── render 250ms ──────────────────┤ paint
 ✗ input blocked the whole time

 React 18+ (fiber + concurrent)
 ├─5ms─┤   ├─5ms─┤   ├─5ms─┤   ├─5ms─┤   ├─5ms─┤   ┤ commit+paint
        ↕         ↕         ↕         ↕
     browser   browser   browser   browser
     handles   paints    handles   paints
     input               input
```

And with an interruption:

```
 ├─5ms─┤  ├─5ms─┤  ├─5ms──╳  KEYSTROKE ARRIVES
                          │
                          └─► discard WIP tree
                              render keystroke update (urgent)
                              commit it  ──► user sees their character instantly
                              THEN restart the low-priority work from scratch
```

---

## The consequence you already live with: two phases

Because render work can be **paused, resumed, abandoned, or re-run**, React had
to split rendering in two.

```
┌─────────────────────── RENDER PHASE ──────────────────────┐
│  beginWork / completeWork over the fiber tree             │
│                                                           │
│  • interruptible          • may run 2+ times              │
│  • may be thrown away     • NOTHING is visible yet        │
│  • MUST BE PURE                                           │
└───────────────────────────────┬───────────────────────────┘
                                │  (tree is complete)
                                ▼
┌─────────────────────── COMMIT PHASE ──────────────────────┐
│  mutate the DOM, attach refs, run effects                 │
│                                                           │
│  • synchronous            • runs exactly once             │
│  • uninterruptible        • side effects belong HERE      │
│  • atomic — user sees all of it or none of it             │
└───────────────────────────────────────────────────────────┘
```

| | Render phase | Commit phase |
|---|---|---|
| Interruptible | **yes** | no |
| Can run twice | **yes** | no |
| Can be discarded | **yes** | no |
| Side effects allowed | **no** | yes |
| DOM touched | no | yes |

### This table is the origin of half of React's "rules"

| Rule you know | Why, mechanically |
|---|---|
| "Components must be pure" | The function body is render-phase code. It may run twice and be discarded. A `fetch()` there fires unpredictably. |
| StrictMode double-invokes components in dev | React *deliberately* renders twice to surface impurity — because in concurrent mode it genuinely might. |
| `useEffect` exists | It is the escape hatch that defers your side effect to the **commit** phase, where "runs once" is guaranteed. |
| Don't mutate refs during render | State updates are queued and replayable; ref mutations are not — a discarded render would leave them applied. |
| Don't `setState` during another component's render | It would invalidate work already done in a phase that's supposed to be pure. |

---

## The timeline nobody mentions

| Year | Version | What happened |
|---|---|---|
| 2017 | **16** | Fiber ships. `shouldYield()` always returns false → fully synchronous. |
| 2020 | 17 | No user-facing features. Event delegation moved to root. Stepping stone. |
| 2022 | **18** | Concurrency turned **on**: `createRoot`, transitions, automatic batching, Suspense SSR. |
| 2024 | 19 | Built on it further: Actions, `use`, Server Components. |

**Five years between shipping the engine and shipping the feature.** React 16 and
17 ran Fiber in synchronous mode — the architecture was in place and stabilising
in production while the capability stayed switched off.

That is the real lesson of Fiber: they did not add concurrency to React.
They spent two years rewriting the engine so that concurrency *became possible*,
shipped the rewrite invisibly, and turned it on years later.

---

## Key takeaways

- The stack reconciler was recursive, so rendering could never be interrupted.
- A **Fiber is a reified stack frame** — a heap object holding what a call frame holds.
- `child` / `sibling` / `return` replace recursion with an interruptible `while` loop.
- `workInProgress` is the single pointer that makes pause/resume possible.
- Interruptibility forced the **render / commit** split, which in turn is *why*
  components must be pure and why `useEffect` exists.
- Fiber shipped in 16 (2017); concurrency shipped in 18 (2022).

---

> next: [Part 2 — The Fiber Engine](../part-2-fiber-engine/README.md) — we open a fiber up, field by field.

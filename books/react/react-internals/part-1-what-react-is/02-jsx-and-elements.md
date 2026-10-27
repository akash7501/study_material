# 02 — JSX → createElement → Elements

> **Part 1 · Topic 2** — [index](README.md) · prev: [01 — Package architecture](01-package-architecture.md) · next: [03 — Stack reconciler → Fiber](03-stack-reconciler-to-fiber.md)

---

## JSX is not part of React

JSX is a **syntax extension to JavaScript**. No browser understands it.
It is erased at build time by Babel / SWC / esbuild / TypeScript.

By the time your code runs, there is no JSX. There are only function calls.

---

## The two transforms

### Classic runtime (React ≤ 16, still supported)

```jsx
<div className="box">Hello</div>
```
compiles to:
```js
React.createElement("div", { className: "box" }, "Hello")
```

This is why old codebases needed `import React from 'react'` in every file even
when `React` was never written — the compiled output referenced it.

### Automatic runtime (React 17+, the default today)

```jsx
<div className="box">Hello</div>
```
compiles to:
```js
import { jsx as _jsx } from "react/jsx-runtime";

_jsx("div", { className: "box", children: "Hello" })
```

Three differences that matter:

| | Classic | Automatic |
|---|---|---|
| Import needed | `import React` manually | compiler injects `react/jsx-runtime` |
| `children` | third+ arguments | **inside props** |
| `key` | a regular prop, extracted later | **separate third argument** |

> **`key` being a separate argument is why `props.key` is always `undefined`.**
> It never enters props. It's metadata for the reconciler, not data for your component.

There are two functions: `jsx(type, props, key)` for the common case and
`jsxs(...)` for static children arrays (a micro-optimisation so React can skip
key-warning checks it knows are unnecessary).

---

## What actually comes out

`jsx()` barely does anything. It builds and freezes a plain object:

```js
{
  $$typeof: Symbol.for('react.transitional.element'),
  //        ^ React 19. Was Symbol.for('react.element') in 16–18.

  type:  "div",          // string      → host element
                         // function    → function component
                         // class       → class component
                         // Symbol      → Fragment / Suspense / Portal / Profiler
                         // object      → memo / lazy / forwardRef / Context

  key:   null,
  props: { className: "box", children: "Hello" },

  _owner: null,          // DEV only — which component created this
  _store: {},            // DEV only — key-warning bookkeeping
}
```

**That is a React Element.** And that is the entire output of JSX.

At this point:
- **No DOM node was created.**
- **No component function was called.**
- **Nothing was rendered.**

### Nesting is just nesting

```jsx
<ul>
  <li>a</li>
  <li>b</li>
</ul>
```
→
```js
{
  type: "ul",
  props: {
    children: [
      { type: "li", props: { children: "a" } },
      { type: "li", props: { children: "b" } },
    ]
  }
}
```

A tree of objects. That's the whole "virtual DOM" — a somewhat unfortunate name,
because it is not a DOM and React core doesn't use the term.

### `<MyComponent />` holds a *reference*

```jsx
<MyComponent userId={7} />
```
→
```js
{ type: MyComponent, props: { userId: 7 } }
```

`type` is **your actual function object**, uncalled. React decides later —
at a time and priority of its choosing, possibly never, possibly twice — whether
to invoke it. That deferral is the whole basis of concurrent rendering.

---

## Three things this object shape explains

### ① `$$typeof` is a Symbol for security

Symbols cannot survive JSON serialisation. `JSON.parse` can never produce one.

So if a server does this:

```js
const message = JSON.parse(untrustedUserInput);
return <div>{message}</div>;
```

an attacker can craft a payload that *looks* like an element:

```json
{ "type": "div", "props": { "dangerouslySetInnerHTML": { "__html": "<script>…" } } }
```

…but **cannot forge `$$typeof`**, because JSON has no symbol type. React checks
the field before treating an object as an element, refuses, and the attack dies.

(Fallback: in environments without `Symbol`, React uses the number `0xeac7` —
which spells "React" in leetspeak. The protection is weaker there.)

### ② `<Comp />` is not `Comp()`

```jsx
function Parent() {
  return <Child />;    // → element { type: Child }. React owns the call.
}

function Parent() {
  return Child();      // → Child's output, spliced inline into Parent.
}
```

| | `<Child />` | `Child()` |
|---|---|---|
| Gets its own Fiber | ✅ | ❌ |
| Gets its own hook list | ✅ | ❌ — hooks become **Parent's** hooks |
| Can hold independent state | ✅ | ❌ |
| `React.memo` can skip it | ✅ | ❌ |
| Can be a Suspense/error boundary child | ✅ | ❌ |

This is the precise, mechanical version of *"components are not functions you call."*
They are functions **React** calls — because React must wrap each invocation in a
Fiber to give it identity.

### ③ Elements are cheap on purpose

Every render allocates a brand-new tree of these objects. That sounds wasteful,
and it is — slightly. But they are tiny, short-lived, and extremely
GC-friendly.

The expensive operation is **touching the DOM** (layout, style recalc, paint).
The element tree exists so React can **compare before it touches**. Trading cheap
object allocation for avoided DOM work is the core bet of the whole library.

---

## Special `type` values

Not every `type` is a string or your function:

```js
<></>                 → type: Symbol.for('react.fragment')
<Suspense>            → type: Symbol.for('react.suspense')
createPortal(...)     → $$typeof: Symbol.for('react.portal')
memo(Comp)            → type: { $$typeof: Symbol(react.memo),    type: Comp }
forwardRef(fn)        → type: { $$typeof: Symbol(react.forward_ref), render: fn }
lazy(() => …)         → type: { $$typeof: Symbol(react.lazy),    _payload, _init }
<Ctx.Provider>        → type: { $$typeof: Symbol(react.provider), _context: Ctx }
```

The reconciler reads `type`, maps it to a **fiber tag** (a number), and switches
on that tag to decide how to process the node. We'll see the full tag list in Topic 04.

---

## React 19 changes worth knowing

- `$$typeof` is now `Symbol.for('react.transitional.element')` (a migration marker
  while React moves toward a new element shape).
- **`ref` is a normal prop** for function components. `element.ref` no longer exists
  as a top-level field — reading it warns. `forwardRef` is no longer needed for
  new code.

---

## The three-tree model

```
   Elements        →        Fibers        →      Host instances
 what you                React's internal         real DOM nodes
 described               work units + state

 immutable               mutable, persistent      mutable
 thrown away and         reused across            only touched when a
 rebuilt each render     renders                  diff demands it
```

Every render:
**new element tree** → reconciler diffs it against the **existing fiber tree**
→ produces a minimal list of **host mutations** → commits them.

Most "weird React behaviour" is a question about the middle column.

---

## Key takeaways

- JSX is compile-time sugar. At runtime there is only `jsx()` / `createElement()`.
- An Element is a **frozen plain object**: `{ $$typeof, type, key, props }`.
- Creating an element **calls nothing and renders nothing**.
- `$$typeof` as a Symbol is a real XSS mitigation.
- `key` lives outside props because it is reconciler metadata, not component data.
- `<Comp />` creates a Fiber-owned boundary; `Comp()` does not.

---

> next: [03 — Stack reconciler → Fiber](03-stack-reconciler-to-fiber.md)

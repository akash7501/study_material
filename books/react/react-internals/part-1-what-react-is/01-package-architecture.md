# 01 — Package Architecture: React Is Not One Library

> **Part 1 · Topic 1** — [index](README.md) · next: [02 — JSX and Elements](02-jsx-and-elements.md)

---

## The claim that breaks most mental models

**The `react` package cannot render anything.**

It has no idea what a DOM node is. It contains no code that touches a screen.
If you install only `react` and call `createElement`, you get a JavaScript object
and nothing else happens.

React is actually **four separate packages** with sharply divided jobs.

---

## The four packages

```
┌─────────────────────────────────────────────────────────────┐
│  react                                                      │
│  Describes UI. createElement, Component, hooks (as stubs),  │
│  Context, memo, lazy, Fragment.                             │
│  Knows nothing about DOM, native, canvas, or terminals.     │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│  react-reconciler                                           │
│  THE ENGINE. Fiber tree, diffing, hooks implementation,     │
│  Suspense, context propagation, error boundaries.           │
│  Host-agnostic — parameterised by a "host config".          │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│  scheduler                                                  │
│  Cooperative time-slicing. Priority queue + MessageChannel. │
│  Knows nothing about UI at all — it is a generic            │
│  "run this callback soon, at this priority" library.        │
└─────────────────────────────────────────────────────────────┘
                            │
┌─────────────────────────────────────────────────────────────┐
│  react-dom                                                  │
│  A HOST CONFIG. Teaches the reconciler about the DOM.       │
│  Plus DOM-only extras: synthetic events, hydration,         │
│  attribute/property handling, `dangerouslySetInnerHTML`.    │
└─────────────────────────────────────────────────────────────┘
```

---

## What a "host config" is

The reconciler never calls `document.createElement`. It calls **you** — or rather,
it calls an object of primitive operations that the renderer supplies.

```js
const hostConfig = {
  // creating
  createInstance(type, props)       { return document.createElement(type) },
  createTextInstance(text)          { return document.createTextNode(text) },

  // tree surgery
  appendChild(parent, child)        { parent.appendChild(child) },
  insertBefore(parent, child, before) { parent.insertBefore(child, before) },
  removeChild(parent, child)        { parent.removeChild(child) },

  // updating
  prepareUpdate(instance, type, oldProps, newProps) { /* compute diff */ },
  commitUpdate(instance, payload)                   { /* apply diff */ },

  // scheduling / environment
  getCurrentEventPriority() { /* ... */ },
  supportsMutation: true,
  // ...~30 methods total
};

const MyRenderer = ReactReconciler(hostConfig);
```

That is the entire contract. Swap the host config, and the *same* reconciler
drives something else:

| Renderer | What `createInstance` returns |
|---|---|
| `react-dom` | a DOM element |
| `react-native` | a native view handle (iOS/Android) |
| `react-three-fiber` | a Three.js object3D |
| `ink` | a terminal text box |
| `react-pdf` | a PDF drawing primitive |
| `react-test-renderer` | a plain JSON object |

**All of them have identical hooks, identical keys, identical Suspense,
identical batching** — because all of that lives one layer up, in the reconciler.

---

## Why `react` has hooks that don't do anything

Open the React source and `useState` is roughly this:

```js
export function useState(initialState) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState);
}
```

That's the whole implementation in the `react` package. It's a **forwarder**.
The real `useState` lives in the reconciler, and the reconciler *installs itself*
into a shared mutable slot (`ReactCurrentDispatcher.current`) right before it
calls your component.

This indirection is why:

- You can import `useState` from `react` but it behaves differently on mount vs update
  (the reconciler swaps in a different dispatcher object for each case).
- Calling a hook outside a component throws **"Invalid hook call"** — the dispatcher
  slot is `null` because no reconciler is mid-render.
- Having **two copies of `react`** in `node_modules` breaks everything — one copy's
  dispatcher slot gets filled, the other copy's stays empty.

We come back to this properly in Topic 09.

---

## Why they split it this way

Three real reasons, not architectural purity:

**1. One engine, many targets.** React Native was the forcing function. Facebook
did not want two reconcilers to keep in sync.

**2. Bundle size.** A web app shouldn't ship native code paths. The split lets
the bundler drop what the host doesn't need.

**3. The engine can be rewritten without breaking the API.** This is the big one.
React 16 replaced the entire reconciler (stack → Fiber) and almost no application
code changed, because applications depend on `react`, not on the engine.

---

## The practical takeaway

When you learn a React behaviour, immediately ask: **which layer owns this?**

| Behaviour | Lives in | Therefore… |
|---|---|---|
| `key` on a list | reconciler | works identically in React Native |
| Hooks rules | reconciler | universal |
| Automatic batching | reconciler | universal |
| Suspense | reconciler | universal |
| `onClick` synthetic events | react-dom | **web only** — RN has its own system |
| `className` vs `class` | react-dom | web only |
| Hydration | react-dom | web only |
| Time slicing / yielding | scheduler | depends on the environment's task queue |

---

## Key takeaways

- `react` = a **description** library. `react-dom` = a **renderer**. The real
  algorithm is in `react-reconciler`, which neither of them is.
- A renderer is defined by a ~30-method **host config** object.
- Hooks in the `react` package are forwarders to a dispatcher the reconciler installs.
- This separation is why the engine could be completely rewritten twice without
  breaking userland code.

---

> next: [02 — JSX → createElement → Elements](02-jsx-and-elements.md)

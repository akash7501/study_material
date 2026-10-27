# React Under The Hood — Reverse Engineering React

A top-down deep dive into **how React is built** and **how it works internally**.
Not how to *use* React — how React itself works.

Target version: **React 19** (differences from 17 / 18 are called out where relevant).

---

## How this folder is organised

```
react-internals/
├── README.md                     ← you are here (master index + progress)
├── 00-the-big-picture.md         ★ START HERE — full pipeline diagram,
│                                    JSX → JS → Elements → Fibers → DOM
│
├── part-1-what-react-is/         Foundations: packages, JSX, why Fiber exists
├── part-2-fiber-engine/          The Fiber data structure + render/commit pipeline
├── part-3-state/                 Hooks internals, update queues, batching
├── part-4-concurrency/           Scheduler, lanes, Suspense, events
│
├── mini-react/                   Hands-on: build a working ~300-line React clone
└── interview-prep/               Same concepts, in very simple English
```

### Two tracks, read whichever you need

| Track | Folders | Style |
|---|---|---|
| **Deep track** | `part-1` … `part-4` | Full detail, real source names, diagrams |
| **Simple track** | `interview-prep/` | Same ideas in plain English, for speaking out loud |

They are kept separate on purpose. The deep notes stay accurate; the interview
notes stay simple. Read the deep one to *understand*, the simple one to *explain*.

### Diagrams

Every topic file contains diagrams:
- **ASCII block diagrams** — work in any editor, any terminal, GitHub, Notepad
- **Mermaid flow / state diagrams** — render automatically in VS Code
  (Markdown Preview), GitHub, and Obsidian

The master pipeline, flow chart, and update state machine all live in
[`00-the-big-picture.md`](00-the-big-picture.md).

**Naming convention:** `NN-topic-name.md` where `NN` is the global topic number
(01–14). The number never changes, so links stay stable even if a file moves.

Each part folder has its own `README.md` acting as a local index.

---

## Progress

**Legend:** `[x]` written · `[ ]` pending

### Overview
- [x] [00 — The Big Picture](00-the-big-picture.md) — the whole pipeline in diagrams

### Part 1 — What React Actually Is
- [x] [01 — Package architecture](part-1-what-react-is/01-package-architecture.md)
- [x] [02 — JSX → createElement → Elements](part-1-what-react-is/02-jsx-and-elements.md)
- [x] [03 — Stack reconciler → Fiber](part-1-what-react-is/03-stack-reconciler-to-fiber.md)

### Part 2 — The Fiber Engine
- [x] [04 — The Fiber node, field by field](part-2-fiber-engine/04-fiber-node.md)
- [x] [05 — Double buffering: `current` vs `workInProgress`](part-2-fiber-engine/05-double-buffering.md)
- [x] [06 — Render phase: `beginWork`, diffing, keys, bailouts](part-2-fiber-engine/06-render-phase.md)
- [x] [07 — Complete phase: `completeWork`, host instances, flag bubbling](part-2-fiber-engine/07-complete-phase.md)
- [x] [08 — Commit phase: three sub-phases, refs, effect ordering](part-2-fiber-engine/08-commit-phase.md)

### Part 3 — State
- [x] [09 — Hooks internals: the linked list and the dispatcher swap](part-3-state/09-hooks-internals.md) ⭐
- [ ] 10 — Updates & batching: update queues, eager state, React 18 batching

### Part 4 — Concurrency & The Rest
- [ ] 11 — Scheduler & Lanes: priority bitmasks, time slicing
- [ ] 12 — Suspense & transitions
- [ ] 13 — Context propagation & error boundaries
- [ ] 14 — The synthetic event system

### Side quests
- [ ] mini-react — build it alongside the theory
- [x] [interview-prep / Part 1 Q&A](interview-prep/part-1-qa.md) — 13 questions
- [x] [interview-prep / Part 2 Q&A](interview-prep/part-2-qa.md) — 38 questions (topics 04–08)
- [x] [interview-prep / Part 3 Q&A](interview-prep/part-3-qa.md) — 13 questions (topic 09) ⭐

---

## The one idea to hold onto

Every React behaviour lives in one of two places:

| Lives in the **reconciler** | Lives in the **host config** |
|---|---|
| Universal across all renderers | DOM-specific |
| keys, hooks, Suspense, batching, lanes | `className`, synthetic events, hydration |

Knowing which half a feature belongs to is most of the insight.

---

## The three-tree model

Keep this picture in your head for the whole series:

```
Elements          →      Fibers              →      Host instances
(what you                (React's internal          (real DOM nodes)
 described)               work units + state)

immutable                mutable, persistent        mutable
recreated every          reused across renders      only touched when a
render                                              diff demands it
```

Almost every confusing React behaviour — state resetting, effects not re-running,
inputs losing focus — is a question about the **middle column**.

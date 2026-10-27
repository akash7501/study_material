# Part 3 — State

How React remembers things between renders, and how updates travel.

| # | Topic | File | Status |
|---|---|---|---|
| 09 | Hooks internals: the linked list + dispatcher swap | [09-hooks-internals.md](09-hooks-internals.md) | **✓ written** |
| 10 | Updates & batching: update queues, eager state, React 18 batching | `10-updates-and-batching.md` | pending |

**Prev:** [Part 2 — The Fiber Engine](../part-2-fiber-engine/README.md)
**Next:** [Part 4 — Concurrency](../part-4-concurrency/README.md)

---

## What this part answers

1. Where is `useState`'s value physically stored?
2. Why are hooks a linked list and not an array or an object?
3. What is the "dispatcher" and why does it get swapped on every render?
4. Why do the Rules of Hooks exist? (Spoiler: they are a *consequence* of the
   data structure, not a style rule.)
5. Why does calling a hook outside a component throw "Invalid hook call"?
6. What exactly does `setState` do — and why is it async-looking?
7. What is the eager-state bailout that makes `setCount(same)` free?
8. What changed about batching in React 18, and why did it change?
9. Why does the updater form `setCount(c => c + 1)` behave differently?

Plain-English versions: [../interview-prep/part-3-qa.md](../interview-prep/part-3-qa.md)
⭐ The most-asked internals topic in React interviews.

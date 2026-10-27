# Part 4 — Concurrency & The Rest

The features Fiber was built for, plus the remaining subsystems.

| # | Topic | File | Status |
|---|---|---|---|
| 11 | Scheduler & Lanes: priority bitmasks, time slicing | `11-scheduler-and-lanes.md` | pending |
| 12 | Suspense & transitions | `12-suspense-and-transitions.md` | pending |
| 13 | Context propagation & error boundaries | `13-context-and-errors.md` | pending |
| 14 | The synthetic event system | `14-event-system.md` | pending |

**Prev:** [Part 3 — State](../part-3-state/README.md)

---

## What this part answers

1. Why is priority a **bitmask** (lanes) and not a number? What did lanes fix
   that the old `expirationTime` model could not?
2. Why `MessageChannel` instead of `setTimeout` or `requestIdleCallback`?
3. How does `shouldYield()` decide when to stop?
4. How does Suspense work mechanically — what does "throwing a promise" mean?
5. What does `startTransition` actually change about an update?
6. How does `useDeferredValue` produce two renders?
7. How does context propagate without re-rendering every component?
8. How does an error unwind the fiber tree to the nearest boundary?
9. Why does React attach one listener at the root instead of per element?
10. How does the event system assign priority per event type?

Plain-English versions: `../interview-prep/part-4-qa.md` *(pending)*

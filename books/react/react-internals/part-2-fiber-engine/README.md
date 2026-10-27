# Part 2 — The Fiber Engine

The core of React. This is where reconciliation actually happens.

| # | Topic | File | Status |
|---|---|---|---|
| 04 | The Fiber node, field by field | [04-fiber-node.md](04-fiber-node.md) | **✓ written** |
| 05 | Double buffering: `current` vs `workInProgress` | [05-double-buffering.md](05-double-buffering.md) | **✓ written** |
| 06 | Render phase: `beginWork`, diffing, keys, bailouts | [06-render-phase.md](06-render-phase.md) | **✓ written** |
| 07 | Complete phase: `completeWork`, host instances, flags | [07-complete-phase.md](07-complete-phase.md) | **✓ written** |
| 08 | Commit phase: three sub-phases, refs, effect order | [08-commit-phase.md](08-commit-phase.md) | **✓ written** |

**🏁 Part 2 is complete.**

**Prev:** [Part 1 — What React Actually Is](../part-1-what-react-is/README.md)
**Next:** [Part 3 — State](../part-3-state/README.md)

---

## What this part answers

1. What are all the fields on a Fiber and why does each one exist?
2. What is the `alternate` pointer and why are there always two trees?
3. How does React decide to reuse a fiber vs destroy and recreate it?
4. What does `key` actually do inside the diffing algorithm?
5. Why is the list diff O(n) and not O(n³)?
6. What is a "bailout" and when does React skip an entire subtree?
7. Where exactly do DOM nodes get created — and why bottom-up?
8. What are effect flags and why are they bubbled up the tree?
9. What is the exact order that `useLayoutEffect`, `useEffect`, refs, and
   `componentDidMount` fire in?

Plain-English versions: [../interview-prep/part-2-qa.md](../interview-prep/part-2-qa.md)
*(currently covers Topics 04–05; grows as we write 06–08)*

# Part 1 — What React Actually Is

Foundations. Before touching the engine, we establish three things:
what the packages are, what JSX produces, and why the engine was rewritten.

| # | Topic | File |
|---|---|---|
| 01 | Package architecture — why React is four libraries, not one | [01-package-architecture.md](01-package-architecture.md) |
| 02 | JSX → `createElement` → Elements | [02-jsx-and-elements.md](02-jsx-and-elements.md) |
| 03 | Stack reconciler → Fiber — why recursion was a dead end | [03-stack-reconciler-to-fiber.md](03-stack-reconciler-to-fiber.md) |

**Next:** [Part 2 — The Fiber Engine](../part-2-fiber-engine/README.md)

---

## What you should be able to answer after Part 1

1. Why can't the `react` package render anything by itself?
2. What exactly does `<div />` evaluate to at runtime?
3. Why is `$$typeof` a Symbol and not a string?
4. Why is `<Comp />` different from `Comp()`?
5. Why could the React 15 reconciler never be made interruptible?
6. What is a Fiber, in one sentence?
7. Why must component functions be pure?

Plain-English answers: [interview-prep/part-1-qa.md](../interview-prep/part-1-qa.md)

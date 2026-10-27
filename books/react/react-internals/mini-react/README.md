# mini-react — Build React From Scratch

Theory tells you *what*. Writing it tells you *why*.

The plan: build a working React clone in ~300 lines, growing it one step at a
time alongside the main notes. No build tools, no dependencies — a single HTML
file you open in a browser.

---

## Build order

Each step is runnable. You never have a broken half-thing.

| Step | Build | Unlocks | After reading |
|---|---|---|---|
| 1 | `createElement()` | element objects | Topic 02 |
| 2 | `render()` — naive recursive | something on screen | Topic 02 |
| 3 | The fiber node + work loop | pausable rendering | Topics 03–04 |
| 4 | `requestIdleCallback` slicing | you can *see* it yield | Topic 03 |
| 5 | Render / commit split | nothing half-painted | Topic 03 |
| 6 | Reconciliation + `alternate` | updates instead of rebuilds | Topics 05–06 |
| 7 | Deletions + effect flags | removing nodes works | Topics 06–07 |
| 8 | Function components | `<App />` works | Topic 06 |
| 9 | `useState` | real state | Topic 09 |
| 10 | `useEffect` | side effects after commit | Topics 08–09 |

---

## Status

Not started. We begin this at **Topic 04** — the first thing you write in a
React clone is the fiber structure, so that is the natural entry point.

---

## Files (planned)

```
mini-react/
├── README.md          ← you are here
├── step-01-elements.html
├── step-02-naive-render.html
├── step-03-fibers.html
├── ...
└── step-10-complete.html
```

Each file is standalone and self-contained — open it in a browser, open devtools,
and step through it. Copying forward and changing one thing per step is the point:
you see exactly which line buys which capability.

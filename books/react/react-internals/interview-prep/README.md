# Interview Prep — React Internals in Simple English

**This folder is different from the rest.**

The main notes explain React **deeply and correctly**.
This folder explains the **same things in very simple English** — the way you
would actually say them out loud to an interviewer.

---

## The rule for this folder

> No heavy words. No long sentences. Use everyday examples.
> If a 10-year-old cannot follow it, rewrite it.

Every answer has three layers:

| Layer | What it is | When to use it |
|---|---|---|
| **🗣 SAY THIS** | 2–4 simple sentences | Your first answer. Always start here. |
| **➕ IF THEY ASK MORE** | A bit more detail | When they say "can you explain more?" |
| **🎯 ONE LINE TO REMEMBER** | The idea in one sentence | Your memory hook |

---

## Files

| File | Covers |
|---|---|
| [part-1-qa.md](part-1-qa.md) | Packages, JSX, Elements, why Fiber exists |
| part-2-qa.md | Fiber node, two trees, render & commit *(pending)* |
| part-3-qa.md | Hooks internals, batching *(pending)* |
| part-4-qa.md | Scheduler, lanes, Suspense, events *(pending)* |

---

## 5 tips for answering internals questions

**1. Start short.** Give 3 sentences, then stop. Let them ask for more.
A long answer sounds like you memorised it. A short, confident answer sounds
like you understand it.

**2. Use a real-life example.** "React is like a builder who makes a plan on
paper first, then builds the wall." Interviewers remember examples.

**3. Say "why", not just "what".** Everyone can say *"Fiber makes rendering
interruptible."* Very few can say *"because the JavaScript call stack cannot be
paused, so React built its own stack out of objects."* That is the answer that
gets you hired.

**4. It is fine to say "I am not sure".** Then add what you *do* know.
"I am not fully sure about the exact function name, but the idea is…"

**5. Connect internals to real problems.** If you can say *"this is why your
input loses focus when you change the key"*, you sound like someone who has
actually debugged React, not someone who read a blog.

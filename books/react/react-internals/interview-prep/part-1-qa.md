# Part 1 — Interview Q&A (Simple English)

Covers: packages, JSX, Elements, and why Fiber exists.
Deep version: [part-1 notes](../part-1-what-react-is/README.md)

---

## Q1. What is React, really?

**🗣 SAY THIS**

> React is a library that helps me describe what the screen should look like.
> I do not touch the DOM myself. I just tell React "this is how it should look",
> and React finds out what changed and updates only that part.

**➕ IF THEY ASK MORE**

> React is actually split into separate packages. The `react` package only
> *describes* the UI — it cannot draw anything. The actual drawing is done by a
> renderer like `react-dom` for web, or `react-native` for mobile. The real
> engine that does the comparing is a third package called `react-reconciler`,
> and both web and mobile share it.

**🎯 ONE LINE**
> I describe. React compares. The renderer draws.

---

## Q2. Why is `react` and `react-dom` two different packages?

**🗣 SAY THIS**

> Because React is not only for the browser. The same React can run on mobile,
> on a terminal, even on 3D canvas. So the "thinking" part is kept separate from
> the "drawing" part. `react` does the thinking. `react-dom` does the drawing
> on a web page.

**➕ IF THEY ASK MORE**

> The engine is given a small object with basic instructions — how to create an
> element, how to add a child, how to remove a child. That object is called a
> host config. Change that object, and the same React draws on a different
> screen. That is how React Native works with the same code.

**🎯 ONE LINE**
> One brain, many hands.

**💡 Real example to give**
> It is like a architect and a builder. The architect's plan is the same.
> One builder builds in brick, another in wood. The plan did not change.

---

## Q3. What happens to JSX? Does the browser understand it?

**🗣 SAY THIS**

> No, the browser does not understand JSX at all. JSX is removed before the code
> reaches the browser. A tool like Babel converts it into normal JavaScript
> function calls. So in the browser there is no JSX — only JavaScript.

**➕ IF THEY ASK MORE**

> `<div className="box">Hi</div>` becomes a function call like
> `jsx("div", { className: "box", children: "Hi" })`.
> And that function just returns a normal JavaScript object. It does not create
> any HTML. It only *describes* the HTML.

**🎯 ONE LINE**
> JSX is sugar. After build, it is just function calls.

---

## Q4. What does `<div />` actually return?

**🗣 SAY THIS**

> It returns a plain JavaScript object. Something like
> `{ type: "div", props: { ... }, key: null }`.
> That's all. No DOM element is created, nothing appears on screen.
> It is only a description of what we want.

**➕ IF THEY ASK MORE**

> People call it the virtual DOM, but really it is just a small object. React
> makes a fresh tree of these objects on every render, compares it with the old
> one, and only then touches the real DOM. Creating objects is cheap. Touching
> the DOM is expensive. That is the whole trick.

**🎯 ONE LINE**
> An element is a description, not a thing.

**💡 Real example to give**
> It is like writing a shopping list. Writing "milk, bread" on paper is cheap
> and fast. Actually going to the shop is slow. React writes the list first,
> compares it with the old list, and only buys what is different.

---

## Q5. There is a `$$typeof` field in every element. Why?

**🗣 SAY THIS**

> It is a security check. It is a Symbol, and Symbols cannot be written in JSON.
> So if someone sends bad data from the server, that data can never pretend to be
> a real React element. React checks this field and refuses to render fake ones.

**➕ IF THEY ASK MORE**

> Imagine a server sends user input and we put it directly inside JSX. An attacker
> could try to send an object that looks like an element with dangerous HTML
> inside it. But `JSON.parse` can never produce a Symbol. So the fake object fails
> the check and React does not render it. It protects against one type of XSS attack.

**🎯 ONE LINE**
> JSON cannot make a Symbol, so JSON cannot fake an element.

---

## Q6. Why is `<MyComp />` different from `MyComp()`? They look the same.

**🗣 SAY THIS**

> They are very different. `<MyComp />` just creates an object and gives it to
> React. React calls the function later, and gives that component its own
> identity — its own state, its own hooks. If I write `MyComp()` myself, I am
> calling it directly, so React never knows it exists. Its hooks and state get
> mixed into the parent component.

**➕ IF THEY ASK MORE**

> When React calls it, it creates a Fiber node for that component. A Fiber is like
> a box that stores that component's state. With a direct call, there is no box,
> so no separate state, and `React.memo` also stops working for it.

**🎯 ONE LINE**
> `<Comp />` gives it a house. `Comp()` makes it sleep in the parent's house.

---

## Q7. Why did React rewrite everything for Fiber? What was the problem?

**🗣 SAY THIS**

> In the old React, rendering was one big recursive function. Once it started,
> it could not stop until it finished. If the page was big, it could block the
> browser for a long time. The user types something and nothing happens, because
> JavaScript is still busy.

**➕ IF THEY ASK MORE**

> The real problem is that you cannot pause a JavaScript function in the middle.
> There is no pause button on the call stack. So React made its own stack using
> normal JavaScript objects. Each object is called a Fiber. Now React can do a
> little work, stop, let the browser breathe, and continue later — because it
> just has to remember which object it stopped at.

**🎯 ONE LINE**
> You cannot pause the JavaScript stack, so React built its own stack it *can* pause.

**💡 Real example to give**
> Old React was like a person who starts cleaning the whole house and does not
> answer the doorbell until finished. New React cleans one room, checks the door,
> cleans another room, checks the door again.

---

## Q8. So what is a Fiber in simple words?

**🗣 SAY THIS**

> A Fiber is a plain JavaScript object that represents one component in the tree.
> It stores what that component is, what props it got, what state it has, and
> pointers to its child, its next sibling, and its parent.

**➕ IF THEY ASK MORE**

> It is basically React's own version of a function call frame. A normal function
> call keeps its arguments and local variables in memory, and the computer
> controls it. A Fiber keeps the same information, but React controls it. That is
> why React can stop and continue whenever it wants.

**🎯 ONE LINE**
> A Fiber is one unit of work, and also the memory box for one component.

---

## Q9. Why does React use `child`, `sibling`, `return` instead of a children array?

**🗣 SAY THIS**

> Because with these three pointers, React can walk the whole tree using a simple
> loop instead of recursion. And a loop can be stopped in the middle. Recursion
> cannot.

**➕ IF THEY ASK MORE**

> The rule is simple: go down to the child if there is one. If not, go to the
> sibling. If there is no sibling, go back up to the parent and try its sibling.
> The pointer is named `return` and not `parent` on purpose — it means "where to
> go back to", exactly like a return address in a function call.

**🎯 ONE LINE**
> Three pointers turn recursion into a loop, and a loop can pause.

---

## Q10. What is the render phase and the commit phase?

**🗣 SAY THIS**

> React works in two steps. In the render phase it calls my components and
> figures out what changed — but it does not touch the screen at all. In the
> commit phase it actually updates the DOM. The render phase can be paused or
> even thrown away. The commit phase cannot — it happens in one go.

**➕ IF THEY ASK MORE**

> This is why side effects are not allowed in the component body. That code is in
> the render phase, and React might run it two times or throw the result away.
> So if I call an API there, it may fire twice or at a wrong time. That is why
> `useEffect` exists — it moves my side effect into the commit phase, which runs
> exactly once.

**🎯 ONE LINE**
> Render = plan, can be cancelled. Commit = do it, cannot be cancelled.

**💡 Real example to give**
> Render phase is writing the answer on rough paper. You can cut it, change it,
> throw it away. Commit phase is writing on the final answer sheet. Once written,
> it is written.

---

## Q11. Why must components be pure? Why does StrictMode call my component twice?

**🗣 SAY THIS**

> Because the component body runs in the render phase, and React may run it more
> than once or throw away the result. If my function changes something outside
> itself, that change may happen twice or at the wrong moment. StrictMode calls
> it twice in development on purpose, so that if my component is not pure, the
> bug shows up immediately instead of in production.

**🎯 ONE LINE**
> React may call your component twice, so make it safe to call twice.

---

## Q12. Is virtual DOM the reason React is fast?

**🗣 SAY THIS**

> Not exactly. The virtual DOM is not magically fast. Comparing two object trees
> still takes time. React is fast mainly because it *avoids* work — it skips
> parts of the tree where nothing changed, and it batches many updates into one
> DOM change. Also, hand-written DOM code can be faster than React. React's real
> benefit is that it stays fast and correct in a large app without me managing
> every update by hand.

**➕ IF THEY ASK MORE**

> React even has short-circuits. If I call `setState` with the same value, React
> compares it and does nothing at all. If a component's props did not change, it
> reuses the whole old subtree without calling the function again. Most of the
> speed comes from work React decides *not* to do.

**🎯 ONE LINE**
> React is not fast because it does clever work. It is fast because it skips work.

---

## Q13. Fiber came in React 16, but concurrency came in React 18. Why the gap?

**🗣 SAY THIS**

> Because React 16 only changed the internal architecture, not the behaviour. The
> new engine could pause, but React kept that switched off and kept rendering
> synchronously. They let it run in production for years to make sure it was
> stable. Only in React 18 they turned the pausing on, with `createRoot` and
> transitions.

**🎯 ONE LINE**
> They shipped the engine in 2017 and switched it on in 2022.

---

## Quick revision table

| Question | One-line answer |
|---|---|
| What is React? | I describe, React compares, the renderer draws. |
| Why 4 packages? | One brain, many hands. |
| What is JSX? | Sugar. It becomes plain function calls at build time. |
| What does `<div/>` return? | A plain object. A description, not a thing. |
| Why `$$typeof` Symbol? | JSON cannot make Symbols, so JSON cannot fake an element. |
| `<Comp/>` vs `Comp()`? | One gets its own house (fiber), one does not. |
| Why Fiber? | The JS stack cannot pause, so React built one that can. |
| What is a Fiber? | One unit of work + the memory box of one component. |
| Why child/sibling/return? | Turns recursion into a pausable loop. |
| Render vs commit? | Plan (cancellable) vs do it (final). |
| Why pure components? | React may run your function twice. |
| Why is React fast? | Because it skips work, not because it does clever work. |

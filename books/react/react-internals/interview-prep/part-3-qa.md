# Part 3 — Interview Q&A (Simple English)

Covers Topic 09 so far: hooks internals.
*(Topic 10 — updates and batching — will be added next.)*

Deep version: [part-3 notes](../part-3-state/README.md)

> ⭐ **This is the most asked internals topic in React interviews.**
> If you only prepare one file, prepare this one.

---

# Topic 09 — Hooks Internals

---

## Q1. A function component has no instance. So where is `useState` stored?

**🗣 SAY THIS**

> It is stored on the Fiber, not inside the function. Every component has a Fiber
> object, and the Fiber has a field called `memoizedState`. For a function
> component that field holds a linked list, and each item in the list is one hook.

**➕ IF THEY ASK MORE**

> A class component keeps state on `this`, because it has a real instance. A
> function component has nothing — it runs and finishes, and all local variables
> disappear. So React keeps the values outside the function, on the Fiber, and
> gives them back on the next render.

**🎯 ONE LINE**
> The state does not live in your function. It lives on the Fiber.

---

## Q2. How does React know which `useState` is which?

**🗣 SAY THIS**

> Only by the order you call them. The hooks are stored in a simple list, and
> there are no names in that list. The first `useState` gets the first item, the
> second gets the second item, and so on.

**💡 Real example to give**

```jsx
const [name] = useState('Ada')   // list item 1
const [age]  = useState(36)      // list item 2
const ref    = useRef(null)      // list item 3
```

> React does not store "name" or "age" anywhere. It only counts positions.

**🎯 ONE LINE**
> Position is the only identity a hook has.

---

## Q3. ⭐ Why can't we call hooks inside an `if`?

**🗣 SAY THIS**

> Because React matches hooks by position. If a hook is skipped in one render,
> every hook after it shifts up by one position, so they all read the wrong
> values.

**💡 Real example to give**

```jsx
if (loggedIn) {
  const [name]  = useState('Ada')
}
const [theme] = useState('dark')
```

> First render with `loggedIn` true, the list is: Ada, dark.
> Next render with `loggedIn` false, only one hook runs, and it takes position 1.
> So `theme` now reads "Ada". The values are shifted and completely wrong.

**➕ IF THEY ASK MORE**

> React also detects this and throws. If you render fewer hooks than last time
> you get "Rendered fewer hooks than expected". If you render more, you get
> "Rendered more hooks than during the previous render".

**🎯 ONE LINE**
> The hook list has no names, so skipping one shifts all the others.

> **Better way to state the rule:** it is not "no hooks in if". It is "the number
> and order of hook calls must be the same on every render". That is why a loop
> with a fixed count is actually fine, and why an early return between two hooks
> is not.

---

## Q4. Why does calling a hook outside a component throw an error?

**🗣 SAY THIS**

> React keeps a shared slot that holds the real hook functions. Before calling
> your component it puts the correct version in that slot, and right after your
> component finishes it replaces that slot with a version where every hook just
> throws an error. So outside a render, you always hit the throwing version.

**➕ IF THEY ASK MORE**

> The same error also happens if two copies of React are installed. Then there
> are two separate slots. React fills one slot, your component reads the other
> one, and that one is still the throwing version.

**🎯 ONE LINE**
> Outside a render, the hook slot holds a version that only throws.

---

## Q5. What is the difference between `useState` and `useReducer` internally?

**🗣 SAY THIS**

> There is almost none. `useState` is `useReducer` with a tiny built-in reducer:

```js
function basicStateReducer(state, action) {
  return typeof action === 'function' ? action(state) : action
}
```

> That single line is the whole difference between `setCount(5)` and
> `setCount(c => c + 1)`. If you pass a function, React calls it with the current
> state. If you pass a value, it just uses that value.

**🎯 ONE LINE**
> `useState` is `useReducer` with a one-line reducer.

---

## Q6. Why is the `setState` function stable? Why don't I need it in dependency arrays?

**🗣 SAY THIS**

> Because React creates it only once, on the first render, and already binds it
> to that component's Fiber and its queue. After that it returns the exact same
> function every time. So its reference never changes.

**➕ IF THEY ASK MORE**

> That is also why you can call it from a setTimeout, a promise, or an event
> handler. It already knows where to put the update. It does not need to be
> inside a render.

**🎯 ONE LINE**
> It is created once at mount and never recreated.

---

## Q7. What happens when I call `setState` with the same value?

**🗣 SAY THIS**

> React compares the new value with the current one. If they are the same, it
> does not schedule anything at all. No render, no work.

**➕ IF THEY ASK MORE**

> But this only works when there are no other updates already waiting. If
> something is already queued, React cannot know what the value will be at that
> point, so it must schedule and find out. Also, React may still render the
> component one extra time before it settles — the shortcut prevents the
> scheduling, not every possible render.

**🎯 ONE LINE**
> Same value with an empty queue means React does literally nothing.

---

## Q8. ⭐ Why do I get an old value inside a `setTimeout` or an interval?

**🗣 SAY THIS**

> Because every render is a separate function call, and each one has its own
> variables. Your callback captured the value from the render where it was
> created. Later renders create new variables, but your old callback is still
> holding the old one.

**💡 Real example to give**

```jsx
useEffect(() => {
  setInterval(() => {
    setCount(count + 1)   // always uses the first render's count
  }, 1000)
}, [])
```

> The effect ran once, in the first render, where `count` was 0. So it is always
> 0 + 1. It is not a React bug — it is a normal JavaScript closure.

**➕ IF THEY ASK MORE**

> There are three ways to fix it. Use the function form, `setCount(c => c + 1)`,
> so you never read the old variable. Or add `count` to the dependency array so
> the effect is recreated with a fresh value. Or keep the value in a `useRef`,
> because a ref is one single object that never changes.

**🎯 ONE LINE**
> Each render has its own variables, and your callback kept an old one.

---

## Q9. What is `useRef` internally? Why doesn't changing it re-render?

**🗣 SAY THIS**

> `useRef` is just one object `{ current: value }` created on the first render
> and returned unchanged forever. When you change `.current`, you are only
> changing a normal JavaScript object. React is not involved at all, so nothing
> re-renders.

**🎯 ONE LINE**
> It is one plain object that React hands back every render.

---

## Q10. How does `useMemo` decide to recompute?

**🗣 SAY THIS**

> It stores the value and the dependency array together. On the next render it
> compares each dependency one by one using `Object.is`. If all of them match, it
> returns the stored value. If any is different, it runs the function again.

**➕ IF THEY ASK MORE**

> This is why passing an object or an array directly as a dependency never works.
> `[{ a: 1 }]` is a new object every render, so the comparison always fails.
> Also, React says it may throw away memoized values to free memory, so never put
> something your app needs for correctness inside `useMemo`.

**🎯 ONE LINE**
> It compares each dependency with `Object.is` and reuses the value if all match.

---

## Q11. How does React know whether to run an effect?

**🗣 SAY THIS**

> Effects are also stored in a list on the Fiber. On every render React adds the
> effect to the list, but it only marks it as "needs to run" if the dependencies
> changed. The commit phase then runs only the marked ones.

**➕ IF THEY ASK MORE**

> The important part is that the effect is always added to the list, even when it
> will not run. That is because effects are matched by position too, just like
> hooks. If React skipped adding it, the positions would shift.

**🎯 ONE LINE**
> Always added to the list, but only marked when the dependencies changed.

---

## Q12. ⭐ How does React keep state correct if it can pause and skip updates?

**🗣 SAY THIS**

> Each hook keeps two things: the value it is showing now, and a safe starting
> point called the base state, plus a list of updates that were skipped. If a low
> priority update is skipped, React remembers the state from before it and keeps
> that update in the list. Later it replays all of them from that starting point,
> in the original order.

**💡 Real example to give**

> State is 0. Two updates come: "+1" at low priority and "+10" at high priority.
> React is doing an urgent render, so it skips +1 and applies +10. The user sees
> 10 immediately. But React remembered the starting point 0 and kept both updates.
> Later it replays them in order: 0, then +1, then +10, giving 11.

**➕ IF THEY ASK MORE**

> One important detail: once an update is skipped, every update after it must
> also be kept, even if it was applied. Otherwise the replay would lose it and
> the final result would be wrong.

**🎯 ONE LINE**
> React keeps a safe starting point so skipped updates can be replayed in order.

> **This is an advanced answer.** Most candidates cannot explain this. If you can,
> it signals that you understand concurrent rendering and not just the hook API.

---

## Q13. How would you actually see the hook list in a real app?

**🗣 SAY THIS**

> From the console. Get the Fiber from a DOM node and walk the list:

```js
let hook = fiber.memoizedState, i = 0
while (hook) { console.log(i++, hook.memoizedState); hook = hook.next }
```

> You will see the state values, the refs as `{current: ...}`, the memo values
> with their dependencies, and the effects — all in order, with no names.

**🎯 ONE LINE**
> Seeing the list once makes the Rules of Hooks obvious forever.

---

## Quick revision table

| Question | One-line answer |
|---|---|
| Where is `useState` stored? | On the Fiber, in `memoizedState`, as a linked list. |
| How are hooks identified? | Only by position. There are no names. |
| Why no hooks in `if`? | Skipping one shifts every hook after it. |
| The real rule? | Same number and same order of hook calls, every render. |
| Why "Invalid hook call"? | Outside a render, the hook slot only throws. |
| `useState` vs `useReducer`? | `useState` is `useReducer` with a one-line reducer. |
| Why is the setter stable? | Created once at mount, bound to the Fiber. |
| `setState` with same value? | Nothing is scheduled at all. |
| Why stale values in timers? | Each render has its own variables; the callback kept an old one. |
| What is `useRef`? | One plain object returned every render. |
| How does `useMemo` compare? | `Object.is` on each dependency. |
| Are effects always stored? | Yes, but only marked to run when deps change. |
| How is state safe when paused? | A base state plus skipped updates, replayed in order. |

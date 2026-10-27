# Part 2 — Interview Q&A (Simple English)

Covers all of Part 2 — Topics 04–08: the Fiber node, the two trees, the render
phase and diffing, the complete phase, and the commit phase. **38 questions.**

Deep version: [part-2 notes](../part-2-fiber-engine/README.md)

---

## Q1. What is stored inside a Fiber?

**🗣 SAY THIS**

> A Fiber is just a normal JavaScript object for one component. It stores what
> type of thing it is, the props it got, the state it has, pointers to its child,
> sibling and parent, and some flags about what needs to change on the screen.

**➕ IF THEY ASK MORE**

> It also stores the priority of pending work, and a pointer called `alternate`
> that points to the previous version of the same component. So one Fiber answers
> four questions: what am I, where am I in the tree, what do I remember, and what
> needs to be done to me.

**🎯 ONE LINE**
> One Fiber = one component's identity, position, memory, and to-do list.

---

## Q2. What is the `tag` field?

**🗣 SAY THIS**

> It is just a number that tells React what kind of node this is. For example 0
> means function component, 1 means class component, 5 means a normal HTML
> element like a div. React uses a switch on this number to decide how to handle
> the node.

**➕ IF THEY ASK MORE**

> They used a number instead of a string because this switch runs millions of
> times. Comparing numbers is faster than comparing strings. You see this style
> everywhere in React's code — numbers and bitmasks instead of strings and arrays.

**🎯 ONE LINE**
> `tag` is the "what kind of node am I" number.

---

## Q3. Where is the real DOM element stored?

**🗣 SAY THIS**

> In a field called `stateNode`. For a div or span, `stateNode` is the actual DOM
> element. For a class component it is the class instance. For a function
> component it is `null`, because a function component has no instance.

**➕ IF THEY ASK MORE**

> That `null` is the real reason you could not put a `ref` directly on a function
> component in older React. There was simply no object to give you. That is why
> `forwardRef` existed. In React 19 `ref` is a normal prop, so it works directly.

**🎯 ONE LINE**
> `stateNode` is the bridge from React's world to the real DOM.

---

## Q4. Where is `useState` value actually stored?

**🗣 SAY THIS**

> On the Fiber, in a field called `memoizedState`. For a function component this
> field is the first hook in a linked list. Each hook points to the next one.

**➕ IF THEY ASK MORE**

> Important point: the hooks have no names in that list. There is no "this is the
> count hook". React finds the third hook only by starting at the first and
> following `next` three times. So the order must be exactly the same on every
> render. That is the whole reason behind the Rules of Hooks — it is not a style
> rule, it is because of this data structure.

**🎯 ONE LINE**
> Hooks are a linked list with no names, so only the order identifies them.

---

## Q5. What are `flags` and `subtreeFlags`?

**🗣 SAY THIS**

> `flags` says what React must do to this one node — insert it, update it, attach
> a ref, run an effect. `subtreeFlags` is a summary of all the flags of the
> children below it.

**➕ IF THEY ASK MORE**

> The summary is very useful. During commit, React looks at a node and if
> `subtreeFlags` is empty, it knows nothing below needs changing, so it skips the
> whole branch without even entering it. React uses the same idea in the render
> phase with a field called `childLanes`. A summary on the parent lets you skip a
> whole branch.

**🎯 ONE LINE**
> A summary on the parent so React can skip an entire branch.

**💡 Real example to give**
> Like a folder that says "nothing changed inside me". You don't need to open it.

---

## Q6. Why does React keep two trees?

**🗣 SAY THIS**

> Because React can stop rendering in the middle and even throw the work away.
> If it changed the existing tree directly and then stopped halfway, the tree
> would be half old and half new — broken. So React keeps the old tree safe and
> builds the new one separately. When the new one is fully ready, it switches.

**➕ IF THEY ASK MORE**

> It is the same idea as double buffering in games. You never draw on the screen
> directly. You draw on a hidden page and then flip. The user sees the old frame
> or the new frame, never a half-drawn one.

**🎯 ONE LINE**
> Build the new one safely on the side, then switch in one step.

**💡 Real example to give**
> Like writing a new version of a document in a separate file. The old file keeps
> working until the new one is complete. Then you rename it.

---

## Q7. What is the `alternate` pointer?

**🗣 SAY THIS**

> Every Fiber has a twin — the same component in the other tree. `alternate`
> points to that twin. So one is the version on screen and the other is the
> version being built.

**➕ IF THEY ASK MORE**

> React never creates a third copy. It only keeps these two and reuses them again
> and again. So after the first update, React allocates almost no new Fiber
> objects. That saves a lot of memory and garbage collection work.

**🎯 ONE LINE**
> Each component has exactly two Fibers that swap roles forever.

---

## Q8. How does React know if it is mounting or updating?

**🗣 SAY THIS**

> It checks if the Fiber has an `alternate`. If there is no previous version, it
> is a mount. If there is one, it is an update.

**➕ IF THEY ASK MORE**

> That one check decides a lot of things: whether `useState` creates a new hook or
> reads the old one, whether `useEffect` runs directly or compares the dependency
> array, whether React creates a DOM node or just updates props, and whether
> `componentDidMount` or `componentDidUpdate` is called. And `prevProps` in
> `componentDidUpdate` comes from the old Fiber's `memoizedProps`.

**🎯 ONE LINE**
> No previous version means mount. A previous version means update.

---

## Q9. How does React actually "switch" the two trees? Is it expensive?

**🗣 SAY THIS**

> No, it is one line. React just changes one pointer: `root.current = newTree`.
> That is the whole switch. It is instant.

**➕ IF THEY ASK MORE**

> And the timing of that line matters. It happens after React changes the DOM but
> before it runs layout effects. The reason is that unmount cleanup code should
> still see the old tree, and `useLayoutEffect` and refs should see the new tree.
> So the switch is placed exactly between them.

**🎯 ONE LINE**
> The swap is one pointer assignment.

---

## Q10. If React copies the tree on every render, isn't that slow?

**🗣 SAY THIS**

> React does not copy the whole tree. It copies only the nodes it actually walks
> into. If a part of the tree did not change, React does not even copy it — both
> trees point to the exact same objects for that part.

**➕ IF THEY ASK MORE**

> This is called a bailout. If the props did not change and there is no pending
> state update in that branch, React returns immediately. That branch is not
> re-rendered, not compared, and not even copied. Most of React's speed comes
> from work it decides not to do.

**🎯 ONE LINE**
> Unchanged branches are shared, not copied.

---

## Q11. Both trees point to the same DOM node. Is that a problem?

**🗣 SAY THIS**

> No, it is intentional. There is only one real DOM. The two trees are just two
> notebooks about the same DOM. Both Fibers keep the same `stateNode`, so React
> does not create duplicate DOM nodes.

**🎯 ONE LINE**
> Two notebooks, one real house.

---

## Q12. How would you actually look at a Fiber in a real app?

**🗣 SAY THIS**

> In the browser console. React puts a hidden key on every DOM node that starts
> with `__reactFiber$`. If you read that property, you get the real Fiber object
> and you can explore it.

```js
const el = document.querySelector('#root > *');
const key = Object.keys(el).find(k => k.startsWith('__reactFiber$'));
const fiber = el[key];

fiber.type          // the component or tag name
fiber.memoizedProps // its props
fiber.alternate     // the twin
fiber.return        // the parent
```

**➕ IF THEY ASK MORE**

> This is also how React DevTools works internally, and how React finds which
> component an event belongs to when you click something.

**🎯 ONE LINE**
> Every DOM node secretly carries a pointer back to its Fiber.

> **Saying this in an interview is strong** — it shows you have actually opened
> the console and looked, not just read an article.

---

---
---

# Topic 06 — The Render Phase & Diffing

---

## Q13. What is the render phase actually doing?

**🗣 SAY THIS**

> In the render phase React calls my components and compares the new output with
> the old tree, to find out what changed. It does not touch the screen at all in
> this phase. It only prepares a list of changes.

**➕ IF THEY ASK MORE**

> It walks the tree node by node. Going down it calls the component and creates
> the children. Coming up it creates the DOM nodes in memory and works out which
> props changed. At the end it has a complete new tree plus marks saying what
> needs to be inserted, updated or deleted.

**🎯 ONE LINE**
> Render phase finds out what changed. It changes nothing.

---

## Q14. Why is React's diffing O(n) when comparing two trees is normally much slower?

**🗣 SAY THIS**

> Comparing two trees properly is very slow — roughly n cubed. React does not do
> that. It makes two assumptions to make it linear. First, if the element type at
> a position changed, it does not try to match anything, it just deletes the old
> one and makes a new one. Second, for lists it asks the developer to give a key,
> instead of guessing which items moved.

**➕ IF THEY ASK MORE**

> There is also a third rule: React never compares across different parents. If
> you move a component to a different parent in the tree, React destroys it and
> creates it again, even if it looks exactly the same.

**🎯 ONE LINE**
> React is fast because it refuses to be perfect. Two assumptions turn n³ into n.

---

## Q15. What happens when the element type changes at a position?

**🗣 SAY THIS**

> React deletes the whole old subtree and builds a new one. It does not try to
> reuse anything inside. So all state in that subtree is lost and the DOM nodes
> are recreated.

**💡 Real example to give**

> This is why you should never define a component inside another component:

```jsx
function Parent() {
  function Child() { return <input /> }   // ❌ new function every render
  return <Child />
}
```

> Every render makes a new `Child` function, so the type is different every time,
> so React destroys and recreates it. The input loses its text and loses focus on
> every keystroke.

**➕ IF THEY ASK MORE**

> The same behaviour is also used as a feature. If I write
> `<Profile key={userId} />` and the userId changes, the key changes, so React
> remounts it and all the state inside resets. That is the official way to reset
> a component.

**🎯 ONE LINE**
> Different type means delete and rebuild, never reuse.

---

## Q16. How does React match an old node with a new one?

**🗣 SAY THIS**

> It checks two things: first the key, then the type. Both must match. If both
> match, React reuses the old Fiber and just updates the props, so the state is
> kept. If either one is different, the old Fiber is thrown away.

**🎯 ONE LINE**
> Key first, then type. Both must agree, or state is lost.

---

## Q17. Explain the list diffing algorithm.

**🗣 SAY THIS**

> React does it in two passes. In the first pass it walks the old list and the new
> list side by side, position by position, as long as the keys match. For most
> real cases — adding to the end, changing some text, nothing moved — this first
> pass handles everything and it is very fast.

> If the keys stop matching, it goes to the second pass. There it puts all the
> remaining old children into a Map by their key, and then goes through the new
> list and picks each one from the Map. A Map lookup is O(1), so the whole thing
> stays linear.

**➕ IF THEY ASK MORE**

> At the end, anything still left in the Map was not used by the new list, so
> those get deleted.

**🎯 ONE LINE**
> Fast walk first. Build a key Map only if something actually moved.

---

## Q18. Why exactly is `key={index}` bad?

**🗣 SAY THIS**

> Because the key should identify the *item*, but the index identifies the
> *position*. If you add an item at the start, every item moves to a new index,
> so every key now points to a different item. React thinks nothing moved and
> everything just changed its props.

**💡 Real example to give**

> List is A, B, C. I add "New" at the front.
> React sees key 0 before = A, key 0 now = New. Same key, same type, so it reuses
> that Fiber. So the Fiber that belonged to A is now showing New. The state goes
> with the position, not with the item. If row A had a checked checkbox, now the
> wrong row is checked.

**➕ IF THEY ASK MORE**

> With a real id as key, React's first pass stops immediately, it builds the Map,
> every item is matched to its own Fiber, and only one node is marked as moved.
> Correct and also less work.

**🎯 ONE LINE**
> Index keys match positions. Real keys match items.

**When is index key okay?**
> Only if the list is never reordered, never filtered, and items are only added
> at the end.

---

## Q19. What is a bailout?

**🗣 SAY THIS**

> A bailout is when React decides it does not need to re-render something and
> skips it. Most of React's speed comes from this, not from clever work.

**➕ IF THEY ASK MORE**

> There are a few levels. The cheapest one is when you call setState with the
> same value — React compares it and does not even schedule anything. Then there
> is a check at the start of each node: if the props object is literally the same
> object and there is no pending update in that branch, React returns immediately
> and skips the entire subtree. `memo` adds another level where React compares
> the props values one by one.

**🎯 ONE LINE**
> A bailout is React deciding not to work.

---

## Q20. How does React skip a whole subtree without going into it?

**🗣 SAY THIS**

> Every Fiber stores a summary of pending work in its children, in a field called
> `childLanes`. When React reaches a node it checks that summary. If it is empty,
> React knows for sure that nothing below needs updating, so it returns
> immediately and never enters that branch.

**➕ IF THEY ASK MORE**

> The summary is kept correct because when you call setState, React walks up from
> that component to the root and marks the priority on every parent on the way.
> So the information is always ready before the next render starts.

**🎯 ONE LINE**
> Parents keep a summary, so React can skip a branch without opening it.

---

## Q21. Why does `memo` sometimes not work?

**🗣 SAY THIS**

> Because `memo` compares props one level deep using `Object.is`. If I pass an
> object or a function created inline in JSX, it is a new value every render, so
> the comparison always fails and `memo` never helps.

```jsx
<Child config={{ a: 1 }} />        // ❌ new object every render
<Child onClick={() => doIt()} />   // ❌ new function every render
```

**➕ IF THEY ASK MORE**

> That is why `useMemo` and `useCallback` exist — to keep the same reference
> between renders so the comparison can succeed. Also, `memo` can never skip a
> component that has its own pending state update. Your own setState always wins.

**🎯 ONE LINE**
> `memo` compares references, and inline objects are new references every time.

---

## Q22. Where does "Invalid hook call" come from?

**🗣 SAY THIS**

> React keeps a shared slot called the dispatcher, which holds the real hook
> implementations. Just before calling my component, React puts the correct one
> in that slot. Just after, it replaces it with a version where every hook throws
> an error. So if I call a hook outside a component, I hit that error version.

**➕ IF THEY ASK MORE**

> The same error also appears if there are two copies of React in node_modules,
> because then one copy fills its slot and my component reads the other copy's
> empty slot.

**🎯 ONE LINE**
> Outside a render, the hook slot holds a version that only throws.

---

## Q23. Where does "Too many re-renders" come from?

**🗣 SAY THIS**

> React allows calling setState during render, because sometimes you need to
> adjust state from props. When that happens React just calls the component again
> immediately in a loop. But it counts, and after 25 times it throws that error,
> to stop an infinite loop.

**💡 Real example to give**

> It usually happens when someone writes `onClick={handleClick()}` instead of
> `onClick={handleClick}`. The function runs during render, calls setState,
> render again, forever.

**🎯 ONE LINE**
> React re-runs your component up to 25 times, then gives up.

---

## Q24. Is React's list reordering optimal?

**🗣 SAY THIS**

> No, and that is intentional. React uses a simple one-pass greedy rule. A node
> stays where it is only if it was already after everything placed before it.
> Otherwise React marks it as moved.

**💡 Real example to give**

> Moving an item to the end of the list is one DOM move.
> Moving an item to the front makes React move all the other items instead. So
> prepending to a long list does more DOM work than needed.

**➕ IF THEY ASK MORE**

> The result is always correct and no state is lost — it just does a few extra
> DOM moves. React accepted that trade to keep the algorithm linear and simple.

**🎯 ONE LINE**
> Moving to the end is cheap. Moving to the front moves everything else.

---

---
---

# Topic 07 — The Complete Phase

---

## Q25. When are the real DOM elements created?

**🗣 SAY THIS**

> Surprisingly, in the render phase — not in the commit phase. While React walks
> back up the tree it calls `document.createElement` for each host element. But
> those elements are not added to the page yet. They are kept separate, in memory.

**➕ IF THEY ASK MORE**

> Creating a DOM element is cheap, because the browser does no layout and no
> painting for an element that is not on the page. Only attaching it to the page
> is expensive. So React does all the creation in the phase that can be thrown
> away, and pays the expensive cost only once at the end.

**🎯 ONE LINE**
> Nodes are created early, but kept off the page until commit.

---

## Q26. Why does React build the tree from bottom to top?

**🗣 SAY THIS**

> Because a parent cannot add children that do not exist yet. So React finishes
> the children first, then creates the parent and puts the finished children
> inside it. By the time it reaches the top, the whole tree is already assembled.

**➕ IF THEY ASK MORE**

> That is why the first render of a page is only one DOM insertion. React builds
> the complete tree in memory and then adds the top node to the page once. One
> insertion means one layout and one paint, instead of hundreds.

**🎯 ONE LINE**
> Children must exist before the parent can hold them.

**💡 Real example to give**
> Like packing a box. You fill the small boxes first, put them in the big box,
> and carry the big box into the room once.

---

## Q27. Each node is visited twice. Why?

**🗣 SAY THIS**

> Once going down and once coming up. Going down, React calls the component and
> creates the children. Coming up, React creates the DOM node, works out which
> props changed, and passes a summary to the parent.

**🎯 ONE LINE**
> Down to find the work, up to finish the work.

---

## Q28. Where is it decided which props changed?

**🗣 SAY THIS**

> In the complete phase, while walking up. React compares the old props with the
> new props and makes a small list of only the things that changed. The commit
> phase just applies that list.

**➕ IF THEY ASK MORE**

> The reason it is done here and not in commit is that comparing is pure
> calculation, so it is safe to pause it or do it twice. The commit phase blocks
> the screen, so it must be as short as possible. The rule is: compare in render,
> apply in commit.

**🎯 ONE LINE**
> Compare in render, apply in commit.

---

## Q29. Does `<div>hello</div>` create a text node in React?

**🗣 SAY THIS**

> No. If the only child is a simple string or number, React does not create a
> separate child for it. It sets the text directly as a property on the div. So
> there is one node instead of two.

**➕ IF THEY ASK MORE**

> But `<div>hello {name}</div>` is different, because now the children are an
> array. In that case React does create separate text nodes.

**🎯 ONE LINE**
> A single text child becomes a property, not a child node.

---

## Q30. How do error boundaries and Suspense find their handler?

**🗣 SAY THIS**

> When a component throws, React marks that node as incomplete and then walks up
> the parent pointers one by one, looking for a node that can handle it — an
> error boundary for an error, or a Suspense boundary for a promise. If it does
> not find one, it reaches the root and the whole render fails.

**➕ IF THEY ASK MORE**

> The nice part is that React does not need try-catch for this. Because the tree
> is a normal object structure with parent pointers, unwinding is just a loop
> going upward. That also means it can be paused and resumed, which a real
> JavaScript stack unwind cannot.

**🎯 ONE LINE**
> React walks up the parent pointers until it finds someone who can handle it.

---

---
---

# Topic 08 — The Commit Phase

---

## Q31. What happens in the commit phase?

**🗣 SAY THIS**

> This is where React actually changes the page. It inserts, removes and updates
> the DOM nodes, attaches refs, and runs the effects. It happens in one go and
> cannot be paused.

**➕ IF THEY ASK MORE**

> It has three steps. First, read the old DOM if anyone needs it. Second, change
> the DOM. Third, run layout effects and attach refs. After that the browser
> paints, and only then the normal `useEffect` functions run.

**🎯 ONE LINE**
> Render decides. Commit does.

---

## Q32. What is the difference between `useEffect` and `useLayoutEffect`?

**🗣 SAY THIS**

> `useLayoutEffect` runs immediately after React changes the DOM but **before**
> the browser paints. `useEffect` runs **after** the browser has painted. So
> layout effect blocks the screen, and the normal effect does not.

**💡 Real example to give**

> Say I want to measure an element and move it. With `useLayoutEffect` the user
> only ever sees the final position. With `useEffect` the browser first paints
> the wrong position, then it jumps. That jump is a visible flicker.

**➕ IF THEY ASK MORE**

> So the rule is simple: if the user would see a wrong frame, use
> `useLayoutEffect`. For everything else — API calls, subscriptions, logging —
> use `useEffect`, because it does not block the screen.

**🎯 ONE LINE**
> Layout effect runs before paint, normal effect runs after paint.

---

## Q33. In what order do effects run between parent and child?

**🗣 SAY THIS**

> Children first, then parents. React finishes the whole bottom of the tree
> before coming up.

**➕ IF THEY ASK MORE**

> That is why a ref inside a child is already set when the parent's
> `useLayoutEffect` runs. If it was the other way round, the parent would try to
> measure a child that is not ready yet.

**🎯 ONE LINE**
> Always bottom to top. Children are ready before the parent needs them.

---

## Q34. On an update, does the cleanup of a component run right before its own effect?

**🗣 SAY THIS**

> No, and this surprises most people. React runs **all** the cleanups in the
> whole tree first, and only then runs all the new effects.

```
  1. child cleanup
  2. parent cleanup      ← all cleanups first
  3. child effect
  4. parent effect       ← then all effects
```

**➕ IF THEY ASK MORE**

> The reason is safety. A parent's cleanup might remove something that a child's
> new effect wants to use. If they were mixed together, a component could set
> itself up against a half-destroyed tree. Doing all cleanups first makes sure
> every effect starts in a clean, consistent state.

**🎯 ONE LINE**
> All cleanups in the tree first, then all setups.

---

## Q35. When is a ref actually set?

**🗣 SAY THIS**

> React sets the old ref to `null` while it is changing the DOM, and sets the new
> value later, in the layout step. So there is a small gap where the ref is null
> on purpose.

**➕ IF THEY ASK MORE**

> That gap guarantees you never see a ref pointing to a node that was just
> removed. It is also why a callback ref is called twice when it changes — once
> with null and once with the node. And if you write `ref={el => ...}` inline,
> it is a new function on every render, so React detaches and attaches it every
> time.

**🎯 ONE LINE**
> Cleared during the DOM change, set after it.

---

## Q36. Is it guaranteed that `useEffect` runs after paint?

**🗣 SAY THIS**

> Usually yes, but not always. React schedules it as a separate task, so normally
> the browser paints first. But if something urgent happens — a click, a new
> update, or `flushSync` — React will run the pending effects earlier, to stay
> correct.

**🎯 ONE LINE**
> Normally after paint, but React will run it earlier if correctness needs it.

---

## Q37. Why does StrictMode run my effect twice?

**🗣 SAY THIS**

> In development React mounts the component, runs the effect, runs the cleanup,
> and runs the effect again. It is testing whether my effect can be safely
> stopped and started again.

**➕ IF THEY ASK MORE**

> If something breaks when it runs twice, that means my cleanup is missing or
> incomplete — for example a subscription I never close or a timer I never
> clear. So it is not a bug in React, it is React showing me my bug early. In
> production it runs only once.

**🎯 ONE LINE**
> React tests whether your effect can survive being restarted.

---

## Q38. Can you say the full order of what happens on an update?

**🗣 SAY THIS**

> Yes:

```
 1. component functions run              (render phase)
 2. DOM nodes created in memory, props compared
 3. cleanup of removed components, refs cleared
 4. DOM is changed
 5. React switches to the new tree
 6. useLayoutEffect runs, refs attached   (children first)
 7. browser paints
 8. all useEffect cleanups, then all useEffect bodies
```

**🎯 ONE LINE**
> Render, mutate, layout effects, paint, then normal effects.

> **This is a strong answer to memorise.** Many candidates know the hooks but
> cannot put them on a timeline. Being able to say this clearly separates you.

---

## Quick revision table

| Question | One-line answer |
|---|---|
| What is in a Fiber? | Identity, position, memory, and to-do list of one component. |
| What is `tag`? | A number for "what kind of node am I". |
| Where is the DOM node? | In `stateNode`. `null` for function components. |
| Where is `useState` stored? | In `memoizedState`, a linked list with no names. |
| Why Rules of Hooks? | Because only the order identifies a hook. |
| What is `subtreeFlags`? | A summary so React can skip a whole branch. |
| Why two trees? | So half-finished work never reaches the screen. |
| What is `alternate`? | The twin Fiber in the other tree. |
| Mount or update? | No alternate = mount. Has alternate = update. |
| How does the swap work? | One line: `root.current = newTree`. |
| Is copying slow? | Unchanged branches are shared, not copied. |
| How to see a Fiber? | `__reactFiber$...` key on any DOM node. |
| What does render phase do? | Finds what changed. Changes nothing. |
| Why is diffing O(n)? | Two assumptions: type change = rebuild, key = identity. |
| Type changed? | Delete and rebuild. All state lost. |
| How are nodes matched? | Key first, then type. Both must agree. |
| List diff algorithm? | Fast side-by-side walk, then a key Map if something moved. |
| Why is `key={index}` bad? | It matches positions instead of items, so state follows the slot. |
| What is a bailout? | React deciding not to work. |
| How is a subtree skipped? | `childLanes` summary on the parent. |
| Why does `memo` fail? | Inline objects and functions are new references every render. |
| "Invalid hook call"? | Outside a render, the dispatcher slot only throws. |
| "Too many re-renders"? | 25 render-phase updates in a row, then React gives up. |
| Is reordering optimal? | No. Moving to the front moves everything else. |
| When are DOM nodes created? | In the render phase, but kept off the page. |
| Why bottom-up? | A parent cannot hold children that don't exist yet. |
| Why is first render one insertion? | The whole tree is assembled in memory first. |
| Where is the prop diff done? | Complete phase. Compare in render, apply in commit. |
| Does `<div>hi</div>` make a text node? | No — a single string child becomes a property. |
| How is an error caught? | React walks up the parent pointers to find a boundary. |
| What is the commit phase? | Render decides, commit does. One go, cannot pause. |
| `useEffect` vs `useLayoutEffect`? | Before paint vs after paint. |
| Parent or child effect first? | Child first, always bottom to top. |
| Cleanup order on update? | All cleanups in the tree first, then all setups. |
| When is a ref set? | Cleared during the DOM change, set after it. |
| Is `useEffect` always after paint? | Usually, but React runs it earlier if needed. |
| Why does StrictMode run effects twice? | To check your cleanup actually works. |

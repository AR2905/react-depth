# ⚛️ Step-by-step: How a React Component Renders
---

## Summary (one-liner)
When React data (state/props/context) changes, React **re-runs the component render**, builds a new **Virtual DOM**, **diffs** it against the previous Virtual DOM (reconciliation), and **patches the real DOM** with only the minimal changes — all orchestrated by the React **Fiber** engine for scheduling and responsiveness.

---

## Step 1 — The trigger (what causes a render)
A render starts when any of the following happen:
1. **Initial mount** — the component is mounted for the first time.  
2. **State update** — `setState()` or a `useState`/`useReducer` update.  
3. **Props change** — parent passes new props.  
4. **Context value change** — a used context provider updates.  
5. **Force update** — using `forceUpdate()` or equivalent.  
6. **Parent re-render** — unless child is memoized and props unchanged.

> Quick note: React treats functional components as pure functions of props + state — when React decides to render, it *calls* the function again to produce UI description (VDOM).

---

## Step 2 — Component function/class `render()` executes
- **Functional components**: the function is invoked; hooks (like `useState`, `useEffect` declarations) are processed in the same order every render. The function returns JSX.
- **Class components**: React calls the `render()` method which returns JSX.

This output is *not* the real DOM — it’s a tree of plain JavaScript objects (the Virtual DOM / element descriptors).

```jsx
function Counter({ initial = 0 }) {
  const [count, setCount] = useState(initial);
  return (
    <div>
      <h1>{count}</h1>
      <button onClick={() => setCount(c => c + 1)}>Inc</button>
    </div>
  );
}
```

---

## Step 3 — React builds the Virtual DOM (VDOM)
From the returned JSX, React creates a lightweight JS object tree representing elements and props:
```js
{ type: 'div', props: { children: [...] } }
```
This tree is cheap to create and easy to compare.

---

## Step 4 — Reconciliation (VDOM vs previous VDOM)
**Reconciliation** = comparing the new VDOM tree with the previous VDOM tree to find the smallest set of changes. React follows a set of _heuristics_ that make reconciliation fast (not a full tree comparison brute force).

**Key rules used during diffing**:
1. **Different element types → replace node**  
   - `<div>...</div>` → `<span>...</span>` → React unmounts old node and mounts new one.  
2. **Same type → compare props**  
   - If props differ, update attributes/props on that node only.  
3. **Children lists → use `key`**  
   - When rendering lists, React uses `key` to match items between renders and avoid re-creating DOM nodes unnecessarily. Without reliable keys, React falls back to index-based matching and may re-create/move nodes inefficiently.  
4. **Text nodes** → updated directly when text changes.  

> The goal: minimize real DOM mutations because DOM updates are expensive.

---

## Step 5 — Diffing examples (concrete)
### Example A — simple text change
Before: `<h1>Hello</h1>`  
After: `<h1>Hello World</h1>`  
Result: Only the text node updates.

### Example B — element type change
Before: `<div class="card" />`  
After: `<span class="card" />`  
Result: React replaces the `div` with a `span` (unmount old, mount new).

### Example C — list with keys
```jsx
// Before
items = [{id:1, name:'A'}, {id:2, name:'B'}]

// After (B moved in front)
items = [{id:2, name:'B'}, {id:1, name:'A'}]

// If using key={item.id}: React will reorder DOM nodes.
// If no keys or index keys: React may destroy and recreate nodes.
```

---

## Step 6 — React Fiber: scheduling and incremental work
React Fiber (React 16+) re-architected the reconciler so rendering work is split into **units of work**. Reasons:
- Make rendering **interruptible** (React can pause work to keep app responsive).  
- Prioritize important updates (e.g., animations, user input).

Implication: the **render phase** can be paused, aborted, or resumed — which makes large trees render more smoothly. The **commit phase** (DOM updates) runs synchronously once a chunk of work is ready to apply.

---

## Step 7 — Two-phase process: Render Phase vs Commit Phase
React rendering is conceptually two-phase:

1. **Render Phase (diffing, pure)**  
   - Build new VDOM tree.  
   - Compare with old VDOM (reconciliation).  
   - Prepare side-effects but **do not mutate the DOM**.  
   - This phase is *interruptible* (Fiber allows pausing).

2. **Commit Phase (apply changes, side-effects)**  
   - Apply DOM mutations (inserts, updates, removals).  
   - Run lifecycle methods and effects that must happen after DOM update (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount` cleanup).  
   - This phase is **synchronous** and completes without interruption.

**Effects timing**:  
- `useLayoutEffect` (and legacy `componentDidMount` behavior for layout) runs **after DOM mutations but before the browser paints** — useful for measuring DOM/layout.  
- `useEffect` runs **after the browser paints** (asynchronous), ideal for network requests, subscriptions, logging.

---

## Step 8 — Commit: update real DOM and refs
During commit React:
- Applies attribute/property changes to DOM nodes.  
- Inserts new DOM nodes and removes old ones.  
- Calls refs (assigns DOM refs).  
- Invokes lifecycle methods tied to commit.  

Because DOM updates happen only in commit phase, React keeps UI consistent and predictable.

---

## Step 9 — Browser paint & post-commit effects
After the commit finishes, the browser repaints/reflows as needed. Then React runs **passive effects** (callbacks from `useEffect`). This ordering guarantees that `useLayoutEffect` can measure layout synchronously after DOM changes but before paint, while `useEffect` won't block painting.

---

## Step 10 — Example walkthrough (initial mount → update)
**Initial render (mount):**  
1. App mounts → React calls component functions → builds VDOM for whole tree.  
2. Reconciliation with empty previous tree → all nodes are created.  
3. Commit phase mounts real DOM nodes.  
4. `useLayoutEffect` runs → browser paints → `useEffect` runs.  

**Update (user clicks button):**  
1. `setState` called → React schedules an update for that component.  
2. Render phase: React calls function again → returns new VDOM (with updated text).  
3. Diffing compares changed node(s) → determines minimal patch.  
4. Commit phase: patch real DOM (update text), update refs, call lifecycle hooks.  
5. Browser paints → `useEffect` runs (if used).

---

## Step 11 — When *doesn't* React re-render?
- Component does **not** re-render if props and state are identical and its parent re-renders (unless parent forces a new prop reference).  
- Use `React.memo()` / `PureComponent` to avoid re-render if props unchanged.  
- Avoid creating new object/array literals inline if it causes prop identity to change each render.

---

## Step 12 — Common optimizations (practical)
- **Use keys** for list items (`key` stable & unique).  
- **React.memo** for pure functional components.  
- **useMemo** to memoize expensive computed values.  
- **useCallback** to memoize callback references passed down as props.  
- **Avoid anonymous functions/objects** in props when possible.  
- **Split big components** into smaller ones (reduces work per update).  
- **Code-splitting** + `React.lazy` to reduce initial work.

---

## Step 13 — Edge cases & gotchas
- **Mutating DOM outside React** (e.g., manual DOM changes) can confuse reconciliation. Prefer refs + controlled updates.  
- **Missing keys** in dynamic lists lead to incorrect moves/updates.  
- **Index-as-key** is okay for static lists, bad for lists that reorder/add/remove.  
- **Forcing synchronous state updates in event handlers** is safe; but avoid heavy synchronous work during commit (affects UI responsiveness).

---

## Step 14 — Quick cheat sheet (for interviews)
- Virtual DOM = lightweight JS tree of UI description.  
- Reconciliation = diffing new VDOM vs old VDOM to compute minimal DOM changes.  
- Diffing rules: different types → replace; same type → update props; lists use keys.  
- Render phase = build and diff VDOM (interruptible).  
- Commit phase = apply DOM changes, run layout effects (synchronous).  
- `useLayoutEffect` before paint; `useEffect` after paint.  
- Fiber = scheduler that splits work, enabling prioritization and interruption.  
- Optimize with keys, memoization, and smaller components.

---

## Step 15 — Short sample Q&A (to practice)
**Q: What happens when you call `setState`?**  
A: React schedules an update → render phase builds new VDOM → reconciliation diffs with old VDOM → commit applies minimal DOM updates → effects/lifecycles run per timing rules.

**Q: Why are keys important?**  
A: Keys let React match list children between renders. Stable unique keys reduce unnecessary DOM operations and preserve component identity.

**Q: When should I use `useLayoutEffect` vs `useEffect`?**  
A: Use `useLayoutEffect` for reading/writing layout (measurements) before paint. Use `useEffect` for non-blocking side-effects after paint (network, subscriptions).

---

## Mermaid flow (visual)
```mermaid
flowchart TD
  A[Trigger: state/props/context change] --> B[Render Phase: call component render]
  B --> C[Build New Virtual DOM]
  C --> D[Diffing / Reconciliation (compare old VDOM)]
  D --> E{Changes?}
  E -- yes --> F[Prepare patches (Fiber scheduling)]
  F --> G[Commit Phase: apply patches to real DOM]
  G --> H[Run layout effects (useLayoutEffect)]
  H --> I[Browser paint]
  I --> J[Run passive effects (useEffect)]
  E -- no --> J
```


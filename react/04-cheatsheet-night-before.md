# ⚡ Night-Before Cheat Sheet — React Interview

Read this last tonight, and once more in the morning. Pure recall, no theory.

---

## 🪝 Every hook in one line each

| Hook | What it does | 🧸 ELI5 |
|---|---|---|
| `useState` | Local state + re-render | Your backpack |
| `useEffect` | Side effect after paint | "After drawing, do this" |
| `useLayoutEffect` | Side effect **before** paint | Check the wall before anyone sees it |
| `useContext` | Read a context value | Listen to the loudspeaker |
| `useReducer` | State via a rulebook | Follow the recipe card |
| `useMemo` | Cache a **value** | Write the answer on your hand |
| `useCallback` | Cache a **function** | Same, but for a recipe |
| `useRef` | Mutable box, no re-render | Sticky note on the fridge |
| `useImperativeHandle` | Choose what a ref exposes | Only these buttons are pressable |
| `useTransition` | Mark an update non-urgent | "Do this whenever you get a moment" |
| `useDeferredValue` | Lag a value behind | Use yesterday's number for now |
| `useId` | SSR-safe unique id | Matching name tags on both sides |
| `useSyncExternalStore` | Safe external subscription | Don't show a half-old picture |
| `useOptimistic` (19) | Show the result before it lands | Heart turns red instantly |
| `use` (19) | Read a promise/context | Can be conditional, unlike other hooks |

---

## 📖 One-sentence definitions (say these first, then the analogy)

| Term | Definition |
|---|---|
| **React** | A declarative, component-based JavaScript library for building user interfaces. |
| **JSX** | A syntax extension compiled to `React.createElement` calls that return plain objects. |
| **Component** | A function that takes props and returns a React element describing the UI. |
| **Props** | Read-only inputs passed from parent to child; immutable in the receiving component. |
| **State** | Data owned by a component that, when changed via its setter, triggers a re-render. |
| **Virtual DOM** | An in-memory JS tree React diffs against the previous one to compute minimal DOM mutations. |
| **Reconciliation** | The diffing process, made O(n) by two heuristics: different types replace the subtree, and keys identify stable children. |
| **Key** | A prop giving a list item stable identity across renders so React can match old and new elements. |
| **Hook** | A function letting function components use state and other React features; tracked by call order. |
| **Side effect** | Work reaching outside the render process — fetching, subscriptions, timers, DOM writes. |
| **Custom hook** | A `use`-prefixed function calling other hooks to share stateful **logic** — not state. |
| **Controlled input** | An input whose value is driven by React state (single source of truth). |
| **Uncontrolled input** | An input whose value the DOM owns, read imperatively via a ref. |
| **Lifting state up** | Moving shared state to the closest common ancestor of the components that need it. |
| **Prop drilling** | Passing props through components that don't use them, purely to reach a descendant. |
| **Context** | Dependency injection through the tree; **not** a state manager — no selectors, all consumers re-render. |
| **Memoization** | Caching a result between renders, keyed by a dependency array compared with `Object.is`. |
| **Re-render** | React calling a component function again — not necessarily a DOM update. |
| **Stale closure** | A callback still referencing values captured from an earlier render. |
| **Race condition** | Async responses arriving out of order, so a stale one overwrites fresher state. |
| **Derived state** | A value computable from existing state/props — should be calculated, never stored. |
| **Error boundary** | A class component catching render-phase errors in its subtree and rendering a fallback. |
| **Portal** | Rendering into a DOM node outside the parent's hierarchy while staying in the React tree. |
| **Code splitting** | Breaking the bundle into chunks loaded on demand at dynamic `import()` boundaries. |
| **Virtualization** | Rendering only the visible window of a list plus overscan, keeping DOM size constant. |
| **Fiber** | The reconciler representing work as a linked-list tree, making rendering interruptible. |
| **Concurrent rendering** | React preparing UI versions at multiple priorities and interrupting low-priority work. |
| **Transition** | A state update marked non-urgent, which React may interrupt, pause, or restart. |
| **Suspense** | A boundary rendering a fallback while a descendant signals it isn't ready. |
| **Hydration** | Attaching the client React tree and event handlers to server-rendered HTML. |
| **Tearing** | Components in one commit reading different values of the same external store. |
| **RSC** | Components executing only on the server, shipping zero JS and streaming a serialized UI payload. |
| **Server state** | Remotely-owned async data that can go stale — needs caching and revalidation, not Redux. |

---

## 🔥 The traps they love to set

```jsx
// 1. Stale state — the "adds 1 not 3" trap
setCount(count + 1); setCount(count + 1);   // ❌ = 1
setCount(c => c + 1); setCount(c => c + 1); // ✅ = 2

// 2. Mutation — React won't notice
items.push(x); setItems(items);             // ❌
setItems([...items, x]);                    // ✅

// 3. The `0` leak
{items.length && <List />}                  // ❌ renders "0"
{items.length > 0 && <List />}              // ✅

// 4. Calling instead of passing
<button onClick={handleClick()}>            // ❌ fires on render
<button onClick={handleClick}>              // ✅

// 5. Index as key on a mutable list → state lands on the wrong row

// 6. Unmemoized provider value → every consumer re-renders
<Ctx.Provider value={{ a, b }}>                        // ❌
<Ctx.Provider value={useMemo(() => ({a,b}), [a,b])}>   // ✅

// 7. async straight into useEffect
useEffect(async () => {}, []);              // ❌ returns a Promise, not cleanup

// 8. Race condition — old response overwrites new
return () => controller.abort();            // ✅ always clean up fetches

// 9. Uncontrolled → controlled warning
useState()                                  // ❌ undefined
useState("")                                // ✅

// 10. Object literal in a dependency array → effect runs every render
```

---

## 📋 Snippets to have in muscle memory

**Debounce hook:**
```jsx
function useDebounce(value, delay = 500) {
  const [d, setD] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setD(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return d;
}
```

**Fetch with abort + all three states:**
```jsx
useEffect(() => {
  const c = new AbortController();
  setLoading(true);
  fetch(url, { signal: c.signal })
    .then(r => { if (!r.ok) throw new Error(r.status); return r.json(); })
    .then(setData)
    .catch(e => { if (e.name !== "AbortError") setError(e); })
    .finally(() => setLoading(false));
  return () => c.abort();
}, [url]);
```

**Controlled form, one handler for all fields:**
```jsx
const [form, setForm] = useState({ email: "", password: "" });
const onChange = e => setForm(p => ({ ...p, [e.target.name]: e.target.value }));
<input name="email" value={form.email} onChange={onChange} />
```

**Immutable array updates:**
```jsx
[...arr, item]                                        // add
arr.filter(x => x.id !== id)                          // remove
arr.map(x => x.id === id ? { ...x, done: true } : x)  // update
```

**Context + safe consumer hook:**
```jsx
const Ctx = createContext(null);
export const useCtx = () => {
  const v = useContext(Ctx);
  if (!v) throw new Error("useCtx must be inside <Provider>");
  return v;
};
```

---

## 🎤 Answers to have word-ready

**"Why is my component re-rendering?"**
> State changed, props changed, the parent re-rendered, or a context it consumes changed.
> The one people forget is the parent — a parent re-render re-renders the subtree regardless
> of props. I'd profile first, then fix it structurally by moving state down or passing
> `children`, before reaching for memo.

**"Explain the Virtual DOM."**
> A lightweight JS representation of the UI. React diffs the new tree against the old and
> applies the minimal set of real DOM mutations, because DOM writes are the expensive part.
> That process is reconciliation, and it's O(n) thanks to two heuristics: different element
> types replace the subtree, and keys identify stable children.

**"Why keys?"**
> Stable identity during reconciliation. Index keys break on reorder, insert, or delete —
> React matches the wrong old element to the new one and component state ends up on the wrong
> row.

**"useMemo vs useCallback vs React.memo?"**
> `useMemo` caches a value, `useCallback` caches a function, `React.memo` skips a component's
> re-render on shallow-equal props. They work as a team — memo on the child only pays off if
> the parent keeps prop references stable. And I don't apply them by default; they cost memory
> and add stale-closure risk.

**"How do you handle a race condition?"**
> The effect cleanup either aborts the request with `AbortController` or sets an `ignore`
> flag so only the latest effect writes state. In production I use TanStack Query, which
> handles it along with caching and dedup.

**"What's the difference between server and client state?"**
> Server state is async, shared, and can go stale — it needs caching, dedup and
> revalidation. Client state is synchronous and owned by the app. Mixing them is why people
> end up hand-writing loading flags for every endpoint in Redux.

---

## 🧮 Quick comparison tables

**Props vs State**
| | Props | State |
|---|---|---|
| Owner | Parent | The component |
| Mutable | ❌ | ✅ via setter |

**`useState` vs `useRef`**
| | `useState` | `useRef` |
|---|---|---|
| Re-renders | ✅ | ❌ |
| For | Things on screen | Things not on screen |

**`useEffect` vs `useLayoutEffect`**
| | `useEffect` | `useLayoutEffect` |
|---|---|---|
| Timing | After paint | Before paint |
| Blocks paint | ❌ | ✅ |
| Use for | Fetch, subscribe | DOM measurement, anti-flicker |

**Rendering strategies**
| | Built | Best for |
|---|---|---|
| CSR | In browser | Dashboards behind login |
| SSR | Per request | Personalized + SEO |
| SSG | At build | Docs, marketing |
| ISR | Build + revalidate | Big catalogs |

---

## ⚛️ React 18 / 19 talking points

**React 18:** `createRoot`, automatic batching everywhere, `useTransition`,
`useDeferredValue`, `useId`, `useSyncExternalStore`, streaming SSR, StrictMode double-effects.

**React 19:** `use()` hook, Server Components + Server Actions, `useOptimistic`,
`useActionState`, `ref` as a plain prop (no `forwardRef`), document metadata hoisting,
React Compiler for automatic memoization.

Mentioning **React Compiler** and **`useSyncExternalStore`** unprompted signals you actually
follow React, not just tutorials.

---

## 🕐 Morning of

- [ ] Re-read this file (10 min)
- [ ] Say your 3 STAR stories out loud once
- [ ] Have your 2–3 questions for them ready
- [ ] Test camera, mic, and your coding environment **before** the call
- [ ] Water nearby. Notes closed unless it's explicitly open-book.

**Last thing:** they're not trying to catch you out. They want to know what it's like to
work with you. Think out loud, name your tradeoffs, and say "I don't know" when you don't.

**Go get it. 🍀**

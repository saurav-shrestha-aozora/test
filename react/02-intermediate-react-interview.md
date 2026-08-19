# 🟡 INTERMEDIATE React Interview — Questions & Answers (Explained Like You're 5)

> **Every question has 4 parts:** **📖 Definition** (formal — open with this) · **🧸 ELI5**
> (the analogy) · **Code** · **🗣️ Say this** (what to speak out loud).
>
> This is the level where interviews are won or lost. Juniors get asked "what is state?".
> Intermediates get asked **"why did this re-render?"** and **"how would you fix this bug?"**

---

## Q1. What is `useReducer` and when do you use it instead of `useState`?

**📖 Definition:** `useReducer` is a hook for managing state through a **reducer function** — a
pure function with the signature `(state, action) => newState`. It returns the current state and
a **`dispatch`** function that sends action objects to the reducer. It centralizes all state
transition logic in one testable place, and the `dispatch` identity is **guaranteed stable across
renders**, so it can be passed down without breaking memoization.

**🧸 ELI5:** `useState` is telling someone "put the toy here." `useReducer` is handing them a
**rulebook**: "when I say ADD_TOY, do this; when I say REMOVE_TOY, do that." All the rules live in
one place.

```jsx
import { useReducer } from "react";

const initialState = { count: 0, step: 1 };

function reducer(state, action) {
  switch (action.type) {
    case "increment": return { ...state, count: state.count + state.step };
    case "decrement": return { ...state, count: state.count - state.step };
    case "setStep":   return { ...state, step: action.payload };
    case "reset":     return initialState;
    default: throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: "increment" })}>+</button>
      <button onClick={() => dispatch({ type: "setStep", payload: 5 })}>Step 5</button>
    </>
  );
}
```

**Use `useReducer` when:** multiple state values change together · the next state depends on the
previous in complex ways · you have 4+ related `useState` calls · you want the transition logic
unit-testable in isolation.

**🗣️ Say this:** "`useReducer` centralizes transitions into a pure function, which makes complex
state predictable and testable. It also gives me a stable `dispatch` to pass down instead of many
callbacks, which avoids re-render churn."

---

## Q2. What is the Context API?

**📖 Definition:** Context is React's built-in mechanism for **passing data through the component
tree without explicitly threading props at every level**. `createContext` produces a
Provider/Consumer pair; a `<Provider value={…}>` makes a value available to every descendant, and
`useContext` reads the value from the **nearest** matching Provider above. Context is a
**dependency-injection tool, not a state manager** — it has no selectors, so **every consumer
re-renders whenever the provider's `value` reference changes**.

**🧸 ELI5:** Instead of passing a note kid-to-kid across the classroom, the teacher puts it on a
**loudspeaker** 📢. Anyone who wants to listen hears it directly.

```jsx
import { createContext, useContext, useState, useMemo } from "react";

const ThemeContext = createContext(null);

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  const toggle = () => setTheme(t => (t === "light" ? "dark" : "light"));

  const value = useMemo(() => ({ theme, toggle }), [theme]);   // 👈 see the trap below

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}

// Consume — with a safety check (do this, it impresses)
export function useTheme() {
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error("useTheme must be used inside <ThemeProvider>");
  return ctx;
}

function Button() {
  const { theme, toggle } = useTheme();
  return <button onClick={toggle}>Theme is {theme}</button>;
}
```

**⚠️ The performance trap every interviewer looks for:**
```jsx
// ❌ New object EVERY render → every consumer re-renders, even unchanged ones
<Ctx.Provider value={{ user, setUser }}>

// ✅ Memoize it
const value = useMemo(() => ({ user, setUser }), [user]);
<Ctx.Provider value={value}>
```

**🗣️ Say this:** "Context solves prop drilling, but it isn't a state manager — any change
re-renders every consumer. I memoize the provider value and split contexts by update frequency,
e.g. a rarely-changing theme context separate from a fast-changing one."

---

## Q3. `useMemo` — what and when?

**📖 Definition:** `useMemo` **caches the result of a computation between renders**. It takes a
factory function and a dependency array, calls the factory only when a dependency has changed
(compared with `Object.is`), and otherwise returns the previously computed value. Its two
legitimate purposes are: skipping **genuinely expensive** recomputation, and preserving a
**stable object/array reference** across renders so downstream memoization or effect dependencies
work.

**🧸 ELI5:** You do a hard math problem and write the answer on your hand. Next time someone asks
the **same** question, read your hand instead of redoing the math.

```jsx
function ProductList({ products, query }) {
  // ❌ Re-filters 10,000 items on EVERY render, even when only `theme` changed
  const filtered = products.filter(p => p.name.includes(query));

  // ✅ Only recomputes when products or query actually change
  const filtered = useMemo(
    () => products.filter(p => p.name.includes(query)),
    [products, query]
  );

  return <ul>{filtered.map(p => <li key={p.id}>{p.name}</li>)}</ul>;
}
```

**Second reason — a stable reference:**
```jsx
const config = useMemo(() => ({ sort: "asc", limit: 10 }), []);
<MemoizedChild config={config} />
```

**⚠️ Don't memo everything.** `useMemo` costs memory plus a dependency comparison. For `a + b`
it's slower than just computing it.

**🗣️ Say this:** "`useMemo` caches a computed *value* between renders. I reach for it for
genuinely expensive computations or to keep references stable for memoized children — not by
default, because it isn't free."

---

## Q4. `useCallback` — what and when?

**📖 Definition:** `useCallback` **memoizes a function definition**, returning the same function
reference between renders until one of its dependencies changes. It is exactly equivalent to
`useMemo(() => fn, deps)`. It exists because function literals create a new reference on every
render, which defeats `React.memo` on a child and retriggers effects that list the function as a
dependency.

**🧸 ELI5:** Same idea as `useMemo`, but it remembers a **function** instead of a value.

```jsx
const Child = React.memo(({ onClick }) => {
  console.log("Child rendered");
  return <button onClick={onClick}>Click</button>;
});

function Parent() {
  const [count, setCount] = useState(0);

  // ❌ new function each render → Child re-renders on every count change
  // const handle = () => console.log("clicked");

  // ✅ stable reference → Child never re-renders
  const handle = useCallback(() => console.log("clicked"), []);

  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child onClick={handle} />
    </>
  );
}
```

**Rule of thumb:** it only helps when the consumer is memoized, or the function is a hook
dependency. Otherwise it's pure overhead.

**🗣️ Say this:** "`useCallback` memoizes a function reference. It's only useful when the child is
wrapped in `React.memo` or the function is a dependency of another hook."

---

## Q5. What is `React.memo`?

**📖 Definition:** `React.memo` is a **higher-order component** that memoizes the rendered output
of a function component. Before re-rendering, React **shallow-compares** the new props with the
previous props; if all are referentially equal, it **skips the re-render** and reuses the last
result. An optional second argument supplies a custom comparison function (`areEqual`) which
returns `true` to skip the render. It is a performance optimization only — never rely on it for
correctness.

**🧸 ELI5:** A **bouncer at the door**. "Are your props the same as last time? Then you don't need
to come in and redo your work."

```jsx
const ExpensiveList = React.memo(function ExpensiveList({ items }) {
  return <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul>;
});

// Custom comparison — return true to SKIP the re-render
const Row = React.memo(
  ({ user }) => <div>{user.name}</div>,
  (prev, next) => prev.user.id === next.user.id
);
```

**⚠️ It does a SHALLOW compare — these all defeat it:**
```jsx
<Memoized items={[1, 2, 3]} />          // ❌ new array every render
<Memoized style={{ color: "red" }} />   // ❌ new object every render
<Memoized onClick={() => {}} />         // ❌ new function every render
```

**🗣️ Say this:** "`React.memo` shallow-compares props and skips re-rendering when they're equal.
It only pays off if the props are referentially stable, so it usually needs `useMemo`/
`useCallback` on the parent side."

---

## Q6. "Why is my component re-rendering?" (asked constantly)

**📖 Definition:** A **re-render** is React calling a component function again to produce a new
element tree. It is *not* the same as a DOM update — React may re-render and then commit nothing
if the output is unchanged. A component re-renders when: its **state** changes, its **props**
change, its **parent re-renders** (by default, regardless of whether its own props changed), a
**context it consumes** changes, or a hook it uses schedules an update.

**🧸 ELI5:** If mom redecorates the living room, everyone in the house gets asked to stand up and
be re-checked — even the kid whose room didn't change.

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child />   {/* 😱 re-renders too, even with zero props */}
    </>
  );
}
```

**Fix 1 — `React.memo`:**
```jsx
const Child = React.memo(() => <p>I'm stable now</p>);
```

**Fix 2 — composition (no memo needed, usually better):**
```jsx
function Parent({ children }) {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      {children}   {/* 👈 element created by the grandparent — doesn't re-render */}
    </>
  );
}

<Parent><ExpensiveChild /></Parent>
```

**🗣️ Say this:** "A parent re-render re-renders the whole subtree by default. Before reaching for
memo I profile with React DevTools, then usually fix it structurally — moving state down or
lifting content up via `children`."

---

## Q7. What is a custom hook?

**📖 Definition:** A custom hook is a **JavaScript function whose name begins with `use` and which
calls other hooks**. It is the standard mechanism for extracting and reusing **stateful logic**
between components. Critically, custom hooks share **logic, not state** — each component that
calls the hook gets its own completely independent state. The naming convention is what allows
the linter to enforce the Rules of Hooks.

**🧸 ELI5:** You keep making the same sandwich every day, so you write the recipe on a card 📇 and
just follow the card. Each person who follows it gets their *own* sandwich.

```jsx
// 📇 Recipe 1 — fetch anything
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    setLoading(true);
    fetch(url, { signal: controller.signal })
      .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
      .then(setData)
      .catch(e => { if (e.name !== "AbortError") setError(e); })
      .finally(() => setLoading(false));

    return () => controller.abort();   // 🧹 cancels the old request
  }, [url]);

  return { data, loading, error };
}

// 📇 Recipe 2 — debounce a value
function useDebounce(value, delay = 500) {
  const [debounced, setDebounced] = useState(value);

  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);     // 🧹 restart the timer on every keystroke
  }, [value, delay]);

  return debounced;
}

// 📇 Recipe 3 — localStorage-backed state
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initial;
    } catch { return initial; }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

// 📇 Recipe 4 — previous value
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; }, [value]);
  return ref.current;
}

// 📇 Recipe 5 — media query
function useMediaQuery(query) {
  const [matches, setMatches] = useState(() => window.matchMedia(query).matches);

  useEffect(() => {
    const mql = window.matchMedia(query);
    const onChange = (e) => setMatches(e.matches);
    mql.addEventListener("change", onChange);
    return () => mql.removeEventListener("change", onChange);
  }, [query]);

  return matches;
}
```

**Composing two — a search box that doesn't hammer the API:**
```jsx
function Search() {
  const [query, setQuery] = useState("");
  const debouncedQuery = useDebounce(query, 500);
  const { data, loading } = useFetch(`/api/search?q=${debouncedQuery}`);

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {loading ? <Spinner /> : <Results data={data} />}
    </>
  );
}
```

**🗣️ Say this:** "Custom hooks extract stateful logic for reuse. Each caller gets isolated
state — hooks share logic, not state. That's what HOCs and render props used to do, without the
wrapper hell."

---

## Q8. `useRef` — the three real uses

**📖 Definition:** `useRef(initialValue)` returns a **stable mutable container** `{ current }`
that persists across the component's entire lifetime and whose mutation does **not** schedule a
re-render. Passing it to a DOM element's `ref` attribute makes React assign the underlying DOM
node to `.current` after commit and `null` on unmount.

```jsx
// 1️⃣ DOM access
const inputRef = useRef(null);
<input ref={inputRef} />
inputRef.current.focus();

// 2️⃣ Mutable value that must NOT trigger a re-render (timers, sockets, counters)
const timerRef = useRef(null);
const start = () => { timerRef.current = setInterval(tick, 1000); };
const stop  = () => { clearInterval(timerRef.current); };

// 3️⃣ Remember the previous value
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; }, [value]);
  return ref.current;
}
```

| | `useState` | `useRef` |
|---|---|---|
| Triggers re-render | ✅ Yes | ❌ No |
| Survives re-renders | ✅ | ✅ |
| Read during render | ✅ Safe | ⚠️ Avoid (breaks concurrent rendering) |
| Use for | Anything shown on screen | Anything not shown on screen |

---

## Q9. `useEffect` vs `useLayoutEffect`?

**📖 Definition:** Both accept an effect and a dependency array; they differ in **timing**.
`useEffect` is a **passive effect** — React fires it *asynchronously after* the browser has
painted, so it never blocks visual updates. `useLayoutEffect` is a **layout effect** — it fires
**synchronously after DOM mutation but before the browser paints**, so state updates made inside
it are flushed before the user sees anything. Because it blocks paint, it should be reserved for
DOM measurement and flicker prevention. It also warns during SSR, since there is no layout on the
server.

**🧸 ELI5:** `useEffect` = "paint the wall, THEN check it." `useLayoutEffect` = "check it BEFORE
anyone sees the wall."

```jsx
// ❌ Visible flicker — user sees position 0, then it jumps
useEffect(() => {
  setHeight(ref.current.getBoundingClientRect().height);
}, []);

// ✅ No flicker — runs synchronously before paint
useLayoutEffect(() => {
  setHeight(ref.current.getBoundingClientRect().height);
}, []);
```

**🗣️ Say this:** "`useLayoutEffect` fires after DOM mutation but before paint, so it's for
measurement and preventing flicker. It blocks painting, so I default to `useEffect` and only
escalate when there's a visible flash."

---

## Q10. Explain reconciliation and the diffing algorithm

**📖 Definition:** **Reconciliation** is the process by which React determines what changed
between the previous element tree and the next one. A general tree-diff is O(n³), so React uses an
O(n) **heuristic** algorithm built on two assumptions: **(1)** two elements of **different types**
produce entirely different trees, so React unmounts the old subtree and mounts a fresh one; and
**(2)** developers can hint at which children are stable across renders using the **`key`** prop.
Elements of the same type keep their DOM node and component instance, and only changed attributes
are patched.

**🧸 ELI5:** React compares the old picture and the new picture using two shortcuts, because
comparing perfectly would take forever.

**Shortcut 1 — different type = throw the subtree away:**
```jsx
<div><Counter /></div>   →   <span><Counter /></span>
// div ≠ span → React unmounts Counter and mounts a FRESH one. State is LOST.
```

**Shortcut 2 — same type = patch attributes only:**
```jsx
<div className="a" title="x" />  →  <div className="b" title="x" />
// Only className is updated. DOM node and component state preserved.
```

**Shortcut 3 — lists match by `key`.**

**The pro trick — force a remount:**
```jsx
<UserProfile key={userId} userId={userId} />   // changing key = fresh state
```

**🗣️ Say this:** "Reconciliation is O(n) instead of O(n³) because of two heuristics: different
element types produce different trees, and keys identify stable children. I sometimes exploit
that by changing `key` to intentionally reset a component's state."

---

## Q11. What is an Error Boundary?

**📖 Definition:** An error boundary is a **class component** that implements
`static getDerivedStateFromError()` and/or `componentDidCatch()`. It **catches JavaScript errors
thrown during rendering, in lifecycle methods, and in constructors of its child tree**, logs them,
and renders a fallback UI instead of unmounting the whole tree. It does **not** catch errors in
event handlers, asynchronous code, server-side rendering, or errors thrown in the boundary itself.
There is currently no hook equivalent.

**🧸 ELI5:** A **safety net** under the trapeze. If one acrobat falls, the whole circus doesn't
shut down — you just show "this act is having trouble."

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };          // update state → render fallback
  }

  componentDidCatch(error, info) {
    logToService(error, info.componentStack);  // side effect: report it
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong 😢</h2>
          <button onClick={() => this.setState({ hasError: false })}>Try again</button>
        </div>
      );
    }
    return this.props.children;
  }
}

<ErrorBoundary><Dashboard /></ErrorBoundary>
```

**🗣️ Say this:** "Error boundaries catch render-phase errors in their subtree and show a fallback
instead of white-screening. I place them at meaningful UI seams — per route, per widget. They
don't catch event handler or async errors, so those need try/catch and a global handler."

---

## Q12. What are Portals?

**📖 Definition:** A portal renders children into a **DOM node outside the parent component's DOM
hierarchy**, while keeping the child in the same position in the **React tree**. Created with
`createPortal(children, domNode)`. Because the React tree is unchanged, **context still flows
down and events still bubble through the React tree**, not the DOM tree. Portals exist to escape
CSS containment — `overflow: hidden`, `z-index` stacking contexts, and `transform`.

**🧸 ELI5:** A **secret tunnel** 🕳️. Your modal *belongs* to this component, but it *appears*
somewhere else in the HTML — usually at the end of `<body>`.

```jsx
import { createPortal } from "react-dom";

function Modal({ isOpen, onClose, children }) {
  useEffect(() => {
    const onKey = (e) => e.key === "Escape" && onClose();
    document.addEventListener("keydown", onKey);
    document.body.style.overflow = "hidden";     // lock scroll

    return () => {
      document.removeEventListener("keydown", onKey);
      document.body.style.overflow = "";
    };
  }, [onClose]);

  if (!isOpen) return null;

  return createPortal(
    <div className="overlay" onClick={onClose}>
      <div className="modal" onClick={e => e.stopPropagation()}>{children}</div>
    </div>,
    document.body           // 👈 renders HERE in the DOM
  );
}
```

**🗣️ Say this:** "Portals move the DOM output elsewhere while keeping the React tree intact — so
context and event bubbling still work as if it were nested. That's how modals escape
`overflow: hidden` and z-index traps."

---

## Q13. Code splitting with `lazy` and `Suspense`?

**📖 Definition:** **Code splitting** is breaking the bundle into chunks loaded on demand rather
than shipping everything upfront. `React.lazy(() => import("./X"))` wraps a dynamic import and
returns a component that suspends while its chunk downloads; a `<Suspense fallback={…}>` ancestor
renders the fallback during that period. The bundler creates a separate chunk at each dynamic
`import()` call site.

**🧸 ELI5:** Don't carry every schoolbook all day. Grab the math book **only when math starts**.

```jsx
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./Dashboard"));
const Settings  = lazy(() => import("./Settings"));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
        <Route path="/settings"  element={<Settings />} />
      </Routes>
    </Suspense>
  );
}
```

**Preloading on hover — a nice detail to mention:**
```jsx
const preload = () => import("./Dashboard");
<Link to="/dashboard" onMouseEnter={preload}>Dashboard</Link>
```

**🗣️ Say this:** "`React.lazy` plus dynamic import splits the bundle so route code downloads on
demand. I split at route boundaries first, then heavy components like editors or charts, and pair
it with an error boundary since a chunk can fail to load."

---

## Q14. HOC vs Render Props vs Custom Hooks?

**📖 Definition:** All three are patterns for **sharing logic between components**.
A **Higher-Order Component** is a function that takes a component and returns a new enhanced
component (`withAuth(Page)`). A **render prop** is a prop whose value is a function that a
component calls to determine what to render, letting the consumer control output while the
provider owns the logic. A **custom hook** is a `use`-prefixed function that shares logic directly,
without introducing any wrapper components. Hooks superseded both for logic reuse; HOCs and render
props remain useful when the *rendering itself* must be wrapped or delegated.

**🧸 ELI5:** Three ways to share the same superpower with different components.

```jsx
// 1️⃣ HOC — wraps a component
function withAuth(Component) {
  return function Wrapped(props) {
    const user = useUser();
    if (!user) return <Login />;
    return <Component {...props} user={user} />;
  };
}
const ProtectedPage = withAuth(Page);

// 2️⃣ Render props — pass a function as children
function MouseTracker({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const onMove = e => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener("mousemove", onMove);
    return () => window.removeEventListener("mousemove", onMove);
  }, []);
  return children(pos);
}
<MouseTracker>{({ x, y }) => <p>{x}, {y}</p>}</MouseTracker>

// 3️⃣ Custom hook — the modern answer ✅
function Component() {
  const { x, y } = useMouse();
  return <p>{x}, {y}</p>;
}
```

**🗣️ Say this:** "Custom hooks replaced both for logic reuse — no wrapper hell, no nesting
pyramid, better TypeScript inference. HOCs still fit cross-cutting concerns that wrap rendering,
and render props still fit headless components that hand back render control."

---

## Q15. Compound components pattern?

**📖 Definition:** The compound component pattern splits a single logical component into **several
cooperating sub-components that share implicit state** through context (e.g. `<Tabs>`,
`<Tabs.Tab>`, `<Tabs.Panel>`). The parent owns the state; the children subscribe to it. This gives
consumers full control over markup and layout without the parent exposing a large configuration
prop surface — modeled on native `<select>` / `<option>`.

**🧸 ELI5:** Like `<select>` and `<option>` — pieces that only make sense together and quietly
share information behind the scenes.

```jsx
const TabsContext = createContext();

function Tabs({ children, defaultTab = 0 }) {
  const [active, setActive] = useState(defaultTab);
  const value = useMemo(() => ({ active, setActive }), [active]);
  return <TabsContext.Provider value={value}>{children}</TabsContext.Provider>;
}

Tabs.List = ({ children }) => <div role="tablist">{children}</div>;

Tabs.Tab = ({ index, children }) => {
  const { active, setActive } = useContext(TabsContext);
  return (
    <button role="tab" aria-selected={active === index} onClick={() => setActive(index)}>
      {children}
    </button>
  );
};

Tabs.Panel = ({ index, children }) => {
  const { active } = useContext(TabsContext);
  return active === index ? <div role="tabpanel">{children}</div> : null;
};

// Usage — flexible, readable, no prop explosion
<Tabs>
  <Tabs.List>
    <Tabs.Tab index={0}>Profile</Tabs.Tab>
    <Tabs.Tab index={1}>Settings</Tabs.Tab>
  </Tabs.List>
  <Tabs.Panel index={0}>Profile content</Tabs.Panel>
  <Tabs.Panel index={1}>Settings content</Tabs.Panel>
</Tabs>
```

**🗣️ Say this:** "Compound components give consumers layout freedom while the parent owns shared
state via context. It avoids the 20-prop configuration API that component libraries drift into."

---

## Q16. `forwardRef` and `useImperativeHandle`?

**📖 Definition:** `ref` is not a normal prop and is not forwarded automatically.
**`forwardRef`** wraps a component so it receives a `ref` as a second argument, which it can attach
to an inner DOM node — letting parents reach through the abstraction.
**`useImperativeHandle(ref, createHandle, deps)`** customizes what that ref exposes, replacing the
raw DOM node with a **restricted, intentional API**. In **React 19**, `ref` is a regular prop on
function components, so `forwardRef` is no longer required.

**🧸 ELI5:** `forwardRef` = passing the door handle through the wall. `useImperativeHandle` =
deciding **which buttons** the outsider is allowed to press.

```jsx
// forwardRef — let the parent reach the inner DOM node
const Input = forwardRef(function Input(props, ref) {
  return <input ref={ref} {...props} />;
});

// useImperativeHandle — expose ONLY a chosen API
const FancyInput = forwardRef(function FancyInput(props, ref) {
  const inputRef = useRef();

  useImperativeHandle(ref, () => ({
    focus: () => inputRef.current.focus(),
    clear: () => { inputRef.current.value = ""; },
    // .value, .style etc. deliberately NOT exposed
  }), []);

  return <input ref={inputRef} />;
});
```

---

## Q17. How do you avoid race conditions in data fetching?

**📖 Definition:** A **race condition** occurs when multiple asynchronous requests are in flight
simultaneously and **responses arrive in a different order than the requests were made**, so a
stale response overwrites fresher state. In React the fix lives in the effect's **cleanup
function**, which runs before the next effect: either set an `ignore` flag that the resolved
handler checks before calling `setState`, or **abort** the request outright with `AbortController`.

**🧸 ELI5:** You order pizza, change your mind and order sushi. The pizza arrives *last* and you
end up eating pizza even though you asked for sushi.

```jsx
// ❌ BUGGY — a slow old response can overwrite a fast new one
useEffect(() => {
  fetch(`/api/search?q=${query}`).then(r => r.json()).then(setResults);
}, [query]);

// ✅ FIX 1 — ignore stale responses
useEffect(() => {
  let ignore = false;
  fetch(`/api/search?q=${query}`)
    .then(r => r.json())
    .then(data => { if (!ignore) setResults(data); });
  return () => { ignore = true; };
}, [query]);

// ✅ FIX 2 — actually cancel (better: saves bandwidth)
useEffect(() => {
  const controller = new AbortController();
  fetch(`/api/search?q=${query}`, { signal: controller.signal })
    .then(r => r.json())
    .then(setResults)
    .catch(e => { if (e.name !== "AbortError") setError(e); });
  return () => controller.abort();
}, [query]);
```

**🗣️ Say this:** "Responses can arrive out of order, so cleanup either aborts the request or sets
an ignore flag so only the latest effect writes state. In production I use TanStack Query, which
handles this plus caching and dedup."

---

## Q18. Why use TanStack Query / SWR instead of `useEffect` + `fetch`?

**📖 Definition:** These are **server-state management libraries**. They treat remote data as a
**cache keyed by a query key**, and provide declarative fetching with automatic deduplication,
background revalidation, stale-while-revalidate semantics, retries with backoff, pagination,
mutation with optimistic updates and rollback, and garbage collection of unused entries — all of
which must otherwise be hand-written per endpoint.

**🧸 ELI5:** `useEffect` + `fetch` is walking to the store every time you want milk. Query
libraries keep a **fridge** — check it first, and only go to the store when the milk is old.

```jsx
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";

function Users() {
  const { data, isLoading, error } = useQuery({
    queryKey: ["users"],
    queryFn: () => fetch("/api/users").then(r => r.json()),
    staleTime: 5 * 60 * 1000,   // fresh for 5 min — no refetch
  });

  if (isLoading) return <Spinner />;
  if (error) return <Error msg={error.message} />;
  return <ul>{data.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}

// Mutation with optimistic update + rollback
function useAddTodo() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (todo) => fetch("/api/todos", { method: "POST", body: JSON.stringify(todo) }),
    onMutate: async (newTodo) => {
      await qc.cancelQueries({ queryKey: ["todos"] });
      const previous = qc.getQueryData(["todos"]);
      qc.setQueryData(["todos"], old => [...old, newTodo]);   // 👈 instant UI
      return { previous };
    },
    onError: (err, vars, ctx) => qc.setQueryData(["todos"], ctx.previous),  // rollback
    onSettled: () => qc.invalidateQueries({ queryKey: ["todos"] }),
  });
}
```

**🗣️ Say this:** "Server state and client state are different problems. Server state is
asynchronous, shared, and can go stale — a cache library models that properly, so I stop
reimplementing loading, error, and dedup logic in every component."

---

## Q19. Context vs Redux vs Zustand — how do you choose?

**📖 Definition:** **Context** is dependency injection with no selector granularity — all
consumers re-render on any value change. **Redux Toolkit** is a centralized single-store
architecture with reducers, middleware, devtools and time-travel debugging, using `useSelector`
for granular subscriptions. **Zustand** is a minimal external store with hook-based selectors and
no Provider requirement. Both external stores use `useSyncExternalStore` internally, which is what
lets them subscribe safely under concurrent rendering.

| Need | Use |
|---|---|
| Theme, locale, current user — rarely changes | **Context** |
| Server data (API responses) | **TanStack Query** |
| Small/medium global client state | **Zustand** |
| Large app, complex flows, big team, time-travel debugging | **Redux Toolkit** |
| Form state | **react-hook-form** |
| URL-shaped state (filters, tabs, page) | **the URL itself** |

```jsx
// Zustand — tiny, no provider
import { create } from "zustand";

const useStore = create((set) => ({
  count: 0,
  increment: () => set(s => ({ count: s.count + 1 })),
}));

function Counter() {
  const count = useStore(s => s.count);   // 👈 selector = only this slice re-renders
  const increment = useStore(s => s.increment);
  return <button onClick={increment}>{count}</button>;
}
```

```jsx
// Redux Toolkit — modern, non-boilerplate Redux
import { createSlice, configureStore } from "@reduxjs/toolkit";

const counter = createSlice({
  name: "counter",
  initialState: { value: 0 },
  reducers: { increment: (state) => { state.value += 1; } },  // Immer makes this safe
});
```

**🗣️ Say this:** "I separate server state from client state first — that removes most of what
people put in Redux. Then Context for low-frequency global values, and Zustand or RTK only if
there's genuinely complex shared client state."

---

## Q20. React Router — the essentials

**📖 Definition:** React Router is a **client-side routing library** that maps URL paths to
components without a full page reload, using the History API. Core concepts: **route matching**
(`<Routes>` / `<Route>`), **nested routes** rendering into an `<Outlet>`, **dynamic segments**
(`:id`) read with `useParams`, programmatic navigation with `useNavigate`, and query strings as
state via `useSearchParams`.

```jsx
import { BrowserRouter, Routes, Route, Link, useParams,
         useNavigate, useSearchParams, Navigate, Outlet, useLocation } from "react-router-dom";

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="users/:id" element={<UserDetail />} />
          <Route element={<ProtectedRoute />}>
            <Route path="dashboard" element={<Dashboard />} />
          </Route>
          <Route path="*" element={<NotFound />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}

// Nested layout
function Layout() {
  return (<><nav><Link to="/">Home</Link></nav><Outlet /></>);   // 👈 child renders at Outlet
}

// Route params + programmatic nav
function UserDetail() {
  const { id } = useParams();
  const navigate = useNavigate();
  return <button onClick={() => navigate(-1)}>Back from user {id}</button>;
}

// Protected route
function ProtectedRoute() {
  const { user, loading } = useAuth();
  const location = useLocation();
  if (loading) return <Spinner />;
  if (!user) return <Navigate to="/login" state={{ from: location }} replace />;
  return <Outlet />;
}

// Query params as state — survives refresh, shareable
function ProductList() {
  const [searchParams, setSearchParams] = useSearchParams();
  const category = searchParams.get("category") ?? "all";
  return (
    <select value={category} onChange={e => setSearchParams({ category: e.target.value })}>
      <option value="all">All</option>
      <option value="books">Books</option>
    </select>
  );
}
```

---

## Q21. Testing with React Testing Library

**📖 Definition:** React Testing Library is a testing utility built on the principle that **tests
should resemble how users interact with the software**. It provides queries that find elements by
accessible attributes — role, label, text — rather than by implementation details like component
internals, state, or CSS classes. The result is tests that survive refactors and implicitly
verify accessibility.

```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";

test("adds a todo", async () => {
  const user = userEvent.setup();
  render(<TodoApp />);

  await user.type(screen.getByRole("textbox"), "Buy milk");
  await user.click(screen.getByRole("button", { name: /add/i }));

  expect(screen.getByText("Buy milk")).toBeInTheDocument();
});
```

**Query priority (this is the RTL philosophy):**
`getByRole` > `getByLabelText` > `getByPlaceholderText` > `getByText` > `getByTestId` (last resort)

- `getBy` — must exist now, throws otherwise
- `queryBy` — may not exist, returns `null` (use for absence assertions)
- `findBy` — async, waits for it to appear

**🗣️ Say this:** "I test behavior, not implementation. If I refactor `useState` to `useReducer`
the tests shouldn't change — that's the signal I'm testing the right thing."

---

## Q22. What's the "derived state" antipattern?

**📖 Definition:** **Derived state** is any value that can be **computed from existing props or
state**. Storing it in its own state variable creates a **second source of truth** that must be
kept manually in sync, typically via an effect — which adds an extra render pass and introduces
a window where the two disagree. The correct approach is to compute it during render, memoizing
only if the computation is genuinely expensive.

**🧸 ELI5:** Don't write down an answer you can just **calculate**. If you know price and
quantity, don't store the total in a separate box that can go stale.

```jsx
// ❌ ANTIPATTERN — two sources of truth, they will drift apart
const [items, setItems] = useState([]);
const [total, setTotal] = useState(0);
useEffect(() => { setTotal(items.reduce((s, i) => s + i.price, 0)); }, [items]);

// ✅ Just calculate during render
const total = items.reduce((s, i) => s + i.price, 0);

// ✅ Memoize only if genuinely expensive
const total = useMemo(() => items.reduce((s, i) => s + i.price, 0), [items]);
```

**🗣️ Say this:** "If a value can be computed from existing state or props, it shouldn't be state.
Storing it duplicates the source of truth and adds a render cycle."

---

## Q23. Lazy initial state

**📖 Definition:** `useState` accepts either an initial value or an **initializer function**. When
given a plain value, that expression is **evaluated on every render** even though the result is
discarded after the first. Passing a function defers the computation so React calls it **only
during the initial mount**.

```jsx
// ❌ expensiveInit() runs on EVERY render (result discarded, but it still runs)
const [state, setState] = useState(expensiveInit());

// ✅ Runs only ONCE, on mount
const [state, setState] = useState(() => expensiveInit());

// Real example
const [items, setItems] = useState(() => JSON.parse(localStorage.getItem("items") || "[]"));
```

---

## Q24. What is automatic batching in React 18?

**📖 Definition:** **Batching** is grouping multiple state updates into a **single re-render** for
performance. React 17 batched only updates originating inside React event handlers; updates in
promises, `setTimeout`, and native event handlers each triggered their own render. **React 18
batches everywhere by default** ("automatic batching") when using `createRoot`. `flushSync` opts
out for the rare case where you need the DOM updated synchronously.

**🧸 ELI5:** Instead of three trips to the kitchen for three snacks, make **one trip**.

```jsx
function handleClick() {
  setCount(c => c + 1);
  setFlag(f => !f);
  setName("x");
  // React 18 → ONE re-render ✅  (React 17 inside a promise → three re-renders)
}

// Opt out when you must read layout between updates
import { flushSync } from "react-dom";
flushSync(() => setCount(c => c + 1));
// DOM is updated here
```

---

## Q25. Debouncing and throttling in React

**📖 Definition:** **Debouncing** delays invoking a function until a specified period has passed
**without further calls** — it collapses a burst into one trailing call (ideal for search-as-you-
type). **Throttling** guarantees a function runs **at most once per interval** regardless of call
frequency — it enforces a maximum rate (ideal for scroll and resize). In React, debouncing the
*value* rather than the *callback* keeps the input controlled and responsive.

```jsx
// Debounce — "wait until they stop typing"
function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

// Throttle — "at most once every 200ms"
function useThrottle(callback, delay) {
  const lastRun = useRef(0);
  return useCallback((...args) => {
    const now = performance.now();
    if (now - lastRun.current >= delay) {
      lastRun.current = now;
      callback(...args);
    }
  }, [callback, delay]);
}
```

**🗣️ Say this:** "Debounce waits for a pause — good for search inputs. Throttle guarantees a
maximum rate — good for scroll and resize."

---

## Q26. Infinite scroll with IntersectionObserver

**📖 Definition:** **IntersectionObserver** is a browser API that asynchronously reports when a
target element enters or leaves a viewport or ancestor. Compared with scroll event listeners it
runs off the main thread and fires only on threshold crossings rather than continuously. Infinite
scroll observes a **sentinel** element at the end of the list and requests the next page when it
becomes visible.

```jsx
function InfiniteList() {
  const [items, setItems] = useState([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const sentinelRef = useRef(null);

  useEffect(() => {
    fetch(`/api/items?page=${page}`)
      .then(r => r.json())
      .then(data => {
        setItems(prev => [...prev, ...data.items]);
        setHasMore(data.hasMore);
      });
  }, [page]);

  useEffect(() => {
    if (!hasMore) return;
    const observer = new IntersectionObserver(
      ([entry]) => { if (entry.isIntersecting) setPage(p => p + 1); },
      { rootMargin: "200px" }          // start loading before it's visible
    );
    const el = sentinelRef.current;
    if (el) observer.observe(el);
    return () => observer.disconnect();   // 🧹
  }, [hasMore]);

  return (
    <>
      {items.map(i => <Row key={i.id} item={i} />)}
      {hasMore && <div ref={sentinelRef}>Loading...</div>}
    </>
  );
}
```

---

## Q27. How do you render a huge list (10,000+ rows)?

**📖 Definition:** **Virtualization** (windowing) renders only the subset of list items currently
visible in the viewport, plus a small **overscan** buffer, while a spacer element preserves the
full scroll height. DOM node count stays constant regardless of dataset size, so memory and render
time no longer scale with the data. Libraries: TanStack Virtual, react-window, react-virtuoso.

**🧸 ELI5:** You only have 10 seats. Don't invite 10,000 people — invite the 10 who fit and swap
them as people scroll.

```jsx
import { useVirtualizer } from "@tanstack/react-virtual";

function BigList({ items }) {
  const parentRef = useRef();

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 50,
    overscan: 5,
  });

  return (
    <div ref={parentRef} style={{ height: 600, overflow: "auto" }}>
      <div style={{ height: virtualizer.getTotalSize(), position: "relative" }}>
        {virtualizer.getVirtualItems().map(v => (
          <div key={v.key} style={{
            position: "absolute", top: 0, left: 0, width: "100%",
            height: v.size, transform: `translateY(${v.start}px)`,
          }}>
            {items[v.index].name}
          </div>
        ))}
      </div>
    </div>
  );
}
```

**🗣️ Say this:** "Virtualization renders only the visible window plus overscan, so DOM node count
stays constant. The tradeoffs are Ctrl+F not finding off-screen rows and extra work for
variable-height items."

---

## Q28. `useId` — what's it for?

**📖 Definition:** `useId` generates a **unique, stable identifier string** that is guaranteed to
match between the server-rendered HTML and the client hydration pass. It exists specifically for
**accessibility attributes** that need to link elements (`htmlFor`/`id`, `aria-describedby`) in
reusable components, where a hardcoded id would collide across multiple instances. It is **not**
for list keys.

```jsx
function FormField({ label }) {
  const id = useId();
  return (
    <>
      <label htmlFor={id}>{label}</label>
      <input id={id} />
    </>
  );
}
```

---

## Q29. How do you optimize a slow React app?

**📖 Definition:** React performance optimization is a **measurement-driven process**, not a set
of reflexes. The workflow is: profile to find the actual bottleneck, classify it (excessive
re-renders vs. one expensive render vs. too many DOM nodes vs. bundle size vs. network), apply the
narrowest fix, then re-measure. Structural fixes — relocating state, composition — are generally
preferable to memoization, which adds memory cost and stale-closure risk.

```
1. MEASURE first — React DevTools Profiler, Lighthouse, the Performance tab.
2. CLASSIFY:
   Too many re-renders?  → memo / move state down / split context
   One slow render?      → useMemo the expensive computation
   Too many DOM nodes?   → virtualization
   Big bundle?           → code split, tree shake
   Slow network?         → cache, prefetch, paginate
3. Fix ONE thing, measure again.
```

**Concrete wins, in order of payoff:**
```jsx
// 1. Move state DOWN so fewer components re-render
// ❌ typing re-renders <ExpensiveTree />
function App() {
  const [text, setText] = useState("");
  return <><input value={text} onChange={e=>setText(e.target.value)} /><ExpensiveTree /></>;
}
// ✅ isolate the state
function SearchInput() { const [text, setText] = useState(""); return <input /*...*/ />; }
function App() { return <><SearchInput /><ExpensiveTree /></>; }

// 2. Lift content up via children (see Q6)
// 3. React.memo + useCallback/useMemo at the boundary
// 4. Virtualize long lists
// 5. Code split routes
// 6. Debounce expensive input handlers
```

**🗣️ Say this:** "I always profile before optimizing. Most React performance problems are
re-render problems, and the cheapest fixes are structural — moving state down or passing children
— before adding memoization, which has its own cost."

---

## Q30. How do you handle forms at scale? (react-hook-form)

**📖 Definition:** react-hook-form manages form state with **uncontrolled inputs registered via
refs**, so keystrokes do not re-render the whole form — only subscribed fields update. Validation
is pluggable via **resolvers**, commonly backed by a schema library such as Zod, giving one
declarative source of truth for both validation rules and inferred TypeScript types.

```jsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  email: z.string().email("Invalid email"),
  password: z.string().min(8, "At least 8 characters"),
});

function LoginForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } =
    useForm({ resolver: zodResolver(schema) });

  return (
    <form onSubmit={handleSubmit(async (data) => await login(data))}>
      <input {...register("email")} />
      {errors.email && <span>{errors.email.message}</span>}

      <input type="password" {...register("password")} />
      {errors.password && <span>{errors.password.message}</span>}

      <button disabled={isSubmitting}>Login</button>
    </form>
  );
}
```

**🗣️ Say this:** "react-hook-form keeps inputs uncontrolled, so typing doesn't re-render the whole
form — that matters on large forms. Pairing it with a Zod schema means one source of truth for
validation I can share with backend types."

---

## Q31. Accessibility basics in React

**📖 Definition:** Web accessibility (a11y) is the practice of making applications usable by
people with disabilities, formalized by **WCAG** and implemented in the browser through the
**accessibility tree**, which assistive technology reads. In React the primary lever is
**semantic HTML** — native elements carry built-in roles, keyboard behavior, and focus
management. ARIA attributes supplement semantics only where no native element exists.

```jsx
// ✅ Semantic HTML first — a <button> gives keyboard + screen reader support for free
<button onClick={close}>Close</button>
// ❌ <div onClick={close}>Close</div>  — not focusable, no Enter/Space, no role

// Label every input
<label htmlFor={id}>Email</label>
<input id={id} />

// Images
<img src="chart.png" alt="Revenue rose 20% in Q3" />   // meaningful
<img src="divider.png" alt="" />                       // decorative

// Announce dynamic changes
<div role="status" aria-live="polite">{message}</div>

// Modals need focus management
<div role="dialog" aria-modal="true" aria-labelledby={titleId}>
```

**🗣️ Say this:** "I reach for semantic HTML before ARIA — the first rule of ARIA is not to use
ARIA. I check keyboard navigation, focus trapping in modals, visible focus rings, and contrast,
and run axe DevTools plus jest-axe in CI."

---

## Q32. What are stale closures?

**📖 Definition:** A **stale closure** occurs when a function captures variables from a particular
render's scope and continues to reference those now-outdated values in a later render. Because
every render creates fresh bindings, a callback stored in an effect with an empty dependency array
permanently sees the values from the mount render. Fixes: list the value as a dependency, use the
**functional updater** form to eliminate the dependency, or store the latest value in a ref.

**🧸 ELI5:** You take a **photo** of the room. Later the room changes, but your photo still shows
the old room.

```jsx
// ❌ BUG — the interval closes over count = 0 forever
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000);  // always 0 + 1
  return () => clearInterval(id);
}, []);   // empty deps → the closure never updates

// ✅ FIX 1 — functional update, no dependency on count at all
setCount(c => c + 1);

// ✅ FIX 2 — a ref holding the latest value
const countRef = useRef(count);
useEffect(() => { countRef.current = count; }, [count]);
```

**🗣️ Say this:** "A stale closure is a callback holding values from an old render because the
dependency array didn't refresh it. The functional updater usually removes the dependency
entirely, which is the cleanest fix."

---

## Q33. Environment variables & config

**📖 Definition:** Build tools **statically replace** environment-variable references at build
time, inlining the literal values into the bundle. Only variables with the tool's public prefix
are exposed (`VITE_` for Vite, `NEXT_PUBLIC_` for Next.js) — a deliberate guard against leaking
server secrets. **Anything in the client bundle is public**, regardless of prefix.

```jsx
const API = import.meta.env.VITE_API_URL;        // Vite
const API = process.env.NEXT_PUBLIC_API_URL;     // Next.js
```

**⚠️ Never put a secret key in frontend code** — anyone can read it in DevTools. Secrets belong on
the server.

---

## Q34. Optimistic UI updates

**📖 Definition:** An **optimistic update** applies the expected result of a mutation to local
state **immediately**, before the server confirms it, and **rolls back** to the previous value if
the request fails. It trades a small risk of a temporarily incorrect UI for a large gain in
perceived responsiveness. Requirements: retain the previous state for rollback, and surface
failure to the user. React 19 formalizes the pattern with `useOptimistic`.

**🧸 ELI5:** You press "like" and the heart turns red **immediately**, before the server answers.
If the server says no, you quietly turn it back.

```jsx
function LikeButton({ postId, initialLiked }) {
  const [liked, setLiked] = useState(initialLiked);

  const toggle = async () => {
    const previous = liked;
    setLiked(!liked);                                 // 1️⃣ update instantly

    try {
      await fetch(`/api/posts/${postId}/like`, { method: "POST" });
    } catch {
      setLiked(previous);                             // 2️⃣ roll back
      toast.error("Couldn't save your like");
    }
  };

  return <button onClick={toggle}>{liked ? "❤️" : "🤍"}</button>;
}
```

---

## Q35. The "uncontrolled → controlled" warning

**📖 Definition:** React classifies an input as controlled or uncontrolled **on its first render**,
based on whether `value` is defined. If `value` starts as `undefined` (uncontrolled) and later
becomes a string (controlled), React warns, because switching modes mid-life produces unpredictable
behavior. The fix is to always initialize with a defined value.

```jsx
// ❌ value starts undefined
const [name, setName] = useState();
<input value={name} onChange={...} />

// ✅ Always initialize
const [name, setName] = useState("");

// ✅ Or guard a null API value
<input value={user?.name ?? ""} onChange={...} />
```

---

## Q36. How do you share state between sibling components?

**📖 Definition:** Siblings cannot communicate directly — React's data flow is unidirectional. State
must be **hoisted to a common ancestor** or moved into a shared external source. The options,
ordered from least to most coupling:

```
1. Lift state to the common parent    ← default choice
2. Context                            ← when the tree is deep
3. State library (Zustand/Redux)      ← when it's truly app-wide
4. URL / search params                ← when it should be shareable & bookmarkable
5. TanStack Query cache               ← when it's server data
```

---

## Q37. `React.memo` vs `useMemo` vs `useCallback`

**📖 Definition:** All three are memoization utilities operating at different granularities.
`React.memo` memoizes a **component's rendered output** based on shallow prop equality. `useMemo`
memoizes a **computed value** inside a component. `useCallback` memoizes a **function reference**
inside a component. They are complementary: the hooks keep props referentially stable so
`React.memo` can actually skip work.

| | Memoizes | Used on |
|---|---|---|
| `React.memo` | a **component** | wrapping a component |
| `useMemo` | a **value** | inside a component |
| `useCallback` | a **function** | inside a component |

---

## Q38. How would you implement dark mode?

**📖 Definition:** A theming system resolves an initial preference (persisted choice, else the
`prefers-color-scheme` media query), applies it as a **data attribute or class on the root
element**, and persists changes. Driving the palette through **CSS custom properties** means theme
switches are handled entirely by CSS, with no React re-render cost for styling.

```jsx
const ThemeContext = createContext();

export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState(() => {
    const saved = localStorage.getItem("theme");
    if (saved) return saved;
    return window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light";
  });

  useEffect(() => {
    document.documentElement.dataset.theme = theme;   // CSS reads [data-theme="dark"]
    localStorage.setItem("theme", theme);
  }, [theme]);

  const value = useMemo(
    () => ({ theme, toggle: () => setTheme(t => (t === "dark" ? "light" : "dark")) }),
    [theme]
  );

  return <ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>;
}
```

**🗣️ Say this:** "I persist the choice, fall back to `prefers-color-scheme`, and drive the palette
with CSS custom properties on a `data-theme` attribute so there's no re-render cost. For SSR I set
the attribute in an inline script before paint to avoid a flash of the wrong theme."

---

## Q39. Common intermediate bugs you should be able to spot

```jsx
// 🐛 1. Object/array literal in a dependency array → runs every render
useEffect(() => {...}, [{ a: 1 }]);         // new object every time
useEffect(() => {...}, [user.id]);          // ✅ compare a primitive

// 🐛 2. Provider value not memoized → all consumers re-render
<Ctx.Provider value={{ a, b }}>                    // ❌
const value = useMemo(() => ({ a, b }), [a, b]);   // ✅

// 🐛 3. Missing cleanup → memory leak / duplicate listeners
// 🐛 4. Index key + reordering → state lands on the wrong row
// 🐛 5. setState during render → infinite loop
// 🐛 6. Mutating props or state
// 🐛 7. useCallback with a non-memoized child → pure overhead
// 🐛 8. Non-unique keys → silently wrong DOM updates
```

---

## Q40. Live-coding: debounced search with cancellation

**📖 Definition:** This combines three intermediate concepts: **debouncing** the input value to
limit request frequency, **aborting** in-flight requests on cleanup to prevent race conditions, and
handling the **empty-query** edge case without a wasted request.

```jsx
function SearchUsers() {
  const [query, setQuery] = useState("");
  const [results, setResults] = useState([]);
  const [loading, setLoading] = useState(false);
  const debouncedQuery = useDebounce(query, 400);

  useEffect(() => {
    if (!debouncedQuery.trim()) { setResults([]); return; }

    const controller = new AbortController();
    setLoading(true);

    fetch(`/api/users?q=${encodeURIComponent(debouncedQuery)}`, { signal: controller.signal })
      .then(r => r.json())
      .then(setResults)
      .catch(e => { if (e.name !== "AbortError") console.error(e); })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [debouncedQuery]);

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} placeholder="Search..." />
      {loading && <Spinner />}
      <ul>{results.map(u => <li key={u.id}>{u.name}</li>)}</ul>
    </div>
  );
}
```

**Narrate as you code:** debounce so we don't hammer the API → abort so an old response can't
overwrite a new one → clear results on empty query → key by stable id.

---

## ✅ Intermediate checklist

- [ ] I can give the formal definition AND explain the Context re-render problem
- [ ] I can explain *why* a component re-rendered and name three fixes
- [ ] I know when memoization helps and when it's pure overhead
- [ ] I can write a custom hook with cleanup from memory
- [ ] I can fix a race condition two different ways
- [ ] I can explain reconciliation and the `key`-remount trick
- [ ] I can write an error boundary and say what it doesn't catch
- [ ] I can defend a state management choice with tradeoffs

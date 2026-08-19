# 🔴 SENIOR React Interview — Questions & Answers (Explained Like You're 5)

> **Every question has 4 parts:** **📖 Definition** (formal — open with this) · **🧸 ELI5**
> (the analogy) · **Code** · **🗣️ Say this** (what to speak out loud).
>
> At senior level nobody asks "what is `useState`". They ask **"what would you do, and what
> would it cost?"** Every answer below ends with a **tradeoff**, because naming the tradeoff is
> what separates senior from intermediate.

---

## Q1. What is React Fiber?

**📖 Definition:** Fiber is the **reconciliation engine** introduced in React 16, replacing the
recursive stack-based reconciler. It represents each element as a **fiber node** — a plain object
in a linked-list tree (`child`, `sibling`, `return` pointers) — so reconciliation becomes an
iterative loop over **units of work** rather than an uninterruptible recursive call stack. This
makes rendering **interruptible, resumable, and abortable**, enabling prioritization, time
slicing, and concurrent features. React maintains two trees (`current` and `workInProgress`) in a
**double-buffering** scheme, swapping them at commit.

**🧸 ELI5:** Old React was a kid who, once he started coloring, refused to stop until the whole
page was done — even if mom called him for dinner. Fiber is the same kid, but now he colors **a
little at a time**, looks up to check if anything more urgent is happening, and can even **throw
away** a drawing he no longer needs.

```js
// Simplified shape of a fiber node
{
  type: Component,          // what to render
  key,
  stateNode,                // the DOM node or class instance
  child, sibling, return,   // linked list pointers instead of recursion
  pendingProps, memoizedProps, memoizedState,
  flags,                    // commit work: Placement / Update / Deletion
  lanes,                    // priority
  alternate,                // the other tree (double buffering)
}
```

**Two phases — this is the answer they want:**

| Phase | What happens | Interruptible? |
|---|---|---|
| **Render / reconcile** | Build the new tree, call components, compute the diff | ✅ Yes — can pause, resume, or be discarded |
| **Commit** | Apply DOM mutations, run layout effects, then passive effects | ❌ No — synchronous and atomic |

**Why it matters practically:** because render can be discarded and re-run, your render function
must be **pure**. Side effects in render are genuinely dangerous now, not just untidy.

**🗣️ Say this:** "Fiber replaced the recursive stack reconciler with a linked list of units of
work, so React can yield to the browser between units. That's the foundation for priority lanes,
time slicing, and concurrent features. The practical consequence for my code is that render must
be pure and idempotent."

---

## Q2. What is concurrent rendering?

**📖 Definition:** Concurrent rendering is a set of capabilities — not a mode you switch on —
that let React **prepare multiple versions of the UI at the same time** and **interrupt an
in-progress render** when higher-priority work arrives. Updates are assigned to **lanes** by
priority; a low-priority render can be paused, discarded, or restarted without ever being shown
to the user. It is opt-in per update, via transitions and Suspense; `createRoot` merely makes the
capability available.

**🧸 ELI5:** A nurse in a waiting room. A kid with a paper cut is being treated, but someone
rushes in bleeding — the nurse **pauses** the paper cut and handles the emergency first.

**Priority lanes (highest first):**
```
Discrete input   — click, keypress                    ← never delayed
Continuous input — scroll, mousemove, drag
Default          — normal setState
Transition       — startTransition / useTransition    ← interruptible
Idle             — offscreen, prerender
```

```jsx
import { createRoot } from "react-dom/client";
createRoot(document.getElementById("root")).render(<App />);
```

**🗣️ Say this:** "Concurrency isn't 'faster rendering' — total work is the same or slightly more.
It's about *scheduling*: keeping the app responsive by letting urgent updates interrupt non-urgent
ones. Nothing is concurrent unless I opt in with transitions or Suspense."

---

## Q3. `useTransition` and `startTransition`?

**📖 Definition:** `useTransition` returns `[isPending, startTransition]`. State updates wrapped in
`startTransition` are marked as **transitions** — non-urgent updates that React may **interrupt,
pause, or restart** if higher-priority work (typing, clicking) arrives. `isPending` is a boolean
indicating that a transition is in flight, allowing a stale-but-interactive UI instead of a
blocking spinner. `startTransition` is also exported standalone for use outside components.

**🧸 ELI5:** You tell React: "Typing is **urgent** — show my letters now. Filtering 10,000 rows is
**not urgent** — do it when you have a moment, and it's fine to throw it away if I type again."

```jsx
function SearchableList({ items }) {
  const [query, setQuery] = useState("");          // urgent — the input
  const [list, setList] = useState(items);         // non-urgent — the results
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    setQuery(e.target.value);                      // 1️⃣ urgent: input never lags

    startTransition(() => {                        // 2️⃣ interruptible
      setList(items.filter(i => i.name.includes(e.target.value)));
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      <div style={{ opacity: isPending ? 0.6 : 1 }}>
        {list.map(i => <Row key={i.id} item={i} />)}
      </div>
    </>
  );
}
```

**⚠️ Critical detail interviewers probe:** the value you need to stay urgent must **not** be inside
the transition, or the input will still lag.

**🗣️ Say this:** "`useTransition` marks an update as non-urgent so React can interrupt and restart
it. `isPending` lets me show staleness without a spinner flash. It's for expensive *rendering*,
not slow *networks* — that's Suspense's job."

---

## Q4. `useDeferredValue` — how is it different?

**📖 Definition:** `useDeferredValue(value)` returns a **deferred copy** of a value that lags
behind during urgent updates. React first renders with the previous (stale) value at high
priority, then re-renders with the new value at transition priority — interruptibly. It addresses
the same problem as `useTransition` but from the **consumer side**: use it when you receive a
value you don't control (e.g. a prop) and therefore can't wrap the state update itself.

**🧸 ELI5:** Same goal, different handle. `useTransition` wraps the **update**; `useDeferredValue`
wraps the **value**.

```jsx
function Results({ query }) {           // query is a prop — I can't wrap its setter
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  const results = useMemo(() => expensiveFilter(deferredQuery), [deferredQuery]);

  return <div style={{ opacity: isStale ? 0.5 : 1 }}>{/* ... */}</div>;
}
```

**🗣️ Say this:** "Use `useTransition` when I control the update; `useDeferredValue` when I only
receive the value. Both need the expensive child memoized, otherwise deferring changes nothing."

---

## Q5. Explain Suspense properly.

**📖 Definition:** `<Suspense>` is a boundary component that renders a **fallback** while any
component in its subtree is **suspended** — i.e. has signalled it is not yet ready by throwing a
promise (or, in React 19, via `use()`). It moves loading state **out of individual components and
into the tree structure**, making "what shows a spinner, and when" a placement decision. Suspense
supports three use cases: code splitting (`React.lazy`), data fetching (framework- or
`use()`-driven), and **streaming SSR**, where the server flushes HTML for each boundary as it
resolves. Suspense does **not** catch errors — pair it with an error boundary.

**🧸 ELI5:** A component raises its hand and says "I'm not ready yet!" React shows the
**placeholder** for that whole area until everyone inside is ready.

```jsx
<Suspense fallback={<Skeleton />}>
  <UserProfile userId={id} />       {/* may suspend on data or code */}
</Suspense>
```

```jsx
// React 19 — the use() hook can read a promise
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);      // suspends until it resolves
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}
```

**Nested Suspense = progressive reveal — mention this:**
```jsx
<Suspense fallback={<PageSkeleton />}>
  <Header />                             {/* shows as soon as it's ready */}
  <Suspense fallback={<FeedSkeleton />}>
    <Feed />                             {/* streams in later */}
  </Suspense>
</Suspense>
```

**🗣️ Say this:** "Suspense lets a component declaratively signal 'not ready' and lets the parent
decide the loading UI, which moves loading state out of every leaf. The architectural win is that
boundary placement becomes a design decision — I control which parts of the page pop in and in
what order, avoiding waterfalls and layout shift."

---

## Q6. What are React Server Components?

**📖 Definition:** React Server Components (RSC) execute **exclusively on the server** and never
ship their code to the browser. They render to a **serialized description of the UI** (the RSC
payload) that is streamed to the client and merged into the React tree. They can be `async` and
access server-only resources — databases, filesystem, secrets — directly. They **cannot** use
state, effects, or event handlers. **Client Components**, marked with the `"use client"`
directive, are the interactive islands; the directive marks a **boundary**, so everything imported
beneath it also ships to the client. Props crossing the boundary must be **serializable**.

**🧸 ELI5:** Some toys stay at **grandma's house** (the server) and never come home. You get sent
a **photo** of the finished toy instead of the toy plus all its instructions. Less in your bag
(bundle), and grandma can reach the pantry (the database) directly.

```jsx
// app/posts/page.jsx — a Server Component by default in Next.js App Router
import db from "@/lib/db";

export default async function PostsPage() {
  const posts = await db.post.findMany();   // 👈 runs on the server, no API route needed

  return (
    <div>
      {posts.map(p => <PostCard key={p.id} post={p} />)}
      <LikeButton />                        {/* a Client Component island */}
    </div>
  );
}

// components/LikeButton.jsx
"use client";                               // 👈 this and its imports ship to the browser
import { useState } from "react";

export default function LikeButton() {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>{liked ? "❤️" : "🤍"}</button>;
}
```

| | Server Component | Client Component |
|---|---|---|
| Ships JS to browser | ❌ Zero | ✅ Yes |
| Hooks / state | ❌ No | ✅ Yes |
| `onClick` | ❌ No | ✅ Yes |
| Direct DB / secrets | ✅ Yes | ❌ Never |
| `async` component | ✅ Yes | ❌ No |

**Rules that trip people up:** a Client Component **cannot import** a Server Component, but it
*can* receive one as `children` — that's the composition escape hatch.

**🗣️ Say this:** "RSCs move component execution to the server so data-fetching code and heavy
dependencies never reach the client bundle — a markdown renderer can weigh zero KB on the client.
The tradeoff is a more complex mental model with two component kinds, a serialization boundary,
and framework lock-in. I'd adopt them for content-heavy, data-driven apps and stay with a client
SPA for highly interactive dashboards."

---

## Q7. SSR vs SSG vs ISR vs CSR — how do you choose?

**📖 Definition:** These are **rendering strategies** distinguished by *when* and *where* HTML is
produced. **CSR** ships an empty shell and renders in the browser. **SSR** renders HTML on the
server **per request**. **SSG** renders HTML **at build time** and serves it as static files.
**ISR** serves static HTML but **regenerates it in the background** after a revalidation interval
or an on-demand trigger, combining static performance with eventual freshness.

| Strategy | HTML built | Best for | Cost |
|---|---|---|---|
| **CSR** | In the browser | Dashboards behind login | Bad SEO, slow first paint |
| **SSR** | Per request | Personalized, always-fresh pages | Server cost, TTFB depends on data |
| **SSG** | At build time | Marketing, docs, blogs | Rebuild to update; slow builds at scale |
| **ISR** | Build + revalidate | Large catalogs that change sometimes | Some users see slightly stale content |

```jsx
// Next.js App Router
export const dynamic = "force-static";     // SSG
export const revalidate = 3600;            // ISR — regenerate hourly

fetch(url, { cache: "no-store" });                    // SSR, always fresh
fetch(url, { next: { revalidate: 60 } });             // ISR, 60s
fetch(url, { next: { tags: ["posts"] } });            // on-demand invalidation
```

**🗣️ Say this:** "I choose per route, not per app. Marketing pages are static, product pages are
ISR with on-demand revalidation from a CMS webhook, and the logged-in dashboard is client-rendered
because it's personal and not SEO-relevant."

---

## Q8. What is hydration, and what causes hydration errors?

**📖 Definition:** **Hydration** is the process by which React takes server-rendered HTML and
**attaches the client-side React tree to it** — building the fiber tree and wiring event handlers
— rather than re-creating the DOM. React expects the client's first render to produce **exactly
the markup the server produced**. A **hydration mismatch** means they disagree; React 18 then
discards the server HTML for that subtree and client-renders it, forfeiting the SSR benefit and
often causing a visible flash. Causes are almost always **non-deterministic render output**:
timestamps, randomness, locale, `window`/`localStorage` access, or invalid HTML nesting the browser
silently corrects.

**🧸 ELI5:** The server sends a **photo** of the page — you see it but nothing works. Hydration is
React walking through the photo attaching all the wires so buttons actually click. If the photo
and React's own drawing disagree, React gets confused.

```jsx
// ❌ 1. Non-deterministic values differ between server and client
<p>{new Date().toLocaleString()}</p>
<p>{Math.random()}</p>

// ❌ 2. Browser-only APIs during render
const width = window.innerWidth;           // window is undefined on the server

// ❌ 3. Invalid HTML nesting — the browser "fixes" it, React doesn't expect that
<p><div>hi</div></p>

// ✅ Fix — render only after mount
function ClientOnly({ children }) {
  const [mounted, setMounted] = useState(false);
  useEffect(() => setMounted(true), []);
  return mounted ? children : null;
}

// ✅ Or Next.js dynamic import with SSR off
const Chart = dynamic(() => import("./Chart"), { ssr: false });

// ✅ Or suppress for a single unavoidable node
<time suppressHydrationWarning>{new Date().toISOString()}</time>
```

**🗣️ Say this:** "Hydration attaches listeners and builds the fiber tree over server HTML. A
mismatch makes React 18 fall back to client rendering for that subtree, costing the SSR benefit.
The root cause is almost always non-deterministic output, so I isolate that behind a mount check.
Selective hydration means React prioritizes hydrating whichever boundary the user just interacted
with."

---

## Q9. What is `useSyncExternalStore` and why does it exist?

**📖 Definition:** `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot)` is the
official API for **subscribing to state that lives outside React** — Redux stores, browser APIs,
`localStorage`, WebSockets. It exists because concurrent rendering can **pause mid-render**; if an
external store mutates during that pause, different components in the same commit could read
different values, producing an inconsistent UI. This is called **tearing**. The hook guarantees a
consistent snapshot across the commit, and the third argument supplies the value used during SSR.

**🧸 ELI5:** React can now pause halfway through drawing. If an **outside** thing changes mid-draw,
half the page shows the old value and half the new — a **torn** picture. This hook prevents that.

```jsx
import { useSyncExternalStore } from "react";

function createStore(initial) {
  let state = initial;
  const listeners = new Set();
  return {
    getState: () => state,
    setState: (next) => { state = next; listeners.forEach(l => l()); },
    subscribe: (l) => { listeners.add(l); return () => listeners.delete(l); },
  };
}

const store = createStore({ count: 0 });

function Counter() {
  const state = useSyncExternalStore(
    store.subscribe,       // 1️⃣ subscribe fn
    store.getState,        // 2️⃣ client snapshot
    () => ({ count: 0 })   // 3️⃣ server snapshot (for SSR)
  );
  return <button onClick={() => store.setState({ count: state.count + 1 })}>{state.count}</button>;
}

// Practical use — subscribing to a browser API
function useOnlineStatus() {
  return useSyncExternalStore(
    (cb) => {
      window.addEventListener("online", cb);
      window.addEventListener("offline", cb);
      return () => {
        window.removeEventListener("online", cb);
        window.removeEventListener("offline", cb);
      };
    },
    () => navigator.onLine,
    () => true            // assume online during SSR
  );
}
```

**🗣️ Say this:** "It's the official way to read external mutable state without tearing under
concurrent rendering. Redux, Zustand and Jotai all use it internally. Before it, libraries used
`useEffect` + `useState`, which could show inconsistent values mid-render."

---

## Q10. How do you decide on a state management architecture?

**📖 Definition:** State architecture is the practice of **classifying state by its nature and
lifetime**, then choosing the mechanism that matches. The primary distinction is **server state**
(asynchronous, remotely owned, shared, can go stale — needs caching and revalidation) versus
**client state** (synchronous, locally owned). Secondary categories are **URL state** (should
survive refresh and be shareable), **global client state**, and **local component state**. Most
architectural pain comes from managing server state with client-state tools.

**🧸 ELI5:** Don't keep everything in one giant toy box. Sort by **where it lives** and **how often
it changes**.

```
Server state  → TanStack Query / RTK Query / SWR
                (async, cached, shared, can go stale)

URL state     → search params, route params
                (shareable, bookmarkable, survives refresh — filters, tabs, page)

Client global → Zustand / Jotai / Redux Toolkit
                (cart, multi-step wizard, editor undo stack)

Context       → theme, locale, auth user
                (rarely changes, truly tree-wide)

Local state   → useState / useReducer
                (DEFAULT — start here, promote only when proven necessary)

Form state    → react-hook-form
                (high-frequency, isolate from everything else)
```

**🗣️ Say this:** "The most common architectural mistake I see is putting server data in Redux and
hand-writing loading flags, cache invalidation, and dedup per endpoint. Separating server state into
a query cache typically deletes most of the global store. Then I keep client state as local as
possible and promote only when two distant components genuinely need it."

---

## Q11. How do you design a component library API?

**📖 Definition:** Component API design is the practice of defining a component's **public
contract** — its props, composition slots, and imperative handles — to be **hard to misuse, easy to
extend, and stable over time**. The dominant principle is **composition over configuration**:
expose structural slots via `children` rather than accumulating boolean and string props. Mature
libraries also support **both controlled and uncontrolled** usage, forward refs, spread unknown
props, and separate **headless behavior** (state, keyboard, ARIA) from presentation.

```jsx
// ❌ BAD — prop explosion, closed to extension
<Modal title="Delete" showCloseButton confirmText="Yes" cancelText="No"
       onConfirm={...} onCancel={...} size="md" showFooter headerColor="red" />

// ✅ GOOD — composition; consumers assemble what they need
<Modal open={open} onOpenChange={setOpen}>
  <Modal.Header>Delete item?</Modal.Header>
  <Modal.Body>This can't be undone.</Modal.Body>
  <Modal.Footer>
    <Button variant="ghost" onClick={close}>Cancel</Button>
    <Button variant="danger" onClick={confirm}>Delete</Button>
  </Modal.Footer>
</Modal>
```

**Principles worth naming out loud:**
1. **Composition over configuration** — `children` beats a 20th boolean prop
2. **Controlled + uncontrolled** — support both, like the DOM does
3. **Polymorphic `as`** — `<Button as="a" href="/x" />`
4. **Forward refs and spread the rest** — never block `aria-*`, `data-*`, or `className`
5. **Headless core + styled skin** — behavior and a11y separate from visuals
6. **Accessible by default** — the consumer shouldn't have to remember ARIA

```jsx
// Supporting controlled AND uncontrolled
function useControllableState({ value, defaultValue, onChange }) {
  const [internal, setInternal] = useState(defaultValue);
  const isControlled = value !== undefined;
  const current = isControlled ? value : internal;

  const setState = useCallback((next) => {
    if (!isControlled) setInternal(next);
    onChange?.(next);
  }, [isControlled, onChange]);

  return [current, setState];
}
```

**🗣️ Say this:** "I optimize for the API being hard to misuse and easy to extend. Every prop is a
permanent maintenance contract, so I prefer composition slots. I build on a headless primitive like
Radix so accessibility and focus management are correct by construction, not by review."

---

## Q12. Walk me through debugging a performance problem in production.

**📖 Definition:** Production performance work is a **measurement-driven diagnostic process**:
quantify with **field data** (Real User Monitoring — actual users, actual devices), reproduce under
representative conditions using a **production build**, localize the bottleneck with profiling
tools, classify it into a known category, apply the narrowest fix, and re-measure. Lab data
(Lighthouse on a dev machine) is a debugging aid, not evidence.

```
1. QUANTIFY — what is slow, and by how much? RUM: LCP, INP, CLS. Not "it feels slow."
2. REPRODUCE — same device class, same network throttle, PRODUCTION build.
   Dev builds are 2-5× slower and lie to you.
3. LOCATE
   Performance flame chart  → JS, layout, paint, or network?
   React Profiler           → which component, and why did it render?
   Network waterfall        → request waterfalls, missing preloads
   Coverage tab             → how much shipped JS is unused?
4. CLASSIFY
   Load        → bundle size, waterfalls, no code splitting
   Runtime     → re-render storms, expensive renders, huge lists
   Interaction → long tasks blocking the main thread (INP)
   Memory      → leaked listeners/timers, growing caches
5. FIX ONE THING, MEASURE, REPEAT. Add a CI regression guard.
```

| Symptom | Fix | Cost |
|---|---|---|
| Big initial bundle | Route-based code splitting | Loading states, chunk-load errors |
| Re-render storms | Move state down / `children` / memo | Memo has memory + complexity cost |
| Long list | Virtualization | Ctrl+F breaks, variable heights fiddly |
| Slow filter/sort | `useMemo` + `useDeferredValue` | Stale UI during the deferred pass |
| Blocked main thread | Web Worker | Serialization overhead, more infra |
| Waterfall requests | Parallel fetch / preload / RSC | Restructuring the data layer |

**🗣️ Say this:** "I never optimize without a measurement and a target. The most common real cause I
find is a context or top-level state re-rendering a large subtree on every keystroke — and the fix
is structural, not memoization."

---

## Q13. When should you NOT memoize?

**📖 Definition:** Memoization is **caching, and caching has costs**: retained memory, a dependency
comparison on every render, and a class of correctness bugs from incorrect dependency arrays
(stale closures). It is a net loss when the memoized computation is cheaper than the comparison,
when the consuming component isn't memoized, or when the props change every render anyway. It is
an optimization, never a correctness mechanism — code must be correct without it.

**🧸 ELI5:** Writing the answer on your hand takes time too. For `2 + 2`, just do the math.

```jsx
// ❌ Costs more than it saves
const doubled = useMemo(() => count * 2, [count]);
const name = useMemo(() => `${first} ${last}`, [first, last]);

// ❌ useCallback with a non-memoized child = pure overhead
const onClick = useCallback(() => {}, []);
<PlainChild onClick={onClick} />

// ❌ memo where props change every render anyway
const Row = React.memo(({ item, onSelect }) => /* ... */);
<Row item={item} onSelect={() => select(item.id)} />   // new function every render 🙃
```

**🗣️ Say this:** "Memoization is a cache, and caches have invalidation bugs and memory cost. I add
it where the Profiler shows a measurable win, or where a reference must be stable for correctness —
like an effect dependency. React Compiler is making most manual memoization unnecessary, which is a
good signal about how often we were doing it wrong by hand."

---

## Q14. What is the React Compiler?

**📖 Definition:** The React Compiler is a **build-time optimizing compiler** that analyzes
component code and automatically inserts memoization equivalent to hand-written `useMemo`,
`useCallback`, and `React.memo`. It performs data-flow analysis to determine which values can
safely be cached and which must be recomputed. It relies on components following the **Rules of
React** — pure render, no mutation of props or state, no side effects during render — and **bails
out safely** on code it cannot prove is compliant, which makes adoption incremental.

**🧸 ELI5:** A robot helper that reads your code and writes all the "remember this" notes for you.

```jsx
// You write this:
function ProductList({ products, query }) {
  const filtered = products.filter(p => p.name.includes(query));
  const handleClick = (id) => selectProduct(id);
  return <List items={filtered} onClick={handleClick} />;
}
// The compiler emits memoization automatically — no useMemo/useCallback needed.
```

**🗣️ Say this:** "The compiler auto-memoizes at build time via data-flow analysis, bailing out
safely where it can't prove purity. Strategically it means I write straightforward, pure components
and stop hand-tuning memoization — but I still need to understand *why* it memoizes, because I'll
be debugging the output."

---

## Q15. How do you prevent and find memory leaks?

**📖 Definition:** A memory leak in a React app is **retained memory that is no longer reachable
through the UI but is still referenced**, preventing garbage collection. The usual causes are
subscriptions, timers, and observers created in effects without a corresponding cleanup; closures
holding references to unmounted components' DOM nodes; and caches that grow without bound. Detection
is by **heap snapshot comparison** across repeated mount/unmount cycles, inspecting **detached DOM
node** counts and retainer chains.

**The four usual sources:**
```jsx
// 1. Uncleaned subscriptions
useEffect(() => {
  const sub = source.subscribe(handler);
  return () => sub.unsubscribe();          // 🧹
}, []);

// 2. Timers
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);          // 🧹
}, []);

// 3. Observers holding detached DOM
useEffect(() => {
  const node = ref.current;                // capture — ref.current may be null at cleanup
  const obs = new ResizeObserver(cb);
  obs.observe(node);
  return () => obs.disconnect();           // 🧹
}, []);

// 4. Unbounded caches / arrays that only grow
setLogs(prev => [...prev, entry].slice(-500));   // cap it
```

**How to find one:**
```
DevTools → Memory → heap snapshot
→ interact (mount/unmount the suspect route 10×)
→ second snapshot → Comparison view
→ Are Detached HTMLElement counts growing and never dropping?
→ Inspect the retainer chain to see what holds the reference.
```

**🗣️ Say this:** "Every subscription needs a symmetric teardown, and StrictMode's double-mount in
development is a free leak detector — if double-mounting breaks something, cleanup is missing. For
confirmation I compare heap snapshots across repeated mount/unmount cycles."

---

## Q16. Security in a React app — what do you watch for?

**📖 Definition:** The dominant client-side threat is **Cross-Site Scripting (XSS)** — injecting
attacker-controlled script into a trusted page. React escapes all interpolated string values by
default, so the realistic vectors are the explicit escape hatch `dangerouslySetInnerHTML`,
attacker-controlled URLs in `href`/`src` (e.g. `javascript:` schemes), and injecting untrusted data
into script contexts. The secondary concern is **credential handling** — anything in the client
bundle is public, and all authorization must be enforced server-side.

```jsx
// ❌ XSS — the number one React-specific footgun
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// ✅ Sanitize if you truly must render HTML
import DOMPurify from "dompurify";
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userContent) }} />

// ❌ javascript: URLs from user input
<a href={userUrl}>link</a>            // userUrl = "javascript:alert(1)"
const safe = /^https?:\/\//.test(userUrl) ? userUrl : "#";   // ✅ validate protocol

// ❌ Secrets in the client bundle — everything shipped is public
const API_KEY = import.meta.env.VITE_STRIPE_SECRET;   // visible in DevTools
```

**Auth token storage — a very common senior question:**

| Storage | XSS risk | CSRF risk | Verdict |
|---|---|---|---|
| `localStorage` | ❌ Readable by any script | ✅ Safe | Avoid for long-lived tokens |
| `httpOnly` cookie | ✅ Not script-readable | ❌ Needs `SameSite`/CSRF token | **Best** |
| In-memory | ✅ Good | ✅ Good | Best for short access tokens; lost on refresh |

**🗣️ Say this:** "React escapes interpolated values by default, so the main vectors are
`dangerouslySetInnerHTML` and URL attributes. For auth I use httpOnly, Secure, SameSite cookies with
a short-lived access token in memory — that survives XSS token theft. Plus a strict CSP, dependency
scanning, and never trusting client-side authorization checks, since they're UX only."

---

## Q17. How would you architect a large React app?

**📖 Definition:** Large-application architecture is primarily about **managing dependencies between
modules**. The dominant pattern is **feature-sliced (vertical) organization** — grouping code by
business domain rather than by technical type — where each feature exposes a **public API** through
an index module and its internals are private. The critical part is the **enforced dependency
direction**: features may not reach into each other's internals, and shared code must never depend
on features. Enforcement belongs in lint rules and CI, not convention.

```
src/
├── app/                    # composition root: providers, router, layout
├── pages/ (or routes/)     # route-level components — thin, assemble features
├── features/               # 👈 vertical slices by domain
│   ├── auth/
│   │   ├── api/            # queries & mutations for this feature
│   │   ├── components/     # components only this feature uses
│   │   ├── hooks/
│   │   ├── types.ts
│   │   └── index.ts        # 👈 the PUBLIC API — nothing else may be imported
│   ├── checkout/
│   └── dashboard/
├── shared/ (or lib/)       # cross-feature: ui kit, hooks, utils, api client
└── types/
```

**The rules that keep it from rotting:**
1. Features **may not** import each other's internals — only from `feature/index.ts`
2. `shared/` must never import from `features/` (one-way dependency direction)
3. Enforce with ESLint `import/no-restricted-paths` or dependency-cruiser in CI
4. Colocate: a component's test, styles, and story live beside it
5. Route-level code splitting falls out of this structure naturally

**🗣️ Say this:** "I organize by feature, not file type, because change is feature-shaped — a
`components/` folder with 300 files tells you nothing about the domain. The critical part isn't the
folder names, it's the enforced dependency direction. Without lint enforcement, module boundaries
are just a suggestion and they erode in about three sprints."

---

## Q18. Design a real-time collaborative feature (live dashboard or chat).

**📖 Definition:** A real-time feature maintains a **persistent bidirectional connection**
(WebSocket) or a **server-push stream** (Server-Sent Events) and reconciles pushed updates with
locally cached state. The engineering challenges are **connection lifecycle** (reconnect with
exponential backoff, plus a state resynchronization on reconnect to close the gap), **message
throughput** (coalescing updates so high-frequency messages don't trigger a render per message),
**ordering and idempotency**, and **conflict resolution** (last-write-wins, OT, or CRDT).

```jsx
// Connection layer with reconnect + backoff
function useWebSocket(url) {
  const [status, setStatus] = useState("connecting");
  const wsRef = useRef(null);
  const retriesRef = useRef(0);
  const handlersRef = useRef(new Set());

  useEffect(() => {
    let closed = false;
    let timer;

    const connect = () => {
      const ws = new WebSocket(url);
      wsRef.current = ws;

      ws.onopen = () => { setStatus("open"); retriesRef.current = 0; };
      ws.onmessage = (e) => {
        const msg = JSON.parse(e.data);
        handlersRef.current.forEach(h => h(msg));
      };
      ws.onclose = () => {
        if (closed) return;
        setStatus("reconnecting");
        const delay = Math.min(1000 * 2 ** retriesRef.current++, 30000);  // backoff
        timer = setTimeout(connect, delay);
      };
    };

    connect();
    return () => { closed = true; clearTimeout(timer); wsRef.current?.close(); };  // 🧹
  }, [url]);

  const subscribe = useCallback((handler) => {
    handlersRef.current.add(handler);
    return () => handlersRef.current.delete(handler);
  }, []);

  const send = useCallback((data) => {
    if (wsRef.current?.readyState === WebSocket.OPEN) {
      wsRef.current.send(JSON.stringify(data));
    }
  }, []);

  return { status, subscribe, send };
}
```

**The design decisions to talk through — this is what's actually assessed:**

| Concern | Decision |
|---|---|
| Transport | WebSocket for bidirectional; SSE if server→client only (simpler, auto-reconnect) |
| Reconnection | Exponential backoff + jitter; refetch a snapshot on reconnect to close the gap |
| Message volume | Batch/throttle updates into one state commit per animation frame |
| Ordering | Server sequence numbers; drop or reorder out-of-sequence messages |
| Offline | Queue outgoing messages; show a connection banner |
| Conflicts | Last-write-wins vs OT vs CRDT — say which and why |
| Re-render cost | Route messages into a store with selectors, not into Context |
| Backpressure | At 1000 msg/sec, coalesce — don't setState 1000 times |

**🗣️ Say this:** "The React part is the easy part. The hard parts are reconnection with state
reconciliation, and not melting the main thread under message volume — so I coalesce updates per
frame and use a selector-based store so only affected rows re-render."

---

## Q19. How do you approach a large migration?

**📖 Definition:** A migration is a **long-lived refactor across a codebase that must keep shipping
throughout**. The governing strategy is the **strangler fig pattern**: introduce the new approach
alongside the old, migrate incrementally at natural boundaries (route, module, feature), and
prevent regression with automated enforcement so the old pattern can only shrink. Big-bang branches
fail because they accumulate merge conflicts faster than they make progress.

```
1. INVENTORY   — what breaks? Codemods available? Which deps are unmaintained?
2. DE-RISK     — upgrade in a branch, run the full suite, check bundle + perf deltas
3. INCREMENTAL — small, independently revertible PRs. Never a 3-month branch.
4. GUARDRAILS  — a lint rule blocking the old pattern in NEW code, so the delta only shrinks
5. MEASURE     — track remaining old-pattern files; make the number visible to the team
```

**React 18 specifics worth naming:**
```jsx
ReactDOM.render(<App />, root);      // ❌ removed in 19
createRoot(root).render(<App />);    // ✅

// Automatic batching may change behavior in async code
// StrictMode double-invokes effects → exposes real cleanup bugs
// TypeScript: children is no longer implicit in FC
```

**🗣️ Say this:** "I favor the strangler pattern: run old and new side by side, migrate at a natural
boundary like a route, and enforce direction with lint so the codebase can't regress. A migration
that can't ship incrementally is one that gets abandoned half-finished."

---

## Q20. What's your testing strategy for a large React codebase?

**📖 Definition:** A testing strategy allocates effort across levels by **confidence per unit of
cost and maintenance**. **Unit tests** verify pure logic in isolation (fast, cheap, low
integration confidence). **Integration tests** render a feature with its real internal collaborators
and mock only at the **network boundary** (best value in React — they survive refactors).
**End-to-end tests** drive a real browser against a running system (highest confidence, slowest,
most brittle — reserve for critical revenue paths). **Static analysis** (TypeScript, ESLint) is the
widest and cheapest layer.

```
        /\        E2E (Playwright) — few, slow, high confidence
       /  \       → critical paths only: signup, checkout, core flow
      /____\
     /      \     Integration (RTL + MSW) — the bulk of the value 👈
    /        \    → a feature rendered whole, network mocked at the HTTP layer
   /__________\
  /            \  Unit (Vitest) — pure logic: reducers, utils, formatters
 /______________\
                  Static (TypeScript + ESLint) — widest, cheapest net
```

```jsx
// Integration test with MSW — mock the network, not your own modules
import { setupServer } from "msw/node";
import { http, HttpResponse } from "msw";

const server = setupServer(
  http.get("/api/users", () => HttpResponse.json([{ id: 1, name: "Ada" }]))
);
beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test("shows users, then an error when the API fails", async () => {
  render(<UsersPage />);
  expect(await screen.findByText("Ada")).toBeInTheDocument();

  server.use(http.get("/api/users", () => new HttpResponse(null, { status: 500 })));
  await userEvent.click(screen.getByRole("button", { name: /refresh/i }));
  expect(await screen.findByRole("alert")).toHaveTextContent(/something went wrong/i);
});
```

**🗣️ Say this:** "I weight toward integration tests because they survive refactors — mocking at the
network layer with MSW means I can rewrite the data layer without touching a test. I avoid mocking
my own modules; that's how you get a green suite over broken code. And I don't chase a coverage
number — I make sure critical paths and *error* paths are covered, since error paths are where
production actually breaks."

---

## Q21. How do you handle errors at an application level?

**📖 Definition:** Application-level error handling is a **layered containment strategy**. Error
boundaries at multiple granularities limit the **blast radius** of a render error so a failure
degrades one widget rather than the whole page. Because boundaries do not catch event-handler,
asynchronous, or resource-loading errors, these are covered separately with `try/catch` and global
`error` / `unhandledrejection` listeners. All errors are reported to a monitoring service with
context (component stack, release version, user breadcrumbs), and users see actionable plain
language, never a stack trace.

```jsx
// Layered strategy — each layer has an owner
<GlobalErrorBoundary>            {/* last resort full-page fallback + report */}
  <RouterProvider>
    <RouteErrorBoundary>         {/* one broken route doesn't kill the shell */}
      <Suspense fallback={<Skeleton />}>
        <WidgetErrorBoundary>    {/* one broken widget doesn't kill the page */}
          <RevenueChart />
        </WidgetErrorBoundary>
      </Suspense>
    </RouteErrorBoundary>
  </RouterProvider>
</GlobalErrorBoundary>
```

```jsx
// What boundaries DON'T catch — cover these separately
window.addEventListener("unhandledrejection", (e) => report(e.reason));
window.addEventListener("error", (e) => report(e.error));

const onSubmit = async () => {
  try { await save(); }
  catch (e) {
    report(e);
    toast.error("Couldn't save. Try again.");   // actionable, no stack trace
  }
};
```

**🗣️ Say this:** "I place boundaries at UI seams so failures degrade gracefully instead of
white-screening. Every boundary reports to Sentry with the component stack, release, and breadcrumb
trail. I track error rate per release so a bad deploy is visible in minutes, not in support
tickets."

---

## Q22. How do you keep bundle size under control?

**📖 Definition:** Bundle size management combines **analysis** (what is in the bundle and what is
actually used), **splitting** (deferring code to on-demand chunks), **tree shaking** (eliminating
unreachable exports, which requires ESM and side-effect-free modules), **dependency discipline**,
and — critically — an **automated budget in CI** that fails a pull request on regression. Without
the last one, manual cleanups regress within a quarter.

```bash
npx vite-bundle-visualizer
npx source-map-explorer 'dist/assets/*.js'
```

```jsx
// Route-level splitting — the biggest single win
const Admin = lazy(() => import("./routes/Admin"));

// Import only what you use
import debounce from "lodash-es/debounce";   // ✅ not: import _ from "lodash"
import { format } from "date-fns";           // ✅ modular

// Defer heavy, rarely-used dependencies
const openEditor = async () => {
  const { default: Monaco } = await import("@monaco-editor/react");  // 2MB, on demand
};

// Replace heavyweights
// moment (~230KB) → date-fns / Temporal / Intl
// lodash (~70KB)  → native methods
```

**🗣️ Say this:** "One-off cleanups regress, so the real answer is a CI budget that fails the PR on
regression. Beyond that: split at routes, audit before adding a dependency, and prefer platform
APIs — `Intl`, `structuredClone`, `URLSearchParams` — over a package."

---

## Q23. What are Core Web Vitals and how do you improve them in React?

**📖 Definition:** Core Web Vitals are Google's standardized **user-centric performance metrics**.
**LCP** (Largest Contentful Paint) measures loading — when the largest visible element renders.
**INP** (Interaction to Next Paint), which replaced FID in 2024, measures responsiveness — the
latency from a user interaction to the next visual update, across the whole interaction including
the React re-render. **CLS** (Cumulative Layout Shift) measures visual stability. They are assessed
on **field data** at the 75th percentile, not lab data.

| Metric | Measures | Good | React levers |
|---|---|---|---|
| **LCP** | Largest content paint | < 2.5s | SSR/SSG, preload the hero image, `fetchpriority="high"`, cut render-blocking JS |
| **INP** | Interaction → next paint | < 200ms | Break up long tasks, `useTransition`, virtualize, debounce, Web Worker |
| **CLS** | Layout shift | < 0.1 | Reserve space: width/height on images, skeletons sized like the content |

```jsx
import { onCLS, onINP, onLCP } from "web-vitals";
onLCP(m => analytics.send("lcp", m.value));
onINP(m => analytics.send("inp", m.value));
onCLS(m => analytics.send("cls", m.value));
```

**🗣️ Say this:** "INP is the one React apps typically fail, because it measures the *whole*
interaction including the render. That maps directly to long tasks from big re-render trees — so
the fix is re-render optimization plus yielding via transitions. And I trust field data over lab
data; Lighthouse on my laptop isn't the user on a mid-range Android."

---

## Q24. Micro-frontends — would you use them?

**📖 Definition:** Micro-frontends decompose a frontend into **independently developed, tested, and
deployed applications** composed at runtime (commonly via Module Federation) or build time. The
problem they solve is **organizational**: removing release coupling between teams. The costs are
technical: duplicated dependencies unless singletons are shared correctly, framework version skew,
harder end-to-end debugging and observability, and cross-app state, auth, and design-system
versioning becoming distributed-systems problems.

**🧸 ELI5:** Instead of one giant LEGO castle built by 50 kids fighting over one pile, each kid
builds their own room and you snap the rooms together.

```jsx
const RemoteCart = lazy(() => import("cart/CartWidget"));

<ErrorBoundary fallback={<CartUnavailable />}>
  <Suspense fallback={<CartSkeleton />}>
    <RemoteCart />
  </Suspense>
</ErrorBoundary>
```

**🗣️ Say this:** "Micro-frontends solve an *organizational* problem — independent deployment for
teams that would otherwise block each other — not a technical one. Under roughly 3–4 teams, a
well-structured monorepo with enforced module boundaries gives most of the benefit at a fraction of
the cost. I'd only reach for them when release coupling is measurably the bottleneck."

---

## Q25. How do you handle internationalization?

**📖 Definition:** Internationalization (i18n) is designing an application so it can be adapted to
different languages and regions **without code changes**; **localization** (l10n) is the adaptation
itself. It covers message externalization into translation catalogs, **ICU MessageFormat** for
plurals/gender/select, locale-aware formatting via the browser `Intl` APIs, **bidirectional text**
(RTL) via CSS logical properties, and locale-based code splitting so only the active locale ships.

```jsx
import { useTranslation } from "react-i18next";

function Cart({ count, total }) {
  const { t } = useTranslation();
  return (
    <>
      {/* plurals handled by the library, not by if/else */}
      <p>{t("cart.items", { count })}</p>
      <p>{new Intl.NumberFormat(locale, { style: "currency", currency: "JPY" }).format(total)}</p>
    </>
  );
}
```

**What people get wrong (name these):** concatenating strings (word order differs) · hardcoded
plural rules (Arabic has six forms, Japanese one) · forgetting RTL (needs
`margin-inline-start`, not `margin-left`) · text expansion (German ~30% longer breaks fixed-width
buttons) · manual date/number formatting instead of `Intl` · shipping every locale in the main
bundle.

**🗣️ Say this:** "I treat translation keys as an interface: no concatenation, full sentences with
interpolation, ICU format for plurals. Locale bundles load on demand, and the locale comes from the
URL so pages stay cacheable and shareable."

---

## Q26. How do you do feature flags in React?

**📖 Definition:** A feature flag (toggle) is a **runtime conditional that decouples deployment from
release**, letting code ship dark and be enabled per environment, cohort, or percentage without a
redeploy — and disabled instantly if it misbehaves. Flags must **default to off/safe** on failure to
resolve, and must evaluate **consistently between server and client** or they cause hydration
mismatches. Each flag is debt with an owner and a removal date; unbounded flags produce untestable
combinatorial states.

```jsx
const FlagContext = createContext({});

export function useFlag(name) {
  const flags = useContext(FlagContext);
  return flags[name] ?? false;      // safe default: OFF
}

function Checkout() {
  const newFlow = useFlag("new-checkout-flow");
  return newFlow ? <CheckoutV2 /> : <CheckoutV1 />;
}
```

**🗣️ Say this:** "Flags decouple deploy from release, which is what lets you ship incrementally and
roll back in seconds. The discipline part is treating every flag as debt with an owner and removal
date — otherwise you get exponential combinations nobody can test."

---

## Q27. How do you review React code?

**📖 Definition:** Code review is a **quality gate and a knowledge-transfer mechanism**. Effective
review separates **blocking correctness issues** from **non-blocking suggestions** explicitly, so
authors know what gates the merge. Recurring findings should be **promoted into automation** — a
lint rule, a type, or a shared abstraction — because review comments do not scale and tooling does.

```
CORRECTNESS
  □ Effect dependencies complete? Cleanup present?
  □ Stable keys — never index on a mutable list
  □ Race conditions on async (abort or ignore flag)
  □ Loading AND error states handled, not just the happy path
  □ State updated immutably

DESIGN
  □ Is this state, or is it derived?
  □ Could composition replace this prop/context?
  □ One responsibility? Is logic extractable to a hook?
  □ Does this belong in this feature, per our boundary rules?

USER-FACING
  □ Keyboard reachable, labeled, focus managed
  □ No layout shift; skeleton matches final size
  □ Error message is actionable and human

TESTS
  □ Queries by role/label, not test id or implementation
  □ Error path covered, not just the happy path
```

**🗣️ Say this:** "I separate blocking comments from suggestions explicitly. And I'd rather push a
repeated correction into a lint rule or shared hook than write the same comment a fifth time."

---

## Q28. System design: build a Twitter/X-style feed.

**📖 Definition:** A feed is an **infinite, dynamically-updating, ordered list of heterogeneous
items** loaded incrementally. The core design decisions are the **pagination model** (cursor-based,
not offset — offset duplicates and skips items when the underlying list mutates), the **rendering
strategy** (virtualization for constant DOM size), the **cache model** (normalized, so a like
updates every copy of that post), the **real-time strategy** (notify rather than auto-insert,
because injecting content above the viewport causes layout shift and mis-taps), and **failure
isolation** (per-item error boundaries).

**Requirements to clarify first (always — it's half the score):**
> How many posts? Real-time or polled? Media? Offline? Which platforms? SEO needed?

```
┌─ Route (RSC or SSR) — first page server-rendered for fast LCP + SEO
│
├─ Data layer: TanStack Query useInfiniteQuery
│   • cursor pagination (NOT offset)
│   • normalized cache so a like updates every copy of that post
│   • optimistic mutations for like/repost, with rollback
│
├─ Rendering
│   • virtualized list (dynamic heights + measured cache)
│   • IntersectionObserver sentinel to fetch the next page
│   • images: aspect-ratio box reserved → zero CLS; lazy below the fold
│
├─ Real-time
│   • WebSocket/SSE for a "12 new posts" banner — do NOT auto-prepend
│     (never move content under the user's thumb)
│
└─ Resilience
    • error boundary per post: one malformed post can't kill the feed
    • skeletons matching final dimensions
    • scroll restoration on back navigation
```

```jsx
const { data, fetchNextPage, hasNextPage, isFetchingNextPage } = useInfiniteQuery({
  queryKey: ["feed"],
  queryFn: ({ pageParam }) => fetchFeed({ cursor: pageParam, limit: 20 }),
  initialPageParam: null,
  getNextPageParam: (lastPage) => lastPage.nextCursor,
  staleTime: 30_000,
});
```

**Tradeoffs to state out loud:** cursor pagination is stable under inserts but can't "jump to page
5" · virtualization keeps DOM constant but breaks in-page find · optimistic likes feel instant but
need rollback · a "new posts" banner is less live but far better UX than auto-insertion.

**🗣️ Say this:** "I'd start by clarifying scale and real-time requirements, because they change the
whole design. The decisions I'd defend hardest are cursor pagination over offset, and a 'new posts'
banner instead of auto-prepending — both are correctness issues disguised as UX preferences."

---

## Q29. Why did React move from classes to hooks?

**📖 Definition:** Hooks were introduced in React 16.8 to address three structural limitations of
class components. **(1) No primitive for reusing stateful logic** — HOCs and render props worked but
produced deeply nested "wrapper hell". **(2) Lifecycle methods group code by *timing*, not by
*concern*** — a single feature is split across `componentDidMount`, `componentDidUpdate`, and
`componentWillUnmount`, while unrelated features share a method. **(3) `this` semantics** — binding,
and callbacks capturing the wrong props. Hooks also suit **concurrent rendering** better, since a
function component has no instance to keep consistent across interrupted renders.

```jsx
// Classes: one concern, three places
componentDidMount()    { this.subscribe(this.props.id); }
componentDidUpdate(p)  { if (p.id !== this.props.id) { this.unsub(); this.subscribe(this.props.id); } }
componentWillUnmount() { this.unsub(); }

// Hooks: one concern, one place
useEffect(() => {
  const sub = subscribe(id);
  return () => sub.unsubscribe();
}, [id]);
```

**🗣️ Say this:** "Hooks organize code by *concern* rather than by *lifecycle timing*, and they made
stateful logic composable as plain functions. The deeper reason is that they fit concurrent
rendering — a function component with no instance is much easier for React to call multiple times
or discard mid-render."

---

## Q30. Behavioral: tell me about a hard technical decision.

**📖 Definition:** Behavioral questions assess **decision-making process** under real constraints,
not outcomes. The **STAR** structure (Situation, Task, Action, Result) provides the frame; at senior
level the differentiator is the **Action** section explicitly naming the **alternatives considered
and what was traded away**, plus a **quantified Result** and a retrospective judgement.

```
SITUATION  — context + constraint (team size, deadline, scale, existing debt)
TASK       — what you specifically owned
ACTION     — options weighed, and WHY you picked one. Name what you gave up. ← scores highest
RESULT     — a number if you have one, plus what you'd do differently now
```

**Example skeleton to adapt:**
> "Our dashboard took 8s to interactive on mid-tier Android. Profiling showed 60% of the bundle was
> a charting library used on one route, and every keystroke in the global filter re-rendered 200
> rows through Context.
> I considered three options: memoize aggressively, swap the chart library, or restructure. I chose
> to split the chart route and move filter state out of Context into a selector-based store —
> memoization alone would have hidden the problem without fixing the architecture.
> TTI went to 2.1s and INP from 340ms to 90ms. The tradeoff was a two-week migration and a new
> dependency the team had to learn. In hindsight I'd have added the CI bundle budget first, because
> the regression took a year to become visible."

**Prepare 3 stories:** a performance win · a bad decision you owned and corrected · a disagreement
you resolved with a colleague.

---

## Q31. Rapid-fire senior questions

**Q: Why must render be pure?**
Concurrent React may call it multiple times, pause it, or discard the result. Side effects would
fire unpredictably.

**Q: What does `flushSync` do and why avoid it?**
Forces a synchronous re-render and DOM commit, opting out of batching and concurrency. Use only
when you must read layout immediately after an update.

**Q: `key` on a component — what's the trick?**
Changing it unmounts and remounts, resetting all state. The cleanest way to reset a form when the
selected entity changes.

**Q: Why is `useEffect` the wrong place for most data fetching now?**
Waterfalls (a child can't start until the parent renders), no cache, no dedup, manual race handling,
and it runs after paint. Frameworks and query libraries fetch earlier and share results.

**Q: What is tearing?**
Two components rendering different values of the same external store within one commit — possible
under concurrent rendering. Solved by `useSyncExternalStore`.

**Q: When do you reach for a Web Worker?**
Any single main-thread task over ~50ms that isn't DOM work: parsing large JSON, crypto, image
processing, heavy diffing.

**Q: What are Server Actions?**
Functions marked `"use server"` that a Client Component can call directly; React handles the RPC.
They remove hand-written API routes for mutations — but they're public endpoints, so they need
explicit auth checks.

**Q: How does React 19's `use()` differ from other hooks?**
It can be called conditionally and inside loops. It reads a promise or context and integrates with
Suspense.

**Q: What's the cost of Context for high-frequency updates?**
Every consumer re-renders on every value change, with no selector granularity. For anything changing
per keystroke or per frame, use a store with selectors.

---

## ✅ Senior checklist

- [ ] I can define Fiber's two phases and say why render must be pure
- [ ] I can distinguish `useTransition` from `useDeferredValue` and when each applies
- [ ] I can explain RSC boundaries, serialization limits, and the bundle payoff
- [ ] I can name the cause of hydration mismatches and three fixes
- [ ] I can defend a state architecture, splitting server / URL / client / local
- [ ] I have a repeatable performance methodology — measure, classify, fix one thing
- [ ] I can say when NOT to memoize and why
- [ ] I can design a feed / chat / dashboard and name tradeoffs unprompted
- [ ] I have 3 STAR stories, each with a number and a "what I'd change"

---

## 🎯 The senior meta-skill

For **every** answer, land these three beats:

1. **What** it is — the definition (30 seconds — show you know it)
2. **When** you'd use it (show judgment)
3. **What it costs** (show seniority) ← almost nobody does this, and it's what gets you hired

> "X solves Y, at the cost of Z. I'd use it when Y matters more than Z."

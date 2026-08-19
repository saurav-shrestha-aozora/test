# 🟢 JUNIOR React Interview — Questions & Answers (Explained Like You're 5)

> **Every question has 4 parts:**
> **📖 Definition** — the formal, textbook answer. Open with this.
> **🧸 ELI5** — the analogy, so you actually understand it (and can fall back on it).
> **Code** — a real example.
> **🗣️ Say this** — the exact sentence to speak in the room.

---

## Q1. What is React?

**📖 Definition:** React is an open-source JavaScript **library** (not a framework) created
and maintained by Meta, used for building user interfaces. It is **declarative** (you
describe the desired UI for a given state, not the steps to get there), **component-based**
(UIs are composed from independent, reusable pieces), and **unopinionated** about routing,
data fetching, and styling. React maintains an in-memory representation of the UI and
synchronizes it with the host environment — the browser DOM via `react-dom`, native views
via React Native.

**🧸 ELI5:** Imagine a big LEGO castle. When one window breaks, you don't rebuild the whole
castle — you swap out that one brick. React builds websites out of small reusable LEGO
bricks called **components**, and only swaps the bricks that changed.

```jsx
function Welcome() {
  return <h1>Hello!</h1>;
}
```

**🗣️ Say this:** "React is a JavaScript library for building user interfaces from reusable
components. It's declarative — I describe *what* the screen should look like for a given
state, and React figures out *how* to update the DOM."

---

## Q2. What is JSX?

**📖 Definition:** JSX (JavaScript XML) is a **syntax extension** to JavaScript that allows
writing HTML-like markup inside JavaScript files. It is not valid JavaScript — a compiler
(Babel, SWC, or esbuild) transforms each JSX element into a `React.createElement()` call (or,
with the modern JSX transform, a `_jsx()` call from `react/jsx-runtime`), which returns a
plain JavaScript object called a **React element**.

**🧸 ELI5:** JSX is HTML wearing a JavaScript costume. It *looks* like HTML, but it lives
inside your JavaScript file.

```jsx
const name = "Bikrant";
const element = <h1>Hello, {name}!</h1>;   // { } means "put JavaScript here"
```

Compiles to:

```js
const element = React.createElement("h1", null, "Hello, ", name, "!");
```

**⚠️ Gotchas:**
- `class` → `className` (`class` is a reserved JS word)
- `for` → `htmlFor`
- Every tag must close: `<img />`, `<br />`
- A component can only return **one** parent element

**🗣️ Say this:** "JSX is syntactic sugar for `React.createElement`. It lets me write markup
inside JavaScript, and the build step compiles it down to function calls that produce plain
objects."

---

## Q3. What is a component?

**📖 Definition:** A component is an **independent, reusable, self-contained unit of UI**. In
modern React it is a JavaScript function that accepts a single object argument (`props`) and
returns a React element describing what should appear on screen. Components can be composed
— a component's output may include other components — forming a tree. Component names must
be capitalized so JSX distinguishes them from built-in HTML elements.

**🧸 ELI5:** A component is a **cookie cutter**. Make the shape once, then stamp out as many
cookies as you want — each with different sprinkles (props).

```jsx
// The cookie cutter
function Cookie({ flavor }) {
  return <div className="cookie">{flavor} cookie</div>;
}

// Stamping cookies
<Cookie flavor="Chocolate" />
<Cookie flavor="Vanilla" />
```

**Rule:** `<cookie />` is treated as an HTML tag; `<Cookie />` is your component.

**🗣️ Say this:** "A component is a reusable, self-contained piece of UI — just a function
that takes props and returns JSX."

---

## Q4. Function component vs Class component?

**📖 Definition:** A **class component** is an ES6 class extending `React.Component` that
implements a `render()` method, holds state in `this.state`, and hooks into lifecycle methods
such as `componentDidMount`. A **function component** is a plain function returning JSX; since
React 16.8 it accesses state and lifecycle behavior through **hooks**. Function components are
the officially recommended form; class components remain supported but receive no new features.

**🧸 ELI5:** Class components are the old flip phone. Function components are the smartphone.
Both make calls — nobody buys flip phones anymore.

```jsx
// ❌ OLD (class)
class Hello extends React.Component {
  render() {
    return <h1>Hi {this.props.name}</h1>;
  }
}

// ✅ MODERN (function)
function Hello({ name }) {
  return <h1>Hi {name}</h1>;
}
```

**🗣️ Say this:** "Function components with hooks are the modern standard — shorter, easier to
test, no `this` binding problems, and they let me extract stateful logic into custom hooks,
which classes never did cleanly."

---

## Q5. What are props?

**📖 Definition:** Props (short for "properties") are the **read-only inputs** passed from a
parent component to a child component. React collects all JSX attributes into a single object
and passes it as the component's first argument. Props are **immutable from the child's
perspective** — a component must never modify its own props. This constraint is what makes
components behave like pure functions of their inputs.

**🧸 ELI5:** Props are the **lunchbox your mom packed**. She decides what's inside. You can eat
it, but you cannot change what's in it and hand it back. Props flow **down**, and they're
**read-only**.

```jsx
function Child({ name, age }) {
  // props.name = "changed";  ❌ NEVER
  return <p>{name} is {age} years old</p>;
}

function Parent() {
  return <Child name="Bikrant" age={25} />;
}
```

**Passing everything at once:**
```jsx
const user = { name: "Bikrant", age: 25 };
<Child {...user} />   // spread
```

**🗣️ Say this:** "Props are read-only inputs passed from parent to child. React has one-way
data flow — data goes down, events come back up through callbacks."

---

## Q6. What is state?

**📖 Definition:** State is **data owned and managed internally by a component** that can change
over time and, when changed, causes React to re-render that component. Unlike props, state is
private to the component that declares it. State is declared with the `useState` hook, which
returns a tuple: the current value for this render, and a setter function that schedules an
update.

**🧸 ELI5:** State is **your own backpack**. What's in it is yours and you can change it whenever
you want. When you change it, React redraws your component so the screen matches your backpack.

```jsx
import { useState } from "react";

function Counter() {
  const [count, setCount] = useState(0);
  //      ↑value   ↑updater      ↑initial value

  return (
    <div>
      <p>You clicked {count} times</p>
      <button onClick={() => setCount(count + 1)}>Click me</button>
    </div>
  );
}
```

**🗣️ Say this:** "State is data owned and managed by the component. Updating it through the
setter tells React to re-render."

---

## Q7. Props vs State — the difference?

**📖 Definition:** Both are plain JavaScript objects holding data that influences render output.
The difference is **ownership and mutability**: props are passed *in* from a parent and are
immutable within the receiving component; state is declared *inside* the component and is
mutable via its setter. A parent's state, when passed down, becomes the child's props.

**🧸 ELI5:** Props = the lunchbox **someone gave** you (can't change it).
State = **your backpack** (you can change it).

| | Props | State |
|---|---|---|
| Who owns it | The parent | The component itself |
| Can you change it | ❌ Read-only | ✅ Yes, via setter |
| Triggers re-render | Yes, when the parent changes it | Yes, when you set it |
| Comes from | Outside | Inside |

```jsx
function Parent() {
  const [theme, setTheme] = useState("dark");  // STATE here...
  return <Child theme={theme} />;              // ...becomes PROPS there
}
```

**🗣️ Say this:** "Same data, different perspective — one component's state becomes its child's
props."

---

## Q8. What is `useState`?

**📖 Definition:** `useState` is a React hook that adds local state to a function component. It
takes an initial value (or a lazy initializer function) and returns an array of exactly two
elements: the current state value for this render, and a **setter function** that schedules a
re-render with a new value. State updates must be **immutable** — you provide a new value rather
than mutating the existing one — because React uses `Object.is` reference comparison to detect
changes.

**🧸 ELI5:** A **magic box** with two things inside: what it's holding, and a remote control to
change what it's holding.

```jsx
const [value, setValue] = useState(initialValue);
```

**Different types:**
```jsx
const [name, setName]     = useState("");                 // string
const [age, setAge]       = useState(0);                  // number
const [isOpen, setIsOpen] = useState(false);              // boolean
const [items, setItems]   = useState([]);                 // array
const [user, setUser]     = useState({ name: "", email: "" }); // object
```

**⚠️ Objects & arrays — always make a COPY:**
```jsx
// ❌ WRONG — mutating (React sees the same object → no re-render)
user.name = "Bikrant";
setUser(user);

// ✅ RIGHT — new object
setUser({ ...user, name: "Bikrant" });

// ✅ Arrays
setItems([...items, newItem]);                                    // add
setItems(items.filter(i => i.id !== id));                         // remove
setItems(items.map(i => i.id === id ? { ...i, done: true } : i)); // update
```

**🗣️ Say this:** "`useState` returns the current value and a setter. State must be updated
immutably because React compares by reference to decide whether to re-render."

---

## Q9. Why can't I change state directly?

**📖 Definition:** React determines whether to re-render by performing a **shallow reference
comparison** (`Object.is`) between the previous and next state. Mutating an object or array in
place leaves the reference unchanged, so React concludes nothing changed and **bails out of
re-rendering**. Immutable updates also preserve the previous state as a distinct value, which is
what makes features like `React.memo`, `useMemo` dependency checks, and time-travel debugging
possible.

**🧸 ELI5:** You show your friend a drawing, then erase and redraw on the **same paper**. Your
friend glances over and says "looks the same to me" and doesn't look again. React checks whether
it's a **new paper**, not what's drawn on it.

```jsx
// ❌ React doesn't notice
count = count + 1;
items.push("new");

// ✅ React notices — brand new value / new reference
setCount(count + 1);
setItems([...items, "new"]);
```

**🗣️ Say this:** "React uses `Object.is` reference comparison. Mutating keeps the same
reference, so React bails out of re-rendering. Always create a new value."

---

## Q10. Why does state look "one step behind"?

**📖 Definition:** State updates in React are **asynchronous and batched**. Calling a setter does
not immediately change the state variable; it **schedules** a re-render. Within a single render,
the state variable is a **constant snapshot** captured by that render's closure — it never
changes mid-render. Multiple setter calls in the same event are batched into one re-render. When
the next value depends on the previous one, use the **functional updater** form, which receives
the latest pending state.

**🧸 ELI5:** You ask mom for candy. She says "okay, after dinner." Ask right now — do you have
candy? No, not yet. State updates are **scheduled**, not instant.

```jsx
const handleClick = () => {
  setCount(count + 1);
  console.log(count);  // 😱 still the OLD value
};
```

**The classic trap — this adds 1, not 3:**
```jsx
setCount(count + 1);  // count is 0 → schedules 1
setCount(count + 1);  // count is STILL 0 → schedules 1
setCount(count + 1);  // count is STILL 0 → schedules 1
// Result: 1
```

**✅ Fix — functional update:**
```jsx
setCount(prev => prev + 1);  // 0 → 1
setCount(prev => prev + 1);  // 1 → 2
setCount(prev => prev + 1);  // 2 → 3
// Result: 3 ✅
```

**🗣️ Say this:** "State updates are asynchronous and batched. The variable is a snapshot of that
render. When the new value depends on the old one, I use the functional updater."

---

## Q11. What is the Virtual DOM?

**📖 Definition:** The Virtual DOM is an **in-memory tree of plain JavaScript objects** that
represents the desired UI structure. When state changes, React builds a new virtual tree and
compares it with the previous one — a process called **reconciliation** — then computes and
applies the **minimal set of mutations** needed to bring the real DOM into agreement. The
benefit is not raw speed but that it lets you write declarative code while React handles
efficient imperative DOM updates.

**🧸 ELI5:** Before rearranging your bedroom, you sketch the new layout, compare it with a sketch
of the old layout, then only move the furniture that actually changed.

```
1. State changes
2. React builds a NEW virtual tree (plain JS objects — cheap)
3. React DIFFS it against the OLD virtual tree
4. React updates ONLY the changed real DOM nodes  ← the slow part, minimized
```

**🗣️ Say this:** "The Virtual DOM is a lightweight JS representation of the UI. Real DOM
mutations are expensive, so React diffs two virtual trees and applies the minimal set of
changes. That process is called reconciliation."

---

## Q12. What is `useEffect`?

**📖 Definition:** `useEffect` is a hook for performing **side effects** — operations that reach
outside the React rendering process, such as network requests, subscriptions, timers, logging,
or direct DOM manipulation. The effect function runs **after** the component has rendered and the
browser has painted. It may return a **cleanup function**, which React calls before the next
run of the effect and when the component unmounts. Its second argument, the dependency array,
controls when it re-runs.

**🧸 ELI5:** A note that says **"after I finish drawing, do this"** — hang the picture on the
wall, or call grandma.

```jsx
import { useEffect, useState } from "react";

function Timer() {
  const [seconds, setSeconds] = useState(0);

  useEffect(() => {
    const id = setInterval(() => setSeconds(s => s + 1), 1000);

    return () => clearInterval(id);   // 🧹 CLEANUP
  }, []);                             // 👈 dependency array

  return <p>{seconds}s</p>;
}
```

**🗣️ Say this:** "`useEffect` runs side effects after render — anything reaching outside React,
like network calls or subscriptions. The returned function is the cleanup."

---

## Q13. What does the dependency array do?

**📖 Definition:** The dependency array is the second argument to `useEffect` (and to `useMemo`
/ `useCallback`). React compares each dependency against its value from the previous render using
`Object.is`; if any has changed, the effect re-runs. **Omitting** the array means the effect runs
after every render. An **empty** array means it runs only on mount (and cleans up on unmount).
Every reactive value used inside the effect — props, state, or derived values — should be listed.

**🧸 ELI5:** It's a **list of things to watch**. "Only redo this when one of these changes."

```jsx
useEffect(() => { ... });            // no array → after EVERY render 😱
useEffect(() => { ... }, []);        // empty   → ONCE on mount ✅
useEffect(() => { ... }, [userId]);  // mount + whenever userId changes ✅
```

**Real example:**
```jsx
function Profile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(r => r.json())
      .then(setUser);
  }, [userId]);          // 👈 new userId = new fetch

  if (!user) return <p>Loading...</p>;
  return <h1>{user.name}</h1>;
}
```

**⚠️ Infinite loop trap:**
```jsx
// render → effect → setState → render → effect → 💥
useEffect(() => { setCount(count + 1); });
```

**🗣️ Say this:** "The dependency array controls when the effect re-runs. Missing deps cause
stale closures; unstable deps cause infinite loops."

---

## Q14. What is cleanup in `useEffect`?

**📖 Definition:** The cleanup function is the **function returned by an effect**. React invokes
it (a) immediately before re-running the effect due to changed dependencies, and (b) when the
component unmounts. Its purpose is to **undo** what the effect set up — removing event listeners,
clearing timers, closing sockets, aborting requests — so that resources are released and no two
copies of a subscription exist simultaneously.

**🧸 ELI5:** If you turn on the tap, **turn it off** before leaving the room — otherwise the
house floods.

```jsx
useEffect(() => {
  const onResize = () => console.log(window.innerWidth);
  window.addEventListener("resize", onResize);

  return () => window.removeEventListener("resize", onResize);  // 🧹
}, []);
```

**Needs cleanup:** `setInterval` / `setTimeout`, event listeners, WebSockets, subscriptions,
in-flight fetches.

**🗣️ Say this:** "Cleanup runs before the effect re-runs and on unmount. Without it you leak
memory and get 'setState on an unmounted component' problems."

---

## Q15. Why do lists need a `key`?

**📖 Definition:** A `key` is a **special string attribute** that gives each element in a list a
**stable identity across renders**. During reconciliation React uses keys to match elements in
the previous tree with elements in the next tree, so it can determine which items were added,
removed, moved, or updated — rather than re-creating the whole list. Keys must be **unique among
siblings** and **stable over time**. Using the array index as a key breaks this guarantee whenever
the list is reordered, filtered, or has items inserted or removed.

**🧸 ELI5:** Kids in a line, all wearing the same shirt. If one leaves, you can't tell who moved.
Give each a **name tag** — now you always know who's who.

```jsx
// ❌ No key — React warns, rendering can go wrong
{todos.map(todo => <li>{todo.text}</li>)}

// ❌ Index as key — breaks on reorder / delete / insert
{todos.map((todo, i) => <li key={i}>{todo.text}</li>)}

// ✅ Stable unique ID
{todos.map(todo => <li key={todo.id}>{todo.text}</li>)}
```

**Why index keys are a real bug:**
```
List: [🍎 Apple, 🍌 Banana]   keys 0, 1
Delete Apple → [🍌 Banana]    key 0
React thinks: "item 0 just changed its text from Apple → Banana"
If each row had an input, the typed text stays on the WRONG row 🐛
```

**🗣️ Say this:** "Keys give React a stable identity per item during reconciliation. Index keys
break on reorder, filter, or insert — component state ends up attached to the wrong item."

---

## Q16. How do you render a list?

**📖 Definition:** Lists are rendered by **transforming an array of data into an array of React
elements**, conventionally with `Array.prototype.map()`. React accepts arrays of elements as
valid children and renders them in order. Each element in the array requires a unique `key`.

**🧸 ELI5:** One cookie cutter, one row of dough — stamp one cookie per piece of data.

```jsx
function TodoList({ todos }) {
  if (todos.length === 0) return <p>Nothing to do! 🎉</p>;

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          {todo.done ? "✅" : "⬜"} {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

**🗣️ Say this:** "`.map()` turns an array of data into an array of elements — each one needs a
stable key."

---

## Q17. How do you do conditional rendering?

**📖 Definition:** Conditional rendering is the practice of **returning different React elements
based on runtime conditions**, using ordinary JavaScript control flow. React treats `null`,
`undefined`, `false`, and `true` as "render nothing", which makes short-circuit expressions
usable directly in JSX. There is no special template syntax — it's just JavaScript.

**🧸 ELI5:** "If it's raining show the umbrella, otherwise show the sunglasses."

```jsx
// 1️⃣ Ternary — one of two
{isLoggedIn ? <Dashboard /> : <Login />}

// 2️⃣ && — show or show nothing
{hasError && <p className="error">Something broke!</p>}

// 3️⃣ Early return — cleanest guard
function Profile({ user }) {
  if (!user) return <Spinner />;
  return <h1>{user.name}</h1>;
}

// 4️⃣ Variable — many branches
let content;
if (status === "loading")    content = <Spinner />;
else if (status === "error") content = <ErrorBox />;
else                         content = <Data />;
return <div>{content}</div>;
```

**⚠️ The `&&` number trap:** `0` is falsy but is **not** ignored by React — it renders as text.
```jsx
{items.length && <List />}      // ❌ length 0 renders a literal "0"
{items.length > 0 && <List />}  // ✅ force a real boolean
```

**🗣️ Say this:** "Ternary for either/or, `&&` for optional UI, early return for guards. I make
sure `&&` gets a real boolean so `0` doesn't leak into the output."

---

## Q18. How do you handle events?

**📖 Definition:** React attaches event handlers **declaratively via camelCase JSX props**
(`onClick`, `onChange`, `onSubmit`) rather than through `addEventListener`. You pass a
**function reference**, and React invokes it with a SyntheticEvent when the event fires.
Internally React uses **event delegation**: a single listener per event type on the root
container dispatches to the correct handler, rather than one listener per DOM node.

**🧸 ELI5:** "When someone presses this button, run this instruction."

```jsx
function Button() {
  const handleClick = (e) => {
    e.preventDefault();
    console.log("clicked!");
  };

  return <button onClick={handleClick}>Click</button>;
}
```

**⚠️ The single biggest beginner bug:**
```jsx
<button onClick={handleClick}>    // ✅ pass the function
<button onClick={handleClick()}>  // ❌ CALLS it during render
```

**Passing arguments:**
```jsx
<button onClick={() => handleDelete(todo.id)}>Delete</button>
```

**🗣️ Say this:** "React uses camelCase event props and wraps native events in a SyntheticEvent
for cross-browser consistency. I pass a function reference, not a call — unless I need an
argument, then I wrap it in an arrow function."

---

## Q19. What is a Synthetic Event?

**📖 Definition:** A SyntheticEvent is React's **cross-browser wrapper around the native browser
event**. It exposes the same interface as the native event (`preventDefault`, `stopPropagation`,
`target`, `currentTarget`) but normalizes behavior across browsers, with the underlying native
event available at `e.nativeEvent`. Since React 17, React attaches its delegated listeners to
the **root container** rather than to `document`, which allows multiple React versions to
coexist on one page.

**🧸 ELI5:** Browsers speak slightly different languages. React hires a **translator** so every
browser sounds the same to you.

```jsx
const handleClick = (e) => {
  e.preventDefault();
  e.stopPropagation();
  console.log(e.target.value);
  console.log(e.nativeEvent);   // the real browser event if you need it
};
```

**🗣️ Say this:** "SyntheticEvent is React's cross-browser wrapper around native events. Since
React 17 the listeners attach to the root container rather than `document`."

---

## Q20. Controlled vs Uncontrolled inputs?

**📖 Definition:** A **controlled component** is a form input whose displayed value is driven by
React state — it receives `value` and an `onChange` handler, making React the **single source of
truth**. An **uncontrolled component** lets the DOM node itself hold the value; React reads it
imperatively via a `ref` when needed, and initial content is set with `defaultValue` /
`defaultChecked`. File inputs (`<input type="file">`) are always uncontrolled because their value
is read-only for security reasons.

**🧸 ELI5:** **Controlled** = you hold the puppet's strings, React decides what it shows.
**Uncontrolled** = the puppet moves on its own and you peek at it when you need to.

```jsx
// ✅ CONTROLLED — React state is the single source of truth
function ControlledForm() {
  const [email, setEmail] = useState("");
  return <input value={email} onChange={(e) => setEmail(e.target.value)} />;
}

// UNCONTROLLED — the DOM owns the value, you read it with a ref
function UncontrolledForm() {
  const inputRef = useRef();
  const submit = () => console.log(inputRef.current.value);

  return (
    <>
      <input ref={inputRef} defaultValue="hi" />
      <button onClick={submit}>Send</button>
    </>
  );
}
```

**🗣️ Say this:** "Controlled inputs keep the value in React state — needed for live validation,
formatting, or conditionally disabling submit. Uncontrolled is lighter for simple forms, and file
inputs are always uncontrolled."

---

## Q21. Build a full form (very common live-coding task)

**📖 Definition:** A React form is a controlled component pattern in which form field values live
in state, changes are captured by `onChange` handlers, and submission is intercepted with
`onSubmit` plus `event.preventDefault()` to stop the browser's default full-page reload. Multiple
fields are commonly managed in a single state object using **computed property keys** driven by
each input's `name` attribute.

```jsx
function SignupForm() {
  const [form, setForm] = useState({ name: "", email: "", password: "" });
  const [errors, setErrors] = useState({});

  // ONE handler for every field — uses the input's name attribute
  const handleChange = (e) => {
    const { name, value } = e.target;
    setForm(prev => ({ ...prev, [name]: value }));   // 👈 computed key
  };

  const validate = () => {
    const err = {};
    if (!form.name) err.name = "Name is required";
    if (!form.email.includes("@")) err.email = "Invalid email";
    if (form.password.length < 6) err.password = "Min 6 characters";
    return err;
  };

  const handleSubmit = (e) => {
    e.preventDefault();                 // stop the page reload
    const err = validate();
    setErrors(err);
    if (Object.keys(err).length === 0) console.log("Submit!", form);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" value={form.name} onChange={handleChange} />
      {errors.name && <span>{errors.name}</span>}

      <input name="email" value={form.email} onChange={handleChange} />
      {errors.email && <span>{errors.email}</span>}

      <input name="password" type="password" value={form.password} onChange={handleChange} />
      {errors.password && <span>{errors.password}</span>}

      <button type="submit">Sign up</button>
    </form>
  );
}
```

**🗣️ Say this:** "One state object plus the `name` attribute with a computed key lets a single
handler cover every field."

---

## Q22. What is "lifting state up"?

**📖 Definition:** Lifting state up is the pattern of **moving shared state to the closest common
ancestor** of the components that need it. The ancestor owns the state and passes the value down
as props, along with callback functions that let descendants request changes. This preserves a
single source of truth and keeps sibling components synchronized, since React's data flow is
unidirectional.

**🧸 ELI5:** Two kids both want to know the score. If each keeps their own score they'll disagree.
So **mom** keeps the score and tells both.

```jsx
function Parent() {
  const [count, setCount] = useState(0);   // 👈 lifted here

  return (
    <>
      <Display count={count} />
      <Controls onIncrement={() => setCount(c => c + 1)} />
    </>
  );
}

function Display({ count })        { return <h1>{count}</h1>; }
function Controls({ onIncrement }) { return <button onClick={onIncrement}>+1</button>; }
```

**🗣️ Say this:** "When two components need the same data, I move that state to their closest
common ancestor and pass the value down plus a callback up."

---

## Q23. What is prop drilling?

**📖 Definition:** Prop drilling is the situation where a prop is passed through **multiple
intermediate components that do not use it themselves**, purely to reach a deeply nested
descendant. It couples unrelated components to data they don't care about, makes refactoring
harder, and causes intermediate components to re-render unnecessarily. Common remedies are the
Context API, component composition via `children`, and external state libraries.

**🧸 ELI5:** Passing a note from the front row to the back row — every kid in between has to hold
it even though the note isn't for them.

```jsx
<App user={user}>
  <Layout user={user}>
    <Sidebar user={user}>
      <Avatar user={user} />   {/* finally used here */}
```

**🗣️ Say this:** "Prop drilling is threading props through components that don't use them. I
solve it with Context for genuinely global data, or with composition when the tree is shallow —
Context has re-render costs, so it isn't my first reach."

---

## Q24. What is the `children` prop?

**📖 Definition:** `children` is a **special prop** automatically populated by React with whatever
JSX is nested between a component's opening and closing tags. It enables **composition**: a
component can define structure and behavior while leaving its inner content to the caller,
without knowing anything about that content in advance.

**🧸 ELI5:** A **gift box**. The box doesn't care what's inside — put anything in it.

```jsx
function Card({ title, children }) {
  return (
    <div className="card">
      <h2>{title}</h2>
      <div className="body">{children}</div>   {/* 👈 whatever you nest */}
    </div>
  );
}

<Card title="Profile">
  <p>Anything can go here</p>
  <button>Even components</button>
</Card>
```

**🗣️ Say this:** "`children` is whatever's nested inside a component's tags. It's the core of
composition in React and often removes the need for Context entirely."

---

## Q25. What is a Fragment?

**📖 Definition:** A Fragment is a built-in React component that lets a component **return
multiple children without adding an extra node to the DOM**. It exists because a component must
return a single root element. Written either as `<React.Fragment>` or with the shorthand `<>…</>`
— the shorthand cannot accept props, so the long form is required when a `key` is needed.

**🧸 ELI5:** An **invisible bag**. It holds your stuff together, but nobody sees the bag.

```jsx
// ❌ Can't return two siblings
return <h1>Hi</h1><p>Bye</p>;

// 😐 Works but adds a useless div
return <div><h1>Hi</h1><p>Bye</p></div>;

// ✅ Fragment — no extra DOM node
return <><h1>Hi</h1><p>Bye</p></>;

// ✅ Long form — required when you need a key
{items.map(i => (
  <React.Fragment key={i.id}>
    <dt>{i.term}</dt>
    <dd>{i.desc}</dd>
  </React.Fragment>
))}
```

**🗣️ Say this:** "Fragments group children without adding a DOM node — important for valid HTML
inside tables and for not breaking flex/grid layouts."

---

## Q26. What are the Rules of Hooks?

**📖 Definition:** Two constraints govern hook usage. **(1) Only call hooks at the top level** —
never inside conditions, loops, or nested functions. **(2) Only call hooks from React function
components or from other custom hooks** — never from plain JavaScript functions or class
components. These rules exist because React associates hook state with a component instance **by
call order**, storing hooks in an ordered linked list on the fiber; if the order varies between
renders, React would return the wrong state for a given hook.

**🧸 ELI5:** Hooks are a **numbered checklist** React reads top to bottom, in the same order every
time. Skip a line one day and React loses track of whose stuff is whose.

**Rule 1 — top level only:**
```jsx
// ❌ WRONG — hook order changes between renders 💥
if (isLoggedIn) {
  const [name, setName] = useState("");
}

// ✅ RIGHT
const [name, setName] = useState("");
if (isLoggedIn) { /* use name here */ }
```

**Rule 2 — React functions only:**
```jsx
// ❌ WRONG
function normalFunction() { useState(0); }

// ✅ RIGHT
function useMyHook() { const [x, setX] = useState(0); return x; }
```

**🗣️ Say this:** "React tracks hooks by call order in a linked list per component, so the order
must be identical on every render. That's exactly why they can't be conditional."

---

## Q27. What is `useRef` (junior level)?

**📖 Definition:** `useRef` returns a **mutable object with a single `.current` property** that
persists for the full lifetime of the component. Unlike state, **mutating `.current` does not
trigger a re-render**, and the value is not part of the render output. It has two primary uses:
holding a reference to a DOM node (via the `ref` attribute) and storing any mutable value that
should survive re-renders without causing them.

**🧸 ELI5:** A **sticky note on the fridge**. You can write on it and read it, but changing it
does NOT redecorate the house.

**Use 1 — grab a DOM element:**
```jsx
function AutoFocus() {
  const inputRef = useRef(null);

  useEffect(() => {
    inputRef.current.focus();   // 👈 the real DOM node
  }, []);

  return <input ref={inputRef} />;
}
```

**Use 2 — remember a value without re-rendering:**
```jsx
const renderCount = useRef(0);
renderCount.current++;          // does NOT trigger a re-render
```

**🗣️ Say this:** "`useRef` is a mutable `.current` box that survives re-renders and never
triggers one. I use it for DOM access and for values I need to remember but not display."

---

## Q28. How do you fetch data?

**📖 Definition:** In a client-side React component, data fetching is a **side effect** and
therefore belongs in `useEffect` (or, preferably in production, a data-fetching library). A
correct implementation tracks three UI states — **loading, error, and success** — and cleans up
in the effect's return function to prevent stale responses from overwriting newer data or being
applied after unmount.

```jsx
function Users() {
  const [users, setUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;               // 👈 guards against race conditions

    async function load() {
      try {
        setLoading(true);
        const res = await fetch("/api/users");
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        const data = await res.json();
        if (!cancelled) setUsers(data);
      } catch (e) {
        if (!cancelled) setError(e.message);
      } finally {
        if (!cancelled) setLoading(false);
      }
    }

    load();
    return () => { cancelled = true; };  // 🧹
  }, []);

  if (loading) return <p>Loading...</p>;
  if (error)   return <p>Error: {error}</p>;

  return <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>;
}
```

**🗣️ Say this:** "Always handle three states — loading, error, success. The `cancelled` flag
stops an old response from overwriting a newer one and avoids setting state after unmount."

---

## Q29. Why does my `useEffect` run twice?

**📖 Definition:** In **development only**, when the app is wrapped in `<React.StrictMode>`,
React 18+ intentionally **mounts, unmounts, and remounts** each component, running every effect
twice with its cleanup in between. This is a deliberate diagnostic that surfaces effects which
are not resilient to being set up and torn down repeatedly — i.e. effects with missing or
incorrect cleanup. It does **not** occur in production builds.

**🧸 ELI5:** React runs a deliberate **fire drill** to check your cleanup actually works.

```jsx
<React.StrictMode>
  <App />
</React.StrictMode>
```

**🗣️ Say this:** "StrictMode double-invokes effects in development only, to surface missing
cleanup. It doesn't happen in production. If double-firing breaks my code, my effect isn't
cleaned up properly — that's my bug, not React's."

---

## Q30. What is the component lifecycle with hooks?

**📖 Definition:** The component lifecycle is the sequence of phases a component passes through:
**mounting** (first inserted into the tree), **updating** (re-rendered due to state/prop
changes), and **unmounting** (removed from the tree). Class components expose these through
dedicated methods; function components express the same phases through `useEffect` and its
dependency array plus cleanup function.

**🧸 ELI5:** Born 👶 → grows up 🧒 → says goodbye 👋

```jsx
useEffect(() => {
  console.log("👶 MOUNT");
  return () => console.log("👋 UNMOUNT");
}, []);

useEffect(() => {
  console.log("🧒 UPDATE — count changed");
}, [count]);
```

| Class method | Hook equivalent |
|---|---|
| `componentDidMount` | `useEffect(fn, [])` |
| `componentDidUpdate` | `useEffect(fn, [dep])` |
| `componentWillUnmount` | the function returned from `useEffect` |

---

## Q31. How do you style React components?

**📖 Definition:** React has no built-in styling system. Styles are applied through the
`className` prop (referencing ordinary CSS), the `style` prop (an object with camelCased CSS
properties, applied inline), or third-party approaches: **CSS Modules** (build-time scoped class
names), **CSS-in-JS** libraries, and **utility-first frameworks** such as Tailwind.

```jsx
// 1️⃣ CSS class
import "./Button.css";
<button className="btn btn-primary">Click</button>

// 2️⃣ Inline style — object, camelCase keys
<div style={{ backgroundColor: "red", fontSize: 16 }}>Hi</div>

// 3️⃣ Conditional class
<button className={`btn ${isActive ? "active" : ""}`}>Click</button>

// 4️⃣ CSS Modules — scoped, no name collisions
import styles from "./Button.module.css";
<button className={styles.primary}>Click</button>

// 5️⃣ Tailwind
<button className="px-4 py-2 bg-blue-500 text-white rounded">Click</button>
```

---

## Q32. What is `React.StrictMode`?

**📖 Definition:** `StrictMode` is a **development-only** wrapper component that activates extra
checks and warnings for its subtree. It renders no visible UI and has **zero effect in
production**. It double-invokes render functions, state updater functions, and effects to expose
impure renders and missing cleanup, and warns about deprecated or unsafe legacy APIs.

**🧸 ELI5:** A **strict teacher** who checks your homework twice to catch mistakes early — only
during practice, never during the real exam.

**It catches:** unsafe lifecycles, missing effect cleanup, unexpected side effects in render.

---

## Q33. How do you give props a default value?

**📖 Definition:** Default prop values specify what a component should use when the parent does
not supply a given prop. In function components this is done with **ES6 default parameters** in
the destructuring pattern. The legacy `Component.defaultProps` static property is deprecated for
function components as of React 19.

```jsx
// ✅ Modern — default parameters
function Button({ text = "Click me", type = "button" }) {
  return <button type={type}>{text}</button>;
}
```

---

## Q34. What is one-way data flow?

**📖 Definition:** One-way (unidirectional) data flow means data moves in a **single direction —
from parent to child** — through props. A child cannot modify data it receives; to affect
ancestor state it must invoke a **callback function** passed down as a prop. This makes data
movement traceable: for any piece of state, there is exactly one owner and one direction of
propagation.

**🧸 ELI5:** Water flows **down** a waterfall, never up. To send something back up, the parent
hands the child a **walkie-talkie** (a callback).

```jsx
function Parent() {
  const [msg, setMsg] = useState("");
  return <Child onSend={setMsg} />;    // 👈 walkie-talkie going down
}

function Child({ onSend }) {
  return <button onClick={() => onSend("hello!")}>Send up</button>;
}
```

---

## Q35. `key` vs `id` vs `ref`?

**📖 Definition:** `key` and `ref` are **reserved React props** — they are consumed by React
itself and are *not* forwarded to the component as props. `key` identifies list items during
reconciliation; `ref` obtains a handle to a DOM node or component instance. `id` is an ordinary
HTML attribute passed straight through to the DOM.

| | Purpose |
|---|---|
| `key` | For **React** — identify list items while diffing. Not readable inside the component. |
| `id` | For **HTML/CSS/JS** — a normal DOM attribute. |
| `ref` | For **you** — a handle on the real DOM node. |

---

## Q36. `export default` vs named export?

**📖 Definition:** These are **ES module** export forms. A module may have at most **one default
export**, imported without braces under any local name. It may have any number of **named
exports**, which must be imported inside braces using their exact declared name (or renamed with
`as`).

```jsx
// default — one per file, import under any name
export default function Button() {}
import Button from "./Button";
import Btn from "./Button";        // also fine

// named — many per file, the name must match
export function Button() {}
export function Input() {}
import { Button, Input } from "./components";
```

---

## Q37. What actually happens when you call `setState`?

**📖 Definition:** Calling a state setter **enqueues an update** on the component's fiber and
schedules a re-render at the appropriate priority. React then enters the **render phase**: it
calls the component function to produce a new element tree and diffs it against the previous
one. It then enters the **commit phase**: it applies the minimal DOM mutations, runs layout
effects synchronously, and finally flushes passive effects (`useEffect`).

```
1. You click the button 🔘
2. React enqueues the update (doesn't run it yet)
3. React finishes the current work
4. React re-runs your component function       ← render phase
5. React diffs the new tree against the old
6. React mutates only what differs in the DOM  ← commit phase
```

---

## Q38. Common junior mistakes (knowing these = instant credibility)

```jsx
// ❌ 1. Calling the handler instead of passing it
<button onClick={handleClick()}>

// ❌ 2. Mutating state
items.push(x); setItems(items);

// ❌ 3. Index as key in a dynamic list
todos.map((t, i) => <li key={i}>)

// ❌ 4. Missing dependency → stale value
useEffect(() => { console.log(count) }, []);

// ❌ 5. Hook inside a condition
if (x) { useState() }

// ❌ 6. Forgetting e.preventDefault() on form submit

// ❌ 7. Accidentally rendering 0
{count && <p>hi</p>}

// ❌ 8. async function passed straight to useEffect
useEffect(async () => {}, []);   // returns a Promise, not a cleanup function
```

---

## Q39. How do you create a React app?

**📖 Definition:** A React application requires a build toolchain to compile JSX and bundle
modules. **Vite** is the current standard for single-page apps, using native ESM in development
and Rollup for production builds. **Next.js** is the standard when server-side rendering, file-
based routing, or SEO are required. **Create React App** was the historical default and is now
deprecated.

```bash
npm create vite@latest my-app -- --template react
cd my-app && npm install && npm run dev
```

**🗣️ Say this:** "Vite for SPAs, Next.js when I need routing, SSR, or SEO out of the box. CRA is
deprecated."

---

## Q40. Live-coding warmup: build a Todo app

**📖 Definition:** This exercise combines the core junior concepts: controlled input, immutable
array updates (add / toggle / remove), list rendering with stable keys, conditional styling, and
derived values computed during render rather than stored in state.

```jsx
import { useState } from "react";

export default function TodoApp() {
  const [todos, setTodos] = useState([]);
  const [text, setText] = useState("");

  const addTodo = (e) => {
    e.preventDefault();
    if (!text.trim()) return;
    setTodos(prev => [...prev, { id: crypto.randomUUID(), text, done: false }]);
    setText("");
  };

  const toggle = (id) =>
    setTodos(prev => prev.map(t => (t.id === id ? { ...t, done: !t.done } : t)));

  const remove = (id) => setTodos(prev => prev.filter(t => t.id !== id));

  return (
    <div>
      <form onSubmit={addTodo}>
        <input value={text} onChange={(e) => setText(e.target.value)} />
        <button type="submit">Add</button>
      </form>

      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            <input type="checkbox" checked={todo.done} onChange={() => toggle(todo.id)} />
            <span style={{ textDecoration: todo.done ? "line-through" : "none" }}>
              {todo.text}
            </span>
            <button onClick={() => remove(todo.id)}>❌</button>
          </li>
        ))}
      </ul>

      <p>{todos.filter(t => !t.done).length} left</p>   {/* derived, not stored */}
    </div>
  );
}
```

**Also practice:** Counter, Stopwatch, Search filter, Accordion, Star rating, Light/dark toggle,
Fetch + render a user list.

---

## ✅ Junior checklist — tick these before you sleep

- [ ] I can give the formal definition AND an analogy for props vs state
- [ ] I know why keys matter and why index keys break
- [ ] I can update string / array / object state immutably
- [ ] I can write `useEffect` with correct deps and cleanup
- [ ] I can build a controlled form from scratch
- [ ] I can fetch data with loading + error states
- [ ] I know both Rules of Hooks and *why* they exist
- [ ] I can build a Todo app in under 15 minutes

# 🚀 React Interview Prep — Start Here

**Your interview is tomorrow.** Here's exactly what to read and in what order.

---

## 📁 The files

| File | Level | Questions | Read if... |
|---|---|---|---|
| [01-junior-react-interview.md](01-junior-react-interview.md) | 🟢 Junior | 40 | 0–2 yrs, or you want the fundamentals airtight |
| [02-intermediate-react-interview.md](02-intermediate-react-interview.md) | 🟡 Intermediate | 40 | 2–5 yrs — **this is where most interviews are decided** |
| [03-senior-react-interview.md](03-senior-react-interview.md) | 🔴 Senior | 31 + rapid-fire | 5+ yrs, lead/architect roles |
| [04-cheatsheet-night-before.md](04-cheatsheet-night-before.md) | ⚡ All | Cram sheet | Read this last, and again in the morning |

Every answer has the same four parts:

- **📖 Definition** — the formal, textbook answer. **Open with this.** It's what an interviewer
  is listening for in the first 15 seconds
- **🧸 ELI5** — a plain analogy so you actually *understand* it, not memorize it. This is your
  fallback when they push deeper, and what makes the definition stick
- **Code** — a real, runnable example
- **🗣️ Say this** — the exact sentence to speak in the room

**How to combine them out loud:** lead with the definition, then *"the way I think about it
is…"* and give the analogy, then the tradeoff. Definition alone sounds memorized; analogy alone
sounds vague. Together they sound like someone who genuinely understands it.

---

## ⏱️ If you only have tonight

**3 hours available:**
```
45 min  → Junior file. Skim fast. Any question you can't answer in 20 seconds, mark it.
75 min  → Intermediate file. Read properly. This is the highest-value hour.
30 min  → Senior file: Q1–Q6 (Fiber, concurrent, transitions, Suspense, RSC) + Q31 rapid-fire.
20 min  → Cheat sheet.
10 min  → Write out your 3 STAR stories in bullet points.
```

**1 hour available:**
```
30 min  → Intermediate file, Q1–Q10 + Q29 (performance) + Q32 (stale closures)
15 min  → Cheat sheet
15 min  → Code a Todo app from scratch, no reference. Time yourself.
```

**Then sleep.** Genuinely — a rested brain beats one more hour of cramming, especially for
live coding.

---

## 🎯 The 10 questions most likely to come up

Whatever level you're interviewing for, be able to answer these cold:

1. Props vs state — and why state must be immutable
2. Why do lists need keys, and why is the index a bad key?
3. `useEffect` — dependency array, cleanup, and why it runs twice in dev
4. Why is my component re-rendering, and how do I stop it?
5. `useMemo` vs `useCallback` vs `React.memo` — and when NOT to use them
6. What is the Virtual DOM / reconciliation?
7. Write a custom hook (they'll ask for `useDebounce` or `useFetch`)
8. Controlled vs uncontrolled inputs
9. Context — what's the performance problem with it?
10. How do you handle a race condition in data fetching?

---

## 💬 How to actually answer in the room

**The 3-beat structure — use it on every single question:**

```
1. WHAT it is        (~30 seconds — proves you know it)
2. WHEN you'd use it (proves judgment)
3. WHAT it costs     (proves seniority) ← almost nobody does this
```

> *"Context solves prop drilling, at the cost of re-rendering every consumer on any change.
> I use it for low-frequency values like theme or auth, and reach for a selector-based store
> when the value changes often."*

That one sentence pattern will carry you through most of the interview.

---

## ✅ Do these things

- **Think out loud during live coding.** Silence reads as being stuck. Narrate: *"I'll debounce
  this so we don't hammer the API, then abort on cleanup so a stale response can't overwrite."*
- **Ask clarifying questions before coding.** "Should this handle pagination?" scores points.
- **Say "I don't know, but here's how I'd find out."** It beats bluffing every time, and
  interviewers can always tell.
- **Name a tradeoff, unprompted.** This is the single biggest signal of level.
- **Have 2–3 questions ready for them.** Ask about testing culture, how they handle tech
  debt, or what the last hard architectural decision was.

## ❌ Don't do these

- Don't memorize answers word-for-word — you'll sound scripted and fall apart on follow-ups
- Don't claim experience you don't have; the follow-up question will find it
- Don't say "React is fast because of the Virtual DOM" — it's not *why*, and it's a known
  interviewer trigger. Say it minimizes expensive DOM mutations
- Don't start coding immediately — 30 seconds of clarification saves 10 minutes of rework

---

## 🖥️ Live-coding problems to practice

**Junior:** Counter · Todo list · Search filter · Star rating · Accordion · Dark mode toggle

**Intermediate:** Debounced search with cancellation · Pagination · Custom `useFetch` ·
Modal with portal + focus trap · Multi-step form wizard · Shopping cart with `useReducer`

**Senior:** Infinite scroll + virtualization · Autocomplete with keyboard nav + a11y ·
Undo/redo · Real-time list over WebSocket · Design a data table (sort/filter/paginate/select)

---

## 🧠 One-line memory hooks

| Concept | Hook |
|---|---|
| Props | Lunchbox mom packed — you can't change it |
| State | Your own backpack — you can |
| `useEffect` | "After I finish drawing, do this" |
| Cleanup | Turn off the tap before leaving the room |
| Keys | Name tags so React knows who's who |
| Virtual DOM | Compare two sketches, move only the furniture that changed |
| `useMemo` | Write the hard answer on your hand |
| `useRef` | Sticky note on the fridge — no redecorating |
| Context | Teacher's loudspeaker instead of passing notes |
| Suspense | "I'm not ready yet!" — show the placeholder |
| Fiber | Color a bit at a time, look up, and be interruptible |
| RSC | The toy stays at grandma's; you get sent the photo |

---

**Good luck tomorrow. You've got this. 🍀**

# JavaScript Event Loop & Async Execution Model

> **Session:** May 28, 2026
> **Topics:** Call Stack → Event Loop → Macrotask Queue → Microtask Queue → Promise Chaining Order → `async/await` Internals
> **Tags:** [javascript, event-loop, async, await, promises, microtasks, macrotasks, concurrency]

---

## The Call Stack

Synchronous code runs line by line on the call stack. Nothing fancy — JS executes it immediately in order.

---

## The Two Queues

When async work is involved, JS doesn't put everything in one place. There are **two separate queues**:

| Queue | What goes here | Priority |
|---|---|---|
| **Microtask Queue** | `Promise.then()`, `queueMicrotask()` | **Higher** — drained completely first |
| **Macrotask Queue** | `setTimeout`, `setInterval`, browser events | **Lower** — picked up one at a time after microtasks |

### The Rule
When the call stack is empty:
1. Drain **all** microtasks first
2. Then pick up **one** macrotask
3. Drain microtasks again
4. Repeat

---

## Example 1 — Basic Order

```javascript
console.log("1");
setTimeout(() => console.log("2"), 0);
Promise.resolve().then(() => console.log("3"));
console.log("4");
```

**Output:** `1, 4, 3, 2`

- `1` and `4` → synchronous, run immediately
- `Promise.then` → microtask queue
- `setTimeout` → macrotask queue
- Stack empties → microtask (`3`) runs first → then macrotask (`2`)

---

## Example 2 — Multiple Promise Chains

```javascript
Promise.resolve()
  .then(() => {
    console.log("A");
    return Promise.resolve();
  })
  .then(() => console.log("B"));

Promise.resolve().then(() => console.log("C"));
```

**Output:** `A, C, B`

- Both chains queue their first `.then` as microtasks
- `A` runs first (chain 1), `C` was already queued (chain 2)
- Returning `Promise.resolve()` inside `.then` adds an **extra microtask hop** before `B` runs
- By the time `B` is queued, `C` is already ahead of it

> **Key insight:** Two parallel chains don't wait for each other — the microtask queue interleaves them.

---

## Example 3 — `async/await` Internals

```javascript
async function fetchData() {
  console.log("A");
  const data = await fetch("...");
  console.log("B");
}

fetchData();
console.log("C");
```

**Output:** `A, C, B`

- `fetchData()` is called → starts executing → logs `A`
- Hits `await` → **pauses the function** (not all of JS)
- Control returns to the call stack → logs `C`
- `fetch` resolves → `B` is added to microtask queue → logs `B`

### What `await` actually does
- Pauses **only the current function**
- JS is free to do other work in the meantime
- Everything after the `await` line is essentially a `.then()` callback under the hood

---

## Mental Model Summary

```
Synchronous code
      ↓
Call stack empties
      ↓
Drain microtask queue (ALL of them)
      ↓
Pick one macrotask
      ↓
Drain microtask queue again
      ↓
Repeat...
```

---

## Quiz Results ✅

| Q | Topic | Result |
|---|---|---|
| Nested setTimeout inside a Promise chain | Macrotask/Microtask ordering | ✅ |
| What does `await` pause? | async/await internals | ✅ |
| Two parallel Promise chains | Microtask interleaving | ✅ |
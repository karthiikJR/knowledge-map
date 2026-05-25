# Idempotency — Learning Notes

> **Session:** May 25, 2026  
> **Topics:** Idempotency → HTTP methods - Natural Idempotency
> **Tags:** [idempotency, backend, distributed-systems, http, rest-api, payments, database, transactions, next-js, spring-boot, supabase]

---

## What is Idempotency?

> **No matter how many times you perform an operation, the outcome is the same as doing it once.**

---

## Why It Matters (The Boutique Problem)

A user clicks "Place Order". The server processes it but the response never reaches the browser. The user retries.

Without idempotency:
- **Duplicate order** gets created
- **Double payment** is charged
- **Inventory is reduced twice**

---

## HTTP Methods — Natural Idempotency

| Method | Idempotent? | Why |
|--------|-------------|-----|
| GET    | ✅ Yes | Fetching data changes nothing |
| PUT    | ✅ Yes | Sets resource to a fixed value — same result every time |
| POST   | ❌ No  | Creates a new resource each time |
| PATCH  | ❌ No (usually) | Applies a delta — `price += 200` run 3 times = wrong |

---

## The Idempotency Key Pattern

### Concept
The **frontend** generates a unique key for each operation attempt and sends it with the request. The server uses this key to detect retries.

### Flow (Pseudocode)
```
isDuplicate = fetchIdempotent(user_id, key)

if (isDuplicate) {
  return the same response as the first request
}

transaction {
  storeIdempotent(user_id, key, response)
  res = storeOrder(data)
}

return res
```

> ⚠️ The idempotency store and order creation must be in the **same transaction** — if the order fails, the key should not be stored.

---

## Storage Design

### Idempotency Table
| Column | Purpose |
|--------|---------|
| `id` | Primary key |
| `user_id` | Scope key per user |
| `key` | The idempotency key from the client |
| `order_id` | Reference to the created order (to reconstruct response) |
| `created_at` | Used to calculate TTL |

### TTL
Keys should expire after **10–15 minutes** — the realistic window in which a user would retry the same operation.

### Response Strategy
Two valid approaches:
- **Store raw response JSON** — simple to return, costs storage
- **Store order_id and refetch** — leaner storage, slightly more complex reconstruction

For the boutique, refetching from the order table is perfectly reasonable.

---

## Where Does the Check Live? (Next.js Architecture)

```
types → service → BFF controller (route.ts) → actions → components
```

The idempotency check belongs in **route.ts (BFF controller)** — as the very first thing, before any action is called.

> The controller decides *if* the work should be done. The action does the work.

---

## Key Takeaways

1. Idempotency = same outcome regardless of how many times an operation runs
2. POST is not naturally idempotent — you have to engineer it
3. The frontend generates the key, the backend enforces it
4. Always wrap the key storage + order creation in a **transaction**
5. Use a TTL to clean up old keys automatically
6. Check at the **controller layer** — fail fast, before any business logic runs

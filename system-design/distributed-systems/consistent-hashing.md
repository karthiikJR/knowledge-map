# Consistent Hashing

> **Session:** May 10, 2026  
> **Topics:** Naive Modulo Caching → Consistent Hashing → Virtual Nodes (vnodes)
> **Tags:** [distributed-systems, caching, consistent-hashing, virtual-nodes, cache-stampede, load-balancing]

---

## The Problem — Naive Modulo Caching

When distributing keys across cache nodes, a naive approach is:

```
node = hash(key) % N
```

This works fine until you **add or remove a node**. Changing `N` means recomputing every key's destination.

**Example — going from 3 to 4 nodes:**

| key | % 3 | % 4 | same? |
|-----|-----|-----|-------|
| 0   | 0   | 0   | ✅    |
| 1   | 1   | 1   | ✅    |
| 2   | 2   | 2   | ✅    |
| 3   | 0   | 3   | ❌    |
| 4   | 1   | 0   | ❌    |
| 5   | 2   | 1   | ❌    |
| 6   | 0   | 2   | ❌    |
| 7   | 1   | 3   | ❌    |
| 8   | 2   | 0   | ❌    |
| 9   | 0   | 1   | ❌    |
| 10  | 1   | 2   | ❌    |
| 11  | 2   | 3   | ❌    |

**~75% of keys get remapped** when adding just one node.

### Cache Stampede (Thundering Herd)
Those remapped keys are now pointing to nodes that never stored them → **cache miss**. The application falls through to the database for every miss. At scale (e.g. 10M sessions), this means millions of simultaneous DB hits → database gets crushed. This is called a **cache stampede**.

---

## The Solution — Consistent Hashing

### The Ring
Imagine a circle from 0° to 360°. Place nodes at positions determined by `hash(node_id)`:

```
A → 0°
B → 120°
C → 240°
```

To find which node owns a key: hash the key to a degree, then go **clockwise until you hit a node**.

### Adding a Node
Add Node D at 60°. Only keys in the arc **0° → 60°** (previously owned by B going clockwise) get remapped to D. Everything else is untouched.

**~1/N keys remapped** instead of ~(N-1)/N.

| Approach           | Keys remapped (3→4 nodes) |
|--------------------|--------------------------|
| Naive modulo       | ~75%                     |
| Consistent hashing | ~25%                     |

As your cluster grows (e.g. 100 nodes), adding one node only remaps **~1%** of keys.

---

## The Problem with Consistent Hashing — Uneven Distribution

Node positions are determined by `hash(node_id)` — you can't control them. This can result in one node owning 60% of the ring while another owns 10% → **load imbalance / hotspots**.

---

## The Fix — Virtual Nodes (vnodes)

Instead of each physical node occupying one position on the ring, it occupies **many positions**:

```
hash("NodeA-1") → 15°
hash("NodeA-2") → 130°
hash("NodeA-3") → 290°
```

With enough virtual nodes (e.g. 100 per physical node), the law of large numbers kicks in — each physical node ends up owning roughly **1/N** of the ring naturally, without careful placement.

**Used in production by:** Redis Cluster, Apache Cassandra, Amazon DynamoDB, CDN edge caches.

---

## Key Takeaways

1. `hash(key) % N` is simple but catastrophic at scale when N changes.
2. Consistent hashing limits remapping to ~1/N of keys on topology changes.
3. Virtual nodes solve the uneven distribution problem without extra hardware.
4. The underlying idea: **only neighbours are affected by a change**, not the whole cluster.

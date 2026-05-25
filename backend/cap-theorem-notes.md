# Distributed Systems — CAP Theorem & Database Consistency

> **Session:** May 11, 2026  
> **Topics:** Cache Stampede (review) → CAP Theorem → Consistency Models → Replication Strategies
> **Tags:** [distributed-systems, cap-theorem, cache-stampede, consistency-models, replication, eventual-consistency, strong-consistency, acid, base, databases]

---

## Cache Stampede (Review)

**Definition:**  
When **concurrent requests** hit a cache key that has expired, instead of reading from the cache the data is read from the database — making it overwhelmed and unresponsive.

### Prevention Strategies

| Strategy | How it works | Tradeoff |
|---|---|---|
| **Mutex / Lock** | Only one request recomputes the value, rest wait | Waiting requests hold open threads — app server can run out of threads |
| **Probabilistic Early Expiration** | Refresh the cache before TTL hits zero, so the key never goes cold | Slightly wasteful — refreshes data that might not be read |

---

## CAP Theorem

In a distributed system, you can only guarantee **2 out of 3** properties:

| Letter | Stands For | Meaning |
|---|---|---|
| **C** | Consistency | Every read returns the latest write |
| **A** | Availability | Every request gets a response |
| **P** | Partition Tolerance | System keeps working even when nodes can't communicate |

### Why P is Non-Negotiable

Network partitions **will** happen — cables get cut, hardware fails, datacenters lose connectivity. You cannot architect around physics.

Since P is mandatory, the **real choice** is always:

- **CP** — Consistent + Partition Tolerant → sacrifice availability (reject requests during partition)
- **AP** — Available + Partition Tolerant → sacrifice consistency (serve stale data during partition)

> **Why not CA?**  
> CA assumes partitions never happen — which only holds for a single-node database. A single node is a single point of failure. CA doesn't exist at production scale.

---

## Consistency Models

### Strong Consistency (CP)
- Every read is guaranteed to return the latest write
- Writes are slow — must propagate to all nodes before confirming
- During a partition, writes are **rejected** to protect correctness
- Example: **Bank balance** — you cannot show stale data

### Eventual Consistency (AP)
- Reads may return stale data temporarily
- System will *eventually* converge to the same state across all nodes
- No formal guarantee on **how long** "eventually" takes — could be seconds, could be longer
- During a partition, stale reads are served rather than rejecting requests
- Example: **Social media feed** — a post showing up 2 seconds late is acceptable

---

## BASE vs ACID

| | ACID | BASE |
|---|---|---|
| **Stands for** | Atomicity, Consistency, Isolation, Durability | **B**asically **A**vailable, **S**oft state, **E**ventual consistency |
| **Priority** | Data correctness | Availability |
| **Used by** | PostgreSQL, relational DBs | MongoDB (default), Cassandra |

- **Basically Available** — system guarantees a response, but it may be stale
- **Soft state** — data may change over time even without new writes, as replicas sync
- **Eventual consistency** — all nodes will converge, but not immediately

---

## Replication Strategies

How data is copied from primary to replica nodes:

### Synchronous Replication (CP)
- Primary writes data, **waits** for replica to confirm receipt, then responds to user
- User experiences **latency** on writes
- Data is always consistent across nodes
- Use for: **inventory, payments, anything where stale = harmful**

### Asynchronous Replication (AP)
- Primary writes data, **immediately** responds to user, replica catches up later
- Fast response times
- Replica may serve stale data temporarily
- Use for: **product images, descriptions, anything where stale = acceptable**

---

## Real World Application — Boutique Store

| Feature | Strategy | Reason |
|---|---|---|
| Inventory count | **CP + Synchronous replication** | Overselling = real business harm |
| Product images / descriptions | **AP + Asynchronous replication** | Slightly stale image causes no harm |

### The Tradeoff
Using synchronous replication for inventory means the user experiences a **slight delay** at checkout while Mumbai confirms with Frankfurt. That is the cost of correctness.

---

## Where Popular Databases Fall

| Database | Default | Configurable? |
|---|---|---|
| **PostgreSQL** | CP (ACID) | Yes — via replication config (sync vs async) |
| **MongoDB** | AP (BASE) | Yes — via **write concern** setting |
| **Cassandra** | AP | Yes — via consistency level per query |

> Key insight: Most modern databases don't lock you into one model. You can tune consistency **per operation** based on how critical the data is.

---

## Key Takeaways

1. CAP theorem is a real architectural decision, not just theory
2. P is always required — your real choice is CP vs AP
3. Different data in the same app can have different consistency requirements
4. Eventual consistency has **no time guarantee** — design accordingly
5. Strong consistency = latency cost on writes
6. The right choice depends on your **business requirements**, not just technical preference

---

*Next topics to explore: Distributed Locking, Database Indexing (B-trees)*

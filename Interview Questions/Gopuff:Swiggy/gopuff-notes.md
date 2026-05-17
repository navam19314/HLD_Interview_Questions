# 💨 Gopuff (Local Delivery Service) — System Design Revision Notes

> Based on **Hello Interview's Delivery Framework**
> Time budget: **5min Requirements → 2min Entities → 5min API → 15min HLD → 10min Deep Dives**

---

## 0. Problem Snapshot

**Gopuff** = rapid (≤1 hour) delivery of convenience-store goods from **500+ micro Distribution Centers (DCs)**.

🔑 **Single most important insight:** This problem is about **aggregating availability across nearby DCs** + **preventing double-booking on orders**. Two contradictory non-functional requirements:
- Availability reads = **fast (<100ms), high throughput, can tolerate slight staleness**
- Orders = **strongly consistent, must not double-book**

These two opposing needs drive the whole design.

---

## 1. Functional Requirements (~5 min — first half)

### Core (in scope)
1. Customers can **query availability** of items by **location** (effective availability = **union of inventory in all nearby DCs**, deliverable in 1 hour)
2. Customers can **order multiple items at once**

### Below the line (out of scope)
- Payments / checkout
- Driver routing & deliveries
- Search functionality / catalog APIs
- Cancellations & returns

⚠️ Stay narrow — this problem is **availability + ordering**, not a full e-commerce platform.

---

## 2. Non-Functional Requirements

| # | Requirement | Quantified Target |
|---|---|---|
| 1 | **Availability requests fast** (for search-like UX) | < 100 ms |
| 2 | **Orders strongly consistent** — no double-booking same physical item | Hard constraint |
| 3 | **Scale** | 10k DCs, 100k items in catalog |
| 4 | **Order volume** | O(10M orders/day) |

### Below the line
- Privacy & security
- Disaster recovery

🔑 **The tension:** reads want availability + speed (lean toward eventual consistency + caching). Writes (orders) want strict consistency. Different parts of the system need different CAP tradeoffs.

---

## 3. Core Entities (~2 min)

🔑 **Most important conceptual distinction:** `Item` vs `Inventory`.
Think Class vs Instance in OOP.
- API consumers care about **Items** (e.g., "Cheetos")
- The system tracks **Inventory** = a *physical* instance of an Item at a *specific* DC

### Entity list
- **Item** — type of product (e.g., Cheetos). What the user sees.
- **Inventory** — physical instance of an Item at a DC. We sum these to get availability.
- **DistributionCenter (DC)** — physical location storing Inventory.
- **Order** — collection of Inventory ordered by a user + shipping/billing info.

🔑 Start with **concrete physical entities** (Item, DC) and work up to abstract ones (Order). Don't miss any.

---

## 4. API Design (~5 min)

Just 2 endpoints needed.

### Get availability by location
```http
GET /availability?lat={lat}&long={long}&keyword={kw}&page={n}
→ Item[] with quantities
```
- Location passed so backend can find nearby DCs
- Paginated to avoid overwhelming the client

### Place an order
```http
POST /orders
{
  "lat": ..., "long": ...,
  "items": [{ "itemId": "...", "quantity": ... }]
}
→ Order
```
- 🔑 **Location is passed to order endpoint too** — must verify inventory is close enough to deliver in 1 hour before processing

User identity comes from auth header, not request body.

---

## 5. High-Level Design (~10–15 min)

Build one functional requirement at a time.

---

### Flow 1 — Query Availability

Two steps to keep this under 100ms:

#### Step A — Find nearby DCs
- Internal API: `(lat, long) → DC[]`
- Pull DCs from a DC table (lat/long stored)
- Crude version: **Euclidean** or **Haversine** distance, filter by radius
- ⚠️ This is naive (ignores roads, traffic) — fixed in Deep Dive 1

#### Step B — Look up inventory in those DCs
- Query `Inventory` table joined with `Item` table
- Sum quantities across all nearby DCs → return to user
- Use **Postgres** for both tables

🔑 **In a real system you'd separate Catalog from Inventory** (different workloads/consumers) and add an **Elasticsearch** search index. Out of scope here — note it to interviewer.

### Components after Flow 1
| Service | Role |
|---|---|
| **Availability Service** | Handles user availability requests |
| **Nearby Service** | Returns DCs within 1-hour delivery; calls external **Travel Time Service** |
| **Inventory Table (Postgres)** | Stores inventory per item per DC |

### Availability request walkthrough
1. User → Availability Service with `(items, lat, long)`
2. Availability Service → Nearby Service with `(lat, long)`
3. Nearby Service returns list of serviceable DCs
4. Availability Service queries DB filtering by those DC IDs
5. Sum quantities → return to client

---

### Flow 2 — Place an Order

🔑 **Hard requirement: strong consistency.** Two users CANNOT order the same physical item.

This is the **classic double-booking problem** — needs locking.

#### 👍 Good — Two data stores + distributed lock
- Separate DBs for orders & inventory
- Acquire lock on inventory → create order → decrement inventory → release lock
- Lets you pick best DB per use case (KV for inventory, relational for orders)

⚠️ **Nasty failure modes:**
- **Crash mid-flow** — order created but inventory not decremented → next user can order same item. Need a sweeper to detect/reverse.
- **Deadlock** — User1 holds lock on A, wants B. User2 holds B, wants A. Neither proceeds.

#### 🔥 Great — Single Postgres transaction (ACID)
- Put orders + inventory in **same Postgres DB**
- Wrap everything in **one transaction with `SERIALIZABLE` isolation**
- Atomically: check inventory > 0 → record order → decrement inventory
- If two users hit the same item, **one transaction fails to commit** → cleanly rejected

**Tradeoff:** can't use specialized stores for each; coupling on scaling. Acceptable for the simplicity.

🔑 **Rule of thumb:** when atomicity across multiple things is required → colocate in an ACID store. Cross-DB transactions are possible but the overhead isn't worth it in an interview.

#### Order transaction walkthrough
1. User → Orders Service with item list
2. Orders Service builds single transaction sent to **Postgres leader**:
   - Check inventory for A, B, C > 0
   - If any out of stock → **whole order fails** (return meaningful error — "device + battery should fail together")
   - If all available → mark inventory rows "ordered", insert into `Orders` + `OrderItems` tables
   - Commit
3. Return order to user

---

### Putting It All Together

| Component | Role |
|---|---|
| **Availability Service** | Reads — uses Nearby Service + queries inventory from **read replicas** |
| **Orders Service** | Writes — single atomic transaction to **Postgres leader** |
| **Nearby Service** | Shared by both — computes serviceable DCs |
| **Postgres** | Inventory + Orders + OrderItems, **partitioned by region**; **leader for writes, replicas for reads** |

🔑 **Read/write separation:**
- Availability tolerates slight staleness → read replicas
- Orders need linearizable consistency → leader only

---

## 6. Deep Dive 1 — Realistic Travel Time (Traffic + Drive Time, Not "As the Crow Flies")

**Problem:** A DC across a river or behind heavy traffic might be 5 miles away but 70 minutes to drive. Naive distance filtering misses this.

### ❌ Bad — Simple SQL distance
Euclidean / Haversine threshold.
- Ignores traffic, road conditions, real geography.
- DCs in the same city are over-/under-counted.

### ❌ Bad — Travel Time Service against ALL DCs
- Sync DC table periodically (every 5 min) into service memory (DCs rarely change — they're buildings!)
- For each request, call external Travel Time Service for every DC
- ⚠️ Far too many calls — most DCs aren't remotely close enough

### 🔥 Great — Travel Time Service against **NEARBY** DCs (two-stage filter)
1. **Coarse filter** — pre-filter to candidate DCs within a fixed radius (e.g., **60 miles** — most optimistic 1-hour drive)
2. **Precise filter** — pass candidates to **external Travel Time Service** (accounting for traffic, roads)
3. Return DCs with ETA ≤ 1 hour

**Why this works:** Drastically reduces external API calls while remaining accurate. The 60-mile prefilter eliminates >99% of irrelevant DCs.

🔑 **General pattern:** two-stage filter — cheap coarse filter, then expensive precise filter — is a recurring trick in geo problems (also seen in Uber, Yelp).

---

## 7. Deep Dive 2 — Make Availability Fast & Scalable (20k QPS)

### Capacity math (do this live)
```
Order volume: 10M orders/day
Seconds/day: ~100k (86,400 ≈ 100k for napkin math)
Page views per purchase: ~10 (search, home, browsing)
Conversion rate: 5% (only 5% of viewers buy)

Queries = 10M / 100k * 10 / 0.05
       = 100 * 10 / 0.05
       = 20,000 QPS for availability
```

🔑 **20k QPS is a lot for a single Postgres.** Must scale reads.

### 🔥 Great — Cache (Redis) with short TTL

- Redis between Availability Service and DB
- Cache key = (location bucket / DC set + item) → quantity
- **TTL ~1 minute** — short enough to stay fresh, long enough to absorb most queries

**Cache invalidation:**
- Orders Service **explicitly expires** affected cache entries when it writes inventory
- Short TTL is a safety net for missed invalidations

**Why short TTL is key:** Inventory drops to zero quickly; users see stale "in stock" if TTL too long.

### 🔥 Great — Postgres Read Replicas + Partitioning

#### Partitioning by region
- Group DCs by **first 3 digits of zipcode** → region ID
- Partition inventory table by region ID
- Most availability queries hit **only 1–2 partitions** (a user's local region)
- Each partition is smaller → faster queries, less contention

#### Read replicas
- **Availability queries** → read replicas (tolerable staleness)
- **Order transactions** → leader (strong consistency)

🔑 **Reuse the same DB.** No new tech; just leverage Postgres's built-in replication and partitioning.

### Combined: Cache + Partitioned Replicas
Most reads served by Redis. Cache misses go to a region-partitioned read replica, hitting only the relevant slice of data. Each layer absorbs load before it reaches the leader.

⚠️ **Operational concerns:**
- Right-size replica count for traffic
- Don't overload one partition (popular regions need more replicas)
- Monitor cache hit rate

---

## 8. Key Numbers Cheatsheet

| Metric | Number |
|---|---|
| DCs | 10k (500+ active per article) |
| Items in catalog | 100k |
| Orders/day | 10M |
| Availability QPS (computed) | **20k** |
| Page views per order | 10 |
| Conversion rate | 5% |
| Delivery time SLA | 1 hour |
| Coarse-filter radius | 60 miles |
| Availability latency target | < 100 ms |
| Cache TTL | ~1 minute |
| Seconds/day for napkin math | ~100k |

---

## 9. Tradeoffs Cheatsheet

| Decision | Chosen | Why | Tradeoff |
|---|---|---|---|
| Order DB strategy | **Single Postgres transaction (ACID)** | Atomic, simple, no deadlock complexity | Coupled scaling for orders + inventory |
| Inventory DB | **Postgres** (not a KV store) | ACID transactions with orders | Less optimal for raw inventory ops alone |
| Catalog vs Inventory | **Same DB** (for interview) | Simpler; meets requirements | Ideally separate + Elasticsearch in real life |
| Nearby DC algorithm | **Coarse 60mi + Travel Time API for candidates** | Accurate w/o blowing up API costs | More complex than single check |
| Read scaling | **Redis cache + read replicas + region partitioning** | Defense in depth | Cache invalidation logic; partition sizing |
| Cache TTL | **~1 min** | Fresh enough, absorbs spikes | Some stale data possible |
| Write path | **Leader only** | Strong consistency for orders | Leader is bottleneck; mitigated by partitioning |
| Region key | **First 3 digits of zipcode** | Coarse-grained but effective | Imperfect; some cross-region queries |
| Order failure semantics | **All-or-nothing** | Avoids nonsense partial orders (device w/o battery) | Frustrating for user → return meaningful error |

---

## 10. Common Pitfalls & Gotchas

- ⚠️ **Item vs Inventory confusion** — the most common modeling mistake. Item = class; Inventory = instance at a DC.
- ⚠️ **Distance-as-crow-flies is wrong** for delivery — rivers, borders, traffic. Use Travel Time service for the precise check.
- ⚠️ **Calling Travel Time API for every DC** — too expensive. Coarse-filter first.
- ⚠️ **Distributed locks for ordering** — deadlocks and crash-recovery failure modes. Prefer single-DB ACID transaction.
- ⚠️ **Not separating reads and writes** — orders to leader, availability to replicas. Different consistency needs.
- ⚠️ **Cache without explicit invalidation** — orders must expire cache entries; don't rely on TTL alone.
- ⚠️ **Forgetting capacity math** — 20k QPS is the kind of number that justifies caching + replicas. Do the math live.
- ⚠️ **Partial order success** — better to fail the whole order than ship half (a phone with no charger is useless).
- ⚠️ **Forgetting location on order endpoint** — must verify inventory is reachable before committing.
- ⚠️ **Don't add Elasticsearch / separate catalog DB unprompted** — out of scope; mention it but don't build it.

---

## 11. Level Expectations

### 🟢 Mid-level (80% breadth / 20% depth)
- Clear APIs + entity model (especially **Item vs Inventory** distinction)
- Both flows working: availability + orders
- It's OK to start with a "bad" solution if discussion is good
- Don't need to immediately jump to the optimal solution
- Probing questions from interviewer expected

### 🟡 Senior (60% breadth / 40% depth)
- Speed through initial HLD
- Spend time on **critical paths**: atomic order transactions + scaling availability
- Should proactively identify high read load
- Articulate tradeoffs (single-DB transaction vs distributed lock; cache + replicas + partitioning)
- Anticipate bottlenecks before being asked

### 🔴 Staff+ (40% breadth / 60% depth)
- Breeze past basics
- Deep dives on **2–3 areas**: travel-time filtering, locking strategies, partitioning scheme
- "Been there, done that" depth — real-world tradeoffs
- Unique insights on follow-ups of increasing difficulty
- Steer the conversation; treat interviewer as peer
- Could go deep on: cross-region replication, partition rebalancing, hotspot mitigation, cache stampedes

---

## 🎯 30-Second Recall (eve-of-interview)

> **Local delivery, 1-hour SLA, 500+ DCs.** Two opposing constraints: availability reads are **read-heavy (~20k QPS), <100ms, eventually consistent OK** → Redis cache (~1 min TTL) + Postgres read replicas + **partition by zipcode region**. Orders need **strong consistency** → single Postgres ACID transaction with `SERIALIZABLE` isolation, written to leader (no distributed locks, no deadlocks). Key entity distinction: **Item** (class) vs **Inventory** (physical instance at a DC). Nearby DCs found via **two-stage filter**: coarse 60mi radius → precise Travel Time API call on candidates. Don't forget capacity math: 10M orders/day × 10 views × 1/5% = 20k QPS.

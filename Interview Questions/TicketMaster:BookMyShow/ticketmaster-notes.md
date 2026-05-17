# 🎟️ Ticketmaster (Ticket Booking) — System Design Revision Notes

> Based on **Hello Interview's Delivery Framework**
> Time budget: **5min Requirements → 2min Entities → 5min API → 15min HLD → 10min Deep Dives**

---

## 0. Problem Snapshot

**Ticketmaster** = online platform to buy tickets for concerts, sports, theater, etc.

🔑 **Single most important insight:** This problem is the canonical **"Dealing with Contention"** question. Reads (view + search) want **availability + speed**; the booking path needs **strict consistency** to prevent double-booking — even when 10M users are racing for the same Taylor Swift tickets. The whole design pivots on this asymmetry.

---

## 1. Functional Requirements (~5 min — first half)

### Core (in scope)
1. Users can **view events**
2. Users can **search events**
3. Users can **book tickets** to events

### Below the line (out of scope)
- View one's booked events
- Admins / event coordinators adding events
- Dynamic pricing for popular events

---

## 2. Non-Functional Requirements

| # | Requirement | Quantified Target |
|---|---|---|
| 1 | **Mixed CAP:** availability for view/search; **consistency for booking** (no double-booking) | Hard constraint on booking |
| 2 | **Scale** — high throughput for popular events | 10M users on one event |
| 3 | **Low-latency search** | < 500 ms |
| 4 | **Read-heavy** | 100:1 read-to-write |

### Below the line
- GDPR / privacy
- Fault tolerance
- Secure transactions (PCI)
- CI/CD
- Backups

🔑 **The CAP split is the most important framing:** different parts of the same system need different tradeoffs. Mention this explicitly.

---

## 3. Core Entities (~2 min)

1. **Event** — date, description, type, performer/team. Central content entity.
2. **User** — the booker.
3. **Performer** — artist, team, company; name, bio, links.
4. **Venue** — location, capacity, **seat map** (JSON/related table with sections, rows, seat coords for rendering).
5. **Ticket** — `eventId`, seat info, price, **status (available/reserved/booked)**. One ticket per seat per event.
6. **Booking** — `userId`, `ticketId[]`, total price, status (in-progress/confirmed). Groups multi-seat transactions.

🔑 **Why a separate Booking entity?** Could fold into Ticket, but Booking groups multiple tickets under one payment + status. Worth separating.

🔑 **Where does the seat map live?** On `Venue`. Client combines venue seat map + per-ticket status to render the interactive seat selection UI.

---

## 4. API Design (~5 min)

### View an event
```http
GET /events/:eventId → Event & Venue & Performer & Ticket[]
```
Tickets returned so client can render the seat map.

### Search events
```http
GET /events/search?keyword=&start=&end=&pageSize=&page= → Event[]
```

### Book tickets (will evolve!)
```http
POST /bookings/:eventId → bookingId
{
  "ticketIds": [...],
  "paymentDetails": ...
}
```

⚠️ **Tell the interviewer up front:** "This will evolve into **two endpoints** — one to **reserve** a ticket (hold it), one to **confirm purchase** (after payment)."

---

## 5. High-Level Design (~10–15 min)

### Flow 1 — View Event

| Component | Role |
|---|---|
| **Client** | Web/mobile |
| **API Gateway** | Routing, auth, rate limiting, logging |
| **Event Service** | Reads event/venue/performer/tickets |
| **Events DB (Postgres)** | Events, performers, venues, tickets |

**Path:** Client → API Gateway → Event Service → Events DB → response

---

### Flow 2 — Search Events

Start dumb, evolve later (Deep Dive 4).

- **Search Service** receives query params, runs `LIKE '%keyword%'` style query on Events DB
- ⚠️ Full table scans, slow — fixed in deep dive

---

### Flow 3 — Book Tickets (the hard one)

🔑 **Core requirement:** No double-booking. Two users CANNOT pay for the same ticket.

#### New tables
- **Tickets** — `eventId, seat, price, status, bookingId`
- **Bookings** — `userId, ticketIds[], total, status`

#### Components
| Component | Role |
|---|---|
| **Booking Service** | Orchestrates booking; talks to DB + payment processor |
| **Payment Processor (Stripe)** | External; handles card transactions; webhooks back to us |

#### Naive flow (will get better)
1. User picks seats → `POST /bookings`
2. Booking Service starts a DB transaction:
   - Check selected tickets are `available`
   - Set `status = 'booked'`
   - Insert Booking row
3. Commit → return success; or fail if someone beat them

⚠️ **Massive UX problem:** User types card info, *then* finds out ticket is gone. Fixed in Deep Dive 1 (reservations).

#### On sharing a DB across services
"Database per service" is a guideline, not a law. Here, data is tightly coupled (Bookings → Tickets → Events) and we need ACID across them. Sharing one Postgres is the right call. **Mention the tradeoff** instead of dogmatically splitting.

---

## 6. Deep Dive 1 — Ticket Reservation (Solving Contention)

The classic ticketing problem. Four solutions with sharply different tradeoffs.

### ❌ Bad — Long-Running DB Locks (`SELECT FOR UPDATE`)
- Open a DB transaction, lock the row for 5+ minutes while user checks out
- ⚠️ DB locks are designed for **near-instant** operations
- ⚠️ Strains DB resources; risk of contention, deadlocks
- ⚠️ Doesn't scale under load; bad failure modes (crashes leave hung locks)

### 👍 Good — Status field + expiration + cron job
- Ticket has 3 states: `available`, `reserved`, `booked`
- On select → flip to `reserved`, store expiration timestamp
- **Cron job** periodically reverts expired reservations to `available`

⚠️ **Problems:**
- **Delay** between expiration and cron sweep → tickets sit unavailable longer than needed (bad for popular events)
- **Reliability** — if cron fails, *everything is stuck reserved*

### 🔥 Great — Implicit status via short transactions
🔑 **The clever idea:** Don't run a cron at all. The "real" status of a ticket = `available OR (reserved AND expired)`.

In a short transaction:
1. Begin transaction
2. Check ticket is `AVAILABLE` **OR** (`RESERVED` and expiration < now)
3. Update to `RESERVED` with expiration = now + 10 min
4. Commit

**Pros:** Only one user wins the reservation. Expired reservations are reclaimable on next attempt. **System doesn't depend on the cron running** — sweep is optional cleanup, not correctness-critical.

**Cons:** Read queries slightly slower (two-condition filter). Mitigate with **compound index** or **materialized view**. Optional cleanup cron keeps the table tidy.

### 🔥 Great — Distributed Lock (Redis) with TTL

🔑 **Why Redis when Postgres already has consistency?** Because Redis gives you **automatic key expiration** built-in. Postgres has no row-level TTL — you'd need cron-like app logic. Redis also handles lock acquire/release extremely fast under high concurrency.

#### Flow
1. User selects ticket → `SET ticketId userId NX EX 600` in Redis (atomic — only one client wins)
2. On purchase complete → manually release lock + flip DB ticket to `booked`
3. On TTL expiry → Redis auto-releases the lock

**Multi-ticket booking:** acquire locks sequentially; if one fails, release the rest. **Atomic multi-lock** possible with a Lua script if tickets hash to the same node.

#### Challenges

**Read-path complexity:** Reservations live in Redis, not DB. Seat map needs to show reserved seats. Two options:
- Maintain `event:{eventId}:reserved` Redis Set, queried alongside reads (one extra round-trip)
- Write-through `reserved` status to DB; Redis TTL is source of truth; periodic sweep cleans DB

**Failure handling:** If Redis goes down → degraded UX, but **no double-booking** because Postgres still uses OCC / row-level locking as final safety net. Better outcome than "all tickets appear unavailable" (cron failure mode).

**TTL expiring during payment:** Rare race — A's TTL expires at minute 10, payment completes at minute 11, B grabbed it in between. OCC at commit time means **only one DB transaction succeeds**. Loser gets **automatic Stripe refund**. Mitigate by: generous TTL + extending lock on payment initiation.

### Final booking flow (with Redis lock)

1. User selects seat → `POST /bookings` with `ticketId`
2. Booking Service acquires Redis lock (10-min TTL)
3. Booking Service writes Booking row with `status = in-progress`
4. Returns `bookingId` → client goes to payment page
   - If user abandons → Redis auto-releases lock after 10 min
5. Client tokenizes card via **Stripe.js** (our server never sees raw card numbers — PCI compliance)
6. Client sends payment token + `bookingId` → our server creates Stripe PaymentIntent
7. Stripe processes → webhook back to us
8. **Webhook handler** (must be **idempotent** — Stripe retries on failure; key on `bookingId`):
   - DB transaction: flip Ticket to `sold`, flip Booking to `confirmed`
9. Done.

🔑 **Idempotency on webhooks** is non-negotiable — Stripe retries.

---

## 7. Deep Dive 2 — Scale View API to 10M Concurrent Users

When tickets drop, the same event page gets refreshed millions of times.

### 🔥 Great — Caching + Load Balancing + Horizontal Scaling

#### Caching (Redis / Memcached)
- Cache `eventId → eventObject` (event details, venue info, performer bio)
- High read rate, low update frequency = perfect cache target
- **Read-through:** cache miss → DB read → populate cache
- **Cache invalidation:**
  - **DB triggers** notify cache on event/venue updates
  - **TTL** — long for static (venue), short for volatile (availability)

#### Load balancing
- Round Robin / Least Connections across service instances
- Standard — mention but don't dwell

#### Horizontal scaling
- **Event Service is stateless** → easy horizontal scale
- Auto-scaling group + load balancer

⚠️ Cache–DB consistency for things that *do* change (event date change, lineup change) is the main challenge. Triggers + TTL combo handles it.

---

## 8. Deep Dive 3 — UX During High-Demand Events (Real-Time Seat Map)

For popular events, seat map goes stale in seconds. Users keep clicking taken seats.

### 👍 Good — SSE (Server-Sent Events) for live seat-map updates
- Server pushes `seat_taken` events to all clients viewing that event
- **Unidirectional** (server → client) which is all we need
- Works well for **moderately** popular events
- ⚠️ For Taylor Swift: seat map fills instantly → disorienting, useless

### 🔥 Great — Virtual Waiting Queue (the real solution)

🔑 **Senior vs Staff insight:** Sometimes the best engineering answer is a **product decision**, not a technical one.

#### How it works
1. Admin-enables queue for hot events
2. User requests booking page → placed in queue
3. Persistent connection (SSE or WebSocket) for position updates
4. Queue backed by **Redis sorted set** (timestamp = score)
5. Periodically dequeue users → notify them via connection that they can proceed
6. Mark user as "admitted" in `admitted:{eventId}` Redis set (with TTL)
7. **Booking Service rejects** any reservation request from users not in the admitted set

#### Why this wins
- Caps load on Booking Service to a manageable rate
- Users get **clear feedback** (position, ETA) instead of frantic clicking
- System stays responsive end-to-end
- Bonus: prevents bot scalpers from flooding directly

⚠️ Long wait times still frustrate — mitigate with accurate ETAs + position updates.

### SSE vs WebSocket choice
SSE is simpler (server → client only). WebSocket if you need bidirectional. For queue updates → SSE.

---

## 9. Deep Dive 4 — Fast Search (<500ms)

### The problem
```sql
SELECT * FROM Events
WHERE name LIKE '%Taylor%' OR description LIKE '%Taylor%'
```
Wildcard at the start = full table scan. Dies at scale.

### 👍 Good — Indexes + query optimization
- B-tree indexes on frequently-filtered columns (name, date, performer, venue)
- `EXPLAIN` queries, avoid `SELECT *`, use `LIMIT`, prefer `UNION` over `OR`
- ⚠️ **Standard indexes don't help with `LIKE '%foo%'`** — wildcard prefix kills them
- ⚠️ Indexes slow writes + use storage

### 🔥 Great — Full-text indexes in the DB
- Postgres: `tsvector` + **GIN index**
- MySQL: built-in FULLTEXT indexes
- Much faster than `LIKE` for word-based search
- ⚠️ Don't use Lucene under the hood (that's Elasticsearch's edge)

### 🔥 Great — Elasticsearch (or similar full-text search engine)
- Built on **inverted indexes** — map word → docs containing it; instant lookups
- Excels at full-text search, complex queries, high traffic
- **Fuzzy search** — handles typos ("Tayler Swift" → "Taylor Swift") — very hard in SQL
- Keep ES in sync via **Change Data Capture (CDC)** from Postgres
- ⚠️ Operational cost: another cluster to run; sync complexity

🔑 **Decision rubric:** small data + simple needs → DB full-text. Large catalog + fuzzy + complex queries → Elasticsearch.

---

## 10. Deep Dive 5 — Cache Search Queries

Frequent repeated queries (Taylor Swift, Lakers, etc.) shouldn't keep hitting Elasticsearch.

### 👍 Good — App-level cache (Redis / Memcached)
```
key:   search:keyword=Taylor Swift&start=...&end=...
value: [event1, event2, event3]
ttl:   24 hours
```
- Construct cache keys from query params
- TTL for freshness
- ⚠️ Invalidation tricky (new events make cached results stale)

### 🔥 Great — Elasticsearch built-in caches + CDN
- ES has **shard-level query cache** (filter results) + **shard-level request cache** (full response — great for aggregations)
- **CDN (CloudFront)** for geographically distributed search results
  - Only works if results are **non-personalized**
- Adaptive caching — system learns which queries are hottest

⚠️ Real-time consistency hard with CDN cache → invalidation when new events publish.

---

## 11. Key Numbers Cheatsheet

| Metric | Number |
|---|---|
| Read:write ratio | 100:1 |
| Search latency target | < 500 ms |
| Concurrent users on hot event | 10M |
| Reservation TTL | ~10 min |
| Cache TTL (static venue/event data) | long (hours) |
| Cache TTL (availability) | short (seconds) |
| Booking ticket states | available / reserved / booked |

---

## 12. Tradeoffs Cheatsheet

| Decision | Chosen | Why | Tradeoff |
|---|---|---|---|
| DB | **Postgres** (shared across services) | ACID transactions for booking; tightly coupled data | Violates "DB per service" dogma — acceptable |
| Reservation mechanism | **Redis distributed lock with TTL** | Auto-expiration; fast under high concurrency | Read path complexity; Redis as dependency |
| Reservation fallback | **OCC/row locks in Postgres** | Even if Redis fails, no double-booking | Worse UX, but consistency preserved |
| Status model | **Implicit (status + expiration)** if not using Redis | Cron not correctness-critical | Slightly slower reads |
| Payment | **Stripe + tokenization (Stripe.js)** | PCI compliance, never see raw cards | External dependency |
| Webhook handler | **Idempotent on bookingId** | Stripe retries on failure | Must check current state before update |
| View scaling | **Cache + LB + horizontal scaling** | Static data, read-heavy | Cache invalidation on rare updates |
| Hot-event UX | **Virtual waiting queue** (admin-enabled) | Caps system load, clear user feedback | Queue UX still frustrating; needs accurate ETAs |
| Search | **Elasticsearch + CDC from Postgres** | Inverted index, fuzzy, complex queries | Operational cost; sync complexity |
| Search caching | **Redis app cache + ES built-in + CDN** | Multi-layer; serve repeat queries fast | Invalidation hard with new events |
| Real-time updates | **SSE for normal events; WebSocket for queue** | Unidirectional is enough | Need to handle reconnects |

---

## 13. Common Pitfalls & Gotchas

- ⚠️ **Long-running DB transactions** for reservations — kills DB under load
- ⚠️ **Cron-only reservation cleanup** — system breaks if cron lags or fails
- ⚠️ **Forgetting webhook idempotency** — Stripe retries duplicate state changes
- ⚠️ **Letting raw card numbers touch your server** — PCI nightmare. Tokenize on client with Stripe.js.
- ⚠️ **`LIKE '%foo%'`** with leading wildcard — no index can help; needs full-text
- ⚠️ **One global system for hot events** — virtual queue + horizontal scale; don't try to "muscle through" 10M concurrent
- ⚠️ **Treating Redis lock failure as "double booking risk"** — OCC at DB layer is the real safety net
- ⚠️ **Caching everything** — invalidate on event updates; use DB triggers + TTL
- ⚠️ **Skipping the reservation step** — bad UX; users hate finding out their ticket is gone after typing card info
- ⚠️ **"Database per service" dogma** — explain the tradeoff; sometimes one DB is correct
- ⚠️ **Forgetting CDC** for ES sync — ES will drift from Postgres without it

---

## 14. Level Expectations

### 🟢 Mid-level (80% breadth / 20% depth)
- Clear APIs + entity model
- Working HLD for view + book + search
- Solve "no double booking" with at least the **"Good Solution"** (status + cron)
- Don't need to drive every deep dive; interviewer will probe
- Surface familiarity with API Gateway, caching, etc.

### 🟡 Senior (60% breadth / 40% depth)
- Speed through HLD
- Reach **distributed lock with TTL (Redis)** or implicit-status great solution
- Use **Elasticsearch** for search
- Discuss sharding, replication, scaling strategies
- Articulate tradeoffs proactively
- Discuss hot-event handling with reasonable detail

### 🔴 Staff+ (40% breadth / 60% depth)
- Breeze past basics
- Deep-dive **2–3 areas**: contention strategies, hot-event UX, search infra
- Propose **product-level solutions** (virtual queue) where they're better than technical ones
- Real-world insight: PCI tokenization, Stripe webhook idempotency, OCC as safety net
- Anticipate edge cases: TTL expiration during payment, multi-ticket atomic reservation, Redis failover
- Treat interviewer as peer; share novel insights

---

## 🎯 30-Second Recall (eve-of-interview)

> **Ticketmaster.** Mixed CAP: **availability for view/search**, **consistency for booking**. Postgres holds Events/Venues/Performers/Tickets/Bookings (shared DB is fine — ACID matters more than dogma). View scales with **Redis cache + LB + horizontal scaling on stateless Event Service**. Search starts with `LIKE`, evolves to **Postgres full-text** → **Elasticsearch + CDC** (inverted index, fuzzy matching), then **multi-layer cache** (app + ES built-in + CDN). Booking is the heart: **Redis distributed lock with TTL** for reservations (10 min) — auto-expiration solves the cron problem; **OCC at Postgres** is the safety net. Alternative: **implicit status** (available OR reserved-but-expired) in short transactions. **Stripe.js tokenizes cards client-side** (PCI), webhook back → **idempotent handler** on `bookingId` → flip Ticket to `sold`, Booking to `confirmed`. For 10M-user Taylor Swift drops: **virtual waiting queue** (Redis sorted set; SSE for position updates; admitted set gates Booking Service). Real-time seat map via **SSE**. Watch out: TTL expiring during payment (auto-refund), webhook retries, Redis failover.

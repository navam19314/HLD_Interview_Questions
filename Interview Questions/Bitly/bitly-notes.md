# 🔗 Bitly (URL Shortener) — System Design Revision Notes

> Based on **Hello Interview's Delivery Framework**
> Time budget: **5min Requirements → 2min Entities → 5min API → 15min HLD → 10min Deep Dives**

---

## 0. Problem Snapshot

**Bit.ly** converts long URLs into short, shareable links and redirects users back to the original.

🔑 **Single most important insight:** This is an **extremely read-heavy** system (~1000 reads : 1 write). Every design decision should reflect that asymmetry.

---

## 1. Functional Requirements (~5 min — first half)

### Core (in scope)
1. Users can submit a long URL → receive a shortened version
   - Optional: **custom alias** (e.g., `short.ly/my-link`)
   - Optional: **expiration date**
2. Users can access the original URL via the short URL (redirect)

### Below the line (out of scope — but mention you considered)
- User authentication / account management
- Click analytics (counts, geo, etc.)

⚠️ **Trap:** Don't bloat scope. Top 3 features only. Long lists hurt you.

---

## 2. Non-Functional Requirements

| # | Requirement | Quantified Target |
|---|---|---|
| 1 | **Uniqueness** of short codes (each maps to exactly one long URL) | Hard constraint |
| 2 | **Low latency** redirects | < 100 ms |
| 3 | **High availability** (availability > consistency per CAP) | 99.99% |
| 4 | **Scale** | 1B URLs, 100M DAU |

🔑 **Mention explicitly:** Read:write ratio is heavily skewed → caching strategy, DB choice, and overall architecture all hinge on this.

⚠️ Skip upfront back-of-the-envelope math unless it directly drives a decision. Do math *when needed* during HLD/deep dives.

---

## 3. Core Entities (~2 min)

Just a bulleted list — don't sweat columns yet.

1. **Original URL** — the long one
2. **Short URL** — short code → long URL mapping
3. **User** — who created it

> Tell interviewer: *"I'll flesh out columns when we hit the data model in HLD."*

---

## 4. API Design (~5 min)

REST + correct HTTP verb is 90% of the answer here.

### Shorten a URL
```http
POST /urls
{
  "long_url": "https://www.example.com/some/very/long/url",
  "custom_alias": "optional_custom_alias",
  "expiration_date": "optional_expiration_date"
}
→
{
  "short_url": "http://short.ly/abc123"
}
```
**Why POST?** Creating a new resource (new DB entry).

### Redirect
```http
GET /{short_code}
→ HTTP 302 Found
  Location: https://www.original-long-url.com
```
**Why GET?** Reading an existing resource.

🔑 **Never derive current user from request body** — always pull from the auth token.

---

## 5. High-Level Design (~10–15 min)

### Minimum components to start
1. **Client** — web/mobile
2. **Primary Server** — business logic, validation, short-code generation
3. **Database** — stores `short_code → long_url, alias, expiration, created_at`

---

### Flow 1 — Shorten a URL (`POST /urls`)

1. Server **validates** long URL format (e.g., `is-url` library or regex).
2. *(Optional dedup check — skipped by most shorteners because different users want separate aliases / expirations / analytics.)*
3. **Generate short code** — treat as magic function for now (deep-dived later).
   - If `custom_alias` provided → check uniqueness, use directly.
   - ⚠️ **Namespace collision risk:** custom aliases could collide with future counter-generated codes.
   - **Mitigation:** prefix generated codes with a char custom aliases can't use, OR use separate namespaces.
4. **Insert** `{short_code, long_url, expiration, created_at}` into DB.
5. Return `short_url` to client.

---

### Flow 2 — Redirect (`GET /{short_code}`)

1. Browser → GET `/abc123` to your server (you own `short.ly`).
2. Server **looks up** `abc123` in DB.
3. Check expiration → if expired, return **`410 Gone`**.
4. Return redirect response with `Location: <long_url>`.

🧹 **Cleanup:** background job to delete expired rows; **cache TTL ≤ URL expiration** so stale entries auto-evict.

#### 🔑 301 vs 302 — pick **302**

| Code | Meaning | Browser cache? | Hits server again? |
|------|---------|---|---|
| **301** Moved Permanently | Permanent redirect | ✅ Aggressively | ❌ No (after first hit) |
| **302** Found | Temporary redirect | ❌ No | ✅ Every time |

**Why 302 wins for Bitly:**
- More control — can update/expire/delete links later
- Prevents browser caching that breaks updates
- Enables click tracking (every click hits us)

---

## 6. Deep Dive 1 — Generating Unique Short URLs

**Constraints to balance:**
1. **Unique** (no collisions)
2. **Short** (it's a URL *shortener*)
3. **Efficient** to generate

---

### ❌ Bad: Long URL Prefix
Take first N chars of the long URL.
**Why it fails:** Two URLs with the same prefix → same short code → ambiguous redirects. Fails constraint #1.

---

### ✅ Great: Hash Function + Base62

```python
canonical_url = canonicalize(input_url)   # lowercase host, strip default ports, etc.
hash_code = SHA256(canonical_url)
short_code = base62_encode(hash_code)[:8]
```

**Pros:**
- Deterministic → same URL always → same short code (free dedup)
- High entropy → low collision probability
- No coordination needed across servers

**Why base62 (not base64)?**
- 62 chars = `[a–z, A–Z, 0–9]`
- Base64 includes `+` (interpreted as space in query strings) and `/` (URL path separator) → unsafe in URLs

**Collision math:**
- 8 chars × base62 = 62⁸ ≈ **218 trillion** possible codes
- Probability of next collision = n / |S| where n = codes used

**Challenges & fixes:**
- ⚠️ Collisions still possible at scale → **UNIQUE constraint** + **bounded retries (3–5)** with a random salt
- If you need multiple codes per URL OR want to prevent guessability → add a **secret salt / HMAC**

---

### ✅ Great: Counter + Base62 (Hello Interview's preferred)

Increment a global counter, base62-encode it.

**Why Redis for the counter?**
- 🔑 **Single-threaded** → no race conditions
- **Atomic `INCR`** → two simultaneous calls always return different values (e.g., 1000 and 1001)
- Extremely fast (100k+ ops/sec)

**Pros:**
- Guaranteed unique — no collision retries
- Computationally cheap
- Short codes stay compact for a long time

**Length math (must memorize):**
| URLs stored | Base62 length |
|---|---|
| 1B (10⁹) | **6 chars** (e.g., `15ftgG`) |
| 62⁶ ≈ 56B | still 6 chars |
| 62⁷ ≈ 3.5T | 7 chars |

**Challenges:**
- ⚠️ **Distributed coordination** — need a single source of truth (solved with central Redis — see Deep Dive 3)
- ⚠️ **Predictable codes** → URL enumeration attacks. Mitigation: **XOR with secret key** before base62 encoding (reversible, but obfuscated)

---

## 7. Deep Dive 2 — Fast Redirects (< 100ms)

The redirect path is the hot path. Without optimization → **full table scan** = death.

---

### 👍 Good: B-Tree Index / Primary Key on `short_code`

- Most relational DBs default to B-tree → **O(log n)** lookups
- Make `short_code` the **primary key** → free index + uniqueness enforced

**Challenge:** Even with indexing, a single DB can choke on the read volume.

**Math for scale:**
```
100M DAU × 5 redirects/day = 500M redirects/day
500M / 86,400s ≈ 5,787 RPS (average)
× 100× peak multiplier ≈ 600k RPS
```
A single Postgres node can't comfortably serve 600k RPS from disk.

---

### 🔥 Great: In-Memory Cache (Redis / Memcached)

Cache `short_code → long_url` mappings in memory between server and DB.

**Latency comparison (memorize):**
| Storage | Access time | Ops/sec |
|---|---|---|
| **Memory** | ~100 ns (0.0001 ms) | Millions |
| **SSD** | ~0.1 ms | ~100k IOPS |
| **HDD** | ~10 ms | ~100–200 IOPS |

🔑 Memory is **~1000× faster than SSD**, **~100,000× faster than HDD**.

**Pattern:** check cache → hit returns immediately, miss falls through to DB → populate cache.

**Challenges:**
- Cache invalidation on expiration/update (minor — URLs rarely change)
- Cold start / cache warm-up
- Eviction policy → **LRU** is standard
- Memory cost vs hit rate tradeoff

---

### 🚀 Great: CDN + Edge Computing

Host short URL domain on a CDN with global PoPs. Use **Cloudflare Workers** or **AWS Lambda@Edge** to do the redirect *at the edge* — never hits origin.

**Pros:**
- Redirect happens closest to the user → near-zero latency for popular codes
- Origin server load drops massively

**Cons / tradeoffs:**
- Cache invalidation across PoPs is hard
- Edge function limits (memory, exec time, libraries)
- Higher cost
- Debugging distributed edge is painful

**When to use:** Global product, high traffic, latency-sensitive.

---

## 8. Deep Dive 3 — Scaling to 1B URLs + 100M DAU

### Storage math (must do live)
```
Per row: short_code (8B) + long_url (100B) + created (8B)
       + alias (100B) + expiration (8B) + metadata buffer
     ≈ 200B → round up to 500B
1B rows × 500B = 500 GB
```
🔑 **500 GB fits comfortably on a single modern Postgres SSD instance.** No need to shard yet.

### Write rate
```
~100k new URLs/day = ~1 write/sec → trivial.
```

### DB choice
- Reads offloaded to cache → DB just needs to be reliable
- **Postgres / MySQL / DynamoDB** — all fine. Pick what you know.
- Default: **Postgres**.

### High Availability of DB
1. **Replication** — read replicas + automatic failover
2. **Backups** — periodic snapshots to separate location

---

### Scaling the Primary Server

🔑 **Split into two microservices** (asymmetric workload):
- **Read Service** → redirects (cache-first, scales horizontally)
- **Write Service** → URL creation

Both **horizontally scale** independently behind a load balancer.

---

### ⚠️ Big problem: Distributed Counter

If you horizontally scale the Write Service, **each instance needs the same global counter** to avoid collisions.

#### Solution: Centralized Redis Counter
- All Write Service instances → call Redis `INCR` for next value
- Redis single-threaded + atomic → guaranteed unique
- Network overhead is negligible

#### Optimization: Counter Batching
1. Each Write Service requests **batch of 1000** counter values from Redis
2. Redis atomically does `INCRBY 1000`, returns batch start
3. Instance uses values locally
4. When exhausted → request another batch

🔑 **Reduces Redis load 1000×** while preserving uniqueness.

#### Counter HA
- **Redis Sentinel** or **Redis Cluster** with auto-failover
- A single Redis instance handles 100k+ ops/sec → plenty
- If Redis fails before replicating latest counter → lose a few values; **uniqueness preserved** (we don't need continuity)
- 🔑 **UNIQUE constraint on `short_code` is the ultimate safety net**

#### Multi-region
- Allocate **disjoint counter ranges** per region (Region A: 0–1B, Region B: 1B–2B)
- No cross-region coordination needed
- Local writes, globally distributed reads via cache/CDN

---

## 9. Key Numbers Cheatsheet

| Metric | Number |
|---|---|
| DAU | 100M |
| Total URLs | 1B |
| Redirects/day | 500M |
| Avg RPS | ~5,800 |
| Peak RPS (100×) | ~600k |
| Storage | 500 GB |
| Writes/sec | ~1 |
| Short code length @1B URLs | 6 chars |
| Code space @8 chars base62 | 218 trillion |
| Memory access | ~100 ns |
| SSD access | ~0.1 ms |
| HDD access | ~10 ms |

---

## 10. Tradeoffs Cheatsheet

| Decision | Chosen | Why | Tradeoff |
|---|---|---|---|
| Short code gen | **Counter + base62** | Guaranteed unique, compact | Needs centralized counter; predictable |
| Encoding | **Base62** | URL-safe (no `+`, `/`) | Slightly smaller alphabet than base64 |
| Redirect code | **302** | Control, no caching, trackable | More server load vs 301 |
| Cache | **Redis** | 1000× faster than disk, ops/sec | Invalidation, cost, warm-up |
| DB | **Postgres** | Boring, reliable, indexes | Single-node ceiling (mitigated by cache) |
| Scaling writes | **Centralized Redis counter + batching** | Globally unique, fast | SPOF — mitigated by Sentinel/Cluster |
| Multi-region | **Disjoint counter ranges** | Zero coordination | Wastes range if region underused |
| Read scaling | **Cache + read replicas + CDN edge** | Sub-100ms globally | Cost, complexity |

---

## 11. Common Pitfalls & Gotchas

- ⚠️ **Base64 in URLs breaks** — `/` is a path separator, `+` becomes space in query strings. Use **base62**.
- ⚠️ **301 caches in browser** → can't update/delete short URLs. Always **302**.
- ⚠️ **Custom alias namespace** can collide with counter-generated codes → use prefix or separate namespace.
- ⚠️ **Predictable counter-based codes** → enumeration attacks. Use XOR-with-secret if security matters.
- ⚠️ **Sequential counter in distributed system** → can't just use auto-increment per server. Centralize.
- ⚠️ **Hash function dedup** is a feature OR a bug depending on requirements. Confirm with interviewer.
- ⚠️ **Cache TTL** should ≤ URL expiration time, else stale redirects.
- ⚠️ **Skipping capacity math early is fine** — but do it when it changes your design (e.g., sharding decision).

---

## 12. Level Expectations

### 🟢 Mid-level
- Working high-level design (shorten + redirect flow)
- Recognize short codes need uniqueness
- Propose **one** code-gen approach (hash or counter)
- Understand 302 redirect (even if didn't know the code)
- With prompting → recognize cache helps reads

### 🟡 Senior
- **Drive** the conversation, proactively flag key challenges
- Articulate tradeoffs: hashing (collisions) vs counter (coordination)
- Discuss caching strategy + invalidation in detail
- Justify DB choice
- Recognize read/write service split
- Understand counter coordination via Redis

### 🔴 Staff+
- Read-heavy framing from the start
- **Proactively** discuss multi-region, counter range allocation, Redis failover
- Security implications of predictable codes + mitigations
- Custom alias collision prevention
- URL expiration cleanup strategies
- Show product/ops thinking, not just textbook solution

---

## 🎯 30-Second Recall (eve-of-interview)

> Read-heavy URL shortener. **Counter + base62** for unique codes (Redis atomic `INCR`, batching for scale, disjoint ranges per region). **B-tree index + Redis cache + CDN edge** for sub-100ms redirects. **Read/write service split**. **Postgres** for 500GB of mappings. **302** redirect (not 301). **UNIQUE constraint** is the safety net. Watch out for: base64-vs-base62, custom alias namespace, predictable enumeration, cache TTL ≤ URL expiration.

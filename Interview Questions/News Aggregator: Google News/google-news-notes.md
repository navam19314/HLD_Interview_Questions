# 📰 Google News (News Aggregator) — System Design Revision Notes

> Based on **Hello Interview's Delivery Framework**
> Time budget: **5min Requirements → 2min Entities → 5min API → 15min HLD → 10min Deep Dives**

---

## 0. Problem Snapshot

**Google News** = aggregator that pulls articles from thousands of publishers worldwide and displays them in an infinite-scroll feed. **Doesn't host content** — clicks redirect to the publisher's site.

🔑 **Single most important insight:** This is an **extreme read-heavy** system with one structural advantage that makes scaling tractable: **news consumption is regional**. Build many smaller regional systems, not one giant global one.

---

## 1. Functional Requirements (~5 min — first half)

### Core (in scope)
1. View **aggregated feed** of articles from thousands of publishers worldwide
2. **Infinite scroll** through the feed
3. Click an article → redirect to publisher's site (we don't host content)

### Below the line (out of scope)
- Feed customization based on interests *(covered in bonus deep dive)*
- Save articles for later
- Share to social media

---

## 2. Non-Functional Requirements

| # | Requirement | Quantified Target |
|---|---|---|
| 1 | **Availability > Consistency** (CAP) | Stale is OK; empty feed is not |
| 2 | **Scale** | 100M DAU, spikes to **500M** |
| 3 | **Low latency** feed loads | **< 200 ms** |

### Below the line
- Privacy / security
- Breaking-news spike handling *(actually a deep dive)*
- Observability
- Resilience to publisher API failures

🔑 **Read:write asymmetry is extreme** — billions of feed reads vs ~thousands of article writes per day. Drives caching strategy hard.

---

## 3. Core Entities (~2 min)

1. **Article** — `id, title, summary, thumbnail URL, publish date, publisher ID, region, media URLs`
2. **Publisher** — `id, name, URL, feed URL, region`. Origin of content.
3. **User** — `id, region` (region inferred from IP or explicit). Even anonymous users are tracked at this level.

🔑 Don't enumerate every column — explain **relationships** and **purpose**.

---

## 4. API Design (~5 min)

Only one user-facing endpoint needed.

### Get feed
```http
GET /feed?page={page}&limit={limit}&region={region}
→ Article[]
```

⚠️ Start with offset-based pagination. **Will evolve to cursor-based** in Deep Dive 1.

### Article click → no API
Browser navigates directly to the `url` field on the article. Real Google News would route through a tracking endpoint (`GET /article/{id}` → log click → 302 redirect), but that's out of scope.

---

## 5. High-Level Design (~10–15 min)

Two distinct subsystems with **completely different scaling needs**:
- **Ingestion** (write-heavy, batch, background)
- **Serving** (read-heavy, real-time, user-facing)

🔑 **Keep these separated** — different scale profiles, update frequencies, and operational needs.

---

### Flow 1 — Ingestion: Collect from Publishers

**Data Collection Service** (background workers):
1. Queries DB for list of publishers + their RSS feed URLs
2. Polls each publisher's RSS feed every **3–6 hours** (frequency tuned per publisher)
3. Parses XML → extracts article content + metadata
4. Downloads media (images) → stores in **Object Storage** (S3)
5. Writes article rows to **Database** with media URLs pointing to S3

### Why RSS?
- Simple XML format over HTTP — just a GET request
- Standardized, lightweight, widely supported by publishers
- Good fit for aggregators

### Why download thumbnails ourselves (don't hotlink)?
- ✅ Fast & reliable — don't depend on publisher servers
- ✅ Standardize size/quality for consistent UX
- ✅ Publishers may rotate URLs or 404 over time

---

### Flow 2 — Serving: User Feed Requests

**Feed Service** (user-facing):
1. Client → `GET /feed?region=US&limit=20` → **API Gateway** (auth, rate limit, validation)
2. API Gateway → **Feed Service**
3. Feed Service queries DB for recent articles in user's region, ordered by publish date
4. Returns articles (with S3-backed thumbnail URLs) to client

### Components
| Component | Role |
|---|---|
| **Data Collection Service** | Background ingestion from RSS feeds |
| **Database** | Articles, publishers, metadata |
| **Object Storage (S3)** | Article thumbnails |
| **API Gateway** | Routing, auth, rate limiting |
| **Feed Service** | Serves user feed requests |
| **Client** | Web/mobile app |

---

### Flow 3 — Infinite Scroll (initial naive version)

Offset-based pagination — easy to start, broken at scale.

1. First load → `GET /feed?region=US&limit=20&page=1`
2. Response includes articles + `total_pages`, `current_page`
3. As user scrolls → `page=2`, `page=3`, ...
4. DB query uses `OFFSET (page-1) * limit LIMIT 20`

⚠️ **Big problems** with offset-based pagination — fixed in Deep Dive 1.

---

### Flow 4 — Article Click

Browser handles this natively. We store the publisher's URL on the article; clicking just navigates there. No infra needed (unless you add click tracking).

---

## 6. Deep Dive 1 — Pagination Consistency & Efficiency

### Why offset-based pagination breaks

During a busy news day, 50–100 new articles publish per hour. If a user is on page 2 and new articles get prepended to page 1:
- ⚠️ Articles **shift down** → user sees the same article on page 3 that was already on page 2 (**duplicates**)
- ⚠️ Or, articles **shift off** the bottom of page 2 before user requests page 3 (**skipped content**)

Also, large `OFFSET` queries are slow (the DB has to count through skipped rows).

### Solution — Cursor-based pagination
- Use a **monotonically increasing article ID** as the cursor (or a `(publish_time, id)` composite cursor)
- Client sends `cursor=<last_article_id>` instead of `page=N`
- Query: `WHERE id < cursor ORDER BY id DESC LIMIT 20`
- New articles inserted *above* the cursor don't disturb the user's scroll position
- DB query is O(log n) via index, regardless of how deep the user scrolls

🔑 **Cursor-based pagination = the standard fix for "live" feeds where content gets prepended.**

---

## 7. Deep Dive 2 — Low Latency (<200ms) Feed Requests

### The pressure
- 100M DAU × 5–10 refreshes/day = **500M–1B feed requests/day**
- Even with indexing, raw DB queries can't hit <200ms at this scale

### Solution — Aggressive caching of pre-computed regional feeds

🔑 **Pattern: Scaling Reads.** News reads massively outweigh news writes — pre-compute the answer once, serve it billions of times.

#### Cache structure
- **Redis** key per region: `feed:US`, `feed:UK`, `feed:IN`, ...
- Value = list of ~recent N articles (e.g., 2,000 per region)
- Cache updated when ingestion completes a new batch

#### Request path
1. Feed Service receives request
2. Look up `feed:{region}` in Redis
3. Slice based on cursor → return
4. DB only hit on cache miss or cache rebuild

#### Why this is so effective
- ~2,000 articles per region fits easily in a single Redis instance (small dataset)
- Read throughput per Redis instance: ~100k ops/sec
- Same regional feed serves *every* user in that region

⚠️ **Write path:** ingestion updates Redis after writing to DB. Use cache-aside or write-through. Short TTL (~minutes) as a safety net.

---

## 8. Deep Dive 3 — Articles in Feeds Within 30 Minutes of Publication

### The problem
Polling RSS every 3–6 hours is way too slow. Breaking news → users see it first on Twitter/social, then come to Google News and find stale content.

### Hybrid Ingestion Approach

🔑 **Different publishers, different mechanisms** — pick the right tool per publisher.

| Mechanism | Use For | Latency |
|---|---|---|
| **Webhooks** | Premium real-time partners (top publishers) | Seconds |
| **Frequent RSS polling** | Cooperative publishers with feeds | Minutes |
| **Web scraping** | Publishers without RSS/webhooks | Variable |

### Webhooks (push, not pull)
- Publisher calls our endpoint when they publish
- Lowest latency, but requires publisher cooperation
- Reserved for top-tier partnerships (worthwhile because of traffic value)

### Frequent RSS polling
- Drop interval from 3–6 hours to **5–10 minutes** for important publishers
- **Adaptive polling**: monitor each publisher's typical publish rate and tune frequency

### Web scraping fallback
- For publishers with no RSS, scrape their site directly
- Respect `robots.txt`, rate limits, and politeness rules
- More fragile and expensive — last resort

### Interview tip
🔑 This is great back-and-forth material:
- "Can I black-box the ingestion pipeline?" (mid-level: yes; senior+: probably not)
- "Do publishers maintain RSS feeds?"
- "Can we assume premium publishers will implement webhooks for us?"

---

## 9. Deep Dive 4 — Media (Thumbnails) Efficiency

### What we actually need
Only **thumbnails** — we don't host full articles. Simpler than Dropbox-scale media.

### Why we copy publisher images instead of hotlinking
- Publisher images can be slow / unavailable / URL-rotated
- Standardize sizes & formats for consistent UX

### Solution — S3 + CloudFront CDN + multiple sizes

1. **Ingestion** downloads source image → resizes to multiple thumbnail sizes (e.g., 200×200, 400×400, retina variants) → stores all in **S3**
2. Article record stores S3 URLs
3. **CDN (CloudFront)** caches thumbnails at edge PoPs globally
4. Clients fetch from nearest edge → fast worldwide

### Cost optimization
- Set long `Cache-Control` TTLs (thumbnails rarely change)
- Lazy-generate large sizes only if accessed
- Compress aggressively (WebP / AVIF)

---

## 10. Deep Dive 5 — Traffic Spikes During Breaking News

### The challenge
Normal 100M DAU can spike to **10M concurrent users** when major news breaks (elections, disasters, celebrity events).

🔑 **The regional advantage** — news is mostly regional. Don't build one global system; build **N regional systems** that each handle their local spike.

### Per-region scaling — per layer

#### Feed Service (stateless app layer)
- Single instance: ~10k–100k concurrent connections
- **Solution**: **horizontal scaling with auto-scaling groups**
- Multiple instances behind a load balancer; cloud auto-scaler watches CPU/memory and provisions more
- Stateless → spin up/down in seconds, pay only for peak

#### Database
- Can't handle 10M concurrent reads on its own
- ✅ **Already mostly offloaded by the Redis cache** — most reads never reach the DB
- Cache becomes the actual bottleneck

#### Cache (Redis)
- Single Redis: ~**100k req/sec**
- Need: 10M concurrent → ~**100 Redis instances per stressed region** (in practice fewer per region)
- **Solution: read replicas**
  - Writes (new articles, cache updates) → master
  - Reads (feed requests) → load-balanced across replicas (round-robin / least-connections)
  - **Redis Sentinel** for automatic failover; replication lag < 200ms (acceptable for news)
- Each regional master only holds ~2,000 recent articles → no sharding complexity; just replicate

#### Math
```
10M concurrent ÷ 100k req/sec/Redis instance ≈ 100 Redis instances during peak
```
Per region this is less — scale up affected regions only during their spike, leave others at baseline.

🔑 **Spikes are localized**: an India election spike doesn't need US Redis to scale.

---

## 11. Bonus Deep Dive 6 — Category Feeds (Sports, Politics, Tech, ...)

### The problem
Regional feed (`feed:US`) isn't granular enough. Users want `Sports`, `Politics`, `Tech`, etc. — 25+ categories. During the Super Bowl, 10M users hit `feed:US:sports` at once.

### Solution — Pre-compute category feeds in cache
- Cache key: `feed:{region}:{category}` → `feed:US:sports`, `feed:US:politics`, ...
- Same architecture as regional feeds: pre-compute on ingestion, serve from Redis
- Article records get a `category` tag (assigned by publisher or by classifier on ingest)
- During ingestion, fan article out into all relevant category caches

### Tradeoff
- More cache keys → more memory
- But each key is small (~hundreds of articles) → totally manageable
- Categories with low traffic can have looser caching

---

## 12. Bonus Deep Dive 7 — Personalized Feeds

### The challenge
Same regional feed for everyone is bland. Users want feeds prioritizing their interests, trusted publishers, similar content to articles they've engaged with.

### Hybrid personalization — dynamic feed assembly

🔑 **The actual ranking is an ML model** — abstract it away; we're not in an MLE interview.

#### Architecture
1. **Pre-compute base feeds** (regional, categorical) — same as before
2. **User profile** stored separately (preferred categories, publishers, reading history)
3. At request time:
   - Pull a candidate pool from base regional/category caches
   - Apply user's personal **ranking/scoring function (ML model)** to re-order
   - Mix with editorial-importance / trending boosts
4. Return top N to user

#### Why this still hits <200ms
- Candidate pool comes from cache (fast)
- Scoring runs over a few hundred candidates, not the whole catalog
- ML model is a black box from the system-design perspective

### Tradeoff
Personal feed assembly per request adds CPU. Mitigations:
- Cache the personalized feed per-user with short TTL (~1 min)
- Run scoring asynchronously and cache result
- Use lightweight model for online ranking; heavier offline jobs for embeddings

---

## 13. Key Numbers Cheatsheet

| Metric | Number |
|---|---|
| DAU | 100M |
| Peak DAU | 500M |
| Feed loads/user/day | 5–10 |
| Total feed requests/day | 500M–1B |
| Concurrent users during breaking news | 10M |
| Articles per region (cached) | ~2,000 |
| RSS poll interval (default) | 3–6 hours |
| RSS poll interval (frequent) | 5–10 min |
| Latency target | < 200 ms |
| Redis ops/sec per instance | ~100k |
| Redis instances per peak region | ~100 |
| Redis replication lag | < 200 ms |
| Article freshness goal | < 30 min |

---

## 14. Tradeoffs Cheatsheet

| Decision | Chosen | Why | Tradeoff |
|---|---|---|---|
| Pagination | **Cursor-based (monotonic ID)** | No duplicates, no skips on live feed; O(log n) query | Less convenient for "jump to page N" |
| Feed serving | **Pre-computed regional feeds in Redis** | Same answer reused billions of times | Cache invalidation on new article publish |
| Thumbnail hosting | **Copy to S3 + CDN** | Fast, reliable, standardized | Storage + bandwidth cost |
| Ingestion | **Hybrid (webhook + polling + scrape)** | Right tool per publisher | Operational complexity |
| Geographic scope | **Regional deployments** | Localizes spikes; sub-50ms cache hits | More infra to operate |
| Scaling reads | **Redis read replicas + Sentinel** | 100k QPS per replica; auto failover | Slight replication lag (acceptable) |
| Feed Service | **Stateless + auto-scaling** | Quick elastic scale on spikes | Need good LB + health checks |
| Personalization | **Pre-computed candidates + online ranking** | Stays under 200ms | More complex than static feed |
| Categories | **Separate cache key per `region:category`** | Fast filtering | More memory, more keys to invalidate |
| Click tracking | **Out of scope** (optional 302 redirect) | Keeps base design simple | Misses analytics signal in MVP |

---

## 15. Common Pitfalls & Gotchas

- ⚠️ **Offset pagination on live feeds** → duplicates + skipped articles when new content arrives. Use cursor-based.
- ⚠️ **One global system** → spikes anywhere break everything. Go regional.
- ⚠️ **Hotlinking publisher images** → fragile, slow, breaks UX. Copy to S3.
- ⚠️ **Polling every 3–6 hours** → users see breaking news on Twitter first. Use webhooks for top publishers.
- ⚠️ **Querying DB on every feed request** → blows past 200ms. Cache pre-computed regional feeds.
- ⚠️ **Forgetting that ingestion ≠ serving** — they have opposite scaling profiles. Don't couple them.
- ⚠️ **Not exploiting regional locality** — the natural sharding key is free. Use it.
- ⚠️ **Trying to design the ranking ML model** in detail → not an MLE interview. Black-box it.
- ⚠️ **Treating "click tracking" as part of base design** → it's just a 302 redirect; mention it but don't dwell.
- ⚠️ **Asking "is this consistent?"** — explicitly chose availability > consistency. Stale articles are fine; an empty feed isn't.

---

## 16. Level Expectations (inferred)

### 🟢 Mid-level
- API + entity model
- Functional HLD covering ingestion + serving + click-through
- May black-box the ingestion pipeline if interviewer allows
- With prompting → recognize caching helps reads
- Recognize regional aspect

### 🟡 Senior
- Drive past basics quickly
- Proactively identify need for cursor-based pagination
- Pre-computed regional feed caching
- Hybrid ingestion (webhook + poll + scrape)
- Articulate tradeoffs (offset vs cursor; copy vs hotlink; one global vs regional)

### 🔴 Staff+
- Frame regional architecture upfront
- Deep-dive on adaptive polling, dedup across sources, rate-limit handling for publisher APIs
- Personalization architecture without conflating with ML model details
- Discuss publisher API rate limits, deduplication strategies (hash of `title + url`)
- Show product-thinking: why thumbnails not hotlinked; why 302 tracking exists

---

## 🎯 30-Second Recall (eve-of-interview)

> **News aggregator, doesn't host content.** 100M DAU → 1B feed reads/day. Read-heavy; **availability > consistency**. **Two subsystems**: ingestion (Data Collection Service polling RSS every 3–6 hrs, downloads thumbnails to S3) and serving (Feed Service behind API Gateway). Key insight: **news is regional** → build N regional deployments, each with its own Redis. **Pre-compute regional feeds in Redis** (`feed:US` etc.) — ~2,000 articles per region, ~100k QPS/replica, scale with read replicas + Sentinel. **Cursor-based pagination** (monotonic article ID) to handle continuous publishing. **Hybrid ingestion** for < 30-min freshness: webhooks (premium), frequent RSS (cooperative), scraping (last resort). **Thumbnails on S3 + CloudFront CDN** with multiple sizes. Spikes during breaking news: **stateless Feed Service auto-scales**, **Redis adds replicas in affected region only**. Bonus: category feeds = more cache keys (`feed:US:sports`); personalization = pre-compute candidates + online ML ranking.

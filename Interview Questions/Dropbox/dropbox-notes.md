# ☁️ Dropbox (File Storage & Sync) — System Design Revision Notes

> Based on **Hello Interview's Delivery Framework**
> Time budget: **5min Requirements → 2min Entities → 5min API → 15min HLD → 10min Deep Dives**

---

## 0. Problem Snapshot

**Dropbox** = cloud-based file storage that lets users upload, download, share, and **auto-sync** files across devices.

🔑 **Single most important insight:** The defining challenge is **handling large blobs** (up to 50GB) efficiently — upload, download, and sync all hinge on doing this well. **Blob Storage + presigned URLs + chunking** is the spine of the entire design.

---

## 1. Functional Requirements (~5 min — first half)

### Core (in scope)
1. Users can **upload** a file from any device
2. Users can **download** a file from any device
3. Users can **share** a file with other users (and view files shared with them)
4. Files **auto-sync** across devices

### Below the line (out of scope)
- File editing
- File previewing without downloading
- Designing blob storage itself (use S3 as a black box)

---

## 2. Non-Functional Requirements

| # | Requirement | Quantified Target |
|---|---|---|
| 1 | **High availability** (availability > consistency per CAP) | Eventually consistent is fine — OK if a Germany→US sync takes a few seconds |
| 2 | **Large file support** | Up to **50 GB** |
| 3 | **Security & reliability** | Recover from loss/corruption; encrypt in transit + at rest |
| 4 | **Low latency** for upload / download / sync | Optimize all three |

### Below the line
- Per-user storage limits
- Versioning
- Virus / malware scanning

🔑 **CAP framing:** A stock-trading app needs strict consistency. A file store doesn't — eventual consistency is fine.

---

## 3. Core Entities (~2 min)

1. **File** — the raw bytes
2. **FileMetadata** — name, size, mimeType, owner, status, chunks
3. **User** — the user

Tell interviewer: *"Columns to follow as we build the design."*

---

## 4. API Design (~5 min)

⚠️ **Tell the interviewer up front:** "These APIs will likely evolve as we work through tradeoffs in HLD."

### Upload (will evolve → presigned URL flow)
```http
POST /files
{ File, FileMetadata }
```

### Download (will evolve → presigned URL)
```http
GET /files/{fileId} → File & FileMetadata
```

### Share
```http
POST /files/{fileId}/share
{ "users": [User] }
```

### Sync — fetch changes since last poll
```http
GET /files/changes?since={timestamp} → ChangeEvent[]
```
Each `ChangeEvent` = `{ fileId, changeType (created/updated/deleted), updatedMetadata }`.

🔑 **User identity** always comes from auth header (JWT / session token) — **never** from request body.

---

## 5. High-Level Design (~10–15 min)

Build one functional requirement at a time.

---

### Flow 1 — Upload a File

**Two things to decide:** where do file *bytes* go, where does *metadata* go?

**Metadata DB:** **DynamoDB** (or Postgres — both work). Loosely structured, main query = "files by user".
```json
{
  "id": "123",
  "name": "file.txt",
  "size": 1000,
  "mimeType": "text/plain",
  "uploadedBy": "user1"
}
```

**File bytes:** evolve through 3 approaches:

#### ❌ Bad — Upload to single backend server
- Stores file on server's local disk.
- **Problems:** doesn't scale, not reliable, server failure = data loss.

#### 👍 Good — Backend uploads to Blob Storage (S3)
- Client → backend → S3.
- Scales + reliable.
- ⚠️ **Wastes bandwidth:** file uploaded twice (client→backend, backend→S3).

#### 🔥 Great — Client uploads **directly to S3** via Presigned URL

🔑 **The pattern: bypass app servers for data; keep them as control plane only.**

**Three-step flow:**
1. Client → backend: `POST /files/presigned-url` with metadata
   - Backend writes metadata row with `status = "uploading"`
   - Backend uses S3 SDK to generate a **presigned URL** (signed locally — no S3 call)
   - Returns URL to client
2. Client → S3: `PUT` the file directly to the presigned URL
3. S3 → backend: **S3 Event Notification** fires → backend flips metadata `status = "uploaded"`

**Why presigned URLs?**
- File never touches our backend → faster, cheaper
- URL grants scoped, time-limited upload permission to a specific S3 key
- Backend stays as control plane only

---

### Flow 2 — Download a File

Same three-tier evolution:

#### ❌ Bad — Download via backend (double-hop)
S3 → backend → client. Slow + expensive.

#### 👍 Good — Presigned S3 download URL
1. Client → backend: `GET /files/{fileId}/presigned-url`
2. Client downloads directly from S3 using URL.
- ⚠️ Still slow for globally distributed users (S3 is single-region).

#### 🔥 Great — Serve via CDN with signed URL
- CDN (CloudFront) caches the file at edge PoPs near users.
- Backend generates a **CDN signed URL** (not S3 directly).
- First request = CDN cache miss → fetches from S3, caches at edge.
- Subsequent requests = served from edge.

⚠️ **CDN cost trap:** Be strategic — set `cache-control` headers; only cache hot files. Don't waste money on rarely-accessed files.

---

### Flow 3 — Share Files

Need to fetch "files shared *with* me" fast.

#### ❌ Bad — Sharelist on file metadata
```json
{ ..., "sharelist": ["user2", "user3"] }
```
- Getting "files I own" = easy (index on `uploadedBy`).
- Getting "files shared with me" = ⚠️ **full table scan** of every `sharelist`. Terrible.

#### 👍 Good — Cache the inverse mapping
- Maintain a cache: `user1 → [fileId1, fileId2, ...]`
- ⚠️ Must keep cache in sync with the source-of-truth `sharelist`. Sync issues.

#### 🔥 Great — Normalize into a `SharedFiles` table

| userId (Partition Key) | fileId (Sort Key) |
|---|---|
| user1 | fileId1 |
| user1 | fileId2 |
| user2 | fileId3 |

- **Composite primary key** `(userId, fileId)`.
- Drop the `sharelist` field entirely — one source of truth.
- Query: `SELECT fileId WHERE userId = ?` — fast with index.
- Tradeoff: slight index lookup overhead vs. zero sync risk. Worth it.

---

### Flow 4 — Sync Across Devices

Two directions:

#### Local → Remote
Client-side **sync agent**:
1. Watches local Dropbox folder using OS-native FS events (**FileSystemWatcher** on Windows, **FSEvents** on macOS)
2. Queues changes locally
3. Uploads via the same upload API (presigned URL flow)
4. **Conflict resolution: last-write-wins** (versioning would handle this better but it's out of scope)

🔑 Remote is the **source of truth**.

#### Remote → Local — two options + a hybrid

| Approach | How it works | Pros | Cons |
|---|---|---|---|
| **Polling** | Client periodically calls `GET /files/changes?since=...` | Simple, no persistent connection | Slow detection, wasted bandwidth |
| **WebSocket / SSE** | Server pushes change events through a persistent connection | Real-time | Complex, connections drop |

#### 🔥 Hybrid (winning approach)
- **One WebSocket per device/session** (NOT one per file)
- Server pushes change events in real time
- Client also **polls every few minutes** as a safety net (catches missed events when WS drops)

Best of both worlds: real-time updates + guaranteed eventual consistency.

---

### Final Architecture — All Components

| Component | Role |
|---|---|
| **Uploader / Downloader (Client)** | Chunks files, manages local sync, communicates with backend + S3 + CDN |
| **LB + API Gateway** | Routing, SSL termination, rate limiting, validation |
| **File Service** | Control plane: reads/writes metadata, generates presigned URLs (locally, no S3 call), enforces permissions |
| **File Metadata DB** | DynamoDB / Postgres — file metadata + `SharedFiles` table |
| **S3 (Blob Storage)** | Stores actual file bytes; clients upload/download directly |
| **CDN (CloudFront)** | Caches files at edge for fast downloads; serves signed URLs |

---

## 6. Deep Dive 1 — Supporting Large Files (up to 50 GB)

### Why a single POST request fails
- ⚠️ **Timeouts** — math: 50 GB × 8 / 100 Mbps = **~1.1 hours** of upload
- ⚠️ **Payload limits** — API Gateway caps at **10 MB**; browsers/servers cap request sizes
- ⚠️ **Network interruptions** — one disconnect = restart 49 GB
- ⚠️ **Poor UX** — no progress feedback

### The Two Required UX Features
1. **Progress indicator**
2. **Resumable uploads** — pause/resume without re-uploading

### Solution: Chunking (5–10 MB chunks)

🔑 **Chunking happens on the CLIENT, never the server.** Server-side chunking defeats the purpose — the file already arrived in one piece.

### Tracking chunks in metadata
```json
{
  "id": "123",
  "name": "file.txt",
  "status": "uploading",
  "chunks": [
    { "id": "chunk1", "status": "uploaded" },
    { "id": "chunk2", "status": "uploading" },
    { "id": "chunk3", "status": "not-uploaded" }
  ]
}
```

### Keeping `chunks` in sync — two approaches

#### 👍 Good — Client PATCH after each chunk
```http
PATCH /files/{fileId}/chunks
{ "chunks": [{ "id": "chunk1", "status": "uploaded" }] }
```
⚠️ **Security risk:** malicious client could mark chunks as uploaded without uploading. State drift = hard to debug.

#### 🔥 Great — Server-Side Chunk Verification with ETags
- Each S3 chunk upload returns an **ETag**
- Client sends ETags in PATCH request
- Backend verifies by calling S3's **`ListParts` API**
- 🔑 **"Trust but verify"** — accept client PATCHes for real-time progress, then periodically verify server-side before marking file `uploaded`

⚠️ **Why we can't just use S3 event notifications:** they fire only on **`CompleteMultipartUpload`** (final assembly), not for individual chunks.

### Identifying files & chunks: Fingerprinting

- Don't use file name — two users can upload same name; same user can have collisions
- Use a **fingerprint**: SHA-256 hash of **file content**
- Used for:
  - **Deduplication** — same fingerprint? already uploaded
  - **Resumability** — same fingerprint + `status=uploading` → fetch existing chunk state and resume
- Fingerprint each **chunk** as well as the whole file
- ⚠️ **Note:** fingerprint identifies content, not the *record*. `fileId` (UUID) is still the record's primary key; `fingerprint` is a separate field.

### Full Large-File Upload Flow

1. Client chunks file into 5–10 MB pieces; computes fingerprint per chunk + for whole file
2. Client asks backend: "do you already have this fingerprint for me?"
   - If yes + `status=uploading` → resume
3. If no → client POSTs to initiate upload:
   - Backend calls S3 **`CreateMultipartUpload`** → gets `uploadId`
   - Backend generates **presigned URL per chunk** (with `uploadId` + `partNumber`)
   - Saves metadata row with `status=uploading`
4. Client uploads each chunk to its presigned URL → S3 returns ETag
   - Client PATCHes backend with ETag → backend verifies via `ListParts`
5. When all chunks `uploaded`:
   - Backend calls S3 **`CompleteMultipartUpload`** → S3 assembles into single object
   - Backend flips metadata `status=uploaded`

🔑 **This is exactly AWS's S3 Multipart Upload API.** Mention familiarity but **explain how it works** — don't just name-drop.

### Downloads of large files
- After `CompleteMultipartUpload`, file is a single S3 object — normal download works
- For huge files, clients use **HTTP Range requests** to download byte ranges in parallel / resume

---

## 7. Deep Dive 2 — Speed: Faster Uploads, Downloads, Syncing

Already done:
- **CDN** for downloads (closer to user)
- **Chunking** for uploads (parallel chunks, adaptive sizes)

### Delta Sync — only sync changed chunks

🔑 **Problem with fixed-size chunks:** insert 1 byte at the start of a file → **every** chunk boundary shifts → every fingerprint changes → entire file re-syncs. Delta sync useless.

#### Solution: Content-Defined Chunking (CDC)
- Chunk boundaries determined by **content**, using a **rolling hash** (e.g., **Rabin fingerprinting**)
- Small edit affects only chunks around the change
- 🔑 **This is how Dropbox actually does efficient delta sync.**

### Compression (client-side)

- Reduces bytes transferred → faster
- Done on **client** before uploading (since S3 stores compressed bytes as-is)
- Decompressed on client after download

⚠️ **When to compress:**
- **Text files** = great (5 GB → 1 GB easy)
- **Media (JPG, MP4, PNG)** = already compressed, gain ~few %, not worth CPU

### Algorithm choice

| Algorithm | Notes |
|---|---|
| **Gzip** | Most widely supported, baseline |
| **Brotli** | Better ratio than Gzip (esp. for text); supported by all modern browsers |
| **Zstandard (zstd)** | Best speed/ratio balance; tunable; great for client-side |

🔑 **Compress BEFORE encrypt.** Encryption randomizes bytes → kills compressibility.

### Client decides compression based on:
- File type
- File size
- Current network conditions

---

## 8. Deep Dive 3 — File Security

### 1. Encryption in Transit
- **HTTPS** everywhere — no-brainer

### 2. Encryption at Rest
- **S3 server-side encryption** — enable on upload; S3 manages keys
- Even if someone gets the file bytes, they can't decrypt without keys

### 3. Access Control (ACL)
- The `SharedFiles` table is your basic ACL
- Backend checks before generating any presigned URL

### 4. Signed URLs — leaked-link protection

⚠️ **Problem:** authorized user shares download link on Twitter → public access.

**Fix:** Short-lived **signed URLs** (e.g., 5 min expiry).

🔑 **Signed URLs are bearer tokens** — anyone with a valid, unexpired URL can download. Short TTL limits exposure but doesn't fully prevent sharing. For higher security: **IP binding** or require auth cookies alongside.

### How CDN signed URLs work (CloudFront)
1. **Generate** — server creates URL with signature (URL path + expiration + optional restrictions), signed with content provider's **private key**
2. **Distribute** — sent to authorized user
3. **Validate** — CDN verifies signature with **public key**, checks expiry, serves or denies

---

## 9. Key Numbers Cheatsheet

| Metric | Number |
|---|---|
| Max file size | 50 GB |
| Chunk size | 5–10 MB |
| 50 GB @ 100 Mbps | ~1.1 hours upload |
| API Gateway payload cap | 10 MB |
| Signed URL TTL | ~5 minutes (typical) |
| Sync WS connection | 1 per device, NOT per file |
| Hash algorithm for fingerprint | SHA-256 |

---

## 10. Tradeoffs Cheatsheet

| Decision | Chosen | Why | Tradeoff |
|---|---|---|---|
| File storage | **S3 (Blob Storage)** | Cheap, durable, unlimited | Extra integration complexity |
| Upload path | **Direct client → S3 via presigned URL** | No double-transfer | Coordination via S3 event notifications |
| Download path | **CDN signed URL** | Edge caching, global low latency | CDN cost; caching strategy needed |
| Share storage | **Separate `SharedFiles` table** | Fast lookup both directions | Index lookup vs sync risk — worth it |
| Sync mechanism | **WebSocket + polling fallback** | Real-time + reliable | More complex than pure polling |
| Large-file upload | **Client chunking + S3 Multipart Upload** | Resumability + parallelism + progress | More complex client logic |
| Chunk verification | **ETag + ListParts (trust but verify)** | Server-side integrity | Periodic verification cost |
| Delta sync | **Content-Defined Chunking** | Small edits = small re-sync | More complex than fixed chunking |
| Compression | **Client-side, before encryption** | Backend stays out of data path | Skip for media files |
| Security | **Signed URLs + S3 SSE + HTTPS** | Defense in depth | Signed URLs are bearer tokens (IP-bind for stricter) |

---

## 11. Common Pitfalls & Gotchas

- ⚠️ **Don't upload file twice** — client → backend → S3 is wasteful. Use presigned URLs.
- ⚠️ **Don't chunk on the server** — defeats the purpose. Chunk on the client.
- ⚠️ **S3 event notifications fire on completion, not per chunk** — use `ListParts` for chunk-level verification.
- ⚠️ **Don't identify files by name** — use fingerprint (SHA-256 of content).
- ⚠️ **Fixed chunk boundaries break delta sync** — use Content-Defined Chunking with rolling hash.
- ⚠️ **Compress before encrypt** — encrypted data is incompressible.
- ⚠️ **Don't compress media** — already compressed; CPU not worth it.
- ⚠️ **Signed URLs are bearer tokens** — keep TTL short; consider IP binding for sensitive files.
- ⚠️ **One WebSocket per device, not per file** — don't open N connections.
- ⚠️ **API Gateway has a 10 MB payload cap** — single POST can't handle 50 GB.
- ⚠️ **Don't pass user ID in request body** — always derive from auth header.

---

## 12. Level Expectations

### 🟢 Mid-level (80% breadth / 20% depth)
- Clear API + data model
- Functional HLD covering upload, download, share
- Probing questions OK ("how do you avoid uploading twice?" → reason to presigned URLs)
- Don't need to immediately know presigned URLs, multipart upload, chunking — but must reason to them with prompts
- Mixture of driving and being driven by interviewer

### 🟡 Senior (60% breadth / 40% depth)
- Quickly get past basic HLD to spend time on **large-file handling**
- Proactively walk through chunking, resumability, multipart upload
- Articulate tradeoffs (presigned URL vs backend proxy, fingerprinting, etc.)
- Hands-on familiarity with file upload APIs is a plus
- Anticipate problems; suggest improvements

### 🔴 Staff+ (40% breadth / 60% depth)
- Breeze through basics — interviewer assumes you know them
- Deep-dive into chunking, **CDC + rolling hashes**, ETag verification flow
- Discuss real-world tradeoffs from experience
- Steer the conversation; intervene only to focus, not steer
- Treat interviewer as peer — articulate trade-offs clearly
- Could go deep on multi-region replication, encryption-key management, audit logs, etc.

---

## 🎯 30-Second Recall (eve-of-interview)

> **File store with 50GB blobs.** Metadata in **DynamoDB/Postgres**; bytes in **S3** via **presigned URLs** (client → S3 directly, never through backend). Downloads via **CDN signed URLs**. Share via a normalized **`SharedFiles` table** with `(userId, fileId)` composite key. Sync = **WebSocket push + polling fallback**, last-write-wins. Large files = **client-side chunking (5–10 MB)** + **S3 Multipart Upload** + **ETag/ListParts verification** + **SHA-256 fingerprints** for dedup & resumability. Fast sync via **Content-Defined Chunking (rolling hash)**. Compress before encrypt, client-side, skip for media. Signed URLs are short-lived bearer tokens. Backend = pure control plane.

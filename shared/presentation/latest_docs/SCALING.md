# Scaling Guide — Warga.jp

## TL;DR

1. **Vertical first.** Resize the VPS before adding complexity.
2. **Add Litestream next** — for backup/DR value even on a single node, and as the first step toward replicas.
3. **Read replicas only when the vertical ceiling is hit** — route anonymous GETs to replicas, session-bearing requests to primary.
4. **Postgres migration is the last resort** — the export tool (`/admin/migrasi`) exists for this, but trigger it only if Litestream replication lag becomes user-visible.

---

## Current Baseline (already optimised)

Before adding capacity, note what the app already does well at the single-node level:

| Optimisation | Where |
|---|---|
| WAL mode + 5 s busy timeout on all 4 DBs | `migrations/migration.go` `openDB` |
| Connection pool sized to `max(NumCPU*2, 4)` per DB | same |
| Single aggregate SQL queries (GROUP BY prefecture, window functions) instead of per-pref fan-out | `performance.md` § Per-prefecture queries |
| Singleflight coalescing concurrent cold-cache misses | `performance.md` § Singleflight |
| No-TTL in-memory caches warmed at startup (counts, newest, featured) | `rules/infra-caching.md` § HTTP Caching |
| TTL caches for content pages + search results (5 min) | same |
| CDN edge caching via Cloudflare (`max-age=300, stale-while-revalidate=3600`) | `VPS_SETUP.md` § 2 |
| Buffered-channel analytics writer (no per-request goroutines) | `performance.md` § Fire-and-forget DB writes |

The CDN absorbs the majority of public read traffic — the Go origin only handles cache misses. Most capacity headroom is in the existing architecture.

---

## Priority 1 — Vertical Scaling

**Do this first.** SQLite is a single-file database; scaling up the machine is always cheaper and simpler than distributing it.

| Axis | Upgrade | What it unlocks for this stack |
|---|---|---|
| **vCPU** | 2 → 4 or 4 → 8 | WAL allows N concurrent readers; more CPUs serve more concurrent requests |
| **RAM** | 2 → 4 GB | OS page cache holds more SQLite pages in memory; Go in-memory caches (search, content) can grow without GC pressure |
| **Disk** | SATA SSD → NVMe | SQLite WAL checkpointing is I/O-bound; faster disk directly reduces write latency for form submissions |
| **GOMAXPROCS** | (verify after resize) | `runtime.GOMAXPROCS` defaults to CPU count on Linux — confirm after resize that it reflects the new vCPU count |

**Trigger:** CPU sustained > 70% for more than a few minutes, or p95 origin latency visibly increasing.

**Ceiling estimate:** A well-tuned single Go + SQLite node handles thousands of requests/second for read-heavy community content. For the target audience size (Indonesian community in Japan), the vertical ceiling is unlikely to be reached soon.

---

## Priority 2 — SQLite Read Replicas

When vertical scaling is no longer sufficient, read replicas extend capacity without restructuring the app.

### Option A — Litestream (recommended first step)

[Litestream](https://litestream.io/) streams SQLite WAL frames to object storage (S3, R2, GCS) continuously. A replica node restores from that stream.

```
Primary VPS:   Go app + Litestream → streams WAL to R2/S3
Replica VPS:   Litestream restores ← R2/S3 + Go app (read-only)
```

- Caddy (or a load balancer) routes `POST` requests and session-bearing `GET` requests to the primary; anonymous `GET` requests to replica(s).
- Replication lag is typically 100–500 ms — acceptable for a content site where new approvals appearing 1 second later on replicas is not user-visible.
- Litestream is free; you pay only for object storage egress.
- Even before adding replica nodes, Litestream on the primary alone provides continuous off-site backup — an improvement over the current nightly cron backup.

### Option B — LiteFS

[LiteFS](https://github.com/superfly/litefs) replicates SQLite across nodes using a FUSE filesystem and a primary-lease model. Writes are automatically forwarded to the primary.

- Better when you cannot cleanly separate read/write traffic at the Caddy/load-balancer level.
- Worse operationally — requires FUSE on every node and a lease manager (etcd or Consul).
- More appropriate for Fly.io deployments where LiteFS is a first-class primitive.

### Option C — Migrate to Postgres

The project has a `/admin/migrasi` export tool that generates a Postgres-compatible SQL dump of `app.db` content. Use this path if:

- Horizontal writes (not just reads) are needed across multiple nodes.
- LiteStream replication lag causes user-visible staleness.
- The team is comfortable operating Postgres.

This is a significant operational event. Do not trigger it prematurely.

---

## The Sessions Problem

`admin.db` and `user.db` store sessions. A replica receives session rows from the primary with a short delay — a user who just logged in might hit a replica that doesn't yet know about their session.

**Preferred mitigation:** Route any request that carries a session cookie to the primary. Only truly anonymous GET requests (no session cookie) go to replicas. For warga.jp — where the vast majority of traffic is anonymous public browsing — this keeps the replica read ratio high while keeping correctness simple.

**Alternatives (higher complexity):**
- Replace DB sessions with signed cookies (stateless — no DB lookup needed on any node).
- Shared session store (single Redis/Valkey instance shared across nodes).

---

## Recommended Execution Order

```
Now         → Vertical: resize VPS (CPU + RAM + NVMe)
Soon        → Add Litestream to primary for continuous off-site backup (free, low-risk, pays off even solo)
When needed → Add replica node(s); split traffic in Caddy (anon GET → replica, POST/session → primary)
If lag hurts → Evaluate LiteFS or Postgres migration
```

---

## What NOT to Do

- **No extra CDN-bypass cache** (Varnish, etc.) — Cloudflare already handles this layer; adding another cache introduces invalidation complexity with no benefit.
- **No SQLite sharding by prefecture across machines** — all content lives in the single `app.db` with indexed `WHERE prefecture = ?` queries; splitting across machines adds network latency and coordination overhead with no benefit at current scale.
- **No premature Postgres migration** — the export tool exists for when it's truly needed; SQLite + Litestream scales further than most community sites ever require.
- **No `SetMaxOpenConns(1)` on any DB** — WAL concurrency requires more than one connection per DB; see `performance.md` § SQLite connection pool.
- **No Redis/key-value store for the search cache on a single node.** The current in-process `globalSearchCache` / `prefSearchCache` (5-min TTL) plus singleflight already gives the optimal single-node behaviour: zero network RTT on hits and thundering-herd protection on cold misses. Redis would *add* latency per hit (network round-trip) and introduce a new operational dependency. Revisit only when running multiple app instances (see § Priority 2) — at that point a shared Valkey/Redis becomes useful so each replica doesn't pay its own cold-miss cost after restart.

---

## Trigger Thresholds

| Signal | Recommended action |
|---|---|
| Sustained CPU > 70% | Vertical: add vCPUs |
| Sustained memory > 80% | Vertical: add RAM |
| p95 origin latency regression on public pages | Check CDN hit rate first; if hit rate is fine, add vCPUs |
| Search cache hit rate dropping (more misses) | Check if `searchCacheTTL` should be raised; if load is genuine, vertical first |
| Write contention errors (`SQLITE_BUSY` after 5 s) | Increase `_busy_timeout` in DSN; then vertical NVMe |
| Vertical ceiling confirmed hit | Litestream + replica node |
| Litestream lag > 2 s sustained | LiteFS or Postgres migration |

---

## In-Product Health Indicator (`/admin/sistem`)

The admin panel surfaces these thresholds as a live page so an operator does not need to SSH into the box to know when an upgrade is due.

- **Dashboard tile** (`/admin`) — compact "VPS" card with overall severity badge (green / amber / red) plus a one-line summary like `CPU 23%, RAM 41%, Disk 18%`. Links to the detail page.
- **Detail page** (`/admin/sistem`) — per-metric table (CPU 1m load, RAM, Disk, Goroutine, Go heap), DB sizes, build info, and a reference table of the warn/critical thresholds.

**Severity rules** (encoded in `presentation/admin/system/viewmodel/viewmodel.go`):

| Metric | Warn | Critical |
|---|---|---|
| CPU 1m load / vCPU | 70% | 95% |
| RAM | 70% | 85% |
| Disk (filesystem holding `dbs/`) | 75% | 90% |
| Goroutines | 500 | 2000 |

When any row hits **critical**, treat that as the same signal as the rows in "Trigger Thresholds" above and act accordingly (CPU → add vCPUs, RAM → add RAM, Disk → enlarge storage or prune analytics).

**Implementation:** `lib/sysmetrics/sysmetrics.go` samples once per request via [`github.com/shirou/gopsutil/v4`](https://github.com/shirou/gopsutil) (cross-platform — works identically on Linux production and macOS dev). Reads are cheap (single syscall per metric); the page is gated by the admin auth middleware. No background sampler, no metrics DB.

# Performance

Rules distilled from the April 2026 perf audit. Apply them when writing new repository functions, migrations, middleware, and handlers — not after the fact.

---

## Per-prefecture queries

All content tables (`generic_info_post`, `community_info`, `reports`) live in the unified `dbs/app.db`. Scope queries to a prefecture with an indexed `WHERE prefecture = ?` clause — do **not** goroutine-fan-out per-prefecture work.

```go
// correct: single indexed query
posts, err := GetAllGenericInfoPostByStatus(appDB, prefecture, "approved")

// wrong: one goroutine per prefecture, one DB open per prefecture
for _, pref := range prefectures {
    go func(pref string) { ... queryDB(prefectureDBMap[pref]) ... }(pref)
}
```

**Why:** 47 goroutines + 47 DB connections add coordination overhead; a single `WHERE prefecture = ?` query on a composite index is faster and simpler.

---

## SQLite connection pool

`migrations/migration.go` `openDB` already sets:

```go
maxConns := max(runtime.NumCPU()*2, 4)
db.SetMaxOpenConns(maxConns)
db.SetMaxIdleConns(maxConns / 2)
db.SetConnMaxLifetime(30 * time.Minute)
```

Do **not** lower `MaxOpenConns` to 1 on any opened DB. WAL mode (`_journal_mode=WAL` in the DSN) allows N concurrent readers + 1 writer; `busy_timeout=5000` already serialises write contention.

**Why:** `SetMaxOpenConns(1)` silently cancels WAL reader concurrency and bottlenecks concurrent reads.

---

## Indexes on content tables

Any new content table queried with `WHERE prefecture = ? AND status = ? ORDER BY created_at DESC` needs a composite index:

```go
`CREATE INDEX IF NOT EXISTS idx_<table>_pref_status_created ON <table> (prefecture, status, created_at DESC)`
```

Also add a cross-prefecture index for admin list views:

```go
`CREATE INDEX IF NOT EXISTS idx_<table>_status_created ON <table> (status, created_at DESC)`
```

Also add a lookup index for any table where writes use a non-PK filter (e.g. the `reports` move UPDATE filters by `content_type + prefecture + content_id`):

```go
`CREATE INDEX IF NOT EXISTS idx_<table>_<columns> ON <table> (col1, col2, col3)`
```

Mirror all indexes in `migrations/export.go` inside both `writePostgresSchema` and `writeMySQLSchema`.

**Why:** Unindexed list queries become full table scans as content grows. Non-PK UPDATE/DELETE filters are equally unindexed without an explicit index.

---

## List-view SELECT: truncate large text columns

List and excerpt queries must use `substr(<col>, 1, 240) AS <col>` for any text column whose full value is only needed on the detail page. Detail-by-ID queries leave the column untouched.

```sql
-- list query (correct)
SELECT id, title, substr(content, 1, 240) AS content, status, created_at FROM generic_info_post ...

-- detail query (leave as-is)
SELECT id, title, content, status, created_at FROM generic_info_post WHERE id = ?
```

**Why:** Markdown bodies can be several KB; transferring and allocating the full string on every list render is measurable waste as content grows.

**Pattern references:** `domain/prefecture/repository/repository.go` `GetAllGenericInfoPostByStatus`; `domain/community/repository/repository.go` `GetApprovedCommunityPage`.

---

## OFFSET pagination is fine — until it isn't

Current public list pages use `LIMIT ? OFFSET ?` (`GetApprovedCommunityPage`, `GetApprovedGenericInfoPostPage`) with `pagination.PageSize = 10`. SQLite reads + discards `OFFSET` rows before returning `LIMIT`, but with the `(status, created_at DESC)` index in place the engine walks the index B-tree (not the table) and skips index leaves cheaply — sub-ms per query at realistic depth.

**Threshold to switch to keyset pagination:** any single prefecture's approved-row count for one table grows past **~5,000**, or users/crawlers regularly hit pages past **~50**. Past that, OFFSET cost grows linearly with page depth and a covering index alone won't save it.

**Keyset shape (when the time comes):**

```sql
-- instead of LIMIT 10 OFFSET 490
SELECT ... FROM generic_info_post
WHERE status = 'approved' AND (created_at, id) < (?, ?)
ORDER BY created_at DESC, id DESC
LIMIT 10
```

The cursor is the previous page's last `(created_at, id)`. Cost becomes O(LIMIT) regardless of depth and composes with the existing index without schema changes.

**Why:** OFFSET is "free" only while the skip count stays small. Once a single prefecture has tens of thousands of approved rows, deep-page renders will start showing up in latency tails — easier to plan the migration in advance than discover it via a Cloudflare alert.

**Admin analytics (`domain/analytics/repository/repository.go`) stays OFFSET** — admin-only, low traffic, date-range filter keeps the working set small.

---

## Response caching for aggregate / crawler endpoints

Any endpoint that (a) aggregates across prefectures and (b) has no per-user state must:

1. Run a single aggregate SQL query (e.g. `GROUP BY prefecture` or window function) rather than fanning out per-prefecture.
2. Memoise the rendered output behind a no-TTL `mu`/`loaded` cache (same pattern as `globalFeaturedCache`).
3. Emit `Cache-Control: public, max-age=<seconds>` on every response.
4. Wire an `Invalidate*Cache()` call into every admin mutation that changes the underlying data.

```go
var (
    myEndpointMu    sync.RWMutex
    myEndpointCache struct {
        data   []byte
        loaded bool
    }
)

func InvalidateMyEndpointCache() {
    myEndpointMu.Lock()
    myEndpointCache = struct{ data []byte; loaded bool }{}
    myEndpointMu.Unlock()
}
```

**Why:** Without caching, aggregate queries run on every crawler hit.

**Pattern reference:** `presentation/home/handler/sitemap.go`.

---

## Windowed caches must not back filtered views

A windowed cache (one that caps results — e.g. top-N per prefecture) is only consistent with **aggregate counts computed from the same window**. If the count that labels a filter chip comes from a full-corpus aggregate query, the filtered view must also query the full corpus — never the windowed cache.

**The failure mode:** count shown on the chip = 1 (from `COUNT(*) WHERE status='approved' AND category=?`); filter result = 0 (from an in-memory scan of a cache that only keeps top-3 per prefecture). The listing exists in the DB but was displaced from the cache by newer entries.

**Rule:** when you add a filter to an endpoint backed by a windowed in-memory cache:
1. If the filter count label comes from an unwindowed aggregate, the filter result must also come from an unwindowed DB query.
2. Do **not** add an in-memory loop over the windowed cache as a shortcut — it looks free but silently drops results.
3. If the filtered subset is small and uncacheable, a direct DB query per request is fine. Add a TTL cache keyed by filter value only if the query shows up in profiling.

**Current instance:** `communityIndexCache` keeps top-3 approved listings per prefecture. The `.tipe-strip` chip counts come from `CountApprovedCommunityByCategory` (full scan). The tipe-filtered view therefore calls `GetApprovedCommunityByCategory` (full scan, no cap) — not the cache.

---

## Context propagation

Every repository function that issues a DB query must accept `ctx context.Context` and use `db.QueryContext(ctx, …)` / `db.ExecContext(ctx, …)`.

```go
func SearchCommunity(ctx context.Context, db *sql.DB, prefecture, query string) ([]model.CommunityInfo, error) {
    rows, err := db.QueryContext(ctx, `SELECT ... FROM community_info WHERE ...`, ...)
    ...
}
```

- Pass `r.Context()` when the computation belongs to a single request (cancel on disconnect).
- Pass `context.Background()` when multiple requesters share the same singleflight computation.

**Why:** Queries without context keep running after client disconnect.

**Pattern references:** `domain/community/repository/repository.go` `SearchCommunity`; `domain/prefecture/repository/repository.go` `SearchGenericInfoPost`.

---

## Singleflight for cold cache misses

Any shared in-memory cache whose cold path is expensive (multi-DB fan-out) must coalesce concurrent misses with `golang.org/x/sync/singleflight`:

```go
var sf singleflight.Group

v, _, _ := sf.Do(cacheKey, func() (interface{}, error) {
    // double-check: another flight may have just populated the cache
    if cached, hit := checkCache(cacheKey); hit {
        return cached, nil
    }
    // use context.Background() — not r.Context() — so one disconnecting
    // requester can't cancel the shared computation for all waiters
    result := expensiveFanOut(context.Background(), ...)
    storeCache(cacheKey, result)
    return result, nil
})
```

**Why:** Two concurrent cold-miss searches each ran the aggregate query independently.

**Pattern reference:** `presentation/home/handler/handler.go` `globalSF`.

---

## In-memory IP-keyed maps need a sweeper

Any rate-limiter, login-attempt tracker, or other per-IP in-memory map must start a background sweeper goroutine in its constructor. The sweeper ticks at `2 × window` and deletes entries that have expired:

```go
func New(max int, window time.Duration) *Limiter {
    l := &Limiter{entries: make(map[string]*entry), max: max, window: window}
    go l.sweep()
    return l
}

func (l *Limiter) sweep() {
    ticker := time.NewTicker(2 * l.window)
    defer ticker.Stop()
    for range ticker.C {
        l.mu.Lock()
        for k, e := range l.entries {
            if time.Since(e.firstAt) >= l.window {
                delete(l.entries, k)
            }
        }
        l.mu.Unlock()
    }
}
```

**Why:** `entries` maps without a sweeper accumulate stale entries forever in long-lived processes.

**Pattern references:** `lib/ratelimit/ratelimit.go` `sweep()`; `presentation/admin/handler/auth.go` `loginLimiter.sweep()`.

---

## Fire-and-forget DB writes: single writer via buffered channel

Middleware or hooks that write analytics/audit data on every request must **not** spawn one goroutine per request. Use a buffered channel drained by a single writer goroutine started in the constructor:

```go
const writeQueueSize = 512

type Tracker struct {
    db *sql.DB
    ch chan writeRecord
}

func New(db *sql.DB) *Tracker {
    t := &Tracker{db: db, ch: make(chan writeRecord, writeQueueSize)}
    go t.writer()
    return t
}

func (t *Tracker) writer() {
    for rec := range t.ch { t.db.Exec(`INSERT ...`, rec.fields...) }
}

// enqueue non-blocking — drop on full queue, never block the response
select {
case t.ch <- rec:
default:
}
```

**Counter-example that is fine:** `go notify.Submission(...)` for SMTP notifications — those are one-shot per user action, not per-request hot-path writes.

**Why:** One goroutine per request all contending on a write mutex serialises under burst load.

**Pattern reference:** `lib/analytics/tracker.go`.


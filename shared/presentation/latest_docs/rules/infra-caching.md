# Infrastructure — Caching

## HTTP Caching

### Main page rendering and caching
`IndexHandler` in `presentation/home/handler/handler.go` renders the apex homepage dynamically on every origin request, then sets:
```
Cache-Control: public, max-age=300, stale-while-revalidate=3600
```
Cloudflare caches the rendered HTML for 5 minutes, so the origin only handles cache misses.

The handler closes over in-memory caches of two kinds:

| Cache | Variable | TTL | Warmup | Purpose |
|---|---|---|---|---|
| Search results | `globalSearchCache map[string]searchEntry` | 5 min | lazy | Cross-prefecture search, keyed by query string |
| Per-pref search | `prefSearchCache map[string]searchEntry` | 5 min | lazy | Prefecture-scoped search, keyed by `"<pref>:<query>"` |
| Recommended panduan | `globalFeaturedCache featuredCacheEntry` | **none** | startup | Single struct; warmed at `IndexHandler` init, invalidated by admin rekomendasi mutations |
| Per-prefecture counts | `globalCountsCache countsCacheEntry` | **none** | startup | Enriched sections with panduan/komunitas counts; warmed at `IndexHandler` init, invalidated by admin approve/reject/delete/add |
| Newest panduan | `globalNewestCache newestCacheEntry` | **none** | startup | Cross-prefecture newest-panduan list (5 items); warmed at `IndexHandler` init, invalidated by admin panduan mutations (add/approve/reject/delete) |
| Featured komunitas | `globalFeaturedKomunitasCache featuredKomunitasCacheEntry` | **none** | startup | Curated komunitas list; warmed at `IndexHandler` init, invalidated by admin featured-komunitas mutations |
| National content | `globalNationalCache nationalCacheEntry` | **none** | startup | Newest panduan + komunitas from `national.db`; warmed at `IndexHandler` init, invalidated alongside `globalCountsCache` |

**Rules:**
- Do **not** apply `Cache-Control: public` to pages that contain user-specific content or CSRF tokens embedded in HTML.
- When user login is re-enabled, check for a session cookie before setting `Cache-Control: public` — logged-in users need personalised HTML and must bypass CDN caching.
- No-TTL caches (`globalFeaturedCache`, `globalCountsCache`, `globalNewestCache`, `globalFeaturedKomunitasCache`, `globalNationalCache`) **must be warmed at startup** inside the handler factory (before `return func(...)`), not lazily on first request. This ensures zero cold-load cost for users. The lazy path inside the request handler is only a fallback for the single request that arrives right after an admin invalidation.
- When warming multiple independent caches at startup, fan them out with `sync.WaitGroup` so their DB work runs concurrently — see `IndexHandler` startup block as the canonical example.

### Current browser cache headers

| Route | Handler file | `Cache-Control` | TTL |
|---|---|---|---|
| `/static/pico.red.min.css` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/base.css` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/public.css` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/admin.css` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/admin.js` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/fonts/*.woff2` | `router/router.go` | `public, max-age=31536000, immutable` | 1 year |
| `/static/logo/*.png` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/logo/warga-jp-avatar.svg` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/logo/warga-jp-og-1200x630.png` | `router/router.go` | `public, max-age=604800` | 7 days |
| `/favicon.ico` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/og-image.png` | `presentation/home/handler/ogimage.go` | `public, max-age=604800` + ETag | 7 days |
| `/robots.txt` | `presentation/home/handler/robots.go` | `public, max-age=86400` | 1 day |
| `/kebijakan-privasi` | `presentation/privacy/handler/handler.go` | `public, max-age=3600` | 1 hour |
| `/syarat-ketentuan` | `presentation/terms/handler/handler.go` | `public, max-age=3600` | 1 hour |
| `/` (no search, anonymous) | `presentation/home/handler/handler.go` | `public, max-age=300, stale-while-revalidate=3600` + ETag | 5 min |
| `/kontak` | — | _(none — CSRF token in HTML)_ | — |
| `/komunitas/tambah` | — | _(none — CSRF token in HTML)_ | — |
| `/panduan/tambah` | — | _(none — CSRF token in HTML)_ | — |
| `/sitemap.xml` | `presentation/home/handler/sitemap.go` | `public, max-age=3600` + ETag | 1 hour |
| Admin pages | — | _(no-store)_ | — |

### ETag / conditional revalidation

Three handlers emit weak ETags so Cloudflare's revalidation requests return `304 Not Modified` (zero body) until content actually changes:

| Route | ETag derivation | Bumps when |
|---|---|---|
| `/` | `W/"<version.Build>-<homeVersion>"` | Any `Invalidate*Cache()` call on the homepage caches |
| `/sitemap.xml` | `W/"<version.Build>-<sitemapVersion>"` | `InvalidateSitemapCache()` is called |
| `/og-image.png` | `W/"<version.Build>"` | Redeploy only (image is code-generated) |

**Pattern:** an `atomic.Uint64` counter sits next to each cache. Every `Invalidate*Cache()` function calls `counter.Add(1)`. The handler computes the ETag once per request from `version.Build` + the counter's current value, checks `If-None-Match`, and returns 304 if it matches — otherwise sets the `ETag` header and renders normally.

**Why both halves matter:** `version.Build` invalidates ETags on every deploy (so a code change forces a body refresh even if no admin mutation happened). The counter invalidates them within a deploy whenever an admin mutates content.

**When to add an ETag to a new handler:**
- The response is fully determined by an in-memory cache that exposes an `Invalidate*Cache()` function.
- The body is large enough that returning 304 (no body) is meaningfully cheaper than re-sending it (rough threshold: a few KB or above).
- The page is hit often enough by Cloudflare revalidation to matter (e.g. anything with `max-age` ≥ 5 min and steady traffic).

Don't add ETags to admin pages (no-store) or to pages with embedded CSRF tokens (those don't have a stable version).

### Cache-Control guidelines for new pages
| Page type | Recommended header |
|---|---|
| Truly static content (no DB, no CSRF; only changes on redeploy) | `public, max-age=86400` |
| Pre-rendered from startup data (static until restart) | `public, max-age=300, stale-while-revalidate=3600` |
| Dynamic but same for all anonymous users | `public, max-age=60` |
| Contains CSRF token embedded in HTML | _(no Cache-Control — omit entirely)_ |
| User-specific content | `private, no-store` |
| Admin pages | `no-store` |

Security and session headers (`Set-Cookie`) do not automatically prevent browser caching with `Cache-Control: public`, but many CDNs and Caddy (with cache plugin) will refuse to cache responses that set cookies. Keep `Set-Cookie` off the fast path for public cached responses.

## Content Page Caching

### When to add a cache
Content that requires admin approval before going public changes rarely. Any public listing or detail page backed by such content is a good candidate for a TTL cache: skip the DB query on cache hits, serve the cached viewmodel directly.

Current cached pages:
| Handler | Variable | TTL | Max entries | Key format | Invalidation function |
|---|---|---|---|---|---|
| `presentation/community/handler` `IndexHandler` | `globalListCache` | 5 min | 300 | `"<prefecture>:<page>"` | `InvalidateCommunityListCache()` |
| `presentation/community/handler` `DetailHandler` | `globalDetailCache` | 5 min | 500 | `"<prefecture>:<id>"` | `InvalidateCommunityDetailCache()` |
| `presentation/panduan/handler` `DetailHandler` | `globalPanduanDetailCache` | 5 min | 500 | `"<prefecture>:<id>"` | `InvalidatePanduanDetailCache()` |

### How to add a cache to a new content feature

Use `lib/cache` — a generic LRU + TTL cache backed by `github.com/hashicorp/golang-lru/v2/expirable`. Hard capacity cap: when full, the least recently used entry is evicted before inserting. Entries also expire after the configured TTL regardless of access.

**1. Declare constants and the cache as a package-level variable:**
```go
const myFeatureListCacheMaxSize  = 300
const myFeatureDetailCacheMaxSize = 500
const myFeatureCacheTTL = 5 * time.Minute

var globalMyFeatureListCache   = cache.New[string, viewmodel.MyFeatureViewModel](myFeatureCacheTTL, myFeatureListCacheMaxSize)
var globalMyFeatureDetailCache = cache.New[string, viewmodel.MyFeatureDetailViewModel](myFeatureCacheTTL, myFeatureDetailCacheMaxSize)

func InvalidateMyFeatureListCache()   { globalMyFeatureListCache.Purge() }
func InvalidateMyFeatureDetailCache() { globalMyFeatureDetailCache.Purge() }
```

> **Why package-level, not local to the handler factory?** A local variable inside `func IndexHandler(...) http.HandlerFunc { cache := ... }` is invisible to the rest of the codebase. Admin mutation handlers cannot call `cache.Purge()` on it. This caused a bug where editing a community or panduan title left stale data on the public listing and detail pages for up to 5 minutes even though the server-side DB was already updated.

**2. Use the package-level cache inside the handler factory:**
```go
func IndexHandler(prefectures []prefectureModel.Prefecture, prefectureDBMap map[string]*sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        page := parsePage(r)
        key := fmt.Sprintf("%s:%d", prefecture.Value, page)
        if vm, ok := globalMyFeatureListCache.Get(key); ok {
            templ.Handler(view.MyFeatureScreen(vm)).ServeHTTP(w, r)
            return
        }
        vm := viewmodel.NewMyFeatureViewModel(prefecture, db, page)
        globalMyFeatureListCache.Set(key, vm)
        templ.Handler(view.MyFeatureScreen(vm)).ServeHTTP(w, r)
    }
}
```

**3. Cache key conventions:**
- List pages: `"<prefecture_value>:<page_number>"` — e.g. `"tokyo:1"`
- Detail pages: `"<prefecture_value>:<id>"` — e.g. `"tokyo:42"`
- If a page has no prefecture context (global): use the page or id directly as key

**4. Do not cache:**
- Pages with CSRF tokens embedded in HTML (submission forms, admin pages)
- Pages that show user-specific content
- Admin-facing pages (they need live data)

### Invalidation strategy

There are two cache patterns in this project:

**TTL-based with explicit invalidation** (`lib/cache`, content list/detail pages): entries expire after 5 minutes, but admin mutation handlers also call `Purge()` immediately so changes are visible right away. `lib/cache` uses LRU eviction when full, so the capacity cap is hard.

**Rule: every admin handler that mutates content (edit title, approve, reject, delete) must call the corresponding `Invalidate*Cache()` for every TTL cache that holds that content.** Omitting this call causes a staleness window equal to the TTL — the admin can see the change in the DB but the public site serves the old version.

Current wiring for content mutations:
| Admin action | Caches invalidated |
|---|---|
| Panduan edit (title/content) | `InvalidatePanduanDetailCache`, `InvalidatePanduanIndexCache`, `InvalidateFeaturedCache`, `InvalidateSitemapCache` |
| Panduan approve/reject/delete | `InvalidateCountsCache`, `InvalidateNewestCache`, `InvalidatePanduanIndexCache`, `InvalidateSitemapCache` |
| Panduan move (cross-prefecture) | `InvalidatePanduanIndexCache`, `InvalidatePanduanDetailCache`, `InvalidateCountsCache`, `InvalidateNewestCache`, `InvalidateFeaturedCache`, `InvalidateSitemapCache` |
| Community edit (name/content) | `InvalidateCommunityDetailCache`, `InvalidateCommunityListCache`, `InvalidateCommunityIndexCache`, `InvalidateFeaturedKomunitasCache`, `InvalidateSitemapCache` |
| Community approve/reject/delete | `InvalidateCountsCache`, `InvalidateCommunityIndexCache`, `InvalidateSitemapCache` |
| Community move (cross-prefecture) | `InvalidateCommunityIndexCache`, `InvalidateCommunityListCache`, `InvalidateCommunityDetailCache`, `InvalidateCountsCache`, `InvalidateFeaturedKomunitasCache`, `InvalidateSitemapCache` |

**No-TTL with explicit invalidation** (`globalFeaturedCache`, `globalCountsCache`, `globalNewestCache`): cache lives until an admin mutation explicitly calls the corresponding invalidation function. Use this pattern when:
- The data rarely changes (only via admin action)
- Staleness is not acceptable (counts, curated lists)
- The load cost justifies warming at startup (concurrent DB queries)

Rules for no-TTL caches:
1. **Warm at startup** inside the handler factory, before `return func(...)`, so no user request pays the cold-load cost. If warming multiple independent caches at startup, fan them out with `sync.WaitGroup` — startup time becomes `max(T_slowest)` instead of the sum.
2. **Keep the lazy fallback** inside the request handler for the single request that arrives right after an invalidation.
3. **Call the invalidation function** from every admin handler that mutates the underlying data. If you add a new admin action, check whether it affects a no-TTL cache and add the call.

### Changing the TTL or max size
Constants live at the top of each handler file (not a shared constant). Change them per-handler as needed. The search cache TTL (`searchCacheTTL = 5 * time.Minute`) in `presentation/home/handler/handler.go` is separate from content page caches.

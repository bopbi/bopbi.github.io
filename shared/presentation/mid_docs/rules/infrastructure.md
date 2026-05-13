# Infrastructure

## Constants — `constant` package

### `constant.BaseURL()`
Returns the scheme + domain for the apex site: `"http://machine.local"` in dev, `"https://warga.jp"` in production. Use this whenever linking to the apex domain from templates or components — never hardcode `"https://"` for links that must work in dev.

```go
// chrome.templ — correct
<a href={ constant.BaseURL() + "/komunitas" }>Komunitas</a>

// wrong — always https, breaks HTTP dev environment
<a href={ "https://" + constant.DomainName() + "/komunitas" }>Komunitas</a>
```

`constant.DomainName()` is still correct for building canonical URLs (which are always absolute `https://`) and for SEO meta tags — those intentionally use `https://` regardless of environment.

### `constant.KBRITokyoURL`
Hard-coded URL for the Indonesian embassy in Tokyo (`https://www.kemlu.go.id/tokyo`). Used in `presentation/contact/view/contact.templ`. Update here if Kemlu moves the page — do **not** hardcode the URL directly in templates.

```go
<a href={ templ.URL(constant.KBRITokyoURL) } target="_blank" rel="noopener">KBRI Tokyo</a>
```

### Error page pre-rendering — `response.Init()`
`presentation/shared/response/response.go` pre-renders the 404 and 500 error pages for zero-allocation serving. This must happen **after** `constant.SetDomainName()` is called, otherwise nav links render with an empty domain.

`main.go` calls `response.Init()` immediately after the domain/env block:
```go
constant.SetDomainName("machine.local")
// ... env setup ...
response.Init()   // must come after SetDomainName
```

**Do not** move `response.Init()` before `SetDomainName`, and do not revert it to an `init()` function in `response.go` — `init()` runs before `main()` and the domain name is not set yet.

## Database Migrations
- Versioned migrations stored in `schema_migrations` table
- Use `runMigration(db, version, name, func)` to add new migrations
- Use `columnExists(db, table, column)` helper for idempotent column additions — implemented in `migrations/migration.go` using `PRAGMA table_info`
- **Never use `DROP TABLE` in migrations**
- Adding a column: `ALTER TABLE ADD COLUMN` (guard with `columnExists` check)
- Removing a column: `ALTER TABLE DROP COLUMN` (guard with `columnExists` check; supported in SQLite ≥ 3.35, which the bundled `go-sqlite3` provides)

### Schema changes must also update the export tool
Any change to a table schema (new column, renamed column, new table, dropped column) requires
updating **both** of the following files or the generated SQL dump will be wrong:

| File | What to update |
|---|---|
| `migrations/export.go` | Add/remove the column from the `SELECT` query, the `rows.Scan(...)` call, the `INSERT INTO ... (cols)` column list, and the `fmt.Fprintf` value list for the affected table's export function |
| `migrations/export.go` `writePostgresSchema` / `writeMySQLSchema` | Add/remove the column from the `CREATE TABLE` DDL block for that table |

For a **new table**, also add a new `exportXxx(...)` function and call it from `ExportToSQL`
in the appropriate section (shared tables vs. per-prefecture loop).

Special cases to remember:
- Per-prefecture tables omit `id` in `INSERT` (target DB auto-assigns to avoid cross-prefecture conflicts).
- Shared tables (`users`, `sessions`, `admins`, `admin_sessions`) preserve `id` for FK integrity — also add the table name to `writeSequenceResets`.
- Reserved words in MySQL need renaming: the `condition` column in `recycled_goods` was renamed to `item_condition` in the exported schema.

## Database — Nullable Columns

### Always scan nullable TEXT columns via `sql.NullString`
Any column defined as `TEXT` without `NOT NULL` in the schema can contain SQL `NULL`. Scanning `NULL` into a Go `string` with `rows.Scan` returns an error, which causes the row to be silently skipped via `continue` — the query appears to succeed but returns fewer rows than expected.

**Affected columns in `generic_info_post`:** `category`, `submitter_email`, `submitter_whatsapp`, `submitter_line`

Use a dedicated scan helper (see `scanGenericInfoPost` in `domain/prefecture/repository/repository.go`) that scans nullable columns into `sql.NullString` and extracts `.String`:

```go
var email sql.NullString
rows.Scan(..., &email, ...)
post.SubmitterEmail = email.String  // "" when NULL
```

**Rule:** For any new table with nullable TEXT columns, write a `scanXxx` helper and use it in every query function for that table. Never scan a nullable column directly into `string`.

### Symptoms of this bug
- Dashboard `COUNT(*)` shows N rows but the list view shows fewer (or zero)
- No errors in logs — the scan error is caught and `continue`d silently
- Only rows with all non-NULL values appear

### Seed data must use explicit status
The `seedGenericInfoPost` function inserts admin-created placeholder content. It must always set `status = 'approved'` explicitly — do not rely on the column default, which is `'pending'` (intended for user submissions). Seeded rows that are pending will inflate the pending count on the admin dashboard.

```go
// correct
INSERT INTO generic_info_post (title, content, category, status)
VALUES (?, ?, ?, 'approved')

// wrong — inherits DEFAULT 'pending', shows as user submission needing review
INSERT INTO generic_info_post (title, content, category)
VALUES (?, ?, ?)
```

If existing seeded rows are stuck as `'pending'`, fix them with a versioned migration:
```go
UPDATE generic_info_post SET status = 'approved'
WHERE title = '<seed title>' AND status = 'pending'
```

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

**Rules:**
- Do **not** apply `Cache-Control: public` to pages that contain user-specific content or CSRF tokens embedded in HTML.
- When user login is re-enabled, check for a session cookie before setting `Cache-Control: public` — logged-in users need personalised HTML and must bypass CDN caching.
- No-TTL caches (`globalFeaturedCache`, `globalCountsCache`, `globalNewestCache`) **must be warmed at startup** inside the handler factory (before `return func(...)`), not lazily on first request. This ensures zero cold-load cost for users. The lazy path inside the request handler is only a fallback for the single request that arrives right after an admin invalidation.

### Current browser cache headers

| Route | Handler file | `Cache-Control` | TTL |
|---|---|---|---|
| `/static/pico.red.min.css` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/base.css` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/public.css` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/admin.css` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/admin.js` | `router/router.go` | `public, max-age=604800, immutable` | 7 days |
| `/static/fonts/*.woff2` | `router/router.go` | `public, max-age=31536000, immutable` | 1 year |
| `/og-image.png` | `presentation/home/handler/og_image.go` | `public, max-age=604800` | 7 days |
| `/robots.txt` | `presentation/home/handler/robots.go` | `public, max-age=86400` | 1 day |
| `/kebijakan-privasi` | `presentation/privacy/handler/handler.go` | `public, max-age=3600` | 1 hour |
| `/syarat-ketentuan` | `presentation/terms/handler/handler.go` | `public, max-age=3600` | 1 hour |
| `/` (no search, anonymous) | `presentation/home/handler/handler.go` | `public, max-age=300, stale-while-revalidate=3600` | 5 min |
| `/kontak` | — | _(none — CSRF token in HTML)_ | — |
| `/komunitas/tambah` | — | _(none — CSRF token in HTML)_ | — |
| `/panduan/tambah` | — | _(none — CSRF token in HTML)_ | — |
| `/sitemap.xml` | — | _(none — dynamic DB query)_ | — |
| Admin pages | — | _(no-store)_ | — |

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
| Handler | Cache | TTL | Max entries | Key format |
|---|---|---|---|---|
| `presentation/community/handler` `IndexHandler` | community list | 5 min | 300 | `"<prefecture>:<page>"` |
| `presentation/community/handler` `DetailHandler` | community detail | 5 min | 500 | `"<prefecture>:<id>"` |
| `presentation/panduan/handler` `DetailHandler` | panduan detail | 5 min | 500 | `"<prefecture>:<id>"` |

### How to add a cache to a new content feature

Use `lib/cache` — a generic TTL cache with no external dependencies.

**1. Declare constants at the top of the handler file:**
```go
const myFeatureListCacheMaxSize  = 300
const myFeatureDetailCacheMaxSize = 500
const myFeatureCacheTTL = 5 * time.Minute
```

**2. Initialise the cache inside the handler factory closure** (same pattern as `ratelimit.New`):
```go
func IndexHandler(prefectures []prefectureModel.Prefecture, prefectureDBMap map[string]*sql.DB) http.HandlerFunc {
    listCache := cache.New[string, viewmodel.MyFeatureViewModel](myFeatureCacheTTL, myFeatureListCacheMaxSize)
    return func(w http.ResponseWriter, r *http.Request) {
        page := parsePage(r)
        key := fmt.Sprintf("%s:%d", prefecture.Value, page)
        if vm, ok := listCache.Get(key); ok {
            templ.Handler(view.MyFeatureScreen(vm)).ServeHTTP(w, r)
            return
        }
        vm := viewmodel.NewMyFeatureViewModel(prefecture, db, page)
        listCache.Set(key, vm)
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

**TTL-based** (`lib/cache`, search caches): no explicit invalidation — entries expire after 5 minutes. Suitable for content that can tolerate a short staleness window (list/detail pages, search results).

**No-TTL with explicit invalidation** (`globalFeaturedCache`, `globalCountsCache`, `globalNewestCache`): cache lives until an admin mutation explicitly calls `InvalidateFeaturedCache()` / `InvalidateCountsCache()` / `InvalidateNewestCache()`. Use this pattern when:
- The data rarely changes (only via admin action)
- Staleness is not acceptable (counts, curated lists)
- The load cost justifies warming at startup (concurrent DB queries)

Rules for no-TTL caches:
1. **Warm at startup** inside the handler factory, before `return func(...)`, so no user request pays the cold-load cost.
2. **Keep the lazy fallback** inside the request handler for the single request that arrives right after an invalidation.
3. **Call the invalidation function** from every admin handler that mutates the underlying data. If you add a new admin action, check whether it affects a no-TTL cache and add the call.

### Changing the TTL or max size
Constants live at the top of each handler file (not a shared constant). Change them per-handler as needed. The search cache TTL (`searchCacheTTL = 5 * time.Minute`) in `presentation/home/handler/handler.go` is separate from content page caches.

## Legal Pages

> **Every change to `privacy.templ` or `terms.templ` requires four things done in the same commit — see the checklist below. Skipping any step leaves the site in an inconsistent state.**

### Checklist for every legal page update

When changing `privacy.templ` or `terms.templ`:

1. Update the page content in the `.templ` file.
2. Update the corresponding date constant in `presentation/shared/legal/dates.go`.
3. **Prepend** a new entry to `Entries` in `presentation/shared/changelog/entries.go` — `/weblogs` is the user-facing record of legal changes and must stay current.
4. Run `templ generate` on the affected view package.

All four in the same commit.

The same applies to any other major platform change (new features, significant behaviour changes, etc.) — step 3 only.

### Updating the "last updated" date

The Privacy Policy (`/kebijakan-privasi`) and Terms & Conditions (`/syarat-ketentuan`) each show a "Terakhir diperbarui" date. These dates live in one file:

```
presentation/shared/legal/dates.go
```

```go
const PrivacyUpdated = "19 April 2026"  // update when changing privacy.templ
const TermsUpdated   = "18 April 2026"  // update when changing terms.templ
```

Do not edit the date inside the `.templ` files directly — the constants are the single source of truth.

The `.templ` files import this package and render the value via `{ legal.PrivacyUpdated }` / `{ legal.TermsUpdated }`.

### Announcing changes in the changelog

Prepend to the `Entries` slice in `presentation/shared/changelog/entries.go` — newest first, never append:

```
presentation/shared/changelog/entries.go
```

```go
var Entries = []Entry{
    // newest first — always prepend, never append
    {
        Date:        "19 April 2026",
        Kind:        "kebijakan",   // "feat" | "kebijakan" | "" (omit chip)
        Title:       "Pembaruan Kebijakan Privasi — Pencatatan Kata Kunci Pencarian",
        Description: "Kebijakan Privasi diperbarui untuk mencerminkan bahwa kata kunci yang dimasukkan di kolom pencarian kini dicatat secara anonim di server kami.",
    },
}
```

`Kind` controls the coloured chip on the `/weblogs` timeline. Allowed values:

| Value | Label | Colour |
|---|---|---|
| `"feat"` | Fitur | Blue |
| `"kebijakan"` | Kebijakan | Amber |
| `""` | _(no chip)_ | — |

Use `"kebijakan"` for privacy policy / terms changes. Use `"feat"` for new public features. Leave `Kind` empty for minor operational entries that don't fit either category.

**Rule:** The slice is capped at 45 entries (`MaxEntries` in `entries.go`). When the slice reaches 45, remove the oldest entry (the last element) in the same commit as the new prepend. The `GetEntries()` helper enforces this cap at runtime as a safety net, but the slice itself should never exceed 45.

## SEO

### Slug URLs for detail pages
Detail pages should use `/{id}-{slug}` URLs (e.g. `/komunitas/1-komunitas-indonesia-di-tokyo`, `/panduan/3-tips-hidup-di-tokyo`) instead of bare `/{id}` URLs. Both komunitas and panduan detail handlers already follow this pattern.

**How it works:**
- `lib/slug/slug.go` — `Slugify(name string) string`: lowercases, keeps ASCII letters/digits, replaces everything else with hyphens, collapses consecutive hyphens, caps at 80 chars. Returns `""` for names with no ASCII content (e.g. Japanese-only).
- The URL path is built in the viewmodel, not the template. Store it in a computed field (e.g. `CommunityInfo.Slug`) so templates stay simple.
- If `Slugify` returns `""`, fall back to `/{feature}/{id}` (no trailing hyphen). Always handle the empty-slug case.
- The detail handler parses only the numeric prefix: split the `{id}` param on the first `-` and call `strconv.Atoi` on the prefix. This keeps old bare-ID URLs working.

```go
// handler — works for both /komunitas/1 and /komunitas/1-some-slug
idStr := chi.URLParam(r, "id")
if i := strings.IndexByte(idStr, '-'); i != -1 {
    idStr = idStr[:i]
}
id, err := strconv.Atoi(idStr)
```

### Detail page SEO fields
When adding a new detail page using `LayoutCategory`, the viewmodel must populate four fields and pass them to the layout:

| Viewmodel field | Layout arg | Value |
|---|---|---|
| `MetaDescription string` | 5th (`description`) | Content truncated to ≤160 runes; append `"..."` if truncated |
| `CanonicalURL string` | 6th (`canonicalURL`) | Full absolute URL: `"https://" + prefecture.Value + "." + constant.DomainName() + slugPath` |
| `PublishedTime string` | 7th (`publishedTime`) | RFC3339 from `CreatedAt` via `formatter.FormatRFC3339` |
| `ModifiedTime string` | 8th (`modifiedTime`) | RFC3339 from `AdminEditedAt ?? UpdatedAt` via `formatter.FormatRFC3339` |

| Caller | description value |
|---|---|
| Listing pages, pref homepage, WIP pages | `""` (pass `"", "", "", ""` for last four args) |
| `CommunityDetailScreen` | truncated community description (≤160 runes) |
| `PanduanDetailScreen` | truncated panduan content (≤160 runes) |

The canonical URL consolidates bare-ID URLs (e.g. `/panduan/3`) and slug URLs (e.g. `/panduan/3-tips-hidup`) into a single canonical, preventing duplicate-content penalties.

### Meta description length
Truncate descriptions to **160 runes** (not bytes — use `[]rune` to handle multi-byte characters safely) before storing in `MetaDescription`. Append `"..."` when truncated.

### Open Graph and language
`layout_category.templ` emits the following for all pages:
- `<html lang="id">` — Indonesian content
- `<meta property="og:title">` — always set from `title`
- `<meta property="og:description">` — set when `description` is non-empty
- `<meta property="og:type">` — `"article"` when `canonicalURL` is non-empty, `"website"` otherwise
- `<meta property="og:site_name">` — `"Warga.jp"`
- `<meta property="og:locale" content="id_ID"/>` — always set (all three layouts)
- `<meta property="og:image">` — 1200×630 `/og-image.png` (all three layouts)
- `<meta property="article:published_time">` — set when `publishedTime` is non-empty and `canonicalURL` is set
- `<meta property="article:modified_time">` — set when `modifiedTime` is non-empty and `canonicalURL` is set
- Twitter Card (`twitter:card=summary_large_image`, `twitter:title`, `twitter:description`, `twitter:image`) — `LayoutCategory` and `LayoutIndex`

### noindex on error pages
`LayoutGeneric` accepts a `noindex bool` second parameter. Pass `true` for 404/500 error templates (`presentation/shared/view/error.templ`); pass `false` for all other pages. This emits `<meta name="robots" content="noindex"/>` so error pages are never indexed.

### LayoutIndex SEO
`LayoutIndex` accepts `description string` and `canonicalURL string` after the title. The apex homepage (`presentation/home/view/index.templ`) passes a static description and the apex canonical URL. Gate canonical and og:description tags on non-empty values (same pattern as `LayoutCategory`).

### Sitemap lastmod
`presentation/home/handler/sitemap.go` includes `<lastmod>` on panduan and komunitas entries using `updated_at` from the database rows already fetched for the sitemap. The helper `sitemapDate(s string) string` extracts the `YYYY-MM-DD` prefix from SQLite datetime strings. No extra queries.

### JSON-LD structured data
`presentation/shared/component/jsonld.templ` exports three zero-cost JSON-LD components:

| Component | Schema type | Used on |
|---|---|---|
| `WebSiteSchema(siteURL, logoURL)` | `WebSite` + SearchAction | Apex homepage (`index.templ`) |
| `ArticleSchema(headline, canonicalURL, publishedTime, modifiedTime, logoURL)` | `Article` | Panduan + comunitas detail pages |
| `BreadcrumbSchema([]BreadcrumbItem)` | `BreadcrumbList` | Panduan + comunitas detail pages |

`BreadcrumbItem` has `Name` and `URL` fields. Detail pages construct the list inline: `[Home, PrefectureLabel, ArticleTitle]`.

**Author / publisher**: always `{ "@type": "Organization", "name": "Warga.jp" }` — never expose submitter contact info (email, WhatsApp, Line) in structured data.

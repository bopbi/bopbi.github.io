# Content & UI

## Static Assets

### Pico.css
Pico.css is self-hosted. The file is embedded into the binary at build time via `static/static.go`:
```go
//go:embed pico.red.min.css
var FS embed.FS
```
It is served from `/static/pico.red.min.css` with `Cache-Control: public, max-age=604800, immutable` (7 days). The file bytes are read once at startup in `Router()` and written directly per-request — no disk I/O per hit.

**To upgrade Pico.css:**
1. `curl -sL "https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.red.min.css" -o static/pico.red.min.css`
2. `go build ./...` — the new file is embedded automatically
3. Redeploy — the old file is evicted from browser caches after 7 days, or immediately on a hard refresh

**Do not** add `<link rel="preconnect">` for cdn.jsdelivr.net — there is no CDN dependency anymore. The CSP in `Caddyfile.prod` reflects this: `style-src 'self'` and `font-src 'self'` (no cdn.jsdelivr.net).

### Adding new static assets
If you add another embedded asset (e.g. a custom font), follow the same pattern:
1. Place the file in `static/`
2. Add `//go:embed <filename>` to `static/static.go`
3. Read bytes once at startup in `router.go`, serve with appropriate `Content-Type` and `Cache-Control`
4. Add `/static/` prefix check already in `requireSubdomain` (main.go) — no change needed there

### version.Build and cache busting
`version/version.go` declares:
```go
var Build = "dev"
```
It defaults to `"dev"` for local builds. In production, `scripts/build-prod.sh` injects the git SHA via `-ldflags "-X website/version.Build=<sha>"`.

**Uses:**
- **Cache busting**: all layout templates append `?v=` + `version.Build` to stylesheet `<link>` hrefs, so browsers pick up new CSS on every deploy.
- **DEV indicator**: all three shared layouts (`layout_category.templ`, `layout_index.templ`, `layout_generic.templ`) render a red **DEV** badge next to the flag `<h1>` when `version.Build == "dev"`. This badge never appears in production. Do not remove this check.

## Home page region selector
The apex homepage (`presentation/home/view/index.templ`) no longer renders a flat prefecture grid. In the default no-search state it renders, in order:

1. The curated `⭐ Rekomendasi` section when `vm.FeaturedPanduan` is non-empty — yellow infobox (`.featured-section`).
2. The cross-prefecture `🆕 Panduan Terbaru` compact list when `vm.NewestPanduan` is non-empty — blue infobox (`.newest-section`). See "Newest Panduan (Panduan Terbaru)" below.
3. A contribute CTA strip (`.contribute-cta`) inviting visitors to submit a panduan or komunitas — dashed border (no fill) to read as an invitation rather than content. Centered, `max-width: 40rem`, with `2.5rem` top / `3.5rem` bottom margin so it sits clearly apart from the blue infobox above and the region heading below. Always rendered in the no-search state; buttons link to `/panduan/tambah` and `/komunitas/tambah`, both of which are apex-accessible (see `allowedPaths`). See "Contribute CTA" below.
4. A region selector section `<section class="region-section">` headed **`🗾 Pilih Wilayah`** (emoji anchors the heading to match the two sections above; no container, navigation-style). The heading carries a `2px` amber border-bottom (`#e8c547`, matching Rekomendasi) — intentional bookending so the page reads as amber → blue → amber from top to bottom, rather than the red primary accent which is reserved for action (Cari button, CTA buttons).
5. One `<details class="region-group">` accordion per item in `vm.RegionGroups`.

Each region accordion summary shows:
- `region-title` — the region label (for example, `Kanto`)
- `region-prefs-hint` — a compact comma-separated prefecture list in parentheses

When expanded, the body renders a responsive `.region-pref-grid` of `<article>` cards. Each card corresponds to one prefecture section (`PrefectureSubDomainSection`) and contains:

- **`<h3><a href={s.BaseURL}>{ s.Label }</a></h3>`** — the prefecture name (with emoji) as a link to the prefecture homepage (`s.BaseURL`, e.g. `"https://tokyo.warga.jp"`). The link inherits the card text colour and has no underline at rest; it gains underline on hover. Do not make the `<h3>` plain text — users expect the prefecture name to be clickable.
- A `<nav class="pref-nav"><ul class="nav-list">` listing each category link (Panduan, Komunitas, etc.) with optional approved-content counts appended as plain text `(n)` via `categoryCountLabel`.

**Card interaction design:**
- Cards get a subtle border + background shift on `:hover` (`.region-pref-grid article:hover`) to signal interactivity.
- The `<h3>` link uses `color: inherit; text-decoration: none` at rest and `text-decoration: underline` on hover — it matches the card heading colour regardless of theme, while remaining clearly navigable.

**CSS ownership** (`static/public.css`):
- `.region-group summary[role="button"]` owns the accordion button colors, border, spacing, and chevron placement.
- `.region-prefs-hint` must be styled explicitly for contrast on the red button; do **not** rely on Pico's default muted text color there.
- `.region-pref-grid` remains CSS-driven for responsiveness.

**Important Pico.css override rules for `<summary role="button">`:**

1. **Specificity**: target `.region-group summary[role="button"]` (and its pseudo-elements/states) directly — a generic selector like `.region-group summary::after` is not enough because Pico styles `details summary[role=button]::after` with specificity 0,2,2, which beats a single class + element.

2. **Chevron visibility on light backgrounds**: Pico explicitly sets `filter: brightness(0) invert(1)` on `summary[role=button]:not(.outline)::after`, making the chevron white (designed for dark/colored button backgrounds). When the summary has a white or light background, the chevron becomes invisible. Fix with:
   ```css
   .region-group summary[role="button"]::after {
       filter: none !important;  /* override Pico's white invert */
       margin-left: auto;        /* push chevron to the right edge in flex layout */
   }
   ```
   The `!important` is required because Pico's rule has higher specificity. Without `margin-left: auto`, the chevron may appear in the wrong position when the summary uses `display: flex`.

**Caching note:** `public.css` is served with `Cache-Control: public, max-age=604800, immutable`. In production this is safe because layout templates append `?v=` + `version.Build`, and production builds replace `version.Build` with the current git SHA so each deploy gets a new CSS URL. In local development, `version.Build` is usually just `dev`, so the stylesheet URL does not change between edits; if the browser keeps showing old accordion styles, use a hard refresh or another cache-busting query value.

## Search

### Adding a new content type to search
1. Add a `SearchXxx(db *sql.DB, query string) ([]model.Xxx, error)` function in the relevant repository. Use `WHERE status = 'approved' AND (<fields> LIKE ?) LIMIT 10`.
2. In `presentation/home/handler/handler.go`, call it in **both** `searchAllPrefectures` (cross-prefecture, inside the goroutine) and `searchSinglePrefecture` (prefecture-scoped). Map results to `viewmodel.SearchResult`.
3. Populate `Excerpt` with `viewmodel.HighlightExcerpt(markdown.PlainTextExcerpt(text, 400), query, 150)` — strip markdown first so `<mark>` tags wrap clean text, not raw `**bold**` or `[link](url)` syntax. Import `"website/lib/markdown"`.
4. No viewmodel or templ changes needed — the result list renders generically.

### Query length enforcement
Constants in `presentation/home/handler/handler.go`:
```go
const minQueryLen = 3   // queries shorter than this show a "enter at least 3 chars" hint
const maxQueryLen = 100 // queries longer than this are silently truncated
```
Both limits apply to **both** search paths (cross-prefecture and prefecture-scoped). The `SearchTooShort bool` field on `IndexViewModel` and `PrefIndexViewModel` signals to the template that a too-short query was submitted.

### Excerpt highlighting
`SearchResult.Excerpt` is an **HTML string**, not plain text. It is built by `viewmodel.HighlightExcerpt()` which:
- Truncates to 150 runes
- HTML-escapes all text
- Wraps query matches in `<mark>` tags (styled yellow via `style.templ`)

**Always** render it with `@templ.Raw(result.Excerpt)`. Never use `{ result.Excerpt }` — that would double-escape the HTML entities and show raw `&lt;mark&gt;` tags to the user.

### Cross-prefecture result ordering
After all goroutines return, `searchAllPrefectures` sorts results with `sort.Slice` by:
1. `PrefectureLabel` A→Z
2. `Type` (`"komunitas"` before `"panduan"`)
3. `Title` A→Z

Do not rely on goroutine completion order — it is non-deterministic.

### Layouts — which to use
There are four layouts:
- **`LayoutCategory`** — Prefecture-specific **listing** pages that inject a search bar + category nav sidebar automatically. Used for the per-prefecture komunitas list (`community.templ`), report forms, and disabled feature stubs. **Do not use for detail pages** — it adds an extra sidebar that conflicts with the template's own `.layout` + `.sidebar` structure.
- **`LayoutIndex`** — Homepage-style pages without sidebar (main homepage at `warga.jp`). Signature: `LayoutIndex(title, description, canonicalURL string, loggedIn bool, csrfToken string, user model.User)`.
- **`LayoutGeneric`** — Simple pages with no sidebar or user context (submission forms, privacy policy, error pages, apex panduan/komunitas index). Signature: `LayoutGeneric(title string, noindex bool, current string)`. The `current` string is passed to `Masthead` to highlight the active nav item — pass `""` when no nav item should be highlighted.
- **`LayoutPrefecture`** — Prefecture-scoped pages that need full SEO meta but manage their own layout inside `{ children... }`. Used for: panduan detail, komunitas detail, and the prefecture homepage (`pref_index.templ`). Signature: `LayoutPrefecture(title, description, canonicalURL, current, publishedTime, modifiedTime string)`. Pass `""` for `publishedTime`/`modifiedTime` on listing pages; pass RFC3339 strings on detail pages. Pass `""` for `canonicalURL` when there is no per-page canonical URL.

**Rule: Never use `LayoutCategory` for a page that has its own `.layout` + `.sidebar` HTML structure.** `LayoutCategory` renders its own search bar and category nav sidebar inside the `<main>` element, so nesting another sidebar inside `{ children... }` creates a double-sidebar layout. Use `LayoutPrefecture` instead.

Use `LayoutGeneric` for pages that stand alone and don't need prefecture navigation or user state (e.g. `/kebijakan-privasi`, `/komunitas/tambah`, `/panduan/tambah`).

**Rule:** All three layouts share the same footer. When adding a new footer link, update **all three** files:
- `presentation/shared/layout/layout_generic.templ`
- `presentation/shared/layout/layout_index.templ`
- `presentation/shared/layout/layout_category.templ`

### `LayoutCategory` signature
`LayoutCategory` takes 9 arguments: `title`, `prefectureLabel`, `categoryUrls`, `searchQuery string`, `description string`, `canonicalURL string`, `publishedTime string`, `modifiedTime string`, `current string`.

- **4th argument (`searchQuery`)**: Pass `""` from all listing pages (the search bar is still rendered but empty). `pref_index.templ` no longer uses `LayoutCategory`.
- **5th argument (`description`)**: Pass `""` — `LayoutCategory` is no longer used for detail pages; all detail pages use `LayoutPrefecture`.
- **6th argument (`canonicalURL`)**: Pass `""` — same as above; detail-page canonicals are handled in `LayoutPrefecture`.
- **7th–8th arguments (`publishedTime`, `modifiedTime`)**: Pass `""` — detail pages use `LayoutPrefecture`.
- **9th argument (`current`)**: Nav item to highlight — `"komunitas"`, `"panduan"`, etc. Passed directly to `Masthead`.

When adding a new page that uses `LayoutCategory`, pass `"", "", "", "", ""` for arguments 4–8 and the nav label for argument 9.

### `LayoutPrefecture` signature
`LayoutPrefecture(title, description, canonicalURL, current, publishedTime, modifiedTime string)`.

- **`description`**: Pass `""` when there is no meaningful per-page description (e.g. the prefecture homepage). Non-empty value is emitted as `<meta name="description">` and OG tags.
- **`canonicalURL`**: Pass the full absolute URL on detail pages (triggers `<link rel="canonical">` and `og:type=article`). Pass `""` on listing pages (emits `og:type=website` instead).
- **`current`**: Nav item to highlight — `"panduan"` or `"komunitas"`. Passed to `Masthead`.
- **`publishedTime` / `modifiedTime`**: RFC3339 strings for `article:published_time` / `article:modified_time`; pass `""` on listing pages.

Use `LayoutPrefecture` for any page that needs full SEO meta and manages its own inner layout (e.g. a `.layout` + `.sidebar` + `.content` grid). It is a bare `<main class="wrap">{ children... }</main>` — no search bar, no category sidebar.

### Changing the TTL or cache cap
Constants are at the top of `presentation/home/handler/handler.go`:
```go
const searchCacheTTL  = 5 * time.Minute
const maxCacheEntries = 500
```
Both `globalSearchCache` and `prefSearchCache` use the same values.

### Do not add CSRF to the search form
Search is a `GET` request — CSRF only applies to state-changing `POST` requests. Adding a CSRF token to a GET form breaks links, bookmarks, and browser history.

## Recommended Content (Rekomendasi)

The homepage shows a curated "⭐ Rekomendasi" section above the prefecture grid. Admin manages the list at `/admin/rekomendasi`.

### Data model
`featured_panduan` table in `admin.db` (migration version 3):

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | |
| `prefecture` | TEXT | Prefecture slug (e.g. `"tokyo"`) |
| `post_id` | INTEGER | ID of the `generic_info_post` row |
| `title` | TEXT | Cached at feature time — not kept in sync with edits |
| `url` | TEXT | Full absolute URL, cached at feature time |
| `display_order` | INTEGER | Lower = shown first; normalised to 0,1,2… on every reorder |
| `created_at` | DATETIME | |

`UNIQUE(prefecture, post_id)` prevents duplicates. Repository functions are in `domain/admin/repository/featured.go`.

### Adding a panduan to rekomendasi
From `/admin/panduan?prefecture=<pref>`, click **★ Rekomendasikan** on any approved panduan. The handler (`PanduanFeatureHandler`) snapshots the title and URL into `featured_panduan` at that moment.

### Keeping the featured title in sync
`featured_panduan` stores a **cached copy** of the title and URL. Any operation that changes a panduan's title or slug must also call `adminRepository.SyncFeaturedPanduanTitle(adminDB, pref, postID, newTitle, newURL)` and `homeHandler.InvalidateFeaturedCache()` — otherwise the homepage recommendation will show the stale title indefinitely.

**Current operations that do this:**
- `PanduanEditPostHandler` (`presentation/admin/panduan/handler/handler.go`) — syncs title+URL after every successful edit

**Rule:** If you add a new code path that can change a panduan title (bulk edit, import, etc.), it must follow the same pattern. The URL must be recomputed using `slug.Slugify(title)` with the same format as `PanduanFeatureHandler`:
```go
s := slug.Slugify(newTitle)
var url string
if s == "" {
    url = fmt.Sprintf("https://%s.%s/panduan/%d", pref, constant.DomainName(), id)
} else {
    url = fmt.Sprintf("https://%s.%s/panduan/%d-%s", pref, constant.DomainName(), id, s)
}
_ = adminRepository.SyncFeaturedPanduanTitle(adminDB, pref, id, newTitle, url)
homeHandler.InvalidateFeaturedCache()
```
`SyncFeaturedPanduanTitle` is a no-op if the panduan is not currently featured, so it is safe to call unconditionally.

### Ordering
The `/admin/rekomendasi` page shows ↑/↓ buttons per row. Each click POSTs to `/admin/rekomendasi/{id}/naik` or `/turun`. On every move, **all** `display_order` values are rewritten as sequential integers `0,1,2,…` to prevent ties. The homepage reads rows `ORDER BY display_order ASC, created_at ASC`.

### Homepage display cap
The homepage renders **at most 3** featured items regardless of how many are stored. The cap is applied in `IndexHandler` (`presentation/home/handler/handler.go`) after resolving from cache — the cache itself holds the full list, so admin reordering is unaffected.

### Homepage cache
`IndexHandler` warms `globalFeaturedCache` (a single in-memory struct) at startup. There is **no TTL** — the cache is valid until `InvalidateFeaturedCache()` is called. Every admin mutation that changes the featured list (add, remove, reorder, or title sync) calls `InvalidateFeaturedCache()`, so changes are visible on the next homepage request with no delay.

### CSS classes
All in `static/public.css` under the `/* ─ Rekomendasi` block:

| Class | Element | Purpose |
|---|---|---|
| `.rekomendasi` | `<div>` | CSS grid, 3-col auto-fill, card layout |
| `.rek-card` | `<a>` | Individual card — white bg, border, hover lift |
| `.rek-pref` | `<span>` | Prefecture label — small, muted pill |
| `.rek-foot` | `<div>` | Card footer row (date, meta) |

Section uses the shared `.section-head.amber` heading pattern.

## Newest Panduan (Panduan Terbaru)

The homepage shows a `🆕 Panduan Terbaru` section directly below Rekomendasi — a cross-prefecture feed of the 5 most recently approved panduan. This surfaces platform activity to returning visitors without requiring them to drill into each prefecture.

### Data source
No new repo function — `loadNewestPanduan` in `presentation/home/handler/handler.go` reuses `prefRepository.GetApprovedGenericInfoPostPage(db, newestPerPrefLimit, 0)` with offset `0` to pull the newest N per prefecture. It fans out one goroutine per prefecture (47 total, same pattern as `loadSectionsWithCounts` and `searchAllPrefectures`), merges via channel, sorts by `created_at DESC` (ISO timestamps sort lexicographically), and truncates to `newestDisplayLimit`.

### Constants
```go
const (
    newestPerPrefLimit = 3 // pull from each prefecture
    newestDisplayLimit = 5 // show after merge + sort
)
```

### Display
Compact list of rows inside a blue infobox — each row is `{prefecture pill} {title link}`. Section is hidden when `?search=` is present. Markup lives in `presentation/home/view/index.templ` inside the no-search branch, between the `featured-section` block and the `region-group` accordion.

### Homepage cache
`IndexHandler` warms `globalNewestCache` (single in-memory struct, RWMutex-guarded) at startup. There is **no TTL** — the cache is valid until `InvalidateNewestCache()` is called.

### Invalidation
`InvalidateNewestCache()` must be called alongside every existing `InvalidateCountsCache()` call for panduan mutations. Current call sites in `presentation/admin/panduan/handler/handler.go`:
- Add (`PanduanAddPostHandler`)
- Approve (`PanduanApproveHandler`)
- Reject (`PanduanRejectHandler`)
- Delete (`PanduanDeleteHandler`)

**Rule:** If you add a new admin code path that changes the set of approved panduan in any prefecture DB, it must call both `InvalidateCountsCache()` and `InvalidateNewestCache()` — otherwise the homepage shows stale data.

### CSS classes
All in `static/public.css` under the `/* ─ Terbaru list` block:

| Class | Element | Purpose |
|---|---|---|
| `.terbaru-list` | `<ul>` | Plain list, no bullets; each `<li>` contains one `<a>` |
| `.terbaru-list a` | `<a>` | Flex row with `t-pref`, `t-title`, `t-date` spans |
| `.t-pref` | `<span>` | Prefecture label — small, muted |
| `.t-title` | `<span>` | Panduan title — grows to fill remaining space |
| `.t-date` | `<span>` | Short date or type label — right-aligned, muted |

The same `.terbaru-list` markup is reused for cross-prefecture search results.

**Search excerpt alignment:** `.search-excerpt` is a `<p>` sibling of the `<a>` inside `<li>` (outside the grid). To align it with the title column, use `padding-left: calc(20px + 100px + 16px)` (outer padding + first column + gap). Use `:has()` to reduce the `<a>`'s bottom padding when an excerpt follows:
```css
.terbaru-list a:has(+ .search-excerpt) { padding-bottom: 4px; }
.terbaru-list .search-excerpt { padding: 0 20px 12px calc(20px + 100px + 16px); margin: 0; }
```

Section uses the shared `.section-head.blue` heading pattern.

## Apex page layouts (`/panduan`, `/komunitas`)

Both apex index pages share a two-part structure: a stats header + a content body. All CSS lives in `static/public.css` under the `/* Apex` block.

### Shared header — `.apex-head`

```html
<section class="apex-head">
  <div class="apex-head-text">
    <div class="eyebrow"><span class="pin"></span> Seluruh Jepang</div>
    <h1>...</h1>
    <p class="dek">...</p>
  </div>
  <div class="apex-head-stats">
    <dl>
      <dt>...</dt><dd>{ count }</dd>
      <dt>Prefektur terisi</dt><dd>{ n }<span class="of">/47</span></dd>
    </dl>
  </div>
</section>
```

| Class | Purpose |
|---|---|
| `.apex-head` | Two-column flex row (text left, stats right); stacks on mobile |
| `.apex-head-text .eyebrow` | Small label above heading |
| `.apex-head-text .pin` | Inline emoji marker |
| `.apex-head-text .dek` | Subtitle / lead paragraph |
| `.apex-head-stats dl/dt/dd/.of` | Stats column: `dt` muted label, `dd` large number, `.of` muted `/47` suffix |

Both viewmodels expose `TotalCount int` and `PrefecturesWithContent int` for the stats column.

### Panduan list — `.panduan-list`

```html
<ul class="panduan-list">
  <li>
    <a href="...">
      <div class="pl-main">
        <div class="pl-meta">
          <span class="pl-pref">{ pref label }</span>
          <span class="pl-kind tips">tips</span>  <!-- or "pengalaman", or omit -->
        </div>
        <h3>{ title }</h3>
      </div>
      <div class="pl-stamp"><span class="d">{ day.month }</span>{ year }</div>
    </a>
  </li>
</ul>
```

| Class | Purpose |
|---|---|
| `.panduan-list` | Vertical list of panduan rows |
| `.pl-meta` | Flex row: pref label + kind badge |
| `.pl-pref` | Prefecture label (emoji + name), mono font |
| `.pl-kind` | Kind badge — neutral; `.tips` adds blue background, `.pengalaman` adds amber |
| `.pl-stamp` | Right-aligned date — `.d` = large day.month, sibling text = small year |

`IndexItem.Category` holds the raw category string (`"tips"`, `"pengalaman"`, or `""`). The template applies it as a CSS class: `class={ "pl-kind " + item.Category }`.

`IndexItem.CreatedAtDay` holds `"18.04"` format; `IndexItem.CreatedAtYear` holds `"2026"`.

### Panduan region stack — `.region-stack`

```html
<div class="region-stack">
  <details class="reg" open>  <!-- open for first 2 regions -->
    <summary class="reg-head">
      <span class="reg-title">Kanto</span>
      <span class="reg-count">8 panduan</span>
    </summary>
    <div class="reg-grid">
      <a class="pref" href="..."><span class="pref-flag">🗼</span><span class="pref-name">Tokyo</span><span class="pref-count">2</span></a>
      <a class="pref empty" href="...">...</a>  <!-- count == 0 -->
    </div>
  </details>
</div>
```

Region data comes from `RegionGroups []RegionGroupVM` on `PanduanIndexViewModel`. Each `RegionGroupVM` has `Label`, `TotalCount`, and `Prefectures []PrefectureCount`. Built by `loadPanduanIndex` using `prefectureRepository.RegionOrder` + `prefectureRepository.PrefectureRegion`. Empty prefectures are always included (`.pref.empty`, count `—`).

**Pico conflict:** see § Pico.css conflict patterns → `details summary::after` in a flex context.

| Class | Purpose |
|---|---|
| `.region-stack` | Flex column of `.reg` accordions |
| `.reg` | `<details>` wrapper — white bg, border, rounded corners; `border-color: var(--ink)` when open |
| `.reg-head` | `<summary>` — 3-col grid (`1fr auto 1fr`); `::before` text `▸` on left, title centered, count flush right |
| `.reg-title` | Region name, bold, `text-align: center` (col 2 of grid) |
| `.reg-sub` | Muted mono suffix inside `.reg-title` — "· N prefektur" |
| `.reg-count` | Mono "N panduan" label, `justify-self: end`; `.muted` variant when count is 0 |
| `.reg-grid` | 4-column grid of `.pref` links (2-col on mobile); `gap: 1px`, `background: var(--rule-soft)` creates grid lines |
| `.pref` | Individual prefecture link — emoji flag, name, count in a 3-col grid |
| `.pref.empty` | Greyed-out prefecture with `—` count |

### Komunitas apex — `.kom-list-apex` and `.pref-komunitas-stack`

```html
<ul class="kom-list-apex">
  <li>
    <a href="...">
      <div class="kla-head"><span class="kla-pref">{ pref }</span></div>
      <h3>{ name }</h3>
      <p>{ excerpt }</p>
      <div class="kla-foot"><span class="kla-date">Update { date }</span></div>
    </a>
  </li>
</ul>

<div class="pref-komunitas-stack">
  <div class="pk-group">
    <div class="pk-head">
      <span class="pk-pref">{ pref label }</span>
      <a class="pk-more" href="...">Lihat semua di { name } →</a>
    </div>
    <div class="pk-items">
      <a class="pk-item" href="...">
        <span class="pk-flag">{ emoji }</span>
        <span class="pk-name">{ title }</span>
      </a>
    </div>
  </div>
</div>
```

**Important:** `.kom-list-apex` uses `display: flex` on the `<ul>`. PicoCSS will add bullet markers to each `<li>` unless you explicitly set `list-style: none` on the `<li>` (not just the `<ul>`):
```css
.kom-list-apex li { list-style: none; }
```

| Class | Purpose |
|---|---|
| `.kom-list-apex` | Flex column `<ul>` of community cards; each card is a full-width `<a>` |
| `.kla-head/.kla-pref` | Card header — prefecture label |
| `.kla-foot/.kla-date` | Card footer — "Update {date}" muted label |
| `.pref-komunitas-stack` | Flex column of `.pk-group` prefecture groups |
| `.pk-group` | One prefecture group — `.pk-head` (label + "see all" link) + `.pk-items` grid |
| `.pk-items` | 2-col grid of `.pk-item` compact links |
| `.pk-item` | Emoji flag + community name |

## Contribute CTA (apex homepage)

A dashed-border "Punya info yang berguna…" strip sits between `🆕 Panduan Terbaru` and `🗾 Pilih Wilayah` on the apex homepage (`presentation/home/view/index.templ`). It invites visitors to submit a panduan or komunitas without first drilling into a prefecture subdomain.

### Why the panduan form has a prefecture dropdown
The komunitas submit form already had a prefecture dropdown; the panduan submit form used to read the prefecture from the subdomain only, so it could not be submitted from apex. To let the CTA work end-to-end, the panduan form now mirrors komunitas:

- `presentation/panduan/viewmodel/submit_viewmodel.go` exposes a `Prefectures []model.Prefecture` field on all three constructors (base, error, success). The base constructor pre-selects the first prefecture when none is provided (apex case); the subdomain path continues to pre-select the current prefecture.
- `presentation/panduan/view/submit.templ` renders a `<select name="prefecture" required>` mirroring `presentation/community/view/submit.templ`.
- `presentation/panduan/handler/handler.go` `SubmitPostHandler` reads the prefecture from `r.FormValue("prefecture")` (not from the subdomain context) and validates by looking up the target DB in `prefectureDBMap[selectedPrefecture]` — unknown values fall through to a "Prefecture tidak valid" error render. CSRF, honeypot, knowledge captcha, 64 KB body cap, and 5/hour rate limit are unchanged.

**Rule:** both `/panduan/tambah` and `/komunitas/tambah` must remain in `allowedPaths` (`main.go`) so apex visitors can reach them; removing either breaks the homepage CTA.

### CTA markup and styling
Markup is purely static (no viewmodel fields), so it bakes into the pre-rendered apex homepage byte buffer without additional cache invalidation.

```html
<div class="contrib">
  <h3>Kamu pernah ngurus sesuatu yang belum ada di sini?</h3>
  <p>Tulis panduanmu — supaya warga lain nggak mulai dari nol.</p>
  <div class="contrib-btns">
    <a href="/panduan/tambah" class="btn-outline">＋ Bagikan Panduan</a>
    <a href="/komunitas/tambah" class="btn-outline">＋ Bagikan Komunitas</a>
  </div>
</div>
```

CSS lives in `static/public.css` under the `/* ─ Contrib CTA` block:

| Class | Purpose |
|---|---|
| `.contrib` | Amber-soft background, centered, generous vertical padding |
| `.contrib-btns` | Flex row, `gap: 12px`, wraps on narrow viewports |
| `.btn-outline` | Outline button — ink border, white fill, hover fill |

**Copywriting:** the buttons read `＋ Bagikan Panduan` / `＋ Bagikan Komunitas`. Do not use "Ajukan" (reads bureaucratic). The `＋` prefix carries the "add/create" signal.

## Footer

The footer is rendered by `SiteFooter()` in `presentation/shared/component/chrome.templ`. All three layouts call `@component.SiteFooter()` — to add, remove, or rename a footer link, **only `chrome.templ` needs to change** (one place, not three).

### Links
Current footer links (in order): Beranda · Kontak · Kebijakan Privasi · Log Perubahan · Syarat & Ketentuan. All links use `constant.BaseURL()` so the scheme is correct in dev (`http://`) and production (`https://`).

`/weblogs` ("Log Perubahan") is a **user-facing** changelog of platform changes (new features, privacy policy and terms updates). It belongs in the footer — do not remove it.

### What not to do
- Do **not** put the copyright on the same row as the nav links.
- Do **not** use text-middot separators (`&bull;`) between items — they orphan when the row wraps.

## Root font-size (responsive typography)

`static/public.css` bumps the root font-size on larger viewports:

```css
@media (min-width: 992px) { html { font-size: 17px; } }
@media (min-width: 1400px) { html { font-size: 18px; } }
```

Phones and small tablets keep Pico's default. Every rem-based declaration in the stylesheet scales together, so 14"/16" laptops get breathing room without per-class font-size overrides.

**Rule:** prefer rem over px when writing new styles so future root-size adjustments propagate automatically. If you need a fixed size (e.g. icon sprites, hairlines), use px explicitly and comment why.

## Pico.css conflict patterns

Pico applies opinionated styles to HTML elements that can clash with custom layouts. These patterns appear repeatedly across the codebase — know them before writing new templates.

### `<article>` as a card
Pico styles `<article>` with a card appearance: background, padding, box-shadow, border-radius. If you use `<article>` semantically but don't want the card look, add an explicit reset:
```css
article.my-class { background: none; box-shadow: none; padding: 0; margin-bottom: 0; border-radius: 0; }
```

### `article > header` and `article > footer` negative margins
Pico adds negative horizontal margins + card-section background to direct `<header>` and `<footer>` children of `<article>`. **Fix:** use `<div class="article-head">` and `<div class="article-footer">` instead of `<header>`/`<footer>` inside article elements.

### Form element bottom margins
Pico adds `margin-bottom: var(--pico-spacing)` (≈1rem) to `input`, `select`, and `textarea`. Inside custom form layouts this stacks badly. **Fix:**
```css
.my-form input:not([type="radio"]):not([type="checkbox"]):not([type="hidden"]):not([tabindex="-1"]),
.my-form textarea,
.my-form select {
    margin: 0 !important;
    height: auto !important;  /* Pico also forces ~50px height on inputs */
}
```

### Label bottom margin
Pico adds `margin-bottom: calc(var(--pico-spacing) * .375)` (~6px) to all `label` elements. Inside a flex column form-row with its own `gap`, this doubles the spacing. **Fix:** `label { margin-bottom: 0; }` scoped to the form container.

### Radio/checkbox labels get `width: fit-content`
Pico targets `label:has([type=checkbox],[type=radio])` with `width: fit-content` at specificity `(0,1,1)` (element + `:has()` with attribute selector). A plain class selector like `.topic-opt` is only `(0,1,0)` and loses. **Fix:** use a parent+class selector to reach `(0,2,0)`:
```css
.topic-row .topic-opt { width: auto; display: flex; }
```
This also prevents the item from shrinking out of its CSS grid cell. Apply the same pattern to any clickable label card inside a grid.

### `summary[role=button]::after` chevron forced white
See the existing note in **§ Home page region selector** above — same fix applies everywhere (`filter: none !important; margin-left: auto`).

### `details summary::after` — hiding Pico's chevron and using your own indicator
PicoCSS adds a chevron to **all** `<summary>` elements via `details summary::after { float: right; background-image: var(--pico-icon-chevron); ... }`. When the `<summary>` uses `display: flex` or `display: grid`, `float: right` pulls the chevron out of the flow.

**Preferred approach (used in `.reg-head`):** hide the Pico `::after` entirely and render your own indicator via `::before`:
```css
.region-stack .reg-head::after { display: none; }   /* kill Pico's chevron */
.reg-head::before {
    content: "▸";
    justify-self: start;                             /* grid col 1, left-aligned */
    font-family: var(--mono);
    font-size: 12px;
    color: var(--ink-3);
    display: inline-block;
    transition: transform .15s;
}
.reg[open] .reg-head::before { transform: rotate(90deg); }
```

**Alternative** (if you want to reuse Pico's SVG chevron): override `float` and layout properties with a more-specific class+class selector (specificity 0,2,1 beats Pico's 0,1,3):
```css
.region-stack .reg-head::after {
    content: "";
    background-image: var(--pico-icon-chevron);
    background-position: center;
    background-size: 1rem auto;
    background-repeat: no-repeat;
    width: 1rem; height: 1rem;
    float: none;
    flex-shrink: 0;
    display: inline-block;
    transform: rotate(-90deg);
    transition: transform .2s ease-in-out;
    filter: none;
    opacity: 0.6;
}
.reg[open] .reg-head::after { transform: rotate(0deg); }
```

Also cancel PicoCSS's `details[open] > summary { margin-bottom: var(--pico-spacing) }` with:
```css
.reg[open] > .reg-head { margin-bottom: 0; }
```
And suppress PicoCSS's accordion colour changes on open/focus state so the title stays `var(--ink)`:
```css
.reg-head { color: var(--ink); }
.reg[open] > .reg-head { color: var(--ink); }
.reg-head:focus, .reg-head:focus-visible { outline: none; color: var(--ink); }
```

### `section` default margin-bottom
Pico adds `section { margin-bottom: var(--pico-block-spacing-vertical) }` (1rem) to every `<section>`. This adds ~16px after every section on top of its own padding-bottom, making section gaps larger than the design intends. **Global reset** in `public.css`:
```css
section { margin-bottom: 0; }
```

### `ul > li` list markers inside flex containers
PicoCSS applies `list-style: disc` to `ul > li`. Setting `list-style: none` only on the `<ul>` is not enough when the `<ul>` uses `display: flex` — the browser still renders the `::marker` pseudo-element on each `<li>`.

**Fix:** add `list-style: none` on the `<li>` itself:
```css
.my-flex-list li { list-style: none; }
```

### Design token reference
All custom properties are declared in `:root` in `static/public.css`. Key tokens:

| Token | Value | Use |
|---|---|---|
| `--sans` | Plus Jakarta Sans, system fallback | Body and UI text |
| `--mono` | JetBrains Mono, monospace fallback | Code, labels, dates |
| `--merah` | `#c0392b` | Primary red — CTA buttons, accents |
| `--merah-soft` | `#fcebea` | Red tint backgrounds |
| `--ink` | `#1a1a1a` | Primary text |
| `--ink-2` | `#555` | Secondary text |
| `--ink-3` | `#999` | Muted text, hints |
| `--paper` | `#fafafa` | Input backgrounds |
| `--paper-2` | `#f4f4f4` | Card/section backgrounds |
| `--rule` | `#e2e2e2` | Borders |
| `--amber` | `#f5c542` | Callout accent |
| `--radius` | `8px` | Default border radius |
| `--radius-sm` | `6px` | Small elements (inputs, pills) |
| `--radius-lg` | `12px` | Cards |

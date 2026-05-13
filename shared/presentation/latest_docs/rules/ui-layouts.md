# UI — Layouts, Static Assets & Search

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

### version package
`version/version.go` declares three vars, all injected by `scripts/build-prod.sh` via `-ldflags -X`:

| Var | Default | Production value |
|-----|---------|-----------------|
| `Build` | `"dev"` | short git SHA (`git rev-parse --short HEAD`) |
| `Version` | `"dev"` | semver tag (`git describe --tags --always --dirty`, e.g. `v0.3.1`) |
| `BuildTime` | `""` | UTC ISO 8601 timestamp (`date -u +%Y-%m-%dT%H:%M:%SZ`) |

**Uses:**
- **Cache busting**: all layout templates append `?v=` + `version.Build` to stylesheet `<link>` hrefs, so browsers pick up new CSS on every deploy. Always use `version.Build`, never hardcode paths.
- **DEV indicator**: all three shared layouts render an amber-on-ink **DEV** badge when `version.Build == "dev"`. Never appears in production. Do not remove this check.
- **Admin header badge**: shows `DEV` or `version.Version` (e.g. `v0.3.1`) on production builds.
- **Public footer**: `SiteFooter` appends `version.Version` as a small tag in the copyright line.
- **`/ping` endpoint**: returns JSON `{"version":"v0.3.1","build":"abc1234","build_time":"...","status":"ok"}`.

**Tagging a release:** `git tag v0.3.1` before running `build-prod.sh`. `git describe` picks it up automatically.

---

## Layouts

There are four layouts. Use exactly the one that matches the page type.

| Layout | Signature | Used for |
|---|---|---|
| `LayoutGeneric` | `(title, description, canonicalURL, noindex, current)` — pending struct refactor | Listing pages, forms, legal pages, auth, errors |
| `LayoutIndex` | `(title, description, canonicalURL, noindex)` | Apex homepage only — hardcodes `Masthead("beranda")` |
| `LayoutPrefecture` | `(title, description, canonicalURL, current, publishedTime, modifiedTime, noindex)` — pending struct refactor | Prefecture detail + index pages with full SEO meta |
| `LayoutCategory` | `(props CategoryLayoutProps)` — struct in `presentation/shared/layout/props.go` | **Disabled WIP features only** (marketplace, job, culinary, etc.) — do not use for new pages |

**Rule: Never use `LayoutCategory` for active pages.** It renders a legacy search bar + `NavigationPrefectureComponent` sidebar inside `<main>`. All active public pages use `LayoutPrefecture` with their own `.layout + .sidebar + .content` structure instead. Using `LayoutCategory` on a page that also builds a custom sidebar creates a double-sidebar layout.

Use `LayoutGeneric` for pages that stand alone without prefecture navigation (e.g. `/kebijakan-privasi`, `/komunitas/tambah`, `/panduan/tambah`).

**Rule:** All four layouts share the same `<head>` and footer implementations. When adding a meta tag, favicon, or stylesheet link, edit only `presentation/shared/layout/head.templ` (`PublicHead`). When changing the footer, edit only `presentation/shared/component/chrome.templ` (`SiteFooter`). Do not edit the four layout files individually for either.

### `LayoutPrefecture` signature

`LayoutPrefecture(title, description, canonicalURL, current, publishedTime, modifiedTime string, noindex bool)` — **pending struct refactor** (7 params, violates the ≥5-param rule; see `features.md`).

- **`description`**: Pass `""` on listing pages. Non-empty emits `<meta name="description">` and OG tags.
- **`canonicalURL`**: Full absolute URL on detail pages (triggers `<link rel="canonical">` and `og:type=article`). Pass `""` on listing pages.
- **`current`**: Nav item to highlight — `"panduan"` or `"komunitas"`.
- **`publishedTime` / `modifiedTime`**: RFC3339 strings for article time fields; pass `""` on listing pages.

Use `LayoutPrefecture` for any prefecture page that needs full SEO meta and manages its own inner layout (`.layout` + `.sidebar` + `.content`).

#### Standard `.layout + .sidebar + .content` pages

All active prefecture pages — panduan detail, komunitas detail, and laporkan (report) forms — share the same outer structure. Follow this pattern for any new page:

```
<aside class="sidebar">
    <h5>Navigasi</h5>
    <ul>
        <li><a href="{ baseURL }/">Panduan { Label }</a> <span class="c">{ PanduanCount }</span></li>
        <li><a href="{ baseURL }/komunitas">Komunitas { Label }</a> <span class="c">{ KomunitasCount }</span></li>
    </ul>

    <!-- Panduan detail only: TOC if post has ## headings -->
    <h5>Daftar isi</h5>
    <ol class="toc-sidebar"> ... </ol>

    <h5>Tentang ... ini</h5>
    <ul class="meta-list">
        <li><span class="k">...</span><span class="v">...</span></li>
    </ul>

    <div class="contrib-mini">
        <h5 class="contrib-mini-title">Ada yang salah?</h5>
        <p>...</p>
        <a href="/feature/{id}/laporkan" class="btn-outline sm">Laporkan ...</a>
    </div>
</aside>
```

- Sidebar report link **must** point directly to `/{feature}/{id}/laporkan`, not to a `#section` anchor on the same page.
- Viewmodel must set `PrefectureSubDomainSection.BaseURL`, `.PanduanCount`, and `.KomunitasCount`.
- `.toc-sidebar` is styled in `public.css` with CSS counters; use only for panduan (or any markdown content with `##` headings).
- Report form pages (`/laporkan`) use the same layout: sidebar with navigation + "Konten dilaporkan" meta-list, content with `pref-section-head`, `.form-card`, and `.submit-alert-success/error`.

### `LayoutGeneric` signature

`LayoutGeneric(title, description, canonicalURL, noindex, current)` — **pending struct refactor** (5 params, violates the ≥5-param rule; see `features.md`). Pass `""` for description/canonicalURL to suppress those tags. Pass `noindex=true` for forms, auth, error, and search-result pages. Pass `""` for `current` when no nav item should be highlighted.

---

## Search

### Adding a new content type to search
1. Add a `SearchXxx(db *sql.DB, query string) ([]model.Xxx, error)` function in the relevant repository. Use `WHERE status = 'approved' AND (<fields> LIKE ?) LIMIT 10`.
2. In `presentation/home/handler/handler.go`, call it in **both** `searchAllPrefectures` (cross-prefecture, inside the goroutine) and `searchSinglePrefecture` (prefecture-scoped). Map results to `viewmodel.SearchResult`.
3. Populate `Excerpt` with `viewmodel.HighlightExcerpt(markdown.PlainTextExcerpt(text, 400), query, 150)` — strip markdown first so `<mark>` tags wrap clean text, not raw `**bold**` or `[link](url)` syntax. Import `"website/lib/markdown"`.
4. No viewmodel or templ changes needed — the result list renders generically.

### Query length enforcement
```go
const minQueryLen = 3   // queries shorter than this show a "enter at least 3 chars" hint
const maxQueryLen = 100 // queries longer than this are silently truncated
```
Both limits apply to cross-prefecture and prefecture-scoped search. `SearchTooShort bool` on the viewmodel signals the template.

### Excerpt highlighting
`SearchResult.Excerpt` is an HTML string built by `viewmodel.HighlightExcerpt()`. Always render with `@templ.Raw(result.Excerpt)`. Never use `{ result.Excerpt }` — that double-escapes the HTML entities.

### Cross-prefecture result ordering
After all goroutines return, `searchAllPrefectures` sorts results by: `PrefectureLabel` A→Z → `Type` (`"komunitas"` before `"panduan"`) → `Title` A→Z. Do not rely on goroutine completion order.

### Cache TTL
```go
const searchCacheTTL  = 5 * time.Minute
const maxCacheEntries = 500
```
Both `globalSearchCache` and `prefSearchCache` use these values. Constants live at the top of `presentation/home/handler/handler.go`.

### Do not add CSRF to search
Search is a `GET` request — CSRF only applies to state-changing `POST` requests. Adding a CSRF token to a GET form breaks links, bookmarks, and browser history.

---

## Footer

The footer is rendered by `SiteFooter()` in `presentation/shared/component/chrome.templ`. All three layouts call `@component.SiteFooter()` — to add, remove, or rename a footer link, **only `chrome.templ` needs to change**.

Current footer links (in order): Beranda · Kontak · Kebijakan Privasi · Log Perubahan · Syarat & Ketentuan. All links use `constant.BaseURL()` so the scheme is correct in dev (`http://`) and production (`https://`).

`/weblogs` ("Log Perubahan") is a **user-facing** changelog of platform changes. Do not remove it from the footer.

**Do not** put the copyright on the same row as the nav links. **Do not** use text-middot separators (`&bull;`) between items — they orphan when the row wraps.

The footer panel is **ink** (`background: var(--ink)`) with light-translucent type. All link colors inside `.foot` use `rgba(250,250,247,...)` so they read against near-black. When adding a new footer block, follow this convention or it'll be illegible.

---

## Chrome theme — red masthead, red hero/contrib, ink footer

Locked direction (2026-05). All four chrome surfaces share a single colour story:

| Surface | Background | Notes |
|---|---|---|
| `.masthead` | `var(--merah)` | white wordmark, amber `.dot`, white nav (translucent), white CTA with red text. **No bottom border** — flows flush into the apex hero. |
| `.hero` (apex `homeMode` only) | `var(--merah)` | full-bleed via negative-margin breakout. White type, amber `.accent` on H1. |
| `.hero.hero-compact` (apex `searchMode`) | transparent paper | explicitly resets all red overrides — it sits on cream like the page-head. |
| `.pref-hero` | transparent paper | unrelated class, prefecture pages — never touched by direction-2. |
| `.contrib` (apex contribute band) | `var(--merah)` | same full-bleed breakout. `.btn-outline` inside is overridden white-on-red. |
| `.foot` | `var(--ink)` | light-translucent type. `margin-top: 56px` default; `0` when a `.contrib` is the direct sibling above. |

**Full-bleed breakout pattern** (used by `.hero` and `.contrib`):

```css
margin: 0 calc(50% - 50vw);
padding: <vert> calc(50vw - 50% + 24px) <vert>;
```

The math: `<main class="wrap">` is `max-width: 1120px; padding: 0 24px`. A child's `50%` is half the wrap's content-box width; `50% - 50vw` is the negative offset to the viewport edge. The `+ 24px` in the padding adds back the gutter so internal content stays gutter-aligned with the rest of the page. Children also get `max-width: 1120px; margin-inline: auto` so the breakout doesn't make text spans wider than the rest of the page.

**Do not change the masthead/hero/contrib/footer backgrounds without a design pass** — they are structural, not decorative. If you need a red callout inside content, use `.btn-cta` or `.merah-soft` tints, not another red panel.

---

## Root font-size (responsive typography)

`static/public.css` bumps the root font-size on larger viewports:

```css
@media (min-width: 992px) { html { font-size: 17px; } }
@media (min-width: 1400px) { html { font-size: 18px; } }
```

Phones and small tablets keep Pico's default. Every rem-based declaration scales together.

**Rule:** prefer rem over px when writing new styles. If you need a fixed size (e.g. icon sprites, hairlines), use px explicitly.

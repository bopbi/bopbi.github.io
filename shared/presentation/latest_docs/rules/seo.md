# SEO

## Layout signatures

Four layouts exist. Only use the one that matches the page type.

| Layout | Signature | Used for |
|---|---|---|
| `LayoutGeneric` | `(title, description, canonicalURL, noindex, current)` — pending struct refactor | Listing pages, legal pages, forms, auth, errors |
| `LayoutIndex` | `(title, description, canonicalURL, noindex)` | Apex homepage only — hardcodes `Masthead("beranda")` |
| `LayoutPrefecture` | `(title, description, canonicalURL, current, publishedTime, modifiedTime, noindex)` — pending struct refactor (7 params) | Prefecture detail + index pages |
| `LayoutCategory` | `(props CategoryLayoutProps)` — struct in `presentation/shared/layout/props.go` | Prefecture komunitas category page |

All four layouts emit the full OG/Twitter block, font preload, and favicons via a shared `PublicHead(HeadProps)` component (`presentation/shared/layout/head.templ`). No layout is meta-blind. To add or change any `<head>` content, edit `head.templ` only.

## When to pass what

| Page type | description | canonicalURL | noindex |
|---|---|---|---|
| Detail page (panduan/komunitas) | ≤160-rune excerpt from content | full `https://` URL | `false` |
| Listing page (`/panduan`, `/komunitas`) | Static site-level copy | `"https://" + constant.DomainName() + "/panduan"` | `false` |
| Prefecture index (`<pref>.warga.jp/`) | `vm.MetaDescription` (built in viewmodel) | `vm.BaseURL` | `false` |
| Prefecture search results (`<pref>.warga.jp/?search=…`) | pass through `LayoutPrefecture` with `noindex=true` | `vm.BaseURL` (prefecture root) | `true` |
| Legal/info pages (`/kontak`, `/kebijakan-privasi`, etc.) | Static descriptive copy | `"https://" + constant.DomainName() + "/kontak"` | `false` |
| Submit forms (`/tambah`) | `""` | apex `"https://" + constant.DomainName() + "/<path>/tambah"` (consolidates all subdomain copies) | `true` |
| Auth pages (login, signup) | `""` | `""` | `true` |
| Error pages (404, 500) | `""` | `""` | `true` |
| Search results (`/?search=…`) | pass through `LayoutIndex` with `noindex=true` | `""` | `true` |

## Detail page SEO fields

Viewmodel must populate four fields for `LayoutPrefecture` / `LayoutCategory`:

| Viewmodel field | Value |
|---|---|
| `MetaDescription string` | `markdown.PlainTextExcerpt(content, 160)` — helper in `lib/markdown` |
| `CanonicalURL string` | `"https://" + prefecture.Value + "." + constant.DomainName() + slugPath` |
| `PublishedTime string` | RFC3339 from `CreatedAt` via `formatter.FormatRFC3339` |
| `ModifiedTime string` | RFC3339 from `AdminEditedAt ?? UpdatedAt` via `formatter.FormatRFC3339` |

`og:type` is `"article"` when `canonicalURL` is non-empty **and** the layout sets `IsArticle: true` (`LayoutPrefecture` and `LayoutCategory`). `LayoutGeneric` and `LayoutIndex` always emit `og:type=website` regardless of canonical URL. `PublicHead` handles this automatically via `HeadProps.IsArticle`.

## Meta description length

Truncate to **160 runes** (not bytes — use `[]rune` or `lib/markdown.PlainTextExcerpt`). The `PlainTextExcerpt(content, maxRunes)` helper strips markdown and truncates in one call. Never expose submitter contact fields (email, WhatsApp, LINE) in descriptions — see `forms.md § Contact fields`.

## Slug URLs

Detail URLs use `/{id}-{slug}` (e.g. `/panduan/3-tips-hidup`). The handler parses only the numeric prefix — bare-ID URLs (`/panduan/3`) continue to work.

```go
idStr := chi.URLParam(r, "id")
if i := strings.IndexByte(idStr, '-'); i != -1 {
    idStr = idStr[:i]
}
id, err := strconv.Atoi(idStr)
```

`lib/slug/slug.go` `Slugify(s)` lowercases, keeps ASCII letters/digits, collapses hyphens, caps at 80 chars. Returns `""` for Japanese-only input — always handle the empty-slug fallback.

`constant.DomainName()` is correct for canonical URLs (always `https://`) even in dev, where `constant.BaseURL()` returns `http://`.

## JSON-LD structured data

All components live in `presentation/shared/component/jsonld.templ`.

| Component | Schema type | Used on |
|---|---|---|
| `WebSiteSchema(siteURL, logoURL)` | `WebSite` + SearchAction | Apex homepage only |
| `OrganizationSchema(siteURL, logoURL)` | `Organization` | Apex homepage only — once is enough for Google's knowledge graph |
| `ArticleSchema(headline, canonicalURL, publishedTime, modifiedTime, logoURL)` | `Article` | Panduan + komunitas detail pages |
| `BreadcrumbSchema([]BreadcrumbItem)` | `BreadcrumbList` | Panduan + komunitas detail pages |
| `ItemListSchema([]ListItem)` | `ItemList` | Apex `/panduan` and `/komunitas` listing pages |

`BreadcrumbItem` and `ListItem` both have `Name` and `URL` string fields.

Author/publisher in `ArticleSchema` is always `{ "@type": "Organization", "name": "Warga.jp" }` — never expose submitter info.

### Implementation: `jsonLDScript` helper

Templ treats `<script>` tags as raw text elements (per the HTML spec) — `@` expressions inside them are **not** processed and get written literally as Go source code. All JSON-LD components use a Go helper to build the complete `<script>` tag as a string, then inject it via `@templ.Raw()` at the component's top level (outside any `<script>` tags):

```go
func jsonLDScript(v any) string {
    return `<script type="application/ld+json">` + marshalJSON(v) + `</script>`
}

templ WebSiteSchema(siteURL string, logoURL string) {
    @templ.Raw(jsonLDScript(WebSiteSchemaData{...}))
}
```

**Never** write `@templ.Raw(...)` or any `@expression` inside `<script>...</script>` in a templ file — it will emit literal Go source code to the browser.

## Sitemap (`/sitemap.xml`)

Handler: `presentation/home/handler/sitemap.go`. Cached in-memory with no TTL; invalidated by admin mutations via `InvalidateSitemapCache()`.

Current apex URLs included:

| URL | changefreq | priority |
|---|---|---|
| `/` | weekly | 1.0 |
| `/panduan` | daily | 0.9 |
| `/komunitas` | daily | 0.9 |
| `/weblogs` | weekly | 0.4 |
| `/kontak` | monthly | 0.3 |
| `/syarat-ketentuan` | monthly | 0.3 |
| `/kebijakan-privasi` | monthly | 0.3 |

Per-prefecture entries (generated concurrently): `<pref>.warga.jp/`, `<pref>.warga.jp/komunitas`, all approved panduan detail URLs, all approved komunitas detail URLs. `<lastmod>` comes from `updated_at` via `sitemapDate(s string)` (extracts `YYYY-MM-DD` prefix from SQLite datetime strings).

When adding a new public content type: add its detail URLs to the per-prefecture goroutine block and add any new apex listing URL to the static `urls` slice.

## robots.txt (`/robots.txt`)

Handler: `presentation/home/handler/robots.go`. Disallows `/admin/`. References the sitemap URL. Cached 1 day.

## Heading outline

Follow the `h1 → h2 → h3` hierarchy within page content. The `h1` is the article/page title. `h2` introduces major sections (related posts, CTAs). `h3` is used for card titles within a section. `h4`/`h5` are acceptable inside `<aside>` widgets (TOC, sidebar navigation) — asides are outside the main document outline.

## Font preload

All four layouts include:
```html
<link rel="preload" as="font" type="font/woff2" crossorigin href="/static/fonts/plus-jakarta-sans-variable.woff2">
```
This is already in every layout — do not add it again when working on individual pages.

## Pages duplicated across subdomains (`allowedPaths`)

Some paths are listed in `main.go`'s `allowedPaths` and therefore serve identical HTML on the apex AND on every prefecture subdomain (~49 copies). Examples: `/panduan`, `/komunitas`, `/panduan/tambah`, `/komunitas/tambah`, `/kontak`, `/syarat-ketentuan`, `/kebijakan-privasi`, `/weblogs`.

For these pages, **always pass an apex canonical** — `"https://" + constant.DomainName() + "/<path>"` — so Google understands the ~49 URLs are one canonical resource. Without this, Google reports the prefecture-subdomain copies under "Crawled - currently not indexed" or "Duplicate without user-selected canonical."

When adding a new path to `allowedPaths`, also add an apex canonical to its template.

## GSC: "Indexed, though blocked by robots.txt"

This means Google has the URL in its index (usually from external backlinks or stale history) but is blocked from fetching it. On this site it has appeared for `http://www.warga.jp/`. The cause is **not** the site's own `robots.txt` (which only disallows `/admin/`) — it is that requests for `www.warga.jp` hit a fallback response (Cloudflare's edge or Caddy's catch-all) whose robots policy disallows crawling.

**Fix at the edge, not in app code.** `Caddyfile.prod` has a `www.warga.jp` site block that 301-redirects to `https://warga.jp{uri}`. The DNS record for `www` (see `VPS_SETUP.md`) must point at the VPS for the redirect to take effect. Once both are in place Google sees a real 301 and consolidates the URL into the apex.

If a similar report ever shows up for an unexpected host, check (a) DNS resolves it to the VPS, (b) Caddy has a matching site block, (c) the block returns a real response (200 or a 301 to the canonical), not a default fallback.

## GSC: "Crawled - currently not indexed"

Like "Alternate page with proper canonical tag," this status is **informational, not an error**. Google fetched the page but chose not to index it. Common causes on this site:

- **Submit forms** (`/panduan/tambah`, `/komunitas/tambah`): served on every subdomain, intentionally `noindex`. Apex canonical (added per the section above) consolidates all 49 copies into one signal.
- **Search results** (`/?search=…`): `noindex` by design — see "Alternate page with proper canonical tag" section.

When a new URL appears under this status, check first that (a) the layout passes `noindex=true` if it's a form/auth/error page, and (b) any subdomain duplicates have an apex canonical. If both are correct, the report is just Google describing the system's intended behavior.

## GSC: "Alternate page with proper canonical tag"

This Google Search Console status is **informational, not an error**. It means Google found a URL, saw a canonical tag pointing to a different URL, and correctly chose to index the canonical version instead. It indicates the system is working as intended.

**Why it appears for `?search=…` URLs**: the homepage hero and nasional rail contain `<a href="/?search=…">` shortcut links — Google follows those anchors and discovers the search-result pages. Those pages carry `noindex` + canonical-back-to-root, so Google drops them from the index, then reports them under this status.

**How we handle it**:
1. All search-result pages emit `noindex` + a canonical pointing to the indexable root (apex or prefecture root). Covered by `LayoutIndex` (apex) and `LayoutPrefecture` (prefecture subdomains, `noindex=true` when `SearchQuery != ""` or `SearchTooShort`).
2. All internal `<a href="…?search=…">` links carry `rel="nofollow"` — this signals Google not to crawl those URLs in the first place, reducing future noise in GSC.

**What to do when you see this status**: nothing, if the URLs are all `?search=…` variants. If non-search URLs appear (e.g. form submissions, auth pages), verify their layouts pass `noindex=true`.

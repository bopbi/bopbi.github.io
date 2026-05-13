# Features

## Adding New Features

### Standalone public form (no DB, no prefecture)

For pages like `/kontak` that collect visitor input and send it somewhere (email, webhook) without writing to a DB, the pattern is simpler than a full feature:

- **No domain model or repository** — nothing is persisted.
- **Uses `LayoutGeneric`** — no prefecture sidebar needed.
- **Protections required on every public POST handler** (rate limit, body cap 64 KB, honeypot, CSRF, captcha, field length caps, email validation, enum whitelist) — see `docs/rules/security-auth.md` + `docs/rules/security-validation.md` for the implementation pattern of each.
- **Rate limit**: use a stricter limit than submission forms — `ratelimit.New(3, time.Hour)` for direct-to-admin forms.
- **Fire-and-forget email**: `go notify.Contact(emailCfg, email, message)`.
- **Add to `allowedPaths`** in `main.go` and add "Hubungi Kami" footer link to all three layouts.

See `presentation/contact/` for the reference implementation.

### Public feature
1. Create `domain/<feature>/model/model.go` — define struct
2. Create `domain/<feature>/repository/repository.go` — DB operations; **include `CountApprovedXxx` and `GetApprovedXxxPage` for pagination** (see Pagination checklist below)
3. Create `presentation/<feature>/handler/handler.go` — HTTP handlers; **parse `?page=N`** and pass to viewmodel; **fire `go notify.Submission(...)` after successful saves**; POST handlers must include all 8 protections listed in **Protections required on every public POST handler** (rate limit, body limit, honeypot, CSRF, captcha, field length caps, email validation, enum whitelist)
4. Create `presentation/<feature>/viewmodel/viewmodel.go` — view models; **accept `page int`, include `pagination.Pagination` field**
5. Create `presentation/<feature>/view/<feature>.templ` — template; **include pagination nav** after the item loop
6. Run `templ generate`
7. Update `router/router.go` — add routes; **pass `emailCfg`** to submission handlers
8. Add to `allowedPaths` in `main.go` if the route is on the apex domain (no subdomain)

### Listing + detail pattern
Features with a list page and a detail page follow this pattern (see komunitas and panduan):

- **Listing page**: Shows a short HTML excerpt (150 runes of visible text) of the main text field. Add an `Excerpt string` computed field to the model struct (not stored in DB) and populate it via `markdown.RenderMarkdownExcerpt(content, 150)` — this renders inline markdown (bold, italic, code) but strips `<a>` tags (links become plain text) so the excerpt is safe inside a card `<a>` element. Render with `@templ.Raw(item.Excerpt)` in the template (not `{ item.Excerpt }`, which would double-escape). Import `"website/lib/markdown"`. The title links to the detail page.
- **Secondary / companion cards** (e.g. "Panduan terkait", Rekomendasi, search results): use `markdown.PlainTextExcerpt(content, 120)` instead — strips all markdown syntax and returns plain text. Render with `{ item.Excerpt }` (no `@templ.Raw`). Never use raw `strings.TrimSpace(content)` for card excerpts; blockquote markers (`>`), bold (`**`), and other syntax will appear verbatim.
- **Block-level lines are skipped entirely in both excerpt helpers** (headings, lists, fenced code, blockquotes). The text *inside* a blockquote is not extracted — the whole line is dropped. A post that starts with (or consists only of) blockquotes will produce an empty excerpt in list views. This is intentional: excerpts show only prose text. Do not change `isBlockLine` to include blockquote content in excerpts.
- **Detail page**: Shows the full content using a two-zone layout. Use `LayoutPrefecture` (not `LayoutCategory`) to avoid a double-sidebar; pass `"panduan"` or `"komunitas"` as the `current` arg to highlight the correct nav item. For markdown content, render server-side with `lib/markdown.RenderMarkdown` and inject via `@templ.Raw(vm.ContentHTML)`. The sidebar must populate `PrefectureSubDomainSection.BaseURL`, `PanduanCount`, and `KomunitasCount` (query `CountApprovedGenericInfoPost` + `CountApprovedCommunity` in the viewmodel).
  - **Panduan detail structure**: article head (`kind-row`, `h1`, `byline-row`) is **full-width** above the grid — no dek/subtitle; `PlainTextExcerpt` produces unreliable results for structured articles so the field was removed from `PanduanDetailViewModel`, wrapped in `<article class="guide-article"><div class="article-head">`. Below it, `<div class="article-body layout">` starts the two-column grid with `<aside class="sidebar">` (TOC via `.toc-sidebar` if `##` headings exist, metadata, report widget) and `<div class="prose content">` for the body. The `<div class="article-footer">` (related posts + CTA) is full-width after the grid, outside `.article-body`.
  - **Komunitas detail structure**: uses the standard `.layout` + `.sidebar` + `.content` grid without a separate full-width article head.
  - Standard sidebar sections: content-specific extras (e.g. TOC for panduan), metadata (category/dates/reading time), and a report widget linking directly to `/{feature}/{id}/laporkan`. No "Navigasi" nav-link section on panduan detail.
- **Contact info**: Never shown on public listing or detail pages — admin panel only. See **Form Design → Contact fields**.
- **Route order in router**: Register `/feature/tambah` **before** `/feature/{id}` to ensure chi matches the static segment first. (Chi resolves static over wildcard, but keeping this order is safer and more readable.)
- **Pagination**: Every listing page must use `lib/pagination`. See the full checklist below.

### Pagination checklist (required for every new listing page)

Every public listing page must be paginated. Page size is controlled by `pagination.PageSize` (currently 10). Follow this checklist:

**1. Repository — add two functions:**
```go
// Count approved rows (single SELECT COUNT(*))
func CountApprovedXxx(db *sql.DB) int {
    var count int
    db.QueryRow("SELECT COUNT(*) FROM xxx WHERE status = 'approved'").Scan(&count)
    return count
}

// Fetch one page of approved rows
func GetApprovedXxxPage(db *sql.DB, limit, offset int) ([]model.Xxx, error) {
    rows, err := db.Query(`
        SELECT ... FROM xxx
        WHERE status = 'approved'
        ORDER BY created_at DESC
        LIMIT ? OFFSET ?
    `, limit, offset)
    // ... scan rows
}
```

**2. Viewmodel — add `Pagination` field and accept `page int`:**
```go
import "website/lib/pagination"

type XxxViewModel struct {
    // ... existing fields
    Pagination pagination.Pagination
}

func NewXxxViewModel(prefecture model.Prefecture, db *sql.DB, page int) XxxViewModel {
    total := repository.CountApprovedXxx(db)
    pag := pagination.New(page, total, pagination.PageSize)
    offset := pagination.Offset(pag.CurrentPage, pagination.PageSize)

    items, err := repository.GetApprovedXxxPage(db, pagination.PageSize, offset)
    // ... populate items

    return XxxViewModel{
        // ...
        Pagination: pag,
    }
}
```

**3. Handler — parse `?page=N` and pass to viewmodel:**
```go
page := 1
if p, err := strconv.Atoi(r.URL.Query().Get("page")); err == nil && p > 1 {
    page = p
}
vm := viewmodel.NewXxxViewModel(prefecture, db, page)
```

**4. View — add pagination nav after the item loop:**
```templ
import "fmt"

if vm.Pagination.TotalPages > 1 {
    <nav class="pagination">
        if vm.Pagination.HasPrev {
            <a href={ templ.URL(fmt.Sprintf("?page=%d", vm.Pagination.CurrentPage-1)) } role="button" class="outline">← Sebelumnya</a>
        }
        <span class="pagination-info">
            Halaman { fmt.Sprintf("%d", vm.Pagination.CurrentPage) } dari { fmt.Sprintf("%d", vm.Pagination.TotalPages) }
        </span>
        if vm.Pagination.HasNext {
            <a href={ templ.URL(fmt.Sprintf("?page=%d", vm.Pagination.CurrentPage+1)) } role="button" class="outline">Selanjutnya →</a>
        }
    </nav>
}
```

The pagination nav is only rendered when `TotalPages > 1`, so pages with fewer than 10 items show no nav at all. Relative `?page=N` links work correctly on any prefecture subdomain without knowing the full URL.

## Unit Testing
- Test files go in the same package as the code under test (e.g. `package repository`)
- File location: `domain/<feature>/repository/repository_test.go`
- Use `github.com/DATA-DOG/go-sqlmock` to mock `*sql.DB` — do not spin up a real SQLite DB in tests
- Use a `setupTestDB(t *testing.T)` helper that calls `sqlmock.New()` and registers `t.Cleanup` to close the DB
- Use table-driven tests with `t.Run` for functions that have multiple input/output variants
- Functions that don't touch the DB (e.g. pure slice helpers) can be tested directly without mocks

## Templ Patterns
- Import viewmodel: `import "website/presentation/<feature>/viewmodel"`
- Import layout: `import "website/presentation/shared/layout"`
- Use `fmt.Sprintf()` for dynamic URLs
- Run `templ generate` after creating/modifying `.templ` files
- **Minimize JavaScript usage** - Prefer server-side rendering; use JavaScript only when absolutely necessary
- **Pico conflicts** — Pico styles `<article>` as a card and `article > header/footer` with negative margins. Use `<div>` wrappers inside article instead of `<header>`/`<footer>`. Reset article card styling with `background:none; box-shadow:none; padding:0` when using article semantically. See `docs/rules/ui-pico-conflicts.md` for the full list of overrides.

### Prefer structs over many parameters
When **any** Go function has **5 or more parameters**, define a named struct instead — regardless of parameter type. Call sites use named fields, which are self-documenting and impossible to silently reorder.

The real risk is adjacent parameters of the same type (e.g. three consecutive `string` args) that a compiler won't catch if swapped. A struct makes each argument's intent explicit at the call site.

**Wrong** — positional args, easy to misorder silently:
```go
func NewPrefIndexViewModel(prefecture model.Prefecture, db *sql.DB, searchQuery string,
    searchResults []SearchResult, tooShort bool, page int,
    allSections []model.PrefectureSubDomainSection) PrefIndexViewModel {
```

**Correct** — named struct, zero values are safe defaults:
```go
type PrefIndexParams struct {
    Prefecture    model.Prefecture
    DB            *sql.DB
    SearchQuery   string
    SearchResults []SearchResult
    TooShort      bool
    Page          int
    AllSections   []model.PrefectureSubDomainSection
}

func NewPrefIndexViewModel(p PrefIndexParams) PrefIndexViewModel {
```

```go
// call site — named fields, impossible to silently swap
viewmodel.NewPrefIndexViewModel(viewmodel.PrefIndexParams{
    Prefecture:    prefecture,
    DB:            prefectureDB,
    SearchQuery:   query,
    SearchResults: searchResults,
    TooShort:      tooShort,
    Page:          page,
    AllSections:   enrichedSections,
})
```

**Naming and placement:**
- Plain Go (viewmodel/handler/repository): name `<Func>Params`, place in the same `.go` file as the function.
- Templ functions: name `<Func>Props`, place in a separate `props.go` file (not inside the `.templ` file) to avoid forcing a regeneration when the struct changes.

**Existing precedents:** `IndexParams`, `PrefIndexParams` (`presentation/home/viewmodel/viewmodel.go`), `ReportParams` (`presentation/report/viewmodel/viewmodel.go`), `SignupParams` (`domain/user/repository/repository.go`), `FeaturedPanduanParams` (`domain/admin/repository/featured.go`), `CategoryLayoutProps` (`presentation/shared/layout/props.go`).

### Conditional text rendering
**Never** write conditional strings as raw text inside a templ `if/else` block. The emoji and the following word can be split into separate text nodes, causing the text portion to appear invisible in the browser.

**Wrong:**
```templ
<span>
    if result.Type == "komunitas" {
        🤝 Komunitas
    } else {
        🔰 Panduan
    }
</span>
```

**Correct:** use a Go helper function and render with `{ }` interpolation:
```go
func typeLabel(t string) string {
    if t == "komunitas" {
        return "🤝 Komunitas"
    }
    return "🔰 Panduan"
}
```
```templ
<span>{ typeLabel(result.Type) }</span>
```

This rule applies to any conditional string — not just emoji+text combinations. When in doubt, compute the string in Go and interpolate it.

## Router conventions

**Use `r.Route(prefix, fn)` whenever 3 or more routes share a path prefix.** Flat `r.Get("/admin/komunitas/...")` lists are only acceptable for one-off endpoints.

**Use `r.Group(fn)` to share middleware** across a span of routes without a common prefix. Currently used for the protected `/admin` sub-group; reach for it any time a future middleware (auth, rate limit, content negotiation) should apply to a span of sibling routes.

**Cross-cutting concerns belong in middleware, not in handlers.** When the same check must run at the top of every handler in a span of routes — auth, a rate-limit guard, request validation — mount it once as middleware on the enclosing `r.Group` / `r.Route` rather than copy-pasting an `if !check(...) { return }` block into each handler. Canonical examples: the global rate limiter (`globalLimiter.Middleware` in `main.go`) and `adminMiddleware.RequireAdmin` (on the protected `/admin` sub-group). Per-handler bespoke limits (custom Indonesian copy, per-endpoint thresholds) are the exception — see `security-auth.md` § Rate limiting.

**Nest `r.Route` for resource grouping under a section.** Inside `/admin`, each resource (`/komunitas`, `/panduan`, `/laporan`, etc.) gets its own nested `r.Route`. The handler instance (`h := adminXxxHandler.NewHandler(...)`) is declared inside the closure when single-use, or just above the sub-route when shared between siblings (e.g. `featuredHandlers` is declared once above both `/rekomendasi` and `/rekomendasi-komunitas`).

**Static assets go through `registerStaticFile`.** Don't hand-roll new `r.Get(path, func(w, r){ ... })` blocks for embedded static assets — call `registerStaticFile(r, urlPath, fsPath, contentType, cacheControl)`, picking one of `cacheStatic` / `cacheFont` / `cacheOG`. If a new asset needs a different cache policy, add a named constant rather than inlining the literal.

**`/tambah` before `/{id}`** — register static segments before wildcard segments inside any `r.Route` block. Chi resolves static first regardless, but explicit ordering is more readable.

**Startup side-effects stay outside `r.Route` blocks.** Calls like `WarmCommunityIndex` / `WarmPanduanIndex` run once at startup, not per-request. Placing them inside a route closure obscures this.

**Canonical example: `router/router.go`** — refactored to this pattern; mirror its shape when adding new routes.

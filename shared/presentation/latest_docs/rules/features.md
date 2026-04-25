# Features

## Adding New Features

### Standalone public form (no DB, no prefecture)

For pages like `/kontak` that collect visitor input and send it somewhere (email, webhook) without writing to a DB, the pattern is simpler than a full feature:

- **No domain model or repository** — nothing is persisted.
- **Uses `LayoutGeneric`** — no prefecture sidebar needed.
- **Protections required on every public POST handler** (rate limit, body cap 64 KB, honeypot, CSRF, captcha, field length caps, email validation, enum whitelist) — see `docs/rules/security.md` for the implementation pattern of each.
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
- **Detail page**: Shows the full content. For markdown content, render it server-side with `lib/markdown.RenderMarkdown` and inject via `@templ.Raw(vm.ContentHTML)` in the templ template. Use `LayoutPrefecture` (not `LayoutCategory`) to avoid a double-sidebar; pass `"panduan"` or `"komunitas"` as the `current` arg to highlight the correct nav item.
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
- **Pico conflicts** — Pico styles `<article>` as a card and `article > header/footer` with negative margins. Use `<div>` wrappers inside article instead of `<header>`/`<footer>`. Reset article card styling with `background:none; box-shadow:none; padding:0` when using article semantically. See `docs/rules/content.md` § Pico.css conflict patterns for the full list of overrides.

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

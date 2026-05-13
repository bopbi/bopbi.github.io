# Rules & Conventions

Project rules split by topic. Read the one that matches your task instead of scanning the full set.

## Files

| File | Scope |
|---|---|
| [`general.md`](./general.md) | Cross-cutting session rules — never-auto-commit, no user-supplied image URLs. |
| [`security.md`](./security.md) | Auth, CSRF, rate limiting, honeypot, captcha, body size, field length caps, email validation, enum whitelisting, markdown sanitising, subdomain context key, error helpers, `allowedPaths`, trailing slash, admin logout, security headers, CSP, `admin.js`. |
| [`forms.md`](./forms.md) | Form design (one field per data point), submitter contact privacy rule, mandatory/optional label markers, inline alerts, reports/laporan status flow, email notifications, markdown preview / edit-lagi flow. |
| [`admin.md`](./admin.md) | Admin dashboard counts, admin page chrome (nav, favicon, auth guard), admin action button colour system, admin detail pages, prefecture selector pattern, article edit policy. |
| [`features.md`](./features.md) | Adding new public features, standalone forms, listing + detail pattern, pagination checklist, templ conventions, unit testing. |
| [`content.md`](./content.md) | Static assets (Pico, fonts, cache busting), design tokens, Pico conflict patterns, home page sections, search, layouts, Rekomendasi, Newest Panduan, Contribute CTA, footer, responsive root font-size. |
| [`infrastructure.md`](./infrastructure.md) | DB migrations, nullable-column scan helpers, HTTP caching headers + homepage caches, content page caching (`lib/cache`), legal pages (privacy/terms + changelog), SEO (slug URLs, meta description, OpenGraph). |

## If you're working on…

| Task | Read |
|---|---|
| Adding a new public POST form | `security.md` (all of Auth Patterns) + `forms.md` |
| Adding a new listing + detail page | `features.md` § Public feature + Pagination + `infrastructure.md` § Content Page Caching |
| Re-enabling user login / signup | `../SECURITY_TODO.md` first, then `security.md` |
| Debugging "Invalid CSRF token" | `security.md` § CSRF three-step checklist |
| Anything in the admin panel | `admin.md` |
| Changing a legal page | `infrastructure.md` § Legal Pages |
| Adding a DB column or new table | `infrastructure.md` § Database Migrations + § Nullable Columns |
| Editing the apex homepage | `content.md` § Home page region selector + § Rekomendasi + § Newest Panduan + § Contribute CTA |
| Editing `layout_*.templ` | `content.md` § Layouts + § Footer |
| Writing `.templ` files | `features.md` § Templ Patterns |
| CSS styling / Pico conflicts / design tokens | `content.md` § Pico.css conflict patterns + § Design token reference |
| Re-ordering / styling admin action buttons | `admin.md` § Admin action button colour system |
| Adding a new search content type | `content.md` § Search |
| Adjusting cache TTLs or invalidation | `infrastructure.md` § HTTP Caching + § Content Page Caching |
| Writing tests | `features.md` § Unit Testing |
| Adding SEO to a new detail page | `infrastructure.md` § SEO |
| Building / deploying to production | `../TOOLS.md` § Production Build |

## Common Task Checklists

### New public submission form
1. Create `domain/`, `presentation/` packages (model, repository, handler, viewmodel, view)
2. GET handler: generate viewmodel with `CSRFToken`, `CaptchaQuestion`, `CaptchaAnswer`; call `adminRepository.SetCSRFTokenCookie(w, vm.CSRFToken)` before rendering
3. POST handler in order: body limit → honeypot → CSRF → rate limit → captcha → field caps → email validation → enum whitelist → DB write → `go notify.Submission(...)` — see `security.md` for each
4. Viewmodel: expose `ErrorMsg string`, `SuccessMsg string`; re-set CSRF cookie on every error re-render
5. Template: honeypot `<input name="website">`, CSRF hidden input, captcha fields, alert boxes — see `forms.md`
6. Every `<label>` gets a `<span class="opt">wajib</span>` or `<span class="opt">opsional</span>` marker matching handler validation — see `forms.md` § Mandatory / optional indicators
7. If the form collects submitter contact (email/WhatsApp/LINE), keep those fields under a clearly-private section and never expose them on public pages — see `forms.md` § Contact fields
8. `router/router.go`: register GET + POST; register `/tambah` before `/{id}`
9. `main.go` `allowedPaths`: add path if apex-accessible

### New DB column
1. Add `runMigration` in `migrations/migration.go` guarded by `columnExists`
2. Update `migrations/export.go`: SELECT, `rows.Scan`, INSERT cols, INSERT values, CREATE TABLE DDL
3. Scan nullable TEXT columns via `sql.NullString` — see `infrastructure.md` § Nullable Columns

### New listing page with pagination
1. Repository: `CountApprovedXxx(db)` + `GetApprovedXxxPage(db, limit, offset int)`
2. Viewmodel: `Pagination pagination.Pagination` field; accept `page int` arg
3. Handler: parse `?page=N`; wrap response in `lib/cache` (key `"<pref>:<page>"`, 5-min TTL)
4. Template: pagination nav guarded by `vm.Pagination.TotalPages > 1`
5. Detail page SEO: populate `MetaDescription`, `CanonicalURL`, `PublishedTime`, `ModifiedTime` — see `infrastructure.md` § SEO

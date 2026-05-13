# Rules & Conventions

Project rules split by topic. Read the one that matches your task instead of scanning the full set.

## ⚠ Hard Rules (every session, no exceptions)

- **Never run `git commit` (or any git write command) without explicit user instruction.** This includes staging files (`git add`), committing, pushing, or amending.
- **Do not re-enable user-supplied image URLs under any circumstances.** No `image_urls` column, no removal of `html.SkipImages`, no new form fields accepting external image URLs. External image URLs leak visitor IPs and load untrusted content. If image support is ever needed, images must be uploaded to self-hosted storage — never loaded from a URL typed by the user.

## Files

| File | Scope |
|---|---|
| [`../DESIGN.md`](../DESIGN.md) | Design system: type, color tokens, component patterns, layout rules, copywriting tone, don'ts. Read before adding any new UI surface. |
| [`../schema.md`](../schema.md) | Full DB schema: all 4 SQLite files (app.db, admin.db, user.db, analytics.db), content privacy rule, domain models. |
| [`../routes.md`](../routes.md) | Full route table: public routes, admin routes, subdomain routes. |
| [`../analytics.md`](../analytics.md) | Analytics: middleware, search/captcha tracking, dashboard, WAL + covering indexes, pruning cron, deployment notes. |
| [`../deploy.md`](../deploy.md) | Production build & deploy: CGo cross-compilation, prerequisites, `build-prod.sh`, CSS versioning, `deploy.sh`, Caddyfile deploy. |
| [`../vps-operations.md`](../vps-operations.md) | VPS operations: SQLite backup cron, maintenance mode, analytics pruning, verify-deployment checklist. |
| [`../SECURITY.md`](../SECURITY.md) | Security model overview, new-handler checklist, audited-clean inventory. |
| [`../security-audit-lessons.md`](../security-audit-lessons.md) | Nine lessons from the codebase security audit (the *why* behind each rule). |
| [`admin-dashboard.md`](./admin-dashboard.md) | Admin action queue (pending counts, filter+sort), VPS health indicator, Kontak management. |
| [`admin-list-pages.md`](./admin-list-pages.md) | Prefecture chip selector, admin search, action button colour system, empty-state phrasing, detail pages, public-URL actions, cross-prefecture move, prefecture selector, CSRF on admin forms. |
| [`admin-layout.md`](./admin-layout.md) | Stylesheet split, admin nav bar, favicon, auth guard middleware, adding a new admin page. |
| [`admin-js.md`](./admin-js.md) | `static/admin.js` behaviours: prefecture select, confirm dialog, clipboard copy; which templates load it. |
| [`admin-edit-policy.md`](./admin-edit-policy.md) | Article edit policy: who can edit, AnyStatus variant, approval side-effect, `admin_edited_at`. |
| [`security-auth.md`](./security-auth.md) | Auth patterns, session cookies, admin route gate, rate limiting (`lib/ratelimit`), brute-force protection, password length, session revocation, honeypot, captcha (`lib/captcha`), body size caps, context key, error helpers. |
| [`security-validation.md`](./security-validation.md) | CSRF three-step checklist, field length caps, email validation, enum whitelisting, markdown sanitising (`SkipHTML`), `allowedPaths`, trailing-slash routing, admin logout, contact-field isolation. |
| [`security-headers.md`](./security-headers.md) | Security headers table, full CSP value, no-inline-styles rule, no-inline-scripts rule. |
| [`infra-db.md`](./infra-db.md) | `constant.BaseURL()` / `KBRITokyoURL` / `response.Init()`, DB migrations (`runMigration`, idempotency via `schema_migrations`), adding a new prefecture, export tool checklist, nullable-column scan helpers. |
| [`infra-caching.md`](./infra-caching.md) | HTTP caching headers table, homepage no-TTL caches, ETag pattern, `Cache-Control` guidelines, content page caching (`lib/cache`), invalidation wiring table. |
| [`infra-content.md`](./infra-content.md) | Legal pages checklist (privacy/terms), "last updated" date constants, `/weblogs` changelog entries. |
| [`forms.md`](./forms.md) | Form design (one field per data point), submitter contact privacy rule, mandatory/optional label markers, inline alerts, reports/laporan status flow, email notifications, markdown preview / edit-lagi flow. |
| [`features.md`](./features.md) | Adding new public features, standalone forms, listing + detail pattern, pagination checklist, templ conventions, unit testing. |
| [`homepage.md`](./homepage.md) | Homepage section order, Rekomendasi, Panduan Terbaru, Nasional Rail, Contribute CTA, their caches and CSS classes. |
| [`apex-listing-pages.md`](./apex-listing-pages.md) | Apex listing page layouts (`/panduan`, `/komunitas`): `.apex-head`, national pin, `.panduan-list`, `.region-stack`, komunitas classes, `?tipe=` filter. |
| [`ui-layouts.md`](./ui-layouts.md) | Static assets (Pico.css, version/cache-busting), layouts (which to use + signatures), search, footer, chrome theme, root font-size. |
| [`ui-pico-conflicts.md`](./ui-pico-conflicts.md) | Pico.css conflict patterns: article cards, form margins, label width, chevron, pill search bar, flex list markers. |
| [`ui-tokens.md`](./ui-tokens.md) | Design token reference (`:root` vars in `base.css`): fonts, colours, radius. |
| [`seo.md`](./seo.md) | Layout SEO signatures, noindex policy, detail page fields, JSON-LD schemas, sitemap, robots.txt, slug URLs, heading outline, font preload. |
| [`performance.md`](./performance.md) | Multi-prefecture fan-out, SQLite connection pool, hot-table indexes, list-view SELECT truncation, response caching for aggregate endpoints, context propagation, singleflight, in-memory IP-map sweepers, single-writer pattern, parallel startup. |

## If you're working on…

| Task | Read |
|---|---|
| Adding any new route | `features.md` § Router conventions |
| Adding a new public POST form | `security-auth.md` + `security-validation.md` + `forms.md` |
| Adding a new listing + detail page | `features.md` § Public feature + Pagination + `infra-caching.md` § Content Page Caching |
| Re-enabling user login / signup | `security-auth.md` (all of Auth Patterns) |
| Debugging "Invalid CSRF token" | `security-validation.md` § CSRF three-step checklist |
| Admin dashboard / action queue | `admin-dashboard.md` |
| Anything in the admin panel | `admin-list-pages.md` + `admin-layout.md` (pick the section that matches) |
| Adding a new admin list page | `admin-list-pages.md` § Prefecture chip selector + § Admin search + § CSRF on admin mutation forms |
| Admin action button colours | `admin-list-pages.md` § Admin action button colour system |
| Editing `admin.js` / adding admin-side JS behaviour | `admin-js.md` |
| Article edit policy | `admin-edit-policy.md` |
| Changing a legal page | `infra-content.md` § Checklist + § Updating the "last updated" date + § Announcing changes |
| Adding a DB column or new table | `infra-db.md` § Database Migrations + § Nullable Columns |
| Editing the apex homepage | `homepage.md` (all sections + their caches) |
| Editing apex listing pages (`/panduan`, `/komunitas`) | `apex-listing-pages.md` |
| Editing `layout_*.templ` | `ui-layouts.md` § Layouts + § Footer |
| Writing `.templ` files | `features.md` § Templ Patterns |
| CSS styling / Pico conflicts | `ui-pico-conflicts.md` |
| Design tokens | `ui-tokens.md` |
| Adding a new search content type | `ui-layouts.md` § Search |
| Adjusting cache TTLs or invalidation | `infra-caching.md` |
| Writing tests | `features.md` § Unit Testing |
| Adding SEO to a new detail page | `seo.md` § Detail page SEO fields |
| Adding a new content type to the sitemap | `seo.md` § Sitemap |
| Editing layout SEO params | `seo.md` § Layout signatures |
| Building / deploying to production | `../deploy.md` |
| Provisioning a new VPS / server setup | `../VPS_SETUP.md` + `../vps-operations.md` |
| VPS backup / maintenance mode / verify deployment | `../vps-operations.md` |
| Understanding or modifying analytics | `../analytics.md` |
| Adding a repository function that loops all prefectures | `performance.md` § Multi-prefecture fan-out |
| Adding a new SQLite table to prefecture DBs | `infra-db.md` § Database Migrations + `performance.md` § Indexes + § List-view SELECT |
| Adding middleware that writes per-request data to SQLite | `performance.md` § Fire-and-forget DB writes |
| Adding a new shared in-memory cache | `infra-caching.md` § Content Page Caching + `performance.md` § Singleflight + § Response caching |
| Adding a new IP-keyed map or rate limiter | `performance.md` § In-memory IP-keyed maps |
| Checking when to upgrade the VPS / debugging system health | `../SCALING.md` § Trigger Thresholds + § In-Product Health Indicator (`/admin/sistem`) |
| Security headers or CSP | `security-headers.md` |
| Reviewing the security model or audit lessons | `../SECURITY.md` + `../security-audit-lessons.md` |

## Common Task Checklists

### New public submission form
1. Create `domain/`, `presentation/` packages (model, repository, handler, viewmodel, view)
2. GET handler: generate viewmodel with `CSRFToken`, `CaptchaQuestion`, `CaptchaAnswer`; call `adminRepository.SetCSRFTokenCookie(w, vm.CSRFToken)` before rendering
3. POST handler in order: body limit → honeypot → CSRF → rate limit → captcha → field caps → email validation → enum whitelist → DB write → `go notify.Submission(...)` — see `security-auth.md` + `security-validation.md` for each
4. Viewmodel: expose `ErrorMsg string`, `SuccessMsg string`; re-set CSRF cookie on every error re-render
5. Template: honeypot `<input name="website">`, CSRF hidden input, captcha fields, alert boxes — see `forms.md`
6. Every `<label>` gets a `<span class="opt">wajib</span>` or `<span class="opt">opsional</span>` marker matching handler validation — see `forms.md` § Mandatory / optional indicators
7. If the form collects submitter contact (email/WhatsApp/LINE), keep those fields under a clearly-private section and never expose them on public pages — see `forms.md` § Contact fields
8. `router/router.go`: register GET + POST; register `/tambah` before `/{id}`
9. `main.go` `allowedPaths`: add path if apex-accessible

### New admin list page
1. Repository: `GetAll<Xxx>WithPending(db)` for the full list; `Search<Xxx>WithPending(db, q string)` for text search — use `lib/search.LikePattern(q)` and `LIKE ? ESCAPE '\'` for each text column
2. Viewmodel: add `Query string` field; accept `q string`; branch to the search function when `q != ""`; add `Chips []shared.PrefectureChip` and `CSRFToken string`
3. Handler: `q := search.SanitizeQuery(r.URL.Query().Get("q"))`; pass `q` into the viewmodel; call `adminRepository.SetCSRFTokenCookie(w, vm.CSRFToken)` before render
4. Template: `@sharedview.AdminNav("key")` right after `<main class="container">` (add the key to `nav.templ` if this is a new page — see `admin-layout.md` § Consistent nav bar); then `@sharedview.PrefectureChips(vm.Chips, vm.SelectedPrefecture, "/admin/<path>")` and `@sharedview.AdminSearchBar(vm.Query, vm.SelectedPrefecture, "/admin/<path>", "Cari <thing>…")` inside `<article>`, above the CTA and table
5. Empty state: branch on `vm.Query != ""` → "Tidak ada hasil untuk …" vs the default "Belum ada … di …" — see `admin-list-pages.md` § Empty-state phrasing
6. CSRF: every `<form method="POST">` in the action column carries `<input type="hidden" name="csrf_token" value={ vm.CSRFToken }/>` and the corresponding handler calls `adminRepository.VerifyCSRFToken(w, r)` — see `admin-list-pages.md` § CSRF on admin mutation forms

### New DB column
1. Add `runMigration` in `migrations/migration.go` (idempotent — `schema_migrations` ensures it runs exactly once)
2. Update `migrations/export.go`: SELECT, `rows.Scan`, INSERT cols, INSERT values, CREATE TABLE DDL
3. Scan nullable TEXT columns via `sql.NullString` — see `infra-db.md` § Nullable Columns

### New listing page with pagination
1. Repository: `CountApprovedXxx(db)` + `GetApprovedXxxPage(db, limit, offset int)`
2. Viewmodel: `Pagination pagination.Pagination` field; accept `page int` arg
3. Handler: parse `?page=N`; wrap response in `lib/cache` (key `"<pref>:<page>"`, 5-min TTL)
4. Template: pagination nav guarded by `vm.Pagination.TotalPages > 1`
5. Detail page SEO: populate `MetaDescription`, `CanonicalURL`, `PublishedTime`, `ModifiedTime` — see `seo.md` § Detail page SEO fields

# Warga.jp - Project Context

## Overview
Warga.jp is a **community platform** for Indonesian citizens living in Japan. Its purpose is to ease access to information, strengthen ties between Indonesians across prefectures, and support day-to-day life abroad.

Users can browse communities and guides organised by Japanese prefecture, and submit new content which requires admin approval before being displayed.

## Tech Stack
- **Language:** Go
- **Router:** github.com/go-chi/chi/v5
- **Templating:** github.com/a-h/templ (Templ)
- **Database:** SQLite (github.com/mattn/go-sqlite3)
- **Auth:** Session-based with bcrypt password hashing
- **CSS:** Pico.css (self-hosted, embedded in binary via `embed.FS`, served from `/static/pico.red.min.css`)
- **Admin JS:** `static/admin.js` (embedded in binary, served from `/static/admin.js`) — handles prefecture select navigation, `data-confirm` dialogs, and `data-copy` clipboard buttons; all admin-side interactivity lives here to comply with the `script-src 'self'` CSP (inline event handlers are blocked)
- **Markdown:** github.com/gomarkdown/markdown
- **System metrics:** github.com/shirou/gopsutil/v4 — cross-platform host metrics (CPU load, RAM, disk usage) for the `/admin/sistem` health page

## Architecture Pattern
```
domain/          # Business logic, models, repositories
presentation/    # HTTP handlers, views, viewmodels
migrations/      # Database schema + SQLite → Postgres/MySQL export tool
cmd/migrate/     # CLI binary for generating SQL migration dumps
dbs/             # SQLite database files
lib/             # Utilities: analytics, cache, captcha, formatter, markdown, navigation, notify, pagination, ratelimit, search, slug, social, sysmetrics
constant/        # Constants (domain name, KBRITokyoURL, SubdomainKey)
router/          # Route definitions
```

## Database Structure

Four SQLite files (`dbs/app.db`, `dbs/admin.db`, `dbs/user.db`, `dbs/analytics.db`). Full schema: see **[`docs/schema.md`](schema.md)**.

## Key Routes

Public, admin, and subdomain routes: see **[`docs/routes.md`](routes.md)**.

## User Submission Workflow

### Protections
All public POST forms apply: rate limiting, 64 KB body cap, honeypot field, CSRF, knowledge captcha (HMAC-signed Indonesian questions, never exposed in HTML), per-field length caps, email validation, and enum whitelisting. Status workflow: `pending` → `approved` or `rejected`. See `docs/rules/security-auth.md` and `docs/rules/security-validation.md` for implementation details of each protection.

Panduan forms support a server-side **markdown preview cycle** (Pratinjau → Edit Lagi → Ajukan) with all form state (CSRF token, captcha, contact fields) preserved in hidden inputs. See `docs/rules/forms.md` § Markdown preview.

### Flow
1. User visits `/komunitas/tambah` or `/panduan/tambah`
2. Reads motivational message with prefecture context
3. Fills form with contact info, content, and knowledge captcha
4. *(Panduan only)* User may click "Pratinjau" to preview rendered markdown; clicks "Edit Lagi" to return to the editor, or proceeds to submit
5. Submission saved with `pending` status
6. Admin reviews in admin panel (`/admin/komunitas` or `/admin/panduan`)
7. Admin clicks "Lihat" to view full detail (raw content + rendered markdown for panduan)
8. Admin clicks "Setujui" (Approve) or "Tolak" (Reject)
8. Approved content appears publicly

### Motivational Messaging
Blue accent boxes including the prefecture name on: `/komunitas`, `/komunitas/tambah`, `<subdomain>/`, and `/panduan/tambah`. Styled via CSS class (not inline styles — CSP blocks those).

## Content Security
Markdown rendered via `lib/markdown.RenderMarkdown` with `SkipHTML`, `SkipImages`, `Safelink`, `NofollowLinks`, `NoreferrerLinks`, `HrefTargetBlank`. A custom render hook blocks `javascript:`, `data:`, Unicode homograph, and punycode URLs. User-supplied image URLs are **permanently disabled** — do not re-enable; see `docs/rules/README.md` § Hard Rules. Security headers (`X-Frame-Options`, CSP, etc.) are set in `Caddyfile.prod`, not Go. See `docs/rules/security-validation.md` § Markdown rendering and `docs/rules/security-headers.md` for full details.

Two excerpt helpers in `lib/markdown`:

- `RenderMarkdownExcerpt(content string, maxRunes int) string` — renders inline markdown as HTML (bold, italic, code) but strips `<a>` tags so the result is safe inside card `<a>` elements. Truncates at `maxRunes` of visible text and closes any open inline tags. Use this for **listing card excerpts** (`@templ.Raw(item.Excerpt)`).
- `PlainTextExcerpt(content string, maxRunes int) string` — strips all Markdown syntax (block lines, inline markers, links) and rune-truncates to pure text. Use this for **SEO meta descriptions** and as the input to `HighlightExcerpt` for search snippets.

Both helpers share `isBlockLine` which **skips block-level lines entirely** — headings (`#`), lists (`-`, `+`, `*`, `1.`), blockquotes (`>`), and fenced code blocks. The text inside a blockquote is not extracted; the whole line is dropped. A post whose content consists mainly of blockquotes will produce an empty excerpt — this is intentional: list views show only prose text.

## Search
Two scopes via `?search=<query>` (3–100 chars, rate-limited 20/min/IP):
- **Cross-prefecture** (`warga.jp/?search=`): single query against `app.db` with no `WHERE prefecture` filter, sorts by prefecture → type → title, caches 5 min in `globalSearchCache`.
- **Prefecture-scoped** (`<pref>/?search=`): single query with `WHERE prefecture = ?`, caches by `"<pref>:<query>"` key, 5 min.

`SearchResult.Excerpt` is HTML-safe (query matches in `<mark>`); always render with `@templ.Raw(result.Excerpt)`. Homepage no-TTL caches (`globalFeaturedCache`, `globalCountsCache`, `globalNewestCache`, `globalNationalCache`) are warmed at startup and invalidated by admin mutations. `globalNationalCache` is cleared inside `InvalidateCountsCache()` so no extra call sites are needed. Homepage viewmodel constructor: `viewmodel.NewIndexViewModel(viewmodel.IndexParams{...})` — accepts a struct, zero values are safe. See `docs/rules/ui-layouts.md` § Search and `docs/rules/homepage.md` § Nasional Rail for details.

## Feature Status
**Live:** prefecture listing, komunitas + panduan (submission, approval, admin management, detail pages, markdown preview), search (cross-pref + scoped, cached), reports/laporan, admin dashboard + password change, email notifications, pagination, SEO (slug URLs, OG tags, JSON-LD, sitemap), analytics (`/admin/analitik`, cookie-free), `/kontak`, legal pages (`/kebijakan-privasi`, `/syarat-ketentuan`), `/weblogs` changelog, SQLite → Postgres/MySQL migration dump (`/admin/migrasi`).

**Disabled (router commented out):** kuliner, jasa, lapak, loker, lungsuran, diskusi.

## Environment Variables
See [DEVELOPMENT.md](DEVELOPMENT.md#7-environment-variables) for the full list of environment variables and their defaults. Key variables: `APP_ENV` (`""` = dev, `"production"` = prod); SMTP vars for email notifications (all optional in dev).

## Error Handling
Pre-rendered HTML error pages in `presentation/shared/view/error.templ`. Serve via `response.Response404(w)` / `response.Response500(w)` — never use `http.NotFound` or raw `http.Error` for user-facing pages. `response.Init()` must be called **after** `constant.SetDomainName()` in `main.go` (nav links need the domain set first).

## Styling
- **CSS**: `pico.red.min.css` + `base.css` + `public.css` (public) or `admin.css` (admin), all embedded via `embed.FS`, served with `?v=<build>` cache-busting. `base.css`, `public.css`, `admin.css`, and `admin.js` are minified in-place during `scripts/build-prod.sh` (via `tdewolff/minify`) before the binary is linked; source files are restored via `trap EXIT` so the working tree stays clean. `pico.red.min.css` is pre-minified and left untouched. See `docs/rules/admin-layout.md` § Stylesheet split for the loading matrix; see `docs/deploy.md` for the one-time install step.
- **Design tokens** (`:root` in `base.css`): `--sans`, `--mono`, `--merah`/`--merah-soft`/`--merah-ink`, `--ink`/`--ink-2`/`--ink-3`, `--paper`/`--paper-2`, `--rule`/`--rule-soft`, `--amber`/`--amber-soft`/`--amber-ink`, `--radius`/`--radius-sm`/`--radius-lg`.
- **Fonts**: Plus Jakarta Sans + JetBrains Mono, self-hosted WOFF2 in `static/fonts/`.
- **Shared chrome**: `component.Masthead()` + `component.SiteFooter()` in `presentation/shared/component/chrome.templ` — do not duplicate in individual templates.
- See `docs/rules/ui-pico-conflicts.md` for Pico conflict patterns and `docs/rules/ui-tokens.md` for the full design token reference.

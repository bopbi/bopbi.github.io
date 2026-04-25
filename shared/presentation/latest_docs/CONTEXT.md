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
- **Admin JS:** `static/admin.js` (embedded in binary, served from `/static/admin.js`) — handles prefecture select navigation and `data-confirm` dialogs; all admin-side interactivity lives here to comply with the `script-src 'self'` CSP (inline event handlers are blocked)
- **Markdown:** github.com/gomarkdown/markdown

## Architecture Pattern
```
domain/          # Business logic, models, repositories
presentation/    # HTTP handlers, views, viewmodels
migrations/      # Database schema + SQLite → Postgres/MySQL export tool
cmd/migrate/     # CLI binary for generating SQL migration dumps
dbs/             # SQLite database files
lib/             # Utilities: analytics, captcha, formatter, navigation, markdown, slug, ratelimit, pagination, notify
constant/        # Constants (domain name, KBRITokyoURL, SubdomainKey)
router/          # Route definitions
```

## Database Structure
- `dbs/admin.db` - Admin users & sessions
- `dbs/user.db` - User accounts & sessions
- `dbs/<prefecture>.db` - Prefecture-specific data (tokyo.db, osaka.db, etc.)

### Key Tables
- `admins` - Admin user accounts
- `admin_sessions` - Admin login sessions
- `users` - Regular user accounts
- `sessions` - User login sessions
- `community_info` - Community listings with status and submitter info
- `generic_info_post` - Information/guide posts with status and submitter info
- `reports` - Content complaints submitted by visitors (per-prefecture; fields: `content_type`, `content_id`, `reason`, `status`)
- `marketplace` - Buy/sell listings (feature disabled, table exists)
- `recycled_goods` - Second-hand items (feature disabled, table exists)
- `schema_migrations` - Tracks applied migrations

### Table Schema Changes (Versioned)
Both `community_info` and `generic_info_post` tables have been extended with:
- `status` - Submission status: `pending`, `approved`, `rejected`
- `submitter_email` - Submitter's email
- `submitter_whatsapp` - Submitter's WhatsApp number
- `submitter_line` - Submitter's Line ID

The original `contact` column in `community_info` was **removed** (migration 9) because it duplicated the submitter contact fields. The `submitter_email`, `submitter_whatsapp`, and `submitter_line` fields are **submitter-only** with a privacy-by-deletion lifecycle: stored as plaintext while pending/rejected (visible to admin for moderation), then **NULLed immediately on approval** — never retained for publicly-visible content. A one-time backfill migration (per-prefecture v3) applied this to all existing approved rows. See `docs/rules/forms.md` § Contact fields for the full rule and admin UX note.

> **Schema change rule:** Any addition, removal, or rename of a column or table must also be
> reflected in `migrations/export.go` (the SQL dump generator). See **Rules > Database Migrations**
> for the exact checklist.

## Key Routes

### Public Routes
- `/` - Homepage with four stacked sections in order: curated `⭐ Rekomendasi`, cross-prefecture `🆕 Panduan Terbaru` feed (5 newest approved panduan), a contribute CTA strip (`+ Bagikan Panduan` / `+ Bagikan Komunitas`), and a `🗾 Pilih Wilayah` region-selector accordion; `/?search=<query>` switches to cross-prefecture search results mode (all four sections hidden)
- `<subdomain>/` - Prefecture homepage (panduan list); `/?search=<query>` switches to prefecture-scoped search results mode
- `/ping` - Health check endpoint
- `/robots.txt` - Blocks `/admin/` from crawlers; links to sitemap
- `/sitemap.xml` - Dynamic sitemap: apex static pages + all prefecture homepages + all approved panduan/komunitas detail pages
- `/og-image.png` - Shared OG image (1200×630 PNG, Indonesian flag colors, generated once and cached)
- `/kebijakan-privasi` - Privacy policy page (static, uses LayoutGeneric)
- `/syarat-ketentuan` - Terms & Conditions page (static, uses LayoutGeneric)
- `/kontak` - Contact form (GET/POST); CSRF + knowledge captcha + rate limit 3/hour per IP; sends email to admin via `lib/notify` with Reply-To set to visitor's email
- `/daftar` - User signup (commented out in router)
- `/login` - User login (commented out in router)
- `/logout` - User logout (commented out in router)
- `/komunitas` — On the **apex** domain: full cross-prefecture index with `.apex-head` (total count + prefectures-with-content stats), `.kom-list-apex` newest-10 cards, `.pref-komunitas-stack` per-prefecture groups paginated by 12. On a **prefecture subdomain**: shows only that prefecture's community list with pagination. Cache: `communityIndexCache` (no-TTL struct, warmed at startup, invalidated by `InvalidateCommunityIndexCache()`).
- `/komunitas/{id}-{slug}` - Community detail (e.g. `/komunitas/1-komunitas-indonesia-di-tokyo`); bare `/komunitas/{id}` also works for backward compatibility
- `/komunitas/tambah` - Submit new community (GET/POST); apex-accessible (form carries a prefecture dropdown)
- `/komunitas/{id}/laporkan` - Report a community item (GET/POST)
- `/panduan` — Apex panduan index: `.apex-head` (total count + prefectures-with-content stats), `.panduan-list` newest-10 feed with kind badges (`tips`/`pengalaman`), `.region-stack` accordion grouping all 47 prefectures into 6 regions with per-prefecture counts (first 2 regions open by default; empty prefectures render as `.pref.empty`). Cache: `panduanIndexCache` (no-TTL struct, warmed at startup, invalidated by `InvalidatePanduanIndexCache()`).
- `/panduan/{id}-{slug}` - Panduan detail page (full rendered markdown content); bare `/panduan/{id}` also works for backward compatibility
- `/panduan/tambah` - Submit new panduan/guide (GET/POST); apex-accessible (form carries a prefecture dropdown — the selected prefecture determines which DB the submission lands in); supports markdown preview cycle (Pratinjau → Edit Lagi → Ajukan)
- `/panduan/{id}/laporkan` - Report a panduan item (GET/POST)

> Note: Panduan listing at the prefecture level has no dedicated route — it is served from `<subdomain>/` (the prefecture homepage, `pref_index.templ`, using `LayoutPrefecture`). Each entry shows a 150-rune plain-text excerpt (via `markdown.PlainTextExcerpt`) with a "Baca selengkapnya →" link to `/panduan/{id}`. Full content is rendered on the detail page. The search form embedded in `pref_index.templ` submits to `/?search=` on the current subdomain, so search results appear on the prefecture homepage. Pages using `LayoutCategory` (e.g. komunitas listing) have a search bar injected by the layout itself — do **not** use `LayoutCategory` for pages that define their own `.layout` + `.sidebar` structure, as it creates a double sidebar.

### Admin Routes
- `/admin` - Admin dashboard (summary of komunitas + panduan counts per prefecture; each count cell links directly to the manage page filtered to that prefecture)
- `/admin/login` - Admin login (GET/POST; POST protected by IP rate limiter)
- `/admin/logout` - Admin logout (GET or POST)
- `/admin/komunitas` - Manage communities (with prefecture selector)
- `/admin/komunitas/tambah` - Add community form
- `/admin/komunitas/{id}/lihat` - View community detail (metadata + raw description)
- `/admin/komunitas/{id}/edit` - Edit community form
- `/admin/komunitas/{id}/hapus` - Delete community
- `/admin/komunitas/{id}/setujui` - Approve pending community
- `/admin/komunitas/{id}/tolak` - Reject pending community
- `/admin/laporan` - Manage content reports (with prefecture selector)
  - `POST /{id}/nonaktifkan` — reject content (hide from public) + resolve report
  - `POST /{id}/hapus-konten` — permanently delete content + resolve report
  - `POST /{id}/abaikan` — dismiss report; content unchanged
- `/admin/panduan` - Manage panduan/guide posts (with prefecture selector)
- `/admin/panduan/tambah` - Add panduan form
- `/admin/panduan/{id}/lihat` - View panduan detail (metadata, raw content, rendered markdown)
- `/admin/panduan/{id}/edit` - Edit panduan form
- `/admin/panduan/{id}/hapus` - Delete panduan
- `/admin/panduan/{id}/setujui` - Approve pending panduan
- `/admin/panduan/{id}/tolak` - Reject pending panduan
- `/admin/ganti-password` - Change admin password (GET: form, POST: verifies current password before updating)
- `/admin/migrasi` - Generate SQLite → Postgres/MySQL SQL dump (GET: form, POST: preview or download)

### Subdomain Routes
- `<subdomain>.warga.jp` - Prefecture-specific content
- Example: `tokyo.warga.jp/komunitas`

## User Submission Workflow

### Protections
All public POST forms apply: rate limiting, 64 KB body cap, honeypot field, CSRF, knowledge captcha (HMAC-signed Indonesian questions, never exposed in HTML), per-field length caps, email validation, and enum whitelisting. Status workflow: `pending` → `approved` or `rejected`. See `docs/rules/security.md` for implementation details of each protection.

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
Markdown rendered via `lib/markdown.RenderMarkdown` with `SkipHTML`, `SkipImages`, `Safelink`, `NofollowLinks`, `NoreferrerLinks`, `HrefTargetBlank`. A custom render hook blocks `javascript:`, `data:`, Unicode homograph, and punycode URLs. User-supplied image URLs are **permanently disabled** — do not re-enable; see `docs/rules/general.md`. Security headers (`X-Frame-Options`, CSP, etc.) are set in `Caddyfile.prod`, not Go. See `docs/rules/security.md` § Markdown rendering and § Security Headers for full details.

Two excerpt helpers in `lib/markdown`:

- `RenderMarkdownExcerpt(content string, maxRunes int) string` — renders inline markdown as HTML (bold, italic, code) but strips `<a>` tags so the result is safe inside card `<a>` elements. Truncates at `maxRunes` of visible text and closes any open inline tags. Use this for **listing card excerpts** (`@templ.Raw(item.Excerpt)`).
- `PlainTextExcerpt(content string, maxRunes int) string` — strips all Markdown syntax (block lines, inline markers, links) and rune-truncates to pure text. Use this for **SEO meta descriptions** and as the input to `HighlightExcerpt` for search snippets.

## Search
Two scopes via `?search=<query>` (3–100 chars, rate-limited 20/min/IP):
- **Cross-prefecture** (`warga.jp/?search=`): fans out goroutines to all 47 DBs, sorts by prefecture → type → title, caches 5 min in `globalSearchCache`.
- **Prefecture-scoped** (`<pref>/?search=`): single DB, caches by `"<pref>:<query>"` key, 5 min.

`SearchResult.Excerpt` is HTML-safe (query matches in `<mark>`); always render with `@templ.Raw(result.Excerpt)`. Homepage no-TTL caches (`globalFeaturedCache`, `globalCountsCache`, `globalNewestCache`) are warmed at startup and invalidated by admin mutations. See `docs/rules/content.md` § Search for the new-content-type checklist and key files.

## Feature Status
**Live:** prefecture listing, komunitas + panduan (submission, approval, admin management, detail pages, markdown preview), search (cross-pref + scoped, cached), reports/laporan, admin dashboard + password change, email notifications, pagination, SEO (slug URLs, OG tags, JSON-LD, sitemap), analytics (`/admin/analitik`, cookie-free), `/kontak`, legal pages (`/kebijakan-privasi`, `/syarat-ketentuan`), `/weblogs` changelog, SQLite → Postgres/MySQL migration dump (`/admin/migrasi`).

**Disabled (router commented out):** kuliner, jasa, lapak, loker, lungsuran, diskusi.

## Environment Variables
See [DEVELOPMENT.md](DEVELOPMENT.md#7-environment-variables) for the full list of environment variables and their defaults. Key variables: `APP_ENV` (`""` = dev, `"production"` = prod); SMTP vars for email notifications (all optional in dev).

## Error Handling
Pre-rendered HTML error pages in `presentation/shared/view/error.templ`. Serve via `response.Response404(w)` / `response.Response500(w)` — never use `http.NotFound` or raw `http.Error` for user-facing pages. `response.Init()` must be called **after** `constant.SetDomainName()` in `main.go` (nav links need the domain set first).

## Styling
- **CSS**: `pico.red.min.css` + `base.css` + `public.css` (public) or `admin.css` (admin), all embedded via `embed.FS`, served with `?v=<build>` cache-busting. `base.css`, `public.css`, `admin.css`, and `admin.js` are minified in-place during `scripts/build-prod.sh` (via `tdewolff/minify`) before the binary is linked; source files are restored via `trap EXIT` so the working tree stays clean. `pico.red.min.css` is pre-minified and left untouched. See `docs/rules/admin.md` "Stylesheet split" for the loading matrix; see `docs/TOOLS.md` § Production Build for the one-time install step.
- **Design tokens** (`:root` in `base.css`): `--sans`, `--mono`, `--merah`/`--merah-soft`/`--merah-ink`, `--ink`/`--ink-2`/`--ink-3`, `--paper`/`--paper-2`, `--rule`/`--rule-soft`, `--amber`/`--amber-soft`/`--amber-ink`, `--radius`/`--radius-sm`/`--radius-lg`.
- **Fonts**: Plus Jakarta Sans + JetBrains Mono, self-hosted WOFF2 in `static/fonts/`.
- **Shared chrome**: `component.Masthead()` + `component.SiteFooter()` in `presentation/shared/component/chrome.templ` — do not duplicate in individual templates.
- See `docs/rules/content.md` for Pico conflict patterns and full design token reference.

# Route Table

## Public Routes
- `/` - Homepage with four stacked sections in order: curated `⭐ Rekomendasi`, cross-prefecture `🆕 Panduan Terbaru` feed (5 newest approved panduan), a contribute CTA strip (`+ Bagikan Panduan` / `+ Bagikan Komunitas`), and a `🗾 Pilih Wilayah` region-selector accordion; `/?search=<query>` switches to cross-prefecture search results mode (all four sections hidden)
- `<subdomain>/` - Prefecture homepage (panduan list); `/?search=<query>` switches to prefecture-scoped search results mode
- `/ping` - Health check — returns JSON `{"version":"v0.x.y","build":"<sha>","build_time":"<iso8601>","status":"ok"}`
- `/weblogs` - Web request log viewer (access-controlled GET handler)
- `/robots.txt` - Blocks `/admin/` from crawlers; links to sitemap
- `/sitemap.xml` - Dynamic sitemap: apex static pages + all prefecture homepages + all approved panduan/komunitas detail pages
- `/og-image.png` - Shared OG image (1200×630 PNG, Indonesian flag colors, generated once and cached)
- `/kebijakan-privasi` - Privacy policy page (static, uses LayoutGeneric)
- `/syarat-ketentuan` - Terms & Conditions page (static, uses LayoutGeneric)
- `/kontak` - Contact form (GET/POST); CSRF + knowledge captcha + rate limit 3/hour per IP; sends email to admin via `lib/notify` with Reply-To set to visitor's email
- `/daftar` - User signup (commented out in router)
- `/login` - User login (commented out in router)
- `/logout` - User logout (commented out in router)
- `/komunitas` — On the **apex** domain: full cross-prefecture index with `.apex-head` (total count + prefectures-with-content stats), `.tipe-strip` filter chips, `.kom-list-apex` newest cards, a `region-pin` national card at the top of "Per prefektur" (when `NationalCount > 0`), `.pref-komunitas-stack` per-prefecture groups paginated by 12 (national excluded from groups). Accepts repeated `?tipe={slug}` params for **multi-select** category filtering (`GetApprovedCommunityByCategories` — full DB scan with `IN (?,...)`, no per-prefecture cap; chips toggle slugs in/out via `tipeToggleURL`; shows `.filter-active` notice and `.empty-tipe` card when zero results; `noindex` when any filter is active). `SelectedTipes []string` in `CommunityIndexViewModel`. On a **prefecture subdomain**: shows only that prefecture's community list with pagination. Cache: `communityIndexCache` (no-TTL struct, warmed at startup, invalidated by `InvalidateCommunityIndexCache()`); the apex cache is not used for tipe-filtered requests because it only keeps top-3 per prefecture.
- `/komunitas/{id}-{slug}` - Community detail; bare `/komunitas/{id}` also works. Body section shows prose description + `.kd-socials` social link pills (parsed from `Platform: URL` lines via `lib/social.ParseLinks`).
- `/komunitas/tambah` - Submit new community (GET/POST); apex-accessible (form carries a prefecture dropdown)
- `/komunitas/{id}/laporkan` - Report a community item (GET/POST); uses `LayoutPrefecture` with `.layout + .sidebar + .content`. Sidebar: navigation (back to item, panduan/komunitas list), "Konten dilaporkan" meta (type + title). Content: `pref-section-head`, `.form-card` with reason textarea + captcha, `.submit-alert-success/error` feedback.
- `/panduan` — Apex panduan index: `.apex-head` (total count + prefectures-with-content stats), `.panduan-list` newest-10 feed with kind badges (`tips`/`pengalaman`), a `region-pin` national card at the top of "Per prefektur" (when `NationalCount > 0`), `.region-stack` accordion grouping all 47 geographic prefectures into 6 regions with per-prefecture counts (first 2 regions open by default; empty prefectures render as `.pref.empty`; `national` has no region key and is excluded from region groups). Cache: `panduanIndexCache` (no-TTL struct, warmed at startup, invalidated by `InvalidatePanduanIndexCache()`).
- `/panduan/{id}-{slug}` - Panduan detail page. Full-width `<div class="article-head">` (kind badge, h1, byline-row — no dek) sits above `<div class="article-body layout">` which is a two-column grid: `<aside class="sidebar">` (TOC via `.toc-sidebar` if the post has `##` headings, metadata, report widget) + `<div class="prose content">` (rendered markdown). `<div class="article-footer">` (related posts, CTA) is full-width after the grid. All wrapped in `<article class="guide-article">` to reset PicoCSS card styling. Bare `/panduan/{id}` also works for backward compatibility.
- `/panduan/tambah` - Submit new panduan/guide (GET/POST); apex-accessible (form carries a prefecture dropdown — the selected prefecture determines which DB the submission lands in); supports markdown preview cycle (Pratinjau → Edit Lagi → Ajukan)
- `/panduan/{id}/laporkan` - Report a panduan item (GET/POST); same layout as `/komunitas/{id}/laporkan`.

> Note: Panduan listing at the prefecture level has no dedicated route — it is served from `<subdomain>/` (the prefecture homepage, `pref_index.templ`, using `LayoutPrefecture`). The page shows **only panduan** — the komunitas section was removed; the sidebar Komunitas link navigates to `<subdomain>/komunitas`. Each panduan entry shows a 150-rune plain-text excerpt (via `markdown.PlainTextExcerpt`) with a "Baca selengkapnya →" link to `/panduan/{id}`. Full content is rendered on the detail page. The search form embedded in `pref_index.templ` submits to `/?search=` on the current subdomain, so search results appear on the prefecture homepage.

## Admin Routes
- `/admin` - Admin dashboard / action queue: shows only prefectures with pending komunitas, panduan, or reports sorted by urgency (rows with no pending work are hidden); each pending count links to the manage page filtered to that prefecture. Compact tiles above show VPS health and pending totals for Komunitas, Panduan, Laporan, Kontak.
- `/admin/login` - Admin login (GET/POST; POST protected by IP rate limiter)
- `/admin/logout` - Admin logout (GET or POST)
- `/admin/komunitas` - Manage communities (with prefecture selector)
- `/admin/komunitas/tambah` - Add community form
- `/admin/komunitas/{id}/lihat` - View community detail (metadata + raw description)
- `GET/POST /admin/komunitas/{id}/edit` - Edit community form
- `POST /admin/komunitas/{id}/hapus` - Delete community
- `POST /admin/komunitas/{id}/setujui` - Approve pending community
- `POST /admin/komunitas/{id}/tolak` - Reject pending community
- `POST /admin/komunitas/{id}/rekomendasi` - Feature community on homepage Rekomendasi
- `POST /admin/komunitas/{id}/pindah` - Move community to a different prefecture
- `/admin/laporan` - Manage content reports (with prefecture selector)
  - `POST /{id}/nonaktifkan` — reject content (hide from public) + resolve report
  - `POST /{id}/hapus-konten` — permanently delete content + resolve report
  - `POST /{id}/abaikan` — dismiss report; content unchanged
- `/admin/panduan` - Manage panduan/guide posts (with prefecture selector)
- `/admin/panduan/tambah` - Add panduan form
- `/admin/panduan/{id}/lihat` - View panduan detail (metadata, raw content, rendered markdown)
- `GET/POST /admin/panduan/{id}/edit` - Edit panduan form
- `POST /admin/panduan/{id}/hapus` - Delete panduan
- `POST /admin/panduan/{id}/setujui` - Approve pending panduan
- `POST /admin/panduan/{id}/tolak` - Reject pending panduan
- `POST /admin/panduan/{id}/rekomendasikan` - Feature panduan on homepage Rekomendasi
- `POST /admin/panduan/{id}/pindah` - Move panduan to a different prefecture
- `/admin/rekomendasi` - Manage featured panduan on homepage Rekomendasi (GET: list; `POST /{id}/hapus`, `POST /{id}/naik`, `POST /{id}/turun` — remove or reorder entries)
- `/admin/rekomendasi-komunitas` - Manage featured komunitas on homepage Rekomendasi (`POST /{id}/hapus`, `/naik`, `/turun`)
- `/admin/kontak` - Contact-form message queue (GET: list)
  - `GET /admin/kontak/{id}` - View message detail
  - `POST /admin/kontak/{id}/dibalas` - Mark message as replied
  - `POST /admin/kontak/{id}/arsipkan` - Archive message
  - `POST /admin/kontak/{id}/kembalikan` - Restore archived message to inbox
- `/admin/ganti-password` - Change admin password (GET: form, POST: verifies current password before updating)
- `/admin/ganti-email` - Change admin email
- `/admin/migrasi` - Generate SQLite → Postgres/MySQL SQL dump (`GET`: form + preview; `POST /download`: download as file)
- `/admin/analitik` - Read-only analytics dashboard (pageviews, search queries) over `dbs/analytics.db`
- `/admin/sistem` - VPS health indicator (CPU/RAM/Disk/goroutine/heap with severity badges); see `docs/SCALING.md` § In-Product Health Indicator

## Subdomain Routes
- `<subdomain>.warga.jp` - Prefecture-specific content (e.g. `tokyo.warga.jp/komunitas`)
- `national.warga.jp` - Nation-wide content; backed by rows with `prefecture = 'national'` in `app.db`

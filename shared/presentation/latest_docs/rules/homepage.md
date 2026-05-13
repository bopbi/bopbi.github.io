# Homepage & Apex Pages

Implementation detail for the apex homepage (`warga.jp/`) and the apex listing pages (`/panduan`, `/komunitas`). Read this when working on homepage sections, their caches, or the apex listing page layouts.

---

## Homepage section order

The apex homepage (`presentation/home/view/index.templ`) in the default no-search state renders sections in this order:

1. `⭐ Rekomendasi` — when `vm.FeaturedPanduan` is non-empty. Amber section head (`.section-head.amber`). Cards: `.rek-card` inside `.rekomendasi` flex row.
2. `🆕 Panduan Terbaru + 🤝 Komunitas` — combined `.highlights-row` with two `.hl-col` columns. Left: `.terbaru-list` under `.hl-subhead`. Right: `.kom-rail` of `.kom-mini` cards. Shown when `vm.NewestPanduan` or `vm.FeaturedKomunitas` is non-empty.
3. `🗂 Telusuri komunitas berdasarkan tipe` — `.section.section-tight` > `.tipe-browse`. 10 chips linking to `warga.jp/komunitas?tipe={slug}#list`. Shown when `len(vm.CategoryCounts) > 0`.
4. `🇮🇩 Berlaku di seluruh Jepang` — when `vm.NationalPanduanCount > 0 || vm.NationalKomunitasCount > 0`. Ink section head. Two-column rail: `.nasional-rail`.
5. Contribute CTA — always rendered. Full-bleed red `.contrib` band (negative-margin breakout, white CTAs).
6. `🗾 Pilih prefektur` — when `vm.RegionGroups` is non-empty. Single `.wilayah-grid` with dual-count rows (panduan + komunitas per prefecture). National pin at top (`region-pin`, `grid-column: 1/-1`). Each region is a `<details class="region">` element: open by default if it has prefectures, closed if empty. Template component: `regionBlockDual`.

The old separate "Panduan per prefektur" and "Komunitas per prefektur" grids (`regionBlock`, `regionBlockKomunitas`) have been replaced by the single `regionBlockDual` grid.

**CSS ownership** (`static/public.css`):
- `.highlights-row` — 2-column grid (1.55fr 1fr). `.hl-col` children. Stacks at ≤860px.
- `.kom-rail` — flex column of `.kom-mini` compact cards in the right highlights column.
- `.tipe-browse` — 2-column grid (220px 1fr): `.tb-label` + `.tb-chips`. Chips are `.tb-chip` with `.emo` and `.n` spans. Stacks at ≤860px.
- `.wilayah-grid` — 2-column CSS grid, 1-column at ≤860px.
- `.region` — white card, border, border-radius. Uses `<details>` element with `open` attribute on populated regions.
- `.region-head` — `<summary>` styled as flex row: region label (`h4`) + meta caption (`.n`, mono font). Browser/PicoCSS chevron suppressed via `::after { display: none !important }`.
- `.region-prefs.dual` — flex column of `.pref-row` grid rows (name · panduan count · komunitas count). Replaces the old pill-chip `.region-prefs` on the homepage.
- `.region.region-pin` — full-width (`grid-column: 1 / -1`) pinned national row with a 3px red left-accent bar (`::before`).

**Caching note:** `public.css` is served with `Cache-Control: public, max-age=604800, immutable`. In production, layout templates append `?v=` + `version.Build` (the current git SHA), so each deploy gets a new CSS URL. In local dev, `version.Build == "dev"` so the URL stays constant; use a hard refresh if you see stale styles.

---

## Rekomendasi (⭐ Featured Panduan)

The homepage shows a curated "⭐ Rekomendasi" section. Admin manages the list at `/admin/rekomendasi`.

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

`UNIQUE(prefecture, post_id)` prevents duplicates. Repository: `domain/admin/repository/featured.go`.

### Adding a panduan to rekomendasi
From `/admin/panduan?prefecture=<pref>`, click **★ Rekomendasikan** on any approved panduan. The handler snapshots the title and URL into `featured_panduan` at that moment.

**Maximum 3 items** (same limit for komunitas rekomendasi). The handler checks `CountFeaturedPanduan(adminDB)` before inserting. If already 3, it redirects to `/admin/rekomendasi` so the admin can remove one first.

### Keeping the featured title in sync
`featured_panduan` stores a **cached copy** of the title and URL. Any operation that changes a panduan's title or slug must call `adminRepository.SyncFeaturedPanduanTitle(adminDB, pref, postID, newTitle, newURL)` and `homeHandler.InvalidateFeaturedCache()`.

**Rule:** If you add a new code path that can change a panduan title, it must follow the same pattern. URL format:
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
`SyncFeaturedPanduanTitle` is a no-op if the panduan is not currently featured.

### Ordering
`/admin/rekomendasi` has ↑/↓ buttons per row. On every move, all `display_order` values are rewritten as sequential integers `0,1,2,…`. Homepage reads rows `ORDER BY display_order ASC, created_at ASC`.

### Homepage cache
`IndexHandler` warms `globalFeaturedCache` at startup. **No TTL** — valid until `InvalidateFeaturedCache()` is called. Every admin mutation (add, remove, reorder, title sync) calls `InvalidateFeaturedCache()`.

### CSS classes
| Class | Element | Purpose |
|---|---|---|
| `.rekomendasi` | `<div>` | CSS grid, 3-col auto-fill, card layout |
| `.rek-card` | `<a>` | Individual card — white bg, border, hover lift |
| `.rek-pref` | `<span>` | Prefecture label — small, muted pill |
| `.rek-foot` | `<div>` | Card footer row (date, meta) |

---

## Panduan Terbaru (🆕 Newest)

A cross-prefecture feed of the 5 most recently approved panduan. Surfaces platform activity to returning visitors.

### Data source
`loadNewestPanduan` in `presentation/home/handler/handler.go` reuses `prefRepository.GetApprovedGenericInfoPostPage(db, newestPerPrefLimit, 0)` with offset `0`. It fans out one goroutine per entry in `GetAllPrefectures()` (48 total), merges via channel, sorts by `created_at DESC`, and truncates to `newestDisplayLimit`.

```go
const (
    newestPerPrefLimit = 3 // pull from each prefecture
    newestDisplayLimit = 5 // show after merge + sort
)
```

### Homepage cache
`globalNewestCache` (single in-memory struct, RWMutex-guarded) warmed at startup. **No TTL** — valid until `InvalidateNewestCache()` is called.

### Invalidation
`InvalidateNewestCache()` must be called alongside every existing `InvalidateCountsCache()` call for panduan mutations. Current call sites in `presentation/admin/panduan/handler/handler.go`: Add, Approve, Reject, Delete.

**Rule:** Any new admin code path that changes the set of approved panduan in any prefecture DB must call both `InvalidateCountsCache()` and `InvalidateNewestCache()`.

### CSS classes
| Class | Element | Purpose |
|---|---|---|
| `.terbaru-list` | `<ul>` | Plain list, no bullets |
| `.terbaru-list a` | `<a>` | Flex row with `t-pref`, `t-title`, `t-date` spans |
| `.t-pref` | `<span>` | Prefecture label — small, muted |
| `.t-title` | `<span>` | Panduan title — grows to fill remaining space |
| `.t-date` | `<span>` | Short date or type label — right-aligned, muted |

The same `.terbaru-list` markup is reused for cross-prefecture search results.

**`.nasional` badge variant:** When a panduan is from `national.warga.jp`, the `.t-pref` span gets `.nasional` modifier. Same pattern for `.rek-pref.nasional` and `.km-pref.nasional`. Detection: `n.Prefecture == "national"` via the `tPrefClass()` Go helper.

---

## Nasional Rail (🇮🇩 Berlaku di seluruh Jepang)

Between Panduan Terbaru and Komunitas — surfaces content from `national.warga.jp`. Only renders when `vm.NationalPanduanCount > 0 || vm.NationalKomunitasCount > 0`.

### Data source
`loadNationalContent` queries `national.db` directly:
- Panduan: `prefRepository.GetApprovedGenericInfoPostPage(db, 5, 0)` — newest 5 approved panduan.
- Komunitas: `communityRepository.GetApprovedCommunityPage(db, 4, 0)` — newest 4 approved komunitas.

```go
const nationalPanduanLimit   = 5
const nationalKomunitasLimit = 4
```

### Caching
`globalNationalCache` warmed at startup, no TTL. Cleared by `InvalidateCountsCache()` (called on all admin panduan/komunitas mutations). No separate invalidation function needed.

### Viewmodel fields (`IndexViewModel`)
| Field | Type | Source |
|---|---|---|
| `NationalPanduanCount` | `int` | Extracted from `enrichedSections` (BaseURL == `https://national.{domain}`) |
| `NationalKomunitasCount` | `int` | Same |
| `NationalPanduan` | `[]SearchResult` | `globalNationalCache.panduan` |
| `NationalKomunitas` | `[]FeaturedKomunitasItem` | `globalNationalCache.komunitas` |

### CSS classes
| Class | Element | Purpose |
|---|---|---|
| `.nasional-rail` | `<div>` | 2-column CSS grid (`1.4fr 1fr`); 1-column on mobile |
| `.nas-col` | `<div>` | Individual column — white bg, border, border-radius |
| `.nas-col header` | `<header>` | Column header — flex row with `h3` + `.more` link |
| `.nas-list` | `<ul>` | Row list inside the column |
| `.nas-list a` | `<a>` | Row — 2-col grid: `.nas-title` + `.nas-meta` |
| `.nas-foot` | `<div>` | Column footer — mono, muted, search links |

### `IndexParams` struct
`NewIndexViewModel` accepts a single `IndexParams` struct. Zero values are safe.

```go
staticVM := viewmodel.NewIndexViewModel(viewmodel.IndexParams{
    Sections:               enrichedSections,
    FeaturedPanduan:        featuredItems,
    NewestPanduan:          newestItems,
    FeaturedKomunitas:      featuredKomunitasItems,
    ApprovedKomunitasTotal: approvedKomunitasTotal,
    NationalPanduanCount:   nationalPanduanCount,
    NationalKomunitasCount: nationalKomunitasCount,
    NationalPanduan:        nationalPanduan,
    NationalKomunitas:      nationalKomunitas,
})
```

---

## Apex listing pages

Layouts, CSS classes, data flows, and the `?tipe=` filter for `/panduan` and `/komunitas`: see **[`docs/rules/apex-listing-pages.md`](apex-listing-pages.md)**.

---

## Contribute CTA

A dashed-border strip between `🆕 Panduan Terbaru` and `🗾 Pilih Wilayah`. Invites visitors to submit without drilling into a prefecture subdomain.

### Why the panduan form has a prefecture dropdown
The panduan submit form mirrors the komunitas form: it exposes a `Prefectures []model.Prefecture` field and a `<select name="prefecture">` so it can be submitted from the apex homepage.

**Rule:** both `/panduan/tambah` and `/komunitas/tambah` must stay in `allowedPaths` (`main.go`). Removing either breaks the homepage CTA.

### Markup and styling
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

| Class | Purpose |
|---|---|
| `.contrib` | Full-bleed red panel (negative-margin breakout), white type, radial highlight bottom-left |
| `.contrib-btns` | Flex row, `gap: 12px`, wraps on narrow viewports |
| `.contrib .btn-outline` | Inside `.contrib`: white fill, red text, ink-fill on hover (overrides default outline button) |

**Copywriting:** buttons read `＋ Bagikan Panduan` / `＋ Bagikan Komunitas`. Do not use "Ajukan". The `＋` prefix carries the "add/create" signal.

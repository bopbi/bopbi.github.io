# Apex Listing Pages (`/panduan`, `/komunitas`)

Both apex index pages share a two-part structure: a stats header + a content body. CSS lives in `static/public.css` under the `/* Apex` block.

## Shared header — `.apex-head`

```html
<section class="apex-head">
  <div class="apex-head-text">
    <div class="eyebrow"><span class="pin"></span> Seluruh Jepang</div>
    <h1>...</h1>
    <p class="dek">...</p>
  </div>
  <div class="apex-head-stats">
    <dl>
      <dt>...</dt><dd>{ count }</dd>
      <dt>Prefektur terisi</dt><dd>{ n }<span class="of">/48</span></dd>
    </dl>
  </div>
</section>
```

Both viewmodels expose `TotalCount int`, `PrefecturesWithContent int`, and `NationalCount int` for the stats column. `NationalCount` drives the national pin block (see below).

## National pin block — `.region-pin` (shared pattern)

Both `/panduan` and `/komunitas` render a `region-pin` card at the top of the "Per prefektur" section when `vm.NationalCount > 0`. This mirrors the homepage `🗾` region grids. The block links to `national.warga.jp` (or `national.warga.jp/komunitas`) with a `.pin-link`, and shows the count badge and a `.pin-note` caption.

**Data flow:**
- `/panduan`: `NationalCount` = `countMap["national"]` extracted at the end of `loadPanduanIndex` — already queried during the goroutine fanout, no extra DB call.
- `/komunitas`: `NationalCount` captured during the receive loop in `loadCommunityIndex` (`r.count` when `r.pref.Value == Pref_National`). National is **excluded from `PrefectureGroups`** — it must not appear in the flat per-prefecture list.

The outer "Per prefektur" section renders if `len(vm.RegionGroups) > 0 || vm.NationalCount > 0` (panduan) or `len(vm.PrefectureGroups) > 0 || vm.NationalCount > 0` (komunitas), so national-only content still gets a section.

## Panduan list — `.panduan-list`

```html
<ul class="panduan-list">
  <li>
    <a href="...">
      <div class="pl-main">
        <div class="pl-meta">
          <span class="pl-pref">{ pref label }</span>
          <span class="pl-kind tips">tips</span>  <!-- or "pengalaman", or omit -->
        </div>
        <h3>{ title }</h3>
      </div>
      <div class="pl-stamp"><span class="d">{ day.month }</span>{ year }</div>
    </a>
  </li>
</ul>
```

| Class | Purpose |
|---|---|
| `.pl-kind` | Kind badge — neutral; `.tips` adds blue background, `.pengalaman` adds amber |
| `.pl-stamp` | Right-aligned date — `.d` = large day.month, sibling text = small year |

`IndexItem.Category` holds the raw category string (`"tips"`, `"pengalaman"`, or `""`). Apply it as a CSS class: `class={ "pl-kind " + item.Category }`.

## Panduan region stack — `.region-stack`

```html
<div class="region-stack">
  <details class="reg" open>  <!-- open for first 2 regions -->
    <summary class="reg-head">
      <span class="reg-title">Kanto</span>
      <span class="reg-count">8 panduan</span>
    </summary>
    <div class="reg-grid">
      <a class="pref" href="...">...</a>
      <a class="pref empty" href="...">...</a>  <!-- count == 0 -->
    </div>
  </details>
</div>
```

Region data comes from `RegionGroups []RegionGroupVM`. Each has `Label`, `TotalCount`, and `Prefectures []PrefectureCount`. Empty prefectures always included as `.pref.empty`.

| Class | Purpose |
|---|---|
| `.reg-head` | `<summary>` — 3-col grid (`1fr auto 1fr`); `::before` text `▸` on left, title centered, count right |
| `.reg-grid` | 4-column grid of `.pref` links (2-col on mobile); `gap: 1px`, `background: var(--rule-soft)` creates grid lines |
| `.pref` | Individual prefecture link — emoji flag, name, count in a 3-col grid |
| `.pref.empty` | Greyed-out prefecture with `—` count |

## Komunitas apex — `.kom-list-apex` and `.pref-komunitas-stack`

**Important:** `.kom-list-apex` uses `display: flex` on the `<ul>`. PicoCSS will add bullet markers unless you explicitly set `list-style: none` on the `<li>` (not just the `<ul>`):
```css
.kom-list-apex li { list-style: none; }
```

| Class | Purpose |
|---|---|
| `.kom-list-apex` | Flex column `<ul>` of community cards |
| `.kla-head/.kla-pref` | Card header — prefecture label |
| `.pref-komunitas-stack` | Flex column of `.pk-group` prefecture groups |
| `.pk-items` | 2-col grid of `.pk-item` compact links |

## Tipe filter (`?tipe=`)

The apex `/komunitas` page accepts a single `?tipe={slug}` query parameter. When present and valid:

1. **Handler** reads `?tipe=` in `IndexHandler` (after the search branch), validates it against `communityModel.CategoryBySlug`, then calls `communityRepository.GetApprovedCommunityByCategory(appDB, tipe)` — a full DB scan with no per-prefecture cap. The apex index cache (`cached.newest` / `cached.groups`) is intentionally not used here: the cache only keeps the top-3 newest per prefecture and would silently exclude older listings of that category.
2. **`SelectedTipe`** is set on the viewmodel; `NationalCount` is omitted (zero) for filtered views; pagination is skipped.
3. **Template** — `.tipe-strip` chip matching `SelectedTipe` gets `class="ts-chip cur"` via `tipeChipClass`. The strip anchor chips have `id="list"` on the first `<section>`, not the strip div, so `#list` jumps past the strip to results.
4. **Filter notice** — `.filter-active` `<p>` renders below the strip when `vm.SelectedTipe != ""`. Shows "Menampilkan komunitas **{emoji} {label}** · N hasil" and a "× hapus filter" `.hapus` link back to `/komunitas#list`.
5. **Empty state** — when the filtered `NewestItems` is empty, `.empty-tipe` renders instead of the `<ul>`. Contains: `.empty-mark` mono eyebrow, `h3`, recruiting paragraph, `btn-cta` linking to `/komunitas/tambah?tipe={slug}`, `.empty-foot` "lihat semua" escape. Section meta updates to "belum ada listing yang cocok".
6. **`noindex`** — filtered pages are not indexed (`vm.SelectedTipe != ""` passed as noindex flag to layout).

# Admin List Pages

## Prefecture chip selector for admin list pages

Per-prefecture admin list pages (panduan, komunitas, laporan) must use the shared chip row component instead of a `<select>` dropdown. Chips show pending counts per prefecture at a glance, so an admin landing on the page immediately sees which queues have work.

**Component:** `presentation/admin/shared/view/chips.templ` — `PrefectureChips(chips []shared.PrefectureChip, selectedValue string, baseURL string)`.

**Pattern in the list template:**
```templ
import sharedview "website/presentation/admin/shared/view"

// inside <article>, between the <h2> and the table:
<h2>{ vm.Title }</h2>
@sharedview.PrefectureChips(vm.Chips, vm.SelectedPrefecture, "/admin/panduan")
```

**Pattern in the list viewmodel:** the constructor takes the `prefDBMap map[string]*sql.DB` and computes chips via the shared builder:
```go
chips := shared.PanduanChips(shared.BuildPrefectureSummaries(prefectures, prefDBMap))
```
Use `PanduanChips`, `KomunitasChips`, or `LaporanChips` depending on which count field should drive the badge. All three live in `presentation/admin/shared/viewmodel/viewmodel.go`.

**`PrefectureChip` has two independent count fields:**
- `Pending int` — yellow attention badge (`.chip-badge`). Shown only when `> 0`. Used for items needing admin action (panduan pending, reports pending).
- `Count int` — neutral grey badge (`.chip-count`). Shown only when `> 0`. Used for total approved content count, so the admin can see at a glance which prefectures have content even when nothing is pending. `KomunitasChips` populates this with `KomunitasApproved`; panduan and laporan chips leave it zero.

**Rules:**
- Do not keep a `<select>` fallback. The chip row is the single selector on list pages.
- Prefectures with `Pending > 0` render in amber (`has-pending`); the current prefecture is highlighted in the primary accent (`is-current`); zero-pending prefectures are muted. Styles live in `static/admin.css` under the "Prefecture chips" section — do not inline chip CSS in templates.
- When adding a new per-prefecture list page, decide which count field is appropriate: use `Pending` for action-required queues, `Count` for informational content totals.
- Pages with no per-prefecture scoping do **not** get a chip row: dashboard (shows all prefectures in the summary table), rekomendasi (curation list is global across prefectures), analitik (site-wide metrics), migrasi (global tool). If a new admin page is per-prefecture, add the chip row from the start.
- Form pages (tambah/edit) do not get a chip row either — they inherit the prefecture from the incoming link via a hidden field (see `panduan/view/form.templ:68`).

## Admin search on list pages

**Every new admin list page must ship with a search box.** Search is not a later enhancement — wire it up as part of the initial implementation alongside the chip row and CSRF token. Releasing a list page without search forces the admin to scroll through growing data with no way to filter.

Admin kelola list pages (panduan, komunitas, laporan, rekomendasi) support a server-side text search via `?q=...`.

**Component:** `presentation/admin/shared/view/searchbar.templ` — `AdminSearchBar(query, selectedPrefecture, baseURL, placeholder string)`.

**Placement in templates:** right after `@sharedview.PrefectureChips(...)` and before the "Tambah baru" CTA:
```templ
@sharedview.PrefectureChips(vm.Chips, vm.SelectedPrefecture, "/admin/panduan")
@sharedview.AdminSearchBar(vm.Query, vm.SelectedPrefecture, "/admin/panduan", "Cari panduan…")
```
For pages without per-prefecture scoping (rekomendasi, kontak), pass `""` as the second argument. The bar will omit the hidden `prefecture` input automatically.

**Handler pattern:** read and sanitise `q` before building the viewmodel:
```go
q := search.SanitizeQuery(r.URL.Query().Get("q"))
vm := viewmodel.NewPanduanListViewModel(db, adminDB, pref, prefectureDB, q)
```
Import: `"website/lib/search"`.

**Viewmodel pattern:** accept `q string`, branch on it to call the search or list repository function, and expose it on the viewmodel so templates can render the empty-state message and pre-fill the search input:
```go
if q != "" {
    posts, err = repository.SearchGenericInfoPostWithPending(db, q)
} else {
    posts, err = repository.GetAllGenericInfoPostWithPending(db)
}
```

**Empty-state phrasing with search active:**
```templ
if vm.Query != "" {
    <p>Tidak ada hasil untuk "{ vm.Query }" di { vm.SelectedPrefectureLabel }.</p>
} else {
    <p>Belum ada panduan di { vm.SelectedPrefectureLabel }.</p>
}
```

**LIKE sanitisation:** all `q` values are piped through `lib/search.SanitizeQuery` (trim, 200-rune cap) then `lib/search.LikePattern` (escape `\`, `%`, `_`) inside the search repository functions. SQL uses `LIKE ? ESCAPE '\'` to treat the escaped characters as literals.

**Non-goals:** no global admin search across prefectures/content types, no dashboard search, no FTS/ranking — plain `LIKE '%q%'` is the contract.

## Admin action button colour system

All admin list pages (panduan, komunitas, laporan, rekomendasi) use a consistent set of CSS classes from `static/admin.css` for action buttons. All buttons in the action column are sized via `.btn-sm` and grouped in a `.admin-actions` flex wrapper. The same classes apply to the Edit link on detail pages.

| Class | Colour | Used for |
|---|---|---|
| `btn-approve` | Green | Setujui — approve a pending submission |
| `btn-warn` | Orange | Tolak (panduan/komunitas), Nonaktifkan (laporan) — soft-reject / take-down |
| `btn-view` | Gray `#6b7280` | Lihat — read-only navigation (applied to `<a role="button">`) |
| `btn-edit` | Blue `#3b82f6` | Edit — navigate to edit form (applied to `<a role="button">`, on list rows AND detail pages) |
| `btn-reject` | Red | Hapus, Hapus Konten — permanent delete (destructive) |
| `btn-neutral` | Gray `#6b7280` | Rekomendasikan, Abaikan, Urutkan ↑/↓ — neutral/non-destructive action |

**Rules:**
- Every button in the action column must have `btn-sm`. Never use a full-width PicoCSS button in a table action cell.
- Wrap all action cell content in `<div class="admin-actions">` — this is a flex row with `gap: 0.3rem` and `flex-wrap: wrap` so buttons line up horizontally and wrap cleanly on narrow viewports. Never use `class="form-inline"` on action forms; that wrapper is deprecated in favour of `admin-actions`.
- Forms inside `.admin-actions` use `display: inline-flex` (not `display: contents` — broken in Safari) so they shrink to their button's natural size.
- `btn-view` and `btn-edit` use `!important` on their colour properties because PicoCSS overrides `<a role="button">` colours with theme rules; the flag is intentional, not lazy.
- Navigation actions (Lihat, Edit) stay as `<a href>` elements — they are GET navigations, not form submissions. Use `role="button"` to give them button appearance while keeping correct semantics and browser behaviour (e.g. middle-click to open in new tab).
- The detail-page "Edit" button also takes `btn-edit` so it matches the list-row treatment (`panduan/view/detail.templ`, `community/view/detail.templ`).

## Empty-state phrasing

Admin list pages use a single phrasing pattern for empty states so the UI feels coherent across sections.

| Scope | Phrasing | Example |
|---|---|---|
| Per-prefecture list (no search) | `Belum ada {thing} di { vm.SelectedPrefectureLabel }.` | `Belum ada panduan di Tokyo.` |
| Per-prefecture list (search active, 0 results) | `Tidak ada hasil untuk "{ vm.Query }" di { vm.SelectedPrefectureLabel }.` | `Tidak ada hasil untuk "visa" di Tokyo.` |
| Global list (no prefecture filter, no search) | `Belum ada {thing}.` | `Belum ada panduan yang difitur.` |
| Global list (search active, 0 results) | `Tidak ada hasil untuk "{ vm.Query }".` | `Tidak ada hasil untuk "visa".` |
| Analytics tables | `Belum ada data.` | (no prefecture, no thing name) |

**Rules:**
- Use "Belum ada" (not "Tidak ada") — tone matches "not yet" rather than "none exist", which fits a site that's still growing.
- Render as plain `<p>…</p>`. Do not wrap in `<em>`; italic text implies a caveat that empty-state messages don't need.
- Always name the prefecture on per-prefecture lists so the admin can see *which* queue is empty without scanning the chip row again.

## CSRF on admin mutation forms

Every admin POST — both form pages and list-row buttons — uses the double-submit cookie pattern. The list viewmodel generates a token via `adminRepository.GenerateCSRFToken()`, the list handler sets it as a cookie via `adminRepository.SetCSRFTokenCookie(w, vm.CSRFToken)` before rendering, and each inline `<form>` in `.admin-actions` carries a `<input type="hidden" name="csrf_token" value={ vm.CSRFToken }/>`.

On the server, every mutation handler calls `r.ParseForm()` then `adminRepository.VerifyCSRFToken(w, r)`; the helper writes a 403 and returns false on mismatch, so the handler just needs:

```go
if err := r.ParseForm(); err != nil {
    http.Error(w, "Invalid form data", http.StatusBadRequest)
    return
}
if !adminRepository.VerifyCSRFToken(w, r) {
    return
}
```

**Rule:** when adding a new admin POST — list-row button or form page — add the hidden input in the template and call `VerifyCSRFToken` in the handler before any state change. A single token per page render is fine; each form on the page reuses it.

> For the full CSRF three-step checklist (GET handler, cookie set, re-set on error), see [`security-validation.md` § CSRF](./security-validation.md).

## Admin detail pages ("Lihat")
Admin list pages for panduan and komunitas include a **"Lihat"** link per row that opens a read-only detail page (`/admin/panduan/{id}/lihat`, `/admin/komunitas/{id}/lihat`).

**What each detail page shows:**
- **Panduan** — metadata table (ID, status, category, prefecture, submitter info, timestamps) + raw content in `<pre>` + rendered markdown in `.markdown-preview` div
- **Komunitas** — metadata table (ID, status, location, submitter info, timestamps) + raw description in `<pre>` (plain text, no markdown rendering — komunitas descriptions are short-form and don't use markdown)

**Rules:**
- Fetch with `GetGenericInfoPostByIDAnyStatus` / `GetCommunityByIDAnyStatus` so the detail page works for any status (pending, approved, rejected)
- The detail page has an "Edit" button linking to the edit form and a "← Kembali" button linking back to the list (both preserve `?prefecture=`)
- No write operations — detail pages are purely read-only GET handlers

## Public URL actions ("Lihat di Situs" / "Salin URL")

Approved panduan and komunitas entries show two extra actions: a link that opens the public page in a new tab and a clipboard-copy button. These appear **on both the list row and the detail page** for `status == "approved"` entries only.

**Where implemented:**
- List: `presentation/admin/community/view/list.templ` and `presentation/admin/panduan/view/list.templ` — rendered inline in the `admin-actions` cell
- Detail: `presentation/admin/community/view/detail.templ` and `presentation/admin/panduan/view/detail.templ` — rendered in `form-actions`

**How `PublicURL` is computed:**

The URL is built in the **viewmodel constructor**, not in the template. For list pages, this requires a list-item wrapper type (e.g. `CommunityListItem`) that embeds the domain model and adds `PublicURL`:

```go
type CommunityListItem struct {
    communityModel.CommunityInfo
    PublicURL string
}
```

The constructor maps `[]CommunityInfo` → `[]CommunityListItem`, computing the slug-based URL for each entry:

```go
s := slug.Slugify(c.Name)
if s == "" {
    publicURL = fmt.Sprintf("https://%s.%s/komunitas/%d", prefecture, constant.DomainName(), c.ID)
} else {
    publicURL = fmt.Sprintf("https://%s.%s/komunitas/%d-%s", prefecture, constant.DomainName(), c.ID, s)
}
```

The same pattern applies to panduan (`PanduanDetailViewModel.PublicURL` uses the title slug).

**Rules:**
- Only show these actions when `status == "approved"` — pending/rejected entries have no canonical public URL.
- "Salin URL" requires `admin.js` in the template's `<head>` (`data-copy` attribute is inert without it). Both list and detail templates for panduan and komunitas already load it.
- When adding a new content type that has public detail pages, follow the same wrapper-type pattern to expose `PublicURL` from the viewmodel.

## Cross-prefecture move

Panduan and komunitas can be moved from one prefecture (including `national`) to another via a **"Pindah Prefektur"** collapsible that sits below the `form-actions` button row on the admin detail page. The move is a distinct action separate from editing content fields.

**UI layout**: the `<details class="move-panel">` is placed **outside and below** the `.form-actions` div — not inside it. This avoids the Pico CSS sizing mismatch where `details > summary[role="button"]` renders at a different height than sibling `<a role="button">` and `<button>` elements inside a flex row. The `.move-panel` CSS class constrains the summary to `fit-content` width so it doesn't stretch full-width like a block element. Do not move the `<details>` back inside `.form-actions`.

**Routes:**
- `POST /admin/panduan/{id}/pindah`
- `POST /admin/komunitas/{id}/pindah`

**Why a cross-DB operation**: each prefecture has its own SQLite file (`dbs/<pref>.db`). There is no `prefecture` column — the prefecture is encoded in which DB the row lives in. Moving a row is therefore a 3-step cross-DB operation with no shared transaction:

1. Read full row from source DB (`GetGenericInfoPostByIDAnyStatus` / `GetCommunityByIDAnyStatus`)
2. INSERT into destination DB via `InsertExistingGenericInfoPost` / `InsertExistingCommunity` — these repo functions write all columns explicitly (status, created_at, admin_edited_at, submitter_*) to preserve audit history; only the autoincrement ID changes
3. DELETE from source DB — if this step fails after a successful INSERT, the failure is logged and the duplicate is the lesser evil (no rollback that would re-introduce the old ID)

**Featured guard**: the move is blocked if the row has a `featured_panduan` / `featured_komunitas` entry keyed on the source `(prefecture, id)`. Because a move changes the ID, the featured reference would become dangling. The handler calls `IsPanduanFeatured` / `IsKomunitasFeatured` before touching any DB; if blocked it redirects back to the detail page with `?err=featured` and the template shows `.admin-notice-warn`. Admin must un-feature via `/admin/rekomendasi` first.

**ID change caveat**: the new row gets a fresh autoincrement ID in the destination DB. The public URL changes (`/panduan/<newID>-<slug>`). The detail page redirects to `/admin/panduan/<newID>/lihat?prefecture=<dst>` after a successful move. There is no undo; the admin can move it back manually.

**National is not special at the DB layer**: `national` is just another key in `PrefDBs` (`dbs/national.db`). The move handler treats `dst="national"` identically to any other prefecture.

**Cache invalidations after a move** — both source and destination are covered because all `Invalidate*` functions do a global `Purge()`:

Panduan move: `InvalidatePanduanIndexCache`, `InvalidatePanduanDetailCache`, `InvalidateCountsCache`, `InvalidateNewestCache`, `InvalidateFeaturedCache`, `InvalidateSitemapCache`

Komunitas move: `InvalidateCommunityIndexCache`, `InvalidateCommunityListCache`, `InvalidateCommunityDetailCache`, `InvalidateCountsCache`, `InvalidateFeaturedKomunitasCache`, `InvalidateSitemapCache`

## Prefecture selector for admin handlers
Admin handlers that manage per-prefecture content (e.g. panduan, komunitas) must read the selected prefecture from the `?prefecture=` query param — never hardcode a specific prefecture. This was a past bug where all panduan write handlers were hardcoded to `prefectureDB["tokyo"]`.

**Pattern:**
```go
// helper — call at the top of every handler that needs a prefecture DB
func selectedPrefecture(r *http.Request, prefectureDB map[string]*sql.DB) (string, *sql.DB) {
    pref := r.URL.Query().Get("prefecture")
    db, ok := prefectureDB[pref]
    if !ok {
        return "", nil
    }
    return pref, db
}

// in each handler:
pref, db := selectedPrefecture(r, prefectureDB)
if db == nil {
    http.Error(w, "Invalid prefecture", http.StatusBadRequest)
    return
}
```

**Rules:**
- All redirects after POST (create, edit, delete, approve, reject) must carry `?prefecture=xxx`.
- All form links in the view (edit, hapus, setujui, tolak) must include `?prefecture=xxx`.
- The form itself must carry the prefecture in a hidden field or query param so the POST handler can read it.
- The list view's "add" button must carry `?prefecture=xxx`.

Copy the pattern from `presentation/admin/panduan/handler/handler.go`.

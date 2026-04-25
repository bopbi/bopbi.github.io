# Admin

## Admin dashboard — per-prefecture counts must link to the manage page

Every numeric count shown in the "Per Prefektur" table on the dashboard must be a link that navigates directly to the relevant manage page with the prefecture pre-selected. A bare number with no link forces the admin to manually switch the selector on the manage page — that is the exact UX issue this rule exists to prevent.

**Pattern:**
```templ
<td>
    <a href={ templ.URL("/admin/panduan?prefecture=" + p.Value) }>
        <span class="status-pending">{ fmt.Sprintf("%d", p.PanduanPending) }</span>
        { " / " }
        <span class="status-approved">{ fmt.Sprintf("%d", p.PanduanApproved) }</span>
        { " / " }
        <span class="status-rejected">{ fmt.Sprintf("%d", p.PanduanRejected) }</span>
    </a>
</td>
```

If a new content type is added to the dashboard table, its count column must follow the same pattern from the start.

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

**Rules:**
- Do not keep a `<select>` fallback. The chip row is the single selector on list pages.
- Prefectures with `Pending > 0` render in amber (`has-pending`); the current prefecture is highlighted in the primary accent (`is-current`); zero-pending prefectures are muted. Styles live in `static/admin.css` under the "Prefecture chips" section — do not inline chip CSS in templates.
- Pages with no per-prefecture scoping do **not** get a chip row: dashboard (shows all prefectures in the summary table), rekomendasi (curation list is global across prefectures), analitik (site-wide metrics), migrasi (global tool). If a new admin page is per-prefecture, add the chip row from the start.
- Form pages (tambah/edit) do not get a chip row either — they inherit the prefecture from the incoming link via a hidden field (see `panduan/view/form.templ:68`).

## Stylesheet split

Admin and public pages load different CSS files. **Do not mix them.**

| Layer | Admin pages | Public pages |
|---|---|---|
| PicoCSS | `pico.red.min.css` | `pico.red.min.css` |
| Shared primitives | `base.css` | `base.css` |
| Site-specific | `admin.css` | `public.css` |

- `base.css` — design tokens, font faces, button primitives, shared utilities (`.text-muted`, `.form-card`, `.pagination`, etc.)
- `public.css` — public site styles only. Starts with `body { background: var(--paper); }` which overrides Pico's dark background. Never load this on admin pages.
- `admin.css` — admin-only components: status badges, action buttons, migration page, detail/preview blocks.
- Admin pages use `data-theme="dark"` on `<html>`; public pages use `data-theme="light"`. The split ensures a public CSS change cannot affect admin rendering.

## Admin page
Admin pages follow a different pattern from public pages:

- **No shared layout** — every admin view is a self-contained `<!DOCTYPE html>` templ file. Do not use `LayoutGeneric` for admin pages — it loads `public.css` and forces `data-theme="light"`, breaking the dark admin theme.
- **Favicon** — every admin view must include the ⚙️ emoji favicon in `<head>`, immediately after the viewport meta tag:
  ```html
  <link rel="icon" href="data:image/svg+xml,<svg xmlns=%22http://www.w3.org/2020/svg%22 viewBox=%220 0 100 100%22><text y=%22.9em%22 font-size=%2290%22>⚙️</text></svg>"/>
  ```
- **Auth guard** — every handler must call `requireAdmin(adminDB, w, r)` and return early if it returns false:
  ```go
  if !requireAdmin(adminDB, w, r) {
      return
  }
  ```
- **Pre-auth exception** — `/admin/login` is the only admin page that omits the nav bar. Because it is pre-auth, it cannot link to protected routes. It still uses the same standalone `<!DOCTYPE html>`, admin favicon, and stylesheets as every other admin page — just no `<nav>` block.
- **Consistent nav bar** — every authenticated admin view uses a two-section Pico CSS dropdown nav. Copy from an existing view and adjust only the `aria-current="page"` placement:

  ```templ
  <nav>
      <ul>
          <li>
              <h1>
                  Admin Panel
                  if version.Build == "dev" {
                      <small class="badge-dev">DEV</small>
                  }
              </h1>
          </li>
          <li><a href="/admin">Dashboard</a></li>
          <li>
              <details class="dropdown">
                  <summary>Konten</summary>
                  <ul>
                      <li><a href="/admin/panduan">Panduan</a></li>
                      <li><a href="/admin/komunitas">Komunitas</a></li>
                      <li><a href="/admin/laporan">Laporan</a></li>
                      <li><a href="/admin/rekomendasi">Rekomendasi</a></li>
                      <li><a href="/admin/analitik">Analitik</a></li>
                      <li><a href="/admin/migrasi">Migrasi DB</a></li>
                  </ul>
              </details>
          </li>
      </ul>
      <ul>
          <li>
              <details class="dropdown">
                  <summary>Akun</summary>
                  <ul dir="rtl">
                      <li><a href="/admin/ganti-password">Ganti Password</a></li>
                      <li><a href="/admin/ganti-email">Ganti Email</a></li>
                      <li><a href="/admin/logout">Logout</a></li>
                  </ul>
              </details>
          </li>
      </ul>
  </nav>
  ```

  Rules:
  - `dir="rtl"` on the Akun dropdown `<ul>` makes it open leftward so it does not overflow off the right screen edge.
  - Place `aria-current="page"` on the `<a>` of the current page. Pico CSS styles the highlighted link automatically — no `open` attribute is needed to make the active link visible.
  - **Never add `open` to any `<details class="dropdown">` in the admin nav.** Pico CSS renders the submenu with `position: absolute`, which overlays page content below the nav. Any interactive element (button, link) positioned in that overlay zone will not receive clicks — the hidden nav links intercept them instead. This was a confirmed bug: ↑/↓ ordering buttons on the Rekomendasi page were unclickable because the Konten submenu (forced open) overlapped the table.
  - Do not use inline `style=` on any nav element — use CSS classes from `admin.css`.
- **Register the route** in `router/router.go` inside the admin routes section (no middleware group needed — auth is handled per-handler).
- **Login redirect** — after successful login, users land on `/admin` (the dashboard). Do not change this to point at a specific section.

## Admin action button colour system

All admin list pages (panduan, komunitas, laporan, rekomendasi) use a consistent set of CSS classes from `static/admin.css` for action buttons. All buttons in the action column are sized via `.btn-sm` and grouped in a `.admin-actions` flex wrapper. The same classes apply to the Edit link on detail pages.

| Class | Colour | Used for |
|---|---|---|
| `btn-approve` | Green | Setujui — approve a pending submission |
| `btn-warn` | Orange | Tolak (panduan/komunitas), Nonaktifkan (laporan) — soft-reject / take-down |
| `btn-view` | Gray `#6b7280` | Lihat — read-only navigation (applied to `<a role="button">`) |
| `btn-edit` | Blue `#3b82f6` | Edit — navigate to edit form (applied to `<a role="button">`, on list rows AND detail pages) |
| `btn-reject` | Red | Hapus, Hapus Konten — permanent delete (destructive) |
| `btn-neutral` | Gray `#888` | Rekomendasikan, Abaikan, Urutkan ↑/↓ — neutral/non-destructive action |

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
| Per-prefecture list | `Belum ada {thing} di { vm.SelectedPrefectureLabel }.` | `Belum ada panduan di Tokyo.` |
| Global list (no prefecture filter) | `Belum ada {thing}.` | `Belum ada panduan yang difitur.` |
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

## Admin detail pages ("Lihat")
Admin list pages for panduan and komunitas include a **"Lihat"** link per row that opens a read-only detail page (`/admin/panduan/{id}/lihat`, `/admin/komunitas/{id}/lihat`).

**What each detail page shows:**
- **Panduan** — metadata table (ID, status, category, prefecture, submitter info, timestamps) + raw content in `<pre>` + rendered markdown in `.markdown-preview` div
- **Komunitas** — metadata table (ID, status, location, submitter info, timestamps) + raw description in `<pre>` (plain text, no markdown rendering — komunitas descriptions are short-form and don't use markdown)

**Rules:**
- Fetch with `GetGenericInfoPostByIDAnyStatus` / `GetCommunityByIDAnyStatus` so the detail page works for any status (pending, approved, rejected)
- The detail page has an "Edit" button linking to the edit form and a "← Kembali" button linking back to the list (both preserve `?prefecture=`)
- No write operations — detail pages are purely read-only GET handlers

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

## Article Edit Policy

### Who can edit articles
Only admins can edit articles (panduan and komunitas) after submission. There is no public user account system — submissions track contact info only (email, WhatsApp, Line), not a registered user identity. Creator editing is therefore not feasible without a separate auth system.

Edit handlers: `presentation/admin/panduan/handler/handler.go` (`PanduanEditHandler`, `PanduanEditPostHandler`) and `presentation/admin/community/handler/handler.go` (`CommunityEditHandler`, `CommunityEditPostHandler`).

### Approval side-effect: submitter contact is permanently deleted
Clicking **Setujui** (approve) on a komunitas or panduan calls `ClearCommunitySubmitterContact` / `ClearGenericInfoPostSubmitterContact` immediately after `UpdateStatus`. All three submitter contact columns (`submitter_email`, `submitter_whatsapp`, `submitter_line`) are NULLed in the same request. **This is irreversible.** If you need to contact the submitter (thank-you, follow-up), copy their email/WhatsApp from the detail view *before* clicking Setujui. Rejection (`Tolak`) intentionally leaves contact intact.

### Status after admin edit
Approved articles **stay approved** after an admin edit. Admin edits are trusted — no re-review cycle is triggered.

### Admin edit indicator (`admin_edited_at`)
Both `generic_info_post` and `community_info` tables have an `admin_edited_at DATETIME` nullable column (added in migration version 2). It is set **only** by `UpdateGenericInfoPost` and `UpdateCommunity` — not by `UpdateGenericInfoPostStatus` / `UpdateCommunityStatus` (approve/reject).

This distinction matters: `updated_at` is also touched on approve/reject, so it cannot be used as a reliable "was this content edited?" signal. `admin_edited_at` is the authoritative flag.

When non-null, the public detail page shows **"✏ Diperbarui admin: [date]"** below the creation date. The formatted value is computed in the detail viewmodel (`AdminEditedAtFormatted`) using `lib/formatter.FormatDate`.

**Rule:** Never set `admin_edited_at` from a status-change function. Only set it from a function that modifies the article content itself.

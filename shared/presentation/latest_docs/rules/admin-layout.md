# Admin Layout

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
- **Auth guard — already handled by middleware. Do not add per-handler checks.** Never write a `requireAdmin` helper, inline `GetCurrentAdmin → redirect` check, or any other per-handler auth gate inside an `/admin/...` handler. The chi middleware `adminMiddleware.RequireAdmin` (in `presentation/admin/middleware/auth.go`) is mounted on the protected `/admin` sub-group in `router/router.go` and enforces auth before any handler in that group runs. If your handler needs the current admin object (e.g. to display the email, or to compare the current password), read it from request context:
  ```go
  admin := adminmiddleware.AdminFromContext(r.Context())
  ```
  The middleware stashes it there once per request — no second DB lookup. **Why the rule:** every admin handler package previously carried its own copy of `requireAdmin`; 9 duplicates and 30+ call sites were all eliminated by a single middleware. Do not reintroduce them. The login and logout routes are siblings *outside* the protected group — that is why they remain reachable while unauthenticated. New routes that must be reachable pre-auth belong in the public sibling block, not under the protected group.
- **Consistent nav bar** — rendered by the shared `AdminNav` component in `presentation/admin/shared/view/nav.templ`. Every authenticated admin template calls it as:
  ```templ
  import sharedview "website/presentation/admin/shared/view"
  ...
  @sharedview.AdminNav("current-page-key")
  ```
  Recognized keys: `"dashboard"`, `"panduan"`, `"komunitas"`, `"laporan"`, `"rekomendasi"`, `"kontak"`, `"analitik"`, `"migrasi"`, `"sistem"`, `"ganti-password"`, `"ganti-email"`. Pass `""` (empty string) for pages like create/edit forms that have no matching nav link (no link gets highlighted). The key maps to `aria-current="page"` on the correct `<a>` tag — Pico CSS applies the active style automatically.

  **Adding a new admin page:**
  1. Add a `<li><a href="/admin/newpage">Label</a></li>` entry inside the Konten `<ul>` in `nav.templ` and decide on its key string.
  2. Call `@sharedview.AdminNav("newpage")` in the new template — one line.
  3. Register the route in `router/router.go` inside the relevant `r.Route` sub-block under `/admin`. See `features.md` § Router conventions.

  Rules:
  - `dir="rtl"` on the Akun dropdown `<ul>` makes it open leftward so it does not overflow off the right screen edge.
  - **Never add `open` to any `<details class="dropdown">` in the admin nav.** Pico CSS renders the submenu with `position: absolute`, which overlays page content below the nav. Any interactive element (button, link) positioned in that overlay zone will not receive clicks — the hidden nav links intercept them instead. This was a confirmed bug: ↑/↓ ordering buttons on the Rekomendasi page were unclickable because the Konten submenu (forced open) overlapped the table.
  - Do not use inline `style=` on any nav element — use CSS classes from `admin.css`.
  - The login page (`/admin/login`) intentionally omits the nav bar since it is pre-auth. Do not add `AdminNav` to the login template.
- **Register the route** in `router/router.go` inside the relevant `r.Route` sub-block under the protected `/admin` sub-group (e.g. inside `r.Route("/komunitas", ...)` for a komunitas-related route, or directly under the top-level `/admin` closure for a section-level page like `/admin/sistem`). See the Auth guard bullet above and `features.md` § Router conventions.
- **Login redirect** — after successful login, users land on `/admin` (the dashboard). Do not change this to point at a specific section.

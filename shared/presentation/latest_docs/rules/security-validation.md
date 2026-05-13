# Security — Validation & CSRF

## CSRF — three-step checklist for every form handler
CSRF validation requires all three steps to be present. Missing any one causes "Invalid CSRF token" at runtime.

| Step | Where | What |
|---|---|---|
| 1. Generate | Viewmodel constructor | `csrfToken, _ := repository.GenerateCSRFToken()` — store in the viewmodel |
| 2. Set cookie | **GET handler**, before rendering | `adminRepository.SetCSRFTokenCookie(w, vm.CSRFToken)` |
| 3. Validate | **POST handler**, after `ParseForm` | Compare `r.FormValue("csrf_token")` with `r.Cookie("csrf_token")` |

Step 2 is the one most often forgotten. Without it the browser has no cookie to compare against, so every POST returns 403.

The same cookie must also be **re-set on every error re-render** inside the POST handler — each re-render generates a new token in the viewmodel, so the cookie must be updated to match before writing the response.

**Pattern to follow** (see `presentation/panduan/handler/handler.go` or `presentation/community/handler/handler.go`):
```go
// GET handler
vm := viewmodel.NewXxxSubmitViewModel(...)
adminRepository.SetCSRFTokenCookie(w, vm.CSRFToken) // ← must not be missing
templ.Handler(view.SubmitScreen(vm)).ServeHTTP(w, r)

// POST handler — use a renderError helper to avoid repeating the cookie set
renderError := func(msg string) {
    vm := viewmodel.NewXxxSubmitViewModelWithError(..., msg)
    adminRepository.SetCSRFTokenCookie(w, vm.CSRFToken) // ← re-set on every re-render
    templ.Handler(view.SubmitScreen(vm)).ServeHTTP(w, r)
}
```

> **Cross-reference:** The Markdown preview / edit-lagi flow for panduan forms has additional CSRF and captcha rules (no token rotation, no captcha re-verify on preview/edit). See [`forms.md` § Markdown preview / edit-lagi](./forms.md#markdown-preview--edit-lagi--token-and-state-preservation).

## Field length caps — all text inputs
Every text field stored in the DB or forwarded via email **must** have a per-field length cap enforced server-side in the POST handler. The 64 KB body limit is a backstop, not a substitute — a single uncapped field can absorb the entire budget.

Check with `len([]rune(value))` (rune count, not byte count) to handle multi-byte characters fairly:
```go
if len([]rune(r.FormValue("name"))) > 200 {
    renderError("Nama terlalu panjang (maks. 200 karakter)")
    return
}
```

**Standard limits used in this project:**

| Field type | Limit |
|---|---|
| Title / name | 200 runes |
| Short text (location) | 200–500 runes |
| Description / bio | 5 000 runes |
| Long-form content (panduan) | 50 000 runes |
| Email address | 200 runes |
| WhatsApp / Line ID | 50 runes |
| Free-text reason / message | 2 000 runes |

Apply caps after the required-field checks, before the DB write or email send.

## Email fields — format validation and header injection prevention
Any field that accepts an email address must be validated server-side with `net/mail.ParseAddress`, regardless of `type="email"` on the HTML input (browser validation is bypassable).

```go
import "net/mail"

if _, err := mail.ParseAddress(email); err != nil {
    renderError("Format email tidak valid")
    return
}
```

**Why this matters for SMTP:** `lib/notify` constructs raw MIME messages by string concatenation. An email containing `\r\n` would inject additional headers (e.g. `Bcc:`, `X-Custom:`). `mail.ParseAddress` rejects any address with newlines, eliminating the injection vector. Apply this check to every email field that ends up in an SMTP header (`Reply-To`, `To`, `Cc`).

## Enum / select fields — whitelist validation
Any `<select>` input with a fixed set of allowed values must be validated against that whitelist in the POST handler. Do not trust that the submitted value matches one of the `<option>` values — an attacker can POST arbitrary strings.

```go
// model defines the allowed values
validCategory := false
for _, cat := range prefectureModel.PanduanCategories {
    if r.FormValue("category") == cat.Value {
        validCategory = true
        break
    }
}
if !validCategory {
    renderError("Kategori tidak valid")
    return
}
```

Add whitelist validation immediately after the required-field checks, before the DB write.

## Markdown rendering — always use SkipHTML
`lib/markdown.RenderMarkdown` is the only place markdown is converted to HTML. It must keep the `html.SkipHTML` flag to prevent raw HTML tags embedded in user-submitted content from reaching the browser.

Current flags (all required):
```go
htmlFlags := html.HrefTargetBlank | html.SkipHTML | html.SkipImages | html.Safelink | html.NofollowLinks | html.NoreferrerLinks
```

- `SkipHTML` — strips `<script>`, `<iframe>`, `<img onerror=...>` and all other raw HTML blocks
- `SkipImages` — strips markdown image syntax (`![alt](url)`) — user-supplied image URLs are a security risk; see [Rules README § Hard Rules](./README.md)
- `Safelink` — blocks `javascript:` protocol links
- `NofollowLinks` + `NoreferrerLinks` — user-submitted links must not pass SEO signals or referrer data

Do **not** remove any of these flags. The rendered HTML is injected via `@templ.Raw()` in the templ template, so there is no auto-escaping safety net.

## `allowedPaths` — apex-domain routes without a subdomain

`requireSubdomain` in `main.go` returns 404 for any request that has no subdomain, unless the path is in the `allowedPaths` map or starts with `/admin`. When adding a new public route that should be accessible at the apex domain (`warga.jp/...`), add it to `allowedPaths`:

```go
var allowedPaths = map[string]struct{}{
    "/":                  {},
    "/ping":              {},
    "/robots.txt":        {},
    "/sitemap.xml":       {},
    "/og-image.png":      {},
    "/favicon.ico":       {},
    "/kebijakan-privasi": {},
    "/syarat-ketentuan":  {},
    "/weblogs":           {},
    "/kontak":            {},
    "/panduan/tambah":    {},
    "/komunitas/tambah":  {},
    // add new apex routes here
}
```

Note: `/static/` paths are auto-allowed by prefix check and do not need individual entries. `/favicon.ico` is the exception — it does not start with `/static/` so it requires an explicit entry.

Routes that only exist on subdomains (e.g. `/komunitas`, `/panduan/*`) do **not** need to be in `allowedPaths`. The two submission forms (`/panduan/tambah`, `/komunitas/tambah`) are intentionally apex-accessible — both forms carry a prefecture dropdown so visitors can submit from the apex homepage CTA without first drilling into a subdomain.

**Rule:** Registering a route in `router/router.go` is not enough on its own. Any new page that must be reachable at the apex domain (no subdomain) requires a matching entry in `allowedPaths` in `main.go`. Without it the middleware returns 404 before the router is reached. The checklist for a new apex-domain page is:

1. Add the route in `router/router.go`
2. Add the path to `allowedPaths` in `main.go`

## Trailing-slash routing
`middleware.RedirectSlashes` is registered in `main.go` and issues a 301 redirect from any trailing-slash URL to the canonical no-slash version (e.g. `/admin/` → `/admin`). Do not register duplicate routes with and without a trailing slash — the middleware handles it.

## Admin logout accepts GET and POST
`/admin/logout` is registered for both GET and POST so that `<a href="/admin/logout">` nav links work without a form. This is intentional for an internal admin panel. If a public-facing logout is ever added, it should be POST-only with CSRF.

## Contact field isolation — public vs admin repository queries

Submitter contact fields (`submitter_email`, `submitter_whatsapp`, `submitter_line`) must never appear in public responses. The enforcement is structural: public-path repository functions use scan helpers that do not include contact columns.

| Scan helper | Columns | Used by |
|---|---|---|
| `scanPublicCommunityInfo` | 8 (no submitter fields) | `GetApprovedCommunityPage`, `GetCommunityByIDAndStatus`, `SearchCommunity` |
| `scanPublicGenericInfoPost` | 8 (no submitter fields) | `GetApprovedGenericInfoPostPage`, `GetGenericInfoPostByIDAndStatus`, `SearchGenericInfoPost`, `GetRelatedPosts` |
| `scanCommunityInfo` | 11 (includes submitter fields) | Admin-path functions only |
| `scanGenericInfoPost` | 11 (includes submitter fields) | Admin-path functions only |

**Rule:** any new public-path query (approved-only, list or detail) must use a `scanPublic*` helper. A query that also returns pending/rejected rows (admin moderation views) may use the full scan helper. If adding a new content type with submitter contact fields, add the equivalent public scan helper before writing any repository functions.

This is defence-in-depth on top of the `ClearCommunitySubmitterContact` / `ClearGenericInfoPostSubmitterContact` wipe-on-approval mechanism. Even if the wipe fails or is skipped, the contact data never enters the public struct.

## Reference

[Ruby on Rails Security Guide](https://guides.rubyonrails.org/security.html) — the most comprehensive framework-agnostic reference for web security patterns. Covers session management, CSRF, SQL/command injection, password handling, file uploads, CSP, HSTS, and more. Even though this project uses Go, the guide's principles apply directly. Consult it when adding any new auth flow, file handling, or user-facing form.

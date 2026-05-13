# Security

## Auth Patterns
- CSRF token required for all POST forms
- Session cookie: `admin_session_token` for admins
- Cookie settings: `HttpOnly: true`, `Secure: true`, `SameSite: http.SameSiteLaxMode`

> Outstanding security work is tracked in [`docs/SECURITY_TODO.md`](../SECURITY_TODO.md). Check it before starting auth-adjacent work.

### Rate limiting — shared `lib/ratelimit` package

All IP-based rate limiting uses `lib/ratelimit` (`Limiter` + `ClientIP`). Close over a `*ratelimit.Limiter` in the handler factory and call `limiter.Allow(ratelimit.ClientIP(r))` at the top of the POST handler.

```go
func MyPostHandler(...) http.HandlerFunc {
    limiter := ratelimit.New(5, time.Hour) // max 5 per IP per hour
    return func(w http.ResponseWriter, r *http.Request) {
        if !limiter.Allow(ratelimit.ClientIP(r)) {
            http.Error(w, "Terlalu banyak permintaan.", http.StatusTooManyRequests)
            return
        }
        // ...
    }
}
```

Current limits:

| Endpoint | Limit |
|---|---|
| `POST /admin/login` | 5 attempts / 15 min (own `loginLimiter` in auth.go — predates the shared package) |
| `POST /komunitas/tambah` | 5 submissions / hour |
| `POST /panduan/tambah` | 5 submissions / hour |
| `POST /panduan/tambah` (`action=preview`) | 30 previews / hour (own `previewLimiter`) — markdown render is CPU-bound; submit limiter is bypassed by preview, so it gets its own cap |
| `POST /*/laporkan` | 10 reports / hour (shared limiter across komunitas + panduan report handlers) |
| `GET /?search=` and `GET <subdomain>/?search=` | 20 queries / minute (shared `searchLimiter` in `IndexHandler`) |

`ratelimit.ClientIP(r)` resolves the real IP via `X-Real-IP` → `X-Forwarded-For` → `RemoteAddr`. Requires the reverse proxy (Caddy) to set `X-Real-IP`.

Limiters reset on server restart — acceptable for spam prevention. If persistence across restarts is ever needed, move counts to the DB.

### Admin login brute-force protection
`LoginPostHandler` in `presentation/admin/handler/auth.go` carries its own in-memory `loginLimiter` (predates `lib/ratelimit`). It blocks an IP after **5 failed attempts within 15 minutes** with HTTP 429.

Key rules:
- Call `limiter.recordFailure(ip)` on every credential failure (wrong username/password).
- Call `limiter.reset(ip)` on successful login.
- Do **not** count CSRF failures or empty-field validation against the limiter — those are not credential guesses.
- Constants `maxLoginAttempts = 5` and `loginWindowDur = 15 * time.Minute` are at the top of `auth.go`.

### Honeypot field
All public POST forms include a hidden `<input type="text" name="website">` wrapped in a `.hp` div (positioned off-screen via CSS). Real users never see or fill it. Bots that auto-fill form fields will fill it, and the POST handler silently drops the request (`return` with no response body).

**Rule:** Every POST form must include the honeypot check immediately after `ParseForm`, before any validation:
```go
if r.FormValue("website") != "" {
    return // silent discard — likely a bot
}
```

`FormPostComponent` in `presentation/shared/component/form.templ` already emits the honeypot field automatically. Forms that don't use `FormPostComponent` (panduan/tambah, komunitas/tambah) add it manually in their `.templ` file.

Do **not** return an error — silently returning gives bots no information about why the submission was rejected.

### Knowledge captcha — `lib/captcha`
All public submission forms use an Indonesian knowledge captcha instead of math (`a + b = ?`). The correct answer is **never exposed in HTML** — only an HMAC-signed token is sent to the browser.

**How it works:**
- `captcha.Generate()` picks a random question from a pool of Indonesian-specific questions in `lib/captcha/captcha.go` (e.g. "Ibu kota Jepang adalah?") and returns `(question string, token string)`. The token is `base64(HMAC-SHA256(answer|timestamp) + "|" + timestamp)`.
- The token goes in a `<input type="hidden" name="captcha_answer">` field; the question is shown as a label above a `<input type="text" name="captcha_user">` answer field.
- `captcha.Verify(token, userAnswer string) bool` decodes the token, checks the 30-minute TTL, and confirms the user's answer (case-insensitive, trimmed) matches via HMAC comparison — iterating the question pool internally (O(N)).

**Viewmodel pattern** (all 4 submission viewmodels):
```go
import "website/lib/captcha"

question, token := captcha.Generate()
// store in vm:
CaptchaQuestion: question,
CaptchaAnswer:   token,   // HMAC token — safe to put in hidden field
```

**Handler pattern** (all 4 POST handlers):
```go
if !captcha.Verify(r.FormValue("captcha_answer"), r.FormValue("captcha_user")) {
    renderError("Jawaban captcha salah")
    return
}
```

The secret key is generated once at package `init()` via `crypto/rand`. Tokens do not survive server restarts — in-flight forms submitted after a restart will fail captcha and require a page refresh (acceptable).

### Request body size limit
All public POST handlers must call `http.MaxBytesReader` before `ParseForm` to prevent large payload attacks:
```go
r.Body = http.MaxBytesReader(w, r.Body, 64*1024) // 64 KB limit
if err := r.ParseForm(); err != nil {
    http.Error(w, "Invalid form data", http.StatusBadRequest)
    return
}
```
When the body exceeds the limit, `ParseForm` returns an error and the handler returns 400. The 64 KB limit is generous for text forms but blocks multi-megabyte payloads.

### CSRF — three-step checklist for every form handler
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

### Field length caps — all text inputs
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

### Email fields — format validation and header injection prevention
Any field that accepts an email address must be validated server-side with `net/mail.ParseAddress`, regardless of `type="email"` on the HTML input (browser validation is bypassable).

```go
import "net/mail"

if _, err := mail.ParseAddress(email); err != nil {
    renderError("Format email tidak valid")
    return
}
```

**Why this matters for SMTP:** `lib/notify` constructs raw MIME messages by string concatenation. An email containing `\r\n` would inject additional headers (e.g. `Bcc:`, `X-Custom:`). `mail.ParseAddress` rejects any address with newlines, eliminating the injection vector. Apply this check to every email field that ends up in an SMTP header (`Reply-To`, `To`, `Cc`).

### Enum / select fields — whitelist validation
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

### Markdown rendering — always use SkipHTML
`lib/markdown.RenderMarkdown` is the only place markdown is converted to HTML. It must keep the `html.SkipHTML` flag to prevent raw HTML tags embedded in user-submitted content from reaching the browser.

Current flags (all required):
```go
htmlFlags := html.HrefTargetBlank | html.SkipHTML | html.SkipImages | html.Safelink | html.NofollowLinks | html.NoreferrerLinks
```

- `SkipHTML` — strips `<script>`, `<iframe>`, `<img onerror=...>` and all other raw HTML blocks
- `SkipImages` — strips markdown image syntax (`![alt](url)`) — user-supplied image URLs are a security risk; see [`general.md` § No User-Supplied Image URLs](./general.md)
- `Safelink` — blocks `javascript:` protocol links
- `NofollowLinks` + `NoreferrerLinks` — user-submitted links must not pass SEO signals or referrer data

Do **not** remove any of these flags. The rendered HTML is injected via `@templ.Raw()` in the templ template, so there is no auto-escaping safety net.

### Context key for subdomain
Always retrieve the subdomain via `constant.SubdomainKey`, not the raw string `"subdomain"`. `SubdomainKey` is type `contextKey` (a named type), so `ctx.Value("subdomain")` returns `nil` and causes a panic on type assertion.

```go
// correct
if v, ok := r.Context().Value(constant.SubdomainKey).(string); ok { ... }

// wrong — different key type, always returns nil
subdomain := r.Context().Value("subdomain").(string) // panics
```

### Error responses — always use `response` package helpers
Never call `http.NotFound`, `http.Error`, or write raw error text directly for user-facing error pages. Always use the helpers in `presentation/shared/response/response.go`:

```go
response.Response404(w)  // 404 — styled "Halaman Tidak Ditemukan" page
response.Response500(w)  // 500 — styled "Terjadi Kesalahan" page
```

Both helpers pre-render their pages once at startup via `init()` — there is no per-request rendering cost. Using separate `bytes.Buffer` instances for each page in `init()` is required; sharing one buffer causes the second render to overwrite the first.

`http.Error` is still acceptable for non-user-facing responses (e.g. API endpoints, bot-facing handlers like the honeypot check).

### `allowedPaths` — apex-domain routes without a subdomain

`requireSubdomain` in `main.go` returns 404 for any request that has no subdomain, unless the path is in the `allowedPaths` map or starts with `/admin`. When adding a new public route that should be accessible at the apex domain (`warga.jp/...`), add it to `allowedPaths`:

```go
var allowedPaths = map[string]struct{}{
    "/":                  {},
    "/ping":              {},
    "/robots.txt":        {},
    "/sitemap.xml":       {},
    "/og-image.png":      {},
    "/kebijakan-privasi": {},
    "/syarat-ketentuan":  {},
    "/weblogs":           {},
    "/kontak":            {},
    "/panduan/tambah":    {},
    "/komunitas/tambah":  {},
    // add new apex routes here
}
```

Routes that only exist on subdomains (e.g. `/komunitas`, `/panduan/*`) do **not** need to be in `allowedPaths`. The two submission forms (`/panduan/tambah`, `/komunitas/tambah`) are intentionally apex-accessible — both forms carry a prefecture dropdown so visitors can submit from the apex homepage CTA without first drilling into a subdomain.

**Rule:** Registering a route in `router/router.go` is not enough on its own. Any new page that must be reachable at the apex domain (no subdomain) requires a matching entry in `allowedPaths` in `main.go`. Without it the middleware returns 404 before the router is reached. The checklist for a new apex-domain page is:

1. Add the route in `router/router.go`
2. Add the path to `allowedPaths` in `main.go`

### Trailing-slash routing
`middleware.RedirectSlashes` is registered in `main.go` and issues a 301 redirect from any trailing-slash URL to the canonical no-slash version (e.g. `/admin/` → `/admin`). Do not register duplicate routes with and without a trailing slash — the middleware handles it.

### Admin logout accepts GET and POST
`/admin/logout` is registered for both GET and POST so that `<a href="/admin/logout">` nav links work without a form. This is intentional for an internal admin panel. If a public-facing logout is ever added, it should be POST-only with CSRF.

## Security Headers

Security headers are set in `Caddyfile.prod` (Caddy `header` block), not in Go. This keeps header logic out of application code and lets you update it without a redeploy.

### Current headers

| Header | Value |
|---|---|
| `X-Frame-Options` | `DENY` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `Content-Security-Policy` | `default-src 'self'; script-src 'self'; connect-src 'self'; style-src 'self'; img-src 'self' data:; font-src 'self'; frame-ancestors 'none';` |
| `Server` | removed (`-Server`) |

### CSP rules — no inline styles or scripts

The CSP is `style-src 'self'` and `script-src 'self'`. Both rules are enforced by Caddy and **cannot be overridden in Go code**.

**Never use `style="..."` attributes on any HTML element.** Inline styles are blocked by `style-src 'self'`. Add a class to the appropriate CSS file (`static/base.css`, `static/public.css`, or `static/admin.css`) instead:

```css
/* admin.css */
.badge-dev { font-size: 0.5em; vertical-align: middle; background: #e74c3c; ... }
```

```templ
/* template */
<small class="badge-dev">DEV</small>   ✓
<small style="background:#e74c3c">DEV</small>  ✗  blocked by CSP
```

Only scripts from the same origin are allowed. **Never use inline event handlers** (`onchange="..."`, `onclick="..."`, `onsubmit="..."`) or inline `<script>` tags in any template — admin pages included. They require `'unsafe-inline'` which the CSP explicitly omits, so they are silently blocked in production and nothing happens. This is the root cause behind the admin prefecture selector not navigating.

#### Admin JavaScript — `static/admin.js`

All admin-side behaviour is in `static/admin.js`, served from the same origin and therefore allowed by CSP. It is embedded in the binary via `static/static.go` and registered in `router.go` at `/static/admin.js`.

Add `<script src={ "/static/admin.js?v=" + version.Build } defer></script>` to the `<head>` of any admin template that needs JS.

**Always include `defer`.** Without it the script executes synchronously during HTML parsing, before the rest of the DOM exists. `getElementById` and `querySelectorAll` return `null`/empty at that point, so no listeners are attached and nothing works in production (locally it may appear to work by coincidence). `defer` delays execution until after parsing completes, so all elements are present. Do not wrap code in `DOMContentLoaded` — with `defer` it is unnecessary and redundant.

**Current behaviours wired in `admin.js`:**

| Behaviour | How it is declared in HTML | How `admin.js` handles it |
|---|---|---|
| Prefecture `<select>` auto-navigates on change | `id="prefecture-select"` on the `<select>`, form `action` is the target URL | `change` event listener reads `sel.form.action + '?prefecture=' + value` and assigns `window.location.href` |
| Confirm dialog before destructive action | `data-confirm="…"` on the `<button>` | `click` listener calls `window.confirm(el.dataset.confirm)` and cancels the event if the user declines |

**Pattern for a prefecture selector (do not use `onchange`):**

```templ
<form method="GET" action="/admin/panduan" class="form-filter">
    <label for="prefecture-select">Prefecture:</label>
    <select id="prefecture-select" name="prefecture">
        for _, pref := range vm.Prefectures {
            if pref.Value == vm.SelectedPrefecture {
                <option value={ pref.Value } selected>{ pref.Label }</option>
            } else {
                <option value={ pref.Value }>{ pref.Label }</option>
            }
        }
    </select>
</form>
```

**Pattern for a confirm-before-submit button (do not use `onclick`):**

```templ
<form method="POST" action={ templ.URL(...) } class="form-inline">
    <button type="submit" data-confirm="Yakin ingin menghapus?">Hapus</button>
</form>
```

If adding new external script sources, update `Caddyfile.prod`; no Go code changes are needed.

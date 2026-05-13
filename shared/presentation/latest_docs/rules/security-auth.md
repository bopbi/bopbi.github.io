# Security — Auth & Rate Limiting

## Auth Patterns
- CSRF token required for all POST forms
- Session cookie: `admin_session_token` for admins, `session_token` for users
- Cookie settings: `HttpOnly: true`, `Secure: constant.SecureCookies`, `SameSite: http.SameSiteLaxMode`
- **Admin route gate** — all routes under the protected `/admin` sub-group in `router/router.go` are gated by `adminMiddleware.RequireAdmin` (in `presentation/admin/middleware/auth.go`). Handlers under that group can assume an authenticated admin and read it via `adminmiddleware.AdminFromContext(r.Context())`. Do not reimplement the gate per handler — see `admin-layout.md` § Admin page → Auth guard.

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
| `POST /daftar` _(disabled)_ | 5 attempts / 15 min (own `loginLimiter` in `presentation/user/handler/handler.go`) |
| `POST /login` _(disabled)_ | 5 attempts / 15 min (own `loginLimiter` in `presentation/user/handler/handler.go`) |
| `POST /komunitas/tambah` | 5 submissions / hour |
| `POST /panduan/tambah` | 5 submissions / hour |
| `POST /panduan/tambah` (`action=preview`) | 30 previews / hour (own `previewLimiter`) — markdown render is CPU-bound; submit limiter is bypassed by preview, so it gets its own cap |
| `POST /*/laporkan` | 10 reports / hour (shared limiter across komunitas + panduan report handlers) |
| `POST /kontak` | 3 submissions / hour |
| `GET /?search=` and `GET <subdomain>/?search=` | 20 queries / minute (shared `searchLimiter` in `IndexHandler`) |

`ratelimit.ClientIP(r)` resolves the real IP via `X-Real-IP` → `X-Forwarded-For` → `RemoteAddr`. The chi `middleware.RealIP` in `main.go` already rewrites `r.RemoteAddr` from those headers for the whole request chain, so `ClientIP` is effectively a fallback for callers without the middleware in front. Caddy must set `X-Real-IP` for both layers to work.

Limiters reset on server restart — acceptable for spam prevention. If persistence across restarts is ever needed, move counts to the DB.

**Global / group-scoped rate limit — use `(*Limiter).Middleware`**

For limiters that should apply to a span of routes rather than one handler (e.g. the 200 req/min global limiter in `main.go`), wire the limiter as middleware:

```go
globalLimiter := ratelimit.New(200, time.Minute)
r.Use(globalLimiter.Middleware) // 429 with "Too many requests" before downstream handlers
```

`Middleware` is the canonical wrapper for global / group rate limits. Per-handler limiters (with custom Indonesian copy and per-endpoint thresholds) keep using the inline `limiter.Allow(ratelimit.ClientIP(r))` pattern shown above — bespoke messages aren't yet supported by the helper.

### Admin login brute-force protection
`LoginPostHandler` in `presentation/admin/handler/auth.go` carries its own in-memory `loginLimiter` (predates `lib/ratelimit`). It blocks an IP after **5 failed attempts within 15 minutes** with HTTP 429.

Key rules:
- Call `limiter.recordFailure(ip)` on every credential failure (wrong username/password).
- Call `limiter.reset(ip)` on successful login.
- Do **not** count CSRF failures or empty-field validation against the limiter — those are not credential guesses.
- Constants `maxLoginAttempts = 5` and `loginWindowDur = 15 * time.Minute` are at the top of `auth.go`.

### Password length — 8–72 bytes

All password fields must enforce both a minimum and a maximum length **before** hashing:

```go
if len(newPassword) < 8 {
    renderError("Password minimal 8 karakter")
    return
}
if len(newPassword) > 72 {
    renderError("Password maksimal 72 karakter")
    return
}
```

The upper bound of 72 comes from bcrypt's input limit. bcrypt silently truncates input at 72 bytes; any two passwords that share the same first 72 bytes hash identically. Without the cap, a 200-character password and a 72-character prefix of it become the same credential — a subtle but exploitable collision. Use `len(password)` (byte count) not `len([]rune(password))` here — bcrypt's limit is byte-based.

### Session revocation on credential change

Any handler that changes a password (or other credential) must revoke all existing sessions for that account immediately after the DB write succeeds:

```go
if err := adminRepository.UpdateAdminPassword(adminDB, admin.ID, newPassword); err != nil {
    renderError("Gagal menyimpan password baru")
    return
}
_ = adminRepository.DeleteAllAdminSessionsByAdminID(adminDB, admin.ID)
http.SetCookie(w, &http.Cookie{
    Name:     "admin_session_token",
    Value:    "",
    Path:     "/",
    HttpOnly: true,
    Secure:   constant.SecureCookies,
    MaxAge:   -1,
    SameSite: http.SameSiteLaxMode,
})
http.Redirect(w, r, "/admin/login", http.StatusSeeOther)
```

Without this, a stolen session token remains valid until it expires naturally — the password change provides no incident response benefit. The redirect forces the admin to re-authenticate with the new credential, confirming the change took effect.

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
- `captcha.VerifyOutcome(token, userAnswer string) (Outcome, string)` decodes the token, checks the 30-minute TTL, and confirms the user's answer — returns `(OutcomePass, questionText)`, `(OutcomeFail, questionText)`, or `(OutcomeExpired, "")`. The question text is empty for expired/malformed tokens where no question could be identified.
- `captcha.Verify(token, userAnswer string) bool` is a thin wrapper for call sites that don't need the outcome detail.

**Outcome type:**
```go
type Outcome string
const (
    OutcomePass    Outcome = "pass"
    OutcomeFail    Outcome = "fail"
    OutcomeExpired Outcome = "expired"
)
```

**Viewmodel pattern** (all 4 submission viewmodels):
```go
import "website/lib/captcha"

question, token := captcha.Generate()
// store in vm:
CaptchaQuestion: question,
CaptchaAnswer:   token,   // HMAC token — safe to put in hidden field
```

**Handler pattern** (all 4 POST handlers) — use `VerifyOutcome` so the outcome is recorded:
```go
captchaOutcome, captchaQuestion := captcha.VerifyOutcome(r.FormValue("captcha_answer"), r.FormValue("captcha_user"))
tracker.RecordCaptcha("contact", string(captchaOutcome), captchaQuestion) // form name: contact | report | panduan | community
if captchaOutcome != captcha.OutcomePass {
    renderError("Jawaban captcha salah")
    return
}
```

The secret key is generated once at package `init()` via `crypto/rand`. Tokens do not survive server restarts — in-flight forms submitted after a restart will fail captcha and require a page refresh (acceptable).

### Request body size limit
All POST handlers — public and admin — must call `http.MaxBytesReader` before `ParseForm` to prevent large payload attacks:
```go
r.Body = http.MaxBytesReader(w, r.Body, 64*1024) // 64 KB limit
if err := r.ParseForm(); err != nil {
    http.Error(w, "Invalid form data", http.StatusBadRequest)
    return
}
```
When the body exceeds the limit, `ParseForm` returns an error and the handler returns 400.

Size guidelines:

| Handler type | Limit | Rationale |
|---|---|---|
| Auth forms (login, password, email) | 4 KB | Only short credentials — generously large |
| Content forms with long text (community, panduan add/edit) | 64 KB | Markdown content can be several KB |
| Small admin actions (migration download, list mutations) | 4 KB | Only form params, no large payloads |

Admin handlers are behind auth + CSRF, but the cap is defence-in-depth against oversized requests that could consume memory or slow the server.

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

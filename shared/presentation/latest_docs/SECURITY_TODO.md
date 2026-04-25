# Security TODO

Findings from a broader security audit (beyond the panduan markdown preview feature). This list is the authoritative backlog — once an item is fixed, remove it from this file. When the file is empty, delete it.

Items are grouped by exploitability. Fix **Live issues** first.

---

## Live issues — exploitable in production now

### [x] #7 No CSRF token on admin list-row mutation POSTs — FIXED

All four list pages now carry a per-request `csrf_token` on every mutation form, and each handler calls `adminRepository.VerifyCSRFToken(w, r)` before any state change. The shared helper lives at `domain/admin/repository/repository.go` so new list-row POSTs can adopt the pattern with one line.

---

### [x] #1 Admin session expiry never checked — FIXED

**Where:** `domain/admin/repository/repository.go:103-116` (`GetAdminSessionByToken`)

**Problem:** The query fetches sessions by `session_token` alone. It does not filter on `expires_at`, so a stolen or leaked session cookie stays valid forever — even long after the 7-day TTL written at login. The `expires_at` column exists and is populated, but nothing reads it.

```go
err := db.QueryRow(`
    SELECT id, admin_id, session_token, expires_at, created_at
    FROM admin_sessions
    WHERE session_token = ?
`, token).Scan(...)
```

**Fix:** Add `AND expires_at > datetime('now')` to the WHERE clause so expired rows are treated as if they don't exist. Consider a lightweight periodic sweep (`DELETE FROM admin_sessions WHERE expires_at < datetime('now')`) so the table doesn't grow without bound, but the filter is the load-bearing fix.

**Blast radius:** High — any session token ever issued remains a valid admin credential until the row is manually deleted.

---

### [x] #2 No body-size cap on admin login — FIXED

**Where:** `presentation/admin/handler/auth.go:106` (`LoginPostHandler`)

**Problem:** Every other form handler in the repo wraps `r.Body` in `http.MaxBytesReader` before `ParseForm`. The admin login handler does not, so a client can POST an arbitrary-size body and force the server to buffer all of it.

**Fix:** Add at the top of the handler, before `ParseForm`:
```go
r.Body = http.MaxBytesReader(w, r.Body, 4*1024)
```

**Blast radius:** Low — login is already rate-limited (5/15min/IP), so the practical amplification is bounded. This is a consistency/defence-in-depth fix, not an active bleed.

---

## Dead-code issues — not exploitable today, but fix before re-enabling

These live in `presentation/user/handler/handler.go`, which is currently **not wired** into the router (the signup/login routes are commented out at `router/router.go:119-123`). Fix these before un-commenting, or delete the handler if it's abandoned.

### [ ] #3 Username enumeration via error leak

**Where:** `presentation/user/handler/handler.go:59`

**Problem:** On signup failure, the handler writes `"Failed to create user: " + err.Error()` straight to the response. A duplicate-username insert will leak the DB constraint message, letting an attacker enumerate existing usernames.

**Fix:** Return a generic message (`"Gagal mendaftar"`) and log the underlying error server-side.

---

### [ ] #4 Missing `return` after `http.NotFound` in signup

**Where:** `presentation/user/handler/handler.go:52-54`

**Problem:**
```go
if !found {
    http.NotFound(w, r)
}
// …execution continues with zero-value prefecture
```
After writing the 404, the handler keeps running and can touch a zero-value prefecture struct. Latent bug — behaviour undefined depending on downstream reads.

**Fix:** Add `return` immediately after `http.NotFound(w, r)`.

---

### [ ] #5 `Secure: true` hardcoded on user session cookies

**Where:** `presentation/user/handler/handler.go:85`, `:154`, `:179`

**Problem:** Cookies are set with `Secure: true` unconditionally. This breaks dev over HTTP. Every other handler in the repo uses `constant.SecureCookies` (which switches on environment).

**Fix:** Replace the three literals with `constant.SecureCookies`.

---

### [ ] #6 User signup/login lacks rate limit, body cap, captcha

**Where:** `presentation/user/handler/handler.go` — both the signup POST and login POST.

**Problem:** Neither handler has any of the submission guardrails applied to the rest of the codebase:
- no `http.MaxBytesReader`
- no `lib/ratelimit` wrapper
- no captcha / honeypot

**Fix:** Before re-enabling the routes, mirror the admin-login pattern (`5 attempts / 15 min / IP`), add a body cap, and add a honeypot field. Captcha is a judgement call — match whatever the contact/panduan forms use at the time of re-enable.

---

## Audited and clean (reference)

Recording the "we checked this" items so the next audit doesn't redo them from scratch:

- **SQL injection:** all 82 query call sites use parameterised placeholders; no string concatenation into SQL.
- **XSS / `@templ.Raw`:** only used for hardened gomarkdown output (SkipHTML + URL-block hook) and for server-built search-excerpt highlighting with escaped input.
- **Email header injection:** contact form uses `mail.ParseAddress`, which rejects CRLF per RFC 5322.
- **OG image DoS:** generation guarded by `sync.Once`.
- **Search:** rate-limited, query length capped, result set capped.
- **CSRF cookie `HttpOnly`:** correct — tokens are rendered server-side into the form, so JS never needs to read them.

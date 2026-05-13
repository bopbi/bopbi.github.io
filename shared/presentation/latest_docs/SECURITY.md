# Security

Overview of the security model, lessons from the audit, and a new-handler checklist. For implementation patterns (CSRF, rate limiting, captcha, field caps) see [`docs/rules/security-auth.md`](./rules/security-auth.md) and [`docs/rules/security-validation.md`](./rules/security-validation.md).

---

## Security model

Defense in depth: every POST handler passes through multiple independent layers. No single layer is load-bearing on its own.

```
internet → Caddy (TLS, connection timeouts, header size limit)
         → Go server (ReadHeaderTimeout, ReadTimeout, WriteTimeout, IdleTimeout)
         → global per-IP rate limiter (200 req/min — all endpoints)
         → per-endpoint rate limiter (5–30 req/min — auth and submission forms)
         → body cap → ParseForm → honeypot → CSRF → captcha
         → field caps → enum whitelist → email validation → DB write
```

Sessions are validated at the DB query level with `AND expires_at > datetime('now')` so the expiry is enforced even if application logic has a gap. Security headers (CSP, X-Frame-Options, etc.) are set in Caddy, not in Go, so they apply to every response regardless of handler.

## DDoS prevention

Three layers work together. Do not remove any of them.

### Caddy — connection and timeout hardening (`Caddyfile.prod`)

The global `servers` block in `Caddyfile.prod` sets timeouts on all connections before they reach Go:

| Setting | Value | Protects against |
|---|---|---|
| `read_header` | 5s | Slowloris (attacker sends headers one byte at a time, never completing) |
| `read_body` | 15s | Slow POST (attacker dribbles the request body) |
| `write` | 30s | Slow read (attacker reads the response one byte at a time, holding the connection) |
| `idle` | 90s | Keep-alive abuse (attacker holds idle connections open) |
| `max_header_size` | 64 KB | Header flood (attacker sends huge Cookie or custom headers) |

### Go server — defence-in-depth timeouts (`main.go`)

The app binds to `127.0.0.1:8080`, not `0.0.0.0:8080`. This means only processes on the same machine (i.e. Caddy) can reach Go, regardless of firewall configuration. It also means IP-based protections (rate limiting, header parsing) cannot be bypassed by forging `X-Real-IP` from an external connection.

`http.Server` carries the same timeout set at the Go level. Since the Go app only receives connections from Caddy (localhost), these protect against anything that bypasses the reverse proxy:

```go
ReadHeaderTimeout: 5 * time.Second,
ReadTimeout:       15 * time.Second,
WriteTimeout:      30 * time.Second,
IdleTimeout:       90 * time.Second,
MaxHeaderBytes:    1 << 16, // 64 KB
```

### Go middleware — global per-IP rate limiter (`main.go`)

A `globalLimiter.Middleware` (from `lib/ratelimit`) is wired into the chi chain in `main.go` and applies to **all routes** before any routing or DB access, capping each IP at **200 requests per minute**. This blocks single-source floods against static files, GET content pages, the sitemap, and any other endpoint that has no per-handler limiter.

The 200 req/min threshold (≈3.3 req/sec) is generous for real users but blocks simple flood tools. The middleware sits after `chi/middleware.Logger` (so blocked requests appear in logs for incident triage) and after `chi/middleware.RealIP` (so the limiter and Logger see the visitor's real IP, not Caddy's loopback). It runs before `response.Recoverer` and `requireSubdomain` to short-circuit all downstream work.

Per-endpoint limiters (login, form submissions, search) add a second, tighter layer on top for high-value targets. See `docs/rules/security-auth.md` for the full rate limit table.

---

## PII isolation

Submitter contact fields (`submitter_email`, `submitter_whatsapp`, `submitter_line`) are collected at submission time and wiped from the DB on approval via `ClearCommunitySubmitterContact` / `ClearGenericInfoPostSubmitterContact`. That is the primary protection.

A second, structural layer: public-path repository functions do not SELECT contact columns at all. Two scan helpers enforce this at the row level:

- `scanPublicCommunityInfo` — 8 columns, no submitter fields. Used by `GetApprovedCommunityPage`, `GetCommunityByIDAndStatus` (→ `GetCommunityByID`), `SearchCommunity`.
- `scanPublicGenericInfoPost` — 8 columns, no submitter fields. Used by `GetApprovedGenericInfoPostPage`, `GetGenericInfoPostByIDAndStatus` (→ `GetGenericInfoPostByID`), `SearchGenericInfoPost`, `GetRelatedPosts`.

Admin-path functions (`GetCommunityByIDAnyStatus`, `GetAllCommunityWithPending`, `SearchCommunityWithPending`, and the equivalent `generic_info_post` variants) use the full scan helpers — moderators need contact details before approval.

Template-level protection ("the template just doesn't render the field") is fragile: it breaks silently when a viewmodel field is added or a template is copied. Structural isolation means contact fields cannot appear in a public response even if a template or viewmodel is modified carelessly. The data simply isn't in the struct.

When adding a new public listing or detail query: use `scanPublicCommunityInfo` / `scanPublicGenericInfoPost` (or add an equivalent public scan helper for new tables). Admin queries that need contact data use the full scan helpers. See `domain/community/repository/repository.go` for the reference implementation.

---

## Lessons from the audit

Nine lessons from a full codebase audit: see **[`docs/security-audit-lessons.md`](security-audit-lessons.md)**.

---

## New handler checklist

The correct sequence for any POST handler. Each step must come before the ones below it.

| Step | Code | Notes |
|---|---|---|
| 1. Rate limit | `if limiter.isBlocked(ip) { ... return }` | Auth: count credential failures. Submissions: count all requests. |
| 2. Body cap | `r.Body = http.MaxBytesReader(w, r.Body, N)` | Before `ParseForm`. Auth: 4 KB. Forms with long text: 64 KB. |
| 3. Parse form | `r.ParseForm()` | Return 400 on error. |
| 4. Honeypot | `if r.FormValue("website") != "" { return }` | Silent discard — no error body. |
| 5. CSRF | Compare `r.FormValue("csrf_token")` with cookie | Return 403 on mismatch. |
| 6. Required fields | Check each required value is non-empty | Return 400 with generic message. |
| 7. Field length caps | `len([]rune(value)) > limit` | Rune count, not byte count. |
| 8. Enum whitelist | Check select values against the model's allowed set | Return 400 on invalid value. |
| 9. Email validation | `mail.ParseAddress(email)` | Rejects CRLF, preventing header injection. |
| 10. Captcha | `captcha.Verify(token, userAnswer)` | Public submission forms only; not auth. |
| 11. DB write | `repository.Create(...)` | Generic error response; log real error. |
| 12. Side effects | `go notify.Submission(...)` | Async; must not block the response. |

Not every handler uses every step. Steps 8–10 only apply where relevant. Steps 1 and 10 apply differently to auth vs submission forms (see notes).

---

## What was audited and found clean

Items confirmed during the audit. Record them here so the next audit knows what was already checked.

| Area | Finding |
|---|---|
| SQL injection | All query call sites use parameterised placeholders. No string concatenation into SQL anywhere in the codebase. |
| XSS / `@templ.Raw` | Only used for hardened gomarkdown output (`SkipHTML` + URL-block hook) and for server-built search-excerpt highlighting with escaped input. No user-controlled strings reach `@templ.Raw`. |
| Email header injection | Contact form uses `mail.ParseAddress`, which rejects CRLF per RFC 5322. |
| OG image DoS | Generation guarded by `sync.Once`. Can only be triggered once per server lifetime. |
| Search | Rate-limited, query length capped, result set capped. |
| CSRF cookie `HttpOnly` | Correct — tokens are rendered server-side into the form, so JS never needs to read them. `HttpOnly` does not prevent the pattern. |
| PII in public responses | Public-path repository queries do not SELECT contact columns. `scanPublicCommunityInfo` / `scanPublicGenericInfoPost` enforce this structurally. |
| Session expiry (admin + user) | Both `GetAdminSessionByToken` and `GetSessionByToken` enforce `AND expires_at > datetime('now')` in the DB query. A stolen token returns no row once it expires. |
| Password max length | Admin password change caps at 72 bytes before hashing. Prevents bcrypt's silent truncation collision (passwords sharing the same first 72 bytes would otherwise be equivalent). |
| HSTS | `Strict-Transport-Security: max-age=31536000; includeSubDomains` set in `Caddyfile.prod`. Browser caches HTTPS-only instruction for 1 year across all subdomains. |
| CSP `object-src` | Explicitly `object-src 'none'` in `Caddyfile.prod`. Blocks Flash and browser plugin execution rather than falling back to `default-src 'self'`. |
| Loopback binding | App binds to `127.0.0.1:8080`. Direct external access is structurally impossible; `X-Real-IP` forgery cannot bypass rate limits. |
| Admin session revocation on password change | `ChangePasswordPostHandler` calls `DeleteAllAdminSessionsByAdminID` after the hash update, then clears the cookie and redirects to login. Stolen tokens do not survive a password change. |
| Admin POST body caps | All admin POST handlers (`ChangePassword`, `ChangeEmail`, `CommunityAdd`, `CommunityEdit`, `PanduanAdd`, `PanduanEdit`, migration download) cap request bodies with `http.MaxBytesReader` before `ParseForm`. |
| Generic errors in public handlers | `komunitas/tambah` and `panduan/tambah` POST handlers return a generic error string; the real `err.Error()` is logged server-side only. |

---

## References

- [Ruby on Rails Security Guide](https://guides.rubyonrails.org/security.html) — framework-agnostic reference for web security patterns: session management, CSRF, injection, password handling, redirects, file uploads, CSP, and more. Used as the basis for the audit that surfaced the HSTS gap, bcrypt truncation risk, and session expiry issue.

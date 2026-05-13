# Security Audit Lessons

Extracted from a full codebase audit. Each lesson produced a principle that now applies to all handlers. For the active implementation rules and checklist, see [`SECURITY.md`](SECURITY.md) and [`docs/rules/security-auth.md`](rules/security-auth.md) / [`docs/rules/security-validation.md`](rules/security-validation.md).

---

## Lessons from the audit

Seven issues were found. Each one produced a principle that now applies to all handlers.

### 1. Session expiry belongs in the SQL WHERE clause

**What happened:** `GetAdminSessionByToken` fetched rows by `session_token` alone. The `expires_at` column existed and was populated, but was never read. A stolen token stayed valid forever.

**Principle:** Never rely on application code to enforce a DB-level constraint. Write the check into the query so the row simply doesn't exist from the DB's perspective when it's expired. Checking in Go after the fetch means the row is already loaded and partially trusted.

```sql
-- correct
WHERE session_token = ? AND expires_at > datetime('now')

-- wrong — the app then has to remember to check expires_at itself
WHERE session_token = ?
```

### 2. Auth endpoints need the same guardrails as public forms

**What happened:** The admin login handler had no `http.MaxBytesReader` call; the user signup/login handlers had no body cap, no rate limit, and no honeypot.

**Principle:** Auth endpoints are higher-value targets than submission forms. Apply every submission-form guardrail to auth endpoints too. The risk of missing a guardrail on `/login` is higher than missing it on `/panduan/tambah` — attackers specifically look for unprotected auth surfaces.

The required set for any auth POST:
- `http.MaxBytesReader` (body cap)
- IP-based rate limiter with failure counting (not just request counting)
- Honeypot check

Rate limiters on auth must count **credential failures**, not all requests — CSRF failures and blank-field validation do not count. Only wrong username/password counts, because that's what a brute-force looks like.

### 3. Error messages must be generic — auth and DB writes alike

**What happened:** The signup handler returned `"Failed to create user: " + err.Error()`. A duplicate-username DB constraint produced a message like `UNIQUE constraint failed: users.username`, which lets an attacker enumerate valid usernames by watching the error message change. The same pattern was found in public submission handlers (`komunitas/tambah`, `panduan/tambah`) where `err.Error()` was appended to the user-facing message.

**Principle:** Any error that originates from the DB or internal logic is a potential data leak. Return one generic message for all failure modes. Log the real error server-side.

```go
// correct
log.Printf("CreateCommunity error: %v", err)
renderError("Gagal mengirim pengajuan")

// wrong — leaks SQLite constraint details to end users
renderError("Gagal mengirim pengajuan: " + err.Error())
```

The same rule applies to login: return "Username atau password salah" regardless of whether the username doesn't exist or the password is wrong. Two different messages allow username enumeration.

### 4. Always return after writing an HTTP response

**What happened:** The signup handler called `http.NotFound(w, r)` but had no `return`. Execution continued with a zero-value `prefecture` struct, and the handler then tried to create a user with that empty value.

**Principle:** Writing to `http.ResponseWriter` does not stop execution. Every call to `http.Error`, `http.NotFound`, `http.Redirect`, or any response-writing function must be immediately followed by `return`.

```go
// correct
if !found {
    http.NotFound(w, r)
    return
}

// wrong — execution continues with zero-value prefecture
if !found {
    http.NotFound(w, r)
}
```

Go does not raise an error if you write to a `ResponseWriter` that already has a status code — it silently ignores the second write. The bug manifests not as a crash but as unexpected behaviour downstream.

### 5. Never hardcode environment-specific values

**What happened:** Three cookie `Secure: true` literals in the user handler broke local development over HTTP. Every other cookie in the codebase used `constant.SecureCookies`, which switches based on the domain name.

**Principle:** Any value that differs between environments (dev/prod) must live in a named constant or environment variable. Hardcoded values pass in production, silently break in dev, and are invisible in code review.

```go
// correct — works in both environments
Secure: constant.SecureCookies,

// wrong — breaks dev over HTTP
Secure: true,
```

`constant.SecureCookies` is set to `false` when `domainName == "machine.local"` and `true` everywhere else. Use it for every cookie, including session-clearing (logout) cookies.

### 6. CSRF protection must cover every mutation, including admin list actions

**What happened:** Admin list pages had mutation forms (approve, reject, delete) with no CSRF token. Any page the admin visited while logged in could trigger those actions via a cross-origin POST.

**Principle:** CSRF tokens are required for every form that changes state — not just public-facing forms. Admin-only forms are still susceptible to CSRF attacks because the admin's browser makes the request with their session cookie regardless of where the form came from.

The `adminRepository.VerifyCSRFToken(w, r)` helper reduces this to one line per handler. New mutation forms must include both the hidden field in the template and the verify call in the handler.

### 7. Disabled code still gets audited

**What happened:** The user signup/login handler was disabled in the router, which led to its security issues being deferred. When it came time to re-enable, it had four separate issues accumulated from the initial write.

**Principle:** Commented-out routes are pre-loaded technical debt. Either fix the code now and keep it ready to ship, or delete it. A handler sitting in the repo in a broken state will eventually be uncommented without anyone remembering to check it first.

### 8. Bind the app to loopback, not all interfaces

**What happened:** The app listened on `:8080` (all interfaces). If port 8080 was reachable directly — due to a firewall gap or a second network interface — an attacker could bypass Caddy entirely: no TLS, no security headers, and no trusted reverse-proxy context. Without that trusted context, `X-Real-IP` and `X-Forwarded-For` headers are attacker-controlled, so rate limits (which key on client IP) could be evaded by forging those headers.

**Principle:** Bind to `127.0.0.1` explicitly. This is one character change but closes the entire class of "direct port access" attacks. The app only ever needs to be reachable from Caddy on the same machine.

```go
// correct — loopback only
addr := "127.0.0.1:8080"

// wrong — reachable on every network interface
addr := ":8080"
```

### 9. Revoke all sessions when a password changes

**What happened:** `ChangePasswordPostHandler` updated the password hash but left all `admin_sessions` rows in place. A stolen session token remained valid until natural expiry even after the password changed — undermining account recovery and incident response.

**Principle:** A password change is an implicit "sign out everywhere." After writing the new hash, delete all sessions for that admin and redirect to login. This is a one-line call (`DeleteAllAdminSessionsByAdminID`) plus a cookie clear and redirect.

```go
// after UpdateAdminPassword succeeds
_ = adminRepository.DeleteAllAdminSessionsByAdminID(adminDB, admin.ID)
http.SetCookie(w, &http.Cookie{Name: "admin_session_token", MaxAge: -1, ...})
http.Redirect(w, r, "/admin/login", http.StatusSeeOther)
```

The same principle applies to any future "change email" or "change secret" flow — any credential rotation should invalidate existing sessions.

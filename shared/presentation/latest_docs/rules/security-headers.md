# Security Headers

Security headers are set in `Caddyfile.prod` (Caddy `header` block), not in Go. This keeps header logic out of application code and lets you update it without a redeploy.

## Current headers

| Header | Value |
|---|---|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` — forces HTTPS for 1 year on all subdomains |
| `X-Frame-Options` | `DENY` |
| `X-Content-Type-Options` | `nosniff` |
| `Referrer-Policy` | `strict-origin-when-cross-origin` |
| `X-XSS-Protection` | `0` — disables the broken legacy browser XSS auditor |
| `Content-Security-Policy` | see below |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=(), payment=(), usb=(), interest-cohort=()` |
| `Server` | removed (`-Server`) |
| `X-Powered-By` | removed (`-X-Powered-By`) |

Full CSP value:
```
default-src 'self';
script-src 'self';
connect-src 'self';
style-src 'self';
img-src 'self' data:;
font-src 'self';
object-src 'none';
frame-ancestors 'none';
form-action 'self';
base-uri 'self';
upgrade-insecure-requests;
```

Key directives to know:
- `object-src 'none'` — blocks Flash and browser plugins explicitly
- `form-action 'self'` — prevents forms from submitting to external URLs
- `base-uri 'self'` — prevents `<base href="...">` injection from hijacking relative links
- `upgrade-insecure-requests` — forces HTTPS for all sub-resources

## CSP rules — no inline styles or scripts

The CSP is `style-src 'self'` and `script-src 'self'`. Both rules are enforced by Caddy on production and **cannot be overridden in Go code**.

> **Local dev has no Caddy, so inline style violations are invisible locally.** The page looks fine on `http://machine.local` but breaks silently on production. Always test layout-critical pages after deploy, or run with `APP_ENV=production` behind a local Caddy instance.

**Never use `style="..."` attributes on any HTML element.** Inline styles are blocked by `style-src 'self'`. Add a class to the appropriate CSS file (`static/base.css`, `static/public.css`, or `static/admin.css`) instead:

```css
/* admin.css */
.badge-dev { font-size: 0.5em; vertical-align: middle; background: #e74c3c; ... }
```

```templ
/* template */
<small class="badge-dev">DEV</small>   ✓
<small style="background:#e74c3c">DEV</small>  ✗  blocked by CSP — silent on local, broken on prod
```

**Layout inline styles are the most common offender.** A body centering pattern like `<body style="display:flex; justify-content:center">` looks fine locally, but on production the flex is silently stripped — content falls to the top-left. The admin login page was broken this way. Always put layout CSS in a class.

Only scripts from the same origin are allowed. **Never use inline event handlers** (`onchange="..."`, `onclick="..."`, `onsubmit="..."`) or inline `<script>` tags in any template — admin pages included. They require `'unsafe-inline'` which the CSP explicitly omits, so they are silently blocked in production and nothing happens. This is the root cause behind the admin prefecture selector not navigating.

> For `admin.js` behaviour patterns (which templates load it, `data-confirm`, `data-copy`, prefecture selector): see [`admin-js.md`](./admin-js.md).

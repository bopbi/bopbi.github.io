# Development Setup: Warga.jp

## Overview

```
Browser
  → http://<pref>.machine.local:8080   (subdomain resolved via /etc/hosts)
  → Go app :8080                        (subdomain extracted from Host header)
  → dbs/<pref>.db                       (SQLite file auto-created on first run)
```

---

## 1. Prerequisites

| Tool | Install |
|---|---|
| Go 1.25+ | [go.dev/dl](https://go.dev/dl) |
| `templ` CLI | `go install github.com/a-h/templ/cmd/templ@latest` |
| C compiler (CGo) | macOS: Xcode Command Line Tools (`xcode-select --install`) |

> `x86_64-linux-musl-gcc` is only needed for production cross-compilation (`scripts/build-prod.sh`) — not for local dev.

---

## 2. First Run

```bash
git clone <repo-url>
cd wargajp
templ generate
go run .
```

Server starts on `:8080`. Open `http://machine.local:8080`.

---

## 3. `/etc/hosts` Setup — Subdomain Routing

The app reads the subdomain from the `Host` header to select the correct prefecture DB. Without `/etc/hosts` entries, subdomain URLs won't resolve locally.

Edit `/etc/hosts` (requires `sudo`):

```bash
sudo nano /etc/hosts
```

### Apex domain (required)

```
127.0.0.1   machine.local
::1         machine.local
```

### All 47 prefecture subdomains — IPv4

```
127.0.0.1   aichi.machine.local
127.0.0.1   akita.machine.local
127.0.0.1   aomori.machine.local
127.0.0.1   chiba.machine.local
127.0.0.1   ehime.machine.local
127.0.0.1   fukui.machine.local
127.0.0.1   fukuoka.machine.local
127.0.0.1   fukushima.machine.local
127.0.0.1   gifu.machine.local
127.0.0.1   gunma.machine.local
127.0.0.1   hiroshima.machine.local
127.0.0.1   hokkaido.machine.local
127.0.0.1   hyogo.machine.local
127.0.0.1   ibaraki.machine.local
127.0.0.1   ishikawa.machine.local
127.0.0.1   iwate.machine.local
127.0.0.1   kagawa.machine.local
127.0.0.1   kagoshima.machine.local
127.0.0.1   kanagawa.machine.local
127.0.0.1   kochi.machine.local
127.0.0.1   kumamoto.machine.local
127.0.0.1   kyoto.machine.local
127.0.0.1   mie.machine.local
127.0.0.1   miyagi.machine.local
127.0.0.1   miyazaki.machine.local
127.0.0.1   nagano.machine.local
127.0.0.1   nagasaki.machine.local
127.0.0.1   nara.machine.local
127.0.0.1   niigata.machine.local
127.0.0.1   oita.machine.local
127.0.0.1   okayama.machine.local
127.0.0.1   okinawa.machine.local
127.0.0.1   osaka.machine.local
127.0.0.1   saga.machine.local
127.0.0.1   saitama.machine.local
127.0.0.1   shiga.machine.local
127.0.0.1   shimane.machine.local
127.0.0.1   shizuoka.machine.local
127.0.0.1   tochigi.machine.local
127.0.0.1   tokushima.machine.local
127.0.0.1   tokyo.machine.local
127.0.0.1   tottori.machine.local
127.0.0.1   toyama.machine.local
127.0.0.1   wakayama.machine.local
127.0.0.1   yamagata.machine.local
127.0.0.1   yamaguchi.machine.local
127.0.0.1   yamanashi.machine.local
```

### All 47 prefecture subdomains — IPv6

```
::1   aichi.machine.local
::1   akita.machine.local
::1   aomori.machine.local
::1   chiba.machine.local
::1   ehime.machine.local
::1   fukui.machine.local
::1   fukuoka.machine.local
::1   fukushima.machine.local
::1   gifu.machine.local
::1   gunma.machine.local
::1   hiroshima.machine.local
::1   hokkaido.machine.local
::1   hyogo.machine.local
::1   ibaraki.machine.local
::1   ishikawa.machine.local
::1   iwate.machine.local
::1   kagawa.machine.local
::1   kagoshima.machine.local
::1   kanagawa.machine.local
::1   kochi.machine.local
::1   kumamoto.machine.local
::1   kyoto.machine.local
::1   mie.machine.local
::1   miyagi.machine.local
::1   miyazaki.machine.local
::1   nagano.machine.local
::1   nagasaki.machine.local
::1   nara.machine.local
::1   niigata.machine.local
::1   oita.machine.local
::1   okayama.machine.local
::1   okinawa.machine.local
::1   osaka.machine.local
::1   saga.machine.local
::1   saitama.machine.local
::1   shiga.machine.local
::1   shimane.machine.local
::1   shizuoka.machine.local
::1   tochigi.machine.local
::1   tokushima.machine.local
::1   tokyo.machine.local
::1   tottori.machine.local
::1   toyama.machine.local
::1   wakayama.machine.local
::1   yamagata.machine.local
::1   yamaguchi.machine.local
::1   yamanashi.machine.local
```

> You don't need all 47. Only add the prefectures you're actively testing. The apex entries (`machine.local`) are always required.

---

## 4. How Subdomain Routing Works in Dev

`main.go` calls `constant.SetDomainName("machine.local")` when `APP_ENV` is unset (development mode). The `subdomainMiddleware` extracts the part before the first `.` in the `Host` header:

- `machine.local` → no subdomain → serves apex routes (`/`, `/kontak`, admin, etc.)
- `tokyo.machine.local` → subdomain `tokyo` → opens `dbs/tokyo.db`

Routes that require a subdomain (e.g. `/komunitas`, `/panduan/*`) return 404 on the apex domain — this matches production behaviour.

---

## 5. Database Files

`migrations.ExecuteMigration()` runs at startup and auto-creates any missing SQLite DB files in `dbs/`. No manual setup needed — just run the app and the files appear.

All DB files are opened with WAL journal mode (`_journal_mode=WAL`), 5-second busy timeout, and `SetMaxOpenConns(1)`. In dev you will see `-wal` and `-shm` sidecar files alongside each `.db` file — that is normal and expected.

Files created on first run:

| File | Purpose |
|---|---|
| `dbs/<prefecture>.db` | One per prefecture — content (panduan, komunitas, etc.) |
| `dbs/user.db` | Registered users and sessions |
| `dbs/admin.db` | Admin accounts, sessions, and featured content |
| `dbs/analytics.db` | Page-view analytics (see Section 9 below) |

The `dbs/` directory is gitignored. To share seed data between machines, copy the `.db` files manually or use `rsync`.

### Creating the admin account

`dbs/admin.db` is created automatically on first run, but it contains no admin users. Create one with:

```bash
go run scripts/create_admin.go -username=admin -email=admin@example.com -password=yourpassword
```

Run this from the project root. The script creates `dbs/admin.db` and the `admins` table if they don't exist, hashes the password with bcrypt, and inserts the row. Running it again with the same username or email will fail with an error (UNIQUE constraint) rather than overwriting.

After creating the account, log in at `http://machine.local:8080/admin/login`.

> **After wiping `dbs/`:** re-run this script before starting the app — the app does not seed any admin users automatically.

---

## 6. Rebuilding Templates

Run after any `.templ` file change:

```bash
templ generate
go run .
```

For continuous development, use watch mode in a separate terminal:

```bash
templ generate --watch
```

Then in another terminal:

```bash
go run .
```

---

## 7. Environment Variables

| Variable | Dev default | Effect |
|---|---|---|
| `APP_ENV` | `""` | Empty = development; set to `"production"` for prod mode |
| `SMTP_HOST` | unset | Email notifications disabled when unset |
| `SMTP_PORT` | unset | |
| `SMTP_USER` | unset | |
| `SMTP_PASS` | unset | |
| `NOTIFY_TO` | unset | Recipient for submission/contact notifications |

SMTP is fully optional in development — form submissions save to the DB normally; only the email notification is skipped.

---

## 8. Analytics in Dev

The analytics middleware is active in development. Both page views and search queries are written to `dbs/analytics.db` automatically — no setup needed.

Two things behave differently from production:

- **Country is always `XX`** — there is no Cloudflare proxy in dev, so the `CF-IPCountry` header is never set.
- **IP is `::1` or `127.0.0.1`** — all requests come from localhost, so all visitor hashes collapse to a single value per day.

To view the dashboard, log in to the admin panel and visit:

```
http://machine.local:8080/admin/analitik
```

To inspect the raw data:

```bash
# Page views
sqlite3 dbs/analytics.db "SELECT subdomain, path, country, seen_at FROM pageviews ORDER BY seen_at DESC LIMIT 20;"

# Search queries
sqlite3 dbs/analytics.db "SELECT query, subdomain, seen_at FROM search_queries ORDER BY seen_at DESC LIMIT 20;"
```

> Admin routes (`/admin/*`) are intentionally excluded from page-view tracking. Search queries are only recorded after the query passes length validation (≥ 3 chars) and the rate limiter.

---

## 9. Quick Smoke Test

After `go run .` starts:

```bash
# Apex domain
curl -s http://machine.local:8080/ping           # → "pong"

# Prefecture subdomain
curl -sI http://tokyo.machine.local:8080/         # → HTTP 200

# 404 page
curl -sI http://machine.local:8080/tidak-ada     # → HTTP 404 with styled page

# Analytics dashboard (not logged in → redirect, not 500)
curl -sI http://machine.local:8080/admin/analitik | head -2
```

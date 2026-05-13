# Analytics

Analytics are built into the binary — no separate service, no JS beacon, no GeoIP database. `analytics.db` is created by the app on startup.

## How it works

- **Page-view tracking:** `lib/analytics/tracker.go` middleware fires a goroutine after each response. Filters out `/admin`, `/static/`, `/ping`, `/robots.txt`, `/sitemap.xml`, `/og-image.png`, and non-GET requests.
- **Search query tracking:** `RecordSearch()` is called from `IndexHandler` after each validated search (≥ 3 chars). Writes one row to `search_queries` per search.
- **Captcha tracking:** `RecordCaptcha(form, outcome, question string)` is called from each public POST handler immediately after `captcha.VerifyOutcome()`. Records the form name (`contact`, `report`, `panduan`, `community`), outcome (`pass`/`fail`/`expired`), and the question text if it could be determined. Question is empty for expired or malformed tokens.
- **Country detection:** reads Cloudflare's `CF-IPCountry` header (free on every proxied request). Defaults to `XX` when absent (DNS-only mode or local dev).
- **Visitor hashing:** SHA-256 of `(daily_salt | IP | UserAgent)`, truncated to 16 bytes. Salt rotates at UTC midnight — same visitor collapses to one hash within a day; hashes are unrecoverable the next day (GDPR-friendly, no cookies). Not applied to captcha records — aggregate counts only.
- **Writes are non-blocking** — channel-enqueued to a single background writer goroutine; dropped silently if the 512-slot buffer is full.

## Dashboard

Log in and visit `/admin/analitik`. Shows:
- Total pageviews and approximate daily-unique visitors over a rolling 7 / 30 / 90-day window
- **Halaman Teratas** — top paths by subdomain + path, paginated at 20 rows (`?paths_page=N`)
- **Pencarian Teratas** — top search queries by query + subdomain, paginated at 20 rows (`?searches_page=N`)
- **Negara** — country breakdown (2-letter ISO; `XX` = unknown), capped at 20 rows
- **Captcha** — pass/fail/expired totals; per-form breakdown with fail rate; top-10 questions with the most fail answers (hidden until data exists)

The `?days=` param is whitelisted to `7`, `30`, or `90` — arbitrary values default to 30.

## Performance

`analytics.db` is opened with WAL journal mode, decoupled from the main app write mutex. The dashboard uses composite covering indexes (`idx_pv_covering`, `idx_sq_covering`, `idx_cv_covering`) so GROUP BY queries never need a table scan.

If `PRAGMA journal_mode;` returns `delete` instead of `wal` on any DB, the file was re-created without the pragma (e.g. by an external tool). Re-run with the correct DSN.

## Dev behaviour

The analytics middleware is active in development. Both page views and search queries are written to `dbs/analytics.db` automatically — no setup needed.

Two things behave differently from production:

- **Country is always `XX`** — there is no Cloudflare proxy in dev, so the `CF-IPCountry` header is never set.
- **IP is `::1` or `127.0.0.1`** — all requests come from localhost, so all visitor hashes collapse to a single value per day.

To inspect the raw data:

```bash
# Page views
sqlite3 dbs/analytics.db "SELECT subdomain, path, country, seen_at FROM pageviews ORDER BY seen_at DESC LIMIT 20;"

# Search queries
sqlite3 dbs/analytics.db "SELECT query, subdomain, seen_at FROM search_queries ORDER BY seen_at DESC LIMIT 20;"
```

## Deployment notes

- No extra setup steps are needed — `analytics.db` is created by the app on startup.
- **Country data requires Cloudflare proxy to be on (orange cloud).** When running in DNS-only mode, all countries appear as `XX`.
- For pruning old rows, see `docs/vps-operations.md` § Analytics — Pruning old rows.

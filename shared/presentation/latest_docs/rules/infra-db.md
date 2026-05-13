# Infrastructure — Database

## Constants — `constant` package

### `constant.BaseURL()`
Returns the scheme + domain for the apex site: `"http://machine.local"` in dev, `"https://warga.jp"` in production. Use this whenever linking to the apex domain from templates or components — never hardcode `"https://"` for links that must work in dev.

```go
// chrome.templ — correct
<a href={ constant.BaseURL() + "/komunitas" }>Komunitas</a>

// wrong — always https, breaks HTTP dev environment
<a href={ "https://" + constant.DomainName() + "/komunitas" }>Komunitas</a>
```

`constant.DomainName()` is still correct for building canonical URLs (which are always absolute `https://`) and for SEO meta tags — those intentionally use `https://` regardless of environment.

### `constant.KBRITokyoURL`
Hard-coded URL for the Indonesian embassy in Tokyo (`https://www.kemlu.go.id/tokyo`). Used in `presentation/contact/view/contact.templ`. Update here if Kemlu moves the page — do **not** hardcode the URL directly in templates.

```go
<a href={ templ.URL(constant.KBRITokyoURL) } target="_blank" rel="noopener">KBRI Tokyo</a>
```

### Error page pre-rendering — `response.Init()`
`presentation/shared/response/response.go` pre-renders the 404 and 500 error pages for zero-allocation serving. This must happen **after** `constant.SetDomainName()` is called, otherwise nav links render with an empty domain.

`main.go` calls `response.Init()` immediately after the domain/env block:
```go
constant.SetDomainName("machine.local")
// ... env setup ...
response.Init()   // must come after SetDomainName
```

**Do not** move `response.Init()` before `SetDomainName`, and do not revert it to an `init()` function in `response.go` — `init()` runs before `main()` and the domain name is not set yet.

## Database Migrations
- Versioned migrations stored in `schema_migrations` table
- Use `runMigration(db, version, name, func)` to add new migrations
- **Never use `DROP TABLE` in migrations**
- **Idempotency comes from `schema_migrations`**, not from existence guards: `runMigration` checks whether the version has been applied before running the function — so each migration runs exactly once. There is no `columnExists` helper.
- Adding a column: `ALTER TABLE table ADD COLUMN col TYPE` inside a versioned `runMigration` call — the version gate means it will only run once.
- Removing a column: `ALTER TABLE table DROP COLUMN col` (SQLite ≥ 3.35, provided by the bundled `go-sqlite3`)

### Adding a new prefecture-like entry (e.g. `national`)

Adding a new entry to `GetAllPrefectures()` in `domain/prefecture/repository/repository.go` requires **no migration code**. `ExecuteMigration` receives the full list at startup and calls `openAndMigrateDB` for every entry — the new DB file is created and all existing migrations are applied automatically. The export tool (`cmd/migrate`, `/admin/migrasi`) also iterates the same list, so the new DB is included in the SQL dump without any changes.

What you do need to update when adding a new entry:
- `domain/prefecture/model/model.go` — add a `Pref_Xxx = "xxx"` constant
- `domain/prefecture/repository/repository.go` — add the entry to `GetAllPrefectures()`; add to `PrefectureRegion` if it belongs to a geographic region (leave absent for non-geographic entries like `national` — they are safely excluded from region-grouped UI)
- `Caddyfile.prod` — add `xxx.warga.jp` to the host block
- `docs/DEVELOPMENT.md` — add `xxx.machine.local` to both IPv4 and IPv6 `/etc/hosts` blocks

### Schema changes must also update the export tool
Any change to a table schema (new column, renamed column, new table, dropped column) requires
updating **both** of the following files or the generated SQL dump will be wrong:

| File | What to update |
|---|---|
| `migrations/export.go` | Add/remove the column from the `SELECT` query, the `rows.Scan(...)` call, the `INSERT INTO ... (cols)` column list, and the `fmt.Fprintf` value list for the affected table's export function |
| `migrations/export.go` `writePostgresSchema` / `writeMySQLSchema` | Add/remove the column from the `CREATE TABLE` DDL block for that table |

For a **new table**, also add a new `exportXxx(...)` function and call it from `ExportToSQL`
in the appropriate section (shared tables vs. per-prefecture loop).

Special cases to remember:
- Per-prefecture tables omit `id` in `INSERT` (target DB auto-assigns to avoid cross-prefecture conflicts).
- Shared tables (`users`, `sessions`, `admins`, `admin_sessions`) preserve `id` for FK integrity — also add the table name to `writeSequenceResets`.
- Reserved words in MySQL need renaming: the `condition` column in `recycled_goods` was renamed to `item_condition` in the exported schema.

## Database — Nullable Columns

### Always scan nullable TEXT columns via `sql.NullString`
Any column defined as `TEXT` without `NOT NULL` in the schema can contain SQL `NULL`. Scanning `NULL` into a Go `string` with `rows.Scan` returns an error, which causes the row to be silently skipped via `continue` — the query appears to succeed but returns fewer rows than expected.

**Affected columns in `generic_info_post`:** `category`, `submitter_email`, `submitter_whatsapp`, `submitter_line`

Use a dedicated scan helper (see `scanGenericInfoPost` in `domain/prefecture/repository/repository.go`) that scans nullable columns into `sql.NullString` and extracts `.String`:

```go
var email sql.NullString
rows.Scan(..., &email, ...)
post.SubmitterEmail = email.String  // "" when NULL
```

**Rule:** For any new table with nullable TEXT columns, write a `scanXxx` helper and use it in every query function for that table. Never scan a nullable column directly into `string`.

### Symptoms of this bug
- Dashboard `COUNT(*)` shows N rows but the list view shows fewer (or zero)
- No errors in logs — the scan error is caught and `continue`d silently
- Only rows with all non-NULL values appear

### Seed data must use explicit status
The `seedGenericInfoPost` function inserts admin-created placeholder content. It must always set `status = 'approved'` explicitly — do not rely on the column default, which is `'pending'` (intended for user submissions). Seeded rows that are pending will inflate the pending count on the admin dashboard.

```go
// correct
INSERT INTO generic_info_post (title, content, category, status)
VALUES (?, ?, ?, 'approved')

// wrong — inherits DEFAULT 'pending', shows as user submission needing review
INSERT INTO generic_info_post (title, content, category)
VALUES (?, ?, ?)
```

If existing seeded rows are stuck as `'pending'`, fix them with a versioned migration:
```go
UPDATE generic_info_post SET status = 'approved'
WHERE title = '<seed title>' AND status = 'pending'
```

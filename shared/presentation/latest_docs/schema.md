# Database Schema

## Database Structure

Four SQLite files:

- `dbs/app.db` - All content: panduan, komunitas, and reports across all prefectures
- `dbs/admin.db` - Admin users, sessions, featured-content curation, contact messages
- `dbs/user.db` - User accounts & sessions
- `dbs/analytics.db` - Pageviews & search queries

### Key Tables

All four DBs track applied migrations in a `schema_migrations (version, applied_at)` table.

#### app.db

**`generic_info_post`** — panduan/guide posts (`prefecture = 'national'` for nation-wide)

| Column | Type | Notes |
|---|---|---|
| `prefecture` | TEXT NOT NULL | PK part 1 |
| `id` | INTEGER NOT NULL | PK part 2; per-prefecture sequential |
| `title` | TEXT NOT NULL | |
| `content` | TEXT NOT NULL | Markdown source |
| `category` | TEXT | nullable; e.g. `tips`, `pengalaman` |
| `submitter_email` | TEXT | NULLed on approval (privacy rule) |
| `submitter_whatsapp` | TEXT | NULLed on approval |
| `submitter_line` | TEXT | NULLed on approval |
| `status` | TEXT NOT NULL DEFAULT `'pending'` | `pending` / `approved` / `rejected` |
| `created_at` | DATETIME | default `CURRENT_TIMESTAMP` |
| `updated_at` | DATETIME | default `CURRENT_TIMESTAMP` |
| `admin_edited_at` | DATETIME | nullable; set when admin edits content |

Indexes: `(prefecture, status, created_at DESC)`, `(status, created_at DESC)`

---

**`community_info`** — community listings

| Column | Type | Notes |
|---|---|---|
| `prefecture` | TEXT NOT NULL | PK part 1 |
| `id` | INTEGER NOT NULL | PK part 2; per-prefecture sequential |
| `name` | TEXT NOT NULL | |
| `description` | TEXT | nullable; rendered with `RenderMarkdownHardWrap` (single newlines → `<br>`); `Platform: URL` lines (e.g. `Instagram: https://...`) are parsed by `lib/social.ParseLinks` and rendered separately as social link pills — they are not part of the prose |
| `location` | TEXT | nullable |
| `image_urls` | TEXT | nullable; comma-separated URLs |
| `submitter_email` | TEXT | NULLed on approval |
| `submitter_whatsapp` | TEXT | NULLed on approval |
| `submitter_line` | TEXT | NULLed on approval |
| `category` | TEXT NOT NULL DEFAULT `'hobi'` | one of 10 canonical tipe slugs (see `domain/community/model/category.go`) |
| `status` | TEXT NOT NULL DEFAULT `'pending'` | `pending` / `approved` / `rejected` |
| `created_at` | DATETIME | default `CURRENT_TIMESTAMP` |
| `updated_at` | DATETIME | default `CURRENT_TIMESTAMP` |
| `admin_edited_at` | DATETIME | nullable |

Indexes: `(prefecture, status, created_at DESC)`, `(status, created_at DESC)`, `(prefecture, category)`

---

**`reports`** — content complaints

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement; global |
| `prefecture` | TEXT NOT NULL | matches content's prefecture |
| `content_type` | TEXT NOT NULL | `panduan` or `komunitas` |
| `content_id` | INTEGER NOT NULL | pairs with `prefecture` to identify content |
| `reason` | TEXT NOT NULL | |
| `kontak` | TEXT NOT NULL DEFAULT `''` | optional reporter email; added in migration 4, not in the initial `reports` table |
| `status` | TEXT NOT NULL DEFAULT `'pending'` | `pending` / `resolved` |
| `created_at` | DATETIME | default `CURRENT_TIMESTAMP` |

Indexes: `(prefecture, status, created_at DESC)`, `(status, created_at DESC)`, `(content_type, prefecture, content_id)`

---

#### admin.db

**`admins`**

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `username` | TEXT NOT NULL UNIQUE | |
| `email` | TEXT NOT NULL UNIQUE | |
| `password` | TEXT NOT NULL | bcrypt hash |
| `status` | TEXT NOT NULL | default `active` |
| `created_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | |
| `updated_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | |

**`admin_sessions`**

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `admin_id` | INTEGER NOT NULL | FK → `admins.id` |
| `session_token` | TEXT NOT NULL UNIQUE | |
| `expires_at` | DATETIME NOT NULL | |
| `created_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | |

**`featured_panduan`** — homepage curation for panduan

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `prefecture` | TEXT NOT NULL | references `generic_info_post.prefecture` |
| `post_id` | INTEGER NOT NULL | references `generic_info_post.id` |
| `title` | TEXT NOT NULL | denormalised for display |
| `url` | TEXT NOT NULL | canonical URL |
| `display_order` | INTEGER NOT NULL | default 0; lower = higher |
| `created_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | |

Unique constraint: `(prefecture, post_id)`

**`featured_komunitas`** — homepage curation for komunitas

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `prefecture` | TEXT NOT NULL | references `community_info.prefecture` |
| `community_id` | INTEGER NOT NULL | references `community_info.id` |
| `title` | TEXT NOT NULL | denormalised for display |
| `url` | TEXT NOT NULL | canonical URL |
| `display_order` | INTEGER NOT NULL | default 0 |
| `created_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | |

Unique constraint: `(prefecture, community_id)`

**`contact_messages`**

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `sender_email` | TEXT NOT NULL | |
| `message` | TEXT NOT NULL | |
| `status` | TEXT NOT NULL DEFAULT `'pending'` | `pending` / `replied` / `archived` |
| `created_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | |
| `replied_at` | DATETIME | nullable |
| `archived_at` | DATETIME | nullable |

Index: `(status, created_at DESC)`

---

#### user.db

**`users`**

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `username` | TEXT NOT NULL UNIQUE | |
| `email` | TEXT NOT NULL UNIQUE | |
| `password` | TEXT NOT NULL | bcrypt hash |
| `prefecture` | TEXT | nullable |
| `status` | TEXT NOT NULL | default `active` |
| `created_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | |
| `updated_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | |

**`sessions`**

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `user_id` | INTEGER NOT NULL | FK → `users.id` |
| `session_token` | TEXT NOT NULL UNIQUE | |
| `expires_at` | DATETIME NOT NULL | |
| `created_at` | DATETIME DEFAULT CURRENT_TIMESTAMP | |

---

#### analytics.db

**`pageviews`**

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `path` | TEXT NOT NULL | request path |
| `subdomain` | TEXT NOT NULL DEFAULT `''` | prefecture subdomain or `''` for apex |
| `country` | TEXT NOT NULL DEFAULT `'XX'` | 2-letter ISO from `CF-IPCountry`; `XX` if unknown |
| `visitor_hash` | TEXT NOT NULL | SHA-256(`daily_salt\|IP\|UA`) truncated to 16 bytes; rotates at UTC midnight |
| `seen_at` | DATETIME NOT NULL | |

Covering index: `(seen_at, subdomain, path, visitor_hash)`

**`search_queries`**

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `query` | TEXT NOT NULL | sanitised search term |
| `subdomain` | TEXT NOT NULL DEFAULT `''` | |
| `country` | TEXT NOT NULL DEFAULT `'XX'` | |
| `visitor_hash` | TEXT NOT NULL | |
| `seen_at` | DATETIME NOT NULL | |

Covering index: `(seen_at, query, subdomain, visitor_hash)`

**`captcha_verifications`**

| Column | Type | Notes |
|---|---|---|
| `id` | INTEGER PK | autoincrement |
| `form` | TEXT NOT NULL | `contact` \| `report` \| `panduan` \| `community` |
| `outcome` | TEXT NOT NULL | `pass` \| `fail` \| `expired` |
| `question` | TEXT NOT NULL DEFAULT `''` | question text if determinable; empty for expired/malformed tokens |
| `seen_at` | DATETIME NOT NULL | |

Covering index: `(seen_at, outcome, form)`

---

## Content Privacy Rule

The `submitter_email`, `submitter_whatsapp`, and `submitter_line` fields are **submitter-only** with a privacy-by-deletion lifecycle: stored as plaintext while pending/rejected (visible to admin for moderation), then **NULLed immediately on approval** — never retained for publicly-visible content. See `docs/rules/forms.md` § Contact fields for the full rule and admin UX note.

> **Schema change rule:** Any addition, removal, or rename of a column or table must also be
> reflected in `migrations/export.go` (the SQL dump generator). See `docs/rules/infra-db.md` § Database Migrations
> for the exact checklist.

## Domain Models (in-code, not DB tables)

**`Prefecture`** (`domain/prefecture/model/model.go`) — `{Value, Emoji, Label}` where `Label` is plain text ("Tokyo") and `Emoji` is a separate field ("🗼"). Mirrors the shape of `Category{Slug, Label, Emoji}` in `domain/community/model/category.go`. The static slice lives in `domain/prefecture/repository/repository.go` (`GetAllPrefectures()`). `Prefecture.Display()` returns `Emoji + " " + Label` for contexts that need the combined string (admin email subjects, admin panel headers).

**`Category`** (`domain/community/model/category.go`) — `{Slug, Label, Emoji}` for the 10 komunitas tipe slugs. `CategoryBySlug(slug)` for lookup.

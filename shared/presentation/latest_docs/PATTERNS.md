# Patterns & Examples

## Code Generation

```bash
# Generate all templ files
templ generate

# Generate specific templ file
templ generate -f path/to/file.templ
```

If `templ generate` reports `updates=0` but changes were made, the cache may be stale. Delete the affected `_templ.go` first:

```bash
rm presentation/some/view/file_templ.go
templ generate
```

---

## Add New Feature

Eight steps. Each step includes the pattern and a concrete code example.

### 1. Domain model — `domain/<feature>/model/model.go`

```go
type Feature struct {
    ID   int
    Name string
}
```

### 2. Repository — `domain/<feature>/repository/repository.go`

```go
func CreateFeature(db *sql.DB, feature Feature) error {
    stmt, err := db.Prepare("INSERT INTO features(name) VALUES(?)")
    if err != nil { return err }
    defer stmt.Close()
    _, err = stmt.Exec(feature.Name)
    return err
}

func GetFeatures(db *sql.DB) ([]Feature, error) {
    rows, err := db.Query("SELECT id, name FROM features")
    if err != nil { return nil, err }
    defer rows.Close()

    var features []Feature
    for rows.Next() {
        var f Feature
        if err := rows.Scan(&f.ID, &f.Name); err != nil { return nil, err }
        features = append(features, f)
    }
    return features, nil
}
```

### 3. Handler — `presentation/<feature>/handler/handler.go`

```go
func HandleFeature(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

### 4. Viewmodel — `presentation/<feature>/viewmodel/viewmodel.go`

```go
type FeatureViewModel struct {
    Features []Feature
}
```

### 5. Template — `presentation/<feature>/view/<feature>.templ`

```go
import (
    "website/presentation/<feature>/viewmodel"
    "website/presentation/shared/layout"
    "fmt"
)
```

### 6. Generate templ

```bash
templ generate
```

### 7. Update router — `router/router.go`

Wrap the feature's routes in `r.Route("/<feature>", func(r chi.Router) { ... })` (3+ routes always group). Register `/tambah` before `/{id}`. See `docs/rules/features.md` § Router conventions.

### 8. Update `allowedPaths` — `main.go`

Add the path if it is apex-accessible.

---

## Preview / Edit-Lagi Viewmodel (Markdown Forms)

The viewmodel carries a `PreviewHTML` field. When non-empty the template switches to preview mode; when empty it renders the editor.

```go
type SubmitViewModel struct {
    CSRFToken       string
    CaptchaQuestion string
    CaptchaAnswer   string
    PreviewHTML     string   // non-empty = preview mode
    ContentText     string
    CaptchaUser     string
    // ... other preserved form fields ...
}

// Preview: reuse CSRF + captcha from the request, render markdown HTML.
func NewPreviewViewModel(r *http.Request, previewHTML string) SubmitViewModel {
    return SubmitViewModel{
        CSRFToken:       r.FormValue("csrf_token"),
        CaptchaQuestion: r.FormValue("captcha_question"),
        CaptchaAnswer:   r.FormValue("captcha_answer"),
        PreviewHTML:     previewHTML,
        ContentText:     r.FormValue("content"),
        CaptchaUser:     r.FormValue("captcha_user"),
    }
}

// Edit-Lagi: same preserved fields, PreviewHTML empty → textarea editor.
func NewEditViewModel(r *http.Request) SubmitViewModel {
    return SubmitViewModel{
        CSRFToken:       r.FormValue("csrf_token"),
        CaptchaQuestion: r.FormValue("captcha_question"),
        CaptchaAnswer:   r.FormValue("captcha_answer"),
        ContentText:     r.FormValue("content"),
        CaptchaUser:     r.FormValue("captcha_user"),
    }
}
```

---

## Admin Detail Viewmodel

```go
// Panduan: renders markdown for the detail page
type PanduanDetailViewModel struct {
    HeadTitle               string
    Title                   string
    Panduan                 *model.GenericInfoPost
    RenderedContent         string   // markdown.RenderMarkdown(panduan.Content)
    SelectedPrefecture      string
    SelectedPrefectureLabel string
}

// Komunitas: plain text only, no markdown rendering
type CommunityDetailViewModel struct {
    HeadTitle               string
    Title                   string
    Community               *model.CommunityInfo
    SelectedPrefecture      string
    SelectedPrefectureLabel string
}
```

---

## Database Migration

Migrations live in `migrations/migration.go`. Each DB has its own version sequence tracked in `schema_migrations`. Add a new migration as a standalone function, then wire it into `ExecuteMigration()` with the next version number for that DB.

```go
// 1. Write a migration function
func migrateAddNewColumn(db *sql.DB) error {
    _, err := db.Exec("ALTER TABLE table_name ADD COLUMN new_column TEXT")
    return err
}

// 2. Wire it in ExecuteMigration() after the last existing runMigration call for that DB
runMigration(appDb, 3, "Add new column", migrateAddNewColumn)
```

`runMigration` is idempotent — it skips any version already recorded in `schema_migrations`, so each migration runs exactly once. No `columnExists` guard is needed.

Current migration versions:

| DB | Version | Description |
|---|---|---|
| `app.db` | 1 | Unified schema (`generic_info_post`, `community_info`, `reports` + indexes) |
| `app.db` | 2 | `idx_reports_content` on `reports (content_type, prefecture, content_id)` |
| `app.db` | 3 | `category` column + `idx_ci_pref_category` on `community_info` |
| `app.db` | 4 | `kontak` column on `reports` |
| `app.db` | 5 | Rename `category = 'ibadah'` → `'keagamaan'` in `community_info` |
| `admin.db` | 1–5 | `admins`, `admin_sessions`, `featured_panduan`, `contact_messages`, `featured_komunitas` |
| `analytics.db` | 1 | `pageviews` table |
| `analytics.db` | 2 | `search_queries` table |
| `analytics.db` | 3 | Covering indexes (`idx_pv_covering`, `idx_sq_covering`) |

Also mirror any new table or index in `migrations/export.go` (`writePostgresSchema` and `writeMySQLSchema`).

---

## Repository Unit Test

Use a real in-memory SQLite DB — not sqlmock. Create the table schema in `newTestDB`, seed rows directly, then call the repository function and assert on the result.

```go
// domain/<feature>/repository/repository_test.go
package repository

import (
    "database/sql"
    "testing"

    _ "github.com/mattn/go-sqlite3"
)

func newTestDB(t *testing.T) *sql.DB {
    t.Helper()
    db, err := sql.Open("sqlite3", ":memory:")
    if err != nil {
        t.Fatalf("open: %v", err)
    }
    _, err = db.Exec(`CREATE TABLE feature (
        prefecture TEXT NOT NULL,
        id         INTEGER NOT NULL,
        name       TEXT NOT NULL,
        status     TEXT NOT NULL DEFAULT 'pending',
        PRIMARY KEY (prefecture, id)
    )`)
    if err != nil {
        t.Fatalf("create table: %v", err)
    }
    t.Cleanup(func() { db.Close() })
    return db
}

func TestGetFeatureByID(t *testing.T) {
    db := newTestDB(t)
    db.Exec(`INSERT INTO feature (prefecture, id, name, status) VALUES ('tokyo', 1, 'Test', 'approved')`)

    got, err := GetFeatureByID(db, "tokyo", 1)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if got == nil {
        t.Fatal("expected result, got nil")
    }

    missing, err := GetFeatureByID(db, "tokyo", 999)
    if err != nil {
        t.Fatalf("unexpected error: %v", err)
    }
    if missing != nil {
        t.Errorf("expected nil for missing row, got %+v", missing)
    }
}
```

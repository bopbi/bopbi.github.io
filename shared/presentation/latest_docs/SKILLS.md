# SKILL.md - Execute Specific Capabilities

## Code Generation

### Generate Templ Files
```bash
templ generate
```

### Generate Specific Templ File
```bash
templ generate -f path/to/file.templ
```

## Add New Feature

Follow these steps to add a new feature:

1. **Create domain model** - `domain/<feature>/model/model.go`
   ```go
   type Feature struct {
       ID   int
       Name string
   }
   ```

2. **Create repository** - `domain/<feature>/repository/repository.go`
   ```go
   func CreateFeature(db *sql.DB, feature Feature) error
   func GetFeatures(db *sql.DB) ([]Feature, error)
   ```

3. **Create handler** - `presentation/<feature>/handler/handler.go`
   ```go
   func HandleFeature(w http.ResponseWriter, r *http.Request)
   ```

4. **Create viewmodel** - `presentation/<feature>/viewmodel/viewmodel.go`
   ```go
   type FeatureViewModel struct {
       Features []Feature
   }
   ```

5. **Create template** - `presentation/<feature>/view/<feature>.templ`
   ```go
   import "website/presentation/<feature>/viewmodel"
   ```

6. **Generate templ** - Run `templ generate`

7. **Update router** - Add routes in `router/router.go`

8. **Update allowed paths** - Add to `main.go` if needed

## Database Migration

To add a new database column:

1. Check if column exists:
   ```go
   if !columnExists(db, "table_name", "new_column") {
       // Add column
   }
   ```

2. Add migration:
   ```go
   runMigration(db, 2, "add_new_column", func(db *sql.DB) error {
       _, err := db.Exec("ALTER TABLE table_name ADD COLUMN new_column TEXT")
       return err
   })
   ```

3. Run migration in `main.go`:
   ```go
   migrateDB(db)
   ```

## Submit Content (User Workflow)

1. User visits submission page (`/komunitas/tambah` or `/panduan/tambah`)
2. Read motivational message with prefecture context
3. Fill form with:
   - Content details
   - Contact info (email required, WhatsApp & Line optional)
   - Knowledge captcha solution
4. *(Panduan only)* Optionally click "Pratinjau" to preview rendered markdown
   - Server renders content via `lib/markdown.RenderMarkdown` and re-renders the page in preview mode
   - Click "Edit Lagi" to return to the editor (all fields preserved)
   - Click "Ajukan" to submit directly from preview
5. Submit - saved with `pending` status

## Approve/Reject Content (Admin Workflow)

1. Admin reviews pending content in `/admin/komunitas` or `/admin/panduan`
2. Click "Lihat" to view full detail (metadata, raw content, rendered markdown for panduan)
3. Click "Setujui" to approve or "Tolak" to reject
4. Content status updates: `pending` → `approved` or `rejected`
5. Approved content appears publicly

## Database Operations

### Create Record
```go
func CreateFeature(db *sql.DB, feature Feature) error {
    stmt, err := db.Prepare("INSERT INTO features(name) VALUES(?)")
    if err != nil { return err }
    defer stmt.Close()
    _, err = stmt.Exec(feature.Name)
    return err
}
```

### Get Records
```go
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

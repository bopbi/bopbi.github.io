# Code Examples

## Model

```go
// domain/<feature>/model/model.go
type Feature struct {
    ID   int
    Name string
}
```

## Repository

```go
// domain/<feature>/repository/repository.go

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

## Handler

```go
// presentation/<feature>/handler/handler.go
func HandleFeature(w http.ResponseWriter, r *http.Request) {
    // ...
}
```

## Viewmodel

```go
// presentation/<feature>/viewmodel/viewmodel.go
type FeatureViewModel struct {
    Features []Feature
}
```

## Preview Viewmodel (Markdown Forms)

```go
// presentation/<feature>/viewmodel/viewmodel.go

// The viewmodel carries a PreviewHTML field. When non-empty, the template
// switches to preview mode; when empty, it renders the editor.
type SubmitViewModel struct {
    CSRFToken       string
    CaptchaQuestion string
    CaptchaAnswer   string
    PreviewHTML     string   // non-empty = preview mode
    // ... preserved form fields ...
    ContentText     string
    CaptchaUser     string
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
        // ... preserve all other form values ...
    }
}

// Edit-Lagi: same preserved fields, but PreviewHTML empty → textarea editor.
func NewEditViewModel(r *http.Request) SubmitViewModel {
    return SubmitViewModel{
        CSRFToken:       r.FormValue("csrf_token"),
        CaptchaQuestion: r.FormValue("captcha_question"),
        CaptchaAnswer:   r.FormValue("captcha_answer"),
        ContentText:     r.FormValue("content"),
        CaptchaUser:     r.FormValue("captcha_user"),
        // ... preserve all other form values ...
    }
}
```

## Admin Detail Viewmodel

```go
// presentation/admin/<feature>/viewmodel/viewmodel.go

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

## Templ Template

```go
// presentation/<feature>/view/<feature>.templ
import (
    "website/presentation/<feature>/viewmodel"
    "website/presentation/shared/layout"
    "fmt"
)
```

## Repository Unit Test

```go
// domain/<feature>/repository/repository_test.go
package repository

import (
    "testing"
    "github.com/DATA-DOG/go-sqlmock"
)

func setupTestDB(t *testing.T) (*sql.DB, sqlmock.Sqlmock, error) {
    db, mock, err := sqlmock.New()
    if err != nil {
        return nil, nil, err
    }
    t.Cleanup(func() { db.Close() })
    return db, mock, nil
}

// Table-driven test example
func TestGetFeatureByID(t *testing.T) {
    db, mock, err := setupTestDB(t)
    if err != nil {
        t.Fatalf("Setup failed: %v", err)
    }

    tests := []struct {
        name      string
        id        int
        expectRow bool
    }{
        {"Existing", 1, true},
        {"NonExistent", 999, false},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            rows := sqlmock.NewRows([]string{"id", "name"})
            if tt.expectRow {
                rows.AddRow(tt.id, "Feature Name")
            }

            mock.ExpectQuery(`SELECT.*FROM features WHERE id = \?`).
                WithArgs(tt.id).
                WillReturnRows(rows)

            result, err := GetFeatureByID(db, tt.id)
            if err != nil {
                t.Fatalf("Unexpected error: %v", err)
            }
            if tt.expectRow && result == nil {
                t.Error("Expected result, got nil")
            }
            if !tt.expectRow && result != nil {
                t.Errorf("Expected nil, got %+v", result)
            }
        })
    }
}
```

## Database Migration

```go
runMigration(db, 2, "add_new_column", func(db *sql.DB) error {
    if !columnExists(db, "table_name", "new_column") {
        _, err := db.Exec("ALTER TABLE table_name ADD COLUMN new_column TEXT")
        return err
    }
    return nil
})
```

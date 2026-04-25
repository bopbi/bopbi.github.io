# Rules & Conventions

## Auth Patterns
- CSRF token required for all POST forms
- Session cookie: `admin_session_token` for admins
- Cookie settings: `HttpOnly: true`, `Secure: true`, `SameSite: http.SameSiteLaxMode`

## Adding New Features
1. Create `domain/<feature>/model/model.go` - Define struct
2. Create `domain/<feature>/repository/repository.go` - Database operations
3. Create `presentation/<feature>/handler/handler.go` - HTTP handlers
4. Create `presentation/<feature>/viewmodel/viewmodel.go` - View models
5. Create `presentation/<feature>/view/<feature>.templ` - Template
6. Run `templ generate`
7. Update `router/router.go` - Add routes
8. Add allowed paths if needed in `main.go`

## Database Migrations
- Versioned migrations stored in `schema_migrations` table
- Use `runMigration(db, version, name, func)` to add new migrations
- Use `columnExists(db, table, column)` helper for idempotent column additions
- **Never use `DROP TABLE` in migrations**
- New columns must use `ALTER TABLE ADD COLUMN`

## Unit Testing
- Test files go in the same package as the code under test (e.g. `package repository`)
- File location: `domain/<feature>/repository/repository_test.go`
- Use `github.com/DATA-DOG/go-sqlmock` to mock `*sql.DB` — do not spin up a real SQLite DB in tests
- Use a `setupTestDB(t *testing.T)` helper that calls `sqlmock.New()` and registers `t.Cleanup` to close the DB
- Use table-driven tests with `t.Run` for functions that have multiple input/output variants
- Functions that don't touch the DB (e.g. pure slice helpers) can be tested directly without mocks

## Templ Patterns
- Import viewmodel: `import "website/presentation/<feature>/viewmodel"`
- Import layout: `import "website/presentation/shared/layout"`
- Use `fmt.Sprintf()` for dynamic URLs
- Run `templ generate` after creating/modifying `.templ` files
- **Minimize JavaScript usage** - Prefer server-side rendering; use JavaScript only when absolutely necessary

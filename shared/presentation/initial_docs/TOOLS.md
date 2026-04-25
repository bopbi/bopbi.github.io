# Tools & Commands

## Running the App

```bash
# Development
APP_ENV=development go run main.go

# Production
APP_ENV=production go run main.go
```

## Code Generation

```bash
# Generate all templ files
templ generate

# Generate specific templ file
templ generate -f path/to/file.templ
```

## Testing

```bash
# Run all tests
go test ./...

# Run tests for a specific package
go test ./domain/prefecture/repository/...

# Run with verbose output
go test -v ./...

# Run a specific test
go test -v -run TestGetAllPrefectures ./domain/prefecture/repository/...
```

## Helper Functions (main.go)

- `runMigration(db, version, name, func)` - Apply a versioned migration
- `columnExists(db, table, column)` - Check if column exists before adding
- `migrateDB(db)` - Run all pending migrations

# Tools & Commands

## Running the App

```bash
# Development
APP_ENV=development go run main.go

# Production
APP_ENV=production go run main.go
```

## Production Build & Deploy

```bash
# Build a Linux/AMD64 production binary (runs templ generate + minify + go build)
./scripts/build-prod.sh

# Build + rsync to VPS + restart service
./scripts/deploy.sh <user> <vps-host>
```

**One-time setup (macOS):**
```bash
brew install FiloSottile/musl-cross/musl-cross          # cross-compiler
go install github.com/a-h/templ/cmd/templ@latest         # templ code generator
go install github.com/tdewolff/minify/v2/cmd/minify@latest  # CSS/JS minifier
```

`build-prod.sh` minifies `base.css`, `public.css`, `admin.css`, and `admin.js` in-place before embedding them into the binary, then restores the originals via a `trap EXIT`. Source files in `static/` are always clean after the script finishes (success or failure). Dev (`go run main.go`) is unaffected — it serves the readable source files directly.

## Code Generation

```bash
# Generate all templ files
templ generate

# Generate specific templ file
templ generate -f path/to/file.templ
```

**Important:** Never produce a named binary with `go build -o ...`. Use `go build ./...` (no `-o`) only to check for Go compilation errors. The server is run separately via `go run main.go`.

If `templ generate` reports `updates=0` but you know changes were made, the generator's cache may be stale. Force regeneration by deleting the affected `_templ.go` file(s) first:

```bash
rm presentation/some/view/file_templ.go
templ generate
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

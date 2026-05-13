# Production Build & Deploy

For initial VPS provisioning (DNS setup, Caddy install, systemd unit, first DB upload), see `docs/VPS_SETUP.md`.

---

## Background: why cross-compilation needs a special setup

This project uses `github.com/mattn/go-sqlite3` which is a **CGo package** — it compiles C
code alongside Go. Cross-compiling CGo from ARM macOS to x86_64 Linux requires a C
cross-compiler (`x86_64-linux-musl-gcc`). The build script handles this automatically.

## Step 1 — Install prerequisites (once, on your Mac)

```bash
# C cross-compiler for linux/amd64
brew install FiloSottile/musl-cross/musl-cross

# templ code generator
go install github.com/a-h/templ/cmd/templ@latest

# CSS/JS minifier
go install github.com/tdewolff/minify/v2/cmd/minify@latest
```

## Step 2 — Build the production binary

```bash
./scripts/build-prod.sh
```

This script:
1. Checks that `templ`, `minify`, and `x86_64-linux-musl-gcc` are available
2. Runs `templ generate` to regenerate all `*_templ.go` files
3. Minifies CSS/JS assets in-place (originals restored on exit via `trap EXIT`)
4. Cross-compiles to `scripts/build/wargajp` (Linux AMD64)

Output: `scripts/build/wargajp` — a single self-contained binary (~30 MB) with Pico.css and all
templates embedded. No other files need to be deployed alongside it.

> **Never** produce a named binary with `go build -o ...`. Use `go build ./...` (no `-o`) only to check for compilation errors.

## CSS cache busting and versioning

The build script injects three vars into the binary at compile time:

```bash
-X website/version.Build=${GIT_COMMIT}
-X website/version.Version=${GIT_VERSION}
-X website/version.BuildTime=${BUILD_TIME}
```

Every template references CSS files with a version query string:

```html
<link rel="stylesheet" href="/static/base.css?v=b6192eb"/>
```

This means each deploy serves CSS at a new URL, so Cloudflare automatically caches the
new version without needing a manual cache purge. Local dev uses `?v=dev`.

**Rule:** always use `version.Build` in CSS `href` attributes — never hardcode paths without the version suffix. Public templates load `base.css` + `public.css`; admin templates load `base.css` + `admin.css`.

**Tagging a release:** run `git tag v0.x.y` before `build-prod.sh`. `git describe --tags` picks it up; without a tag it falls back to the short SHA. The version surfaces in the startup log, `/ping` JSON, admin header, and public footer.

## Step 3 — One-time DB consolidation (first deploy only)

If migrating from an older version that used per-prefecture `.db` files:

```bash
./scripts/consolidate-dbs.sh <user> <vps-host>
```

The script stops the service, runs the migration as `wargajp` user, and starts again. The migration is idempotent — re-running is safe. Fresh deployments skip this step entirely.

## Step 4 — Deploy

```bash
./scripts/deploy.sh <user> <vps-host>
# example: ./scripts/deploy.sh ubuntu 1.2.3.4
```

The script builds the binary (via `build-prod.sh`), uploads it via rsync, moves it into place atomically, and restarts the service. The maintenance page (served by Caddy) shows automatically during the ~2-second restart window.

## Deploying Caddyfile changes

`deploy.sh` only touches the Go binary. When `Caddyfile.prod` changes:

```bash
# 1. Upload new config
scp Caddyfile.prod ubuntu@<VPS_IP>:~/Caddyfile

# 2. SSH in and deploy
ssh ubuntu@<VPS_IP>
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak   # backup
sudo mv ~/Caddyfile /etc/caddy/Caddyfile
sudo systemctl reload caddy                              # validates before applying
sudo journalctl -u caddy -n 20 --no-pager               # confirm no errors
```

`reload` (not `restart`) applies the new config gracefully. If invalid, caddy keeps running with the old config. Restore from backup if needed: `sudo cp /etc/caddy/Caddyfile.bak /etc/caddy/Caddyfile && sudo systemctl reload caddy`

> **Note:** `caddy validate` does not work standalone because `CLOUDFLARE_API_TOKEN` is only available via `--envfile`. Rely on `systemctl reload caddy` for validation.

## Verify after any deploy

```bash
curl -s https://warga.jp/ping | jq
openssl s_client -connect warga.jp:443 -servername warga.jp </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates
```

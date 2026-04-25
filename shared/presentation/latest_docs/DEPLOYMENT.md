# Deployment Guide: Warga.jp on VPS with Caddy

## Overview

```
Internet
  → DNS: warga.jp + per-prefecture A records → VPS IP   (DNS provider with API access, DNS-only)
  → Caddy :443 on VPS                                    (TLS via DNS-01 challenge)
  → Go app :8080 on VPS                                  (subdomain extracted from Host header)
  → /opt/wargajp/dbs/*.db                                (50 SQLite files, persisted on VPS)
```

The Go app listens on `:8080`. Caddy terminates TLS and forwards requests. The subdomain
in the `Host` header (`tokyo.warga.jp` → `tokyo`) tells the app which prefecture DB to use.

---

## 0. VPS Spec

### Recommended minimum

| Resource | Minimum | Comfortable |
|---|---|---|
| vCPU | 1 | 1–2 |
| RAM | 1 GB | **2 GB** |
| Storage | 20 GB SSD | 40 GB NVMe |
| Transfer | 500 GB/mo | 1 TB/mo |

**2 GB RAM is recommended.** The app opens 50 SQLite connections (47 prefecture DBs + `user.db` + `admin.db` + `analytics.db`), each consuming ~5–10 MB in WAL mode. Combined with the Go binary (~20 MB), Caddy (~30 MB), and OS overhead (~150–200 MB), the working set sits around 600 MB–1 GB. 1 GB is technically sufficient but leaves little headroom.

**NVMe SSD preferred.** SQLite writes are I/O bound. NVMe reduces write latency noticeably compared to SATA SSD, which matters for form submissions.

### Operating system

**Ubuntu 22.04 LTS** — recommended. Supported until April 2027, largest community, most documentation. All commands in this guide (`systemctl`, `useradd`, `apt`, etc.) are written for Ubuntu/Debian.

**Debian 12 (Bookworm)** is also a good choice — more minimal, same `apt` commands, identical setup steps.

Avoid CentOS (end of life), Rocky Linux/AlmaLinux (`dnf`/`rpm` differ from this guide), and FreeBSD (no `systemd`).

### Datacenter location

Choose a datacenter in or near Japan. The target audience (Indonesian community in Japan) will see ~10–20 ms latency from a Tokyo-region server vs. ~200 ms from Europe or the US.

### When to scale up

- **RAM**: upgrade to 4 GB if sustained memory usage exceeds 70%
- **CPU**: rarely the bottleneck — the search feature fires goroutines per query but Go schedules them efficiently and results are cached
- **Storage**: grows only with DB content (text-heavy, small rows); 20 GB is sufficient for years of organic growth

---

## 1. VPS Initial Setup

Run these steps once after provisioning the VPS.

### Update the system

```bash
apt update && apt upgrade -y
```

### Install required packages

```bash
apt install -y curl git
```

`curl` is used for health checks, `git` for optional source checkout.

### Create the Caddy system user

The Caddy systemd service runs as `caddy`. Create the user and its home directory before deploying the service:

```bash
sudo useradd -r -s /bin/false -m -d /home/caddy caddy
sudo mkdir -p /etc/caddy
```

The `-m -d /home/caddy` flags create the home directory. Caddy uses it to store TLS certificates and autosave config — without it, certificate issuance will fail.

---

## 2. DNS Setup

TLS via DNS-01 ACME challenge requires API access to your DNS provider. Your DNS provider
must have a Caddy plugin available — check the [caddy-dns](https://github.com/caddy-dns)
organisation for supported providers.

Add explicit A records for the apex domain and each prefecture subdomain pointing to your VPS IP:

```
A   warga.jp        <VPS IP>
A   aichi           <VPS IP>
A   akita           <VPS IP>
A   aomori          <VPS IP>
A   chiba           <VPS IP>
A   ehime           <VPS IP>
A   fukui           <VPS IP>
A   fukuoka         <VPS IP>
A   fukushima       <VPS IP>
A   gifu            <VPS IP>
A   gunma           <VPS IP>
A   hiroshima       <VPS IP>
A   hokkaido        <VPS IP>
A   hyogo           <VPS IP>
A   ibaraki         <VPS IP>
A   ishikawa        <VPS IP>
A   iwate           <VPS IP>
A   kagawa          <VPS IP>
A   kagoshima       <VPS IP>
A   kanagawa        <VPS IP>
A   kochi           <VPS IP>
A   kumamoto        <VPS IP>
A   kyoto           <VPS IP>
A   mie             <VPS IP>
A   miyagi          <VPS IP>
A   miyazaki        <VPS IP>
A   nagano          <VPS IP>
A   nagasaki        <VPS IP>
A   nara            <VPS IP>
A   niigata         <VPS IP>
A   oita            <VPS IP>
A   okayama         <VPS IP>
A   okinawa         <VPS IP>
A   osaka           <VPS IP>
A   saga            <VPS IP>
A   saitama         <VPS IP>
A   shiga           <VPS IP>
A   shimane         <VPS IP>
A   shizuoka        <VPS IP>
A   tochigi         <VPS IP>
A   tokushima       <VPS IP>
A   tokyo           <VPS IP>
A   tottori         <VPS IP>
A   toyama          <VPS IP>
A   wakayama        <VPS IP>
A   yamagata        <VPS IP>
A   yamaguchi       <VPS IP>
A   yamanashi       <VPS IP>
```

> **Why explicit records instead of `*.warga.jp`?**
> A wildcard A record is convenient but routes *any* subdomain someone guesses or
> brute-forces to your VPS — including typos, scanners, and subdomains you never
> intended to expose. Explicit records give you full control over what resolves.

> **Note:** Set all records to **DNS only** (gray cloud in Cloudflare) since Caddy
> handles TLS directly via the DNS-01 challenge.

Then create an API token scoped to DNS editing for `warga.jp` only — you'll need it
when configuring Caddy.

### Proxy mode

**Recommended: Proxied (orange cloud)**

Set both A records to **Proxied**. All traffic routes through Cloudflare's edge network —
you get CDN caching, DDoS protection, and your VPS IP is hidden from the public.

Before enabling, verify TCP 443 connectivity from your VPS:

```bash
curl -v https://cloudflare.com
```

If that succeeds, flip both records to orange cloud, then run the UFW firewall script
to restrict port 80/443 to Cloudflare IPs only (see Section 2a below).

> **Note on DNS-01 and proxied mode:** the ACME DNS-01 challenge used by Caddy for
> TLS works via the Cloudflare API — it does not require direct inbound
> connections — so it works correctly with the proxy on.

**Fallback: DNS-only (grey cloud)**

If you run into issues with proxied mode (site unreachable, TLS errors, connection
timeouts), switch both A records back to **DNS only** (grey cloud) as a quick fallback.
Traffic will go directly to your VPS IP, bypassing Cloudflare entirely.

Steps to fall back:
1. In Cloudflare dashboard, click the orange cloud icon on both A records to toggle
   them to grey (DNS only)
2. Wait for DNS to propagate (~1–2 minutes with Cloudflare)
3. Disable UFW restrictions on the VPS so direct traffic is accepted:
   ```bash
   ufw disable
   ```
4. Verify the site is reachable again:
   ```bash
   curl -I https://warga.jp/ping
   ```

Once the site is stable, investigate the root cause before re-enabling the proxy.
Common issues: SSL mode mismatch in Cloudflare (set to **Full (strict)**), Cloudflare
blocking a specific port, or the VPS firewall not updated with new Cloudflare IP ranges.

---

## 2a. Firewall Setup (proxied mode only)

Skip this section if you are using DNS-only mode.

When Cloudflare proxy is on, lock down the VPS so that ports 80 and 443 only accept
connections from Cloudflare's edge IP ranges. This prevents anyone from bypassing
Cloudflare and hitting your origin directly.

Upload and run the script from your local machine:

```bash
rsync -avz scripts/setup-cloudflare-ufw.sh user@your-vps:~/
ssh user@your-vps "bash setup-cloudflare-ufw.sh"
```

The script:
- Allows SSH from anywhere (so you don't lock yourself out)
- Allows port 80/443 only from Cloudflare IPv4 and IPv6 ranges
- Denies all other inbound traffic
- Enables UFW

> Cloudflare's IP ranges occasionally change. Re-run the script if you add new ranges,
> or check the current list at https://www.cloudflare.com/ips/

---

## 3. Caddy Setup

The standard Caddy package does **not** include DNS provider plugins. You must build a
custom Caddy binary using `xcaddy`. This is done **locally on your Mac** — no Go installation
needed on the VPS.

### Build Caddy with your DNS provider plugin

This project uses **Cloudflare** for DNS. If you are on a different provider, browse the
[caddy-dns](https://github.com/caddy-dns) organisation for the matching plugin.

| DNS provider | Plugin import path | Env var names |
|---|---|---|
| Cloudflare | `github.com/caddy-dns/cloudflare` | `CLOUDFLARE_API_TOKEN` |
| Porkbun | `github.com/caddy-dns/porkbun` | `PORKBUN_API_KEY`, `PORKBUN_SECRET_KEY` |
| Route 53 (AWS) | `github.com/caddy-dns/route53` | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |
| Namecheap | `github.com/caddy-dns/namecheap` | `NAMECHEAP_API_KEY` |

> If your DNS provider does not have a caddy-dns plugin, delegate your zone to Cloudflare
> (free) and use the Cloudflare plugin instead. Point the A records to your VPS IP and keep
> the Cloudflare proxy **off** (DNS only).

Install `xcaddy` on your Mac (once):

```bash
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
```

Cross-compile Caddy for Linux AMD64:

```bash
GOOS=linux GOARCH=amd64 xcaddy build --with github.com/caddy-dns/cloudflare
```

This produces a `caddy` binary in the current directory. Upload it to the VPS:

```bash
rsync -avz caddy user@your-vps:~/caddy
ssh user@your-vps "sudo mv ~/caddy /usr/bin/caddy && sudo chmod +x /usr/bin/caddy"
ssh user@your-vps "caddy version"  # verify
```

> `rsync` cannot write directly to `/usr/bin/` as a non-root user. Upload to the home
> directory first, then move it into place with `sudo`.

### Configure Caddyfile.prod for your DNS provider

Before copying to the VPS, edit `Caddyfile.prod` locally with your specific values:

**1. Set your Let's Encrypt email** (top of the file):

```
{
    email you@example.com    ← replace with your real email
}
```

Caddy registers a Let's Encrypt account using this address. You'll receive expiry warnings
here if automatic renewal ever fails.

**2. Set the DNS provider in the `tls` block** (already configured for Cloudflare):

```
tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}
```

`{env.CLOUDFLARE_API_TOKEN}` is read at runtime from `/etc/caddy/caddy.env`, which is
passed to Caddy via `--envfile /etc/caddy/caddy.env` in the systemd unit (see
"Store the DNS API token" below for how to create that file).

If you switch providers, update this block to match the plugin's expected parameters
(see the plugin README and the provider table above).

**How DNS-01 works:** Caddy asks Let's Encrypt to issue certificates for each domain listed
in `Caddyfile.prod` (the apex domain and all prefecture subdomains).
Let's Encrypt issues a challenge token that Caddy must publish as a `_acme-challenge` TXT
record in your DNS zone. The DNS provider plugin does this automatically using the API
token you supply.

### Copy the Caddyfile to the VPS

```bash
scp Caddyfile.prod user@your-vps:~/Caddyfile
ssh user@your-vps "sudo mv ~/Caddyfile /etc/caddy/Caddyfile"
```

It configures:
- TLS via DNS-01 challenge for `warga.jp` and all 47 prefecture subdomains
- `reverse_proxy localhost:8080` for all domains, with a custom maintenance page on 502/503/504
- Security headers (see table below)

| Header | Value | Purpose |
|---|---|---|
| `X-Frame-Options` | `DENY` | Prevent clickjacking via iframes |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limit referrer leakage |
| `X-XSS-Protection` | `0` | Disable the broken legacy browser XSS auditor (it can be exploited) |
| `Content-Security-Policy` | see file | Restrict resource origins; `script-src 'self'` allows only self-hosted scripts; `connect-src 'self'` restricts connections to same origin; `form-action 'self'` prevents form hijacking to external URLs; `base-uri 'self'` blocks `<base>` tag injection; `upgrade-insecure-requests` forces HTTPS sub-resources |
| `Permissions-Policy` | see file | Disable camera, microphone, geolocation, payment, USB, and interest-cohort features |
| `-Server` | (removed) | Don't advertise Caddy version |

> **Never use inline styles or inline scripts in templates.** The CSP enforces
> `style-src 'self'` and `script-src 'self'`, which means:
>
> - `style=""` attributes → **blocked** — use a CSS class in `static/base.css`, `static/public.css`, or `static/admin.css` instead
> - `<style>` blocks in HTML → **blocked** — same fix
> - `<script>` tags with inline code or `onclick=` attributes → **blocked** — serve JS
>   as an external `.js` file from `'self'` instead
>
> Inline styles silently break layout in production while appearing fine in local dev
> (which has no CSP). If something looks correct locally but broken on the site, check
> the browser DevTools console for CSP violation errors first.

### Store the DNS API token

Generate a Cloudflare API token at **My Profile → API Tokens → Create Token**.
Use the **Edit zone DNS** template and scope it to `warga.jp` only.

SSH into the VPS, then run:

```bash
mkdir -p /etc/caddy

# Prompt for the token so it doesn't appear in shell history
read -rsp "Cloudflare API token: " CF_TOKEN && echo
echo "CLOUDFLARE_API_TOKEN=$CF_TOKEN" | sudo tee /etc/caddy/caddy.env > /dev/null
unset CF_TOKEN

sudo chmod 600 /etc/caddy/caddy.env
sudo chown caddy:caddy /etc/caddy/caddy.env
```

### Run Caddy (systemd)

```bash
sudo tee /etc/systemd/system/caddy.service << 'EOF'
[Unit]
Description=Caddy
After=network.target

[Service]
User=caddy
Group=caddy
ExecStart=/usr/bin/caddy run --config /etc/caddy/Caddyfile --envfile /etc/caddy/caddy.env
ExecReload=/usr/bin/caddy reload --config /etc/caddy/Caddyfile
TimeoutStopSec=5s
Restart=on-failure
AmbientCapabilities=CAP_NET_BIND_SERVICE
CapabilityBoundingSet=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now caddy
```

### Verify certificate issuance

After starting Caddy, confirm the cert was issued before going live:

```bash
# Watch Caddy logs for ACME/TLS activity
journalctl -u caddy -f

# Once running, check the cert details
openssl s_client -connect warga.jp:443 -servername warga.jp </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

A successful cert will show the domain name and issuer from Let's Encrypt.

> **Renewal:** Caddy renews certificates automatically ~30 days before expiry. No cron job
> or manual action needed. Renewal uses the same DNS-01 challenge, so keep the API token in
> `caddy.env` valid indefinitely.

---

## 4. App Directory Setup (one-time on VPS)

```bash
# Create dedicated app user (no login shell)
sudo useradd -r -s /bin/false wargajp

# Create app directory with dbs/ and static/ subfolders
sudo mkdir -p /opt/wargajp/dbs /opt/wargajp/static
sudo chown -R wargajp:wargajp /opt/wargajp
```

### Allow passwordless sudo for deploy commands

The deploy script runs `sudo` over a non-interactive SSH session, which cannot prompt for
a password. It uses `sudo install` to atomically place the binary with correct ownership,
and `sudo systemctl restart` to restart the service. Grant passwordless sudo for those two
commands only:

```bash
sudo tee /etc/sudoers.d/wargajp-deploy << 'EOF'
ubuntu ALL=(ALL) NOPASSWD: /usr/bin/install -o wargajp -g wargajp -m 755 /home/ubuntu/wargajp /opt/wargajp/wargajp, /usr/bin/systemctl restart wargajp
EOF
sudo chmod 440 /etc/sudoers.d/wargajp-deploy
```

Replace `ubuntu` with your SSH username if different.

The deploy flow is:
1. rsync uploads the binary to `~/wargajp` (writable by the SSH user)
2. `sudo install` atomically copies it to `/opt/wargajp/wargajp`, sets owner to `wargajp:wargajp` and permissions to `755` — no separate `chown`/`chmod` needed
3. `sudo systemctl restart wargajp` restarts the service

### Upload static files (first deploy only)

The `static/` directory contains the maintenance page. Upload it from your local machine:

```bash
rsync -avz static/ user@your-vps:~/static/
ssh user@your-vps "sudo cp -r ~/static/. /opt/wargajp/static/ && sudo chown -R wargajp:wargajp /opt/wargajp/static"
```

> The `static/` directory is read by Caddy (not the Go app), so it does not need to be
> re-uploaded on every deploy — only when you change `static/maintenance.html`.

### Create the admin account (first deploy only)

Run this **locally** before uploading databases — it creates `dbs/admin.db` with the hashed admin user:

```bash
go run scripts/create_admin.go -username=admin -email=admin@example.com -password=yourpassword
```

The script creates `dbs/admin.db` and the `admins` table if missing, bcrypt-hashes the password, and inserts the row. Running it again with the same username or email will fail (UNIQUE constraint) rather than overwrite.

### Upload databases (first deploy only)

From your **local machine**:

```bash
rsync -avz dbs/ user@your-vps:~/dbs/
ssh user@your-vps "sudo cp -r ~/dbs/. /opt/wargajp/dbs/ && sudo chown -R wargajp:wargajp /opt/wargajp/dbs"
```

> **Important:** `dbs/` is never overwritten by subsequent deploys. Only the binary is updated.
> The SQLite files are the live database — treat them like production data.

---

## 5. Systemd Unit for the App

Run this on the VPS:

```bash
sudo tee /etc/systemd/system/wargajp.service << 'EOF'
[Unit]
Description=Warga.jp Web Application
After=network.target

[Service]
Type=simple
User=wargajp
WorkingDirectory=/opt/wargajp
ExecStart=/opt/wargajp/wargajp
Environment="APP_ENV=production"
# Email notifications (optional — remove these lines to disable)
Environment="SMTP_HOST=smtp.gmail.com"
Environment="SMTP_PORT=587"
Environment="SMTP_USER=yourname@gmail.com"
Environment="SMTP_PASS=your_app_password"
Environment="NOTIFY_TO=yourname@gmail.com"
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload        # load the new unit file
sudo systemctl enable wargajp       # start automatically on boot
sudo systemctl start wargajp        # start now
sudo systemctl status wargajp       # verify it's running
```

### Common service commands

```bash
sudo systemctl status wargajp       # check current state + last few log lines
sudo systemctl restart wargajp      # restart (used after every deploy)
sudo systemctl stop wargajp         # stop (triggers maintenance page via Caddy)
sudo systemctl start wargajp        # start again after manual stop
sudo journalctl -u wargajp -f       # tail live logs
sudo journalctl -u wargajp -n 50    # last 50 log lines
```

---

## 6. Build & Deploy

### Background: why cross-compilation needs a special setup

This project uses `github.com/mattn/go-sqlite3` which is a **CGo package** — it compiles C
code alongside Go. Cross-compiling CGo from ARM macOS to x86_64 Linux requires a C
cross-compiler (`x86_64-linux-musl-gcc`). The build script handles this automatically.

### Step 1 — Install prerequisites (once, on your Mac)

```bash
# C cross-compiler for linux/amd64
brew install FiloSottile/musl-cross/musl-cross

# templ code generator
go install github.com/a-h/templ/cmd/templ@latest
```

### Step 2 — Build the production binary

```bash
./scripts/build-prod.sh
```

This script:
1. Checks that `templ` and `x86_64-linux-musl-gcc` are available
2. Runs `templ generate` to regenerate all `*_templ.go` files
3. Cross-compiles to `bin/wargajp` (Linux AMD64)

Output: `scripts/build/wargajp` — a single self-contained binary (~30 MB) with Pico.css and all
templates embedded. No other files need to be deployed alongside it.

### CSS cache busting

The build script injects the current git commit hash into the binary at compile time:

```bash
-X website/version.Build=$(git rev-parse --short HEAD)
```

Every template references CSS files with a version query string:

```html
<link rel="stylesheet" href="/static/base.css?v=b6192eb"/>
<link rel="stylesheet" href="/static/public.css?v=b6192eb"/>
```

This means each deploy serves CSS at a new URL, so Cloudflare automatically caches the
new version without needing a manual cache purge. Local dev uses `?v=dev` (the default
when the variable is not set).

**Rule:** if you add a new template or layout, always use `version.Build` in the CSS
`href` attributes — never hardcode paths without the version suffix. Public templates load `base.css` + `public.css`; admin templates load `base.css` + `admin.css`.

### Step 3 — Build and deploy

Use the deploy script, which builds the binary and uploads it in one step:

```bash
./scripts/deploy.sh <user> <vps-host>
```

Example:

```bash
./scripts/deploy.sh ubuntu 1.2.3.4
```

The script will:
1. Build the production binary via `build-prod.sh`
2. Upload it to `~/wargajp` on the VPS via rsync
3. Move it to `/opt/wargajp/wargajp`, set ownership and permissions
4. Restart the `wargajp` systemd service

The maintenance page (served by Caddy) is shown automatically during the ~2 second restart window.

> **Cloudflare cache:** No manual cache purge needed. CSS files are served with a git
> commit hash in the URL (`?v=abc1234`), so each deploy automatically gets a fresh cache
> entry without touching Cloudflare.

---

## 7. SQLite Backup

Add a daily cron on the VPS to back up all databases:

```bash
# /etc/cron.d/wargajp-backup
0 2 * * * root tar -czf /backups/wargajp-$(date +\%F).tar.gz /opt/wargajp/dbs/
```

Create the backup directory first: `mkdir -p /backups`

For off-site backups, pipe to `rclone` or `s3cmd` instead of a local path.

---

## 8. Maintenance Mode

### How it works

`Caddyfile.prod` uses Caddy's `handle_response` directive to intercept 502/503/504 responses
from the Go app and serve `/opt/wargajp/static/maintenance.html` instead. This kicks in
automatically whenever the app process is down — no manual action required for normal deploys.

The restart during `systemctl restart wargajp` takes ~1–2 seconds. Visitors hitting the
site in that window see the maintenance page rather than a browser error.

### Normal deploys (no action needed)

```bash
make deploy-remote   # restarts the app — maintenance page shown automatically for ~2 sec
```

### Longer maintenance windows (optional manual toggle)

If you need the site down for several minutes (e.g. a large DB migration), stop the app
before starting work and start it again when done:

```bash
# Enable maintenance mode
ssh user@your-vps "systemctl stop wargajp"

# ... do your work ...

# Disable maintenance mode
ssh user@your-vps "systemctl start wargajp"
```

Caddy keeps running the whole time — it serves the maintenance page for every request
while the app is stopped, and switches back to proxying as soon as the app is up again.

### Updating the maintenance page

Edit `static/maintenance.html` locally, then upload:

```bash
rsync -avz static/ user@your-vps:~/static/
ssh user@your-vps "sudo cp -r ~/static/. /opt/wargajp/static/ && sudo chown -R wargajp:wargajp /opt/wargajp/static"
```

No Caddy or app reload needed.

---

## 9. Embedded Analytics

Analytics are built into the binary — no separate service, no JS beacon, no GeoIP database.

### How it works

- **Page-view tracking:** `lib/analytics/tracker.go` — a chi middleware that fires a goroutine after each response. It writes one row to `pageviews` per recorded request. Filters out `/admin`, `/static/`, `/ping`, `/robots.txt`, `/sitemap.xml`, `/og-image.png`, and all non-GET requests.
- **Search query tracking:** `lib/analytics/tracker.go` — `RecordSearch()` is called from `IndexHandler` after each validated search query (≥ 3 chars) passes the rate limiter. Writes one row to `search_queries` per search, for both cross-prefecture and prefecture-scoped searches.
- **Country detection:** reads Cloudflare's `CF-IPCountry` header (free on every proxied request). Defaults to `XX` when the header is absent (e.g. Cloudflare proxy disabled or DNS-only mode).
- **Visitor hashing:** SHA-256 of `(daily_salt | IP | UserAgent)`, truncated to 16 bytes. The salt rotates at UTC midnight — same visitor collapses to one hash within a day, and hashes become unrecoverable the next day (GDPR-friendly, no cookies).
- **Writes are non-blocking** — both `Middleware` and `RecordSearch` fire goroutines after the response is sent, so analytics can never delay a request.

### Database

`dbs/analytics.db` is created automatically on first run. All database files (including `analytics.db`) are opened with WAL journal mode, which means readers never block writers and writes never stall page requests. The analytics tracker owns its own write mutex, fully decoupled from the shared prefecture/user/admin DB mutex.

| Migration | Description |
|---|---|
| v1 | `pageviews` table + single-column indexes |
| v2 | `search_queries` table + single-column indexes |
| v3 | Replaces single-column indexes with composite covering indexes (`idx_pv_covering`, `idx_sq_covering`) for faster dashboard GROUP BY queries |

It is included in the nightly backup cron alongside the other DB files.

### Dashboard

Log in to the admin panel and visit `/admin/analitik`. The page shows:

- Total pageviews and approximate daily-unique visitors over a rolling 7 / 30 / 90-day window
- **Halaman Teratas** — top paths grouped by subdomain + path, paginated at 20 rows per page (`?paths_page=N`)
- **Pencarian Teratas** — top search queries grouped by query + subdomain, paginated at 20 rows per page (`?searches_page=N`)
- **Negara** — country breakdown (2-letter ISO codes from `CF-IPCountry`; `XX` = unknown), capped at 20 rows

The `?days=` query parameter is whitelisted to `7`, `30`, or `90` — arbitrary values are silently ignored and default to 30 days. Page params (`?paths_page=`, `?searches_page=`) are preserved in pagination links alongside `?days=`.

### Performance considerations

Adding the analytics middleware increased page load time measurably before the following mitigations were applied. If you ever see a latency regression after a DB-related change, check these areas first.

**SQLite WAL mode (all DBs)**
All 54 database files are opened with `?_journal_mode=WAL&_busy_timeout=5000&_synchronous=NORMAL&_cache_size=-8000`. Without WAL, SQLite holds an exclusive write lock for every INSERT, which blocks concurrent readers. With WAL, readers and writers proceed independently — a background analytics write never stalls a page render.

**Connection pool cap (`SetMaxOpenConns(1)`)**
Each SQLite file is opened with a maximum of one concurrent connection. SQLite is not multi-writer safe; allowing Go's default pool of 25 connections to the same file causes `SQLITE_BUSY` errors under load. The 5-second `_busy_timeout` pragma handles the rare case where two goroutines queue on the same file.

**Decoupled write mutex**
The analytics `Tracker` owns its own `sync.Mutex` — it is not shared with the prefecture or user/admin DBs. Before this, a slow analytics flush (writing to `analytics.db`) could block a handler that also needed to write to a prefecture DB because both goroutines contended on the same mutex.

**Composite covering indexes on analytics tables**
The dashboard queries aggregate millions of rows with `GROUP BY` across a date window. The single-column indexes from migrations v1/v2 required a full table scan for the `WHERE seen_at >= ...` filter followed by a hash aggregation. Migration v3 replaced them with composite covering indexes:

| Index | Columns |
|---|---|
| `idx_pv_covering` | `seen_at, subdomain, path, visitor_hash` |
| `idx_sq_covering` | `seen_at, query, subdomain, visitor_hash` |

These allow the query planner to satisfy the `WHERE`, `GROUP BY`, and `COUNT(DISTINCT visitor_hash)` entirely from the index without touching the table rows.

**What to watch if latency increases again**
- Check `PRAGMA journal_mode;` on each DB — it should return `wal`. If it returns `delete`, the DB was opened without the pragma (e.g. by an external tool that re-created the file).
- Run `EXPLAIN QUERY PLAN SELECT ...` on the slow dashboard query in `sqlite3` to confirm the covering index is being used.
- If `analytics.db` grows very large (millions of rows), the covering index scan still reads more blocks — the pruning cron (see below) is the correct fix.
- If a new DB write path is added (new table, new handler), make sure it uses the `openDB()` helper in `migrations/migration.go` rather than a bare `sql.Open("sqlite3", path)`.

### No extra deployment steps needed

`analytics.db` is created by the app on startup. No systemd unit, no DNS record, and no Caddyfile change is required beyond what is already documented in this guide.

> **Country data requires Cloudflare proxy to be enabled (orange cloud).** When running in DNS-only mode, all countries will appear as `XX`.

### Pruning old rows (recommended)

The `pageviews` table grows indefinitely — rows are never deleted automatically. Add a monthly cron on the VPS to remove entries older than 12 months:

```bash
# /etc/cron.d/wargajp-analytics-prune
0 3 1 * * root sqlite3 /opt/wargajp/dbs/analytics.db "DELETE FROM pageviews WHERE seen_at < datetime('now', '-12 months'); DELETE FROM search_queries WHERE seen_at < datetime('now', '-12 months');"
```

Runs at 03:00 on the 1st of each month. Adjust the retention window (`-12 months`) to taste.

> The dashboard only queries up to 90 days, so anything beyond that is invisible in the UI. 12 months keeps a reasonable historical buffer for manual SQL queries while bounding disk usage.

---

## 10. Verify the Deployment

```bash
# App health check (apex domain)
curl -I https://warga.jp/ping

# Subdomain TLS + routing
curl -I https://tokyo.warga.jp/ping

# Prefecture DB selection (should render the Aichi panduan list)
curl -s https://aichi.warga.jp/ | grep -i panduan

# Analytics dashboard redirect (not logged in → 303 to /admin/login, not 500)
curl -sI https://warga.jp/admin/analitik | head -2

# Analytics DB row count (SSH into VPS first)
sqlite3 /opt/wargajp/dbs/analytics.db "SELECT COUNT(*) FROM pageviews;"

# App logs
journalctl -u wargajp -f

# Caddy TLS/proxy logs
journalctl -u caddy -f
```

The first three `curl` checks returning `200 OK` confirms DNS, TLS, proxy, and subdomain routing.
The analytics redirect check confirms the dashboard route is wired (a `303` means the handler is
reachable and auth is working; `500` would indicate a DB or wiring problem).

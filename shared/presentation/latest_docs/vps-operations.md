# VPS Operations

Ongoing operational runbooks for the production VPS. For one-time provisioning (DNS, Caddy setup, systemd unit, first deploy), see `docs/VPS_SETUP.md`.

---

## SQLite Backup

Add a daily cron on the VPS to back up all databases:

```bash
# /etc/cron.d/wargajp-backup
0 2 * * * root tar -czf /backups/wargajp-$(date +\%F).tar.gz /opt/wargajp/dbs/
```

Create the backup directory first: `mkdir -p /backups`

For off-site backups, pipe to `rclone` or `s3cmd` instead of a local path.

---

## Maintenance Mode

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

## Analytics — Pruning old rows

The `pageviews` table grows indefinitely. Add a monthly cron on the VPS to remove entries older than 12 months:

```bash
# /etc/cron.d/wargajp-analytics-prune
0 3 1 * * root sqlite3 /opt/wargajp/dbs/analytics.db "DELETE FROM pageviews WHERE seen_at < datetime('now', '-12 months'); DELETE FROM search_queries WHERE seen_at < datetime('now', '-12 months');"
```

Runs at 03:00 on the 1st of each month. The dashboard only queries up to 90 days, so anything beyond that is invisible in the UI; 12 months keeps a historical buffer for manual queries while bounding disk usage.

---

## Verify Deployment

```bash
# App health check — returns JSON {version, build, build_time, status}
curl -s https://warga.jp/ping | jq

# Subdomain TLS + routing
curl -s https://tokyo.warga.jp/ping | jq

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

# VPS Setup Guide: Warga.jp on Caddy

## Overview

```
Internet
  → DNS: warga.jp + per-prefecture A records → VPS IP   (DNS provider with API access, DNS-only)
  → Caddy :443 on VPS                                    (TLS via DNS-01 challenge)
  → Go app :8080 on VPS                                  (subdomain extracted from Host header)
  → /opt/wargajp/dbs/*.db                                (4 SQLite files, persisted on VPS)
```

The Go app listens on `:8080`. Caddy terminates TLS and forwards requests. The subdomain
in the `Host` header (`tokyo.warga.jp` → `tokyo`) tells the app which prefecture to scope
content queries to.

---

## 0. VPS Spec

### Recommended minimum

| Resource | Minimum | Comfortable |
|---|---|---|
| vCPU | 1 | 1–2 |
| RAM | 1 GB | **2 GB** |
| Storage | 20 GB SSD | 40 GB NVMe |
| Transfer | 500 GB/mo | 1 TB/mo |

**2 GB RAM is recommended.** The app opens 4 SQLite connections (`app.db`, `admin.db`, `user.db`, `analytics.db`) with a pool sized to `max(NumCPU×2, 4)` per DB. Combined with the Go binary (~20 MB), Caddy (~30 MB), and OS overhead (~150–200 MB), the working set sits well under 500 MB. 1 GB is sufficient; 2 GB leaves comfortable headroom for OS page cache (which SQLite benefits from significantly).

**NVMe SSD preferred.** SQLite writes are I/O bound. NVMe reduces write latency noticeably compared to SATA SSD, which matters for form submissions.

### Operating system

**Ubuntu 22.04 LTS** — recommended. Supported until April 2027, largest community, most documentation. All commands in this guide (`systemctl`, `useradd`, `apt`, etc.) are written for Ubuntu/Debian.

**Debian 12 (Bookworm)** is also a good choice — more minimal, same `apt` commands, identical setup steps.

Avoid CentOS (end of life), Rocky Linux/AlmaLinux (`dnf`/`rpm` differ from this guide), and FreeBSD (no `systemd`).

### Datacenter location

Choose a datacenter in or near Japan. The target audience (Indonesian community in Japan) will see ~10–20 ms latency from a Tokyo-region server vs. ~200 ms from Europe or the US.

### When to scale up

See [docs/SCALING.md](SCALING.md) for the full scaling strategy: vertical vs horizontal priority, SQLite read-replica options (Litestream/LiteFS/Postgres), the sessions-in-SQLite constraint, and trigger thresholds for each stage.

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
A   www             <VPS IP>
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

> **Why a `www` record?**
> Without an explicit `www.warga.jp` record, requests for `www.warga.jp` either fail
> at DNS or land on a Cloudflare/edge fallback whose robots policy confuses Google
> Search Console (the URL ends up reported under "Indexed, though blocked by
> robots.txt"). The `www.warga.jp` block in `Caddyfile.prod` 301-redirects to apex,
> so Google consolidates it into a single canonical host — but Caddy can only do
> that if DNS resolves the name to the VPS first.

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
- TLS via DNS-01 challenge for `warga.jp` and all 48 subdomains (47 prefectures + `national`)
- `reverse_proxy localhost:8080` for all domains, with a custom maintenance page on 502/503/504
- Security headers (see `docs/rules/security-headers.md` for the full table)

> **Never use inline styles or inline scripts in templates.** The CSP enforces
> `style-src 'self'` and `script-src 'self'`. Inline styles silently break layout in
> production while appearing fine in local dev (which has no CSP). See `docs/rules/security-headers.md`
> § CSP rules for examples of common violations.

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

`deploy.sh` runs `sudo` over a non-interactive SSH session (no TTY), which cannot prompt
for a password. It uses `sudo install` to atomically place the binary and `sudo systemctl
restart` to restart the service. Grant passwordless sudo for those two commands only:

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

> **Note:** `consolidate-dbs.sh` (the one-time data migration script) uses `ssh -t` to
> allocate a pseudo-TTY, so it can prompt for your sudo password interactively. You do
> not need to add those commands to the NOPASSWD list.

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
# Outbound mail via Resend (warga.jp verified there). Inbound is Cloudflare Email Routing — no app config needed for inbound.
# Remove the SMTP_* lines to disable email notifications (contact messages still persist in admin.db).
Environment="SMTP_HOST=smtp.resend.com"
Environment="SMTP_PORT=465"
Environment="SMTP_USER=resend"
Environment="SMTP_PASS=re_your_api_key"
Environment="SMTP_FROM=kontak@warga.jp"
Environment="NOTIFY_TO=your-personal@example.com"
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

## 6. Operations

SQLite backups, maintenance mode, analytics pruning cron, and the verify-deployment checklist: see **[`docs/vps-operations.md`](../vps-operations.md)**.

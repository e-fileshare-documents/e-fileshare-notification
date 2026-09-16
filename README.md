<p align="center">
  <img src="https://img.shields.io/badge/privacy-first-10b981?style=flat-square&logo=shield&logoColor=white" alt="Privacy First">
  <img src="https://img.shields.io/badge/zero-tracking-1a1a1a?style=flat-square" alt="Zero Tracking">
  <img src="https://img.shields.io/badge/docker-ready-2496ed?style=flat-square&logo=docker&logoColor=white" alt="Docker Ready">
  <img src="https://img.shields.io/badge/python-3.11+-3776ab?style=flat-square&logo=python&logoColor=white" alt="Python 3.11+">
</p>

<h1 align="center">🔗 Cloak.URL</h1>
<p align="center"><strong>Private, self-hosted URL shortener — zero tracking, zero logs, zero analytics.</strong></p>
<p align="center"><code>yourdomain.com/blog/code</code> not <code>blog.yourdomain.com/code</code></p>

---

## 🚀 Quick Install

### Option 1: Download ZIP (Recommended)

```bash
wget -q https://github.com/sarakmacbook/Cloak.URL/archive/refs/heads/main.zip -O cloak-url.zip
unzip -q cloak-url.zip
cd Cloak.URL-main
bash install.sh
```

### Option 2: Git Clone

```bash
git clone https://github.com/sarakmacbook/Cloak.URL.git
cd Cloak.URL
bash install.sh
```

The installer asks **4 questions** (all have defaults):

| Question | Default | What it does |
|----------|---------|--------------|
| **Deployment method** | Cloudflare Tunnel | `1` = Tunnel (no open ports), `2` = Nginx |
| **Port** | `3000` | Host port (maps to container port 3000) |
| **Database location** | `./data` | Where SQLite DB lives |
| **Domain** | `localhost` | Your public domain |
| **Cloudflare account tag** *(optional, tunnel only)* | skip | Makes the dashboard links deep-link into your Zero Trust account |

Press **Enter** to accept defaults. Done in 30 seconds.

When it finishes, the installer prints the **ready-to-open Cloudflare links** for
your domain — no hunting through menus.

---

## 💾 Database Location Options

During install, pick where your SQLite database lives:

### Option 1: Project Folder (Default)
```
./data/urls.db
```
- **Pros:** Easy to backup, visible files, portable
- **Cons:** Deleted if you delete the project folder
- **Best for:** Development, small deployments

### Option 2: Docker Volume
```
Docker named volume: cloak-url-data
```
- **Pros:** Survives container deletion, managed by Docker
- **Cons:** Hidden path, harder to backup manually
- **Best for:** Production, automated backups

### Option 3: Custom Path
```
/var/lib/cloak-url/data/urls.db
/mnt/external-drive/cloak-url/data/urls.db
```
- **Pros:** Full control, can mount external drives, easy to back up
- **Cons:** You manage permissions
- **Best for:** Servers with dedicated storage, NAS, external drives

**Change later:** Edit `docker-compose.yml` → `volumes:` section, then `docker compose up -d`.

---

## 🔌 Connect a Domain (Cloudflare Tunnel, ~3 min)

New in v1.2: Cloak.URL **gives you the URL to connect Cloudflare** for any domain you type,
then checks whether the domain actually reaches this install.

Open the UI → **🌐 Custom domain · Cloudflare Tunnel**, type `links.mybrand.com`, press
**Get setup link**. You get:

1. **A one-click link** into Cloudflare — `one.dash.cloudflare.com/.../networks/tunnels/create`
   (straight into your account when `CLOUDFLARE_ACCOUNT_TAG` is set), plus `My tunnels`,
   `DNS · <zone>` and `Add domain to Cloudflare` shortcuts.
2. **The exact route to create** — hostname `links.mybrand.com` → service `http://cloak:3000`
   (copy buttons), the `docker-compose.yml` tunnel block, and an optional `config.yml`/CNAME variant.
3. **A Verify button** — resolves the hostname, then makes **one** HTTPS request to
   `https://links.mybrand.com/api/health` and tells you what's missing.

| Check result | Meaning | What to do |
|--------------|---------|------------|
| ✅ Connected | Tunnel live and serving **this** install | Nothing — save the domain as default |
| 🟠 Connector offline | Cloudflare answers, no healthy `cloudflared` (Error 1033/1034) | `docker compose up -d tunnel` → `logs tunnel` |
| 🟠 No tunnel route | On Cloudflare, hostname not mapped (Error 1016 / 530) | Add the published-application route |
| 🔴 No DNS record | Hostname doesn't resolve | Add it to the zone (proxied) or let the route create it |
| 🟠 TLS / cert issue | No certificate for that hostname | Deeper subdomains need an Advanced Certificate |
| 🟠 Another app answers | Route points at the wrong service | Point it at `http://cloak:3000` |
| ⚪ Private address | Resolves to LAN/loopback/link-local | Not probed on purpose — no SSRF by design |

Saved domains show up as a dropdown on the **Custom Domain** field, can be starred as
**default** (auto-filled for new links), and every `/api/shorten` response for an
unverified domain comes back with `domain_advice` — the Cloudflare link + a Verify button,
right next to your new short URL.

> **Privacy:** verification stores only the hostname. No Cloudflare API calls, no API
> tokens, no cookies, and nothing is sent anywhere except a single `GET /api/health`
> to the hostname you asked about. Requests to private/link-local addresses are refused.

---

## 🌐 Deployment Methods

### Cloudflare Tunnel (Recommended)

No open ports. No static IP. Works behind any router.

```bash
bash install.sh
# → Press Enter (selects Cloudflare Tunnel)
# → Paste your Cloudflare token
# → Enter domain: mybrand.com
```

**Get token:** [one.dash.cloudflare.com](https://one.dash.cloudflare.com) → Networks → Tunnels → Create → Docker → Copy token

**Add hostname:** Cloudflare dashboard → Networking → Tunnels → your tunnel → Routes →
**Add a route → Published application** → `mybrand.com` → HTTP `http://cloak:3000`

Deep links Cloak.URL generates for you (same ones the UI shows):

| Purpose | Link |
|---------|------|
| Create a tunnel | `https://one.dash.cloudflare.com[/<account-tag>]/networks/tunnels/create` |
| Tunnel list | `https://one.dash.cloudflare.com[/<account-tag>]/networks/tunnels` |
| Zone DNS records | `https://dash.cloudflare.com/?to=/:account/<zone>/dns` |
| Add a domain to Cloudflare | `https://dash.cloudflare.com/?to=/:account/add-site` |
| Error 1033 troubleshooting | [error-1033 docs](https://developers.cloudflare.com/support/troubleshooting/http-status-codes/cloudflare-1xxx-errors/error-1033/) |

> **Note:** Use container port `3000` for the tunnel, not your custom host port.

### Nginx

For VPS with static IP.

```bash
bash install.sh
# → Type 2 (selects Nginx)
# → Enter domain: mybrand.com
# → sudo bash nginx/install-nginx.sh
# → sudo certbot --nginx  # optional HTTPS
```

**Point DNS:** `A  mybrand.com  YOUR_SERVER_IP`

---

## 📁 URL Examples

```
Simple:          mybrand.com/abc123
With prefix:     mybrand.com/blog/post-2026
With password:   mybrand.com/secret/doc
With expiration: mybrand.com/go/sale
```

---

## ⚙️ Configuration

Edit `docker-compose.yml` and restart:

```bash
nano docker-compose.yml
docker compose up -d
```

The installer also writes a small **`.env`** (`TUNNEL_SERVICE`, `TUNNEL_NAME`,
`CLOUDFLARE_ACCOUNT_TAG`) — compose reads it automatically, so the domain panel's links
keep matching your setup after re-deploys. Tokens stay in `docker-compose.yml`.

| Variable | Default | Description |
|----------|---------|-------------|
| `METHOD` | `cloudflare` | `cloudflare` or `nginx` |
| `BASE_URL` | `http://localhost:3000` | Your domain |
| `PORT` | `3000` | Host port |
| `DB_HOST_PATH` | `./data` | Host database directory |
| `TUNNEL_TOKEN` | — | Cloudflare token |
| `TUNNEL_SERVICE` | `http://cloak:3000` | Service address shown in the connect panel |
| `TUNNEL_NAME` | `cloak-url` | Suggested tunnel name |
| `CLOUDFLARE_ACCOUNT_TAG` | — | Zero Trust tag from `one.dash.cloudflare.com/<tag>/…` — makes links deep-link. Not a secret |
| `DOMAIN_CHECK_TIMEOUT` | `6` | Seconds for a domain verify (DNS + HTTPS) |
| `DOMAIN_CHECK_COOLDOWN` | `5` | Minimum seconds between checks of the same hostname |
| `MAX_DOMAINS` | `100` | Cap on saved/remembered hostnames |

---

## 🔌 API

```bash
wget -qO- https://mybrand.com/api/shorten \
  --post-data='{"url":"https://example.com","path_prefix":"blog","custom_code":"post-2026"}' \
  --header="Content-Type: application/json"
```

### Domain / Cloudflare endpoints

```bash
# Cloudflare connect links + copy-paste snippets for a hostname
curl 'https://mybrand.com/api/cloudflare/setup?domain=links.mybrand.com'

# Save a domain (note: only the hostname is stored)
curl -X POST https://mybrand.com/api/domains -H 'Content-Type: application/json' \
  -d '{"domain":"links.mybrand.com","note":"brand","is_primary":true}'

# Verify it: DNS + one HTTPS probe of https://<domain>/api/health
curl -X POST https://mybrand.com/api/domains/verify -H 'Content-Type: application/json' \
  -d '{"domain":"links.mybrand.com"}'

curl https://mybrand.com/api/domains                                            # list + status
curl -X POST https://mybrand.com/api/domains/primary -d '{"domain":""}' -H 'Content-Type: application/json'  # clear default
curl -X POST https://mybrand.com/api/domains/delete  -d '{"domain":"links.mybrand.com"}' -H 'Content-Type: application/json'
curl https://mybrand.com/api/health                                              # what the verifier looks for
```

`POST /api/shorten` also returns `domain_advice` (`verified: false` + `links`) when the
domain you used has not been verified.

---

## 🧪 Development

Run it without Docker, then try the domain panel:

```bash
DB_PATH=./data/urls.db BASE_URL=http://localhost:3000 python3 app.py
```

Tests are stdlib-only (`unittest`) and cover the domain states — `live`, `tunnel_down`
(Error 1033), `no_route` (1016), `dns_missing`, `foreign_origin`, the private-address SSRF
guard, cooldowns, and the redirect/lookup priority:

```bash
python3 -m unittest discover -s tests -t .        # 27 tests
node tools/ui-smoke.js                            # renders the panel against a live server
```

`tools/ui-smoke.js` executes `index.html`'s inline script in a tiny DOM shim and asserts the
connect panel, verify status box, saved-domain list and the "not connected yet" banner render
(and escape) correctly. Set `CLOAK_BASE` if the app is not on `:3000`.

---

## 🛡️ Privacy

```
✓ No IP logging        ✓ No click counters
✓ No User-Agent logs   ✓ No third-party scripts
✓ No Referer logs      ✓ No cookies
✓ No analytics         ✓ No external APIs
✓ No Cloudflare tokens ✓ No cookies for domain checks
```

Domain verification is the only outbound request the app makes, and only when **you** press
Verify — it asks about *your* hostname, not about your visitors.

---

## 📝 License

MIT — free to use, modify, and self-host.

---

<p align="center">Built for privacy. No analytics. No tracking. Just links.</p>

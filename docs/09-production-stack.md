# Production Stack: vanderdev.net

## Why This Exists

The development workflow needs a production target. Without one, code is written and tested but never deployed. vanderdev.net is where the infrastructure meets the public internet.

The site is a React single-page application that hosts an infrastructure command center dashboard, three AI-powered chatbots, weather and environmental data displays, and government tools. You can see it live at [vanderdev.net](https://vanderdev.net). It runs on a Hostinger VPS with 29 Docker containers, sits behind Cloudflare for edge protection, and renews its own SSL certificates automatically.

This chapter covers the application architecture, every Docker container running on the VPS, the nginx reverse proxy configuration, the CI/CD pipeline, and the edge security layer. By the end, you will understand how a single VPS serves a production React app alongside a self-hosted Supabase stack, a monitoring pipeline, and three RAG-powered chatbots -- and how to reproduce it.

---

## Tech Stack

The frontend is a standard modern React application. Nothing exotic -- the choices prioritize stability and broad community support.

| Technology | Version | Purpose |
|------------|---------|---------|
| React | 18.2.0 | UI framework |
| Vite | 5.4.2 | Build tool and dev server |
| React Router DOM | 6.30.3 | Client-side routing (SPA) |
| Tailwind CSS | 3.4.1 | Utility-first styling |
| ReactMarkdown | 9.0.1 | Bot response rendering |
| remark-gfm | 4.0.1 | Tables, strikethrough in markdown |
| Recharts | 3.6.0 | Dashboard charts |

**Why these versions matter:** The React 18 + Vite 5 combination gives you fast hot module replacement during development and a production build that is a static `dist/` directory. No server-side rendering, no runtime Node process. The VPS serves static files through nginx. This is as simple as deployment gets.

---

## Application Routes

The app is a single-page application with client-side routing. Every route below is handled by React Router -- the server always returns `index.html` and the JavaScript takes over from there.

| Route | Component | Description |
|-------|-----------|-------------|
| `/` | Dashboard | Infrastructure command center with live status tiles |
| `/waterbot` | WaterBot | California water regulations assistant |
| `/bizbot` | BizBot | Business licensing assistant |
| `/kiddobot` | KiddoBot | Childcare assistance assistant |
| `/weather` | WeatherStation | Ambient weather station data |
| `/space-weather` | SpaceWeather | NOAA space weather displays |
| `/stream-gauges` | StreamGauge | USGS stream gauge data |
| `/accessview` | AccessView | Access management document processing |

Each bot has its own route, its own UI components, and its own data files. But they share a common bot library (`src/lib/bots/`) for chat rendering, markdown formatting, and session management. That shared library is covered in the chatbot chapter.

---

## Source Structure

Understanding where things live in the codebase makes everything else in this chapter easier to follow.

```
src/
  App.jsx                    # Routes + layout (sidebar + mobile nav)
  main.jsx                   # React entry point
  index.css                  # Tailwind + custom CSS (dark theme, glows)

  components/
    BotHeader.jsx            # Shared bot header (back, reset, copy)
    Sidebar.jsx              # Desktop sidebar + MobileNav
    Icons.jsx                # 60+ SVG icons via createIcon() factory
    # Dashboard tiles:
    ServerStatus.jsx         # VPS uptime and resource usage
    N8nStatus.jsx            # n8n workflow execution status
    MeshHealth.jsx           # Tailscale device connectivity
    DNSHealth.jsx            # Pi-hole blocking statistics
    MCPServices.jsx          # MCP server status on ServerM2P
    BackupStatus.jsx         # Last backup times and health
    AlertsFeed.jsx           # Live ntfy.sh notification stream
    BotHealth.jsx            # Webhook response times per bot
    DockerResources.jsx      # Container status and memory usage
    SecurityPanel.jsx        # Security audit summary
    QuickLinks.jsx           # Shortcut links to services

    bizbot/                  # BizBot-specific components
    kiddobot/                # KiddoBot-specific components
    waterbot/                # WaterBot-specific components
      IntakeForm.jsx         # Water system profiling wizard
      FundingNavigator.jsx   # 5-step funding wizard (58 programs)
    streamgauge/             # Stream gauge components
    weatherstation/          # Weather station components
    spaceweather/            # Space weather components

  hooks/
    useBotPersistence.js     # Session persistence (sessionStorage)
    usePolledFetch.js        # Polling data hook for dashboard tiles

  lib/bots/                  # Shared bot library
    ChatMessage.jsx          # Markdown rendering with react-markdown + remark-gfm
    autoLinkUrls.js          # URL detection in bot responses
    DecisionTreeView.jsx     # Interactive decision trees for bots
    RAGButton.jsx            # RAG source citation button

  data/                      # Static data files
    funding-programs.json    # 58 California water funding programs
    funding-decision-tree.json
    permit-decision-tree.json
    kiddobot-program-tree.json
    kiddobot-county-rr.json

  utils/
    matchFundingPrograms.js  # Deterministic funding matcher
    formatRelativeTime.js    # Time formatting

  pages/
    Dashboard.jsx            # Command Center (~7 tile sections)
    WaterBot.jsx
    BizBot.jsx
    KiddoBot.jsx
    WeatherStation.jsx
    SpaceWeather.jsx
    StreamGauge.jsx
    AccessView.jsx
```

The separation is intentional. Pages define layout and data flow. Components handle rendering. Hooks handle state and fetching. The `lib/bots/` directory is shared across all three chatbots so changes to markdown rendering, citation display, or chat styling propagate to every bot at once.

---

## The Dashboard -- Infrastructure Command Center

The dashboard at `vanderdev.net/` provides real-time visibility into the entire infrastructure. It is the single pane of glass for everything documented in this guide.

**What each tile shows:**

- **System Health** -- VPS uptime, CPU, memory, disk usage
- **n8n Workflows** -- Workflow execution counts, success/failure rates
- **Mesh Health** -- Tailscale device status across all 7 machines (online/offline, last seen)
- **DNS Health** -- Pi-hole blocking statistics from both NetSentry and AlertNode
- **MCP Services** -- Status of all MCP servers running on ServerM2P
- **Alerts Feed** -- Live stream of ntfy.sh notifications (infrastructure alerts, backup status)
- **Backup Status** -- Last backup timestamp and health for each backup job
- **Bot Health** -- Webhook response times for all 3 chatbots
- **Docker Resources** -- Container count, status, and aggregate memory usage

**How the data flows:**

```
Dashboard tiles (React)
    |
    | fetch every 10 seconds (usePolledFetch hook)
    |
    v
api.vanderdev.net/infrastructure-status
    |
    | Flask API (vps-api container)
    |
    v
Docker API, Prometheus, disk stats, etc.
```

The `vps-api` container runs a Flask application that aggregates status from Docker, Prometheus, and system-level metrics into a single JSON response. The dashboard polls this endpoint every 10 seconds. No WebSockets, no complex state management -- just a periodic fetch with a loading spinner.

---

## VPS Docker Architecture

The VPS runs 29 Docker containers organized into functional categories. This is the complete inventory, verified against the running system.

### Container Inventory

| Category | Count | Containers | Purpose |
|----------|-------|-----------|---------|
| Core | 6 | vanderdev-website, nginx-proxy, nginx-proxy-acme, n8n, vps-api, ecos-form | Website serving, reverse proxy, SSL automation, workflow engine, status API, government forms |
| Supabase | 9 | supabase-db, supabase-kong, supabase-rest, supabase-studio, supabase-auth, supabase-meta, supabase-realtime, supabase-storage, supabase-imgproxy | PostgreSQL + pgvector, API gateway, admin UI, auth, realtime subscriptions, file storage |
| Monitoring | 5 | grafana, prometheus, node-exporter, influxdb, portainer | Dashboards, metrics collection, time-series data, container management UI |
| Data Collection | 2 | ambient-weather-collector, usgs-stream-collector | Weather station and stream gauge data ingestion |
| Caching | 1 | n8n-cache-redis | Redis for n8n workflow caching |
| Alerting | 1 | alertmanager | Prometheus alerts routed to ntfy.sh |
| Uptime | 1 | uptime-kuma | Synthetic monitoring with 11 configured monitors |
| Utilities | 3 | pgadmin, telemetry-proxy, health-monitor | Database admin UI, telemetry routing, health check service |
| App | 1 | accessview-api | Document processing API for AccessView |

### Memory Tiers

Container memory limits are not optional on a shared VPS. Without them, one runaway container can exhaust all available memory and crash everything else on the server (an OOM kill cascade). Every container has an explicit memory cap.

| Tier | Limit | Containers |
|------|-------|-----------|
| Heavy | 2 GB | n8n, supabase-db |
| Medium | 512 MB | grafana, supabase-kong, supabase-studio, pgadmin, supabase-rest, supabase-auth, portainer, uptime-kuma |
| Light | 256 MB | influxdb, vps-api, ecos-form, n8n-cache-redis, alertmanager, supabase-realtime, supabase-storage, supabase-imgproxy, accessview-api |
| Minimal | 128 MB | node-exporter, health-monitor, telemetry-proxy, ambient-weather-collector, usgs-stream-collector, supabase-meta |

**Why n8n and supabase-db get 2 GB:** n8n runs complex multi-step workflows with large payloads (RAG ingestion, document generation). The Supabase PostgreSQL database holds pgvector embeddings that consume significant memory during similarity searches. Both will hit OOM errors with less.

**Why Redis is only 256 MB:** It is configured with `allkeys-lru` eviction, meaning it discards the least-recently-used keys when it approaches the limit. This is a cache, not a persistent store -- data loss is acceptable.

---

## Nginx Configuration

Nginx is the single entry point for all HTTP traffic. The `nginx-proxy` container handles reverse proxying, and its companion `nginx-proxy-acme` handles automatic SSL certificate renewal.

### Reverse Proxy Routing

All traffic arrives at nginx and gets routed based on the hostname.

| Domain | Routes To | Container |
|--------|-----------|-----------|
| `vanderdev.net` | Static files in `/dist/` | vanderdev-website |
| `api.vanderdev.net` | Flask API | vps-api |
| `n8n.vanderdev.net` | n8n workflow UI and webhooks | n8n |
| `grafana.vanderdev.net` | Grafana dashboards | grafana |
| `portainer.vanderdev.net` | Container management UI | portainer |

This is subdomain-based routing. Each service gets its own subdomain, and nginx proxies to the appropriate container based on the `Host` header. No path-based routing conflicts, no shared namespaces.

### Rate Limiting

Four rate-limiting zones protect against abuse. Each zone targets a different traffic pattern.

| Zone | Rate | Burst | Applies To |
|------|------|-------|------------|
| `bot_webhook` | 10 requests/second | -- | Bot webhook endpoints |
| `api_limit` | 2 requests/second | 20 | API endpoints |
| `general` | 30 requests/second | 50 | All other traffic |
| `accessview_api` | 2 requests/minute | 3 | Document processing (expensive) |

The `accessview_api` zone is deliberately aggressive. Document processing is CPU-intensive. Two requests per minute with a burst of 3 prevents a single client from monopolizing server resources.

### Security Headers

The nginx configuration sets security headers that achieve an A+ rating on security scanning tools.

| Header | Value | Purpose |
|--------|-------|---------|
| Strict-Transport-Security | `max-age=31536000; includeSubDomains` | Force HTTPS for one year, including all subdomains |
| X-Frame-Options | `DENY` | Prevent the site from being embedded in iframes (clickjacking protection) |
| X-Content-Type-Options | `nosniff` | Prevent MIME type sniffing |
| Referrer-Policy | `strict-origin-when-cross-origin` | Limit referrer information sent to other origins |
| Permissions-Policy | Restrictive | Disable unused browser APIs (camera, microphone, etc.) |
| Content-Security-Policy | Restrictive | Control which resources the browser can load |

### CORS and Authentication

- **CORS:** Restricted to `https://vanderdev.net` only. No wildcard origins. API requests from other domains are rejected.
- **Bot webhook auth:** Each bot webhook endpoint requires an `X-Bot-Token` header. This token is checked at the nginx level before the request ever reaches n8n. If the token is missing or wrong, nginx returns 403 immediately.

---

## Cloudflare Edge Protection

Cloudflare sits in front of the VPS as the first line of defense. All DNS records for `vanderdev.net` are proxied through Cloudflare, meaning visitors never see the VPS's real IP address.

**Configuration:**

| Setting | Value | Why |
|---------|-------|-----|
| Plan | Free | Sufficient for this traffic level |
| SSL Mode | Full (Strict) | Cloudflare validates the origin certificate. End-to-end encryption. |
| TLS Version | 1.3 minimum | Modern encryption only |
| CAA Record | `letsencrypt.org` | Only Let's Encrypt can issue certificates for this domain |
| Proxied Domains | `vanderdev.net`, `www`, `api`, `n8n` | Traffic flows through Cloudflare |

**Real client IP restoration:** When Cloudflare proxies a request, the source IP is a Cloudflare edge server, not the actual visitor. Nginx is configured to read the `CF-Connecting-IP` header and use that as the real client address. Without this, rate limiting and logging would see all traffic as coming from Cloudflare.

**What Cloudflare provides at the free tier:**

- DDoS protection (automatic)
- Global CDN for static assets
- DNS hosting with fast propagation
- SSL termination at the edge (the visitor connects to Cloudflare over TLS, Cloudflare connects to your origin over TLS)
- Basic WAF rules
- Bot management (challenge suspicious traffic)

---

## CI/CD: GitHub Actions

Deployment is a GitHub Actions workflow that triggers on every push to `main` or on manual dispatch. The entire pipeline takes about 60 seconds.

### The Pipeline

```
Push to main (or manual trigger)
    |
    v
1. Checkout code
2. Setup Node.js 20 (with npm cache)
3. npm ci (install dependencies from lockfile)
4. npm run build (Vite produces dist/)
5. Setup SSH key (from VPS_SSH_KEY secret)
6. rsync -avz --delete dist/ → VPS:/root/vanderdev-website/dist/
    |
    v
nginx serves updated dist/ immediately
```

### Why rsync and Not a Container Rebuild

The website is static files. After `npm run build`, the entire application is a collection of HTML, JavaScript, CSS, and asset files in a `dist/` directory. There is no server process to restart. Nginx serves these files directly. Replacing them with rsync is instant -- the next request from a browser gets the new files.

A container rebuild would mean pulling a Node image, running `npm ci` and `npm run build` inside it, and restarting the container. This takes longer, uses more VPS resources, and introduces a brief downtime window. For a static site, rsync is simpler and better.

### The rsync Flags

```bash
rsync -avz --delete dist/ root@203.0.113.50:/root/vanderdev-website/dist/
```

| Flag | Meaning |
|------|---------|
| `-a` | Archive mode (preserves permissions, timestamps, symlinks) |
| `-v` | Verbose output (visible in GitHub Actions logs) |
| `-z` | Compress during transfer (faster over the network) |
| `--delete` | Remove files on the VPS that no longer exist locally |

The `--delete` flag is important. Without it, old files from previous builds would accumulate on the server. Vite generates hashed filenames (`index-abc123.js`), so old builds leave behind stale files that waste disk space and could cause confusion.

### GitHub Secrets

The pipeline needs one secret:

| Secret | Purpose |
|--------|---------|
| `VPS_SSH_KEY` | Ed25519 private key authorized on the VPS. Used by rsync to connect over SSH. |

This key is stored in the repository's GitHub Actions secrets (Settings > Secrets and variables > Actions). It never appears in logs or in the codebase.

### Hot-Reload Alternative

For quick testing that does not need the CI pipeline:

```bash
ssh vps "cd /root/vanderdev-website && git pull && npm run build"
```

This pulls the latest code directly on the VPS and rebuilds. It is faster than waiting for GitHub Actions but requires the full Node.js toolchain installed on the VPS. Use it for rapid iteration during development. Use the CI pipeline for production deploys.

---

## WordPress Coexistence

The VPS also hosts WordPress (for legacy content). The `.htaccess` file in the `public/` directory manages the boundary between the React SPA and WordPress.

**How it works:**

1. `DirectoryIndex index.html` gives the React SPA priority
2. WordPress paths are excluded from SPA routing:
   - `/wp-admin/` -- WordPress admin panel
   - `/wp-login.php` -- WordPress login page
   - `/wp-json/` -- WordPress REST API
3. All other non-file requests rewrite to `index.html` (React Router handles them)
4. `Cache-Control: no-cache` on `index.html` ensures browsers always fetch the latest version after a deploy

**Why this matters:** Without the WordPress exclusions, React Router would intercept WordPress URLs and show a blank page or a 404. The `.htaccess` rules ensure WordPress paths bypass the SPA entirely and hit Apache/WordPress directly.

---

## Reproduction Steps

You do not need 29 containers to get a production website running. Start with the essentials and add services as your needs grow.

### 1. Provision a VPS

Any VPS provider works. The requirements:

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| OS | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| RAM | 4 GB | 8 GB (for the full container stack) |
| Storage | 40 GB | 80 GB (Supabase and monitoring data accumulate) |
| CPU | 2 vCPUs | 4 vCPUs |

### 2. Install Docker and Docker Compose

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Verify
docker --version
docker compose version
```

Docker Compose V2 is included with modern Docker installations. You should not need to install it separately.

### 3. Set Up nginx-proxy with Auto SSL

The nginx-proxy ecosystem handles reverse proxying and certificate management automatically.

```bash
# Create a Docker network for proxy communication
docker network create nginx-proxy

# Start nginx-proxy
docker run -d \
  --name nginx-proxy \
  --net nginx-proxy \
  -p 80:80 -p 443:443 \
  -v /var/run/docker.sock:/tmp/docker.sock:ro \
  -v certs:/etc/nginx/certs \
  -v vhost:/etc/nginx/vhost.d \
  -v html:/usr/share/nginx/html \
  nginxproxy/nginx-proxy

# Start the ACME companion (auto SSL)
docker run -d \
  --name nginx-proxy-acme \
  --net nginx-proxy \
  --volumes-from nginx-proxy \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v acme:/etc/acme.sh \
  -e DEFAULT_EMAIL=your@email.com \
  nginxproxy/acme-companion
```

Now any container with `VIRTUAL_HOST` and `LETSENCRYPT_HOST` environment variables will automatically get a reverse proxy entry and an SSL certificate.

### 4. Deploy the React App

```bash
# Clone your repo
git clone https://github.com/your-org/your-site.git /root/your-site
cd /root/your-site

# Install and build
npm ci
npm run build

# Run the website container
docker run -d \
  --name website \
  --net nginx-proxy \
  -e VIRTUAL_HOST=yourdomain.com \
  -e LETSENCRYPT_HOST=yourdomain.com \
  -v /root/your-site/dist:/usr/share/nginx/html:ro \
  nginx:alpine
```

The ACME companion will detect the new container, request a Let's Encrypt certificate, and configure nginx-proxy to route traffic to it. Give it 2-3 minutes on first deploy.

### 5. Set Up Cloudflare DNS

1. Create a free Cloudflare account
2. Add your domain and follow the nameserver migration instructions
3. Create A records pointing to your VPS IP:

| Type | Name | Content | Proxy |
|------|------|---------|-------|
| A | `@` | Your VPS IP | Proxied (orange cloud) |
| A | `www` | Your VPS IP | Proxied |
| A | `api` | Your VPS IP | Proxied |
| CAA | `@` | `0 issue "letsencrypt.org"` | -- |

4. Set SSL/TLS mode to **Full (Strict)**

### 6. Add a CI/CD Pipeline

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npm run build
      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.VPS_SSH_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H your-vps-ip >> ~/.ssh/known_hosts
      - name: Deploy
        run: rsync -avz --delete dist/ root@your-vps-ip:/root/your-site/dist/
```

Add the `VPS_SSH_KEY` secret in your repository settings.

### 7. Add Services Incrementally

Do not deploy 29 containers on day one. Add services in this order, verifying each one works before moving on:

| Order | Service | Why Now |
|-------|---------|---------|
| 1 | Website + nginx-proxy + ACME | The core: a working production site with SSL |
| 2 | Supabase | Database + pgvector for chatbot RAG |
| 3 | n8n | Workflow automation for bot orchestration |
| 4 | Prometheus + Grafana | Monitoring (you need visibility before adding more) |
| 5 | Alertmanager + Uptime Kuma | Alerting (now you know when things break) |
| 6 | Redis, data collectors, utilities | Supporting services for specific features |

### 8. Configure Rate Limiting and Security Headers

Add a custom nginx configuration file for your virtual host:

```bash
# Create custom config (nginx-proxy reads vhost.d/ automatically)
cat > /path/to/vhost.d/yourdomain.com << 'EOF'
# Rate limiting zones
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=2r/s;
limit_req_zone $binary_remote_addr zone=general:10m rate=30r/s;

# Security headers
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Frame-Options "DENY" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Apply rate limiting
limit_req zone=general burst=50 nodelay;
EOF
```

### 9. Verify

After deployment, check these things:

```bash
# SSL certificate is valid
curl -vI https://yourdomain.com 2>&1 | grep "SSL certificate"

# SPA routing works (all paths return index.html)
curl -s -o /dev/null -w "%{http_code}" https://yourdomain.com/waterbot
# Should return 200

# Security headers are present
curl -I https://yourdomain.com 2>&1 | grep -i "strict-transport"

# API subdomain routes correctly
curl https://api.yourdomain.com/health

# Containers are running
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

---

## Gotchas

These are the things that will waste your time if you do not know about them upfront.

| Issue | Details | Fix |
|-------|---------|-----|
| rsync `--delete` replaces everything | The entire `dist/` directory is replaced on deploy. There are no partial updates or rollbacks built in. | If you need rollbacks, keep the previous `dist/` as `dist.bak/` before deploying. A one-liner in the CI script handles this. |
| Supabase containers are memory-hungry | Kong (the API gateway) routinely runs at 70-75% of its 512 MB limit. Studio is similarly heavy. | Monitor container memory with `docker stats`. If you hit OOM kills, increase the limit or reduce the Supabase components you run. |
| Container memory limits are mandatory | Without explicit limits, one container can allocate all available memory and trigger an OOM kill cascade that crashes every container on the VPS. | Set `mem_limit` on every container in your `docker-compose.yml`. No exceptions. |
| ACME needs port 80 open | The Let's Encrypt validation process uses HTTP-01 challenges over port 80. If Cloudflare's "Always Use HTTPS" redirect is enabled, the challenge will fail. | In Cloudflare, create a Page Rule that disables "Always Use HTTPS" for `http://yourdomain.com/.well-known/acme-challenge/*`. Or use DNS-01 challenges instead. |
| Uptime Kuma port 3001 is publicly bound | By default, Uptime Kuma binds to `0.0.0.0:3001`, making it accessible from the internet. | Bind it to `127.0.0.1:3001` in Docker or restrict access to your Tailscale network. It does not need to be public. |
| WordPress path conflicts | If WordPress and the React SPA share the same document root, Apache/nginx routing rules can conflict. | Keep the `.htaccess` exclusions for WordPress paths (`/wp-admin/`, `/wp-login.php`, `/wp-json/`) and test after every deploy. |
| `Cache-Control: no-cache` on index.html | Without this, browsers may cache the old `index.html` and load stale JavaScript bundles after a deploy. Users see a broken site until they hard-refresh. | Set `no-cache` on `index.html` specifically. Vite's hashed filenames handle cache-busting for all other assets. |
| CF-Connecting-IP restoration | If you do not configure nginx to trust Cloudflare IPs and read the `CF-Connecting-IP` header, all traffic appears to come from Cloudflare edge servers. Rate limiting, logging, and ban lists become useless. | Use the `set_real_ip_from` directive in nginx with Cloudflare's published IP ranges, and `real_ip_header CF-Connecting-IP`. |
| SSH key in GitHub Actions | The `VPS_SSH_KEY` secret must be the private key, not the public key. It must have a trailing newline. GitHub Actions is particular about key formatting. | Paste the key including the `-----BEGIN` and `-----END` lines. Add a newline at the end. Test locally with `ssh -i keyfile root@vps-ip` before adding it to GitHub. |

## Step-by-Step Build Order

This is the capstone chapter -- a linear checklist for reproducing the entire infrastructure from scratch. Every phase references concepts from earlier chapters, but this is the one document you follow start to finish. Work through it in order. Each phase builds on the previous one.

---

## Prerequisites

Before you start, gather these accounts, hardware, and credentials.

### Hardware Requirements

| Tier | Machines | Notes |
|------|----------|-------|
| **Minimum Viable** | 1 Mac + 1 VPS | Enough for Claude Code + production deploy |
| **Comfortable** | 2 Macs + 1 VPS | Separate dev machines, full skill ecosystem |
| **Full Reproduction** | 2 Macs + 1 Mac mini server + 1 VPS + 2 Raspberry Pis + 1 phone | Everything described in this guide (7 machines) |

### Accounts

- **GitHub** -- free account, used for repos, CI/CD, and GitHub Actions
- **Cloudflare** -- free tier covers DNS, proxy, SSL, and basic WAF
- **VPS provider** -- Hostinger, Hetzner, DigitalOcean, or similar (Ubuntu 24.04 LTS)
- **Tailscale** -- free tier supports up to 100 devices
- **Google account** -- for Google Drive offsite backup (15GB free, 100GB for $3/mo)
- **Domain** -- one domain (e.g., `yourdomain.net`), roughly $12/year

### Software

- **Claude Code subscription** -- Pro ($20/mo) minimum. Max 20x ($200/mo) recommended for heavy use. See Appendix C for cost breakdown.
- **Docker** -- OrbStack recommended on macOS, standard Docker on Linux
- **Node.js 20+** -- required for MCP servers and skills

---

## Phase 1: Network Foundation (Day 1)

Goal: every machine can reach every other machine, at home and remotely.

- [ ] **1.1** Set up your router. Omada gives you VLANs and static DHCP; consumer-grade works fine if you assign static IPs manually.
- [ ] **1.2** Assign static LAN IPs to every machine. Pick a scheme (e.g., `.100` = server, `.101-.102` = dev Macs, `.112-.113` = Pis) and stick with it.
- [ ] **1.3** Install Tailscale on all devices. Note each machine's Tailscale IP. Enable MagicDNS if you want hostname-based access.
- [ ] **1.4** Generate Ed25519 SSH keys on each machine:
  ```bash
  ssh-keygen -t ed25519 -C "yourname@machinename"
  ```
- [ ] **1.5** Distribute public keys to every machine you need to reach. Use `ssh-copy-id` or manually append to `~/.ssh/authorized_keys`.
- [ ] **1.6** Create your SSH config with smart routing. The pattern: use LAN IPs when at home, Tailscale IPs when remote. See Appendix B for the `is-home.sh` template.
  ```
  Match host "server" exec "~/.ssh/is-home.sh"
      HostName ServerName.local
  Match host "server"
      HostName <tailscale-ip>
  ```
- [ ] **1.7** Test SSH to every machine from every other machine -- both on LAN and via Tailscale.
- [ ] **1.8** (Optional) Set up Pi-hole DNS on 1-2 Raspberry Pis. Primary Pi handles DNS + dashboard; secondary Pi provides redundancy + push notifications. Configure your router to hand out the Pi-hole IPs as DNS servers via DHCP.

---

## Phase 2: Home Server Setup (Day 1-2)

Goal: a headless always-on machine running MCP servers, backups, and monitoring.

- [ ] **2.1** Set up your Mac mini (or similar always-on machine) as a headless server. Enable Remote Login (SSH) in System Settings. Disable sleep: `sudo pmset -a sleep 0 displaysleep 0`.
- [ ] **2.2** Attach external storage for backups. A 4TB SSD over USB-C or Thunderbolt is recommended. Format as APFS or ext4 depending on OS.
- [ ] **2.3** Install Docker. On macOS, OrbStack is lighter than Docker Desktop and handles containers plus Linux VMs. On Linux, install Docker Engine directly.
- [ ] **2.4** Set up MCP servers. At minimum, deploy:

  | MCP Server | Purpose | Recommended Port |
  |------------|---------|-----------------|
  | Memory | Persistent knowledge graph across sessions | 8083 |
  | GitHub | GitHub operations from Claude Code | 8088 |
  | Filesystem | Local file access for Claude Code | stdio (local) |

  Optional but valuable:

  | MCP Server | Purpose | Recommended Port |
  |------------|---------|-----------------|
  | Perplexity | Web research from Claude Code | 8084 |
  | n8n MCP | Workflow automation access | 8081 |
  | LLM Gateway | Multi-model routing | 3333 |
  | Google Sheets | Spreadsheet access | 8089 |
  | Google Drive | Drive file access | 8087 |
  | Docling | Document conversion | 8085 |

- [ ] **2.5** Expose MCP servers as SSE endpoints using `supergateway` or `mcp-proxy`. This lets all dev machines connect to one central set of MCP servers.
- [ ] **2.6** Create LaunchAgents (macOS) or systemd units (Linux) for MCP server persistence. Every MCP server should restart automatically on boot and after crashes.
- [ ] **2.7** Test MCP connectivity from your dev machine. Run Claude Code, check that `mcp__memory__search_nodes` and similar tools appear in the tool list.

---

## Phase 3: VPS Provisioning (Day 2)

Goal: a production server with HTTPS, monitoring, and hardening.

- [ ] **3.1** Provision a VPS. Ubuntu 24.04 LTS, 4+ vCPU, 16GB RAM recommended. Smaller works for simple sites; larger if you run Supabase or multiple services.
- [ ] **3.2** Install Docker and Docker Compose on the VPS:
  ```bash
  curl -fsSL https://get.docker.com | sh
  ```
- [ ] **3.3** Set up nginx-proxy + ACME companion for automatic HTTPS:
  ```bash
  # nginx-proxy handles reverse proxying by container env vars
  # acme-companion handles Let's Encrypt cert issuance and renewal
  ```
- [ ] **3.4** Configure Cloudflare DNS. Create an A record pointing your domain to the VPS IP. Create a CNAME for `www` pointing to the apex domain.
- [ ] **3.5** Enable Cloudflare proxy (orange cloud icon) and set SSL mode to Full (Strict). This gives you Cloudflare's WAF, DDoS protection, and edge caching for free.
- [ ] **3.6** (Optional) Deploy Supabase stack for PostgreSQL + pgvector. Useful if your bots or apps need a database with vector search.
- [ ] **3.7** (Optional) Deploy n8n for workflow automation. Pairs well with bot webhooks and scheduled tasks.
- [ ] **3.8** Set up monitoring:

  | Component | Purpose |
  |-----------|---------|
  | Prometheus | Metrics collection |
  | Grafana | Dashboards and visualization |
  | Alertmanager | Alert routing to ntfy |
  | node_exporter | Host-level metrics |

- [ ] **3.9** Harden the VPS:
  - Create swap (2-4GB) for memory pressure safety
  - Set Docker memory limits on every container
  - Install and configure fail2ban
  - Add nginx rate limiting (per-IP and per-endpoint zones)
  - Add security headers (X-Frame-Options, CSP, HSTS)
  - Disable password SSH auth, root login via password
- [ ] **3.10** Install Tailscale on the VPS so it joins your mesh network.

---

## Phase 4: Dev Machine Setup (Day 2-3)

Goal: Claude Code fully configured with MCP tools, skills, and memory.

- [ ] **4.1** Install Claude Code. Requires an active subscription (Pro or Max).
  ```bash
  npm install -g @anthropic-ai/claude-code
  ```
- [ ] **4.2** Create a dotfiles repo. This is the backbone of your configuration. Structure:
  ```
  dotfiles/
  ├── .claude/
  │   ├── commands/       # Skills (slash commands)
  │   ├── docs/           # Reference docs for CLAUDE.md
  │   └── scripts/        # Helper scripts for skills
  ├── backup/             # Backup scripts
  ├── scripts/            # Infrastructure scripts
  ├── setup.sh            # Symlink installer
  └── CLAUDE.md           # Global instructions
  ```
- [ ] **4.3** Run `setup.sh` to install symlinks and LaunchAgents. The script should symlink `~/.claude/commands` to your dotfiles commands, not copy files.
- [ ] **4.4** Configure `~/.claude.json` pointing to your MCP servers. See Appendix B for the template. Replace `<server-ip>` with your home server's LAN IP.
- [ ] **4.5** Create your first skill. Start with something you do frequently -- an audit script, a deploy command, or a research template. Place it in `dotfiles/.claude/commands/your-skill.md`.
- [ ] **4.6** Set up auto-memory. Create `~/.claude/projects/<project>/memory/MEMORY.md` for your main projects. Claude Code will read this automatically at session start.
- [ ] **4.7** Test the full loop: start a Claude Code session, verify MCP tools appear, run a skill, confirm memory writes persist.

---

## Phase 5: Production Deploy (Day 3)

Goal: a live website deployed via CI/CD.

- [ ] **5.1** Create your web application. React + Vite is a solid default -- fast builds, easy to deploy as static files behind nginx.
- [ ] **5.2** Set up GitHub Actions CI/CD. The pattern:
  ```yaml
  # On push to main:
  # 1. npm ci && npm run build
  # 2. rsync dist/ to VPS via SSH
  # 3. nginx serves the dist/ directory
  ```
- [ ] **5.3** Add your VPS SSH key as a GitHub Actions secret. Generate a deploy-only key pair for this purpose -- do not reuse your personal key.
- [ ] **5.4** Push a commit and verify the site is live at your domain.
- [ ] **5.5** (Optional) Create bot knowledge bases. Ingest markdown files into pgvector via OpenAI embeddings for RAG-powered chatbots.
- [ ] **5.6** (Optional) Set up n8n webhook workflows for bots. n8n receives the chat request, queries the vector DB, and returns a response.

---

## Phase 6: Automation (Day 3-4)

Goal: backups run automatically, failures alert you, nothing requires daily attention.

- [ ] **6.1** Create backup scripts. The pattern: each script handles one machine, logs to CSV, and calls a shared alert function on failure. Example: `mac-backup.sh` backs up the local Mac, `vps-backup.sh` pulls VPS data, `pi-backup.sh` pulls Pi configs.
- [ ] **6.2** Create LaunchAgent plists (macOS) or systemd timers (Linux) for each backup script. Schedule them staggered (e.g., 2:00 AM, 2:30 AM, 3:00 AM) to avoid resource contention.
- [ ] **6.3** Set `pmset` wake schedules on all Macs so they wake for backups:
  ```bash
  sudo pmset repeat wakeorpoweron MTWRFSU 01:55:00
  ```
- [ ] **6.4** Set up offsite backup using rclone to Google Drive:
  ```bash
  rclone sync /path/to/backups gdrive:/BACKUPS/ --transfers 4
  ```
- [ ] **6.5** Set up ntfy.sh for push notifications. Self-host on a Pi or use the public instance. Configure your backup scripts to POST to ntfy on failure.
- [ ] **6.6** Create a `backup-health-check.sh` that runs weekly, summarizes backup status across all machines, and sends a digest notification. Silence is healthy -- you only get noise when something breaks.

---

## Phase 7: Validation (Day 4)

Goal: prove everything works end to end.

- [ ] **7.1** Test backup chain: manually trigger a backup, verify the archive lands on the server, verify it syncs offsite to Google Drive.
- [ ] **7.2** Test DNS failover: stop the primary Pi-hole, verify the secondary takes over and DNS still resolves.
- [ ] **7.3** Test monitoring: trigger a Prometheus alert (e.g., stop node_exporter briefly), verify the alert fires and you get an ntfy notification.
- [ ] **7.4** Test CI/CD: push a trivial change to your repo, verify GitHub Actions runs and the change appears on your live site.
- [ ] **7.5** Run your audit skill from Claude Code. It should report all systems green.
- [ ] **7.6** (Optional) Run evaluation against your chatbots -- send test queries, verify RAG retrieval quality.

---

## What to Do First (Minimum Viable)

If you want the fastest path to "Claude Code with a production target," skip everything optional and do this:

| Step | Time | What You Get |
|------|------|-------------|
| Install Claude Code + create `CLAUDE.md` | 30 min | AI-powered dev workflow |
| Provision VPS + deploy nginx + configure Cloudflare | 2 hours | HTTPS production server |
| Deploy a React app via GitHub Actions | 1 hour | Automated CI/CD pipeline |
| Create 2-3 skills for your workflows | 1 hour | Custom slash commands |
| **Total** | **~4.5 hours** | **Full AI-powered dev + production deploy** |

You can layer in MCP servers, backups, monitoring, Pis, and home automation later. The core loop -- Claude Code writing code, GitHub Actions deploying it, nginx serving it -- works with just a laptop and a VPS.

---

## Build Order Dependency Graph

For visual thinkers, here is what depends on what:

```
Phase 1 (Network)
  └─> Phase 2 (Home Server) ─────> Phase 4 (Dev Machine)
  └─> Phase 3 (VPS) ─────────────> Phase 5 (Production)
                                      │
Phase 4 + Phase 5 ──────────────> Phase 6 (Automation)
                                      │
Phase 6 ──────────────────────────> Phase 7 (Validation)
```

Phases 2 and 3 can run in parallel. Phase 4 needs Phase 2 (for MCP servers). Phase 5 needs Phase 3 (for the VPS). Everything else is sequential.

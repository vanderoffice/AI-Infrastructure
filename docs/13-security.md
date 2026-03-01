# Hardening and Access Control

## Why This Exists

Security is the first priority in the entire infrastructure. It comes before research, before action, before documentation. That ordering is not aspirational -- it is enforced in CLAUDE.md as a hard rule.

The attack surface is real. A public-facing VPS runs 29 Docker containers, serves a production website, and accepts SSH connections from multiple machines. A home server hosts databases, API keys, and workflow engines. Two development machines have full filesystem access and shell execution through Claude Code. If any of these are compromised, the blast radius covers everything from production services to personal credentials.

This chapter documents every hardening measure in the stack -- from SSH key configuration to container memory limits to Cloudflare edge protection. The goal is defense in depth: multiple independent layers so that a failure in any one layer does not compromise the system.

---

## SSH Security

SSH is the primary access method between machines. Every machine in the fleet is reachable via SSH, and the configuration is locked down to eliminate the most common attack vectors.

### Key Configuration

| Setting | Value | Why |
|---------|-------|-----|
| Key type | Ed25519 | Modern elliptic curve. Faster and more secure than RSA. Shorter keys (256-bit vs RSA 4096-bit) with equivalent or better security. |
| IdentitiesOnly | yes | Prevents the SSH agent from offering every loaded key to every server. Only the key specified in the config is tried. |
| AddKeysToAgent | yes | The key is added to the SSH agent on first use. You enter the passphrase once per session. |
| Private key permissions | 600 | Owner read/write only. SSH refuses to use keys with looser permissions. |
| Public key permissions | 644 | World-readable is fine -- it is the public half. |
| ~/.ssh/ permissions | 700 | Owner access only for the directory. |

**One key across all machines.** The infrastructure uses a single Ed25519 key shared across StudioM4, OfficeM4P, and ServerM2P. This is a deliberate simplicity choice. For a personal infrastructure with a small number of machines, managing separate key pairs per machine adds complexity without meaningful security benefit. The private key is protected by a passphrase and the SSH agent.

**No passwords.** Password authentication is disabled on every machine. `PasswordAuthentication no` in `/etc/ssh/sshd_config`. If you do not have the Ed25519 key, you do not get in. This eliminates brute-force password attacks entirely.

**No RSA.** RSA keys are not used. Ed25519 is the standard for new key generation. If you encounter an RSA key in the setup, it is legacy and should be replaced.

### Generating Your Keys

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

This produces two files: `~/.ssh/id_ed25519` (private, never share) and `~/.ssh/id_ed25519.pub` (public, copy to servers). Copy the public key to each machine:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@hostname
```

Then disable password authentication on the remote machine by editing `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart SSH: `sudo systemctl restart sshd`

---

## VPS Hardening

The VPS is the only machine directly exposed to the public internet. It received a systematic hardening process that covers memory management, rate limiting, intrusion prevention, and HTTP security headers.

### Swap and Memory Management

| Setting | Value | Purpose |
|---------|-------|---------|
| Swap file | 4GB | Prevents OOM (out-of-memory) kills |
| vm.swappiness | 10 | Strongly prefers RAM over swap (only swaps under pressure) |

Without swap, a memory spike in any container can trigger the Linux OOM killer, which terminates processes unpredictably. The 4GB swap file is a safety net -- the system should rarely touch it with `swappiness=10`, but when it does, it prevents hard crashes.

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p
```

### Container Memory Limits

Every Docker container has a memory ceiling. Without limits, a single runaway container can consume all available RAM and take down every other service on the machine.

| Tier | Memory Limit | Containers |
|------|-------------|------------|
| Heavy | 2GB | n8n, supabase-db |
| Medium | 512MB | grafana, kong, studio, pgadmin |
| Light | 256MB | influxdb, vps-api, ecos-form |
| Minimal | 128MB | node-exporter, health-monitor |

These limits are set in `docker-compose.yml` using the deploy section:

```yaml
services:
  n8n:
    deploy:
      resources:
        limits:
          memory: 2G
```

**Note:** The `deploy.resources.limits` syntax requires Docker Compose v2 or later. If you are using the legacy `docker-compose` (with the hyphen), you need to upgrade to the `docker compose` plugin.

### Nginx Rate Limiting

Rate limiting prevents abuse by capping how many requests a client can make per second. Four zones cover different parts of the application:

| Zone | Rate | Burst | Purpose |
|------|------|-------|---------|
| bot_webhook | 10 req/s | -- | Bot webhook endpoints. Higher rate because webhooks are machine-to-machine. |
| api_limit | 2 req/s | 20 | API endpoints. Low sustained rate with burst allowance for legitimate spikes. |
| general | 30 req/s | 50 | General web traffic. Generous for normal browsing. |
| accessview_api | 2 req/min | 3 | Document processing. Intentionally very low -- these are expensive operations. |

Rate limiting is configured in the nginx `http` block with `limit_req_zone` directives, then applied to specific `location` blocks with `limit_req`.

### fail2ban

fail2ban watches log files for patterns that indicate attacks and automatically bans offending IP addresses using firewall rules.

| Jail | What It Watches | What It Blocks |
|------|----------------|----------------|
| sshd | `/var/log/auth.log` | SSH brute-force attempts (245 historical bans, 1413 failed attempts) |
| nginx-http-auth | nginx error log | Failed HTTP authentication attempts |
| nginx-botsearch | nginx access log | Automated scanners probing for common vulnerability paths (/wp-admin, /phpmyadmin, etc.) |
| nginx-limit-req | nginx error log | Clients exceeding rate limits |

The sshd jail is the most active. The VPS sees constant automated SSH login attempts from botnets. With password authentication disabled and fail2ban active, these attempts are automatically blocked after a few failures. The 245 historical bans and 1413 failed attempts are normal for any public-facing server -- it is background noise that fail2ban handles automatically.

```bash
sudo apt install fail2ban
sudo systemctl enable fail2ban
```

Configure jails in `/etc/fail2ban/jail.local` (do not edit `jail.conf` directly -- it gets overwritten on updates).

### Security Headers

The VPS serves HTTP responses with security headers that instruct browsers to enforce additional protections. These headers earned an A+ rating on security scanning tools.

| Header | Value | What It Does |
|--------|-------|--------------|
| Strict-Transport-Security | max-age=31536000; includeSubDomains | Forces HTTPS for one year. Browsers will not even attempt HTTP. |
| X-Frame-Options | DENY | Prevents the site from being embedded in iframes (clickjacking protection). |
| X-Content-Type-Options | nosniff | Prevents browsers from guessing MIME types (type confusion attacks). |
| Referrer-Policy | strict-origin-when-cross-origin | Limits referrer information sent to external sites. |
| Permissions-Policy | (restrictive) | Disables browser features the site does not use (camera, microphone, geolocation). |
| Content-Security-Policy | (configured) | Restricts where scripts, styles, and resources can load from. |
| X-Permitted-Cross-Domain-Policies | none | Blocks Flash and Acrobat from loading cross-domain content. |

These headers are added in the nginx server block. They cost nothing in performance and close entire categories of client-side attacks.

---

## Cloudflare Edge Protection

Cloudflare sits between the public internet and the VPS. All traffic to vanderdev.net passes through Cloudflare first, which provides DDoS protection, TLS termination, and caching -- all on the free tier.

| Setting | Value |
|---------|-------|
| SSL mode | Full (Strict) |
| TLS version | 1.3 |
| CAA DNS record | Only letsencrypt.org can issue certificates |
| Proxied domains | vanderdev.net, www, api, n8n |

### How Full (Strict) SSL Works

```
Browser --> Cloudflare (TLS 1.3) --> VPS (valid Let's Encrypt cert)
```

Full (Strict) means Cloudflare encrypts traffic in both directions AND validates the origin server's certificate. The VPS has a real Let's Encrypt certificate, not a self-signed one. This prevents man-in-the-middle attacks between Cloudflare and the origin.

### CAA Record

A CAA (Certificate Authority Authorization) DNS record tells the world that only Let's Encrypt is authorized to issue certificates for vanderdev.net. If an attacker compromises a different CA and tries to issue a fraudulent certificate for your domain, the CAA record instructs that CA to refuse.

### Real IP Restoration

When traffic arrives at nginx through Cloudflare, the source IP address is Cloudflare's, not the actual client's. This breaks fail2ban (it would ban Cloudflare instead of the attacker) and logging.

The fix: nginx uses the `CF-Connecting-IP` header to restore the real client IP. The configuration includes 22 Cloudflare CIDR ranges in the `set_real_ip_from` directives, telling nginx to trust the `CF-Connecting-IP` header from those ranges.

**This is critical for fail2ban.** Without real IP restoration, fail2ban sees all traffic as coming from Cloudflare and either bans Cloudflare (breaking your site) or does nothing useful.

---

## Credential Management

API keys and secrets are a prime target. The infrastructure uses an SSE proxy pattern that keeps keys off development machines entirely.

### The SSE Proxy Pattern

```
Dev machine (StudioM4/OfficeM4P)
  |
  SSE connection (no API key)
  |
ServerM2P (MCP servers with API keys in .env files)
  |
  Authenticated API calls
  |
External service (Perplexity, Anthropic, etc.)
```

Development machines connect to MCP servers on ServerM2P via SSE (Server-Sent Events). The MCP servers hold the API keys in `.env` files with `600` permissions (owner read/write only). The development machines never see the keys -- they send requests to the MCP server, which adds authentication and forwards them.

**What this means in practice:**

- `~/.claude.json` on dev machines contains SSE URLs, never API keys
- If a dev machine is compromised, no API keys are exposed
- Key rotation happens in one place (ServerM2P), not on every machine
- `.env` files on ServerM2P are readable only by root

### Bot Webhook Tokens

Bot webhook tokens are stored at `/root/.bot-webhook-token` on the VPS with restrictive permissions. These tokens authenticate incoming webhook requests from messaging platforms. They are not in version control and not in any configuration file that leaves the VPS.

**Current gap:** There is no automated rotation for bot webhook tokens. Rotation is manual. This is a known area for improvement.

---

## Vaultwarden -- Self-Hosted Password Manager

Vaultwarden is a lightweight, self-hosted implementation of the Bitwarden password manager API. It runs on ServerM2P and is accessible only from the local network.

| Property | Value |
|----------|-------|
| Host | ServerM2P (192.168.0.100) |
| Reverse proxy | Caddy |
| URL | https://vault.vanderdev.local |
| Public exposure | None -- internal network only |
| TLS | Internal certificate via Caddy |

The `vault.vanderdev.local` hostname resolves via Pi-hole's local DNS (covered in the DNS chapter). This means Vaultwarden is accessible from any device on the network by name, but it is completely invisible to the public internet. No port forwarding, no Cloudflare proxy, no public DNS record.

---

## Claude Code Security Model

Claude Code has full access to the filesystem and shell. That is what makes it powerful. The security boundary is defined in `settings.json`, which creates two lists: things Claude Code can do automatically, and things it can never do.

### Allowed (Auto-Approved)

These tools run without asking for permission:

| Tool | What It Does |
|------|-------------|
| Read, Write, Edit | Full filesystem access (read, create, modify files) |
| Glob, Grep | File search and content search |
| Bash | Shell command execution |
| All MCP tools | External service access via MCP servers |
| WebFetch | HTTP requests to 40+ whitelisted domains |

### Denied (Absolute Block)

These commands are blocked absolutely. Claude Code cannot run them even if explicitly asked.

| Category | Blocked Commands |
|----------|-----------------|
| Destructive | `rm -rf`, `sudo`, `dd`, `mkfs`, `shutdown`, `reboot` |
| System | `csrutil`, `nvram`, `visudo`, `passwd` |
| Secrets | Direct access to `.env` files, SSH private keys, AWS/GCloud credential files |

The deny list is not a suggestion -- it is enforced at the tool level. If Claude Code tries to run `rm -rf /`, the command is rejected before it reaches the shell. There is no override, no "are you sure?" prompt, no way to bypass it.

### Hook Logging

A `PreToolUse` hook logs all non-standard Bash commands to a JSONL file. This creates an audit trail of everything Claude Code executes. If something goes wrong, you can review the log to see exactly what commands were run, in what order, and what their outputs were.

---

## Access Matrix

Who can reach what, and how.

| From | To | Method | Notes |
|------|----|--------|-------|
| StudioM4 / OfficeM4P | All machines | SSH (Ed25519) | LAN or Tailscale |
| VPS | ServerM2P | Tailscale + SSH | Encrypted mesh, no port forwarding |
| S23+ (Android) | All machines | Tailscale | Mobile access from anywhere |
| External (internet) | VPS only | HTTPS via Cloudflare | Ports 80/443 only, everything else blocked |
| External (internet) | Internal machines | Blocked | No port forwarding, no public IPs |

**Key principle:** Internal machines are never directly reachable from the internet. The VPS is the only public-facing machine, and it only exposes ports 80 and 443 through Cloudflare. If you need to reach a home machine from outside, you use Tailscale, which creates an encrypted WireGuard tunnel without opening any ports on your router.

---

## Prometheus Alerting

Eight alert rules monitor the infrastructure and fire notifications when things go wrong. Alerts flow through a pipeline: Prometheus evaluates rules, fires alerts to Alertmanager, and Alertmanager sends push notifications via ntfy.

```
Prometheus (evaluates rules)
  |
Alertmanager (deduplicates, routes)
  |
ntfy.sh (push notification to phone)
  topic: vander-infra
```

### Alert Rules

| Alert | Condition | Severity |
|-------|-----------|----------|
| High CPU | CPU usage > threshold for sustained period | Warning |
| Disk space | Filesystem usage above threshold | Critical |
| Container restarts | Container restarting repeatedly | Warning |
| Memory pressure | RAM usage above threshold | Warning |
| Service down | Monitored service unreachable | Critical |
| SSL cert expiry | Certificate expiring within N days | Warning |
| Backup staleness | Backup not completed within expected window | Critical |
| Response time | HTTP response time above threshold | Warning |

**Design philosophy:** Silence equals healthy. You do not get daily "everything is fine" messages. Alerts only fire when something is wrong. This prevents alert fatigue -- when your phone buzzes, it means something needs attention.

---

## Reproduction Steps

### Step 1: SSH Keys

Generate an Ed25519 key pair on your primary development machine:

```bash
ssh-keygen -t ed25519 -C "your-email@example.com"
```

Copy the public key to every machine you need to access:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@hostname
```

### Step 2: Disable Password Authentication

On every machine, edit `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PubkeyAuthentication yes
```

Restart SSH. **Test that key-based login works before disconnecting your current session.** If you lock yourself out, you will need physical access or a console to recover.

### Step 3: fail2ban on the VPS

```bash
sudo apt install fail2ban
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Edit `/etc/fail2ban/jail.local` to enable the sshd, nginx-http-auth, nginx-botsearch, and nginx-limit-req jails. Set ban times and retry limits appropriate for your risk tolerance.

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### Step 4: Swap File

```bash
fallocate -l 4G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
echo 'vm.swappiness=10' >> /etc/sysctl.conf
sysctl -p
```

### Step 5: Container Memory Limits

Add memory limits to every service in your `docker-compose.yml`:

```yaml
deploy:
  resources:
    limits:
      memory: 512M
```

Start with generous limits and tighten based on actual usage. Use `docker stats` to see real-time memory consumption per container.

### Step 6: Nginx Rate Limiting and Security Headers

Add rate limit zones to the nginx `http` block and apply them to specific locations. Add security headers to the `server` block. Test with `curl -I https://yourdomain.com` to verify headers are present.

### Step 7: Cloudflare

Sign up for the free tier. Add your domain and update your registrar's nameservers to Cloudflare's. Enable Full (Strict) SSL mode. Add a CAA record allowing only your certificate authority (letsencrypt.org). Proxy your DNS records (orange cloud icon).

### Step 8: Vaultwarden (Optional)

Deploy Vaultwarden as a Docker container behind a reverse proxy (Caddy or nginx). Configure a local DNS record in Pi-hole to point a `.local` hostname to the server. Do not expose Vaultwarden to the public internet.

### Step 9: Claude Code Deny List

In your Claude Code `settings.json`, define the deny list of commands that should never be executed. This is your safety net against destructive operations.

### Step 10: Prometheus Alerting

Deploy Prometheus and Alertmanager as Docker containers. Define alert rules for the metrics that matter most to you (disk, CPU, memory, service health, cert expiry). Configure Alertmanager to send notifications to ntfy or your preferred notification service.

---

## Gotchas and Common Issues

**Container memory limits require Compose v2+.** The `deploy.resources.limits.memory` syntax is a Docker Compose v2 feature. If you are using the legacy `docker-compose` binary (with the hyphen), upgrade to the `docker compose` CLI plugin. The old binary does not support deploy sections outside of Docker Swarm mode.

**fail2ban with Cloudflare needs real IP restoration.** By default, fail2ban sees Cloudflare's IP addresses, not the actual client IPs. You must configure nginx to restore the real IP using `set_real_ip_from` directives for Cloudflare's CIDR ranges and the `CF-Connecting-IP` header. Without this, fail2ban either bans Cloudflare (taking your site offline) or is completely ineffective.

**Cloudflare caches aggressively.** Static assets are cached at the edge by default, which is usually what you want. But `index.html` should never be cached -- if it is, users will not see new deployments. Set `Cache-Control: no-cache` on `index.html` in your nginx configuration.

**Let's Encrypt validation needs port 80.** Let's Encrypt uses HTTP-01 challenges on port 80 to validate domain ownership. If you block port 80 entirely, certificate renewal will fail. Keep port 80 open but redirect all traffic to HTTPS. Cloudflare handles this gracefully in proxy mode.

**The deny list is absolute.** Commands in the Claude Code deny list cannot be overridden by the user, by CLAUDE.md instructions, or by any other mechanism. This is by design. If you need to run a denied command, you do it manually in a separate terminal. The deny list is your last line of defense against accidental destruction.

**Bot webhook tokens are not auto-rotated.** Webhook tokens on the VPS are static until manually changed. This is a known gap. Consider periodic manual rotation and keep the tokens out of version control.

**Cloudflare IP ranges change.** The `set_real_ip_from` directives in nginx reference Cloudflare's published IP ranges. These ranges change occasionally. Cloudflare publishes them at `https://www.cloudflare.com/ips/`. If real IP restoration stops working, check whether the ranges in your nginx config are current.

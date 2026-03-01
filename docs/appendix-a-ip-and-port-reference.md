## IP and Port Reference

This appendix documents every machine, IP address, and listening port in the infrastructure. Use it as a quick-reference when debugging connectivity, writing firewall rules, or onboarding a new device.

---

## Machine IP Reference

| Machine | Role | LAN IP | Tailscale IP | SSH Alias(es) | User |
|---------|------|--------|--------------|---------------|------|
| StudioM4 | Dev (primary) | 192.168.0.102 | 100.68.91.96 | (local) | slate |
| OfficeM4P | Dev (secondary) | 192.168.0.101 | 100.102.76.65 | office, officem4p | quartz |
| ServerM2P | Home server | 192.168.0.100 | 100.74.27.128 | server, serverm2p | commandervander |
| VPS | Production | 212.38.95.33 | 100.111.63.3 | vps | root |
| NetSentry | DNS primary | 192.168.0.113 | 100.72.66.9 | netsentry | pi |
| AlertNode | DNS secondary + alerts | 192.168.0.112 | 100.69.47.71 | alertnode | pi |
| S23+ | Mobile | -- | 100.113.201.9 | -- | -- |

**Notes:**
- StudioM4 has no SSH alias because it is the primary dev machine (you are already on it).
- S23+ is a phone on the Tailscale mesh for remote access. It does not run SSH.
- All LAN IPs are statically assigned via DHCP reservation on the router.

---

## ServerM2P Port Map

The home server runs the most services. Ports are grouped by category.

### Reverse Proxy

| Port | Service | Type |
|------|---------|------|
| 80 | Caddy (HTTP) | Docker |
| 443 | Caddy (HTTPS) | Docker |

Caddy reverse-proxies internal services to `*.vanderdev.local` hostnames. Services behind Caddy (like Vaultwarden and Jellyfin) do not expose host ports directly.

### MCP Servers

| Port | Service | Type | Protocol |
|------|---------|------|----------|
| 3333 | LLM Gateway | Native node | SSE |
| 8081 | n8n-local MCP | Native (mcp-proxy) | SSE |
| 8082 | n8n-public MCP | Native (mcp-proxy) | SSE |
| 8083 | Memory MCP | Native (supergateway) | SSE |
| 8084 | Perplexity MCP | Native (supergateway) | SSE |
| 8085 | Docling MCP | Native (supergateway) | SSE |
| 8087 | Google Drive MCP | Native (supergateway) | SSE |
| 8088 | GitHub MCP | Native (supergateway) | SSE |
| 8089 | Google Sheets MCP | Native (supergateway) | SSE |

All MCP servers expose SSE endpoints. Dev machines connect via `http://192.168.0.100:<port>/sse` (LAN) or the Tailscale IP when remote.

### Databases and Storage

| Port | Service | Type |
|------|---------|------|
| 5432 | Supabase PostgreSQL | Docker |
| 5433 | PostGIS | Docker |
| 5050 | pgAdmin | Docker |
| 8086 | InfluxDB | Docker |

### Automation and Monitoring

| Port | Service | Type |
|------|---------|------|
| 5678 | n8n | Docker |
| 9000 | Portainer | Docker |
| 9090 | Prometheus | Docker |
| 3001 | Grafana | Docker |
| 9100 | node_exporter | Homebrew |

### Smart Home

| Port | Service | Type |
|------|---------|------|
| 8123 | Home Assistant | Docker |
| 8080 | Zigbee2MQTT | Docker |
| 1883 | Mosquitto MQTT (TCP) | Docker |
| 9001 | Mosquitto MQTT (WebSocket) | Docker |

---

## VPS Port Map

The VPS exposes minimal ports. All application traffic routes through nginx-proxy with automatic HTTPS.

| Port | Service | Notes |
|------|---------|-------|
| 22 | SSH | Key-only auth, fail2ban protected |
| 80 | nginx-proxy (HTTP) | Redirects to HTTPS |
| 443 | nginx-proxy (HTTPS) | Let's Encrypt via ACME companion |
| 3001 | Uptime Kuma | Status monitoring dashboard |

All other services (website, bots, Supabase, n8n) are Docker-internal and accessed via nginx reverse proxy using `VIRTUAL_HOST` and `LETSENCRYPT_HOST` environment variables. No additional host ports are exposed.

### Docker Services Behind nginx-proxy

These services do not bind to host ports. They are accessed via their domain names through nginx:

- Production website (static files served by nginx)
- Bot API endpoints (webhook-based)
- Supabase (PostgreSQL + REST + Studio + Auth + Storage)
- n8n workflows

---

## NetSentry Port Map

NetSentry is the primary DNS server and runs a monitoring dashboard.

| Port | Service | Notes |
|------|---------|-------|
| 53 | Pi-hole DNS | UDP and TCP, handles all LAN DNS queries |
| 80 | Pi-hole web UI (HTTP) | Admin dashboard |
| 443 | Pi-hole web UI (HTTPS) | TLS-secured admin dashboard |
| 3000 | Homepage dashboard | Infrastructure overview page |
| 3001 | Uptime Kuma | Service availability monitoring |
| 5335 | Unbound | Recursive DNS resolver, localhost only |

Unbound listens on localhost only -- Pi-hole forwards upstream queries to it, but no other machine connects to port 5335 directly.

---

## AlertNode Port Map

AlertNode is the secondary DNS server and handles push notifications.

| Port | Service | Notes |
|------|---------|-------|
| 53 | Pi-hole DNS | Redundant DNS, takes over if NetSentry is down |
| 80 | Pi-hole web UI (HTTP) | Admin dashboard |
| 443 | Pi-hole web UI (HTTPS) | TLS-secured admin dashboard |
| 8080 | ntfy | Self-hosted push notification server |

AlertNode runs on a Raspberry Pi 3B+. It runs warm (60-65C) and may thermal-throttle under sustained load. A heatsink or small fan is recommended.

---

## Port Allocation Convention

If you are adding new services, follow this allocation pattern:

| Range | Category |
|-------|----------|
| 80, 443 | Web / reverse proxy |
| 1000-2000 | IoT protocols (MQTT, etc.) |
| 3000-3999 | Dashboards and UIs |
| 5000-5999 | Databases and admin tools |
| 8000-8099 | MCP servers and automation |
| 8100-8199 | Smart home |
| 9000-9100 | Monitoring and management |

This is a convention, not a hard rule. The goal is predictability -- when you see port 808x, you know it is an MCP server.

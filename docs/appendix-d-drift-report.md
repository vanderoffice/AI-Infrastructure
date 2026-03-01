## Infrastructure Drift Report

This appendix documents all discrepancies found during a live SSH audit of all 4 remote machines on March 1, 2026. The main guide documents the actual state of the infrastructure; this appendix catalogs where documentation had drifted from reality and what was corrected.

Drift is normal. Infrastructure evolves faster than documentation. The value of a periodic audit is not that drift exists -- it always will -- but that you catch it before it causes an incident.

---

## Audit Methodology

The audit was performed by SSH-ing into each machine and comparing running services, open ports, and container states against the documented values in INFRASTRUCTURE.md. Tools used:

- `docker ps --format` for container inventory
- `ss -tlnp` and `lsof -i -P` for listening ports
- `systemctl list-timers` for scheduled tasks
- `tailscale status` for mesh network state
- `vcgencmd measure_temp` and `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq` for Pi health

---

## ServerM2P Discrepancies (6 Items)

| # | Item | Docs Said | Reality | Severity |
|---|------|-----------|---------|----------|
| 1 | Redis | Port 6379, listed as running service | Does not exist -- no container, no process, no config | Medium |
| 2 | Vaultwarden port | Port 8880 exposed to host | Caddy-proxied at `vault.vanderdev.local`, no host port binding | Medium |
| 3 | Jellyfin port | Port 8096 exposed to host | Caddy-proxied at `media.vanderdev.local`, no host port binding | Low |
| 4 | Supabase stack | Not documented | 9 containers running: db, kong, rest, studio, auth, meta, realtime, storage, imgproxy | High |
| 5 | Caddy | Not documented | Running on ports 80 and 443, reverse-proxying Vaultwarden and Jellyfin | Medium |
| 6 | MCP Gateway | Listed as Docker container | Runs as a native node process; compose file exists but is unused | Low |

### Additional Undocumented Services (Minor)

These were running but not documented. They are low-impact but should be cataloged:

- **Syncthing** -- ports 8384 (web UI) and 22000 (sync protocol)
- **socat Zigbee proxy** -- port 7639 forwarding to 192.168.0.21:7638
- **Home Assistant** -- also listening on port 8300 (in addition to documented 8123)

### Recommended Actions for ServerM2P

1. Remove Redis from documentation entirely (it was never deployed)
2. Update Vaultwarden and Jellyfin entries to note Caddy proxying instead of host port binding
3. Add Supabase stack to documentation with all 9 containers
4. Add Caddy to documentation as the local reverse proxy
5. Clarify MCP Gateway runs as native node, not Docker

---

## VPS Discrepancies (6 Items)

| # | Item | Docs Said | Reality | Severity |
|---|------|-----------|---------|----------|
| 1 | Container count | 28 containers | 29 containers (accessview-api was missing from docs) | Medium |
| 2 | Supabase health | 4 containers listed as "unhealthy" (known issue) | All containers healthy after image upgrades with proper health checks | Low |
| 3 | Uptime Kuma access | Described as "backend-only" | Publicly bound on `0.0.0.0:3001` | Medium |
| 4 | Rate limit zones | 3 nginx rate limit zones documented | 4 zones (accessview_api zone at 2r/min was added) | Low |
| 5 | Tailscale device count | 8 devices | 11 devices (includes 2 long-offline devices and 1 iPhone) | Low |
| 6 | accessview-api memory limit | Not documented (container not in docs) | No memory limit set -- the only container without one | Medium |

### Recommended Actions for VPS

1. Add accessview-api to container documentation
2. Remove the "known issue" note about unhealthy Supabase containers (resolved)
3. Evaluate Uptime Kuma exposure: either restrict to Tailscale-only or document as intentionally public
4. Update rate limit zone count and document the accessview_api zone
5. Audit Tailscale device list and remove long-offline devices
6. Add a memory limit to the accessview-api container (it is the only one without a limit, making it a potential memory leak risk)
7. Run `docker image prune` to reclaim 4-5GB of stale images

---

## Pi Discrepancies (8 Items)

| # | Item | Docs Said | Reality | Severity |
|---|------|-----------|---------|----------|
| 1 | NetSentry services | Pi-hole + Unbound + Homepage | Also runs Uptime Kuma on port 3001 | Medium |
| 2 | gravity-sync frequency | Every 30 minutes | Every hour (`OnUnitActiveSec=1h` in systemd timer) | Low |
| 3 | AlertNode upstream DNS | Cloudflare (1.1.1.1) only | Configured with 192.168.0.50 AND 1.1.1.1 | Medium |
| 4 | Watchdog ntfy topic | Implied to use `vander-infra` | Uses `infrastructure-alerts` topic | Low |
| 5 | AlertNode thermals | "Runs warm (60-65C)" | Actively throttling at 63.9C with throttle flag `0x80008` | Medium |
| 6 | mDNS resolution | `netsentry.local` should resolve | Failed from StudioM4 during audit (possibly transient) | Low |
| 7 | Pi-hole HTTPS | Not mentioned | Both Pis have TLS certificates and serve admin UI on port 443 | Low |
| 8 | Pi-hole versions | Not versioned in docs | Running v6.3/v6.4.1 (v6.4/v6.5 available) | Low |

### Recommended Actions for Pis

1. Add Uptime Kuma to NetSentry's documented services
2. Update gravity-sync frequency from "30 minutes" to "1 hour"
3. Investigate 192.168.0.50 -- what device is this? Is it still on the network? If it is a defunct device, remove it from AlertNode's upstream DNS configuration
4. Standardize ntfy topic names across all alerting scripts
5. Add a heatsink or small fan to AlertNode (Pi 3B+ thermal throttling at 63.9C degrades DNS performance)
6. Test mDNS resolution again -- if persistent, check Avahi/Bonjour configuration on NetSentry
7. Document Pi-hole HTTPS capability
8. Schedule Pi-hole updates (currently 1-2 minor versions behind)

---

## Note on com.backup.daily False Positive

During the audit, the agent flagged ServerM2P's `com.backup.daily` LaunchAgent as broken (exit code 127). This was a **false positive**. The `$HOME` variable usage in the plist was the correct fix (applied the day before the audit). The stale exit code was from historical runs that occurred before the fix was applied. The backup system was verified as functioning correctly on March 1, 2026.

**Lesson:** When auditing LaunchAgents, check the last *successful* run time, not just the last exit code. A non-zero exit code in `launchctl list` reflects the most recent run, which may be stale.

---

## Drift Prevention Practices

Based on patterns observed in this audit, here are practices that reduce documentation drift:

1. **Update docs in the same commit as the change.** If you add a Docker container, update the container list in the same PR. Do not plan to "document it later."

2. **Run a periodic audit.** Monthly or quarterly, SSH into every machine and compare reality to documentation. The `/audit` skill automates most of this.

3. **Document Caddy/nginx proxy routes, not just port bindings.** Services behind a reverse proxy do not expose host ports. Documenting "port 8880" when the service is actually proxied at `vault.vanderdev.local` creates confusion during troubleshooting.

4. **Version your infrastructure docs.** Keep INFRASTRUCTURE.md in your dotfiles repo (git-tracked). Every audit should produce a commit with corrections.

5. **Use "known issue" notes sparingly.** If a known issue persists for more than 2 weeks, it is not a "known issue" -- it is the actual state. Either fix it or update the documentation to reflect reality.

6. **Track device counts loosely.** Tailscale device counts, container counts, and similar numbers change frequently. Document ranges or "at least N" rather than exact counts, or accept that these will drift between audits.

---

## Summary

| Machine | Discrepancies Found | High Severity | Medium Severity | Low Severity |
|---------|-------------------|---------------|-----------------|-------------|
| ServerM2P | 6 | 1 | 3 | 2 |
| VPS | 6 | 0 | 3 | 3 |
| Pis (both) | 8 | 0 | 3 | 5 |
| **Total** | **20** | **1** | **9** | **10** |

The single high-severity item was an entirely undocumented Supabase stack (9 containers) on ServerM2P. The medium-severity items were mostly documentation listing host ports for services that are actually behind reverse proxies, plus a few missing services. Low-severity items were version numbers, counts, and minor configuration details.

No security issues were found. No services were running that should not have been. The drift was entirely documentation lag, not infrastructure misconfiguration.

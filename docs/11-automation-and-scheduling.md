# LaunchAgents, Backups, and Monitoring

An infrastructure with seven machines generates a surprising amount of maintenance work. Backups need to run every night. Logs need to sync from five machines down to a central store. Health checks need to verify that services are still running. Audit scripts need to detect configuration drift before it causes a problem. If you did all of this by hand, you would spend hours each week on housekeeping -- and you would still miss things, because humans forget and sleep through alarms.

Automation eliminates this entirely. Every recurring task in this infrastructure runs on a schedule, unattended, while you sleep. macOS LaunchAgents handle the Macs. Cron handles the VPS and Pis. A layered monitoring stack watches everything and only bothers you when something breaks. This chapter explains how all of it works, how the pieces fit together, and how to reproduce it for your own setup.

---

## macOS LaunchAgents -- Scheduled Tasks for Macs

If you have used Linux, you know cron: a daemon that runs commands on a schedule defined in a crontab. macOS has cron too, but it also has a more capable system called `launchd`. LaunchAgents are the user-facing layer of `launchd`, and they are what this infrastructure uses for every scheduled task on every Mac.

### What LaunchAgents Can Do

LaunchAgents support several scheduling modes that cron cannot match:

| Mode | How It Works | Example |
|------|-------------|---------|
| Calendar-based | Fire at a specific date/time (like cron) | "Run at 2:27 AM every day" |
| Interval-based | Fire every N seconds | "Check disk health every 300 seconds" |
| Event-based | Fire when a file changes or a path appears | "Run when a USB drive is mounted" |
| KeepAlive | Restart automatically if the process dies | "Keep this MCP server running forever" |

All LaunchAgents also get automatic stdout/stderr logging -- you specify log file paths in the plist, and macOS captures all output. No need to manually redirect output or wrap commands in logging scripts.

### How They Work

A LaunchAgent is a plist file -- an XML document that describes what to run, when to run it, and how to handle its lifecycle. These files live in `~/Library/LaunchAgents/`. When macOS boots (or when you load one manually), `launchd` reads every plist in that directory and begins managing the jobs.

Here is a minimal example -- a LaunchAgent that runs a backup script at 2:27 AM every day:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.backup.daily</string>

    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>__HOME__/dotfiles/backup/backup-monitor.sh</string>
    </array>

    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>2</integer>
        <key>Minute</key>
        <integer>27</integer>
    </dict>

    <key>StandardOutPath</key>
    <string>__HOME__/.local/logs/infra/backup.log</string>
    <key>StandardErrorPath</key>
    <string>__HOME__/.local/logs/infra/backup.log</string>
</dict>
</plist>
```

Notice the `__HOME__` placeholders. LaunchAgent plists require absolute paths -- environment variables like `$HOME` are not expanded inside XML. This infrastructure solves that problem with templates: the plists in the dotfiles repo use `__HOME__` and `__USER__` as placeholders, and the `setup.sh` installer replaces them with the actual values at install time using `sed`. This means the same plist template works on every Mac regardless of the username or home directory path.

### Managing LaunchAgents

The `launchctl` command manages LaunchAgents. Here are the operations you will use most:

```bash
# Load (activate) a LaunchAgent
launchctl load ~/Library/LaunchAgents/com.backup.daily.plist

# Unload (deactivate) a LaunchAgent
launchctl unload ~/Library/LaunchAgents/com.backup.daily.plist

# List all loaded agents and their status (0 = last run succeeded)
launchctl list | grep com.backup

# Bootstrap (modern equivalent of load, for macOS 10.10+)
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.backup.daily.plist

# Kickstart (run immediately, do not wait for schedule)
launchctl kickstart -k gui/$(id -u)/com.backup.daily
```

A common gotcha: if you edit a plist file while it is loaded, `launchd` does not pick up the changes automatically. You must unload and reload the agent. The `setup.sh` installer handles this -- it unloads every agent, copies the templated plists, and reloads them.

### LaunchAgents vs. LaunchDaemons

LaunchAgents run as your user. LaunchDaemons run as root. The distinction matters:

| | LaunchAgents | LaunchDaemons |
|---|---|---|
| **Location** | `~/Library/LaunchAgents/` | `/Library/LaunchDaemons/` |
| **Runs as** | Your user | root |
| **Requires** | User login session | Just boot |
| **Use for** | Backups, syncs, user tools | System services, network daemons |

This infrastructure uses LaunchAgents for everything. Backups, log syncs, MCP servers, health checks -- all of them run as the logged-in user. This is a deliberate choice: LaunchAgents can access user-level SSH keys, user-level configs, and user-level paths without `sudo`. The only exception is the `pmset` wake schedule (covered below), which requires root.

---

## Per-Machine Agent Assignments

Not every machine runs the same jobs. Each Mac has a role, and its LaunchAgents reflect that role.

### Agents on All Macs (StudioM4, OfficeM4P, ServerM2P)

These three agents run on every Mac in the fleet:

| Agent | Purpose | Schedule |
|-------|---------|----------|
| `com.backup.daily` | Daily backup to ServerM2P | 2:27 AM |
| `com.logs.sync` | Log sync to ServerM2P | 2:20 AM |
| `com.claude.session-rename` | Auto-rename Claude Code sessions | 2:15 AM |

The session-rename agent deserves a brief explanation. Claude Code sessions are identified by UUID by default, which makes it hard to find a specific session later. The `auto-rename-sessions.sh` script runs nightly and renames sessions based on their content, giving you meaningful names like "waterbot-rag-pipeline" instead of "a3f8c2d1-7b4e-4f9a-b2c1-8d3e5f6a7b8c."

### Agents on ServerM2P Only

ServerM2P is the always-on hub. It runs the most agents because it handles MCP hosting, backup aggregation, and offsite syncing:

| Agent | Purpose | Schedule |
|-------|---------|----------|
| `com.mcp.gateway` | MCP Gateway (port 3333) | KeepAlive |
| `com.mcp.memory` | Memory MCP (port 8083) | KeepAlive |
| `com.mcp.n8n-local` | n8n-local MCP (port 8081) | KeepAlive |
| `com.mcp.n8n-public` | n8n-public MCP (port 8082) | KeepAlive |
| `com.mcp.perplexity` | Perplexity MCP (port 8084) | KeepAlive |
| `com.backup.offsite` | Google Drive offsite sync | 6:15 AM |
| `com.backup.healthcheck` | Backup health check | 7:30 AM |
| `com.logs.aggregate` | Log aggregation from all sources | 3:45 AM |
| `com.maindrive.monitor` | MAINDRIVE health monitoring | Interval |
| `com.backup.pi` | Pi backup (pull from AlertNode + NetSentry) | 3:40 AM |

The five MCP agents use `KeepAlive` mode. This means `launchd` starts them at boot and restarts them if they crash. They are not scheduled tasks -- they are persistent services. This is how the MCP hub stays available 24/7 without manual intervention.

The MAINDRIVE monitor runs on an interval (not a calendar schedule). It periodically checks the health of the 4TB external SSD that stores all backups. If the drive disconnects or shows SMART warnings, the monitor fires an alert.

### Agents on OfficeM4P Only

| Agent | Purpose | Schedule |
|-------|---------|----------|
| `com.audit.nightly` | Quick infrastructure audit | 3:15 AM |
| `com.audit.weekly` | Full infrastructure audit | Monday 4:00 AM |
| `com.granola.mcp-proxy` | Granola MCP proxy (port 8080) | KeepAlive |
| `com.gis.mount` | Auto-mount GIS SMB share | OnDemand |

OfficeM4P runs the audit agents because it has the most compute headroom (M4 Pro, 24GB). The nightly audit runs a quick check -- SSH connectivity, service status, backup recency. The weekly audit on Monday mornings runs the full suite: dotfiles parity, LaunchAgent consistency, MCP connectivity, disk health, and configuration drift detection.

### Agents on StudioM4 Only

| Agent | Purpose | Schedule |
|-------|---------|----------|
| `com.phone.remote-scan` | Phone scan via ADB | 2:30 AM |
| `com.gis.mount` | Auto-mount GIS SMB share | OnDemand |

StudioM4 has the fewest unique agents. The phone scan agent connects to the Samsung Galaxy S23+ over ADB (Android Debug Bridge) to pull media and sync data while the phone charges overnight.

---

## The Wake Schedule

There is a problem with scheduling tasks at 2:00 AM on laptops: laptops sleep. A LaunchAgent that fires at 2:27 AM does nothing if the machine is asleep at 2:27 AM. macOS will run the task when the machine next wakes, but that might be 7:00 AM when you open the lid -- and now your backup competes with your morning work.

The fix is `pmset`, the macOS power management tool. This command tells the hardware (not the OS -- the hardware) to wake the machine at a specific time:

```bash
sudo pmset repeat wake MTWRFSU 02:25:00
```

This wakes every Mac at 2:25 AM, seven days a week. The first scheduled task fires at 2:27 AM -- two minutes later, giving the machine time to fully wake, reconnect to the network, and establish Tailscale tunnels.

Important details:

- `pmset` is a per-machine hardware setting. You must run this command on each Mac separately. It is not synced by dotfiles.
- The `MTWRFSU` string means Monday through Sunday (every day of the week).
- The machine wakes, runs its tasks, and goes back to sleep automatically (macOS will sleep again after the idle timeout).
- ServerM2P runs headless with sleep disabled, so it does not need `pmset` for waking -- but the other two Macs do.

---

## The Complete Nightly Schedule

Every automated task in the infrastructure fits into a coordinated nightly sequence. The timing is deliberate: logs sync before backups (so the backup includes the latest logs), backups complete before the health check (so the check validates tonight's backup), and the offsite sync runs last (so it pushes the most current data to Google Drive).

| Time (Pacific) | Machine | Action | Script |
|----------------|---------|--------|--------|
| 2:15 AM | All Macs | Session auto-rename | `auto-rename-sessions.sh` |
| 2:20 AM | StudioM4 | Log sync to ServerM2P | `log-sync.sh` |
| 2:22 AM | OfficeM4P | Log sync to ServerM2P | `log-sync.sh` |
| 2:27 AM | StudioM4 | Backup to ServerM2P | `backup-monitor.sh` |
| 2:27 AM | OfficeM4P | Backup to ServerM2P | `backup-monitor.sh` |
| 2:30 AM | StudioM4 | Phone scan (ADB) | `phone-scan.sh` |
| 3:00 AM | VPS | Local backup (PostgreSQL, Docker) | `backup-all.sh` |
| 3:15 AM | OfficeM4P | Quick audit | `quick-audit.sh` |
| 3:15 AM | VPS | Log sync to ServerM2P | `sync-logs.sh` |
| 3:30 AM | VPS | Rsync backup to ServerM2P | `vps-backup.sh` |
| 3:40 AM | ServerM2P | Pi backup (AlertNode + NetSentry) | `pi-backup.sh` |
| 3:45 AM | ServerM2P | Aggregate logs | `aggregate-logs.sh` |
| 6:15 AM | ServerM2P | Offsite sync to Google Drive | `offsite-sync.sh` |
| 7:30 AM | ServerM2P | Backup health check | `backup-health-check.sh` |

### Weekly Operations

Some tasks do not need to run every night. These run on a weekly cadence:

| Schedule | Machine | Action |
|----------|---------|--------|
| Sunday 3:30 AM | VPS | Hostinger API snapshot |
| Sunday 8:00 AM | ServerM2P | Weekly backup digest (dead man's switch) |
| Monday 4:00 AM | OfficeM4P | Full infrastructure audit |

The Sunday digest is special. It is a dead man's switch -- its job is to prove the monitoring system itself is still running. If you do not receive the Sunday digest, the monitoring is broken and you need to investigate. More on this in the monitoring section below.

---

## The Backup System -- 3-2-1 Strategy

The backup system follows the 3-2-1 rule, a strategy widely recommended by storage professionals and disaster recovery specialists:

- **3** copies of every important file
- **2** different storage media
- **1** copy offsite (away from your physical location)

Here is how this infrastructure implements it:

| Copy | Location | Medium | Protects Against |
|------|----------|--------|-----------------|
| 1 (original) | The machine where the data lives | Internal SSD | Nothing -- this IS the data |
| 2 (network) | ServerM2P, on MAINDRIVE (4TB SSD) | External SSD | Machine failure, theft, accidental deletion |
| 3 (offsite) | Google Drive `/BACKUPS/` | Cloud storage | House fire, burglary, natural disaster |

### The Backup Chain

Data flows through four stages, from source machines to offsite storage:

```
Stage 1: Mac Backups (2:27 AM)
    StudioM4  ──rsync over SSH──▶  ServerM2P:/Volumes/MAINDRIVE/BACKUPS/StudioM4/daily/
    OfficeM4P ──rsync over SSH──▶  ServerM2P:/Volumes/MAINDRIVE/BACKUPS/OfficeM4P/daily/

Stage 2: VPS Backup (3:00 AM local, then 3:30 AM)
    VPS ──backup-all.sh──▶ local tarball (PostgreSQL dump, Docker volumes, configs)
    VPS ──rsync──▶ ServerM2P:/Volumes/MAINDRIVE/BACKUPS/VPS/weekly/

Stage 3: Pi Backup (3:40 AM)
    ServerM2P ──SSH pull──▶ AlertNode configs, Pi-hole data
    ServerM2P ──SSH pull──▶ NetSentry configs, Pi-hole data, Uptime Kuma DB

Stage 4: Offsite Sync (6:15 AM)
    ServerM2P ──rclone sync──▶ Google Drive /BACKUPS/
```

Notice the direction of data flow. Mac backups push to ServerM2P (the Macs initiate the rsync). Pi backups are pulled by ServerM2P (the server SSH-es into the Pis and copies data out). This matters because the Pis are low-power devices -- it is more reliable to have the server drive the backup than to ask a Pi to push data.

### What Gets Backed Up (Mac)

The Mac backup script (`mac-backup.sh`) is selective. It backs up data that is hard to recreate, and it skips data that is easy to rebuild:

**Backed up:**

- Work outputs (`~/Documents/Work/`)
- Claude Code configuration (`~/.claude.json`)
- n8n configuration (`.n8n/`)
- Obsidian vault (`~/.local/share/obsidian-vault/`)
- Reference data and datasets
- Claude Code local config and session history
- Presentations (`~/presentations/`)
- Infrastructure metrics
- Shell configurations (`.zshrc`, `.zprofile`)

**Not backed up:**

- Symlinked Claude Code config (lives in git via dotfiles -- GitHub is the backup)
- Skill dependencies like `node_modules/` (rebuildable with `npm install`)
- Git repositories (GitHub is the source of truth, and auto-push hooks keep them current)
- Application binaries (reinstallable via Homebrew and the `Brewfile`)
- System caches and temporary files

This distinction keeps backup sizes small and backup times fast. A nightly Mac backup transfers only the delta (changed files) via rsync, which typically completes in under a minute.

### What Gets Backed Up (VPS)

The VPS backup is more comprehensive because production data lives there:

- PostgreSQL database dumps (all databases, including Supabase and pgvector data)
- Docker volumes (persistent data for all containers)
- Nginx configuration
- SSL certificates
- n8n workflows and credentials
- Environment files (encrypted)
- Application source code and build artifacts

### The Shared Helper: backup-common.sh

All six backup scripts in the infrastructure source a shared helper file called `backup-common.sh`. This file provides three functions that keep backup alerting consistent:

| Function | Purpose | Example Output |
|----------|---------|----------------|
| `send_backup_alert()` | Send an ntfy notification with structured context | Priority, title, machine, duration, error details |
| `format_duration()` | Convert seconds to human-readable time | `127` becomes `2m 7s` |
| `format_size()` | Convert bytes to human-readable sizes | `1073741824` becomes `1.0 GB` |

By centralizing these functions, every backup script produces consistent, structured alerts. A failure notification always includes the machine name, the script that failed, how long it ran before failing, and the relevant error output. You never get a bare "backup failed" message with no context.

### Retention Policy

Not all backup data is kept forever. Retention policies balance storage cost against recovery needs:

| Data Type | Retention | Reasoning |
|-----------|-----------|-----------|
| Mac daily backups | 60 days on MAINDRIVE | Two months of rollback for any file |
| VPS weekly backups | 60 days | Enough history to recover from slow-burn data corruption |
| Google Drive offsite | Latest only (sync, not archive) | Disaster recovery, not version history |
| Permission/audit logs | 90 days | Compliance and troubleshooting window |
| Session logs | 60 days | Enough to trace any recent issue |

The Google Drive offsite is intentionally "latest only." `rclone sync` mirrors the current state of MAINDRIVE's backup directory to Google Drive -- it does not keep old versions. If you need point-in-time recovery, that is what the 60-day retention on MAINDRIVE provides. The offsite copy is strictly for disaster recovery: if your house burns down, you can recover the most recent backup from Google Drive and rebuild.

---

## Monitoring Stack

Backups protect your data. Monitoring protects your uptime. This infrastructure uses a layered monitoring approach: metrics collection with Prometheus, visualization with Grafana, synthetic monitoring with Uptime Kuma, and push notifications with ntfy. Each layer catches different categories of problems.

### Prometheus + Grafana (on VPS)

Prometheus is a time-series database that collects metrics by scraping HTTP endpoints on a schedule. Every machine in the infrastructure runs `node-exporter`, a small agent that exposes system metrics (CPU, memory, disk, network) on an HTTP port. Prometheus scrapes these endpoints every 15 seconds and stores the data.

Grafana connects to Prometheus and turns the raw metrics into dashboards. You can see CPU usage across all seven machines on a single screen, drill into disk utilization trends over the past week, or compare network throughput between the VPS and the home server.

The key metrics monitored:

| Metric | What It Tells You | Alert Threshold |
|--------|-------------------|-----------------|
| CPU usage | Whether a machine is overloaded | Above 90% for 5 minutes |
| Memory usage | Whether a machine is running out of RAM | Above 85% for 5 minutes |
| Disk usage | Whether a drive is filling up | Above 80% capacity |
| Network I/O | Whether traffic patterns are abnormal | No fixed threshold (anomaly-based) |
| Container status | Whether Docker containers are running | Any container down for 2 minutes |
| System uptime | Whether a machine has rebooted unexpectedly | Uptime less than 1 hour (unexpected reboot) |

Prometheus has 8 alert rules configured. When a rule fires (a condition is true for longer than its threshold), Prometheus sends the alert to Alertmanager.

### Alertmanager and ntfy

Alertmanager is Prometheus's companion. It receives alerts from Prometheus and decides what to do with them: group related alerts, suppress duplicates, wait for a condition to persist before notifying, and route alerts to the appropriate notification channel.

In this infrastructure, Alertmanager routes all alerts to ntfy.sh. ntfy (pronounced "notify") is a simple push notification service. You subscribe to a topic on your phone, and any HTTP POST to that topic appears as a push notification. You can self-host ntfy on a Raspberry Pi (as this infrastructure does on AlertNode) or use the free cloud service at ntfy.sh.

The flow looks like this:

```
node-exporter (on each machine)
    ▼ metrics via HTTP
Prometheus (on VPS)
    ▼ alert rules evaluate
Alertmanager (on VPS)
    ▼ HTTP POST
ntfy (on AlertNode or ntfy.sh)
    ▼ push notification
Your phone
```

### Uptime Kuma -- Synthetic Monitoring

Prometheus monitors internal metrics (CPU, disk, memory). Uptime Kuma monitors external availability -- it acts like a user and checks whether services are actually reachable.

This infrastructure runs two Uptime Kuma instances:

| Instance | Location | Purpose |
|----------|----------|---------|
| VPS (port 3001) | Public internet | Monitors services from outside the LAN |
| NetSentry (port 3001) | Inside the LAN | Monitors services from inside the network |

Running two instances from different network locations catches different failure modes. If a service is reachable from inside the LAN but not from the internet, that is a firewall or DNS issue, not a service failure. If it is unreachable from both locations, the service itself is down.

The VPS instance monitors 11 endpoints, including:

- The production website (vanderdev.net)
- Each chatbot endpoint
- The MCP gateway (via Tailscale)
- The Pi-hole DNS servers
- n8n webhook endpoints
- Grafana and Prometheus themselves

Uptime Kuma checks each endpoint every 60 seconds and sends an ntfy notification if any check fails three times in a row. This three-strike approach prevents false alarms from transient network blips.

### The Dead Man's Switch

All of the monitoring described above has a single point of failure: the monitoring system itself. If Prometheus crashes and Alertmanager goes down with it, you receive no alerts -- and the silence feels exactly like "everything is healthy."

The dead man's switch solves this. Every Sunday at 8:00 AM, `backup-health-check.sh --weekly-digest` runs on ServerM2P and sends a summary notification to the `your-backup-topic` ntfy topic. This notification includes:

- Backup status for every machine (last successful backup time, size)
- Any warnings or anomalies from the past week
- Confirmation that the monitoring chain (ServerM2P to ntfy to your phone) is functional

The critical insight: **the notification itself is the test.** If you receive the Sunday digest, the monitoring system is working. If you do not, something in the chain is broken and you need to investigate immediately. Set a recurring reminder on your phone for Sunday morning. If the digest has not arrived by 9:00 AM, SSH into ServerM2P and start debugging.

---

## Alerting Philosophy

The most common mistake in monitoring is over-alerting. If your phone buzzes every time a backup succeeds, every time a health check passes, and every time a metric is within normal range, you will learn to ignore notifications. And then when a real failure fires, you will ignore that too.

This infrastructure follows a strict principle: **silence equals healthy.** You never receive a notification that says "backup succeeded" or "all systems normal." You only receive notifications when something is wrong. If your phone is quiet, everything is fine.

### ntfy Topics

Alerts are separated into two topics so you can set different notification behaviors for each:

| Topic | Purpose | Typical Volume |
|-------|---------|----------------|
| `your-infra-topic` | Infrastructure alerts (service down, audit failures, disk warnings) | 0-2 per week |
| `your-backup-topic` | Backup alerts (failure-only) + weekly digest | 1 per week (just the digest) |

On a good week, you receive exactly one notification: the Sunday digest. Any additional notification means something needs attention.

### Priority Levels

ntfy supports priority levels that control how aggressively the notification demands your attention:

| Priority | Phone Behavior | When to Use |
|----------|---------------|-------------|
| 1 (min) | Silent, no badge | Debug and test notifications |
| 2 (low) | Silent, badge only | Weekly digest, informational warnings |
| 3 (default) | Normal sound | Standard alerts (backup failure, service restart) |
| 4 (high) | Louder sound, persistent | Service down, critical failures |
| 5 (urgent) | Persistent, bypasses Do Not Disturb | Emergency only (data loss risk, security breach) |

The priority is set in the `send_backup_alert()` function based on the severity of the event. A single backup failure on a non-critical machine gets priority 3. An unreachable machine gets priority 4. Priority 5 is reserved for scenarios you hope never happen -- and in practice, it has never fired.

---

## Cron on Linux (VPS and Pis)

The VPS runs Ubuntu and the Pis run Raspberry Pi OS (Debian-based). These machines use cron instead of LaunchAgents. Cron is simpler -- a single file defines all scheduled tasks:

```bash
# Example crontab on VPS (crontab -e)
0 3 * * *   /root/scripts/backup-all.sh >> /var/log/backup.log 2>&1
15 3 * * *  /root/scripts/sync-logs.sh >> /var/log/log-sync.log 2>&1
30 3 * * *  /root/scripts/vps-backup.sh >> /var/log/vps-backup.log 2>&1
30 3 * * 0  /root/scripts/hostinger-snapshot.sh >> /var/log/snapshot.log 2>&1
```

The format is `minute hour day-of-month month day-of-week command`. The `>>` appends output to a log file, and `2>&1` redirects stderr to the same file. Unlike LaunchAgents, cron does not provide built-in logging -- you must handle it yourself.

Cron on the Pis is minimal. The Pis do not initiate their own backups (ServerM2P pulls data from them). The Pi crontabs primarily handle:

- Pi-hole gravity list updates
- Unbound root hints refresh
- ntfy service health checks (on AlertNode)
- Uptime Kuma database maintenance (on NetSentry)

---

## Reproduction Steps

You do not need seven machines to benefit from automation. Start with one Mac and one backup destination, then expand as your infrastructure grows.

### Step 1: Create Your First Backup Script

Write a bash script that rsyncs your important directories to a backup destination. This can be an external drive, a NAS, or another computer on your network:

```bash
#!/bin/bash
# mac-backup.sh -- minimal example
DEST="user@backup-server:/backups/$(hostname)/daily/"
rsync -az --delete \
    ~/Documents/ \
    ~/.claude.json \
    ~/.zshrc \
    "$DEST"
```

Test it manually first. Run it, verify the files appear at the destination, and make sure the rsync excludes are correct before automating it.

### Step 2: Create a LaunchAgent Plist

Write a plist that runs your backup script on a schedule. Use `__HOME__` placeholders so the template works across machines:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.backup.daily</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>__HOME__/dotfiles/backup/mac-backup.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>2</integer>
        <key>Minute</key>
        <integer>27</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>__HOME__/.local/logs/infra/backup.log</string>
    <key>StandardErrorPath</key>
    <string>__HOME__/.local/logs/infra/backup.log</string>
</dict>
</plist>
```

### Step 3: Template and Install

Add a templating step to your `setup.sh` that replaces `__HOME__` and loads the agent:

```bash
# In setup.sh
HOME_DIR=$(eval echo ~)
for plist in dotfiles/launchagents/*.plist; do
    dest="$HOME_DIR/Library/LaunchAgents/$(basename "$plist")"
    sed "s|__HOME__|$HOME_DIR|g; s|__USER__|$(whoami)|g" "$plist" > "$dest"
    launchctl unload "$dest" 2>/dev/null
    launchctl load "$dest"
done
```

### Step 4: Set the Wake Schedule

```bash
sudo pmset repeat wake MTWRFSU 02:25:00
```

Run this on every Mac that needs to wake for scheduled tasks.

### Step 5: Set Up ntfy for Notifications

Install ntfy on a Raspberry Pi or sign up at ntfy.sh. Subscribe to your topic on your phone. Then add notification calls to your backup script:

```bash
# At the end of mac-backup.sh, on failure
if [ $? -ne 0 ]; then
    curl -s -d "Backup failed on $(hostname)" \
        -H "Title: Backup Failure" \
        -H "Priority: 4" \
        "https://ntfy.sh/your-backup-topic"
fi
```

### Step 6: Add Prometheus and Grafana (Optional)

If you run a VPS or home server with Docker, add Prometheus and Grafana to your Docker Compose stack. Install `node-exporter` on every machine you want to monitor. Configure Prometheus scrape targets and Alertmanager routes. This is a larger project -- Chapter 6 (Production VPS) covers the Docker Compose setup in detail.

### Step 7: Set Up the Dead Man's Switch

Create a weekly health check script that sends a summary notification every Sunday. Subscribe to the notification and set a phone reminder to check for it. If it does not arrive, your monitoring is broken.

### Step 8: Test the Full Chain

Trigger a failure deliberately. Stop a service, fill a disk, or introduce an error in a backup script. Verify that the alert arrives on your phone with the correct priority, title, and context. If it does not, walk the chain: Did the script detect the failure? Did it call ntfy? Did ntfy deliver? Is your phone subscribed to the right topic?

---

## Gotchas and Hard-Won Lessons

These are the problems you will hit. Every one of them was discovered the hard way.

**LaunchAgents require absolute paths.** The `$HOME` variable is not expanded inside plist XML. If you write `$HOME/dotfiles/backup.sh` in a plist, `launchd` will look for a literal directory called `$HOME` and fail silently. Use the templating approach with `__HOME__` placeholders and `sed` replacement.

**pmset is a hardware setting, not a software config.** The wake schedule is stored in the Mac's SMC (System Management Controller), not in a config file. Dotfiles sync does not propagate it. You must run `sudo pmset repeat wake` on each Mac individually after setup.

**The 2-minute wake buffer is important.** The wake schedule fires at 2:25 AM and the first task runs at 2:27 AM. Those two minutes give the machine time to fully wake, reconnect to Wi-Fi, establish Tailscale tunnels, and mount any network shares. If your machine is slow to wake (older hardware, complex network), increase the buffer.

**rclone requires interactive OAuth setup.** The first time you configure rclone for Google Drive, it opens a browser window for OAuth authentication. This is a one-time interactive step that cannot be automated. Do it manually on the machine that will run the offsite sync, then the token is saved locally and refreshed automatically on subsequent runs.

**ntfy topic names must match exactly.** The topic name in your alert scripts must exactly match the topic you subscribe to on your phone. There is no validation -- if the names do not match, notifications silently go nowhere. Use a consistent naming convention and document your topics.

**The dead man's switch only works if you check for it.** The Sunday digest proves the monitoring system works, but only if a human notices when it is missing. Set a recurring calendar reminder. If the digest has not arrived by 9:00 AM Sunday, that is your signal to investigate.

**launchctl errors are cryptic.** When a LaunchAgent fails to load, `launchctl` gives you an error code (like `78` or `113`) with no human-readable explanation. The most common causes: malformed XML (validate with `plutil`), missing executable (check the path), or a duplicate Label (another plist uses the same label string).

**Backup scripts should be idempotent.** If a backup is interrupted halfway through and re-runs the next night, it should produce the correct result. `rsync` handles this naturally (it only transfers changed files), but if your backup script has setup or teardown steps, make sure they are safe to repeat.

**Log rotation prevents disk fills.** Every LaunchAgent that writes to a log file will eventually fill the disk if the log is never rotated. Either configure `newsyslog` to rotate your logs, or add size-based truncation to your scripts. The infrastructure uses a combination: 60-day retention with automatic cleanup by the log aggregation script.

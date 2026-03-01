## Config File Templates

This appendix provides annotated templates for the key configuration files in the infrastructure. Copy them, replace the placeholder values, and adapt to your setup.

---

## ~/.claude.json (Dev Machine MCP Configuration)

This file tells Claude Code where to find your MCP servers. It lives on each dev machine and points to your home server.

```json
{
  "mcpServers": {
    "memory": {
      "type": "sse",
      "url": "http://<server-ip>:8083/sse"
    },
    "llm-gateway": {
      "type": "sse",
      "url": "http://<server-ip>:3333/sse"
    },
    "perplexity": {
      "type": "sse",
      "url": "http://<server-ip>:8084/sse"
    },
    "github": {
      "type": "sse",
      "url": "http://<server-ip>:8088/sse"
    },
    "google-sheets": {
      "type": "sse",
      "url": "http://<server-ip>:8089/sse"
    },
    "google-drive": {
      "type": "sse",
      "url": "http://<server-ip>:8087/sse"
    },
    "n8n-local": {
      "type": "sse",
      "url": "http://<server-ip>:8081/sse"
    },
    "docling": {
      "type": "sse",
      "url": "http://<server-ip>:8085/sse"
    },
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-filesystem", "/path/to/your/projects"]
    }
  }
}
```

**Key points:**
- Replace `<server-ip>` with your home server's LAN IP (e.g., `192.168.0.100`).
- All SSE servers run on the home server. Only `filesystem` runs locally via stdio.
- This file is machine-specific. It is NOT part of your dotfiles repo (each machine may have different filesystem paths).
- Never put API keys directly in this file. API keys live on the server side, inside the MCP server processes.
- After editing, restart Claude Code for changes to take effect.

### Minimal Version

If you are just starting out, you only need memory and filesystem:

```json
{
  "mcpServers": {
    "memory": {
      "type": "sse",
      "url": "http://<server-ip>:8083/sse"
    },
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-filesystem", "/path/to/your/projects"]
    }
  }
}
```

---

## SSH Config with Smart Routing

This SSH config uses `Match` blocks and a helper script to route connections over LAN when at home and Tailscale when remote.

### ~/.ssh/config

```
# ─── Smart routing: LAN at home, Tailscale when remote ───

Match host "server" exec "~/.ssh/is-home.sh"
    HostName ServerName.local

Match host "server"
    HostName <server-tailscale-ip>

Host server
    User <username>
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    AddKeysToAgent yes
    ServerAliveInterval 60

# ─── VPS (always Tailscale or public IP) ───

Host vps
    HostName <vps-ip>
    User root
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    ServerAliveInterval 60

# ─── Second dev machine ───

Match host "office" exec "~/.ssh/is-home.sh"
    HostName OfficeMac.local

Match host "office"
    HostName <office-tailscale-ip>

Host office
    User <username>
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
    AddKeysToAgent yes

# ─── Raspberry Pis ───

Host netsentry
    HostName <netsentry-lan-ip>
    User pi
    IdentityFile ~/.ssh/id_ed25519

Host alertnode
    HostName <alertnode-lan-ip>
    User pi
    IdentityFile ~/.ssh/id_ed25519
```

### ~/.ssh/is-home.sh

```bash
#!/bin/bash
# Returns 0 (true) if on the home network, 1 (false) otherwise.
# SSH Match blocks use the exit code to decide which HostName to use.

HOME_NETWORK_PREFIX="192.168.0."

if ifconfig 2>/dev/null | grep -q "$HOME_NETWORK_PREFIX"; then
    exit 0
else
    exit 1
fi
```

Make it executable: `chmod +x ~/.ssh/is-home.sh`

**How it works:** SSH evaluates `Match` blocks top to bottom. When you run `ssh server`:
1. First `Match` checks if `is-home.sh` exits 0. If yes, uses the `.local` mDNS hostname (LAN).
2. If not at home, the second `Match` fires and uses the Tailscale IP.
3. The `Host server` block supplies the username, key, and keepalive settings regardless of which `Match` was used.

---

## infrastructure.sh Key Variables

The `infrastructure.sh` script in your dotfiles repo defines machine metadata, sync targets, and alert configuration. These variables are sourced by backup scripts, health checks, and the audit skill.

```bash
#!/bin/bash
# infrastructure.sh — Machine definitions and infrastructure config
# Sourced by: backup scripts, health checks, audit, sync

# ─── Machine Definitions ───
# Format: alias:name:tailscale_ip:lan_ip:user:role:always_on
MACHINES="studio:StudioM4:100.68.91.96:192.168.0.102:slate:dev:no"
MACHINES="$MACHINES office:OfficeM4P:100.102.76.65:192.168.0.101:quartz:dev:yes"
MACHINES="$MACHINES server:ServerM2P:100.74.27.128:192.168.0.100:commandervander:server:yes"
MACHINES="$MACHINES vps:VPS:100.111.63.3:212.38.95.33:root:production:yes"
MACHINES="$MACHINES netsentry:NetSentry:100.72.66.9:192.168.0.113:pi:dns:yes"
MACHINES="$MACHINES alertnode:AlertNode:100.69.47.71:192.168.0.112:pi:dns:yes"

# ─── Sync Targets ───
# Machines that receive dotfiles on sync
SYNC_MACHINES="server office vps"

# ─── Paths ───
LOG_DIR="$HOME/.local/logs/infra"
BACKUP_LOG="$LOG_DIR/backup-log.csv"
METRICS_FILE="$LOG_DIR/metrics.csv"
HISTORY_DIR="$HOME/dotfiles/docs/history"

# ─── Alert Configuration ───
NTFY_TOPIC="vander-infra"
NTFY_BACKUP_TOPIC="vanderbackups"
NTFY_URL="http://192.168.0.112:8080"

# ─── Thresholds ───
DISK_WARN_THRESHOLD=80
DISK_CRIT_THRESHOLD=90
MEMORY_WARN_THRESHOLD=85
TEMP_WARN_THRESHOLD=70
```

**Adapt for your setup:** Change machine names, IPs, usernames, and ntfy topics. The `always_on` field tells health checks which machines should always be reachable (backup alerts fire if an `always_on` machine is unreachable).

---

## LaunchAgent Plist Template

LaunchAgents are macOS's built-in task scheduler. Each plist file defines one scheduled task. Place them in `~/Library/LaunchAgents/`.

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
    <string>__HOME__/.local/logs/infra/backup-error.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/opt/homebrew/bin</string>
    </dict>
</dict>
</plist>
```

**Key points:**
- `__HOME__` and `__USER__` are placeholder tokens replaced by `setup.sh` at install time with the actual home directory and username. This keeps plists portable across machines with different usernames.
- `StartCalendarInterval` runs the task at the specified time daily. Omit a key (like `Day`) to mean "every" (every day, in this case).
- `StandardOutPath` and `StandardErrorPath` capture logs. Always point these to `~/.local/logs/infra/` for consistency.
- The `PATH` environment variable must include Homebrew paths, because LaunchAgents run in a minimal environment without your shell profile.

### Loading and Managing LaunchAgents

```bash
# Load (register) a LaunchAgent
launchctl load ~/Library/LaunchAgents/com.backup.daily.plist

# Unload (deregister) a LaunchAgent
launchctl unload ~/Library/LaunchAgents/com.backup.daily.plist

# Check if it's loaded
launchctl list | grep com.backup

# Manually trigger a run (useful for testing)
launchctl start com.backup.daily
```

---

## CLAUDE.md Structure Guide

Your global `CLAUDE.md` file (at `~/.claude/CLAUDE.md`) is the single most important configuration file in the system. It defines how Claude Code behaves across all sessions. Here is the recommended structure:

```markdown
# Claude Code Instructions

## Hard Gates
<!-- Rules that MUST be followed. Use these for: -->
<!-- - Safety checks (never commit secrets) -->
<!-- - Research requirements (read docs before acting) -->
<!-- - File organization (where outputs go) -->

## Priority Order
<!-- What takes precedence when rules conflict -->
<!-- Security > Research > Action > Documentation -->

## Operating Modes
<!-- Different behavior profiles -->
<!-- Development mode, Writing mode, etc. -->

## Communication Style
<!-- How Claude should talk to you -->
<!-- Casual in chat, professional in deliverables -->

## MCP and Tools
<!-- Which MCP servers to prefer for which tasks -->
<!-- Research routing rules -->
```

**Best practices:**
- Keep gates short and specific. Vague instructions get ignored.
- Use `(BLOCKING)` to mark gates that must be checked before proceeding.
- Reference external docs instead of inlining everything. CLAUDE.md should be an index, not a novel.
- Test your gates by deliberately triggering them in a Claude Code session.

---

## dotfiles/setup.sh Pattern

The setup script creates symlinks from your home directory to the dotfiles repo. This is how configuration stays in sync across machines via git.

```bash
#!/bin/bash
set -euo pipefail

DOTFILES="$HOME/dotfiles"
HOME_DIR="$HOME"

# ─── Symlink .claude directories ───
# Use directory-level symlinks, not file-by-file
for dir in commands skills docs scripts; do
    target="$HOME_DIR/.claude/$dir"
    source="$DOTFILES/.claude/$dir"
    if [ -L "$target" ]; then
        echo "Already linked: $target"
    elif [ -d "$target" ]; then
        echo "WARNING: $target is a real directory, replacing with symlink"
        rm -rf "$target"
        ln -s "$source" "$target"
    else
        ln -s "$source" "$target"
    fi
done

# ─── Install LaunchAgents ───
PLIST_DIR="$HOME_DIR/Library/LaunchAgents"
mkdir -p "$PLIST_DIR"

for plist in "$DOTFILES"/launchagents/*.plist; do
    name=$(basename "$plist")
    # Replace __HOME__ and __USER__ placeholders
    sed "s|__HOME__|$HOME_DIR|g; s|__USER__|$USER|g" \
        "$plist" > "$PLIST_DIR/$name"
    launchctl unload "$PLIST_DIR/$name" 2>/dev/null || true
    launchctl load "$PLIST_DIR/$name"
    echo "Installed: $name"
done

echo "Setup complete."
```

**Critical rule:** Use directory-level symlinks for `.claude/commands/`, `.claude/skills/`, etc. Do NOT symlink individual files. New skills added to the dotfiles repo appear automatically without re-running setup.

# Chapter 3: The Dotfiles Framework

When you operate one machine, configuration lives in your head. You know where things are, what is installed, and how the shell behaves because you set it up once and you use it every day. When you operate three machines -- or five, or seven -- that mental model collapses. A setting changed on your workstation at home does not follow you to your office machine. A backup script improved on one box stays improved on that box alone. You install a new Claude Code skill on Monday and forget to copy it to the other machines by Wednesday. Within a few weeks, your fleet is a patchwork of slightly different configurations that break in slightly different ways.

A dotfiles repository eliminates this drift. It is a single git repo that holds every shared configuration file, script, scheduled task, and behavioral instruction. You commit on one machine, push to GitHub, pull on the others, and every machine is identical. This chapter explains the specific dotfiles framework that powers the multi-machine infrastructure described in this guide -- its directory structure, its installer, its sync engine, and the hard-won lessons that shaped its design.

---

## What Lives in the Dotfiles Repo

The repository lives at `~/dotfiles/` on every machine and is hosted on GitHub. It contains six categories of configuration:

| Category | Purpose | Examples |
|----------|---------|----------|
| Claude Code config | Behavioral instructions, permissions, 30+ custom skills | CLAUDE.md, settings.json, commands/ |
| SSH configuration | Connection templates and authorized keys | config.template, authorized_keys.master |
| Backup scripts | The entire backup chain (6 scripts + shared helper) | mac-backup.sh, vps-backup.sh, offsite-sync.sh |
| Scheduled tasks | macOS LaunchAgents and LaunchDaemons | 22 plists for backups, syncs, health checks |
| Bootstrap scripts | New machine provisioning from scratch | bootstrap-mac.sh, bootstrap-pi.sh, Brewfile |
| Documentation | The single source of truth for the entire infrastructure | INFRASTRUCTURE.md (~1054 lines) |

---

## The Full Directory Tree

Understanding the layout is essential before diving into the mechanics. Here is the complete structure:

```
dotfiles/
├── .claude/                 # Claude Code config (symlinked into ~/.claude/)
│   ├── CLAUDE.md            # Global behavioral instructions
│   ├── settings.json        # Permissions, hooks, model preferences
│   ├── settings.local.json.example  # Machine-specific template
│   ├── config/
│   │   └── infrastructure.sh  # Centralized machine data + helper functions
│   ├── commands/             # 30+ custom slash commands (skills)
│   │   ├── agents/           # Subagent definitions
│   │   ├── gsd/              # Get Shit Done framework (22 commands)
│   │   ├── deck/             # Presentation builder
│   │   ├── bot-audit/        # Bot health assessment
│   │   ├── bot-ingest/       # RAG ingestion pipeline
│   │   ├── bot-eval/         # Bot testing
│   │   ├── bot-scaffold/     # Bot overhaul scaffolding
│   │   ├── bot-refresh/      # Quarterly maintenance
│   │   ├── docx/             # Word document generation
│   │   ├── pptx/             # PowerPoint generation
│   │   ├── xlsx/             # Excel generation
│   │   ├── pdf/              # PDF handling
│   │   ├── gov-factory/      # Government automation
│   │   ├── gov-forms/        # Government form filling
│   │   ├── mcp-builder/      # MCP server scaffolding
│   │   ├── skill-creator/    # Skill creation
│   │   ├── site-manager/     # Presentation site management
│   │   ├── media/            # Media pipeline
│   │   ├── obsidian-markdown/ # Obsidian vault editing
│   │   ├── audit.md          # Infrastructure audit
│   │   ├── sync.md           # Cross-machine sync
│   │   ├── write.md, edit.md, review.md, research.md  # Writing tools
│   │   └── transcript.md     # YouTube transcript processing
│   ├── docs/                 # Reference documentation
│   │   ├── anti-patterns.md
│   │   ├── backup-strategy.md
│   │   ├── file-organization.md
│   │   ├── obsidian-safety-rules.md
│   │   ├── rag-quality-checklist.md
│   │   ├── research-routing.md
│   │   ├── skill-triggers.md
│   │   ├── writing-standards.md
│   │   └── INFRASTRUCTURE.md (symlink → dotfiles/docs/)
│   ├── get-shit-done/        # GSD templates, references, workflows
│   ├── scripts/              # 24 utility scripts
│   │   ├── dotfiles-sync.sh  # Cross-machine sync engine
│   │   ├── log-permission-request.sh  # PreToolUse hook
│   │   ├── ql-markdown.sh    # PostToolUse hook (Quick Look preview)
│   │   └── auto-rename-sessions.sh, quick-audit.sh, etc.
│   └── skills/               # n8n suite, audit, sync skills
├── backup/                   # Backup system (6 scripts + shared helper)
│   ├── backup-common.sh      # send_backup_alert(), format_duration(), format_size()
│   ├── mac-backup.sh         # Daily Mac → ServerM2P
│   ├── vps-backup.sh         # VPS → ServerM2P
│   ├── pi-backup.sh          # Pi backup (from ServerM2P)
│   ├── offsite-sync.sh       # ServerM2P → Google Drive (rclone)
│   └── backup-health-check.sh # Daily/weekly health checks
├── bootstrap/                # New machine setup
│   ├── bootstrap-mac.sh      # 10-step Mac provisioning
│   ├── bootstrap-pi.sh       # Role-based Pi setup
│   ├── bootstrap-vps.sh      # VPS provisioning
│   └── Brewfile              # Homebrew package manifest
├── docs/                     # Infrastructure documentation
│   ├── INFRASTRUCTURE.md     # Single source of truth (~1054 lines)
│   ├── INFRASTRUCTURE-CHANGELOG.md
│   ├── AUDIO.md
│   ├── PI-APPLICATIONS.md
│   ├── SMARTHOME.md
│   ├── backup-strategy.md
│   └── history/              # Per-machine infrastructure history
├── git-hooks/
│   └── post-commit           # Auto-push after every commit
├── launchagents/             # 22 macOS LaunchAgent plists
├── launchdaemons/            # 3 macOS LaunchDaemon plists
├── logs/                     # Log sync system (4 scripts)
├── mcp-configs/              # Per-machine MCP config templates
│   ├── studiom4.json
│   ├── officem4p.json
│   ├── serverm2p.json
│   └── vps.json
├── scripts/                  # Utility scripts
│   ├── is-home.sh            # Home network detection (used by SSH)
│   ├── media-pipeline.sh     # VPS media hosting pipeline
│   └── check-mesh.sh, network-scan.sh, etc.
├── shell/
│   └── zshenv                # ~/.zshenv (PATH, certificates)
├── ssh/
│   ├── config.template       # SSH config template
│   └── authorized_keys.master
├── vscode/
│   └── settings.json         # VS Code settings
├── .sync-state/              # Cross-machine sync state tracking
├── setup.sh                  # Main installer
└── README.md
```

This is a substantial tree, and it grew organically over months. You do not need to build all of this on day one. Start with `.claude/`, `backup/`, `bootstrap/`, and `scripts/`, then expand as your needs grow.

---

## setup.sh -- The Main Installer

The installer is the most important file in the repository. It transforms a bare machine into a fully configured member of the fleet. Here is what each step does, in order.

### Step-by-Step Walkthrough

**Step 1: Create the base directory.**
Create `~/.claude/` if it does not exist. This is where Claude Code looks for its configuration on every machine.

**Step 2: Symlink top-level config files.**
Create symlinks for `CLAUDE.md`, `settings.json`, and `README.md` from `~/dotfiles/.claude/` into `~/.claude/`. These are individual file symlinks -- editing either the source or the target edits the same file.

**Step 3: Create directory-level symlinks.**
This is the most important step. Instead of symlinking individual files inside `commands/`, `skills/`, `docs/`, `scripts/`, and `get-shit-done/`, the installer symlinks the entire directory:

```bash
ln -sfn ~/dotfiles/.claude/commands ~/.claude/commands
ln -sfn ~/dotfiles/.claude/skills   ~/.claude/skills
ln -sfn ~/dotfiles/.claude/docs     ~/.claude/docs
ln -sfn ~/dotfiles/.claude/scripts  ~/.claude/scripts
ln -sfn ~/dotfiles/.claude/get-shit-done ~/.claude/get-shit-done
```

The `-n` flag is critical. It tells `ln` to treat an existing symlink to a directory as a file to be replaced, rather than creating a symlink inside the target directory. Without `-n`, running setup.sh twice would create `~/.claude/commands/commands` instead of replacing the symlink.

**Step 4: Copy settings.local.json.example.**
If `~/.claude/settings.local.json` does not already exist, copy the example file. This file holds machine-specific overrides (things that should differ between machines, like local file paths or machine-specific permissions). It is never committed to the dotfiles repo.

**Step 5: Symlink VS Code settings.**
Link VS Code's `settings.json` from the dotfiles repo so editor configuration is also shared.

**Step 6: Set global git hooks path.**
```bash
git config --global core.hooksPath ~/dotfiles/git-hooks
```
This tells git to use the hooks in the dotfiles repo for every git repository on the machine. The post-commit hook (covered below) auto-pushes dotfiles changes.

**Step 7: Symlink SSH config and shell environment.**
Link `~/.ssh/config` from the template and `~/.zshenv` from the shell directory.

**Step 8: Install machine-specific LaunchAgents.**
The installer detects the hostname and installs the appropriate set of LaunchAgent plist files:

| Machine | LaunchAgents |
|---------|-------------|
| All Macs | backup, log-sync, session-rename |
| ServerM2P only | MCP servers, offsite backup, health check, log aggregate, MAINDRIVE monitor |
| OfficeM4P only | nightly audit, weekly audit, Granola MCP proxy, GIS mount |
| StudioM4 only | phone scan, GIS mount |

**Step 9: Template LaunchAgent paths.**
LaunchAgent plist files require absolute paths -- macOS's `launchd` does not expand `$HOME` or `~` in plist files. The installer solves this with placeholders:

```xml
<!-- In the plist template -->
<string>__HOME__/dotfiles/backup/mac-backup.sh</string>
```

At install time, setup.sh replaces `__HOME__` and `__USER__` with the actual values for the current machine. The resulting plist is copied (not symlinked) into `~/Library/LaunchAgents/` because it now contains machine-specific absolute paths.

**Step 10: Validate all symlinks.**
The installer checks every expected symlink and reports any that are broken or missing. This catches problems immediately instead of letting them surface hours later when a script fails.

---

## Directory-Level Symlinks -- Why This Matters

This design choice deserves its own section because it is the single most important architectural decision in the dotfiles repo, and because getting it wrong causes subtle, infuriating failures.

### The Problem with File-Level Symlinks

If you symlink individual files, adding a new skill requires three steps:

1. Create the skill file in `~/dotfiles/.claude/commands/`
2. Run setup.sh (or manually create the symlink) on every machine
3. Hope you do not forget a machine

With directory-level symlinks, adding a new skill requires one step:

1. Create the skill file in `~/dotfiles/.claude/commands/`

It appears on every machine the next time `git pull` runs. No setup.sh re-run needed.

### The Critical Gotcha

Never run `mkdir -p ~/.claude/commands/` on a machine where that path is a symlink. The `mkdir -p` command will silently remove the symlink and create a real directory. Everything in the dotfiles repo disappears from that machine's view, replaced by an empty directory. No error message. No warning.

The sync script (`dotfiles-sync.sh`) includes an auto-repair step that detects real directories where symlinks should exist and converts them back. But prevention is better than repair: train yourself to never create directories inside `~/.claude/` manually.

### How to Verify

Check that a path is a symlink, not a real directory:

```bash
# Good -- this is a symlink
$ ls -la ~/.claude/commands
lrwxr-xr-x  1 user  staff  38 Feb 27 10:00 commands -> /Users/user/dotfiles/.claude/commands

# Bad -- this is a real directory
$ ls -la ~/.claude/commands
drwxr-xr-x  15 user  staff  480 Feb 27 10:00 commands
```

The `l` at the beginning of the permissions string tells you it is a symlink. A `d` means it is a real directory and something has gone wrong.

---

## bootstrap-mac.sh -- New Machine Setup

When a brand new Mac arrives, this script takes it from factory state to fully operational. It runs ten steps:

| Step | Action | What It Does |
|------|--------|-------------|
| 1 | Xcode CLI Tools | Installs the compiler toolchain that Homebrew and many packages depend on |
| 2 | Homebrew | Installs the macOS package manager itself |
| 3 | Brewfile | Installs all packages from the manifest (see next section) |
| 4 | setup.sh | Runs the dotfiles installer described above |
| 5 | SSH config | Deploys SSH keys and config from the dotfiles repo |
| 6 | LaunchAgents | Installs and loads the appropriate scheduled tasks for this machine |
| 7 | Required directories | Creates standard directory structure (`~/Documents/Work/`, `~/.local/logs/`, etc.) |
| 8 | GIS Python venv | Sets up a Python virtual environment for geospatial tools (project-specific) |
| 9 | Syncthing | Configures file sync between machines (for non-git content like media) |
| 10 | Post-setup reminders | Prints a list of manual steps (sign into accounts, configure Obsidian, etc.) |

The bootstrap script is idempotent -- you can run it again without damage. Each step checks whether its work has already been done before proceeding.

There are also `bootstrap-pi.sh` (which accepts a `--role` flag for netsentry or alertnode Pis) and `bootstrap-vps.sh` (for provisioning the VPS), but the Mac bootstrap is the most complex because macOS machines carry the most configuration.

---

## Brewfile -- The Package Manifest

A Brewfile is Homebrew's way of declaring every package, cask (GUI application), and VS Code extension that should be installed on a machine. Instead of running `brew install` thirty times, you run `brew bundle` once and it installs everything in the manifest.

### Key Packages

| Category | Packages |
|----------|----------|
| Core CLI | git, gh (GitHub CLI), jq, curl, rsync |
| Languages | node, python@3.12, uv (Python package manager), pipx |
| Infrastructure | nmap, rclone, tailscale |
| Document tools | pandoc |
| AI tooling | claude-code |
| Editor extensions | 36 VS Code extensions (linting, formatting, language support) |

The Brewfile lives in `~/dotfiles/bootstrap/Brewfile` and is version-controlled like everything else. When you need a new tool on all machines, add it to the Brewfile, commit, sync, and run `brew bundle` on each machine.

### Why a Brewfile Instead of a Script

A shell script that runs `brew install git && brew install jq && ...` works, but it has two problems. First, it is slow -- each `brew install` invocation resolves dependencies independently. `brew bundle` resolves them all at once. Second, a Brewfile is declarative: it describes the desired state, not the steps to get there. If a package is already installed, `brew bundle` skips it. If a package was removed from the Brewfile, `brew bundle --cleanup` uninstalls it. The Brewfile is the truth; the machine converges to match.

---

## Cross-Machine Sync (dotfiles-sync.sh)

The sync script is the engine that keeps all machines identical. It is invoked by the `/sync` slash command in Claude Code, but it is also a standalone bash script that can be run manually.

### What It Does, Step by Step

**1. Source infrastructure data.**
The script sources `~/.claude/config/infrastructure.sh`, which contains hostnames, IP addresses, SSH aliases, and helper functions for every machine in the fleet. This means the sync script does not hardcode any machine names or addresses.

**2. Connectivity check.**
SSH connection tests run in parallel against the server, office machine, and VPS. Unreachable machines are noted but do not block the sync -- the script syncs what it can reach.

**3. Validation.**
Before committing anything, the script checks for problems:
- Merge conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) in any tracked file
- Invalid JSON in any `.json` file
- Broken symlinks in critical paths

If validation fails, the sync stops. This prevents pushing broken configuration to other machines.

**4. Local commit.**
```bash
git add -A
git commit -m "sync: $(hostname) $(date +%Y-%m-%d-%H%M)"
```
The commit message includes the hostname and timestamp so you can trace which machine made which change.

**5. Push to origin.**
The script pushes to GitHub. If the branch has diverged (because another machine pushed since the last sync), it runs `git pull --rebase` first to incorporate remote changes, then pushes again.

**6. Pull on remotes.**
For each reachable machine, the script SSHs in and runs a pull sequence:
```bash
ssh machine "cd ~/dotfiles && git stash && git pull --rebase && git stash pop"
```
The `stash`/`pop` handles the case where a remote machine has uncommitted local changes (perhaps a settings.local.json edit that should not be committed).

**7. Verification.**
After all pulls complete, the script checks that every reachable machine has the same git hash at HEAD. If any machine is behind, it reports the discrepancy.

### Modes

| Flag | Behavior |
|------|----------|
| (none) | Full sync: commit, push, pull all remotes, verify |
| `--dry-run` | Show what would happen without making any changes |
| `--rollback` | Revert the last sync commit on all machines |
| `--force` | Skip validation checks (use with caution) |

---

## Post-Commit Auto-Push Hook

The `git-hooks/post-commit` file is a git hook that runs after every commit in the dotfiles repo. Its job is simple: push to origin immediately.

```bash
#!/bin/bash
# Auto-push dotfiles after every commit
git push origin "$(git branch --show-current)" 2>/dev/null &
```

The push runs in the background (`&`) so it does not block your terminal. It is silent on success. This means the workflow is:

1. Edit a config file on StudioM4
2. Commit (manually or via `/sync`)
3. The hook auto-pushes to GitHub
4. Other machines can now `git pull` to get the change

Combined with the sync script, this creates a fast propagation loop: commit on one machine, and within seconds the change is available on GitHub for other machines to pull.

### Why Not Auto-Pull?

You might wonder why remote machines do not auto-pull on a schedule. The reason is safety. An auto-pull could overwrite local work in progress or introduce a broken config at an inconvenient time (mid-backup, mid-deployment). The sync script pulls explicitly, with stash protection and verification. The auto-push hook handles the "I made a change and want it available" direction. The "I want to consume a change" direction stays intentional.

---

## Portable Path Rules

This section exists because of a real incident: nine files in the dotfiles repo contained hardcoded paths like `/Users/slate/Documents/...`, and every one of them broke when pulled to a machine with a different username. The fix was straightforward, but the lesson was worth codifying into rules.

### The Rules

| Context | Correct Syntax | Example |
|---------|---------------|---------|
| Markdown / YAML files | `~/` | `~/Documents/Work/Deliverables/` |
| Bash code blocks inside `.md` | `$HOME` | `$HOME/Documents/GitHub/` |
| JavaScript | `require('os').homedir()` | `path.join(os.homedir(), 'dotfiles')` |
| Shell scripts (`.sh`) | `${HOME}` or `$HOME` | `${HOME}/dotfiles/backup/mac-backup.sh` |

### What to Never Do

Never hardcode a username into a path:

```bash
# WRONG -- breaks on any machine with a different username
/Users/slate/dotfiles/backup/mac-backup.sh

# RIGHT -- works everywhere
${HOME}/dotfiles/backup/mac-backup.sh
```

This applies to every file in the dotfiles repo: scripts, documentation, skill definitions, LaunchAgent templates (which use the `__HOME__` placeholder instead), and configuration files. The audit skill (`/audit`) now includes a check that greps for hardcoded usernames across the entire repo.

---

## Reproduction Steps

You do not need seven machines to benefit from a dotfiles repo. Even two machines make the investment worthwhile. Here is how to build your own, modeled on the framework described in this chapter.

### Step 1: Create the Repository

```bash
mkdir ~/dotfiles
cd ~/dotfiles
git init
```

Create the core directory structure:

```bash
mkdir -p .claude/commands .claude/docs .claude/scripts .claude/config
mkdir -p backup bootstrap scripts ssh shell
```

### Step 2: Write Your CLAUDE.md

This is your behavioral instruction file for Claude Code. Place it at `~/dotfiles/.claude/CLAUDE.md`. Chapter 7 covers how to write one in detail. For now, start with the basics: your preferred communication style, your priority order (security > research > action > documentation), and any hard rules you want Claude to follow.

### Step 3: Create settings.json

This file controls Claude Code permissions and hooks. Place it at `~/dotfiles/.claude/settings.json`. A minimal starting point:

```json
{
  "permissions": {
    "allow": ["Read", "Glob", "Grep", "WebSearch"],
    "deny": []
  }
}
```

### Step 4: Write setup.sh

Start with the essentials and expand later. A minimal installer:

```bash
#!/bin/bash
set -euo pipefail

DOTFILES="${HOME}/dotfiles"

# Create base directory
mkdir -p "${HOME}/.claude"

# Symlink top-level config
ln -sf "${DOTFILES}/.claude/CLAUDE.md" "${HOME}/.claude/CLAUDE.md"
ln -sf "${DOTFILES}/.claude/settings.json" "${HOME}/.claude/settings.json"

# Directory-level symlinks (the important part)
ln -sfn "${DOTFILES}/.claude/commands" "${HOME}/.claude/commands"
ln -sfn "${DOTFILES}/.claude/docs"     "${HOME}/.claude/docs"
ln -sfn "${DOTFILES}/.claude/scripts"  "${HOME}/.claude/scripts"

# SSH config
ln -sf "${DOTFILES}/ssh/config.template" "${HOME}/.ssh/config"

# Git hooks
git config --global core.hooksPath "${DOTFILES}/git-hooks"

echo "Setup complete."
```

### Step 5: Write a Brewfile

List every package you want on all machines:

```ruby
# ~/dotfiles/bootstrap/Brewfile
brew "git"
brew "gh"
brew "jq"
brew "curl"
brew "rsync"
brew "node"
brew "python@3.12"
```

### Step 6: Create the Post-Commit Hook

```bash
mkdir -p ~/dotfiles/git-hooks
cat > ~/dotfiles/git-hooks/post-commit << 'EOF'
#!/bin/bash
git push origin "$(git branch --show-current)" 2>/dev/null &
EOF
chmod +x ~/dotfiles/git-hooks/post-commit
```

### Step 7: Run setup.sh on Your Primary Machine

```bash
cd ~/dotfiles
chmod +x setup.sh
./setup.sh
```

Verify the symlinks:

```bash
ls -la ~/.claude/commands  # Should show -> ~/dotfiles/.claude/commands
ls -la ~/.claude/CLAUDE.md # Should show -> ~/dotfiles/.claude/CLAUDE.md
```

### Step 8: Push and Clone to Additional Machines

```bash
# On the primary machine
cd ~/dotfiles
git add -A
git commit -m "Initial dotfiles setup"
git remote add origin git@github.com:YOUR_USER/dotfiles.git
git push -u origin main

# On each additional machine
git clone git@github.com:YOUR_USER/dotfiles.git ~/dotfiles
cd ~/dotfiles
./setup.sh
```

### Step 9: Set Up Cross-Machine Sync

Once you have two or more machines, write a sync script (or use the `/sync` pattern described above). At minimum, it should:

1. Commit local changes
2. Push to GitHub
3. SSH into each remote machine and pull

Start simple. Add validation, parallel connectivity checks, and rollback support as your needs grow.

---

## Gotchas and Lessons Learned

These are the problems that cost hours to debug. Learn from them instead of repeating them.

### The mkdir Trap

**Problem:** Running `mkdir -p ~/.claude/commands/` on a machine where `commands` is a symlink silently replaces the symlink with a real, empty directory. All skills vanish. No error message.

**Fix:** Never create directories inside `~/.claude/` manually. If a symlink is broken, re-run setup.sh or recreate the symlink with `ln -sfn`. Add an auto-repair step to your sync script that checks for real directories where symlinks should exist.

### LaunchAgent Path Expansion

**Problem:** macOS `launchd` does not expand `$HOME` or `~` in plist files. A plist containing `~/dotfiles/backup/mac-backup.sh` will fail silently -- launchd will try to find a literal directory called `~` in the filesystem root.

**Fix:** Use placeholder tokens like `__HOME__` in your plist templates and replace them with actual paths at install time in setup.sh. Copy (do not symlink) the resulting plists into `~/Library/LaunchAgents/`.

### The Skill Token Bloat Problem

**Problem:** Claude Code discovers skills by recursively scanning all markdown files under `~/.claude/commands/`. If you install a skill's dependencies (like `node_modules/`) inside that directory, every markdown file in `node_modules/` becomes a "skill entry." A single `node_modules/` folder can add hundreds of phantom skills, wasting approximately 6,000 tokens per turn across every conversation.

**Fix:** Keep skill dependencies outside the scan path. Use `~/.claude/lib/<skill-name>/` for `node_modules/`, `.venv/`, or any other dependency directory. The skill's code references dependencies using a resolver script that adjusts the module path at runtime.

### The .sync-state Directory

**Tip:** The `.sync-state/` directory tracks the last-sync timestamp for each machine. When debugging why a machine is out of date, check this directory first. It will tell you when each machine last successfully synced and whether the sync completed or failed partway through.

### The settings.local.json Pattern

**Tip:** Use `settings.local.json` (not tracked by git) for machine-specific overrides. This is where per-machine differences live -- file paths that differ between machines, machine-specific tool permissions, or local-only hooks. The dotfiles repo ships a `settings.local.json.example` as a template. Actual `settings.local.json` files are gitignored.

---

## Summary

The dotfiles framework is the configuration backbone of the multi-machine infrastructure. It solves one problem -- keeping all machines identical -- and it solves it completely through six mechanisms:

1. **A single git repository** (`~/dotfiles/`) holds all shared configuration
2. **Directory-level symlinks** ensure new files appear on all machines without re-running the installer
3. **setup.sh** transforms a bare machine into a configured fleet member in one command
4. **bootstrap-mac.sh** provisions a brand-new machine from factory state
5. **dotfiles-sync.sh** propagates changes across all machines with validation and verification
6. **The post-commit hook** auto-pushes changes to GitHub so they are immediately available

The next chapters build on this foundation. Chapter 7 dives deep into CLAUDE.md -- the behavioral instruction file that setup.sh symlinks into place. Chapter 8 covers MCP configuration, which uses the `mcp-configs/` directory from this repo. Chapter 10 covers the backup scripts that live in the `backup/` directory. Every major system in this infrastructure traces back to a file in the dotfiles repo.

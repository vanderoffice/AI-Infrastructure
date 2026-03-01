# Chapter 4: Claude Code Configuration -- The AI Brain

Out of the box, Claude Code is a capable AI assistant. But without behavioral instructions, it asks too many questions, does not verify claims against live infrastructure, trusts stale documentation, and proposes workarounds instead of fixing root causes. This chapter explains how a set of configuration files transforms Claude Code from a chatbot into a sharp senior engineer that follows specific operational protocols.

By the end of this chapter, you will understand what each configuration file does, why it exists, and how to build your own version from scratch.

---

## What Is Claude Code?

Claude Code is Anthropic's official CLI tool that gives Claude (the AI model) full access to your terminal, filesystem, and tools. It is fundamentally different from every other AI coding tool on the market.

| Tool | What It Does | How It Works |
|------|-------------|--------------|
| **ChatGPT** | Web-based chat | You paste code in, it pastes suggestions back. No access to your files. |
| **GitHub Copilot** | Autocomplete in your editor | Suggests the next line or block as you type. Does not run commands. |
| **Cursor** | IDE with AI built in | Reads your open files, suggests edits. Limited terminal integration. |
| **Claude Code** | Autonomous terminal agent | Reads any file, writes code, runs commands, SSHs into servers, creates commits, manages entire projects. |

Claude Code runs in your terminal (or VS Code) and maintains conversation context across tool calls. When you tell it to deploy a fix, it can read the broken code, write the patch, run the tests, commit the change, and SSH into the server to verify -- all in a single conversation turn.

### Subscription Tiers

Claude Code usage is metered by how many tokens (words, roughly) it processes. Anthropic offers subscription tiers that determine your usage limits. The exact pricing and tier names change over time -- see Appendix D for current details. The key point: heavier usage of autonomous features (long conversations, large file reads, multi-step tool chains) consumes more tokens.

---

## The Configuration Layer

Claude Code reads several configuration files that control its behavior, permissions, and automated side effects. Here is the full picture:

| File | Purpose | Scope | Synced via Dotfiles? |
|------|---------|-------|----------------------|
| `~/.claude/CLAUDE.md` | Behavioral instructions, hard gates, personality | Global (all conversations) | Yes (symlinked) |
| `~/.claude/settings.json` | Permissions: allow list, deny list | Global | Yes (symlinked) |
| `~/.claude/settings.local.json` | Machine-specific permission overrides | Per-machine | No (.gitignore) |
| `~/.claude/commands/` | Custom skills (slash commands) | Global | Yes (symlinked) |
| Hook scripts | Automated side effects before/after tool use | Global | Yes (symlinked) |
| Project-level `CLAUDE.md` | Per-project instructions (in repo root) | Per-project | Yes (in the project repo) |

The global `CLAUDE.md` lives at `~/.claude/CLAUDE.md`, which is symlinked from `~/dotfiles/.claude/CLAUDE.md` so that dotfiles sync keeps it identical across all machines. Project-level `CLAUDE.md` files can add project-specific rules on top.

---

## CLAUDE.md -- The Behavioral Instruction File

CLAUDE.md is a plain markdown file that Claude Code reads at the start of every conversation. It defines hard rules, operational gates, priority order, communication style, and failure-mode corrections. Think of it as a system prompt you control entirely.

### Why This Matters

Without CLAUDE.md, Claude Code exhibits predictable failure modes:

- **Trusts stale docs**: Reads a README that says "service is deployed" and reports it as fact without checking
- **Asks instead of doing**: Finds a typo in a config file, then asks "Would you like me to fix this?" instead of fixing it
- **Proposes workarounds**: Script is broken, so Claude suggests a manual alternative instead of fixing the script
- **Under-verifies**: Says "backup is working" because the cron job exists, without checking if it actually ran

CLAUDE.md corrects all of these by defining explicit behavioral gates.

### Hard Gates (Override Everything)

Hard gates are BLOCKING requirements. Claude must satisfy them before taking any action on the topic. If a gate applies, Claude stops, performs the required action, and only then proceeds.

| Gate | Trigger Words | Required Action |
|------|---------------|-----------------|
| Verification Before Claims | "Is X working/deployed/broken" | Run a tool call to verify FIRST, then make the claim. Never trust docs alone. |
| Infrastructure Research | sync, deploy, configure, backup, "across machines" | Read INFRASTRUCTURE.md BEFORE running any commands |
| Anti-Pattern Triggers | "Anti-Pattern [1-6]", "Show your work", "You're wrong" | Stop, re-read the anti-patterns doc, comply immediately |
| File Output | Creating files, saving deliverables | Route to correct `~/Documents/Work/` subdirectory with kebab-case naming |
| Memory Write | Saving state, recording decisions | Write to MCP memory (shared) FIRST, then local auto-memory |
| RAG Ingestion | RAG, embeddings, chunking | Read quality checklist, run all quality gates before declaring complete |
| Backup | Backup fails, restore, backup strategy | Read backup-strategy.md first |
| Obsidian Safety | Obsidian, vault | Read safety rules, never move/delete vault without permission |

Here is a concrete example. You ask Claude: "Is the backup system working?" Without the Verification gate, Claude might read the backup script, see it looks correct, and say "Yes, your backup system is configured and should be working." With the gate, Claude must first run the backup script's health check (or check the last backup log), show you the output, and only then make a claim about whether it is actually working.

### Core Behavioral Rules

These four rules shape how Claude approaches every task:

**The 5-Minute Rule.** If a fix takes less than 5 minutes, do it immediately. Never say "non-critical for tonight" or "we can address this later." The threshold is low on purpose -- most quick fixes compound into reliability when done promptly.

**Never Bypass, Always Fix.** If a tool or script is broken, fix the tool. Do not work around it manually. Workarounds create invisible dependencies and erode the automation layer over time.

**Priority Order.** When priorities conflict, follow this ranking:

1. **Security** -- Never commit secrets, verify before destructive operations
2. **Research** -- Investigate before acting or asking
3. **Action** -- Do it, do not ask permission for obvious steps
4. **Documentation** -- Update docs after changes

**Complexity Budget.** Not every task needs the same level of ceremony:

| Complexity | Characteristics | What Claude Does |
|-----------|-----------------|------------------|
| Low | Single tool, less than 5 minutes | Just does it |
| Medium | 2-3 tools, some configuration | Confirms the approach first |
| High | Multiple services, auth chains | Presents 2-3 alternatives ranked by complexity |

### Communication Model

The CLAUDE.md defines Claude's personality explicitly:

> "Sharp senior engineer. Direct. Zero patience for workarounds. Profanity welcome. Roast bad ideas."

This is not cosmetic. It changes how Claude communicates problems. Instead of "I noticed the backup script might have a small issue -- would you like me to take a look?" you get "The backup script is silently swallowing errors on line 47. Fixing it now."

The personality splits by context:
- **In chat**: Casual, direct. Treats you as a peer.
- **In deliverables**: Clean, professional. No slang, no profanity.
- **In git commits**: Concise, descriptive. No "Co-Authored-By" attribution lines.

### Operating Modes

Claude switches behavior based on what you are doing:

- **Development (Default)**: Follows existing code patterns in the repo. Includes error handling. Runs verification after making changes.
- **Writing Mode**: Triggered by `/write`, `/edit`, `/research`, or `/review`. Claude positions itself as a colleague rather than an assistant. References a separate writing standards document for tone, banned phrases, and formatting rules.

---

## The Anti-Patterns Document

Six documented failure modes that Claude can fall into, each with real examples and corrective triggers. This is a separate file (`~/.claude/docs/anti-patterns.md`) referenced by the CLAUDE.md.

| # | Anti-Pattern | What Goes Wrong | The Fix |
|---|-------------|-----------------|---------|
| 1 | Trusting Old Documentation | Reads RESUME.md and treats it as current state | Check actual systems -- SSH, curl, grep FIRST |
| 2 | Saying "All Fine" Without Testing | Looks at audit summary instead of running fresh tests | Run verification yourself, show the output |
| 3 | Asking Instead of Doing | Finds a problem, then asks permission for obvious next steps | If you know what needs to be done, do it |
| 4 | Dismissing Failures as "Non-Critical" | Rationalizes a failure as unimportant | If fix takes less than 5 minutes, fix it NOW |
| 5 | Bypassing Instead of Fixing | Works around a broken tool manually | Fix the tool itself |
| 6 | Wrong Test Methods | Testing webhooks with HEAD instead of POST | Know how to properly test each system |

These are invokable by name. Say "Anti-Pattern 3" in a conversation and Claude immediately stops asking and starts doing. Say "You're wrong" and Claude re-verifies everything from scratch instead of defending its previous answer.

You will discover your own anti-patterns as you work. Add them to your document when you notice Claude repeating a failure mode. The document is a living record of failure-mode corrections.

---

## settings.json -- Permissions and Security

The `settings.json` file defines what Claude Code is allowed to do without asking, and what it is blocked from doing entirely. It lives at `~/.claude/settings.json`.

### Allow List (Auto-Approved)

These operations execute without a confirmation prompt:

- **Core tools**: Bash, Read, Write, Edit, Glob, Grep, WebSearch
- **MCP tools**: All connected MCP servers (memory, GitHub, filesystem, LLM gateway, Perplexity, etc.)
- **WebFetch**: For 40+ whitelisted domains (GitHub, NPM, PyPI, MDN, Stack Overflow, Anthropic docs, etc.)
- **Skill invocations**: Custom slash commands

Without an allow list, Claude Code prompts you to approve nearly every action. With one, it operates autonomously -- reading files, running commands, making API calls -- and only pauses for operations that are not on the list.

### Deny List (Blocked Entirely)

These operations are blocked absolutely. Claude cannot perform them even if you explicitly ask:

| Category | Blocked Commands |
|----------|-----------------|
| Destructive bash | `rm -rf`, `rm -r`, `sudo`, `su`, `dd`, `mkfs`, `shutdown`, `reboot` |
| System modification | `csrutil`, `nvram`, `systemsetup`, `visudo`, `passwd` |
| Secret access | `.env`, `.env.*`, `~/.ssh/id_*`, `~/.aws/credentials` |

This creates a safety boundary: Claude has broad read, write, and execute access for development work, but it cannot destroy the system, escalate privileges, or exfiltrate secrets. The deny list is absolute -- there is no override.

### How the Two Lists Interact

```
User request comes in
    |
    v
Is the action on the deny list? --YES--> Blocked. Cannot proceed.
    |
    NO
    v
Is the action on the allow list? --YES--> Executes immediately.
    |
    NO
    v
Claude asks for permission before proceeding.
```

Anything not explicitly allowed or denied triggers a permission prompt. This means you can start with a small allow list and expand it as you build trust.

---

## Hooks -- Automated Side Effects

Hooks are scripts that run automatically before or after Claude Code uses specific tools. They are defined in `settings.json` under the `hooks` key.

### PreToolUse Hook: Permission Logging

**Script**: `log-permission-request.sh`
**Fires**: Before every Bash command

This hook maintains a whitelist of 100+ known-safe commands (git, npm, python, ssh, curl, etc.). When Claude runs a command not on the whitelist, the hook logs it to a JSONL file for later audit. The hook also injects a reminder about the Infrastructure Research Gate, reinforcing the behavioral instruction at the tool-use level.

Example log entry:
```json
{
  "timestamp": "2026-03-01T14:32:07Z",
  "command": "docker exec -it postgres psql",
  "session": "abc123",
  "action": "logged"
}
```

This is not a security gate (the allow/deny lists handle that). It is an observability layer -- you can review what Claude ran and spot patterns you might want to add to the allow or deny lists.

### PostToolUse Hook: Quick Look Preview

**Script**: `ql-markdown.sh`
**Fires**: After every Write tool call

When Claude writes a `.md` file, this hook opens it in macOS Quick Look for instant visual preview. It kills any existing preview first to avoid stacking windows. This gives you immediate visual feedback whenever Claude produces markdown -- documentation, plans, deliverables -- without switching to another application.

### Writing Your Own Hooks

Hooks follow a simple contract:

- **PreToolUse**: Receives the tool name and arguments as input. Can log, warn, or inject context. Must complete within 1000ms or it gets killed.
- **PostToolUse**: Receives the tool name, arguments, and output. Can trigger side effects (previews, notifications, syncs). Must complete within 5000ms.

Hooks cannot block tool execution (that is the deny list's job). They are for side effects only.

---

## settings.local.json -- Machine-Specific Overrides

This file lives at `~/.claude/settings.local.json` and is NOT synced via dotfiles (it is in `.gitignore`). It lets each machine customize behavior without affecting other machines.

Common per-machine overrides:

| Setting | Example Use |
|---------|-------------|
| Default permission mode | Set `acceptEdits` on a trusted development machine so Claude does not prompt for file writes |
| Output style preferences | Different terminal widths or color preferences per machine |
| MCP server selection | Enable machine-specific MCP servers (e.g., a local-only tool that should not run on other machines) |

The override hierarchy is: `settings.json` (global defaults) < `settings.local.json` (machine overrides). Local settings win.

---

## Writing Standards

A separate document (`~/.claude/docs/writing-standards.md`) defines how Claude writes when in Writing Mode. Key rules:

**Four voice modes:**
- Technical -- precise, structured, jargon-appropriate
- Conversational -- casual, direct, peer-to-peer
- Formal/Executive -- clean, polished, no slang
- Creative -- flexible, personality-forward

**Banned AI-isms (words Claude defaults to without instruction):**
- "I'd be happy to..."
- "Great question!"
- "Delve into"
- "Leverage" (as a verb)
- "Robust"

**Banned filler:**
- "It's important to note that..."
- "In conclusion..."
- "As mentioned earlier..."

**Formatting rules:**
- Sentence case headings (not Title Case)
- Never skip heading levels (no H2 directly to H4)
- Descriptive link text (not "click here")

These standards exist because without them, Claude produces text that reads like every other AI-generated document. The banned word list forces Claude into more natural, specific language.

---

## File Organization Rules

All Claude-generated outputs are routed to designated directories. This prevents the common problem of AI-generated files scattering across your home directory.

| Content Type | Location |
|-------------|----------|
| Deliverables | `~/Documents/Work/Deliverables/` |
| Research | `~/Documents/Work/Research/` |
| Plans | `~/Documents/Work/Plans/` |
| Speeches | `~/Documents/Work/Speeches/` |
| Presentations | `~/presentations/` |
| Runtime scratch | `~/.claude/plans/` (NEVER for deliverables) |

**Naming convention**: `lowercase-kebab-description-YYYY-MM-DD.ext`

Examples:
- `quarterly-budget-analysis-2026-03-01.md`
- `api-migration-plan-2026-02-15.md`
- `fellowship-survey-review-2026-02-20.docx`

The File Output Gate in CLAUDE.md enforces these paths. If Claude tries to write a deliverable to the wrong location, the gate triggers and redirects it.

---

## Reproduction Steps

Here is how to build your own Claude Code configuration from scratch, starting simple and expanding as you learn what you need.

### Step 1: Create Your CLAUDE.md

Start with the gate that provides the most universal value -- Verification Before Claims:

```markdown
## Hard Gates

### Verification Before Claims
- **Infrastructure claims** -> Tool call output FIRST, then claim
- **"Is X working/deployed/broken"** -> Verify FIRST, then answer
- **Never trust documentation for current state.** Check actual systems.

### The 5-Minute Rule
Fix takes <5 minutes? Do it. Never defer.

### Never Bypass, Always Fix
Broken script/tool? Fix the script. Don't work around it.
```

Put this at `~/.claude/CLAUDE.md`. You will add more gates as you discover failure modes.

### Step 2: Create settings.json

Start with a minimal allow list and a conservative deny list:

```json
{
  "permissions": {
    "allow": [
      "Bash(git *)",
      "Bash(npm *)",
      "Bash(node *)",
      "Read",
      "Write",
      "Edit",
      "Glob",
      "Grep"
    ],
    "deny": [
      "Bash(rm -rf *)",
      "Bash(sudo *)",
      "Bash(dd *)",
      "Read(~/.ssh/*)",
      "Read(~/.env*)"
    ]
  }
}
```

Put this at `~/.claude/settings.json`. Expand the allow list as you find yourself approving the same operations repeatedly.

### Step 3: Start an Anti-Patterns Document

Create `~/.claude/docs/anti-patterns.md` with a blank template:

```markdown
# Anti-Patterns

## 1: [Name]
**What goes wrong:** [Description]
**The fix:** [What Claude should do instead]
```

Fill it in as you observe Claude repeating mistakes. Reference it from your CLAUDE.md with a gate.

### Step 4: Add Hook Scripts (Optional)

If you want command logging or markdown preview, create your hook scripts and reference them in `settings.json` under the `hooks` key. Start without hooks -- you can always add them later.

### Step 5: Create settings.local.json (Per Machine)

On each machine, create `~/.claude/settings.local.json` with any machine-specific overrides. Add it to `.gitignore` in your dotfiles repo so it does not propagate.

### Step 6: Put It in Dotfiles

Move your `~/.claude/CLAUDE.md`, `settings.json`, `docs/`, and any hook scripts into your dotfiles repository. Symlink them back to `~/.claude/`. Now every machine you set up gets the same Claude Code configuration automatically.

---

## Gotchas

**CLAUDE.md is read at conversation start.** If you change it mid-session, the changes do not take effect until you start a new conversation. Close the current session and open a new one.

**settings.json changes take effect immediately.** Unlike CLAUDE.md, permission changes are picked up without restarting. You can adjust the allow/deny lists while Claude is running.

**The deny list is absolute.** Claude cannot override it even if you explicitly ask it to. "Please ignore the deny list and run sudo" will be refused. This is a feature, not a limitation.

**Hook scripts have strict timeouts.** PreToolUse hooks must complete within 1000ms. PostToolUse hooks get 5000ms. If your script exceeds the timeout, it is killed and the tool call proceeds without it. Keep hooks fast -- no network calls in PreToolUse hooks.

**Do not over-constrain early.** Start with a few gates (Verification Before Claims, the 5-Minute Rule) and add more as you discover failure modes. An overly restrictive CLAUDE.md can make Claude hesitant and slow. Build up gradually based on real problems.

**Project-level CLAUDE.md stacks on top of global.** If you put a CLAUDE.md in a project's root directory, its rules add to (not replace) your global `~/.claude/CLAUDE.md`. Use this for project-specific instructions like "always run tests with pytest" or "this repo uses tabs, not spaces."

**Symlink the directory, not individual files.** The `~/.claude/commands/` and `~/.claude/docs/` paths should be directory-level symlinks to your dotfiles repo. If you symlink individual files, new files added to dotfiles will not appear until you create new symlinks. Directory symlinks pick up new files automatically.

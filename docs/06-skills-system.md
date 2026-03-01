# Chapter 6: Custom Slash Commands

Claude Code is general-purpose. It can read files, write code, run shell commands, and interact with external services. But "general-purpose" means it does not have deep expertise in your specific workflows. It does not know how you build presentations, how your chatbot RAG pipeline works, or what steps your infrastructure audit requires. You have to explain all of that from scratch every time.

Skills fix this. A skill is a custom slash command that loads specialized knowledge, workflows, and tool configurations into Claude Code on demand. Type `/deck` and Claude knows exactly how to build a Reveal.js presentation with your dark theme, your font stack, and your build pipeline. Type `/bot-audit` and it knows how to assess a chatbot's health across five dimensions. Type `/gsd:execute-plan` and it follows a structured project management workflow with phases, plans, and verification gates.

Skills turn Claude from a generalist into a specialist for each task. You teach it once, and it remembers every time.

---

## How Skills Work

### Discovery

Claude Code automatically scans all `.md` files in `~/.claude/commands/` recursively. Every markdown file it finds becomes a registered slash command. When you type `/` in a Claude Code session, you see the full list. When you invoke one, Claude reads that file's contents into its context window and follows the instructions.

This automatic discovery mechanism has one critical implication: everything under `commands/` that ends in `.md` becomes a skill. This is why dependencies like `node_modules/` or `.venv/` must never live under `commands/`. Claude would try to register every markdown file inside those directories -- README files, changelog files, license files -- as skills. In a real incident with the deck skill, a `node_modules/` directory under `commands/` added roughly 380 phantom skill entries and wasted approximately 6,000 tokens per turn across every conversation, whether or not the deck skill was invoked.

The fix is simple: dependencies go in `~/.claude/lib/<skill-name>/` instead. The `lib/` directory is outside the scan path. Claude never touches it.

### The Two Skill Formats

Skills come in two formats. Choose based on complexity.

**Single-file skills** are a single markdown file with optional YAML frontmatter. Good for focused workflows that fit in one document.

```
~/.claude/commands/audit.md
```

```markdown
---
name: audit
description: "Infrastructure audit"
allowed-tools:
  - Read
  - Bash
  - Write
---

# Infrastructure Audit

[Instructions for Claude when this skill is invoked...]
```

The frontmatter is optional but useful. `name` sets the display name. `description` appears in the skill list. `allowed-tools` restricts which tools Claude can use during the skill -- handy for read-only skills that should never write files.

**Directory-based skills** use a directory with `SKILL.md` as the entry point, plus supporting files organized by purpose.

```
~/.claude/commands/deck/
  SKILL.md              # Entry point (main instructions)
  references/           # Supporting reference documents
  templates/            # Output templates
  scripts/              # Helper scripts (Python, JS)
  docs/                 # Extended documentation
  examples/             # Example files
  assets/               # Static assets
```

When you invoke a directory-based skill, Claude reads `SKILL.md` first. That file tells Claude about the supporting files and when to reference them. Claude pulls in the additional files as needed rather than loading everything at once.

### When to Use Which Format

| Format | When to Use | Examples |
|--------|-------------|---------|
| Single-file | Simple workflows, style guides, checklists | `/audit`, `/write`, `/edit`, `/sync`, `/transcript` |
| Directory-based | Complex pipelines, multiple templates, helper scripts | `/deck`, `/gsd`, `/bot-audit`, `/docx`, `/gov-factory` |

A common pattern: start with a single-file skill, and convert to directory-based when it outgrows one document. The slash command name stays the same either way.

---

## The Complete Skill Inventory

This is the full set of skills in the infrastructure. You would not build all of these on day one. They accumulated over months of work, each one born from a workflow that was repeated often enough to justify codifying it.

### Project Management

| Skill | Type | Purpose |
|-------|------|---------|
| `/gsd:*` (22 commands) | Directory | Full project lifecycle -- init, plan, execute, verify, deploy. Covered in its own chapter. |

The GSD (Get Shit Done) system is the largest skill in the infrastructure. It manages structured project execution with phases, plans, checkpoints, and session handoffs. It gets its own chapter because it is more of a framework than a single skill.

### Document Generation

| Skill | Type | Purpose |
|-------|------|---------|
| `/docx` | Directory | Word document creation and editing via OOXML manipulation |
| `/pptx` | Directory | PowerPoint presentation generation |
| `/xlsx` | Directory | Excel spreadsheet creation with formulas and formatting |
| `/pdf` | Directory | PDF handling -- form filling, text extraction, annotation |

These skills teach Claude the internal XML structures of Microsoft Office formats. Instead of using a library that abstracts away the format, Claude works directly with the XML. This means it can do things most libraries cannot -- conditional formatting, precise layout control, form field population.

### Bot Lifecycle

| Skill | Type | Purpose |
|-------|------|---------|
| `/bot-audit` | Directory | Assess chatbot health across five dimensions: database, URLs, webhooks, knowledge, UI |
| `/bot-ingest` | Directory | RAG pipeline -- chunk documents, generate embeddings, insert into pgvector |
| `/bot-eval` | Directory | Adversarial testing with scoring rubrics and regression suites |
| `/bot-scaffold` | Directory | Generate GSD project files (PROJECT.md + ROADMAP.md) for a bot overhaul |
| `/bot-refresh` | Directory | Quarterly maintenance -- check for stale content, broken URLs, drifted thresholds |

These five skills cover the entire lifecycle of a RAG-powered chatbot. Scaffold sets up the project. Ingest builds the knowledge base. Audit checks health. Eval tests quality. Refresh keeps it current. They are designed to be used in sequence but work independently.

### Presentations

| Skill | Type | Purpose |
|-------|------|---------|
| `/deck` | Directory | Research-to-HTML presentation pipeline (Reveal.js, dark theme, interactive elements) |
| `/deck-components` | Directory | Reusable slide block components -- stat cards, timelines, comparison grids |
| `/site-manager` | Directory | Manage the presentation hub site (decks.json manifest, static HTML pages) |
| `/screenshot-annotator` | Directory | Screenshot capture with CSS annotation overlays |

The deck skill is the most complex single skill. It handles research, outline generation, slide design, build compilation, and deployment. The supporting `/deck-components` skill provides a library of pre-built slide patterns so presentations have visual consistency.

### Government Automation

| Skill | Type | Purpose |
|-------|------|---------|
| `/gov-factory` | Directory | End-to-end pipeline: domain research, presentation generation, knowledge base creation, app scaffolding |
| `/gov-forms` | Directory | Fill California government PDF forms with AI-generated justifications |

These are domain-specific skills for policy work. `/gov-factory` is an assembly line that takes a policy topic and produces a complete package: research summary, slide deck, chatbot knowledge base, and application scaffold. `/gov-forms` handles the tedious work of filling out standardized government forms.

### Infrastructure

| Skill | Type | Purpose |
|-------|------|---------|
| `/audit` | Single-file | Full infrastructure audit across all machines -- SSH connectivity, services, backups, certificates |
| `/sync` | Single-file | Cross-machine dotfiles synchronization (wraps `dotfiles-sync.sh`) |
| `/media` | Directory | Media optimization and VPS upload pipeline -- compress, upload, verify, list, clean |

### Development

| Skill | Type | Purpose |
|-------|------|---------|
| `/mcp-builder` | Directory | Guide for building MCP servers in Python or Node.js |
| `/skill-creator` | Directory | Guide for creating new skills with proper structure |
| `/agents` | Directory | Five subagent definitions for parallel work (see below) |

### Writing

| Skill | Type | Purpose |
|-------|------|---------|
| `/write` | Single-file | Document creation -- structure, tone, audience, formatting |
| `/edit` | Single-file | Document editing -- revision passes, consistency checks |
| `/review` | Single-file | Document review -- feedback, suggestions, scoring |
| `/research` | Single-file | Research workflow -- source discovery, synthesis, citation |

### Specialized

| Skill | Type | Purpose |
|-------|------|---------|
| `/obsidian-markdown` | Directory | Obsidian-flavored markdown -- wikilinks, callouts, dataview queries |
| `/transcript` | Single-file | YouTube video to formatted transcript with Obsidian summary note |
| n8n suite (7 skills) | Directory | n8n workflow patterns, code nodes, expressions, validation |

---

## The Subagent System

The `/agents` skill defines five specialized agents that Claude can spawn for parallel work. Each agent runs in its own context window using Claude's built-in Task tool. They execute simultaneously and return results to the main conversation.

| Agent | Role | Model Tier | Typical Concurrency | Use Case |
|-------|------|------------|---------------------|----------|
| Pathfinder | Find and trace | Fast (Haiku/Sonnet) | 3-5 parallel | Search codebases, find implementations, trace dependencies |
| Auditor | Check and verify | Strong (Opus) | 5-10 parallel | Infrastructure verification, endpoint checks, config audits |
| Crafter | Build and implement | Strong (Opus/Sonnet) | 1-3 parallel | Code implementation, file creation (low concurrency to avoid conflicts) |
| Synthesizer | Distill and adapt | Medium (Sonnet/Opus) | 1 per variant | Documentation, format conversion, content adaptation |
| Architect | Design and plan | Strong (Opus) | 1-2 parallel | System design, implementation planning, architecture decisions |

**How this works in practice.** Say you need to audit your infrastructure. Instead of checking each machine sequentially -- SSH to machine A, check services, SSH to machine B, check services -- Claude spawns five Auditor agents in parallel. Each one SSHes into a different machine, checks its services, and reports back. What would take ten minutes sequentially takes two minutes in parallel.

Or say you are researching a policy topic for a presentation. Claude spawns three Pathfinder agents: one searches government databases, one searches academic papers, one searches news coverage. They run simultaneously, and Claude synthesizes their findings into a single research brief.

The key constraint is conflict avoidance. Pathfinders and Auditors are read-only, so you can run many in parallel safely. Crafters write files, so you keep concurrency low (1-3) and assign each one to a different part of the codebase.

---

## The Skill Trigger Map

Skills are most useful when Claude knows to invoke them automatically. The skill trigger map is a file (`~/.claude/docs/skill-triggers.md`) referenced by CLAUDE.md that teaches Claude which skill to reach for based on what the user asks for.

| When the User Asks To... | Claude Invokes |
|---------------------------|----------------|
| Build a presentation or slide deck | `/deck` |
| Audit a chatbot | `/bot-audit` |
| Ingest documents into a knowledge base | `/bot-ingest` |
| Evaluate bot response quality | `/bot-eval` |
| Start a bot overhaul project | `/bot-scaffold` |
| Write, edit, or review prose | `/write`, `/edit`, `/review` |
| Do research | `/research` |
| Build an n8n workflow | `/n8n-workflow-patterns` |
| Build an MCP server | `/mcp-builder` |
| Fill government forms | `/gov-forms` |
| Process or upload media files | `/media` |
| Edit Obsidian notes | `/obsidian-markdown` |
| Create a new skill | `/skill-creator` |
| Manage the presentations site | `/site-manager` |
| Transcribe a YouTube video | `/transcript` |

Without this map, you would have to type `/deck` explicitly every time you want a presentation. With it, you can say "build me a presentation about water infrastructure funding" and Claude automatically invokes the deck skill before starting work.

---

## Anatomy of a Real Skill

Here is a simplified version of a single-file skill to show the structure. This is the `/audit` skill, which runs a full infrastructure check.

```markdown
---
name: audit
description: "Full infrastructure audit"
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
---

# Infrastructure Audit

## Purpose
Run a comprehensive health check across all machines in the infrastructure.

## Pre-Flight
1. Read ~/dotfiles/docs/INFRASTRUCTURE.md for current machine inventory
2. Read ~/.claude/docs/audit-checklist.md for the full checklist

## Audit Sections

### 1. SSH Connectivity
For each machine in the inventory, verify SSH access:
- `ssh server "hostname && uptime"`
- Record: reachable (yes/no), uptime, load average

### 2. Service Health
For each machine, check expected services:
- ServerM2P: Docker containers, MCP servers, Home Assistant
- VPS: nginx, Node.js apps, PostgreSQL, Docker
- NetSentry: Pi-hole, Unbound
- AlertNode: Pi-hole, ntfy

[... additional sections ...]

## Output
Write results to ~/Documents/Work/Deliverables/infra-audit-YYYY-MM-DD.md
```

When you type `/audit`, Claude reads this entire file into context. It then follows the instructions step by step -- reading the infrastructure doc, SSHing into machines, checking services, and writing the report. You do not need to remember the audit procedure. The skill remembers it for you.

A directory-based skill works the same way, but `SKILL.md` can reference additional files:

```markdown
# In SKILL.md
For slide component patterns, read references/components.md
For the build script, run scripts/build.js
For the output template, use templates/base.html
```

Claude reads `SKILL.md` first, then pulls in supporting files as the workflow requires them.

---

## How to Create New Skills

The `/skill-creator` skill guides you through the process, but here is the manual version.

### Step 1: Identify the Workflow

A good skill candidate is a workflow you repeat and explain to Claude more than twice. Common patterns:

- "Every time I do X, I have to explain these 5 steps"
- "Claude keeps forgetting how my build pipeline works"
- "I want Claude to follow this exact checklist every time"

### Step 2: Choose the Format

| Complexity | Format | Guideline |
|------------|--------|-----------|
| Fits in one document, no helper files | Single-file | Most skills start here |
| Needs templates, scripts, or reference docs | Directory-based | Convert when you outgrow single-file |

### Step 3: Write the Instructions

Write the skill as if you were onboarding a new engineer to your workflow. Include:

- **Purpose** -- what this skill does, in one sentence
- **Pre-flight checks** -- what to read or verify before starting
- **Steps** -- the actual workflow, in order
- **Tool constraints** -- which tools Claude should (or should not) use
- **Output** -- where results go and what format they should be in
- **Gotchas** -- things that commonly go wrong

### Step 4: Place the File

Put the file (or directory) in `~/.claude/commands/`. If your commands directory is symlinked from your dotfiles repo (recommended), the file goes in `~/dotfiles/.claude/commands/` and appears automatically.

```bash
# Single-file
cp my-skill.md ~/.claude/commands/my-skill.md

# Directory-based
mkdir -p ~/.claude/commands/my-skill/
cp SKILL.md ~/.claude/commands/my-skill/SKILL.md
```

### Step 5: Test It

Open a new Claude Code session and type `/my-skill`. Claude should show it in the autocomplete list. Invoke it and verify that Claude follows the instructions correctly. Iterate on the wording until the behavior matches your expectations.

---

## Reproduction Steps

If you are building this from scratch, here is the order of operations.

1. **Create the directory structure.** Set up `~/.claude/commands/` for skills and `~/.claude/lib/` for dependencies. If you are using a dotfiles repo, create the directories there and symlink.

```bash
# In your dotfiles repo
mkdir -p ~/dotfiles/.claude/commands
mkdir -p ~/dotfiles/.claude/lib

# Symlink into place
ln -sf ~/dotfiles/.claude/commands ~/.claude/commands
ln -sf ~/dotfiles/.claude/lib ~/.claude/lib
```

2. **Start with 2-3 single-file skills for your most common workflows.** Good first candidates: a writing style guide, a deployment checklist, a code review process.

3. **Add frontmatter for tool constraints.** If a skill should be read-only (like a review skill), use `allowed-tools` to restrict it.

4. **Convert to directory-based when a skill outgrows one file.** Move the content to `SKILL.md`, create subdirectories for templates, references, and scripts.

5. **Create a skill trigger map.** Write a file that maps task types to skills. Reference it from your CLAUDE.md so Claude learns to auto-invoke skills without explicit `/` commands.

6. **Keep dependencies out of `commands/`.** Any `node_modules/`, `.venv/`, or other dependency directories go in `~/.claude/lib/<skill-name>/`. This is non-negotiable.

7. **Sync across machines.** If you use multiple development machines, your dotfiles sync mechanism handles this. The symlink from `~/.claude/commands/` to `~/dotfiles/.claude/commands/` means a `git pull` on any machine updates all skills.

---

## Gotchas and Lessons Learned

These are problems that were encountered and solved during the development of this skill system. They will save you time.

**Token bloat from dependencies.** Claude Code scans all `.md` files under `commands/` recursively. A `node_modules/` directory can contain hundreds of markdown files (READMEs, changelogs, contributing guides). Each one gets registered as a skill, wasting tokens on every turn of every conversation. The fix: `~/.claude/lib/<skill>/` for all dependencies. There is no `.claudeignore` mechanism -- the only solution is keeping non-skill files out of the scan path.

**Skill size affects context.** When a skill is invoked, its full content loads into Claude's context window. A 10,000-word skill document eats into the space available for your actual conversation. Keep skills focused. Use directory-based format with `references/` for detailed documentation that Claude pulls in only when needed.

**SKILL.md is the entry point, not index.md or README.md.** For directory-based skills, Claude looks for `SKILL.md` specifically. If your entry point is named something else, Claude will still scan the directory but will not know which file to read first.

**Symlinks require directory-level linking.** The setup script creates a directory-level symlink: `~/.claude/commands/ -> ~/dotfiles/.claude/commands/`. This means new skills added to the dotfiles repo appear automatically on all machines after a sync. Do not create `~/.claude/commands/` as a real directory -- that breaks the symlink, and new skills silently stop appearing.

**Skills do not persist between sessions.** Each Claude Code session starts fresh. A skill invoked in one session does not carry over to the next. This is by design -- it keeps sessions clean. If you want behavior that persists across all sessions, put it in CLAUDE.md instead.

**The skill trigger map is a teaching tool, not a constraint.** Claude uses the trigger map as a hint. If you explicitly ask for something without using the slash command, the trigger map helps Claude recognize the intent and invoke the right skill. But Claude can still be asked to do anything -- the map just makes common workflows faster.

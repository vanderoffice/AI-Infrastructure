# Chapter 8: Cross-Session Continuity -- The Memory System

Claude Code has no memory between sessions. Every new conversation starts from zero -- it does not remember what you built yesterday, what decisions were made, or what is running on your infrastructure. This is a fundamental limitation that makes multi-session projects impossible without a memory system.

This infrastructure solves it with a two-tier memory architecture: a shared knowledge graph (MCP Memory) that works across all machines, and local auto-memory files that get injected into every conversation automatically.

---

## The Problem

Without memory, every session is a cold start. You open Claude Code on Monday morning and it has no idea that you spent Friday afternoon deploying a chatbot, resolving a DNS issue, or deciding to switch from Express to Fastify. The consequences are predictable and painful:

| Failure Mode | What Happens | Impact |
|-------------|-------------|--------|
| No project awareness | Claude asks "What project are you working on?" every session | You re-explain context for 5 minutes before getting to work |
| Repeated mistakes | Claude proposes a solution you already tried and rejected last week | Wasted time plus frustration |
| Lost infrastructure knowledge | Claude does not know your machine names, IP addresses, or service layout | Every infrastructure task starts with discovery instead of action |
| Cross-machine amnesia | You work on StudioM4 in the morning, switch to OfficeM4P in the afternoon -- context is gone | Decisions made on one machine are invisible on the other |
| GSD state evaporation | A multi-phase project's progress, current plan, and next steps disappear between sessions | You lose your place in complex projects constantly |
| Convention drift | Claude forgets your coding conventions, file organization rules, and tool preferences | Inconsistent output that requires manual correction |

Every one of these failure modes was encountered in production. The memory system exists because working without it was genuinely unworkable for anything beyond single-session tasks.

---

## Two-Tier Memory Architecture

The memory system has two layers. Each serves a different purpose, and both are necessary.

```
                    +--------------------------+
                    |    MCP Memory Server      |
                    |    (ServerM2P :8083)       |
                    |                            |
                    |  Knowledge Graph:           |
                    |  Entities + Relations +     |
                    |  Observations               |
                    +--------+---+---------------+
                             |   |
                    SSE      |   |      SSE
              +--------------+   +---------------+
              |                                  |
     +--------v--------+              +----------v------+
     |   StudioM4      |              |   OfficeM4P     |
     |                  |              |                 |
     | ~/.claude/       |              | ~/.claude/      |
     | projects/*/      |              | projects/*/     |
     |   memory/        |              |   memory/       |
     |     MEMORY.md    |              |     MEMORY.md   |
     |     topic1.md    |              |     topic1.md   |
     |     topic2.md    |              |     topic2.md   |
     +-----------------+              +-----------------+

     Tier 1: Shared graph             Tier 1: Same shared graph
     Tier 2: Local auto-memory        Tier 2: Local auto-memory
```

### Tier 1: MCP Memory (Shared Source of Truth)

MCP Memory is a knowledge graph that runs as an MCP server on the hub machine (ServerM2P, port 8083). Both dev machines access the same graph over the network via SSE.

**What it stores:**

| Concept | Description | Example |
|---------|-------------|---------|
| Entities | Named things in your system | `WaterBot_v2`, `ServerM2P`, `VPS_Hardening_Complete` |
| Observations | Facts about an entity, timestamped | `[2026-02-14] Production deploy complete, all 4 modes working` |
| Relations | How entities connect to each other | `WaterBot_v2 --deployed_on--> VPS` |

**The MCP tools Claude uses to interact with it:**

| Tool | Purpose |
|------|---------|
| `mcp__memory__create_entities` | Create new entities in the graph |
| `mcp__memory__add_observations` | Add facts to existing entities |
| `mcp__memory__create_relations` | Define connections between entities |
| `mcp__memory__search_nodes` | Find entities by keyword |
| `mcp__memory__read_graph` | Read the full graph (use sparingly) |
| `mcp__memory__open_nodes` | Open specific entities by name |
| `mcp__memory__delete_entities` | Remove entities |
| `mcp__memory__delete_observations` | Remove specific observations |
| `mcp__memory__delete_relations` | Remove relations |

**Example entity in the graph:**

```
Entity: WaterBot_v2
Type: Project
Observations:
  - [2026-02-14] Production deploy complete, all 4 modes working
  - [2026-02-14] Shared ChatMessage.jsx across all 3 bots
  - [2026-02-14] FundingNavigator: 58 programs, 5-step wizard
  - [2026-02-17] Bot overhaul complete, UI polish pass done

Relations:
  - WaterBot_v2 --deployed_on--> VPS
  - WaterBot_v2 --uses--> Supabase_pgvector
  - WaterBot_v2 --shares_components_with--> BizBot
  - WaterBot_v2 --shares_components_with--> KiddoBot
```

When Claude needs to recall project history, it queries the graph: `mcp__memory__search_nodes("WaterBot")` returns the entity with all its observations and relations. This works identically from StudioM4 or OfficeM4P because both machines connect to the same server.

**When to write to MCP Memory:**

- Project milestones reached (deploy, major feature complete)
- Architecture decisions made (chose Fastify over Express, switched from REST to WebSocket)
- Infrastructure changes (new server added, port changed, service migrated)
- New tools configured (MCP server added, skill created)
- Bugs resolved that took significant debugging time
- Workflow decisions (always use bun, never auto-commit)

**Naming convention:** Prefix entities with temporal context. `[2026-02-15] WaterBot_v2_Complete` is better than `WaterBot_Done` because six months from now you need to know when things happened.

### Tier 2: Local Auto-Memory (Per-Machine Cache)

Auto-memory is a set of markdown files that Claude Code automatically injects into every conversation. They live in a directory specific to the project path you are working in.

**Location:** `~/.claude/projects/<path-encoded>/memory/`

The project path is encoded by replacing slashes with dashes. For the home directory (`/Users/slate`), the path becomes `-Users-slate`. So the auto-memory directory is:

```
~/.claude/projects/-Users-slate/memory/
```

**How injection works:**

1. When Claude Code starts a session, it looks for a `MEMORY.md` file in the auto-memory directory matching your current working directory (or any parent directory).
2. The first ~200 lines of `MEMORY.md` are injected into the conversation context automatically.
3. Claude sees these lines at the start of every conversation without you doing anything.
4. Additional topic files in the same directory are available but only loaded when Claude references them -- they are not auto-injected.

**The directory structure for a mature setup:**

```
~/.claude/projects/-Users-slate/memory/
  MEMORY.md                    # Main file (auto-injected, ~200 line budget)
  waterbot.md                  # WaterBot v2.0 details
  cross-bot-standardization.md # Shared bot components
  bot-overhaul-plan.md         # Bot lifecycle skills
  bot-ui-polish.md             # Bot UI standards
  home-assistant.md            # Home Assistant / Zigbee setup
  deck-pipeline-gotchas.md     # Deck build pipeline issues
  local-tools.md               # Machine-specific tools and configs
  completed-projects.md        # Project archive
  completed-decks.md           # Deck inventory
  research-routing.md          # Cost routing rules
  youtube-pipeline.md          # YouTube content pipeline
  mcp-gateway-image.md         # Image generation via MCP Gateway
  govfellows-github.md         # Fellowship GitHub org details
```

**Key difference from MCP Memory:** Auto-memory files are local to each machine. If you update `MEMORY.md` on StudioM4, OfficeM4P does not see the change unless you manually sync the file. This is why MCP Memory is the source of truth and auto-memory is a cache.

---

## The Write Order Rule

The most important rule in the memory system is the write order. CLAUDE.md enforces this as a blocking gate -- meaning Claude must follow it before proceeding with any memory write operation.

### The Rule

1. **ALWAYS write to MCP Memory FIRST.** This is the shared source of truth that all machines can read.
2. **Then update local auto-memory.** This keeps the per-machine cache current for fast context loading.
3. **NEVER write ONLY to local auto-memory.** That creates cross-machine drift.

### Why This Matters

Consider what happens without this rule:

```
Monday afternoon, StudioM4:
  - You finish deploying WaterBot v2
  - Claude writes "WaterBot v2 deployed" to local MEMORY.md
  - Claude does NOT write to MCP Memory
  - Session ends

Tuesday morning, OfficeM4P:
  - You start a new session
  - Claude reads OfficeM4P's MEMORY.md -- no mention of WaterBot v2 deployment
  - You say "check the WaterBot deployment"
  - Claude has no idea it was deployed yesterday
  - You spend 10 minutes re-explaining what happened
```

With the write order rule:

```
Monday afternoon, StudioM4:
  - You finish deploying WaterBot v2
  - Claude writes to MCP Memory: WaterBot_v2 observation "[2026-02-14] Production deploy complete"
  - Claude updates local MEMORY.md as well
  - Session ends

Tuesday morning, OfficeM4P:
  - You start a new session
  - You say "check the WaterBot deployment"
  - Claude queries MCP Memory: search_nodes("WaterBot_v2")
  - Claude finds the deployment observation immediately
  - Work continues without interruption
```

The write order rule is the difference between a memory system that works and one that silently degrades as you switch machines.

---

## MEMORY.md Structure

The main auto-memory file follows a specific structure designed to maximize the value of the ~200 line injection budget. The most important information goes at the top because lines beyond the budget are not guaranteed to be in context.

### Section 1: CRITICAL Rules (Top of File)

These are blocking instructions that correct Claude's most damaging failure modes. They go first because they must always be in context.

```markdown
## CRITICAL: GSD -- Read the Workflow, Don't Improvise (BLOCKING)
**Trigger:** ANY /gsd:* command, GSD session start, GSD session end
- ALWAYS invoke GSD commands via the Skill tool so the workflow .md file loads.
- Never run GSD steps from memory.
```

Each CRITICAL section follows the same pattern:

| Element | Purpose | Example |
|---------|---------|---------|
| Title | What the rule is about | `GSD -- Read the Workflow, Don't Improvise` |
| `(BLOCKING)` tag | Signals this overrides other behavior | Always present on critical rules |
| `**Trigger:**` | When this rule activates | `ANY /gsd:* command, GSD session start` |
| Bullet points | The actual rules | `ALWAYS invoke via Skill tool`, `NEVER improvise from memory` |

Common CRITICAL sections and why they exist:

| Rule | Problem It Solves |
|------|-------------------|
| GSD workflow | Claude was improvising project management steps instead of loading the actual workflow file |
| Time estimation | Claude estimated "days/weeks" for work that actually takes minutes/hours with GSD |
| Right repo, right target | Claude was creating new git repos when work belonged in existing ones |
| Production safety | Claude was treating dev repos as deployment targets |
| zsh reserved variables | Claude used `path` and `status` as variable names, clobbering shell state |
| MCP config location | Claude looked for `~/.mcp.json` instead of `~/.claude.json` |
| Portable paths | Claude hardcoded `/Users/slate/` in configs that needed to work on multiple machines |
| Token bloat prevention | Claude put dependencies under `~/.claude/commands/`, bloating every session |
| Cost routing | Claude routed text generation through the LLM gateway API, double-billing |

Every one of these rules exists because the mistake actually happened. They are not hypothetical -- they are scar tissue from real incidents.

### Section 2: Active Projects

Current state of in-progress work. Each entry includes enough information for Claude to pick up where it left off.

```markdown
## Stem-to-MIDI Pipeline -- GSD INITIALIZED (2026-02-23)
- **Repo:** /Users/slate/projects/stem-to-midi/
- **GSD:** 6 phases, 12 plans, 0 completed | Phase 1 ready to plan
- **Resume:** cd /Users/slate/projects/stem-to-midi && /gsd:plan-phase 1
- **Key changes from research:** Omnizart killed, ServerM2P eliminated,
  audio-separator replaces raw demucs, BS-RoFormer SOTA
- **Stack:** audio-separator + Basic-Pitch + reathon + librosa + watchdog
```

The key elements: repo location, GSD status (phases/plans/completion), a resume command that picks up exactly where you left off, and any important decisions already made.

### Section 3: Infrastructure Knowledge

Stable facts about the system that change infrequently.

```markdown
## Backup Alerting -- OVERHAULED (2026-02-23)
- **Design:** Silence = healthy. No daily success pings. Failure-only.
- **Shared helper:** backup/backup-common.sh
- **All 6 scripts source backup-common.sh**
- **Priorities:** 4 = unreachable, 3 = partial, 2 = warning/digest
```

### Section 4: Completed Projects (Bottom of File)

A single line referencing a topic file, since completed project details rarely need to be in the main injection:

```markdown
## Completed Projects
ECOS, Claude Code Infra, VPS Hardening, AccessView v1.1, CDCR Plan.
See memory/completed-projects.md.
```

---

## What to Save vs. What NOT to Save

Not everything belongs in memory. Overfilling memory with low-value information wastes the ~200 line budget and buries the important rules.

### Save

| Category | Examples | Why |
|----------|---------|-----|
| Stable patterns | "Always use bun instead of npm" | Prevents Claude from proposing the wrong tool every session |
| Architectural decisions | "Chose pgvector over Pinecone for embeddings" | Prevents re-debating settled questions |
| Key file paths | "Production repo: /root/vanderdev-website/" | Prevents Claude from searching for things it should already know |
| Workflow preferences | "Never auto-commit", "Run tests before deploy" | Maintains consistent behavior |
| Solutions to recurring problems | "zsh clobbers PATH if you use `path` as a variable" | Prevents the same debugging session from happening twice |
| Explicit user requests | "Always format dates as YYYY-MM-DD" | User preferences must persist |

### Do NOT Save

| Category | Why Not |
|----------|---------|
| Session-specific context | "Currently debugging the login endpoint" -- irrelevant next session |
| Unverified conclusions | "The backup script probably runs at 3 AM" -- verify first, then save |
| Duplicate of CLAUDE.md | CLAUDE.md is already injected into every session |
| Speculative information | "This might be caused by a race condition" -- save after confirmed |
| Transient state | "Waiting for API key approval" -- save the outcome, not the wait |

---

## Session Hygiene

Session hygiene is the discipline of maintaining memory health across sessions. Without it, memory drifts, stales, and eventually becomes unreliable.

### Starting a GSD Session

When picking up a multi-phase project:

1. **Query MCP Memory** for `[ProjectName]_Session_Handoff` -- this contains the last session's state
2. **Read RESUME.md** (or `.continue-here.md`) in the project directory -- this has the exact pickup point
3. **Read the specific PLAN.md** being executed -- this has the current phase's instructions

This three-step startup ensures Claude has full context before writing a single line of code.

### Ending a GSD Session

1. **Run `/gsd:pause-work`** -- this writes `.continue-here.md` with the current state
2. **Verify MCP Memory was updated** -- any new knowledge from this session should be in the shared graph
3. **Update local auto-memory if needed** -- add new CRITICAL rules, update project status

### Memory Formatting Rules

- **Timestamp everything:** Prefix entries with `[YYYY-MM-DD]` so you can tell when information was recorded
- **Use CRITICAL tags:** Blocking rules get `## CRITICAL:` headers so they stand out
- **Include trigger words:** Each rule specifies what triggers it, so Claude knows when to apply it
- **Keep MEMORY.md under 200 lines:** Use topic files for details, reference them from the main file
- **Archive completed projects:** Move finished project sections to `completed-projects.md`

### When Claude Gets It Wrong

When you say "You're wrong" or "Dig deeper," Claude's required behavior is:

1. Stop assuming
2. Re-read the relevant docs (not memory -- the actual source files)
3. Verify against MCP Memory
4. SSH to investigate if the question involves infrastructure
5. Only then make a corrected claim

This is enforced by CLAUDE.md, but it is worth noting in the memory system context because bad information in memory propagates. If Claude wrote an incorrect observation to MCP Memory, every future session on every machine will inherit that mistake. Correcting memory is as important as writing to it.

---

## How It All Connects

Here is the full lifecycle of memory across sessions and machines:

```
Session Start (StudioM4, Monday morning)
  |
  +-- MEMORY.md auto-injected (~200 lines, automatic)
  |     +-- CRITICAL rules loaded (behavioral gates active)
  |     +-- Active projects loaded (knows what's in flight)
  |     +-- Infrastructure knowledge loaded (knows the system)
  |
  +-- Claude can query MCP Memory for deeper context
  |     +-- search_nodes("WaterBot") -> full entity with history
  |     +-- open_nodes(["VPS_Config"]) -> current VPS state
  |
  +-- /gsd:resume-work loads project state
        +-- Reads RESUME.md -> knows exact pickup point
        +-- Reads current PLAN.md -> knows what to do next

During Work
  |
  +-- Decision made (e.g., "switch from REST to WebSocket")
  |     +-- MCP Memory: add_observations to project entity [FIRST]
  |     +-- MEMORY.md: update project section [SECOND]
  |
  +-- Bug resolved (e.g., "zsh path variable clobbers PATH")
  |     +-- MCP Memory: create entity with observation [FIRST]
  |     +-- MEMORY.md: add CRITICAL rule [SECOND]
  |
  +-- GSD phase completed
        +-- MCP Memory: update project status
        +-- GSD files: STATE.md + SUMMARY.md updated by workflow

Session End (StudioM4, Monday evening)
  |
  +-- /gsd:pause-work -> writes .continue-here.md
  +-- Verify MCP Memory has all new knowledge
  +-- Update MEMORY.md if needed

Session Start (OfficeM4P, Tuesday morning)
  |
  +-- MEMORY.md auto-injected (OfficeM4P's local copy)
  |     +-- May not have Monday's MEMORY.md updates (local only)
  |     +-- BUT: CRITICAL rules and stable knowledge are there
  |
  +-- Claude queries MCP Memory
  |     +-- Finds Monday's decisions and progress [SHARED]
  |     +-- Full project history available
  |
  +-- /gsd:resume-work loads project state
        +-- RESUME.md was committed to git -> available everywhere
        +-- Work continues seamlessly
```

The key insight: MCP Memory bridges the machine gap. Local auto-memory provides fast, always-available context for rules and stable knowledge. Together they give Claude everything it needs to pick up where you left off, regardless of which machine you are on.

---

## Reproduction Steps

### 1. Set Up MCP Memory Server

If you followed Chapter 5, you already have this. If not, here is the quick version.

**Single machine (stdio -- simplest):**

Add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "memory": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic/memory-server"]
    }
  }
}
```

No server to run. Claude Code launches the memory process automatically and it persists its graph to disk.

**Multi-machine (SSE via hub):**

On your hub machine:

```bash
# Install and run with supergateway
npx -y supergateway --port 8083 -- npx -y @anthropic/memory-server
```

On each dev machine, add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "memory": {
      "type": "sse",
      "url": "http://<hub-ip>:8083/sse"
    }
  }
}
```

Make the hub process persistent with a LaunchAgent (macOS) or systemd service (Linux). See Chapter 5 for details.

### 2. Create Your Auto-Memory Directory

```bash
# Replace "yourname" with your actual username
mkdir -p ~/.claude/projects/-Users-yourname/memory/
```

If you work from a specific project directory (e.g., `/Users/yourname/projects/my-app`), you can create a project-specific memory directory as well:

```bash
mkdir -p ~/.claude/projects/-Users-yourname-projects-my-app/memory/
```

The more specific path takes precedence when you are working inside that directory.

### 3. Create Your First MEMORY.md

Start small. You do not need 200 lines on day one. Begin with the rules that matter most to your workflow.

```markdown
# Auto Memory

## CRITICAL: Verify Before Claiming (BLOCKING)
**Trigger:** "Is X working/deployed/broken"
- Run a verification command FIRST, then make the claim.
- Never trust docs or memory alone for infrastructure state.

## Active Projects
(None yet -- add as you start projects)

## Infrastructure
- **Primary machine:** [your machine name]
- **Key directories:** [your important paths]
```

### 4. Add the Memory Write Gate to CLAUDE.md

If you do not already have a CLAUDE.md, create one at `~/.claude/CLAUDE.md`. Add this gate:

```markdown
### Memory Write Gate (BLOCKING)
**Triggers:** Saving project state, session handoff, updating status, recording decisions
1. ALWAYS write to MCP memory FIRST -- this is the shared source of truth
2. Then update local auto-memory as a secondary cache
3. NEVER write ONLY to local auto-memory -- that creates cross-machine drift
```

Even on a single machine, this discipline is worth establishing. If you ever add a second machine, the habit is already in place.

### 5. Start Using Memory

After your first significant session, tell Claude to save what was learned:

```
Save the key decisions from this session to memory.
```

Claude will write to MCP Memory (entities and observations) and update MEMORY.md. Review what it wrote. Edit MEMORY.md if the phrasing is not concise enough or if it saved low-value information.

### 6. Create Topic Files as Knowledge Grows

When MEMORY.md approaches 200 lines, move detailed sections into topic files:

```bash
# Move WaterBot details to a topic file
# Keep a one-line reference in MEMORY.md:
# "## WaterBot v2.0 -- See memory/waterbot.md"
```

Topic files are loaded on demand when Claude references them, so they do not count against the 200-line injection budget.

### 7. Establish a Review Cadence

Once a month (or when memory feels stale), review MEMORY.md:

- Archive completed projects to `completed-projects.md`
- Remove rules for problems that no longer occur
- Update infrastructure knowledge that has changed
- Verify CRITICAL rules are still accurate

Memory is a living document. It requires maintenance, just like code.

---

## Gotchas

These are the problems you will encounter. Knowing them in advance saves real debugging time.

| Issue | Details | Fix |
|-------|---------|-----|
| MEMORY.md truncated after ~200 lines | Claude Code only injects the first ~200 lines into context. Everything below that may not be seen. | Keep MEMORY.md concise. Move details to topic files. Put CRITICAL rules at the top. |
| Path encoding in project directory | The path `/Users/slate` becomes `-Users-slate` in `~/.claude/projects/`. Slashes become dashes. | Check the exact directory name if MEMORY.md is not being injected. |
| MCP Memory server down | If the hub is unreachable, memory tools silently disappear. Claude falls back to local auto-memory only, with no error message. | If Claude seems to have forgotten shared knowledge, check if the hub is running: `curl http://<hub-ip>:8083/sse` |
| Cross-machine drift | Writing only to local MEMORY.md means the other machine never sees the update. This is the most common memory system failure. | Follow the write order rule religiously: MCP Memory first, then local. |
| Topic files not auto-loaded | Only MEMORY.md is auto-injected. Topic files (`waterbot.md`, `infrastructure.md`, etc.) are only read when Claude explicitly references them. | Keep the most important information in MEMORY.md. Use topic files for deep dives. |
| Duplicate instructions | Putting the same rule in both CLAUDE.md and MEMORY.md wastes context window space. | CLAUDE.md is for behavioral instructions. MEMORY.md is for learned knowledge. Do not duplicate between them. |
| Stale observations in MCP Memory | An observation says "backup runs at 3 AM" but you changed it to 5 AM last month. Old observations do not auto-expire. | Periodically audit MCP Memory. Delete outdated observations with `mcp__memory__delete_observations`. |
| Entity naming collisions | Two entities named "Backup" -- one for the script, one for the strategy. | Use descriptive, specific entity names: `Backup_Script_mac`, `Backup_Strategy_Overview`. |
| Memory injection on wrong project | Working in `/Users/slate/projects/myapp` but memory is set up for `/Users/slate`. | Claude walks up the directory tree. The home-level memory will still apply. Project-specific memory directories override for that subtree. |
| Large MCP Memory graph slows searches | After hundreds of entities, `read_graph` becomes expensive. | Use `search_nodes` with specific keywords instead of `read_graph`. Only read the full graph when you need a broad overview. |

# Chapter 7: Project Management for AI -- The GSD Framework

Claude Code is brilliant in short bursts. Ask it to fix a bug, add a feature, write a script -- it does the job well because the entire task fits inside a single conversation. The context window holds everything: the problem, the relevant code, the solution. Quality stays high because nothing gets lost.

But when projects span multiple files, multiple phases, and multiple sessions, quality degrades in predictable ways. The context window fills up with prior work. Claude forgets earlier architectural decisions it made three phases ago. Without structure, it starts contradicting itself, skipping verification steps, and producing half-finished work that drifts from the original vision. A 13-file refactor becomes a mess of inconsistent patterns. A multi-phase deployment leaves orphaned configs and untested edge cases.

GSD (Get Shit Done) is a custom project management framework built entirely as Claude Code skills. It solves the problem by providing structured planning, phased execution with quality gates, session continuity across conversations, and a file-based state system that Claude can read and write autonomously. It is not a wrapper around Jira or Linear. It is 22 slash commands that turn Claude Code from a capable tool into a project-aware execution engine.

The key insight, backed by data: Claude Code operates at approximately 5 minutes per plan. A 10-plan project takes about 50 minutes, not days or weeks. Actual production data from this infrastructure confirms it. WaterBot v2.0 -- 6 phases, 13 plans, 3 UI modes, full production deploy -- completed in 64 minutes. VPS Hardening -- 6 phases, 13 plans, infrastructure configuration plus edge cases plus validation -- completed in 164 minutes (about 13 minutes per plan, because infrastructure work requires more SSH verification).

---

## The Quality Degradation Curve

Context window management is the single most important factor in Claude Code output quality. Everything else -- prompt engineering, model selection, temperature settings -- is secondary to this.

Claude Code's context window has a fixed size. Every message, every file read, every tool call output, every prior conversation turn consumes space in that window. As the window fills, quality degrades along a predictable curve:

| Context Usage | Quality Level | What You See |
|---------------|--------------|--------------|
| 0-30% | Peak quality | Precise, well-verified work. Claude remembers all prior decisions. Catches edge cases. |
| 30-50% | Good quality | Slight repetition. Occasionally re-reads files it already read. Still reliable. |
| 50-70% | Noticeable degradation | Forgets earlier decisions. Starts contradicting prior work. Skips verification steps. |
| 70%+ | Poor quality | Hallucinations. Missed details. Incomplete implementations. Invents files that do not exist. |

Most people hit the degradation wall and blame the model. The model is not the problem. The problem is asking a single conversation to hold an entire project in its head at once.

GSD addresses this systematically:

- **Small plans.** Each plan contains 2-3 tasks. This keeps individual execution sessions short and focused.
- **Context resets.** Running `/clear` between major GSD commands dumps the conversation history and starts fresh. Claude rebuilds context from disk, not from memory.
- **File-based state.** STATE.md, SUMMARY.md, and .continue-here.md files persist project state to the filesystem. Claude reads these files to restore context instead of relying on conversation history.
- **Dependency graphs via frontmatter.** SUMMARY.md files include YAML frontmatter with `requires`, `provides`, `affects`, and `subsystem` fields. When planning a new phase, Claude reads only the frontmatter first (cheap), builds a dependency graph, and then reads full summaries for only the relevant prior work. This keeps context usage low even in 13-phase projects.

---

## The .planning/ Directory Structure

Every GSD project has a `.planning/` directory at its root. This directory is the project's brain -- it holds the vision, the roadmap, the current state, and every plan and summary from every phase. Claude Code reads from and writes to this directory throughout the project lifecycle.

```
.planning/
  PROJECT.md            # Vision, requirements, constraints, key decisions
  ROADMAP.md            # Phase breakdown with research flags
  STATE.md              # Living state: current position, decisions, issues, session log
  ISSUES.md             # Deferred enhancements (not bugs -- things to consider later)
  config.json           # Execution mode, depth, quality gates

  codebase/             # Brownfield analysis (only for existing codebases)
    STACK.md            # Languages, frameworks, versions
    ARCHITECTURE.md     # System architecture and patterns
    STRUCTURE.md        # Directory layout and file organization
    CONVENTIONS.md      # Code style, naming, patterns in use
    TESTING.md          # Test framework, coverage, CI/CD
    INTEGRATIONS.md     # External services, APIs, databases
    CONCERNS.md         # Technical debt, known issues, risks

  milestones/           # Archived roadmaps from completed milestones

  phases/
    01-foundation/
      01-CONTEXT.md     # Phase vision (from discuss-phase)
      01-RESEARCH.md    # Ecosystem research (from research-phase)
      01-01-PLAN.md     # Executable plan (IS the prompt)
      01-01-SUMMARY.md  # What was done, what changed, what to know
      01-01-ISSUES.md   # UAT issues found during verification
      01-01-FIX.md      # Fix plan generated from UAT issues
      .continue-here.md # Session handoff file
    02-core-features/
      02-01-PLAN.md
      02-01-SUMMARY.md
      02-02-PLAN.md
      02-02-SUMMARY.md
      ...
```

### What Each File Does

**PROJECT.md** is written once by `/gsd:new-project` and rarely modified after. It captures the project's purpose, target audience, technical constraints, and key decisions. Think of it as the project charter -- the thing you point to when someone (or Claude) asks "why are we building this?"

**ROADMAP.md** breaks the project into sequential phases. Each phase has a name, a description, and a flag indicating whether it needs ecosystem research before planning. The roadmap is the high-level plan. Individual PLAN.md files are the low-level execution instructions.

**STATE.md** is the living document. It tracks which phase is current, which plans are complete, what decisions have been made, what issues are deferred, and a session-by-session log of what happened. When Claude resumes work after a break, STATE.md is the first file it reads.

**SUMMARY.md files** are written after each plan executes. They record what was built, what files were changed, what decisions were made during execution, and what the next plan needs to know. The YAML frontmatter enables dependency tracking across phases.

---

## The Complete GSD Command Set

GSD consists of 22 slash commands organized into seven categories. Each command is a skill file -- a markdown document that loads instructions and templates when invoked.

### Initialization

| Command | What It Does | Output |
|---------|-------------|--------|
| `/gsd:new-project` | Asks deep questions about your project -- what you are building, why, for whom, what constraints exist | PROJECT.md + config.json |
| `/gsd:create-roadmap` | Analyzes PROJECT.md and breaks the project into sequential phases | ROADMAP.md + STATE.md + phase directories |
| `/gsd:map-codebase` | Spawns 4 parallel subagents to analyze an existing codebase | 7 codebase analysis documents |

### Planning

| Command | What It Does | Output |
|---------|-------------|--------|
| `/gsd:discuss-phase [N]` | Collaborative questioning about a phase -- what should it achieve, what are the risks | CONTEXT.md (phase vision document) |
| `/gsd:research-phase [N]` | Deep ecosystem research -- what tools exist, what patterns are established, what pitfalls to avoid | RESEARCH.md |
| `/gsd:list-phase-assumptions [N]` | Shows Claude's assumptions about a phase before planning begins | Console output only (no files) |
| `/gsd:plan-phase [N]` | Mandatory discovery (reads all prior summaries) then generates executable plans | PLAN.md files (2-3 tasks each) |

### Execution

| Command | What It Does | Output |
|---------|-------------|--------|
| `/gsd:execute-plan [path]` | Runs a PLAN.md -- reads tasks, picks execution strategy, makes atomic commits per task | Commits + SUMMARY.md |
| `/gsd:resume-task [agent-id]` | Resumes a subagent execution that was interrupted | Continues from interruption point |

### Session Management

| Command | What It Does | Output |
|---------|-------------|--------|
| `/gsd:pause-work` | Captures exact state: what is done, in progress, next, any blockers | .continue-here.md |
| `/gsd:resume-work` | Reads STATE.md, finds incomplete plans and handoff files, presents status | Console output + routing |
| `/gsd:progress` | Rich status report with phase completion, plan status, and routing to next action | Console output |

### Verification and Fix

| Command | What It Does | Output |
|---------|-------------|--------|
| `/gsd:verify-work [scope]` | Three-phase verification: data quality, browser testing, user acceptance testing | ISSUES.md (if problems found) |
| `/gsd:plan-fix [plan]` | Generates a fix plan from UAT issues found during verification | FIX.md |
| `/gsd:consider-issues` | Triages deferred issues with full codebase context -- keep, merge, or discard | Updated ISSUES.md |

### Roadmap Management

| Command | What It Does | Output |
|---------|-------------|--------|
| `/gsd:add-phase [description]` | Appends a new phase to the current milestone's roadmap | Updated ROADMAP.md |
| `/gsd:insert-phase [after] [desc]` | Inserts an urgent phase using decimal numbering (e.g., 7.1) without renumbering | Updated ROADMAP.md |
| `/gsd:remove-phase [N]` | Removes a future phase and renumbers subsequent phases | Updated ROADMAP.md |

### Milestone Management

| Command | What It Does | Output |
|---------|-------------|--------|
| `/gsd:discuss-milestone` | Collaborative questioning to define the next milestone's scope and goals | Console output |
| `/gsd:new-milestone [name]` | Creates a new milestone with its own phases and roadmap | New milestone directory + ROADMAP.md |
| `/gsd:complete-milestone [ver]` | Archives the current milestone, creates a git tag, prepares for the next version | Archived roadmap + git tag |

---

## Plans as Prompts -- The Key Design Decision

The most important architectural decision in GSD is this: **PLAN.md IS the executable prompt.** It is not a document that gets transformed into instructions. It is not a specification that Claude interprets. Claude reads PLAN.md and executes it directly, task by task.

This eliminates a translation layer. In traditional project management, a plan document describes what should happen, and then someone (or some tool) translates that into actual work instructions. In GSD, the plan IS the work instruction. There is no gap between "what the plan says" and "what Claude does."

### Task Types

Plans use XML-structured tasks. Each task has a type that tells Claude how to handle it:

| Task Type | Meaning | Example |
|-----------|---------|---------|
| `auto` | Claude executes autonomously, no human input needed | Write a function, create a config file, run tests |
| `checkpoint:human-verify` | Claude pauses for human visual or UX confirmation | Check that the UI looks correct, verify a deployment is live |
| `checkpoint:decision` | Claude pauses for a human implementation choice | Choose between two database schemas, pick a library |
| `checkpoint:human-action` | A truly unavoidable manual step | Create an API key on a third-party service, approve a DNS change |

### Task Structure

Each task in a PLAN.md includes these fields:

- **name** -- What this task is called (used in commit messages)
- **files** -- Which files will be created or modified
- **action** -- What to do, described precisely enough for Claude to execute
- **verify** -- How to confirm the task succeeded (run a test, check a URL, read a file)
- **done** -- The completion criteria that must be true before moving on

A simplified example:

```xml
<task type="auto">
  <name>add user authentication</name>
  <files>src/auth.js, src/middleware/auth.js, tests/auth.test.js</files>
  <action>
    Create a JWT-based authentication module. Use bcrypt for password hashing.
    Add middleware that validates tokens on protected routes.
    Write tests covering login, registration, and token expiration.
  </action>
  <verify>Run npm test -- all auth tests pass</verify>
  <done>Authentication works for login, registration, and protected routes</done>
</task>
```

---

## Three Execution Strategies

When `/gsd:execute-plan` runs, it analyzes the task types in the plan and selects one of three execution strategies. You do not choose the strategy -- GSD picks the right one automatically.

| Strategy | When It Fires | How It Works | Context Impact |
|----------|--------------|--------------|----------------|
| **A (Fully Autonomous)** | All tasks are `auto` | Spawns a subagent that runs the entire plan without interruption | Minimal -- subagent has its own context window |
| **B (Segmented)** | Only `checkpoint:human-verify` checkpoints | Subagent runs autonomous segments, pauses for human verification, resumes | Moderate -- pauses let you verify before continuing |
| **C (Decision-Dependent)** | Has `checkpoint:decision` tasks | Runs in the main context (no subagent) to allow interactive decisions | Higher -- but necessary for decisions that affect later tasks |

### Atomic Commits

Every task produces its own git commit. The commit message follows a structured format:

```
{type}({phase}-{plan}): {task-name}
```

For example:

```
feat(01-01): add user authentication
feat(01-01): create protected route middleware
test(01-01): add auth integration tests
feat(01-02): build user profile page
```

This gives you a clean, traceable git history where every commit maps to a specific task in a specific plan. If something breaks, you can identify exactly which task introduced the problem and review the plan that generated it.

---

## The Config System

`.planning/config.json` controls how GSD behaves for each project. It has three dimensions: mode, depth, and gates.

### Mode

| Mode | Behavior |
|------|----------|
| `interactive` | Confirms before major actions. Asks before creating plans, before executing, before committing. Default and recommended. |
| `yolo` | Fewer confirmations. Trusts the plan and executes. Use when you have high confidence in the roadmap and want speed. |

### Depth

| Depth | Phases | Plans per Phase | Best For |
|-------|--------|----------------|----------|
| `quick` | 3-5 | 1-3 | Small utilities, scripts, single-feature additions |
| `standard` | 5-8 | 3-5 | Most projects. Web apps, APIs, tools, integrations. |
| `comprehensive` | 8-12 | 5-10 | Large systems, infrastructure overhauls, multi-service architectures |

Tasks per plan is always 2-3 regardless of depth. This is a hard constraint -- more tasks per plan degrades quality.

### Gates

```json
{
  "mode": "interactive",
  "depth": "standard",
  "gates": {
    "confirm_project": true,
    "confirm_phases": true,
    "confirm_plan": true,
    "execute_next_plan": true
  },
  "safety": {
    "always_confirm_destructive": true,
    "always_confirm_external_services": true
  }
}
```

Gates control where GSD pauses for human confirmation. The safety gates are independent of mode -- even in `yolo` mode, GSD will still ask before running destructive operations (deleting databases, force-pushing branches) or calling external services (deploying to production, sending emails).

---

## Session Continuity

GSD projects often span multiple Claude Code sessions. You might plan a project in the morning, execute the first three phases after lunch, and finish the rest the next day. The framework handles this through three mechanisms.

### Pausing Work

Running `/gsd:pause-work` creates a `.continue-here.md` file in the current phase directory. This file captures the exact state of work:

- What plans are complete
- What plan was in progress (if any) and how far it got
- What the next plan is
- Any blockers or decisions that need human input
- A suggested next command to run

### Resuming Work

Running `/gsd:resume-work` at the start of a new session does the inverse. It reads STATE.md, scans for incomplete plans and `.continue-here.md` files, and presents a status summary. Then it routes you to the right next action -- whether that is continuing an interrupted plan, starting the next plan, or planning the next phase.

### Frontmatter Dependency Graphs

As a project grows, the `.planning/` directory accumulates many SUMMARY.md files. Reading all of them before planning a new phase would consume significant context window space. GSD solves this with YAML frontmatter in every summary:

```yaml
---
plan: 02-03
phase: 02-core-features
status: complete
requires: [01-01, 01-02, 02-01]
provides: [user-dashboard, analytics-api]
affects: [frontend, database]
subsystem: dashboard
---
```

When planning Phase 4, Claude does not read every summary from Phases 1-3 in full. Instead, it reads only the frontmatter from all summaries (a few lines each), builds a dependency graph, identifies which prior plans are relevant to the current phase, and then reads only those summaries in full. A 13-phase project might have 40+ summaries, but Claude only needs to read 5-6 of them to plan the next phase.

---

## A Typical Workflow

Here is a complete GSD workflow from project start to completion. Notice the `/clear` between every major command -- this is intentional context window management, not a suggestion.

```
/gsd:new-project
  → Deep questioning about your project
  → Generates PROJECT.md + config.json
/clear

/gsd:create-roadmap
  → Analyzes PROJECT.md
  → Generates ROADMAP.md + STATE.md + phase directories
/clear

/gsd:plan-phase 1
  → Reads PROJECT.md, ROADMAP.md
  → Runs discovery (reads existing code if brownfield)
  → Generates PLAN.md files for Phase 1 (2-3 tasks each)
/clear

/gsd:execute-plan .planning/phases/01-foundation/01-01-PLAN.md
  → Picks execution strategy based on task types
  → Executes tasks with atomic commits
  → Writes 01-01-SUMMARY.md
/clear

/gsd:progress
  → Shows completion status across all phases
  → Routes to next action: execute next plan, plan next phase, or verify
/clear

/gsd:execute-plan .planning/phases/01-foundation/01-02-PLAN.md
  → Same cycle: execute, commit, summarize
/clear

... repeat for remaining plans in Phase 1 ...

/gsd:plan-phase 2
  → Reads frontmatter from Phase 1 summaries
  → Plans Phase 2 with full awareness of what Phase 1 built
/clear

... repeat phases until roadmap is complete ...

/gsd:complete-milestone v1.0
  → Archives the roadmap
  → Creates a git tag
  → Prepares for the next milestone
```

### The /clear Discipline

The `/clear` between commands is not optional. It is the single most important habit for maintaining quality across a GSD project. Here is what happens without it:

1. You run `/gsd:new-project`. Claude asks questions, generates PROJECT.md. Context is now at 20%.
2. Without clearing, you run `/gsd:create-roadmap`. Claude reads PROJECT.md (which is already in context, so this is redundant), generates the roadmap. Context is now at 40%.
3. Without clearing, you run `/gsd:plan-phase 1`. Claude reads PROJECT.md and ROADMAP.md again, reads existing code, generates plans. Context is now at 65%.
4. Without clearing, you run `/gsd:execute-plan`. Claude reads the plan, executes tasks, reads and writes files. Context is now at 85%.
5. Quality has degraded significantly. Claude is making mistakes, skipping verification steps, and producing inconsistent code.

With `/clear` between each command, every step starts at 0% context usage. Claude rebuilds its understanding from the files on disk -- which is exactly what those files are for.

---

## Brownfield vs. Greenfield

GSD handles both new projects (greenfield) and existing codebases (brownfield).

### Greenfield Projects

For new projects, the workflow starts with `/gsd:new-project` and proceeds linearly through the roadmap. There is no existing code to analyze, so `/gsd:map-codebase` is skipped.

### Brownfield Projects

For existing codebases, `/gsd:map-codebase` runs before planning begins. It spawns four parallel subagents that analyze:

| Agent | Analyzes | Produces |
|-------|----------|----------|
| Stack Agent | Languages, frameworks, package managers, versions | STACK.md |
| Architecture Agent | Patterns, data flow, service boundaries, API design | ARCHITECTURE.md |
| Structure Agent | Directory layout, file naming, module organization | STRUCTURE.md + CONVENTIONS.md |
| Quality Agent | Tests, CI/CD, integrations, technical debt | TESTING.md + INTEGRATIONS.md + CONCERNS.md |

These seven documents go into `.planning/codebase/` and serve as reference material during planning. When Claude plans Phase 1 of a brownfield project, it reads the codebase analysis to understand what exists before proposing changes. This prevents Claude from suggesting patterns that conflict with the established codebase, recommending libraries that duplicate existing dependencies, or restructuring code in ways that break conventions the team already follows.

---

## Anti-Enterprise Philosophy

GSD is explicitly not enterprise project management. It was designed for a specific workflow: one human with a vision, one AI that implements. If you are looking for a framework that handles 20-person teams, cross-department coordination, or stakeholder alignment meetings, GSD is not it.

What GSD rejects:

| Enterprise Concept | Why GSD Rejects It |
|--------------------|--------------------|
| Story points | Claude does not estimate effort in abstract units. Count plans, multiply by 5 minutes. |
| Sprint ceremonies | There is no standup. There is no retrospective. You run `/gsd:progress` and keep going. |
| RACI matrices | One human decides. One AI executes. The matrix is a 1x1. |
| Gantt charts | Plans execute sequentially. Dependencies are tracked in frontmatter. A Gantt chart adds nothing. |
| Time estimates in days/weeks | GSD operates at 5 minutes per plan. A 10-plan project is 50 minutes, not "about a week." |
| Kanban boards | STATE.md tracks what is done, in progress, and next. That is the board. |
| Velocity tracking | Velocity is constant: Claude runs at Claude speed. There is no sprint-over-sprint variance to track. |

The guiding principle: if it sounds like corporate PM theater, delete it.

---

## Reproduction Steps

Building GSD into your own Claude Code setup requires the skill files and an understanding of the workflow. Here is how to get started.

1. **Get the skill files.** The GSD framework lives in two directories within the dotfiles repo: `commands/gsd/` (the 22 slash commands) and `get-shit-done/` (supporting workflow definitions). These must be placed where Claude Code discovers skills -- under `~/.claude/commands/gsd/` and `~/.claude/get-shit-done/`.

2. **Start a project.** Navigate to a new or existing repository and run `/gsd:new-project`. GSD will ask you a series of questions about what you are building, why, for whom, and what constraints exist. Answer honestly -- these answers become PROJECT.md and directly influence the quality of the roadmap.

3. **Create the roadmap.** Run `/clear`, then `/gsd:create-roadmap`. GSD analyzes PROJECT.md and breaks the project into sequential phases, creating the directory structure and STATE.md.

4. **Plan and execute.** Follow the workflow described above: plan a phase, clear, execute each plan, clear, check progress, clear, repeat. Let GSD route you -- `/gsd:progress` always tells you what to do next.

5. **Use session management.** Run `/gsd:pause-work` before ending any session. Run `/gsd:resume-work` at the start of the next session. This is not optional -- without it, Claude loses context between sessions.

---

## Gotchas and Hard-Won Lessons

These are not theoretical concerns. Every item on this list caused real problems during real projects.

- **Always /clear between GSD commands.** Context buildup is the primary cause of quality degradation. This is the most important habit in GSD and the easiest to forget.

- **Plans must have 2-3 tasks maximum.** More tasks per plan means a longer execution session, which means higher context usage, which means lower quality on the later tasks. If a plan needs more than 3 tasks, split it into two plans.

- **Always run /gsd:pause-work before ending a session.** If you close the terminal without pausing, the state is not saved. The next session will not know where you left off, and Claude will either restart work that was already done or skip work that was not finished.

- **Always run /gsd:resume-work when starting a session.** Do not jump straight to executing a plan. Resume-work reads STATE.md, finds handoff files, and rebuilds context properly. Skipping it means Claude starts without knowing what happened in prior sessions.

- **Do not improvise GSD steps from memory.** Always invoke commands via the slash command so the workflow file loads fresh. Claude sometimes remembers GSD procedures from prior conversations and tries to run them from memory instead of loading the current skill file. The skill file may have been updated since Claude last saw it.

- **Use "comprehensive" depth sparingly.** Comprehensive creates 8-12 phases with 5-10 plans each. That is potentially 120 plans. Unless your project genuinely warrants that scope (a multi-service platform, a full infrastructure overhaul), standard depth is the right choice.

- **Research flags in the roadmap matter.** If a phase is flagged for research in ROADMAP.md, run `/gsd:research-phase` before planning. Skipping research means Claude plans with assumptions instead of facts, and those assumptions compound across later phases.

- **Frontmatter is not decoration.** The `requires`, `provides`, `affects`, and `subsystem` fields in SUMMARY.md frontmatter are how GSD keeps context usage low in large projects. If you manually edit summaries, preserve the frontmatter.

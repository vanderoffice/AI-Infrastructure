# Philosophy & Overview

This chapter explains what this infrastructure is, why it exists, and what it gives you when it is fully built. Read this first. Everything else in the guide is an implementation detail of the ideas here.

---

## The Dual Mission

This infrastructure serves two distinct purposes that share the same toolchain.

**Policy research and professional writing.** Government automation analysis, document generation, presentation creation, meeting transcripts, and professional communications. The workflow looks like: research a policy topic, synthesize findings into a structured document, generate a slide deck, and publish it. Claude Code drives the entire pipeline -- from web research to final deliverable.

**AI-assisted software development.** Production web applications, RAG-powered chatbots, workflow automation, monitoring dashboards, and the infrastructure itself. The workflow looks like: describe what you want built, Claude Code writes the code, tests it, deploys it, and monitors it. The bots, the website, the backup system, the alerting pipeline -- all of it was built and is maintained this way.

These two missions reinforce each other. The writing tools (presentation generators, document formatters, transcript processors) are software projects that Claude Code builds and maintains. The software projects produce deliverables for the policy work. The infrastructure that supports both is itself maintained by Claude Code. It is turtles all the way down.

---

## Claude Code as an Autonomous Development Engine

Claude Code is Anthropic's CLI tool. It puts Claude -- the AI model -- inside your terminal with full access to your filesystem, shell, and connected tools. You type what you want in natural language. Claude Code reads your codebase, writes files, runs commands, checks the output, and iterates until the task is done.

This is different from other AI coding tools in a way that matters.

**ChatGPT** gives you code snippets in a chat window. You copy-paste them into your editor, run them, see the error, paste the error back, and repeat. The AI never touches your actual files.

**GitHub Copilot** autocompletes code as you type. It is fast and useful, but it operates at the line or function level. It does not understand your project architecture, run your tests, or deploy your application.

**Cursor** embeds AI into an editor with more context than Copilot, but it still operates within the boundaries of a code editor. It does not SSH into your servers or manage your infrastructure.

**Claude Code** operates at the project level and beyond. In a single session, it can:

- Read your entire codebase to understand architecture
- Write new files and modify existing ones
- Run tests and fix failures
- SSH into remote servers to check status or deploy
- Create git commits and push to remote repositories
- Execute multi-step plans spanning dozens of files
- Use MCP tools to interact with external services (databases, APIs, workflow engines)

The practical difference: you can tell Claude Code "add a new chatbot to the website that answers questions about water policy using this PDF as its knowledge base" and walk away. It will create the RAG pipeline, build the UI component, wire up the API routes, test the integration, deploy to production, and verify the deployment. That is not a hypothetical. That is how the chatbots in this infrastructure were built.

### The "Sharp Senior Engineer" Model

Claude Code does what you tell it by default. But raw capability is not enough. An AI that asks "should I create a new file?" before every action is not a senior engineer -- it is an intern.

The behavioral instructions in CLAUDE.md transform Claude Code from a capable but passive tool into an opinionated, action-oriented development partner. The key principles:

- **Take action, do not ask permission for obvious steps.** If the fix is clear, make the fix. Do not ask "would you like me to update the import statement?" Just update it.
- **Research before acting or asking.** Read the existing code, check the documentation, look at the infrastructure state. Arrive at the conversation informed.
- **Fix problems instead of working around them.** A broken script gets fixed, not bypassed with a manual workaround.
- **The 5-minute rule.** If a fix takes less than five minutes, do it now. Never defer with "non-critical for tonight."

This is codified in CLAUDE.md as a behavioral framework with hard gates (rules that override everything), priority ordering (security > research > action > documentation), and anti-pattern detection (triggers that force Claude Code to stop and re-evaluate when it falls into bad habits).

The result: Claude Code behaves less like a chatbot and more like a senior engineer who happens to have perfect memory, can read an entire codebase in seconds, and never gets tired.

---

## Why Multiple Machines?

A single laptop can run Claude Code. You do not need seven machines. But role separation makes the system more capable, more reliable, and easier to reason about.

### The Machines and Their Roles

| Machine | Hardware | Role | Why It Exists |
|---------|----------|------|---------------|
| **StudioM4** | Apple M4, macOS | Primary development | Your main workstation. Claude Code runs here. Code gets written here. |
| **OfficeM4P** | Apple M4 Pro, 24GB, macOS | Heavy AI and research | When you need more compute -- deep research, GIS processing, heavy inference. Dedicated desk, dedicated focus. |
| **ServerM2P** | Apple M2 Pro (headless), macOS | Home server and MCP hub | Always-on. Hosts MCP servers that all machines share. Stores backups on a 4TB SSD. Runs the test environment. |
| **VPS** | Ubuntu 24.04 (Hostinger) | Production | The public-facing server. Hosts the website, chatbots, APIs, monitoring, and workflow automation. Lives in a data center. |
| **NetSentry** | Raspberry Pi 5 | Primary DNS and monitoring | Network-wide ad blocking via Pi-hole. Recursive DNS via Unbound. Infrastructure dashboard. |
| **AlertNode** | Raspberry Pi 3B+ | Secondary DNS and alerting | Backup DNS. Push notifications via ntfy. Watches NetSentry (mutual monitoring). |
| **S23+** | Samsung Galaxy S23+ | Mobile access | Remote access to everything via Tailscale. Media capture. Push notification receiver. |

### The Pipeline

Code flows in one direction through the system:

```
Dev (StudioM4 / OfficeM4P) --> Test (ServerM2P) --> Prod (VPS)
```

Development machines are where Claude Code writes and tests code locally. ServerM2P is where you test in a production-like environment before deploying. The VPS is production -- the public internet hits it.

This separation means a bug in development never takes down production. A misconfigured Docker container on ServerM2P does not affect the website. And if the VPS has an issue, your development machines and home server are unaffected.

### The MCP Hub Pattern

ServerM2P's most important role is hosting shared MCP servers. Without it, every machine would need its own copy of every tool -- its own memory server, its own LLM gateway, its own n8n connection. With the hub, tools are installed once on ServerM2P and shared to all machines via SSE (Server-Sent Events) over the Tailscale mesh.

```
StudioM4  ---SSE--->  ServerM2P (MCP Hub)  <---SSE---  OfficeM4P
                           |
                      MCP Gateway (:3333)
                      Memory MCP  (:8083)
                      n8n-local   (:8081)
                      n8n-public  (:8082)
                      Perplexity  (:8084)
                           |
                       <---SSE---  VPS
```

When Claude Code on StudioM4 saves something to memory, Claude Code on OfficeM4P can read it. When Claude Code triggers an n8n workflow, it goes through the same hub regardless of which machine initiated it. One source of truth, multiple access points.

---

## What You Will Have When Done

A mesh of 7 machines where you talk to Claude Code in your terminal and it:

- **Builds full-stack web applications** with React, Vite, and Tailwind. From `npm create vite` to production deployment in a single session.
- **Creates and maintains RAG-powered chatbots** backed by pgvector and Supabase. Ingest a PDF, generate embeddings, build a chat interface, deploy it.
- **Automates government workflows** with n8n. Webhook-triggered document generation, form processing, email routing.
- **Generates presentations from research.** Give it a topic, it researches, outlines, writes, and builds a reveal.js slide deck.
- **Manages its own memory across sessions.** Claude Code remembers project state, architecture decisions, and infrastructure knowledge through a shared memory graph. Start a session on StudioM4, finish it on OfficeM4P.
- **Monitors, backs up, and maintains the entire infrastructure.** Nightly backups with failure-only alerting. Prometheus metrics. Push notifications on your phone when something breaks. Silence when everything is healthy.
- **Syncs configuration across all machines automatically.** Change a dotfile on one machine, it propagates to all others via git hooks.

The infrastructure maintains itself. Backups run nightly. Health checks run on schedule. Alerts fire only when something breaks. The Sunday morning digest proves the alerting system itself is alive. You spend your time building things, not babysitting servers.

---

## Scaling Down -- Minimum Viable Setup

Not everyone needs seven machines. Here is how to scale the system down to what you actually need, and what you give up at each tier.

### Tier 1: One Mac + One VPS

**What you get:** Claude Code on your Mac as the development engine. A VPS as your production target. This is the core experience.

**What you run locally:** Claude Code, MCP servers (memory, LLM gateway) running as stdio processes instead of SSE. Your dotfiles repo for configuration.

**What runs on the VPS:** Your production website, Docker containers, nginx, SSL, monitoring.

**What you lose:**
- No MCP hub (run servers locally on your Mac -- they just are not shareable)
- No backup redundancy (use a cloud backup service instead of a home server)
- No DNS filtering (use a public DNS resolver like 1.1.1.1 or 9.9.9.9)
- No test environment separate from dev or prod
- No second development machine for heavy workloads

**This is a perfectly good setup.** Most developers ship production software with less. The Claude Code + VPS combination alone is more capable than most development environments.

### Tier 2: Add a Home Server

**What changes:** A headless Mac (or Linux box) on your home network running 24/7.

**What you gain:**
- MCP servers run on the home server as SSE, shareable to all your machines
- A 4TB (or whatever you have) backup target on your local network
- A proper test environment: dev on your Mac, test on the server, deploy to VPS
- Docker services that need to be always-on (databases, workflow automation) move off your laptop
- The pipeline becomes real: Dev --> Test --> Prod

**This is the biggest quality-of-life upgrade.** The jump from Tier 1 to Tier 2 is larger than the jump from Tier 2 to the full setup. A persistent MCP hub and local backup storage eliminate two classes of problems at once.

### Tier 3: Add Raspberry Pis

**What changes:** One or two Raspberry Pis on your network.

**What you gain:**
- Network-wide ad blocking and DNS filtering via Pi-hole
- Recursive DNS resolution via Unbound (privacy improvement)
- Dedicated push notification server (ntfy)
- Mutual health monitoring (Pis watch each other and the server)
- DNS redundancy (if one Pi dies, the other handles DNS)

**This is a nice-to-have.** Pi-hole improves your entire network experience, not just development. But it is not critical for the Claude Code workflow. Add it when you want to, not because you need to.

### Tier 4: Full Setup

Add a second development machine and a phone on Tailscale. This is the full 7-machine mesh documented in this guide. You get role separation between primary development and heavy AI/research workloads, mobile access to everything, and the full mutual-monitoring and backup architecture.

---

## How to Use This Guide

**Read linearly on your first pass.** The chapters are ordered so that each one builds on what came before. The philosophy (this chapter) gives you the mental model. Hardware and networking (chapters 02-04) lay the physical foundation. Server setup (05-06) builds the infrastructure layer. Claude Code and MCP (07-08) add the AI layer. Everything after that -- dotfiles, backups, monitoring, chatbots, automation -- runs on top of those foundations.

**Then use it as a reference.** After your first read-through, each chapter is self-contained. Need to add a new MCP server? Go to chapter 08. Backup failing? Chapter 10. New chatbot? Chapter 12. You do not need to re-read the whole guide.

**Let Claude Code help you build it.** The CLAUDE.md at the root of this repository contains instructions that let your own Claude Code instance reference these docs. As you work through each chapter, Claude Code can read the relevant section and help you implement it. The guide teaches the system. The system helps you build the system.

**Chapters are structured for later reuse.** Every H2 header (##) marks a section boundary that maps cleanly to a presentation slide. This is deliberate. The same content that teaches you how to build the infrastructure can be turned into a talk about how the infrastructure works.

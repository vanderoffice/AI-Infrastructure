# AI-Powered Development Infrastructure -- Complete Reproduction Guide

This guide documents a 7-machine mesh network built around Claude Code as an autonomous development engine. The infrastructure serves two missions: policy research and professional writing (government automation, document generation, presentations) and AI-assisted software development (production web apps, RAG-powered chatbots, workflow automation). Every chapter explains the "why" before the "how," and includes reproduction steps so you can build the same system from scratch -- or scale it down to fit your needs.

---

## Chapters

| # | Title | File | Description |
|---|-------|------|-------------|
| 00 | Index & Glossary | [00-index.md](00-index.md) | This file. Table of contents, reading order, and terminology. |
| 01 | Philosophy & Overview | [01-philosophy-and-overview.md](01-philosophy-and-overview.md) | The dual mission, why Claude Code changes the game, and what the finished system looks like. |
| 02 | Hardware & Network | [02-hardware-and-network.md](02-hardware-and-network.md) | Every machine in the mesh, Tailscale, SSH smart routing, network zones. |
| 03 | Dotfiles Framework | [03-dotfiles-framework.md](03-dotfiles-framework.md) | Multi-machine config sync with Git, directory-level symlinks, bootstrap scripts. |
| 04 | Claude Code Configuration | [04-claude-code-configuration.md](04-claude-code-configuration.md) | CLAUDE.md behavioral instructions, settings.json permissions, hooks, anti-patterns. |
| 05 | MCP Servers | [05-mcp-servers.md](05-mcp-servers.md) | Model Context Protocol, SSE hub architecture on ServerM2P, LLM Gateway routing. |
| 06 | Skills System | [06-skills-system.md](06-skills-system.md) | Custom slash commands (30+), skill discovery, subagent system, skill creation. |
| 07 | GSD Framework | [07-gsd-framework.md](07-gsd-framework.md) | AI project management -- phases, plans as executable prompts, session handoffs. |
| 08 | Memory System | [08-memory-system.md](08-memory-system.md) | Cross-session continuity, MCP Memory + auto-memory, the Memory Write Gate. |
| 09 | Production Stack | [09-production-stack.md](09-production-stack.md) | VPS with 29 Docker containers, nginx reverse proxy, Cloudflare, CI/CD with GitHub Actions. |
| 10 | Bot Architecture | [10-bot-architecture.md](10-bot-architecture.md) | RAG-powered chatbots, pgvector, shared UI components, n8n webhook pipelines. |
| 11 | Automation & Scheduling | [11-automation-and-scheduling.md](11-automation-and-scheduling.md) | LaunchAgents, nightly backups, 3-2-1 strategy, monitoring stack, alerting philosophy. |
| 12 | DNS & Smart Home | [12-dns-and-smart-home.md](12-dns-and-smart-home.md) | Dual Pi-hole + Unbound, gravity-sync, Home Assistant, Zigbee2MQTT. |
| 13 | Security | [13-security.md](13-security.md) | SSH hardening, VPS hardening, fail2ban, Cloudflare edge, credential management. |
| 14 | Reproduction Checklist | [14-reproduction-checklist.md](14-reproduction-checklist.md) | Step-by-step build order with 7 phases and verification gates. |

## Appendices

| # | Title | File | Description |
|---|-------|------|-------------|
| A | IP & Port Reference | [appendix-a-ip-and-port-reference.md](appendix-a-ip-and-port-reference.md) | Complete network map -- every machine IP (LAN + Tailscale), every listening port. |
| B | Config File Templates | [appendix-b-config-files.md](appendix-b-config-files.md) | Annotated templates for SSH config, `~/.claude.json`, LaunchAgent plists, CLAUDE.md structure. |
| C | Pricing & Costs | [appendix-c-claude-pricing-and-costs.md](appendix-c-claude-pricing-and-costs.md) | Claude Code subscription tiers, external API costs, monthly totals, cost optimization. |
| D | Drift Report | [appendix-d-drift-report.md](appendix-d-drift-report.md) | All 20 discrepancies found in the 2026-03-01 live audit across 4 machines. |

---

## Reading Order

**First pass (read linearly):** Start at Chapter 01 and read straight through Chapter 14. Each chapter builds on the previous one. The philosophy chapter (01) gives you the mental model. Hardware, network, and dotfiles (02-03) lay the foundation. Claude Code, MCP, and skills (04-06) set up the AI brain. GSD and memory (07-08) add project management and continuity. The remaining chapters (09-14) cover production, bots, automation, DNS, security, and the final reproduction checklist.

**After the first pass:** Use chapters as standalone references. Each one includes its own reproduction steps and can be consulted independently. The appendices are reference material you will revisit often.

**Minimum viable path:** If you only want the core experience (Claude Code + a production target), read 01, 04, 05, and 09. You can skip the home server, Pis, and monitoring until you are ready to expand.

---

## Glossary

| Term | Definition |
|------|------------|
| **Claude Code** | Anthropic's CLI tool that puts the Claude AI model inside your terminal with full filesystem, shell, and tool access. It can read files, write code, run commands, SSH into servers, and execute multi-step plans autonomously. |
| **Claude** | Anthropic's family of large language models (Haiku, Sonnet, Opus). Claude Code uses these models as its reasoning engine. |
| **Anthropic** | The AI safety company that builds Claude and Claude Code. |
| **MCP (Model Context Protocol)** | An open protocol that lets AI models connect to external tools and data sources. Think of it as USB-C for AI -- a standard way to plug capabilities into Claude Code. |
| **SSE (Server-Sent Events)** | A transport method for MCP where a server runs on one machine and clients connect to it over the network. Used to share MCP tools across multiple machines from a central hub. |
| **stdio** | A transport method for MCP where the tool runs as a local process on the same machine as Claude Code. Simpler than SSE but not shareable across the network. |
| **GSD (Get Shit Done)** | A custom project management framework built as a Claude Code skill. It breaks projects into phases and plans, tracks progress, and generates session handoffs so Claude can resume work across sessions. |
| **RAG (Retrieval-Augmented Generation)** | A technique where an AI retrieves relevant documents from a database before generating a response. This grounds the AI's answers in your actual data instead of relying solely on its training. |
| **pgvector** | A PostgreSQL extension that adds vector similarity search. Used to store and query document embeddings for RAG-powered chatbots. |
| **Embeddings** | Numerical representations of text that capture semantic meaning. Similar texts produce similar vectors, enabling semantic search (finding documents by meaning, not just keywords). |
| **Tailscale** | A mesh VPN service built on WireGuard. It creates encrypted tunnels between all your devices so they can communicate as if they were on the same local network, regardless of physical location. |
| **Mesh network** | A network topology where every device can communicate directly with every other device, rather than routing through a central hub. Tailscale creates this across the internet. |
| **WireGuard** | A modern, lightweight VPN protocol. Tailscale uses WireGuard under the hood to create its encrypted tunnels. |
| **n8n** | An open-source workflow automation platform (self-hosted alternative to Zapier). Used here for bot orchestration, webhook processing, and government document automation. |
| **Pi-hole** | A DNS-level ad blocker that runs on Raspberry Pis. It filters network traffic for all devices on the network by blocking known advertising and tracking domains at the DNS level. |
| **Unbound** | A recursive DNS resolver. Instead of forwarding DNS queries to Google or Cloudflare, Unbound resolves them by walking the DNS hierarchy itself, improving privacy. |
| **DNS (Domain Name System)** | The system that translates human-readable domain names (like vanderdev.net) into IP addresses. Pi-hole and Unbound provide custom DNS for the local network. |
| **Docker** | A platform for running applications in isolated containers. Each container packages an application with its dependencies, ensuring consistent behavior across different machines. |
| **Docker Compose** | A tool for defining and running multi-container Docker applications using a YAML configuration file. Most services in this infrastructure are defined in docker-compose.yml files. |
| **Containers** | Lightweight, isolated environments for running applications. Unlike virtual machines, containers share the host OS kernel, making them fast to start and resource-efficient. |
| **LaunchAgents (macOS)** | The macOS scheduling system (similar to cron on Linux). Used to run nightly backups, log syncs, health checks, and other automated tasks on a timer. |
| **Dotfiles** | Configuration files (often starting with a dot, like .zshrc or .gitconfig) that define how your shell, editor, and tools behave. Stored in a git repo and synced across all machines for consistency. |
| **VPS (Virtual Private Server)** | A virtual machine running in a data center, accessible over the internet. Used here as the production server for the public-facing website, chatbots, and APIs. |
| **Supabase** | An open-source backend platform built on PostgreSQL. Provides a database, authentication, real-time subscriptions, and file storage. Self-hosted on the VPS via Docker. |
| **CLAUDE.md** | A markdown file that contains behavioral instructions for Claude Code. Placed at the root of a project or in your home directory, it tells Claude Code how to behave -- what to prioritize, what to avoid, and how to approach problems. |
| **Skills / Slash commands** | Custom commands (like /gsd, /deck, /transcript) that extend Claude Code with domain-specific workflows. Each skill is a markdown file that loads instructions and templates when invoked. |
| **Webhook** | An HTTP callback -- a URL that receives data when an event occurs. Used by n8n to trigger workflows, by chatbots to receive messages, and by monitoring systems to send alerts. |
| **Supabase** | An open-source backend platform built on PostgreSQL. Provides a database, authentication, real-time subscriptions, and file storage. Self-hosted on the VPS via Docker. |
| **Caddy** | A modern web server with automatic HTTPS. Used on ServerM2P to reverse-proxy internal services like Vaultwarden and Jellyfin. |
| **Vaultwarden** | A self-hosted, lightweight implementation of the Bitwarden password manager. Runs behind Caddy on ServerM2P for internal-only access. |
| **ntfy** | A simple push notification server. Self-hosted on AlertNode (Pi 3B+) and used for backup failure alerts, monitoring notifications, and the weekly infrastructure digest. |
| **gravity-sync** | A tool that replicates Pi-hole blocklist and configuration changes from a primary instance to a secondary. Keeps both DNS servers in sync automatically. |
| **OrbStack** | A lightweight Docker Desktop alternative for macOS. Faster to start, lower resource usage, and handles both containers and Linux VMs. |

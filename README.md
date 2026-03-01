# AI-Powered Development Infrastructure

> A complete guide to building a multi-machine AI development environment using Claude Code as an autonomous engineering platform.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Verified: 2026-03-01](https://img.shields.io/badge/Verified-2026--03--01-green.svg)](#drift-report)
[![Chapters: 14](https://img.shields.io/badge/Chapters-14-orange.svg)](#guide-structure)

---

## What This Is

This repository documents a production AI development infrastructure built over four months of daily use. It uses [Claude Code](https://docs.anthropic.com/en/docs/claude-code) -- Anthropic's CLI tool -- as the central development engine across a mesh of seven machines: three Apple Silicon Macs, one Linux VPS, two Raspberry Pis, and one Android phone connected via Tailscale. The infrastructure serves two purposes: policy research and professional writing, and AI-assisted software development.

The system includes behavioral AI instructions that make Claude Code operate like a senior engineer with memory and context, 30+ custom skills (slash commands) for everything from slide deck generation to RAG ingestion, a project management framework called GSD that turns plans into executable prompts, RAG-powered chatbots backed by PostgreSQL/pgvector, automated backups across all machines, network-wide DNS filtering with Pi-hole, infrastructure monitoring with Prometheus and Grafana, and production deployment via GitHub Actions and Docker.

Everything in this guide was verified against live running systems on March 1, 2026. Not documentation-as-wishful-thinking, but actual SSH audits of every machine, every port, every container. Where reality diverged from documentation, we recorded it in [Appendix D](docs/appendix-d-drift-report.md).

## Who This Is For

- **Developers** who want to use Claude Code as more than a chatbot -- as a genuine development partner with memory, skills, and project awareness
- **Teams** building AI-assisted development workflows and looking for patterns that survive daily production use
- **Infrastructure enthusiasts** curious about multi-machine management with AI at the center
- **Anyone** who has thought "I should automate this" and wants a blueprint

**Prerequisites:** You should be comfortable with macOS or Linux, command-line tooling, Docker basics, and SSH. You do not need to be a sysadmin -- the guide explains everything step by step.

## Quick Start

If you want to jump straight to building:

1. Read [Chapter 01: Philosophy & Overview](docs/01-philosophy-and-overview.md) to understand the design principles
2. Follow [Chapter 14: Reproduction Checklist](docs/14-reproduction-checklist.md) for the step-by-step build order

**Minimum viable setup:** 1 Mac + 1 VPS + a Claude Code subscription gets you a fully functional AI development workflow with behavioral instructions, custom skills, and production deployment. You can add machines incrementally from there.

## Guide Structure

| # | Chapter | What You'll Learn |
|---|---------|-------------------|
| 00 | [Index & Glossary](docs/00-index.md) | Terms, reading order, chapter map |
| 01 | [Philosophy & Overview](docs/01-philosophy-and-overview.md) | Why this exists, design principles, what you'll build |
| 02 | [Hardware & Network](docs/02-hardware-and-network.md) | Machine roles, Tailscale mesh, SSH routing |
| 03 | [Dotfiles Framework](docs/03-dotfiles-framework.md) | Multi-machine config sync with Git |
| 04 | [Claude Code Configuration](docs/04-claude-code-configuration.md) | CLAUDE.md behavioral system, permissions, hooks |
| 05 | [MCP Servers](docs/05-mcp-servers.md) | External tool integration, SSE hub pattern |
| 06 | [Skills System](docs/06-skills-system.md) | Custom slash commands, building 30+ skills |
| 07 | [GSD Framework](docs/07-gsd-framework.md) | AI project management -- plans as executable prompts |
| 08 | [Memory System](docs/08-memory-system.md) | Cross-session continuity, dual-tier memory architecture |
| 09 | [Production Stack](docs/09-production-stack.md) | VPS setup, Docker, nginx, CI/CD with GitHub Actions |
| 10 | [Bot Architecture](docs/10-bot-architecture.md) | RAG chatbots, pgvector, shared UI components |
| 11 | [Automation & Scheduling](docs/11-automation-and-scheduling.md) | Backups, monitoring, macOS LaunchAgents, cron |
| 12 | [DNS & Smart Home](docs/12-dns-and-smart-home.md) | Pi-hole, Unbound recursive DNS, Home Assistant |
| 13 | [Security](docs/13-security.md) | VPS hardening, Cloudflare, fail2ban, SSH lockdown |
| 14 | [Reproduction Checklist](docs/14-reproduction-checklist.md) | Step-by-step build order with verification gates |

### Appendices

| # | Appendix | Contents |
|---|----------|----------|
| A | [IP & Port Reference](docs/appendix-a-ip-and-port-reference.md) | Complete network map -- every IP, port, and service |
| B | [Config File Templates](docs/appendix-b-config-files.md) | SSH config, MCP config, LaunchAgent plists, nginx |
| C | [Pricing & Costs](docs/appendix-c-claude-pricing-and-costs.md) | Claude Code tiers, external API costs, monthly totals |
| D | [Drift Report](docs/appendix-d-drift-report.md) | Live audit findings from 2026-03-01 verification |

## Using This Guide with Claude Code

This repository includes a project-level `CLAUDE.md` file. When you clone this repo and open it in Claude Code, your Claude instance automatically reads that file and gains awareness of all 14 chapters. This means you can have a conversation like:

1. Clone this repo: `git clone https://github.com/vanderoffice/AI-Infrastructure.git`
2. Open the directory in Claude Code (or add it to your workspace)
3. Claude Code reads `CLAUDE.md` and knows the chapter map
4. Ask Claude: *"Help me set up the MCP server hub from Chapter 5"* -- it reads the chapter and walks you through it, adapting IPs and paths to your environment

The guide becomes interactive documentation. Instead of reading and translating instructions yourself, Claude reads the chapter and helps you execute it against your actual machines.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    TAILSCALE MESH (WireGuard)                    │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│   StudioM4   │  OfficeM4P   │  ServerM2P   │       VPS          │
│   (Primary   │  (Heavy AI,  │  (MCP Hub,   │   (Production,     │
│    Dev)      │   Research)  │   Backups,   │    29 Docker       │
│              │              │   Home Auto) │    containers)     │
├──────────────┴──────────────┤              ├────────────────────┤
│      Dev Machines           │              │    Public Web      │
│   Claude Code + Skills      │  9 MCP Svrs  │   vanderdev.net    │
│   Dotfiles synced           │  4TB Backup  │   3 AI Chatbots    │
│                             │  Home Asst   │   n8n Workflows    │
├─────────────────────────────┴──────────────┴────────────────────┤
│   NetSentry (Pi 5)          │   AlertNode (Pi 3B+)              │
│   Primary DNS (Pi-hole)     │   Secondary DNS (Pi-hole)         │
│   Unbound recursive         │   ntfy notifications              │
│   Uptime Kuma               │   Watchdog monitoring             │
├─────────────────────────────┼───────────────────────────────────┤
│                    S23+ (Android)                                │
│                    Tailscale mobile access                       │
│                    Remote SSH from anywhere                      │
└─────────────────────────────────────────────────────────────────┘
```

All machines communicate over Tailscale's WireGuard mesh. The VPS is the only machine with a public IP. Everything else is internal, reachable only through the mesh or via Cloudflare-tunneled services.

## Key Technologies

- **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** -- AI development engine, the brain of the operation
- **[Model Context Protocol (MCP)](https://modelcontextprotocol.io/)** -- connects Claude to external tools (memory, APIs, workflows)
- **[Tailscale](https://tailscale.com/)** -- WireGuard mesh VPN connecting all 7 machines
- **[Docker](https://www.docker.com/)** -- containerized services on VPS (29 containers in production)
- **[React](https://react.dev/) + [Vite](https://vite.dev/)** -- frontend framework for web properties and chatbot UIs
- **[PostgreSQL](https://www.postgresql.org/) + [pgvector](https://github.com/pgvector/pgvector)** -- RAG knowledge base for AI chatbots
- **[n8n](https://n8n.io/)** -- workflow automation (webhook handlers, API orchestration)
- **[Pi-hole](https://pi-hole.net/) + [Unbound](https://nlnetlabs.nl/projects/unbound/about/)** -- network-wide DNS filtering with recursive resolution
- **[Prometheus](https://prometheus.io/) + [Grafana](https://grafana.com/)** -- metrics collection and dashboards
- **[nginx](https://nginx.org/)** -- reverse proxy and static file serving
- **[Cloudflare](https://www.cloudflare.com/)** -- DNS, DDoS protection, SSL termination
- **[rclone](https://rclone.org/)** -- offsite backup to Google Drive
- **macOS LaunchAgents** -- scheduled tasks on Mac machines (backup, sync, monitoring)

## Cost Summary

| Setup | Monthly Cost | What You Get |
|-------|-------------|--------------|
| **Minimum Viable** | ~$31/mo | 1 Mac + 1 VPS + Claude Code Max (5x) -- full AI dev workflow |
| **Comfortable** | ~$120/mo | 2 Macs + VPS + Claude Code Max (20x) + domain + monitoring |
| **Full Infrastructure** | ~$224/mo | 7 machines, 29 containers, 3 chatbots, full redundancy |

The largest single cost is the Claude Code Max subscription ($100-$200/mo depending on tier). VPS hosting runs $5-25/mo. Everything else is either free-tier, one-time hardware, or under $10/mo. See [Appendix C](docs/appendix-c-claude-pricing-and-costs.md) for a full breakdown.

## Contributing

This is a documentation-only repository -- there is no application code to run. Contributions are welcome in the form of:

- **Issues** for corrections, unclear instructions, or outdated information
- **Discussions** for questions about adapting the guide to your setup
- **Pull requests** for typo fixes, improved explanations, or additional tips

If you build your own version of this infrastructure, we'd love to hear about it. Open a Discussion and share what you adapted.

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

---

*Built with Claude Code. Verified by SSH. Documented for reproduction.*

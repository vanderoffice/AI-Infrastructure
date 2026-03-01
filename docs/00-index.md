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
| **[Anthropic](https://www.anthropic.com/)** | The AI safety company that builds Claude and Claude Code. |
| **[API](https://en.wikipedia.org/wiki/API)** (Application Programming Interface) | A set of rules that lets software components communicate with each other. REST APIs use HTTP requests to exchange data. |
| **[Caddy](https://caddyserver.com/)** | A modern web server with automatic HTTPS. Used on the home server to reverse-proxy internal services. |
| **[CI/CD](https://en.wikipedia.org/wiki/CI/CD)** (Continuous Integration / Continuous Deployment) | An automated pipeline that tests code on every commit (CI) and deploys passing builds to production (CD). This guide uses GitHub Actions for CI/CD. |
| **[Claude](https://www.anthropic.com/claude)** | Anthropic's family of large language models (Haiku, Sonnet, Opus). Claude Code uses these models as its reasoning engine. |
| **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** | Anthropic's CLI tool that puts the Claude AI model inside your terminal with full filesystem, shell, and tool access. It can read files, write code, run commands, SSH into servers, and execute multi-step plans autonomously. |
| **CLAUDE.md** | A markdown file that contains behavioral instructions for Claude Code. Placed at the root of a project or in your home directory, it tells Claude Code how to behave -- what to prioritize, what to avoid, and how to approach problems. |
| **[CLI](https://en.wikipedia.org/wiki/Command-line_interface)** (Command-Line Interface) | A text-based interface for interacting with software by typing commands. Claude Code runs in your terminal as a CLI tool. |
| **[Cloudflare](https://www.cloudflare.com/)** | A CDN and security platform that sits between users and your server. Provides DNS, DDoS protection, SSL termination, and caching. |
| **[Containers](https://www.docker.com/resources/what-container/)** | Lightweight, isolated environments for running applications. Unlike virtual machines, containers share the host OS kernel, making them fast to start and resource-efficient. |
| **[DNS](https://en.wikipedia.org/wiki/Domain_Name_System)** (Domain Name System) | The system that translates human-readable domain names (like vanderdev.net) into IP addresses. Pi-hole and Unbound provide custom DNS for the local network. |
| **[Docker](https://www.docker.com/)** | A platform for running applications in isolated containers. Each container packages an application with its dependencies, ensuring consistent behavior across different machines. |
| **[Docker Compose](https://docs.docker.com/compose/)** | A tool for defining and running multi-container Docker applications using a YAML configuration file. Most services in this infrastructure are defined in docker-compose.yml files. |
| **[Dotfiles](https://dotfiles.github.io/)** | Configuration files (often starting with a dot, like `.zshrc` or `.gitconfig`) that define how your shell, editor, and tools behave. Stored in a git repo and synced across all machines for consistency. |
| **[Embeddings](https://en.wikipedia.org/wiki/Word_embedding)** | Numerical representations of text that capture semantic meaning. Similar texts produce similar vectors, enabling semantic search (finding documents by meaning, not just keywords). |
| **[fail2ban](https://github.com/fail2ban/fail2ban)** | An intrusion prevention tool that monitors log files and bans IP addresses that show malicious behavior (like repeated login failures). |
| **[Git](https://git-scm.com/)** | A distributed version control system for tracking changes in source code. The backbone of the dotfiles sync and CI/CD pipeline. |
| **[GitHub Actions](https://docs.github.com/en/actions)** | GitHub's built-in CI/CD platform. Runs automated workflows (build, test, deploy) triggered by git events like pushes or pull requests. |
| **[gravity-sync](https://github.com/vmstan/gravity-sync)** | A tool that replicates Pi-hole blocklist and configuration changes from a primary instance to a secondary. Keeps both DNS servers in sync automatically. |
| **GSD (Get Shit Done)** | A custom project management framework built as a Claude Code skill. It breaks projects into phases and plans, tracks progress, and generates session handoffs so Claude can resume work across sessions. |
| **[LaunchAgents](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)** (macOS) | The macOS scheduling system (similar to [cron](https://en.wikipedia.org/wiki/Cron) on Linux). Used to run nightly backups, log syncs, health checks, and other automated tasks on a timer. |
| **[LLM](https://en.wikipedia.org/wiki/Large_language_model)** (Large Language Model) | An AI model trained on large amounts of text data. Claude, GPT, and Gemini are examples. Claude Code uses an LLM as its reasoning engine. |
| **[MCP](https://modelcontextprotocol.io/)** (Model Context Protocol) | An open protocol that lets AI models connect to external tools and data sources. Think of it as USB-C for AI -- a standard way to plug capabilities into Claude Code. |
| **[Mesh network](https://en.wikipedia.org/wiki/Mesh_networking)** | A network topology where every device can communicate directly with every other device, rather than routing through a central hub. Tailscale creates this across the internet. |
| **[n8n](https://n8n.io/)** | An open-source workflow automation platform (self-hosted alternative to Zapier). Used for bot orchestration, webhook processing, and government document automation. |
| **[nginx](https://nginx.org/)** | A high-performance web server and reverse proxy. Serves static files, handles SSL termination, and routes traffic to backend services on the VPS. |
| **[ntfy](https://ntfy.sh/)** | A simple push notification server. Self-hosted on AlertNode (Pi 3B+) and used for backup failure alerts, monitoring notifications, and the weekly infrastructure digest. |
| **[OrbStack](https://orbstack.dev/)** | A lightweight Docker Desktop alternative for macOS. Faster to start, lower resource usage, and handles both containers and Linux VMs. |
| **[pgvector](https://github.com/pgvector/pgvector)** | A PostgreSQL extension that adds vector similarity search. Used to store and query document embeddings for RAG-powered chatbots. |
| **[Pi-hole](https://pi-hole.net/)** | A DNS-level ad blocker that runs on Raspberry Pis. It filters network traffic for all devices on the network by blocking known advertising and tracking domains at the DNS level. |
| **[Prometheus](https://prometheus.io/)** | An open-source monitoring and alerting toolkit. Scrapes metrics from services, stores time-series data, and triggers alerts when thresholds are crossed. |
| **[RAG](https://en.wikipedia.org/wiki/Retrieval-augmented_generation)** (Retrieval-Augmented Generation) | A technique where an AI retrieves relevant documents from a database before generating a response. This grounds the AI's answers in your actual data instead of relying solely on its training. |
| **[rclone](https://rclone.org/)** | A command-line tool for syncing files to cloud storage providers (Google Drive, S3, etc.). Used for offsite backup in this infrastructure. |
| **[React](https://react.dev/)** | A JavaScript library for building user interfaces. The production website and chatbot UIs are built with React. |
| **Skills / Slash commands** | Custom commands (like /gsd, /deck, /transcript) that extend Claude Code with domain-specific workflows. Each skill is a markdown file that loads instructions and templates when invoked. |
| **[SSH](https://en.wikipedia.org/wiki/Secure_Shell)** (Secure Shell) | An encrypted protocol for remote command-line access to servers. All machine-to-machine communication in this infrastructure uses SSH with Ed25519 keys. |
| **[SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)** (Server-Sent Events) | A transport method for MCP where a server runs on one machine and clients connect to it over the network. Used to share MCP tools across multiple machines from a central hub. |
| **stdio** | A transport method for MCP where the tool runs as a local process on the same machine as Claude Code. Simpler than SSE but not shareable across the network. |
| **[Supabase](https://supabase.com/)** | An open-source backend platform built on PostgreSQL. Provides a database, authentication, real-time subscriptions, and file storage. Self-hosted on the VPS via Docker. |
| **[Symlink](https://en.wikipedia.org/wiki/Symbolic_link)** | A symbolic link -- a file that points to another file or directory. Used extensively in the dotfiles framework to link config files from the repo to their expected locations. |
| **[Tailscale](https://tailscale.com/)** | A mesh VPN service built on WireGuard. It creates encrypted tunnels between all your devices so they can communicate as if they were on the same local network, regardless of physical location. |
| **[TLS/SSL](https://en.wikipedia.org/wiki/Transport_Layer_Security)** | Transport Layer Security (formerly Secure Sockets Layer) -- the encryption protocol that makes HTTPS work. Cloudflare and Let's Encrypt provide TLS certificates for this infrastructure. |
| **[Unbound](https://nlnetlabs.nl/projects/unbound/about/)** | A recursive DNS resolver. Instead of forwarding DNS queries to Google or Cloudflare, Unbound resolves them by walking the DNS hierarchy itself, improving privacy. |
| **[Vaultwarden](https://github.com/dani-garcia/vaultwarden)** | A self-hosted, lightweight implementation of the Bitwarden password manager. Runs behind Caddy on the home server for internal-only access. |
| **[Vite](https://vite.dev/)** | A fast build tool for modern web applications. Handles hot module replacement during development and produces optimized static files for production. |
| **[VPN](https://en.wikipedia.org/wiki/Virtual_private_network)** (Virtual Private Network) | A technology that creates encrypted connections between devices over the internet. Tailscale is the VPN used in this infrastructure. |
| **[VPS](https://en.wikipedia.org/wiki/Virtual_private_server)** (Virtual Private Server) | A virtual machine running in a data center, accessible over the internet. Used here as the production server for the public-facing website, chatbots, and APIs. |
| **[Webhook](https://en.wikipedia.org/wiki/Webhook)** | An HTTP callback -- a URL that receives data when an event occurs. Used by n8n to trigger workflows, by chatbots to receive messages, and by monitoring systems to send alerts. |
| **[WireGuard](https://www.wireguard.com/)** | A modern, lightweight VPN protocol. Tailscale uses WireGuard under the hood to create its encrypted tunnels. |
| **[YAML](https://yaml.org/)** | A human-readable data format used for configuration files. Docker Compose files, GitHub Actions workflows, and many other tools use YAML. |

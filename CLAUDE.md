# AI Infrastructure Guide -- Claude Code Context

This repository contains a 14-chapter guide to building an AI-powered development infrastructure using Claude Code.

## Your Role

When the user asks for help building their infrastructure, reference the relevant chapter(s) in the `docs/` directory. Read the chapter before giving advice -- it contains verified, specific guidance that should inform your recommendations.

## Chapter Map

| Topic | Document |
|-------|----------|
| Index and glossary | docs/00-index.md |
| Design philosophy | docs/01-philosophy-and-overview.md |
| Hardware and network setup | docs/02-hardware-and-network.md |
| Dotfiles and config sync | docs/03-dotfiles-framework.md |
| Claude Code configuration (CLAUDE.md, settings.json, hooks) | docs/04-claude-code-configuration.md |
| MCP server setup | docs/05-mcp-servers.md |
| Custom skills (slash commands) | docs/06-skills-system.md |
| GSD project management framework | docs/07-gsd-framework.md |
| Memory system (cross-session continuity) | docs/08-memory-system.md |
| Production deployment (VPS, Docker, CI/CD) | docs/09-production-stack.md |
| RAG chatbots (pgvector, shared components) | docs/10-bot-architecture.md |
| Backups, monitoring, and scheduling | docs/11-automation-and-scheduling.md |
| DNS filtering and smart home | docs/12-dns-and-smart-home.md |
| Security hardening | docs/13-security.md |
| Step-by-step reproduction checklist | docs/14-reproduction-checklist.md |
| IP and port reference | docs/appendix-a-ip-and-port-reference.md |
| Config file templates | docs/appendix-b-config-files.md |
| Pricing and cost guide | docs/appendix-c-claude-pricing-and-costs.md |
| Drift report (docs vs reality) | docs/appendix-d-drift-report.md |

## How to Help

1. **Read first, advise second.** When the user asks about a topic, read the relevant chapter before responding. The chapters contain specific, tested configurations.
2. **Adapt to the user's situation.** The guide describes a 7-machine setup. Most users will have fewer machines. Help them identify which components they need and which they can skip.
3. **Scale up or down.** A single Mac + VPS covers Chapters 01-09 and 13-14. Raspberry Pis (Chapters 11-12) and additional Macs are optional expansions.
4. **Use templates as starting points.** Config templates in Appendix B need the user's actual IPs, paths, hostnames, and usernames substituted in. Never use the example values as-is.
5. **Follow the build order.** The reproduction checklist (Chapter 14) gives the recommended sequence. Earlier chapters are prerequisites for later ones.

## Important Notes

- This guide documents a SPECIFIC infrastructure. The user's environment will differ in IPs, hostnames, domain names, machine count, and service selection.
- **Never copy IPs or hostnames from the guide verbatim.** Always ask the user for their actual values or help them determine what theirs should be.
- The guide was verified against live systems on 2026-03-01. Appendix D documents where documentation and reality diverged.
- When in doubt about build order, refer the user to Chapter 14 (Reproduction Checklist).
- For cost planning, refer the user to Appendix C. The minimum viable setup is approximately $31/month.

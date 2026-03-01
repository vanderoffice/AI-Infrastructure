## Claude Code Pricing and Cost Management

This appendix covers subscription tiers, external API costs, cost routing strategies, and a realistic monthly budget for running the infrastructure described in this guide.

---

## Claude Code Subscription Tiers

| Tier | Price | Usage Level | Best For |
|------|-------|-------------|----------|
| Claude Pro | $20/mo | Standard usage, rate-limited | Learning Claude Code, light personal projects, occasional use |
| Claude Max 5x | $100/mo | 5x Opus usage | Regular development, daily coding sessions, moderate skill use |
| Claude Max 20x | $200/mo | 20x Opus usage | Heavy agentic workflows, GSD project execution, parallel subagents |

### How to Choose

- **Pro ($20/mo)** -- You are exploring Claude Code for the first time. You use it a few times a week for quick tasks, code reviews, or writing. Rate limits may slow you down during heavy sessions, but you are not bothered by that yet.

- **Max 5x ($100/mo)** -- Claude Code is your daily development tool. You run it for an hour or more per day, use several custom skills, and rely on MCP servers for memory and research. You occasionally hit rate limits during long sessions.

- **Max 20x ($200/mo)** -- You are running multi-phase GSD projects, using parallel subagents, or doing intensive AI-assisted development where rate limits would break your flow. This is the tier used to build the infrastructure described in this guide.

**The key signal:** if you are regularly hitting rate limits, move up a tier. The productivity loss from waiting on rate limits almost always costs more than the subscription difference.

### What Is Included

Claude Code orchestration -- tool calls, file reads, bash commands, skill execution, MCP tool invocations -- is included in your subscription. You are paying for usage of the Claude model, not per API call. This is important for cost routing (see below).

---

## External API Costs

These are costs outside your Claude Code subscription that the infrastructure may incur.

| Service | Cost | When It Triggers |
|---------|------|-----------------|
| Perplexity Deep Research | ~$1/call | Only when a skill explicitly requests it or you say "deep research" |
| OpenAI Embeddings (text-embedding-3-small) | ~$0.02 per 1M tokens | RAG ingestion only -- one-time cost per knowledge base |
| Hostinger VPS (or similar) | ~$10-15/mo | Always running -- your production server |
| Domain registration | ~$12/yr (~$1/mo) | Annual renewal |
| Tailscale | Free | Free tier covers up to 100 devices |
| Cloudflare | Free | Free tier covers DNS, proxy, SSL, basic WAF |
| Google Drive storage | Free (15GB) or $3/mo (100GB) | Offsite backup destination via rclone |
| ntfy.sh | Free (self-hosted) | Push notifications from backup scripts and monitoring |

---

## Cost Routing Strategy

The infrastructure uses a deliberate two-tier model to minimize external API spend.

### Tier 1: Claude Native (Default, $0 Additional)

Claude Code's built-in WebSearch handles:
- Fact lookups and verification
- Documentation searches
- Current events and recent data
- General research questions

This is included in your subscription. There is no per-query cost.

### Tier 2: Perplexity Deep Research (Gated, ~$1/call)

Perplexity deep research is reserved for:
- Skills that explicitly call it (e.g., `/deck` for presentation research, `/gov-factory` for policy research)
- Situations where you explicitly request "deep research"
- Multi-source synthesis that requires crawling multiple pages

It is NOT used for general questions, even if the user says "research this."

### The $85 Lesson

In February 2026, 76 ungated Perplexity deep research calls ran in a single month. Analysis showed that 59 of them did not need deep research -- Claude's native WebSearch would have produced equivalent results. The $85 bill led to implementing the gating system:

1. CLAUDE.md defines a research routing gate
2. By default, all research goes through Claude's native WebSearch ($0)
3. Perplexity deep research only fires when a skill explicitly invokes it or the user says "deep research"
4. A routing doc (`~/.claude/docs/research-routing.md`) lists exact rules

**Result:** Monthly Perplexity costs dropped from $85 to $5-10 with no loss in research quality for routine tasks.

### LLM Gateway Cost Awareness

If you run a multi-model LLM Gateway (for accessing GPT, Gemini, or other models), be aware of a subtle billing trap:

- On Max 20x, Claude Code orchestration is included in your subscription
- If you route text synthesis through the LLM Gateway back to Anthropic models, you pay twice -- once via subscription, once via API
- The gateway should only be used for non-Anthropic models (Gemini for image generation, GPT for specific tasks, etc.)
- Text generation that Claude can do natively should stay inline

---

## Monthly Cost Estimates

### Minimum Viable Setup

For someone just starting out: 1 Mac + 1 VPS + Claude Pro.

| Item | Monthly Cost |
|------|-------------|
| Claude Code (Pro) | $20 |
| VPS (basic tier) | $10 |
| Domain | $1 |
| External APIs | $0 |
| Cloud storage | $0 (15GB free) |
| **Total** | **~$31/mo** |

### Full Infrastructure

For the complete setup described in this guide: 2 Macs + Mac mini server + VPS + 2 Pis + Claude Max 20x.

| Item | Monthly Cost |
|------|-------------|
| Claude Code (Max 20x) | $200 |
| VPS (4 vCPU, 16GB) | $15 |
| Domain | $1 |
| External APIs (gated Perplexity) | $5-10 |
| Cloud storage (100GB Google Drive) | $3 |
| **Total** | **~$224-229/mo** |

### Where the Money Goes

```
Claude Code subscription:  ~87% of total cost
VPS hosting:               ~7%
Everything else:           ~6%
```

The Claude Code subscription dominates. Everything else -- hosting, DNS, storage, external APIs -- is noise by comparison. This is why the cost routing strategy focuses on subscription tier selection rather than nickel-and-diming individual API calls.

---

## Cost Optimization Tips

1. **Start at Pro ($20), upgrade only when rate limits hurt.** Do not preemptively buy Max 20x. Let your actual usage drive the decision.

2. **Gate expensive API calls in CLAUDE.md.** Any external API that costs more than a few cents per call should require explicit invocation, not fire automatically.

3. **Use Claude's native WebSearch before reaching for Perplexity.** It covers 80%+ of research needs at $0 additional cost.

4. **Batch RAG ingestion.** Embedding costs are per-token. Ingest your knowledge base once, not incrementally. Re-ingest only when content changes significantly.

5. **Use the free tiers aggressively.** Cloudflare free tier, Tailscale free tier, ntfy self-hosted, and Google Drive 15GB free cover most infrastructure needs without any spending.

6. **Right-size your VPS.** A $10/mo VPS handles a static site, nginx proxy, and a few Docker containers. You only need $15+ if you are running Supabase, monitoring stacks, or multiple production services.

7. **Monitor your spend monthly.** Check Perplexity usage, OpenAI embedding costs, and VPS billing at the start of each month. Surprises usually come from ungated API calls, not from predictable infrastructure costs.

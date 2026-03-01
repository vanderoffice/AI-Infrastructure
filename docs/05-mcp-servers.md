# External Tool Integration

## Why This Exists

Claude Code is powerful on its own, but it can only read your files, run shell commands, and search the web. It cannot access external APIs, query databases, interact with Google Sheets, search a knowledge graph, or trigger automation workflows -- unless you give it a way to do so.

MCP (Model Context Protocol) is that way. It extends Claude Code by connecting it to tool servers that handle the external integrations. This infrastructure centralizes all shared MCP servers on one always-on machine (ServerM2P) so every dev machine on the network can access the same tools, the same knowledge graph, and the same credentials -- without duplicating anything.

If Claude Code is the brain, MCP servers are the hands.

---

## What Is MCP?

MCP (Model Context Protocol) is Anthropic's open standard for connecting AI models to external tools. Think of it like USB for AI -- a standard interface that lets Claude plug into any service.

An MCP server is a small program that:

1. Exposes **tools** (functions) that Claude can call
2. Runs either as a local process or a network service
3. Handles authentication and API calls to external services on Claude's behalf

When Claude needs to search your knowledge graph, read a Google Sheet, or trigger an n8n workflow, it calls the appropriate MCP tool. The server receives the request, talks to the external service, and returns the result. Claude never touches the API key or the raw HTTP call.

### How Claude Discovers Tools

When Claude Code starts, it reads `~/.claude.json` to find all configured MCP servers. It connects to each one and asks "what tools do you have?" Each server responds with a list of tool names, descriptions, and parameter schemas. Claude then knows what it can call and what arguments each tool expects.

If a server is unreachable at startup, its tools simply do not appear. There is no error -- Claude just has fewer capabilities available.

### Transport Types

MCP servers communicate with Claude Code using one of two transport methods.

| Transport | How It Works | When to Use It |
|-----------|-------------|----------------|
| **stdio** | Claude Code launches the server as a local subprocess. They communicate through standard input/output. | Tools that only need local filesystem access, or tools that should run on the same machine as Claude Code. |
| **SSE** | The server runs as an HTTP service on the network. Claude Code connects to it over HTTP using Server-Sent Events. | Tools that multiple machines need to access, or tools that require credentials you do not want on dev machines. |

**stdio** is simpler. No network, no ports, no firewall rules. Claude Code starts the process, uses it, and the process dies when Claude Code exits.

**SSE** is shareable. The server runs on one machine, and any number of Claude Code instances on any machine can connect to it over the LAN (or via Tailscale from anywhere). This is the foundation of the hub architecture.

---

## The SSE Hub Architecture

The naive approach to MCP is running every server locally on every dev machine. This works for one machine, but it falls apart quickly:

- API keys get scattered across multiple machines
- Each machine needs its own copy of every server installed and running
- The knowledge graph (memory) diverges between machines because each has its own instance
- Credential rotation means updating every machine

The hub architecture solves all of this.

```
StudioM4 (dev)                    ServerM2P (hub)
+-----------------+               +---------------------------+
| Claude Code     |   SSE/HTTP    | MCP Gateway    :3333      |
|                 | ------------> | Memory         :8083      |
| ~/.claude.json  |               | n8n-local      :8081      |
| points to hub   |               | n8n-public     :8082      |
+-----------------+               | Perplexity     :8084      |
                                  | Docling        :8085      |
OfficeM4P (dev)                   | Google Drive   :8087      |
+-----------------+   SSE/HTTP    | GitHub         :8088      |
| Claude Code     | ------------> | Google Sheets  :8089      |
|                 |               +---------------------------+
| ~/.claude.json  |                       |
| points to hub   |                  .env files
+-----------------+              (API keys, 600 perms)
```

### What this gives you

- **Centralized credential management.** API keys live in `.env` files on ServerM2P with `600` permissions (owner read/write only). No API key ever appears in `~/.claude.json` on a dev machine.
- **Multi-machine access.** Both dev machines (StudioM4 and OfficeM4P) connect to the same servers. Add a third dev machine and it gets the same tools by pointing at the same hub.
- **Shared state.** The memory knowledge graph is a single instance. Write a fact from StudioM4, read it from OfficeM4P. No sync, no conflicts, no drift.
- **Single point of maintenance.** Update an MCP server, rotate a key, add a new tool -- do it once on ServerM2P and every client benefits immediately.

### The tradeoff

ServerM2P must be running for shared tools to work. If it goes down, Claude Code still works but loses access to memory, the LLM gateway, research tools, and everything else hosted on the hub. This is acceptable because ServerM2P is an always-on headless server with UPS power, but it is a dependency you should be aware of.

---

## The MCP Server Inventory

### Shared Servers (SSE on ServerM2P)

These run on ServerM2P and are accessible from any machine on the LAN or Tailscale mesh.

| Server | Port | Purpose | Backend |
|--------|------|---------|---------|
| MCP Gateway | 3333 | Multi-model LLM routing (Anthropic, OpenAI, Gemini) | Native node process |
| Memory | 8083 | Shared knowledge graph for cross-session memory | supergateway wrapping `@anthropic/memory-server` |
| n8n-local | 8081 | Local n8n workflow API (dev/test) | mcp-proxy to n8n at :5678 |
| n8n-public | 8082 | Production n8n workflow API | mcp-proxy to VPS n8n |
| Perplexity | 8084 | Research search API (deep research, gated) | supergateway wrapping `perplexity-mcp` |
| Docling | 8085 | Document conversion (PDF, DOCX to structured text) | supergateway |
| Google Drive | 8087 | Google Drive file access | supergateway |
| GitHub | 8088 | GitHub API (repos, issues, PRs) | supergateway |
| Google Sheets | 8089 | Spreadsheet read/write | supergateway |

### Per-Machine Servers (stdio)

These run locally on individual machines and are not shared.

| Server | Machine(s) | Purpose | Why Local |
|--------|-----------|---------|-----------|
| filesystem | All dev machines | Local file access (project-scoped) | Needs direct filesystem access on the machine where Claude Code runs |
| granola | OfficeM4P only | Meeting notes API (localhost:8080) | Granola only runs on the machine with the microphone |

### How Each Server Is Used

**Memory** -- The most important MCP server. It maintains a persistent knowledge graph that survives across Claude Code sessions. When Claude learns something (a project decision, infrastructure fact, debugging discovery), it writes to memory. When a new session starts, it reads from memory to restore context. Because it runs on the hub, the same knowledge is available from any machine.

**MCP Gateway** -- Gives Claude Code access to non-Anthropic models (OpenAI, Google Gemini). Used when a task benefits from a different model's strengths. See the LLM Gateway section below for routing details.

**n8n-local / n8n-public** -- Connect Claude Code to n8n workflow automation. Claude can trigger workflows, check execution status, and build new automations. The local instance is for development/testing; the public instance talks to production n8n on the VPS.

**Perplexity** -- Provides deep web research capabilities beyond Claude's built-in web search. Gated because each deep research call costs approximately $1. See the Cost Routing section below.

**Docling** -- Converts documents (PDF, DOCX, HTML) into structured text that Claude can process. Useful when you need to analyze a document that is not plain text.

**Google Drive / Google Sheets** -- Read and write files in Google Drive and data in Google Sheets. Used for collaborative documents and data that lives in Google Workspace.

**GitHub** -- Full GitHub API access: create repos, manage issues, review PRs, search code. More capable than the `gh` CLI for complex multi-step operations.

---

## The LLM Gateway

The MCP Gateway at port 3333 is a special case. It is not wrapping an external service's MCP server -- it is a custom node application that routes LLM requests to different AI providers based on the task type.

### Why It Exists

Claude Code runs on the Claude model. But sometimes you want a different model for a specific task -- Gemini for image generation, a reasoning-optimized model for complex logic, or a fast model for quick classifications. The gateway provides a single MCP interface that routes to the right model automatically.

### Routing Table

| Task Type | Primary Model | Fallback 1 | Fallback 2 |
|-----------|--------------|------------|------------|
| reasoning | Claude Opus 4.6 | o3 | Gemini 3.1 Pro |
| coding | Claude Sonnet 4.6 | GPT-5.2 | Gemini 3 Flash |
| fast | Claude Haiku 4.5 | GPT-5 mini | Gemini 3 Flash |
| research | Gemini Deep Research | Claude Opus 4.6 | o3 |
| image | Gemini Nano Banana Pro | -- | -- |

### Important Cost Rule

If you are on the Claude Max subscription (which includes Claude Code usage), the Claude model is already paid for. Routing text generation through the gateway back to Anthropic's API means you pay twice -- once through your subscription and again through the API. The gateway is for accessing **non-Anthropic models only**. Text synthesis, code generation, and analysis should use Claude Code's built-in model (which is free under the subscription).

---

## Configuration: ~/.claude.json

This is the file that tells Claude Code where to find its MCP servers. It lives at `~/.claude.json` on each machine.

### Why It Is Not in the Dotfiles Repo

`~/.claude.json` is machine-specific. The `filesystem` server needs paths that differ between machines (`/Users/alice` vs `/Users/bob` vs `/Users/serveradmin`). The `granola` server only exists on OfficeM4P. Syncing this file across machines would break things.

Instead, template configs for each machine are stored at `~/dotfiles/mcp-configs/` for reference:

| Template | Machine | Server Count |
|----------|---------|-------------|
| `studiom4.json` | StudioM4 | 11 |
| `officem4p.json` | OfficeM4P | 12 (includes granola) |
| `serverm2p.json` | ServerM2P | 6 (hosts most natively, fewer SSE clients needed) |

### File Structure

The file has two levels: global servers (available everywhere) and project-level servers (available only when working in a specific directory).

```json
{
  "mcpServers": {
    "memory": {
      "type": "sse",
      "url": "http://192.168.1.100:8083/sse"
    },
    "llm-gateway": {
      "type": "sse",
      "url": "http://192.168.1.100:3333/sse"
    },
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-filesystem"]
    }
  },
  "projects": {
    "/Users/alice": {
      "mcpServers": {
        "github": {
          "type": "sse",
          "url": "http://192.168.1.100:8088/sse"
        }
      }
    }
  }
}
```

**Reading this config:**

- `memory` and `llm-gateway` are **global SSE servers**. They point to ServerM2P (192.168.1.100) on their respective ports. Available in every Claude Code session.
- `filesystem` is a **global stdio server**. It runs locally as a subprocess. The `npx -y` command downloads and runs the package if it is not already installed.
- `github` is a **project-level SSE server**. It is only available when Claude Code is working inside `/Users/alice` (or any subdirectory). This scoping lets you control which tools are available in which contexts.

### SSE URL Pattern

All SSE servers follow the same URL pattern:

```
http://<hub-ip>:<port>/sse
```

The `/sse` path is required. It is the SSE endpoint that supergateway and native MCP servers expose. Without it, the connection will fail silently.

---

## How MCP Servers Run on ServerM2P

The hub machine needs to run each MCP server as a persistent process that starts on boot and restarts on failure. There are three patterns used, depending on the underlying server.

### Pattern 1: supergateway (stdio to SSE)

Most MCP servers are distributed as stdio programs -- they expect to be launched as a subprocess and communicate over standard input/output. To make them accessible over the network, **supergateway** wraps them and exposes an SSE endpoint.

```
supergateway --port 8083 -- npx -y @anthropic/memory-server
              ^                    ^
              Listens on this port  Runs this stdio MCP server
```

**Used for:** memory, perplexity, docling, google-drive, github, google-sheets.

### Pattern 2: mcp-proxy (HTTP to MCP)

Some tools already have an HTTP API (like n8n) but do not speak MCP natively. **mcp-proxy** bridges the gap -- it exposes an MCP interface that proxies to the existing HTTP API.

```
mcp-proxy --port 8081 --target http://localhost:5678/api
           ^                    ^
           MCP endpoint         Existing HTTP API
```

**Used for:** n8n-local (proxies to local n8n at :5678), n8n-public (proxies to VPS n8n).

### Pattern 3: Native Node Process

The MCP Gateway is a custom-built node application that runs directly, not through supergateway or mcp-proxy. It starts as a regular node server and natively speaks the SSE protocol.

```
node /path/to/mcp-gateway/index.js
# Listens on port 3333, speaks SSE natively
```

**Used for:** MCP Gateway only.

### Keeping Servers Running with LaunchAgents

On macOS, LaunchAgents ensure each server starts at boot and restarts on failure. Each server has a `.plist` file in `~/Library/LaunchAgents/`.

| LaunchAgent | Server | Port |
|-------------|--------|------|
| `com.mcp.gateway.plist` | MCP Gateway | 3333 |
| `com.mcp.memory.plist` | Memory | 8083 |
| `com.mcp.n8n-local.plist` | n8n-local | 8081 |
| `com.mcp.n8n-public.plist` | n8n-public | 8082 |
| `com.mcp.perplexity.plist` | Perplexity | 8084 |

A LaunchAgent plist looks like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.mcp.memory</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/npx</string>
        <string>-y</string>
        <string>supergateway</string>
        <string>--port</string>
        <string>8083</string>
        <string>--</string>
        <string>npx</string>
        <string>-y</string>
        <string>@anthropic/memory-server</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/mcp-memory.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/mcp-memory.err</string>
</dict>
</plist>
```

**Key settings:**

- `RunAtLoad` -- Start when the user logs in (or when the machine boots, for headless servers)
- `KeepAlive` -- Restart the process if it crashes
- `StandardOutPath` / `StandardErrorPath` -- Log output for debugging

**On Linux:** Replace LaunchAgents with systemd service files. The concept is identical -- a process manager that starts your server and restarts it on failure.

### Credential Storage

API keys and secrets live in `.env` files on ServerM2P. They are never committed to git, never copied to dev machines, and never placed in `~/.claude.json`.

| File | Contents | Permissions |
|------|----------|-------------|
| `~/mcp-gateway/.env` | API keys for Anthropic, OpenAI, Google AI | `600` |
| `~/.config/n8n-mcp/*.env` | n8n API tokens | `600` |
| `~/.config/perplexity/env` | Perplexity API key | `600` |

`600` permissions mean only the file owner can read or write the file. No other user on the system can access the credentials.

---

## Cost Routing

Not all MCP tools are free to use. External API calls cost money, and without guardrails, costs can escalate quickly. This infrastructure uses a two-tier research model that was implemented after a painful lesson.

### The $85 Lesson

Early in this infrastructure's life, Perplexity deep research calls were ungated. Any Claude Code session could trigger them freely. In one billing period, 76 deep research calls ran -- 59 of which did not actually need deep research. The bill was $85 for research that could have been done with Claude's built-in web search (which is included in the subscription at no additional cost).

The fix was a two-tier routing system.

### Tier 1: Claude Native + WebSearch (Default)

**Cost:** $0 additional (included in the Claude subscription).

Handles: fact lookups, technical documentation, how-to questions, verification of claims, quick research, ad-hoc questions.

This is the default for all research. Claude Code's built-in web search is surprisingly capable and covers the vast majority of research needs.

### Tier 2: Perplexity Deep Research (Gated)

**Cost:** Approximately $1 per call.

Used **only** when:

1. A skill explicitly triggers it (like `/deck` or `/gov-factory`, which are presentation and document generation workflows that need comprehensive research)
2. The user explicitly says "deep research" or "use Perplexity"
3. Multi-dimensional analysis where missing perspectives would be harmful (policy research with stakeholder impact, competitive analysis)

**Everything else uses Tier 1.** If you are unsure whether a query needs deep research, it does not.

### LLM Gateway Cost Awareness

The same principle applies to the LLM Gateway. If you are on the Claude Max subscription:

| Route | Cost | When to Use |
|-------|------|-------------|
| Claude Code's built-in model | $0 (subscription) | Text generation, code writing, analysis, summarization -- almost everything |
| Gateway to OpenAI/Gemini | Per-token API cost | When you specifically need a non-Anthropic model's capabilities |
| Gateway to Anthropic API | Per-token API cost | **Never.** This double-bills. Use the built-in model instead. |

---

## Reproduction Steps

You do not need nine MCP servers to benefit from this architecture. Start with memory (the most valuable), add servers as your needs grow, and decide whether to run them locally or on a hub based on how many machines you have.

### 1. Decide on Architecture

| Situation | Recommendation |
|-----------|---------------|
| One dev machine | Run MCP servers locally (stdio). No hub needed. |
| Two or more dev machines | Set up a hub machine and use SSE. |
| One dev machine + VPS | Run servers locally. The VPS does not need MCP access. |

### 2. Install Node.js on the Hub (or Local Machine)

MCP servers and their tooling are Node.js-based.

```bash
# macOS
brew install node

# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo bash -
sudo apt-get install -y nodejs
```

### 3. Install supergateway

supergateway wraps stdio MCP servers and exposes them as SSE endpoints.

```bash
npm install -g supergateway
```

If you are running everything locally (stdio only), you can skip this step.

### 4. Set Up Your First MCP Server (Memory)

Memory is the most impactful MCP server. It gives Claude Code persistent context across sessions.

**Local (stdio) -- single machine:**

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

No additional setup required. Claude Code will download and run the server automatically.

**Hub (SSE) -- multi-machine:**

On the hub machine, start the server with supergateway:

```bash
# Test it manually first
npx -y supergateway --port 8083 -- npx -y @anthropic/memory-server
```

Verify it works from another machine:

```bash
curl http://<hub-ip>:8083/sse
# Should open an SSE connection (streaming response). Ctrl+C to stop.
```

Then add to `~/.claude.json` on each dev machine:

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

### 5. Make It Persistent

The manual `npx supergateway` command from step 4 dies when you close the terminal. To keep it running permanently:

**macOS (LaunchAgent):**

Create `~/Library/LaunchAgents/com.mcp.memory.plist` with the plist structure shown in the LaunchAgents section above. Then load it:

```bash
launchctl load ~/Library/LaunchAgents/com.mcp.memory.plist
```

**Linux (systemd):**

Create `/etc/systemd/system/mcp-memory.service`:

```ini
[Unit]
Description=MCP Memory Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/npx -y supergateway --port 8083 -- npx -y @anthropic/memory-server
Restart=always
RestartSec=5
Environment=HOME=/root
WorkingDirectory=/root

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable mcp-memory
sudo systemctl start mcp-memory
```

### 6. Add More Servers

Repeat the pattern for each additional server you want:

1. Install the underlying MCP package (check npm for available packages)
2. Create a `.env` file with any required credentials (`chmod 600`)
3. Wrap with supergateway (for SSE) or add directly to `~/.claude.json` (for stdio)
4. Create a LaunchAgent or systemd service to keep it running
5. Test: `curl http://<hub-ip>:<port>/sse`
6. Add the SSE URL to `~/.claude.json` on each dev machine

### 7. Verify in Claude Code

Start a new Claude Code session and ask:

```
What MCP tools do you have access to?
```

Claude will list every tool available from all connected MCP servers. If a server is missing, check that it is running (`curl` the SSE endpoint) and that `~/.claude.json` has the correct URL.

**Remember:** You must restart Claude Code after changing `~/.claude.json`. It reads the config at startup.

---

## Gotchas

These are the things that will cost you time if you do not know about them.

| Issue | Details | Fix |
|-------|---------|-----|
| `~/.claude.json` is not synced | This file is machine-specific. It is NOT in the dotfiles repo. | Keep template configs in `~/dotfiles/mcp-configs/` for reference. Copy and customize per machine. |
| Must restart Claude Code after config changes | `~/.claude.json` is read at startup only. Edits take effect on the next session. | Exit Claude Code and start a new session after any config change. |
| Silent failure on unreachable servers | If an SSE server is down, Claude Code starts normally but those tools are missing. No error, no warning. | If tools are unexpectedly missing, check if the hub is reachable: `curl http://<hub-ip>:<port>/sse` |
| supergateway vs mcp-proxy | These are different tools. supergateway wraps stdio MCP servers as SSE. mcp-proxy bridges HTTP APIs to MCP. | Use supergateway when the package is a stdio MCP server. Use mcp-proxy when the service has an HTTP API but no MCP support. |
| Never put API keys in ~/.claude.json | Keys in this file are visible in plaintext and may be backed up or shared accidentally. | Route all authenticated tools through SSE servers on the hub. Keys stay in `.env` files with `600` permissions. |
| The `/sse` path is required | SSE URLs must end with `/sse`. Just the host and port will not work. | Always use `http://<ip>:<port>/sse` as the URL. |
| npx caching can serve stale versions | `npx -y` caches packages. If an MCP server updates, you may be running an old version. | Clear the npx cache with `npx clear-npx-cache` or specify an exact version: `npx -y @anthropic/memory-server@1.2.3` |
| Docker compose file exists but is unused | The MCP Gateway has a `docker-compose.yml` that was an early experiment. The production setup runs as a native node process. | Ignore the compose file. Use the LaunchAgent / systemd approach. |
| Hub dependency | If ServerM2P goes down, all shared MCP tools are unavailable. Claude Code still works but with reduced capabilities. | For critical workflows, consider running essential servers (like memory) locally as a fallback. |

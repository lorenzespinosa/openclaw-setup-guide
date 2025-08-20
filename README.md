# OpenClaw Setup Guide for Law Firms

> A comprehensive, production-tested guide for deploying [OpenClaw](https://github.com/openclaw-ai/openclaw) as a multi-agent AI platform in a law firm environment. Based on real-world experience running **OpsBot** at [Acme Law](https://www.example.com).

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

> **Disclaimer:** This guide is not affiliated with, endorsed by, or officially connected to OpenClaw or its maintainers. It is an independent community resource based on production deployment experience. Use at your own risk. Always review security configurations with your IT team before deploying in a production legal environment.

---

## Table of Contents

- [What is OpenClaw?](#what-is-openclaw)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Installation (Native systemd)](#installation-native-systemd)
- [Configuration](#configuration)
- [Identity Stack](#identity-stack)
- [Multi-Agent Architecture](#multi-agent-architecture)
- [Slack Bot Setup](#slack-bot-setup)
- [n8n MCP Integration](#n8n-mcp-integration)
- [Security Hardening](#security-hardening)
- [Cost Optimization](#cost-optimization)
- [Monitoring & Maintenance](#monitoring--maintenance)
- [Troubleshooting](#troubleshooting)
- [Contributing](#contributing)
- [License](#license)

---

## What is OpenClaw?

OpenClaw is an open-source **multi-agent AI framework** designed to run as Slack bots (and other interfaces). It provides:

- **Multi-agent orchestration** — a central orchestrator routes tasks to specialist agents
- **Identity stack** — persistent persona, memory, and behavioral configuration via markdown files
- **Tool control** — granular allow/deny lists for filesystem, exec, browser, and custom tools
- **MCP (Model Context Protocol) integration** — connect external tool servers (like n8n) to extend agent capabilities
- **Slack-native** — Socket Mode for real-time messaging, DM allowlists, threading support
- **Session management** — conversation compaction, memory search, session spawning

For law firms, this means you can build an AI assistant that:
- Answers internal ops questions from your knowledge base
- Triggers n8n workflows (intake processing, Filevine updates, Clio billing)
- Enforces strict security boundaries (no arbitrary code execution)
- Maintains a consistent persona and firm-specific context

---

## Architecture Overview

```mermaid
graph TB
    subgraph "Slack Workspace"
        U1[User: Attorney] -->|DM| SB[Slack Bot<br/>Socket Mode]
        U2[User: Paralegal] -->|DM| SB
        U3[User: Ops] -->|Channel mention| SB
    end

    subgraph "VPS (Ubuntu 22.04+)"
        SB --> GW[OpenClaw Gateway<br/>loopback:3578]

        subgraph "Agent Layer"
            GW --> ORCH[Orchestrator Agent<br/>Claude/GPT-4o]
            ORCH -->|spawn session| OPS[Ops Agent<br/>exec allowlist]
            ORCH -->|spawn session| RES[Research Agent<br/>read-only]
            ORCH -->|spawn session| DRAFT[Drafting Agent<br/>write-enabled]
        end

        subgraph "Tool Layer"
            OPS --> MCP[n8n MCP Server<br/>localhost:5678]
            RES --> FS[Filesystem<br/>workspace only]
            DRAFT --> FS
            ORCH --> MEM[Memory Search]
        end

        subgraph "External Services"
            MCP --> N8N[n8n Workflows]
            N8N --> FV[Filevine]
            N8N --> LM[Lawmatics]
            N8N --> CL[Clio]
            N8N --> OP[OpenPhone]
            N8N --> AT[Airtable]
        end
    end

    subgraph "LLM Providers"
        ORCH -.->|primary| ANT[Anthropic Claude]
        ORCH -.->|fallback| OR[OpenRouter]
        OPS -.->|budget| OR
    end

    style GW fill:#1a1a2e,stroke:#e94560,color:#fff
    style ORCH fill:#16213e,stroke:#0f3460,color:#fff
    style MCP fill:#533483,stroke:#e94560,color:#fff
```

### Request Flow

```mermaid
sequenceDiagram
    participant U as Slack User
    participant S as Slack (Socket Mode)
    participant G as Gateway
    participant O as Orchestrator
    participant A as Specialist Agent
    participant M as n8n MCP
    participant E as External API

    U->>S: "Check intake status for John Doe"
    S->>G: WebSocket event
    G->>G: Auth check (DM allowlist)
    G->>O: Route to orchestrator
    O->>O: Classify intent → ops query
    O->>A: Spawn ops-agent session
    A->>M: call n8n tool: check_intake
    M->>E: Lawmatics API → GET contact
    E-->>M: Contact data
    M-->>A: Structured response
    A-->>O: Result summary
    O-->>S: Formatted Slack message
    S-->>U: "John Doe — intake received 3/28, consult scheduled 4/1"
```

---

## Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| **VPS** | 2 vCPU, 4 GB RAM | 4 vCPU, 8 GB RAM |
| **OS** | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| **Node.js** | v20 LTS | v22 LTS |
| **npm** | v10+ | Latest |
| **Slack workspace** | Admin access | Admin access |
| **Domain** | Optional | With SSL (Let's Encrypt) |
| **LLM API key** | 1 provider | Anthropic + OpenRouter |

### Provider API Keys

You will need at least one LLM provider key:

- **Anthropic** — primary for orchestrator (`claude-sonnet-4-20250514` or newer)
- **OpenRouter** — fallback and budget models (free tier for heartbeat/compaction)

---

## Installation (Native systemd)

> **Why not Docker?** In production legal environments, native installs give you direct filesystem access for the identity stack, simpler debugging, lower overhead, and no container networking complexity. systemd handles restarts, logging, and resource limits natively.

### 1. System Preparation

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 22 via NodeSource
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Verify
node --version   # v22.x.x
npm --version    # 10.x.x

# Install build tools (needed for some native modules)
sudo apt install -y build-essential git nginx certbot python3-certbot-nginx
```

### 2. Create Service User

```bash
# Dedicated user — no login shell, no home directory clutter
sudo useradd -r -m -d /opt/openclaw -s /usr/sbin/nologin openclaw

# Create workspace directory
sudo mkdir -p /opt/openclaw/workspace
sudo chown -R openclaw:openclaw /opt/openclaw
```

### 3. Install OpenClaw

```bash
# Switch to openclaw user context
sudo -u openclaw bash

cd /opt/openclaw

# Clone the repo
git clone https://github.com/openclaw-ai/openclaw.git app
cd app

# Install dependencies
npm ci --production

# Copy example configs
cp configs/openclaw.json.example /opt/openclaw/openclaw.json
```

### 4. Environment Variables

```bash
sudo nano /opt/openclaw/.env
```

```env
# LLM Providers
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxxx
OPENROUTER_API_KEY=sk-or-xxxxxxxxxxxxx

# Slack
SLACK_BOT_TOKEN=xoxb-xxxxxxxxxxxxx
SLACK_APP_TOKEN=xapp-xxxxxxxxxxxxx
SLACK_SIGNING_SECRET=xxxxxxxxxxxxxxxx

# Gateway
OPENCLAW_GATEWAY_TOKEN=your-secure-random-token-here

# n8n MCP (if using)
N8N_MCP_URL=http://localhost:5678/mcp
N8N_MCP_TOKEN=your-n8n-mcp-token

# Workspace
OPENCLAW_WORKSPACE=/opt/openclaw/workspace
```

```bash
# Lock down permissions
sudo chmod 600 /opt/openclaw/.env
sudo chown openclaw:openclaw /opt/openclaw/.env
```

### 5. Install systemd Services

```bash
# Copy unit files
sudo cp configs/systemd/openclaw.service /etc/systemd/system/
sudo cp configs/systemd/n8n.service /etc/systemd/system/   # if using n8n

# Reload and enable
sudo systemctl daemon-reload
sudo systemctl enable openclaw
sudo systemctl start openclaw

# Check status
sudo systemctl status openclaw
sudo journalctl -u openclaw -f
```

### 6. Nginx Reverse Proxy (Optional)

Only needed if you expose the gateway externally (not recommended for most setups).

```bash
sudo cp configs/nginx/openclaw-proxy.conf /etc/nginx/sites-available/openclaw
sudo ln -s /etc/nginx/sites-available/openclaw /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx

# SSL with Let's Encrypt
sudo certbot --nginx -d openclaw.yourdomain.com
```

---

## Configuration

The main configuration lives in `openclaw.json`. See [`configs/openclaw.json.example`](configs/openclaw.json.example) for a complete annotated example.

### Key Sections

#### Gateway

```json
{
  "gateway": {
    "bind": "loopback",
    "port": 3578,
    "auth": {
      "mode": "token",
      "token": "$OPENCLAW_GATEWAY_TOKEN"
    }
  }
}
```

- **`bind: "loopback"`** — Only accept connections from localhost. Critical for security.
- **`auth.mode: "token"`** — Require a bearer token for all gateway requests.

#### Agent Defaults

```json
{
  "agents": {
    "defaults": {
      "tools": {
        "allow": ["read"],
        "deny": ["write", "exec", "shell", "system.run", "browser"]
      },
      "compaction": {
        "mode": "safeguard",
        "model": "openrouter/minimax/minimax-m2.5:free"
      }
    }
  }
}
```

- **Default deny** — Every agent starts with read-only access. Opt-in to dangerous tools per agent.
- **Compaction** — Uses a free model for conversation compaction to save costs.

#### Slack

```json
{
  "slack": {
    "socketMode": true,
    "dmPolicy": "allowlist",
    "dmAllowlist": ["U01ABC123", "U02DEF456"],
    "streaming": false
  }
}
```

- **Socket Mode** — No public webhook URL needed. Outbound-only connection.
- **DM allowlist** — Only specified Slack user IDs can DM the bot. Everyone else gets ignored.
- **`streaming: false`** — Disable streaming for stability in production.

---

## Identity Stack

The identity stack is a set of markdown files in the workspace directory that define the bot's persona, behavior, and context. This is what makes OpenClaw unique compared to raw API calls.

```
workspace/
├── SOUL.md          # Core persona and behavioral rules
├── AGENTS.md        # Multi-agent instructions and routing
├── USER.md          # User profiles and permissions
├── TOOLS.md         # Available tools and endpoints
└── HEARTBEAT.md     # Health check and self-monitoring config
```

### SOUL.md — The Core Persona

This is the most important file. It defines who the bot IS.

See [`workspace/SOUL.md.example`](workspace/SOUL.md.example) for a complete template.

Key principles:
- **Be specific** — "You are a legal ops assistant at [Firm Name]" beats "You are helpful"
- **Define boundaries** — What the bot should refuse to do
- **Set tone** — Professional but approachable for attorneys, detailed for paralegals
- **Include firm context** — Practice areas, key systems, common workflows

### AGENTS.md — Multi-Agent Routing

Defines how the orchestrator routes tasks to specialist agents.

See [`workspace/AGENTS.md.example`](workspace/AGENTS.md.example) for a complete template.

### USER.md — User Profiles

Maps Slack user IDs to roles and permissions.

See [`workspace/USER.md.example`](workspace/USER.md.example) for a complete template.

### TOOLS.md — Tool Reference

Documents all available tools and MCP endpoints the bot can use.

See [`workspace/TOOLS.md.example`](workspace/TOOLS.md.example) for a complete template.

### HEARTBEAT.md — Health Monitoring

Configures the bot's self-monitoring behavior.

See [`workspace/HEARTBEAT.md.example`](workspace/HEARTBEAT.md.example) for a complete template.

---

## Multi-Agent Architecture

### Why Multi-Agent?

A single agent with all permissions is a security risk. Multi-agent architecture follows the **principle of least privilege**:

| Agent | Purpose | Tools Allowed | Tools Denied |
|---|---|---|---|
| **Orchestrator** | Routes tasks, manages sessions | read, write, memory_search, sessions_spawn | exec, shell |
| **Ops Agent** | Executes n8n workflows | exec (allowlisted bins only) | write |
| **Research Agent** | Reads workspace files, searches memory | read, memory_search | write, exec, shell |
| **Drafting Agent** | Creates/edits documents | read, write | exec, shell |

### Agent Routing Logic

```mermaid
graph TD
    MSG[Incoming Message] --> ORCH[Orchestrator]
    ORCH -->|Intent: workflow/automation| OPS[Ops Agent]
    ORCH -->|Intent: lookup/search| RES[Research Agent]
    ORCH -->|Intent: create/edit document| DRAFT[Drafting Agent]
    ORCH -->|Intent: general question| ORCH_SELF[Self-handle]

    OPS -->|Result| ORCH
    RES -->|Result| ORCH
    DRAFT -->|Result| ORCH
    ORCH_SELF -->|Response| REPLY[Slack Reply]
    ORCH -->|Compiled result| REPLY

    style ORCH fill:#1a1a2e,stroke:#e94560,color:#fff
    style OPS fill:#533483,stroke:#e94560,color:#fff
    style RES fill:#16213e,stroke:#0f3460,color:#fff
    style DRAFT fill:#0f3460,stroke:#16213e,color:#fff
```

### Spawning Sessions

The orchestrator uses `sessions_spawn` to delegate to specialists:

```
Orchestrator receives: "Run the new-intake workflow for Jane Smith"
→ Classifies as: ops/workflow task
→ Spawns: ops-agent session
→ ops-agent calls: exec → curl → n8n webhook
→ Returns result to orchestrator
→ Orchestrator formats and replies in Slack
```

---

## Slack Bot Setup

### 1. Create a Slack App

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Click **Create New App** → **From an app manifest**
3. Select your workspace

### 2. App Manifest

```yaml
display_information:
  name: OpsBot
  description: AI Legal Ops Assistant
  background_color: "#1a1a2e"

features:
  bot_user:
    display_name: OpsBot
    always_online: true
  app_home:
    home_tab_enabled: true
    messages_tab_enabled: true
    messages_tab_read_only_enabled: false

oauth_config:
  scopes:
    bot:
      - app_mentions:read
      - channels:history
      - channels:read
      - chat:write
      - groups:history
      - groups:read
      - im:history
      - im:read
      - im:write
      - mpim:history
      - mpim:read
      - reactions:read
      - reactions:write
      - users:read

settings:
  event_subscriptions:
    bot_events:
      - app_mention
      - message.im
      - message.groups
      - message.mpim
  interactivity:
    is_enabled: true
  org_deploy_enabled: false
  socket_mode_enabled: true
  token_rotation_enabled: true
```

### 3. Enable Socket Mode

1. Go to **Settings** → **Socket Mode**
2. Enable Socket Mode
3. Generate an **App-Level Token** with `connections:write` scope
4. Save this as `SLACK_APP_TOKEN` in your `.env`

### 4. Install to Workspace

1. Go to **Install App**
2. Click **Install to Workspace**
3. Approve the scopes
4. Copy the **Bot User OAuth Token** → `SLACK_BOT_TOKEN`

### 5. Get User IDs for Allowlist

In Slack: click a user's profile → **More** → **Copy member ID**

Add these IDs to your `openclaw.json` under `slack.dmAllowlist`.

### 6. DM Allowlist Behavior

| User Status | DM Behavior |
|---|---|
| In allowlist | Bot responds normally |
| Not in allowlist | Bot ignores the message silently |
| Channel mention | Bot responds to all @mentions (no allowlist) |
| Thread reply | Bot responds if original message was to it |

---

## n8n MCP Integration

MCP (Model Context Protocol) lets OpenClaw agents call external tool servers. n8n can expose its workflows as MCP tools, giving the bot access to your entire automation stack.

### Architecture

```mermaid
graph LR
    subgraph "OpenClaw"
        OPS[Ops Agent] -->|MCP call| MCP_CLIENT[MCP Client]
    end

    subgraph "n8n"
        MCP_CLIENT -->|HTTP/SSE| MCP_SERVER[MCP Server<br/>localhost:5678/mcp]
        MCP_SERVER --> WF1[Intake Workflow]
        MCP_SERVER --> WF2[Filevine Update]
        MCP_SERVER --> WF3[Billing Check]
        MCP_SERVER --> WF4[OpenPhone SMS]
    end

    subgraph "External APIs"
        WF1 --> LM[Lawmatics]
        WF2 --> FV[Filevine]
        WF3 --> CL[Clio]
        WF4 --> OP[OpenPhone]
    end

    style MCP_CLIENT fill:#e94560,stroke:#1a1a2e,color:#fff
    style MCP_SERVER fill:#533483,stroke:#e94560,color:#fff
```

### Setting Up n8n as MCP Server

1. **Install n8n** (if not already running):

```bash
# Install globally
npm install -g n8n

# Or use the systemd service from this repo
sudo cp configs/systemd/n8n.service /etc/systemd/system/
sudo systemctl enable n8n
sudo systemctl start n8n
```

2. **Enable MCP in n8n** (n8n 1.60+):

```bash
# Add to n8n environment
echo 'N8N_MCP_ENABLED=true' >> /opt/n8n/.env
echo 'N8N_MCP_AUTH_TOKEN=your-mcp-token' >> /opt/n8n/.env
sudo systemctl restart n8n
```

3. **Create MCP-compatible workflows** in n8n:
   - Each workflow that should be callable via MCP needs a **Webhook trigger**
   - Set the webhook to accept JSON payloads
   - Name your workflows descriptively (e.g., `mcp_check_intake_status`)
   - Return structured JSON responses

4. **Configure OpenClaw MCP connection** in `openclaw.json`:

```json
{
  "mcp": {
    "servers": {
      "n8n": {
        "url": "http://localhost:5678/mcp",
        "auth": {
          "type": "bearer",
          "token": "$N8N_MCP_TOKEN"
        },
        "timeout": 30000
      }
    }
  }
}
```

### Example n8n MCP Workflow

**Workflow: `mcp_check_intake_status`**

```
Trigger: MCP Tool Call
→ Extract: contact_name from input
→ Lawmatics API: Search contacts by name
→ Transform: Extract status, next appointment, assigned attorney
→ Return: Structured JSON response
```

**Sample MCP tool call from agent:**
```
Tool: n8n.mcp_check_intake_status
Input: { "contact_name": "Jane Smith" }
Output: {
  "status": "consult_scheduled",
  "next_appointment": "2025-04-01T14:00:00Z",
  "assigned_attorney": "Alex Rivera",
  "source": "lawmatics"
}
```

---

## Security Hardening

Security is non-negotiable in a legal environment. OpenClaw handles conversations that may contain client names, case details, and privileged information.

### Core Principles

1. **Default deny** — No agent gets tools it does not explicitly need
2. **Loopback only** — Gateway binds to 127.0.0.1, never 0.0.0.0
3. **No arbitrary exec** — `exec` tool is allowlisted to specific binaries
4. **Workspace isolation** — Filesystem access restricted to workspace directory
5. **Token rotation** — Rotate gateway and API tokens on a schedule
6. **Audit logging** — Log all tool calls and agent actions

### Exec Control

The most dangerous permission. Lock it down hard:

```json
{
  "exec": {
    "security": "allowlist",
    "safeBins": ["curl"],
    "ask": "always",
    "env": {
      "inherit": false,
      "allow": ["PATH", "HOME"]
    }
  }
}
```

- **`safeBins: ["curl"]`** — Only `curl` can be executed. Nothing else.
- **`ask: "always"`** — Requires confirmation before every exec call (in interactive mode).
- **`env.inherit: false`** — Do not pass the full environment to child processes.

### Token Rotation Schedule

| Token | Rotation Frequency | Method |
|---|---|---|
| Gateway token | Monthly | Regenerate, update `.env`, restart |
| Slack bot token | Quarterly | Rotate via Slack admin panel |
| Slack app token | Quarterly | Rotate via Socket Mode settings |
| Anthropic API key | Quarterly | Regenerate in console |
| OpenRouter key | Quarterly | Regenerate in dashboard |
| n8n MCP token | Monthly | Update in n8n settings |

### Network Security

```bash
# UFW firewall rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp    # nginx HTTP (redirect to HTTPS)
sudo ufw allow 443/tcp   # nginx HTTPS
# Do NOT open 3578 (gateway) or 5678 (n8n) externally
sudo ufw enable
```

See [`security/checklist.md`](security/checklist.md) for a complete hardening checklist.

---

## Cost Optimization

Running AI agents 24/7 can get expensive. Here's how to keep costs down without sacrificing quality.

### Tiered Model Routing

```mermaid
graph TD
    REQ[Incoming Request] --> CLASS[Classify Complexity]
    CLASS -->|Simple/FAQ| FREE[Free Model<br/>minimax-m2.5:free<br/>$0.00]
    CLASS -->|Standard ops| MID[Mid-tier<br/>Claude Haiku / GPT-4o-mini<br/>~$0.001/req]
    CLASS -->|Complex/nuanced| TOP[Premium<br/>Claude Sonnet / GPT-4o<br/>~$0.01/req]

    style FREE fill:#27ae60,stroke:#1a1a2e,color:#fff
    style MID fill:#f39c12,stroke:#1a1a2e,color:#fff
    style TOP fill:#e74c3c,stroke:#1a1a2e,color:#fff
```

### Cost-Saving Strategies

| Strategy | Savings | How |
|---|---|---|
| **Free compaction model** | ~40% on compaction | Use `openrouter/minimax/minimax-m2.5:free` for session compaction |
| **Free heartbeat model** | ~100% on health checks | Use free model for periodic heartbeat/self-check |
| **Tiered routing** | ~60% overall | Route simple queries to cheaper models |
| **Aggressive compaction** | ~30% on long sessions | Compact sessions after 20 messages |
| **Cache workspace files** | ~15% on reads | Bot caches SOUL.md etc. instead of re-reading every turn |

### Monthly Cost Estimates (Moderate Usage)

| Component | Without Optimization | With Optimization |
|---|---|---|
| Orchestrator (Sonnet) | $45/mo | $18/mo |
| Specialist agents | $30/mo | $12/mo |
| Compaction | $15/mo | $0/mo (free model) |
| Heartbeat | $5/mo | $0/mo (free model) |
| n8n (self-hosted) | $0/mo | $0/mo |
| VPS | $20/mo | $20/mo |
| **Total** | **$115/mo** | **$50/mo** |

---

## Monitoring & Maintenance

### Health Checks

OpenClaw's heartbeat system periodically verifies:
- Agent responsiveness
- LLM API connectivity
- MCP server availability
- Memory/disk usage

Configure in `HEARTBEAT.md` (see workspace examples).

### Log Monitoring

```bash
# Real-time logs
sudo journalctl -u openclaw -f

# Last 100 lines
sudo journalctl -u openclaw -n 100

# Filter errors
sudo journalctl -u openclaw --priority=err

# n8n logs
sudo journalctl -u n8n -f
```

### Maintenance Tasks

| Task | Frequency | Command/Action |
|---|---|---|
| Check logs for errors | Daily | `journalctl -u openclaw --since today --priority=err` |
| Review agent tool usage | Weekly | Check Airtable audit log |
| Rotate tokens | Monthly | See [Token Rotation Schedule](#token-rotation-schedule) |
| Update OpenClaw | Monthly | `cd /opt/openclaw/app && git pull && npm ci` |
| Update Node.js | Quarterly | `sudo apt update && sudo apt upgrade nodejs` |
| Review DM allowlist | Monthly | Update `openclaw.json` as staff changes |
| Compact old sessions | Weekly | Automatic if configured in `openclaw.json` |
| Backup workspace | Daily | `rsync -a /opt/openclaw/workspace/ /backup/openclaw/` |

### Restart Procedure

```bash
# Graceful restart
sudo systemctl restart openclaw

# If the bot is unresponsive
sudo systemctl stop openclaw
sleep 2
sudo systemctl start openclaw

# Nuclear option — kill and restart
sudo systemctl kill openclaw
sudo systemctl start openclaw
```

---

## Troubleshooting

### Bot Not Responding to DMs

1. Verify user ID is in `dmAllowlist` in `openclaw.json`
2. Check Slack Socket Mode is connected: `journalctl -u openclaw | grep "socket"`
3. Verify `SLACK_APP_TOKEN` and `SLACK_BOT_TOKEN` are correct
4. Ensure the bot is added to the user's DM (send a message first)

### Gateway Connection Refused

1. Check gateway is running: `curl -s http://localhost:3578/health`
2. Verify `bind: "loopback"` and port match your config
3. Check firewall isn't blocking localhost: `sudo ufw status`

### MCP Tools Not Available

1. Verify n8n is running: `curl -s http://localhost:5678/healthz`
2. Check MCP is enabled in n8n: `N8N_MCP_ENABLED=true`
3. Verify MCP token matches between OpenClaw and n8n
4. Check n8n logs for MCP registration errors

### High API Costs

1. Verify free model is set for compaction and heartbeat
2. Check for conversation loops (agent talking to itself)
3. Review session lengths — enable aggressive compaction
4. Check if research agent is re-reading files unnecessarily

### Agent Spawning Failures

1. Verify orchestrator has `sessions_spawn` in its tool allow list
2. Check agent IDs match between `openclaw.json` and `AGENTS.md`
3. Review memory limits — spawned sessions consume RAM

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

---

## License

MIT License. See [LICENSE](LICENSE).

Copyright (c) 2025 Lorenz Espinosa.

---

*Built with real production experience at [Acme Law](https://www.example.com). If this guide saved you time, give it a star.*

## FAQ

**Q: Does this work with Docker?**
A: This guide covers native systemd deployment. Docker works but we recommend native for production stability.

**Q: Which LLM provider should I use?**
A: We recommend Anthropic Claude for primary conversations and a free model (minimax) for heartbeat/compaction to control costs.

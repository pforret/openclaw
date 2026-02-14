# OpenClaw Architecture

This document describes the high-level architecture of OpenClaw — a personal AI assistant platform that unifies multiple messaging channels under a single agentic runtime. The description is language-independent and focuses on structural components, data flow, and integration points.

---

## Overview

OpenClaw is composed of five major subsystems:

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLI / Native Apps                        │
│              (command-line, macOS, iOS, Android)                 │
└──────────────────────────────┬──────────────────────────────────┘
                               │ WebSocket (JSON-RPC)
┌──────────────────────────────▼──────────────────────────────────┐
│                      Gateway (Control Plane)                    │
│  ┌──────────┐  ┌──────────┐  ┌─────────┐  ┌────────────────┐   │
│  │ Channels │  │  Router  │  │  Cron   │  │  Plugin Host   │   │
│  └────┬─────┘  └────┬─────┘  └────┬────┘  └───────┬────────┘   │
│       │              │             │               │            │
│  ┌────▼──────────────▼─────────────▼───────────────▼────────┐   │
│  │                  Agent Runtime (Sessions)                 │   │
│  │  ┌───────────┐  ┌────────┐  ┌────────┐  ┌────────────┐  │   │
│  │  │ LLM Call  │  │ Tools  │  │ Skills │  │  Memory    │  │   │
│  │  └───────────┘  └────────┘  └────────┘  └────────────┘  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
   ┌────────────┐     ┌──────────────┐     ┌──────────────┐
   │  External  │     │    Model     │     │   Storage    │
   │  Channels  │     │  Providers   │     │   (Local)    │
   │ (WhatsApp, │     │ (Anthropic,  │     │  (Sessions,  │
   │  Telegram, │     │  OpenAI,     │     │   Config,    │
   │  Slack...) │     │  Google...)  │     │   Memory)    │
   └────────────┘     └──────────────┘     └──────────────┘
```

---

## 1. Gateway (Messaging Gateway / Control Plane)

The **Gateway** is the central orchestration hub. It runs as a long-lived server process and coordinates all communication between clients, channels, agents, and external services.

### Responsibilities

| Concern | Description |
|---|---|
| **WebSocket Server** | Accepts client connections (CLI, native apps, web UI) using a JSON-RPC 2.0 protocol over WebSocket |
| **Channel Lifecycle** | Starts, stops, and authenticates channel connections (WhatsApp, Telegram, Slack, etc.) |
| **Message Routing** | Routes inbound messages to the correct agent and session based on sender, channel, and group rules |
| **HTTP Endpoints** | Exposes HTTP routes for webhook ingestion, cron callbacks, health checks, and plugin endpoints |
| **Config Management** | Loads, validates, hot-reloads configuration without restart |
| **Device Discovery** | Discovers and pairs local devices (phones, desktops) via mDNS / Bonjour |
| **Presence Tracking** | Tracks online/offline state of agents, channels, and connected clients |
| **Plugin Host** | Loads and manages the lifecycle of plugins (channels, skills, hooks) |
| **Networking** | Optional exposure via Tailscale Serve/Funnel for remote access |

### Protocol

All client ↔ gateway communication uses **JSON-RPC 2.0 over WebSocket**:

```
→ Request:   { method, params, id }
← Response:  { result, id }
← Broadcast: { method, params }       (server-initiated, no id)
```

Key method families:

- `agent.*` — chat, run, status
- `chat.*` — send messages, stream replies
- `config.*` — get, set, patch, apply, reload
- `sessions.*` — list, get, update, spawn
- `cron.*` — add, remove, run
- `devices.*` — register, pair
- `models.*` — list, select
- `hooks.*` — install, remove

### HTTP Interface

A secondary HTTP server handles:

- `POST /hook/:id` — Generic inbound webhook
- `POST /cron/:id` — Cron execution callback
- `GET /healthz` — Health check
- Platform-specific callbacks (e.g., Gmail Pub/Sub)

---

## 2. Channels (External Messaging Platforms)

Channels are **pluggable adapters** that bridge external messaging platforms into the unified OpenClaw message format.

### Supported Channels

| Channel | Connection Method |
|---|---|
| WhatsApp | Persistent WebSocket (Baileys) |
| Telegram | Long-polling / Webhook (Bot API) |
| Discord | WebSocket gateway (Bot) |
| Slack | Socket Mode / Events API (Bot) |
| Signal | Local CLI bridge |
| iMessage | Local BlueBubbles bridge |
| Google Chat | API + Pub/Sub |
| Microsoft Teams | Graph API |
| Matrix | Client-Server API |
| WebChat | Built-in web UI |
| *Others* | Via plugin/extension system |

### Channel Plugin Interface

Each channel implements a standard contract:

```
ChannelPlugin
├── startAccount(context)    → Establish connection, begin listening
├── stopAccount(context)     → Gracefully disconnect
├── inbound(message)         → Normalize incoming message
├── outbound(target, content)→ Deliver message to platform
└── status                   → Connection health & metadata
```

### Inbound Processing Pipeline

When a message arrives from any channel, it passes through:

1. **Normalization** — Convert platform-specific format to unified envelope (sender, text, media, metadata)
2. **Allowlist Check** — Verify sender is authorized (pairing policy)
3. **Mention Gating** — In groups, check if the bot was mentioned or a trigger keyword was used
4. **Command Detection** — Detect built-in commands (e.g., `/new`, `/status`, `/help`)
5. **Media Staging** — Download and stage media files to a temporary sandbox
6. **Session Resolution** — Determine target agent and session key
7. **Debouncing** — Aggregate rapid sequential messages before triggering the agent

### Outbound Delivery

Responses from the agent are:

1. **Chunked** per channel's message length limits
2. **Formatted** for the target platform (Markdown, HTML, plain text)
3. **Media uploaded** to the platform if attachments are present
4. **Delivered** via the channel plugin's outbound handler

---

## 3. Agent Runtime

The **Agent Runtime** manages AI model interaction, tool execution, and multi-turn conversation sessions.

### Session Model

A **session** is a persistent, ordered sequence of conversational turns stored as an append-only log.

```
Session File (append-only log, one entry per line)
├── user          — Inbound message from a human
├── assistant     — Model-generated response
├── tool_call     — Agent-initiated tool invocation
└── tool_result   — Execution result returned to the agent
```

Sessions are identified by a composite key: `(agentId, sessionKey)`. The default session key is `"main"`.

### Execution Loop

```
User message arrives
  │
  ▼
Append user turn to session log
  │
  ▼
Compose system prompt (identity + skills + tools + context)
  │
  ▼
Call LLM with full conversation history
  │
  ▼
Stream response ──► For each tool_call:
  │                    ├── Validate against tool policy
  │                    ├── Execute tool handler
  │                    ├── Append tool_call + tool_result to session
  │                    └── Continue LLM inference with result
  │
  ▼
Append final assistant text to session
  │
  ▼
Deliver response to channel
```

### System Prompt Composition

The system prompt is dynamically assembled from multiple sources:

1. **Agent Identity** — Name, description, persona
2. **Current Context** — Timezone, date, active channel
3. **Tool Definitions** — JSON schemas for all available tools
4. **Skill Instructions** — Injected prompt fragments from installed skills
5. **Safety Guidelines** — Guardrails and usage policies
6. **Model Capabilities** — Adjusted based on selected model

### Multi-Agent Support

Multiple agents can be configured, each with:

- Independent system prompt and persona
- Separate session storage
- Distinct tool permissions
- Different model selections
- Own skill installations

Agents can **spawn sub-agents** via the `sessions.spawn` tool for delegation.

### Context Window Management

When conversation history exceeds the model's context window:

- **Compaction** summarizes older turns while preserving recent context
- Tool results can be truncated or elided
- Session repair recovers from corrupted or oversized logs

---

## 4. Tools

Tools are capabilities that the agent can invoke during a conversation. Each tool is defined by a **name**, **JSON schema** (parameters), and an **execution handler**.

### Built-in Tools

| Tool | Purpose |
|---|---|
| `system.run` | Execute shell commands in a sandboxed PTY |
| `browser.action` | Automate a headless browser (navigate, click, screenshot, scrape) |
| `canvas.eval` | Update a live canvas UI (render HTML/JS content) |
| `canvas.reset` | Clear the canvas |
| `messaging.send` | Send a message via any connected channel |
| `sessions.list` | List available sessions |
| `sessions.get` | Read session history |
| `sessions.update` | Modify session metadata |
| `sessions.spawn` | Create a sub-agent session |
| `camera.snap` | Capture a photo from a connected device |
| `camera.clip` | Record a short video clip |
| `location.get` | Get current location from a device |
| `agents.list` | List available agents |

### Tool Policy

Tool access is governed per-agent:

- **Allow list** — Additional tools beyond defaults
- **Deny list** — Explicitly blocked tools
- **Approval gating** — Require human approval before execution

### Tool Execution Sandboxing

Shell commands (`system.run`) execute in a restricted environment:

- Isolated working directory
- No elevated privileges
- Process timeout enforcement
- File path validation
- Output size limits

---

## 5. Memory (Vector Search & Embeddings)

The **Memory** subsystem provides semantic search over past conversations and documents using vector embeddings.

### Architecture

```
Documents / Messages
        │
        ▼
  ┌─────────────┐
  │  Embedding   │  (OpenAI, Google Gemini, Voyage AI, Ollama)
  │   Provider   │
  └──────┬──────┘
         │ vectors
         ▼
  ┌─────────────┐
  │  Vector DB   │  (SQLite + sqlite-vec)
  │  (Local)     │
  └──────┬──────┘
         │
         ▼
  ┌─────────────┐
  │   Hybrid     │  BM25 (keyword) + Vector (semantic)
  │   Search     │
  └─────────────┘
```

### Features

- **Async batch embedding** — Efficient bulk processing
- **Hybrid search** — Combines BM25 keyword matching with vector similarity
- **Scoped queries** — Filter by agent, session, time range
- **Deduplication** — Avoid re-embedding unchanged content
- **Atomic reindexing** — Safe full re-index without downtime
- **Stale cleanup** — Prune orphaned embeddings

### Embedding Providers

| Provider | Type |
|---|---|
| OpenAI | Cloud API |
| Google Gemini | Cloud API |
| Voyage AI | Cloud API |
| Ollama | Local inference |

---

## 6. Triggers & Scheduling

OpenClaw supports multiple ways to trigger agent activity beyond direct messages.

### Cron Jobs

Scheduled tasks that run agent actions at defined intervals.

```
Cron Definition
├── schedule       — Cron expression (e.g., "0 9 * * MON")
├── agent          — Target agent ID
├── action         — Message or command to execute
├── delivery mode  — async | best-effort | strict
└── cooldown       — Minimum interval between runs
```

Features:
- Persistent job storage (survives restarts)
- Execution history tracking
- Rate limiting and cooldown enforcement
- Multi-agent execution support

### Webhooks

External systems can trigger agent actions via HTTP:

```
POST /hook/:hookId
Authorization: Bearer <token>
Content-Type: application/json

{ "event": "...", "data": { ... } }
```

The payload is routed to the configured agent and session, which processes it as an inbound message.

### Platform Events (Pub/Sub)

Integrations with platform-specific event systems:

- **Gmail Pub/Sub** — New email notifications trigger processing
- **Device Events** — Camera captures, location updates, screen recordings from connected mobile/desktop devices
- **Custom Events** — User-defined event hooks

### Hook System

Hooks are event handlers that execute at specific lifecycle points:

| Hook Point | When It Fires |
|---|---|
| `beforeToolCall` | Before a tool is executed |
| `afterToolCall` | After a tool completes |
| `onMessage` | When a new inbound message arrives |
| `onSession` | Session created, activated, or closed |
| `onCompaction` | Before context window compaction |
| `onGatewayStart` | Gateway process starts |
| `configApply` | Configuration is changed |

Hooks can be **bundled** (built-in), registered by **plugins**, or defined as **custom modules**.

---

## 7. Plugin & Extension System

OpenClaw is designed for extensibility through a layered plugin architecture.

### Plugin Types

| Type | Purpose | Examples |
|---|---|---|
| **Channel Plugin** | Add new messaging platforms | MS Teams, Matrix, Zalo |
| **Skill Plugin** | Add tools and prompt fragments | Web search, email drafting, task management |
| **Hook Plugin** | React to lifecycle events | Logging, analytics, custom routing |
| **HTTP Plugin** | Add custom HTTP endpoints | OAuth callbacks, custom APIs |

### Skill Structure

Skills are the primary extension mechanism for adding agent capabilities:

```
skill/
├── SKILL.md          — Metadata, description, documentation
├── index.*           — Tool definitions (schema + handler)
└── prompt.md         — System prompt fragment injected into agent
```

Skills can be:
- **Bundled** — Shipped with OpenClaw
- **Managed** — Installed from a registry
- **Workspace** — Custom, per-agent local definitions

### Plugin Lifecycle

```
1. Discovery   — Scan file system or registry for manifests
2. Validation  — Verify manifest schema and dependencies
3. Loading     — Import module (lazy or eager)
4. Registration— Register hooks, tools, config extensions
5. Activation  — Plugin begins responding to events
6. Reload      — Hot-reload on config change (no restart)
```

---

## 8. Configuration & State Management

### Configuration Hierarchy

Settings are resolved with the following precedence (highest first):

1. **CLI arguments**
2. **Environment variables** (process-level)
3. **`.env` file** (working directory)
4. **`~/.openclaw/.env`** (daemon-level)
5. **`openclaw.json`** (main config file)
6. **Defaults** (built-in)

### Configuration Domains

| Domain | What It Controls |
|---|---|
| `gateway` | Port, bind address, authentication |
| `agents` | Agent definitions, personas, model assignments |
| `channels` | Per-channel credentials and settings |
| `models` | LLM provider configuration and failover |
| `tools` | Tool permissions and restrictions |
| `hooks` | Hook module paths |
| `memory` | Embedding backend and search settings |
| `sessions` | Retention, pruning, concurrency |
| `cron` | Scheduled job definitions |
| `plugins` | Plugin paths and configuration |

### Hot-Reload

Configuration changes are applied without restarting the gateway:

1. File watcher detects change to config file
2. New config is validated against schema
3. Atomic swap of in-memory state
4. Change broadcast to all connected clients
5. Affected subsystems re-initialize gracefully

### Local Storage Layout

```
~/.openclaw/
├── openclaw.json              — Main configuration
├── .env                       — Environment overrides
├── agents/
│   └── <agentId>/
│       ├── sessions/          — Conversation logs (append-only)
│       ├── memory/            — Vector database files
│       └── workspace/         — Agent-specific skills & files
├── credentials/               — Encrypted auth tokens
├── cache/                     — HTTP and embedding caches
└── plugins/                   — Plugin state and data
```

---

## 9. Model Provider Integration

OpenClaw supports multiple LLM providers with automatic failover.

### Supported Providers

| Provider | Models |
|---|---|
| Anthropic | Claude family |
| OpenAI | GPT-4, GPT-4o, o-series |
| Google | Gemini family |
| Ollama | Any local model |
| Together AI | Open-source models |
| OpenRouter | Multi-provider proxy |
| AWS Bedrock | Managed models |
| GitHub Copilot | Via proxy extension |
| *Others* | Via plugin system |

### Auth Profiles

Each provider connection is managed as an **auth profile**:

- API key or OAuth token
- Rate limit tracking
- Cooldown on failures
- Round-robin or last-used selection

### Failover Strategy

When a model call fails:

1. Classify error (rate limit, auth failure, context overflow, network)
2. Apply cooldown to the failing profile
3. Select next available profile (same provider or fallback provider)
4. Retry with adjusted parameters if needed (e.g., reduced context on overflow)

---

## 10. Security Model

### Authentication

| Boundary | Mechanism |
|---|---|
| Gateway clients | Token-based (`OPENCLAW_GATEWAY_TOKEN`) or password |
| Channel senders | DM pairing policy (pairing code, allowlist, or open) |
| Webhooks | Bearer token per hook |
| Remote access | Tailscale identity or TLS + token |

### DM Pairing Policy

Controls who can message the agent through channels:

- **`pairing`** (default) — Unknown senders receive a one-time pairing code that must be approved
- **`allowlist`** — Only pre-approved senders
- **`open`** — Any sender (requires explicit opt-in)

### Sandboxing

- Shell commands run in an isolated PTY with no elevated privileges
- Media files are staged to a temporary sandbox directory
- File path validation prevents directory traversal
- Process timeouts prevent runaway execution

### Secret Management

- Secrets stored as environment variables or in an encrypted credentials store
- Automatic redaction in logs for patterns matching `*_KEY`, `*_TOKEN`, `PASSWORD`
- Config file should not contain secrets (use `.env` or credentials store)

---

## 11. Deployment Models

### Local Daemon

Runs as a background service on the user's machine:

- **macOS**: launchd agent
- **Linux**: systemd user service
- **Windows**: WSL2 service

### Container

Docker / Docker Compose deployment with isolated services:

- Gateway container (main server)
- Optional browser automation container
- Optional execution sandbox container

### Remote Access

For accessing the gateway from mobile apps or remote machines:

- **Tailscale Serve/Funnel** — Built-in zero-config secure tunneling
- **SSH Tunnel** — Manual port forwarding
- **Reverse Proxy** — Behind nginx/Caddy with TLS

---

## 12. Data Flow Summary

### Complete Message Lifecycle

```
1. External Platform (e.g., WhatsApp)
   │
   ▼
2. Channel Plugin — normalizes to unified format
   │
   ▼
3. Gateway Router — allowlist, mention gating, command detection
   │
   ▼
4. Auto-Reply System — debounce, media staging, session resolution
   │
   ▼
5. Agent Runtime — append to session, compose prompt
   │
   ▼
6. LLM Provider — inference with tools
   │
   ├──► Tool Execution (shell, browser, canvas, messaging...)
   │    │
   │    └──► Result back to LLM for continued reasoning
   │
   ▼
7. Response Streaming — text blocks, tool results, thinking
   │
   ▼
8. Reply Dispatcher — chunk, format per channel limits
   │
   ▼
9. Channel Outbound — upload media, deliver to platform
   │
   ▼
10. External Platform (response delivered)
```

### Trigger-Initiated Flow

```
Trigger Source (Cron / Webhook / Device Event / Pub/Sub)
   │
   ▼
Gateway HTTP or Event Handler
   │
   ▼
Route to Agent + Session
   │
   ▼
Agent Execution Loop (same as steps 5-10 above)
```

---

## Glossary

| Term | Definition |
|---|---|
| **Gateway** | The central WebSocket server that orchestrates all subsystems |
| **Channel** | A pluggable adapter for an external messaging platform |
| **Agent** | A configured AI persona with its own model, tools, and sessions |
| **Session** | A persistent multi-turn conversation between a user and an agent |
| **Skill** | An installable package that adds tools and/or prompt instructions to an agent |
| **Hook** | An event handler that executes at a specific lifecycle point |
| **Plugin** | A loadable module that extends OpenClaw (channels, skills, hooks, or HTTP endpoints) |
| **Auth Profile** | A credential set for connecting to an LLM provider |
| **Compaction** | The process of summarizing old conversation turns to fit the context window |
| **Pairing** | The security handshake that authorizes a new sender to interact with an agent |
| **Node** | A device (phone, desktop) connected to the gateway for hardware access |

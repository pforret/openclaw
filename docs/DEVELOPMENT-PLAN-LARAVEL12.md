# OpenClaw Laravel 12 — Development Plan

A phased roadmap to build the OpenClaw equivalent in Laravel 12, starting from a minimal working prototype and progressing to full feature parity.

---

## Phase Overview

```
Phase 0  ██░░░░░░░░░░░░░░░░░░  Scaffolding & Foundation      (~1 week)
Phase 1  ████░░░░░░░░░░░░░░░░  Core Agent Loop                (~2 weeks)
Phase 2  ██████░░░░░░░░░░░░░░  First Two Channels             (~2 weeks)
Phase 3  ████████░░░░░░░░░░░░  WebSocket Gateway & Web UI     (~2 weeks)
Phase 4  ██████████░░░░░░░░░░  Tools & Sandboxed Execution    (~2 weeks)
Phase 5  ████████████░░░░░░░░  Memory & Vector Search         (~2 weeks)
Phase 6  ██████████████░░░░░░  Multi-Agent, Skills, Plugins   (~2 weeks)
Phase 7  ████████████████░░░░  Triggers, Cron, Webhooks       (~1 week)
Phase 8  ██████████████████░░  Go Sidecars & FrankenPHP       (~2 weeks)
Phase 9  ████████████████████  Remaining Channels & Polish    (~2 weeks)
```

Each phase produces a **working, testable system**. No phase depends on unreleased future work — you can stop at any phase and have a usable product.

---

## Phase 0 — Scaffolding & Foundation

**Goal:** Fresh Laravel 12 project with SQLite, core database schema, contracts, enums, and config files. Nothing runs yet, but the skeleton is in place for all subsequent phases.

### Tasks

#### 0.1 Project Bootstrap

```bash
composer create-project laravel/laravel openclaw-laravel
cd openclaw-laravel
```

- Configure SQLite in `config/database.php`
- Add SQLite performance pragmas in `AppServiceProvider::boot()` (WAL, cache_size, busy_timeout)
- Set up `.env.example` with all expected keys (empty values)

#### 0.2 Enums

Create PHP 8.4 backed enums:

| File | Cases |
|---|---|
| `app/Enums/ChannelType.php` | `Telegram`, `WebChat` (start small, extend later) |
| `app/Enums/TurnRole.php` | `User`, `Assistant`, `ToolCall`, `ToolResult` |
| `app/Enums/DeliveryMode.php` | `Async`, `BestEffort`, `Strict` |
| `app/Enums/PairingPolicy.php` | `Pairing`, `Allowlist`, `Open` |

#### 0.3 Migrations

Create all migrations upfront (even if models aren't used until later phases). This locks in the schema early and prevents migration churn.

| Migration | Key Columns |
|---|---|
| `create_agents_table` | `id`, `name`, `description`, `config` (JSON), `is_active` |
| `create_sessions_table` | `id`, `agent_id` (FK), `key`, `metadata` (JSON) |
| `create_turns_table` | `id`, `session_id` (FK), `role` (enum), `content` (JSON), `created_at` |
| `create_channels_table` | `id`, `type` (enum), `credentials` (encrypted), `is_active`, `metadata` (JSON) |
| `create_auth_profiles_table` | `id`, `provider`, `credentials` (encrypted), `is_active`, `cooldown_until`, `requests_today` |
| `create_webhooks_table` | `id`, `agent_id` (FK), `session_key`, `token`, `is_active` |
| `create_cron_jobs_table` | `id`, `agent_id` (FK), `schedule`, `action`, `delivery_mode`, `cooldown_minutes`, `is_active`, `last_run_at` |
| `create_cron_executions_table` | `id`, `cron_job_id` (FK), `status`, `output`, `executed_at` |
| `create_paired_senders_table` | `id`, `sender_id`, `channel_type`, `pairing_code`, `approved_at` |
| `create_embeddings_table` | `id`, `agent_id` (FK), `session_id` (FK), `turn_id` (FK), `content`, `vector` (BLOB), `content_hash` |
| `create_skills_table` | `id`, `name`, `version`, `prompt_fragment`, `tool_class`, `is_active` |
| `create_agent_skill_table` | `agent_id`, `skill_id` (pivot) |
| `create_plugins_table` | `id`, `name`, `version`, `service_provider`, `is_active` |
| `create_jobs_table` | Laravel queue table |

Run `php artisan migrate`.

#### 0.4 Eloquent Models (Stubs)

Create all models with relationships, casts, and fillable attributes. No business logic yet — just Eloquent scaffolding.

- `Agent` — hasMany Sessions, belongsToMany Skills
- `Session` — belongsTo Agent, hasMany Turns
- `Turn` — belongsTo Session, immutable guard in `booted()`
- `Channel` — encrypted credentials cast
- `AuthProfile` — encrypted credentials cast, `applyCooldown()` method
- `CronJob` — belongsTo Agent, hasMany CronExecutions
- `CronExecution` — belongsTo CronJob
- `Webhook` — belongsTo Agent
- `PairedSender` — `isAuthorized()` static method
- `Embedding` — belongsTo Agent, Session, Turn
- `Skill` — belongsToMany Agents
- `Plugin` — basic CRUD

#### 0.5 Contracts (Interfaces)

| File | Methods |
|---|---|
| `app/Contracts/ChannelDriver.php` | `start()`, `stop()`, `normalizeInbound()`, `send()`, `status()` |
| `app/Contracts/LLMProvider.php` | `chat()`, `stream()` |
| `app/Contracts/EmbeddingProvider.php` | `embed()`, `batchEmbed()` |
| `app/Contracts/Tool.php` | `name()`, `schema()`, `execute()` |

#### 0.6 Config Files

| File | Contents |
|---|---|
| `config/openclaw.php` | Gateway port/token, debounce_ms, pairing_policy |
| `config/agents.php` | Default agent definition (name, model, tool policy) |
| `config/channels.php` | Per-channel credentials (empty, from .env) |
| `config/models.php` | Provider configs + failover_order |
| `config/tools.php` | Global tool settings (bash timeout, max output) |

#### 0.7 Data Transfer Objects

Create simple DTOs (plain PHP classes or readonly classes) used across the codebase:

- `InboundMessage` — sender, text, media, channelType, metadata
- `OutboundMessage` — target, text, media, channelType
- `ToolCall` — name, params, id
- `ToolResult` — output, exitCode, error
- `LLMResponse` — text, toolCalls, finishReason
- `ChannelStatus` — connected, lastActivity, metadata

#### 0.8 Seeders

- `AgentSeeder` — create a default agent with sensible config
- `AuthProfileSeeder` — create one auth profile per configured provider (from .env)

### Deliverable

```bash
php artisan migrate --seed
php artisan tinker
# Agent::first()  → returns the default agent
# Session, Turn, etc. are all queryable
```

### Tests

- Feature test: migrations run without error
- Unit test: `Turn` model prevents updates (immutability guard)
- Unit test: `PairedSender::isAuthorized()` logic for all three policies

---

## Phase 1 — Core Agent Loop (CLI-Only)

**Goal:** A working agent that you can chat with via `php artisan agent:chat`. Single LLM provider (Anthropic), no channels, no tools, no WebSocket. Just the core execution loop.

### Tasks

#### 1.1 Anthropic Provider

Implement `AnthropicProvider` — the first `LLMProvider`:

- HTTP client using `Http::withHeaders()` to call Anthropic Messages API
- Parse response into `LLMResponse` DTO
- Handle tool_use blocks in response (parse but don't execute yet)
- Basic error handling: 429 rate limit, 401 auth, 500 server errors

#### 1.2 Provider Factory & Model Router (Minimal)

- `ProviderFactory::make()` — instantiate provider from config + auth profile
- `ModelRouter::resolve()` — for now, just return the first available provider (no failover yet)

#### 1.3 System Prompt Builder (Minimal)

- Read agent name + description from `Agent` model
- Inject current date/timezone
- No tool schemas or skill fragments yet — just identity + context

#### 1.4 Context Compactor (Stub)

- `prepare()` loads all turns from session, converts to Anthropic message format
- No actual compaction logic yet — just pass through
- Add a hard limit: if turns exceed N (e.g., 50), drop the oldest ones

#### 1.5 Agent Runner

Implement the core loop from the architecture doc:

1. Create user Turn
2. Build system prompt
3. Prepare history
4. Call LLM
5. Create assistant Turn
6. (No tool loop yet)

#### 1.6 Artisan Command: `agent:chat`

Interactive CLI command:

```
php artisan agent:chat

 Assistant (claude-sonnet-4-5-20250929)

You: Hello, who are you?
Assistant: I'm Assistant, a helpful AI assistant...

You: What did I just ask?
Assistant: You asked me who I am...
```

- Reads from stdin in a loop
- Calls `AgentRunner::run()` synchronously (no queue)
- Prints assistant response to stdout
- Creates a session with key `"cli"`
- Supports `Ctrl+C` to exit

#### 1.7 Artisan Command: `doctor` (Minimal)

Check that the basics are healthy:

- SQLite database exists and is writable
- At least one auth profile is configured
- Anthropic API key is set and valid (make a tiny test call)

### Deliverable

```bash
# Set your API key
echo "ANTHROPIC_API_KEY=sk-ant-..." >> .env

# Seed
php artisan migrate:fresh --seed

# Chat
php artisan agent:chat
```

A full multi-turn conversation in the terminal, with sessions persisted in SQLite.

### Tests

- Unit: `AnthropicProvider` parses a mocked API response correctly
- Unit: `SystemPromptBuilder` includes agent name and current date
- Unit: `AgentRunner` creates correct Turn sequence (user → assistant)
- Feature: `agent:chat` command can be invoked and exits cleanly

---

## Phase 2 — First Two Channels: Telegram + WebChat

**Goal:** Receive and reply to messages from Telegram and a built-in web chat UI. The agent is now reachable from outside the terminal.

### Tasks

#### 2.1 Channel Manager

Implement the Laravel Manager pattern:

- `ChannelManager extends Manager`
- `createTelegramDriver()` → returns `TelegramDriver`
- `createWebchatDriver()` → returns `WebChatDriver`
- Register as singleton in `AppServiceProvider`

#### 2.2 Telegram Driver

Implement `ChannelDriver` for Telegram Bot API:

- **Webhook mode**: Register a Telegram webhook URL pointing to `POST /api/channel/telegram/webhook`
- `normalizeInbound()`: Extract sender_id, text, media from Telegram update payload
- `send()`: Call Telegram `sendMessage` API, handle Markdown formatting
- `chunk()`: Split messages at 4096 character limit
- `start()` / `stop()`: Register/deregister the webhook with Telegram API
- `status()`: Ping Telegram `getMe` endpoint

Route:

```php
Route::post('/channel/telegram/webhook', [TelegramWebhookController::class, 'handle']);
```

#### 2.3 WebChat Driver

A simple built-in web UI — no external service:

- **Frontend**: A single Blade view with a chat form (or Livewire component)
- `POST /webchat/send` → creates InboundMessage, fires `MessageReceived`
- Responses displayed by polling `GET /webchat/messages/{session}` or via Livewire
- `send()` is a no-op (response is pulled by the UI)
- No authentication required for now (local use)

#### 2.4 Event: MessageReceived + Listener Chain

Wire up the inbound pipeline from the architecture doc:

| Listener | Phase 2 Scope |
|---|---|
| `NormalizeMessage` | Already normalized by driver — pass through |
| `EnforceAllowlist` | Check `PairedSender::isAuthorized()` — reject unknowns if policy requires |
| `ResolveSession` | Find or create session for this sender + agent |
| `DebounceMessages` | Dispatch `ProcessInboundMessage` with configurable delay |

Listeners not yet needed: `DetectMentionGating` (no groups yet), `DetectCommand` (no commands yet), `StageMedia` (no media yet).

#### 2.5 Job: ProcessInboundMessage

- Receives InboundMessage, Agent, Session
- Calls `AgentRunner::run()` (synchronously within the job)
- After agent completes, dispatches `DeliverOutboundMessage`

#### 2.6 Job: DeliverOutboundMessage

- Receives the assistant's response text + target channel
- Calls `ChannelManager::driver($type)->send()`
- Has retry logic (`$tries = 3`, `$backoff = [2, 10, 30]`)

#### 2.7 Queue Worker Setup

- Configure `QUEUE_CONNECTION=database` in `.env`
- Ensure `jobs` table migration exists (from Phase 0)
- Document: `php artisan queue:work` must be running

#### 2.8 DM Pairing (Basic)

- `PairedSender` model with `isAuthorized()` already exists
- Default policy: `open` (accept everyone) for easy development
- `EnforceAllowlist` listener checks the policy and rejects/accepts
- Artisan command: `php artisan pairing:approve {channel} {code}`

### Deliverable

```bash
# Start the app
php artisan serve &
php artisan queue:work &

# Configure Telegram
# 1. Set TELEGRAM_BOT_TOKEN in .env
# 2. Expose with ngrok: ngrok http 8000
# 3. php artisan channel:start telegram

# Now message your Telegram bot — it replies via the AI agent
# Also visit http://localhost:8000/webchat for the web UI
```

### Tests

- Unit: `TelegramDriver` correctly normalizes a Telegram update fixture
- Unit: `TelegramDriver` chunks long messages at 4096
- Feature: POST to `/api/channel/telegram/webhook` with a valid update dispatches a job
- Feature: WebChat form submission creates a Turn and returns a response
- Feature: `EnforceAllowlist` blocks unauthorized senders when policy is `allowlist`

---

## Phase 3 — WebSocket Gateway & Real-Time UI

**Goal:** Laravel Reverb provides real-time streaming of agent responses. The WebChat UI shows typing indicators and streamed text. REST API for session/agent management.

### Tasks

#### 3.1 Install & Configure Laravel Reverb

```bash
php artisan install:broadcasting
```

- Configure Reverb in `.env` (host, port, app credentials)
- Define broadcasting channels in `routes/channels.php`:
  - `agent.{agentId}` — private, agent activity
  - `session.{agentId}.{sessionKey}` — private, session-specific
  - `gateway` — presence, connected clients

#### 3.2 Broadcast Events

| Event | Payload | Broadcast On |
|---|---|---|
| `AgentResponseChunk` | blockType, payload | `session.{agentId}.{sessionKey}` |
| `AgentTyping` | agentId, isTyping | `session.{agentId}.{sessionKey}` |
| `ChannelStatusChanged` | channelType, status | `gateway` |
| `SessionCreated` | session data | `agent.{agentId}` |

#### 3.3 Update AgentRunner for Streaming

- Modify `AgentRunner::run()` to broadcast `AgentResponseChunk` events as the LLM response is composed
- For non-streaming providers: broadcast the full text as one chunk
- For streaming (future): broadcast deltas as they arrive

#### 3.4 Upgrade WebChat UI

Replace the polling-based WebChat with a real-time version:

- Use Laravel Echo (JS client) to subscribe to the session channel
- Display `AgentResponseChunk` events as they arrive
- Show typing indicator when `AgentTyping` is received
- Support multiple sessions (session switcher sidebar)
- Use Blade + Alpine.js or Livewire for reactivity

#### 3.5 REST API (Session & Agent Management)

```
GET    /api/agents                  → list agents
GET    /api/agents/{id}             → get agent details
POST   /api/agents/{id}/sessions    → create new session
GET    /api/sessions/{id}           → get session with turns
POST   /api/sessions/{id}/message   → send message to session
GET    /api/channels                → list channel statuses
```

All protected by `ValidateGatewayToken` middleware.

#### 3.6 Health Endpoint

```
GET /healthz → { status: "ok", version: "...", uptime: "...", channels: {...} }
```

### Deliverable

```bash
php artisan serve &
php artisan queue:work &
php artisan reverb:start &

# Open http://localhost:8000/webchat
# Messages stream in real-time as the agent responds
```

### Tests

- Feature: Broadcasting an `AgentResponseChunk` reaches the correct channel
- Feature: REST API returns agent list, creates sessions, accepts messages
- Feature: `ValidateGatewayToken` middleware rejects requests without token
- Feature: Health endpoint returns 200

---

## Phase 4 — Tools & Sandboxed Execution

**Goal:** The agent can invoke tools (bash commands, cross-channel messaging). Tool policy enforcement and lifecycle events.

### Tasks

#### 4.1 Tool Contract + Registry + Executor

- `ToolRegistry` — register/resolve tools by name, filter by agent policy
- `ToolExecutor` — resolve tool, fire `ToolExecuting` event, execute, fire `ToolExecuted` event
- Register `ToolRegistry` as singleton in `AppServiceProvider`

#### 4.2 Built-in Tools

| Tool | Class | Description |
|---|---|---|
| `system.run` | `BashTool` | Sandboxed shell execution via `Process` facade |
| `messaging.send` | `MessagingTool` | Send message via any connected channel |

**BashTool:**

- Use `Process::path($sandboxDir)->timeout($timeout)->run($command)`
- Validate: reject `sudo`, `rm -rf /`, path traversal
- Limit output to configurable max (50KB default)
- Create sandbox directory per agent: `storage/app/sandbox/{agentId}/`

**MessagingTool:**

- Accept channel type + recipient + text
- Dispatch `DeliverOutboundMessage` job
- Return confirmation

#### 4.3 Update AgentRunner for Tool Loop

Modify the execution loop to handle tool calls:

1. Parse `tool_use` blocks from LLM response
2. For each tool call:
   - Create `Turn` (role: `ToolCall`)
   - Run through `ToolExecutor::execute()`
   - Create `Turn` (role: `ToolResult`)
   - Append both to history
3. Re-call LLM with updated history
4. Repeat until no more tool calls

#### 4.4 Update SystemPromptBuilder

- Inject tool JSON schemas into the system prompt
- Use `ToolRegistry::schemasFor($agent)` to get allowed tools
- Format as Anthropic-compatible tool definitions

#### 4.5 Tool Policy

In agent config (`config` JSON column):

```json
{
    "tools": {
        "deny": ["system.run"],
        "requireApproval": []
    }
}
```

- `ToolRegistry::schemasFor()` respects the deny list
- `ToolExecutor` checks approval gating (broadcasts approval request via WebSocket, waits — implement as a simple timeout for now)

#### 4.6 Events

| Event | Fires When |
|---|---|
| `ToolExecuting` | Before tool runs (listeners can log, block, gate) |
| `ToolExecuted` | After tool completes (listeners can log, post-process) |

#### 4.7 Artisan: `tool:list`

```bash
php artisan tool:list

 Name              Allowed  Approval Required
 system.run        yes      no
 messaging.send    yes      no
```

### Deliverable

```bash
php artisan agent:chat

You: List the files in the current directory
Assistant: I'll run the `ls` command for you.
[Tool: system.run → ls -la]
Here are the files:
  drwxr-xr-x  app/
  drwxr-xr-x  config/
  ...
```

### Tests

- Unit: `BashTool` rejects `sudo` commands
- Unit: `BashTool` limits output length
- Unit: `ToolRegistry` filters denied tools
- Unit: `ToolExecutor` fires `ToolExecuting` and `ToolExecuted` events
- Feature: Full tool loop — agent calls tool, gets result, continues

---

## Phase 5 — Memory & Vector Search

**Goal:** The agent can search past conversations semantically. Embedding pipeline runs in background.

### Tasks

#### 5.1 SQLite FTS5 Setup

- Create `embeddings_fts` virtual table (FTS5) in a migration
- Trigger-based sync: FTS5 `content` table linked to `embeddings.content`

#### 5.2 sqlite-vec Integration

Two options (choose based on environment):

**Option A — PHP FFI / SQLite extension:**

- Load `sqlite-vec` extension via `DB::statement("SELECT load_extension('vec0')")`
- Requires the `.so`/`.dylib` to be present

**Option B — Pure PHP fallback:**

- Store vectors as BLOB (packed float32 array)
- Compute cosine similarity in PHP for small datasets
- Upgrade to sqlite-vec when available

Start with Option B for portability, document Option A as the performance path.

#### 5.3 Embedding Provider: OpenAI

Implement `EmbeddingProvider` for OpenAI:

- `embed(string $text): array` — call `/v1/embeddings` endpoint
- `batchEmbed(array $texts): array` — batch in one API call
- Return float arrays

#### 5.4 VectorStore Service

- `insert()` — pack vector to BLOB, insert into embeddings table + FTS5
- `nearestNeighbors()` — query vectors by cosine similarity (sqlite-vec or PHP fallback)
- `delete()` — remove by turn_id

#### 5.5 HybridSearch Service

- BM25 query via FTS5 (`embeddings_fts MATCH ?`)
- Vector query via VectorStore
- Reciprocal Rank Fusion to merge results
- Return ranked list of Turn references

#### 5.6 EmbeddingService

- `embedNewTurns(Agent)` — find turns without embeddings, batch embed, store
- Deduplication via `content_hash`

#### 5.7 Job: BatchEmbeddings

- Queued job that runs `EmbeddingService::embedNewTurns()`
- Dispatched by a listener on `ToolExecuted` or `SessionClosed`
- Also dispatchable from scheduler (every 5 minutes)

#### 5.8 Memory Tool

New tool: `memory.search`

- Agent can invoke it to search past conversations
- Calls `HybridSearch::search()` with the query
- Returns matching conversation excerpts

Add to `ToolRegistry`.

#### 5.9 Artisan Commands

- `php artisan memory:search "query"` — search from CLI
- `php artisan memory:reindex` — re-embed all turns
- `php artisan memory:stats` — show embedding counts, index size

### Deliverable

```bash
# After several conversations:
php artisan memory:search "that recipe for pasta"

 Results:
 1. [Session: cooking-tips, Turn #42] "Here's a great recipe for pasta carbonara..."
 2. [Session: main, Turn #15] "You mentioned wanting a pasta recipe yesterday..."
```

The agent can also use `memory.search` during conversations to recall past context.

### Tests

- Unit: `VectorStore` stores and retrieves vectors correctly
- Unit: `HybridSearch` returns ranked results from both BM25 and vector
- Unit: `EmbeddingService` deduplicates by content_hash
- Feature: `memory:search` command returns relevant results
- Feature: Agent uses `memory.search` tool in conversation

---

## Phase 6 — Multi-Agent, Skills, Plugins

**Goal:** Support multiple agents with different personas and capabilities. Installable skills. Plugin architecture.

### Tasks

#### 6.1 Multi-Agent Support

- Seed multiple agents with different configs:
  - "Assistant" — general-purpose, all tools
  - "Coder" — system.run allowed, coding-focused persona
  - "Writer" — no tools, creative writing persona
- `AgentRunner` already works per-agent (receives `Session` which belongs to `Agent`)
- Update `ResolveSession` listener to route based on channel config → agent mapping
- Add `config/agents.php` with multiple agent definitions
- Artisan: `php artisan agent:list`

#### 6.2 Sub-Agent Spawning

New tool: `sessions.spawn`

- Creates a new Session under a different Agent
- Sends a message to the sub-agent
- Returns the sub-agent's response
- Useful for delegation: main agent spawns a coder sub-agent for technical tasks

#### 6.3 Skill System

**Skill structure:**

```
skills/{skill-name}/
├── skill.json       # { name, version, description, tool_class, prompt_file }
├── SomeTool.php     # implements Tool contract
└── prompt.md        # system prompt fragment
```

**SkillLoader service:**

- Scan `skills/` directory
- Load `skill.json` manifests
- Register tools in `ToolRegistry`
- Store prompt fragments in `Skill` model for `SystemPromptBuilder`

**Artisan commands:**

- `php artisan skill:list` — list available and installed skills
- `php artisan skill:install {name}` — activate a skill for an agent
- `php artisan skill:remove {name}` — deactivate

**Bundled skills (ship a few):**

- `web-search` — search the web (calls an external search API)
- `datetime` — current date/time utilities
- `calculator` — evaluate math expressions

#### 6.4 Update SystemPromptBuilder

- Query `$agent->skills` relationship
- Append each skill's `prompt_fragment` to the system prompt
- Include skill tool schemas alongside built-in tool schemas

#### 6.5 Plugin System

**PluginServiceProvider:**

- On boot, scan `plugins/` directory for `composer.json` with `extra.openclaw.provider`
- Auto-register each plugin's ServiceProvider
- Plugins can register:
  - Event listeners (hooks)
  - Additional tools (via `ToolRegistry`)
  - Additional routes
  - Config extensions

**PluginManager service:**

- `discover()` — find plugin manifests
- `activate()` / `deactivate()` — toggle plugins
- `php artisan plugin:list` — show installed plugins

### Deliverable

```bash
php artisan agent:list
 ID  Name        Model                        Skills           Active
 1   Assistant   claude-sonnet-4-5-20250929   web-search, calc  yes
 2   Coder       claude-sonnet-4-5-20250929   datetime          yes

php artisan skill:install web-search --agent=1
# Agent "Assistant" now has web search capability
```

### Tests

- Feature: Two agents produce different responses based on their persona
- Feature: Sub-agent spawning returns a response from the child agent
- Unit: `SkillLoader` discovers and registers skills from the filesystem
- Unit: `SystemPromptBuilder` includes skill prompt fragments
- Feature: Plugin auto-discovery registers a test plugin's ServiceProvider

---

## Phase 7 — Triggers, Cron, Webhooks

**Goal:** The agent can be triggered by scheduled jobs, external webhooks, and lifecycle events.

### Tasks

#### 7.1 Cron Jobs

- `CronJob` model already exists with `isDue()` logic
- Register scheduler in `routes/console.php`:
  - Every minute: query active cron jobs, dispatch `ExecuteCronAction` for due ones
- `ExecuteCronAction` job: calls `AgentRunner::run()` with the cron's action text
- `CronExecution` model records each run (status, output, timestamp)

**Artisan commands:**

- `php artisan cron:list` — show all cron jobs with next run time
- `php artisan cron:add {agent} {schedule} {action}` — create a cron job
- `php artisan cron:remove {id}` — delete a cron job
- `php artisan cron:run {id}` — manually trigger a cron job

#### 7.2 Webhooks

- `WebhookController` accepts `POST /api/hook/{webhook}`
- `ValidateWebhookToken` middleware checks per-webhook bearer token
- On valid request: dispatch `RunAgentSession` with the payload as the user message
- Return 202 Accepted immediately

**Artisan commands:**

- `php artisan webhook:create {agent}` — create a webhook, display URL + token
- `php artisan webhook:list` — list all webhooks
- `php artisan webhook:delete {id}` — remove a webhook

#### 7.3 Complete Event/Listener Wiring

Wire up all remaining events from the architecture:

| Event | Listeners |
|---|---|
| `SessionCreated` | Log, notify via WebSocket |
| `SessionClosed` | Trigger embedding job, cleanup |
| `CompactionStarting` | Metrics logging |
| `ConfigChanged` | Reload channel drivers, clear caches |
| `GatewayStarted` | Warm caches, verify channel connections |

#### 7.4 Artisan: `gateway:serve` (Orchestrator)

A single command that starts everything:

```bash
php artisan gateway:serve
# Starts: HTTP server, queue worker, scheduler, Reverb
# (spawns child processes or documents the multi-command setup)
```

### Deliverable

```bash
# Create a daily summary cron
php artisan cron:add 1 "0 9 * * *" "Give me a summary of today's agenda"

# Create a webhook for CI notifications
php artisan webhook:create 1
# → Webhook URL: http://localhost:8000/api/hook/abc123
# → Token: tok_xyz...

# Trigger it
curl -X POST http://localhost:8000/api/hook/abc123 \
  -H "Authorization: Bearer tok_xyz..." \
  -H "Content-Type: application/json" \
  -d '{"event": "build_failed", "repo": "my-app"}'
```

### Tests

- Unit: `CronJob::isDue()` respects schedule and cooldown
- Feature: Scheduler dispatches `ExecuteCronAction` for due jobs
- Feature: Webhook endpoint accepts valid token and dispatches agent job
- Feature: Webhook endpoint rejects invalid token with 401

---

## Phase 8 — Go Sidecars & FrankenPHP

**Goal:** Production-grade runtime with FrankenPHP worker mode and optional Go sidecars for LLM streaming and long-lived connections.

### Tasks

#### 8.1 FrankenPHP Setup

- Create `Dockerfile` based on `dunglas/frankenphp`
- Install PHP extensions: `pdo_sqlite`, `pcntl`, `intl`, `bcmath`
- Configure Caddy for HTTPS, worker mode, static files
- Create `docker-compose.yml` with services: app, queue-worker, scheduler, reverb
- Document `frankenphp php-server --worker public/index.php` for non-Docker usage

#### 8.2 Go Sidecar: LLM Streaming Proxy

```
go-sidecars/llm-stream/
├── main.go          # HTTP server on :9001
├── providers/       # Anthropic, OpenAI SSE clients
├── handler.go       # POST /v1/chat/stream → SSE response
└── go.mod
```

- Accepts JSON request (system prompt, messages, tools, provider config)
- Streams SSE events back to the caller
- Handles connection keep-alive better than PHP's synchronous HTTP

**Laravel integration:**

- `GoLLMStreamProxy implements LLMProvider`
- Uses `Http::withOptions(['stream' => true])` to consume SSE from Go sidecar
- Yields chunks back to `AgentRunner`
- Falls back to direct PHP HTTP call if sidecar is not running

#### 8.3 Go Sidecar: Channel Bridge (for Future Channels)

```
go-sidecars/channel-bridge/
├── main.go           # HTTP API on :9002
├── bridges/          # WhatsApp, Discord persistent WS clients
├── api.go            # POST /send, GET /status, POST /inbound callback
└── go.mod
```

- Maintains persistent WebSocket connections to platforms
- Calls back to Laravel `POST /api/channel-bridge/inbound` when messages arrive
- Laravel calls `POST :9002/send` for outbound delivery

Implement the scaffold now, actual WhatsApp/Discord bridges in Phase 9.

#### 8.4 Update ModelRouter for Sidecar Awareness

```php
public function resolve(Agent $agent): LLMProvider
{
    // Try Go streaming sidecar first
    if ($this->isSidecarRunning('llm-stream')) {
        return new GoLLMStreamProxy(config('sidecars.llm_stream'));
    }

    // Fallback to direct PHP provider
    return $this->directProvider($agent);
}
```

#### 8.5 Artisan: `doctor` (Extended)

Extend the doctor command to check:

- FrankenPHP worker mode active?
- Go sidecars reachable? (ping :9001, :9002)
- SQLite WAL mode enabled?
- Queue worker running?
- Reverb running?
- All configured channels connected?

#### 8.6 Deployment Documentation

Create `DEPLOYMENT.md` covering:

- Minimal (PHP only, no Docker)
- Docker Compose (recommended)
- FrankenPHP single binary
- Systemd service files
- Supervisor config for Go sidecars

### Deliverable

```bash
docker compose up -d
# → FrankenPHP + Reverb + Queue + Scheduler + Go sidecars all running
# → Automatic HTTPS via Caddy
# → LLM responses stream through Go proxy
```

### Tests

- Feature: App boots correctly under FrankenPHP worker mode
- Integration: Go LLM sidecar handles a streaming request and returns chunks
- Feature: `doctor` command reports status of all subsystems
- Feature: `GoLLMStreamProxy` falls back to PHP when sidecar is unavailable

---

## Phase 9 — Remaining Channels & Polish

**Goal:** Add Discord, Slack, and other channels. Command detection. Media handling. Context compaction. Full feature parity.

### Tasks

#### 9.1 Additional Channel Drivers

| Channel | Driver | Notes |
|---|---|---|
| **Discord** | `DiscordDriver` | Uses Go channel-bridge sidecar (persistent WS) |
| **Slack** | `SlackDriver` | Socket Mode via Slack Bolt PHP or webhook mode |
| **WhatsApp** | `WhatsAppDriver` | Via Go channel-bridge sidecar (Baileys/whatsmeow) |
| **Signal** | `SignalDriver` | Via signal-cli subprocess |

For each:
- Implement `ChannelDriver` contract
- Add webhook route if applicable
- Add Artisan `channel:auth {type}` command
- Test inbound normalization + outbound delivery

#### 9.2 Extend ChannelType Enum

Add all new channel types to `ChannelType` enum. Each case includes `maxLength()` method:

```php
enum ChannelType: string
{
    case Telegram = 'telegram';
    case WebChat  = 'webchat';
    case Discord  = 'discord';
    case Slack    = 'slack';
    case WhatsApp = 'whatsapp';
    case Signal   = 'signal';

    public function maxLength(): int
    {
        return match($this) {
            self::Telegram => 4096,
            self::Discord  => 2000,
            self::Slack    => 40000,
            self::WhatsApp => 65536,
            self::Signal   => 65536,
            self::WebChat  => PHP_INT_MAX,
        };
    }
}
```

#### 9.3 Command Detection

`DetectCommand` listener:

- Parse `/new` → create a new session
- Parse `/status` → return agent status
- Parse `/help` → return available commands
- Parse `/session {name}` → switch session
- Parse `/agent {name}` → switch agent
- If a command is detected, handle it directly and set `$event->rejected = true`

#### 9.4 Mention Gating

`DetectMentionGating` listener:

- In group chats, only process messages that @mention the bot
- Configurable per-channel: `channels.telegram.require_mention = true`
- Skip gating in DMs

#### 9.5 Media Handling

`StageMedia` listener:

- Download media files from inbound message (images, voice, documents)
- Stage to `storage/app/sandbox/{agentId}/media/`
- Include file paths in the Turn content for the agent to reference
- Clean up staged files after session ends

Outbound:

- `DeliverOutboundMessage` job uploads media to the platform if present
- Each driver handles platform-specific upload API

#### 9.6 Context Window Compaction

Upgrade `ContextCompactor`:

- Count tokens (approximate: chars / 4)
- When history exceeds model's context limit:
  1. Keep system prompt + last N turns
  2. Summarize older turns via a quick LLM call ("Summarize this conversation so far")
  3. Replace old turns with the summary as a single "system" message
- Fire `CompactionStarting` event before compacting

#### 9.7 Model Failover (Complete)

Upgrade `ModelRouter`:

- On 429: apply cooldown to current profile, try next provider
- On 401: deactivate profile, alert via log
- On context overflow: retry with compacted history
- On network error: retry same provider with backoff

Add second provider:

- `OpenAIProvider implements LLMProvider`
- Wire into `ProviderFactory`

#### 9.8 Config Hot-Reload

- `php artisan config:reload` — clears config cache, fires `ConfigChanged`
- `ConfigChanged` listener:
  - Restarts channel drivers that changed
  - Re-reads agent configs
  - Broadcasts status update via WebSocket

#### 9.9 Testing & Documentation

- Write a comprehensive test suite for all phases
- Write user-facing documentation:
  - Getting Started guide
  - Channel setup guides (Telegram, Discord, Slack)
  - Skill development guide
  - Plugin development guide
  - Deployment guide (already from Phase 8)

### Deliverable

Full feature parity with OpenClaw:

```bash
docker compose up -d

# Telegram, Discord, Slack, WebChat all connected
# Multiple agents with different personas
# Skills installed and active
# Cron jobs running
# Webhooks accepting external triggers
# Memory search across all conversations
# Tool execution with sandboxing
# Real-time streaming via WebSocket
# Go sidecars for performance
# FrankenPHP worker mode for production
```

---

## Dependency Graph

Which phases can overlap or run in parallel:

```
Phase 0 (Foundation)
  │
  ▼
Phase 1 (Core Agent Loop)
  │
  ├──────────────┬──────────────┐
  ▼              ▼              ▼
Phase 2        Phase 4        Phase 5
(Channels)     (Tools)        (Memory)
  │              │              │
  ▼              │              │
Phase 3        ◄─┘              │
(WebSocket)                     │
  │                             │
  ├─────────────────────────────┘
  ▼
Phase 6 (Multi-Agent, Skills, Plugins)
  │
  ├──────────────┐
  ▼              ▼
Phase 7        Phase 8
(Triggers)     (Go + FrankenPHP)
  │              │
  └──────┬───────┘
         ▼
       Phase 9
  (Channels & Polish)
```

Key insight: **Phases 2, 4, and 5 are independent** after Phase 1 and can be developed in parallel by different people or in any order. They all converge at Phase 6.

---

## Milestone Summary

| Milestone | What You Can Do | Phases |
|---|---|---|
| **M1: CLI Agent** | Chat with AI via terminal | 0 + 1 |
| **M2: Two Channels** | Reach the agent from Telegram and a web page | + 2 |
| **M3: Real-Time** | Streaming responses, REST API, WebSocket | + 3 |
| **M4: Tool Use** | Agent executes commands, sends messages | + 4 |
| **M5: Memory** | Agent recalls past conversations | + 5 |
| **M6: Multi-Agent** | Multiple personas, skills, plugins | + 6 |
| **M7: Triggers** | Cron, webhooks, external events | + 7 |
| **M8: Production** | FrankenPHP, Go sidecars, Docker | + 8 |
| **M9: Full Parity** | All channels, media, compaction, failover | + 9 |

---

## Quick Start (Phase 0 + 1 Checklist)

To get the first working version as fast as possible:

```bash
# 1. Create project
composer create-project laravel/laravel openclaw-laravel
cd openclaw-laravel

# 2. Configure SQLite
# Edit .env: DB_CONNECTION=sqlite
touch database/database.sqlite

# 3. Set API key
echo "ANTHROPIC_API_KEY=sk-ant-your-key" >> .env

# 4. Create migrations + models + enums + config
#    (follow Phase 0 tasks above)

# 5. Implement AnthropicProvider + AgentRunner + agent:chat command
#    (follow Phase 1 tasks above)

# 6. Run
php artisan migrate --seed
php artisan agent:chat
```

You now have a working AI assistant backed by SQLite and the Anthropic API, with a full session history, running entirely locally.

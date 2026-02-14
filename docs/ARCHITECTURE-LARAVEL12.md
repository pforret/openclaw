# OpenClaw on Laravel 12 — Implementation Architecture

This document translates the [language-independent OpenClaw architecture](./ARCHITECTURE.md) into a concrete implementation using **Laravel 12**, **SQLite**, and **FrankenPHP**, with optional **Go modules** for performance-critical subsystems.

---

## Technology Stack

| Layer | Technology |
|---|---|
| **Runtime** | FrankenPHP (PHP 8.4+ with built-in app server) |
| **Framework** | Laravel 12 |
| **Database** | SQLite (local, zero-config) |
| **Queue / Workers** | Laravel Queue with `database` driver (SQLite) |
| **WebSocket** | Laravel Reverb (native PHP WebSocket server on FrankenPHP) |
| **Scheduler** | Laravel `schedule:work` (no system cron needed) |
| **Cache** | SQLite or file-based (via Laravel Cache) |
| **Search** | Laravel Scout + SQLite FTS5 (keyword) + sqlite-vec (vector) |
| **Go Modules** | Optional sidecar binaries for: LLM streaming, browser automation, long-lived channel connections |
| **CLI** | Laravel Artisan commands |

### Why FrankenPHP?

FrankenPHP bundles Caddy + PHP into a single binary, providing:

- **Worker mode** — Keep the Laravel app booted between requests (no cold starts)
- **Built-in HTTPS** — Automatic TLS via Caddy
- **HTTP/2 + HTTP/3** — Modern protocol support out of the box
- **Early Hints (103)** — Faster perceived page loads
- **Single binary deployment** — No nginx/Apache/php-fpm configuration
- **Go interop** — FrankenPHP is written in Go, making it straightforward to embed Go modules as sidecar processes or CGO extensions that share the same deployment artifact

---

## Project Structure

```
openclaw-laravel/
├── app/
│   ├── Models/                    # Eloquent models (SQLite)
│   │   ├── Agent.php
│   │   ├── Session.php
│   │   ├── Turn.php
│   │   ├── Channel.php
│   │   ├── AuthProfile.php
│   │   ├── CronJob.php
│   │   ├── CronExecution.php
│   │   ├── Webhook.php
│   │   ├── Embedding.php
│   │   ├── Skill.php
│   │   ├── Plugin.php
│   │   └── PairedSender.php
│   │
│   ├── Enums/                     # PHP 8.4 enums
│   │   ├── ChannelType.php        # whatsapp, telegram, discord, slack...
│   │   ├── TurnRole.php           # user, assistant, tool_call, tool_result
│   │   ├── DeliveryMode.php       # async, best_effort, strict
│   │   ├── PairingPolicy.php      # pairing, allowlist, open
│   │   └── ToolPolicy.php         # allow, deny, approval_required
│   │
│   ├── Services/                  # Core business logic
│   │   ├── AgentRuntime/
│   │   │   ├── AgentRunner.php            # Main execution loop
│   │   │   ├── SystemPromptBuilder.php    # Prompt composition
│   │   │   ├── ContextCompactor.php       # Context window management
│   │   │   └── ToolExecutor.php           # Tool dispatch & sandboxing
│   │   │
│   │   ├── Channels/
│   │   │   ├── ChannelManager.php         # Channel lifecycle orchestration
│   │   │   ├── InboundPipeline.php        # Normalize → allowlist → route
│   │   │   ├── OutboundDispatcher.php     # Chunk → format → deliver
│   │   │   └── Drivers/                   # Per-platform drivers
│   │   │       ├── TelegramDriver.php
│   │   │       ├── WhatsAppDriver.php
│   │   │       ├── DiscordDriver.php
│   │   │       ├── SlackDriver.php
│   │   │       └── WebChatDriver.php
│   │   │
│   │   ├── LLM/
│   │   │   ├── ModelRouter.php            # Provider selection & failover
│   │   │   ├── ProviderFactory.php        # Instantiate provider clients
│   │   │   └── Providers/
│   │   │       ├── AnthropicProvider.php
│   │   │       ├── OpenAIProvider.php
│   │   │       ├── GeminiProvider.php
│   │   │       └── OllamaProvider.php
│   │   │
│   │   ├── Memory/
│   │   │   ├── EmbeddingService.php       # Batch embedding orchestration
│   │   │   ├── VectorStore.php            # SQLite-vec read/write
│   │   │   └── HybridSearch.php           # BM25 + vector fusion
│   │   │
│   │   ├── Tools/
│   │   │   ├── ToolRegistry.php           # All available tools
│   │   │   ├── BashTool.php               # Sandboxed shell execution
│   │   │   ├── BrowserTool.php            # Headless browser (Go sidecar)
│   │   │   ├── MessagingTool.php          # Cross-channel messaging
│   │   │   └── CanvasTool.php             # Canvas rendering
│   │   │
│   │   └── Plugins/
│   │       ├── PluginManager.php          # Discovery, loading, lifecycle
│   │       └── SkillLoader.php            # Skill resolution & prompt merge
│   │
│   ├── Contracts/                 # Interfaces (dependency inversion)
│   │   ├── ChannelDriver.php              # Channel plugin contract
│   │   ├── LLMProvider.php                # Model provider contract
│   │   ├── EmbeddingProvider.php          # Embedding provider contract
│   │   ├── Tool.php                       # Tool contract
│   │   └── Plugin.php                     # Plugin contract
│   │
│   ├── Events/                    # Laravel Events (= OpenClaw hooks)
│   │   ├── MessageReceived.php
│   │   ├── MessageSending.php
│   │   ├── SessionCreated.php
│   │   ├── SessionClosed.php
│   │   ├── ToolExecuting.php
│   │   ├── ToolExecuted.php
│   │   ├── CompactionStarting.php
│   │   ├── ConfigChanged.php
│   │   └── GatewayStarted.php
│   │
│   ├── Listeners/                 # Event subscribers
│   │   ├── LogMessageActivity.php
│   │   ├── EnforceAllowlist.php
│   │   ├── DetectMentionGating.php
│   │   ├── StageMedia.php
│   │   ├── DebounceMessages.php
│   │   └── EmbedConversation.php
│   │
│   ├── Jobs/                      # Queued jobs (async work)
│   │   ├── ProcessInboundMessage.php
│   │   ├── RunAgentSession.php
│   │   ├── DeliverOutboundMessage.php
│   │   ├── BatchEmbeddings.php
│   │   ├── ExecuteCronAction.php
│   │   └── SyncChannelConnection.php
│   │
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── WebhookController.php      # POST /hook/{id}
│   │   │   ├── CronCallbackController.php # POST /cron/{id}
│   │   │   ├── HealthController.php       # GET /healthz
│   │   │   ├── Api/                       # REST API (optional)
│   │   │   │   ├── AgentController.php
│   │   │   │   ├── SessionController.php
│   │   │   │   ├── ConfigController.php
│   │   │   │   └── ChannelController.php
│   │   │   └── WebChatController.php      # Built-in web UI
│   │   │
│   │   └── Middleware/
│   │       ├── ValidateWebhookToken.php
│   │       └── ValidateGatewayToken.php
│   │
│   ├── Broadcasting/              # WebSocket channels (Laravel Reverb)
│   │   ├── AgentChannel.php               # Private: agent.{id}
│   │   ├── SessionChannel.php             # Private: session.{agentId}.{key}
│   │   └── GatewayChannel.php             # Presence: gateway
│   │
│   ├── Console/
│   │   └── Commands/              # Artisan CLI commands
│   │       ├── GatewayServe.php           # php artisan gateway:serve
│   │       ├── AgentChat.php              # php artisan agent:chat
│   │       ├── ChannelStart.php           # php artisan channel:start {type}
│   │       ├── ChannelStop.php            # php artisan channel:stop {type}
│   │       ├── SkillInstall.php           # php artisan skill:install {name}
│   │       ├── MemorySearch.php           # php artisan memory:search {query}
│   │       ├── MemoryReindex.php          # php artisan memory:reindex
│   │       ├── ConfigReload.php           # php artisan config:reload
│   │       └── Doctor.php                 # php artisan doctor
│   │
│   └── Providers/
│       ├── AppServiceProvider.php         # Core bindings
│       ├── EventServiceProvider.php       # Event → Listener mapping
│       └── PluginServiceProvider.php      # Plugin auto-discovery
│
├── config/
│   ├── openclaw.php               # Main config (= openclaw.json)
│   ├── agents.php                 # Agent definitions
│   ├── channels.php               # Channel credentials & settings
│   ├── models.php                 # LLM provider config
│   └── tools.php                  # Tool permissions
│
├── database/
│   ├── database.sqlite            # Single SQLite file
│   └── migrations/
│       ├── create_agents_table.php
│       ├── create_sessions_table.php
│       ├── create_turns_table.php
│       ├── create_channels_table.php
│       ├── create_auth_profiles_table.php
│       ├── create_cron_jobs_table.php
│       ├── create_cron_executions_table.php
│       ├── create_webhooks_table.php
│       ├── create_embeddings_table.php
│       ├── create_skills_table.php
│       ├── create_plugins_table.php
│       ├── create_paired_senders_table.php
│       └── create_jobs_table.php          # Laravel queue
│
├── go-sidecars/                   # Optional Go modules
│   ├── llm-stream/                # SSE/streaming proxy for LLM APIs
│   ├── browser-server/            # Headless browser automation (Rod/Chromedp)
│   └── channel-bridge/            # Long-lived WS connections (WhatsApp, Discord)
│
├── plugins/                       # Installed plugin packages
├── skills/                        # Bundled skill definitions
├── resources/views/               # WebChat Blade/Livewire UI
├── routes/
│   ├── web.php                    # Web UI routes
│   ├── api.php                    # REST API routes
│   ├── channels.php               # Broadcasting auth
│   └── console.php                # Artisan schedule definitions
│
├── .env                           # Environment secrets
├── Dockerfile                     # FrankenPHP-based image
└── docker-compose.yml
```

---

## Concept Mapping: OpenClaw → Laravel

| OpenClaw Concept | Laravel Equivalent | Notes |
|---|---|---|
| **Gateway** (WebSocket server) | **Laravel Reverb** | Native PHP WebSocket server, runs on FrankenPHP worker mode |
| **JSON-RPC protocol** | **Laravel Broadcasting** (events over WebSocket) | Private/presence channels with typed events |
| **Channel Plugin** | **Contract + Driver** (`ChannelDriver` interface) | Like Mail/Notification drivers — swap implementations |
| **Inbound pipeline** | **Event + Listener chain** | `MessageReceived` event → ordered listeners |
| **Outbound delivery** | **Queued Job** (`DeliverOutboundMessage`) | Async, retryable, per-channel formatting |
| **Agent Runtime** | **Service class** (`AgentRunner`) | Bound in container, injected where needed |
| **Session (conversation)** | **Eloquent Model** (`Session` + `Turn`) | Relational: `sessions` has many `turns` |
| **Append-only log** | **Immutable `Turn` inserts** | Turns are insert-only, never updated |
| **System prompt** | **Builder pattern** (`SystemPromptBuilder`) | Composes fragments from config, skills, tools |
| **Tool** | **Contract + class** (`Tool` interface) | Registry discovers tools, executor dispatches |
| **Tool policy** | **Config + Gate** | `config/tools.php` + Laravel `Gate::allows('use-tool', $tool)` |
| **Memory / embeddings** | **Service + SQLite-vec** | `EmbeddingService` + raw SQLite `sqlite-vec` extension |
| **Hybrid search** | **Scout + FTS5 + vectors** | Laravel Scout driver for FTS5, custom vector query |
| **Cron jobs** | **Laravel Scheduler** (`schedule:work`) | Define in `routes/console.php`, stored in DB |
| **Webhooks** | **Controller + Middleware** | `WebhookController` with `ValidateWebhookToken` |
| **Hooks (lifecycle events)** | **Laravel Events / Observers** | `ToolExecuting`, `SessionCreated`, etc. |
| **Plugin system** | **Laravel Package + ServiceProvider** | Auto-discovered via `PluginServiceProvider` |
| **Skill** | **Artisan-installable package** | Config + prompt fragment + tool class |
| **Config hot-reload** | **Artisan command + event** | `config:reload` clears cache, fires `ConfigChanged` |
| **Auth profiles** | **Eloquent Model** (`AuthProfile`) | Per-provider credentials with cooldown tracking |
| **Model failover** | **Service** (`ModelRouter`) | Try primary → catch → rotate → retry |
| **DM pairing** | **Eloquent Model + Middleware** | `PairedSender` model, `EnforceAllowlist` listener |
| **Secret management** | **Laravel `.env` + `Crypt`** | `.env` for keys, `Crypt::encrypt()` for stored tokens |
| **Multi-agent** | **Polymorphic config** | Each `Agent` model has own settings, sessions, skills |
| **Go sidecar** | **Separate process** (managed by Supervisor/systemd) | Communicates via HTTP/gRPC/Unix socket to Laravel |

---

## 1. Gateway → Laravel Reverb + FrankenPHP

The OpenClaw gateway becomes **Laravel Reverb** running inside **FrankenPHP worker mode**.

### How It Works

```
FrankenPHP (single binary)
├── Caddy (HTTPS, reverse proxy)
├── PHP Worker (Laravel app, kept warm between requests)
├── Reverb (WebSocket server on :8080)
└── Queue Worker (database driver, same process)
```

### Broadcasting Channels

```php
// routes/channels.php

// Agent activity stream (streaming responses, tool calls)
Broadcast::channel('agent.{agentId}', function ($user, $agentId) {
    return $user->canAccessAgent($agentId);
});

// Session-specific updates (typing, presence)
Broadcast::channel('session.{agentId}.{sessionKey}', function ($user, $agentId, $sessionKey) {
    return $user->canAccessSession($agentId, $sessionKey);
});

// Gateway-wide presence (connected clients, channel status)
Broadcast::channel('gateway', function ($user) {
    return $user->isGatewayAuthorized();
});
```

### Events Broadcast Over WebSocket

```php
// Replaces JSON-RPC methods like `chat.message`, `presence.update`

class AgentResponseChunk implements ShouldBroadcast
{
    public function __construct(
        public string $agentId,
        public string $sessionKey,
        public string $blockType,  // text_delta, tool_use, tool_result
        public mixed  $payload,
    ) {}

    public function broadcastOn(): Channel
    {
        return new PrivateChannel("session.{$this->agentId}.{$this->sessionKey}");
    }
}
```

### HTTP Routes (Webhook Ingestion)

```php
// routes/api.php

Route::post('/hook/{webhook}', [WebhookController::class, 'handle'])
    ->middleware(ValidateWebhookToken::class);

Route::post('/cron/{cronJob}', [CronCallbackController::class, 'handle'])
    ->middleware(ValidateGatewayToken::class);

Route::get('/healthz', [HealthController::class, 'check']);
```

---

## 2. Channels → Drivers Implementing a Contract

Each messaging platform is a **driver** implementing the `ChannelDriver` contract, following Laravel's driver pattern (like Mail, Queue, or Filesystem drivers).

### Contract

```php
// app/Contracts/ChannelDriver.php

interface ChannelDriver
{
    /** Establish the connection and begin listening for messages. */
    public function start(Channel $channel): void;

    /** Gracefully close the connection. */
    public function stop(Channel $channel): void;

    /** Normalize a platform-specific payload into a unified Message. */
    public function normalizeInbound(array $payload): InboundMessage;

    /** Deliver a message to the external platform. */
    public function send(OutboundMessage $message): void;

    /** Return connection health and metadata. */
    public function status(Channel $channel): ChannelStatus;
}
```

### Driver Registration (Manager Pattern)

```php
// app/Services/Channels/ChannelManager.php

class ChannelManager extends Manager
{
    public function createTelegramDriver(): ChannelDriver
    {
        return new TelegramDriver(config('channels.telegram'));
    }

    public function createSlackDriver(): ChannelDriver
    {
        return new SlackDriver(config('channels.slack'));
    }

    public function createDiscordDriver(): ChannelDriver
    {
        return new DiscordDriver(config('channels.discord'));
    }

    // ... one method per channel type

    public function getDefaultDriver(): string
    {
        return config('channels.default', 'telegram');
    }
}
```

Registered in the service container:

```php
// AppServiceProvider.php
$this->app->singleton(ChannelManager::class);
```

### Inbound Pipeline (Event + Listener Chain)

The OpenClaw 7-step inbound pipeline maps to a **Laravel event with ordered listeners**:

```php
// app/Events/MessageReceived.php

class MessageReceived
{
    public function __construct(
        public InboundMessage $message,
        public ChannelType    $channelType,
        public ?Agent         $resolvedAgent = null,
        public ?Session       $resolvedSession = null,
        public bool           $rejected = false,
    ) {}
}
```

```php
// app/Providers/EventServiceProvider.php

protected $listen = [
    MessageReceived::class => [
        NormalizeMessage::class,       // 1. Normalize
        EnforceAllowlist::class,       // 2. Allowlist check (may set rejected=true)
        DetectMentionGating::class,    // 3. Mention gating for groups
        DetectCommand::class,          // 4. Command detection (/new, /help)
        StageMedia::class,             // 5. Download & stage media
        ResolveSession::class,         // 6. Determine agent + session
        DebounceMessages::class,       // 7. Aggregate rapid messages
    ],
];
```

If no listener rejects the message, a queued job processes it:

```php
// Inside DebounceMessages listener (final step)
ProcessInboundMessage::dispatch($event->message, $event->resolvedAgent, $event->resolvedSession)
    ->delay(now()->addMilliseconds(config('openclaw.debounce_ms', 800)));
```

### Outbound Delivery

```php
// app/Jobs/DeliverOutboundMessage.php

class DeliverOutboundMessage implements ShouldQueue
{
    use Queueable;

    public int $tries = 3;
    public array $backoff = [2, 10, 30];

    public function handle(ChannelManager $channels): void
    {
        $driver = $channels->driver($this->message->channelType->value);

        // Chunk per channel limits
        $chunks = $driver->chunk($this->message->text, $this->message->channelType->maxLength());

        foreach ($chunks as $chunk) {
            $driver->send(new OutboundMessage(
                target: $this->message->target,
                text:   $chunk,
                media:  $this->message->media,
            ));
        }
    }
}
```

### Go Sidecar for Long-Lived Connections

Some channels (WhatsApp, Discord) require persistent WebSocket connections that outlive a PHP request cycle. These run as **Go sidecar** processes:

```
go-sidecars/channel-bridge/
├── main.go              # Entrypoint
├── whatsapp/bridge.go   # WhatsApp multi-device protocol
├── discord/bridge.go    # Discord gateway WebSocket
└── api.go               # HTTP API for Laravel to call
```

Communication with Laravel:

```
Laravel ←── HTTP/Unix Socket ──→ Go Channel Bridge
         POST /send              (outbound)
         GET  /status            (health)
         ←── POST /inbound ───→  (webhook back to Laravel)
```

The Go bridge calls back into `POST /api/channel-bridge/inbound` when a message arrives, and Laravel handles it from there.

---

## 3. Agent Runtime → Service + Queued Job

### Eloquent Models

```php
// app/Models/Agent.php

class Agent extends Model
{
    protected $casts = [
        'config'      => 'array',   // persona, model, tool policy
        'skill_ids'   => 'array',
        'is_active'   => 'boolean',
    ];

    public function sessions(): HasMany   { return $this->hasMany(Session::class); }
    public function skills(): BelongsToMany { return $this->belongsToMany(Skill::class); }
}
```

```php
// app/Models/Session.php

class Session extends Model
{
    public function agent(): BelongsTo { return $this->belongsTo(Agent::class); }
    public function turns(): HasMany   { return $this->hasMany(Turn::class)->orderBy('id'); }
}
```

```php
// app/Models/Turn.php (append-only)

class Turn extends Model
{
    public $timestamps = false; // use a single `created_at` column

    protected $casts = [
        'role'       => TurnRole::class,      // user, assistant, tool_call, tool_result
        'content'    => 'array',
        'created_at' => 'datetime',
    ];

    // Turns are never updated — insert only
    public static function booted(): void
    {
        static::updating(fn () => throw new \LogicException('Turns are immutable.'));
    }
}
```

### Execution Loop (AgentRunner Service)

```php
// app/Services/AgentRuntime/AgentRunner.php

class AgentRunner
{
    public function __construct(
        private SystemPromptBuilder $promptBuilder,
        private ModelRouter         $modelRouter,
        private ToolExecutor        $toolExecutor,
        private ContextCompactor    $compactor,
    ) {}

    public function run(Session $session, string $userMessage): void
    {
        // 1. Append user turn
        $session->turns()->create([
            'role'    => TurnRole::User,
            'content' => ['text' => $userMessage],
        ]);

        // 2. Build system prompt
        $systemPrompt = $this->promptBuilder->build($session->agent);

        // 3. Compile message history (with compaction if needed)
        $history = $this->compactor->prepare($session);

        // 4. Call LLM (with tool loop)
        $provider = $this->modelRouter->resolve($session->agent);

        do {
            $response = $provider->chat($systemPrompt, $history);

            if ($response->hasToolCalls()) {
                foreach ($response->toolCalls as $toolCall) {
                    // Append tool_call turn
                    $session->turns()->create([
                        'role'    => TurnRole::ToolCall,
                        'content' => $toolCall->toArray(),
                    ]);

                    // Execute & append result
                    $result = $this->toolExecutor->execute($toolCall, $session->agent);

                    $session->turns()->create([
                        'role'    => TurnRole::ToolResult,
                        'content' => $result->toArray(),
                    ]);

                    $history[] = $toolCall;
                    $history[] = $result;
                }
            }

            // Broadcast streaming chunks
            broadcast(new AgentResponseChunk(
                agentId:    $session->agent_id,
                sessionKey: $session->key,
                blockType:  'text_delta',
                payload:    $response->text,
            ));

        } while ($response->hasToolCalls());

        // 5. Append final assistant turn
        $session->turns()->create([
            'role'    => TurnRole::Assistant,
            'content' => ['text' => $response->text],
        ]);
    }
}
```

### Running Asynchronously via Queue

```php
// app/Jobs/RunAgentSession.php

class RunAgentSession implements ShouldQueue
{
    use Queueable;

    public int $timeout = 300; // 5 minutes max

    public function __construct(
        public Session $session,
        public string  $userMessage,
    ) {}

    public function handle(AgentRunner $runner): void
    {
        $runner->run($this->session, $this->userMessage);
    }
}
```

---

## 4. Tools → Contract + Registry

### Tool Contract

```php
// app/Contracts/Tool.php

interface Tool
{
    /** Unique tool name (e.g. "system.run", "browser.action"). */
    public function name(): string;

    /** JSON Schema describing the tool's parameters. */
    public function schema(): array;

    /** Execute the tool and return the result. */
    public function execute(array $params, Agent $agent): ToolResult;
}
```

### Tool Registry

```php
// app/Services/Tools/ToolRegistry.php

class ToolRegistry
{
    /** @var array<string, Tool> */
    private array $tools = [];

    public function register(Tool $tool): void
    {
        $this->tools[$tool->name()] = $tool;
    }

    public function resolve(string $name): Tool
    {
        return $this->tools[$name] ?? throw new ToolNotFoundException($name);
    }

    /** Return schemas for all tools the given agent is allowed to use. */
    public function schemasFor(Agent $agent): array
    {
        return collect($this->tools)
            ->filter(fn (Tool $t) => $this->isAllowed($t, $agent))
            ->map(fn (Tool $t) => ['name' => $t->name(), ...$t->schema()])
            ->values()
            ->all();
    }

    private function isAllowed(Tool $tool, Agent $agent): bool
    {
        $config = $agent->config['tools'] ?? [];
        if (in_array($tool->name(), $config['deny'] ?? [])) return false;
        return true;
    }
}
```

### Tool Executor (with Policy + Events)

```php
// app/Services/AgentRuntime/ToolExecutor.php

class ToolExecutor
{
    public function __construct(
        private ToolRegistry $registry,
    ) {}

    public function execute(ToolCall $call, Agent $agent): ToolResult
    {
        $tool = $this->registry->resolve($call->name);

        // Fire "before" event (= OpenClaw beforeToolCall hook)
        event(new ToolExecuting($tool, $call, $agent));

        // Check approval gating
        $requiresApproval = in_array(
            $call->name,
            $agent->config['tools']['requireApproval'] ?? []
        );

        if ($requiresApproval) {
            // Broadcast approval request, wait for response
            // (handled via WebSocket round-trip)
        }

        $result = $tool->execute($call->params, $agent);

        // Fire "after" event (= OpenClaw afterToolCall hook)
        event(new ToolExecuted($tool, $call, $result, $agent));

        return $result;
    }
}
```

### Example: Bash Tool with Sandboxing

```php
// app/Services/Tools/BashTool.php

class BashTool implements Tool
{
    public function name(): string { return 'system.run'; }

    public function schema(): array
    {
        return [
            'type' => 'object',
            'properties' => [
                'command' => ['type' => 'string', 'description' => 'Shell command to execute'],
            ],
            'required' => ['command'],
        ];
    }

    public function execute(array $params, Agent $agent): ToolResult
    {
        $command = $params['command'];
        $timeout = config('tools.bash.timeout', 30);
        $workdir = storage_path("app/sandbox/{$agent->id}");

        // Validate: no path traversal, no sudo
        $this->validate($command);

        $process = Process::path($workdir)
            ->timeout($timeout)
            ->run($command);

        return new ToolResult(
            output: Str::limit($process->output(), config('tools.bash.max_output', 50000)),
            exitCode: $process->exitCode(),
        );
    }
}
```

---

## 5. Memory → SQLite FTS5 + sqlite-vec

### Database Schema

```sql
-- migration: create_embeddings_table.php

CREATE TABLE embeddings (
    id          INTEGER PRIMARY KEY,
    agent_id    INTEGER NOT NULL REFERENCES agents(id),
    session_id  INTEGER REFERENCES sessions(id),
    turn_id     INTEGER REFERENCES turns(id),
    content     TEXT NOT NULL,
    vector      BLOB,             -- float32[] via sqlite-vec
    created_at  DATETIME NOT NULL,
    content_hash TEXT NOT NULL     -- for deduplication
);

-- FTS5 virtual table for keyword search (BM25)
CREATE VIRTUAL TABLE embeddings_fts USING fts5(
    content,
    content='embeddings',
    content_rowid='id'
);

-- sqlite-vec virtual table for vector search
-- (loaded via PHP SQLite extension or Go sidecar)
```

### Embedding Service

```php
// app/Services/Memory/EmbeddingService.php

class EmbeddingService
{
    public function __construct(
        private EmbeddingProvider $provider,  // OpenAI, Gemini, Ollama...
        private VectorStore       $store,
    ) {}

    /** Embed new turns that haven't been processed yet. */
    public function embedNewTurns(Agent $agent): void
    {
        $unembedded = Turn::whereDoesntHave('embedding')
            ->whereIn('session_id', $agent->sessions()->pluck('id'))
            ->whereIn('role', [TurnRole::User, TurnRole::Assistant])
            ->limit(100)
            ->get();

        if ($unembedded->isEmpty()) return;

        $texts  = $unembedded->pluck('content.text')->all();
        $vectors = $this->provider->batchEmbed($texts);

        foreach ($unembedded as $i => $turn) {
            $this->store->insert(
                agentId:   $agent->id,
                sessionId: $turn->session_id,
                turnId:    $turn->id,
                content:   $texts[$i],
                vector:    $vectors[$i],
            );
        }
    }
}
```

### Hybrid Search

```php
// app/Services/Memory/HybridSearch.php

class HybridSearch
{
    public function __construct(
        private EmbeddingProvider $provider,
        private VectorStore       $store,
    ) {}

    /**
     * Run BM25 keyword + vector similarity, fuse results with RRF.
     */
    public function search(string $query, Agent $agent, int $limit = 20): Collection
    {
        // 1. BM25 keyword results via FTS5
        $keywordResults = DB::select("
            SELECT rowid, rank
            FROM embeddings_fts
            WHERE embeddings_fts MATCH ?
            ORDER BY rank
            LIMIT ?
        ", [$query, $limit]);

        // 2. Vector similarity via sqlite-vec
        $queryVector   = $this->provider->embed($query);
        $vectorResults = $this->store->nearestNeighbors($queryVector, $agent->id, $limit);

        // 3. Reciprocal Rank Fusion
        return $this->reciprocalRankFusion($keywordResults, $vectorResults, $limit);
    }
}
```

### Embedding Provider Contract

```php
// app/Contracts/EmbeddingProvider.php

interface EmbeddingProvider
{
    /** Embed a single text string. */
    public function embed(string $text): array;   // float[]

    /** Embed a batch of texts efficiently. */
    public function batchEmbed(array $texts): array;  // float[][]
}
```

Implementations for OpenAI, Gemini, Ollama each make the appropriate API call and return float arrays.

---

## 6. Triggers & Scheduling → Laravel Scheduler + Events

### Cron Jobs (Laravel Scheduler)

```php
// routes/console.php

use App\Models\CronJob;

Schedule::call(function () {
    CronJob::where('is_active', true)->each(function (CronJob $job) {
        if ($job->isDue()) {
            ExecuteCronAction::dispatch($job);
            $job->recordExecution();
        }
    });
})->everyMinute();
```

```php
// app/Models/CronJob.php

class CronJob extends Model
{
    protected $casts = [
        'schedule'      => 'string',       // cron expression
        'delivery_mode' => DeliveryMode::class,
        'is_active'     => 'boolean',
        'last_run_at'   => 'datetime',
        'cooldown_minutes' => 'integer',
    ];

    public function agent(): BelongsTo { return $this->belongsTo(Agent::class); }
    public function executions(): HasMany { return $this->hasMany(CronExecution::class); }

    public function isDue(): bool
    {
        $cron = new \Cron\CronExpression($this->schedule);

        if (!$cron->isDue()) return false;

        // Enforce cooldown
        if ($this->last_run_at && $this->last_run_at->diffInMinutes(now()) < $this->cooldown_minutes) {
            return false;
        }

        return true;
    }
}
```

### Webhooks

```php
// app/Http/Controllers/WebhookController.php

class WebhookController extends Controller
{
    public function handle(Webhook $webhook, Request $request, AgentRunner $runner): JsonResponse
    {
        $session = $webhook->agent->sessions()
            ->firstOrCreate(['key' => $webhook->session_key ?? 'webhook']);

        RunAgentSession::dispatch(
            $session,
            json_encode($request->all()),
        );

        return response()->json(['status' => 'accepted'], 202);
    }
}
```

### Hook System → Laravel Events + Observers

The OpenClaw hook system maps directly to **Laravel Events**:

| OpenClaw Hook | Laravel Event | Typical Listener |
|---|---|---|
| `beforeToolCall` | `ToolExecuting` | Log, validate, gate approval |
| `afterToolCall` | `ToolExecuted` | Log, post-process results |
| `onMessage` | `MessageReceived` | Allowlist, routing, debounce |
| `onSession` | `SessionCreated` / `SessionClosed` | Analytics, cleanup |
| `onCompaction` | `CompactionStarting` | Backup, metrics |
| `onGatewayStart` | `GatewayStarted` | Warm caches, verify channels |
| `configApply` | `ConfigChanged` | Reload drivers, notify clients |

Plugins register their listeners in their own `ServiceProvider`:

```php
// plugins/my-plugin/src/MyPluginServiceProvider.php

class MyPluginServiceProvider extends ServiceProvider
{
    protected $listen = [
        ToolExecuting::class => [MyCustomToolGuard::class],
        MessageReceived::class => [MyCustomFilter::class],
    ];
}
```

---

## 7. Plugin & Extension System → Laravel Packages + ServiceProviders

### Plugin Contract

```php
// app/Contracts/Plugin.php

interface Plugin
{
    /** Plugin name and version. */
    public function manifest(): PluginManifest;

    /** Called when the plugin is activated. Register tools, listeners, routes. */
    public function boot(): void;

    /** Called when configuration is reloaded. */
    public function onConfigReload(array $config): void;
}
```

### Plugin Discovery & Loading

```php
// app/Services/Plugins/PluginManager.php

class PluginManager
{
    /** Discover plugins from the plugins/ directory. */
    public function discover(): Collection
    {
        return collect(File::directories(base_path('plugins')))
            ->map(fn ($dir) => $this->loadManifest($dir))
            ->filter();
    }

    /** Register a plugin's service provider. */
    public function activate(PluginManifest $manifest): void
    {
        app()->register($manifest->serviceProvider);
        Plugin::where('name', $manifest->name)
            ->update(['is_active' => true]);
    }
}
```

### Skill Structure

```
skills/web-search/
├── skill.json            # Metadata (name, version, description)
├── WebSearchTool.php     # Implements Tool contract
└── prompt.md             # System prompt fragment
```

```json
// skills/web-search/skill.json
{
    "name": "web-search",
    "version": "1.0.0",
    "description": "Search the web using Brave Search API",
    "tool_class": "WebSearchTool",
    "prompt_file": "prompt.md"
}
```

### Skill Loader

```php
// app/Services/Plugins/SkillLoader.php

class SkillLoader
{
    public function __construct(private ToolRegistry $tools) {}

    public function load(string $skillPath): void
    {
        $manifest = json_decode(file_get_contents("{$skillPath}/skill.json"), true);

        // Register tool
        require_once "{$skillPath}/{$manifest['tool_class']}.php";
        $toolClass = $manifest['tool_class'];
        $this->tools->register(new $toolClass());

        // Store prompt fragment for SystemPromptBuilder
        Skill::updateOrCreate(
            ['name' => $manifest['name']],
            [
                'prompt_fragment' => file_get_contents("{$skillPath}/{$manifest['prompt_file']}"),
                'version' => $manifest['version'],
            ]
        );
    }
}
```

---

## 8. Configuration → Laravel Config + .env

### Config Files

```php
// config/openclaw.php

return [
    'gateway' => [
        'port'  => env('OPENCLAW_GATEWAY_PORT', 8080),
        'token' => env('OPENCLAW_GATEWAY_TOKEN'),
    ],

    'debounce_ms' => env('OPENCLAW_DEBOUNCE_MS', 800),

    'pairing_policy' => env('OPENCLAW_PAIRING_POLICY', 'pairing'),
    // 'pairing' | 'allowlist' | 'open'
];
```

```php
// config/agents.php

return [
    'default' => [
        'name'        => env('AGENT_NAME', 'Assistant'),
        'description' => env('AGENT_DESCRIPTION', 'A helpful AI assistant'),
        'model'       => env('AGENT_MODEL', 'claude-sonnet-4-5-20250929'),
        'tools'       => [
            'deny'            => [],
            'requireApproval' => ['system.run'],
        ],
    ],
];
```

```php
// config/models.php

return [
    'providers' => [
        'anthropic' => [
            'api_key'    => env('ANTHROPIC_API_KEY'),
            'default'    => 'claude-sonnet-4-5-20250929',
            'timeout'    => 120,
            'max_retries'=> 3,
        ],
        'openai' => [
            'api_key' => env('OPENAI_API_KEY'),
            'default' => 'gpt-4o',
        ],
        'ollama' => [
            'base_url' => env('OLLAMA_BASE_URL', 'http://localhost:11434'),
            'default'  => 'llama3',
        ],
    ],
    'failover_order' => ['anthropic', 'openai', 'ollama'],
];
```

### Hot-Reload via Artisan

```php
// app/Console/Commands/ConfigReload.php

class ConfigReload extends Command
{
    protected $signature = 'config:reload';

    public function handle(): void
    {
        Artisan::call('config:clear');
        // Re-read .env and config files

        event(new ConfigChanged());

        $this->info('Configuration reloaded.');
    }
}
```

---

## 9. Model Provider Integration → Contract + Driver Pattern

### LLM Provider Contract

```php
// app/Contracts/LLMProvider.php

interface LLMProvider
{
    /** Send a chat completion request with tool definitions. */
    public function chat(
        string $systemPrompt,
        array  $messages,
        array  $tools = [],
    ): LLMResponse;

    /** Stream a chat completion (yields chunks). */
    public function stream(
        string $systemPrompt,
        array  $messages,
        array  $tools = [],
    ): \Generator;
}
```

### Model Router (Failover)

```php
// app/Services/LLM/ModelRouter.php

class ModelRouter
{
    public function __construct(
        private ProviderFactory $factory,
    ) {}

    public function resolve(Agent $agent): LLMProvider
    {
        $preferredProvider = $agent->config['model_provider'] ?? config('models.failover_order.0');
        $failoverOrder     = config('models.failover_order');

        // Try preferred first, then failover
        $ordered = collect([$preferredProvider, ...$failoverOrder])->unique();

        foreach ($ordered as $providerName) {
            $profile = AuthProfile::where('provider', $providerName)
                ->where('is_active', true)
                ->where(fn ($q) => $q->whereNull('cooldown_until')->orWhere('cooldown_until', '<', now()))
                ->first();

            if ($profile) {
                return $this->factory->make($providerName, $profile);
            }
        }

        throw new NoAvailableModelException('All model providers are in cooldown or unconfigured.');
    }
}
```

### Auth Profile Model

```php
// app/Models/AuthProfile.php

class AuthProfile extends Model
{
    protected $casts = [
        'credentials'    => 'encrypted:array',  // Laravel encrypted cast
        'is_active'      => 'boolean',
        'cooldown_until'  => 'datetime',
        'requests_today' => 'integer',
    ];

    public function applyCooldown(int $minutes = 5): void
    {
        $this->update(['cooldown_until' => now()->addMinutes($minutes)]);
    }
}
```

---

## 10. Security → Laravel Middleware + Gates + Encryption

### Gateway Token Authentication

```php
// app/Http/Middleware/ValidateGatewayToken.php

class ValidateGatewayToken
{
    public function handle(Request $request, Closure $next): Response
    {
        $token = $request->bearerToken();

        if (!$token || !hash_equals(config('openclaw.gateway.token'), $token)) {
            abort(401, 'Invalid gateway token');
        }

        return $next($request);
    }
}
```

### DM Pairing (Eloquent Model + Listener)

```php
// app/Models/PairedSender.php

class PairedSender extends Model
{
    protected $casts = [
        'channel_type' => ChannelType::class,
        'approved_at'  => 'datetime',
    ];

    public static function isAuthorized(string $senderId, ChannelType $channel): bool
    {
        return match (config('openclaw.pairing_policy')) {
            'open'      => true,
            'allowlist'  => self::where('sender_id', $senderId)->where('channel_type', $channel)->exists(),
            'pairing'   => self::where('sender_id', $senderId)->where('channel_type', $channel)->whereNotNull('approved_at')->exists(),
            default     => false,
        };
    }
}
```

### Secret Storage

```php
// .env (never committed)
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
OPENCLAW_GATEWAY_TOKEN=random-secret-token
TELEGRAM_BOT_TOKEN=123456:ABC...
```

Stored credentials use Laravel's `encrypted` cast (AES-256-GCM via `APP_KEY`).

---

## 11. Database Schema (SQLite)

All state lives in a single `database/database.sqlite` file.

```
┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│    agents       │───<│   sessions     │───<│     turns      │
│                │    │                │    │                │
│ id             │    │ id             │    │ id             │
│ name           │    │ agent_id  (FK) │    │ session_id (FK)│
│ description    │    │ key            │    │ role (enum)    │
│ config (JSON)  │    │ metadata (JSON)│    │ content (JSON) │
│ is_active      │    │ created_at     │    │ created_at     │
│ created_at     │    │ updated_at     │    └────────────────┘
│ updated_at     │    └────────────────┘
└────────────────┘

┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│  auth_profiles │    │   cron_jobs    │───<│cron_executions │
│                │    │                │    │                │
│ id             │    │ id             │    │ id             │
│ provider       │    │ agent_id  (FK) │    │ cron_job_id(FK)│
│ credentials    │    │ schedule       │    │ status         │
│   (encrypted)  │    │ action         │    │ output (TEXT)  │
│ is_active      │    │ delivery_mode  │    │ executed_at    │
│ cooldown_until │    │ cooldown_min   │    └────────────────┘
│ requests_today │    │ is_active      │
└────────────────┘    │ last_run_at    │
                      └────────────────┘

┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│   webhooks     │    │   embeddings   │    │paired_senders  │
│                │    │                │    │                │
│ id             │    │ id             │    │ id             │
│ agent_id  (FK) │    │ agent_id  (FK) │    │ sender_id      │
│ session_key    │    │ session_id(FK) │    │ channel_type   │
│ token          │    │ turn_id   (FK) │    │ pairing_code   │
│ is_active      │    │ content (TEXT) │    │ approved_at    │
└────────────────┘    │ vector  (BLOB) │    │ created_at     │
                      │ content_hash   │    └────────────────┘
┌────────────────┐    │ created_at     │
│    skills      │    └────────────────┘
│                │
│ id             │    ┌────────────────┐
│ name           │    │    plugins     │
│ version        │    │                │
│ prompt_fragment│    │ id             │
│ tool_class     │    │ name           │
│ is_active      │    │ version        │
└────────────────┘    │ service_provider│
                      │ is_active      │
  ┌─────────────┐     └────────────────┘
  │agent_skill  │
  │ (pivot)     │    ┌────────────────┐
  │             │    │   channels     │
  │ agent_id    │    │                │
  │ skill_id    │    │ id             │
  └─────────────┘    │ type (enum)    │
                      │ credentials   │
                      │   (encrypted) │
                      │ is_active     │
                      │ metadata(JSON)│
                      └────────────────┘
```

### SQLite Configuration for Performance

```php
// config/database.php

'sqlite' => [
    'driver'   => 'sqlite',
    'database' => database_path('database.sqlite'),
    'prefix'   => '',
    'foreign_key_constraints' => true,
    // Performance pragmas
    'options' => [],
],

// In AppServiceProvider::boot()
DB::statement('PRAGMA journal_mode=WAL');      // Concurrent reads + writes
DB::statement('PRAGMA synchronous=NORMAL');    // Good balance of safety + speed
DB::statement('PRAGMA cache_size=-64000');     // 64 MB cache
DB::statement('PRAGMA busy_timeout=5000');     // 5 second busy retry
DB::statement('PRAGMA temp_store=MEMORY');     // In-memory temp tables
```

---

## 12. Go Sidecars (Optional Performance Modules)

For subsystems where PHP's request-response lifecycle is limiting, **Go sidecar processes** handle long-lived connections and CPU-intensive work.

### Architecture

```
┌─────────────────────────────────────────────────┐
│              FrankenPHP (Laravel)                │
│  HTTP ←──→ Routes, Controllers, Queue, Events   │
│  WS   ←──→ Reverb (broadcasting)               │
└──────────────┬────────────┬─────────────────────┘
               │ HTTP/gRPC  │ HTTP/Unix socket
       ┌───────▼──────┐  ┌──▼─────────────────┐
       │  Go: LLM     │  │  Go: Channel Bridge │
       │  Streaming    │  │  (WhatsApp, Discord) │
       │  Proxy        │  │                     │
       │  :9001        │  │  :9002              │
       └──────────────┘  └─────────────────────┘
               │
       ┌───────▼──────┐
       │  Go: Browser  │
       │  Automation   │
       │  (Rod/CDP)    │
       │  :9003        │
       └──────────────┘
```

### When to Use Go vs PHP

| Concern | PHP (Laravel) | Go (Sidecar) |
|---|---|---|
| HTTP request handling | Yes | — |
| Database operations | Yes | — |
| Queue processing | Yes | — |
| Event broadcasting | Yes (Reverb) | — |
| Cron scheduling | Yes | — |
| LLM streaming (SSE) | Possible but blocking | Better: true async I/O |
| Persistent WebSocket clients | Limited by worker lifecycle | Better: goroutines |
| Browser automation | Via HTTP to sidecar | Better: native CDP |
| CPU-heavy embedding | Possible | Better: parallel processing |

### Communication Pattern

```php
// Laravel calls Go sidecar via HTTP

class GoLLMStreamProxy implements LLMProvider
{
    public function stream(string $systemPrompt, array $messages, array $tools = []): \Generator
    {
        $response = Http::withOptions(['stream' => true])
            ->post('http://localhost:9001/v1/chat/stream', [
                'system'   => $systemPrompt,
                'messages' => $messages,
                'tools'    => $tools,
                'provider' => $this->providerConfig,
            ]);

        foreach ($this->readSSE($response) as $chunk) {
            yield LLMChunk::fromArray($chunk);
        }
    }
}
```

### Go Sidecar Process Management

Managed via **Supervisor** or **systemd** alongside FrankenPHP:

```ini
# /etc/supervisor/conf.d/openclaw-sidecars.conf

[program:llm-stream]
command=/opt/openclaw/go-sidecars/llm-stream
autorestart=true
stdout_logfile=/var/log/openclaw/llm-stream.log

[program:channel-bridge]
command=/opt/openclaw/go-sidecars/channel-bridge
autorestart=true
stdout_logfile=/var/log/openclaw/channel-bridge.log
```

---

## 13. Deployment

### Single-Binary with FrankenPHP

```dockerfile
# Dockerfile

FROM dunglas/frankenphp:latest

# Install PHP extensions
RUN install-php-extensions \
    pdo_sqlite \
    pcntl \
    intl \
    bcmath

# Copy application
COPY . /app
WORKDIR /app

# Install dependencies
RUN composer install --no-dev --optimize-autoloader
RUN php artisan config:cache
RUN php artisan route:cache
RUN php artisan view:cache

# Copy Go sidecars (pre-built)
COPY --from=go-builder /sidecars/ /opt/openclaw/go-sidecars/

# Create SQLite database
RUN touch database/database.sqlite
RUN php artisan migrate --force

EXPOSE 443 8080

# FrankenPHP serves HTTP + Reverb WebSocket + worker mode
CMD ["frankenphp", "php-server", "--worker", "public/index.php"]
```

### Docker Compose (Full Stack)

```yaml
# docker-compose.yml

services:
  app:
    build: .
    ports:
      - "443:443"       # HTTPS (Caddy auto-TLS)
      - "8080:8080"     # Reverb WebSocket
    volumes:
      - ./database:/app/database          # Persist SQLite
      - ./storage:/app/storage            # Logs, cache, sandbox
    env_file: .env
    depends_on:
      - llm-stream
      - channel-bridge

  queue-worker:
    build: .
    command: php artisan queue:work --sleep=1 --tries=3
    volumes:
      - ./database:/app/database
      - ./storage:/app/storage
    env_file: .env

  scheduler:
    build: .
    command: php artisan schedule:work
    volumes:
      - ./database:/app/database
    env_file: .env

  llm-stream:
    build: ./go-sidecars/llm-stream
    ports:
      - "9001:9001"
    env_file: .env

  channel-bridge:
    build: ./go-sidecars/channel-bridge
    ports:
      - "9002:9002"
    env_file: .env
```

### Minimal Setup (No Go, No Docker)

For the simplest possible deployment — just PHP:

```bash
# Install
composer create-project openclaw/openclaw
cd openclaw
cp .env.example .env
php artisan key:generate

# Configure
# Edit .env with your API keys and channel tokens

# Setup database
touch database/database.sqlite
php artisan migrate

# Run everything with FrankenPHP
frankenphp php-server --worker public/index.php &
php artisan queue:work &
php artisan schedule:work &
php artisan reverb:start &
```

---

## 14. Artisan CLI Commands

The Artisan CLI replaces the OpenClaw CLI commands:

| OpenClaw CLI | Artisan Command | Description |
|---|---|---|
| `openclaw gateway` | `php artisan gateway:serve` | Start FrankenPHP + Reverb + queue + scheduler |
| `openclaw agent` | `php artisan agent:chat` | Interactive agent conversation in terminal |
| `openclaw message send` | `php artisan message:send {channel} {to} {text}` | Send a message via channel |
| `openclaw channels` | `php artisan channel:list` | Show channel statuses |
| `openclaw channels auth` | `php artisan channel:auth {type}` | Authenticate a channel |
| `openclaw config` | `php artisan config:show` | Display current configuration |
| `openclaw config reload` | `php artisan config:reload` | Hot-reload configuration |
| `openclaw cron` | `php artisan cron:list` | List scheduled jobs |
| `openclaw memory` | `php artisan memory:search {query}` | Semantic search |
| `openclaw memory reindex` | `php artisan memory:reindex` | Re-embed all content |
| `openclaw skills install` | `php artisan skill:install {name}` | Install a skill |
| `openclaw plugins` | `php artisan plugin:list` | List installed plugins |
| `openclaw pairing approve` | `php artisan pairing:approve {channel} {code}` | Approve a sender |
| `openclaw doctor` | `php artisan doctor` | Run diagnostics |
| `openclaw update` | `composer update openclaw/openclaw` | Update framework |

---

## 15. Data Flow Summary (Laravel Edition)

### Complete Message Lifecycle

```
1. External Platform (e.g., Telegram)
   │
   ▼
2. Channel Driver (TelegramDriver) — normalizes to InboundMessage
   │
   ▼
3. Event: MessageReceived — fires listener chain:
   │  ├── NormalizeMessage
   │  ├── EnforceAllowlist         (checks PairedSender model)
   │  ├── DetectMentionGating
   │  ├── DetectCommand
   │  ├── StageMedia               (storage/app/sandbox/)
   │  ├── ResolveSession           (finds/creates Session model)
   │  └── DebounceMessages
   │
   ▼
4. Job: ProcessInboundMessage (queued, database driver)
   │
   ▼
5. AgentRunner::run()
   │  ├── Turn::create(role: user)
   │  ├── SystemPromptBuilder::build()
   │  ├── ContextCompactor::prepare()
   │  └── ModelRouter::resolve() → LLMProvider
   │
   ▼
6. LLMProvider::chat() — calls Anthropic/OpenAI/Gemini/Ollama
   │
   ├──► ToolExecutor::execute()
   │      ├── Event: ToolExecuting
   │      ├── Tool::execute()
   │      ├── Turn::create(role: tool_call)
   │      ├── Turn::create(role: tool_result)
   │      ├── Event: ToolExecuted
   │      └── Continue LLM loop
   │
   ▼
7. Broadcast: AgentResponseChunk (via Reverb WebSocket)
   │
   ▼
8. Job: DeliverOutboundMessage (queued)
   │  ├── ChannelDriver::chunk()
   │  └── ChannelDriver::send()
   │
   ▼
9. External Platform (response delivered)
```

### Trigger-Initiated Flow

```
Trigger Source
├── Cron: schedule:work → ExecuteCronAction job
├── Webhook: POST /hook/{id} → WebhookController
├── Pub/Sub: POST /gmail-push → GmailController
└── Device: POST /device-event → DeviceEventController
   │
   ▼
RunAgentSession::dispatch($session, $payload)
   │
   ▼
(Same flow as steps 5-9 above)
```

---

## Concept Glossary (Laravel Terms)

| OpenClaw Term | Laravel Equivalent |
|---|---|
| Gateway | FrankenPHP + Reverb + Queue Worker + Scheduler |
| Channel Plugin | `ChannelDriver` implementation (Manager pattern) |
| Agent | `Agent` Eloquent Model + config |
| Session | `Session` Eloquent Model |
| Turn (append-only log) | `Turn` Eloquent Model (immutable inserts) |
| Hook | Laravel Event + Listener |
| Tool | Class implementing `Tool` contract, registered in `ToolRegistry` |
| Skill | Installable package: tool class + prompt fragment |
| Plugin | Laravel Package with its own `ServiceProvider` |
| System Prompt | `SystemPromptBuilder` service |
| Model Failover | `ModelRouter` service + `AuthProfile` model |
| Cron Job | `CronJob` model + Laravel Scheduler |
| Webhook | Route + Controller + `ValidateWebhookToken` middleware |
| Pairing | `PairedSender` model + `EnforceAllowlist` listener |
| Memory | `EmbeddingService` + `VectorStore` + `HybridSearch` services |
| Config hot-reload | `config:reload` Artisan command + `ConfigChanged` event |
| Go Sidecar | Separate binary, managed by Supervisor, talks HTTP to Laravel |

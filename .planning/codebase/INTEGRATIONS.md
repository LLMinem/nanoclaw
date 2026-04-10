# External Integrations

**Analysis Date:** 2026-04-10

## APIs & External Services

**AI / LLM:**

- Anthropic Claude - Core AI agent powering all responses
  - SDK/Client: `@anthropic-ai/claude-agent-sdk` (inside container)
  - Auth: Injected by OneCLI credential proxy via `ANTHROPIC_BASE_URL` — containers never see real API keys
  - Entry: `container/agent-runner/src/index.ts` → `query()` function
  - Claude Code CLI also installed globally in container for remote control feature

**OneCLI Credential Gateway:**

- OneCLI - Secure credential injection proxy for containerized agents
  - SDK/Client: `@onecli-sh/sdk` ^0.2.0
  - URL: `http://localhost:10254` (configurable via `ONECLI_URL` env var or `.env`)
  - Host-side: `src/index.ts` (ensures agents exist), `src/container-runner.ts` (applies container config)
  - Function: Intercepts container HTTPS traffic and injects API keys/OAuth tokens
  - Per-group agents: Non-main groups get their own OneCLI agent identifier

**Claude Remote Control:**

- Claude Code `remote-control` subcommand
  - Implementation: `src/remote-control.ts`
  - Spawns `claude remote-control --name "NanoClaw Remote"` as a detached process
  - Produces a `claude.ai/code` URL for browser-based access
  - Triggered via `/remote-control` command in main group chat

## Messaging Channels

**Channel Architecture:**

- Plugin-based channel system with factory pattern
  - Registry: `src/channels/registry.ts` — `registerChannel()` / `getChannelFactory()`
  - Barrel: `src/channels/index.ts` — imports trigger self-registration
  - Interface: `src/types.ts` → `Channel` interface (connect, sendMessage, isConnected, ownsJid, disconnect, setTyping?, syncGroups?)
  - Currently no channels are registered in the base barrel file — channels are added via skills (add-whatsapp, add-telegram, add-slack, add-discord)

**Supported Channel Skills:**

- WhatsApp (skill: `add-whatsapp`) - QR/pairing code auth
- Telegram (skill: `add-telegram`) - Bot token auth
- Slack (skill: `add-slack`) - Socket Mode
- Discord (skill: `add-discord`) - Bot token auth
- Gmail (skill: `add-gmail`) - GCP OAuth

## Data Storage

**Databases:**

- SQLite (embedded) via `better-sqlite3`
  - Location: `store/messages.db`
  - Schema: `src/db.ts` → `createSchema()`
  - Tables:
    - `chats` - Chat/group metadata (jid, name, channel, is_group)
    - `messages` - Full message content with reply context
    - `scheduled_tasks` - Cron/interval/one-time scheduled tasks
    - `task_run_logs` - Execution history for tasks
    - `router_state` - Key-value store for cursor/timestamp state
    - `sessions` - Claude session IDs per group
    - `registered_groups` - Group registrations with config
  - Migrations: Inline ALTER TABLE with try/catch for idempotency
  - Legacy migration: JSON files in `data/` migrated to SQLite on first run

**File Storage:**

- Group working directories: `groups/{folder}/` - Per-group agent workspace
- Group logs: `groups/{folder}/logs/` - Container execution logs
- IPC files: `data/ipc/{folder}/` - JSON files for container↔host communication
- Session data: `data/sessions/{folder}/.claude/` - Claude session state per group
- Conversation archives: Written to `groups/{folder}/conversations/` on session compaction
- Global memory: `groups/global/CLAUDE.md` - Shared instructions for non-main groups

**Caching:**

- Mount allowlist: In-memory singleton cache in `src/mount-security.ts` (reloaded on process restart)
- Sender allowlist: Re-read from disk on each call in `src/sender-allowlist.ts` (no cache)

## Authentication & Identity

**Auth Provider:**

- OneCLI Agent Vault - Manages API credentials for Claude and other services
  - Implementation: `@onecli-sh/sdk` applied in `src/container-runner.ts` → `buildContainerArgs()`
  - Per-group agent identifiers: `group.folder.toLowerCase().replace(/_/g, '-')`
  - Main group uses default OneCLI agent; sub-groups get named agents

**Message Sender Auth:**

- Sender allowlist at `~/.config/nanoclaw/sender-allowlist.json`
  - Implementation: `src/sender-allowlist.ts`
  - Two modes: `trigger` (deny trigger but keep message), `drop` (discard message entirely)
  - Per-chat overrides or global default

**Container Security:**

- Mount allowlist at `~/.config/nanoclaw/mount-allowlist.json`
  - Implementation: `src/mount-security.ts`
  - Validates host paths against allowed roots and blocked patterns
  - Non-main groups forced read-only when `nonMainReadOnly: true`
  - Default blocked: `.ssh`, `.gnupg`, `.aws`, `.kube`, `.docker`, credentials files
- `.env` shadow-mounted as `/dev/null` in main group containers to prevent secret access
- Containers run as non-root user (uid 1000 `node`)
- Group folder names validated against `^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$` pattern

## Monitoring & Observability

**Error Tracking:**

- None (no Sentry, Datadog, etc.)

**Logs:**

- Custom structured logger: `src/logger.ts`
  - Levels: debug, info, warn, error, fatal (threshold from `LOG_LEVEL` env var, default: info)
  - Colorized output to stdout (info/debug) and stderr (warn/error/fatal)
  - JSON-like structured data with timestamps
  - Uncaught exceptions and unhandled rejections routed through logger
- Container logs: Written to `groups/{folder}/logs/container-{timestamp}.log`
- macOS service logs: `logs/nanoclaw.log` and `logs/nanoclaw.error.log` (via launchd plist)

## CI/CD & Deployment

**Hosting:**

- Self-hosted daemon on macOS (primary) or Linux
- macOS: launchd service via `launchd/com.nanoclaw.plist` (RunAtLoad, KeepAlive)

**CI Pipeline:**

- GitHub Actions (`.github/workflows/ci.yml`)
  - Trigger: Pull requests to `main`
  - Steps: `npm ci`, format check, typecheck (`tsc --noEmit`), tests (`vitest run`)
  - Node.js 20 on `ubuntu-latest`

**Additional Workflows:**

- `.github/workflows/label-pr.yml` - Auto-label PRs
- `.github/workflows/update-tokens.yml` - Token updates
- `.github/workflows/bump-version.yml` - Version bumping

**Pre-commit Hook:**

- Husky runs `npm run format:fix` (Prettier auto-format)

## Inter-Process Communication (IPC)

**Container → Host (File-based IPC):**

- Messages: `data/ipc/{folder}/messages/*.json` — Agent sends messages to users
- Tasks: `data/ipc/{folder}/tasks/*.json` — Agent creates/modifies scheduled tasks
- Host polls IPC directories every 1000ms (`IPC_POLL_INTERVAL`)
- Atomic writes: temp file → rename pattern for crash safety
- Per-group IPC namespaces prevent cross-group privilege escalation

**Host → Container (File-based IPC):**

- Input: `data/ipc/{folder}/input/*.json` — Follow-up messages piped to running container
- Close sentinel: `data/ipc/{folder}/input/_close` — Signals container to shut down
- Container polls every 500ms (`IPC_POLL_MS`)

**Container stdout protocol:**

- Sentinel markers: `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---`
- JSON payload between markers: `{status, result, newSessionId, error}`
- Supports streaming: multiple marker pairs per container lifetime

**MCP Server (inside container):**

- `container/agent-runner/src/ipc-mcp-stdio.ts` — Stdio MCP server
- Tools exposed: `send_message`, `schedule_task`, `list_tasks`, `pause_task`, `resume_task`, `cancel_task`, `update_task`, `register_group`
- Context passed via env vars: `NANOCLAW_CHAT_JID`, `NANOCLAW_GROUP_FOLDER`, `NANOCLAW_IS_MAIN`

## Webhooks & Callbacks

**Incoming:**

- None (all channels use polling or persistent connections, not webhooks)

**Outgoing:**

- None

## Environment Configuration

**Required env vars (minimum to run):**

- At least one channel configured (credentials vary by channel skill)
- Docker must be installed and running

**Optional env vars:**

- `ASSISTANT_NAME` - Bot personality name (default: "Andy")
- `ASSISTANT_HAS_OWN_NUMBER` - Whether bot has dedicated phone number
- `ONECLI_URL` - OneCLI gateway URL (default: `http://localhost:10254`)
- `TZ` - IANA timezone (auto-detected from system if not set)
- `CONTAINER_IMAGE` - Docker image name (default: `nanoclaw-agent:latest`)
- `CONTAINER_TIMEOUT` - Hard timeout in ms (default: 1800000 / 30min)
- `IDLE_TIMEOUT` - Idle timeout before container cleanup (default: 1800000 / 30min)
- `MAX_CONCURRENT_CONTAINERS` - Concurrency limit (default: 5)
- `MAX_MESSAGES_PER_PROMPT` - Messages per agent invocation (default: 10)
- `LOG_LEVEL` - Logging threshold: debug, info, warn, error, fatal (default: info)

**Secrets location:**

- `.env` file in project root (read by custom parser, NOT loaded into process.env)
- Actual API keys managed by OneCLI Agent Vault (never exposed to containers)
- `~/.config/nanoclaw/` - Security configs (mount allowlist, sender allowlist)

---

_Integration audit: 2026-04-10_

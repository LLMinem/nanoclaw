# Architecture

**Analysis Date:** 2026-04-10

## Pattern Overview

**Overall:** Event-driven message router with containerized agent execution

**Key Characteristics:**

- Host process polls for messages, routes them to isolated Docker containers running Claude Agent SDK
- Multi-channel abstraction: channels register via factory pattern; host is channel-agnostic
- File-based IPC between host and containers (no network sockets or shared memory)
- Per-group isolation: each group gets its own filesystem namespace, IPC directory, and session
- Security boundary: containers never see real API keys; OneCLI gateway injects credentials into HTTPS traffic
- Concurrency managed by `GroupQueue` with configurable max containers and per-group backoff

## Layers

**Channel Layer:**

- Purpose: Connects to messaging platforms (WhatsApp, Telegram, Slack, Discord, etc.) and delivers inbound/outbound messages
- Location: `src/channels/`
- Contains: Channel registry (`registry.ts`), barrel import file (`index.ts`), and per-channel modules (added by skills)
- Depends on: Nothing in `src/` (channels are self-contained)
- Used by: `src/index.ts` (creates and connects channels at startup)

**Router / Core Loop Layer:**

- Purpose: Polls for new messages, applies trigger/allowlist logic, formats prompts, dispatches to containers, delivers agent output back to channels
- Location: `src/index.ts`, `src/router.ts`
- Contains: Message loop, group registration, state management, channel dispatch
- Depends on: Channel Layer, Database Layer, Container Layer, IPC Layer
- Used by: Entry point (self-contained main loop)

**Container Execution Layer:**

- Purpose: Spawns Docker containers, manages volume mounts, streams agent output
- Location: `src/container-runner.ts`, `src/container-runtime.ts`
- Contains: Volume mount construction, container arg building, output parsing with sentinel markers, timeout/idle management
- Depends on: Config, Group Folder, Mount Security, OneCLI SDK
- Used by: Router (message processing), Task Scheduler

**Agent Runner (In-Container):**

- Purpose: Runs inside Docker container; receives prompt via stdin, invokes Claude Agent SDK, streams results via stdout sentinel markers
- Location: `container/agent-runner/src/index.ts`, `container/agent-runner/src/ipc-mcp-stdio.ts`
- Contains: SDK query loop, IPC message polling, MCP server for agent tools (send_message, schedule_task, register_group, etc.)
- Depends on: `@anthropic-ai/claude-agent-sdk`, `@modelcontextprotocol/sdk`
- Used by: Docker container entrypoint

**Database Layer:**

- Purpose: Persists messages, groups, sessions, tasks, and router state in SQLite
- Location: `src/db.ts`
- Contains: Schema creation, inline migrations, CRUD for all entities
- Depends on: `better-sqlite3`, Config
- Used by: Router, IPC, Task Scheduler

**IPC Layer:**

- Purpose: File-based inter-process communication between containers and host
- Location: `src/ipc.ts` (host-side watcher), `container/agent-runner/src/ipc-mcp-stdio.ts` (container-side MCP server)
- Contains: Per-group namespaced directories (`data/ipc/{group}/messages/`, `data/ipc/{group}/tasks/`, `data/ipc/{group}/input/`)
- Depends on: Database (task CRUD), Group Folder (path validation)
- Used by: Agent Runner writes IPC files; host `ipc.ts` polls and processes them

**Task Scheduler Layer:**

- Purpose: Polls for due scheduled tasks and executes them via container agents
- Location: `src/task-scheduler.ts`
- Contains: Cron/interval/once scheduling, drift-resistant next-run computation, task execution
- Depends on: Container Runner, Database, GroupQueue
- Used by: Main loop (started as independent subsystem)

**Concurrency / Queue Layer:**

- Purpose: Limits concurrent containers, queues work per group, handles retries with exponential backoff
- Location: `src/group-queue.ts`
- Contains: `GroupQueue` class managing active/pending state per group, stdin piping for follow-up messages, idle detection
- Depends on: Config (`MAX_CONCURRENT_CONTAINERS`)
- Used by: Router, Task Scheduler

**Security Layer:**

- Purpose: Validates mount paths, sender allowlists, group folder names
- Location: `src/mount-security.ts`, `src/sender-allowlist.ts`, `src/group-folder.ts`
- Contains: Allowlist validation against external config, path traversal prevention, blocked pattern matching
- Depends on: Config (allowlist file paths)
- Used by: Container Runner, Router, IPC

**Configuration Layer:**

- Purpose: Reads config from `.env` and environment variables without polluting `process.env`
- Location: `src/config.ts`, `src/env.ts`
- Contains: All constants (timeouts, paths, trigger patterns, container image)
- Depends on: `.env` file (read selectively, never loaded into env)
- Used by: Everything

**Setup Layer:**

- Purpose: Interactive CLI for initial configuration (timezone, credentials, container build, group registration, launchd service)
- Location: `setup/`
- Contains: Step-based CLI runner, platform detection, service management
- Depends on: `src/logger.ts`, `src/config.ts`
- Used by: `npm run setup`

## Data Flow

**Inbound Message Flow:**

1. Channel adapter receives message from platform (WhatsApp/Telegram/Slack/Discord)
2. `onMessage` callback in `src/index.ts` checks sender allowlist, applies drop-mode filtering
3. Message stored in SQLite via `storeMessage()` (`src/db.ts`)
4. Main polling loop (`startMessageLoop`) detects new messages via `getNewMessages()`
5. For each group with pending messages, checks trigger pattern and sender permissions
6. If a container is already running for that group, pipes message via IPC file (`queue.sendMessage()`)
7. Otherwise, enqueues via `GroupQueue.enqueueMessageCheck()` which eventually calls `processGroupMessages()`
8. `processGroupMessages()` formats messages as XML (`<messages>` with `<message>` elements) and calls `runAgent()`
9. `runAgent()` writes task/group snapshots, then calls `runContainerAgent()` which spawns Docker container
10. Container's agent-runner reads stdin JSON, invokes Claude Agent SDK `query()`, streams results via stdout markers
11. Host parses `OUTPUT_START_MARKER`/`OUTPUT_END_MARKER` pairs from stdout, calls `onOutput` callback
12. Callback sends agent response back through channel via `channel.sendMessage()`

**IPC Command Flow (Container → Host):**

1. Agent inside container calls MCP tool (e.g., `send_message`, `schedule_task`, `register_group`)
2. MCP server (`ipc-mcp-stdio.ts`) writes JSON file to `/workspace/ipc/messages/` or `/workspace/ipc/tasks/`
3. Host `startIpcWatcher()` polls all group IPC directories every 1 second
4. Host validates authorization (group identity from directory path, isMain from lookup)
5. Host executes action (sends message, creates task, registers group)
6. Host deletes processed IPC file

**Follow-up Message Flow (Host → Container):**

1. New message arrives while container is already running for that group
2. `queue.sendMessage()` writes JSON file to `data/ipc/{group}/input/{timestamp}.json`
3. Agent runner's IPC poll loop (`pollIpcDuringQuery`) detects new file
4. Agent runner pushes text into `MessageStream` async iterable feeding the SDK `query()`
5. SDK processes as a follow-up user turn in the same conversation

**State Management:**

- Router state persisted in SQLite `router_state` table (last seen timestamps)
- Sessions (Claude conversation IDs) persisted in SQLite `sessions` table, keyed by group folder
- Registered groups persisted in SQLite `registered_groups` table
- In-memory caches (`lastTimestamp`, `sessions`, `registeredGroups`, `lastAgentTimestamp`) loaded at startup via `loadState()`
- Cursor recovery: if `lastAgentTimestamp` is missing for a group, recovers from last bot message timestamp in DB

## Key Abstractions

**Channel:**

- Purpose: Represents a messaging platform connection
- Examples: `src/types.ts` (interface), `src/channels/registry.ts` (factory pattern)
- Pattern: Factory registration — each channel module calls `registerChannel(name, factory)` on import. Factory returns `Channel | null` (null if credentials missing). Barrel file `src/channels/index.ts` triggers imports.

**RegisteredGroup:**

- Purpose: A chat/group that the bot is actively monitoring and responding to
- Examples: `src/types.ts` (interface), `src/db.ts` (persistence)
- Pattern: Groups are identified by JID (platform-specific chat ID). Each group has a folder name, trigger pattern, optional container config with additional mounts, and an `isMain` flag for the control group.

**GroupQueue:**

- Purpose: Serializes work per group while allowing concurrency across groups
- Examples: `src/group-queue.ts`
- Pattern: State machine per group (`active`, `idleWaiting`, `pendingMessages`, `pendingTasks`). Limits total active containers via `MAX_CONCURRENT_CONTAINERS`. Drains pending work after each container completes.

**ContainerInput / ContainerOutput:**

- Purpose: Structured protocol between host and container agent
- Examples: `src/container-runner.ts`, `container/agent-runner/src/index.ts`
- Pattern: JSON over stdin (input) and stdout with sentinel markers (output). Markers are `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---`.

**IPC File Protocol:**

- Purpose: Communication between container agents and host process
- Examples: `src/ipc.ts`, `container/agent-runner/src/ipc-mcp-stdio.ts`
- Pattern: Atomic write (write to `.tmp`, rename to `.json`). Per-group namespacing prevents cross-group access. Authorization verified by directory identity, not file contents.

## Entry Points

**Main Application:**

- Location: `src/index.ts`
- Triggers: `npm run dev` (tsx) or `npm start` (compiled)
- Responsibilities: Ensures container runtime, initializes database, loads state, connects channels, starts message loop, IPC watcher, and task scheduler

**Setup CLI:**

- Location: `setup/index.ts`
- Triggers: `npm run setup` → `tsx setup/index.ts --step <name>`
- Responsibilities: Step-based configuration (timezone, environment, container, groups, register, mounts, service, verify)

**Agent Runner (Container):**

- Location: `container/agent-runner/src/index.ts`
- Triggers: Docker container entrypoint reads stdin
- Responsibilities: Parses input, runs Claude Agent SDK query loop, streams results, polls IPC for follow-ups

**MCP Server (Container):**

- Location: `container/agent-runner/src/ipc-mcp-stdio.ts`
- Triggers: Spawned by agent-runner as MCP server for Claude Agent SDK
- Responsibilities: Provides tools (send_message, schedule_task, list_tasks, register_group, etc.) that write IPC files

## Error Handling

**Strategy:** Fail gracefully with retry and cursor rollback

**Patterns:**

- Container errors trigger cursor rollback so messages can be re-processed on retry (unless output was already sent to user, to prevent duplicates)
- `GroupQueue` implements exponential backoff (5s base, max 5 retries) for failed group processing
- Stale/corrupt session detection: if container fails with "no conversation found" or ENOENT on .jsonl, session ID is cleared for fresh retry
- IPC processing errors move files to `data/ipc/errors/` directory instead of deleting
- Container timeout distinguishes "timeout after output" (success, idle cleanup) from "timeout with no output" (error)
- Startup recovery via `recoverPendingMessages()` checks for unprocessed messages across all registered groups

## Cross-Cutting Concerns

**Logging:** Custom structured logger at `src/logger.ts` with pino-like API (`logger.info({data}, 'message')`). Levels: debug/info/warn/error/fatal. Colorized output to stdout (info and below) and stderr (warn and above). Uncaught exceptions and unhandled rejections routed through logger.

**Validation:** Group folder names validated via regex `^[A-Za-z0-9][A-Za-z0-9_-]{0,63}$` with reserved name blocking. Mount paths validated against external allowlist with symlink resolution and blocked pattern matching. Container paths validated for traversal attacks.

**Authentication/Authorization:** Two-tier model:

- _Sender allowlist_ (`~/.config/nanoclaw/sender-allowlist.json`): controls which senders can trigger the bot per chat, with trigger-only or drop modes
- _Group privilege model_: main group has elevated privileges (register groups, refresh metadata, schedule cross-group tasks). Non-main groups can only operate within their own scope. `isMain` cannot be set via IPC.

**Secrets Management:** `.env` is read selectively via `readEnvFile()` — values are NOT loaded into `process.env` to prevent leaking to child processes. Container agents never see real API keys; credentials are injected by OneCLI gateway intercepting HTTPS traffic. `.env` file is shadow-mounted as `/dev/null` in containers.

---

_Architecture analysis: 2026-04-10_

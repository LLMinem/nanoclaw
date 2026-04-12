<!-- GSD:project-start source:PROJECT.md -->
## Project

**MeeqClaw**

A personal AI assistant (Fritz) built on NanoClaw, deployed on a Hetzner VPS, accessible via Telegram. MeeqClaw is meeq's fork of NanoClaw (~4,800 LOC Node.js/TypeScript), replacing OpenClaw (~500k LOC) as the daily-driver AI assistant. Doubles as a hands-on learning project for understanding AI agent architecture -- every change should deepen meeq's understanding of the system.

**Core Value:** Fritz is always reachable via Telegram, understands meeq's context through persistent memory, and can take action on meeq's systems.

### Constraints

- **Timeline**: 3-4 days to functional assistant (OpenClaw still running as fallback)
- **Platform**: Ubuntu 24.04, Hetzner CX32 VPS (primary); macOS MacBook (development only for now)
- **Container**: Docker on Linux (native, no VM overhead)
- **SDK**: Claude Agent SDK (must stay subscription-compatible)
- **Channel**: Telegram primary (sole channel for v1)
- **Branch**: All work on `meeq-claw` branch; `main` tracks upstream
- **Learning**: Changes must be understood, not just applied — this is a learning project
- **Credentials**: OneCLI gateway (setup default)
<!-- GSD:project-end -->

<!-- GSD:stack-start source:codebase/STACK.md -->
## Technology Stack

## Languages
- TypeScript 5.7+ - All host-side source (`src/`), agent-runner (`container/agent-runner/src/`), and setup scripts (`setup/`)
- Bash - Container build script (`container/build.sh`), setup entrypoint (`setup.sh`), container entrypoint (inline in `container/Dockerfile`)
- XML (plist) - macOS launchd service definition (`launchd/com.nanoclaw.plist`)
## Runtime
- Node.js 22 (pinned in `.nvmrc`, `container/Dockerfile` uses `node:22-slim`)
- Minimum Node.js 20 (declared in `package.json` engines)
- npm (lockfile: `package-lock.json` present)
- Two separate package roots:
## Frameworks
- No web framework - NanoClaw is a message-processing daemon, not a web server
- `@anthropic-ai/claude-agent-sdk` ^0.2.76 - Claude Code Agent SDK used inside containers (`container/agent-runner/src/index.ts`)
- `@modelcontextprotocol/sdk` ^1.12.1 - MCP server for container-side IPC tools (`container/agent-runner/src/ipc-mcp-stdio.ts`)
- `@onecli-sh/sdk` ^0.2.0 - OneCLI credential gateway SDK for secure API key injection (`src/index.ts`, `src/container-runner.ts`)
- Vitest ^4.0.18 - Test runner
- TypeScript ^5.7.0 - Compiler (`tsc`)
- tsx ^4.19.0 - Dev-time TypeScript execution (`npm run dev`)
- Prettier ^3.8.1 - Code formatting
- ESLint ^9.35.0 - Linting with flat config
- Husky ^9.1.7 - Git hooks (pre-commit runs `npm run format:fix`)
## Key Dependencies
- `better-sqlite3` 11.10.0 - Embedded SQLite database for messages, tasks, sessions, router state (`src/db.ts`)
- `cron-parser` 5.5.0 - Parses cron expressions for scheduled tasks (`src/ipc.ts`, `src/task-scheduler.ts`)
- `@onecli-sh/sdk` ^0.2.0 - Secure credential proxy for container API access (`src/container-runner.ts`)
- `@anthropic-ai/claude-agent-sdk` ^0.2.76 - Runs Claude Code inside containers (`container/agent-runner/src/index.ts`)
- `@modelcontextprotocol/sdk` ^1.12.1 - MCP stdio server exposing NanoClaw tools to the agent (`container/agent-runner/src/ipc-mcp-stdio.ts`)
- `zod` ^4.0.0 - Schema validation for MCP tool parameters (`container/agent-runner/src/ipc-mcp-stdio.ts`)
- `cron-parser` ^5.0.0 - Validates cron expressions in-container (`container/agent-runner/src/ipc-mcp-stdio.ts`)
- Docker - Container runtime for agent isolation (`src/container-runtime.ts`, `container/Dockerfile`)
- Chromium (inside container) - Browser automation via `agent-browser` global npm package
## TypeScript Configuration
- `"type": "module"` in both `package.json` files (ESM throughout)
- `resolveJsonModule: true` (host only)
- `declarationMap: true` (host only)
## Configuration
- `.env` file in project root, read by custom parser in `src/env.ts`
- Custom env reader does NOT load into `process.env` (secrets stay out of child process environment)
- Keys read from `.env`: `ASSISTANT_NAME`, `ASSISTANT_HAS_OWN_NUMBER`, `ONECLI_URL`, `TZ`
- Process env vars for runtime tuning: `CONTAINER_IMAGE`, `CONTAINER_TIMEOUT`, `CONTAINER_MAX_OUTPUT_SIZE`, `MAX_MESSAGES_PER_PROMPT`, `IDLE_TIMEOUT`, `MAX_CONCURRENT_CONTAINERS`, `LOG_LEVEL`
- `~/.config/nanoclaw/mount-allowlist.json` - Controls which host directories containers can mount
- `~/.config/nanoclaw/sender-allowlist.json` - Controls which message senders can trigger the agent
- `tsconfig.json` - Host TypeScript config
- `container/agent-runner/tsconfig.json` - Container TypeScript config
- `eslint.config.js` - Flat ESLint config (ignores `container/` and `groups/`)
- `.prettierrc` - Single setting: `singleQuote: true`
## Platform Requirements
- Node.js >= 20 (22 recommended via `.nvmrc`)
- Docker installed and running (verified at startup by `src/container-runtime.ts`)
- npm for dependency management
- macOS primary target (launchd plist in `launchd/com.nanoclaw.plist`)
- Linux supported (host gateway handling in `src/container-runtime.ts`)
- Container image `nanoclaw-agent:latest` must be built first (`container/build.sh`)
- OneCLI gateway at `http://localhost:10254` for credential injection (configurable via `ONECLI_URL`)
- Base: `node:22-slim`
- Includes: Chromium, git, curl, CJK/emoji fonts
- Global npm packages: `agent-browser`, `@anthropic-ai/claude-code`
- Runs as non-root `node` user (uid 1000)
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

## Naming Patterns
- Use `kebab-case.ts` for all source files: `container-runner.ts`, `group-queue.ts`, `sender-allowlist.ts`
- Test files are co-located with source, named `{module}.test.ts`: `db.test.ts`, `routing.test.ts`
- Entry points: `index.ts` in `src/` and `src/channels/`
- Use `camelCase` for all functions: `formatMessages()`, `loadSenderAllowlist()`, `getOrRecoverCursor()`
- Prefix boolean-returning functions with `is`/`has`/`should`: `isValidTimezone()`, `shouldDropMessage()`, `isSenderAllowed()`
- Prefix internal test helpers with underscore: `_initTestDatabase()`, `_closeDatabase()`, `_setRegisteredGroups()`, `_resetSchedulerLoopForTests()`
- Annotate test-only exports with `/** @internal - for tests only */` JSDoc
- Use `camelCase` for local variables and parameters: `lastTimestamp`, `missedMessages`, `registeredGroups`
- Use `UPPER_SNAKE_CASE` for module-level constants: `POLL_INTERVAL`, `MAX_CONCURRENT_CONTAINERS`, `DEFAULT_TRIGGER`, `BASE_RETRY_MS`
- Use `camelCase` for mutable module-level state: `lastTimestamp`, `sessions`, `messageLoopRunning`
- Use `PascalCase` for interfaces: `RegisteredGroup`, `NewMessage`, `ScheduledTask`, `Channel`
- Use `PascalCase` for type aliases: `OnInboundMessage`, `OnChatMetadata`, `ChannelFactory`
- Use `PascalCase` for classes: `GroupQueue`
- Define interfaces in `src/types.ts` for shared types; co-locate module-specific interfaces with their module
## Code Style
- Prettier with single quotes (`singleQuote: true`)
- All other Prettier settings are defaults (2-space indent, 80 char print width, trailing commas, semicolons)
- Pre-commit hook runs `npm run format:fix` via Husky
- ESLint 9 with flat config (`eslint.config.js`)
- TypeScript-ESLint recommended rules
- `no-catch-all` plugin warns against generic catch blocks
- Unused variables must be prefixed with `_`: `argsIgnorePattern: '^_'`, `varsIgnorePattern: '^_'`
- `@typescript-eslint/no-explicit-any` set to warn (not error)
- Scope: `src/**/*.{js,ts}` only; `container/`, `groups/`, `dist/` are excluded
- Strict mode enabled (`"strict": true` in `tsconfig.json`)
- Target: ES2022, Module: NodeNext
- ESM modules (`"type": "module"` in package.json)
- All imports use `.js` extension: `import { logger } from './logger.js'`
## Import Organization
- None used. All imports are relative paths with `.js` extensions.
## Error Handling
- Use structured logging with `logger` for all error reporting (never `console.log` or `console.error`)
- Catch blocks must name the error parameter (enforced by `preserve-caught-error` lint rule) or use empty catch
- Empty catch blocks are used for "expected to fail" operations like DB migrations: `try { db.exec('ALTER TABLE ...') } catch { /* column already exists */ }`
- Functions return `'success' | 'error'` string literals rather than throwing for recoverable operations (see `runAgent()` in `src/index.ts`)
- Security-sensitive operations fail closed: `loadMountAllowlist()` returns `null` on error → mounts are blocked
- Stale session detection uses regex matching against error messages, then clears the session for retry
- Validate early and return/skip invalid data with a warning log
- Use type guard functions for runtime validation: `isValidEntry()`, `isValidGroupFolder()`, `isValidTimezone()`
- Parameterized SQL queries throughout — never string interpolation for DB queries
## Logging
- Two calling signatures: `logger.info('simple message')` or `logger.info({ key: 'value' }, 'message with context')`
- Log levels: `debug`, `info`, `warn`, `error`, `fatal`
- Threshold configured via `LOG_LEVEL` env var (default: `info`)
- `warn` and above write to `stderr`; `debug` and `info` to `stdout`
- Uncaught exceptions and unhandled rejections are routed through the logger
- Use `{ err }` key for Error objects — the logger formats stack traces specially
- `debug`: Detailed operational flow (message piping, IPC, cache hits/skips)
- `info`: State transitions, group registration, startup, message counts
- `warn`: Recoverable issues (missing credentials, invalid config, rejected operations)
- `error`: Failed operations that need attention (container failures, agent errors)
- `fatal`: Unrecoverable startup failures, uncaught exceptions (followed by `process.exit(1)`)
## Comments
- Use comments for non-obvious "why" decisions, not "what" the code does
- Reference issue numbers for fixes: `// Fixes #1391`
- Use inline comments for security rationale: `// Mount security: allowlist stored OUTSIDE project root, never mounted into containers`
- Block comments explain migration logic or workarounds for edge cases
- Empty catch blocks must have a comment explaining why: `/* column already exists */`
- Use `/** @internal */` to mark test-only exports
- Full JSDoc only on key public functions and complex helpers
- Inline `//` comments preferred over JSDoc for implementation notes
## Function Design
- Functions are generally short (10-40 lines). Complex orchestration functions (`startMessageLoop`, `processGroupMessages`) are longer but follow a clear sequential flow.
- Use interfaces/types for complex parameter sets rather than positional params
- Callback-based dependency injection for subsystem integration (see `startSchedulerLoop`, `startIpcWatcher` in `src/index.ts`)
- Group related config into objects: `ChannelOpts`, `IpcDeps`
- Functions return explicit types; avoid `any`
- Use `| undefined` for optional returns rather than `null` (except for DB/SQL operations where `null` is idiomatic)
- Async functions that can fail return result types (`'success' | 'error'`) or use boolean return
## Module Design
- Named exports only — no default exports anywhere in the codebase
- Re-exports for backwards compatibility: `export { escapeXml, formatMessages } from './router.js'`
- Test-only exports prefixed with `_` and annotated `@internal`
- `src/channels/index.ts` is a barrel that triggers channel self-registration via side-effect imports
- No other barrel files; modules import directly from their target
- Config values are centralized in `src/config.ts` — import constants from there, not from `.env` directly
- Database operations are centralized in `src/db.ts` — never create `Database` instances elsewhere in production code
- Logger is a singleton exported from `src/logger.ts`
## Security Conventions
- Never pass secrets via environment variables to child processes — use `readEnvFile()` to read selectively
- Container mount paths are validated against an external allowlist (`~/.config/nanoclaw/mount-allowlist.json`)
- Shell commands validate inputs before interpolation: `stopContainer()` rejects names with shell metacharacters
- Group folder names are validated with `isValidGroupFolder()` to prevent path traversal
- Sender authorization is checked at multiple layers (trigger, drop, IPC)
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

## Pattern Overview
- Host process polls for messages, routes them to isolated Docker containers running Claude Agent SDK
- Multi-channel abstraction: channels register via factory pattern; host is channel-agnostic
- File-based IPC between host and containers (no network sockets or shared memory)
- Per-group isolation: each group gets its own filesystem namespace, IPC directory, and session
- Security boundary: containers never see real API keys; OneCLI gateway injects credentials into HTTPS traffic
- Concurrency managed by `GroupQueue` with configurable max containers and per-group backoff
## Layers
- Purpose: Connects to messaging platforms (WhatsApp, Telegram, Slack, Discord, etc.) and delivers inbound/outbound messages
- Location: `src/channels/`
- Contains: Channel registry (`registry.ts`), barrel import file (`index.ts`), and per-channel modules (added by skills)
- Depends on: Nothing in `src/` (channels are self-contained)
- Used by: `src/index.ts` (creates and connects channels at startup)
- Purpose: Polls for new messages, applies trigger/allowlist logic, formats prompts, dispatches to containers, delivers agent output back to channels
- Location: `src/index.ts`, `src/router.ts`
- Contains: Message loop, group registration, state management, channel dispatch
- Depends on: Channel Layer, Database Layer, Container Layer, IPC Layer
- Used by: Entry point (self-contained main loop)
- Purpose: Spawns Docker containers, manages volume mounts, streams agent output
- Location: `src/container-runner.ts`, `src/container-runtime.ts`
- Contains: Volume mount construction, container arg building, output parsing with sentinel markers, timeout/idle management
- Depends on: Config, Group Folder, Mount Security, OneCLI SDK
- Used by: Router (message processing), Task Scheduler
- Purpose: Runs inside Docker container; receives prompt via stdin, invokes Claude Agent SDK, streams results via stdout sentinel markers
- Location: `container/agent-runner/src/index.ts`, `container/agent-runner/src/ipc-mcp-stdio.ts`
- Contains: SDK query loop, IPC message polling, MCP server for agent tools (send_message, schedule_task, register_group, etc.)
- Depends on: `@anthropic-ai/claude-agent-sdk`, `@modelcontextprotocol/sdk`
- Used by: Docker container entrypoint
- Purpose: Persists messages, groups, sessions, tasks, and router state in SQLite
- Location: `src/db.ts`
- Contains: Schema creation, inline migrations, CRUD for all entities
- Depends on: `better-sqlite3`, Config
- Used by: Router, IPC, Task Scheduler
- Purpose: File-based inter-process communication between containers and host
- Location: `src/ipc.ts` (host-side watcher), `container/agent-runner/src/ipc-mcp-stdio.ts` (container-side MCP server)
- Contains: Per-group namespaced directories (`data/ipc/{group}/messages/`, `data/ipc/{group}/tasks/`, `data/ipc/{group}/input/`)
- Depends on: Database (task CRUD), Group Folder (path validation)
- Used by: Agent Runner writes IPC files; host `ipc.ts` polls and processes them
- Purpose: Polls for due scheduled tasks and executes them via container agents
- Location: `src/task-scheduler.ts`
- Contains: Cron/interval/once scheduling, drift-resistant next-run computation, task execution
- Depends on: Container Runner, Database, GroupQueue
- Used by: Main loop (started as independent subsystem)
- Purpose: Limits concurrent containers, queues work per group, handles retries with exponential backoff
- Location: `src/group-queue.ts`
- Contains: `GroupQueue` class managing active/pending state per group, stdin piping for follow-up messages, idle detection
- Depends on: Config (`MAX_CONCURRENT_CONTAINERS`)
- Used by: Router, Task Scheduler
- Purpose: Validates mount paths, sender allowlists, group folder names
- Location: `src/mount-security.ts`, `src/sender-allowlist.ts`, `src/group-folder.ts`
- Contains: Allowlist validation against external config, path traversal prevention, blocked pattern matching
- Depends on: Config (allowlist file paths)
- Used by: Container Runner, Router, IPC
- Purpose: Reads config from `.env` and environment variables without polluting `process.env`
- Location: `src/config.ts`, `src/env.ts`
- Contains: All constants (timeouts, paths, trigger patterns, container image)
- Depends on: `.env` file (read selectively, never loaded into env)
- Used by: Everything
- Purpose: Interactive CLI for initial configuration (timezone, credentials, container build, group registration, launchd service)
- Location: `setup/`
- Contains: Step-based CLI runner, platform detection, service management
- Depends on: `src/logger.ts`, `src/config.ts`
- Used by: `npm run setup`
## Data Flow
- Router state persisted in SQLite `router_state` table (last seen timestamps)
- Sessions (Claude conversation IDs) persisted in SQLite `sessions` table, keyed by group folder
- Registered groups persisted in SQLite `registered_groups` table
- In-memory caches (`lastTimestamp`, `sessions`, `registeredGroups`, `lastAgentTimestamp`) loaded at startup via `loadState()`
- Cursor recovery: if `lastAgentTimestamp` is missing for a group, recovers from last bot message timestamp in DB
## Key Abstractions
- Purpose: Represents a messaging platform connection
- Examples: `src/types.ts` (interface), `src/channels/registry.ts` (factory pattern)
- Pattern: Factory registration — each channel module calls `registerChannel(name, factory)` on import. Factory returns `Channel | null` (null if credentials missing). Barrel file `src/channels/index.ts` triggers imports.
- Purpose: A chat/group that the bot is actively monitoring and responding to
- Examples: `src/types.ts` (interface), `src/db.ts` (persistence)
- Pattern: Groups are identified by JID (platform-specific chat ID). Each group has a folder name, trigger pattern, optional container config with additional mounts, and an `isMain` flag for the control group.
- Purpose: Serializes work per group while allowing concurrency across groups
- Examples: `src/group-queue.ts`
- Pattern: State machine per group (`active`, `idleWaiting`, `pendingMessages`, `pendingTasks`). Limits total active containers via `MAX_CONCURRENT_CONTAINERS`. Drains pending work after each container completes.
- Purpose: Structured protocol between host and container agent
- Examples: `src/container-runner.ts`, `container/agent-runner/src/index.ts`
- Pattern: JSON over stdin (input) and stdout with sentinel markers (output). Markers are `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---`.
- Purpose: Communication between container agents and host process
- Examples: `src/ipc.ts`, `container/agent-runner/src/ipc-mcp-stdio.ts`
- Pattern: Atomic write (write to `.tmp`, rename to `.json`). Per-group namespacing prevents cross-group access. Authorization verified by directory identity, not file contents.
## Entry Points
- Location: `src/index.ts`
- Triggers: `npm run dev` (tsx) or `npm start` (compiled)
- Responsibilities: Ensures container runtime, initializes database, loads state, connects channels, starts message loop, IPC watcher, and task scheduler
- Location: `setup/index.ts`
- Triggers: `npm run setup` → `tsx setup/index.ts --step <name>`
- Responsibilities: Step-based configuration (timezone, environment, container, groups, register, mounts, service, verify)
- Location: `container/agent-runner/src/index.ts`
- Triggers: Docker container entrypoint reads stdin
- Responsibilities: Parses input, runs Claude Agent SDK query loop, streams results, polls IPC for follow-ups
- Location: `container/agent-runner/src/ipc-mcp-stdio.ts`
- Triggers: Spawned by agent-runner as MCP server for Claude Agent SDK
- Responsibilities: Provides tools (send_message, schedule_task, list_tasks, register_group, etc.) that write IPC files
## Error Handling
- Container errors trigger cursor rollback so messages can be re-processed on retry (unless output was already sent to user, to prevent duplicates)
- `GroupQueue` implements exponential backoff (5s base, max 5 retries) for failed group processing
- Stale/corrupt session detection: if container fails with "no conversation found" or ENOENT on .jsonl, session ID is cleared for fresh retry
- IPC processing errors move files to `data/ipc/errors/` directory instead of deleting
- Container timeout distinguishes "timeout after output" (success, idle cleanup) from "timeout with no output" (error)
- Startup recovery via `recoverPendingMessages()` checks for unprocessed messages across all registered groups
## Cross-Cutting Concerns
- _Sender allowlist_ (`~/.config/nanoclaw/sender-allowlist.json`): controls which senders can trigger the bot per chat, with trigger-only or drop modes
- _Group privilege model_: main group has elevated privileges (register groups, refresh metadata, schedule cross-group tasks). Non-main groups can only operate within their own scope. `isMain` cannot be set via IPC.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

| Skill | Description | Path |
|-------|-------------|------|
| add-compact | Add /compact command for manual context compaction. Solves context rot in long sessions by forwarding the SDK's built-in /compact slash command. Main-group or trusted sender only. | `.claude/skills/add-compact/SKILL.md` |
| add-discord | Add Discord bot channel integration to NanoClaw. | `.claude/skills/add-discord/SKILL.md` |
| add-emacs | Add Emacs as a channel. Opens an interactive chat buffer and org-mode integration so you can talk to NanoClaw from within Emacs (Doom, Spacemacs, or vanilla). Uses a local HTTP bridge — no bot token or external service needed. | `.claude/skills/add-emacs/SKILL.md` |
| add-gmail | Add Gmail integration to NanoClaw. Can be configured as a tool (agent reads/sends emails when triggered from WhatsApp) or as a full channel (emails can trigger the agent, schedule tasks, and receive replies). Guides through GCP OAuth setup and implements the integration. | `.claude/skills/add-gmail/SKILL.md` |
| add-image-vision | Add image vision to NanoClaw agents. Resizes and processes WhatsApp image attachments, then sends them to Claude as multimodal content blocks. | `.claude/skills/add-image-vision/SKILL.md` |
| add-macos-statusbar | Add a macOS menu bar status indicator for NanoClaw. Shows a bolt icon with a green/red dot indicating whether NanoClaw is running, with Start, Stop, and Restart controls. macOS only. | `.claude/skills/add-macos-statusbar/SKILL.md` |
| add-ollama-tool | Add Ollama MCP server so the container agent can call local models and optionally manage the Ollama model library. | `.claude/skills/add-ollama-tool/SKILL.md` |
| add-parallel |  | `.claude/skills/add-parallel/SKILL.md` |
| add-pdf-reader | Add PDF reading to NanoClaw agents. Extracts text from PDFs via pdftotext CLI. Handles WhatsApp attachments, URLs, and local files. | `.claude/skills/add-pdf-reader/SKILL.md` |
| add-reactions | Add WhatsApp emoji reaction support — receive, send, store, and search reactions. | `.claude/skills/add-reactions/SKILL.md` |
| add-slack | Add Slack as a channel. Can replace WhatsApp entirely or run alongside it. Uses Socket Mode (no public URL needed). | `.claude/skills/add-slack/SKILL.md` |
| add-telegram | Add Telegram as a channel. Can replace WhatsApp entirely or run alongside it. Also configurable as a control-only channel (triggers actions) or passive channel (receives notifications only). | `.claude/skills/add-telegram/SKILL.md` |
| add-telegram-swarm | Add Agent Swarm (Teams) support to Telegram. Each subagent gets its own bot identity in the group. Requires Telegram channel to be set up first (use /add-telegram). Triggers on "agent swarm", "agent teams telegram", "telegram swarm", "bot pool". | `.claude/skills/add-telegram-swarm/SKILL.md` |
| add-voice-transcription | Add voice message transcription to NanoClaw using OpenAI's Whisper API. Automatically transcribes WhatsApp voice notes so the agent can read and respond to them. | `.claude/skills/add-voice-transcription/SKILL.md` |
| add-whatsapp | Add WhatsApp as a channel. Can replace other channels entirely or run alongside them. Uses QR code or pairing code for authentication. | `.claude/skills/add-whatsapp/SKILL.md` |
| channel-formatting | Convert Claude's Markdown output to each channel's native text syntax before delivery. Adds zero-dependency formatting for WhatsApp, Telegram, and Slack (marker substitution). Also ships a Signal rich-text helper (parseSignalStyles) used by the Signal skill. | `.claude/skills/channel-formatting/SKILL.md` |
| claw | Install the claw CLI tool — run NanoClaw agent containers from the command line without opening a chat app. | `.claude/skills/claw/SKILL.md` |
| convert-to-apple-container | Switch from Docker to Apple Container for macOS-native container isolation. Use when the user wants Apple Container instead of Docker, or is setting up on macOS and prefers the native runtime. Triggers on "apple container", "convert to apple container", "switch to apple container", or "use apple container". | `.claude/skills/convert-to-apple-container/SKILL.md` |
| customize | Add new capabilities or modify NanoClaw behavior. Use when user wants to add channels (Telegram, Slack, email input), change triggers, add integrations, modify the router, or make any other customizations. This is an interactive skill that asks questions to understand what the user wants. | `.claude/skills/customize/SKILL.md` |
| debug | Debug container agent issues. Use when things aren't working, container fails, authentication problems, or to understand how the container system works. Covers logs, environment variables, mounts, and common issues. | `.claude/skills/debug/SKILL.md` |
| get-qodo-rules | "Loads org- and repo-level coding rules from Qodo before code tasks begin, ensuring all generation and modification follows team standards. Use before any code generation or modification task when rules are not already loaded. Invoke when user asks to write, edit, refactor, or review code, or when starting implementation planning." | `.claude/skills/get-qodo-rules/SKILL.md` |
| init-onecli | Install and initialize OneCLI Agent Vault. Migrates existing .env credentials to the vault. Use after /update-nanoclaw brings in OneCLI as a breaking change, or for first-time OneCLI setup. | `.claude/skills/init-onecli/SKILL.md` |
| qodo-pr-resolver | Review and resolve PR issues with Qodo - get AI-powered code review issues and fix them interactively (GitHub, GitLab, Bitbucket, Azure DevOps) | `.claude/skills/qodo-pr-resolver/SKILL.md` |
| setup | Run initial NanoClaw setup. Use when user wants to install dependencies, authenticate messaging channels, register their main channel, or start the background services. Triggers on "setup", "install", "configure nanoclaw", or first-time setup requests. | `.claude/skills/setup/SKILL.md` |
| update-nanoclaw | Efficiently bring upstream NanoClaw updates into a customized install, with preview, selective cherry-pick, and low token usage. | `.claude/skills/update-nanoclaw/SKILL.md` |
| update-skills | Check for and apply updates to installed skill branches from upstream. | `.claude/skills/update-skills/SKILL.md` |
| use-local-whisper | Use when the user wants local voice transcription instead of OpenAI Whisper API. Switches to whisper.cpp running on Apple Silicon. WhatsApp only for now. Requires voice-transcription skill to be applied first. | `.claude/skills/use-local-whisper/SKILL.md` |
| use-native-credential-proxy | Replace OneCLI gateway with the built-in credential proxy. For users who want simple .env-based credential management without installing OneCLI. Reads API key or OAuth token from .env and injects into container API requests. | `.claude/skills/use-native-credential-proxy/SKILL.md` |
| x-integration | X (Twitter) integration for NanoClaw. Post tweets, like, reply, retweet, and quote. Use for setup, testing, or troubleshooting X functionality. Triggers on "setup x", "x integration", "twitter", "post tweet", "tweet". | `.claude/skills/x-integration/SKILL.md` |
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->

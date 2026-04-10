# Codebase Structure

**Analysis Date:** 2026-04-10

## Directory Layout

```
nanoclaw/
├── src/                    # Host application source (TypeScript)
│   ├── channels/           # Channel abstraction and registry
│   │   ├── index.ts        # Barrel import (triggers channel self-registration)
│   │   └── registry.ts     # Channel factory registry
│   ├── index.ts            # Main entry point (message loop, startup)
│   ├── router.ts           # Message formatting and channel routing
│   ├── config.ts           # Configuration constants
│   ├── env.ts              # Selective .env reader
│   ├── types.ts            # Shared TypeScript interfaces
│   ├── db.ts               # SQLite database (schema, CRUD, migrations)
│   ├── container-runner.ts # Docker container spawning and output parsing
│   ├── container-runtime.ts# Container runtime abstraction (docker binary)
│   ├── group-queue.ts      # Per-group concurrency and work queue
│   ├── group-folder.ts     # Group folder validation and path resolution
│   ├── ipc.ts              # Host-side IPC file watcher
│   ├── task-scheduler.ts   # Scheduled task execution loop
│   ├── mount-security.ts   # Additional mount validation against allowlist
│   ├── sender-allowlist.ts # Per-chat sender filtering
│   ├── remote-control.ts   # Claude remote control session management
│   ├── timezone.ts         # IANA timezone validation and formatting
│   ├── logger.ts           # Structured logger (pino-like API)
│   └── *.test.ts           # Co-located test files
├── container/              # Docker container definition
│   ├── Dockerfile          # Container image (node:22-slim + Chromium)
│   ├── build.sh            # Container build script
│   ├── agent-runner/       # Code that runs INSIDE the container
│   │   ├── src/
│   │   │   ├── index.ts    # Agent runner (SDK query loop, IPC polling)
│   │   │   └── ipc-mcp-stdio.ts # MCP server (tools for the agent)
│   │   ├── package.json    # Separate dependency tree (SDK, MCP, zod)
│   │   └── tsconfig.json   # Container TypeScript config
│   └── skills/             # Skills synced into container .claude/skills/
│       ├── agent-browser/  # Browser automation skill
│       ├── capabilities/   # General capabilities skill
│       ├── slack-formatting/# Slack formatting skill
│       └── status/         # Status reporting skill
├── setup/                  # Setup CLI (interactive configuration)
│   ├── index.ts            # CLI entry point (--step <name>)
│   ├── timezone.ts         # Timezone configuration step
│   ├── environment.ts      # Environment/credentials step
│   ├── container.ts        # Container build step
│   ├── groups.ts           # Group directory setup step
│   ├── register.ts         # Channel registration step
│   ├── mounts.ts           # Mount allowlist setup step
│   ├── service.ts          # launchd service setup step
│   ├── verify.ts           # Verification step
│   ├── platform.ts         # Platform detection utilities
│   ├── status.ts           # Status emitter for setup steps
│   └── *.test.ts           # Co-located test files
├── groups/                 # CLAUDE.md templates for agent identity
│   ├── main/               # Template for main control group
│   │   └── CLAUDE.md       # Main group agent instructions
│   └── global/             # Template for non-main groups
│       └── CLAUDE.md       # Global agent instructions (read-only in container)
├── scripts/                # Utility scripts
│   └── run-migrations.ts   # Database migration runner
├── launchd/                # macOS service definition
│   └── com.nanoclaw.plist  # launchd plist for background service
├── config-examples/        # Example configuration files
│   └── mount-allowlist.json# Example mount allowlist
├── docs/                   # Documentation
│   ├── SPEC.md             # System specification
│   ├── SECURITY.md         # Security model documentation
│   ├── SDK_DEEP_DIVE.md    # Claude Agent SDK deep dive
│   └── ...                 # Other docs
├── assets/                 # Static assets (images, etc.)
├── repo-tokens/            # Repository token examples
├── .claude/                # Claude Code configuration
│   ├── settings.json       # Claude Code settings
│   └── skills/             # Host-side skills (setup, customization)
├── package.json            # Host dependencies and scripts
├── tsconfig.json           # Host TypeScript config
├── vitest.config.ts        # Test config (main)
├── vitest.skills.config.ts # Test config (skills)
├── eslint.config.js        # ESLint config
├── .prettierrc             # Prettier config
├── .nvmrc                  # Node version (nvm)
├── .env.example            # Environment variable template
├── CLAUDE.md               # Project-level Claude instructions
├── setup.sh                # Shell wrapper for setup CLI
└── .gitignore              # Git ignore rules
```

## Directory Purposes

**`src/`:**

- Purpose: All host-side TypeScript application code
- Contains: Core modules, channel adapters, database, container management
- Key files: `index.ts` (entry point), `db.ts` (data layer), `container-runner.ts` (execution), `group-queue.ts` (concurrency)

**`src/channels/`:**

- Purpose: Channel abstraction layer for messaging platforms
- Contains: Registry pattern (`registry.ts`), barrel import (`index.ts`). Individual channel implementations are added by skills (e.g., add-whatsapp, add-telegram) and self-register on import.
- Key files: `registry.ts` (factory registration), `index.ts` (import triggers)

**`container/`:**

- Purpose: Everything needed to build and run the Docker container for agent execution
- Contains: Dockerfile, agent-runner source (separate npm project), container skills
- Key files: `Dockerfile`, `agent-runner/src/index.ts`, `agent-runner/src/ipc-mcp-stdio.ts`

**`container/agent-runner/`:**

- Purpose: Separate Node.js project that runs INSIDE containers. Has its own `package.json` and `tsconfig.json`.
- Contains: Agent runner (query loop), MCP server (tools), dependencies on Claude Agent SDK and MCP SDK

**`container/skills/`:**

- Purpose: Skills that get synced into each group's `.claude/skills/` directory before container launch
- Contains: Agent-side skill definitions (browser, capabilities, formatting, status)
- Key files: Each subdirectory is a skill with its own SKILL.md and supporting files

**`setup/`:**

- Purpose: Interactive CLI for initial NanoClaw configuration
- Contains: Step-based modules (timezone, environment, container, groups, register, mounts, service, verify)
- Key files: `index.ts` (CLI dispatcher), step modules

**`groups/`:**

- Purpose: CLAUDE.md templates that define agent identity and instructions
- Contains: `main/CLAUDE.md` (main control group template), `global/CLAUDE.md` (non-main group template)
- Key files: Templates are copied into each group's folder at registration time

**`launchd/`:**

- Purpose: macOS service definition for running NanoClaw as a background daemon
- Contains: `com.nanoclaw.plist` template

**`config-examples/`:**

- Purpose: Example configuration files for user reference
- Contains: `mount-allowlist.json` (example mount security config)

## Key File Locations

**Entry Points:**

- `src/index.ts`: Main application entry point (message loop, startup orchestration)
- `setup/index.ts`: Setup CLI entry point (`--step <name>`)
- `container/agent-runner/src/index.ts`: Container-side agent runner
- `container/agent-runner/src/ipc-mcp-stdio.ts`: Container-side MCP tool server

**Configuration:**

- `src/config.ts`: All runtime constants (paths, timeouts, trigger patterns)
- `src/env.ts`: Selective `.env` file reader
- `.env` (gitignored): Runtime secrets and configuration
- `.env.example`: Template showing required environment variables
- `~/.config/nanoclaw/mount-allowlist.json`: External mount security allowlist (not in repo)
- `~/.config/nanoclaw/sender-allowlist.json`: External sender filtering config (not in repo)

**Core Logic:**

- `src/container-runner.ts`: Container spawning, volume mounts, output streaming
- `src/container-runtime.ts`: Container runtime abstraction (currently Docker)
- `src/group-queue.ts`: Concurrency management and work queuing
- `src/db.ts`: SQLite database schema, migrations, and all CRUD operations
- `src/ipc.ts`: Host-side IPC file watcher (messages, tasks, group registration)
- `src/router.ts`: Message formatting (XML) and channel routing
- `src/task-scheduler.ts`: Scheduled task polling and execution

**Security:**

- `src/mount-security.ts`: Additional mount validation against external allowlist
- `src/sender-allowlist.ts`: Per-chat sender filtering
- `src/group-folder.ts`: Group folder name validation and path traversal prevention

**Testing:**

- `src/*.test.ts`: Co-located test files alongside source modules
- `setup/*.test.ts`: Setup step tests
- `vitest.config.ts`: Main test configuration
- `vitest.skills.config.ts`: Skills-specific test configuration

## Naming Conventions

**Files:**

- `kebab-case.ts`: All TypeScript source files use kebab-case (e.g., `container-runner.ts`, `group-queue.ts`)
- `*.test.ts`: Test files co-located with source, same name + `.test` suffix
- `UPPERCASE.md`: Documentation files (e.g., `CLAUDE.md`, `SPEC.md`)

**Directories:**

- `kebab-case/`: All directories use kebab-case (e.g., `agent-runner/`, `config-examples/`)
- Exception: `CLAUDE.md` files inside group directories follow the SDK convention

**Exports:**

- Named exports only (no default exports)
- `_prefixed` functions exported for testing only (e.g., `_initTestDatabase()`, `_resetForTesting()`)

## Where to Add New Code

**New Channel (e.g., Signal, Matrix):**

- Implementation: `src/channels/{channel-name}.ts` — must call `registerChannel()` from `registry.ts`
- Registration: Add import line to `src/channels/index.ts` barrel file
- Tests: `src/channels/{channel-name}.test.ts`
- Skill: `.claude/skills/add-{channel-name}/` for the installation guide

**New MCP Tool (agent capability):**

- Implementation: Add `server.tool()` call in `container/agent-runner/src/ipc-mcp-stdio.ts`
- Host-side handler: Add case to `processTaskIpc()` in `src/ipc.ts` (if tool needs host-side processing)

**New Host Module:**

- Primary code: `src/{module-name}.ts`
- Tests: `src/{module-name}.test.ts`
- Wire into `src/index.ts` if it needs startup initialization

**New Setup Step:**

- Implementation: `setup/{step-name}.ts` — export a `run(args: string[])` function
- Registration: Add to `STEPS` record in `setup/index.ts`
- Tests: `setup/{step-name}.test.ts`

**New Container Skill:**

- Implementation: `container/skills/{skill-name}/` directory with SKILL.md
- Auto-synced: Skills are copied to each group's `.claude/skills/` by `container-runner.ts`

**New Utility:**

- Shared helpers: `src/{utility-name}.ts`
- If only used in one module, keep it as a local function in that module

**New Scheduled Task Type:**

- Schedule types are defined in `src/types.ts` (`ScheduledTask.schedule_type`)
- Execution logic in `src/task-scheduler.ts` (`computeNextRun()`)
- MCP tool validation in `container/agent-runner/src/ipc-mcp-stdio.ts`
- IPC handling in `src/ipc.ts` (`processTaskIpc()`)

## Special Directories

**`store/` (runtime, gitignored):**

- Purpose: SQLite database (`messages.db`)
- Generated: Yes, at runtime
- Committed: No

**`data/` (runtime, gitignored):**

- Purpose: IPC files, sessions, remote control state
- Subdirectories: `ipc/{group}/messages/`, `ipc/{group}/tasks/`, `ipc/{group}/input/`, `sessions/{group}/`
- Generated: Yes, at runtime
- Committed: No

**`groups/{group-folder}/` (runtime, partially committed):**

- Purpose: Per-group working directory mounted into containers at `/workspace/group`
- Contains: `CLAUDE.md` (agent identity), `logs/` (container logs), `conversations/` (archived transcripts), and any files the agent creates
- Generated: Templates committed (`groups/main/`, `groups/global/`); runtime group folders are gitignored
- Committed: Only templates

**`dist/` (build output, gitignored):**

- Purpose: Compiled JavaScript from TypeScript
- Generated: Yes, by `npm run build`
- Committed: No

**`node_modules/` (dependencies, gitignored):**

- Purpose: npm dependencies
- Generated: Yes, by `npm install`
- Committed: No

**`~/.config/nanoclaw/` (external config, not in repo):**

- Purpose: Security configuration stored outside the project root (tamper-proof from containers)
- Contains: `mount-allowlist.json`, `sender-allowlist.json`
- Generated: By setup or manually
- Committed: Never (external to repo)

---

_Structure analysis: 2026-04-10_

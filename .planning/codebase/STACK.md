# Technology Stack

**Analysis Date:** 2026-04-10

## Languages

**Primary:**

- TypeScript 5.7+ - All host-side source (`src/`), agent-runner (`container/agent-runner/src/`), and setup scripts (`setup/`)

**Secondary:**

- Bash - Container build script (`container/build.sh`), setup entrypoint (`setup.sh`), container entrypoint (inline in `container/Dockerfile`)
- XML (plist) - macOS launchd service definition (`launchd/com.nanoclaw.plist`)

## Runtime

**Environment:**

- Node.js 22 (pinned in `.nvmrc`, `container/Dockerfile` uses `node:22-slim`)
- Minimum Node.js 20 (declared in `package.json` engines)

**Package Manager:**

- npm (lockfile: `package-lock.json` present)
- Two separate package roots:
  - Root: `/package.json` - Host application
  - Container: `/container/agent-runner/package.json` - Agent runner inside Docker

## Frameworks

**Core:**

- No web framework - NanoClaw is a message-processing daemon, not a web server
- `@anthropic-ai/claude-agent-sdk` ^0.2.76 - Claude Code Agent SDK used inside containers (`container/agent-runner/src/index.ts`)
- `@modelcontextprotocol/sdk` ^1.12.1 - MCP server for container-side IPC tools (`container/agent-runner/src/ipc-mcp-stdio.ts`)
- `@onecli-sh/sdk` ^0.2.0 - OneCLI credential gateway SDK for secure API key injection (`src/index.ts`, `src/container-runner.ts`)

**Testing:**

- Vitest ^4.0.18 - Test runner
  - Config: `vitest.config.ts` (main tests in `src/` and `setup/`)
  - Config: `vitest.skills.config.ts` (skill tests in `.claude/skills/**/tests/`)

**Build/Dev:**

- TypeScript ^5.7.0 - Compiler (`tsc`)
- tsx ^4.19.0 - Dev-time TypeScript execution (`npm run dev`)
- Prettier ^3.8.1 - Code formatting
- ESLint ^9.35.0 - Linting with flat config
- Husky ^9.1.7 - Git hooks (pre-commit runs `npm run format:fix`)

## Key Dependencies

**Critical (Host):**

- `better-sqlite3` 11.10.0 - Embedded SQLite database for messages, tasks, sessions, router state (`src/db.ts`)
- `cron-parser` 5.5.0 - Parses cron expressions for scheduled tasks (`src/ipc.ts`, `src/task-scheduler.ts`)
- `@onecli-sh/sdk` ^0.2.0 - Secure credential proxy for container API access (`src/container-runner.ts`)

**Critical (Container):**

- `@anthropic-ai/claude-agent-sdk` ^0.2.76 - Runs Claude Code inside containers (`container/agent-runner/src/index.ts`)
- `@modelcontextprotocol/sdk` ^1.12.1 - MCP stdio server exposing NanoClaw tools to the agent (`container/agent-runner/src/ipc-mcp-stdio.ts`)
- `zod` ^4.0.0 - Schema validation for MCP tool parameters (`container/agent-runner/src/ipc-mcp-stdio.ts`)
- `cron-parser` ^5.0.0 - Validates cron expressions in-container (`container/agent-runner/src/ipc-mcp-stdio.ts`)

**Infrastructure:**

- Docker - Container runtime for agent isolation (`src/container-runtime.ts`, `container/Dockerfile`)
- Chromium (inside container) - Browser automation via `agent-browser` global npm package

## TypeScript Configuration

**Compilation Target:** ES2022
**Module System:** NodeNext (ESM with `.js` extensions in imports)
**Strict Mode:** Enabled
**Source Maps:** Enabled (host), not enabled (container)
**Declaration Files:** Generated for both host and container

**Key tsconfig settings** (`tsconfig.json`):

- `"type": "module"` in both `package.json` files (ESM throughout)
- `resolveJsonModule: true` (host only)
- `declarationMap: true` (host only)

## Configuration

**Environment:**

- `.env` file in project root, read by custom parser in `src/env.ts`
- Custom env reader does NOT load into `process.env` (secrets stay out of child process environment)
- Keys read from `.env`: `ASSISTANT_NAME`, `ASSISTANT_HAS_OWN_NUMBER`, `ONECLI_URL`, `TZ`
- Process env vars for runtime tuning: `CONTAINER_IMAGE`, `CONTAINER_TIMEOUT`, `CONTAINER_MAX_OUTPUT_SIZE`, `MAX_MESSAGES_PER_PROMPT`, `IDLE_TIMEOUT`, `MAX_CONCURRENT_CONTAINERS`, `LOG_LEVEL`

**Security Configs (stored outside project):**

- `~/.config/nanoclaw/mount-allowlist.json` - Controls which host directories containers can mount
- `~/.config/nanoclaw/sender-allowlist.json` - Controls which message senders can trigger the agent

**Build:**

- `tsconfig.json` - Host TypeScript config
- `container/agent-runner/tsconfig.json` - Container TypeScript config
- `eslint.config.js` - Flat ESLint config (ignores `container/` and `groups/`)
- `.prettierrc` - Single setting: `singleQuote: true`

## Platform Requirements

**Development:**

- Node.js >= 20 (22 recommended via `.nvmrc`)
- Docker installed and running (verified at startup by `src/container-runtime.ts`)
- npm for dependency management

**Production:**

- macOS primary target (launchd plist in `launchd/com.nanoclaw.plist`)
- Linux supported (host gateway handling in `src/container-runtime.ts`)
- Container image `nanoclaw-agent:latest` must be built first (`container/build.sh`)
- OneCLI gateway at `http://localhost:10254` for credential injection (configurable via `ONECLI_URL`)

**Container Image:**

- Base: `node:22-slim`
- Includes: Chromium, git, curl, CJK/emoji fonts
- Global npm packages: `agent-browser`, `@anthropic-ai/claude-code`
- Runs as non-root `node` user (uid 1000)

---

_Stack analysis: 2026-04-10_

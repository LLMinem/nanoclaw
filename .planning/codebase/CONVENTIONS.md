# Coding Conventions

**Analysis Date:** 2026-04-10

## Naming Patterns

**Files:**

- Use `kebab-case.ts` for all source files: `container-runner.ts`, `group-queue.ts`, `sender-allowlist.ts`
- Test files are co-located with source, named `{module}.test.ts`: `db.test.ts`, `routing.test.ts`
- Entry points: `index.ts` in `src/` and `src/channels/`

**Functions:**

- Use `camelCase` for all functions: `formatMessages()`, `loadSenderAllowlist()`, `getOrRecoverCursor()`
- Prefix boolean-returning functions with `is`/`has`/`should`: `isValidTimezone()`, `shouldDropMessage()`, `isSenderAllowed()`
- Prefix internal test helpers with underscore: `_initTestDatabase()`, `_closeDatabase()`, `_setRegisteredGroups()`, `_resetSchedulerLoopForTests()`
- Annotate test-only exports with `/** @internal - for tests only */` JSDoc

**Variables:**

- Use `camelCase` for local variables and parameters: `lastTimestamp`, `missedMessages`, `registeredGroups`
- Use `UPPER_SNAKE_CASE` for module-level constants: `POLL_INTERVAL`, `MAX_CONCURRENT_CONTAINERS`, `DEFAULT_TRIGGER`, `BASE_RETRY_MS`
- Use `camelCase` for mutable module-level state: `lastTimestamp`, `sessions`, `messageLoopRunning`

**Types/Interfaces:**

- Use `PascalCase` for interfaces: `RegisteredGroup`, `NewMessage`, `ScheduledTask`, `Channel`
- Use `PascalCase` for type aliases: `OnInboundMessage`, `OnChatMetadata`, `ChannelFactory`
- Use `PascalCase` for classes: `GroupQueue`
- Define interfaces in `src/types.ts` for shared types; co-locate module-specific interfaces with their module

## Code Style

**Formatting:**

- Prettier with single quotes (`singleQuote: true`)
- All other Prettier settings are defaults (2-space indent, 80 char print width, trailing commas, semicolons)
- Pre-commit hook runs `npm run format:fix` via Husky

**Linting:**

- ESLint 9 with flat config (`eslint.config.js`)
- TypeScript-ESLint recommended rules
- `no-catch-all` plugin warns against generic catch blocks
- Unused variables must be prefixed with `_`: `argsIgnorePattern: '^_'`, `varsIgnorePattern: '^_'`
- `@typescript-eslint/no-explicit-any` set to warn (not error)
- Scope: `src/**/*.{js,ts}` only; `container/`, `groups/`, `dist/` are excluded

**TypeScript:**

- Strict mode enabled (`"strict": true` in `tsconfig.json`)
- Target: ES2022, Module: NodeNext
- ESM modules (`"type": "module"` in package.json)
- All imports use `.js` extension: `import { logger } from './logger.js'`

## Import Organization

**Order:**

1. Node.js built-in modules (`fs`, `path`, `os`, `child_process`)
2. Third-party packages (`better-sqlite3`, `@onecli-sh/sdk`, `cron-parser`)
3. Internal modules with `.js` extension (`./config.js`, `./logger.js`, `./types.js`)

**Blank line between each group.** Example from `src/index.ts`:

```typescript
import fs from 'fs';
import path from 'path';

import { OneCLI } from '@onecli-sh/sdk';

import { ASSISTANT_NAME, DEFAULT_TRIGGER, ... } from './config.js';
import { GroupQueue } from './group-queue.js';
```

**Path Aliases:**

- None used. All imports are relative paths with `.js` extensions.

## Error Handling

**Patterns:**

- Use structured logging with `logger` for all error reporting (never `console.log` or `console.error`)
- Catch blocks must name the error parameter (enforced by `preserve-caught-error` lint rule) or use empty catch
- Empty catch blocks are used for "expected to fail" operations like DB migrations: `try { db.exec('ALTER TABLE ...') } catch { /* column already exists */ }`
- Functions return `'success' | 'error'` string literals rather than throwing for recoverable operations (see `runAgent()` in `src/index.ts`)
- Security-sensitive operations fail closed: `loadMountAllowlist()` returns `null` on error → mounts are blocked
- Stale session detection uses regex matching against error messages, then clears the session for retry

**Validation approach:**

- Validate early and return/skip invalid data with a warning log
- Use type guard functions for runtime validation: `isValidEntry()`, `isValidGroupFolder()`, `isValidTimezone()`
- Parameterized SQL queries throughout — never string interpolation for DB queries

## Logging

**Framework:** Custom lightweight logger (`src/logger.ts`) with structured output

**Patterns:**

- Two calling signatures: `logger.info('simple message')` or `logger.info({ key: 'value' }, 'message with context')`
- Log levels: `debug`, `info`, `warn`, `error`, `fatal`
- Threshold configured via `LOG_LEVEL` env var (default: `info`)
- `warn` and above write to `stderr`; `debug` and `info` to `stdout`
- Uncaught exceptions and unhandled rejections are routed through the logger
- Use `{ err }` key for Error objects — the logger formats stack traces specially

**When to log:**

- `debug`: Detailed operational flow (message piping, IPC, cache hits/skips)
- `info`: State transitions, group registration, startup, message counts
- `warn`: Recoverable issues (missing credentials, invalid config, rejected operations)
- `error`: Failed operations that need attention (container failures, agent errors)
- `fatal`: Unrecoverable startup failures, uncaught exceptions (followed by `process.exit(1)`)

## Comments

**When to Comment:**

- Use comments for non-obvious "why" decisions, not "what" the code does
- Reference issue numbers for fixes: `// Fixes #1391`
- Use inline comments for security rationale: `// Mount security: allowlist stored OUTSIDE project root, never mounted into containers`
- Block comments explain migration logic or workarounds for edge cases
- Empty catch blocks must have a comment explaining why: `/* column already exists */`

**JSDoc/TSDoc:**

- Use `/** @internal */` to mark test-only exports
- Full JSDoc only on key public functions and complex helpers
- Inline `//` comments preferred over JSDoc for implementation notes

## Function Design

**Size:**

- Functions are generally short (10-40 lines). Complex orchestration functions (`startMessageLoop`, `processGroupMessages`) are longer but follow a clear sequential flow.

**Parameters:**

- Use interfaces/types for complex parameter sets rather than positional params
- Callback-based dependency injection for subsystem integration (see `startSchedulerLoop`, `startIpcWatcher` in `src/index.ts`)
- Group related config into objects: `ChannelOpts`, `IpcDeps`

**Return Values:**

- Functions return explicit types; avoid `any`
- Use `| undefined` for optional returns rather than `null` (except for DB/SQL operations where `null` is idiomatic)
- Async functions that can fail return result types (`'success' | 'error'`) or use boolean return

## Module Design

**Exports:**

- Named exports only — no default exports anywhere in the codebase
- Re-exports for backwards compatibility: `export { escapeXml, formatMessages } from './router.js'`
- Test-only exports prefixed with `_` and annotated `@internal`

**Barrel Files:**

- `src/channels/index.ts` is a barrel that triggers channel self-registration via side-effect imports
- No other barrel files; modules import directly from their target

**Module Boundaries:**

- Config values are centralized in `src/config.ts` — import constants from there, not from `.env` directly
- Database operations are centralized in `src/db.ts` — never create `Database` instances elsewhere in production code
- Logger is a singleton exported from `src/logger.ts`

## Security Conventions

- Never pass secrets via environment variables to child processes — use `readEnvFile()` to read selectively
- Container mount paths are validated against an external allowlist (`~/.config/nanoclaw/mount-allowlist.json`)
- Shell commands validate inputs before interpolation: `stopContainer()` rejects names with shell metacharacters
- Group folder names are validated with `isValidGroupFolder()` to prevent path traversal
- Sender authorization is checked at multiple layers (trigger, drop, IPC)

---

_Convention analysis: 2026-04-10_

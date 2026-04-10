# Codebase Concerns

**Analysis Date:** 2026-04-10

## Tech Debt

**Module-level mutable state scattered across files:**

- Issue: Multiple files use module-level `let` variables to hold runtime state (`lastTimestamp`, `sessions`, `registeredGroups`, `messageLoopRunning`, `ipcWatcherRunning`, `schedulerRunning`, `activeSession`, `cachedAllowlist`). This makes testing difficult and requires `_resetForTesting()` escape hatches.
- Files: `src/index.ts` (lines 71-78), `src/ipc.ts` (line 28), `src/task-scheduler.ts` (line 243), `src/remote-control.ts` (line 16), `src/mount-security.ts` (lines 17-18), `src/sender-allowlist.ts` (no cache, re-reads every call)
- Impact: Race conditions during concurrent access; test isolation requires manual reset helpers; harder to reason about system state.
- Fix approach: Consolidate state into a `NanoClawApp` class or a `createApp()` factory that returns all state + methods. Pass state through dependency injection rather than relying on module singletons.

**Inline SQL schema migrations using try/catch ALTER TABLE:**

- Issue: Database migrations are implemented as individual `ALTER TABLE ... ADD COLUMN` statements wrapped in try/catch blocks that swallow errors if the column already exists. This approach doesn't track migration versions and will become harder to manage as the schema evolves.
- Files: `src/db.ts` (lines 87-163)
- Impact: No way to know which migrations have run; adding complex migrations (data transforms, index changes) is error-prone; silent failures could mask real errors.
- Fix approach: Implement a `schema_version` table and a numbered migration system. Each migration runs once and is recorded. The `scripts/run-migrations.ts` file already exists but is not used for this purpose.

**`stopContainer` uses `execSync` with string interpolation:**

- Issue: `stopContainer()` in `src/container-runtime.ts` uses `execSync(\`${CONTAINER_RUNTIME_BIN} stop -t 1 ${name}\`)`with template string concatenation rather than`execFileSync`(which avoids shell parsing). There is a name validation regex guard, but using`execFileSync` directly would be defense-in-depth.
- Files: `src/container-runtime.ts` (line 35)
- Impact: Low risk due to the regex guard on line 32, but violates the principle of least privilege for shell access.
- Fix approach: Replace `execSync` with `execFileSync(CONTAINER_RUNTIME_BIN, ['stop', '-t', '1', name])`. The comment on line 30 says "Uses execFileSync to avoid shell injection" but the implementation uses `execSync`.

**Sender allowlist re-reads file on every message:**

- Issue: `loadSenderAllowlist()` reads and parses the JSON file from disk on every invocation. In a high-traffic group, this means reading and parsing a file for every incoming message.
- Files: `src/sender-allowlist.ts` (lines 33-89), called from `src/index.ts` (lines 244, 494)
- Impact: Unnecessary disk I/O on every message. Not a performance crisis given message volume, but wasteful.
- Fix approach: Cache the parsed result with a TTL (e.g., 60 seconds) or watch the file for changes, similar to how `src/mount-security.ts` caches the mount allowlist.

**Backwards compatibility re-export in index.ts:**

- Issue: `src/index.ts` line 69 re-exports `escapeXml` and `formatMessages` from `router.ts` with a comment "Re-export for backwards compatibility during refactor."
- Files: `src/index.ts` (line 69)
- Impact: Minor clutter. Callers should import from `src/router.ts` directly.
- Fix approach: Search for imports from `src/index.ts` that use these symbols and update them, then remove the re-export.

## Known Bugs

**Telegram group metadata backfill defaults to `is_group: 0`:**

- Symptoms: Legacy Telegram chats migrated from before the `channel`/`is_group` columns were added get `is_group = 0` (direct message), even for group chats starting with `tg:-100`.
- Files: `src/db.ts` (lines 143-148), `src/db-migration.test.ts` (lines 49-56 — test confirms this behavior)
- Trigger: Running the migration on a DB that has Telegram group chats prefixed with `tg:-100`.
- Workaround: The test documents this as expected behavior (line 53 asserts `is_group: 0` for `tg:-10012345`), but Telegram groups start with `-100`. The migration backfill on line 145 sets all `tg:` entries to `is_group = 0`.

## Security Considerations

**Container agents run with `bypassPermissions` / `allowDangerouslySkipPermissions`:**

- Risk: Agents inside containers have full tool access without confirmation prompts. While containers provide isolation, a misconfigured mount could expose the host filesystem to unrestricted write access.
- Files: `container/agent-runner/src/index.ts` (lines 474-475)
- Current mitigation: Container sandboxing; mount allowlist at `~/.config/nanoclaw/mount-allowlist.json` (`src/mount-security.ts`); `.env` file shadowed with `/dev/null` mount (`src/container-runner.ts` lines 83-90); project root mounted read-only for main group (lines 75-79).
- Recommendations: The current defense-in-depth is solid. Ensure the mount allowlist file is always required in production (currently, missing allowlist blocks all additional mounts, which is the right default).

**IPC file-based communication has no authentication beyond filesystem permissions:**

- Risk: The IPC system uses JSON files written to `data/ipc/{groupFolder}/`. Authorization is based on which directory a file appears in (the group folder name). If a container can write to another group's IPC directory (via a misconfigured mount), it could escalate privileges.
- Files: `src/ipc.ts` (lines 62-148), `container/agent-runner/src/ipc-mcp-stdio.ts`
- Current mitigation: Each group gets its own IPC directory; directory-based identity is verified in `src/ipc.ts`; non-main groups can only send messages to their own chat JID (line 82-83).
- Recommendations: The current design is appropriate for single-user deployments. Document the trust model clearly.

**Remote control spawns `claude remote-control` with auto-accept:**

- Risk: The remote control feature auto-accepts the "Enable Remote Control?" prompt by piping `y\n` to stdin. The resulting URL provides full remote access to the codebase.
- Files: `src/remote-control.ts` (lines 123-127)
- Current mitigation: Only main group can trigger `/remote-control` (checked in `src/index.ts` line 601). Session state is persisted to disk for crash recovery.
- Recommendations: Consider adding a configurable allowlist of senders who can trigger remote control.

**Script execution in scheduled tasks:**

- Risk: Scheduled tasks can include arbitrary bash scripts that execute inside the container via `execFile('bash', [scriptPath])`.
- Files: `container/agent-runner/src/index.ts` (lines 554-601)
- Current mitigation: Scripts run inside the container sandbox with the same permissions as the agent. 30-second timeout (`SCRIPT_TIMEOUT_MS`). Container isolation prevents host access beyond mounted directories.
- Recommendations: Current mitigation is adequate given the container sandbox.

## Performance Bottlenecks

**Polling-based message loop:**

- Problem: The main message loop polls SQLite every 2 seconds (`POLL_INTERVAL`) for new messages across all registered groups. The IPC watcher polls the filesystem every 1 second (`IPC_POLL_INTERVAL`). The IPC input watcher inside the container polls every 500ms.
- Files: `src/index.ts` (line 539, `POLL_INTERVAL`), `src/ipc.ts` (line 150), `container/agent-runner/src/index.ts` (line 65, `IPC_POLL_MS`)
- Cause: No event-driven notification from channels to the router. SQLite doesn't support change notifications natively.
- Improvement path: For message routing, the channel could directly call `queue.enqueueMessageCheck()` on receipt instead of relying on polling. The polling serves as a crash-recovery fallback but shouldn't be the primary path. The IPC filesystem polling could use `fs.watch()` for immediate notification with polling as fallback.

**SQLite queries scan by timestamp on every poll:**

- Problem: `getNewMessages()` runs a query with `WHERE timestamp > ? AND chat_jid IN (...)` on every 2-second poll. For installations with many groups and high message volume, this creates continuous query load.
- Files: `src/db.ts` (lines 335-370)
- Cause: The `idx_timestamp` index helps, but the query still needs to check all messages newer than `lastTimestamp` across all groups.
- Improvement path: Maintain a per-group monotonic counter or use SQLite's `rowid` for cursor-based pagination instead of timestamp comparison. Or switch to event-driven routing (see above).

**Container startup overhead:**

- Problem: Each agent invocation spawns a new Docker container with multiple bind mounts, copies agent-runner source, syncs skills directories, and writes snapshot files.
- Files: `src/container-runner.ts` (lines 61-224, `buildVolumeMounts`), especially lines 152-206 (skill sync and agent-runner copy on every container start)
- Cause: File copying (`fs.cpSync`) happens on every container start to ensure freshness.
- Improvement path: The agent-runner copy already has a modification time check (lines 199-202). The skill sync could benefit from a similar check. Container reuse (keeping containers alive between messages) is already implemented via the IPC message piping system, which is the primary mitigation.

## Fragile Areas

**Container output parsing:**

- Files: `src/container-runner.ts` (lines 342-396), `container/agent-runner/src/index.ts` (lines 120-124)
- Why fragile: The host and container communicate via sentinel markers (`---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---`) embedded in stdout. If the agent's LLM output happens to contain these exact marker strings, parsing would break. The markers must be kept in sync between two separate codebases (host and container).
- Safe modification: Never change the marker strings in one file without the other. The constant is defined in both `src/container-runner.ts` (lines 34-35) and `container/agent-runner/src/index.ts` (lines 117-118).
- Test coverage: `src/container-runner.test.ts` tests timeout and output parsing behavior with fake processes.

**IPC file-based communication atomicity:**

- Files: `src/group-queue.ts` (lines 160-177), `src/ipc.ts` (lines 74-96), `container/agent-runner/src/ipc-mcp-stdio.ts` (lines 23-35)
- Why fragile: IPC uses atomic write (write to `.tmp` then `rename`) which is correct, but the consumer reads and then deletes with `fs.unlinkSync()`. If the process crashes between reading and deleting, the message could be processed twice on restart.
- Safe modification: The IPC message processing is idempotent for most operations (sending a message twice is annoying but not catastrophic). Task creation uses generated IDs so duplicates would create separate tasks.
- Test coverage: IPC authorization is well-tested in `src/ipc-auth.test.ts`. File-level atomicity is not unit-tested.

**Channel registration barrel file:**

- Files: `src/channels/index.ts`
- Why fragile: The barrel file is currently empty (all channel imports are commented out). Channels self-register via side-effect imports. If a channel skill adds an import line and it's later removed or reordered, the channel silently disappears.
- Safe modification: The empty barrel file is by design — channel skills add their import lines. Check this file after running skill installations.
- Test coverage: `src/channels/registry.test.ts` tests the registry mechanism but not the barrel import pattern.

**Message cursor state management:**

- Files: `src/index.ts` (lines 71-74, 122-137, 257-260)
- Why fragile: The system maintains two cursors: `lastTimestamp` (global "seen" cursor) and `lastAgentTimestamp[chatJid]` (per-group processing cursor). These are persisted to SQLite via `setRouterState()`. If they get out of sync (e.g., crash between advancing one and the other), messages could be skipped or re-processed.
- Safe modification: The `getOrRecoverCursor()` function (lines 122-137) handles the case where `lastAgentTimestamp` is missing by falling back to the last bot message timestamp. There's also `recoverPendingMessages()` (lines 547-563) for startup recovery.
- Test coverage: No direct tests for cursor recovery logic.

## Scaling Limits

**SQLite single-writer bottleneck:**

- Current capacity: Handles typical personal assistant workload (tens of groups, hundreds of messages/day).
- Limit: SQLite has a single-writer lock. Under high concurrent message volume, `storeMessage()` calls from multiple channel callbacks could contend on the write lock.
- Scaling path: Use WAL mode (not currently configured — would improve read concurrency). For much higher scale, consider per-group databases or a different storage engine.

**Container concurrency limit:**

- Current capacity: Default `MAX_CONCURRENT_CONTAINERS = 5`.
- Limit: Each active group conversation spawns a Docker container. With many active groups, container startup and memory usage become bottlenecks.
- Scaling path: Configurable via `MAX_CONCURRENT_CONTAINERS` env var. The `GroupQueue` handles queuing when at capacity. Container reuse via IPC piping reduces the number of container starts.

**Message history growth:**

- Current capacity: All messages for registered groups are stored indefinitely in SQLite.
- Limit: The `messages` table grows without bound. Queries like `getMessagesSince()` use `ORDER BY timestamp DESC LIMIT ?` which mitigates query performance, but disk usage grows indefinitely.
- Scaling path: Implement message retention/pruning (e.g., keep last N days). No pruning mechanism exists currently.

## Dependencies at Risk

**`@onecli-sh/sdk` (`^0.2.0`):**

- Risk: External dependency for credential management and agent lifecycle. If the OneCLI service is unavailable, containers launch without credentials (logged as warning, not fatal).
- Impact: Agent containers would fail to call Anthropic API without credentials.
- Migration plan: The `use-native-credential-proxy` skill exists as an alternative. The code handles `onecliApplied === false` gracefully (`src/container-runner.ts` lines 243-249).

**`@anthropic-ai/claude-agent-sdk` (in container):**

- Risk: The entire agent execution depends on this SDK. Breaking changes in the SDK's `query()` API, message types, or session format would break the agent runner.
- Impact: Complete agent failure inside containers.
- Migration plan: The SDK is pinned in `container/agent-runner/package.json`. Test before upgrading.

## Missing Critical Features

**No message retention/pruning:**

- Problem: The `messages` table grows indefinitely with no cleanup mechanism.
- Blocks: Long-running installations will experience growing disk usage and potentially slower queries.

**No health check endpoint:**

- Problem: No HTTP health check or status endpoint for monitoring. The `launchd/` directory suggests macOS service management, but there's no way to programmatically check if NanoClaw is healthy.
- Blocks: Automated monitoring, container orchestration health checks.

**No graceful container drain on shutdown:**

- Problem: On shutdown, `GroupQueue.shutdown()` detaches containers rather than waiting for them to finish. The comment says "they'll finish on their own via idle timeout" but this means containers run unsupervised after the host exits.
- Files: `src/group-queue.ts` (lines 347-363)
- Blocks: Clean deployment updates; orphaned containers may accumulate if the cleanup on next startup (`cleanupOrphans()`) fails.

## Test Coverage Gaps

**`src/index.ts` (main orchestrator):**

- What's not tested: The main message loop, startup sequence, channel initialization, and the interaction between `processGroupMessages`, `runAgent`, and the queue. The file has 768 lines with no corresponding test file.
- Files: `src/index.ts`
- Risk: Changes to message routing, cursor management, or channel lifecycle could introduce regressions undetected.
- Priority: High — this is the central orchestration file.

**`src/container-runner.ts` (volume mount logic):**

- What's not tested: `buildVolumeMounts()` and `buildContainerArgs()` contain complex logic for determining which directories to mount, `.env` shadowing, skill syncing, and agent-runner copying. Only the timeout/output parsing behavior is tested.
- Files: `src/container-runner.ts` (lines 61-275), `src/container-runner.test.ts` (only tests timeout behavior)
- Risk: Incorrect mounts could expose secrets or break agent functionality.
- Priority: High — security-sensitive code path.

**`container/agent-runner/src/index.ts` (agent runner):**

- What's not tested: The entire agent runner inside the container has no test files. This includes `readStdin()`, `MessageStream`, `runQuery()`, `runScript()`, transcript parsing, and the IPC polling loop.
- Files: `container/agent-runner/src/index.ts` (733 lines), `container/agent-runner/src/ipc-mcp-stdio.ts` (501 lines)
- Risk: Changes to agent execution, session management, or IPC handling could silently break.
- Priority: Medium — container isolation limits blast radius, but debugging failures inside containers is harder.

**`src/mount-security.ts` (mount validation):**

- What's not tested: No test file exists despite 419 lines of security-critical code for validating additional mounts against the allowlist.
- Files: `src/mount-security.ts`
- Risk: Incorrect validation could allow mounting sensitive directories into containers.
- Priority: High — security-critical.

**`src/task-scheduler.ts` (task execution):**

- What's not tested: The `runTask()` function and `startSchedulerLoop()` are not tested. Only `computeNextRun()` is tested via `src/task-scheduler.test.ts`.
- Files: `src/task-scheduler.ts` (lines 78-241)
- Risk: Task execution, session handling, and error recovery could regress.
- Priority: Medium.

---

_Concerns audit: 2026-04-10_

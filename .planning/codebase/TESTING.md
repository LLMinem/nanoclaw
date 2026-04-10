# Testing Patterns

**Analysis Date:** 2026-04-10

## Test Framework

**Runner:**

- Vitest 4.x
- Config: `vitest.config.ts` (main), `vitest.skills.config.ts` (skill tests)

**Assertion Library:**

- Vitest built-in `expect` API

**Run Commands:**

```bash
npm test              # Run all tests (vitest run)
npm run test:watch    # Watch mode (vitest)
```

## Test File Organization

**Location:**

- Co-located with source files in the same directory

**Naming:**

- `{module}.test.ts` — always matches the source file name
- Examples: `src/db.test.ts`, `src/routing.test.ts`, `src/channels/registry.test.ts`

**Test includes (from `vitest.config.ts`):**

```
src/**/*.test.ts
setup/**/*.test.ts
```

**Skill tests use separate config (`vitest.skills.config.ts`):**

```
.claude/skills/**/tests/*.test.ts
```

**Structure:**

```
src/
├── db.ts
├── db.test.ts
├── router.ts
├── formatting.test.ts       # Tests for router.ts functions
├── routing.test.ts          # Tests for index.ts routing logic
├── container-runner.ts
├── container-runner.test.ts
├── channels/
│   ├── registry.ts
│   └── registry.test.ts
setup/
├── platform.test.ts
├── register.test.ts
├── service.test.ts
└── environment.test.ts
```

## Test Structure

**Suite Organization:**

```typescript
import { describe, it, expect, beforeEach } from 'vitest';

import { _initTestDatabase, storeMessage, getMessagesSince } from './db.js';

beforeEach(() => {
  _initTestDatabase();
});

// Section comment separating test groups
// --- getMessagesSince ---

describe('getMessagesSince', () => {
  beforeEach(() => {
    // Setup test data
    storeChatMetadata('group@g.us', '2024-01-01T00:00:00.000Z');
    store({ id: 'm1', chat_jid: 'group@g.us', ... });
  });

  it('returns messages after the given timestamp', () => {
    const msgs = getMessagesSince('group@g.us', '2024-01-01T00:00:02.000Z', 'Andy');
    expect(msgs).toHaveLength(1);
    expect(msgs[0].content).toBe('third');
  });
});
```

**Patterns:**

- Top-level `beforeEach` resets shared state (typically `_initTestDatabase()`)
- Nested `beforeEach` within `describe` blocks populates test-specific data
- Section comments (`// --- Section Name ---`) separate logically distinct test groups within a file
- Tests are descriptive but concise: `it('filters out empty content', () => {...})`
- `afterEach` used for cleanup of temp files and restoring real timers

## Mocking

**Framework:** Vitest built-in `vi.mock()` and `vi.fn()`

**Module Mocking Pattern:**

```typescript
// Mock entire modules at the top of test files
vi.mock('./config.js', () => ({
  CONTAINER_IMAGE: 'nanoclaw-agent:latest',
  CONTAINER_MAX_OUTPUT_SIZE: 10485760,
  DATA_DIR: '/tmp/nanoclaw-test-data',
  GROUPS_DIR: '/tmp/nanoclaw-test-groups',
}));

// Mock logger (very common — used in most test files that mock modules)
vi.mock('./logger.js', () => ({
  logger: {
    debug: vi.fn(),
    info: vi.fn(),
    warn: vi.fn(),
    error: vi.fn(),
  },
}));

// Mock fs with partial overrides (preserving real implementations)
vi.mock('fs', async () => {
  const actual = await vi.importActual<typeof import('fs')>('fs');
  return {
    ...actual,
    default: {
      ...actual,
      existsSync: vi.fn(() => false),
      mkdirSync: vi.fn(),
      writeFileSync: vi.fn(),
    },
  };
});

// Mock child_process
vi.mock('child_process', async () => {
  const actual =
    await vi.importActual<typeof import('child_process')>('child_process');
  return {
    ...actual,
    spawn: vi.fn(() => fakeProc),
  };
});
```

**Function Mock Pattern:**

```typescript
const processMessages = vi.fn(async (_groupJid: string) => {
  return true; // success
});

queue.setProcessMessagesFn(processMessages);
```

**Fake Process Pattern (for container tests):**

```typescript
import { EventEmitter } from 'events';
import { PassThrough } from 'stream';

function createFakeProcess() {
  const proc = new EventEmitter() as EventEmitter & {
    stdin: PassThrough;
    stdout: PassThrough;
    stderr: PassThrough;
    kill: ReturnType<typeof vi.fn>;
    pid: number;
  };
  proc.stdin = new PassThrough();
  proc.stdout = new PassThrough();
  proc.stderr = new PassThrough();
  proc.kill = vi.fn();
  proc.pid = 12345;
  return proc;
}
```

**What to Mock:**

- External dependencies: `child_process`, `fs` (when testing logic not file I/O)
- Configuration modules (`./config.js`) to control values
- Logger (`./logger.js`) to silence output and verify log calls
- Third-party SDKs (`@onecli-sh/sdk`)
- Container runtime (`./container-runtime.js`)

**What NOT to Mock:**

- The module under test
- In-memory SQLite database — use `_initTestDatabase()` which creates a real `:memory:` DB
- Pure utility functions (`escapeXml`, `formatMessages`, `isValidTimezone`)
- Type definitions (`./types.js`)

## Fixtures and Factories

**Test Data Helpers:**

```typescript
// Helper function to create test messages with defaults
function makeMsg(overrides: Partial<NewMessage> = {}): NewMessage {
  return {
    id: '1',
    chat_jid: 'group@g.us',
    sender: '123@s.whatsapp.net',
    sender_name: 'Alice',
    content: 'hello',
    timestamp: '2024-01-01T00:00:00.000Z',
    ...overrides,
  };
}

// Helper to store a message with defaults
function store(overrides: {
  id: string;
  chat_jid: string;
  sender: string;
  sender_name: string;
  content: string;
  timestamp: string;
  is_from_me?: boolean;
}) {
  storeMessage({
    ...overrides,
    is_from_me: overrides.is_from_me ?? false,
  });
}
```

**Test Group Constants:**

```typescript
const MAIN_GROUP: RegisteredGroup = {
  name: 'Main',
  folder: 'whatsapp_main',
  trigger: 'always',
  added_at: '2024-01-01T00:00:00.000Z',
  isMain: true,
};

const testGroup: RegisteredGroup = {
  name: 'Test Group',
  folder: 'test-group',
  trigger: '@Andy',
  added_at: new Date().toISOString(),
};
```

**Location:**

- Helpers and fixtures are defined at the top of each test file (not in shared files)
- No shared test utilities directory — each test file is self-contained

## Coverage

**Requirements:** None enforced. No coverage thresholds configured.

**Current test files (18 total):**

- `src/db.test.ts` (652 lines) — DB operations, message queries, task CRUD, migrations
- `src/formatting.test.ts` (350 lines) — XML escaping, message formatting, trigger patterns, outbound formatting
- `src/routing.test.ts` (170 lines) — JID ownership, getAvailableGroups
- `src/container-runner.test.ts` (229 lines) — Container timeout and output handling
- `src/group-queue.test.ts` (484 lines) — Concurrency, retries, task priority, idle preemption
- `src/task-scheduler.test.ts` (129 lines) — Scheduler loop, interval drift, next-run computation
- `src/sender-allowlist.test.ts` (216 lines) — Config loading, sender/trigger authorization
- `src/ipc-auth.test.ts` (679 lines) — IPC authorization, task scheduling, group registration
- `src/timezone.test.ts` (73 lines) — Timezone validation and formatting
- `src/container-runtime.test.ts` (162 lines) — Container stop, orphan cleanup, runtime detection
- `src/group-folder.test.ts` (43 lines) — Folder name validation, path traversal prevention
- `src/db-migration.test.ts` (67 lines) — Schema migration from legacy DBs
- `src/channels/registry.test.ts` (42 lines) — Channel registration and factory lookup
- `setup/service.test.ts` (187 lines) — Plist and systemd config generation
- `setup/register.test.ts` (464 lines) — Group registration, SQL injection, template copy
- `setup/platform.test.ts` (120 lines) — Platform detection, command existence
- `setup/environment.test.ts` (121 lines) — Environment/credential detection

## Test Types

**Unit Tests:**

- Scope: Individual functions and modules
- Approach: Test pure functions directly, mock dependencies for impure functions
- In-memory SQLite for database tests (not mocked — real queries)
- Examples: `timezone.test.ts`, `formatting.test.ts`, `group-folder.test.ts`

**Integration Tests:**

- Scope: Multi-module interactions (IPC authorization, container runner with mocked process)
- Approach: Real database with mocked I/O boundaries
- Examples: `ipc-auth.test.ts` tests full IPC message flow through `processTaskIpc()`; `container-runner.test.ts` tests output parsing with fake child process

**E2E Tests:**

- Not used. End-to-end testing is manual (run `npm run dev` and send messages).

## Common Patterns

**Database Test Setup:**

```typescript
import { _initTestDatabase } from './db.js';

beforeEach(() => {
  _initTestDatabase(); // Creates fresh in-memory SQLite DB
});
```

**Fake Timer Testing:**

```typescript
beforeEach(() => {
  vi.useFakeTimers();
});

afterEach(() => {
  vi.useRealTimers();
});

it('retries with exponential backoff', async () => {
  queue.enqueueMessageCheck('group1@g.us');
  await vi.advanceTimersByTimeAsync(10); // Let initial call run
  await vi.advanceTimersByTimeAsync(5000); // Wait for first retry
  await vi.advanceTimersByTimeAsync(10); // Let retry execute
  expect(callCount).toBe(2);
});
```

**Async Testing:**

```typescript
it('timeout with no output resolves as error', async () => {
  const resultPromise = runContainerAgent(
    testGroup,
    testInput,
    () => {},
    onOutput,
  );

  // Simulate time passing
  await vi.advanceTimersByTimeAsync(1830000);

  // Simulate process exit
  fakeProc.emit('close', 137);
  await vi.advanceTimersByTimeAsync(10);

  const result = await resultPromise;
  expect(result.status).toBe('error');
  expect(result.error).toContain('timed out');
});
```

**Error/Edge Case Testing:**

```typescript
it('rejects names with shell metacharacters', () => {
  expect(() => stopContainer('foo; rm -rf /')).toThrow(
    'Invalid container name',
  );
  expect(() => stopContainer('foo$(whoami)')).toThrow('Invalid container name');
  expect(() => stopContainer('foo`id`')).toThrow('Invalid container name');
  expect(mockExecSync).not.toHaveBeenCalled();
});
```

**Temp Directory Pattern (for file-based tests):**

```typescript
let tmpDir: string;

beforeEach(() => {
  tmpDir = fs.mkdtempSync(path.join(os.tmpdir(), 'allowlist-test-'));
});

afterEach(() => {
  fs.rmSync(tmpDir, { recursive: true, force: true });
});
```

**Authorization Testing Pattern:**

```typescript
// Replicate the exact check from source code, then test it
function isMessageAuthorized(
  sourceGroup: string,
  isMain: boolean,
  targetChatJid: string,
  registeredGroups: Record<string, RegisteredGroup>,
): boolean {
  const targetGroup = registeredGroups[targetChatJid];
  return isMain || (!!targetGroup && targetGroup.folder === sourceGroup);
}

it('non-main group cannot send to another groups chat', () => {
  expect(isMessageAuthorized('other-group', false, 'main@g.us', groups)).toBe(
    false,
  );
});
```

**Dynamic Import for Module Isolation:**

```typescript
// Used in migration tests to get a fresh module instance
vi.resetModules();
const { initDatabase, getAllChats, _closeDatabase } = await import('./db.js');
```

## Test Conventions

- Use `_` prefix for test-only exports from production code
- Annotate test-only exports with `/** @internal - for tests only */`
- Test files import from `.js` extension (matching ESM convention)
- Descriptive test names explain the behavior being verified
- One assertion per behavior (though multiple `expect` calls per test are common for verifying return structure)
- Security-related tests verify both positive and negative cases (e.g., injection prevention)

---

_Testing analysis: 2026-04-10_

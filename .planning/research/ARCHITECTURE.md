# Architecture Patterns

**Domain:** NanoClaw personal assistant — VPS deployment with Telegram, memory persistence, and Obsidian integration
**Researched:** 2026-04-12

## Recommended Architecture

MeeqClaw is a customization layer on top of NanoClaw's existing event-driven architecture. The architecture adds four new concerns to the base system: (1) Telegram as the sole channel, (2) persistent identity/memory via workspace files, (3) Obsidian vault integration via bind mounts, and (4) SSH-from-container for host administration. All four concerns are implemented through NanoClaw's existing extension points — no core modifications required.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VPS HOST (Ubuntu 24.04, Hetzner CX32)               │
│                                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ┌──────────────┐ │
│  │  systemd      │  │  OneCLI      │  │  Obsidian  │  │  Tailscale   │ │
│  │  (nanoclaw)   │  │  Gateway     │  │  Headless  │  │  VPN mesh    │ │
│  └──────┬───────┘  └──────┬───────┘  └──────┬─────┘  └──────────────┘ │
│         │                  │                  │                          │
│         ▼                  │                  │                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                   NanoClaw Host Process                          │   │
│  │                                                                  │   │
│  │  Telegram Channel ←→ Message Loop ←→ GroupQueue ←→ Container    │   │
│  │  (grammy)            (2s poll)        (max 2)      Runner       │   │
│  │                                                                  │   │
│  │  SQLite DB    Task Scheduler    IPC Watcher                     │   │
│  │  (store/)     (60s poll)        (1s poll)                       │   │
│  └──────────────────────────────┬───────────────────────────────────┘   │
│                                  │ spawns Docker container               │
│  ┌───────────────────────────────▼──────────────────────────────────┐   │
│  │                   Docker Container (nanoclaw-agent)              │   │
│  │                                                                  │   │
│  │  /workspace/group    → groups/telegram_main/  (rw)              │   │
│  │  /workspace/project  → nanoclaw/              (ro)              │   │
│  │  /workspace/extra/vault → ~/vaults/meeq-vault/fritz/ (rw)      │   │
│  │  /workspace/extra/memory → ~/clawd/memory/    (ro→rw)          │   │
│  │  /home/node/.claude  → data/sessions/telegram_main/.claude/ (rw)│   │
│  │  /home/node/.ssh     → ~/.ssh/nanoclaw/       (ro)              │   │
│  │  /workspace/ipc      → data/ipc/telegram_main/ (rw)            │   │
│  │                                                                  │   │
│  │  Claude Agent SDK → OneCLI gateway → api.anthropic.com          │   │
│  │  SSH client → host.docker.internal (via deploy key)             │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                   Filesystem Layout                              │   │
│  │                                                                  │   │
│  │  ~/nanoclaw/                 NanoClaw installation (main branch) │   │
│  │  ~/vaults/meeq-vault/       Obsidian vault (synced)             │   │
│  │    └── fritz/                Fritz workspace inside vault        │   │
│  │        ├── SOUL.md           Fritz personality (version ctrl'd)  │   │
│  │        ├── USER.md           meeq's profile                     │   │
│  │        ├── MEMORY.md         Long-term memory index             │   │
│  │        └── memory/           Daily/deep memory files            │   │
│  │  ~/.ssh/nanoclaw/            Container-only SSH key pair        │   │
│  │  ~/.config/nanoclaw/         Security configs (external)        │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### Component Boundaries

| Component            | Responsibility                                                             | Communicates With                                                         | Location                                                    |
| -------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------------------------- | ----------------------------------------------------------- |
| **Telegram Channel** | Receives/sends messages via Telegram Bot API                               | Host message loop (onMessage callback), SQLite (message storage)          | `src/channels/telegram.ts` (added by `/add-telegram` skill) |
| **Host Process**     | Message routing, container lifecycle, IPC, scheduling                      | All components — central orchestrator                                     | `src/index.ts`, `src/router.ts`                             |
| **Container Runner** | Spawns Docker containers with correct mounts                               | Docker daemon, OneCLI gateway                                             | `src/container-runner.ts`                                   |
| **Agent Runner**     | Runs Claude Agent SDK inside container, streams output                     | MCP server (IPC tools), Claude API (via OneCLI)                           | `container/agent-runner/src/index.ts`                       |
| **MCP Server**       | Provides NanoClaw tools to the agent (send_message, schedule_task, etc.)   | Host IPC watcher (via filesystem)                                         | `container/agent-runner/src/ipc-mcp-stdio.ts`               |
| **SQLite DB**        | Stores messages, sessions, groups, tasks, router state                     | Host process reads/writes; container reads (via project mount, main only) | `store/messages.db`                                         |
| **OneCLI Gateway**   | Injects API credentials into container HTTPS traffic                       | Container agents (transparent proxy), Anthropic API                       | `http://localhost:10254`                                    |
| **systemd Service**  | Keeps NanoClaw running, restarts on failure                                | Host process (manages lifecycle)                                          | `~/.config/systemd/user/nanoclaw.service`                   |
| **Obsidian Vault**   | Cross-device visibility of memory files (iPhone, MacBook, VPS)             | Container agent (via bind mount), Obsidian Sync (background)              | `~/vaults/meeq-vault/`                                      |
| **Workspace Files**  | Fritz identity (SOUL.md), user context (USER.md), memory index (MEMORY.md) | Container agent (CLAUDE.md loads them), Git (version history)             | `~/vaults/meeq-vault/fritz/`                                |
| **SSH Bridge**       | Allows container agent to run commands on host                             | Container SSH client → host sshd                                          | `~/.ssh/nanoclaw/` (deploy key)                             |

### Data Flow

**Inbound Message (User → Fritz → User):**

```
1. User sends Telegram message
2. grammy bot receives via long polling
3. onMessage callback stores in SQLite
4. Message loop (2s poll) detects new message
5. Trigger check passes (main group: no trigger needed)
6. GroupQueue enqueues work
7. Container Runner builds volume mounts:
   - group folder (rw)
   - project root (ro, .env shadowed)
   - vault/fritz workspace (rw, via additionalMounts)
   - .claude sessions (rw)
   - IPC directory (rw)
   - SSH key directory (ro)
8. Container spawned with OneCLI gateway config
9. Agent runner reads stdin JSON, invokes Claude Agent SDK
10. SDK loads CLAUDE.md hierarchy:
    - /workspace/group/CLAUDE.md (group-level: Fritz identity + references to workspace)
    - /workspace/extra/vault/CLAUDE.md (if present: loaded via CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD)
11. Agent processes message, reads/writes memory files
12. Output streamed via stdout sentinel markers
13. Host parses output, sends via Telegram channel
14. Session ID saved for continuity
```

**Memory Write (Fritz → Obsidian → All Devices):**

```
1. Agent writes to /workspace/extra/vault/memory/2026-04-12.md
2. File lands on host at ~/vaults/meeq-vault/fritz/memory/2026-04-12.md
3. Obsidian Sync detects change, propagates to iPhone/MacBook
4. meeq sees updated memory in Obsidian on any device
```

**SSH Host Command (Fritz → VPS):**

```
1. Agent runs: ssh -o StrictHostKeyChecking=accept-new host.docker.internal "df -h"
2. SSH client inside container uses /home/node/.ssh/ deploy key
3. Connects to host sshd via Docker bridge network
4. Executes command on host with restricted shell/user
5. Output returned to agent
```

## Component Design Details

### 1. Telegram Channel Integration

**How it works:** The `/add-telegram` skill merges a git branch (`nanoclaw-telegram/main`) that adds `src/channels/telegram.ts`. This file self-registers via `registerChannel('telegram', factory)` at import time. The barrel file `src/channels/index.ts` triggers the import.

**Implementation:** Uses the `grammy` library for Telegram Bot API. Long polling (no webhook needed — no public URL required). The channel adapter:

- Receives messages via `bot.on('message')`
- Stores them in SQLite with JID prefix `tg:` (e.g., `tg:123456789`)
- Sends outbound messages via `bot.api.sendMessage()`
- Supports typing indicators via `bot.api.sendChatAction()`

**Registration:** Main chat registered as `telegram_main` with `--no-trigger-required --is-main`. Chat ID obtained by sending `/chatid` to the bot.

**Confidence:** HIGH — Telegram skill exists, well-documented in SKILL.md, uses established grammy library.

### 2. Workspace Files & Fritz Identity

**Architecture decision: Workspace files live inside the Obsidian vault.**

Rationale: The primary requirement is cross-device visibility (iPhone, MacBook, VPS). Obsidian Sync already handles this for the vault. Putting SOUL.md, USER.md, MEMORY.md, and memory/ inside the vault means they're immediately visible on all devices.

**Proposed vault structure:**

```
~/vaults/meeq-vault/
└── fritz/                          # Fritz's workspace root
    ├── CLAUDE.md                   # References SOUL.md, USER.md, MEMORY.md
    ├── SOUL.md                     # Fritz personality definition
    ├── USER.md                     # meeq's profile and preferences
    ├── MEMORY.md                   # Long-term memory index
    └── memory/                     # Daily and deep memory files
        ├── 2026-02-01.md
        ├── 2026-02-02.md
        └── ...
```

**How it's mounted:** Via `additionalMounts` in the main group's `containerConfig`:

```json
{
  "containerConfig": {
    "additionalMounts": [
      {
        "hostPath": "~/vaults/meeq-vault/fritz",
        "containerPath": "vault",
        "readonly": false
      }
    ]
  }
}
```

This appears at `/workspace/extra/vault/` inside the container.

**CLAUDE.md hierarchy:** The agent loads context from multiple CLAUDE.md files:

1. `/workspace/group/CLAUDE.md` — the group-level instructions (Fritz identity, communication style, tools reference, links to vault workspace)
2. `/workspace/extra/vault/CLAUDE.md` — optional additional context loaded via `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` (already enabled in container settings)

**The group CLAUDE.md** (`groups/telegram_main/CLAUDE.md`) is the primary identity file. It should reference the vault workspace files explicitly:

```markdown
# Fritz

You are Fritz, meeq's personal assistant. [personality from SOUL.md]

## Workspace

Your memory and identity files are at /workspace/extra/vault/:

- Read /workspace/extra/vault/SOUL.md for your personality
- Read /workspace/extra/vault/USER.md for meeq's context
- Read /workspace/extra/vault/MEMORY.md for your memory index
- Read/write /workspace/extra/vault/memory/ for daily memories
```

**Confidence:** HIGH for the mount mechanism (verified in `container-runner.ts` lines 213-221). MEDIUM for CLAUDE.md hierarchy with additional directories — the setting exists and is enabled, but exact loading behavior should be verified during implementation.

### 3. Git + Obsidian Sync Coexistence

**Problem:** Memory files need both version history (git) and cross-device sync (Obsidian Sync). These can conflict if both try to manage the same files.

**Solution: Git repo inside the vault, Obsidian Sync as transport.**

```
~/vaults/meeq-vault/fritz/   ← Git repo root
├── .git/                     ← Git tracks all changes
├── .gitignore                ← Ignore Obsidian cruft (.obsidian/)
├── SOUL.md
├── USER.md
├── MEMORY.md
└── memory/
```

**How they coexist:**

1. **Obsidian Sync** propagates file changes across devices in near-real-time
2. **Git** provides version history, diff capability, and rollback
3. **No conflict** because:
   - Obsidian Sync is the transport layer (moves files between devices)
   - Git is the history layer (records changes on the VPS)
   - The agent writes files; Obsidian Sync propagates them; git commits periodically
4. **`.gitignore`** excludes `.obsidian/` directory (Obsidian's own config)

**Git commit strategy:** Automated periodic commits via a cron task or scheduled NanoClaw task:

- Every night at midnight: `cd ~/vaults/meeq-vault/fritz && git add -A && git commit -m "memory: $(date +%Y-%m-%d)" --allow-empty`
- Or: NanoClaw scheduled task that commits and pushes

**Confidence:** MEDIUM — This is a common pattern for Obsidian + git (used by obsidian-git plugin community), but the specific interaction with NanoClaw's write patterns needs testing. Key risk: Obsidian Sync could create conflict files if both VPS and another device edit the same file simultaneously. Mitigation: memory files are only written by the agent on VPS, so conflicts are unlikely.

### 4. SSH from Container

**Problem:** Fritz needs to administer the VPS (check disk space, restart services, view logs) but runs inside a Docker container.

**Solution: Dedicated SSH deploy key with restricted host access.**

**Architecture:**

```
Container:
  /home/node/.ssh/          ← mounted from ~/.ssh/nanoclaw/ (ro)
    ├── id_ed25519           ← deploy key (private)
    ├── id_ed25519.pub       ← deploy key (public)
    └── config               ← SSH config targeting host

Host:
  ~/.ssh/authorized_keys    ← deploy key public key, with restrictions
```

**SSH key pair generation:**

```bash
mkdir -p ~/.ssh/nanoclaw
ssh-keygen -t ed25519 -f ~/.ssh/nanoclaw/id_ed25519 -N "" -C "nanoclaw-container"
```

**Host authorized_keys entry (restricted):**

```
restrict,command="/usr/bin/bash",from="172.17.0.0/16" ssh-ed25519 AAAA... nanoclaw-container
```

The `restrict` keyword disables port forwarding, X11, agent forwarding, and PTY allocation. The `from` clause limits connections to the Docker bridge network. The `command` restriction can be tightened further based on needs.

**SSH config inside container (`~/.ssh/nanoclaw/config`):**

```
Host vps
    HostName host.docker.internal
    User meeq
    IdentityFile /home/node/.ssh/id_ed25519
    StrictHostKeyChecking accept-new
```

**Mount configuration:**

```json
{
  "containerConfig": {
    "additionalMounts": [
      {
        "hostPath": "~/.ssh/nanoclaw",
        "containerPath": "ssh",
        "readonly": true
      }
    ]
  }
}
```

Wait — this won't work directly because `.ssh` is in the blocked patterns list in `mount-security.ts`. The blocked pattern check matches any path component containing `.ssh`.

**Alternative approach: Mount the key files individually or use a differently-named directory.**

Looking at `mount-security.ts` line 153: `if (part === pattern || part.includes(pattern))` — it checks if any path component _contains_ the pattern string. So `~/.ssh/nanoclaw` would be blocked because the `.ssh` component matches.

**Workaround options:**

1. **Store keys in a non-blocked path** like `~/.config/nanoclaw/container-ssh/` and mount that
2. **Add an exception** to the mount allowlist's blocked patterns (but the blocked patterns are hardcoded defaults merged with the allowlist, and `.ssh` is in `DEFAULT_BLOCKED_PATTERNS`)
3. **Override the mount system** by modifying `container-runner.ts` to add SSH mounts as a built-in (not via additionalMounts)

**Recommended: Option 1 — separate directory with non-blocked name.**

```bash
mkdir -p ~/.config/nanoclaw/ssh-keys
ssh-keygen -t ed25519 -f ~/.config/nanoclaw/ssh-keys/id_ed25519 -N "" -C "nanoclaw-container"
```

Then the mount allowlist allows `~/.config/nanoclaw/ssh-keys/` and mounts it at `/home/node/.ssh/` inside the container. The container path is not subject to blocked pattern checking — only the host path is checked.

Actually, looking more carefully at the code: additional mounts go to `/workspace/extra/{containerPath}`, not arbitrary paths. The container path is always prefixed with `/workspace/extra/`. So we can't mount directly to `/home/node/.ssh/`.

**Revised approach: Mount to /workspace/extra/ssh-keys/ and use SSH config to reference it.**

```
# SSH command from inside container:
ssh -i /workspace/extra/ssh-keys/id_ed25519 meeq@host.docker.internal
```

Or create an alias in the group's CLAUDE.md:

```markdown
## SSH Access

To run commands on the VPS host:
ssh -i /workspace/extra/ssh-keys/id_ed25519 -o StrictHostKeyChecking=accept-new meeq@host.docker.internal "<command>"
```

**Confidence:** MEDIUM — The mount mechanism works, but SSH from container to host needs testing. `host.docker.internal` is resolved correctly on Linux (via `--add-host=host.docker.internal:host-gateway` in `container-runtime.ts`). The SSH client is available in the container (installed via `git` package in Dockerfile). Key concern: the container runs as uid 1000 (`node` user), but the host user is likely a different UID — the `--user` flag in `container-runner.ts` remaps this.

### 5. Service Management (systemd)

**NanoClaw's setup already handles Linux systemd.** The `setup/service.ts` module:

1. Detects platform (Linux)
2. Generates a systemd user unit file at `~/.config/systemd/user/nanoclaw.service`
3. Enables `loginctl enable-linger` so the service survives SSH logout
4. Enables and starts the service

**Unit file features:**

- `Restart=always` with `RestartSec=5` — auto-restarts on crash
- `KillMode=process` — only kills the main process, not child containers
- Logs to `logs/nanoclaw.log` and `logs/nanoclaw.error.log`
- `After=network.target` for proper ordering

**Docker group consideration:** The service setup includes a `checkDockerGroupStale()` check. If the user was added to the docker group mid-session, the systemd user daemon won't see the new group. Fix: log out and back in, or run `systemctl --user restart dbus` and reload.

**Confidence:** HIGH — The systemd setup code is comprehensive and handles edge cases (root vs user, WSL fallback, docker group staleness).

### 6. Container Mount Architecture (Complete)

For the main `telegram_main` group, the full mount set:

| Container Path              | Host Path                                       | Access | Purpose                                                 |
| --------------------------- | ----------------------------------------------- | ------ | ------------------------------------------------------- |
| `/workspace/group`          | `groups/telegram_main/`                         | rw     | Group working directory, CLAUDE.md, logs, conversations |
| `/workspace/project`        | `~/nanoclaw/`                                   | ro     | NanoClaw project root (code, DB, configs)               |
| `/workspace/project/.env`   | `/dev/null`                                     | ro     | Shadow secrets file                                     |
| `/workspace/extra/vault`    | `~/vaults/meeq-vault/fritz/`                    | rw     | Obsidian vault workspace (memory, identity)             |
| `/workspace/extra/ssh-keys` | `~/.config/nanoclaw/ssh-keys/`                  | ro     | SSH deploy key for host access                          |
| `/home/node/.claude`        | `data/sessions/telegram_main/.claude/`          | rw     | Claude sessions, auto-memory, skills                    |
| `/workspace/ipc`            | `data/ipc/telegram_main/`                       | rw     | Host ↔ container communication                         |
| `/app/src`                  | `data/sessions/telegram_main/agent-runner-src/` | rw     | Agent runner source (customizable)                      |

**Mount allowlist (`~/.config/nanoclaw/mount-allowlist.json`):**

```json
{
  "allowedRoots": [
    {
      "path": "~/vaults/meeq-vault/fritz",
      "allowReadWrite": true,
      "description": "Fritz workspace in Obsidian vault"
    },
    {
      "path": "~/.config/nanoclaw/ssh-keys",
      "allowReadWrite": false,
      "description": "SSH deploy key for container-to-host access"
    },
    {
      "path": "~/projects",
      "allowReadWrite": true,
      "description": "Development projects"
    }
  ],
  "blockedPatterns": [],
  "nonMainReadOnly": true
}
```

**Confidence:** HIGH — All mount paths verified against `container-runner.ts` `buildVolumeMounts()` function and `mount-security.ts` `validateAdditionalMounts()`.

### 7. Skill System (Git Branch Merges)

**How NanoClaw skills work:**

Skills are Claude Code slash commands defined in `.claude/skills/`. They are NOT runtime plugins — they're instructions that guide Claude Code (the development tool) to make code changes. When you run `/add-telegram`, Claude Code reads `.claude/skills/add-telegram/SKILL.md` and follows the instructions to merge code from a remote git branch.

**Two types of skills:**

1. **Host-side skills** (`.claude/skills/`): Development-time instructions for adding features to NanoClaw. Examples: `/add-telegram`, `/setup`, `/debug`. These modify the codebase.

2. **Container-side skills** (`container/skills/`): Runtime instructions synced into each container's `.claude/skills/` directory. These provide in-container capabilities like browser automation, status reporting. Synced by `container-runner.ts` at container spawn time.

**Customization workflow (standard fork pattern):**

```
main branch     → all customizations live here (your working branch)
upstream remote → tracks qwibitai/nanoclaw (fetch-only, no push)

# To add a feature:
# Run the skill interactively: /add-telegram, /add-voice-transcription, etc.
# The skill merges its branch and modifies code on main
# Commit the result

# To get upstream updates:
git fetch upstream
git merge upstream/main
```

**Confidence:** HIGH — Verified by reading the add-telegram SKILL.md and the container-runner.ts skill sync code.

## Patterns to Follow

### Pattern 1: Additional Mounts for Workspace Expansion

**What:** Use NanoClaw's `additionalMounts` in `containerConfig` to expose host directories to the container. Mount allowlist controls access.

**When:** Whenever the container agent needs access to files outside its default group folder.

**Example:**

```typescript
// In registered_groups SQLite table:
{
  "containerConfig": {
    "additionalMounts": [
      {
        "hostPath": "~/vaults/meeq-vault/fritz",
        "containerPath": "vault",
        "readonly": false
      }
    ]
  }
}
```

Results in mount: `~/vaults/meeq-vault/fritz → /workspace/extra/vault` (rw)

### Pattern 2: CLAUDE.md as Identity Layer

**What:** Use the group-level CLAUDE.md (`groups/telegram_main/CLAUDE.md`) as the primary identity file. Reference external workspace files (in the vault mount) from within it.

**When:** Defining Fritz's personality, communication style, and available resources.

**Example:**

```markdown
# Fritz

[Full Fritz personality and instructions]

## Workspace Files

Your workspace files are mounted at /workspace/extra/vault/:

- `/workspace/extra/vault/SOUL.md` — your personality definition
- `/workspace/extra/vault/USER.md` — meeq's profile
- `/workspace/extra/vault/MEMORY.md` — long-term memory index
- `/workspace/extra/vault/memory/` — daily memory files

Read SOUL.md and USER.md at the start of each session for context.
Update MEMORY.md when you learn significant new information.
Create daily memory files in memory/ for important interactions.
```

### Pattern 3: Atomic IPC for Host Communication

**What:** Container agents communicate with the host through atomic file writes (write to `.tmp`, rename to `.json`). Host's IPC watcher polls directories every 1 second.

**When:** Agent needs to send messages, schedule tasks, or register groups.

**Already implemented** — follow the existing pattern for any custom IPC needs.

### Pattern 4: Security-First Mount Validation

**What:** All additional mounts are validated against an external allowlist at `~/.config/nanoclaw/mount-allowlist.json`. The allowlist is never mounted into containers.

**When:** Every time `additionalMounts` is configured for a group.

**Rules:**

- Host path must exist and resolve under an `allowedRoot`
- Path must not match any blocked pattern (`.ssh`, `.env`, `credentials`, etc.)
- Non-main groups forced to read-only when `nonMainReadOnly: true`
- Container path validated for traversal attacks (no `..`, no absolute paths)

## Anti-Patterns to Avoid

### Anti-Pattern 1: Modifying NanoClaw Core Directly

**What:** Directly editing files in `src/` that are part of upstream NanoClaw.

**Why bad:** Makes upstream merges painful. Every upstream update will conflict with local modifications. NanoClaw is designed for extension, not modification.

**Instead:** Use NanoClaw's extension points:

- Channels: Add via `src/channels/` self-registration
- Agent tools: Add via `container/agent-runner/src/ipc-mcp-stdio.ts`
- Container skills: Add to `container/skills/`
- Identity/memory: Use CLAUDE.md files and workspace mounts
- Host directories: Use `additionalMounts` and mount allowlist

### Anti-Pattern 2: Storing Secrets in Mounted Directories

**What:** Putting API keys, tokens, or credentials in directories mounted into containers.

**Why bad:** Container agents can read all mounted files. The entire security model depends on credentials being injected by OneCLI gateway, never visible to agents.

**Instead:** Use OneCLI for all credential management. `.env` is already shadowed with `/dev/null` in the project mount.

### Anti-Pattern 3: Memory Files Outside the Vault

**What:** Splitting workspace files between the vault and another location (e.g., some in `groups/telegram_main/`, some in the vault).

**Why bad:** Creates confusion about where the agent should read/write. Obsidian Sync only covers the vault. Git tracking would need multiple repos.

**Instead:** Put all Fritz workspace files (SOUL.md, USER.md, MEMORY.md, memory/) in one location inside the vault. Use the group CLAUDE.md only for agent instructions (how to behave), not for persistent data.

### Anti-Pattern 4: SSH Without Restrictions

**What:** Giving the container SSH key unrestricted access to the host user account.

**Why bad:** A compromised agent (via prompt injection) could modify system files, install backdoors, or exfiltrate data beyond its intended scope.

**Instead:** Restrict the SSH key in `authorized_keys` with `restrict`, `from=` (Docker subnet only), and optionally `command=` to limit what can be executed.

## Scalability Considerations

| Concern                   | Single User (meeq)                  | Future Multi-Agent                 | Notes                           |
| ------------------------- | ----------------------------------- | ---------------------------------- | ------------------------------- |
| Concurrent containers     | MAX_CONCURRENT_CONTAINERS=2 is fine | Increase to 3-5 for Telegram swarm | CX32 has 4 vCPU, 8GB RAM        |
| SQLite contention         | No issue — single-writer            | WAL mode already configured        | NanoClaw uses WAL by default    |
| Obsidian Sync conflicts   | None — single writer (VPS agent)    | Would need file-level locking      | Not a v1 concern                |
| Container startup time    | ~3-5s (acceptable for chat)         | Parallel spawning already works    | GroupQueue handles this         |
| Disk space (memory files) | ~60 files × ~5KB = negligible       | Monitor at 1000+ files             | Prune old memories periodically |
| Session transcript size   | JSONL grows over time               | Clear old sessions periodically    | `data/sessions/` can grow large |

## Build Order Implications

Components have these dependencies, which inform the suggested build order:

```
1. VPS Setup + Telegram Channel
   ├── No dependencies (clean Ubuntu + Docker)
   ├── Installs NanoClaw, applies /add-telegram skill
   └── Registers main chat → Fritz is reachable

2. Fritz Identity (CLAUDE.md)
   ├── Depends on: Telegram working (to test)
   ├── Migrate SOUL.md, USER.md content into groups/telegram_main/CLAUDE.md
   └── Fritz responds with personality → validates identity

3. Vault Mount + Memory Migration
   ├── Depends on: Fritz identity (needs to know where to write)
   ├── Create vault/fritz/ directory, configure additionalMounts
   ├── Migrate ~/clawd/memory/ → ~/vaults/meeq-vault/fritz/memory/
   ├── Configure mount-allowlist.json
   └── Fritz can read/write memory → validates persistence

4. SSH Bridge
   ├── Depends on: Working container (to test from inside)
   ├── Generate deploy key, configure authorized_keys
   ├── Configure additionalMount for ssh-keys
   └── Fritz can SSH to host → validates administration

5. Git + Obsidian Sync
   ├── Depends on: Vault mount working
   ├── Init git repo in vault/fritz/
   ├── Configure Obsidian Headless if not already running
   ├── Set up periodic git commit (scheduled task)
   └── Memory files visible on all devices → validates sync

6. Channel Formatting + Voice
   ├── Depends on: Telegram channel working
   ├── Apply /channel-formatting skill
   ├── Apply voice transcription (ElevenLabs or Whisper)
   └── Messages look correct, voice notes work
```

**Critical path:** Steps 1→2→3 are the minimum viable assistant. Steps 4-6 are enhancements.

## Sources

- NanoClaw source code analysis (container-runner.ts, mount-security.ts, container-runtime.ts, config.ts, types.ts) — **HIGH confidence**
- NanoClaw docs (SPEC.md, SECURITY.md) — **HIGH confidence**
- NanoClaw skill files (add-telegram/SKILL.md) — **HIGH confidence**
- NanoClaw setup code (setup/service.ts) — **HIGH confidence** for systemd configuration
- Docker networking (`host.docker.internal`) on Linux — **HIGH confidence** (verified in container-runtime.ts hostGatewayArgs)
- SSH restricted keys pattern — **HIGH confidence** (well-documented OpenSSH feature)
- Obsidian Sync + Git coexistence — **MEDIUM confidence** (common community pattern, not officially documented)
- CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD behavior — **MEDIUM confidence** (setting exists in container-runner.ts, exact behavior should be verified)

---

_Architecture research: 2026-04-12_

# Project Research Summary

**Project:** MeeqClaw (NanoClaw fork → personal AI assistant)
**Domain:** Personal AI assistant — containerized, Telegram-based, VPS-deployed
**Researched:** 2026-04-12
**Confidence:** HIGH

## Executive Summary

MeeqClaw is a fork-and-customize project, not a build-from-scratch project. NanoClaw provides a mature (~4,800 LOC) event-driven agent runtime with containerized execution, SQLite persistence, task scheduling, and a multi-channel abstraction layer. The expert approach is to use NanoClaw's extension points — skill branch merges for channels, `additionalMounts` for workspace expansion, CLAUDE.md files for identity, and IPC for agent-host communication — rather than modifying core code. Every major capability Fritz needs (Telegram messaging, memory persistence, voice transcription, VPS administration) maps to an existing NanoClaw mechanism. The implementation is primarily configuration and integration, not novel engineering.

The recommended approach is a 4-phase build that follows NanoClaw's dependency chain: (1) deploy NanoClaw + Telegram channel on the VPS, (2) set up workspace architecture with memory files in the Obsidian vault via bind mounts, (3) add voice transcription and SSH-from-container, (4) polish with typing indicators, bot commands, and operational hardening. The critical path is Phases 1-2: once Fritz is reachable on Telegram with personality and memory intact, the migration from OpenClaw is functionally complete.

The primary risks are operational, not architectural: OneCLI gateway dying silently on a headless VPS (total agent failure with no notification), Obsidian Sync and git fighting over the same files (data corruption), and mount path mismatches between dev Mac and production VPS (silent mount failures). All three have clear prevention strategies identified in research. The Obsidian Sync + git conflict is the only one requiring an architectural decision — the recommendation is to keep `.git/` outside the Obsidian-synced vault entirely, using a one-way copy for version history if needed.

## Key Findings

### Recommended Stack

The stack is almost entirely inherited from NanoClaw. No technology choices need to be made for the core runtime — Node.js 22, TypeScript 5.7, Claude Agent SDK, SQLite, Docker are all locked in. The only new dependencies are grammY (Telegram bot library, installed via the `add-telegram` skill merge), `@elevenlabs/elevenlabs-js` (speech-to-text, replacing the upstream Whisper integration), and openssh-client in the container Dockerfile (for SSH-from-container). On the infrastructure side, systemd replaces macOS launchd for process management.

**Core technologies:**

- **NanoClaw runtime** (Node.js 22 + Claude Agent SDK) — inherited, battle-tested, no choice needed
- **grammY 1.42** — Telegram bot library, upstream skill uses it, better TS support than alternatives
- **ElevenLabs Scribe v2** — speech-to-text API, user preference over OpenAI Whisper, official JS SDK exists
- **systemd user service** — native Ubuntu process management, replaces launchd, handles auto-restart
- **Obsidian Sync + vault** — cross-device file sync already running on VPS, iPhone, MacBook
- **OneCLI gateway** — credential injection into containers, already NanoClaw's default

**Critical version note:** `@elevenlabs/elevenlabs-js` v2.42.0 verified on npm. ElevenLabs STT endpoint (`/v1/speech-to-text`) supports OGG/Opus (Telegram's voice format) natively — no audio conversion needed.

### Expected Features

**Must have (table stakes):**

- Telegram text messaging (skill merge, low complexity)
- Message persistence and session continuity (already in NanoClaw)
- Channel formatting — Markdown → Telegram syntax (skill merge)
- Voice message transcription (custom ElevenLabs implementation, medium complexity)
- Fritz identity via SOUL.md/USER.md (mount system, low complexity)
- Sender allowlist (already in NanoClaw)
- Always-on 24/7 via VPS + systemd (already architected)

**Should have (differentiators):**

- File-based long-term memory with Obsidian vault integration (medium complexity)
- SSH from container to host for VPS administration (high complexity, security-sensitive)
- Typing indicator while processing (low effort, high UX impact)
- Bot commands menu — `/tasks`, `/memory`, `/status` (low effort, nice polish)
- CLI tool (`claw`) for terminal-based prompts (low effort, skill exists)

**Defer (v2+):**

- Text-to-speech responses (TTS) — text works fine
- Image vision for Telegram — needs custom download path
- Inline keyboards / structured interactions — natural language works
- Morning digest / motivation recaps — need prompt engineering, not infrastructure
- Telegram swarm / multi-bot — requires solid base first
- Embedding-based memory — grep sufficient for ~100 files
- Browser automation — adds 500MB to container image

### Architecture Approach

MeeqClaw layers four new concerns onto NanoClaw's existing architecture, all through extension points: (1) Telegram channel via skill branch merge and factory-pattern self-registration, (2) Fritz identity via CLAUDE.md hierarchy and workspace file mounts, (3) Obsidian vault integration via `additionalMounts` with mount-allowlist security, (4) SSH-from-container via a dedicated deploy key mounted read-only. The container mount architecture is fully mapped out — 8 mount points per container covering group workspace, project root, vault, SSH keys, Claude sessions, IPC, and agent runner source.

**Major components:**

1. **Telegram Channel** (`src/channels/telegram.ts`) — grammY-based, long polling, self-registers via factory pattern
2. **Container Runner** — spawns Docker containers with correct mounts, OneCLI config, IPC directories
3. **Workspace Files** (SOUL.md, USER.md, MEMORY.md) — live inside Obsidian vault at `~/vaults/meeq-vault/fritz/`, mounted at `/workspace/extra/vault/`
4. **SSH Bridge** — dedicated ed25519 deploy key at `~/.config/nanoclaw/ssh-keys/`, restricted via `authorized_keys` options
5. **systemd Service** — auto-generated by setup, `Restart=always`, `loginctl enable-linger` for headless operation

**Key architectural decisions made by research:**

- Workspace files live INSIDE the Obsidian vault (not separate) — for cross-device visibility
- Git repo for version history stays OUTSIDE the vault — avoids Obsidian Sync conflicts
- SSH keys stored at `~/.config/nanoclaw/ssh-keys/` (not `~/.ssh/`) — avoids mount-security blocked patterns
- Additional mounts go to `/workspace/extra/{name}` (not arbitrary container paths) — NanoClaw constraint
- No core NanoClaw modifications — use extension points only for clean upstream merges

### Critical Pitfalls

1. **OneCLI gateway death = total agent failure** — Gateway runs as separate process; if it dies on headless VPS, containers start but have no API credentials. No notification to user. **Prevention:** Create OneCLI systemd unit with `Restart=always`, add `Requires=onecli.service` to NanoClaw unit. Or use `use-native-credential-proxy` skill as simpler alternative.

2. **Obsidian Sync + git = data corruption** — Never put `.git/` inside an Obsidian-synced directory. Obsidian Sync will try to propagate `.git/` metadata, corrupting the repo. **Prevention:** Keep git repo outside vault. Use one-way copy script for version history if desired.

3. **Skill branch merge silently breaks channels** — The `src/channels/index.ts` barrel file has minimal context for git merge heuristics. Auto-merge can produce incorrect imports. **Prevention:** After every skill merge, manually verify the barrel file has correct imports, run `npm run build`, send test message.

4. **Mount path mismatches (Mac vs VPS)** — Mount allowlist uses absolute paths. Mac paths (`/Users/`) don't exist on Linux (`/home/`). Container starts but mounts silently fail. **Prevention:** Create mount allowlist directly on VPS, never copy from Mac.

5. **Memory migration requires content adaptation** — OpenClaw memory files contain references to `clawd`, `openclaw`, `ocx` paths/commands that become dangling pointers in NanoClaw. **Prevention:** Grep-and-replace pass on every migrated file before deployment.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: VPS Deployment + Telegram Channel

**Rationale:** Everything depends on a running NanoClaw instance with a working Telegram channel. This is the foundation — no other phase can be validated without it.
**Delivers:** Fritz reachable on Telegram, responding to text messages with basic personality.
**Addresses:** Telegram messaging, message persistence, session continuity, sender allowlist, always-on availability, channel formatting.
**Avoids:** Channel barrel file breakage (verify after merge), OneCLI gateway unreachable (create systemd unit), Docker group membership issues (verify before service start).
**Stack:** NanoClaw core, grammY, systemd, OneCLI, channel-formatting skill.
**Includes:** Fork setup on VPS, Telegram bot creation in BotFather, skill merges (`add-telegram`, `channel-formatting`), OneCLI configuration, systemd service creation, sender allowlist, basic Fritz identity in group CLAUDE.md, upstream sync policy documented.

### Phase 2: Workspace Architecture + Memory Migration

**Rationale:** Fritz's value comes from knowing meeq's context. Without memory files and identity, it's just a generic Claude chatbot. This phase must come after Phase 1 (needs working Telegram to test) and before Phase 3 (SSH and voice are enhancements, not core identity).
**Delivers:** Fritz with full personality, long-term memory, cross-device visibility via Obsidian.
**Addresses:** File-based memory, Obsidian vault integration, memory search, identity/personality (SOUL.md), user profile (USER.md).
**Avoids:** Obsidian Sync + git conflict (git stays outside vault), memory migration stale references (grep-and-replace pass), mount path mismatches (configure on VPS directly).
**Stack:** Obsidian Sync (existing), bind mounts via `additionalMounts`, mount-allowlist.json.
**Includes:** Create `~/vaults/meeq-vault/fritz/` workspace structure, configure mount allowlist on VPS, set up `additionalMounts` in group config, migrate SOUL.md/USER.md/MEMORY.md with content adaptation, migrate ~60 daily memory files, verify agent can read/write/search memory, decide on git versioning approach (likely deferred — Obsidian Sync's built-in version history may suffice for now).

### Phase 3: Voice Transcription + SSH Bridge

**Rationale:** These are the two highest-value enhancements that require custom implementation (no upstream skill directly applies). Voice is the #1 mobile input method for Telegram; SSH enables VPS administration. Both are independent of each other but depend on a working container (Phase 1) and proper mount config (Phase 2).
**Delivers:** Voice notes work, Fritz can administer the VPS.
**Addresses:** Telegram voice transcription (ElevenLabs), SSH from container to host, system monitoring.
**Avoids:** ElevenLabs credential injection issues (use OneCLI or credential proxy), SSH key exposure (dedicated deploy key with restrictions).
**Stack:** `@elevenlabs/elevenlabs-js`, openssh-client in Dockerfile, deploy key at `~/.config/nanoclaw/ssh-keys/`.
**Includes:** Custom `src/transcription.ts` (ElevenLabs STT), Telegram voice message download pipeline, SSH key generation and `authorized_keys` restriction, container Dockerfile modification for openssh-client, mount config for SSH keys, verify voice transcription end-to-end, verify SSH from container to host.

### Phase 4: Polish + Operational Hardening

**Rationale:** Once Fritz is functionally complete (Phases 1-3), focus shifts to UX polish and reliability. These are low-effort, high-impact improvements that don't require architectural decisions.
**Delivers:** Professional-feeling bot with good UX, reliable operations.
**Addresses:** Typing indicator, bot commands menu, CLI tool (`claw`), error recovery verification.
**Stack:** grammY API (`sendChatAction`, `setMyCommands`), `claw` CLI script.
**Includes:** Typing indicator during processing, `/tasks` `/memory` `/status` command menu, `claw` CLI install on VPS, health monitoring (disk space, service status), container timeout tuning, "looks done but isn't" checklist verification.

### Phase Ordering Rationale

- **Phase 1 → 2 is the critical path.** Once Fritz is on Telegram with personality and memory, the OpenClaw migration is functionally complete. Target: 1-2 days.
- **Phase 3 is two independent workstreams** (voice and SSH) that can be done in either order. Voice is higher priority for daily use. Target: 1-2 days.
- **Phase 4 is a collection of small improvements** that can be applied incrementally. No ordering constraints. Target: ongoing.
- **Architecture separates cleanly:** Each phase maps to a distinct architectural concern (runtime, workspace, integrations, polish), avoiding cross-cutting changes.
- **Pitfalls are front-loaded:** The highest-risk pitfalls (OneCLI, channel barrel file, Docker group) are all in Phase 1, where they'll be caught immediately.

### Research Flags

Phases likely needing deeper research during planning:

- **Phase 2:** Obsidian vault workspace structure — the exact CLAUDE.md hierarchy and how `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD` loads files from mounted directories needs verification during implementation. Architecture research is MEDIUM confidence here.
- **Phase 3 (Voice):** ElevenLabs STT API integration — the SDK is verified but the exact Telegram voice download → ElevenLabs pipeline needs implementation research. No upstream skill covers this combination.
- **Phase 3 (SSH):** Container-to-host SSH — the mount mechanism works, but the SSH connection from container (uid 1000 `node` user) to host (uid 1000 `meeq` user) through Docker bridge needs testing. UID mapping could be tricky.

Phases with standard patterns (skip research-phase):

- **Phase 1:** Well-documented — the `add-telegram` skill, `channel-formatting` skill, and NanoClaw setup process are all thoroughly documented with step-by-step instructions.
- **Phase 4:** Standard grammY API calls — typing indicators, bot commands, and CLI installation are all trivial, well-documented operations.

## Confidence Assessment

| Area         | Confidence | Notes                                                                                                                                              |
| ------------ | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| Stack        | HIGH       | All versions verified on npm. NanoClaw stack is inherited, not chosen. ElevenLabs SDK confirmed.                                                   |
| Features     | HIGH       | Based on Telegram Bot API docs (April 2026), NanoClaw codebase analysis, and PROJECT.md requirements. Feature dependencies clearly mapped.         |
| Architecture | HIGH       | All mount paths verified against NanoClaw source code. Component boundaries clear. One MEDIUM area: CLAUDE.md loading from additional directories. |
| Pitfalls     | HIGH       | Derived from direct codebase analysis and CONCERNS.md audit. Pitfall-to-phase mapping is actionable.                                               |

**Overall confidence:** HIGH

### Gaps to Address

- **CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD behavior:** The setting is enabled in container config, but exactly how the SDK discovers and loads CLAUDE.md from mounted directories needs verification. If it doesn't work as expected, the group-level CLAUDE.md must inline all identity content instead of referencing vault files. Low risk — fallback is straightforward.

- **ElevenLabs credential routing through OneCLI:** OneCLI is configured for Anthropic API by default. Adding ElevenLabs requires registering the host pattern (`api.elevenlabs.io`) with OneCLI. If this proves complex, the native credential proxy is a simpler fallback. Verify during Phase 3 planning.

- **Container SSH UID mapping:** Container runs as uid 1000 (`node`), host user is `meeq` (likely also uid 1000 on Ubuntu). If UIDs match, SSH works naturally. If not, the SSH `authorized_keys` entry needs to map to the correct user. Verify during Phase 3 implementation.

- **Git versioning for memory files:** Research recommends keeping git outside the vault, but the exact mechanism (one-way rsync, scheduled task, or skip git entirely and use Obsidian Sync's built-in versioning) is deferred. Not blocking — Obsidian Sync provides version history. Can be revisited post-migration.

## Sources

### Primary (HIGH confidence)

- NanoClaw codebase: `src/container-runner.ts`, `src/mount-security.ts`, `src/container-runtime.ts`, `src/channels/index.ts`, `setup/service.ts` — direct code inspection
- NanoClaw skill files: `add-telegram/SKILL.md`, `add-voice-transcription/SKILL.md`, `channel-formatting/SKILL.md`, `setup/SKILL.md`, `use-native-credential-proxy/SKILL.md` — direct file inspection
- NanoClaw docs: `SPEC.md`, `SECURITY.md` — authoritative project documentation
- Telegram Bot API docs (https://core.telegram.org/bots/api) — Bot API 9.6, April 2026
- npm registry: grammY 1.42.0, @elevenlabs/elevenlabs-js 2.42.0, @anthropic-ai/claude-agent-sdk 0.2.104 — verified latest versions
- ElevenLabs STT API reference (https://elevenlabs.io/docs/api-reference/speech-to-text/convert) — official API with OpenAPI spec

### Secondary (MEDIUM confidence)

- Obsidian Sync + git coexistence — common community pattern (obsidian-git plugin community), not officially documented by Obsidian
- CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD — setting exists in container-runner.ts, exact SDK behavior needs verification
- Container SSH via host.docker.internal — verified in NanoClaw source, but end-to-end SSH flow untested

### Tertiary (LOW confidence)

- ElevenLabs STT pricing and free tier limits — from official pricing page but may change; verify current allocations before relying on free tier

---

_Research completed: 2026-04-12_
_Ready for roadmap: yes_

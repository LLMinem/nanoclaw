# MeeqClaw

## What This Is

A personal AI assistant (Fritz) built on NanoClaw, deployed on a Hetzner VPS, accessible via Telegram. MeeqClaw is meeq's fork of NanoClaw (~4,800 LOC Node.js/TypeScript), replacing OpenClaw (~500k LOC) as the daily-driver AI assistant. Doubles as a hands-on learning project for understanding AI agent architecture -- every change should deepen meeq's understanding of the system.

## Core Value

Fritz is always reachable via Telegram, understands meeq's context through persistent memory, and can take action on meeq's systems.

## Requirements

### Validated

Capabilities already present in the NanoClaw codebase (upstream):

- ✓ Event-driven message router with 2s polling loop — existing
- ✓ Containerized agent execution via Docker + Claude Agent SDK — existing
- ✓ File-based IPC between host and containers (atomic writes, per-group namespacing) — existing
- ✓ Per-group isolation with separate filesystem namespaces — existing
- ✓ SQLite persistence for messages, sessions, groups, tasks, router state — existing
- ✓ Task scheduling (cron, interval, one-time) with drift-resistant timing — existing
- ✓ Concurrency management via GroupQueue (configurable max containers, exponential backoff) — existing
- ✓ Mount security (allowlist, path validation, blocked patterns, symlink resolution) — existing
- ✓ Sender allowlist (per-chat trigger-only/drop modes) — existing
- ✓ Setup CLI with interactive step-based configuration — existing
- ✓ Container build infrastructure (Dockerfile, build script, node:22-slim base) — existing
- ✓ Credential injection via OneCLI gateway (containers never see real API keys) — existing
- ✓ Multi-channel abstraction with factory-pattern registry — existing
- ✓ Session resume across container restarts — existing

### Active

What needs to be built or configured for v1:

- [ ] NanoClaw deployed and running on VPS (Ubuntu 24.04, Hetzner CX32)
- [ ] Telegram channel installed and authenticated
- [ ] Channel formatting (Markdown to Telegram native syntax)
- [ ] Fritz identity adapted for NanoClaw (SOUL.md, USER.md migrated from OpenClaw)
- [ ] Memory files accessible in container via bind mounts
- [ ] MEMORY.md index migrated and adapted from OpenClaw
- [ ] Core workspace files migrated (file-by-file review, OpenClaw references removed)
- [ ] ElevenLabs voice transcription for Telegram voice messages
- [ ] Workspace/vault architecture decided and implemented (memory in Obsidian vault)
- [ ] SSH from container to host for VPS administration
- [ ] Bind mount configuration for projects, memory, vault
- [ ] CLI tool installed (/claw) for terminal-based prompts
- [ ] Credential management configured via OneCLI
- [ ] Upstream update mechanism working (/update-nanoclaw, /update-skills)

### Out of Scope

- Browser automation / Playwright — not day-1, adds container complexity
- WhatsApp, Discord, Slack channels — no use case
- Mac as secondary host — later via Tailscale/SSH, not v1
- Embedding-based memory system — V2+, file-based search sufficient for now
- Agent-as-Copilot inter-agent workflow — valuable but not v1
- Morning digest / motivation recap crons — post-stable, not migration-critical
- Telegram swarm / multi-bot — Week 1+, after base is solid
- X/Twitter integration — Week 1+, after base is solid
- Image vision for Telegram — Week 1+, custom implementation needed
- ChangeDetection.io integration — low priority, post-migration
- Native mode (no container) — keeping containers for security
- Emacs channel — not used
- macOS status bar — only relevant if Mac becomes primary

## Context

**Migration background:** OpenClaw (~500k LOC, 53 config files, 70+ dependencies) lost Anthropic subscription support on April 4, 2026. NanoClaw uses the Claude Agent SDK which remains subscription-compatible. OpenClaw is still running but the migration should complete within 3-4 days.

**Learning priority:** meeq is a solo beginner developer. This project is explicitly a learning vehicle for AI agent development. The agent must not skip the learning process by doing work meeq doesn't understand. Every implementation choice should be explainable.

**ADHD context:** Visible progress matters. Motivational feedback loops (daily/weekly recaps) are a future feature, not v1.

**Existing assets to migrate:**

- `~/clawd/memory/` — ~60+ curated daily and deep memory files (Feb-April 2026)
- `~/clawd/MEMORY.md` — long-term memory index
- `~/clawd/SOUL.md` — Fritz personality definition
- `~/clawd/USER.md` — meeq's profile and preferences
- Additional workspace files (STRATEGY.md, ref/ files) — migrate selectively

**Infrastructure:**

- VPS: Hetzner CX32, Ubuntu 24.04, Docker pre-installed
- Networking: Tailscale connects VPS, MacBook, Raspberry Pi
- Sync: Obsidian Headless on VPS syncs vault across iPhone, MacBook, VPS
- Vault: `~/vaults/meeq-vault/` — the Obsidian vault, synced across devices
- Fork: github.com/LLMinem/nanoclaw (origin), qwibitai/nanoclaw (upstream)
- Branch: `meeq-claw` (all work happens here, `main` tracks upstream)

**Workspace architecture (open):** Memory files should live in the Obsidian vault for cross-device visibility. Core workspace files may live in the vault or separately. Git for memory version history alongside Obsidian Sync is desired but needs research. Agent tooling (`.claude/` sessions, SQLite, IPC) stays inside containers. Multiple bind mounts can cleanly separate concerns. Final architecture requires a dedicated research + decision phase.

## Constraints

- **Timeline**: 3-4 days to functional assistant (OpenClaw still running as fallback)
- **Platform**: Ubuntu 24.04, Hetzner CX32 VPS (primary); macOS MacBook (development only for now)
- **Container**: Docker on Linux (native, no VM overhead)
- **SDK**: Claude Agent SDK (must stay subscription-compatible)
- **Channel**: Telegram primary (sole channel for v1)
- **Branch**: All work on `meeq-claw` branch; `main` tracks upstream
- **Learning**: Changes must be understood, not just applied — this is a learning project
- **Credentials**: OneCLI gateway (setup default)

## Key Decisions

| Decision                                              | Rationale                                                     | Outcome   |
| ----------------------------------------------------- | ------------------------------------------------------------- | --------- |
| VPS as primary host                                   | 24/7 online, Docker pre-installed, reliable                   | ✓ Good    |
| Keep container isolation                              | Security for web content, Bind Mounts + SSH solve convenience | ✓ Good    |
| Host interaction: Mounts + SSH + HTTP + Docker Socket | Covers all use cases cleanly                                  | — Pending |
| OneCLI for credentials                                | Setup default, simpler than rolling own proxy                 | — Pending |
| Fritz identity preserved                              | 2+ months of curated context, personality works               | — Pending |
| Memory in Obsidian vault                              | Cross-device visibility via Obsidian Sync                     | — Pending |
| ElevenLabs over Whisper                               | Better transcription quality (user preference)                | — Pending |
| Use /setup skill for initial setup                    | Comprehensive, handles git/deps/container/channels/service    | — Pending |
| Import workspace files first, adapt after             | Faster path to functional, can iterate                        | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):

1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):

1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---

_Last updated: 2026-04-12 after initialization_

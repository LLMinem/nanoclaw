# Phase 1: VPS Deployment & Telegram Channel - Context

**Gathered:** 2026-04-13
**Status:** Ready for planning

<domain>
## Phase Boundary

Deploy NanoClaw on the Hetzner VPS (Ubuntu 24.04, CX32), install the Telegram channel with message formatting, authenticate the existing Fritz bot, and verify end-to-end messaging. Fritz is reachable on Telegram and responds to text messages with channel-appropriate formatting.

</domain>

<decisions>
## Implementation Decisions

### Telegram Chat Structure

- **D-01:** 1:1 private chat with the existing Fritz bot (direct messages, not a group)
- **D-02:** Migrate the existing Telegram bot from OpenClaw -- no new BotFather registration needed
- **D-03:** Register the 1:1 DM as the NanoClaw "main group"
- **D-04:** Sender allowlist configured with meeq's Telegram user ID only

### Credential Management

- **D-05:** OneCLI gateway for Anthropic OAuth token management (handles token refresh automatically)
- **D-06:** OneCLI scope limited to Anthropic OAuth token only -- no broader secrets migration in this phase
- **D-07:** OneCLI runs as a systemd service on the VPS alongside NanoClaw

### VPS Setup Approach

- **D-08:** Manual step-by-step deployment via SSH directly on the VPS
- **D-09:** User opens an OpenCode session on the VPS and runs commands there (not remotely from Mac)
- **D-10:** The `/setup` skill is consulted as reference, but setup is done manually step-by-step
- **D-11:** Execution is interactive: agent explains each step briefly, waits for user approval, then proceeds
- **D-12:** systemd service for NanoClaw persistence (auto-start on boot, restartable via systemctl, logs via journalctl)

### Skill Installation

- **D-13:** `add-telegram` skill merged into meeq-claw branch (provides Telegram channel module)
- **D-14:** `channel-formatting` skill merged into meeq-claw branch (Markdown to Telegram native syntax)

### The Agent's Discretion

- systemd unit file design (ExecStart, restart policy, dependencies)
- Container build details (image tag, build args)
- Exact order of deployment steps
- Git branch management for skill merges
- .env file structure and contents

</decisions>

<canonical_refs>

## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### NanoClaw Skills

- `.claude/skills/add-telegram/SKILL.md` -- Telegram channel installation instructions, what files it modifies
- `.claude/skills/channel-formatting/SKILL.md` -- Channel formatting installation, Markdown-to-Telegram conversion
- `.claude/skills/setup/SKILL.md` -- Setup CLI reference, step-by-step configuration guide
- `.claude/skills/init-onecli/SKILL.md` -- OneCLI installation and initialization instructions

### Architecture

- `src/channels/registry.ts` -- Channel factory registration pattern (Telegram module must call `registerChannel()`)
- `src/channels/index.ts` -- Barrel file where Telegram import must be added
- `src/container-runner.ts` -- Container spawning, volume mounts, credential injection via OneCLI
- `container/Dockerfile` -- Container image definition (node:22-slim base)
- `container/build.sh` -- Container build script

### Configuration

- `.env.example` -- Environment variable template
- `src/config.ts` -- Runtime constants (paths, timeouts, container image name)
- `src/env.ts` -- Selective .env reader (secrets not loaded into process.env)

### Setup

- `setup/service.ts` -- Service setup step (currently macOS launchd -- needs systemd for Linux)
- `setup/container.ts` -- Container build step
- `setup/register.ts` -- Group registration step
- `setup/environment.ts` -- Environment/credentials step

### Documentation

- `docs/SPEC.md` -- System specification
- `docs/SECURITY.md` -- Security model

</canonical_refs>

<code_context>

## Existing Code Insights

### Reusable Assets

- **Channel registry** (`src/channels/registry.ts`): Factory-pattern registration -- Telegram skill just needs to call `registerChannel('telegram', factory)`
- **Setup CLI** (`setup/`): Step-based modules for timezone, environment, container build, group registration, mounts, verification -- most work on Linux
- **Container build infrastructure** (`container/Dockerfile`, `container/build.sh`): Ready to build on Linux with Docker
- **Mount security** (`src/mount-security.ts`): Allowlist-based mount validation already in place
- **Sender allowlist** (`src/sender-allowlist.ts`): Per-chat sender filtering ready to configure

### Established Patterns

- **Skill installation**: Skills are git branches merged into the working branch; they add files and modify the barrel import
- **Channel self-registration**: Channel modules call `registerChannel()` and are imported via side-effect in `src/channels/index.ts`
- **ESM modules**: All imports use `.js` extension (`import { x } from './y.js'`)
- **Config via .env**: Selective reader in `src/env.ts`, never pollutes `process.env`

### Integration Points

- `src/channels/index.ts` -- Telegram import line must be added here after skill merge
- `src/container-runner.ts` lines 243-249 -- OneCLI credential injection point
- `~/.config/nanoclaw/sender-allowlist.json` -- External config file for sender filtering
- `~/.config/nanoclaw/mount-allowlist.json` -- External config file for mount security

### Platform Concerns

- `setup/service.ts` generates macOS launchd plist -- needs a systemd unit file for Linux VPS
- `setup/platform.ts` has platform detection utilities that may already handle Linux
- `container-runtime.ts` handles Linux host gateway (different from macOS Docker Desktop)

</code_context>

<specifics>
## Specific Ideas

- Migrate the existing Fritz Telegram bot from OpenClaw (same bot token, same @username) -- no new bot creation
- Interactive execution style: agent explains each step, waits for approval, then continues (learning project)
- `ssh vps` alias is configured -- can verify tool availability before deployment session
- Tools likely pre-installed on VPS: Node.js, Docker, git, npm

</specifics>

<deferred>
## Deferred Ideas

- **Multi-session Telegram bots**: User has multiple Telegram bots and wants simultaneous switchable sessions (like ChatGPT). NanoClaw's group architecture supports this (each bot = separate group). Requires dedicated design work -- roadmap backlog item.
- **Broader secrets management overhaul**: User wants to migrate secrets from scattered .env files and ~/.secrets to a proper secrets manager. Orthogonal to NanoClaw, separate project.
- **Adapt setup CLI for Linux**: Modify setup CLI's service step to generate systemd unit files on Linux. Would be a nice contribution but adds scope to Phase 1.

</deferred>

---

_Phase: 01-vps-deployment-telegram-channel_
_Context gathered: 2026-04-13_

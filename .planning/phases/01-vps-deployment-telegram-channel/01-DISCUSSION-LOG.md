# Phase 1: VPS Deployment & Telegram Channel - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md -- this log preserves the alternatives considered.

**Date:** 2026-04-13
**Phase:** 01-vps-deployment-telegram-channel
**Areas discussed:** Telegram chat structure, Credential management, VPS setup approach

---

## Telegram Chat Structure

| Option           | Description                                                                                                     | Selected |
| ---------------- | --------------------------------------------------------------------------------------------------------------- | -------- |
| 1:1 private chat | Direct messages to the bot. Simpler, more private, no group noise. NanoClaw registers this as the 'main group'. | ✓        |
| Telegram group   | A group where you and the bot are members. Allows adding other bots/people later. Needs trigger word.           |          |

**User's choice:** 1:1 private chat (Recommended)
**Notes:** Straightforward choice. Sender allowlist is simple with 1:1 DM -- only the user's Telegram ID.

---

## Bot Identity & Naming

**User's choice:** Not discussed as a gray area -- user clarified they will migrate the existing Fritz Telegram bot from OpenClaw. No new BotFather registration needed.

**Additional context:** User has multiple Telegram bots created hoping for simultaneous chat sessions (like ChatGPT). This didn't work in OpenClaw. User would love multi-session support in NanoClaw but deferred to agent on scope. **Deferred as out-of-scope for Phase 1** -- noted for roadmap backlog.

---

## Credential Management

| Option                  | Description                                                                                        | Selected |
| ----------------------- | -------------------------------------------------------------------------------------------------- | -------- |
| Native credential proxy | Simple: reads API key from .env, injects into container HTTPS traffic. One less service to manage. |          |
| OneCLI gateway          | NanoClaw's default. Manages OAuth lifecycle including token refresh. Extra service to install.     | ✓        |
| You decide              | Let the agent pick.                                                                                |          |

**User's choice:** OneCLI gateway, scoped to Anthropic OAuth token only

**Notes:** User initially considered native proxy but realized manual token refresh would be annoying daily. Chose OneCLI for automatic OAuth token refresh. Explicitly scoped: Anthropic OAuth token only, no broader secrets migration. User also expressed interest in improving their overall secrets management (scattered .env files, ~/.secrets) but agreed that's a separate project.

---

## VPS Setup Approach

| Option                    | Description                                                                                                    | Selected |
| ------------------------- | -------------------------------------------------------------------------------------------------------------- | -------- |
| Manual step-by-step       | Clone repo, install deps, build container, configure .env, create systemd unit, register group -- all via SSH. | ✓        |
| Adapt setup CLI for Linux | Modify the setup CLI's service step to detect Linux and generate systemd unit files.                           |          |
| You decide                | Let the agent pick.                                                                                            |          |

**User's choice:** Manual step-by-step, using `/setup` skill as reference

**Notes:** User wants interactive execution where agent explains each step and waits for approval. This is a learning project -- understanding matters more than automation. User will SSH into the VPS and open an OpenCode session there to run commands directly. The `/setup` skill should be read for reference but not blindly followed.

### Follow-up: Execution Location

| Option                           | Description                                         | Selected |
| -------------------------------- | --------------------------------------------------- | -------- |
| SSH into VPS, run commands there | Run commands directly on VPS. Most straightforward. | ✓        |
| From MacBook via SSH commands    | Stay on Mac, remote-execute via SSH.                |          |
| You decide                       | Let the agent pick.                                 |          |

**User's choice:** SSH into VPS, open OpenCode session there
**Notes:** User has `ssh vps` alias configured. Offered to let agent verify tool availability.

### Follow-up: Service Persistence

| Option              | Description                                                     | Selected |
| ------------------- | --------------------------------------------------------------- | -------- |
| systemd service     | Standard Linux. Auto-starts on boot, restartable via systemctl. | ✓        |
| tmux/screen session | Manual, doesn't survive reboots.                                |          |
| You decide          | Let the agent pick.                                             |          |

**User's choice:** systemd service (Recommended)

---

## The Agent's Discretion

- systemd unit file design details
- Container build specifics
- Deployment step ordering
- Git branch management for skill merges
- .env file structure

## Deferred Ideas

- **Multi-session Telegram bots**: Multiple bots for simultaneous switchable sessions. NanoClaw group architecture supports it. Needs dedicated design work. (User request)
- **Broader secrets management**: User wants to migrate scattered secrets to proper management. Separate project. (User interest)
- **Linux setup CLI adaptation**: Modify service step for systemd. Nice but adds scope. (Agent suggestion)

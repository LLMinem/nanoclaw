# Feature Landscape

**Domain:** Personal AI Assistant (Telegram-based, always-on, solo developer)
**Researched:** 2026-04-12
**Overall confidence:** HIGH — based on Telegram Bot API docs (April 2026), NanoClaw codebase analysis, and established patterns for personal AI assistants

## Table Stakes

Features the user expects from a daily-driver personal AI assistant. Missing = assistant feels broken or unusable.

| Feature                        | Why Expected                                                                                    | Complexity | Already in NanoClaw     | Notes                                                                                                                                                         |
| ------------------------------ | ----------------------------------------------------------------------------------------------- | ---------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Telegram text messaging        | Core interaction loop — type message, get response                                              | Low        | Via skill merge         | `add-telegram` skill provides grammy-based channel. Must merge, configure, register.                                                                          |
| Message persistence            | Conversations survive restarts. User expects continuity.                                        | Low        | Yes (SQLite)            | Messages stored in `messages` table. Sessions resume across restarts.                                                                                         |
| Session continuity             | Agent remembers conversation context within a session                                           | Low        | Yes (Claude SDK)        | Session IDs persisted in SQLite `sessions` table, keyed by group folder.                                                                                      |
| Channel formatting             | Responses render correctly in Telegram (bold, italic, code)                                     | Low        | Via skill merge         | `channel-formatting` skill converts Claude's Markdown to Telegram-native syntax. Without this, raw `**asterisks**` appear.                                    |
| Sender allowlist               | Only authorized users can trigger the bot                                                       | Low        | Yes                     | `sender-allowlist.json` controls per-chat triggers. Critical for a public-facing Telegram bot.                                                                |
| Voice message transcription    | User sends voice notes → agent reads them. Telegram voice messages are native and frequent.     | Medium     | Partial (WhatsApp only) | `add-voice-transcription` skill is WhatsApp-only. Telegram needs custom implementation: download `.ogg` from Telegram Bot API, send to transcription service. |
| `/start` and `/help` commands  | Telegram Bot API requires global commands. User expects `/start` to work.                       | Low        | No                      | grammy framework handles this, but NanoClaw's Telegram channel doesn't implement bot commands yet. Needs trivial handler.                                     |
| Identity/personality (SOUL.md) | Fritz has 2+ months of curated personality. Assistant must feel like Fritz, not generic Claude. | Low        | Yes (mount system)      | Migrated via group CLAUDE.md or SOUL.md in mounted workspace. Claude Agent SDK reads CLAUDE.md files from all mounted directories.                            |
| User profile (USER.md)         | Agent knows user's preferences, context, habits                                                 | Low        | Yes (mount system)      | USER.md accessible via bind mounts. `CLAUDE_CODE_ADDITIONAL_DIRECTORIES_CLAUDE_MD=1` loads CLAUDE.md from all mounted dirs.                                   |
| Always-on availability (24/7)  | Personal assistant must be reachable anytime. Can't require laptop to be open.                  | Low        | Yes (VPS)               | Hetzner CX32 VPS, systemd service. Already architected for this.                                                                                              |
| Error recovery                 | Bot recovers from crashes, retries failed messages                                              | Low        | Yes                     | `GroupQueue` exponential backoff, stale session detection, startup recovery via `recoverPendingMessages()`.                                                   |

## Differentiators

Features that set MeeqClaw apart from a basic chatbot. Not expected by every user, but high-value for this specific use case.

### Memory & Context System

| Feature                                               | Value Proposition                                                                                                 | Complexity | Already in NanoClaw     | Notes                                                                                                                                                                                                                                 |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------- | ---------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| File-based long-term memory (MEMORY.md + daily files) | Agent recalls past conversations, projects, decisions across sessions. Core differentiator vs stateless chatbots. | Medium     | Partial (mount system)  | NanoClaw supports bind mounts and CLAUDE.md loading from additional directories. The memory INDEX file (MEMORY.md) and ~60 daily files from OpenClaw need migration. Agent reads them via filesystem, writes via standard file tools. |
| Claude SDK auto-memory                                | Agent automatically persists user preferences between sessions without manual memory management                   | Low        | Yes                     | `CLAUDE_CODE_DISABLE_AUTO_MEMORY: '0'` already enabled in settings.json. SDK stores preferences in `.claude/` directory.                                                                                                              |
| Obsidian vault integration                            | Memory visible on iPhone/MacBook via Obsidian Sync. Cross-device knowledge access without SSH.                    | Medium     | No (needs mount config) | Requires bind mount from `~/vaults/meeq-vault/` into container. Obsidian Headless on VPS syncs vault. Architecture decision needed: memory inside vault vs separate.                                                                  |
| Memory search (grep-based)                            | Find relevant past context quickly. File-based grep is sufficient for ~100 files.                                 | Low        | Yes (via Claude tools)  | Claude Agent SDK provides `Grep`, `Glob`, `Read` tools natively. Agent can search mounted memory directories. No embedding system needed at this scale.                                                                               |

### VPS System Administration

| Feature                     | Value Proposition                                                            | Complexity | Already in NanoClaw  | Notes                                                                                                                                                                                           |
| --------------------------- | ---------------------------------------------------------------------------- | ---------- | -------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SSH from container to host  | "Check my server status" "Restart that service" — agent manages VPS directly | High       | No                   | Container-to-host SSH requires: SSH key generation, key injection into container, host `authorized_keys` config, restricting commands via `ForceCommand` or sudo allowlist. Security-sensitive. |
| Docker socket access        | Agent can manage other containers on VPS                                     | Medium     | No                   | Mount Docker socket read-only for inspection, writable for management. Risk: container escape if agent is compromised. Recommend read-only initially.                                           |
| System monitoring via tasks | Scheduled checks: disk space, service health, cert expiry → proactive alerts | Medium     | Yes (task scheduler) | Task scheduler supports cron, interval, and one-time tasks with script pre-checks. Script can check conditions and only wake agent when threshold exceeded.                                     |

### Workspace & File Management

| Feature                            | Value Proposition                                                                      | Complexity | Already in NanoClaw    | Notes                                                                                                         |
| ---------------------------------- | -------------------------------------------------------------------------------------- | ---------- | ---------------------- | ------------------------------------------------------------------------------------------------------------- |
| Project directory mounts           | Agent reads/edits code in user's projects. "Look at my nanoclaw code" works naturally. | Low        | Yes (additionalMounts) | `containerConfig.additionalMounts` per group, validated against allowlist. Mount security prevents traversal. |
| Global workspace files (CLAUDE.md) | Shared context across all groups — coding standards, preferences, tool usage patterns  | Low        | Yes                    | `groups/global/` directory mounted read-only to all non-main groups.                                          |
| Atomic file operations             | Agent writes files safely (no partial writes on crash)                                 | Low        | Yes (SDK built-in)     | Claude Agent SDK `Write` and `Edit` tools handle this.                                                        |

### Voice Handling

| Feature                            | Value Proposition                                                                              | Complexity | Already in NanoClaw | Notes                                                                                                                                                                                                                 |
| ---------------------------------- | ---------------------------------------------------------------------------------------------- | ---------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Telegram voice transcription (STT) | Telegram voice notes are the #1 mobile input method. Without this, voice messages are opaque.  | Medium     | No                  | Needs: Telegram `getFile` → download `.ogg` → convert/send to transcription API. Options: OpenAI Whisper API ($0.006/min), ElevenLabs (user preference for quality), or local whisper.cpp (free, but VPS has no GPU). |
| Text-to-speech responses (TTS)     | Agent "speaks back" — send audio messages. Nice for hands-free usage.                          | High       | No                  | Telegram Bot API supports `sendVoice`. Pipeline: generate audio via ElevenLabs/OpenAI TTS → send as voice message. Not table stakes — text responses work fine. Defer to v2.                                          |
| Video note transcription           | Telegram "video circles" also contain audio. Same pipeline as voice but need video extraction. | Low        | No                  | If voice transcription works, extending to video notes is trivial (extract audio track).                                                                                                                              |

### Scheduled/Automated Tasks

| Feature                          | Value Proposition                                                                  | Complexity | Already in NanoClaw            | Notes                                                                                                                                                    |
| -------------------------------- | ---------------------------------------------------------------------------------- | ---------- | ------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cron-based recurring tasks       | "Remind me every morning at 9am" "Check X daily" — scheduled agent invocations     | Low        | Yes                            | Full implementation: cron, interval, once. Script pre-checks for conditional execution. Drift-resistant timing.                                          |
| One-time reminders               | "Remind me about the meeting at 3pm"                                               | Low        | Yes                            | `schedule_type: 'once'` with local timestamp.                                                                                                            |
| Conditional task execution       | Script runs before agent wakes — only triggers if condition met (e.g., disk > 80%) | Low        | Yes                            | `script` field on tasks: bash script outputs `{ "wakeAgent": boolean, "data"?: any }`. Agent only wakes if `wakeAgent: true`.                            |
| Morning digest                   | Daily summary of calendar, weather, tasks, recent memory highlights                | Medium     | No (needs task + integrations) | Task scheduler is ready. Needs: prompt engineering for digest format, optional API integrations (weather, calendar). Good first scheduled task to build. |
| Motivation/accountability recaps | Weekly progress summary, ADHD-friendly reinforcement of accomplishments            | Medium     | No (needs task + memory)       | Requires reading memory files to summarize week's progress. Task scheduler handles scheduling. Prompt engineering for motivational tone.                 |

### Telegram Bot Features (Platform Integration)

| Feature                    | Value Proposition                                                                     | Complexity | Already in NanoClaw     | Notes                                                                                                                                                                                                      |
| -------------------------- | ------------------------------------------------------------------------------------- | ---------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Inline keyboards (buttons) | Structured interactions: "Did you mean X or Y?" with tap-to-select                    | Medium     | No                      | grammy supports `InlineKeyboard`. Requires: extending IPC protocol to support keyboard markup, host-side callback query handling. Good for confirmation flows but not v1.                                  |
| Bot commands menu          | `/tasks`, `/memory`, `/status` visible in Telegram command menu                       | Low        | No                      | grammy `bot.api.setMyCommands()`. Pure convenience — agent already understands natural language. Low effort, nice polish.                                                                                  |
| Image understanding        | User sends photo → agent sees and responds. Screenshots, receipts, whiteboard photos. | Medium     | Partial (WhatsApp only) | `add-image-vision` skill is WhatsApp-specific (Baileys download). Telegram needs: `bot.api.getFile()` → download → resize → base64 → multimodal content block. Same architecture, different download path. |
| File/document handling     | User sends PDF/document → agent reads content                                         | Medium     | No                      | Telegram Bot API provides `getFile` for documents. Combine with PDF extraction (`pdftotext`). Text files can be read directly.                                                                             |
| Message reactions          | React with emoji to acknowledge or respond non-verbally                               | Low        | No                      | Telegram Bot API `setMessageReaction`. grammy supports this. Nice UX touch (👍 to acknowledge a command).                                                                                                  |
| Typing indicator           | Bot shows "typing..." while processing                                                | Low        | No                      | `sendChatAction('typing')`. grammy supports this. Important for UX — long Claude responses take 5-30s. Without indicator, user thinks bot is dead.                                                         |
| Message editing            | Edit previous response instead of sending new message (for streaming)                 | Medium     | No                      | Telegram `editMessageText`. Could enable streaming-like UX where response builds up. Adds complexity to output pipeline. Defer.                                                                            |
| Group chat support         | Bot works in group chats (family, team) with @mention triggers                        | Low        | Yes (architecture)      | NanoClaw's multi-group model with per-group isolation and trigger patterns already supports this. Privacy mode must be disabled for the bot to see all messages.                                           |
| Deep linking               | Share bot link with parameters for specific actions                                   | Low        | No                      | `https://t.me/bot?start=param`. grammy handles this. Not needed for personal use.                                                                                                                          |

### Cross-Device Accessibility

| Feature                 | Value Proposition                                                        | Complexity | Already in NanoClaw     | Notes                                                                                                            |
| ----------------------- | ------------------------------------------------------------------------ | ---------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------- |
| Telegram on any device  | Access Fritz from iPhone, MacBook, iPad — wherever Telegram runs         | Low        | Yes (inherent)          | Telegram is cross-device by design. Bot on VPS is accessible from any Telegram client.                           |
| CLI tool (`/claw`)      | Terminal-based prompts without opening Telegram. For developer workflow. | Low        | Yes (via skill)         | `claw` CLI skill: Python script that sends prompts directly to container. Needs install on VPS.                  |
| Obsidian vault sync     | View/search agent's memory from any device via Obsidian                  | Medium     | No (needs architecture) | Obsidian Headless on VPS + Obsidian Sync. Vault syncs to iPhone/MacBook. Agent writes memory to vault directory. |
| SSH from MacBook to VPS | Direct `claw` access from MacBook via Tailscale                          | Low        | Yes (infra exists)      | Tailscane connects VPS and MacBook. SSH already works. Can run `claw` remotely.                                  |

## Anti-Features

Features to explicitly NOT build. Each represents a trap that adds complexity without proportional value.

| Anti-Feature                              | Why Avoid                                                                                                                       | What to Do Instead                                                                                                                |
| ----------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| Embedding-based semantic memory           | Massive complexity (vector DB, embedding pipeline, retrieval tuning) for ~100 memory files. Grep works perfectly at this scale. | File-based memory with grep search. Claude's tools already support `Grep` and `Glob`. Revisit only if memory exceeds 1000+ files. |
| Multi-channel (WhatsApp, Discord, Slack)  | Zero use case. Solo user on Telegram only. Each channel adds dependency surface and maintenance.                                | Telegram only. NanoClaw's multi-channel abstraction is there if needed later, but don't install unnecessary channels.             |
| Web dashboard / admin UI                  | Engineering effort for a solo user who has terminal access. Over-engineering.                                                   | Use Telegram commands, `claw` CLI, and direct SSH for administration.                                                             |
| Agent-as-Copilot / inter-agent delegation | Cool but premature. Complex orchestration for a system that's still being bootstrapped.                                         | Single agent with task scheduling. Agent Teams (swarm) is a v2 feature after base is solid.                                       |
| Browser automation (Playwright)           | Adds ~500MB to container image. Complex to maintain. Limited use case for personal assistant.                                   | Use X integration skill (Playwright-based) only when X/Twitter is needed. Don't make it a general feature.                        |
| Native mode (no container)                | Loses security boundary. Container isolation prevents agent from modifying host system accidentally.                            | Keep containers. Bind mounts + SSH solve all convenience needs without sacrificing security.                                      |
| Real-time streaming to Telegram           | Telegram doesn't support true streaming. Message editing every 100ms would hit rate limits.                                     | Send complete responses. Use typing indicator for UX. Consider chunked "thinking..." messages for very long tasks.                |
| Custom Telegram Mini App                  | Massive engineering effort for a feature set achievable with text + buttons.                                                    | Use inline keyboards for structured input when needed. Text is the primary interface.                                             |
| WhatsApp as fallback channel              | OpenClaw used WhatsApp, but the migration is FROM WhatsApp. Adding it back creates two maintenance surfaces.                    | Telegram only. Full commitment to the new channel.                                                                                |
| Morning digest with 10 API integrations   | Scope creep. Start simple.                                                                                                      | Morning digest v1: read recent memory files + summarize. Add weather/calendar API later as individual enhancements.               |

## Feature Dependencies

```
Telegram channel (add-telegram) → Voice transcription (needs Telegram file download)
Telegram channel → Image vision (needs Telegram image download)
Telegram channel → Channel formatting (needs channel name in output pipeline)
Telegram channel → Bot commands menu (needs grammy bot instance)
Telegram channel → Typing indicator (needs grammy bot API)
Telegram channel → Message reactions (needs grammy bot API)

Bind mount config → Memory file access (agent reads/writes MEMORY.md, daily files)
Bind mount config → Obsidian vault integration (vault directory mounted into container)
Bind mount config → Project directory access (code editing, navigation)

SSH container-to-host → VPS administration (run commands on host)
SSH container-to-host → Docker socket management (manage other containers)

Task scheduler (existing) → Morning digest (cron task + prompt)
Task scheduler (existing) → Motivation recaps (cron task + memory reading)
Task scheduler (existing) → System monitoring (cron task + script pre-check)

Memory file migration → Memory search works (files must be in place)
Memory file migration → Morning digest (reads recent memory)
Memory file migration → Motivation recaps (reads weekly progress)

Identity migration (SOUL.md, USER.md) → Fritz personality (agent behaves as Fritz)
```

## MVP Recommendation

Prioritize for v1 (3-4 day migration window):

1. **Telegram channel + formatting** — Core interaction. Merge `add-telegram` and `channel-formatting` skills. Configure, register, verify.
2. **Identity migration** — SOUL.md, USER.md, workspace files into group CLAUDE.md or mounted directories. Fritz must feel like Fritz from message #1.
3. **Memory file migration** — Migrate ~60 daily files + MEMORY.md index from OpenClaw. Configure bind mounts. Verify agent can read/search.
4. **Voice transcription for Telegram** — Custom implementation needed (not available as skill). Download `.ogg` from Telegram → transcription service. Use OpenAI Whisper API initially (proven, cheap at $0.006/min). Switch to ElevenLabs if quality is better.
5. **CLI tool** — Install `claw` on VPS for terminal access during development and debugging.
6. **Typing indicator** — Low-effort, high-impact UX. Add `sendChatAction('typing')` when processing starts.

Defer to Week 1+:

- **SSH from container to host** — Security requires careful setup. Not migration-critical.
- **Morning digest / motivation recaps** — Task scheduler exists; need prompt engineering, not infrastructure.
- **X/Twitter integration** — Separate skill, adds Playwright dependency.
- **Telegram swarm / multi-bot** — Requires multiple bot tokens, complex orchestration.
- **Image vision for Telegram** — Needs custom download path. Text-only is fine for migration.
- **Inline keyboards / bot commands** — Polish features. Natural language works.

## Complexity Estimates

| Feature                        | Effort    | Risk   | Notes                                                                     |
| ------------------------------ | --------- | ------ | ------------------------------------------------------------------------- |
| Telegram channel merge         | 1-2 hours | Low    | Well-documented skill, merge + configure                                  |
| Channel formatting merge       | 30 min    | Low    | Separate skill merge                                                      |
| Identity migration             | 1-2 hours | Low    | File copying + editing                                                    |
| Memory file migration          | 2-4 hours | Medium | Architecture decision (vault vs separate), bind mount config              |
| Voice transcription (Telegram) | 4-8 hours | Medium | Custom implementation: Telegram file download + Whisper API integration   |
| CLI tool install               | 30 min    | Low    | Copy script, symlink, test                                                |
| Typing indicator               | 30 min    | Low    | Single `sendChatAction` call in message handler                           |
| SSH container-to-host          | 4-8 hours | High   | Security-sensitive: key management, command restriction                   |
| Morning digest task            | 2-4 hours | Low    | Prompt engineering + cron task                                            |
| Image vision (Telegram)        | 4-6 hours | Medium | Custom Telegram file download, reuse image processing from WhatsApp skill |
| Bot commands menu              | 1 hour    | Low    | `setMyCommands` in Telegram channel init                                  |
| Inline keyboards               | 4-8 hours | Medium | IPC protocol extension, callback query handling                           |

## Sources

- Telegram Bot API documentation (https://core.telegram.org/bots/api) — Bot API 9.6, April 2026 [HIGH confidence]
- Telegram Bot Features guide (https://core.telegram.org/bots/features) [HIGH confidence]
- NanoClaw codebase analysis: `src/`, `container/`, `.claude/skills/` [HIGH confidence]
- NanoClaw skill documentation: `add-telegram`, `add-voice-transcription`, `channel-formatting`, `add-image-vision`, `claw`, `add-telegram-swarm`, `x-integration`, `use-local-whisper` [HIGH confidence]
- PROJECT.md requirements and constraints [HIGH confidence]
- ARCHITECTURE.md system analysis [HIGH confidence]
- OpenAI Whisper API pricing: training data knowledge, $0.006/min [MEDIUM confidence — verify current pricing]
- ElevenLabs transcription capabilities: training data only [LOW confidence — verify current API offerings]

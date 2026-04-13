# Requirements: MeeqClaw

**Defined:** 2026-04-12
**Core Value:** Fritz is always reachable via Telegram, understands meeq's context through persistent memory, and can take action on meeq's systems.

## v1 Requirements

Requirements for initial release. Each maps to roadmap phases.

### Setup & Deployment

- [ ] **SETUP-01**: NanoClaw deployed and running on VPS (Ubuntu 24.04, Hetzner CX32) via systemd service
- [ ] **SETUP-02**: Docker container image builds successfully on VPS
- [ ] **SETUP-03**: OneCLI credential management configured and verified (Claude subscription token)
- [ ] **SETUP-04**: Repository cloned on VPS, `main` branch checked out and active

### Telegram Channel

- [ ] **TELE-01**: Telegram channel installed via add-telegram skill merge
- [ ] **TELE-02**: Channel formatting installed via channel-formatting skill merge (Markdown to Telegram syntax)
- [ ] **TELE-03**: Bot registered with BotFather and authenticated with token
- [ ] **TELE-04**: Main group registered and responding to text messages
- [ ] **TELE-05**: Sender allowlist configured so only meeq can trigger the bot
- [ ] **TELE-06**: Typing indicator shows "typing..." while agent processes messages
- [ ] **TELE-07**: Bot commands menu (/start, /help) visible in Telegram command picker

### Identity & Workspace

- [ ] **IDENT-01**: Fritz personality (SOUL.md) migrated from OpenClaw and adapted for NanoClaw environment
- [ ] **IDENT-02**: User profile (USER.md) migrated from OpenClaw and adapted
- [ ] **IDENT-03**: Core workspace files reviewed file-by-file, OpenClaw-specific references removed or updated
- [ ] **IDENT-04**: Group CLAUDE.md configured with Fritz identity context so agent behaves as Fritz from first message

### Memory

- [ ] **MEM-01**: Memory files (~60 daily and deep memory files, Feb-April 2026) migrated from OpenClaw ~/clawd/memory/
- [ ] **MEM-02**: MEMORY.md long-term memory index migrated from OpenClaw and adapted
- [ ] **MEM-03**: Memory directory accessible in container via bind mount
- [ ] **MEM-04**: Agent can search memory files using Claude's built-in Grep/Glob/Read tools
- [ ] **MEM-05**: Memory files live in Obsidian vault subfolder for cross-device visibility via Obsidian Sync
- [ ] **MEM-06**: Workspace/vault architecture decided and documented (bind mount structure, git strategy for memory)

### Voice

- [ ] **VOICE-01**: Incoming Telegram voice notes auto-transcribed and injected as text prompt
- [ ] **VOICE-02**: Transcribed messages prefixed with "Voice message from user:" or similar disclaimer
- [ ] **VOICE-03**: Transcription API, model, and parameters researched and documented (ElevenLabs Scribe vs alternatives)

### CLI & Access

- [ ] **CLI-01**: /claw CLI tool installed and working on VPS for terminal-based prompts
- [ ] **CLI-02**: claw accessible from MacBook via Tailscale SSH connection

### Maintenance

- [ ] **MAINT-01**: /update-nanoclaw skill working for upstream sync (cherry-pick, selective merge)
- [ ] **MAINT-02**: /update-skills skill working for installed skill branch updates
- [ ] **MAINT-03**: /debug skill available for container troubleshooting
- [ ] **MAINT-04**: /customize skill available for future modifications

## v2 Requirements

Deferred to after v1 migration is complete. Tracked but not in current roadmap.

### VPS System Administration

- **ADMIN-01**: SSH from container to host for VPS command execution (restricted key, ForceCommand)
- **ADMIN-02**: Docker socket access (read-only) for container inspection
- **ADMIN-03**: System monitoring via scheduled tasks (disk, services, certs) with conditional alerts

### Voice (Extended)

- **VOICE-04**: Text-to-speech responses via ElevenLabs (agent sends audio replies)
- **VOICE-05**: Video note transcription (Telegram video circles, extract audio track)

### Scheduled Automation

- **AUTO-01**: Morning digest cron job (recent memory summary, delivered via Telegram)
- **AUTO-02**: Weekly motivation/accountability recap (ADHD-friendly progress summary)

### Telegram (Extended)

- **TELE-08**: X/Twitter integration via browser automation
- **TELE-09**: Telegram swarm / multi-bot with separate bot identities
- **TELE-10**: Image vision for Telegram (photos, screenshots → multimodal content)
- **TELE-11**: Inline keyboards for structured interactions
- **TELE-12**: Message reactions (emoji acknowledgments)
- **TELE-13**: PDF/document handling (extract text from uploaded files)

### System (Extended)

- **SYS-01**: Agent-as-Copilot inter-agent workflow (strategic advisor feeding context to coding agents)
- **SYS-02**: Mac as secondary host via Tailscale (agent can reach MacBook when online)
- **SYS-03**: Embedding-based semantic memory (vector DB, only if file count exceeds 1000+)
- **SYS-04**: ChangeDetection.io integration via REST API (Raspberry Pi)

## Out of Scope

Explicitly excluded. Documented to prevent scope creep.

| Feature                               | Reason                                                       |
| ------------------------------------- | ------------------------------------------------------------ |
| WhatsApp channel                      | No use case, migrating away from WhatsApp-based assistant    |
| Discord channel                       | No use case                                                  |
| Slack channel                         | No use case                                                  |
| Web dashboard / admin UI              | Over-engineering for a solo user with terminal access        |
| Browser automation as general feature | Adds ~500MB to container, limited personal assistant value   |
| Native mode (no container)            | Loses security boundary; bind mounts + SSH solve convenience |
| Real-time streaming to Telegram       | Telegram doesn't support streaming; rate limits on edits     |
| Custom Telegram Mini App              | Massive effort for features achievable with text + buttons   |
| Emacs channel                         | Not used                                                     |
| macOS status bar                      | Only relevant if Mac becomes primary host                    |

## Traceability

Which phases cover which requirements. Updated during roadmap creation.

| Requirement | Phase   | Status  |
| ----------- | ------- | ------- |
| SETUP-01    | Phase 1 | Pending |
| SETUP-02    | Phase 1 | Pending |
| SETUP-03    | Phase 1 | Pending |
| SETUP-04    | Phase 1 | Pending |
| TELE-01     | Phase 1 | Pending |
| TELE-02     | Phase 1 | Pending |
| TELE-03     | Phase 1 | Pending |
| TELE-04     | Phase 1 | Pending |
| TELE-05     | Phase 1 | Pending |
| TELE-06     | Phase 5 | Pending |
| TELE-07     | Phase 5 | Pending |
| IDENT-01    | Phase 2 | Pending |
| IDENT-02    | Phase 2 | Pending |
| IDENT-03    | Phase 2 | Pending |
| IDENT-04    | Phase 2 | Pending |
| MEM-01      | Phase 3 | Pending |
| MEM-02      | Phase 3 | Pending |
| MEM-03      | Phase 3 | Pending |
| MEM-04      | Phase 3 | Pending |
| MEM-05      | Phase 3 | Pending |
| MEM-06      | Phase 2 | Pending |
| VOICE-01    | Phase 4 | Pending |
| VOICE-02    | Phase 4 | Pending |
| VOICE-03    | Phase 4 | Pending |
| CLI-01      | Phase 5 | Pending |
| CLI-02      | Phase 5 | Pending |
| MAINT-01    | Phase 6 | Pending |
| MAINT-02    | Phase 6 | Pending |
| MAINT-03    | Phase 6 | Pending |
| MAINT-04    | Phase 6 | Pending |

**Coverage:**

- v1 requirements: 30 total
- Mapped to phases: 30
- Unmapped: 0

---

_Requirements defined: 2026-04-12_
_Last updated: 2026-04-12 after roadmap creation_

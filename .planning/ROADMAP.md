# Roadmap: MeeqClaw

## Overview

MeeqClaw replaces OpenClaw as meeq's daily-driver AI assistant. The migration deploys a NanoClaw fork on a Hetzner VPS with Telegram as the sole channel, then layers Fritz's identity, memory, and voice capabilities on top. Six phases move from "NanoClaw running on VPS" to "Fritz fully operational with voice, CLI, and maintenance tooling." The critical path is Phases 1-3: once Fritz is on Telegram with personality and memory, the OpenClaw migration is functionally complete.

## Phases

**Phase Numbering:**

- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: VPS Deployment & Telegram Channel** - Deploy NanoClaw on VPS, install Telegram channel with formatting, authenticate and verify end-to-end messaging
- [ ] **Phase 2: Workspace Architecture & Identity** - Decide workspace/vault structure, migrate Fritz's personality and user profile, configure identity context
- [ ] **Phase 3: Memory Migration** - Migrate ~60 memory files to Obsidian vault, set up bind mounts, verify agent can search and read memory
- [ ] **Phase 4: Voice Transcription** - Research and implement ElevenLabs Scribe v2 for Telegram voice note transcription
- [ ] **Phase 5: CLI & Telegram Polish** - Install claw CLI, add typing indicator and bot commands menu
- [ ] **Phase 6: Maintenance & Operations** - Verify upstream sync, skill updates, debug, and customization tools are working

## Phase Details

### Phase 1: VPS Deployment & Telegram Channel

**Goal**: Fritz is reachable on Telegram and responds to text messages with channel-appropriate formatting
**Depends on**: Nothing (first phase)
**Requirements**: SETUP-01, SETUP-02, SETUP-03, SETUP-04, TELE-01, TELE-02, TELE-03, TELE-04, TELE-05
**Success Criteria** (what must be TRUE):

1. User can send a text message to the Fritz Telegram bot and receive a coherent response
2. Bot responses use Telegram-native formatting (bold, italic, code blocks render correctly)
3. Only meeq's messages trigger the bot — messages from other users in a group are ignored
4. NanoClaw service restarts automatically after VPS reboot (systemd persistence)
5. Container image builds and runs successfully on VPS (Docker verified)

**Execution approach:** Run NanoClaw skills interactively on the VPS, one session per skill:

1. `/setup` — Install dependencies, build container, configure timezone and environment
2. `/add-telegram` — Merge Telegram channel code, collect bot token, configure
3. `/init-onecli` — Install OneCLI credential gateway, register Anthropic OAuth token
4. Manual: Create systemd service, register main group, set up sender allowlist, verify e2e

### Phase 2: Workspace Architecture & Identity

**Goal**: Fritz speaks with his established personality and knows who meeq is, with workspace file locations decided and documented
**Depends on**: Phase 1
**Requirements**: MEM-06, IDENT-01, IDENT-02, IDENT-03, IDENT-04
**Success Criteria** (what must be TRUE):

1. Workspace/vault architecture is documented — where identity files, memory, and workspace files live, how they're mounted, and git/sync strategy
2. Fritz responds with his established personality traits (tone, humor, speaking style match SOUL.md)
3. Fritz demonstrates knowledge of meeq's preferences and context from USER.md without being told
4. Workspace files contain no stale OpenClaw references (clawd paths, ocx commands, openclaw mentions removed)
   **Plans**: TBD

### Phase 3: Memory Migration

**Goal**: Fritz can recall past conversations, decisions, and context from 2+ months of curated memory files
**Depends on**: Phase 2
**Requirements**: MEM-01, MEM-02, MEM-03, MEM-04, MEM-05
**Success Criteria** (what must be TRUE):

1. User can ask Fritz about a past event (e.g., "what did we decide about X in March?") and get an accurate answer sourced from memory files
2. Memory files are visible in Obsidian on iPhone and MacBook via Obsidian Sync
3. Agent can create new memory files that persist across container restarts and appear in Obsidian
4. MEMORY.md index is accessible and agent uses it to navigate the memory collection
   **Plans**: TBD

### Phase 4: Voice Transcription

**Goal**: User can send voice notes on Telegram and Fritz responds as if it were a text message
**Depends on**: Phase 1
**Requirements**: VOICE-01, VOICE-02, VOICE-03
**Success Criteria** (what must be TRUE):

1. User sends a Telegram voice note and receives a text response addressing the spoken content
2. Transcribed text appears in the conversation with a "Voice message" prefix so Fritz knows input was spoken
3. Transcription API choice (ElevenLabs Scribe v2 vs alternatives) is researched and documented with rationale
   **Plans**: TBD

### Phase 5: CLI & Telegram Polish

**Goal**: Fritz is accessible from the terminal and the Telegram bot feels polished with standard UX conventions
**Depends on**: Phase 1
**Requirements**: CLI-01, CLI-02, TELE-06, TELE-07
**Success Criteria** (what must be TRUE):

1. User can run `claw "prompt"` on the VPS and receive a response in the terminal
2. User can run `claw` from MacBook via Tailscale SSH and get a response
3. Telegram shows "typing..." indicator while Fritz processes a message
4. Telegram command picker shows bot commands (/start, /help) when user types "/"
   **Plans**: TBD

### Phase 6: Maintenance & Operations

**Goal**: Tools for keeping MeeqClaw updated, debugged, and customizable are verified working
**Depends on**: Phase 1
**Requirements**: MAINT-01, MAINT-02, MAINT-03, MAINT-04
**Success Criteria** (what must be TRUE):

1. Running /update-nanoclaw shows available upstream changes and can selectively merge them
2. Running /update-skills checks installed skill branches for updates
3. Running /debug provides useful diagnostic information about container state and logs
4. Running /customize can guide through adding a new capability or integration
   **Plans**: TBD

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5 → 6

| Phase                                | Status      | Completed |
| ------------------------------------ | ----------- | --------- |
| 1. VPS Deployment & Telegram Channel | Not started | -         |
| 2. Workspace Architecture & Identity | Not started | -         |
| 3. Memory Migration                  | Not started | -         |
| 4. Voice Transcription               | Not started | -         |
| 5. CLI & Telegram Polish             | Not started | -         |
| 6. Maintenance & Operations          | Not started | -         |

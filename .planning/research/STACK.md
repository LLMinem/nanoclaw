# Technology Stack

**Project:** MeeqClaw (NanoClaw fork → personal AI assistant)
**Researched:** 2026-04-12
**Mode:** Ecosystem (Stack dimension)

## Recommended Stack

### Core Framework (Inherited from NanoClaw)

| Technology                       | Version                  | Purpose           | Why                                                                                                        | Confidence |
| -------------------------------- | ------------------------ | ----------------- | ---------------------------------------------------------------------------------------------------------- | ---------- |
| Node.js                          | 22 LTS                   | Runtime           | Pinned in `.nvmrc` and container Dockerfile. NanoClaw is built on it. No choice here.                      | HIGH       |
| TypeScript                       | ^5.7                     | Language          | Already in use. ES2022 target, NodeNext modules, strict mode.                                              | HIGH       |
| `@anthropic-ai/claude-agent-sdk` | ^0.2.76 (latest 0.2.104) | Agent execution   | Core of NanoClaw — runs Claude Code inside containers. Pin to `^0.2.76` initially, update when stable.     | HIGH       |
| `@modelcontextprotocol/sdk`      | ^1.12.1 (latest 1.29+)   | MCP tools         | Container-side IPC tools. Version will auto-bump when claude-agent-sdk updates (it's a dependency).        | HIGH       |
| `better-sqlite3`                 | 11.10.0                  | Database          | Messages, sessions, groups, tasks, router state. Embedded, zero-config, fast. No reason to change.         | HIGH       |
| `@onecli-sh/sdk`                 | ^0.2.0 (latest 0.3.1)    | Credential proxy  | Injects API keys into container HTTPS traffic without exposing secrets. Required by NanoClaw architecture. | HIGH       |
| Docker                           | latest stable            | Container runtime | Native on Linux (no VM overhead). Already pre-installed on Hetzner CX32.                                   | HIGH       |

### Channel Layer (To Be Added via Skill)

| Technology | Version | Purpose          | Why                                                                                                                                                                                                                                                                                                                        | Confidence |
| ---------- | ------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `grammy`   | 1.42.0  | Telegram Bot API | NanoClaw's upstream Telegram skill uses grammY. It's the standard modern Telegram bot framework for Node.js/TypeScript — better TS support than `node-telegram-bot-api`, actively maintained, Deno-compatible. The skill at `qwibitai/nanoclaw-telegram` merges in `src/channels/telegram.ts` with grammY as a dependency. | HIGH       |

**How Telegram gets installed:** The `add-telegram` skill merges code from `git remote add telegram https://github.com/qwibitai/nanoclaw-telegram.git` → `git fetch telegram main` → `git merge telegram/main`. This adds `src/channels/telegram.ts`, test file, barrel import, and `grammy` to `package.json`. Channels self-register via `registerChannel()` in the factory pattern — no core code changes needed.

**grammY in NanoClaw context:** NanoClaw uses long polling (not webhooks) because there's no public HTTP server. grammY supports both modes. The `TelegramChannel` class wraps grammY's `Bot` instance and maps Telegram updates to NanoClaw's `onMessage` callback. Chat IDs are prefixed with `tg:` (e.g., `tg:123456789`).

### Channel Formatting (To Be Added via Skill)

| Technology           | Version        | Purpose                    | Why                                                                                                                                                                                                                                                                                                                                        | Confidence |
| -------------------- | -------------- | -------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| `src/text-styles.ts` | N/A (zero-dep) | Markdown → Telegram syntax | The `channel-formatting` skill adds `parseTextStyles(text, channel)` that converts Claude's Markdown output to each platform's native syntax. For Telegram: preserves `[text](url)` links (Telegram Markdown v1 renders them), converts `**bold**` → `*bold*`, `*italic*` → `_italic_`. Code blocks are protected. No external dependency. | HIGH       |

**How formatting gets installed:** `git fetch upstream skill/channel-formatting` → `git merge upstream/skill/channel-formatting`. Modifies `src/router.ts` to call `parseTextStyles` in `formatOutbound`, and `src/index.ts` to pass `channel.name` to the formatter.

### Voice Transcription (Custom — Not Upstream)

| Technology                  | Version | Purpose             | Why                                                                                                                                                                                                       | Confidence |
| --------------------------- | ------- | ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `@elevenlabs/elevenlabs-js` | ^2.42.0 | Speech-to-text      | User preference for ElevenLabs over OpenAI Whisper. Official JS SDK exists. API endpoint: `POST /v1/speech-to-text` with `model_id: "scribe_v2"`. Multipart form upload, returns JSON with `.text` field. | HIGH       |
| **No** `openai` package     | —       | Explicitly not used | The upstream `add-voice-transcription` skill uses OpenAI Whisper API. MeeqClaw replaces this with ElevenLabs. Don't install the `openai` package.                                                         | HIGH       |

**ElevenLabs STT API details (verified from official docs):**

- Endpoint: `POST https://api.elevenlabs.io/v1/speech-to-text`
- Auth: `xi-api-key` header
- Model: `scribe_v2` (current best, replaces scribe_v1)
- Input: `file` (binary, multipart/form-data) — supports all major audio formats including OGG/Opus (Telegram's voice format)
- Output: `{ language_code, text, words[] }` — we only need `.text`
- Optional: `language_code` parameter for known language (can improve accuracy), `tag_audio_events: false` to skip `(laughter)` tags, `no_verbatim: true` to remove filler words
- Max file size: 3GB (more than enough for voice messages)
- Pricing: Credits-based. Free tier: ~10-20 minutes STT/month. Starter ($5/mo): ~30-60 minutes. Personal voice messages are well within free tier.

**Implementation approach:** Write a custom `src/transcription.ts` that:

1. Takes audio buffer from Telegram voice message (OGG/Opus format)
2. Calls ElevenLabs STT via the JS SDK: `client.speechToText.convert({ model_id: 'scribe_v2', file: audioBuffer })`
3. Returns transcript text
4. Telegram channel code checks for voice messages, transcribes, and delivers as `[Voice: <transcript>]`

This follows the same pattern as the upstream Whisper skill but swaps the API call.

### Infrastructure (Linux VPS)

| Technology               | Version   | Purpose                | Why                                                                                                           | Confidence |
| ------------------------ | --------- | ---------------------- | ------------------------------------------------------------------------------------------------------------- | ---------- |
| Ubuntu                   | 24.04 LTS | Host OS                | Already running on Hetzner CX32. LTS = 5 years of updates.                                                    | HIGH       |
| Docker                   | CE latest | Container runtime      | Pre-installed on VPS. Native Linux containers (no VM overhead like Docker Desktop on macOS).                  | HIGH       |
| systemd                  | native    | Process management     | NanoClaw includes macOS launchd plist; Linux equivalent is a systemd user service. Standard for Ubuntu 24.04. | HIGH       |
| Tailscale                | latest    | VPN/networking         | Already connecting VPS, MacBook, and Raspberry Pi. Used for SSH and Obsidian Sync routing.                    | HIGH       |
| Obsidian Sync (headless) | latest    | Vault synchronization  | Already running on VPS for cross-device vault sync (iPhone, MacBook, VPS).                                    | MEDIUM     |
| git                      | system    | Memory version history | For tracking vault/memory file changes. Alongside Obsidian Sync, not replacing it.                            | MEDIUM     |

### Supporting Libraries (Already in NanoClaw)

| Library       | Version | Purpose           | When Used                                            |
| ------------- | ------- | ----------------- | ---------------------------------------------------- |
| `cron-parser` | 5.5.0   | Cron scheduling   | Task scheduler, validated in-container via MCP tools |
| `zod`         | ^4.0.0  | Schema validation | Container-side MCP tool parameter validation         |
| `tsx`         | ^4.19.0 | Dev execution     | `npm run dev` for development, `npm run setup`       |
| `vitest`      | ^4.0.18 | Testing           | `npm test`                                           |
| `prettier`    | ^3.8.1  | Code formatting   | Pre-commit hook via husky                            |
| `eslint`      | ^9.35.0 | Linting           | Code quality checks                                  |
| `husky`       | ^9.1.7  | Git hooks         | Pre-commit formatting                                |

### SSH from Container (Custom Implementation)

| Technology       | Version | Purpose                   | Why                                                                                                                                                                                                        | Confidence |
| ---------------- | ------- | ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `openssh-client` | system  | SSH from container → host | Install in Dockerfile. Mount a dedicated SSH keypair (container-specific, not user's keys) read-only at `/home/node/.ssh/`. Connect via `host.docker.internal` which resolves to the host on Linux Docker. | MEDIUM     |

**Implementation pattern:**

1. Generate a dedicated SSH keypair on the host: `ssh-keygen -t ed25519 -f ~/.config/nanoclaw/container-ssh-key -N ""`
2. Add the public key to host's `~/.ssh/authorized_keys` with restrictions: `command="/usr/bin/env bash",restrict,pty ssh-ed25519 AAAA...`
3. Add `openssh-client` to the container Dockerfile
4. Mount the private key read-only: add to mount-allowlist and additional mounts
5. SSH command in container: `ssh -i /workspace/extra/container-ssh-key -o StrictHostKeyChecking=no node@host.docker.internal`

**Security considerations:** The SSH key should be restricted with `authorized_keys` options (no port forwarding, limited commands if desired). The container already runs as a non-root user (uid 1000, `node`). Docker's `--add-host=host.docker.internal:host-gateway` is automatically added on Linux by NanoClaw's `hostGatewayArgs()`.

### Obsidian Vault + Git Coexistence

| Technology             | Purpose                    | Why                                                                                                                                                                                                         | Confidence |
| ---------------------- | -------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| Obsidian Sync          | Cross-device vault sync    | Already running. Handles iPhone, MacBook, VPS sync. Proprietary but reliable for multi-device.                                                                                                              | HIGH       |
| git (server-side only) | Memory version history     | Run git on the VPS vault directory for change tracking. NOT using git-based sync — just `git add/commit` for audit trail.                                                                                   | MEDIUM     |
| `.gitignore` in vault  | Exclude Obsidian internals | Must ignore `.obsidian/workspace.json`, `.obsidian/workspace-mobile.json`, `.trash/`, `.obsidian/plugins/*/data.json` (plugin state). Keep `.obsidian/app.json` and `.obsidian/appearance.json` optionally. | MEDIUM     |

**Coexistence strategy:** Obsidian Sync handles real-time multi-device synchronization. Git provides version history on the VPS only (no push/pull, no multi-device git). This avoids the major conflict pattern where git merge conflicts and Obsidian Sync conflicts interact unpredictably.

**Bind mount for agent access:** Mount `~/vaults/meeq-vault/` (or a memory subdirectory within it) into containers via the mount-allowlist. The agent reads/writes memory files. Obsidian Sync propagates changes to other devices. A cron job or inotify-based watcher runs `git add -A && git commit -m "auto: $(date)"` periodically for version history.

## Alternatives Considered

| Category              | Recommended          | Alternative                              | Why Not                                                                                                                                                                                       |
| --------------------- | -------------------- | ---------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Telegram library      | grammY 1.42          | `node-telegram-bot-api`                  | grammY has better TypeScript support, active development, and is what NanoClaw's upstream skill uses. `node-telegram-bot-api` is older, less typed, larger API surface.                       |
| Telegram library      | grammY 1.42          | `telegraf`                               | Telegraf is also good but grammY is NanoClaw's chosen library. Switching would mean maintaining a custom Telegram skill instead of using upstream's.                                          |
| Voice transcription   | ElevenLabs Scribe v2 | OpenAI Whisper API                       | User preference. ElevenLabs Scribe v2 has comparable quality. Upstream skill uses Whisper — we fork the pattern, not the dependency.                                                          |
| Voice transcription   | ElevenLabs cloud API | Local whisper.cpp                        | Local requires significant CPU/RAM. Hetzner CX32 has 4 vCPU / 8GB RAM — could work but adds complexity. Cloud API is simpler for v1.                                                          |
| Process manager       | systemd user service | PM2                                      | systemd is native to Ubuntu, no extra dependency. PM2 adds a Node.js daemon. NanoClaw already has launchd support — systemd is the direct Linux equivalent.                                   |
| Vault sync            | Obsidian Sync        | Syncthing                                | Obsidian Sync is already running and handles conflict resolution for Obsidian-specific files. Syncthing would duplicate effort.                                                               |
| Vault versioning      | git (local only)     | git (push to remote)                     | Multi-device git + Obsidian Sync = guaranteed conflicts. Keep git local to VPS for audit trail only.                                                                                          |
| Container runtime     | Docker CE            | Podman                                   | Docker is pre-installed, NanoClaw is tested with it. Podman is rootless-first which is nice but adds migration complexity for zero benefit in this context.                                   |
| Credential management | OneCLI gateway       | .env injection / native credential proxy | OneCLI is NanoClaw's default. The native credential proxy skill exists as fallback but OneCLI provides better security (rate limits, per-agent policies). Use OneCLI unless it causes issues. |

## What NOT to Use

| Technology                               | Why Not                                                                                                                         |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| `openai` npm package                     | Don't install. We use ElevenLabs for STT, not Whisper. The upstream voice-transcription skill depends on it — we write our own. |
| Apple Container                          | macOS only. VPS runs Linux with native Docker.                                                                                  |
| PM2, forever, nodemon                    | systemd handles process management natively on Ubuntu.                                                                          |
| WhatsApp channel                         | Not needed. Telegram is the sole channel.                                                                                       |
| Nginx/reverse proxy                      | No incoming HTTP traffic needed. Telegram uses long polling (outbound only). OneCLI gateway is localhost-only.                  |
| Embedding databases (pgvector, Pinecone) | Out of scope for v1. File-based memory with grep is sufficient for personal use.                                                |
| Redis/Valkey                             | No need. SQLite handles all persistence. Single-process architecture.                                                           |

## Installation

### On VPS (Production)

```bash
# System dependencies (Ubuntu 24.04)
sudo apt update && sudo apt install -y git curl

# Node.js 22 via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
nvm install 22
nvm use 22

# Clone fork
git clone git@github.com:LLMinem/nanoclaw.git ~/nanoclaw
cd ~/nanoclaw
git checkout meeq-claw

# Install host dependencies
npm install

# Install Telegram channel (via skill merge)
git remote add telegram https://github.com/qwibitai/nanoclaw-telegram.git
git fetch telegram main
git merge telegram/main

# Install channel formatting (via skill merge)
git fetch upstream skill/channel-formatting
git merge upstream/skill/channel-formatting

# Rebuild after merges
npm install
npm run build

# Build container image
./container/build.sh

# ElevenLabs SDK (added to package.json manually — no upstream skill for this)
npm install @elevenlabs/elevenlabs-js
```

### Container Dockerfile Additions

```dockerfile
# Add to container/Dockerfile for SSH support
RUN apt-get update && apt-get install -y openssh-client && rm -rf /var/lib/apt/lists/*
```

### systemd User Service

```ini
# ~/.config/systemd/user/nanoclaw.service
[Unit]
Description=NanoClaw AI Assistant
After=network.target docker.service

[Service]
Type=simple
WorkingDirectory=/home/meeq/nanoclaw
ExecStart=/home/meeq/.nvm/versions/node/v22.*/bin/node dist/index.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production

[Install]
WantedBy=default.target
```

```bash
# Enable lingering (keeps user services running after logout)
sudo loginctl enable-linger meeq

# Enable and start
systemctl --user daemon-reload
systemctl --user enable nanoclaw
systemctl --user start nanoclaw

# Check status
systemctl --user status nanoclaw
journalctl --user -u nanoclaw -f
```

## Version Verification

| Package                          | Claimed   | Verified Source                                       | Date       |
| -------------------------------- | --------- | ----------------------------------------------------- | ---------- |
| `grammy`                         | 1.42.0    | npm registry `/grammy/latest`                         | 2026-04-12 |
| `@elevenlabs/elevenlabs-js`      | 2.42.0    | npm registry `/@elevenlabs/elevenlabs-js/latest`      | 2026-04-12 |
| `@anthropic-ai/claude-agent-sdk` | 0.2.104   | npm registry `/@anthropic-ai/claude-agent-sdk/latest` | 2026-04-12 |
| `@onecli-sh/sdk`                 | 0.3.1     | npm registry `/@onecli-sh/sdk/latest`                 | 2026-04-12 |
| ElevenLabs STT model             | scribe_v2 | Official API docs `/v1/speech-to-text`                | 2026-04-12 |

## Sources

- NanoClaw codebase: `package.json`, `src/container-runner.ts`, `src/container-runtime.ts`, `src/mount-security.ts` — HIGH confidence (direct code inspection)
- NanoClaw skill files: `.claude/skills/add-telegram/SKILL.md`, `.claude/skills/add-voice-transcription/SKILL.md`, `.claude/skills/channel-formatting/SKILL.md` — HIGH confidence (direct file inspection)
- grammY documentation: https://grammy.dev/guide/getting-started — HIGH confidence (official docs)
- grammY npm registry: verified v1.42.0 latest — HIGH confidence
- ElevenLabs STT API: https://elevenlabs.io/docs/api-reference/speech-to-text/convert — HIGH confidence (official API reference with OpenAPI spec)
- ElevenLabs JS SDK: npm registry verified v2.42.0 latest — HIGH confidence
- ElevenLabs pricing: https://elevenlabs.io/pricing — HIGH confidence (official pricing page)
- Claude Agent SDK: npm registry verified v0.2.104 latest — HIGH confidence
- OneCLI SDK: npm registry verified v0.3.1 latest — HIGH confidence
- NanoClaw Telegram repo: https://github.com/qwibitai/nanoclaw-telegram — HIGH confidence (upstream source)
- Docker host gateway: NanoClaw source `src/container-runtime.ts` line 16 — HIGH confidence (code)

---

_Stack research: 2026-04-12_

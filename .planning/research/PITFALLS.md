# Pitfalls Research

**Domain:** NanoClaw fork customization — personal AI assistant deployed on VPS with Docker
**Researched:** 2026-04-12
**Confidence:** HIGH (derived from codebase analysis, skill source code, and CONCERNS.md)

## Critical Pitfalls

### Pitfall 1: Skill Branch Merge Creates Silent Channel Breakage

**What goes wrong:**
Skill branches (e.g., `add-telegram`) modify `src/channels/index.ts` by appending an `import './telegram.js'` line. The barrel file is structurally fragile — it's just comments with side-effect imports (currently 12 lines, all commented out). When merging multiple skill branches, or when upstream updates touch the same barrel file, git may auto-merge cleanly but produce a file with duplicate, missing, or misordered imports. The channel silently disappears at runtime with no error — NanoClaw just doesn't respond on that channel.

**Why it happens:**
Git's 3-way merge considers nearby context lines. The barrel file has minimal context (one-line comments as section markers), making git merge heuristics unreliable. Each skill branch was created from a base where all lines are comments, so two skills adding imports to the same region can conflict or, worse, auto-merge incorrectly.

**How to avoid:**

1. After **every** skill branch merge, manually verify `src/channels/index.ts` contains exactly the expected uncommented import lines. This takes 5 seconds.
2. After merging, always run `npm run build` — a missing channel module file will cause a build error, but a missing import line won't.
3. Send a test message on each configured channel after any merge operation.

**Warning signs:**

- Bot stops responding on one channel after a merge, but other channels work fine.
- `src/channels/index.ts` shows only comment lines and no active imports after a merge.
- `git diff` after merge shows changes to `src/channels/index.ts` that you didn't expect.

**Phase to address:**
Phase 1 (Initial Setup) — verify barrel file state immediately after Telegram skill merge. Create a post-merge checklist.

---

### Pitfall 2: OneCLI Gateway Unreachable on Headless VPS = Total Agent Failure

**What goes wrong:**
OneCLI runs as a local HTTP gateway on port 10254. If the gateway process dies, crashes, or fails to start after reboot, every container launch proceeds but with `onecliApplied === false` (see `container-runner.ts:242-249`). The container starts, but the agent has **no Anthropic credentials** — it silently fails to call the API. The host logs a warning (`OneCLI gateway not reachable — container will have no credentials`) but the bot appears to "accept" the message and then never responds.

**Why it happens:**
OneCLI is a separate process from NanoClaw. On a headless VPS, there's no GUI to notice it died. The NanoClaw systemd unit doesn't declare a dependency on OneCLI. The warning is logged but there's no health-check escalation, no retry, and no notification to the user. `onecli start` may need to be run manually after reboot if OneCLI doesn't have its own systemd unit.

**How to avoid:**

1. Create a separate systemd unit for OneCLI gateway with `Restart=always`.
2. Add `Requires=onecli.service` and `After=onecli.service` to the NanoClaw systemd unit so it doesn't start without OneCLI.
3. Verify OneCLI health as part of NanoClaw startup: `curl -sf http://127.0.0.1:10254/health`.
4. Alternative: skip OneCLI entirely and use the `use-native-credential-proxy` skill, which reads from `.env` directly — simpler for single-user VPS deployments with one credential set.

**Warning signs:**

- Bot accepts messages but never responds (container starts, fails internally).
- `logs/nanoclaw.log` contains repeated `OneCLI gateway not reachable` warnings.
- `curl http://127.0.0.1:10254/health` fails from the VPS.

**Phase to address:**
Phase 1 (Initial Setup) — decide between OneCLI vs. native credential proxy before deployment. If OneCLI, create its systemd unit in the same phase.

---

### Pitfall 3: Docker Group Membership Not Active in Systemd Session

**What goes wrong:**
On Ubuntu, adding the user to the `docker` group (`usermod -aG docker $USER`) doesn't take effect in systemd user sessions until the user fully logs out and back in. The NanoClaw setup script detects this (`checkDockerGroupStale()` in `setup/service.ts:259`) but if you ignore the warning and start the service, every container spawn fails with a Docker socket permission error.

**Why it happens:**
Linux group membership is read at login time. `systemctl --user` services inherit the group list from the user's login session. SSH reconnection may or may not refresh groups depending on PAM config. This is a classic Linux gotcha that catches every first-time Docker-on-VPS user.

**How to avoid:**

1. After adding user to docker group: `sudo setfacl -m u:$(whoami):rw /var/run/docker.sock` for immediate fix.
2. For persistence across Docker restarts, create `/etc/systemd/system/docker.service.d/socket-acl.conf` as documented in the setup skill.
3. Alternatively, run NanoClaw as root (not recommended but eliminates the group issue).

**Warning signs:**

- NanoClaw starts but every message triggers `container exited with code 1`.
- Docker socket permission denied errors in container logs.
- `groups` command in current shell shows `docker` but `systemctl --user show-environment` doesn't reflect it.

**Phase to address:**
Phase 1 (Initial Setup) — the setup skill handles this, but verify it worked before moving on.

---

### Pitfall 4: Obsidian Sync and Git Fighting Over the Same Files

**What goes wrong:**
Obsidian Sync and git both track file changes. When the agent writes a memory file via bind mount, both systems detect the change. Obsidian Sync propagates it to other devices. If the same file is edited on another device (iPhone, MacBook) via Obsidian before the VPS syncs, you get divergent versions. Obsidian Sync handles conflicts by creating duplicate files (`MEMORY 1.md`), while git shows merge conflicts. Neither system knows about the other's conflict resolution.

The worse scenario: git operations (`git add`, `git commit`) create `.git/` directory changes that Obsidian Sync tries to propagate. Obsidian Sync was never designed to sync git metadata — it can corrupt the `.git/` directory, causing repository corruption on other devices.

**Why it happens:**
Obsidian Sync and git are both file-level sync systems with independent conflict detection. They have no awareness of each other. Using both on the same directory tree is fundamentally a dual-master replication problem with no coordinator.

**How to avoid:**

1. **Never put a `.git` repo inside an Obsidian-synced vault.** This is the cardinal rule.
2. Architecture: Obsidian vault at `~/vaults/meeq-vault/` (synced by Obsidian). Git repo for version history stays **outside** the vault in a separate directory.
3. If you want git history for memory files: use a one-way copy/sync script that periodically copies from the vault to a git-tracked directory (cron job, `rsync --delete`). The git-tracked copy is the archive; the vault copy is the live version.
4. Alternatively, use Obsidian's built-in version history (which Obsidian Sync provides) and skip git for memory files entirely.
5. `.gitignore` in the vault should exclude `.obsidian/` if you ever accidentally init a repo there.

**Warning signs:**

- Duplicate files appearing in Obsidian (e.g., `MEMORY 1.md`, `USER 1.md`).
- Git showing modified files you didn't touch.
- `.git/` directory appearing in Obsidian Sync's sync list.
- Obsidian showing "sync conflict" notifications.

**Phase to address:**
Phase 2 (Workspace Architecture) — this must be decided before any memory files are placed. The vault/git boundary is an architectural decision, not something to fix later.

---

### Pitfall 5: Bind Mount Path Mismatches Between Dev Mac and Production VPS

**What goes wrong:**
Mount paths configured on macOS during development (`/Users/meeq/...`) don't exist on the Ubuntu VPS (`/home/meeq/...`). The mount allowlist at `~/.config/nanoclaw/mount-allowlist.json` contains absolute paths. If you develop the mount config on Mac and deploy to VPS, every additional mount fails validation silently (the `mount-security.ts` module logs a warning and blocks the mount, but the container starts without the expected directory).

The container runs, the agent can't find memory files, and responds with "I don't have access to your memory" — which looks like a bug, not a config error.

**Why it happens:**
macOS uses `/Users/`, Linux uses `/home/`. The mount allowlist is deliberately stored outside the project root (`~/.config/nanoclaw/`) for security, meaning it's not in git and not automatically synced between machines. This is security-by-design but creates a deployment gap.

**How to avoid:**

1. Create the mount allowlist directly on the VPS, not by copying from Mac.
2. Use `~` expansion or environment variables in documentation, but know that the actual JSON file needs absolute paths.
3. After deployment, verify mounts are working: send a test message that asks the agent to list files in the mounted directory.
4. Keep a `config-examples/mount-allowlist.example.json` in git with VPS paths, separate from the Mac version.

**Warning signs:**

- Agent says it can't access files that should be mounted.
- `logs/nanoclaw.log` shows `Mount validation failed` or `not under any allowed root` warnings.
- Container starts successfully but workspace directories are empty.

**Phase to address:**
Phase 1 (Initial Setup) — configure mount allowlist on VPS immediately after deployment. Phase 2 (Workspace Architecture) — finalize which paths need mounting.

---

### Pitfall 6: Memory Migration Assumes Same File Structure

**What goes wrong:**
Migrating memory files from OpenClaw (`~/clawd/memory/`, `~/clawd/MEMORY.md`, etc.) to NanoClaw means placing them where the NanoClaw container expects them. But the two systems have different workspace layouts:

- OpenClaw: flat `~/clawd/` directory with everything at root level
- NanoClaw: `groups/<folder>/` per chat, with `groups/global/` for shared files, plus bind-mounted external directories

If you just copy files into the wrong location, the agent can't find them. Worse, if you copy `SOUL.md` and `USER.md` with OpenClaw-specific references (`/clawd/`, OpenClaw commands, `ocx` CLI references), the NanoClaw agent will follow stale instructions and try to use tools/paths that don't exist.

**Why it happens:**
The migration is a content adaptation task, not just a file copy. OpenClaw's memory files contain references to OpenClaw's architecture, commands, and file paths. These references become dangling pointers in NanoClaw.

**How to avoid:**

1. **Audit each file** being migrated for OpenClaw-specific references before copying. Search for: `clawd`, `openclaw`, `ocx`, `/Users/meeq/clawd`, any OpenClaw-specific command names.
2. Place shared files (SOUL.md, USER.md, MEMORY.md) in the main group directory (`groups/main/`) or in `groups/global/` for cross-group access.
3. Place daily memory files in a bind-mounted directory (Obsidian vault) so they're accessible from multiple contexts.
4. Update MEMORY.md index to use NanoClaw-relative paths, not absolute OpenClaw paths.

**Warning signs:**

- Agent mentions "clawd" or OpenClaw concepts in responses.
- Agent tries to run commands that don't exist in NanoClaw.
- Agent says "I can't find my memory files" despite them being on disk.

**Phase to address:**
Phase 2 (Memory Migration) — do a search-and-replace pass on every migrated file. This is a content task, not a deployment task.

---

### Pitfall 7: Upstream Sync Destroys Customizations Via Rebase

**What goes wrong:**
Using `git rebase upstream/main` instead of `git merge upstream/main` rewrites the commit history of the `main` branch. If the branch has already been pushed to `origin`, rebasing requires a force push. Force pushing while another session (or Claude Code itself) has the old history checked out causes divergent states. More practically: rebase resolves conflicts per-commit (potentially dozens of times for a large upstream update) while merge resolves them once.

The `update-nanoclaw` skill offers rebase as an option but warns about it. A beginner developer might choose rebase thinking "linear history is better" without understanding the implications.

**Why it happens:**
Rebase is seductive — cleaner history, no merge commits. But for a fork that periodically syncs with upstream, merge is the correct strategy. Merge preserves the point-in-time divergence, makes rollback trivial (revert the merge commit), and resolves conflicts in a single pass.

**How to avoid:**

1. **Always use merge** for upstream syncs. The `update-nanoclaw` skill defaults to merge — don't change this.
2. Never rebase a branch that has been pushed to a remote.
3. The `/update-nanoclaw` skill creates a backup tag before every sync — always note this tag name.
4. If a merge goes badly: `git reset --hard <backup-tag>` to restore to pre-merge state.

**Warning signs:**

- `git status` shows "Your branch and 'origin/main' have diverged" after an upstream sync.
- `git log` shows your customization commits interleaved with upstream commits (sign of rebase).
- Force push prompts when trying to push.

**Phase to address:**
Phase 1 (Initial Setup) — establish the merge-only policy. Document the upstream sync workflow in the project's own CLAUDE.md.

---

## Technical Debt Patterns

Shortcuts that seem reasonable but create long-term problems.

| Shortcut                                                    | Immediate Benefit                        | Long-term Cost                                                            | When Acceptable                                                             |
| ----------------------------------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| Skip mount allowlist setup, give agent access to everything | Faster setup, agent can "see" all files  | Agent can modify/delete any host file, security boundary meaningless      | Never — the allowlist exists for a reason                                   |
| Put API keys in `.env` instead of OneCLI                    | Simpler, one less service to manage      | Keys visible in plaintext on disk, must manually rotate, no rate limiting | Acceptable if using `use-native-credential-proxy` skill (designed for this) |
| Skip `loginctl enable-linger` on VPS                        | None, just laziness                      | NanoClaw dies when SSH session closes, bot goes offline                   | Never on a VPS deployment                                                   |
| Copy OpenClaw memory files without reviewing                | Faster migration, everything "preserved" | Stale references confuse the agent, wrong paths, wrong commands           | Never — content review is mandatory                                         |
| Skip `npm run build` after skill merges                     | Saves 30 seconds                         | Typescript errors surface at runtime as cryptic failures                  | Never — the build is your safety net                                        |

## Integration Gotchas

Common mistakes when connecting to external services.

| Integration                 | Common Mistake                                            | Correct Approach                                                                                                                                                               |
| --------------------------- | --------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| OneCLI on VPS               | Installing gateway but not creating a systemd unit for it | Create `onecli.service` with `Restart=always`, make NanoClaw depend on it                                                                                                      |
| Telegram Bot                | Not disabling Group Privacy in BotFather                  | Send `/mybots` > select bot > Bot Settings > Group Privacy > Turn off. Then **remove and re-add** bot to group                                                                 |
| Obsidian Headless           | Running Obsidian as root or with wrong user               | Run as same user as NanoClaw, or files will have wrong ownership for bind mounts                                                                                               |
| ElevenLabs in container     | Passing API key directly to container environment         | Use OneCLI or credential proxy to inject the key. If using OneCLI, register with `--host-pattern api.elevenlabs.io`                                                            |
| Docker container networking | Assuming `localhost` in container reaches host            | On Linux, use `host.docker.internal` (NanoClaw adds `--add-host` flag automatically). Verify with `curl http://host.docker.internal:10254/health` from inside a test container |
| systemd user service        | Not setting `Environment=PATH=...` in unit file           | The setup script includes `/usr/local/bin` and `~/.local/bin` in PATH. If you add other tools (e.g., `onecli`), ensure they're on the service's PATH                           |

## Performance Traps

Patterns that work at small scale but fail as usage grows.

| Trap                                    | Symptoms                                                   | Prevention                                                                                                           | When It Breaks                                  |
| --------------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| Unbounded SQLite message table          | Disk fills up, queries slow down                           | Not a v1 concern — implement retention later. Monitor disk with `du -sh store/`                                      | Months of heavy use (thousands of messages/day) |
| Container image not cached locally      | Every container start pulls layers, 30+ second startup     | Run `docker build` once during setup. The image is cached locally. Rebuild only when changing container dependencies | Only if `docker image prune` is run carelessly  |
| Chromium in container image when unused | 500MB+ image size, slower pulls, more memory per container | Not blocking for v1, but if container startup is slow, consider a slim image without Chromium                        | If running on a resource-constrained VPS        |

## Security Mistakes

Domain-specific security issues beyond general web security.

| Mistake                                                  | Risk                                                                  | Prevention                                                                                                               |
| -------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Mounting `.ssh` directory into container                 | Agent has full SSH access to all machines your keys unlock            | `.ssh` is in `DEFAULT_BLOCKED_PATTERNS` in `mount-security.ts` — don't override this                                     |
| SSH from container via mounted keys                      | Container compromise = all SSH targets compromised                    | Use a dedicated SSH key pair for container-to-host only, with `ForceCommand` restriction on the host's `authorized_keys` |
| Mounting Obsidian vault read-write for non-main groups   | Any group's agent can modify your personal notes                      | Use `nonMainReadOnly: true` in mount allowlist (the default)                                                             |
| Leaving `.env` with real API keys after OneCLI migration | Keys are duplicated in plaintext and in vault                         | After `init-onecli`, verify `.env` no longer contains `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN`                   |
| Running NanoClaw as root                                 | Container escape = full host root access                              | Always run as a regular user. The setup script handles user-level systemd                                                |
| `bypassPermissions` in agent-runner                      | Agent can modify any file in mounted directories without confirmation | This is by-design for containerized execution, but reinforces why mount boundaries matter                                |

## UX Pitfalls

Common user experience mistakes in this domain.

| Pitfall                                                 | User Impact                                                | Better Approach                                                                                    |
| ------------------------------------------------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| Not setting `ASSISTANT_NAME` in `.env`                  | Bot responds as "Andy" instead of "Fritz"                  | Set `ASSISTANT_NAME=Fritz` in `.env` before first run                                              |
| Not configuring sender allowlist                        | Anyone who knows your Telegram bot username can talk to it | Create `~/.config/nanoclaw/sender-allowlist.json` with your Telegram user ID                       |
| Agent responds with markdown that Telegram can't render | Raw `**bold**` and `[links](url)` in messages              | Install the `channel-formatting` skill to convert markdown to Telegram-native formatting           |
| Container timeout too short for complex tasks           | Agent gets killed mid-response on long tasks               | Default is 30 minutes (`CONTAINER_TIMEOUT=1800000`). Increase if needed, but don't set to infinity |

## "Looks Done But Isn't" Checklist

Things that appear complete but are missing critical pieces.

- [ ] **Telegram setup:** Bot responds to direct messages but not in groups — verify Group Privacy is OFF in BotFather and bot was re-added to group after changing the setting
- [ ] **OneCLI setup:** `onecli secrets list` shows a secret but containers still fail — verify `ONECLI_URL` is set in `.env` AND the gateway is reachable from inside a container (`docker run --rm --add-host=host.docker.internal:host-gateway curlimages/curl http://host.docker.internal:10254/health`)
- [ ] **Memory files mounted:** Files exist on host but agent can't see them — verify the mount allowlist JSON has correct absolute paths for VPS, not Mac paths
- [ ] **Service running:** `systemctl --user status nanoclaw` shows "active" but bot doesn't respond — check `logs/nanoclaw.log` for channel connection errors, not just service status
- [ ] **Container builds:** `docker build` succeeds but agent fails at runtime — verify the agent-runner `npm install` inside the container completed (check for `node_modules` in build output)
- [ ] **Upstream sync done:** `git merge upstream/main` completed without conflicts but bot behavior changed — check `CHANGELOG.md` for `[BREAKING]` entries that require running migration skills
- [ ] **Fork pushed:** Local changes committed but not pushed to `origin/main` — the VPS deployment pulls from origin, not your local machine
- [ ] **`.env` synced to container env:** Changes to `.env` don't take effect until `mkdir -p data/env && cp .env data/env/env` AND service restart

## Recovery Strategies

When pitfalls occur despite prevention, how to recover.

| Pitfall                                     | Recovery Cost | Recovery Steps                                                                                                                         |
| ------------------------------------------- | ------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| Channel barrel file corrupted               | LOW           | Open `src/channels/index.ts`, add the correct import line, `npm run build`, restart service                                            |
| OneCLI gateway dead                         | LOW           | `onecli start` or restart its systemd unit. If secrets lost: `onecli secrets create` again                                             |
| Docker group not active                     | LOW           | `sudo setfacl -m u:$(whoami):rw /var/run/docker.sock`, then restart NanoClaw service                                                   |
| Obsidian Sync conflict files                | MEDIUM        | Manually compare duplicate files, keep correct version, delete `*1.md` copies. Prevent by separating git and sync                      |
| Mount paths wrong                           | LOW           | Edit `~/.config/nanoclaw/mount-allowlist.json` with correct VPS paths, restart service                                                 |
| Memory files have stale OpenClaw references | MEDIUM        | Grep all memory files for `clawd`, `openclaw`, `ocx`. Replace references. Test by asking agent about itself                            |
| Upstream merge went wrong                   | LOW           | `git reset --hard <backup-tag>` (tag created by `/update-nanoclaw` before every merge)                                                 |
| Rebase corrupted branch history             | HIGH          | If pushed: `git reflog` to find pre-rebase commit, `git reset --hard <ref>`, `git push --force`. If not pushed: `git reflog` and reset |

## Pitfall-to-Phase Mapping

How roadmap phases should address these pitfalls.

| Pitfall                         | Prevention Phase                      | Verification                                                                     |
| ------------------------------- | ------------------------------------- | -------------------------------------------------------------------------------- |
| Channel barrel file breakage    | Phase 1 (Setup)                       | After Telegram merge, `cat src/channels/index.ts` shows active import            |
| OneCLI unreachable              | Phase 1 (Setup)                       | `curl http://127.0.0.1:10254/health` succeeds; systemd unit exists for OneCLI    |
| Docker group membership         | Phase 1 (Setup)                       | `docker info` succeeds from NanoClaw service context                             |
| Obsidian Sync + git conflict    | Phase 2 (Workspace Architecture)      | `.git/` directory is NOT inside `~/vaults/meeq-vault/`                           |
| Mount path mismatches           | Phase 1 (Setup) + Phase 2 (Workspace) | Agent can list files in mounted memory directory                                 |
| Memory migration stale refs     | Phase 2 (Memory Migration)            | `grep -r "clawd\|openclaw\|ocx" groups/` returns zero results                    |
| Upstream sync via rebase        | Phase 1 (Setup)                       | `CLAUDE.md` documents "merge only" policy for upstream syncs                     |
| Sender allowlist missing        | Phase 1 (Setup)                       | `~/.config/nanoclaw/sender-allowlist.json` exists with Telegram user ID          |
| Container timeout too short     | Phase 2 (Configuration)               | `CONTAINER_TIMEOUT` is set appropriately in `.env`                               |
| ElevenLabs credential injection | Phase 3 (Voice Transcription)         | Agent can transcribe a voice message sent via Telegram                           |
| SSH key exposure via mount      | Phase 3 (SSH Setup)                   | Dedicated SSH key pair exists; `authorized_keys` uses `ForceCommand` restriction |

## Sources

- NanoClaw codebase analysis: `src/container-runner.ts`, `src/mount-security.ts`, `src/container-runtime.ts`, `src/channels/index.ts`
- `.planning/codebase/CONCERNS.md` — tech debt, security considerations, fragile areas documented by codebase audit
- Skill source code: `add-telegram/SKILL.md`, `init-onecli/SKILL.md`, `update-nanoclaw/SKILL.md`, `update-skills/SKILL.md`, `setup/SKILL.md`, `use-native-credential-proxy/SKILL.md`
- `setup/service.ts` — systemd unit generation, docker group stale detection, linger enable
- `.planning/PROJECT.md` — project requirements, infrastructure, constraints

---

_Pitfalls research for: MeeqClaw (NanoClaw fork customization)_
_Researched: 2026-04-12_

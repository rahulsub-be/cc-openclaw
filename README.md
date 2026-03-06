# cc-openclaw

Claude Code skills for managing your [OpenClaw](https://openclaw.ai) deployment. These slash commands let you scaffold agents, configure channels, set up memory systems, manage secrets, and operate your gateway — all from within Claude Code.

## What Are Claude Code Skills?

[Claude Code skills](https://docs.anthropic.com/en/docs/claude-code/skills) are reusable playbooks that Claude Code auto-discovers from `.claude/skills/` in your project. Each skill becomes a `/slash-command` you can invoke during a session. They encode best practices, configuration patterns, and operational procedures so you don't have to remember them.

## Quick Start

### Option 1: Clone into your openclaw-home repo

```bash
cd ~/your-openclaw-home-repo
git clone https://github.com/rahulsub-be/cc-openclaw.git .cc-openclaw-tmp
cp -r .cc-openclaw-tmp/.claude/skills/openclaw-* .claude/skills/
rm -rf .cc-openclaw-tmp
```

### Option 2: Use as a standalone project

```bash
git clone https://github.com/rahulsub-be/cc-openclaw.git
cd cc-openclaw
claude  # Skills are auto-discovered when you open Claude Code here
```

### Option 3: Cherry-pick individual skills

Copy just the skills you need into any project's `.claude/skills/` directory:

```bash
cp -r cc-openclaw/.claude/skills/openclaw-status .claude/skills/
```

## Available Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **New Agent** | `/openclaw-new-agent <id> <name>` | Scaffold a complete agent with directory structure, 6 core .md files, and openclaw.json registration |
| **Add Channel** | `/openclaw-add-channel <agent-id> <type>` | Connect Telegram, Slack, WhatsApp, or GChat to an agent with keychain secrets |
| **Add Cron Job** | `/openclaw-add-cron <agent-id>` | Create scheduled tasks — recurring (cron), interval (every), or one-shot (at) |
| **Status Check** | `/openclaw-status` | Health dashboard: gateway, channels, agents, cron jobs, and recent errors |
| **Restart** | `/openclaw-restart` | Restart the gateway with automatic stow conflict resolution |
| **Stow Deploy** | `/openclaw-stow` | Re-deploy config via GNU Stow, resolving the known jobs.json conflict |
| **Add Secret** | `/openclaw-add-secret <service> <ENV_VAR>` | Store a secret in macOS Keychain and wire it into the secrets pipeline |
| **Add Script** | `/openclaw-add-script <agent-id> <name>` | Scaffold a deterministic shell script with JSON output and error handling |
| **Dream Setup** | `/openclaw-dream-setup <agent-id>` | Configure nightly memory distillation with cron job and archive pipeline |

## Skills in Detail

### `/openclaw-new-agent`

Creates a fully-configured agent from scratch:
- Directory structure (`memory/`, `scripts/`, `sessions/`, etc.)
- Six core directive files: `SOUL.md`, `IDENTITY.md`, `USER.md`, `AGENTS.md`, `TOOLS.md`, `SECURITY.md`
- Registration in `openclaw.json`
- Sub-agent wiring (if applicable)
- Stow deployment and gateway restart

### `/openclaw-add-channel`

Handles the full channel setup pipeline per platform:
- **Telegram**: Bot token via BotFather, keychain storage, account + binding config
- **Slack**: Bot token + app token, socket mode setup
- **WhatsApp**: Phone number allowlist
- **GChat**: Binding configuration

All secrets go to macOS Keychain — never written to files.

### `/openclaw-add-cron`

Interactive cron job builder supporting:
- **cron**: Standard cron expressions with timezone (`0 9 * * *`)
- **every**: Fixed intervals (`30m`, `1h`)
- **at**: One-shot execution at a specific time
- Agent turn payloads with configurable models and timeouts
- Isolated or main session targeting

### `/openclaw-status`

One-command health check covering:
- Gateway process (health endpoint, launchd status)
- Channel connectivity (Telegram bots, Slack socket, WhatsApp)
- Agent count and bindings
- Cron job results (last run, errors, next scheduled)
- Recent error log entries

### `/openclaw-restart`

Handles the restart correctly by:
1. Resolving the known `jobs.json` stow conflict (gateway overwrites the symlink)
2. Re-stowing config files
3. Restarting via `launchctl kickstart`
4. Verifying all channels reconnect

### `/openclaw-stow`

Deploys config from your openclaw-home repo to `$HOME` via GNU Stow:
- Detects and resolves known conflicts (jobs.json)
- Verifies key symlinks after deployment
- Reports any unresolved conflicts

### `/openclaw-add-secret`

Full secrets pipeline:
1. Stores value in macOS Keychain (`openclaw.keychain-db`)
2. Updates `openclaw-secrets.sh` (for launchd/gateway)
3. Updates `openclaw-env.sh` (for shell sessions)
4. Updates `secrets.sh` (for provisioning on new machines)

### `/openclaw-add-script`

Scaffolds deterministic shell scripts following production patterns:
- `set -euo pipefail` with structured error handling
- `json-response.sh` shared library (log, json_success, json_error)
- stdout is sacred: only JSON output
- stderr for logging via `log()` function
- Automatic TOOLS.md documentation

### `/openclaw-dream-setup`

Configures nightly memory distillation:
- `DREAM-ROUTINE.md` with distillation spec and token budgets
- Cron job for nightly execution
- QMD memory index paths in openclaw.json
- Session startup sequence in AGENTS.md
- Archive pipeline for old distillations

Token budgets: 2,500/day per distillation, 7,500 rolling 3-day digest.

## How It Works

All skills use automatic repo detection — no hardcoded paths:

```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
```

This resolves your openclaw-home repo location by following the `openclaw.json` symlink that GNU Stow creates. Skills work regardless of where you've cloned your repo.

## Prerequisites

- [OpenClaw](https://openclaw.ai) installed and configured
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [GNU Stow](https://www.gnu.org/software/stow/) (`brew install stow`)
- macOS (skills use `security` CLI for Keychain, `launchctl` for service management)

## Companion Playbooks

These skills are designed to work alongside the [OpenClaw documentation playbooks](https://openclaw.ai/docs) covering:

1. First Agent Setup
2. Memory System (QMD + Dream Routines)
3. Deterministic Scripts
4. Sub-Agent Architecture
5. Cron System
6. Security Model
7. Channel Configuration
8. Stow-Based Deployment
9. Monitor-DevBot Pattern

Each skill references its corresponding playbook for deeper context.

## Contributing

PRs welcome. When adding a new skill:

1. Create `.claude/skills/<skill-name>/SKILL.md`
2. Use YAML frontmatter: `name`, `description`, `argument-hint`, `disable-model-invocation: true`
3. Use `$OPENCLAW_REPO` detection — no hardcoded paths
4. Follow the step-by-step format with bash code blocks
5. Include verification steps
6. Update this README

## License

MIT

# When Microsoft Told Us Our Agents Were Sharing Secrets

## A follow-up to "Managing OpenClaw with Claude Code" — and why we split one gateway into five

---

A few weeks ago, I published [Managing OpenClaw with Claude Code](https://trilogyai.substack.com/p/managing-openclaw-with-claude-code), where I walked through nine Claude Code skills that standardize how you operate an OpenClaw deployment. The skills solved a real problem: configuration drift caused by agents (and humans) making ad-hoc changes to a complex system.

Then a colleague sent me a link that changed how I think about the whole setup.

Microsoft's Defender Security Research Team published [Running OpenClaw Safely: Identity, Isolation & Runtime Risk](https://www.microsoft.com/en-us/security/blog/2026/02/19/running-openclaw-safely-identity-isolation-runtime-risk/), and the core finding hit close to home:

> *"Self-hosted agents execute code with durable credentials and process untrusted input."*

The article flags what they call **credential bleed** — the risk that arises when multiple agents share a runtime environment and all have access to the same pool of secrets. In a typical OpenClaw deployment, you have one gateway process, one config file, and one secrets loader. Every API key, every bot token, every service credential is visible to every agent in that process.

I looked at my own setup: 14 secrets loaded into a single gateway, shared across 15 agents. My personal Telegram bot token was visible to my work agents. My consulting agents could theoretically read my personal Notion API key. The blast radius of a single compromised agent was... everything.

## The Problem with One Big Gateway

Here's the thing about single-gateway deployments: they're simple and they work. One config, one process, one set of logs. When you're getting started with OpenClaw, this is exactly what you want.

But as your agent fleet grows, that simplicity becomes a liability. Consider a typical multi-purpose setup:

- **Personal agents** — your daily assistant, insurance helper, travel planner
- **Work agents** — project management, team coordination
- **Dev tools** — GitHub monitoring, CI/CD automation
- **Client-facing agents** — consulting bots, sales assistants

Each of these groups has different trust boundaries. Your personal Slack token has no business being accessible to a consulting agent. Your client's API keys shouldn't be visible to your personal assistant. Yet in a single-gateway deployment, they all share the same environment variables.

Microsoft's article calls this out explicitly: when skills and external instructions converge in the same runtime, you get a dual supply chain risk. A malicious or compromised plugin in one agent's workflow can harvest credentials meant for an entirely different agent.

## The Multi-Gateway Solution

OpenClaw supports two environment variables that make isolation possible: `OPENCLAW_CONFIG_PATH` and `OPENCLAW_STATE_DIR`. Instead of a single `openclaw.json` and a shared state directory, you can run multiple gateway processes, each with its own config, its own secrets, and its own state.

The architecture looks like this:

```
Tier 1: Personal Gateway
├── Config: ~/.openclaw/configs/openclaw-personal.json
├── Secrets: loaded by start-personal.sh (only personal keys)
├── State: ~/.openclaw-personal/
└── Service: ai.openclaw.gateway.personal (launchd)

Tier 2: Work Gateway
├── Config: ~/.openclaw/configs/openclaw-work.json
├── Secrets: loaded by start-work.sh (only work keys)
├── State: ~/.openclaw-work/
└── Service: ai.openclaw.gateway.work (launchd)

... and so on for each tier
```

Each tier gets:
- **Its own config file** — containing only the agents, channels, and bindings for that tier
- **Its own launcher script** — which reads only that tier's secrets from the macOS Keychain
- **Its own API keys** — separate OpenAI keys, separate Notion integrations, separate bot tokens
- **Its own gateway token** — so tiers can't call each other's gateway APIs
- **Its own launchd service** — managed independently, restartable without affecting other tiers
- **Its own state directory** — isolated logs, sessions, and runtime data

The key insight: your configs still use generic environment variable names (`${OPENCLAW_TELEGRAM_BOT_TOKEN}`, `${OPENCLAW_OPENAI_API_KEY}`). The *launcher scripts* are what map these to different actual values per tier. The personal launcher reads `openclaw-personal-telegram-token` from the keychain; the work launcher reads `openclaw-work-telegram-token`. Same env var name, different secret values, completely isolated processes.

## What We Learned Along the Way

Splitting a running deployment isn't just "copy the config five times." A few things we discovered:

**OpenClaw uses three ports per gateway.** The base port you configure, plus ports at base+2 and base+3 for internal services (browser control, etc.). If your gateways are on consecutive ports, they'll collide. Space them at least 10 apart.

**Channel tokens must live only in the `accounts` section.** In the original single-gateway config, having a `botToken` at both the top-level channel config and inside the account config worked fine because they were different bots. In a per-tier config with one bot per channel, having the same token at both levels causes the gateway to create two listeners — and your bot responds to every message twice. Tokens go in `accounts` only.

**GitHub credentials are separate from OpenClaw secrets.** The `gh` CLI stores tokens in the macOS login keychain, not in OpenClaw's keychain. Agents that work exclusively through `gh` (like DevBot and Monitor) need zero OpenClaw API secrets — just a Telegram token for notifications and a gateway token for authentication.

**Pairing data doesn't carry over.** Each tier's gateway has its own state directory, which means Telegram pairing approvals from the old single gateway don't exist in the new ones. Every user needs to re-pair with each bot after the migration. The launcher scripts help here — they can run arbitrary commands with the tier's env vars loaded:

```bash
bash ~/.openclaw/scripts/start-work.sh openclaw pairing approve telegram <CODE>
```

## How the Skills Evolved

The [cc-openclaw](https://github.com/rahulsub-be/cc-openclaw) skills now support both single-gateway and multi-gateway deployments. Every skill auto-detects the deployment mode:

```bash
TIER_CONFIGS=(~/.openclaw/configs/openclaw-*.json)
[[ -f "${TIER_CONFIGS[0]}" ]] && MULTI_GATEWAY=true || MULTI_GATEWAY=false
```

If tier configs exist, the skill operates in multi-gateway mode. If not, it falls back to the original single-gateway behavior. No configuration needed — it just works based on what's on disk.

Here's what changed in each skill:

### `/openclaw-status` — now shows per-tier health

Instead of checking one port and one log file, the status skill discovers all tiers dynamically, reads each config's `gateway.port`, and presents a per-tier health dashboard. You see which gateways are running, which channels are connected per tier, and where errors are occurring — all in one view.

### `/openclaw-restart` — now takes a tier argument

`/openclaw-restart work` restarts only the work tier. `/openclaw-restart all` bounces everything. In single-gateway mode, it works exactly as before.

### `/openclaw-add-secret` — tier-aware keychain management

Secrets now follow a `openclaw-<tier>-<service>` naming convention in the keychain, and the skill updates the tier's launcher script instead of the old shared `openclaw-secrets.sh`. The skill asks which tier a secret belongs to and wires it into the right places.

### `/openclaw-add-channel` — with the duplicate response fix baked in

The skill now resolves which tier an agent belongs to (by scanning configs), edits the correct tier config, and stores tokens with tier-prefixed keychain names. It also enforces the "tokens only in accounts" rule to prevent the duplicate response bug we discovered.

### `/openclaw-new-agent` — asks which tier

When creating a new agent, the skill now asks which tier it should be registered in, then updates only that tier's config. Agent directories remain shared at `~/.openclaw/agents/` — only the config registration determines which gateway manages an agent.

### The rest — minor but consistent

`/openclaw-add-cron`, `/openclaw-dream-setup`, `/openclaw-add-script`, and `/openclaw-stow` all received the same treatment: agent-to-tier resolution for targeted restarts and tier-specific log verification.

## Should You Adopt Multi-Gateway?

If you're running a handful of personal agents with no sensitive client data, a single gateway is fine. The skills still support it, and the simplicity is a feature.

But if any of these apply to you, it's worth the migration:

- **You mix personal and work agents** in the same deployment
- **You handle client credentials** that should be isolated from your personal keys
- **You run agents with different trust levels** — some process untrusted input (public Telegram bots), others have privileged access (GitHub, cloud infrastructure)
- **You want blast radius control** — a compromised agent should only expose that tier's secrets, not everything

The migration itself is straightforward:

1. **Split your config** into per-tier files under `~/.openclaw/configs/`
2. **Create launcher scripts** that load only each tier's secrets from the keychain
3. **Create separate API keys** per tier (this is the manual work — visiting service dashboards)
4. **Set up launchd plists** per tier for process management
5. **Re-pair Telegram users** with each bot

The cc-openclaw skills handle the ongoing operations from there. Once you're multi-gateway, every `/openclaw-` command is tier-aware automatically.

## What Microsoft Got Right

The security article isn't about OpenClaw specifically — it's about a class of risk that applies to any self-hosted agent system. But the recommendations map cleanly onto what we built:

- **Identity isolation** — each tier has its own API keys and tokens
- **Runtime isolation** — separate processes with separate environment variables
- **Credential rotation** — per-tier secrets can be rotated independently
- **Blast radius control** — a compromised tier exposes only its own secrets
- **Audit trail** — per-tier logs make it clear which gateway handled what

The one thing we didn't implement (yet) is network-level isolation between tiers. Right now, all gateways bind to loopback — they're only accessible locally. But they share the same network namespace. For a truly hardened deployment, you'd want each tier in its own container or VM. That's a problem for another day.

## Try It

The updated skills are available at [github.com/rahulsub-be/cc-openclaw](https://github.com/rahulsub-be/cc-openclaw). Clone, stow, and you've got multi-gateway-aware operations tooling:

```bash
git clone https://github.com/rahulsub-be/cc-openclaw.git ~/cc-openclaw
cd ~/cc-openclaw && stow --no-folding -t ~/your-openclaw-home .
```

If you're already using cc-openclaw, just `git pull` — the symlinks mean the updated skills take effect immediately.

The configuration patterns we worked out — port spacing, account-only tokens, tier-prefixed keychain entries, launcher-as-env-loader — took some trial and error to get right. The skills encode all of it so you don't have to repeat our mistakes.

Non-deterministic systems need deterministic configuration management. And now, they need deterministic *isolation* too.

---

*This is a follow-up to [Managing OpenClaw with Claude Code](https://trilogyai.substack.com/p/managing-openclaw-with-claude-code). The cc-openclaw skills are open source under MIT.*

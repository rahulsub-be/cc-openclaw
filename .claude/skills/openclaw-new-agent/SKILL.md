---
name: openclaw-new-agent
description: Create a new OpenClaw agent with all required directive files, directory structure, and config registration
argument-hint: [agent-id] [agent-name]
disable-model-invocation: true
---

# Create New OpenClaw Agent

Create a new agent with id `$0` and display name `$1`.

## Reference
Read the full playbook at `~/.openclaw/workspace/skills/01-first-agent-setup.md` for detailed guidance.

## Setup Detection

Before starting, detect the user's openclaw-home repo location and deployment mode:
```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
TIER_CONFIGS=(~/.openclaw/configs/openclaw-*.json)
[[ -f "${TIER_CONFIGS[0]}" ]] && MULTI_GATEWAY=true || MULTI_GATEWAY=false
```
If `OPENCLAW_REPO` is empty, ask the user where their openclaw-home repo is.

## Steps

1. **Create directories:**
```bash
AGENT_ID="$0"
mkdir -p ~/.openclaw/agents/$AGENT_ID/{memory,memory/archives,sessions,scripts,scripts/lib,qmd,drafts,refs}
```

2. **Ask the user** for:
   - Agent's role/purpose (1-2 sentences)
   - Standalone top-level agent or sub-agent? If sub-agent, which parent?
   - Model choice: sonnet-4-5 (default), sonnet-4-6, opus-4-6, or haiku-4-5
   - Will it need a messaging channel? (Telegram/Slack/none)
   - **Multi-gateway only:** Which tier should this agent belong to? List available tiers:
     ```bash
     for cfg in ~/.openclaw/configs/openclaw-*.json; do
       basename "$cfg" | sed 's/openclaw-//;s/.json//'
     done
     ```

3. **Generate 6 core .md files** in the openclaw-home repo at `.openclaw/agents/$0/`:
   - `SOUL.md` — Identity, personality, responsibilities, boundaries. For focused sub-agents, keep it under 60 lines with clear responsibilities and key info. For orchestrators, include sub-agent routing and heartbeat monitoring.
   - `IDENTITY.md` — Name, role, model, emoji
   - `USER.md` — Who the agent serves, their timezone, communication preferences
   - `AGENTS.md` — Session startup checklist (read SOUL.md → USER.md → MEMORY.md), workspace hygiene rules, safety rules
   - `TOOLS.md` — Document any CLI tools, browser profile, account flags
   - `SECURITY.md` — Use the standard 5-section template: Credential Handling, Browser Authentication, Cross-Agent Isolation, Disclosure Prevention, Incident Response

4. **Register in config:**

### Multi-gateway
Read `$OPENCLAW_REPO/.openclaw/configs/openclaw-<tier>.json`, add to `agents.list`:
```json
{
  "id": "<agent-id>",
  "name": "<agent-name>",
  "workspace": "$HOME/.openclaw/agents/<agent-id>",
  "agentDir": "$HOME/.openclaw/agents/<agent-id>",
  "model": {"primary": "anthropic/<chosen-model>"}
}
```

### Single-gateway
Read `$OPENCLAW_REPO/.openclaw/openclaw.json`, add to `agents.list` with the same structure.

Replace `$HOME` with the user's actual home directory path.

5. **If sub-agent:** Add to parent's `subagents.allowAgents` array in the same config file (the parent must be in the same tier in multi-gateway mode).

6. **If channel requested:** Follow the `/openclaw-add-channel` skill

7. **Stow and restart:**
```bash
rm -f ~/.openclaw/cron/jobs.json
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
```

### Multi-gateway
```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway.<tier>
```

### Single-gateway
```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

8. **Verify:** Check gateway logs for the new agent, send a test message if channel configured.
- Multi-gateway: `tail -30 ~/.openclaw-<tier>/gateway.log`
- Single-gateway: `tail -30 ~/.openclaw/logs/gateway.log`

## Important
- All files go in the **openclaw-home repo**, NOT directly in `~/.openclaw/agents/<id>/` — stow handles the symlinks
- Agent directories at `~/.openclaw/agents/` are shared across all tiers — only the config registration determines which gateway manages an agent
- Never put actual secrets in any .md file — use `${VAR}` references
- SECURITY.md is non-negotiable for every agent

---
name: openclaw-add-cron
description: Create a new scheduled cron job for an OpenClaw agent (recurring, interval, or one-shot)
argument-hint: [agent-id]
disable-model-invocation: true
---

# Add Cron Job

Create a new scheduled job for agent `$ARGUMENTS`.

## Reference
Read `~/.openclaw/workspace/skills/05-cron-system.md` for detailed guidance.

## Setup Detection

```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
TIER_CONFIGS=(~/.openclaw/configs/openclaw-*.json)
[[ -f "${TIER_CONFIGS[0]}" ]] && MULTI_GATEWAY=true || MULTI_GATEWAY=false
```

If multi-gateway, resolve the agent's tier for log verification:
```bash
for cfg in ~/.openclaw/configs/openclaw-*.json; do
  if grep -q "\"id\": *\"$ARGUMENTS\"" "$cfg" 2>/dev/null; then
    TIER=$(basename "$cfg" | sed 's/openclaw-//;s/.json//')
    break
  fi
done
```

## Steps

1. **Read current jobs:** Read `$OPENCLAW_REPO/.openclaw/cron/jobs.json`

2. **Ask the user:**
   - What should the job do? (describe the task)
   - Schedule type:
     - **cron** — recurring at specific times (e.g., "every day at 9am", "every 15 min")
     - **at** — one-shot at a specific date/time
     - **every** — fixed interval (e.g., "every 30 minutes")
   - For cron: the cron expression and timezone (default: user's local timezone)
   - For at: the ISO 8601 timestamp
   - Payload type:
     - **agentTurn** — agent wakes up and processes a task (needs a message)
     - **systemEvent** — simple notification (no agent computation)
   - Session target: `isolated` (default, recommended) or `main`
   - Delivery channel: `telegram`, `slack`, or `last`
   - Should it wrap a deterministic script? If so, which one?

3. **Generate UUID:**
```bash
python3 -c "import uuid; print(uuid.uuid4())"
```

4. **Create the job entry:**
```json
{
  "id": "<generated-uuid>",
  "agentId": "<agent-id>",
  "name": "<descriptive name>",
  "enabled": true,
  "createdAtMs": <current-epoch-ms>,
  "schedule": {
    "kind": "cron",
    "expr": "<cron-expression>",
    "tz": "<timezone>"
  },
  "sessionTarget": "isolated",
  "wakeMode": "now",
  "payload": {
    "kind": "agentTurn",
    "message": "<task description>",
    "model": "anthropic/claude-sonnet-4-5",
    "timeoutSeconds": 120
  },
  "delivery": {
    "mode": "announce",
    "channel": "last"
  }
}
```

For one-shot reminders, add `"deleteAfterRun": true`.

5. **Add to jobs.json** in the openclaw-home repo

6. **Stow:**
```bash
rm -f ~/.openclaw/cron/jobs.json
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
```

7. **Verify:** The gateway picks up new jobs on the next cron tick. Check logs:
- Multi-gateway: `tail -30 ~/.openclaw-$TIER/gateway.log`
- Single-gateway: `tail -30 ~/.openclaw/logs/gateway.log`

## Important
- Always set `timeoutSeconds` for agentTurn payloads (prevents runaway sessions)
- Use `isolated` session target for scheduled tasks (cheaper, cleaner)
- The gateway overwrites `jobs.json` at runtime — always edit in the repo and stow
- Cron expressions without `tz` default to UTC

---
name: openclaw-status
description: Check OpenClaw gateway health, channel connectivity, agent status, and recent cron job results
disable-model-invocation: true
allowed-tools: Bash, Read, Grep
---

# OpenClaw Status Check

Check the health and status of the entire OpenClaw deployment.

## Setup Detection

```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
```

Detect deployment mode — **multi-gateway** or **single-gateway**:
```bash
TIER_CONFIGS=(~/.openclaw/configs/openclaw-*.json)
[[ -f "${TIER_CONFIGS[0]}" ]] && MULTI_GATEWAY=true || MULTI_GATEWAY=false
```

## Steps — Multi-Gateway Mode

If `MULTI_GATEWAY=true`, check each tier independently:

1. **Discover tiers and ports:**
For each config in `~/.openclaw/configs/openclaw-*.json`:
```bash
TIER=$(basename "$cfg" | sed 's/openclaw-//;s/.json//')
PORT=$(grep -o '"port": *[0-9]*' "$cfg" | grep -o '[0-9]*')
```

2. **Gateway health per tier:**
```bash
curl -s http://localhost:$PORT/health
```

3. **LaunchAgent status per tier:**
```bash
launchctl print gui/$(id -u)/ai.openclaw.gateway.$TIER 2>&1 | head -5
```

4. **Channel connectivity per tier** — check each tier's log:
```bash
tail -50 ~/.openclaw-$TIER/gateway.log | grep -E "telegram|slack|whatsapp|starting provider|socket mode|error"
```

5. **WhatsApp restart loop** — check any tier whose config contains `"whatsapp"`:
```bash
tail -50 ~/.openclaw-$TIER/gateway.log | grep -c "auto-restart attempt"
```
If count > 3, WhatsApp is in a restart loop (needs re-auth: delete `~/.openclaw/credentials/whatsapp/default/` and restart).

6. **Agent count per tier:** Read each tier's config and count entries in `agents.list`.

7. **Cron job status:** Read `~/.openclaw/cron/jobs.json` and for each enabled job, report:
   - Job name
   - Agent ID (and which tier it belongs to)
   - Last run time and status
   - Consecutive errors (if any)
   - Next scheduled run

8. **Recent errors per tier:**
```bash
tail -100 ~/.openclaw-$TIER/gateway.log | grep -i "error\|fail\|crash" | tail -10
```

9. **Present results** as a per-tier summary table:
   - Each tier: name, port, running/stopped (PID), channel connectivity
   - Agents: total per tier, which have channel bindings
   - Cron: jobs with errors or that haven't run recently

## Steps — Single-Gateway Mode

If `MULTI_GATEWAY=false`, fall back to single-gateway checks:

1. **Read port from config:**
```bash
PORT=$(grep -o '"port": *[0-9]*' ~/.openclaw/openclaw.json | grep -o '[0-9]*')
```

2. **Gateway health:**
```bash
curl -s http://localhost:$PORT/health
```

3. **LaunchAgent status:**
```bash
launchctl print gui/$(id -u)/ai.openclaw.gateway 2>&1 | head -5
```

4. **Channel connectivity:**
```bash
tail -50 ~/.openclaw/logs/gateway.log | grep -E "telegram|slack|whatsapp|starting provider|socket mode|error"
```

5. **WhatsApp restart loop check:**
```bash
tail -50 ~/.openclaw/logs/gateway.log | grep -c "auto-restart attempt"
```

6. **Agent count:** Read `~/.openclaw/openclaw.json` and count entries in `agents.list`.

7. **Cron job status:** Read `~/.openclaw/cron/jobs.json` — same as multi-gateway.

8. **Recent errors:**
```bash
tail -100 ~/.openclaw/logs/gateway.log | grep -i "error\|fail\|crash" | tail -10
```

9. **Present results** in a clear summary table:
   - Gateway: running/stopped (PID, uptime)
   - Channels: which are connected, which are failing
   - Agents: total count, which have channel bindings
   - Cron: jobs with errors or that haven't run recently

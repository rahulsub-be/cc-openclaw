---
name: openclaw-status
description: Check OpenClaw gateway health, channel connectivity, agent status, and recent cron job results
disable-model-invocation: true
allowed-tools: Bash, Read, Grep
---

# OpenClaw Status Check

Check the health and status of the entire OpenClaw deployment.

## Steps

1. **Gateway health:**
```bash
curl -s http://localhost:18789/health
```

2. **LaunchAgent status:**
```bash
launchctl print gui/$(id -u)/ai.openclaw.gateway 2>&1 | head -5
```

3. **Channel connectivity** — check last 50 lines of gateway log for channel startup:
```bash
tail -50 ~/.openclaw/logs/gateway.log | grep -E "telegram|slack|whatsapp|starting provider|socket mode|error"
```

4. **WhatsApp restart loop check:**
```bash
tail -50 ~/.openclaw/logs/gateway.log | grep -c "auto-restart attempt"
```
If count > 3, WhatsApp is in a restart loop (needs re-auth: delete `~/.openclaw/credentials/whatsapp/default/` and restart).

5. **Agent count:** Read `~/.openclaw/openclaw.json` and count entries in `agents.list`.

6. **Cron job status:** Read `~/.openclaw/cron/jobs.json` and for each enabled job, report:
   - Job name
   - Agent ID
   - Last run time and status
   - Consecutive errors (if any)
   - Next scheduled run

7. **Recent errors:** Check for errors in gateway log:
```bash
tail -100 ~/.openclaw/logs/gateway.log | grep -i "error\|fail\|crash" | tail -10
```

8. **Present results** in a clear summary table:
   - Gateway: running/stopped (PID, uptime)
   - Channels: which are connected, which are failing
   - Agents: total count, which have channel bindings
   - Cron: jobs with errors or that haven't run recently

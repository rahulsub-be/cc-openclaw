---
name: openclaw-restart
description: Restart the OpenClaw gateway — handles stow conflicts (jobs.json) and verifies all channels reconnect
disable-model-invocation: true
---

# Restart OpenClaw Gateway

Restart the gateway with automatic stow conflict resolution.

## Steps

1. **Resolve stow conflicts:**
```bash
rm -f ~/.openclaw/cron/jobs.json
```
The gateway overwrites `jobs.json` as a real file on every startup, breaking the stow symlink.

2. **Re-stow config:**
```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
```

3. **Restart gateway:**
```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

4. **Wait for startup** (5 seconds), then check logs:
```bash
sleep 5 && tail -30 ~/.openclaw/logs/gateway.log
```

5. **Verify channels connected:**
   - Look for `[telegram] [<agent>] starting provider` lines (one per Telegram bot)
   - Look for `[slack] socket mode connected` lines
   - Check if WhatsApp is starting or in restart loop
   - Note the PID and port number

6. **Report status** to the user:
   - Gateway PID and port
   - Which Telegram bots started
   - Slack connection status
   - WhatsApp status (connected / restart loop / not configured)
   - Any errors in the log

---
name: openclaw-restart
description: Restart OpenClaw gateway(s) — handles stow conflicts and verifies channels reconnect. Supports single-gateway and multi-gateway (per-tier) deployments.
argument-hint: "[tier-name | all]"
disable-model-invocation: true
---

# Restart OpenClaw Gateway

Restart the gateway with automatic stow conflict resolution. Argument `$ARGUMENTS` is the tier name to restart, or "all" to restart every tier. If empty, auto-detect deployment mode.

## Setup Detection

```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
TIER_CONFIGS=(~/.openclaw/configs/openclaw-*.json)
[[ -f "${TIER_CONFIGS[0]}" ]] && MULTI_GATEWAY=true || MULTI_GATEWAY=false
```

## Steps

1. **Resolve stow conflicts:**
```bash
rm -f ~/.openclaw/cron/jobs.json
```
The gateway overwrites `jobs.json` as a real file on every startup, breaking the stow symlink.

2. **Re-stow config:**
```bash
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
```

3. **Restart gateway(s):**

### Multi-gateway mode (`MULTI_GATEWAY=true`)

If `$ARGUMENTS` is a specific tier name:
```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway.$ARGUMENTS
```

If `$ARGUMENTS` is "all" or empty:
```bash
for cfg in ~/.openclaw/configs/openclaw-*.json; do
  TIER=$(basename "$cfg" | sed 's/openclaw-//;s/.json//')
  launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway.$TIER
done
```

### Single-gateway mode (`MULTI_GATEWAY=false`)
```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

4. **Wait for startup** (5 seconds), then check logs:

### Multi-gateway
```bash
sleep 5
for each restarted tier:
  tail -30 ~/.openclaw-$TIER/gateway.log
```

### Single-gateway
```bash
sleep 5 && tail -30 ~/.openclaw/logs/gateway.log
```

5. **Verify channels connected:**
   - Look for `[telegram] [<agent>] starting provider` lines (one per Telegram bot)
   - Look for `[slack] socket mode connected` lines
   - Check if WhatsApp is starting or in restart loop
   - Note the PID and port number

6. **Report status** to the user:
   - Per tier (multi) or overall (single): PID, port, channel status
   - Which Telegram bots started
   - Slack connection status
   - WhatsApp status (connected / restart loop / not configured)
   - Any errors in the log

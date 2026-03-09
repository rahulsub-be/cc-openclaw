---
name: openclaw-add-channel
description: Add a messaging channel binding (Telegram, Slack, WhatsApp, or GChat) to an existing OpenClaw agent
argument-hint: [agent-id] [telegram|slack|whatsapp|gchat]
disable-model-invocation: true
---

# Add Channel to Agent

Add `$1` channel to agent `$0`.

## Reference
Read `~/.openclaw/workspace/skills/07-channel-configuration.md` for detailed guidance.

## Setup Detection

```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
TIER_CONFIGS=(~/.openclaw/configs/openclaw-*.json)
[[ -f "${TIER_CONFIGS[0]}" ]] && MULTI_GATEWAY=true || MULTI_GATEWAY=false
```

## Resolve Config File

### Multi-gateway
Find which tier the agent belongs to:
```bash
for cfg in ~/.openclaw/configs/openclaw-*.json; do
  if grep -q "\"id\": *\"$0\"" "$cfg" 2>/dev/null; then
    TIER=$(basename "$cfg" | sed 's/openclaw-//;s/.json//')
    CONFIG="$OPENCLAW_REPO/.openclaw/configs/openclaw-$TIER.json"
    break
  fi
done
```
If the agent isn't found in any tier config, ask the user which tier it belongs to.

### Single-gateway
```bash
CONFIG="$OPENCLAW_REPO/.openclaw/openclaw.json"
```

## Steps

1. **Read current config:** Read `$CONFIG` to see existing channels and bindings.

2. **Depending on channel type:**

### Telegram
- Ask user for the bot token (from @BotFather)
- Store in keychain:
  - Multi-gateway: `security add-generic-password -s "openclaw-$TIER-telegram-$0-bot-token" -a "openclaw" -w "<TOKEN>" ~/.openclaw/openclaw.keychain-db`
  - Single-gateway: `security add-generic-password -s "openclaw.telegram-$0-bot-token" -a "openclaw" -w "<TOKEN>" ~/.openclaw/openclaw.keychain-db`
- Add to `channels.telegram.accounts` in the config (NEVER add botToken at the top-level telegram config — only in accounts):
  ```json
  "$0": {
    "name": "<agent display name>",
    "enabled": true,
    "dmPolicy": "pairing",
    "botToken": "${OPENCLAW_TELEGRAM_<AGENT_UPPER>_BOT_TOKEN}",
    "groupPolicy": "allowlist",
    "streaming": "partial"
  }
  ```
- Add binding: `{"agentId": "$0", "match": {"channel": "telegram", "accountId": "$0"}}`

### Slack
- Ask user for bot token (`xoxb-...`) and app token (`xapp-...`)
- Store both in keychain (same tier-aware naming as Telegram)
- Add to `channels.slack.accounts` with both tokens, `nativeStreaming: true` (NEVER add tokens or nativeStreaming at the top-level slack config — only in accounts)
- Add binding with accountId

### WhatsApp
- Add phone numbers to `channels.whatsapp.allowFrom` array
- Create binding: `{"agentId": "$0", "match": {"channel": "whatsapp"}}`

### GChat
- Add binding: `{"agentId": "$0", "match": {"channel": "gchat", "accountId": "$0"}}`

3. **Update secrets loader script:**

### Multi-gateway
Add export lines to `$OPENCLAW_REPO/.openclaw/scripts/start-$TIER.sh` (before the `exec` block), following the existing `kc()` pattern in the file.

### Single-gateway
Add export lines to BOTH:
- `$OPENCLAW_REPO/.openclaw/scripts/openclaw-secrets.sh`
- `$OPENCLAW_REPO/.openclaw/scripts/openclaw-env.sh`

4. **Update secrets.sh** — Add entry to the SECRETS array in `$OPENCLAW_REPO/secrets.sh`

5. **Stow and restart:**
```bash
rm -f ~/.openclaw/cron/jobs.json
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
```

### Multi-gateway
```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway.$TIER
```

### Single-gateway
```bash
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

6. **Verify:** Check the tier's gateway logs for the new channel starting, then send a test message.
- Multi-gateway: `tail -30 ~/.openclaw-$TIER/gateway.log`
- Single-gateway: `tail -30 ~/.openclaw/logs/gateway.log`

## Important
- Env var naming convention: `OPENCLAW_TELEGRAM_<AGENT_UPPER>_BOT_TOKEN` (uppercase, underscores)
- **Channel tokens go ONLY in the `accounts` section** — never at the top-level channel config. Adding tokens at both levels causes duplicate responses.
- Never echo the token back to the user after storing it

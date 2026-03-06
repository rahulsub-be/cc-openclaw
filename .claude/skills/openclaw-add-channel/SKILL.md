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
```

## Steps

1. **Read current config:** Read `$OPENCLAW_REPO/.openclaw/openclaw.json` to see existing channels and bindings.

2. **Depending on channel type:**

### Telegram
- Ask user for the bot token (from @BotFather)
- Store in keychain:
  ```bash
  security add-generic-password -s "openclaw.telegram-$0-bot-token" -a "openclaw" -w "<TOKEN>" ~/.openclaw/openclaw.keychain-db
  ```
- Add to `channels.telegram.accounts` in openclaw.json:
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
- Store both in keychain
- Add to `channels.slack.accounts` with both tokens, `mode: "socket"`, `nativeStreaming: true`
- Add binding with accountId

### WhatsApp
- Add phone numbers to `channels.whatsapp.allowFrom` array
- Create binding: `{"agentId": "$0", "match": {"channel": "whatsapp"}}`

### GChat
- Add binding: `{"agentId": "$0", "match": {"channel": "gchat", "accountId": "$0"}}`

3. **Update secrets scripts** — Add export lines to BOTH:
   - `$OPENCLAW_REPO/.openclaw/scripts/openclaw-secrets.sh`
   - `$OPENCLAW_REPO/.openclaw/scripts/openclaw-env.sh`

4. **Update secrets.sh** — Add entry to the SECRETS array in `$OPENCLAW_REPO/secrets.sh`

5. **Stow and restart:**
```bash
rm -f ~/.openclaw/cron/jobs.json
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

6. **Verify:** Check gateway logs for the new channel starting, then send a test message.

## Important
- Env var naming convention: `OPENCLAW_TELEGRAM_<AGENT_UPPER>_BOT_TOKEN` (uppercase, underscores)
- Keychain service naming: `openclaw.telegram-<agent-id>-bot-token` (lowercase, hyphens)
- Never echo the token back to the user after storing it

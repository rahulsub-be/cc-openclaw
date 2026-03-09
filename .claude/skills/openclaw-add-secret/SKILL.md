---
name: openclaw-add-secret
description: Add a new secret to the OpenClaw keychain and update the appropriate launcher/secrets script
argument-hint: [keychain-service-name] [ENV_VAR_NAME]
disable-model-invocation: true
---

# Add Secret to Keychain

Add a new secret with keychain service `$0` and environment variable `$1`.

## Reference
Read `~/.openclaw/workspace/skills/06-security-model.md` for the full security model.

## Setup Detection

```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
TIER_CONFIGS=(~/.openclaw/configs/openclaw-*.json)
[[ -f "${TIER_CONFIGS[0]}" ]] && MULTI_GATEWAY=true || MULTI_GATEWAY=false
```

## Steps

1. **Ask the user** for the secret value. Do NOT echo it back after they provide it.

2. **Determine tier** (multi-gateway only): Ask the user which tier this secret belongs to. List available tiers:
```bash
for cfg in ~/.openclaw/configs/openclaw-*.json; do
  basename "$cfg" | sed 's/openclaw-//;s/.json//'
done
```

3. **Store in keychain:**
```bash
security add-generic-password -s "$0" -a "openclaw" -w "<VALUE>" ~/.openclaw/openclaw.keychain-db
```

### Multi-gateway naming convention
- Keychain service: `openclaw-<tier>-<service-name>` (lowercase, hyphens)
- Env var: `OPENCLAW_<SERVICE_NAME>` (uppercase, underscores)

### Single-gateway naming convention
- Keychain service: `openclaw.<service-name>` (lowercase, hyphens)
- Env var: `OPENCLAW_<SERVICE_NAME>` (uppercase, underscores)

4. **Update the secrets loader script:**

### Multi-gateway
Add export line to the tier's launcher script at `$OPENCLAW_REPO/.openclaw/scripts/start-<tier>.sh` (before the `exec` block):
```bash
export $1=$(kc "$0")
```
The launcher script uses a `kc()` helper to read from keychain. Follow the existing pattern in the file.

### Single-gateway
Add export line to `$OPENCLAW_REPO/.openclaw/scripts/openclaw-secrets.sh` (before the `exec` block):
```bash
export $1=$(security find-generic-password -s "$0" -w "$KC")
```

5. **Update openclaw-env.sh** (optional, for shell access) — Add export line to `$OPENCLAW_REPO/.openclaw/scripts/openclaw-env.sh`:
```bash
export $1=$(security find-generic-password -s "$0" -w "$KC" 2>/dev/null)
```
Note: `openclaw-env.sh` uses `2>/dev/null` on all lookups for silent failure.

6. **Update secrets.sh** — Add to the SECRETS array in `$OPENCLAW_REPO/secrets.sh`:
```bash
"$0|$1|<description>"
```

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

8. **Verify** the secret can be read. For multi-gateway, use the launcher script to load env vars:
```bash
bash ~/.openclaw/scripts/start-<tier>.sh printenv "$1" | wc -c
```

For single-gateway:
```bash
source ~/.openclaw/scripts/openclaw-env.sh && printenv "$1" | wc -c
```
Should output a non-zero character count.

## Important
- NEVER echo the secret value back to the user
- NEVER write the secret value to any file (only the keychain)

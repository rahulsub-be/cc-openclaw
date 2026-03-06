---
name: openclaw-add-secret
description: Add a new secret to the OpenClaw keychain and update openclaw-secrets.sh and openclaw-env.sh
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
```

## Steps

1. **Ask the user** for the secret value. Do NOT echo it back after they provide it.

2. **Store in keychain:**
```bash
security add-generic-password -s "$0" -a "openclaw" -w "<VALUE>" ~/.openclaw/openclaw.keychain-db
```

3. **Update openclaw-secrets.sh** — Add export line to `$OPENCLAW_REPO/.openclaw/scripts/openclaw-secrets.sh` (before the `exec` block):
```bash
export $1=$(security find-generic-password -s "$0" -w "$KC")
```

4. **Update openclaw-env.sh** — Add export line to `$OPENCLAW_REPO/.openclaw/scripts/openclaw-env.sh`:
```bash
export $1=$(security find-generic-password -s "$0" -w "$KC" 2>/dev/null)
```
Note: `openclaw-env.sh` uses `2>/dev/null` on all lookups for silent failure.

5. **Update secrets.sh** — Add to the SECRETS array in `$OPENCLAW_REPO/secrets.sh`:
```bash
"$0|$1|<description>"
```

6. **Stow:**
```bash
rm -f ~/.openclaw/cron/jobs.json
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
```

7. **Verify** the secret can be read:
```bash
source ~/.openclaw/scripts/openclaw-env.sh && printenv "$1" | wc -c
```
Should output a non-zero character count.

## Important
- Naming convention: keychain service = `openclaw.<service-name>` (lowercase, hyphens)
- Naming convention: env var = `OPENCLAW_<SERVICE_NAME>` (uppercase, underscores)
- NEVER echo the secret value back to the user
- NEVER write the secret value to any file (only the keychain)

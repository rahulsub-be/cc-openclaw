---
name: openclaw-stow
description: Re-deploy OpenClaw config files via GNU Stow, automatically resolving the known jobs.json conflict
disable-model-invocation: true
---

# Re-Stow OpenClaw Config

Re-deploy configuration files from the openclaw-home repo to $HOME via GNU Stow.

## Setup Detection

Detect the openclaw-home repo location and deployment mode:
```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
TIER_CONFIGS=(~/.openclaw/configs/openclaw-*.json)
[[ -f "${TIER_CONFIGS[0]}" ]] && MULTI_GATEWAY=true || MULTI_GATEWAY=false
```

## Steps

1. **Check for known conflicts:**
```bash
ls -la ~/.openclaw/cron/jobs.json 2>/dev/null
```
If it's a regular file (not a symlink), remove it — the gateway recreates it on every startup:
```bash
rm -f ~/.openclaw/cron/jobs.json
```

2. **Run stow:**
```bash
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
```

3. **If stow reports other conflicts**, check what they are:
   - If it's a file the gateway created (session files, auth tokens), it's safe to remove and re-stow
   - If it's user-created content, back it up first

4. **Verify key symlinks:**

### Single-gateway
```bash
ls -la ~/.openclaw/openclaw.json
ls -la ~/.openclaw/cron/jobs.json
```
Each should point to the openclaw-home repo.

### Multi-gateway
```bash
ls -la ~/.openclaw/openclaw.json
ls -la ~/.openclaw/cron/jobs.json
ls -la ~/.openclaw/configs/openclaw-*.json
ls -la ~/.openclaw/scripts/start-*.sh
```
All should be symlinks pointing to the openclaw-home repo.

5. **Report result:** Confirm stow succeeded and list any warnings.

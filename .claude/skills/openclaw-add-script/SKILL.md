---
name: openclaw-add-script
description: Scaffold a new deterministic shell script for an OpenClaw agent with JSON output, error handling, and the json-response library
argument-hint: [agent-id] [script-name]
disable-model-invocation: true
---

# Create Deterministic Script

Scaffold a new script `$1` for agent `$0`.

## Reference
Read `~/.openclaw/workspace/skills/03-deterministic-scripts.md` for the full pattern.

## Setup Detection

```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
```

## Steps

1. **Ensure json-response library exists.** Check if `$OPENCLAW_REPO/.openclaw/agents/$0/scripts/lib/json-response.sh` exists. If not, create it:
```bash
mkdir -p "$OPENCLAW_REPO/.openclaw/agents/$0/scripts/lib"
```
Then write the standard library with these functions:
- `log()` — redirect to stderr
- `json_timestamp()` — ISO 8601 UTC
- `json_success <operation> <data_json>` — structured success output
- `json_error <operation> <code> <message> [details]` — structured error output
- `parse_quiet_flag` — parse --quiet from args

See `~/.openclaw/workspace/skills/03-deterministic-scripts.md` for the complete library source.

2. **Ask the user** what the script should do. Get:
   - Purpose (what data does it fetch/process?)
   - Inputs (what arguments does it take?)
   - Output (what JSON structure should it return?)
   - Any external commands it calls (gh, gog, curl, etc.)

3. **Create the script** at `$OPENCLAW_REPO/.openclaw/agents/$0/scripts/$1.sh` using this template:

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "$SCRIPT_DIR/lib/json-response.sh"

# ── Args ──────────────────────────────────────
parse_quiet_flag "$@"
set -- "${REMAINING_ARGS[@]}"

ARG1="${1:-}"
if [[ -z "$ARG1" ]]; then
  json_error "<operation-name>" "MISSING_ARG" "Usage: $0 <arg1>"
  exit 1
fi

# ── Main Logic ────────────────────────────────
log "Starting <operation> for $ARG1..."

# TODO: Implement the actual logic here
# - All logging goes to stderr via log()
# - Only final JSON goes to stdout
# - Check exit codes of all commands
# - Use jq for JSON construction

RESULT="placeholder"

# ── Output ────────────────────────────────────
json_success "<operation-name>" "$(jq -n --arg r "$RESULT" '{result: $r}')"
```

4. **Implement the logic** based on user's description, following these rules:
   - `log()` for all human-readable output (goes to stderr)
   - Only `json_success()` or `json_error()` to stdout
   - Check every command's exit code
   - Use `jq` for JSON construction (never echo raw JSON strings)
   - For state changes: verify → lock → act → verify → report

5. **Make it executable:**
```bash
chmod +x "$OPENCLAW_REPO/.openclaw/agents/$0/scripts/$1.sh"
```

6. **Update TOOLS.md** — Add documentation to `$OPENCLAW_REPO/.openclaw/agents/$0/TOOLS.md`:
```markdown
### $1.sh
**Usage:** `bash scripts/$1.sh <args>`
**Output:** JSON with `{success, operation, timestamp, data: {...}}`
**Exit codes:** 0 = success, 1 = error
```

7. **If critical, update SOUL.md** — For scripts that must always be used (never bypassed), add a mandatory rule:
```markdown
## ⛔ MANDATORY
NEVER <do the thing manually> — use `scripts/$1.sh`
```

8. **Stow to deploy:**
```bash
rm -f ~/.openclaw/cron/jobs.json
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
```

## Important
- All scripts live in the **openclaw-home repo** — stow creates symlinks
- stdout is sacred — only JSON output goes there
- stderr is for logging — use `log()` liberally
- Exit code is law — non-zero means the operation failed

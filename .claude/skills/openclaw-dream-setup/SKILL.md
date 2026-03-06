---
name: openclaw-dream-setup
description: Set up nightly dream routine (memory distillation) for an OpenClaw agent including DREAM-ROUTINE.md, cron job, and archive pipeline
argument-hint: [agent-id]
disable-model-invocation: true
---

# Set Up Dream Routine

Configure nightly memory distillation for agent `$ARGUMENTS`.

## Reference
Read `~/.openclaw/workspace/skills/02-memory-system.md` for the full memory system guide.

## Setup Detection

```bash
OPENCLAW_REPO=$(readlink ~/.openclaw/openclaw.json 2>/dev/null | sed 's|/.openclaw/openclaw.json||')
```

## Steps

1. **Create directories:**
```bash
mkdir -p "$OPENCLAW_REPO/.openclaw/agents/$ARGUMENTS/memory/archives"
```

2. **Ask the user:**
   - What time should the dream routine run? (default: 23:00)
   - What timezone? (default: user's local timezone)
   - Any specific focus areas for distillation? (e.g., "focus on project status and commitments")

3. **Create DREAM-ROUTINE.md** at `$OPENCLAW_REPO/.openclaw/agents/$ARGUMENTS/DREAM-ROUTINE.md`:

```markdown
# DREAM-ROUTINE.md

## Trigger
Nightly cron at <time> <timezone>.

## Process
1. Read today's `memory/YYYY-MM-DD.md`
2. Read `MEMORY.md` for long-term context
3. Distill to `memory/YYYY-MM-DD-DISTILLED.md` (max 2,500 tokens)
4. Update `memory/MEMORY-DIGEST.md` (3-day rolling window, max 7,500 tokens)
5. Archive distillations older than 3 days to `memory/archives/`

## Distillation Format
# Distilled — YYYY-MM-DD

## Decisions
- <decision, who, context>

## Project Updates
- <project>: <change>

## New Context
- <new info worth remembering>

## Completed
- <finished tasks>

## Blockers
- <unresolved issues>

## Tomorrow
- <items needing attention>

## Rules
- NEVER include credentials or secrets
- Stay within 2,500 token budget
- Focus on what CHANGED, not status quo
- If no daily log exists, skip gracefully
```

4. **Create MEMORY.md** if it doesn't exist — a starter template at `$OPENCLAW_REPO/.openclaw/agents/$ARGUMENTS/MEMORY.md`:

```markdown
# MEMORY.md — <Agent Name>

## Active Projects
(To be filled as the agent works)

## Key Contacts
(To be filled)

## Standing Rules
(To be filled)
```

5. **Add cron job** to `$OPENCLAW_REPO/.openclaw/cron/jobs.json`. Generate a UUID first:
```bash
python3 -c "import uuid; print(uuid.uuid4())"
```

Add the job:
```json
{
  "id": "<generated-uuid>",
  "agentId": "$ARGUMENTS",
  "name": "<Agent Name> Dream Routine",
  "enabled": true,
  "schedule": {
    "kind": "cron",
    "expr": "0 <hour> * * *",
    "tz": "<timezone>"
  },
  "sessionTarget": "isolated",
  "wakeMode": "now",
  "payload": {
    "kind": "agentTurn",
    "message": "Run your nightly dream routine. Read DREAM-ROUTINE.md and follow the process exactly. Read today's daily log from memory/, distill it, update the rolling digest, and archive old distillations.",
    "model": "anthropic/claude-sonnet-4-5",
    "timeoutSeconds": 120
  },
  "delivery": {
    "mode": "announce",
    "channel": "last"
  }
}
```

6. **Add QMD paths** to `$OPENCLAW_REPO/.openclaw/openclaw.json` memory.qmd.paths (if not already present):
```json
{
  "path": "$HOME/.openclaw/agents/$ARGUMENTS/memory",
  "name": "$ARGUMENTS-memory",
  "pattern": "**/*.md"
},
{
  "path": "$HOME/.openclaw/agents/$ARGUMENTS",
  "name": "$ARGUMENTS-docs",
  "pattern": "*.md"
}
```
Replace `$HOME` with the user's actual home directory path.

7. **Update AGENTS.md** — Add memory loading to the session startup sequence in `$OPENCLAW_REPO/.openclaw/agents/$ARGUMENTS/AGENTS.md`:
```markdown
## Every Session
1. Read `SOUL.md`
2. Read `USER.md`
3. Read `MEMORY.md` — curated long-term context
4. Read `memory/MEMORY-DIGEST.md` — rolling 3-day digest (if exists)
5. Do NOT load raw daily logs on startup
```

8. **Stow:**
```bash
rm -f ~/.openclaw/cron/jobs.json
cd "$OPENCLAW_REPO" && stow --no-folding -t ~ .
```

9. **Verify:** The dream routine will run at the next scheduled time. To test manually, trigger the agent with the dream routine message.

## Important
- Token budgets are critical: 2,500/day distillation, 7,500 rolling digest
- Daily logs (`memory/YYYY-MM-DD.md`) should be gitignored — they're ephemeral
- MEMORY.md is curated and committed — it's the permanent knowledge base
- Stagger dream times across agents (e.g., 20:30, 20:35, 20:40) to avoid concurrent load

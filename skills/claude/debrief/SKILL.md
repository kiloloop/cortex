---
name: debrief
description: Capture a structured session summary and send it to the cortex inbox. Run at the end of any Claude Code session.
---

# /debrief — Session Debrief

Capture what happened during a Claude Code session and deliver a structured summary to the cortex OACP workspace for morning consolidation by `/sync`.

## Arguments

- `/debrief` — generate and send the debrief
- `/debrief --dry-run` — generate and display the summary without sending

## Instructions

When the user runs `/debrief`, do the following:

### 1. Identify the project

Determine the current project name using this fallback chain:

1. Check `CLAUDE.md` for an explicit project identifier
2. Try `git rev-parse --show-toplevel` and use the basename
3. Fall back to the working directory basename

Normalize: lowercase, hyphens (e.g., `web-app`, `data-pipeline`).

### 2. Gather session context

Review the conversation to capture:

- **What Changed** — features built, bugs fixed, reviews completed, decisions made
- **Decisions Made** — key choices and their rationale
- **Blockers** — anything that stopped progress or needs external input
- **Next Steps** — carry-forward items for the next session
- **Files Modified** — run `git diff --name-only HEAD~5..HEAD 2>/dev/null` or `git status --short`
- **Git Activity** — run `git log --oneline --since="8 hours ago"` for recent commits

Use conversation context as the primary source. Git activity supplements but does not replace session memory.

### 3. Generate the summary

Write a structured markdown summary:

```markdown
# Debrief: <project> — <YYYY-MM-DD HH:MM TZ>

## What Changed
- bullet points of accomplishments

## Decisions Made
- key decisions with rationale

## Blockers
- items blocking progress (or "None")

## Next Steps
- [ ] carry-forward items

## Files Modified
- list of key files changed
```

Keep it concise — 2-6 bullet points per section.

### 4. Send to cortex workspace

Deliver the debrief to the cortex OACP workspace using `oacp send`:

```bash
oacp send --project cortex --to sync --from claude --type debrief \
  --subject "Debrief: <project> <YYYY-MM-DD>" \
  --body-file <temp_file>
```

If `oacp send` is not available, fall back to `send_inbox_message.py`:

```bash
python3 <scripts_dir>/send_inbox_message.py cortex \
  --from claude --to sync --type notification \
  --subject "Debrief: <project> <YYYY-MM-DD>" \
  --body-file <temp_file> \
  --oacp-dir "$OACP_HOME"
```

The debrief lands in `$OACP_HOME/projects/cortex/agents/claude/inbox/`.

If `--dry-run` was specified, display the summary and skip sending.

### 5. Confirm

Print:

```
Debrief sent: <project> <date> → cortex/agents/claude/inbox/
```

## Notes

- This skill runs from **any project repo**, not just the cortex repo
- The debrief targets the cortex OACP workspace regardless of which project generated it
- `/sync` (run separately from the cortex repo) consumes these debriefs
- Keep summaries factual and concise — `/sync` aggregates across all agents and projects

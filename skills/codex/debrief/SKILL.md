---
name: debrief
description: Capture a structured session summary and send it to the cortex OACP workspace. Run at the end of any Codex session.
---

# /debrief - Session Debrief (Codex)

Capture what happened during a Codex session and deliver a structured summary to the cortex OACP workspace for morning consolidation by `/sync`.

## Arguments

- `/debrief` - generate and send the debrief
- `/debrief --dry-run` - generate and display the summary without sending

## Instructions

When the user runs `/debrief`, do the following:

### 1. Resolve cortex configuration

Determine where the cortex repo lives:

1. If the current repo contains `config.yaml` and `SSOT.md`, use the current repo root as `CORTEX_DIR`
2. Otherwise prefer `CORTEX_DIR` from the environment
3. Fall back to `~/cortex`

Read `${CORTEX_DIR}/config.yaml` if it exists; otherwise fall back to `${CORTEX_DIR}/config.example.yaml` as a default template. Expand `~` and `$VARS` in the values you read.

Extract:
- `oacp_home` - defaults to `${OACP_HOME:-$HOME/oacp}`
- `project` - defaults to `cortex`
- `cortex_dir` - defaults to `${CORTEX_DIR}`

Use the resolved values as:
- `OACP_HOME` - expanded `oacp_home`
- `CORTEX_PROJECT` - expanded `project`
- `CORTEX_DIR` - expanded `cortex_dir`

If both repo-local `.codex/` settings and global `~/.codex/` settings exist, prefer repo-local conventions first.

### 2. Identify the current project

Determine the source project using this fallback chain:

1. Check `AGENTS.md` for an explicit project identifier
2. Check repo-local `.codex/` notes or workspace config if present
3. Check `CLAUDE.md` for compatibility with repos that still keep project metadata there
4. Try `git rev-parse --show-toplevel` and use the basename
5. Fall back to the working directory basename

Normalize the project name to lowercase with hyphens.

### 3. Gather session context

Review the conversation and current repo state to capture:

- **What Changed** - features built, fixes landed, reviews completed, decisions made
- **Tasks Touched** - tickets, PRs, or work items moved during the session
- **Decisions Made** - key choices and rationale
- **Blockers** - external dependencies, open questions, or follow-up constraints
- **Next Steps** - carry-forward items for the next session
- **Files Modified** - prefer `git status --short`; use commit history only as a supplement

Use the conversation context as the primary source. Git activity supplements but does not replace the session summary.

### 4. Generate the summary

Write a concise markdown summary in this format:

```markdown
# Debrief: <project> - <YYYY-MM-DD HH:MM TZ>

## What Changed
- bullet points of accomplishments

## Tasks Touched
- PRs, issues, or work items (or "None")

## Decisions Made
- key decisions with rationale

## Blockers
- blocking items (or "None")

## Next Steps
- [ ] carry-forward items

## Files Modified
- key files touched during the session
```

Keep it factual and concise - 2-6 bullet points per section.

### 5. Send to the cortex workspace

Deliver the summary through OACP using a standard inbox message:

```bash
oacp send "${CORTEX_PROJECT}" \
  --from codex \
  --to codex \
  --type notification \
  --subject "Debrief: <project> <YYYY-MM-DD>" \
  --body-file <temp_file> \
  --channel debrief \
  --oacp-dir "${OACP_HOME}"
```

Notes:
- This writes the debrief to `$OACP_HOME/projects/cortex/agents/codex/inbox/`
- Use `channel: debrief` and a `Debrief:` subject so `/sync` can filter safely
- If `oacp` is unavailable, report a blocker instead of writing protocol files by hand
- If `--dry-run` was specified, display the summary and skip the send

### 6. Confirm

Print:

```text
Debrief sent: <project> <date> -> cortex/agents/codex/inbox/
```

## Notes

- This skill runs from any project repo, not only from the cortex repo
- Codex skills may be installed globally in `~/.codex/skills/` or project-locally in `.codex/skills/`; the instructions are the same either way
- The debrief targets the cortex OACP workspace regardless of which project generated it
- Keep summaries factual and concise so `/sync` can aggregate them across repos and runtimes

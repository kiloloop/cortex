# Cortex — Cross-Session Memory for Multi-Agent Teams

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/Status-Alpha-orange)](https://github.com/kiloloop/oacp)
[![OACP](https://img.shields.io/badge/OACP-%3E%3D0.1.0-orange.svg)](https://github.com/kiloloop/oacp)
[![Claude Code](https://img.shields.io/badge/Runtime-Claude_Code-6B4FBB.svg)](https://code.claude.com/docs)
[![Codex](https://img.shields.io/badge/Runtime-Codex-74AA9C.svg)](https://github.com/openai/codex)

> **Reference app, not a standalone tool.** Cortex is an [OACP](https://github.com/kiloloop/oacp) example app that shows how to build cross-session memory on top of the protocol. Requires `oacp-cli`.

Cortex solves a common multi-agent problem: **agents forget everything between sessions.**

Each agent debriefs at session end, and a morning sync consolidates everything into a single source of truth (SSOT) — so every agent starts the day knowing what happened yesterday, across all projects and runtimes.

## The Problem

When you run AI agents across multiple sessions and runtimes (Claude Code, Codex, Gemini), each session starts cold:

- Agent A fixes a bug in the morning, Agent B reintroduces it in the afternoon
- Decisions made in one session are forgotten in the next
- No one remembers what was tried, what failed, or what's in progress

Cortex fixes this with two skills, the OACP workspace, and a shared inbox.

## How It Works

```
SESSION END (any repo, any runtime)       MORNING (cortex repo)
====================================       =====================

[claude code session]
  └─ /debrief                             /sync
       │                                    ↑
       └─ oacp send ──→ $OACP_HOME/        │
                        projects/cortex/    │
                        agents/*/inbox/ ────┘
                                            │
[codex session]                             ├──→ SSOT.md
  └─ /debrief                              │
       │                               enrich from:
       └─ oacp send ──→ (same) ────────→ OACP project memory
                                          git log --since=last_sync
```

1. **`/debrief`** runs at the end of every agent session (from any project repo). It captures a structured summary (what was done, decisions made, blockers, next steps) and sends it to the cortex OACP workspace via `oacp send`.

2. **`/sync`** runs once daily (typically morning, from the cortex repo). It reads all unprocessed debriefs from `$OACP_HOME/projects/cortex/agents/*/inbox/`, enriches with project memory and git history, and rewrites `SSOT.md` — a compact (<60 lines) document that any agent can read at session start. Processed debriefs are moved to each agent's `outbox/`.

See [`SSOT.example.md`](SSOT.example.md) for a sample of what the generated output looks like.

Cortex uses OACP end-to-end: `oacp send` for debrief delivery, OACP workspace layout for agent inboxes, shared memory for the SSOT, and `oacp doctor` for health checks.

## Install

### Prerequisites

- [OACP](https://github.com/kiloloop/oacp) installed (`pip install oacp-cli`)
- At least one supported runtime ([Claude Code](https://code.claude.com/docs) or [Codex](https://github.com/openai/codex))

### Assumptions

- Your project repos are cloned locally (cortex enriches from `git log`)
- OACP memory directories exist for projects you want to enrich from
- `/sync` is a manually invoked daily skill, not a daemon — run it yourself each morning

### Quick Start

```bash
# Clone
git clone https://github.com/kiloloop/cortex.git
cd cortex

# Initialize OACP workspace (creates $OACP_HOME/projects/cortex/)
oacp init cortex --repo .

# Create your config
cp config.example.yaml config.yaml
# Edit config.yaml — see "Configure" below

# Install skills into Claude Code
mkdir -p ~/.claude/skills/debrief ~/.claude/skills/sync
cp skills/claude/debrief/SKILL.md ~/.claude/skills/debrief/SKILL.md
cp skills/claude/sync/SKILL.md ~/.claude/skills/sync/SKILL.md

# Install skills into Codex
mkdir -p ~/.codex/skills/debrief ~/.codex/skills/sync
cp skills/codex/debrief/SKILL.md ~/.codex/skills/debrief/SKILL.md
cp skills/codex/sync/SKILL.md ~/.codex/skills/sync/SKILL.md
```

### Configure

Edit `config.yaml` to point at your projects:

```yaml
# OACP workspace — cortex reads debriefs from here
oacp_home: $OACP_HOME
project: cortex

# Where the cortex repo is cloned (skills reference this)
cortex_dir: ~/cortex

enrich_sources:
  oacp_memory:
    base_dir: $OACP_HOME/projects
    pattern: "*/memory/*.md"

  git_repos:
    - path: ~/my-project
      name: my-project
      lookback_days: 1

max_ssot_lines: 60
```

## Usage

```bash
# End of any session — capture what happened
/debrief

# Morning — consolidate everything into SSOT
/sync

# Preview without writing
/sync --dry-run

# Check workspace health
oacp doctor --project cortex
```

## Architecture

```
cortex/                              # repo (kiloloop/cortex)
├── README.md
├── LICENSE                          # Apache-2.0
├── config.yaml                      # Points to OACP workspace
├── SSOT.md                          # Single source of truth (<60 lines)
├── SSOT.example.md                  # Sample output for new users
├── CONTRIBUTING.md
└── skills/
    ├── claude/
    │   ├── debrief/SKILL.md         # Session-end capture
    │   └── sync/SKILL.md            # Morning consolidation
    └── codex/
        ├── debrief/SKILL.md
        └── sync/SKILL.md

$OACP_HOME/projects/cortex/         # OACP workspace (standard layout)
├── agents/
│   ├── claude/{inbox,outbox,dead_letter}/
│   └── codex/{inbox,outbox,dead_letter}/
├── memory/                          # Shared project memory
├── artifacts/                       # Generated outputs
└── workspace.json
```

The repo contains the app logic (skills, config, SSOT). All runtime data (debriefs, agent state) flows through the OACP workspace — same protocol, same tools, same directory layout as any other OACP project.

## How It Fits Into OACP

Cortex is a reference implementation showing how to build on OACP:

| Layer | Repo | What |
|-------|------|------|
| Protocol | [oacp](https://github.com/kiloloop/oacp) | Coordination spec + CLI |
| Skills | [oacp-skills](https://github.com/kiloloop/oacp-skills) | Reusable agent skills (inbox, review loop) |
| **App** | **cortex** | **Cross-session memory (this repo)** |

It demonstrates:
- Using `oacp send` for structured message delivery
- Reading from OACP agent inboxes for data collection
- Sharing memory across runtimes via the OACP workspace
- Building runtime-specific skills (Claude + Codex) for the same workflow

## Built With

- [OACP](https://github.com/kiloloop/oacp) — file-based agent coordination protocol
- [Claude Code](https://code.claude.com/docs) + [Codex](https://github.com/openai/codex) — supported agent runtimes
- Skills from [oacp-skills](https://github.com/kiloloop/oacp-skills) (coming soon)

## License

Apache 2.0 — see [LICENSE](LICENSE).

# PROJECTS.md

## Architecture: Hybrid Sub-Agent Pipeline

Gajala is the only persistent agent. All other agents are ephemeral sub-agents spawned on demand via `sessions_spawn`.

### Pipeline Agents (Ephemeral)

1. **Scout** — Pain Discovery
   - Spawned by Gajala on demand or daily cron (08:00 UTC)
   - Instructions: `agent-lessons/scout-lessons.md`

2. **Validator** — Idea Approval
   - Spawned by Gajala after Scout passes Gate 1
   - Instructions: `agent-lessons/validator-lessons.md`

3. **Architect** — Technical Blueprint
   - Spawned by Gajala after Validator approves an idea (Gate 2)
   - Instructions: `agent-lessons/architect-lessons.md`

4. **Developer** — [NOT ONBOARDED]
   - Pipeline halts after Architect until Developer is manually onboarded

### Output Folder Structure

```
runs/
├── 2026-04-06/
│   ├── cron/                          ← daily 08:00 UTC trigger
│   │   ├── scout-output.json
│   │   ├── validator-output.json
│   │   └── architect-output.json
│   └── ad-hoc/                        ← manual trigger via Discord
│       ├── scout-output.json
│       ├── validator-output.json
│       └── architect-output.json
├── 2026-04-07/
│   └── cron/
│       └── scout-output.json          ← pipeline stopped at Gate 1
└── ...
```

All outputs are committed and pushed to GitHub (`gajala-idea-factory` repo).

### Pipeline Flow

```
Cron (08:00 UTC) or ad-hoc command
  → mkdir runs/{date}/{type}/
  → Gajala spawns Scout → save scout-output.json → git commit + push
  → Gate 1: Scout quality check
  → Gajala spawns Validator → save validator-output.json → git commit + push
  → Gate 2: Validator approval check
  → Gajala spawns Architect → save architect-output.json → git commit + push
  → Gate 3: Blueprint quality check
  → Save blueprint to blueprints/{slug}-spec.json → git commit + push
  → Gajala announces on Discord
  → Pipeline returns to IDLE
```

### Velocity Rule
SCOUTING → ARCHITECTING must complete within 24 hours.

## Current Project: None (IDLE)

## Completed Projects
- **SalesCmd** (HubSpot CRM CLI) — Go, Cobra, Lipgloss, SQLite — spec in `blueprints/`
- **Terminus Flow** (Rainbow Deployment CLI) — spec in `blueprints/`

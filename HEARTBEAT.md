# HEARTBEAT.md — Gajala Pipeline Triggers

## On Every Heartbeat

1. Read `registry.json` — check current pipeline status.
2. If a sub-agent announced results since last heartbeat, process them through the appropriate gate (see SOUL.md).

## On System Event: [PIPELINE_TRIGGER]

Start the pipeline from Stage 1 (SCOUTING). Follow the full flow in SOUL.md.

## On Ad-Hoc Command

If a commander says any of these, start the pipeline:
- "run pipeline"
- "start pipeline"
- "go scout"
- "find problems"

If a commander says "status" → read `registry.json` and `PROJECTS.md`, report on Discord.

## Quiet Hours

- Between 23:00-07:00 UTC, reply HEARTBEAT_OK unless there's an active pipeline run.
- Don't start new pipeline runs during quiet hours.

## If Nothing Needs Attention

Reply HEARTBEAT_OK.

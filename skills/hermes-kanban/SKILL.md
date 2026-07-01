---
name: hermes-kanban
description: Teaches agents to use Hermes Agent's built-in Kanban system for durable multi-agent task orchestration. Workers drive the board through kanban_* tool calls; humans use the CLI, dashboard, or slash commands. Covers solo dev, fleet farming, role pipelines with retry, and circuit breaker patterns.
license: MIT
metadata:
  author: nousresearch
  version: "1.0.0"
  source: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial
compatibility: Requires Hermes Agent with kanban initialized (hermes kanban init). The dispatcher runs inside the gateway (hermes gateway start). Workers use kanban_* tools — no extra CLI install needed.
---

# Hermes Kanban Skill

Teaches agents to use Hermes Agent's built-in Kanban system for durable multi-agent task orchestration. Unlike ad-hoc subagent delegation, Hermes Kanban provides a persistent SQLite-backed board with per-task workspaces, structured handoffs, retry/recovery, and human-in-the-loop support.

This skill covers the four canonical patterns from the official Hermes Kanban tutorial:

1. **Solo dev shipping a feature** — dependency chains with parent→child promotion
2. **Fleet farming** — parallel independent tasks across multiple specialist profiles
3. **Role pipeline with retry** — multi-stage (spec → implement → review → iterate) with block/unblock
4. **Circuit breaker and crash recovery** — auto-block after N failures, PID-level crash detection

## When to Use This Skill

Activate when the user:
- Mentions Hermes Agent kanban or multi-agent board
- Wants to decompose work across multiple named profiles (researcher, writer, engineer, reviewer)
- Needs structured handoff between agent stages (parent summaries + metadata passed downstream)
- Faces work that might need retry, human review, or survive restarts
- Asks about fleet farming, role pipelines, or circuit breaker patterns

## Architecture Overview

### Two surfaces, one DB

| Surface | Who uses it | Interface |
|---|---|---|
| **kanban\_\* toolset** | Agent workers (the model) | `kanban_show`, `kanban_complete`, `kanban_block`, `kanban_heartbeat`, `kanban_comment`, `kanban_create`, `kanban_link`, `kanban_unblock` |
| **CLI / Dashboard / Slash** | Humans & scripts | `hermes kanban …`, `/kanban …`, dashboard GUI |

Both surfaces route through the same `~/.hermes/kanban.db` (SQLite WAL) — reads see a consistent view, writes can't drift.

### Board layout

Six columns: **Triage** → **Todo** → **Ready** → **In Progress** → **Blocked** → **Done**

- **Triage** — raw ideas. Auto-decomposer (default on) fans out into child tasks routed to best-fit specialist profiles. Manual mode lets you decide.
- **Todo** — created but waiting on dependencies, or not yet assigned.
- **Ready** — assigned and waiting for the dispatcher to claim.
- **In Progress** — active worker. Groupable by profile ("Lanes by profile").
- **Blocked** — worker asked for human input, or the circuit breaker tripped.
- **Done** — completed tasks with structured handoff data.

## Quick Start (for the agent to guide the user)

```
# 1. Init the board (one-time)
hermes kanban init

# 2. Start the gateway (hosts the embedded dispatcher)
hermes gateway start

# 3. Create a task
hermes kanban create "research AI funding landscape" --assignee researcher

# 4. Watch activity live
hermes kanban watch
```

The dispatcher polls every 60 s (default) — or hit **Nudge dispatcher** in the dashboard for an immediate tick.

## How Workers Interact with the Board

**Workers do NOT shell out to `hermes kanban`.** When the dispatcher spawns a worker it sets `HERMES_KANBAN_TASK=t_abcd` in the child's env, auto-injecting the `kanban_*` toolset. The model drives the board through tool calls.

### Worker lifecycle (tool call sequence)

1. `kanban_show()` — read title + body + parent handoffs + prior attempts + comments
2. `cd $HERMES_KANBAN_WORKSPACE` — do the work via terminal/file tools
3. `kanban_heartbeat(note="...")` — signal liveness during long ops (at least once per hour)
4. `kanban_complete(summary="...", metadata={...})` or `kanban_block(reason="...")`

### Recommended handoff metadata

```python
kanban_complete(
    summary="migrated limiter.py to token-bucket; added 14 tests, all pass",
    metadata={
        "changed_files": ["limiter.py", "tests/test_limiter.py"],
        "verification": ["pytest tests/ -q"],
        "dependencies": None,
        "blocked_reason": None,
        "retry_notes": None,
        "residual_risk": ["edge case in concurrent refresh not yet tested"],
    },
)
```

These keys are convention, not schema. Every worker should leave enough evidence for the next reader to answer: What changed? How was it verified? What can unblock/retry? What risk remains?

### Orchestrator patterns

A well-behaved orchestrator does NOT do the work itself. It decomposes, creates child tasks, links dependencies, and steps back:

```python
kanban_create(title="research ICP funding, NA angle", assignee="researcher-a", body="...")
kanban_create(title="research ICP funding, EU angle", assignee="researcher-b", body="...")
kanban_create(
    title="synthesize into launch brief",
    assignee="writer",
    parents=["t_r1", "t_r2"],
    body="one-pager, neutral tone",
)
kanban_complete(summary="decomposed into 2 research tasks → 1 writer with dependency chain")
```

The orchestrator should profile-discover first: `kanban_list()` with no filters to see installed profiles, or check which profiles exist on the user's machine.

## Patterns

### Pattern 1: Solo dev with dependency chain

```
schema  ──parent──→  api  ──parent──→  tests
```

- Only `schema` starts in `ready`. The rest stay in `todo` until their parent is `done`.
- When `schema` completes, the dependency engine auto-promotes `api` to `ready`.
- Downstream workers see the upstream summary + metadata via `kanban_show()`.

CLI creation:
```bash
SCHEMA=$(hermes kanban create "Design auth schema" --assignee backend-dev --json | jq -r .id)
API=$(hermes kanban create "Implement auth API" --assignee backend-dev --parent $SCHEMA --json | jq -r .id)
hermes kanban create "Write auth tests" --assignee qa-dev --parent $API
```

### Pattern 2: Fleet farming (parallel independent)

Create many tasks with different assignees. All get claimed in parallel on each dispatcher tick.

```bash
for lang in Spanish French German; do
    hermes kanban create "Translate to $lang" --assignee translator
done
for i in 1 2 3; do
    hermes kanban create "Transcribe call #$i" --assignee transcriber
done
```

The In Progress column groups by profile ("Lanes by profile" default) — see each worker's active task at a glance.

### Pattern 3: Role pipeline with retry

A PM writes a spec → engineer implements → reviewer rejects → engineer retries → reviewer approves.

- If a worker hits a blocking issue, call `kanban_block(reason="...")` — the task transitions to `blocked`, the current run closes with outcome `blocked`, and the block reason is available to the retry worker.
- A human unblocks via dashboard "Unblock" button, CLI (`hermes kanban unblock <id>`), or slash command.
- The retrying worker calls `kanban_show()` and sees the prior attempt's block reason in `worker_context`.

### Pattern 4: Circuit breaker & crash recovery

- **Circuit breaker**: After `kanban.failure_limit` consecutive spawn failures (default 2), the task auto-blocks with outcome `gave_up`. Use `--max-retries N` to override per task. No more retries until a human unblocks.
- **Crash recovery**: The dispatcher polls `kill(pid, 0)`. When a worker PID disappears before the TTL, the claim releases and the task goes back to `ready` for a fresh attempt. The retrying worker sees the prior crash error in its `worker_context`.

## Boards (Multi-project)

Create isolated boards per project / repo / domain:

```bash
hermes kanban boards list
hermes kanban boards create atm10-server --name "ATM10 Server" --switch
hermes kanban --board atm10-server create "Restart server" --assignee ops
```

Workers spawned for a task see ONLY their board's tasks (`HERMES_KANBAN_BOARD` pinned in env). Linking across boards is not allowed.

## File Attachments

Tasks can carry file attachments (up to 25 MB each). Upload via dashboard drawer. Workers see attachments as absolute paths in their context — read directly with file/terminal tools.

## Goal-mode Cards

Pass `--goal` to run a worker in a goal loop (judge checks output against acceptance criteria, continues until done or budget exhausted):

```bash
hermes kanban create "Translate docs to French" \
    --assignee linguist --goal --goal-max-turns 15
```

## Dashboard

Open the visual board:

```bash
hermes dashboard
```

Features: drag-drop columns, inline create, multi-select bulk actions, clickable card drawer with dependency editor, LLM-driven Decompose/Specify buttons on triage cards, profile-lane grouping, search/tenant/assignee filters, and a **Nudge dispatcher** button.

## Auto vs Manual Orchestration

- **Auto** (default: `kanban.auto_decompose: true`) — dispatcher runs the decomposer on triage tasks each tick (up to 3 per tick). Produces a task graph routed to specialist profiles.
- **Manual** (`auto_decompose: false`) — triage tasks wait for you to click **⚗ Decompose** or run `hermes kanban decompose <id>`.
- Toggle via the **Orchestration: Auto/Manual** pill in the dashboard, or `config.yaml`.

## Configuration

Keys under `kanban:` in `~/.hermes/config.yaml`:

| Key | Default | Purpose |
|---|---|---|
| `auto_decompose` | `true` | Auto-run decomposer on triage tasks |
| `auto_decompose_per_tick` | `3` | Cap per dispatcher tick |
| `orchestrator_profile` | `""` | Profile for root task after decomposition |
| `default_assignee` | `""` | Fallback when LLM picks unknown profile |
| `failure_limit` | `2` | Consecutive failures before circuit breaker |
| `dispatch_interval_seconds` | `60` | Dispatcher tick interval |
| `dispatch_stale_timeout_seconds` | `14400` | Max run time without heartbeat (4 h) |

Auxiliary LLM slots: `auxiliary.kanban_decomposer`, `auxiliary.profile_describer`, `auxiliary.triage_specifier`.

## Reference

- Full tutorial: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial
- Overview & CLI reference: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban
- Worker lanes: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes

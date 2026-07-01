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

Covers the four canonical patterns:
1. **Solo dev with dependency chain** — parent→child auto-promotion
2. **Fleet farming** — parallel tasks across multiple specialist profiles
3. **Role pipeline with retry** — spec → implement → review → iterate with block/unblock
4. **Circuit breaker & crash recovery** — auto-block after N failures, PID-level crash detection

## When to Use This Skill

Activate when the user:
- Mentions Hermes Agent kanban or multi-agent board
- Wants to decompose work across named profiles (researcher, writer, engineer, reviewer)
- Needs structured handoff between agent stages
- Faces work that might need retry, human review, or survive restarts
- Asks about fleet farming, role pipelines, or circuit breaker patterns
- Says phrases like "use kanban", "orchestrate with hermes", "multi-agent board"

## The Kanban Workflow

### Step 1: Initialize the Board and Gateway

Set up the foundation:

```bash
# One-time board initialization
hermes kanban init

# Start the gateway (hosts the embedded dispatcher)
hermes gateway start
```

The board has six columns: **Triage** → **Todo** → **Ready** → **In Progress** → **Blocked** → **Done**.

### Step 2: Create Tasks with Profiles and Dependencies

Create tasks with appropriate profiles and dependency linking:

```bash
hermes kanban create "Design auth schema" --assignee backend-dev
hermes kanban create "Implement auth API" --assignee backend-dev --parent $SCHEMA_ID
```

Key creation options:
- `--assignee <profile>` — route task to a specialist profile
- `--parent <id>` — creates dependency chain (child auto-promoted when parent completes)
- `--goal` — enable judge-loop verification
- `--goal-max-turns <N>` — limit goal-mode iterations
- `--max-retries <N>` — override circuit breaker threshold per task

For orchestrator tasks (decomposition), use `kanban_create()` via the toolset to create child cards programmatically.

### Step 3: Workers Claim and Execute via Tool Calls

Workers do NOT shell out to `hermes kanban` — they use the injected `kanban_*` toolset. The dispatcher sets `HERMES_KANBAN_TASK=t_abcd` in the worker's environment.

Worker lifecycle:
1. `kanban_show()` — read title, body, parent handoffs, prior attempts, comments
2. Work in the task workspace (`$HERMES_KANBAN_WORKSPACE`)
3. `kanban_heartbeat(note="...")` — signal liveness during long ops (at least once per hour)
4. `kanban_complete(summary="...", metadata={...})` or `kanban_block(reason="...")`

### Step 4: Structured Handoff

Every completion should leave metadata for the next downstream worker:

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

Every downstream worker should be able to answer from the metadata: **What changed? How was it verified? What can unblock/retry? What risk remains?**

Orchestrator tasks should profile-discover first via `kanban_list()` with no filters to see installed profiles.

### Step 5: Review and Iterate

- Human reviews completed work via dashboard or CLI
- If blocked, human unblocks via dashboard "Unblock" button, `hermes kanban unblock <id>`, or slash command
- Retrying worker sees prior block reason in `worker_context` via `kanban_show()`
- Circuit breaker auto-blocks after `failure_limit` consecutive spawn failures (default 2) — human must unblock

## Orchestration Patterns

### Pattern 1: Solo Dev with Dependency Chain

```
schema  ──parent──→  api  ──parent──→  tests
```

Only the first task starts in `ready`. When a parent completes, the child auto-promotes to `ready`. Downstream workers see the upstream summary + metadata via `kanban_show()`.

```bash
SCHEMA=$(hermes kanban create "Design auth schema" --assignee backend-dev --json | jq -r .id)
API=$(hermes kanban create "Implement auth API" --assignee backend-dev --parent $SCHEMA --json | jq -r .id)
hermes kanban create "Write auth tests" --assignee qa-dev --parent $API
```

### Pattern 2: Fleet Farming (Parallel)

Create many tasks with different assignees — all get claimed in parallel on each dispatcher tick:

```bash
for lang in Spanish French German; do
    hermes kanban create "Translate to $lang" --assignee translator
done
```

The In Progress column groups by profile ("Lanes by profile" default).

### Pattern 3: Role Pipeline with Retry

PM writes spec → engineer implements → reviewer rejects → engineer retries → reviewer approves.

- If blocked, worker calls `kanban_block(reason="...")` — task transitions to `blocked`
- Human unblocks; retrying worker sees prior block reason in `worker_context`

### Pattern 4: Circuit Breaker & Crash Recovery

- **Circuit breaker**: After `kanban.failure_limit` consecutive spawn failures (default 2), task auto-blocks with outcome `gave_up`. Override per task with `--max-retries N`.
- **Crash recovery**: Dispatcher polls `kill(pid, 0)`. When PID disappears before TTL, claim releases, task goes back to `ready`. Retrying worker sees prior crash error in `worker_context`.

## Error Handling

### Gateway Not Running
The dispatcher needs the gateway: `hermes gateway start`. If workers aren't claiming tasks, check if the gateway is active.

### Circuit Breaker Tripped
A task exceeded `failure_limit` consecutive failures. Run `hermes kanban unblock <id>` or use the dashboard "Unblock" button to reset. Consider increasing `--max-retries` for flaky work.

### Worker Crashed (PID Lost)
The dispatcher detected a disappeared PID. The task returns to `ready` automatically for a fresh attempt. The retrying worker sees the prior crash error in `worker_context` via `kanban_show()`.

### Blocked Task (Human Input Needed)
A worker called `kanban_block(reason="...")`. Review the block reason in the dashboard or via `hermes kanban show <id>`. Unblock with the dashboard button, `hermes kanban unblock <id>`, or slash command.

### Auto Decomposer Issues
If the auto-decomposer isn't routing correctly, switch to manual mode: toggle "Orchestration: Auto/Manual" in the dashboard, or set `kanban.auto_decompose: false` in `~/.hermes/config.yaml`. Then use the **⚗ Decompose** button manually.

### Stale Heartbeat
If a worker doesn't call `kanban_heartbeat()` within `dispatch_stale_timeout_seconds` (default 4 hours), the dispatcher considers it stale. The claim releases and the task goes back to `ready`.

## Configuration

### Boards (Multi-project)

Create isolated boards per project:

```bash
hermes kanban boards create atm10-server --name "ATM10 Server" --switch
hermes kanban --board atm10-server create "Restart server" --assignee ops
```

Workers see ONLY their board's tasks (`HERMES_KANBAN_BOARD` pinned in env). Cross-board linking is not allowed.

### Goal-mode Cards

Pass `--goal` to run a worker in a judge loop — checks output against acceptance criteria, continues until done or budget exhausted:

```bash
hermes kanban create "Translate docs to French" --assignee linguist --goal --goal-max-turns 15
```

### Auto vs Manual Orchestration

- **Auto** (default): dispatcher runs the decomposer on triage tasks each tick (up to 3 per tick)
- **Manual**: triage tasks wait for the **⚗ Decompose** button or `hermes kanban decompose <id>`
- Toggle via the "Orchestration: Auto/Manual" pill in the dashboard, or `kanban.auto_decompose` in config

### Key Config Options

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

### Dashboard

Open visual board with `hermes dashboard`. Features: drag-drop columns, inline create, multi-select bulk actions, card drawer with dependency editor, LLM-driven Decompose/Specify buttons on triage cards, profile-lane grouping, search/assignee filters, and **Nudge dispatcher** button.

### File Attachments

Tasks can carry file attachments (up to 25 MB each). Upload via dashboard drawer. Workers see attachment paths in their context.

## References

See `references/card-templates.md` for template examples and `references/decomposition-patterns.md` for task sizing and dependency guidelines.

- Full tutorial: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial
- Overview & CLI reference: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban
- Worker lanes: https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-worker-lanes

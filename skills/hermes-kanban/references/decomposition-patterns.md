# Decomposition Patterns

Guidelines for breaking down software requirements into Hermes Kanban tasks with correct profile assignment, dependency chaining, and structured handoffs.

## Task Sizing Rules

Tasks should take approximately 1–2 hours of worker time. Apply these heuristics:

### Too Big (Split It)
- Task involves >3 files
- Implementation would take >2 hours of continuous work
- Has multiple "and" clauses (e.g., "create API and tests and docs")
- Combines frontend and backend concerns (different profiles needed)
- Requires handoff between roles (architect → implementer → reviewer)

### Too Small (Combine It)
- Only changes one line or adds minor formatting
- Is just renaming or moving files
- Doesn't deliver testable value on its own
- Would produce trivial handoff metadata

## Profile Discovery

Before decomposing, discover which profiles are available on the user's machine:

```bash
# List all boards (profile info embedded in board config)
hermes kanban boards list

# Check which profiles exist per board
hermes kanban list --all   # shows all tasks across all profiles
```

If profile names are not obvious, ask the user or check `~/.hermes/config.yaml` for profile definitions. Common built-in profiles: `backend-dev`, `frontend-dev`, `qa-dev`, `researcher`, `writer`, `translator`, `transcriber`, `ops`, `architect`, `implementer`, `reviewer`, `linguist`.

## Decomposition Patterns

### Pattern 1: Solo Development Chain

Best for: Single-feature implementation where each step depends on the previous.

Chain: Schema → API → UI → Tests

```
Schema ──parent──→ API ──parent──→ UI ──parent──→ Tests
```

Decomposition approach:
1. **Schema card** — assign to primary profile (e.g., `backend-dev`). Starts in READY immediately.
2. **API card** — assign to same profile, create with `--parent $SCHEMA_ID`. Auto-promoted to READY when schema completes.
3. **UI card** — may switch profile (e.g., `frontend-dev`), create with `--parent $API_ID`.
4. **Test card** — may switch profile (e.g., `qa-dev`), create with `--parent $UI_ID`.

Each downstream worker sees the upstream handoff summary + metadata via `kanban_show()`.

### Pattern 2: Fleet Farming (Parallel)

Best for: Embarrassingly parallel work — translations, data entry, multi-region research, batch processing.

```bash
for region in "NA" "EU" "APAC"; do
    hermes kanban create "Research ICP — $region" --assignee researcher
done
hermes kanban create "Synthesize launch brief" --assignee writer
```

Decomposition approach:
1. Create N independent sibling cards with the same or different profiles — all get claimed in parallel (one per dispatcher tick).
2. Optionally add a synthesis/merge card that depends on all siblings via `--parent $ID1 --parent $ID2`.
3. Workers see sibling tasks in `kanban_show()` context if linked.

### Pattern 3: Role Pipeline (Spec → Implement → Review → Iterate)

Best for: Multi-discipline work requiring handoff between different specialists.

```
Spec ──parent──→ Implement ──parent──→ Review ──parent──→ Iterate (if needed)
```

Decomposition approach:
1. **Spec card** — assign to `architect` or `researcher` profile. Outputs design document.
2. **Implement card** — assign to `implementer` or `backend-dev`. Depends on spec card.
3. **Review card** — assign to `reviewer` or `qa-dev`. Depends on implementation card. If review fails, worker calls `kanban_block(reason="...")`.
4. **Iterate** — unblocked by human; retrying worker sees prior block reason in `worker_context` via `kanban_show()`.

### Pattern 4: Circuit Breaker Awareness

For fragile or risky tasks, set conservative limits:

- `--max-retries 3` — allow 3 spawn failures before auto-block.
- Without override, the default `failure_limit` (2) applies.
- Crash recovery: if a worker PID dies before TTL, the claim releases and the task goes back to READY for a fresh attempt.
- The retrying worker sees the prior crash error in its `worker_context`.

Use higher retries for:
- Ops tasks (deployments, restarts)
- Integration tests against flaky environments
- Long-running data migrations

## Dependency Patterns

### Common Dependency Chains

1. **Database-first chain:**
   `Schema → Model → API → UI`
   (Each step depends on the prior; only the current leaf is in READY.)

2. **Spec-first pipeline:**
   `Spec → Implement → Review → Merge`
   (Role handoff; review may block and retry.)

3. **Fan-out / Fan-in:**
   ```
              ┌── Worker A ──┐
   Config ────┼── Worker B ──┼─── Synthesize
              └── Worker C ──┘
   ```
   (All workers run in parallel; synthesizer depends on all.)

### Identifying Dependencies

Ask these questions:
1. Does this task produce artifacts others need? (schema, API, types, spec doc)
2. Must this task run before others? (security setup, config, migration)
3. Can tasks run in parallel? (independent features, different profiles)

## Handoff Metadata Guidelines

Every `kanban_complete()` call should leave enough evidence for the next reader:

```python
kanban_complete(
    summary="what was done in one line",
    metadata={
        # Required: files touched
        "changed_files": ["src/foo.py"],
        # Required: how to verify it works
        "verification": ["pytest src/foo.py", "npm run typecheck"],
        # Optional: any unresolved dependencies
        "dependencies": None,
        # Important: why blocked (omit if not blocked)
        "blocked_reason": None,
        # Helpful: how many attempts, what changed between retries
        "retry_notes": None,
        # Critical: what might still be wrong
        "residual_risk": ["edge case in concurrent writes not handled"],
    },
)
```

Every downstream worker should be able to answer from the metadata:
- **What changed?** — files, configs, dependencies added
- **How was it verified?** — test commands run and passed
- **What can unblock/retry?** — known failure modes or decisions needed
- **What risk remains?** — untested edge cases, tech debt, deferred work

## Edge Case Handling

### Large Projects (>12 cards)

Split into phases grouped by dependency wave:

| Phase | Focus | Profile | Typical Cards |
|---|---|---|---|
| Phase 1: Foundation | Config, schema, infra | `backend-dev`, `ops` | 3–5 |
| Phase 2: Core Features | Main API, business logic | `backend-dev`, `implementer` | 4–8 |
| Phase 3: Frontend | UI components, state | `frontend-dev` | 4–8 |
| Phase 4: Polish | Tests, docs, edge cases | `qa-dev`, `writer` | 2–4 |

Label cards with a `phase-` prefix in the title or body for board filtering / quick scanning.

### Unclear Requirements

Ask the user for:
- Technology stack (React, Vue, Svelte?)
- Database choice (PostgreSQL, MySQL, SQLite?)
- Deployment target (Vercel, Express, static?)
- Available agent profiles (`hermes kanban list` or check config)

### Refactoring Requests

1. Identify affected modules — create one card per module
2. Assign to same profile (keeps context local)
3. Set `--parent` chain if refactors are order-dependent
4. Include "before/after" in acceptance criteria
5. Highlight `residual_risk` in handoff metadata for any deferred cleanup

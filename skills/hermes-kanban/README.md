# hermes-kanban

Teaches agents to use Hermes Agent's built-in Kanban system for durable multi-agent task orchestration. Workers drive the board through `kanban_*` tool calls; humans use the CLI, dashboard, or slash commands. Covers solo dev, fleet farming, role pipelines with retry, and circuit breaker patterns.

## Usage

This skill is automatically available to skills-compatible agents (Cline, Claude Code, Codex, OpenCode, etc.). The agent will activate it when it detects Hermes Kanban or multi-agent task orchestration tasks.

**Trigger phrases:**
- "Use Hermes kanban for this"
- "Set up a multi-agent board"
- "Fleet farming with profiles"
- "Orchestrate with kanban"
- "Role pipeline with retry"

## Prerequisites

- [Hermes Agent](https://hermes-agent.nousresearch.com) with kanban initialized — run `hermes kanban init` from your repo root
- Gateway must be running for the dispatcher: `hermes gateway start`
- No extra CLI install needed — workers use the built-in `kanban_*` toolset

## Structure

```
hermes-kanban/
├── SKILL.md                      # Agent instructions
├── references/
│   ├── decomposition-patterns.md # Task sizing and dependency guidelines
│   └── card-templates.md         # Card content templates and examples
└── scripts/                      # Reserved for future automation
```

## Installation

```bash
npx skills add dohzoh/hermes-kanban-skill
```

Or copy the `hermes-kanban/` directory into your agent's skills directory.

## How It Works

1. **Init** the board (`hermes kanban init`) and start the gateway (`hermes gateway start`)
2. **Create** tasks with assignee profiles using the CLI or dashboard
3. **Workers** claim tasks via the dispatcher and drive work through `kanban_*` tool calls
4. **Complete** or **block** with structured handoff metadata for downstream tasks
5. **Review** board state via `kanban_show()` or the visual dashboard (`hermes dashboard`)

## License

MIT

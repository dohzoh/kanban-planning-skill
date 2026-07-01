# Kanban Skills

A collection of kanban orchestration skills for AI agents.

## Skills

| Skill | Platform | Description |
|---|---|---|
| `skills/cline-kanban/` | Cline Kanban (CLI) | Decompose requirements into structured kanban cards (~1-2 hours) with clear acceptance criteria, files to modify, and dependency chains. |
| `skills/hermes-kanban/` | Hermes Agent | Multi-agent board with durable SQLite-backed tasks, worker tool surface, structured handoffs, retry/recovery, and human-in-the-loop support. |

## Structure

```
skills/
├── cline-kanban/          # Cline Kanban planning skill
│   ├── SKILL.md           # Agent instructions
│   ├── .clinerules        # Cline-specific rules
│   ├── references/        # Templates and decomposition patterns
│   └── docs/              # Design documents
└── hermes-kanban/         # Hermes Agent kanban skill
    └── SKILL.md           # Agent instructions
```

## Usage

Each skill is self-contained. Point your agent's skill loader to the relevant `skills/<name>/` directory, or copy the desired `SKILL.md` into your agent's skill configuration.

## License

MIT

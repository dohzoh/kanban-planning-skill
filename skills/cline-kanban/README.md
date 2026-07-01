# kanban-planning

Decomposes user requirements into fine-grained kanban cards (~1-2 hours each) with clear acceptance criteria, files to modify, and implementation notes. Reduces agent interpretation ambiguity through structured task decomposition.

## Usage

This skill is automatically available to skills-compatible agents (Cline, Claude Code, Codex, OpenCode, etc.). The agent will activate it when it detects planning or task decomposition tasks.

**Trigger phrases:**
- "Break this down into kanban cards"
- "Create tasks for this feature"
- "Plan this implementation"
- "Decompose this request"

## Prerequisites

- [Kanban CLI](https://github.com/cline/kanban) (optional) — run `npx kanban` from your repo root for automated card creation
- Without kanban CLI, the skill outputs structured markdown format for manual card creation

## Structure

```
kanban-planning/
├── SKILL.md                      # Agent instructions
├── references/
│   ├── decomposition-patterns.md # Task sizing and dependency guidelines
│   └── card-templates.md         # Card content templates and examples
└── scripts/                      # Reserved for future automation
```

## Installation

```bash
npx skills add dohzoh/kanban-planning-skill
```

Or copy the `kanban-planning/` directory into your agent's skills directory.

## How It Works

1. **Analyze** the user's request for project type and key components
2. **Check** kanban CLI availability
3. **Identify** major features (2-4 hour chunks)
4. **Decompose** to tasks (~1 hour each)
5. **Identify** dependencies for linking
6. **Format** cards using structured template
7. **Create** cards via CLI or output markdown

## License

MIT

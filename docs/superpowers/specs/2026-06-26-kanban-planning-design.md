# Kanban Planning Skill Design

A structured approach to decompose software development requests into kanban-compatible tasks that minimize agent interpretation ambiguity.

## Problem Statement

When agents receive markdown plans, interpretation variance leads to inconsistent implementations. Kanban cards with structured decomposition reduce this ambiguity by:
1. Explicitly listing files to modify
2. Including specific acceptance criteria
3. Defining clear dependency chains
4. Enabling parallel work through isolated worktrees

## Skill Structure

```
kanban-planning/
├── SKILL.md           # Required: instructions for agents
├── references/
│   ├── decomposition-patterns.md  # Task sizing guidelines
│   └── card-templates.md          # Card content templates
└── scripts/
    └── create-cards.js            # Optional: helper for CLI integration
```

## Core Workflow

### The 7-Step Decomposition Process

1. **Analyze request** - Identify project type and key components
2. **Check kanban availability** - Verify CLI is installed
3. **Identify major features** - 2-4 hour chunks of user-visible functionality
4. **Decompose to tasks** - Each major feature into ~1 hour tasks
5. **Identify dependencies** - Which tasks block others
6. **Format cards** - Apply standard template
7. **Create cards** - Use CLI or output markdown

## Card Template

Each task card includes:
- Title and phase (setup/implementation/testing/refactoring)
- Description with clear scope
- Technical requirements with specific constraints
- Implementation notes with patterns and references
- Files to modify/create list
- Acceptance criteria with testable outcomes
- Dependency declarations

## Kanban Integration

CLI commands for card creation:
```bash
kanban task add --column backlog --title "..." --description "..."
kanban task link --task-id <id> --linked-task-id <prereq-id>
```

## Error Handling

- Missing kanban: Output markdown with setup instructions
- Ambiguous requirements: Ask clarifying questions
- Large requests: Split into phases
- Circular dependencies: Warn and suggest restructuring

## Configuration

Check for kanban CLI before attempting card creation. If unavailable, guide user to install kanban first or accept markdown output format.
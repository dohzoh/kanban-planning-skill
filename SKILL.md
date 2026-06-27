---
name: kanban-planning
description: Decomposes user requirements into fine-grained kanban cards (~1-2 hours each). Creates task breakdowns with clear acceptance criteria, files to modify, and implementation notes. Use when planning software projects to reduce agent interpretation ambiguity through structured task decomposition.
license: MIT
metadata:
  author: cline-community
  version: "1.0.0"
compatibility: Requires kanban CLI (npx kanban). Works without it by outputting structured markdown format.
---

# Kanban Planning Skill

A structured approach to decompose software development requests into kanban-compatible tasks that minimize agent interpretation ambiguity. Each task is sized for ~1-2 hours of focused work with clear acceptance criteria.

## When to Use This Skill

Activate when the user:
- Asks to break down a feature or project into kanban cards
- Wants to plan a software implementation with isolated tasks
- Requests task decomposition for parallel agent execution
- Says phrases like "plan this", "create tasks", "break it down"

## The Decomposition Process

Follow this 7-step algorithm to convert user requirements into kanban cards:

### Step 1: Analyze the Request
Identify the project type, main goals, and key components mentioned. Look for:
- Application type (web app, CLI tool, library, documentation)
- Key features or functionality
- Technical constraints or preferences

### Step 2: Check Kanban Availability
Before decomposition, check if kanban is available:
```bash
npx kanban --version 2>/dev/null && echo "available" || echo "unavailable"
```
- If available: Create cards via CLI after planning
- If unavailable: Output structured markdown format with instructions

### Step 3: Identify Major Features (2-4 hour chunks)
Break the request into 2-4 major features that represent user-visible functionality. Each feature should:
- Deliver measurable value on its own
- Be independent of other features where possible
- Take 2-4 hours to complete

### Step 4: Decompose to Tasks (~1 hour each)
Each major feature splits into ~1 hour tasks that are:
- Independent work units
- Have clear "done" criteria
- Minimize cross-task coordination

### Step 5: Identify Dependencies
Use kanban's linking to create dependency chains:
- Which tasks produce artifacts others need?
- Which tasks must complete before others can start?
- Are there any circular dependencies to resolve?

### Step 6: Format Cards with Templates
Apply the standard card template to each task (see references/card-templates.md).

### Step 7: Create Cards or Output Structure
If kanban CLI is available, use these commands:
```bash
# Add a task to backlog
kanban task add --column backlog --title "..." --description "..."

# Link dependent tasks
kanban task link --task-id <id> --linked-task-id <prerequisite-id>

# Start a task
kanban task start --task-id <id>
```

If kanban is unavailable, output structured markdown format with clear instructions:
"Kanban CLI not detected. Here are the tasks formatted for manual import. To automate card creation, run `npx kanban` from your repository root first."

## Error Handling

### Missing Kanban CLI
- Check with `npx kanban --version` before attempting card creation
- If unavailable, output markdown format with setup instructions
- Guide user: "For automated card creation, run `npx kanban` in your repo first"

### Ambiguous Requirements
Ask clarifying questions on:
- Technology stack (React, Vue, Svelte?, REST or GraphQL?)
- Database choice (PostgreSQL, MySQL, SQLite?)
- Deployment target (Vercel, Express, static?)

### Large Requests (>12 tasks)
Split into phases:
- "Phase 1: Foundation" - setup, config, core schema
- "Phase 2: Core Features" - main functionality
- "Phase 3: Polish" - tests, docs, edge cases

### Circular Dependencies
Detect and warn: "Card 3 depends on Card 2 which depends on Card 3. Suggest making them independent or combining into one card."

## References

See `references/card-templates.md` for template examples and `references/decomposition-patterns.md` for task sizing guidelines.
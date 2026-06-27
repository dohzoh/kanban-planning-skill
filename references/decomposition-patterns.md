# Decomposition Patterns

Guidelines for breaking down software requirements into appropriate kanban tasks.

## Task Sizing Rules

Tasks should take approximately 1-2 hours. Apply these heuristics:

### Too Big (Split It)
- Task involves >3 files
- Implementation takes >2 hours of continuous work
- Has multiple "and" clauses (e.g., "create API and tests and docs")
- Combines frontend and backend concerns

### Too Small (Combine It)
- Only changes one line or adds minor formatting
- Is just renaming or moving files
- Doesn't deliver testable value on its own

## Feature Categories

### Setup Phase (2-4 tasks)
- Database schema
- Project configuration
- Environment setup
- CI/CD pipeline

### Implementation Phase (4-8 tasks)
- API endpoints
- UI components
- Business logic
- State management

### Testing Phase (2-4 tasks)
- Unit tests
- Integration tests
- End-to-end tests

### Documentation Phase (1-2 tasks)
- README updates
- API documentation
- Inline code comments

## Dependency Patterns

### Common Dependency Chains

1. **Database-first pattern:**
   - Schema task → Model task → API task → UI task

2. **API-first pattern:**
   - API task → SDK/Models task → UI task → Tests task

3. **Parallel pattern:**
   - Config task → {Multiple independent feature tasks} → Integration task

### Identifying Dependencies

Ask these questions:
1. Does this task produce artifacts others need? (schema, API, types)
2. Must this task run before others? (security setup, config)
3. Can tasks run in parallel? (independent features)

## Edge Case Handling

### Large Projects (>12 tasks)
Split into phases labeled clearly:
- "Phase 1: Foundation" - setup, config, core schema
- "Phase 2: Core Features" - main functionality
- "Phase 3: Polish" - tests, docs, edge cases

### Unclear Requirements
Ask for clarification on:
- Technology stack (React, Vue, Svelte?)
- Database choice (PostgreSQL, MySQL, SQLite?)
- Deployment target (Vercel, Express, static?)

### Refactoring Requests
- Identify affected modules
- Create one card per module
- Include "before/after" acceptance criteria
- Add migration/dependency cards if needed
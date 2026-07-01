# Card Templates

Standard templates for different task types. Each card follows the core Hermes Kanban structure (title + body with structured sections).

## Template Structure

```markdown
## {Task Title}
**Assignee:** {profile-name}         # e.g. backend-dev, researcher, writer
**Priority:** {high/medium/low}
**Goal-mode:** {true/false}           # enables judge-loop verification
**Max retries:** {2}                  # circuit breaker threshold

### Description
Clear 1-2 sentence description of what this task accomplishes.

### Technical Requirements
- Specific technical constraints
- API endpoints, database schema, UI components
- Performance or security requirements

### Implementation Notes
- Code patterns to follow
- Libraries or frameworks to use
- Existing code to reference

### Files to Modify/Create
- `file/path.ts` — purpose
- `another/file.ts` — purpose

### Acceptance Criteria
- [ ] Specific testable outcome 1
- [ ] Specific testable outcome 2
- Tests pass

### Dependencies
Depends on: [parent-card-id]           # stays in READY until parent DONE
Blocks: [child-card-id]                # child auto-promoted when this completes
```

## Handoff Metadata Convention

When completing a card, workers leave structured metadata so the next downstream worker or human reviewer understands exactly what happened:

```python
kanban_complete(
    summary="one-line what was done",
    metadata={
        "changed_files": ["src/foo.py", "tests/test_foo.py"],
        "verification": ["pytest tests/ -q", "npm run typecheck"],
        "dependencies": None,          # or ["dep-name"] if unresolved
        "blocked_reason": None,        # or "waiting on PR review"
        "retry_notes": None,           # or "third time — flaky test skipped"
        "residual_risk": ["edge case in concurrent refresh not yet tested"],
    },
)
```

Every completion should answer: **What changed? How was it verified? What can block/retry? What risk remains?**

## Example Templates

### Database Schema Task (solo chain — first card)
```
## Create user authentication schema
Assignee: backend-dev
Priority: high
Goal-mode: false
Max retries: 2

Description: Set up database schema for user authentication with users and sessions tables.

Technical Requirements:
- PostgreSQL with UUID primary keys
- Session expiration in 7 days
- Password hashing with bcrypt

Implementation Notes:
- Use Drizzle ORM pattern from existing codebase
- Add indexes on email and session_token columns
- Reference: db/schema.ts

Files to Modify/Create:
- db/schema.ts — add users and sessions tables
- src/types/auth.ts — export User and Session types

Acceptance Criteria:
- [ ] Migration runs without errors
- [ ] TypeScript types compile correctly
- [ ] `npm run db:generate` succeeds
```

### API Endpoint Task (solo chain — depends on schema)
```
## Implement login API endpoint
Assignee: backend-dev
Priority: high
Goal-mode: false
Max retries: 2

Description: Create POST /api/login endpoint with validation and session management.

Technical Requirements:
- Validate email format and password length
- Return proper HTTP status codes (200, 400, 401)
- Create session token on success

Implementation Notes:
- Use existing middleware from src/middleware/
- Follow route pattern from src/routes/auth.ts
- Integrate with schema from dependency

Files to Modify/Create:
- src/routes/auth.ts — add login handler
- src/lib/session.ts — session creation utility

Acceptance Criteria:
- [ ] Validates email format (400 on invalid)
- [ ] Validates password length (400 on invalid)
- [ ] Creates session on valid credentials (200)
- [ ] Integration tests pass

Dependencies:
Depends on: [schema-card-id]
```

### React Component Task (solo chain — depends on API)
```
## Implement login form component
Assignee: frontend-dev
Priority: high
Goal-mode: true
Max retries: 2

Description: Create a React login form with email/password validation, loading state, and error display.

Technical Requirements:
- Form validation with Zod
- React Hook Form integration
- Error state display for API failures

Implementation Notes:
- Follow existing form patterns in src/components/forms/
- Use Tailwind for styling
- Integrate with /api/login endpoint

Files to Modify/Create:
- src/components/auth/LoginForm.tsx — form component
- src/components/auth/useLoginForm.ts — hook logic

Acceptance Criteria:
- [ ] Form validates email format client-side
- [ ] Form validates password length (>= 8 chars)
- [ ] Error messages display on failed submission
- [ ] Loading spinner during API call
- [ ] Unit tests cover validation logic

Dependencies:
Depends on: [api-card-id]
```

### Fleet Farming Task (parallel pattern)
```
## Translate marketing page to Spanish
Assignee: translator
Priority: medium
Goal-mode: true
Max retries: 1

Description: Translate the marketing landing page content from English to Spanish.

Technical Requirements:
- Preserve all Markdown formatting
- Keep HTML/Jinja2 template variables intact (`{{ var }}`)
- Use formal Latin American Spanish

Files to Modify/Create:
- content/es/marketing.md — translated page

Acceptance Criteria:
- [ ] All sections translated (header, features, pricing, FAQ)
- [ ] Template variables preserved
- [ ] No Markdown syntax broken
- [ ] Native speaker review passes
```

### Role Pipeline Task (spec → implement → review)
```
## Implement rate limiting middleware
Assignee: implementer
Priority: high
Goal-mode: false
Max retries: 3

Description: Implement token-bucket rate limiting middleware per the design spec.

Implementation Notes:
- Follow spec requirements from parent card (architect-designed)
- Use in-memory store, Redis planned for v2
- Reference: src/middleware/rate-limit.ts

Files to Modify/Create:
- src/middleware/rate-limit.ts — new middleware
- tests/test_rate_limit.py — unit tests

Acceptance Criteria:
- [ ] Token bucket algorithm implemented per spec
- [ ] Configurable rate and burst parameters
- [ ] Returns 429 when limit exceeded
- [ ] All tests pass

Dependencies:
Depends on: [spec-card-id]
```

### Testing Task (solo chain — depends on implementation)
```
## Add unit tests for auth utilities
Assignee: qa-dev
Priority: medium
Goal-mode: true
Max retries: 2

Description: Create comprehensive unit tests for authentication utility functions.

Technical Requirements:
- 90%+ code coverage for src/lib/auth.ts
- Mock database calls
- Test edge cases (expired tokens, malformed input)

Implementation Notes:
- Use Vitest pattern from existing tests/
- Follow mock patterns in tests/mocks/
- Run with `npm test`

Files to Modify/Create:
- tests/unit/auth.test.ts — new test file

Acceptance Criteria:
- [ ] All new tests pass
- [ ] Coverage report shows 90%+
- [ ] Edge cases covered (empty input, expired, invalid signature)

Dependencies:
Depends on: [impl-card-id]
```

## CLI Cheat Sheet

```bash
# Create with assignee profile
hermes kanban create "Task title" --assignee profile-name

# Create with parent dependency (chain)
hermes kanban create "Child task" --assignee dev --parent $PARENT_ID

# Create with goal-mode (judge-loop verification)
hermes kanban create "Task title" --assignee dev --goal

# Create with custom max retries
hermes kanban create "Task title" --assignee dev --max-retries 3

# Link existing cards
hermes kanban link $CHILD_ID --parent $PARENT_ID
```

# Card Templates

Standard templates for different task types. Each card follows the core structure:

## Template Structure

```markdown
## {Task Title}
**Phase:** {setup/implementation/testing/refactoring}
**Estimate:** ~{1-2} hours
**Priority:** {high/medium/low}

### Description
Clear 1-2 sentence description of what this task accomplishes.

### Technical Requirements
- Specific technical constraints
- API endpoints, database schema changes, UI components
- Performance or security requirements

### Implementation Notes
- Code patterns to follow
- Libraries or frameworks to use
- Existing code to reference

### Files to Modify/Create
- `file/path.ts` - purpose
- `another/file.ts` - purpose

### Acceptance Criteria
- [ ] Specific testable outcome 1
- [ ] Specific testable outcome 2
- Tests pass with `npm test`

### Dependencies
Blocks: [card-id-2, card-id-3]
Depends on: [card-id-1]
```

## Example Templates

### Database Schema Task
```
## Create user authentication schema
Phase: setup
Estimate: ~1 hour
Priority: high

Description: Set up database schema for user authentication with users and sessions tables.

Technical Requirements:
- PostgreSQL database with UUIDs
- Session expiration in 7 days
- Password hashing with bcrypt

Implementation Notes:
- Use Drizzle ORM pattern from existing codebase
- Add indexes on email and session_token columns
- Reference: db/schema.ts

Files to Modify/Create:
- db/schema.ts - add users and sessions tables
- src/types/auth.ts - export User and Session types

Acceptance Criteria:
- [ ] Migration runs without errors
- [ ] TypeScript types compile correctly
- [ ] `npm run db:generate` succeeds
```

### API Endpoint Task
```
## Implement login API endpoint
Phase: implementation
Estimate: ~2 hours
Priority: high
Depends on: [schema-card-id]

Description: Create POST /api/login endpoint with validation and session management.

Technical Requirements:
- Validate email format and password length
- Return proper HTTP status codes
- Create session token on success

Implementation Notes:
- Use existing middleware from src/middleware/
- Follow route pattern from src/routes/auth.ts
- Integrate with schema from dependency

Files to Modify/Create:
- src/routes/auth.ts - add login handler
- src/lib/session.ts - session creation utility

Acceptance Criteria:
- [ ] Validates email format (400 on invalid)
- [ ] Validates password length (400 on invalid)
- [ ] Creates session on valid credentials (200)
- [ ] Integration tests pass
```

### React Component Task
```
## Implement login form component
Phase: implementation
Estimate: ~2 hours
Priority: high
Depends on: [api-card-id]

Description: Create a React login form with email/password validation and error handling.

Technical Requirements:
- Form validation with Zod
- React Hook Form integration
- Error state display

Implementation Notes:
- Follow existing form patterns in src/components/forms/
- Use Tailwind for styling
- Integrate with /api/login endpoint

Files to Modify/Create:
- src/components/auth/LoginForm.tsx - form component
- src/components/auth/useLoginForm.ts - hook logic

Acceptance Criteria:
- [ ] Form validates email format
- [ ] Form validates password length (>= 8 chars)
- [ ] Error messages display on failed submission
- [ ] Unit tests cover validation logic
```

### Testing Task
```
## Add unit tests for auth utilities
Phase: testing
Estimate: ~1 hour
Priority: medium

Description: Create comprehensive unit tests for authentication utility functions.

Technical Requirements:
- 90% code coverage for src/lib/auth.ts
- Mock database calls
- Test edge cases

Implementation Notes:
- Use Vitest pattern from existing tests/
- Mock following tests/mocks/ pattern
- Run with `npm test`

Files to Modify/Create:
- tests/unit/auth.test.ts - new test file

Acceptance Criteria:
- [ ] All new tests pass
- [ ] Coverage report shows 90%+
- [ ] Tests remain in tests/ directory
```
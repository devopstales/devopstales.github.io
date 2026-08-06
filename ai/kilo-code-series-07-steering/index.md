---
title: Kilo Code: Steering and Custom Agents
url: https://devopstales.github.io/ai/kilo-code-series-07-steering/
date: 2026-03-31
---


Structure is great, but every team has its own "tribal knowledge"—naming conventions, preferred libraries, and security requirements that aren't always obvious to a general LLM.

<!--more-->

{{< content "/filedir/kilo-code-series.html" >}}

In Kilo Code, you provide this context through **Steering** and **Custom Agents**.

## What is Steering?

**Steering** is the practice of providing persistent instructions to Kilo Code about how it should behave. Instead of repeating "use TypeScript, not JavaScript" in every conversation, you define it once in a steering file.

### Without Steering

```bash
User: "Create a user service"
AI: *generates JavaScript code*
User: "Actually, we use TypeScript"
AI: *regenerates in TypeScript*
User: "And we use functional components, not classes"
AI: *regenerates again*
User: "Also, we use Zod for validation"
AI: *regenerates again*
```

### With Steering

```bash
User: "Create a user service"
AI: *generates TypeScript code with functional components and Zod validation*
```

The difference? Steering files that encode your team's preferences.

---

## Steering File Structure

Steering files live in `.kilocode/rules/` and are written in Markdown:

```bash
.kilocode/
└── rules/
    ├── coding-standards.md
    ├── security.md
    ├── testing.md
    ├── documentation.md
    └── architecture.md
```

Each file contains instructions for the AI:

```markdown
# Coding Standards

## Language
- Always use TypeScript for new code
- Use ES2022+ features (optional chaining, nullish coalescing)
- Avoid `any` type; use `unknown` or specific types

## Code Style
- Use functional components over classes
- Prefer composition over inheritance
- Use async/await, not Promise chains
- Name boolean variables with `is`, `has`, `should` prefixes

## Libraries
- Use Zod for input validation
- Use date-fns for date manipulation (not moment.js)
- Use axios for HTTP requests
- Use jest for testing
```

---

## Creating Your First Steering File

### Step 1: Create the Rules Directory

```bash
mkdir -p .kilocode/rules
```

### Step 2: Create a Steering File

```bash
touch .kilocode/rules/coding-standards.md
```

### Step 3: Add Your Rules

`````markdown
# Coding Standards

## Overview
This document defines the coding standards for our team. All AI-generated code must follow these guidelines.

## TypeScript Rules

### Always Use TypeScript
- All new code must be written in TypeScript (.ts, .tsx)
- Never generate JavaScript (.js, .jsx) files
- Use strict mode in tsconfig.json

### Type Safety
- Avoid `any` type - use `unknown` or specific types
- Define interfaces for all objects
- Use type guards for runtime type checking
- Never use `as Type` assertions without validation

### Example - Good vs Bad

**Bad:**
```typescript
function getUser(id: any): any {
  return fetch(`/api/users/${id}`);
}
```

**Good:**
```typescript
interface User {
  id: string;
  name: string;
  email: string;
}

async function getUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) throw new Error('User not found');
  return response.json();
}
```

## Code Organization

### File Structure
- Group by feature, not by type
- Keep related files together
- Max 300 lines per file

### Naming Conventions
- Files: kebab-case (user-service.ts)
- Classes: PascalCase (UserService)
- Functions: camelCase (getUserById)
- Constants: UPPER_SNAKE_CASE (MAX_RETRY_COUNT)
- Types/Interfaces: PascalCase with descriptive names

## Error Handling

### Always Handle Errors
- Wrap async operations in try-catch
- Log errors with context
- Return user-friendly messages
- Never swallow errors silently

### Error Types
- Use custom error classes for domain errors
- Include error codes for programmatic handling
- Add stack traces for debugging

## Dependencies

### Approved Libraries
| Purpose | Library | Version |
|---------|---------|---------|
| HTTP Client | axios | ^1.6.0 |
| Validation | zod | ^3.22.0 |
| Date Handling | date-fns | ^2.30.0 |
| Testing | jest | ^29.7.0 |
| Mocking | msw | ^2.0.0 |

### Forbidden Libraries
- moment.js (use date-fns)
- lodash (use native methods or es-toolkit)
- request (use axios or fetch)
`````

### Step 4: Enable Steering

Kilo Code automatically loads files from `.kilocode/rules/`. Verify:

```bash
kilo-code rules list

# Output:
Available steering rules:
  ✓ coding-standards.md
```

---

## Steering Rule Categories

### 1. Coding Standards

Define how code should be written:

```markdown
# Coding Standards

## React Components
- Use functional components with hooks
- Extract logic into custom hooks
- Use TypeScript interfaces for props
- Implement proper error boundaries

## State Management
- Use Zustand for global state
- Keep state as close to usage as possible
- Never mutate state directly

## Styling
- Use Tailwind CSS for styling
- Follow mobile-first approach
- Use CSS variables for theme colors
```

### 2. Security Rules

Define security requirements:

```markdown
# Security Rules

## Authentication
- Never store passwords in plain text
- Use bcrypt with 12 salt rounds
- Implement JWT with short expiration (15min)
- Use refresh tokens for long sessions

## Input Validation
- Validate ALL user input with Zod
- Sanitize HTML output
- Use parameterized SQL queries
- Implement CSRF protection

## Secrets Management
- Never commit .env files
- Use environment variables for secrets
- Rotate API keys every 90 days
- Use AWS Secrets Manager for production

## Common Vulnerabilities to Prevent
- SQL Injection: Use ORM or parameterized queries
- XSS: Escape all user output
- CSRF: Include CSRF tokens in forms
- IDOR: Always check resource ownership
```

### 3. Testing Rules

Define testing requirements:

`````markdown
# Testing Standards

## Coverage Requirements
- Minimum 80% code coverage
- 100% coverage for critical paths
- All public functions must have tests

## Test Structure
- Use describe/it/test pattern
- Arrange-Act-Assert pattern
- One assertion per test (when possible)
- Descriptive test names

## Test Naming Convention
```typescript
// Format: should [expected behavior] when [condition]
describe('UserService', () => {
  describe('createUser', () => {
    it('should create user with valid data', async () => {});
    it('should throw error when email exists', async () => {});
    it('should hash password before saving', async () => {});
  });
});
```

## Mocking Guidelines
- Mock external services (APIs, databases)
- Use MSW for API mocking
- Never mock business logic
- Reset mocks between tests
`````

### 4. Documentation Rules

Define documentation standards:

`````markdown
# Documentation Standards

## Code Comments
- Comment WHY, not WHAT
- Add JSDoc for all public functions
- Include examples for complex functions
- Document edge cases

## JSDoc Template
```typescript
/**
 * Creates a new user account.
 * 
 * @param data - User registration data
 * @param data.email - User's email address
 * @param data.password - User's password (min 8 chars)
 * @param data.name - User's display name
 * @returns The created user object
 * @throws {ValidationError} If input validation fails
 * @throws {ConflictError} If email already exists
 * 
 * @example
 * ```typescript
 * const user = await createUser({
 *   email: 'user@example.com',
 *   password: 'securePassword123',
 *   name: 'John Doe'
 * });
 * ```
 */
```

## README Requirements
- Installation instructions
- Usage examples
- API documentation
- Contributing guidelines
`````

### 5. Architecture Rules

Define architectural patterns:

```markdown
# Architecture Standards

## Layered Architecture
- Controllers: Handle HTTP requests
- Services: Business logic
- Repositories: Data access
- Models: Data structures

## Dependency Rules
- Controllers → Services → Repositories
- Never skip layers
- No circular dependencies

## API Design
- RESTful endpoints
- Version all APIs (/api/v1/)
- Use proper HTTP status codes
- Return consistent error format

## Database
- Use migrations for schema changes
- Never modify production directly
- Index foreign keys
- Use UUIDs for primary keys
```

---

## Custom Agents

While steering rules apply globally, **Custom Agents** are specialized personas for specific tasks.

### Creating a Custom Agent

Create `.kilocode/agents/security-reviewer.md`:

`````markdown
# Agent: Security Reviewer

## Role
You are a security expert who reviews code for vulnerabilities and compliance with security best practices.

## Expertise
- OWASP Top 10 vulnerabilities
- Authentication and authorization
- Data protection and encryption
- Secure API design
- Security compliance (GDPR, SOC2)

## Behavior
1. Analyze code for security issues
2. Reference specific OWASP categories
3. Provide severity ratings (Critical/High/Medium/Low)
4. Suggest specific fixes with code examples
5. Never modify code directly - only review

## Output Format
```
## Security Review: [File/Feature]

### Issues Found

#### [Issue Name] - [Severity]
**Location:** [File:Line]
**OWASP:** [Category]
**Description:** [What's wrong]
**Impact:** [Potential damage]
**Fix:** [Code example]

### Summary
- Critical: X
- High: Y
- Medium: Z
- Low: W
```

## Restrictions
- Do not modify files
- Do not suggest non-security changes
- Always cite OWASP categories
`````

### Using Custom Agents

```bash
# CLI
kilo-code --agent security-reviewer "Review the authentication code"

# In IDE
/agent security-reviewer
/review src/auth/ for security issues
```

### Example Session

`````markdown
User: /agent security-reviewer
User: Review src/auth/login.ts for security issues

Security Reviewer:
## Security Review: src/auth/login.ts

### Issues Found

#### Hardcoded Secret Key - CRITICAL
**Location:** src/auth/login.ts:15
**OWASP:** A07:2021 - Identification and Authentication Failures
**Description:** JWT secret is hardcoded in the source code
**Impact:** If code is leaked, attackers can forge tokens
**Fix:**
```typescript
// Bad
const JWT_SECRET = 'my-super-secret-key-123';

// Good
const JWT_SECRET = process.env.JWT_SECRET;
// Add to .env.example: JWT_SECRET=<generate-secure-random-string>
```

#### Missing Rate Limiting - HIGH
**Location:** src/auth/login.ts:25
**OWASP:** A07:2021 - Identification and Authentication Failures
**Description:** No rate limiting on login endpoint
**Impact:** Enables brute force attacks
**Fix:**
```typescript
import rateLimit from 'express-rate-limit';

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 5, // 5 attempts per window
  message: 'Too many login attempts'
});

app.post('/login', loginLimiter, loginHandler);
```

#### Password Not Validated - MEDIUM
**Location:** src/auth/login.ts:30
**OWASP:** A07:2021 - Identification and Authentication Failures
**Description:** No minimum password requirements enforced
**Impact:** Users may choose weak passwords
**Fix:**
```typescript
import { z } from 'zod';

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain uppercase letter')
    .regex(/[0-9]/, 'Password must contain a number')
});
```

### Summary
- Critical: 1
- High: 1
- Medium: 1
- Low: 0
`````

---

## More Custom Agent Examples

### Code Reviewer Agent

`.kilocode/agents/code-reviewer.md`:

```markdown
# Agent: Code Reviewer

## Role
You are a senior developer who conducts thorough code reviews.

## Focus Areas
- Code quality and readability
- Best practices adherence
- Performance considerations
- Error handling
- Test coverage

## Review Checklist
- [ ] Code follows team standards
- [ ] Functions are small and focused
- [ ] Error handling is comprehensive
- [ ] Tests cover edge cases
- [ ] No console.log in production code
- [ ] Proper logging implemented
- [ ] No TODO comments without tickets

## Output Format
Provide feedback in this order:
1. Summary (what the code does)
2. Positive observations
3. Required changes (blocking)
4. Suggested improvements (non-blocking)
5. Questions for the author
```

### Documentation Agent

`.kilocode/agents/tech-writer.md`:

`````markdown
# Agent: Technical Writer

## Role
You are a technical writer who creates clear, comprehensive documentation.

## Expertise
- API documentation
- README files
- Architecture decision records (ADRs)
- User guides
- Onboarding documentation

## Writing Style
- Clear and concise
- Active voice
- Include examples
- Assume reader is intelligent but unfamiliar with codebase

## Documentation Templates

### API Endpoint
```markdown
## [Method] [Endpoint]

[Brief description]

### Request
[Headers, body schema]

### Response
[Success and error responses]

### Example
[Code example]
```

### README Section

```markdown
## [Feature Name]

### Overview
[What it does]

### Usage
[How to use it]

### Configuration
[Options and settings]
```

`````

### Performance Agent

`.kilocode/agents/performance-expert.md`:

```markdown
# Agent: Performance Expert

## Role
You are a performance engineer who optimizes application speed and efficiency.

## Expertise
- Database query optimization
- Caching strategies
- Bundle size reduction
- Memory management
- Network optimization

## Analysis Approach
1. Identify bottlenecks
2. Measure current performance
3. Suggest optimizations
4. Provide before/after comparisons

## Common Optimizations
- Add database indexes
- Implement caching (Redis)
- Lazy loading
- Code splitting
- Query optimization (N+1 fixes)
- Connection pooling
```

---

## Agent Configuration

### Global Agent Settings

`.kilocode/agents.json`:

```json
{
  "defaultAgent": "orchestrator",
  "agents": {
    "security-reviewer": {
      "model": "qwen-coder-plus",
      "temperature": 0.1,
      "maxTokens": 4096
    },
    "code-reviewer": {
      "model": "qwen-coder-32b",
      "temperature": 0.3,
      "maxTokens": 8192
    },
    "tech-writer": {
      "model": "qwen-coder-32b",
      "temperature": 0.7,
      "maxTokens": 4096
    }
  },
  "restrictions": {
    "security-reviewer": {
      "canModifyFiles": false,
      "requiresApproval": true
    }
  }
}
```

### Agent-Specific Steering

Agents can have their own steering files:

```bash
.kilocode/
├── rules/
│   └── general.md
└── agents/
    ├── security-reviewer.md
    ├── security-reviewer.rules.md  # Agent-specific rules
    └── code-reviewer.md
```

---

## Best Practices

### 1. Keep Rules Concise

```bash
Bad:  A 500-word paragraph about naming conventions

Good: Bullet points with clear examples
```

### 2. Use Examples

```markdown
Bad:
"Write good commit messages"

Good:
"Write commit messages in imperative mood:
- Good: 'Add user authentication'
- Bad: 'Added user authentication'
- Bad: 'Adding user authentication'"
```

### 3. Organize by Category

```bash
.kilocode/rules/
├── 01-coding-standards.md
├── 02-security.md
├── 03-testing.md
├── 04-documentation.md
└── 05-architecture.md
```

### 4. Version Your Rules

```markdown
# Coding Standards

## Version: 2.1
## Last Updated: 2026-03-31
## Owner: Engineering Team

## Changelog
- v2.1 (2026-03-31): Added React hooks guidelines
- v2.0 (2026-01-15): Migrated to TypeScript standards
- v1.0 (2025-06-01): Initial version
```

### 5. Review and Update

Schedule quarterly reviews:

```bash
# Add to team calendar
# Quarterly: Review and update .kilocode/rules/
```

### 6. Test Your Rules

Verify rules are being followed:

```bash
User: "Create a user service"

Expected: TypeScript code following all rules
Actual: Check if output matches standards

If not matching, refine the rules.
```

---

## Team Onboarding

### New Developer Setup

```bash
# Clone repository
git clone git@github.com:company/project.git

# Kilo Code automatically loads rules
cd project
kilo-code "Explain our coding standards"

# Developer gets instant guidance on team conventions
```

### Rule Discovery

```bash
# List all rules
kilo-code rules list

# Show specific rule
kilo-code rules show security

# Check rule compliance
kilo-code rules check src/
```

---

## Troubleshooting

### Issue: Rules Not Being Followed

**Check:**
1. Rules file exists in `.kilocode/rules/`
2. File is named with `.md` extension
3. Rules are clear and specific
4. No conflicting rules

**Fix:**
```bash
# Verify rules are loaded
kilo-code rules list

# Test with explicit rule reference
kilo-code "Following our coding standards, create a user service"
```

### Issue: Too Many Rules Confusing AI

**Symptoms:** AI produces inconsistent output

**Solution:** Prioritize rules:

```markdown
# CRITICAL (Must Follow)
- Always use TypeScript
- Never commit secrets

# IMPORTANT (Should Follow)
- Use functional components
- Write tests for all functions

# PREFERRED (Nice to Have)
- Order imports alphabetically
- Use specific emoji in commits
```

### Issue: Rules File Too Large

**Solution:** Split into categories:

```bash
Before:
.kilocode/rules/everything.md (500 lines)

After:
.kilocode/rules/coding-standards.md
.kilocode/rules/security.md
.kilocode/rules/testing.md
.kilocode/rules/documentation.md
```


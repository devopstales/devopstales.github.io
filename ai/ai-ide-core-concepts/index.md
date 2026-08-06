---
title: AI IDE Core Concepts: Agents, Commands, Skills, Rules, Context, and Workflows
url: https://devopstales.github.io/ai/ai-ide-core-concepts/
date: 2026-03-22
---


Modern AI-powered IDEs have evolved far beyond simple code completion. Today's tools like **Kilo Code**, **Cursor**, **Windsurf**, and **Google Antigravity** introduce new paradigms: autonomous agents, reusable skills, markdown rules, semantic context, and structured workflows.

<!--more-->

This guide explains these core concepts and shows you how to leverage them effectively in your daily development work.

---

## What Are AI Agents?

**Agents** are autonomous AI entities that can plan, execute, and complete tasks with minimal human intervention. Unlike traditional chatbots that only respond to queries, agents can:

- Read and write files across your codebase
- Run terminal commands and tests
- Debug errors and apply fixes
- Coordinate with other agents in parallel

### Agent Architecture

```bash
┌─────────────────┐
│    Developer    │
│      (User)     │
└────────┬────────┘
         │ Assigns Task
         ▼
┌─────────────────────────────────────────────────────────┐
│                   Agent Manager                         │
└─────────────┬─────────────────────────────────┬─────────┘
              │                                 │
    ┌─────────┼─────────┐              Spawns   │
    │         │         │                       │
    ▼         ▼         ▼                       │
┌────────┐ ┌────────┐ ┌────────┐                │
│ Agent 1│ │ Agent 2│ │ Agent 3│ <──────────────┘
└───┬────┘ └───┬────┘ └───┬────┘
    │ Uses     │ Uses     │ Uses
    │          │          │
    └──────────┼──────────┘
               │
               ▼
    ┌──────────────────────┐
    │       Tools          │
    └──────────┬───────────┘
               │
    ┌──────────┼───────────┬──────────────┐
    │          │           │              │
    ▼          ▼           ▼              ▼
┌────────┐ ┌──────────┐ ┌─────────┐ ┌─────────┐
│File    │ │ Terminal │ │ Browser │ │  Git    │
│System  │ │          │ │         │ │         │
└────────┘ └──────────┘ └─────────┘ └─────────┘
```

### Types of Agents

| Agent Type | Purpose | Example Use Case |
|------------|---------|------------------|
| **Code Agent** | Write and refactor code | Implement a new API endpoint |
| **Spec Agent** | Create and validate specifications | Define API contract before implementation |
| **Debug Agent** | Analyze and fix errors | Investigate failing tests |
| **Test Agent** | Write and run tests | Generate unit tests for new feature |
| **Review Agent** | Code review and quality checks | Review PR for security issues |
| **Browser Agent** | Web interaction and testing | Test login flow in staging environment |

### How to Use Agents

#### In Kilo Code

```bash
# Start an agent with a specific task
kiro agent start "Build user authentication API"

# Run multiple agents in parallel
kiro agent-manager start "Build full-stack feature"
```

#### In Google Antigravity

1. Open Agent Manager (`Cmd + E`)
2. Click "New Task" in the Inbox
3. Describe your task in natural language
4. Select autonomy level (Agent-assisted recommended)
5. Agent executes and presents artifacts for review

#### In Windsurf

```bash
Use Flow by describing your task:
"Fix the failing tests in the auth module"

Flow will:
1. Run tests
2. Analyze failures
3. Examine code
4. Apply fixes
5. Re-run tests
```

### Best Practices for Working with Agents

1. **Be specific in task descriptions**: "Create a REST API endpoint for user login with JWT authentication" is better than "Add login"

2. **Set clear acceptance criteria**: Include expected inputs, outputs, and edge cases

3. **Review artifacts before accepting**: Check code diffs, test results, and screenshots

4. **Use appropriate autonomy levels**: Start with Review-driven, move to Agent-assisted as you build trust

5. **Monitor parallel agents**: Use the Agent Manager dashboard to track progress

---

## Commands: Direct AI Instructions

**Commands** are explicit instructions that trigger specific AI behaviors. They're typically prefixed with `/` or invoked via keyboard shortcuts.

### Common Command Types

| Command | Description | Example |
|---------|-------------|---------|
| `/spec` | Create a specification | `/spec "Build task management API"` |
| `/edit` | Edit selected code | Highlight code, then `/edit "Optimize this"` |
| `/test` | Generate tests | `/test "Add unit tests for this function"` |
| `/explain` | Explain code | `/explain "How does this algorithm work?"` |
| `/refactor` | Refactor code | `/refactor "Extract this into a reusable module"` |
| `/debug` | Debug an issue | `/debug "Why is this test failing?"` |
| `/skill` | Load specialized expertise | `/skill docker-expert` |

### Command Examples in Practice

#### Kilo Code Commands

```bash
/spec "Build a task management API"
→ Creates detailed specification with endpoints, data models, and validation rules

/skill docker-expert
→ Loads Docker expertise for containerization tasks

/mode architect
→ Switches to architecture planning mode
```

#### Cursor Commands

```bash
Cmd + K → Open composer for multi-file edits
@codebase → Search entire codebase for context
@file → Reference specific file
@docs → Reference documentation
```

#### VS Code + Copilot Commands

```
Ctrl + I → Open Copilot inline chat
Ctrl + Shift + I → Open Copilot chat panel
/copilot → Trigger specific Copilot features
```

### Creating Custom Commands

Some IDEs allow you to define custom commands. In Kilo Code, you can create command shortcuts in your configuration:

```json
// .kiro/commands.json
{
  "commands": {
    "deploy": {
      "description": "Deploy to staging",
      "steps": [
        "Run tests",
        "Build Docker image",
        "Push to registry",
        "Update Kubernetes deployment"
      ]
    }
  }
}
```

---

## Skills: On-Demand Expertise

**Skills** are modular packages of expertise that you can load into your AI agent to enhance its capabilities for specific domains or tasks.

### Why Skills Matter

Without skills, AI agents have general knowledge but may lack:
- Deep expertise in your tech stack
- Knowledge of your team's conventions
- Understanding of domain-specific patterns
- Access to internal documentation

Skills bridge this gap by providing focused, contextual expertise.

### Skill Structure

A typical skill includes:

```bash
┌─────────────────────────────────────────┐
│           Skill Package                 │
├─────────────────────────────────────────┤
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Metadata                          │  │
│  │ • Name, Version, Description      │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Rules                             │  │
│  │ • Coding standards, Patterns      │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Tools                             │  │
│  │ • Commands, APIs, Scripts         │  │
│  └───────────────────────────────────┘  │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │ Examples                          │  │
│  │ • Code samples, Templates         │  │
│  └───────────────────────────────────┘  │
│                                         │
└─────────────────────────────────────────┘
```

### Built-in Skills Examples

| Skill | Purpose | When to Use |
|-------|---------|-------------|
| `docker-expert` | Containerization expertise | Building Dockerfiles, debugging containers |
| `kubernetes-guru` | K8s manifests and debugging | Deploying to Kubernetes |
| `security-auditor` | Security review | Pre-deployment security checks |
| `performance-optimizer` | Code optimization | Improving application performance |
| `api-designer` | REST/GraphQL API design | Creating new API endpoints |
| `test-engineer` | Test generation | Writing comprehensive test suites |

### Creating Custom Skills

#### Kilo Code Skill Structure

```bash
skills/
├── docker-expert/
│   ├── skill.md          # Skill definition
│   ├── rules/
│   │   ├── dockerfile.md
│   │   └── compose.md
│   ├── examples/
│   │   ├── basic.Dockerfile
│   │   └── multi-stage.Dockerfile
│   └── tools/
│       └── lint.sh
```

#### Example: Docker Expert Skill

```markdown
# Docker Expert Skill

## Description
Provides expertise in Docker containerization, multi-stage builds, and security best practices.

## Rules
1. Always use multi-stage builds for production images
2. Never run containers as root
3. Use specific base image versions (no `latest`)
4. Minimize layers by combining RUN commands
5. Use `.dockerignore` to exclude unnecessary files

## Patterns

### Multi-stage Build Template

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/index.js"]
```

## Tools
- `docker lint`: Validate Dockerfile
- `docker scan`: Check for vulnerabilities
- `docker buildx`: Multi-architecture builds
```

### Loading Skills

#### In Kilo Code

```bash
# Load a skill
/skill docker-expert

# List available skills
/skill list

# Unload a skill
/skill unload docker-expert
```

#### In Cursor

Skills are implicitly loaded through context:
- Add skill documentation to your codebase
- Reference with `@skills/docker-expert.md`

#### In Windsurf

Skills are part of the Flow configuration:
```json
{
  "flow": {
    "skills": ["docker", "kubernetes", "security"]
  }
}
```

### Best Practices for Skills

1. **Keep skills focused**: One domain per skill
2. **Include examples**: Show, don't just tell
3. **Update regularly**: Keep skills current with best practices
4. **Share with team**: Store skills in version control
5. **Combine skills**: Load multiple skills for complex tasks

---

## Rules: Guiding AI Behavior

**Rules** are constraints and guidelines that shape how AI agents behave, write code, and make decisions. They ensure consistency with your team's standards.

### Rule Types

| Rule Type | Scope | Example |
|-----------|-------|---------|
| **Global Rules** | Entire project | "Always use TypeScript" |
| **File Rules** | Specific files/patterns | "*.test.ts: Use describe/it pattern" |
| **Language Rules** | Specific languages | "Python: Follow PEP 8" |
| **Framework Rules** | Specific frameworks | "React: Use functional components with hooks" |
| **Security Rules** | Security requirements | "Never commit API keys or secrets" |

### Rule Configuration

#### Kilo Code Rules

Rules are defined in `.kiro/rules.md` or `.kilocode/config.json`:

```markdown
# Project Rules

## Code Style
- Use TypeScript for all new code
- Follow ESLint configuration
- Maximum function length: 50 lines
- Require JSDoc for public functions

## Architecture
- Use repository pattern for data access
- All API endpoints must have validation
- Implement error handling middleware

## Testing
- Minimum 80% code coverage
- Unit tests for all business logic
- Integration tests for API endpoints

## Security
- No hardcoded credentials
- Use environment variables for config
- Validate all user inputs
```

#### Cursor Rules

Cursor uses `.cursor/rules.md`:

```markdown
# Cursor Rules

## Always
- Write tests for new features
- Use meaningful variable names
- Add comments for complex logic

## Never
- Don't use `any` type in TypeScript
- Don't commit console.log statements
- Don't use deprecated APIs

## Prefer
- Prefer composition over inheritance
- Prefer async/await over promises
- Prefer functional components
```

#### VS Code + Copilot

Copilot respects editorconfig and can be guided with comments:

```typescript
// RULE: Always validate input before processing
// RULE: Use dependency injection for services
function createUser(userData: CreateUserDto) {
  // Implementation follows rules above
}
```

### Rule Inheritance

```bash
┌─────────────────────────────────┐
│     Global Rules                │
│     ~/.ai-rules.md              │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│     Project Rules               │
│     .kiro/rules.md              │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│     Language Rules              │
│     .kiro/rules/typescript.md   │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│     File Rules                  │
│     Inline comments             │
└─────────────────────────────────┘
```

### Best Practices for Rules

1. **Start minimal**: Begin with 5-10 essential rules
2. **Be specific**: "Use 2-space indentation" vs "Format code nicely"
3. **Make rules actionable**: Rules should be enforceable
4. **Review regularly**: Update rules as standards evolve
5. **Automate enforcement**: Use linters and formatters alongside AI rules

---

## Context: What the AI Knows

**Context** is the information the AI has access to when generating responses. Better context = better responses.

### Context Types

| Context Type | Description | How to Provide |
|--------------|-------------|----------------|
| **File Context** | Current open file | Automatically included |
| **Selection Context** | Highlighted code | Select before asking |
| **Tab Context** | All open tabs | Keep relevant files open |
| **Codebase Context** | Entire project | Via indexing/embeddings |
| **Documentation Context** | External docs | Reference with @docs |
| **Conversation Context** | Chat history | Maintained automatically |
| **Terminal Context** | Recent output | Integrated in some IDEs |

### Context Mechanisms

#### 1. File and Selection Context

The most basic context. Simply:
- Open relevant files
- Select code you're asking about

#### 2. Codebase Indexing

Advanced IDEs index your entire codebase for semantic search:

```bash
┌──────────────────┐
│  Your Codebase   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Code Indexer    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Vector Embeddings│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│   Vector DB      │
│ (Qdrant/Chroma)  │
└────────┬─────────┘
         ▲
         │
┌────────┴─────────┐
│  Query Embedder  │
└────────┬─────────┘
         │
┌────────┴─────────┐
│    User Query    │
└──────────────────┘

         │
         ▼
┌──────────────────┐
│  Relevant Code   │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│    AI Agent      │
└──────────────────┘
```

**Setup in Kilo Code:**

```json
{
  "indexing": {
    "enabled": true,
    "provider": "qdrant",
    "embeddings": "nomic-embed-text",
    "maxContextTokens": 100000
  }
}
```

**Setup in Cursor:**

```json
{
  "cursor.indexing.enabled": true,
  "cursor.indexing.exclude": ["node_modules", "dist", ".git"]
}
```

#### 3. Explicit Context References

Reference specific files or documentation:

```
@src/auth/jwt.ts How do I add refresh tokens here?

@docs/react-hooks What's the best pattern for custom hooks?

@tests Compare with existing test patterns
```

### Context Optimization

#### What to Include

✅ Current file being edited
✅ Related utility functions
✅ Type definitions
✅ Recent error messages
✅ Relevant documentation

#### What to Exclude

❌ Entire node_modules
❌ Build artifacts
❌ Unrelated large files
❌ Sensitive configuration

### Context Window Management

| IDE | Context Window | Optimization |
|-----|----------------|--------------|
| Kilo Code | Up to 1M tokens (with Qwen) | Semantic retrieval |
| Cursor | ~100K tokens | Smart chunking |
| Windsurf | ~100K tokens | Cascade context |
| Antigravity | ~200K tokens | Agent-specific context |
| VS Code + Copilot | ~10K tokens | File-based only |

### Best Practices for Context

1. **Keep relevant files open**: Tab context is free context
2. **Use @ references**: Be explicit about what to include
3. **Clean your workspace**: Close unrelated files
4. **Index your codebase**: Enable semantic search
5. **Reference errors**: Include full error messages

---

## Workflows: Structured AI Development

**Workflows** are structured sequences of AI-assisted steps to complete complex tasks. They transform ad-hoc AI usage into repeatable processes.

### Common Workflow Patterns

#### 1. Spec-Driven Development

```bash
┌───────────────────────┐
│ Define Requirements   │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ AI Creates            │
│ Specification         │
└───────────┬───────────┘
            │
            ▼
      ┌───────────┐
      │Review Spec│
      └─────┬─────┘
            │
    ┌───────┴───────┐
    │ Revise        │ Approve
    │ (back to      │
    │  spec)        │
    │               ▼
    │      ┌───────────────────────┐
    │      │ AI Implements Code    │
    │      └───────────┬───────────┘
    │                  │
    │                  ▼
    │      ┌───────────────────────┐
    │      │ AI Writes Tests       │
    │      └───────────┬───────────┘
    │                  │
    │                  ▼
    │           ┌───────────┐
    │           │Run Tests  │
    │           └─────┬─────┘
    │                 │
    │         ┌───────┴───────┐
    │         │ Fail          │ Pass
    │         │               │
    │         ▼               ▼
    │ ┌─────────────┐ ┌─────────────┐
    │ │ AI Debugs   │ │    Merge    │
    │ └──────┬──────┘ └─────────────┘
    │        │
    └────────┘
```

**Example in Kilo Code:**

```bash
# Step 1: Create specification
/spec "Build user authentication API with JWT"

# Step 2: Review and approve spec
# (AI presents spec for review)

# Step 3: Implement
/implement

# Step 4: Test
/test

# Step 5: Review and merge
```

#### 2. Test-Driven Development with AI

```bash
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│   Describe      │──>│  AI Writes      │──>│ Run Test        │──>│ AI Implements   │
│   Feature       │   │  Test           │   │ - Fails         │   │ Code            │
└─────────────────┘   └─────────────────┘   └─────────────────┘   └────────┬────────┘
                                                                           │
                                                                           ▼
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐           │
│    Commit       │<──│  AI Refactors   │<──│ Run Test        │<──────────┘
└─────────────────┘   └─────────────────┘   │ - Passes        │
                                            └─────────────────┘
```

**Example:**

```bash
User: "I need a function that validates email addresses"

AI: I'll write a test first:

```typescript
describe('validateEmail', () => {
  it('should accept valid emails', () => {
    expect(validateEmail('test@example.com')).toBe(true);
  });
  
  it('should reject invalid emails', () => {
    expect(validateEmail('invalid')).toBe(false);
  });
});
```

Run test? [Yes/No]
```

#### 3. Refactoring Workflow

```bash
┌───────────────────────┐
│ Identify Code Smell   │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│    AI Analyzes        │
└───────────┬───────────┘
            │
            ▼
┌───────────────────────┐
│ AI Proposes           │
│ Refactoring           │
└───────────┬───────────┘
            │
            ▼
      ┌───────────┐
      │Review Plan│
      └─────┬─────┘
            │
    ┌───────┴───────┐
    │ Revise        │ Approve
    │ (back to      │
    │  proposal)    │
    │               ▼
    │      ┌───────────────────────┐
    │      │ AI Creates Branch     │
    │      └───────────┬───────────┘
    │                  │
    │                  ▼
    │      ┌───────────────────────┐
    │      │ AI Applies Changes    │
    │      └───────────┬───────────┘
    │                  │
    │                  ▼
    │           ┌───────────┐
    │           │Run Tests  │
    │           └─────┬─────┘
    │                 │
    │         ┌───────┴───────┐
    │         │ Fail          │ Pass
    │         │               │
    │         ▼               ▼
    │ ┌─────────────┐ ┌─────────────┐
    │ │ AI Fixes    │ │ Create PR   │
    │ └──────┬──────┘ └─────────────┘
    │        │
    └────────┘
```

#### 4. Debugging Workflow

```bash
1. AI reads error message
2. AI analyzes stack trace
3. AI searches codebase for related code
4. AI identifies potential causes
5. AI proposes fixes
6. AI applies fix and re-runs
7. AI documents the issue
```

### Workflow Configuration

#### Kilo Code Modes

Kilo Code uses modes to structure workflows:

```json
{
  "modes": {
    "architect": {
      "description": "Plan and design before coding",
      "tools": ["spec", "diagram"],
      "autoApprove": false
    },
    "code": {
      "description": "Write and refactor code",
      "tools": ["edit", "create", "delete"],
      "autoApprove": false
    },
    "debug": {
      "description": "Investigate and fix issues",
      "tools": ["read", "run", "test"],
      "autoApprove": true
    },
    "test": {
      "description": "Write and run tests",
      "tools": ["test", "run"],
      "autoApprove": false
    }
  }
}
```

#### Antigravity Planning Modes

| Mode | Behavior |
|------|----------|
| **Planning** | Deep analysis, multi-step plans |
| **Fast** | Quick responses for simple tasks |
| **Agent-driven** | High autonomy |
| **Agent-assisted** | Balanced (recommended) |
| **Review-driven** | Always asks for approval |

### Creating Custom Workflows

#### Example: API Development Workflow

```markdown
# API Development Workflow

## Trigger
When creating new API endpoints

## Steps

1. **Specification**
   - Define endpoint URL and method
   - Define request/response schema
   - Define authentication requirements

2. **Implementation**
   - Create route handler
   - Add validation middleware
   - Implement business logic
   - Add error handling

3. **Testing**
   - Unit tests for business logic
   - Integration tests for endpoint
   - Load test for performance

4. **Documentation**
   - Update OpenAPI spec
   - Add usage examples
   - Document error cases

5. **Review**
   - Security review
   - Performance review
   - Code style check
```

### Workflow Best Practices

1. **Document your workflows**: Create markdown guides for common tasks
2. **Automate repetitive steps**: Use scripts and custom commands
3. **Review at key checkpoints**: Spec approval, code review, test results
4. **Iterate and improve**: Refine workflows based on experience
5. **Share with team**: Standardize workflows across your organization

---

## Putting It All Together: A Complete Example

Let's build a feature using all concepts:

### Scenario: Add User Profile Pictures

```bash
# 1. Load relevant skills
/skill docker-expert
/skill kubernetes-guru

# 2. Create specification
/spec "Add user profile picture upload with S3 storage"

# AI creates spec with:
# - API endpoints (POST /users/:id/avatar)
# - File validation (max 5MB, image types only)
# - S3 bucket configuration
# - CDN integration

# 3. Review and approve spec
# (Review artifacts in Agent Manager)

# 4. Switch to code mode
/mode code

# 5. Implement with context
@src/users @src/storage "Implement according to spec"

# 6. AI writes code following rules:
# - Uses TypeScript (global rule)
# - Follows repository pattern (project rule)
# - Includes validation (skill expertise)

# 7. Run tests
/test

# 8. AI debugs any failures
/debug "Fix failing tests"

# 9. Create Docker configuration
@skill docker-expert "Create production Dockerfile"

# 10. Deploy workflow
kiro agent start "Deploy to staging"

# Agent:
# - Builds Docker image
# - Pushes to registry
# - Updates Kubernetes deployment
# - Runs smoke tests
# - Reports success with artifacts
```

---

## Comparison: Feature Support Across IDEs

| Feature | Kilo Code | Cursor | Windsurf | Antigravity | VS Code + Copilot |
|---------|-----------|--------|----------|-------------|-------------------|
| **Autonomous Agents** | ✅ Full | ⚠️ Limited | ✅ Flow | ✅ Full | ❌ |
| **Custom Commands** | ✅ `/` commands | ⚠️ Keyboard shortcuts | ⚠️ Flow triggers | ✅ Natural language | ⚠️ Extensions |
| **Skills System** | ✅ Built-in | ❌ | ❌ | ⚠️ Via context | ⚠️ Via extensions |
| **Markdown Rules** | ✅ `.kiro/rules.md` | ✅ `.cursor/rules.md` | ⚠️ Config-based | ⚠️ Settings-based | ❌ |
| **Codebase Indexing** | ✅ Qdrant | ✅ Local | ✅ Local | ⚠️ Limited | ⚠️ Enterprise only |
| **Structured Workflows** | ✅ Modes + Spec | ⚠️ Composer | ✅ Flow | ✅ Planning modes | ❌ |
| **Parallel Agents** | ✅ Git worktrees | ❌ | ❌ | ✅ | ❌ |

---

## Conclusion

Understanding these core concepts transforms how you work with AI IDEs:

| Concept | Key Takeaway |
|---------|--------------|
| **Agents** | Autonomous workers that execute tasks, not just suggest code |
| **Commands** | Explicit instructions for specific AI behaviors |
| **Skills** | Modular expertise you load for domain-specific tasks |
| **Rules** | Constraints that ensure AI follows your standards |
| **Context** | Information the AI uses—better context = better results |
| **Workflows** | Structured processes for repeatable AI-assisted development |

**Getting Started:**

1. **Pick one IDE** and learn its specific implementation
2. **Start with rules**—define your non-negotiables first
3. **Add skills gradually**—load expertise as needed
4. **Experiment with agents**—start with Review-driven autonomy
5. **Document workflows**—create repeatable processes for your team

The future of software development isn't just about AI writing code—it's about orchestrating intelligent agents with the right context, skills, and rules to build better software faster.


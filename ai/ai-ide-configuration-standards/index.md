---
title: AI IDE Configuration Standards: From .vscode to AGENTS.md
url: https://devopstales.github.io/ai/ai-ide-configuration-standards/
date: 2026-03-21
---


As AI tools become more integrated into our development workflows, the way we configure our projects is changing. We've moved beyond simple `.gitignore` and `.env` files into a world where we need to provide specific "instructions" and "context" to our AI assistants.

<!--more-->

In this post, we'll explore the different IDE configuration standards, from the classic `.vscode` folder to emerging open-source standards like `AGENTS.md`.

---

## 1. .vscode (The Industry Standard)

Visual Studio Code popularized the idea of project-specific configuration folders. While not AI-specific, it remains the foundation for many AI-native IDEs that are forks of VS Code.

### Directory Structure

```bash
project/
├── .vscode/
│   ├── settings.json      # Editor settings
│   ├── tasks.json         # Build tasks
│   ├── launch.json        # Debug configurations
│   ├── extensions.json    # Recommended extensions
│   └── snippets/          # Code snippets
│       ├── javascript.json
│       └── typescript.json
└── src/
```

### Key Files

#### settings.json

Project-specific editor settings:

```json
{
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "files.exclude": {
    "**/node_modules": true,
    "**/dist": true,
    "**/.git": true
  },
  "search.exclude": {
    "**/node_modules": true,
    "**/bower_components": true,
    "**/*.code-search": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true,
  "eslint.validate": [
    "javascript",
    "javascriptreact",
    "typescript",
    "typescriptreact"
  ],
  "[typescript]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "[typescriptreact]": {
    "editor.defaultFormatter": "esbenp.prettier-vscode"
  },
  "editor.rulers": [80, 100, 120],
  "workbench.editorAssociations": {
    "*.md": "vscode.markdown.preview.editor"
  }
}
```

#### tasks.json

Build, test, and deployment scripts:

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "Build",
      "type": "npm",
      "script": "build",
      "group": "build",
      "presentation": {
        "reveal": "never"
      },
      "problemMatcher": ["$tsc"]
    },
    {
      "label": "Test",
      "type": "npm",
      "script": "test",
      "group": "test",
      "presentation": {
        "reveal": "never"
      }
    },
    {
      "label": "Start Dev Server",
      "type": "npm",
      "script": "dev",
      "group": {
        "kind": "build",
        "isDefault": true
      },
      "isBackground": true,
      "problemMatcher": {
        "pattern": {
          "regexp": "^.*ready in.*$",
          "file": 1
        },
        "background": {
          "activeOnStart": true,
          "beginsPattern": "starting",
          "endsPattern": "ready"
        }
      }
    },
    {
      "label": "Lint",
      "type": "npm",
      "script": "lint",
      "group": "test",
      "problemMatcher": ["$eslint-stylish"]
    }
  ]
}
```

#### launch.json

Debugger configurations:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Current File",
      "skipFiles": ["<node_internals>/**"],
      "program": "${file}",
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    },
    {
      "type": "node",
      "request": "launch",
      "name": "Debug All Tests",
      "skipFiles": ["<node_internals>/**"],
      "program": "${workspaceFolder}/node_modules/.bin/jest",
      "args": ["--runInBand"],
      "cwd": "${workspaceFolder}",
      "console": "integratedTerminal"
    },
    {
      "type": "chrome",
      "request": "launch",
      "name": "Launch Chrome",
      "url": "http://localhost:3000",
      "webRoot": "${workspaceFolder}/src",
      "runtimeArgs": ["--disable-dev-shm-usage"]
    }
  ]
}
```

#### extensions.json

Recommended extensions for the project:

```json
{
  "recommendations": [
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "bradlc.vscode-tailwindcss",
    "formulahendry.auto-rename-tag",
    "christian-kohler.path-intellisense",
    "streetsidesoftware.code-spell-checker",
    "GitHub.copilot",
    "GitHub.copilot-chat"
  ],
  "unwantedRecommendations": [
    "eg2.vscode-npm-script"
  ]
}
```

### Purpose

- Ensure every developer on a team has a consistent editor experience
- Provide access to the same build/debug tools
- Standardize code formatting and linting
- Recommend useful extensions

### AI Integration

While `.vscode` wasn't designed for AI, AI extensions use it:

```json
// settings.json - AI-specific settings
{
  "github.copilot.enable": true,
  "github.copilot.editor.enableAutoCompletions": true,
  "continue.enableTabAutocomplete": true,
  "kiloCode.provider": "qwen-code",
  "kiloCode.model": "qwen-coder-plus"
}
```

---

## 2. .cursor (The AI-Native Fork)

[Cursor](https://cursor.com/) has introduced its own set of standards to help its AI engine understand your codebase better.

### Directory Structure

```bash
project/
├── .cursor/
│   ├── rules/              # AI rules directory
│   │   ├── coding-standards.md
│   │   ├── testing.md
│   │   └── security.md
│   ├── config.json         # Cursor-specific config
│   └── prompts/            # Custom prompts
│       └── code-review.md
└── src/
```

### Key Features

#### .cursorrules

A file (or directory) where you can define specific rules for the AI:

```markdown
# .cursor/rules/coding-standards.md

## Code Style
- Always use TypeScript for new code
- Use functional components in React
- Prefer composition over inheritance
- Use async/await, not Promise chains

## Naming Conventions
- Components: PascalCase (UserProfile)
- Functions: camelCase (getUserById)
- Constants: UPPER_SNAKE_CASE (MAX_RETRY_COUNT)
- Files: kebab-case (user-profile.tsx)

## Imports
- Group imports: React, third-party, internal, relative
- Use absolute imports from @/
- No circular dependencies

## Error Handling
- Always handle errors explicitly
- Use custom error classes
- Log errors with context
- Never swallow errors silently

## Testing
- Write tests for all public functions
- Use describe/it/test pattern
- Mock external dependencies
- Aim for 80%+ coverage
```

#### config.json

Cursor-specific configuration:

```json
{
  "ai": {
    "model": "claude-3-5-sonnet-20241022",
    "temperature": 0.3,
    "maxTokens": 8192
  },
  "indexing": {
    "enabled": true,
    "ignorePatterns": [
      "node_modules/**",
      "dist/**",
      "build/**",
      "*.min.js",
      "coverage/**"
    ],
    "maxFileSize": "1MB"
  },
  "composer": {
    "autoApprove": false,
    "requireConfirmation": true
  },
  "privacy": {
    "mode": "local",
    "sendTelemetry": false
  }
}
```

#### Custom Prompts

`.cursor/prompts/code-review.md`:

```markdown
# Code Review Prompt

When reviewing code, follow this checklist:

## Security
- [ ] No hardcoded secrets
- [ ] Input validation present
- [ ] SQL injection prevented
- [ ] XSS prevention in place

## Performance
- [ ] No N+1 queries
- [ ] Proper indexing
- [ ] Caching where appropriate
- [ ] No memory leaks

## Code Quality
- [ ] Follows style guide
- [ ] Functions are small
- [ ] Clear variable names
- [ ] Comments explain why, not what

## Testing
- [ ] Tests cover edge cases
- [ ] Mocks used appropriately
- [ ] Test names are descriptive

Provide feedback in this format:
1. Summary
2. Critical issues (blocking)
3. Suggestions (non-blocking)
4. Questions
```

### Purpose

- Provide the AI with project-specific "tribal knowledge"
- Define coding standards for AI to follow
- Customize AI behavior for the project
- Create reusable prompt templates

---

## 3. .kilocode (Kiro.dev / Kilo Code)

[Kiro.dev](https://kiro.dev) and the Kilo Code ecosystem use a structured approach to manage "agents" and "modes" within your repository.

### Directory Structure

```bash
project/
├── .kilocode/
│   ├── config.json         # Main configuration
│   ├── mcp.json            # MCP server config
│   ├── agents.json         # Agent definitions
│   ├── skills.json         # Skills configuration
│   ├── rules/              # Steering rules
│   │   ├── coding-standards.md
│   │   ├── security.md
│   │   ├── testing.md
│   │   └── documentation.md
│   ├── agents/             # Custom agents
│   │   ├── security-reviewer.md
│   │   ├── code-reviewer.md
│   │   └── tech-writer.md
│   ├── skills/             # Custom skills
│   │   ├── docker-expert/
│   │   │   ├── skill.json
│   │   │   └── prompt.md
│   │   └── aws-helper/
│   ├── specs/              # Specifications
│   │   ├── feature-auth.md
│   │   └── feature-api.md
│   ├── tasks/              # Task definitions
│   │   └── phase-1/
│   └── index/              # Indexing data
│       └── qdrant/
├── .kilocodemodes          # Mode definitions
├── .kilocodeignore         # Indexing exclusions
└── src/
```

### Key Files

#### config.json

Main Kilo Code configuration:

```json
{
  "provider": "qwen-code",
  "model": "qwen-coder-plus",
  "mode": "orchestrator",
  "indexing": {
    "enabled": true,
    "provider": "qdrant",
    "embedding": {
      "model": "nomic-embed-text",
      "provider": "ollama",
      "endpoint": "http://localhost:11434"
    },
    "vectorStore": {
      "provider": "qdrant",
      "endpoint": "http://localhost:6333",
      "collectionName": "my-project"
    },
    "chunking": {
      "strategy": "code-aware",
      "chunkSize": 512,
      "overlap": 64
    }
  },
  "steering": {
    "rules": ["coding-standards", "security", "testing"]
  },
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  },
  "agentManager": {
    "enabled": true,
    "maxParallelAgents": 4,
    "worktreeBasePath": "../worktrees"
  }
}
```

#### .kilocode/rules/

Steering rules in Markdown format:

```markdown
# .kilocode/rules/coding-standards.md

## TypeScript Standards

### Always Use TypeScript
- All new code must be TypeScript (.ts, .tsx)
- Never use `any` type
- Define interfaces for all objects

### Code Style
- Use functional components
- Use async/await
- Prefer const over let

### Libraries
- Use Zod for validation
- Use date-fns for dates
- Use axios for HTTP
```

#### .kilocodemodes

Custom mode definitions:

```json
{
  "modes": {
    "security-review": {
      "baseMode": "architect",
      "systemPrompt": "You are a security expert reviewing code.",
      "tools": ["code-scanner", "dependency-checker"],
      "restrictions": {
        "cannotModifyFiles": true
      }
    },
    "documentation": {
      "baseMode": "code",
      "systemPrompt": "You are a technical writer.",
      "tools": ["file-reader", "markdown-generator"]
    }
  }
}
```

#### .kilocodeignore

Prevents the AI from indexing sensitive files:

```bash
# .kilocodeignore
node_modules/
dist/
build/
*.min.js
coverage/
.env
*.log
.secrets/
```

### Purpose

- Create a highly disciplined and specialized AI agent
- Follow strict operational modes
- Enable parallel agent execution
- Integrate with external tools via MCP

---

## 4. .qwen/ (Qwen Code CLI)

As the Qwen ecosystem (from Alibaba Cloud) grows, it has introduced its own configuration standards for its CLI tools and agents.

### Directory Structure

```bash
project/
├── .qwen/
│   ├── config.json         # Qwen configuration
│   ├── context/            # Context files
│   │   └── project-context.md
│   └── cache/              # Cached data
└── src/
```

### Key Files

#### config.json

Qwen-specific configuration:

```json
{
  "model": {
    "name": "qwen-coder-plus",
    "temperature": 0.3,
    "maxTokens": 100000,
    "topP": 0.9
  },
  "context": {
    "maxTokens": 1000000,
    "smartTrimming": true,
    "priorityFiles": [
      "README.md",
      "package.json",
      "tsconfig.json"
    ]
  },
  "indexing": {
    "enabled": true,
    "strategy": "semantic",
    "chunkSize": 4096,
    "overlapSize": 512
  },
  "caching": {
    "enabled": true,
    "directory": "~/.cache/qwen-code",
    "ttl": 3600
  }
}
```

#### project-context.md

Project-specific context for the AI:

```markdown
# Project Context

## Overview
This is a Node.js REST API for an e-commerce platform.

## Tech Stack
- Runtime: Node.js 20
- Framework: Express.js
- Database: PostgreSQL with Prisma ORM
- Authentication: JWT with refresh tokens

## Key Directories
- src/controllers/ - Request handlers
- src/services/ - Business logic
- src/models/ - Database models
- src/middleware/ - Express middleware

## Important Commands
- `npm run dev` - Start development server
- `npm run build` - Build for production
- `npm test` - Run tests
- `npm run db:migrate` - Run database migrations

## Architecture Notes
- All controllers use async/await
- Services handle business logic
- Models use Prisma Client
- Middleware handles auth and validation
```

### Purpose

- Configure which specific version of the Qwen-Coder model to use
- Manage the temporary files and context windows
- Optimize performance and cost-efficiency

---

## 5. AGENTS.md / CLAUDE.md (The "README for Machines")

A new emerging standard is the `AGENTS.md` or `CLAUDE.md` file—a human and machine-readable document that explains how to work with the codebase.

### Purpose

This is the "README for machines"—a document specifically designed to help AI assistants understand your project quickly.

### Example: AGENTS.md

`````markdown
# Agent Instructions

## Quick Start

To work on this project, an AI agent should:

1. Understand the project structure (see below)
2. Follow the coding standards
3. Run tests before committing
4. Update documentation for new features

## Project Structure

```
project/
├── src/
│   ├── controllers/    # HTTP request handlers
│   ├── services/       # Business logic
│   ├── models/         # Database models
│   ├── middleware/     # Express middleware
│   └── utils/          # Utility functions
├── tests/
│   ├── unit/           # Unit tests
│   ├── integration/    # Integration tests
│   └── e2e/            # End-to-end tests
├── docs/               # Documentation
└── scripts/            # Build/deploy scripts
```

## Coding Standards

### TypeScript
- Use strict mode
- No `any` types
- Define interfaces for objects

### Testing
- Jest for unit tests
- Supertest for API tests
- Playwright for E2E

### Code Style
- Prettier for formatting
- ESLint for linting
- 2-space indentation

## Common Tasks

### Adding a New API Endpoint

1. Create controller in `src/controllers/`
2. Create service in `src/services/`
3. Add route in `src/routes/`
4. Write tests in `tests/`
5. Update API documentation

### Fixing a Bug

1. Reproduce the bug
2. Write a failing test
3. Fix the code
4. Verify test passes
5. Run full test suite

## Testing Commands

```bash
# Run all tests
npm test

# Run specific test file
npm test -- path/to/test.ts

# Run with coverage
npm run test:coverage

# Run E2E tests
npm run test:e2e
```

## Deployment

```bash
# Build for production
npm run build

# Start production server
npm start

# Deploy to staging
npm run deploy:staging

# Deploy to production
npm run deploy:prod
```

## Troubleshooting

### Common Issues

**Database Connection Failed**
- Check DATABASE_URL environment variable
- Ensure database is running
- Run migrations: `npm run db:migrate`

**Tests Failing**
- Clear Jest cache: `npm run test:clear`
- Check for uncommitted changes
- Verify environment variables

## Contact

For questions about this project, contact the team via Slack #dev-channel.
`````

### Example: CLAUDE.md

`````markdown
# Claude Instructions

## How to Help

When assisting with this codebase:

1. **Read this file first** for context
2. **Check existing code** before suggesting changes
3. **Follow patterns** already established
4. **Write tests** for any new code
5. **Update docs** for new features

## What NOT to Do

- ❌ Don't use `any` types
- ❌ Don't skip error handling
- ❌ Don't add new dependencies without approval
- ❌ Don't modify config files without explanation
- ❌ Don't commit without running tests

## Preferred Patterns

### API Response Format
```typescript
{
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
  };
}
```

### Error Handling
```typescript
try {
  await someOperation();
} catch (error) {
  logger.error('Operation failed', { error, context });
  throw new AppError('OPERATION_FAILED', 'Human readable message');
}
```

### Logging
```typescript
import { logger } from '@/utils/logger';

logger.info('User logged in', { userId });
logger.warn('Rate limit approaching', { current: 90, max: 100 });
logger.error('Database connection failed', { error });
```

## Review Checklist

Before suggesting code changes, verify:
- [ ] TypeScript types are correct
- [ ] Error handling is present
- [ ] Tests are included
- [ ] No console.log in production code
- [ ] Follows existing patterns
`````

### Benefits

- **Fast Context**: AI gets up to speed quickly
- **Consistent Output**: AI follows your patterns
- **Reduced Tokens**: Less back-and-forth needed
- **Team Alignment**: Humans also benefit from the documentation

---

## Configuration Comparison Table

| Standard | Primary Use | AI-Specific | File Format | Complexity |
|----------|-------------|-------------|-------------|------------|
| **.vscode** | Editor config | No | JSON | Medium |
| **.cursor** | AI rules | Yes | MD + JSON | Low |
| **.kilocode** | Agent config | Yes | JSON + MD | High |
| **.qwen** | Model config | Yes | JSON | Medium |
| **AGENTS.md** | Documentation | Yes | Markdown | Low |

---

## Best Practices

### 1. Keep Configuration Versioned

```bash
# Always commit configuration
git add .vscode/ .kilocode/ AGENTS.md
git commit -m "Add AI configuration"
```

### 2. Use Templates

Create templates for new projects:

```bash
templates/
├── .kilocode/
│   └── config.template.json
├── AGENTS.md.template
└── .cursorrules.template
```

### 3. Document Your Configuration

```markdown
# .kilocode/README.md

## Configuration Overview

This directory contains Kilo Code configuration:

- `config.json` - Main configuration
- `rules/` - Steering rules for AI behavior
- `agents/` - Custom agent definitions
- `skills/` - Specialized skill packages

## Adding New Rules

1. Create file in `rules/`
2. Add to `config.json` steering array
3. Test with: `kiro rules check`
```

### 4. Separate Secrets

```json
// Good: Reference environment variables
{
  "apiKey": "${GITHUB_TOKEN}"
}

// Bad: Hardcoded secrets
{
  "apiKey": "ghp_xxxxxxxxxxxx"  // Never do this!
}
```

### 5. Validate Configuration

```bash
# Validate Kilo Code config
kiro config validate

# Validate Cursor rules
cursor rules check

# Validate AGENTS.md
kiro agents validate AGENTS.md
```

---

## Conclusion

The evolution of IDE configuration reflects the evolution of development itself:

| Era | Configuration Focus |
|-----|---------------------|
| **Pre-2015** | Build tools, compilers |
| **2015-2020** | Editor settings, extensions |
| **2020-2025** | AI assistants, code completion |
| **2025+** | Agentic AI, autonomous workflows |

**Recommendations:**

1. **Start with AGENTS.md** - Easy to create, immediate value
2. **Add .cursor or .kilocode** - Depending on your AI tool
3. **Maintain .vscode** - For editor consistency
4. **Document everything** - Future you will thank you

---

## Quick Reference

### Configuration Files Summary

| File | Purpose | Tool |
|------|---------|------|
| `.vscode/settings.json` | Editor settings | VS Code |
| `.cursor/rules/` | AI rules | Cursor |
| `.kilocode/config.json` | Agent config | Kilo Code |
| `.qwen/config.json` | Model config | Qwen |
| `AGENTS.md` | Project documentation | All AI tools |
| `CLAUDE.md` | Claude instructions | Claude |
| `.stbt.conf` | Test configuration | stb-tester |

### Useful Commands

```bash
# Kilo Code
kiro config validate
kiro rules list
kiro agents list

# Cursor
cursor rules check

# Qwen
qwen-code config show
qwen-code context status
```


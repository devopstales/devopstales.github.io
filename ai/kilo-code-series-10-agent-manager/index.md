---
title: Kilo Code: Parallel Agents and the Agent Manager
url: https://devopstales.github.io/ai/kilo-code-series-10-agent-manager/
date: 2026-04-03
---


Welcome to the final installment of our **Kilo Code Deep Dive** series. Throughout this journey, we've explored installation, Qwen integration, Modes, Indexing, Spec-Driven Development, Steering, MCP, and Skills.

<!--more-->

{{< content "/filedir/kilo-code-series.html" >}}

![Parallel AI agents working together](/img/developer-workflow.webp)
*Agent Manager coordinates multiple AI agents working in parallel on different tasks*

Today, we explore one of Kilo Code's most advanced features: **Parallel Agents** and the **Kilo Agent Manager**.

## The Power of Parallel Agents

Traditional AI coding is sequential:

```bash
Task: Build a complete feature with API, database, and frontend

Sequential Approach (Traditional):
┌───────────────────────────────────────┐
│  Hour 1-2: Design database schema     │
│  Hour 3-4: Implement API endpoints    │
│  Hour 5-6: Build frontend components  │
│  Hour 7-8: Write tests                │
│                                       │
│  Total: 8 hours                       │
└───────────────────────────────────────┘

Parallel Approach (Kilo Agent Manager):
┌──────────────────────────────────────────────────────┐
│  Agent 1: Database schema (Worktree: feature-db)     │
│  Agent 2: API endpoints (Worktree: feature-api)      │
│  Agent 3: Frontend components (Worktree: feature-ui) │
│  Agent 4: Tests (Worktree: feature-tests)            │
│                                                      │
│  All agents work simultaneously...                   │
│  Orchestrator merges results                         │
│                                                      │
│  Total: 2-3 hours                                    │
└──────────────────────────────────────────────────────┘
```

### Benefits of Parallel Agents

| Benefit | Description |
|---------|-------------|
| **Speed** | 3-4x faster for complex tasks |
| **Specialization** | Each agent focuses on its domain |
| **Isolation** | No conflicts between concurrent changes |
| **Quality** | More focused attention per task |
| **Scalability** | Add more agents for bigger projects |

---

## Git Worktree Integration

### What are Git Worktrees?

Git worktrees allow you to have multiple working directories for the same repository:

```bash
Repository: my-app/
├── .git/
├── main/           ← Your main working directory
├── feature-db/     ← Worktree for database work
├── feature-api/    ← Worktree for API work
├── feature-ui/     ← Worktree for UI work
└── feature-tests/  ← Worktree for test work

All worktrees share the same .git directory!
```

### Why Worktrees for Parallel Agents?

1. **Isolation**: Each agent works in its own space
2. **No Conflicts**: Changes don't interfere with each other
3. **Easy Merge**: Git handles the integration
4. **Clean History**: Each agent's work is tracked separately

### Creating Worktrees

```bash
# Create worktree for database agent
git worktree add ../my-app-feature-db feature/database-schema

# Create worktree for API agent
git worktree add ../my-app-feature-api feature/api-endpoints

# Create worktree for UI agent
git worktree add ../my-app-feature-ui feature/frontend-components

# List all worktrees
git worktree list

# Output:
/path/to/my-app          main
/path/to/my-app-feature-db  feature/database-schema
/path/to/my-app-feature-api feature/api-endpoints
/path/to/my-app-feature-ui  feature/frontend-components
```

---

## Agent Manager Architecture

```bash
┌───────────────────────────────────────────────┐
│                    Kilo Agent Manager         │
├───────────────────────────────────────────────┤
│                                               │
│  ┌─────────────┐                              │
│  │Orchestrator │ ← Coordinates all agents     │
│  │   Agent     │                              │
│  └──────┬──────┘                              │
│         │                                     │
│    ┌────┴────┬────────────┬────────────┐      │
│    │         │            │            │      │
│    ▼         ▼            ▼            ▼      │
│ ┌──────┐ ┌──────┐    ┌──────┐    ┌──────┐     │
│ │Agent │ │Agent │    │Agent │    │Agent │     │
│ │  1   │ │  2   │    │  3   │    │  4   │     │
│ │(DB)  │ │(API) │    │ (UI) │    │(Test)│     │
│ └──┬───┘ └──┬───┘    └──┬───┘    └──┬───┘     │
│    │        │           │           │         │
│    ▼        ▼           ▼           ▼         │
│ ┌──────┐ ┌──────┐  ┌───────┐  ┌────────┐      │
│ │Work- │ │Work- │  │ Work- │  │ Work-  │      │
│ │ tree │ │ tree │  │ tree  │  │ tree   │      │
│ │  1   │ │  2   │  │   3   │  │   4    │      │
│ └──────┘ └──────┘  └───────┘  └────────┘      │
│                                               │
└───────────────────────────────────────────────┘
```

---

## Setting Up Agent Manager

### Step 1: Configure Agent Manager

Create `.kilocode/agent-manager.json`:

```json
{
  "enabled": true,
  "maxParallelAgents": 4,
  "worktreeBasePath": "../my-app-worktrees",
  "orchestrator": {
    "model": "qwen-coder-plus",
    "temperature": 0.3
  },
  "agents": {
    "default": {
      "model": "qwen-coder-32b",
      "temperature": 0.5,
      "maxTokens": 8192
    },
    "database": {
      "model": "qwen-coder-plus",
      "temperature": 0.2,
      "skills": ["database-expert", "sql-optimizer"]
    },
    "api": {
      "model": "qwen-coder-32b",
      "temperature": 0.4,
      "skills": ["api-design", "rest-best-practices"]
    },
    "frontend": {
      "model": "qwen-coder-32b",
      "temperature": 0.6,
      "skills": ["react-expert", "typescript-pro"]
    },
    "testing": {
      "model": "qwen-coder-32b",
      "temperature": 0.3,
      "skills": ["testing-expert", "jest-master"]
    }
  },
  "merge": {
    "strategy": "orchestrator-review",
    "autoCommit": false,
    "createPullRequest": true
  }
}
```

### Step 2: Define Agent Roles

Create `.kilocode/agents-config.json`:

```json
{
  "roles": {
    "database": {
      "description": "Database schema and migrations",
      "worktreePrefix": "feature-db",
      "branches": {
        "prefix": "feature/db"
      },
      "skills": ["database-expert", "postgres-pro"],
      "mcpServers": ["postgres"]
    },
    "api": {
      "description": "API endpoints and business logic",
      "worktreePrefix": "feature-api",
      "branches": {
        "prefix": "feature/api"
      },
      "skills": ["api-design", "express-expert"],
      "mcpServers": []
    },
    "frontend": {
      "description": "UI components and styling",
      "worktreePrefix": "feature-ui",
      "branches": {
        "prefix": "feature/ui"
      },
      "skills": ["react-expert", "tailwind-master"],
      "mcpServers": []
    },
    "testing": {
      "description": "Unit and integration tests",
      "worktreePrefix": "feature-tests",
      "branches": {
        "prefix": "feature/tests"
      },
      "skills": ["testing-expert", "jest-master"],
      "mcpServers": []
    }
  }
}
```

---

## Running Parallel Agents

### Starting an Agent Session

```bash
# Start agent manager with task description
kilo-code agent-manager start "Build a user profile feature with avatar upload"

# Output:
🚀 Kilo Agent Manager Starting...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📋 Task Analysis:
   Building user profile feature with avatar upload

📊 Creating Agent Assignments:
   ✓ Agent 1 (database): Schema for user profiles and avatars
   ✓ Agent 2 (api): Profile CRUD and file upload endpoints
   ✓ Agent 3 (frontend): Profile page and avatar upload UI
   ✓ Agent 4 (testing): Tests for all components

🌳 Setting Up Worktrees:
   ✓ Created worktree: feature-db-001
   ✓ Created worktree: feature-api-001
   ✓ Created worktree: feature-ui-001
   ✓ Created worktree: feature-tests-001

🤖 Launching Agents:
   ✓ Agent 1 started (PID: 12345)
   ✓ Agent 2 started (PID: 12346)
   ✓ Agent 3 started (PID: 12347)
   ✓ Agent 4 started (PID: 12348)

⏳ Agents are working... Check status with: kilo-code agent-manager status
```

### Monitoring Progress

```bash
# Check agent status
kilo-code agent-manager status

# Output:
Agent Manager Status
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Session: session-20260403-001
Started: 2026-04-03 10:30:00
Elapsed: 45 minutes

┌───────────┬────────────────┬─────────┬──────────┐
│ Agent     │ Task           │ Status  │ Progress │
├───────────┼────────────────┼─────────┼──────────┤
│ database  │ Schema design  │ Done    │ 100%     │
│ api       │ Endpoints      │ Working │ 75%      │
│ frontend  │ UI components  │ Working │ 60%      │
│ testing   │ Test suite     │ Waiting │ 0%       │
└───────────┴────────────────┴─────────┴──────────┘

Recent Activity:
  [10:45] database: Created migration file
  [10:52] api: Implemented profile endpoint
  [10:55] frontend: Created ProfilePage component
  [11:00] api: Added file upload handler
```

### Viewing Agent Logs

```bash
# View all logs
kilo-code agent-manager logs

# View specific agent logs
kilo-code agent-manager logs --agent api

# Follow logs in real-time
kilo-code agent-manager logs --follow

# Output (follow mode):
[11:05:23] [api] Creating file upload handler...
[11:05:45] [api] Implemented multer middleware
[11:06:02] [api] Added S3 upload integration
[11:06:15] [api] Testing upload endpoint...
[11:06:30] [api] ✓ Upload tests passing
```

---

## Agent Coordination

### Inter-Agent Communication

Agents can communicate through the orchestrator:

```bash
Agent 2 (API) → Orchestrator: "I need the user schema to build endpoints"
Orchestrator → Agent 1 (Database): "Please share the user schema"
Agent 1 (Database) → Orchestrator: "Here's the schema"
Orchestrator → Agent 2 (API): "Schema received, proceeding"
```

### Dependency Management

```json
{
  "dependencies": {
    "api": ["database"],
    "frontend": ["api"],
    "testing": ["database", "api", "frontend"]
  }
}
```

This ensures agents wait for dependencies:

```bash
┌──────────────────────────────────────────────────────────┐
│ Timeline:                                                │
│                                                          │
│ Minute 0-30:  [database] ████████████████████████████    │
│                                                          │
│ Minute 30-60: [api]     ████████████████████████████     │
│               [frontend] ████████ (waiting for API spec) │
│                                                          │
│ Minute 60-90: [frontend] ████████████████████            │
│               [testing]  ████████████████████████████    │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### Sharing Artifacts

Agents share outputs through a common directory:

```bash
.kilocode/
└── agent-artifacts/
    ├── database/
    │   └── schema.json      # Shared with API agent
    ├── api/
    │   └── openapi.yaml     # Shared with frontend agent
    ├── frontend/
    │   └── components.json  # Shared with testing agent
    └── testing/
        └── coverage.json    # Final report
```

---

## Merging Results

### Review Mode

Before merging, the orchestrator reviews all changes:

```bash
kilo-code agent-manager review

# Output:
Review Session
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Changes Summary:
┌───────────┬────────────┬──────────────┬───────────┐
│ Agent     │ Files      │ Lines Added  │ Lines Del │
├───────────┼────────────┼──────────────┼───────────┤
│ database  │ 3          │ 145          │ 0         │
│ api       │ 5          │ 312          │ 12        │
│ frontend  │ 8          │ 567          │ 23        │
│ testing   │ 6          │ 423          │ 0         │
└───────────┴────────────┴──────────────┴───────────┘
Total: 22 files, 1,447 additions, 35 deletions

Conflicts Detected: 0 ✓

Preview Changes:
  kilo-code agent-manager diff

Merge Options:
  kilo-code agent-manager merge --approve
  kilo-code agent-manager merge --reject <agent>
  kilo-code agent-manager merge --request-changes <agent>
```

### Merge Strategies

**Strategy 1: Orchestrator Review (Default)**
```json
{
  "merge": {
    "strategy": "orchestrator-review"
  }
}
```

**Strategy 2: Auto-Merge (Fast)**
```json
{
  "merge": {
    "strategy": "auto",
    "requireTests": true
  }
}
```

**Strategy 3: Manual Review (Safe)**
```json
{
  "merge": {
    "strategy": "manual",
    "createPullRequest": true
  }
}
```

### Executing the Merge

```bash
# Review and approve merge
kilo-code agent-manager merge --approve

# Output:
Merging Changes...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

✓ Merging database changes...
✓ Merging API changes...
✓ Merging frontend changes...
✓ Merging test changes...

✓ All changes merged successfully!
✓ Created commit: "feat: Add user profile with avatar upload"
✓ Created PR: #142 - User Profile Feature

Next Steps:
  - Review PR: https://github.com/org/repo/pull/142
  - Clean up worktrees: kilo-code agent-manager cleanup
```

---

## Advanced Patterns

### Pattern 1: Microservices Development

```json
{
  "session": {
    "name": "microservices-feature",
    "agents": [
      {
        "role": "user-service",
        "worktree": "feature/user-svc",
        "directory": "services/user-service"
      },
      {
        "role": "order-service",
        "worktree": "feature/order-svc",
        "directory": "services/order-service"
      },
      {
        "role": "api-gateway",
        "worktree": "feature/gateway",
        "directory": "services/api-gateway"
      },
      {
        "role": "integration-tests",
        "worktree": "feature/integration",
        "directory": "tests/integration"
      }
    ]
  }
}
```

### Pattern 2: Full-Stack Feature

```json
{
  "session": {
    "name": "fullstack-feature",
    "agents": [
      {
        "role": "backend",
        "worktree": "feature/backend",
        "skills": ["api-design", "database-expert"]
      },
      {
        "role": "frontend",
        "worktree": "feature/frontend",
        "skills": ["react-expert", "state-management"]
      },
      {
        "role": "mobile",
        "worktree": "feature/mobile",
        "skills": ["react-native", "mobile-ui"]
      },
      {
        "role": "qa",
        "worktree": "feature/qa",
        "skills": ["testing-expert", "e2e-testing"]
      }
    ]
  }
}
```

### Pattern 3: Refactoring Project

```json
{
  "session": {
    "name": "refactor-monolith",
    "agents": [
      {
        "role": "analyzer",
        "worktree": "refactor/analyze",
        "task": "Analyze codebase and identify refactoring opportunities"
      },
      {
        "role": "extractor",
        "worktree": "refactor/extract",
        "task": "Extract modules into separate packages"
      },
      {
        "role": "migrator",
        "worktree": "refactor/migrate",
        "task": "Update imports and dependencies"
      },
      {
        "role": "validator",
        "worktree": "refactor/validate",
        "task": "Run tests and verify functionality"
      }
    ]
  }
}
```

---

## Performance Optimization

### Resource Allocation

```json
{
  "resources": {
    "maxCpuPerAgent": 2,
    "maxMemoryPerAgent": "4GB",
    "maxConcurrentAgents": 4,
    "priorityMode": "balanced"
  }
}
```

### Caching

```json
{
  "caching": {
    "enabled": true,
    "sharedModelCache": true,
    "embeddingCache": true,
    "cacheLocation": "~/.cache/kilo-agents"
  }
}
```

### Batch Processing

For large tasks, split into batches:

```bash
# Process in batches of 2 agents
kilo-code agent-manager start "Refactor all components" --batch-size 2

# Output:
Batch 1/3: Processing components A-H
Batch 2/3: Processing components I-P
Batch 3/3: Processing components Q-Z
```

---

## Troubleshooting

### Issue: Agent Stuck

```bash
# Check agent status
kilo-code agent-manager status

# Restart specific agent
kilo-code agent-manager restart --agent api

# View agent logs for errors
kilo-code agent-manager logs --agent api --last 100
```

### Issue: Merge Conflicts

```bash
# Preview conflicts
kilo-code agent-manager conflicts

# Resolve conflicts manually
kilo-code agent-manager resolve --agent api

# Or let orchestrator resolve
kilo-code agent-manager resolve --auto
```

### Issue: Worktree Problems

```bash
# List worktrees
git worktree list

# Remove problematic worktree
git worktree remove ../my-app-worktrees/feature-api-001 --force

# Recreate worktree
kilo-code agent-manager recreate --agent api
```

### Issue: Out of Memory

```json
{
  "resources": {
    "maxMemoryPerAgent": "2GB",
    "maxConcurrentAgents": 2
  }
}
```

---

## Best Practices

### 1. Start Small

```bash
Good:  Start with 2 agents, scale up
Bad:   Start with 8 agents immediately
```

### 2. Clear Task Definitions

```bash
Good:  "Build user authentication with JWT"
Bad:   "Make the app better"
```

### 3. Monitor Regularly

```bash
# Check status every 15 minutes
watch -n 900 'kilo-code agent-manager status'
```

### 4. Clean Up After Sessions

```bash
# Remove worktrees after merge
kilo-code agent-manager cleanup

# Or keep for debugging
kilo-code agent-manager cleanup --keep-failed
```

### 5. Use Appropriate Models

```json
{
  "agents": {
    "database": { "model": "qwen-coder-plus" },
    "frontend": { "model": "qwen-coder-32b" },
    "testing": { "model": "qwen-coder-32b" }
  }
}
```

---

## Series Conclusion

Congratulations on completing the **Kilo Code Deep Dive** series! Here's a quick recap:

| Part | Topic | Key Takeaway |
|------|-------|--------------|
| 1 | Introduction | Kilo Code is an agentic AI platform |
| 2 | Installation | Multiple installation options available |
| 3 | Qwen Integration | Free tier with 1M token context |
| 4 | Modes | Specialized personas for different tasks |
| 5 | Indexing | Semantic search across codebase |
| 6 | SDD | Spec-driven over vibe coding |
| 7 | Steering | Custom rules and agents |
| 8 | MCP | Connect to external tools |
| 9 | Skills | Modular expertise packages |
| 10 | Agent Manager | Parallel agents with worktrees |

### The Kilo Code Advantage

```bash
┌──────────────────────────────────────────────────────┐
│                    Kilo Code Ecosystem               │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Modes         → Specialized AI personas             │
│  Indexing      → Semantic code search                │
│  SDD           → Structured development              │
│  Steering     → Custom rules and agents              │
│  MCP           → External integrations               │
│  Skills        → Modular expertise                   │
│  Parallel      → Multiple agents working together    │
│                                                      │
│  Result: 3-4x faster development with higher quality │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## Quick Reference

### Agent Manager Commands

```bash
# Start session
kilo-code agent-manager start "task description"

# Check status
kilo-code agent-manager status

# View logs
kilo-code agent-manager logs [--agent <name>] [--follow]

# Review changes
kilo-code agent-manager review

# Merge changes
kilo-code agent-manager merge --approve

# Clean up
kilo-code agent-manager cleanup
```

### Git Worktree Commands

```bash
# Add worktree
git worktree add <path> <branch>

# List worktrees
git worktree list

# Remove worktree
git worktree remove <path>

# Prune stale worktrees
git worktree prune
```

### Configuration Files

```bash
.kilocode/
├── agent-manager.json    # Agent Manager config
├── agents-config.json    # Agent roles config
└── agent-artifacts/      # Shared outputs
```


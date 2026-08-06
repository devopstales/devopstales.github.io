---
title: Kilo Code: Advanced MCP Integration
url: https://devopstales.github.io/ai/kilo-code-series-08-mcp/
date: 2026-04-01
---


In our previous post, we looked at how Steering can provide persistent rules for your agents. But what if the agent needs to control the world?

<!--more-->

{{< content "/filedir/kilo-code-series.html" >}}

This is where **Advanced MCP Integration** comes in. Developed by Anthropic and adopted by the entire AI industry, the Model Context Protocol (MCP) is an open standard that allows AI models to connect to external data and tools. Instead of the AI being a "brain in a jar," MCP gives it "hands" and "eyes."

## What is MCP?

**Model Context Protocol (MCP)** is an open standard that defines how AI assistants interact with external systems. Think of it as USB-C for AI tools—a universal connector.

### The Problem MCP Solves

Without MCP, every AI tool needs custom integrations:

```bash
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   GitHub    │────▶│  Custom     │────▶│   AI        │
│   API       │     │  Integration│     │   Assistant │
└─────────────┘     └─────────────┘     └─────────────┘

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Slack     │────▶│  Custom     │────▶│   AI        │
│   API       │     │  Integration│     │   Assistant │
└─────────────┘     └─────────────┘     └─────────────┘

┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Database  │────▶│  Custom     │────▶│   AI        │
│             │     │  Integration│     │   Assistant │
└─────────────┘     └─────────────┘     └─────────────┘

Result: Fragile, hard-to-maintain integrations
```

With MCP, there's a standard interface:

```bash
┌─────────────┐
│   GitHub    │
│   MCP       │
│   Server    │
└──────┬──────┘
       │
┌──────┴──────┐     ┌─────────────┐
│   MCP       │────▶│   AI        │
│   Host      │     │   Assistant │
│   (Kilo)    │     │             │
└──────┬──────┘     └─────────────┘
       │
┌──────┴──────┐
│   Slack     │
│   MCP       │
│   Server    │
└─────────────┘

Result: Standard, swappable integrations
```

---

## MCP Architecture

### Core Components

| Component | Description | Example |
|-----------|-------------|---------|
| **MCP Host** | The AI application | Kilo Code, Claude Desktop |
| **MCP Server** | Provides tools/resources | GitHub MCP, PostgreSQL MCP |
| **MCP Client** | Connects host to server | Built into Kilo Code |

### MCP Capabilities

1. **Tools**: Functions the AI can call
   - `github.create_issue()`
   - `postgres.query()`
   - `filesystem.read_file()`

2. **Resources**: Data the AI can access
   - GitHub issues
   - Database tables
   - File system contents

3. **Prompts**: Pre-defined templates
   - Code review template
   - Bug report template
   - PR description template

---

## Installing MCP Servers

### Method 1: npm Installation

Many MCP servers are published as npm packages:

```bash
# GitHub MCP Server
npm install -g @modelcontextprotocol/server-github

# PostgreSQL MCP Server
npm install -g @modelcontextprotocol/server-postgres

# Filesystem MCP Server
npm install -g @modelcontextprotocol/server-filesystem

# Slack MCP Server
npm install -g @modelcontextprotocol/server-slack
```

### Method 2: Docker Installation

Some servers run better in containers:

```bash
# PostgreSQL MCP Server via Docker
docker run -d \
  -p 3000:3000 \
  -e DATABASE_URL="postgresql://user:pass@localhost:5432/mydb" \
  mcp/postgres-server

# Redis MCP Server via Docker
docker run -d \
  -p 3001:3001 \
  -e REDIS_URL="redis://localhost:6379" \
  mcp/redis-server
```

### Method 3: npx (No Installation)

Run servers directly with npx:

```bash
# Run GitHub MCP without installing
npx @modelcontextprotocol/server-github

# Run in Kilo Code config
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"]
    }
  }
}
```

---

## Configuring MCP in Kilo Code

### Configuration File

Create or edit `.kilocode/mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      },
      "enabled": true
    },
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem"],
      "args": ["/Users/paladm/projects"],
      "enabled": true
    },
    "postgres": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      },
      "enabled": true
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-slack"],
      "env": {
        "SLACK_BOT_TOKEN": "${SLACK_BOT_TOKEN}"
      },
      "enabled": false
    }
  }
}
```

### Environment Variables

Store secrets in `.env` file:

```bash
# .kilocode/.env
GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
SLACK_BOT_TOKEN=xoxb-xxxxxxxxxxxx-xxxxxxxxxxxx-xxxxxxxxxxxx
```

### Verify MCP Setup

```bash
# List configured servers
kilo-code mcp list

# Output:
Configured MCP Servers:
  ✓ github (connected)
  ✓ filesystem (connected)
  ✓ postgres (connected)
  ✗ slack (disabled)

# Test server connection
kilo-code mcp test github

# Output:
Testing GitHub MCP Server...
✓ Connection successful
✓ Available tools: 15
✓ Available resources: 8
```

---

## Using MCP Tools

### GitHub Integration

Once connected, Kilo Code can interact with GitHub:

```bash
User: "Create a GitHub issue for the login bug we found"

Kilo Code (via MCP):
I'll create a GitHub issue with the details we discussed.

Using GitHub MCP Server:
- Repository: myorg/myapp
- Title: Login fails with valid credentials
- Labels: bug, priority-high
- Assignee: @current-user

Creating issue...
✓ Issue #142 created: https://github.com/myorg/myapp/issues/142

Would you like me to:
1. Create a branch for this issue?
2. Add more details to the issue?
3. Notify the team on Slack?
```

**Available GitHub Tools:**

| Tool | Description |
|------|-------------|
| `github.create_issue` | Create new issue |
| `github.update_issue` | Update existing issue |
| `github.create_pull_request` | Create PR |
| `github.get_file` | Read file from repo |
| `github.search_code` | Search codebase |
| `github.list_issues` | List repository issues |
| `github.get_pull_request` | Get PR details |
| `github.create_comment` | Add comment to issue/PR |

### Filesystem Integration

Access files beyond the current project:

```bash
User: "Copy the config from our templates repo"

Kilo Code (via MCP):
I'll use the Filesystem MCP to access the templates repo.

Reading from: /Users/paladm/templates/webapp/config.json
Writing to: /Users/paladm/projects/myapp/config.json

✓ File copied successfully

Contents:
{
  "app": {
    "name": "MyApp",
    "version": "1.0.0"
  },
  "database": {
    "host": "localhost",
    "port": 5432
  }
}
```

**Available Filesystem Tools:**

| Tool | Description |
|------|-------------|
| `filesystem.read_file` | Read file contents |
| `filesystem.write_file` | Write file contents |
| `filesystem.list_directory` | List directory contents |
| `filesystem.create_directory` | Create new directory |
| `filesystem.delete_file` | Delete a file |
| `filesystem.move_file` | Move/rename file |
| `filesystem.search` | Search for files |

### Database Integration

Query databases directly:

`````bash
User: "Check how many users signed up this week"

Kilo Code (via MCP):
I'll query the PostgreSQL database.

Executing query:
```sql
SELECT COUNT(*) as new_users, 
       DATE(created_at) as signup_date
FROM users
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY DATE(created_at)
ORDER BY signup_date;
```

Results:
| signup_date | new_users |
|-------------|-----------|
| 2026-03-25 | 45 |
| 2026-03-26 | 52 |
| 2026-03-27 | 38 |
| 2026-03-28 | 67 |
| 2026-03-29 | 71 |
| 2026-03-30 | 89 |
| 2026-03-31 | 43 |

Total: 405 new users this week
`````

**Available Database Tools:**

| Tool | Description |
|------|-------------|
| `postgres.query` | Execute SQL query |
| `postgres.list_tables` | List all tables |
| `postgres.describe_table` | Show table schema |
| `postgres.insert` | Insert row |
| `postgres.update` | Update rows |
| `postgres.delete` | Delete rows |

---

## Building Custom MCP Servers

### Why Build Custom Servers?

While many MCP servers exist, you might need custom integrations for:
- Internal APIs
- Proprietary databases
- Company-specific tools
- Legacy systems

### MCP Server Structure

A basic MCP server in TypeScript:

```typescript
// my-custom-server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

// Create server instance
const server = new Server(
  {
    name: 'my-custom-server',
    version: '1.0.0',
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// Define available tools
server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: 'get_user_data',
        description: 'Get user data from internal API',
        inputSchema: {
          type: 'object',
          properties: {
            userId: {
              type: 'string',
              description: 'The user ID to fetch',
            },
          },
          required: ['userId'],
        },
      },
      {
        name: 'create_report',
        description: 'Generate a custom report',
        inputSchema: {
          type: 'object',
          properties: {
            reportType: {
              type: 'string',
              enum: ['daily', 'weekly', 'monthly'],
            },
            startDate: {
              type: 'string',
              description: 'Start date (ISO format)',
            },
            endDate: {
              type: 'string',
              description: 'End date (ISO format)',
            },
          },
          required: ['reportType'],
        },
      },
    ],
  };
});

// Handle tool calls
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === 'get_user_data') {
    const { userId } = args as { userId: string };
    
    // Fetch from internal API
    const response = await fetch(`https://api.internal/users/${userId}`);
    const data = await response.json();
    
    return {
      content: [
        {
          type: 'text',
          text: JSON.stringify(data, null, 2),
        },
      ],
    };
  }

  if (name === 'create_report') {
    const { reportType, startDate, endDate } = args as {
      reportType: string;
      startDate?: string;
      endDate?: string;
    };
    
    // Generate report logic here
    const report = await generateReport(reportType, startDate, endDate);
    
    return {
      content: [
        {
          type: 'text',
          text: report,
        },
      ],
    };
  }

  throw new Error(`Unknown tool: ${name}`);
});

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error('MCP Server running on stdio');
}

main().catch((error) => {
  console.error('Server error:', error);
  process.exit(1);
});
```

### Package Configuration

Create `package.json`:

```json
{
  "name": "@mycompany/mcp-custom-server",
  "version": "1.0.0",
  "type": "module",
  "bin": {
    "mcp-custom-server": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0"
  }
}
```

### Register Custom Server

Add to `.kilocode/mcp.json`:

```json
{
  "mcpServers": {
    "custom": {
      "command": "node",
      "args": ["/path/to/my-custom-server/dist/index.js"],
      "env": {
        "API_KEY": "${INTERNAL_API_KEY}"
      }
    }
  }
}
```

---

## MCP Resources

Resources let AI read external data without tools:

### GitHub Resources

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

```bash
User: "Show me the open issues in our repo"

Kilo Code:
Reading from github://myorg/myapp/issues?state=open

Found 12 open issues:

1. #142 - Login fails with valid credentials (High)
2. #141 - Mobile menu not responsive (Medium)
3. #140 - Add dark mode toggle (Low)
...
```

### Database Resources

```bash
User: "What tables do we have?"

Kilo Code:
Reading from postgres://localhost/mydb/tables

Available tables:
- users (15,234 rows)
- products (892 rows)
- orders (45,678 rows)
- reviews (23,456 rows)
```

---

## MCP Prompts

Prompts are reusable templates for common tasks:

### Creating a Prompt Server

```typescript
// prompts-server.ts
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import {
  ListPromptsRequestSchema,
  GetPromptRequestSchema,
} from '@modelcontextprotocol/sdk/types.js';

const server = new Server({
  name: 'prompts-server',
  version: '1.0.0',
}, {
  capabilities: {
    prompts: {},
  },
});

server.setRequestHandler(ListPromptsRequestSchema, async () => {
  return {
    prompts: [
      {
        name: 'code_review',
        description: 'Generate a code review for a pull request',
        arguments: [
          {
            name: 'pr_url',
            description: 'URL of the pull request',
            required: true,
          },
        ],
      },
      {
        name: 'bug_report',
        description: 'Create a structured bug report',
        arguments: [
          {
            name: 'description',
            description: 'Description of the bug',
            required: true,
          },
          {
            name: 'steps',
            description: 'Steps to reproduce',
            required: true,
          },
        ],
      },
    ],
  };
});

server.setRequestHandler(GetPromptRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  if (name === 'code_review') {
    return {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: `Please review this pull request: ${args.pr_url}

Focus on:
1. Code quality and best practices
2. Security vulnerabilities
3. Performance implications
4. Test coverage
5. Documentation

Provide specific suggestions with code examples.`,
          },
        },
      ],
    };
  }

  if (name === 'bug_report') {
    return {
      messages: [
        {
          role: 'user',
          content: {
            type: 'text',
            text: `Create a structured bug report:

Description: ${args.description}
Steps to Reproduce: ${args.steps}

Format the report with:
- Summary
- Environment
- Steps to Reproduce
- Expected Behavior
- Actual Behavior
- Screenshots (if applicable)
- Severity Assessment`,
          },
        },
      ],
    };
  }
});
```

### Using Prompts

`````bash
User: /prompt code_review https://github.com/myorg/myapp/pull/50

Kilo Code:
Using prompt: code_review

## Code Review for PR #50

### Summary
This PR adds user authentication with JWT tokens. Overall good implementation with a few concerns.

### Security Issues

#### 1. Hardcoded Secret (Critical)
Line 23: JWT secret is hardcoded
```typescript
// Bad
const secret = 'my-secret-key';
```
Fix: Use environment variable

### Code Quality

#### 2. Missing Error Handling
The login function doesn't handle database errors...

[Full review continues]
`````

---

## Common MCP Integrations

### Development Tools

| Server | Purpose | Install |
|--------|---------|---------|
| GitHub | Issues, PRs, code | `@modelcontextprotocol/server-github` |
| GitLab | GitLab API | `@modelcontextprotocol/server-gitlab` |
| Jira | Issue tracking | `@modelcontextprotocol/server-jira` |
| Linear | Project management | `@modelcontextprotocol/server-linear` |

### Databases

| Server | Purpose | Install |
|--------|---------|---------|
| PostgreSQL | PostgreSQL queries | `@modelcontextprotocol/server-postgres` |
| MySQL | MySQL queries | `@modelcontextprotocol/server-mysql` |
| MongoDB | MongoDB queries | `@modelcontextprotocol/server-mongodb` |
| Redis | Redis operations | `@modelcontextprotocol/server-redis` |

### Communication

| Server | Purpose | Install |
|--------|---------|---------|
| Slack | Slack messaging | `@modelcontextprotocol/server-slack` |
| Discord | Discord bot | `@modelcontextprotocol/server-discord` |
| Email | Send emails | `@modelcontextprotocol/server-email` |

### File & System

| Server | Purpose | Install |
|--------|---------|---------|
| Filesystem | File operations | `@modelcontextprotocol/server-filesystem` |
| Memory | Persistent memory | `@modelcontextprotocol/server-memory` |
| Shell | Command execution | `@modelcontextprotocol/server-shell` |

---

## Security Best Practices

### 1. Limit Server Permissions

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/allowed/path"],
      "readonly": true
    }
  }
}
```

### 2. Use Environment Variables for Secrets

```bash
# Good: Secrets in environment
export GITHUB_TOKEN=ghp_xxxx
kilo-code start

# Bad: Secrets in config file
{
  "github": {
    "token": "ghp_xxxx"  // Don't do this!
  }
}
```

### 3. Enable Only Needed Servers

```json
{
  "mcpServers": {
    "github": { "enabled": true },
    "postgres": { "enabled": true },
    "slack": { "enabled": false },  // Disable unused
    "shell": { "enabled": false }   // Dangerous, disable
  }
}
```

### 4. Audit Tool Usage

```bash
# Log MCP tool calls
kilo-code mcp logs

# Review recent activity
kilo-code mcp audit --last 24h
```

---

## Troubleshooting

### Issue: Server Won't Connect

```bash
# Check server status
kilo-code mcp status github

# Test connection
kilo-code mcp test github

# View logs
kilo-code mcp logs github
```

**Common fixes:**
1. Verify environment variables are set
2. Check network connectivity
3. Ensure server package is installed
4. Review server logs for errors

### Issue: Tool Not Available

```bash
# List available tools
kilo-code mcp tools github

# Refresh server
kilo-code mcp restart github
```

### Issue: Permission Denied

```bash
# Check filesystem permissions
ls -la /path/to/directory

# Update MCP config with correct path
{
  "filesystem": {
    "args": ["/correct/path"]
  }
}
```


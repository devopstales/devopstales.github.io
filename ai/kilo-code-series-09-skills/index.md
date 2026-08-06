---
title: Kilo Code Skills: Extending Agent Capabilities
url: https://devopstales.github.io/ai/kilo-code-series-09-skills/
date: 2026-04-02
---


In our previous post, we explored Advanced MCP Integration and how to connect your agent to external tools and data. But what if you need specialized expertise for specific tasks?

<!--more-->

{{< content "/filedir/kilo-code-series.html" >}}

This is where **Kilo Code Skills** come in. Skills are a lightweight, open format for extending your agent's capabilities without bloating the main context window.

## What are Kilo Code Skills?

**Skills** are modular knowledge packages that provide specialized expertise to Kilo Code agents. Think of them as "skill books" your agent can read when needed.

### Analogy: Generalist vs. Specialist

```bash
Without Skills:
┌───────────────────────────────────────┐
│         General AI Agent              │
│  - Knows a little about everything    │
│  - Jack of all trades, master of none │
│  - Context window fills up quickly    │
└───────────────────────────────────────┘

With Skills:
┌─────────────────────────────────────┐
│         General AI Agent            │
│  - Core reasoning capabilities      │
│  - Loads skills on demand           │
└─────────────────────────────────────┘
         │
    ┌────┴────┬────────────┬──────────┐
    ▼         ▼            ▼          ▼
┌────────┐ ┌────────┐ ┌────────┐ ┌─────────┐
│ Docker │ │ AWS    │ │ GraphQL│ │ Security│
│ Skill  │ │ Skill  │ │ Skill  │ │ Skill   │
└────────┘ └────────┘ └────────┘ └─────────┘
```

### Benefits of Skills

| Benefit | Description |
|---------|-------------|
| **Modular** | Load only what you need |
| **Reusable** | Share across projects |
| **Versioned** | Track changes over time |
| **Composable** | Combine multiple skills |
| **Lightweight** | Markdown-based, easy to edit |

---

## Skill Structure

A skill is a directory with a specific structure:

```bash
skills/
└── docker-expert/
    ├── skill.json      # Skill metadata
    ├── prompt.md       # Main system prompt
    ├── examples/       # Example interactions
    │   ├── example-1.md
    │   └── example-2.md
    ├── templates/      # Reusable templates
    │   └── dockerfile-template.md
    └── resources/      # Reference materials
        └── docker-cheatsheet.md
```

### skill.json

```json
{
  "name": "docker-expert",
  "version": "1.2.0",
  "description": "Expert knowledge for Docker containerization, Docker Compose, and container orchestration",
  "author": "Your Team",
  "tags": ["docker", "containers", "devops", "deployment"],
  "triggers": [
    "docker",
    "container",
    "dockerfile",
    "docker-compose",
    "kubernetes"
  ],
  "capabilities": [
    "Write Dockerfiles",
    "Debug container issues",
    "Optimize container images",
    "Create Docker Compose configurations",
    "Security best practices"
  ],
  "dependencies": [],
  "compatibility": {
    "minKiloVersion": "1.0.0"
  }
}
```

### prompt.md

The main instruction file:

`````markdown
# Skill: Docker Expert

## Role
You are a Docker and containerization expert. You help developers create efficient, secure, and production-ready container configurations.

## Expertise Areas

### 1. Dockerfile Creation
- Multi-stage builds for minimal images
- Security hardening (non-root users, minimal base images)
- Layer optimization for faster builds
- Proper .dockerignore usage

### 2. Docker Compose
- Multi-service orchestration
- Network configuration
- Volume management
- Environment variable handling

### 3. Container Debugging
- Analyzing container logs
- Resource limit tuning
- Network troubleshooting
- Health check configuration

### 4. Best Practices
- Image size optimization
- Security scanning
- CI/CD integration
- Production deployment patterns

## Response Format

When helping with Docker tasks:

1. **Analyze** the requirements
2. **Propose** a solution with explanation
3. **Provide** complete, working code
4. **Explain** key decisions
5. **Suggest** optimizations

## Examples

### Good Dockerfile Pattern

```dockerfile
# Multi-stage build for Node.js app
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/package.json ./

USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s CMD wget -q --spider http://localhost:3000/health || exit 1
CMD ["node", "dist/index.js"]
```

## Common Patterns

### Python Application
[Template for Python apps]

### Go Application
[Template for Go apps]

### Static Site
[Template for static sites]
`````

---

## Creating Your First Skill

### Step 1: Create Skill Directory

```bash
mkdir -p .kilocode/skills/docker-expert
```

### Step 2: Create skill.json

```bash
cat > .kilocode/skills/docker-expert/skill.json << 'EOF'
{
  "name": "docker-expert",
  "version": "1.0.0",
  "description": "Docker containerization expertise",
  "author": "DevOps Team",
  "tags": ["docker", "containers", "devops"],
  "triggers": ["docker", "container", "dockerfile"]
}
EOF
```

### Step 3: Create prompt.md

`````bash
cat > .kilocode/skills/docker-expert/prompt.md << 'EOF'
# Skill: Docker Expert

## Role
You are a Docker expert specializing in containerization best practices.

## Guidelines
1. Always use multi-stage builds when possible
2. Prefer Alpine base images for smaller size
3. Run as non-root user
4. Include health checks
5. Minimize layers for caching efficiency

## Common Patterns

### Node.js Application
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
USER node
EXPOSE 3000
CMD ["node", "index.js"]
```

### Python Application
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
USER nobody
CMD ["python", "app.py"]
```
EOF
`````

### Step 4: Register the Skill

Add to `.kilocode/skills.json`:

```json
{
  "skills": [
    {
      "path": "./skills/docker-expert",
      "enabled": true,
      "autoLoad": false
    }
  ]
}
```

---

## Using Skills

### Manual Activation

`````bash
User: /skill docker-expert
Skill loaded: docker-expert v1.0.0

User: Create a Dockerfile for our Express.js API

Docker Expert:
I'll create an optimized Dockerfile for your Express.js application.

## Dockerfile

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:20-alpine AS runner
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/src ./src
COPY --from=builder /app/package.json ./

USER node
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q --spider http://localhost:3000/health || exit 1

CMD ["node", "src/index.js"]
```

## Key Features
- Multi-stage build reduces final image size
- Non-root user for security
- Health check for container orchestration
- Optimized layer caching
`````

### Auto-Activation

Skills can auto-load based on triggers:

```json
{
  "skills": [
    {
      "path": "./skills/docker-expert",
      "enabled": true,
      "autoLoad": true,
      "triggers": ["dockerfile", "docker-compose", "container"]
    }
  ]
}
```

```bash
User: I need to create a docker-compose file for our microservices

Auto-detected: docker-expert skill loaded

Docker Expert:
I'll help you create a Docker Compose configuration...
```

### Skill Discovery

```bash
# List available skills
kilo-code skills list

# Output:
Available Skills:
  ✓ docker-expert (v1.0.0) - Docker containerization
  ✓ aws-helper (v2.1.0) - AWS services and deployment
  ✓ graphql-pro (v1.5.0) - GraphQL schema and resolvers
  ✓ security-audit (v1.0.0) - Security best practices

# Show skill details
kilo-code skills show docker-expert

# Output:
Skill: docker-expert
Version: 1.0.0
Author: DevOps Team
Description: Docker containerization expertise
Tags: docker, containers, devops
Triggers: docker, container, dockerfile
Capabilities:
  - Write Dockerfiles
  - Debug container issues
  - Optimize container images
```

---

## Advanced Skill Examples

### AWS Helper Skill

`.kilocode/skills/aws-helper/skill.json`:

```json
{
  "name": "aws-helper",
  "version": "2.1.0",
  "description": "AWS cloud services expertise including EC2, S3, Lambda, RDS, and CloudFormation",
  "author": "Cloud Team",
  "tags": ["aws", "cloud", "infrastructure", "devops"],
  "triggers": ["aws", "ec2", "s3", "lambda", "cloudformation", "terraform"],
  "capabilities": [
    "Design AWS architectures",
    "Write CloudFormation templates",
    "Configure IAM policies",
    "Set up CI/CD pipelines",
    "Cost optimization"
  ]
}
```

`.kilocode/skills/aws-helper/prompt.md`:

`````markdown
# Skill: AWS Helper

## Role
You are an AWS cloud architect with deep expertise in designing and implementing cloud infrastructure.

## Core Services Expertise

### Compute
- EC2 instance types and selection
- Lambda functions and triggers
- ECS/EKS container orchestration
- Elastic Beanstalk deployments

### Storage
- S3 buckets and lifecycle policies
- EBS volumes and snapshots
- EFS shared filesystems
- Glacier archival

### Database
- RDS (PostgreSQL, MySQL, Aurora)
- DynamoDB NoSQL
- ElastiCache (Redis, Memcached)
- Redshift data warehouse

### Networking
- VPC design and subnets
- Security groups and NACLs
- Load balancers (ALB, NLB)
- CloudFront CDN

### Security
- IAM roles and policies
- KMS encryption
- Secrets Manager
- WAF and Shield

## Infrastructure as Code

### CloudFormation Template Structure
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Application infrastructure

Parameters:
  Environment:
    Type: String
    Default: production

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16

Outputs:
  VPCId:
    Value: !Ref VPC
```

### Terraform Structure
```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "main-vpc"
  }
}
```

## Best Practices

1. **Security First**
   - Least privilege IAM policies
   - Encrypt data at rest and in transit
   - Enable CloudTrail logging
   - Use VPC endpoints

2. **Cost Optimization**
   - Right-size instances
   - Use Spot instances for workloads
   - Enable S3 lifecycle policies
   - Monitor with Cost Explorer

3. **High Availability**
   - Multi-AZ deployments
   - Auto Scaling groups
   - Health checks
   - Backup strategies
`````

### GraphQL Pro Skill

`.kilocode/skills/graphql-pro/skill.json`:

```json
{
  "name": "graphql-pro",
  "version": "1.5.0",
  "description": "GraphQL schema design, resolvers, and optimization techniques",
  "author": "Backend Team",
  "tags": ["graphql", "api", "schema", "resolvers"],
  "triggers": ["graphql", "schema", "resolver", "apollo", "relay"],
  "capabilities": [
    "Design GraphQL schemas",
    "Implement resolvers",
    "Optimize N+1 queries",
    "Set up subscriptions",
    "Security and authorization"
  ]
}
```

`.kilocode/skills/graphql-pro/prompt.md`:

`````markdown
# Skill: GraphQL Pro

## Role
You are a GraphQL expert specializing in schema design, efficient resolvers, and API optimization.

## Schema Design Principles

### Type Definitions
```graphql
type User {
  id: ID!
  email: String!
  name: String
  posts: [Post!]!
  createdAt: DateTime!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
  publishedAt: DateTime
}

type Query {
  user(id: ID!): User
  users(limit: Int, offset: Int): [User!]!
  post(id: ID!): Post
  posts(authorId: ID): [Post!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

input CreateUserInput {
  email: String!
  name: String
  password: String!
}
```

### Resolver Pattern
```typescript
const resolvers = {
  Query: {
    user: async (_, { id }, { dataSources }) => {
      return dataSources.userAPI.getUserById(id);
    },
  },
  User: {
    posts: async (user, _, { dataSources }) => {
      return dataSources.postAPI.getPostsByAuthor(user.id);
    },
  },
};
```

### N+1 Query Solution (DataLoader)
```typescript
import DataLoader from 'dataloader';

const userLoader = new DataLoader(async (userIds) => {
  const users = await db.user.findMany({
    where: { id: { in: userIds } },
  });
  return userIds.map(id => users.find(u => u.id === id));
});

// In resolver
const user = await userLoader.load(authorId);
```

## Best Practices

1. **Schema Design**
   - Use singular names for single objects
   - Use plural names for lists
   - Include pagination arguments
   - Document with descriptions

2. **Performance**
   - Implement DataLoader for batching
   - Add query depth limiting
   - Use query complexity analysis
   - Cache resolver results

3. **Security**
   - Implement authentication
   - Add authorization checks
   - Validate input thoroughly
   - Rate limit queries
`````

---

## Skill Templates

### Template: API Integration Skill

`````markdown
# Skill: [API Name] Integration

## Role
You are an expert in [API Name] integration.

## Authentication
- API key configuration
- OAuth 2.0 flow
- Token refresh handling

## Common Operations
- [Operation 1]: Description and example
- [Operation 2]: Description and example
- [Operation 3]: Description and example

## Error Handling
- Rate limiting (429)
- Authentication errors (401)
- Not found (404)
- Server errors (500)

## Code Examples

### Initialize Client
```javascript
const client = new APIClient({
  apiKey: process.env.API_KEY,
  baseURL: 'https://api.example.com',
});
```

### Make Request
```javascript
const result = await client.getData({
  param1: 'value',
  param2: 123,
});
```
`````

### Template: Testing Skill

`````markdown
# Skill: Testing Expert

## Role
You are a testing expert specializing in unit, integration, and E2E tests.

## Testing Frameworks
- Jest (JavaScript/TypeScript)
- pytest (Python)
- RSpec (Ruby)
- JUnit (Java)

## Test Structure
```typescript
describe('UserService', () => {
  describe('createUser', () => {
    it('should create user with valid data', async () => {
      // Arrange
      const userData = { email: 'test@example.com' };
      
      // Act
      const user = await userService.create(userData);
      
      // Assert
      expect(user.email).toBe(userData.email);
      expect(user.id).toBeDefined();
    });
    
    it('should throw error for duplicate email', async () => {
      // Arrange
      await userService.create({ email: 'existing@example.com' });
      
      // Act & Assert
      await expect(
        userService.create({ email: 'existing@example.com' })
      ).rejects.toThrow('Email already exists');
    });
  });
});
```

## Best Practices
- One assertion per test (when possible)
- Descriptive test names
- Arrange-Act-Assert pattern
- Mock external dependencies
- Test edge cases
`````

---

## Sharing Skills

### Publishing to Skill Registry

Create a `skill-registry.json` in a shared location:

```json
{
  "registry": "https://skills.kilocode.dev",
  "skills": [
    {
      "name": "docker-expert",
      "version": "1.2.0",
      "repository": "https://github.com/myorg/kilo-skills",
      "path": "skills/docker-expert"
    },
    {
      "name": "aws-helper",
      "version": "2.1.0",
      "repository": "https://github.com/myorg/kilo-skills",
      "path": "skills/aws-helper"
    }
  ]
}
```

### Installing Shared Skills

```bash
# Install from registry
kilo-code skills install docker-expert

# Install from GitHub
kilo-code skills install github:myorg/kilo-skills/docker-expert

# Install from local path
kilo-code skills install ./shared-skills/security-audit
```

### Creating a Skill Package

```bash
# Package skill for distribution
kilo-code skills package ./skills/docker-expert

# Output: docker-expert-1.2.0.tar.gz

# Publish to registry
kilo-code skills publish docker-expert-1.2.0.tar.gz
```

---

## Skill Composition

Skills can depend on other skills:

```json
{
  "name": "fullstack-deploy",
  "version": "1.0.0",
  "description": "Full-stack deployment expertise",
  "dependencies": [
    "docker-expert@^1.0.0",
    "aws-helper@^2.0.0",
    "graphql-pro@^1.0.0"
  ]
}
```

When loaded, all dependencies are activated:

```bash
User: /skill fullstack-deploy

Loading skills:
  ✓ docker-expert v1.2.0
  ✓ aws-helper v2.1.0
  ✓ graphql-pro v1.5.0
  ✓ fullstack-deploy v1.0.0

All skills loaded successfully!
```

---

## Best Practices

### 1. Keep Skills Focused

```bash
Good:  docker-expert (only Docker)
       aws-helper (only AWS)

Bad:   devops-master (Docker + AWS + K8s + Terraform + everything)
```

### 2. Use Clear Triggers

```json
{
  "triggers": ["dockerfile", "docker-compose", "container"]
}
```

### 3. Include Examples

```markdown
## Examples

### Basic Dockerfile
[example code]

### Production Dockerfile
[example code]

### Multi-stage Build
[example code]
```

### 4. Version Your Skills

```json
{
  "version": "1.2.0"
}
```

Follow semantic versioning:
- MAJOR: Breaking changes
- MINOR: New features
- PATCH: Bug fixes

### 5. Document Dependencies

```json
{
  "dependencies": [
    "docker-expert@^1.0.0"
  ]
}
```

### 6. Test Your Skills

```bash
# Test skill loading
kilo-code skills test docker-expert

# Test skill responses
kilo-code skills test docker-expert --prompt "Create a Dockerfile"
```

---

## Troubleshooting

### Issue: Skill Not Loading

```bash
# Check skill path
kilo-code skills show docker-expert

# Verify skill.json
cat .kilocode/skills/docker-expert/skill.json

# Check for syntax errors
kilo-code skills validate docker-expert
```

### Issue: Skill Not Triggering

**Check:**
1. Triggers are defined in skill.json
2. Auto-load is enabled
3. Trigger words match exactly

```json
{
  "triggers": ["docker", "container"],
  "autoLoad": true
}
```

### Issue: Skill Conflicts

When multiple skills have same triggers:

```json
{
  "skills": [
    {
      "path": "./skills/docker-expert",
      "priority": 1
    },
    {
      "path": "./skills/devops-general",
      "priority": 2
    }
  ]
}
```

Higher priority loads first.


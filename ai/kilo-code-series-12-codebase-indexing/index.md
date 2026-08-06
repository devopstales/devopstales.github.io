---
title: Kilo Code: Mastering Codebase Indexing for Semantic AI Search
url: https://devopstales.github.io/ai/kilo-code-series-12-codebase-indexing/
date: 2026-04-05
---


One of the most powerful features of Kilo Code is its ability to understand your **entire codebase semantically**—not just through keyword matching, but by grasping the actual meaning and relationships in your code.

<!--more-->

{{< content "/filedir/kilo-code-series.html" >}}

![Advanced codebase indexing with semantic search](/img/coding-terminal.webp)
*Semantic search finds code by meaning, not just keywords*

This is powered by **Codebase Indexing**, a feature that transforms how AI interacts with your repository. In this post, we'll dive deep into how codebase indexing works, why it's essential for effective AI development, and how to configure it for optimal performance.

## The Problem: Traditional Search Falls Short

Before we understand codebase indexing, let's look at the problem it solves.

### Keyword Search Limitations

Traditional code search tools (grep, VS Code search, etc.) rely on **exact text matching**:

```bash
Search: "user authentication"

Results:
✓ Files containing "user authentication"
✗ Files about "login flow" (no match)
✗ Files about "session validation" (no match)
✗ Files about "identity verification" (no match)
```

**The problem?** These are all related concepts, but keyword search can't find them because they use different words.

### Context Window Limitations

Even if you could search perfectly, LLMs have **context window limits**:

```bash
┌─────────────────────────────────────────────────────┐
│              LLM Context Window (e.g., 100K tokens) │
│                                                     │
│ ┌─────────────────────────────────────────────────┐ │
│ │ Your entire codebase: 500K+ tokens              │ │
│ │ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ │ │
│ │ │ File A  │ │ File B  │ │ File C  │ │  ...    │ │ │
│ │ └─────────┘ └─────────┘ └─────────┘ └─────────┘ │ │
│ │                                                 │ │
│ │    Doesn't fit in context window!               │ │
│ └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

You can't feed your entire repository to the AI for every question.

## The Solution: Codebase Indexing

**Codebase Indexing** solves both problems by:

1. **Creating semantic embeddings** of your code (understanding meaning, not just text)
2. **Storing embeddings in a vector database** for fast similarity search
3. **Retrieving relevant code** based on your query's meaning
4. **Feeding only relevant context** to the AI

### How It Works

```bash
┌────────────────────────────────────────────────────────────┐
│              Kilo Code Codebase Indexing Pipeline          │
│                                                            │
│  STEP 1: Indexing (One-time + Incremental Updates)         │
│  ───────────────────────────────────────────────────────── │
│                                                            │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Your       │───>│   Code       │───>│   Vector     │  │
│  │   Codebase   │    │   Embedder   │    │   Database   │  │
│  │              │    │              │    │   (Qdrant)   │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│                                                            │
│  STEP 2: Query Time (Every AI Request)                     │
│  ────────────────────────────────────────────────          │
│                                                            │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   User       │───>│   Query      │───>│   Vector     │  │
│  │   Question   │    │   Embedder   │    │   Search     │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│                              │                    │        │
│                              │                    ▼        │
│                              │         ┌──────────────┐    │
│                              │         │   Relevant   │    │
│                              └────────>│   Code       │    │
│                                        │   Snippets   │    │
│                                        └──────┬───────┘    │
│                                               │            │
│                                               ▼            │
│                                        ┌──────────────┐    │
│                                        │   AI Agent   │    │
│                                        │   (with      │    │
│                                        │   context)   │    │
│                                        └──────────────┘    │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

## Key Benefits

### 1. **Semantic Understanding**

The AI finds code based on **meaning**, not just keywords:

```bash
Query: "How do users log in?"

Results include:
✓ LoginController.authenticate()
✓ SessionValidator.validate()
✓ IdentityService.verifyCredentials()
✓ AuthMiddleware.checkToken()
```

### 2. **Cross-File Context**

The AI understands relationships **across files**:

```bash
Query: "Where is the database connection configured?"

Results include:
✓ config/database.ts (connection setup)
✓ src/lib/db.ts (connection pool)
✓ src/repositories/user.ts (usage example)
✓ .env.example (configuration variables)
```

### 3. **Faster, More Accurate Responses**

By providing only relevant context:

- **Reduced token usage** = lower costs
- **Less noise** = better AI responses
- **Faster processing** = quicker answers

### 4. **Works with Large Codebases**

Indexing scales to **hundreds of thousands of lines** of code:

| Codebase Size | Index Size | Query Time |
|---------------|------------|------------|
| 10K lines | ~50 MB | < 100ms |
| 100K lines | ~500 MB | < 200ms |
| 1M lines | ~5 GB | < 500ms |

## Architecture Components

Kilo Code's indexing system consists of three main components:

### 1. **Embedding Model**

Converts code into vector representations:

```bash
Code: "function authenticate(user, password) { ... }"
       │
       ▼
Embedding: [0.123, -0.456, 0.789, ..., -0.321]
           (1024-dimensional vector)
```

**Recommended models:**

| Model | Dimensions | Speed | Accuracy | Best For |
|-------|------------|-------|----------|----------|
| `nomic-embed-text` | 768 | Fast | Good | General purpose |
| `mxbai-embed-large` | 1024 | Medium | Better | Large codebases |
| `bge-m3` | 1024 | Medium | Best | Multi-language |

### 2. **Vector Database**

Stores and searches embeddings efficiently:

**Kilo Code supports:**

- **Qdrant** (Recommended) - Fast, scalable, easy to self-host
- **Chroma** - Simple, good for local development
- **Pinecone** - Managed service, no infrastructure

### 3. **Index Manager**

Handles incremental updates and cache invalidation:

```bash
File changed → Detect → Re-embed → Update index
                                    │
                                    ▼
                            No full re-index needed!
```

## Configuration Guide

### Step 1: Choose Your Stack

**For most users, we recommend:**

```json
{
  "indexing": {
    "enabled": true,
    "provider": "qdrant",
    "embeddingModel": "nomic-embed-text",
    "maxContextTokens": 100000
  }
}
```

### Step 2: Set Up Qdrant

**Option A: Local Qdrant (Recommended for Development)**

```bash
# Run Qdrant with Docker
docker run -d \
  -p 6333:6333 \
  -p 6334:6334 \
  -v qdrant_storage:/qdrant/storage \
  qdrant/qdrant
```

**Option B: Self-Hosted Qdrant**

```bash
# Install Qdrant
curl -fsSL https://qdrant.tech/install.sh | bash

# Start Qdrant
./qdrant
```

**Option C: Qdrant Cloud**

```bash
# Sign up at https://cloud.qdrant.io
# Get your API key and endpoint
```

### Step 3: Configure Kilo Code

Create or update `.kilocode/indexing.json`:

```json
{
  "indexing": {
    "enabled": true,
    "provider": "qdrant",
    "qdrant": {
      "url": "http://localhost:6333",
      "apiKey": null,
      "collectionName": "kilocode-codebase"
    },
    "embeddingModel": "nomic-embed-text",
    "embeddingProvider": "ollama",
    "ollama": {
      "url": "http://localhost:11434",
      "model": "nomic-embed-text"
    },
    "maxContextTokens": 100000,
    "includePatterns": [
      "**/*.ts",
      "**/*.tsx",
      "**/*.js",
      "**/*.jsx",
      "**/*.py",
      "**/*.go",
      "**/*.rs",
      "**/*.java",
      "**/*.md"
    ],
    "excludePatterns": [
      "node_modules/**",
      "dist/**",
      "build/**",
      "vendor/**",
      ".git/**",
      "**/*.min.js",
      "**/*.bundle.js",
      "**/test/**",
      "**/*.test.ts",
      "**/*.spec.ts"
    ],
    "chunkSize": 512,
    "chunkOverlap": 50,
    "incrementalIndexing": true,
    "watchMode": true
  }
}
```

### Step 4: Build the Index

```bash
# In your project directory
kilo-code index build

# Or through the Kilo UI
# Command Palette → Kilo Code: Rebuild Index
```

**Initial indexing progress:**

```bash
🔍 Kilo Code Indexing Service

Scanning repository...
✓ Found 1,247 files
✓ Filtering excluded patterns...
✓ 892 files to index

Creating embeddings...
[████████████████░░░░] 75% (669/892 files)
Estimated time remaining: 2 minutes

Indexing complete!
✓ 892 files indexed
✓ 12,453 chunks created
✓ Index size: 487 MB
```

## Advanced Configuration

### Multi-Language Projects

```json
{
  "indexing": {
    "includePatterns": [
      "**/*.ts",      // TypeScript
      "**/*.py",      // Python
      "**/*.go",      // Go
      "**/*.rs",      // Rust
      "**/*.sql",     // SQL
      "**/*.graphql", // GraphQL
      "**/*.md"       // Documentation
    ],
    "languageWeights": {
      "typescript": 1.0,
      "python": 1.0,
      "go": 1.0,
      "sql": 0.8,
      "markdown": 0.5
    }
  }
}
```

### Performance Tuning

```json
{
  "indexing": {
    "chunkSize": 512,        // Larger = fewer chunks, less precise
    "chunkOverlap": 50,      // Higher = better context, more storage
    "batchSize": 32,         // Higher = faster indexing, more memory
    "parallelWorkers": 4,    // Match your CPU cores
    "cacheEmbeddings": true  // Avoid re-embedding unchanged files
  }
}
```

### Storage Optimization

```json
{
  "indexing": {
    "storagePath": ".kilocode/index",
    "compressionEnabled": true,
    "compressionLevel": 6,
    "retentionDays": 90,
    "cleanupOnStartup": true
  }
}
```

## Usage Examples

### Example 1: Finding Implementation Patterns

```bash
Query: "How do we handle API errors in this codebase?"

Kilo Code (with indexing):
Based on your codebase, here's how API errors are handled:

1. **Error Wrapper** (`src/lib/api-error.ts`)
   - Custom ApiError class with status codes
   - Used in 47 files across the project

2. **Error Handler Middleware** (`src/middleware/error-handler.ts`)
   - Centralized error handling for Express routes
   - Converts errors to consistent JSON responses

3. **Common Patterns Found:**
   - try/catch with ApiError wrapping
   - Error logging with correlation IDs
   - Client-friendly error messages

Would you like me to show examples from specific files?
```

### Example 2: Understanding Architecture

```bash
Query: "Explain the authentication flow"

Kilo Code (with indexing):
Based on your codebase, here's the authentication flow:

1. **Entry Point** (`src/routes/auth.ts`)
   - POST /auth/login endpoint
   - Validates credentials via AuthService

2. **Service Layer** (`src/services/auth-service.ts`)
   - Calls UserRepository.findByEmail()
   - Validates password with bcrypt
   - Generates JWT token

3. **Token Validation** (`src/middleware/auth-middleware.ts`)
   - Verifies JWT signature
   - Attaches user to request context
   - Used by 23 protected routes

4. **Session Management** (`src/services/session-service.ts`)
   - Tracks active sessions in Redis
   - Handles token refresh

Here's a diagram of the flow...
```

### Example 3: Refactoring Assistance

```bash
Query: "I need to add rate limiting to all API endpoints"

Kilo Code (with indexing):
Based on your codebase structure, here's how to add rate limiting:

**Current Architecture:**
- Express.js with middleware pattern
- 47 API routes across 12 route files
- Redis already used for sessions

**Recommended Approach:**

1. **Create Rate Limit Middleware**
   - Use express-rate-limit with Redis store
   - Reuse existing Redis connection

2. **Apply to Routes**
   - Add to `src/middleware/index.ts`
   - Import in route files

3. **Configuration**
   - Add rate limit config to `src/config/rate-limit.ts`
   - Environment variables for thresholds

Here's the implementation...
```

## Troubleshooting

### Issue: Indexing is Slow

**Solutions:**

```json
{
  "indexing": {
    "parallelWorkers": 8,      // Increase for more CPU cores
    "batchSize": 64,           // Larger batches
    "excludePatterns": [       // Exclude more files
      "**/test/**",
      "**/*.test.*",
      "**/*.spec.*",
      "**/fixtures/**",
      "**/mocks/**"
    ]
  }
}
```

### Issue: Search Results Are Irrelevant

**Solutions:**

```json
{
  "indexing": {
    "chunkSize": 256,          // Smaller chunks for precision
    "embeddingModel": "bge-m3", // Better accuracy model
    "similarityThreshold": 0.7  // Higher threshold
  }
}
```

### Issue: Index is Too Large

**Solutions:**

```json
{
  "indexing": {
    "chunkSize": 1024,         // Larger chunks
    "excludePatterns": [       // Exclude more
      "**/*.md",
      "docs/**",
      "examples/**"
    ],
    "compressionEnabled": true,
    "retentionDays": 30
  }
}
```

### Issue: Qdrant Connection Fails

**Check:**

```bash
# Verify Qdrant is running
curl http://localhost:6333

# Check Kilo Code config
kilo-code config get indexing.qdrant.url

# Test connection
kilo-code index test-connection
```

## Best Practices

### 1. **Index Only What You Need**

```json
{
  "includePatterns": ["**/*.ts", "**/*.tsx"],
  "excludePatterns": ["**/node_modules/**", "**/dist/**"]
}
```

### 2. **Use Incremental Indexing**

```json
{
  "incrementalIndexing": true,
  "watchMode": true
}
```

### 3. **Clean Index Periodically**

```bash
# Rebuild index monthly
kilo-code index rebuild

# Or clean old indexes
kilo-code index cleanup --older-than 30d
```

### 4. **Monitor Index Health**

```bash
# Check index status
kilo-code index status

# View index statistics
kilo-code index stats
```

### 5. **Use Appropriate Embedding Model**

| Use Case | Recommended Model |
|----------|-------------------|
| General purpose | `nomic-embed-text` |
| Multi-language | `bge-m3` |
| Maximum accuracy | `mxbai-embed-large` |
| Limited resources | `all-minilm` |

## Performance Benchmarks

### Index Size by Codebase

| Lines of Code | Index Size | Build Time |
|---------------|------------|------------|
| 10K | 50 MB | 30 seconds |
| 50K | 250 MB | 2 minutes |
| 100K | 500 MB | 4 minutes |
| 500K | 2.5 GB | 15 minutes |
| 1M | 5 GB | 30 minutes |

### Query Performance

| Index Size | P50 Latency | P95 Latency | P99 Latency |
|------------|-------------|-------------|-------------|
| 100K lines | 50ms | 100ms | 200ms |
| 500K lines | 100ms | 200ms | 400ms |
| 1M lines | 200ms | 400ms | 800ms |

## Conclusion

Codebase Indexing is the **foundation** that makes Kilo Code's AI truly useful for large projects. Without it, the AI is working blind—guessing at your code structure and missing critical context.

**Key takeaways:**

- ✅ Indexing enables semantic search (meaning, not keywords)
- ✅ Qdrant + Ollama is the recommended stack
- ✅ Configure exclude patterns to reduce index size
- ✅ Incremental indexing keeps index fresh efficiently
- ✅ Proper indexing = better AI responses + lower costs

With codebase indexing properly configured, your AI assistant transforms from a generic coding helper into an expert on **your** codebase.


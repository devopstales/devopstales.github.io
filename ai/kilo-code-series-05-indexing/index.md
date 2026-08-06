---
title: Kilo Code: Codebase Indexing with Nomic and Qdrant
url: https://devopstales.github.io/ai/kilo-code-series-05-indexing/
date: 2026-03-29
---


In our previous post, we explored the different modes of Kilo Code. While these modes are powerful, their effectiveness depends on the quality of the context they can access.

<!--more-->

{{< content "/filedir/kilo-code-series.html" >}}

![Semantic codebase indexing visualization](/img/ai-brain.webp)
*Codebase indexing enables AI to understand your entire project semantically*

This is where **Codebase Indexing** comes in. Instead of just searching for keywords, Kilo Code can perform **semantic search**, allowing the agent to understand the "meaning" of your code across your entire project.

## Why Codebase Indexing Matters

Traditional search tools like `grep` or VS Code's search are limited to exact text matching:

```bash
# grep finds exact matches only
grep -r "validateEmail" src/

# Misses: validate_email, emailValidation, checkEmail, etc.
```

With **semantic search**, Kilo Code understands that these are related concepts:

```
User query: "How do we validate email addresses?"

Semantic search finds:
- validateEmail() function
- email_validation.py module
- EmailValidator class
- check_email_format() utility
```

This is achieved through **vector embeddings** - numerical representations of code that capture semantic meaning.

---

## Architecture Overview

Kilo Code's indexing system consists of three components:

```bash
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Code Files    │────▶│  Embedding      │────▶│   Qdrant        │
│   (Your Repo)   │     │  Model          │     │   Vector DB     │
│                 │     │  (nomic-embed)  │     │                 │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌─────────────────┐
                        │   Ollama        │
                        │   (Local AI)    │
                        └─────────────────┘
```

**Components:**

| Component | Purpose | Options |
|-----------|---------|---------|
| **Embedding Model** | Converts code to vectors | nomic-embed-text, mxbai-embed-large |
| **Vector Database** | Stores and searches embeddings | Qdrant (local), Chroma, Weaviate |
| **Runtime** | Runs embedding model locally | Ollama, LM Studio |

---

## Step 1: Install Ollama

Ollama is the easiest way to run embedding models locally.

### macOS

```bash
# Install with Homebrew
brew install ollama

# Or download from https://ollama.com
```

### Linux

```bash
# Official install script
curl -fsSL https://ollama.com/install.sh | sh

# Or use package manager
sudo apt install ollama  # Ubuntu/Debian
sudo dnf install ollama  # Fedora/RHEL
```

### Windows

```powershell
# Download installer from https://ollama.com
# Or use winget
winget install Ollama.Ollama
```

### Verify Installation

```bash
ollama --version
# Output: ollama version 0.5.x

ollama serve &
# Starts the Ollama server (port 11434)
```

---

## Step 2: Download Embedding Model

Kilo Code recommends **nomic-embed-text** for code indexing:

```bash
# Pull the embedding model
ollama pull nomic-embed-text

# Alternative: mxbai-embed-large (slightly better for code)
ollama pull mxbai-embed-large
```

### Model Comparison

| Model | Dimensions | Max Tokens | Speed | Quality |
|-------|------------|------------|-------|---------|
| `nomic-embed-text` | 768 | 8192 | Fast | Good |
| `mxbai-embed-large` | 1024 | 512 | Medium | Better |
| `all-minilm` | 384 | 512 | Very Fast | Basic |

**Recommendation:** Use `nomic-embed-text` for most projects. It offers the best balance of speed and quality.

### Test the Model

```bash
# Generate embeddings for test text
curl http://localhost:11434/api/embeddings -d '{
  "model": "nomic-embed-text",
  "prompt": "function validateEmail(email) { return email.includes(\"@\"); }"
}'

# Returns a 768-dimensional vector
{"embedding": [0.0234, -0.0156, 0.0891, ...]}
```

---

## Step 3: Install Qdrant

Qdrant is a vector similarity search engine that stores your code embeddings.

### Option A: Docker (Recommended)

```bash
# Pull Qdrant image
docker pull qdrant/qdrant

# Run Qdrant
docker run -d \
  -p 6333:6333 \
  -p 6334:6334 \
  -v $(pwd)/qdrant_storage:/qdrant/storage \
  qdrant/qdrant
```

**Ports:**
- `6333`: REST API
- `6334`: gRPC API (faster for large datasets)

### Option B: Local Binary

**macOS:**
```bash
brew install qdrant
qdrant
```

**Linux:**
```bash
# Download binary
wget https://github.com/qdrant/qdrant/releases/latest/download/qdrant-x86_64-unknown-linux-gnu.tar.gz
tar -xzf qdrant-*.tar.gz
./qdrant
```

**Windows:**
```powershell
# Download from GitHub releases
# https://github.com/qdrant/qdrant/releases
```

### Verify Qdrant

```bash
# Check if Qdrant is running
curl http://localhost:6333/

# Expected response:
{"title":"qdrant - vector search engine","version":"1.x.x"}
```

---

## Step 4: Configure Kilo Code Indexing

### Create Configuration File

Create `.kilocode/indexing.json` in your project root:

```json
{
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
      "collectionName": "my-project-codebase"
    },
    "chunking": {
      "strategy": "code-aware",
      "chunkSize": 512,
      "overlap": 64
    },
    "filters": {
      "include": [
        "**/*.js",
        "**/*.ts",
        "**/*.py",
        "**/*.go",
        "**/*.rs",
        "**/*.java",
        "**/*.md"
      ],
      "exclude": [
        "**/node_modules/**",
        "**/dist/**",
        "**/build/**",
        "**/*.min.js",
        "**/*.bundle.js",
        "**/vendor/**",
        "**/.git/**",
        "**/test/**",
        "**/*.test.*",
        "**/*.spec.*"
      ]
    }
  }
}
```

### Configuration Options Explained

**Embedding Settings:**

| Option | Description | Default |
|--------|-------------|---------|
| `model` | Ollama embedding model | `nomic-embed-text` |
| `provider` | Embedding provider | `ollama` |
| `endpoint` | Ollama API endpoint | `http://localhost:11434` |

**Vector Store Settings:**

| Option | Description | Default |
|--------|-------------|---------|
| `provider` | Vector database | `qdrant` |
| `endpoint` | Qdrant API endpoint | `http://localhost:6333` |
| `collectionName` | Collection name | `{project-name}` |

**Chunking Settings:**

| Option | Description | Default |
|--------|-------------|---------|
| `strategy` | Chunking approach | `code-aware` |
| `chunkSize` | Tokens per chunk | `512` |
| `overlap` | Overlap between chunks | `64` |

### Chunking Strategies

**Code-Aware (Recommended):**
Respects code boundaries (functions, classes):

```javascript
// Keeps function intact
function calculateTotal(items) {
  // Entire function in one chunk
  return items.reduce((sum, item) => sum + item.price, 0);
}
```

**Fixed-Size:**
Splits at exact token count:

```bash
Chunk 1: function calculateTotal(items) {
Chunk 2:   return items.reduce((sum, item) =>
Chunk 3:     sum + item.price, 0);
}
```

**Line-Based:**
Splits at line boundaries:

```bash
Chunk 1: Lines 1-20
Chunk 2: Lines 21-40
```

---

## Step 5: Build the Index

### CLI Command

```bash
# Build index for current directory
kilo-code index build

# Build with verbose output
kilo-code index build --verbose

# Rebuild from scratch
kilo-code index rebuild

# Build specific paths only
kilo-code index build src/ lib/
```

### Expected Output

```bash
$ kilo-code index build

🔍 Kilo Code Indexing Service
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

📁 Scanning project: /Users/paladm/my-project
📊 Found 247 files (1.2 MB total)
⚙️  Applying filters...
   ✓ Included: 189 files
   ✗ Excluded: 58 files (node_modules, dist, tests)

🔄 Chunking code...
   Created 1,247 chunks (avg 423 tokens)

🧮 Generating embeddings...
   [████████████████████] 100% | 1,247/1,247 chunks
   Model: nomic-embed-text
   Time: 2m 34s

💾 Storing in Qdrant...
   Collection: my-project-codebase
   Vectors: 1,247
   Dimensions: 768

✅ Indexing complete!
   Search ready in ~50ms
```

### Index Status

```bash
# Check index status
kilo-code index status

# Output:
Index Status: Ready
Collection: my-project-codebase
Documents: 1,247 chunks
Last Updated: 2026-03-29 22:30:00
Size: 45.2 MB (vectors + metadata)
```

---

## Step 6: Use Semantic Search

### In Kilo Code IDE

**Method 1: Search Panel**
1. Open the Kilo Code sidebar
2. Click the "Search" tab
3. Type your query in natural language

**Method 2: Chat Integration**
```
User: "Where is the email validation logic?"

Kilo Code (with indexing):
I found several relevant files:

1. src/utils/emailValidator.js (95% match)
   - validateEmail() function
   - checkEmailDomain() function
   
2. src/services/userService.js (78% match)
   - Uses email validation during registration
   
3. src/middleware/validation.js (65% match)
   - Email format middleware

Would you like me to show the code from any of these files?
```

### In Kilo Code CLI

```bash
# Semantic search
kilo-code search "user authentication logic"

# Search with file filter
kilo-code search "database connection" --filter "*.py"

# Search with limit
kilo-code search "API endpoints" --limit 5
```

### Search Results Format

```bash
$ kilo-code search "password hashing"

🔍 Search Results for: "password hashing"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. src/auth/password.js (Score: 0.94)
   ─────────────────────────────────────
   Location: Lines 15-42
   
   function hashPassword(plainPassword) {
     const salt = bcrypt.genSaltSync(12);
     return bcrypt.hashSync(plainPassword, salt);
   }
   
   function verifyPassword(plain, hashed) {
     return bcrypt.compareSync(plain, hashed);
   }
   ─────────────────────────────────────

2. src/models/User.js (Score: 0.87)
   ─────────────────────────────────────
   Location: Lines 28-35 (pre-save hook)
   
   userSchema.pre('save', function(next) {
     if (this.isModified('password')) {
       this.password = hashPassword(this.password);
     }
     next();
   });
   ─────────────────────────────────────

3. docs/security.md (Score: 0.72)
   ─────────────────────────────────────
   Location: Section 3.2
   
   ## Password Storage
   We use bcrypt with 12 salt rounds for
   password hashing. Never store plain text.
   ─────────────────────────────────────
```

---

## Advanced Configuration

### Multi-Project Indexing

For monorepos or multiple related projects:

```json
{
  "indexing": {
    "collections": [
      {
        "name": "frontend",
        "path": "./packages/frontend",
        "filters": { "include": ["**/*.tsx", "**/*.ts"] }
      },
      {
        "name": "backend",
        "path": "./packages/backend",
        "filters": { "include": ["**/*.py"] }
      },
      {
        "name": "shared",
        "path": "./packages/shared",
        "filters": { "include": ["**/*.ts", "**/*.json"] }
      }
    ]
  }
}
```

### Incremental Indexing

For large codebases, enable incremental updates:

```json
{
  "indexing": {
    "incremental": true,
    "watchMode": true,
    "debounceMs": 5000,
    "batchSize": 100
  }
}
```

**How it works:**
1. Initial full index build
2. Watch for file changes
3. Re-index only modified files
4. Update vector database incrementally

### Custom Metadata

Add custom metadata to improve search:

```json
{
  "indexing": {
    "metadata": {
      "includeGitBlame": true,
      "includeFileStats": true,
      "includeDependencies": true,
      "customFields": {
        "team": "platform",
        "service": "api-gateway"
      }
    }
  }
}
```

---

## Troubleshooting

### Issue: Ollama Connection Failed

```bash
# Check if Ollama is running
ps aux | grep ollama

# Start Ollama server
ollama serve

# Test connection
curl http://localhost:11434/api/version

# Check firewall
sudo lsof -i :11434
```

### Issue: Qdrant Collection Error

```bash
# Check Qdrant status
curl http://localhost:6333/

# List collections
curl http://localhost:6333/collections

# Delete problematic collection
curl -X DELETE http://localhost:6333/collections/my-project-codebase

# Rebuild index
kilo-code index rebuild
```

### Issue: Slow Indexing

**Symptoms:** Indexing takes >30 minutes for medium projects

**Solutions:**

1. **Reduce chunk size:**
```json
{
  "chunking": {
    "chunkSize": 256,
    "overlap": 32
  }
}
```

2. **Exclude more files:**
```json
{
  "filters": {
    "exclude": [
      "**/node_modules/**",
      "**/dist/**",
      "**/*.test.*",
      "**/*.md",
      "**/docs/**",
      "**/coverage/**"
    ]
  }
}
```

3. **Use faster model:**
```bash
ollama pull all-minilm
```

```json
{
  "embedding": {
    "model": "all-minilm"
  }
}
```

### Issue: Poor Search Results

**Symptoms:** Search doesn't find relevant code

**Solutions:**

1. **Rebuild index:**
```bash
kilo-code index rebuild --force
```

2. **Adjust chunking:**
```json
{
  "chunking": {
    "strategy": "code-aware",
    "chunkSize": 512,
    "overlap": 128
  }
}
```

3. **Check embedding model:**
```bash
# Test model quality
ollama run nomic-embed-text "generate embeddings for: authentication"
```

### Issue: Out of Memory

**Symptoms:** Ollama or Qdrant crashes during indexing

**Solutions:**

1. **Limit Ollama memory:**
```bash
# Set memory limit (in GB)
OLLAMA_MAX_VRAM=4 ollama serve
```

2. **Reduce batch size:**
```json
{
  "indexing": {
    "batchSize": 50
  }
}
```

3. **Use smaller model:**
```bash
ollama pull all-minilm:22m
```

---

## Performance Optimization

### Index Size vs. Search Quality

| Configuration | Index Size | Search Speed | Quality |
|--------------|------------|--------------|---------|
| Default | Medium | ~50ms | Good |
| High Quality | Large | ~100ms | Better |
| Fast Search | Small | ~20ms | Basic |

### High Quality Configuration

```json
{
  "indexing": {
    "embedding": {
      "model": "mxbai-embed-large"
    },
    "chunking": {
      "chunkSize": 512,
      "overlap": 128
    },
    "vectorStore": {
      "quantization": false
    }
  }
}
```

### Fast Search Configuration

```json
{
  "indexing": {
    "embedding": {
      "model": "all-minilm"
    },
    "chunking": {
      "chunkSize": 256,
      "overlap": 32
    },
    "vectorStore": {
      "quantization": true
    }
  }
}
```

### Quantization

Qdrant supports vector quantization to reduce storage:

```json
{
  "vectorStore": {
    "quantization": {
      "type": "scalar",
      "quantile": 0.99,
      "granularity": 0.01
    }
  }
}
```

**Benefits:**
- 4x smaller index size
- Faster search
- Minimal quality loss

---

## Best Practices

### 1. Index Only What You Need

```json
{
  "filters": {
    "include": ["src/**/*", "lib/**/*"],
    "exclude": ["**/*.test.*", "**/mocks/**", "**/fixtures/**"]
  }
}
```

### 2. Use Code-Aware Chunking

Always prefer `code-aware` chunking for source code:

```json
{
  "chunking": {
    "strategy": "code-aware"
  }
}
```

### 3. Schedule Regular Rebuilds

For active projects, rebuild weekly:

```bash
# Add to crontab
0 2 * * 0 cd /path/to/project && kilo-code index rebuild
```

### 4. Monitor Index Health

```bash
# Add health check to CI/CD
kilo-code index status --json | jq '.status'
```

### 5. Use Collection Namespaces

For multiple projects on same Qdrant instance:

```json
{
  "vectorStore": {
    "collectionName": "team-project-service"
  }
}
```

---

## Real-World Example: Large Monorepo

Here's how to index a large monorepo efficiently:

```json
{
  "indexing": {
    "enabled": true,
    "provider": "qdrant",
    "collections": [
      {
        "name": "web-frontend",
        "path": "./apps/web",
        "filters": {
          "include": ["**/*.tsx", "**/*.ts", "**/*.css"],
          "exclude": ["**/*.test.*", "**/node_modules/**"]
        }
      },
      {
        "name": "api-backend",
        "path": "./apps/api",
        "filters": {
          "include": ["**/*.py"],
          "exclude": ["**/tests/**", "**/__pycache__/**"]
        }
      },
      {
        "name": "shared-libraries",
        "path": "./packages",
        "filters": {
          "include": ["**/*.ts", "**/*.json"],
          "exclude": ["**/dist/**", "**/node_modules/**"]
        }
      }
    ],
    "embedding": {
      "model": "nomic-embed-text",
      "provider": "ollama"
    },
    "vectorStore": {
      "provider": "qdrant",
      "endpoint": "http://localhost:6333"
    },
    "incremental": true,
    "watchMode": true
  }
}
```

**Build all collections:**
```bash
kilo-code index build --all
```

**Search across all collections:**
```bash
kilo-code search "authentication middleware" --collections web-frontend,api-backend
```

---


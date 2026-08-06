---
title: Kilo Code: Integrating with Qwen Code CLI
url: https://devopstales.github.io/ai/kilo-code-series-03-qwen-integration/
date: 2026-03-27
---


In our previous posts, we covered the basics of Kilo Code and how to get it installed on your machine. One of the most powerful features of Kilo Code is its ability to integrate with various AI providers.

<!--more-->

{{< content "/filedir/kilo-code-series.html" >}}

Today, we're focusing on a particularly high-value integration: **Qwen Code CLI**. By linking these two tools, you can unlock a massive 1 million token context window and a generous free tier of 2,000 requests per day.

## Why Qwen Code CLI?

Qwen3-Coder is Alibaba's state-of-the-art open-weight coding model. Here's why it's the perfect companion for Kilo Code:

| Feature | Qwen Code CLI | Alternatives |
|---------|---------------|--------------|
| **Free Tier** | 2,000 requests/day | 50-100 requests/day |
| **Context Window** | 1M tokens | 100K-200K tokens |
| **Cost** | Free (with API key) | $10-30/month |
| **Code Quality** | SOTA for open weights | Proprietary models |
| **Latency** | ~2-5 seconds | ~5-15 seconds |

### What You'll Get

- **2,000 free requests per day** (resets daily)
- **1 million token context** (entire codebases in one prompt)
- **Specialized coding models** (Qwen3-Coder-32B, Qwen3-Coder-Plus)
- **Low latency** responses optimized for code generation
- **No credit card required** for free tier

---

## Step 1: Install Qwen Code CLI

### Method 1: npm (Recommended)

```bash
npm install -g @qwen-code/qwen-code
```

### Method 2: bun (Fastest)

```bash
bun install -g @qwen-code/qwen-code
```

### Method 3: pip (Python)

```bash
pip install qwen-code-cli
```

### Verify Installation

```bash
qwen-code --version
# Output: qwen-code version 1.0.0
```

---

## Step 2: Get Your API Key

### Option A: Free Tier (Recommended for Getting Started)

1. Visit [https://dashscope.aliyun.com](https://dashscope.aliyun.com)
2. Sign up with Alibaba Cloud account (or create one)
3. Navigate to **API Keys** in the dashboard
4. Click **Create New API Key**
5. Copy and save your key securely

**Free Tier Limits:**
- 2,000 requests per day
- 1M tokens per request
- Rate limit: 10 requests/minute

### Option B: Pay-As-You-Go (For Heavy Usage)

If you exceed the free tier:
- $0.002 per 1K tokens (input)
- $0.006 per 1K tokens (output)
- Still significantly cheaper than GPT-4 or Claude

### Option C: Self-Hosted (For Enterprises)

Run Qwen locally with Ollama:

```bash
# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Pull Qwen model
ollama pull qwen2.5-coder:32b

# Qwen Code CLI will auto-detect Ollama
```

---

## Step 3: Configure Qwen Code CLI

### Initialize Configuration

```bash
qwen-code config init
```

This creates a config file at `~/.qwen-code/config.json`:

```json
{
  "api_key": "sk-your-api-key-here",
  "model": "qwen-coder-plus",
  "base_url": "https://dashscope.aliyuncs.com/api/v1",
  "timeout": 120,
  "max_tokens": 100000
}
```

### Test Your Setup

```bash
qwen-code "Hello! Can you help me write Python code?"
```

**Expected response:**
```
Hello! I'm Qwen Code, your AI coding assistant. I'd be happy to help you write Python code. What would you like to build?
```

---

## Step 4: Integrate with Kilo Code

### In Kilo Code IDE

1. Open **Settings** (Cmd+, or Ctrl+,)
2. Navigate to **Kilo Code → AI Provider**
3. Select **Qwen Code CLI** from dropdown
4. Enter path to qwen-code binary (usually auto-detected)

**Settings JSON:**
```json
{
  "kiloCode.provider": "qwen-code",
  "kiloCode.qwenCode.path": "/usr/local/bin/qwen-code",
  "kiloCode.qwenCode.model": "qwen-coder-plus",
  "kiloCode.qwenCode.maxTokens": 100000
}
```

### In Kilo Code CLI

```bash
# Set Qwen as default provider
kilo-code config set provider qwen-code

# Configure model
kilo-code config set model qwen-coder-plus

# Set API key (or use environment variable)
kilo-code config set api_key sk-your-api-key-here
```

**Alternative: Environment Variables**

Add to your `~/.zshrc` or `~/.bashrc`:

```bash
export QWEN_CODE_API_KEY="sk-your-api-key-here"
export QWEN_CODE_MODEL="qwen-coder-plus"
export QWEN_CODE_MAX_TOKENS="100000"
```

### In VS Code Extension

1. Open Command Palette (`Cmd+Shift+P` or `Ctrl+Shift+P`)
2. Type "Kilo Code: Configure Provider"
3. Select **Qwen Code CLI**
4. Follow the setup wizard

---

## Step 5: Test the Integration

### Quick Test

Open a project folder and try:

**In Kilo Code IDE:**
```
Press Cmd+K and type:
"Create a simple REST API with Express.js"
```

**In Kilo Code CLI:**
```bash
cd ~/projects/test-app
kilo-code "Create a Python Flask app with user authentication"
```

### Expected Behavior

1. Kilo Code sends your request to Qwen Code CLI
2. Qwen analyzes your codebase context
3. Response appears in the chat/terminal
4. Code suggestions are generated
5. You can accept, reject, or modify the suggestions

### Sample Session

```bash
$ kilo-code "Create a function that validates email addresses"

🤖 Kilo Code (powered by Qwen):
I'll create an email validation function with comprehensive checks.

📝 Proposed changes:

--- lib/email_validator.py ---
import re

def validate_email(email: str) -> tuple[bool, str]:
    """
    Validate an email address.
    
    Returns:
        tuple: (is_valid, error_message)
    """
    if not email:
        return False, "Email is required"
    
    if len(email) > 254:
        return False, "Email too long"
    
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    if not re.match(pattern, email):
        return False, "Invalid email format"
    
    return True, "Valid email"

---

Accept these changes? (y/n/diff)
```

---

## Advanced Configuration

### Model Selection

Qwen offers multiple models for different use cases:

| Model | Best For | Context | Speed |
|-------|----------|---------|-------|
| `qwen-coder-32b` | General coding tasks | 256K | Fast |
| `qwen-coder-plus` | Complex reasoning | 1M | Medium |
| `qwen-coder-flash` | Quick completions | 128K | Very Fast |

**Change model:**
```bash
kilo-code config set model qwen-coder-plus
```

### Context Window Tuning

For large codebases:

```json
{
  "qwenCode": {
    "maxContextTokens": 1000000,
    "indexingStrategy": "semantic",
    "chunkSize": 4096,
    "overlapSize": 512
  }
}
```

### Rate Limiting

Avoid hitting API limits:

```json
{
  "qwenCode": {
    "requestsPerMinute": 8,
    "retryDelay": 5000,
    "maxRetries": 3
  }
}
```

---

## Troubleshooting

### Issue: "API Key Not Found"

```bash
# Check if API key is set
echo $QWEN_CODE_API_KEY

# If empty, set it:
export QWEN_CODE_API_KEY="sk-your-key"

# Or update config:
qwen-code config set api_key sk-your-key
```

### Issue: "Model Not Available"

```bash
# List available models
qwen-code models list

# Update to available model:
qwen-code config set model qwen-coder-32b
```

### Issue: "Rate Limit Exceeded"

```json
// Reduce request frequency
{
  "qwenCode": {
    "requestsPerMinute": 5,
    "enableCaching": true
  }
}
```

### Issue: "Context Window Exceeded"

```json
// Reduce context size
{
  "qwenCode": {
    "maxContextTokens": 500000,
    "smartContextTrimming": true
  }
}
```

---

## Cost Optimization Tips

### 1. Use the Right Model

- **qwen-coder-flash**: Quick edits, completions (cheapest)
- **qwen-coder-32b**: Standard development work
- **qwen-coder-plus**: Complex architecture, debugging (most expensive)

### 2. Enable Caching

```json
{
  "qwenCode": {
    "enableCache": true,
    "cacheDir": "~/.cache/qwen-code",
    "cacheTTL": 3600
  }
}
```

### 3. Limit Context

Only index relevant files:

```json
{
  "qwenCode": {
    "indexing": {
      "include": ["src/**/*", "lib/**/*"],
      "exclude": ["node_modules/**", "dist/**", "*.min.js"]
    }
  }
}
```

### 4. Monitor Usage

```bash
# Check daily usage
qwen-code usage

# Output:
# Today's usage: 847 / 2000 requests
# Tokens used: 45.2M / 100M
# Estimated cost: $0.00 (free tier)
```

---

## What's Next?

Now that you have Qwen Code CLI integrated, you're ready to explore **Kilo Code Modes** - specialized personas for different development tasks.

**Coming up:** [Kilo Code Series #4: Understanding Modes and the Orchestrator](/ai/kilo-code-series-04-modes/)

In the next post, we'll cover:
- Orchestrator mode for task coordination
- Architect mode for system design
- Code mode for implementation
- Debug mode for troubleshooting
- Creating custom modes

---

## Quick Reference

### Environment Variables

```bash
export QWEN_CODE_API_KEY="sk-..."
export QWEN_CODE_MODEL="qwen-coder-plus"
export QWEN_CODE_MAX_TOKENS="100000"
export QWEN_CODE_BASE_URL="https://dashscope.aliyuncs.com/api/v1"
```

### Configuration Files

| Location | Purpose |
|----------|---------|
| `~/.qwen-code/config.json` | Qwen CLI config |
| `.kilocode/config.json` | Project-specific config |
| `~/.kilo-code/settings.json` | Kilo Code IDE settings |

### Useful Commands

```bash
# Test Qwen installation
qwen-code "Hello"

# Check usage
qwen-code usage

# List models
qwen-code models list

# Change model
qwen-code config set model qwen-coder-plus

# In Kilo Code
kilo-code config get provider
kilo-code config set model qwen-coder-plus
```


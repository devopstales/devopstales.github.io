---
title: Kilo Code: Installation and Setup Guide
url: https://devopstales.github.io/ai/kilo-code-series-02-install/
date: 2026-03-26
---


In our first post, we introduced the core concepts of Kilo Code and why it's a game-changer for agentic software development. Now, it's time to get your hands dirty.

<!--more-->

{{< content "/filedir/kilo-code-series.html" >}}

![Developer setting up AI coding environment](/img/code-editor.webp)
*Getting started with Kilo Code takes just minutes*

In this guide, we'll walk you through the installation process for the three main ways to use Kilo Code: the **Kilo Code IDE**, the **Kilo Code CLI**, and **IDE Extensions**.

## Option 1: Kilo Code IDE (Recommended for Most Users)

The Kilo Code IDE is a standalone, AI-native IDE based on VS Code. All your favorite extensions work out of the box, but with deep AI integration.

### Step 1: Download Kilo Code

Visit [https://kilo-code.dev](https://kilo-code.dev) and download the installer for your platform:

| Platform | Download | Size |
|----------|----------|------|
| **macOS** (Intel/Apple Silicon) | `.dmg` | ~150 MB |
| **Windows** (x64/ARM) | `.exe` | ~180 MB |
| **Linux** (deb/rpm/AppImage) | Multiple formats | ~160 MB |

### Step 2: Install

**macOS:**

```bash
# Download and mount
open ~/Downloads/kilo-code.dmg

# Drag to Applications folder
# Or use terminal:
cp -R /Volumes/Kilo Code/Kilo Code.app /Applications/
```

**Windows:**

```powershell
# Run the installer
Start-Process ~\Downloads\kilo-code-setup.exe
```

**Linux (Ubuntu/Debian):**

```bash
# Download and install
sudo dpkg -i kilo-code_1.0.0_amd64.deb
sudo apt-get install -f  # Fix dependencies if needed
```

### Step 3: First Launch

1. Open Kilo Code IDE
2. You'll see a welcome screen with setup options
3. Choose your preferred AI provider (we'll configure Qwen in the next post)
4. Sign in with GitHub (optional, for settings sync)

### Step 4: Migrate VS Code Settings (Optional)

Kilo Code can import your existing VS Code configuration:

```bash
File → Preferences → Settings → Import from VS Code
```

This migrates:
- Keyboard shortcuts
- Themes and icons
- Snippets
- Workspace settings

---

## Option 2: Kilo Code CLI (For Terminal Lovers)

The Kilo Code CLI brings agentic AI directly to your terminal. It's lightweight, scriptable, and perfect for CI/CD pipelines.

### Prerequisites

- **Node.js** 18+ or **bun** runtime
- **Git** installed and configured
- Terminal with true color support (optional, for TUI)

### Installation Methods

**Method 1: npm (Recommended)**

```bash
npm install -g kilo-code
```

**Method 2: bun (Fastest)**

```bash
bun install -g kilo-code
```

**Method 3: Homebrew (macOS/Linux)**

```bash
brew install kilo-code-dev/kilo-code/kilo-code
```

**Method 4: Binary Download**

```bash
# macOS (Apple Silicon)
curl -L https://github.com/kilo-code-dev/kilo-code/releases/latest/download/kilo-code-darwin-arm64 -o /usr/local/bin/kilo-code
chmod +x /usr/local/bin/kilo-code

# Linux (x64)
curl -L https://github.com/kilo-code-dev/kilo-code/releases/latest/download/kilo-code-linux-x64 -o /usr/local/bin/kilo-code
chmod +x /usr/local/bin/kilo-code
```

### Verify Installation

```bash
kilo-code --version
# Output: kilo-code version 1.0.0

kilo-code doctor
# Checks your environment and reports any issues
```

### Quick Start

```bash
# Navigate to your project
cd ~/projects/my-app

# Start Kilo Code interactive mode
kilo-code

# Or run a single command
kilo-code "Create a new Python Flask app with user authentication"
```

---

## Option 3: IDE Extensions

If you prefer to stay in your existing IDE, Kilo Code offers extensions for popular editors.

### VS Code / VSCodium

1. Open Extensions panel (`Ctrl+Shift+X` or `Cmd+Shift+X`)
2. Search for "Kilo Code"
3. Click **Install**
4. Reload window when prompted

**Direct Install:**
```bash
code --install-extension kilo-code.kilo-code
```

### JetBrains IDEs (IntelliJ, PyCharm, WebStorm)

1. Open **Settings** → **Plugins**
2. Search for "Kilo Code"
3. Click **Install**
4. Restart IDE

**Manual Install:**
1. Download from [JetBrains Marketplace](https://plugins.jetbrains.com/plugin/kilo-code-kilo-code)
2. **Settings** → **Plugins** → ⚙️ → **Install Plugin from Disk**
3. Select downloaded `.zip` file

### Neovim / Vim

Add to your plugin manager configuration:

**Lazy.nvim:**

```lua
{
  "kilo-code-dev/kilo.nvim",
  dependencies = { "nvim-lua/plenary.nvim" },
  config = function()
    require("kilo").setup({
      provider = "qwen",  -- or your preferred provider
    })
  end
}
```

**Packer.nvim:**

```lua
use {
  "kilo-code-dev/kilo.nvim",
  requires = { "nvim-lua/plenary.nvim" },
  config = function()
    require("kilo").setup()
  end
}
```

---

## Post-Installation Setup

### 1. Configure AI Provider

Kilo Code supports multiple AI providers. We'll cover Qwen integration in detail in the next post, but here's a quick start:

**In Kilo Code IDE:**

```bash
Settings → Kilo Code → AI Provider → Select "Qwen Code CLI"
```

**In Kilo Code CLI:**

```bash
kilo-code config set provider qwen
kilo-code config set model qwen-coder-32b
```

### 2. Set Up Project Configuration

Create a `.kilocode/` folder in your project root:

```bash
mkdir -p .kilocode
touch .kilocode/config.json
```

**Basic config.json:**

```json
{
  "provider": "qwen",
  "model": "qwen-coder-32b",
  "mode": "orchestrator",
  "indexing": {
    "enabled": true,
    "provider": "qdrant"
  }
}
```

### 3. Create Your First Agent Session

**In Kilo Code IDE:**

1. Open a project folder
2. Press `Cmd+K` (macOS) or `Ctrl+K` (Windows/Linux)
3. Type: "Explain this codebase to me"

**In Kilo Code CLI:**

```bash
cd ~/projects/my-app
kilo-code "Analyze this codebase and create a summary"
```

---

## Troubleshooting

### Common Installation Issues

**Issue: "Command not found" after CLI install**

```bash
# Check your PATH
echo $PATH

# npm global bin location
npm bin -g

# Add to your shell config (~/.zshrc or ~/.bashrc)
export PATH="$(npm bin -g):$PATH"
```

**Issue: Kilo Code IDE won't start on Linux**

```bash
# Install required dependencies
sudo apt-get install -y libgtk-3-0 libnss3 libasound2

# Or use AppImage instead
chmod +x kilo-code.AppImage
./kilo-code.AppImage
```

**Issue: Extension doesn't load in VS Code**

1. Check VS Code version (must be 1.85+)
2. Reload window: `Cmd+Shift+P` → "Developer: Reload Window"
3. Check output panel: `View → Output → Kilo Code`

### Verify Everything Works

Run the diagnostic command:

```bash
kilo-code doctor
```

**Expected output:**

```
✓ Kilo Code CLI is installed (v1.0.0)
✓ Node.js is available (v20.11.0)
✓ Git is configured
✓ AI provider is connected
✓ Indexing service is running
✓ All systems operational!
```

---


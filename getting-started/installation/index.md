# Installation Guide

Get HoloDeck installed and ready to build AI agents.

## Prerequisites

- **Python 3.10+** (check with `python --version`)
- **uv** - The fast Python package installer
- **Node.js 18+** (required for Claude Agent SDK backend) — check with `node --version`

> **Note**: Node.js is only required if you plan to use `provider: anthropic`. Other providers (OpenAI, Azure, Ollama) do not require it.

### Installing uv

If you don't have uv installed:

```
# macOS/Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# macOS (Homebrew)
brew install uv

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Verify uv is installed:

```
uv --version
```

## Install HoloDeck CLI

Install HoloDeck as a global tool using uv:

```
uv tool install holodeck-ai@latest --prerelease allow --python 3.10
```

This installs the `holodeck` command-line tool globally, available from any directory.

### Install with Vector Store Providers (Optional)

If you plan to use semantic search with vector databases, install with extras:

```
# Individual providers
uv tool install "holodeck-ai[postgres]@latest" --prerelease allow --python 3.10
uv tool install "holodeck-ai[qdrant]@latest" --prerelease allow --python 3.10
uv tool install "holodeck-ai[pinecone]@latest" --prerelease allow --python 3.10
uv tool install "holodeck-ai[chromadb]@latest" --prerelease allow --python 3.10

# Or install all vector store providers at once
uv tool install "holodeck-ai[vectorstores]@latest" --prerelease allow --python 3.10
```

## Verify Installation

Check that HoloDeck is installed correctly:

```
holodeck --version
# Output: holodeck 0.2.0
```

View available commands:

```
holodeck --help
```

## Set Up LLM Provider

HoloDeck supports multiple LLM providers. Ollama is recommended for local development as it requires no API keys and runs entirely on your machine.

### Ollama (Recommended)

Ollama runs LLMs locally on your machine - no API keys required.

**Install Ollama:**

```
# macOS
brew install ollama

# Linux
curl -fsSL https://ollama.com/install.sh | sh

# Windows: Download from https://ollama.com/download
```

**Start Ollama and pull a model:**

```
# Start the Ollama service
ollama serve

# Pull a model (in another terminal)
ollama pull llama3.2
# Or for a smaller model:
ollama pull phi3
```

**Verify Ollama is running:**

```
curl http://localhost:11434/api/tags
```

No environment variables needed - HoloDeck connects to Ollama at `http://localhost:11434` by default.

### Cloud Providers (Optional)

For cloud-based LLMs, set up credentials using environment variables or a `.env` file.

#### Environment Variables

```
# OpenAI
export OPENAI_API_KEY="sk-..."

# Azure OpenAI
export AZURE_OPENAI_API_KEY="your-key-here"
export AZURE_OPENAI_ENDPOINT="https://your-resource.openai.azure.com/"

# Anthropic (API Key)
export ANTHROPIC_API_KEY="sk-ant-..."

# Anthropic (OAuth Token — recommended for Claude Code users)
export CLAUDE_CODE_OAUTH_TOKEN="your-oauth-token"
```

#### `.env` File

Create a `.env` file in your project directory:

```
# .env (never commit this file!)
OPENAI_API_KEY=sk-...
```

Add `.env` to `.gitignore`:

```
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
```

## Supported LLM Providers

HoloDeck supports multiple LLM providers:

### Ollama (Recommended)

Run LLMs locally with no API keys. Supports Llama, Mistral, Phi, and many more models.

```
# No environment variables needed
# Default endpoint: http://localhost:11434
```

### OpenAI

```
OPENAI_API_KEY=sk-...
OPENAI_ORG_ID=optional-org-id
```

### Azure OpenAI

```
AZURE_OPENAI_API_KEY=your-key-here
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
```

### Anthropic

Claude models via the native Claude Agent SDK backend. Requires Node.js 18+.

```
# Option 1: API Key
ANTHROPIC_API_KEY=sk-ant-...

# Option 2: OAuth Token (recommended for Claude Code users)
CLAUDE_CODE_OAUTH_TOKEN=your-oauth-token
```

## Upgrading HoloDeck

To upgrade to the latest version:

```
uv tool upgrade holodeck-ai
```

To reinstall with a specific version:

```
uv tool install holodeck-ai@0.3.0 --prerelease allow --python 3.10 --force
```

## Uninstalling

To remove HoloDeck:

```
uv tool uninstall holodeck-ai
```

## Troubleshooting

### "Python 3.10+ required"

Check your Python version and upgrade if needed:

```
python --version

# macOS (Homebrew)
brew install python@3.10

# Ubuntu/Debian
sudo apt-get install python3.10

# Windows: Download from python.org
```

### "holodeck: command not found"

The CLI isn't in your PATH. Try:

```
# Reinstall HoloDeck
uv tool install holodeck-ai@latest --prerelease allow --python 3.10 --force

# Ensure uv tools are in PATH
# Add to your shell profile (~/.bashrc, ~/.zshrc, etc.):
export PATH="$HOME/.local/bin:$PATH"

# Then reload your shell
source ~/.zshrc  # or ~/.bashrc
```

### "uv: command not found"

Install uv first. See [Installing uv](#installing-uv) above.

### "Error: API key not found" or "Invalid credentials"

Verify your environment variables are set:

```
# Check if variables are set
echo $AZURE_OPENAI_API_KEY  # macOS/Linux
echo %AZURE_OPENAI_API_KEY%  # Windows

# Or check .env file exists
cat .env
```

If using a `.env` file, ensure it's in your project directory.

## Next Steps

- [Quickstart Guide](https://docs.useholodeck.ai/getting-started/quickstart/index.md) - Build your first agent in 5 minutes
- [Agent Configuration Guide](https://docs.useholodeck.ai/guides/agent-configuration/index.md) - Full configuration reference
- [Example Agents](https://docs.useholodeck.ai/examples/index.md) - Browse example agents
- [Global Configuration](https://docs.useholodeck.ai/guides/global-config/index.md) - Configure defaults

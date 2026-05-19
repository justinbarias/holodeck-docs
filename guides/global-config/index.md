# Global Configuration Guide

This guide explains HoloDeck's global configuration system for shared settings across agents.

## Overview

Global configuration lives at `~/.holodeck/config.yaml` and provides default settings that apply to all agents. Use global config to:

- Set default LLM providers and credentials
- Define reusable vectorstore connections
- Configure deployment defaults
- Store API keys securely
- Reduce duplication across agent.yaml files

## Basic Structure

```
# config.yaml (project root or ~/.holodeck/config.yaml)

providers:
  azure_openai:
    provider: azure_openai
    name: gpt-4o
    temperature: 0.3
    max_tokens: 2048
    endpoint: ${AZURE_OPENAI_ENDPOINT}
    api_key: ${AZURE_OPENAI_API_KEY}

execution:
  file_timeout: 30
  llm_timeout: 60
  download_timeout: 30
  cache_enabled: true
  cache_dir: .holodeck_cache
  verbose: false
```

## Configuration Precedence

When multiple configuration sources define the same setting, HoloDeck applies them in priority order:

```
1. agent.yaml (Highest Priority)
   ├─ Explicit values in agent configuration
   │
2. Environment Variables (High Priority)
   ├─ ${VAR_NAME} patterns in agent.yaml or global config
   │
3. Project-level config.yaml (Medium Priority)
   ├─ Same directory as agent.yaml
   │
4. ~/.holodeck/config.yaml (Lowest Priority)
   └─ User home directory global defaults
```

### Precedence Diagram

```
┌──────────────────────────────┐
│   agent.yaml explicit        │  Takes precedence
├──────────────────────────────┤
│   Environment variables      │  Used if agent.yaml absent
├──────────────────────────────┤
│  Project-level config.yaml   │  Used if env var absent
├──────────────────────────────┤
│ ~/.holodeck/config.yaml      │  Fallback default
└──────────────────────────────┘
```

### Examples

#### Example 1: Provider Override

Global config:

```
providers:
  openai:
    model: gpt-4o-mini
    temperature: 0.7
```

Agent config:

```
model:
  provider: openai
  name: gpt-4o # Overrides global default
  temperature: 0.5 # Overrides global default
```

Result: Agent uses `gpt-4o` at temperature `0.5` (agent config wins)

#### Example 2: Environment Variable

Global config:

```
providers:
  openai:
    api_key: ${OPENAI_API_KEY}
```

Agent config:

```
model:
  provider: openai
```

Environment:

```
export OPENAI_API_KEY="sk-..."
```

Result: Uses environment variable for API key

#### Example 3: Full Precedence Chain

Global config:

```
providers:
  default_model: gpt-4o-mini

deployment:
  default_port: 8000
```

Agent config:

```
model:
  provider: openai
  # No explicit temperature

deployment:
  port: 8080 # Overrides global
```

Environment:

```
export TEMPERATURE=0.5
```

Result: Model uses `gpt-4o-mini`, port is `8080`, temperature is `0.5`

## Inheritance Rules

Not all agent settings are inherited from global config. Here's what you can and cannot configure globally:

### Settings That CAN Be Inherited

- LLM provider credentials (API keys, endpoints)
- Default model names and settings (temperature, max_tokens)
- Vectorstore configurations
- Deployment settings

### Settings That CANNOT Be Inherited (Agent-Level Only)

- **response_format**: Define structured output schema in each agent.yaml, not in global config
- **tools**: Tools must be defined per agent based on agent capabilities
- **instructions**: System prompt must be specific to each agent
- **evaluations**: Evaluation metrics are typically agent-specific
- **test_cases**: Test cases validate individual agent behavior

```
# .holodeck/config.yaml - Only shared settings here
providers:
  openai:
    api_key: ${OPENAI_API_KEY}
    organization: my-org
    temperature: 0.7

deployment:
  default_port: 8000
```

```
# agent.yaml - Agent-specific settings here
name: my-agent

model:
  provider: openai
  # Inherits temperature: 0.7 from global, can override

response_format: # Cannot be inherited, must define here
  type: object
  properties:
    answer:
      type: string

instructions: # Must be defined here
  inline: "You are a helpful assistant"
```

## Providers Section

Defines LLM provider configurations with credentials and defaults.

```
providers:
  azure_openai:
    provider: azure_openai # Required: provider type
    name: gpt-4o # Required: model name
    temperature: 0.3 # Optional: temperature (0.0-2.0)
    max_tokens: 2048 # Optional: max tokens
    endpoint: ${AZURE_OPENAI_ENDPOINT} # Required: Azure endpoint
    api_key: ${AZURE_OPENAI_API_KEY} # Required: API key

  openai:
    provider: openai
    name: gpt-4o-mini
    temperature: 0.7
    api_key: ${OPENAI_API_KEY}
    organization: my-org # Optional

  anthropic:
    provider: anthropic
    name: claude-3-sonnet
    temperature: 0.5
    api_key: ${ANTHROPIC_API_KEY}
```

### Provider Configuration Fields

Each provider must have:

- **provider** (Required): Provider type - `openai`, `azure_openai`, or `anthropic`
- **name** (Required): Model identifier (e.g., `gpt-4o`, `claude-3-sonnet`)
- **api_key** (Required): API authentication key (use `${ENV_VAR}` for environment variables)

Optional fields:

- **temperature** (Optional): Float 0.0-2.0, defaults to provider's default
- **max_tokens** (Optional): Maximum response length
- **endpoint** (Required for Azure): Azure OpenAI endpoint URL
- **organization** (Optional for OpenAI): Organization ID

## Execution Section

Configures execution settings for agent test runs and file processing.

```
execution:
  file_timeout: 30 # Timeout for file processing (seconds)
  llm_timeout: 60 # Timeout for LLM API calls (seconds)
  download_timeout: 30 # Timeout for downloading files (seconds)
  cache_enabled: true # Enable caching of file downloads
  cache_dir: .holodeck_cache # Directory for cache storage
  verbose: false # Enable verbose logging
  quiet: false # Enable quiet mode
```

### Execution Fields

- **file_timeout** (Optional): Seconds to wait for file operations (default: 30)
- **llm_timeout** (Optional): Seconds to wait for LLM API calls (default: 60)
- **download_timeout** (Optional): Seconds to wait for file downloads (default: 30)
- **cache_enabled** (Optional): Enable caching of downloaded files (default: true)
- **cache_dir** (Optional): Directory for storing cached files (default: `.holodeck_cache`)
- **verbose** (Optional): Enable verbose logging output (default: false)
- **quiet** (Optional): Enable quiet mode, suppressing non-critical output (default: false)

## Environment Variables

Replace sensitive values with environment variables using `${VAR_NAME}` syntax:

```
providers:
  openai:
    api_key: ${OPENAI_API_KEY} # Reads from environment
    organization: my-org # Literal value
```

### Setting Environment Variables

**On Linux/macOS**:

```
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
```

**On Windows**:

```
set OPENAI_API_KEY=sk-...
set ANTHROPIC_API_KEY=sk-ant-...
```

**In .env file** (automatic loading):

Create `.env` in project directory:

```
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
```

HoloDeck automatically loads `.env` file if present.

### Variable Precedence

For `${VARIABLE_NAME}`:

1. Check environment variable
1. Check .env file
1. Return empty string if not found (error at agent runtime)

## File Locations

### Default Location

```
~/.holodeck/config.yaml
```

On different operating systems:

- **Linux**: `/home/username/.holodeck/config.yaml`
- **macOS**: `/Users/username/.holodeck/config.yaml`
- **Windows**: `C:\Users\username\.holodeck\config.yaml`

### Custom Location (Future)

```
holodeck --config /path/to/custom.yaml ...
```

## Complete Example

```
# config.yaml (project root or ~/.holodeck/config.yaml)

# LLM Provider Configurations
providers:
  azure_openai:
    provider: azure_openai
    name: gpt-4o
    temperature: 0.3
    max_tokens: 2048
    endpoint: ${AZURE_OPENAI_ENDPOINT}
    api_key: ${AZURE_OPENAI_API_KEY}

  openai:
    provider: openai
    name: gpt-4o-mini
    temperature: 0.7
    max_tokens: 1024
    api_key: ${OPENAI_API_KEY}
    organization: acme-corp

  anthropic:
    provider: anthropic
    name: claude-3-sonnet
    temperature: 0.5
    api_key: ${ANTHROPIC_API_KEY}

# Execution Configuration
execution:
  file_timeout: 30
  llm_timeout: 60
  download_timeout: 30
  cache_enabled: true
  cache_dir: .holodeck_cache
  verbose: false
```

## Usage Patterns

### Pattern 1: Secure API Keys

Keep secrets in global config with environment variable substitution:

Global config (project root):

```
# config.yaml
providers:
  azure_openai:
    provider: azure_openai
    name: gpt-4o
    endpoint: ${AZURE_OPENAI_ENDPOINT}
    api_key: ${AZURE_OPENAI_API_KEY}
```

Agent config:

```
# agent.yaml
model:
  provider: azure_openai
  # Credentials come from global config
```

Environment:

```
export AZURE_OPENAI_ENDPOINT="https://..."
export AZURE_OPENAI_API_KEY="..."
```

### Pattern 2: Execution Defaults

Set timeouts and caching for all agents:

Global config:

```
# config.yaml
providers:
  azure_openai:
    provider: azure_openai
    name: gpt-4o
    api_key: ${AZURE_OPENAI_API_KEY}

execution:
  file_timeout: 30
  llm_timeout: 60
  cache_enabled: true
  verbose: false
```

All agents inherit these execution settings automatically.

### Pattern 3: Multiple Providers

Configure multiple providers for different use cases:

Global config:

```
# config.yaml
providers:
  azure_openai:
    provider: azure_openai
    name: gpt-4o
    api_key: ${AZURE_OPENAI_API_KEY}
    endpoint: ${AZURE_OPENAI_ENDPOINT}

  openai:
    provider: openai
    name: gpt-4o-mini
    api_key: ${OPENAI_API_KEY}

execution:
  llm_timeout: 60
```

Agent config (use either provider):

```
# agent.yaml
model:
  provider: azure_openai # or openai
  # Model name and settings come from global config
```

## Creating Configuration

You can create configuration files using the `holodeck config init` command or manually.

### Using the CLI (Recommended)

The CLI provides a convenient way to initialize configuration files with default settings.

**Initialize Global Configuration:**

```
holodeck config init -g
# Creates ~/.holodeck/config.yaml
```

**Initialize Project Configuration:**

```
holodeck config init -p
# Creates config.yaml in the current directory
```

### Manual Creation

Global config can be created at two locations with different precedence:

1. **Project-level**: `config.yaml` in same directory as `agent.yaml` (higher priority)
1. **User-level**: `~/.holodeck/config.yaml` in home directory (lower priority)

#### Project-Level Config (Recommended for Teams)

Create `config.yaml` alongside your agents:

```
my-project/
├── config.yaml          # Project-specific configuration
├── agent1/
│   └── agent.yaml
└── agent2/
    └── agent.yaml
```

Content of `config.yaml`:

```
providers:
  azure_openai:
    provider: azure_openai
    name: gpt-4o
    api_key: ${AZURE_OPENAI_API_KEY}
    endpoint: ${AZURE_OPENAI_ENDPOINT}

execution:
  llm_timeout: 60
```

#### User-Level Config (Global Defaults)

Create `~/.holodeck/config.yaml` in your home directory:

```
mkdir -p ~/.holodeck

cat > ~/.holodeck/config.yaml << 'EOF'
providers:
  azure_openai:
    provider: azure_openai
    name: gpt-4o
    api_key: ${AZURE_OPENAI_API_KEY}
    endpoint: ${AZURE_OPENAI_ENDPOINT}
EOF
```

### Setting Environment Variables

```
export AZURE_OPENAI_API_KEY="..."
export AZURE_OPENAI_ENDPOINT="https://..."
```

Or in `.env` file at project root:

```
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_ENDPOINT=...
```

### Running an Agent

```
# Uses agent.yaml in current directory by default
holodeck test

# Or specify explicit path
holodeck test agent.yaml
```

The agent will automatically load config from project root or `~/.holodeck/`.

## Troubleshooting

### Error: "api_key not found"

- Check global config exists at `~/.holodeck/config.yaml`
- Verify environment variable is set: `echo $OPENAI_API_KEY`
- Check variable name matches in config

### Error: "invalid provider"

- Check spelling of provider in agent.yaml
- Valid providers: `openai`, `azure_openai`, `anthropic`

### Agent ignoring global config

- Verify global config file exists
- Check file permissions: `ls -la ~/.holodeck/`
- Verify YAML syntax: `cat ~/.holodeck/config.yaml`

### Environment variable not expanding

- Check syntax: `${VAR_NAME}` (with braces)
- Verify variable exists: `env | grep VAR_NAME`
- Note: `$VAR_NAME` (without braces) is not expanded

## Best Practices

1. **Keep Secrets Secure**: Never commit API keys to version control
1. **Use Environment Variables**: Store keys in env, not YAML
1. **Global Defaults**: Use global config for shared organization settings
1. **Per-Agent Overrides**: Use agent.yaml for agent-specific settings
1. **Don't Over-Configure**: Keep global config minimal and focused
1. **Document Settings**: Add comments to explain why settings exist
1. **Version Control**: Commit `config.yaml.example` with placeholders, not real keys

## Example: Secure Setup

```
# 1. Create project-level config
cat > config.yaml << 'EOF'
providers:
  azure_openai:
    provider: azure_openai
    name: gpt-4o
    api_key: ${AZURE_OPENAI_API_KEY}
    endpoint: ${AZURE_OPENAI_ENDPOINT}

execution:
  llm_timeout: 60
  file_timeout: 30
EOF

# 2. Create agent config
cat > agent.yaml << 'EOF'
name: my-agent

model:
  provider: azure_openai

instructions:
  inline: "You are a helpful assistant."

test_cases:
  - input: "Hello!"
    ground_truth: "Hi there! How can I help?"
    evaluations:
      - f1_score

evaluations:
  model:
    provider: azure_openai
  metrics:
    - metric: f1_score
      threshold: 0.7
EOF

# 3. Create .env file with secrets (DO NOT commit)
cat > .env << 'EOF'
AZURE_OPENAI_API_KEY=your-key-here
AZURE_OPENAI_ENDPOINT=https://your-endpoint.openai.azure.com/
EOF

# 4. Run agent (config and env automatically loaded)
holodeck test
```

## Next Steps

- See [Agent Configuration Guide](https://docs.useholodeck.ai/guides/agent-configuration/index.md) for agent-specific settings
- See [File References Guide](https://docs.useholodeck.ai/guides/file-references/index.md) for path resolution

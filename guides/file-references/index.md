# File References Guide

This guide explains how file paths work in HoloDeck configurations.

## Quick start

Point a config field at a file with a relative path, or inject an environment variable with `${VAR}`. Both resolve when the agent loads.

```
# agent.yaml — paths resolve relative to this file's directory
instructions:
  file: system_prompt.txt            # → project/system_prompt.txt

tools:
  - type: vectorstore
    source: ${DATA_DIR}/kb.json      # ${VAR} expands from the environment
```

```
export DATA_DIR=/home/user/data
holodeck chat agent.yaml
```

If a referenced file is missing, the load fails fast with the resolved path it expected to find — no silent runtime surprises.

## How it works

Every file path in an `agent.yaml` is resolved relative to the directory containing that YAML (not your shell's working directory), so agents stay portable across machines. Absolute paths (`/...`), home expansion (`~`), and environment variables (`${VAR}`) are all supported, and `${VAR}` is the recommended way to keep secrets and machine-specific roots out of the config. HoloDeck validates every path at load time. The sections below cover the resolution rules, validation, and project-layout patterns in detail.

## Path Resolution Rules

### Rule 1: Relative Paths (Default)

Paths are relative to the **directory containing agent.yaml**:

```
project/
├── agent.yaml          (references below)
├── system_prompt.txt   ← relative path: system_prompt.txt
├── data/
│   └── kb.json         ← relative path: data/kb.json
└── tools/
    └── search.py       ← relative path: tools/search.py
```

Usage in `agent.yaml`:

```
# From project/agent.yaml

instructions:
  file: system_prompt.txt        # Resolves to: project/system_prompt.txt

tools:
  - type: vectorstore
    source: data/kb.json         # Resolves to: project/data/kb.json

  - type: function
    file: tools/search.py        # Resolves to: project/tools/search.py
```

### Rule 2: Absolute Paths

Paths starting with `/` are absolute:

```
instructions:
  file: /etc/holodeck/system_prompt.txt  # Absolute path

tools:
  - type: vectorstore
    source: /data/knowledge_base/        # Absolute path
```

### Rule 3: Home Directory Expansion

`~` expands to user home directory:

```
instructions:
  file: ~/templates/prompt.txt    # Expands to: /home/user/templates/prompt.txt
```

On different systems:

- **Linux**: `/home/username/`
- **macOS**: `/Users/username/`
- **Windows**: `C:\Users\username\`

## Common Path Patterns

### Sibling Files

Instructions in same directory as agent:

```
# project/agent.yaml
instructions:
  file: system_prompt.txt

# File: project/system_prompt.txt
```

### Subdirectories

Tools in subdirectory:

```
# project/agent.yaml
tools:
  - type: function
    file: tools/my_tool.py

# File: project/tools/my_tool.py
```

### Parent Directory

Reference parent directory with `..`:

```
# project/agents/support/agent.yaml
instructions:
  file: ../../shared/system_prompt.txt

# File: project/shared/system_prompt.txt
```

### Deeply Nested

Multiple levels:

```
# project/agents/v2/beta/agent.yaml
tools:
  - type: vectorstore
    source: ../../../../data/kb/

# File: project/data/kb/
```

## Validation

HoloDeck validates file paths during configuration loading:

### Validation Rules

1. **File must exist**: Checked when agent loads
1. **Readable**: File must be readable by current user
1. **Type checking**: File type must match context
1. Instructions: Text file (`.txt`, `.md`, etc.)
1. Vectorstore: Data file or directory
1. Function: Python file (`.py`)
1. Prompt: Text file

### Validation Errors

```
Error: File not found
Path: tools/search.py
Expected at: /home/user/project/tools/search.py
Suggestions:
- Check file exists in project directory
- Use relative path from agent.yaml directory
- Use absolute path if outside project
```

## Examples by File Type

### Instructions File

Structure:

```
project/
├── agent.yaml
└── system_prompt.txt
```

agent.yaml:

```
name: my-agent

instructions:
  file: system_prompt.txt
```

system_prompt.txt:

```
You are a helpful assistant.
Answer questions clearly and concisely.
```

### Vectorstore Data Files

Structure for single file:

```
project/
├── agent.yaml
└── knowledge_base.json
```

agent.yaml:

```
tools:
  - name: search-kb
    type: vectorstore
    source: knowledge_base.json
```

Structure for directory:

```
project/
├── agent.yaml
└── data/
    ├── doc1.md
    ├── doc2.md
    └── doc3.txt
```

agent.yaml:

```
tools:
  - name: search-docs
    type: vectorstore
    source: data/
```

### Function Tool Files

Structure:

```
project/
├── agent.yaml
└── tools/
    ├── search.py
    └── database.py
```

agent.yaml:

```
tools:
  - name: search-function
    type: function
    file: tools/search.py
    function: search_database

  - name: get-user
    type: function
    file: tools/database.py
    function: get_user
```

### Prompt Tool Files

Structure:

```
project/
├── agent.yaml
└── prompts/
    ├── summarize.txt
    └── classify.txt
```

agent.yaml:

```
tools:
  - name: summarize
    type: prompt
    file: prompts/summarize.txt
    parameters:
      text:
        type: string

  - name: classify
    type: prompt
    file: prompts/classify.txt
    parameters:
      text:
        type: string
```

## Complex Project Structures

### Monorepo with Multiple Agents

```
project/
├── shared/
│   ├── system_prompts/
│   │   ├── support.txt
│   │   ├── sales.txt
│   │   └── backend.txt
│   ├── data/
│   │   ├── kb.json
│   │   └── faq.csv
│   └── tools/
│       ├── common.py
│       ├── database.py
│       └── api.py
├── agents/
│   ├── support/
│   │   └── agent.yaml
│   ├── sales/
│   │   └── agent.yaml
│   └── backend/
│       └── agent.yaml
```

Support agent config:

```
# agents/support/agent.yaml

instructions:
  file: ../../shared/system_prompts/support.txt

tools:
  - name: search-kb
    type: vectorstore
    source: ../../shared/data/kb.json

  - name: query-db
    type: function
    file: ../../shared/tools/database.py
    function: query
```

### Shared Templates Directory

```
project/
├── templates/
│   ├── system_prompt.txt
│   └── prompts/
│       ├── summarize.txt
│       └── analyze.txt
├── data/
│   └── kb/
└── agents/
    └── agent.yaml
```

Agent config:

```
# agents/agent.yaml

instructions:
  file: ../templates/system_prompt.txt

tools:
  - name: search
    type: vectorstore
    source: ../data/kb/

  - name: summarize
    type: prompt
    file: ../templates/prompts/summarize.txt
    parameters:
      text:
        type: string
```

## Environment Variables in Paths

Paths can include environment variables:

```
instructions:
  file: ${TEMPLATE_DIR}/system_prompt.txt

tools:
  - type: vectorstore
    source: ${DATA_DIR}/knowledge_base/
```

Environment setup:

```
export TEMPLATE_DIR=/home/user/templates
export DATA_DIR=/home/user/data
```

Result:

```
instructions.file → /home/user/templates/system_prompt.txt
tools[0].source → /home/user/data/knowledge_base/
```

## Best Practices

1. **Use Relative Paths**: Keep agents portable across machines

   ```
   instructions:
     file: system_prompt.txt  # Good
     # NOT: file: /home/user/project/system_prompt.txt
   ```

1. **Organize by Type**: Group related files

   ```
   project/
   ├── prompts/          ← All prompt templates
   ├── data/             ← All data files
   ├── tools/            ← All function tools
   └── agent.yaml
   ```

1. **Document Structure**: Include README

   ```
   project/
   ├── README.md         ← Explain file structure
   ├── agent.yaml
   ├── system_prompt.txt
   └── ...
   ```

1. **Use Consistent Naming**: Predictable organization

   ```
   project/
   ├── prompts/          ← Not prompt/ or system_prompts/
   ├── tools/            ← Not tool/ or functions/
   └── data/             ← Not database/ or knowledge_base/
   ```

1. **Don't Hardcode Paths**: Use environment variables or global config

   ```
   # Wrong:
   instructions:
     file: /Users/john/project/system_prompt.txt

   # Right:
   instructions:
     file: system_prompt.txt
   ```

1. **Verify on Startup**: HoloDeck checks all paths when agent loads

1. No runtime surprises

1. Errors caught early

## Troubleshooting

### Error: "File not found"

**Problem**: File doesn't exist at specified path

**Solutions**:

1. Check file exists in filesystem:

   ```
   ls -la project/system_prompt.txt
   ```

1. Verify path is relative to agent.yaml directory:

   ```
   agent.yaml location: /home/user/project/
   referenced file: system_prompt.txt
   expected location: /home/user/project/system_prompt.txt
   ```

1. Check absolute path if using one:

   ```
   ls -la /full/path/to/file.txt
   ```

1. Expand environment variables manually:

   ```
   echo "${TEMPLATE_DIR}/system_prompt.txt"
   ```

### Error: "Permission denied"

**Problem**: File exists but not readable

**Solutions**:

1. Check file permissions:

   ```
   ls -la project/system_prompt.txt
   # Should have read permission for current user
   ```

1. Make file readable:

   ```
   chmod 644 project/system_prompt.txt
   ```

1. Check directory permissions:

   ```
   chmod 755 project/
   ```

### Error: "Invalid file type"

**Problem**: File format doesn't match context

**Solutions**:

1. For instructions: Use text files (`.txt`, `.md`)
1. For vectorstore: Use data files (`.json`, `.csv`, `.md`) or directories
1. For function tools: Use Python files (`.py`)
1. For prompts: Use text files (`.txt`, `.md`)

### Error: "Path traversal outside project"

**Problem**: Path goes outside allowed directory

**Solutions**:

1. Use absolute paths for files outside project:

   ```
   instructions:
     file: /etc/holodeck/system_prompt.txt
   ```

1. Or relative path if within allowed scope:

   ```
   instructions:
     file: ../../shared/prompt.txt
   ```

### Path Not Expanding (Environment Variable)

**Problem**: `${VAR}` not replaced with actual value

**Solutions**:

1. Check variable is exported:

   ```
   export TEMPLATE_DIR=/path/to/templates
   ```

1. Not just set locally:

   ```
   # Wrong:
   TEMPLATE_DIR=/path/to/templates
   holodeck test

   # Right:
   export TEMPLATE_DIR=/path/to/templates
   holodeck test
   ```

1. Or use absolute path instead:

   ```
   instructions:
     file: /path/to/templates/system_prompt.txt
   ```

## Next Steps

- See [Agent Configuration Guide](https://docs.useholodeck.ai/guides/agent-configuration/index.md) for usage examples
- See [Global Configuration Guide](https://docs.useholodeck.ai/guides/global-config/index.md) for environment variables
- See [Tools Reference](https://docs.useholodeck.ai/guides/tools/index.md) for specific tool file requirements

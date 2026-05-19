# Configuration Loading and Management API

This section documents the HoloDeck configuration system, including YAML loading, validation, environment variable substitution, schema validation, and configuration management.

## Overview

The configuration system is organized across seven modules:

| Module       | Purpose                                                                      |
| ------------ | ---------------------------------------------------------------------------- |
| `loader`     | YAML parsing, global/project config loading, merging, and MCP server helpers |
| `env_loader` | Environment variable substitution (`${VAR}` pattern) and `.env` file loading |
| `validator`  | Pydantic error flattening for human-readable validation messages             |
| `defaults`   | Built-in default constants and embedding dimension lookup                    |
| `context`    | Request-scoped `ContextVar` for agent base directory                         |
| `manager`    | Configuration file creation, path resolution, and YAML generation            |
| `schema`     | JSON Schema validation for LLM response formats                              |

______________________________________________________________________

## ConfigLoader

The main entry point for loading HoloDeck agent configurations from YAML.

## `ConfigLoader()`

Loads and validates agent configuration from YAML files.

This class handles:

- Parsing YAML files into Python dictionaries
- Loading global configuration from ~/.holodeck/config.yaml
- Merging configurations with proper precedence
- Resolving file references (instructions, tools)
- Converting validation errors into human-readable messages
- Environment variable substitution

Initialize the ConfigLoader with empty caches.

Source code in `src/holodeck/config/loader.py`

```
def __init__(self) -> None:
    """Initialize the ConfigLoader with empty caches."""
    self._user_config_loaded = False
    self._user_config: GlobalConfig | None = None
    self._project_configs: dict[str, GlobalConfig | None] = {}
```

### `parse_yaml(file_path)`

Parse a YAML file and return its contents as a dictionary.

Parameters:

| Name        | Type  | Description                    | Default    |
| ----------- | ----- | ------------------------------ | ---------- |
| `file_path` | `str` | Path to the YAML file to parse | *required* |

Returns:

| Type             | Description |
| ---------------- | ----------- |
| \`dict[str, Any] | None\`      |

Raises:

| Type                | Description                |
| ------------------- | -------------------------- |
| `FileNotFoundError` | If the file does not exist |
| `ConfigError`       | If YAML parsing fails      |

Source code in `src/holodeck/config/loader.py`

```
def parse_yaml(self, file_path: str) -> dict[str, Any] | None:
    """Parse a YAML file and return its contents as a dictionary.

    Args:
        file_path: Path to the YAML file to parse

    Returns:
        Dictionary containing parsed YAML content, or None if file is empty

    Raises:
        FileNotFoundError: If the file does not exist
        ConfigError: If YAML parsing fails
    """
    path = Path(file_path)

    try:
        with open(path, encoding="utf-8") as f:
            content = yaml.safe_load(f)
            return content if content is not None else {}
    except OSError as e:
        raise FileNotFoundError(
            file_path,
            f"Configuration file not found at {file_path}. "
            f"Please ensure the file exists at this path.",
        ) from e
    except yaml.YAMLError as e:
        raise ConfigError(
            "yaml_parse",
            f"Failed to parse YAML file {file_path}: {str(e)}",
        ) from e
```

### `load_agent_yaml(file_path, substitute_env=True)`

Load and validate an agent configuration from YAML.

This method:

1. Parses the YAML file with env var substitution (single pass)
1. Loads and merges user + project configs (project overrides user)
1. Merges global config into agent config
1. Validates against Agent schema
1. Returns an Agent instance

Configuration precedence (highest to lowest):

1. agent.yaml explicit settings
1. Environment variables
1. Project-level config.yaml/config.yml
1. Global ~/.holodeck/config.yaml/config.yml

Parameters:

| Name             | Type   | Description                                                                                                                                                                                                                                                                               | Default    |
| ---------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `file_path`      | `str`  | Path to agent.yaml file                                                                                                                                                                                                                                                                   | *required* |
| `substitute_env` | `bool` | When True (default), ${VAR} references in both the agent YAML and any referenced test_cases_file are resolved against the process environment. When False, the raw ${VAR} literal is preserved — used by structural consumers like holodeck deploy build that don't need runtime secrets. | `True`     |

Returns:

| Type    | Description              |
| ------- | ------------------------ |
| `Agent` | Validated Agent instance |

Raises:

| Type                | Description                 |
| ------------------- | --------------------------- |
| `FileNotFoundError` | If file doesn't exist       |
| `ConfigError`       | If YAML parsing fails       |
| `ValidationError`   | If configuration is invalid |

Source code in `src/holodeck/config/loader.py`

```
def load_agent_yaml(self, file_path: str, substitute_env: bool = True) -> Agent:
    """Load and validate an agent configuration from YAML.

    This method:
    1. Parses the YAML file with env var substitution (single pass)
    2. Loads and merges user + project configs (project overrides user)
    3. Merges global config into agent config
    4. Validates against Agent schema
    5. Returns an Agent instance

    Configuration precedence (highest to lowest):
    1. agent.yaml explicit settings
    2. Environment variables
    3. Project-level config.yaml/config.yml
    4. Global ~/.holodeck/config.yaml/config.yml

    Args:
        file_path: Path to agent.yaml file
        substitute_env: When True (default), ``${VAR}`` references in both
            the agent YAML and any referenced ``test_cases_file`` are
            resolved against the process environment. When False, the raw
            ``${VAR}`` literal is preserved — used by structural consumers
            like ``holodeck deploy build`` that don't need runtime secrets.

    Returns:
        Validated Agent instance

    Raises:
        FileNotFoundError: If file doesn't exist
        ConfigError: If YAML parsing fails
        ValidationError: If configuration is invalid
    """
    path = Path(file_path)
    try:
        agent_config = _read_yaml_with_env_substitution(
            path, substitute_env=substitute_env
        )
    except OSError as e:
        raise FileNotFoundError(
            file_path,
            f"Configuration file not found at {file_path}. "
            f"Please ensure the file exists at this path.",
        ) from e
    except yaml.YAMLError as e:
        raise ConfigError(
            "yaml_parse",
            f"Failed to parse YAML file {file_path}: {str(e)}",
        ) from e

    if not agent_config:
        agent_config = {}

    # Load and deep-merge user + project configs (project overrides user)
    agent_dir = str(path.parent)
    user_config = self.load_global_config()
    project_config = self.load_project_config(agent_dir)
    config = self._merge_global_configs(user_config, project_config)

    # Merge configurations with proper precedence
    merged_config = self.merge_configs(agent_config, config)

    # Resolve external test_cases_file reference (if any) into inline
    # `test_cases` before schema validation. Agent model uses
    # `extra="forbid"`, so the key must be removed from the dict.
    _resolve_test_cases_file(
        merged_config, path.parent, substitute_env=substitute_env
    )

    # Expose the agent directory on ``sys.path`` so ``CodeMetric``'s
    # grader resolver (``importlib.import_module``) can locate user-land
    # grader packages (e.g. ``graders.my_benchmarks``) placed next to
    # ``agent.yaml``.
    agent_dir_abs = str(path.parent.resolve())
    if agent_dir_abs not in sys.path:
        sys.path.insert(0, agent_dir_abs)

    # Validate against Agent schema
    try:
        agent = Agent(**merged_config)
        return agent
    except PydanticValidationError as e:
        error_messages = flatten_pydantic_errors(e)
        error_text = "\n".join(error_messages)
        raise ConfigError(
            "agent_validation",
            f"Invalid agent configuration in {file_path}:\n{error_text}",
        ) from e
```

### `load_global_config()`

Load global configuration from ~/.holodeck/config.yml|config.yaml.

Results are cached after first load.

Returns:

| Type           | Description |
| -------------- | ----------- |
| \`GlobalConfig | None\`      |

Raises:

| Type          | Description                               |
| ------------- | ----------------------------------------- |
| `ConfigError` | If YAML parsing fails or validation fails |

Source code in `src/holodeck/config/loader.py`

```
def load_global_config(self) -> GlobalConfig | None:
    """Load global configuration from ~/.holodeck/config.yml|config.yaml.

    Results are cached after first load.

    Returns:
        GlobalConfig instance, or None if no config file exists

    Raises:
        ConfigError: If YAML parsing fails or validation fails
    """
    if self._user_config_loaded:
        return self._user_config

    home_dir = Path.home()
    holodeck_dir = home_dir / ".holodeck"
    result = self._load_config_file(
        holodeck_dir, "global_config", "global configuration"
    )
    self._user_config = result
    self._user_config_loaded = True
    return result
```

### `load_project_config(project_dir)`

Load project-level configuration from config.yml|config.yaml.

Results are cached per project_dir after first load.

Parameters:

| Name          | Type  | Description                    | Default    |
| ------------- | ----- | ------------------------------ | ---------- |
| `project_dir` | `str` | Path to project root directory | *required* |

Returns:

| Type           | Description |
| -------------- | ----------- |
| \`GlobalConfig | None\`      |

Raises:

| Type          | Description                               |
| ------------- | ----------------------------------------- |
| `ConfigError` | If YAML parsing fails or validation fails |

Source code in `src/holodeck/config/loader.py`

```
def load_project_config(self, project_dir: str) -> GlobalConfig | None:
    """Load project-level configuration from config.yml|config.yaml.

    Results are cached per project_dir after first load.

    Args:
        project_dir: Path to project root directory

    Returns:
        GlobalConfig instance, or None if no config file exists

    Raises:
        ConfigError: If YAML parsing fails or validation fails
    """
    if project_dir in self._project_configs:
        return self._project_configs[project_dir]

    project_path = Path(project_dir)
    result = self._load_config_file(
        project_path, "project_config", "project configuration"
    )
    self._project_configs[project_dir] = result
    return result
```

### `merge_configs(agent_config, global_config)`

Merge agent config with global config using proper precedence.

Precedence (highest to lowest):

1. agent.yaml explicit settings
1. Environment variables (already substituted)
1. Global settings (merged user + project)

Merges:

- Global LLM provider configs into agent model and evaluation model
- Global vectorstore configs into tool database fields (by name reference)

Keys don't get overwritten if they already exist in the agent config.

Parameters:

| Name            | Type             | Description                   | Default                                       |
| --------------- | ---------------- | ----------------------------- | --------------------------------------------- |
| `agent_config`  | `dict[str, Any]` | Configuration from agent.yaml | *required*                                    |
| `global_config` | \`GlobalConfig   | None\`                        | GlobalConfig instance (merged user + project) |

Returns:

| Type             | Description                     |
| ---------------- | ------------------------------- |
| `dict[str, Any]` | Merged configuration dictionary |

Source code in `src/holodeck/config/loader.py`

```
def merge_configs(
    self, agent_config: dict[str, Any], global_config: GlobalConfig | None
) -> dict[str, Any]:
    """Merge agent config with global config using proper precedence.

    Precedence (highest to lowest):
    1. agent.yaml explicit settings
    2. Environment variables (already substituted)
    3. Global settings (merged user + project)

    Merges:
    - Global LLM provider configs into agent model and evaluation model
    - Global vectorstore configs into tool database fields (by name reference)

    Keys don't get overwritten if they already exist in the agent config.

    Args:
        agent_config: Configuration from agent.yaml
        global_config: GlobalConfig instance (merged user + project)

    Returns:
        Merged configuration dictionary
    """
    if not agent_config:
        return {}

    if not global_config:
        return agent_config

    # Merge LLM provider configs (dict key lookup with .provider fallback)
    if "model" in agent_config and global_config.providers:
        agent_model_provider = agent_config["model"].get("provider")
        if agent_model_provider:
            _merge_provider_into_model(
                agent_config["model"],
                agent_model_provider,
                global_config.providers,
            )

        # Also merge global provider config to evaluation model
        if (
            "evaluations" in agent_config
            and isinstance(agent_config["evaluations"], dict)
            and "model" in agent_config["evaluations"]
            and isinstance(agent_config["evaluations"]["model"], dict)
        ):
            eval_model: dict[str, Any] = agent_config["evaluations"]["model"]
            eval_model_provider = eval_model.get("provider")
            if eval_model_provider:
                _merge_provider_into_model(
                    eval_model,
                    eval_model_provider,
                    global_config.providers,
                )

    # Resolve vectorstore references in tools
    if (
        global_config.vectorstores
        and "tools" in agent_config
        and isinstance(agent_config["tools"], list)
    ):
        self._resolve_vectorstore_references(
            agent_config["tools"], global_config.vectorstores
        )

    # Merge global MCP servers into agent tools
    if global_config.mcp_servers and len(global_config.mcp_servers) > 0:
        tools_missing = "tools" not in agent_config
        tools_invalid = not isinstance(agent_config.get("tools"), list)
        if tools_missing or tools_invalid:
            agent_config["tools"] = []

        self._merge_mcp_servers(agent_config["tools"], global_config.mcp_servers)

    # Merge global deployment config (agent.yaml takes precedence)
    if global_config.deployment:
        if "deployment" not in agent_config:
            agent_config["deployment"] = global_config.deployment.model_dump(
                exclude_unset=True
            )
        else:
            global_deploy = global_config.deployment.model_dump(exclude_unset=True)
            agent_deploy = agent_config["deployment"]
            if isinstance(agent_deploy, dict):
                _deep_merge(global_deploy, agent_deploy)
                agent_config["deployment"] = global_deploy

    return agent_config
```

### `resolve_file_path(file_path, base_dir)`

Resolve a file path relative to base directory.

Parameters:

| Name        | Type  | Description                                 | Default    |
| ----------- | ----- | ------------------------------------------- | ---------- |
| `file_path` | `str` | Path to resolve (absolute or relative)      | *required* |
| `base_dir`  | `str` | Base directory for relative path resolution | *required* |

Returns:

| Type  | Description               |
| ----- | ------------------------- |
| `str` | Absolute path to the file |

Raises:

| Type                | Description                        |
| ------------------- | ---------------------------------- |
| `FileNotFoundError` | If the resolved file doesn't exist |

Source code in `src/holodeck/config/loader.py`

```
def resolve_file_path(self, file_path: str, base_dir: str) -> str:
    """Resolve a file path relative to base directory.

    Args:
        file_path: Path to resolve (absolute or relative)
        base_dir: Base directory for relative path resolution

    Returns:
        Absolute path to the file

    Raises:
        FileNotFoundError: If the resolved file doesn't exist
    """
    path = Path(file_path)

    if path.is_absolute():
        resolved = path
    else:
        resolved = (Path(base_dir) / file_path).resolve()

    if not resolved.exists():
        raise FileNotFoundError(
            str(resolved),
            f"Referenced file not found: {resolved}\n"
            f"Please ensure the file exists at this path.",
        )

    return str(resolved)
```

### `resolve_execution_config(cli_config, yaml_config, project_config, user_config, defaults)`

Resolve execution configuration with priority hierarchy.

Configuration priority (highest to lowest):

1. CLI flags (cli_config)
1. agent.yaml execution section (yaml_config)
1. Project config execution section (project_config from ./config.yaml)
1. User config execution section (user_config from ~/.holodeck/config.yaml)
1. Environment variables (HOLODECK\_\* vars)
1. Built-in defaults

Parameters:

| Name             | Type              | Description                  | Default                                                  |
| ---------------- | ----------------- | ---------------------------- | -------------------------------------------------------- |
| `cli_config`     | \`ExecutionConfig | None\`                       | Execution config from CLI flags (optional)               |
| `yaml_config`    | \`ExecutionConfig | None\`                       | Execution config from agent.yaml (optional)              |
| `project_config` | \`ExecutionConfig | None\`                       | Execution config from project config.yaml (optional)     |
| `user_config`    | \`ExecutionConfig | None\`                       | Execution config from ~/.holodeck/config.yaml (optional) |
| `defaults`       | `dict[str, Any]`  | Dictionary of default values | *required*                                               |

Returns:

| Type              | Description                                        |
| ----------------- | -------------------------------------------------- |
| `ExecutionConfig` | Resolved ExecutionConfig with all fields populated |

Source code in `src/holodeck/config/loader.py`

```
def resolve_execution_config(
    self,
    cli_config: ExecutionConfig | None,
    yaml_config: ExecutionConfig | None,
    project_config: ExecutionConfig | None,
    user_config: ExecutionConfig | None,
    defaults: dict[str, Any],
) -> ExecutionConfig:
    """Resolve execution configuration with priority hierarchy.

    Configuration priority (highest to lowest):
    1. CLI flags (cli_config)
    2. agent.yaml execution section (yaml_config)
    3. Project config execution section (project_config from ./config.yaml)
    4. User config execution section (user_config from ~/.holodeck/config.yaml)
    5. Environment variables (HOLODECK_* vars)
    6. Built-in defaults

    Args:
        cli_config: Execution config from CLI flags (optional)
        yaml_config: Execution config from agent.yaml (optional)
        project_config: Execution config from project config.yaml (optional)
        user_config: Execution config from ~/.holodeck/config.yaml (optional)
        defaults: Dictionary of default values

    Returns:
        Resolved ExecutionConfig with all fields populated
    """
    resolved: dict[str, Any] = {}

    fields = list(ExecutionConfig.model_fields.keys())

    for field in fields:
        # Priority 1: CLI flag
        if cli_config and getattr(cli_config, field, None) is not None:
            resolved[field] = getattr(cli_config, field)
        # Priority 2: agent.yaml execution section
        elif yaml_config and getattr(yaml_config, field, None) is not None:
            resolved[field] = getattr(yaml_config, field)
        # Priority 3: Project config execution section
        elif project_config and getattr(project_config, field, None) is not None:
            resolved[field] = getattr(project_config, field)
        # Priority 4: User config execution section (~/.holodeck/)
        elif user_config and getattr(user_config, field, None) is not None:
            resolved[field] = getattr(user_config, field)
        # Priority 5: Environment variable
        elif (env_value := _get_env_value(field, os.environ)) is not None:
            resolved[field] = env_value
        # Priority 6: Built-in default
        else:
            resolved[field] = defaults.get(field)

    return ExecutionConfig(**resolved)
```

## Module-Level Functions (loader)

### load_agent_with_config

Convenience function that creates a `ConfigLoader`, loads the agent, sets the `agent_base_dir` context variable, and resolves the execution config in one call.

## `load_agent_with_config(agent_config_path, cli_config=None)`

Load agent and resolve execution config in one call.

Encapsulates the config-loading boilerplate shared across CLI commands:

- Creates ConfigLoader, loads agent.yaml
- Sets agent_base_dir context variable
- Resolves execution config with full priority hierarchy
- Returns (agent, resolved_config, loader)

Parameters:

| Name                | Type              | Description             | Default                                  |
| ------------------- | ----------------- | ----------------------- | ---------------------------------------- |
| `agent_config_path` | `str`             | Path to agent.yaml file | *required*                               |
| `cli_config`        | \`ExecutionConfig | None\`                  | Optional execution config from CLI flags |

Returns:

| Type                                          | Description                                              |
| --------------------------------------------- | -------------------------------------------------------- |
| `tuple[Agent, ExecutionConfig, ConfigLoader]` | Tuple of (Agent, resolved ExecutionConfig, ConfigLoader) |

Source code in `src/holodeck/config/loader.py`

```
def load_agent_with_config(
    agent_config_path: str,
    cli_config: ExecutionConfig | None = None,
) -> tuple[Agent, ExecutionConfig, ConfigLoader]:
    """Load agent and resolve execution config in one call.

    Encapsulates the config-loading boilerplate shared across CLI commands:
    - Creates ConfigLoader, loads agent.yaml
    - Sets agent_base_dir context variable
    - Resolves execution config with full priority hierarchy
    - Returns (agent, resolved_config, loader)

    Args:
        agent_config_path: Path to agent.yaml file
        cli_config: Optional execution config from CLI flags

    Returns:
        Tuple of (Agent, resolved ExecutionConfig, ConfigLoader)
    """
    from holodeck.config.context import agent_base_dir
    from holodeck.config.defaults import DEFAULT_EXECUTION_CONFIG

    loader = ConfigLoader()
    agent = loader.load_agent_yaml(agent_config_path)

    # Set the base directory context for resolving relative paths in tools
    agent_dir = str(Path(agent_config_path).parent.resolve())
    agent_base_dir.set(agent_dir)

    # Resolve execution config (CLI > agent.yaml > project > user > env > defaults)
    # Configs are already cached from load_agent_yaml, no duplicate I/O
    project_config = loader.load_project_config(agent_dir)
    project_execution = project_config.execution if project_config else None
    user_config = loader.load_global_config()
    user_execution = user_config.execution if user_config else None

    resolved_config = loader.resolve_execution_config(
        cli_config=cli_config,
        yaml_config=agent.execution,
        project_config=project_execution,
        user_config=user_execution,
        defaults=DEFAULT_EXECUTION_CONFIG,
    )

    return agent, resolved_config, loader
```

### save_global_config

## `save_global_config(config, path=None)`

Save GlobalConfig to ~/.holodeck/config.yaml.

Creates the ~/.holodeck/ directory if it doesn't exist. Preserves existing fields when updating.

Parameters:

| Name     | Type           | Description                   | Default                                                    |
| -------- | -------------- | ----------------------------- | ---------------------------------------------------------- |
| `config` | `GlobalConfig` | GlobalConfig instance to save | *required*                                                 |
| `path`   | \`Path         | None\`                        | Optional custom path (defaults to ~/.holodeck/config.yaml) |

Returns:

| Type   | Description                            |
| ------ | -------------------------------------- |
| `Path` | Path where the configuration was saved |

Raises:

| Type          | Description         |
| ------------- | ------------------- |
| `ConfigError` | If file write fails |

Source code in `src/holodeck/config/loader.py`

```
def save_global_config(
    config: GlobalConfig,
    path: Path | None = None,
) -> Path:
    """Save GlobalConfig to ~/.holodeck/config.yaml.

    Creates the ~/.holodeck/ directory if it doesn't exist.
    Preserves existing fields when updating.

    Args:
        config: GlobalConfig instance to save
        path: Optional custom path (defaults to ~/.holodeck/config.yaml)

    Returns:
        Path where the configuration was saved

    Raises:
        ConfigError: If file write fails
    """
    if path is None:
        path = Path.home() / ".holodeck" / "config.yaml"

    try:
        path.parent.mkdir(parents=True, exist_ok=True)

        config_dict = config.model_dump(
            exclude_unset=True, exclude_none=True, mode="json"
        )

        yaml_content = yaml.dump(
            config_dict,
            default_flow_style=False,
            sort_keys=False,
            allow_unicode=True,
        )

        path.write_text(yaml_content, encoding="utf-8")
        logger.debug(f"Saved global configuration to {path}")
        return path

    except OSError as e:
        raise ConfigError(
            "global_config_write",
            f"Failed to write global configuration to {path}: {e}",
        ) from e
```

### MCP Server Helpers

Functions for adding, removing, and listing MCP servers in agent and global configs.

## `add_mcp_server_to_agent(agent_path, mcp_tool)`

Add an MCP server to agent.yaml tools list.

Parameters:

| Name         | Type      | Description                  | Default    |
| ------------ | --------- | ---------------------------- | ---------- |
| `agent_path` | `Path`    | Path to agent.yaml file      | *required* |
| `mcp_tool`   | `MCPTool` | MCPTool configuration to add | *required* |

Raises:

| Type                   | Description                      |
| ---------------------- | -------------------------------- |
| `FileNotFoundError`    | If agent.yaml doesn't exist      |
| `DuplicateServerError` | If server already configured     |
| `ConfigError`          | If YAML parsing or writing fails |

Source code in `src/holodeck/config/loader.py`

```
def add_mcp_server_to_agent(
    agent_path: Path,
    mcp_tool: MCPTool,
) -> None:
    """Add an MCP server to agent.yaml tools list.

    Args:
        agent_path: Path to agent.yaml file
        mcp_tool: MCPTool configuration to add

    Raises:
        FileNotFoundError: If agent.yaml doesn't exist
        DuplicateServerError: If server already configured
        ConfigError: If YAML parsing or writing fails
    """
    loader = ConfigLoader()

    try:
        agent_config = loader.parse_yaml(str(agent_path))
    except FileNotFoundError as e:
        raise FileNotFoundError(
            str(agent_path),
            "No agent.yaml found. Use --agent to specify a file "
            "or -g for global install.",
        ) from e

    if agent_config is None:
        agent_config = {}

    if "tools" not in agent_config:
        agent_config["tools"] = []

    _check_mcp_duplicate(agent_config["tools"], mcp_tool)

    tool_dict = mcp_tool.model_dump(exclude_unset=True, exclude_none=True, mode="json")

    agent_config["tools"].append(tool_dict)

    try:
        yaml_content = yaml.dump(
            agent_config,
            default_flow_style=False,
            sort_keys=False,
            allow_unicode=True,
        )
        agent_path.write_text(yaml_content, encoding="utf-8")
        logger.debug(f"Added MCP server '{mcp_tool.name}' to {agent_path}")

    except OSError as e:
        raise ConfigError(
            "agent_config_write",
            f"Failed to write agent configuration to {agent_path}: {e}",
        ) from e
```

## `add_mcp_server_to_global(mcp_tool, global_path=None)`

Add an MCP server to global config mcp_servers list.

Parameters:

| Name          | Type      | Description                  | Default                                                    |
| ------------- | --------- | ---------------------------- | ---------------------------------------------------------- |
| `mcp_tool`    | `MCPTool` | MCPTool configuration to add | *required*                                                 |
| `global_path` | \`Path    | None\`                       | Optional custom path (defaults to ~/.holodeck/config.yaml) |

Returns:

| Type   | Description                            |
| ------ | -------------------------------------- |
| `Path` | Path where the configuration was saved |

Raises:

| Type                   | Description                      |
| ---------------------- | -------------------------------- |
| `DuplicateServerError` | If server already configured     |
| `ConfigError`          | If YAML parsing or writing fails |

Source code in `src/holodeck/config/loader.py`

```
def add_mcp_server_to_global(
    mcp_tool: MCPTool,
    global_path: Path | None = None,
) -> Path:
    """Add an MCP server to global config mcp_servers list.

    Args:
        mcp_tool: MCPTool configuration to add
        global_path: Optional custom path (defaults to ~/.holodeck/config.yaml)

    Returns:
        Path where the configuration was saved

    Raises:
        DuplicateServerError: If server already configured
        ConfigError: If YAML parsing or writing fails
    """
    if global_path is None:
        global_path = Path.home() / ".holodeck" / "config.yaml"

    loader = ConfigLoader()
    global_config = loader.load_global_config()

    if global_config is None:
        global_config = GlobalConfig(
            providers=None,
            vectorstores=None,
            execution=None,
            deployment=None,
            mcp_servers=None,
        )

    if global_config.mcp_servers is None:
        global_config.mcp_servers = []

    existing_tools = [t.model_dump(mode="json") for t in global_config.mcp_servers]

    _check_mcp_duplicate(existing_tools, mcp_tool)

    global_config.mcp_servers.append(mcp_tool)

    return save_global_config(global_config, global_path)
```

## `remove_mcp_server_from_agent(agent_path, server_name)`

Remove an MCP server from agent.yaml tools list.

Parameters:

| Name          | Type   | Description                      | Default    |
| ------------- | ------ | -------------------------------- | ---------- |
| `agent_path`  | `Path` | Path to agent.yaml file          | *required* |
| `server_name` | `str`  | Name of the MCP server to remove | *required* |

Raises:

| Type                  | Description                          |
| --------------------- | ------------------------------------ |
| `FileNotFoundError`   | If agent.yaml doesn't exist          |
| `ServerNotFoundError` | If server not found in configuration |
| `ConfigError`         | If YAML parsing or writing fails     |

Source code in `src/holodeck/config/loader.py`

```
def remove_mcp_server_from_agent(
    agent_path: Path,
    server_name: str,
) -> None:
    """Remove an MCP server from agent.yaml tools list.

    Args:
        agent_path: Path to agent.yaml file
        server_name: Name of the MCP server to remove

    Raises:
        FileNotFoundError: If agent.yaml doesn't exist
        ServerNotFoundError: If server not found in configuration
        ConfigError: If YAML parsing or writing fails
    """
    loader = ConfigLoader()

    try:
        agent_config = loader.parse_yaml(str(agent_path))
    except FileNotFoundError as e:
        raise FileNotFoundError(
            str(agent_path),
            f"Agent file not found: {agent_path}",
        ) from e

    if agent_config is None:
        agent_config = {}

    tools = agent_config.get("tools", [])

    original_len = len(tools)
    tools = [
        tool
        for tool in tools
        if not (tool.get("type") == "mcp" and tool.get("name") == server_name)
    ]

    if len(tools) == original_len:
        raise ServerNotFoundError(server_name, str(agent_path))

    agent_config["tools"] = tools

    try:
        yaml_content = yaml.dump(
            agent_config,
            default_flow_style=False,
            sort_keys=False,
            allow_unicode=True,
        )
        agent_path.write_text(yaml_content, encoding="utf-8")
        logger.debug(f"Removed MCP server '{server_name}' from {agent_path}")

    except OSError as e:
        raise ConfigError(
            "agent_config_write",
            f"Failed to write agent configuration to {agent_path}: {e}",
        ) from e
```

## `remove_mcp_server_from_global(server_name, global_path=None)`

Remove an MCP server from global config mcp_servers list.

Parameters:

| Name          | Type   | Description                      | Default                                                    |
| ------------- | ------ | -------------------------------- | ---------------------------------------------------------- |
| `server_name` | `str`  | Name of the MCP server to remove | *required*                                                 |
| `global_path` | \`Path | None\`                           | Optional custom path (defaults to ~/.holodeck/config.yaml) |

Returns:

| Type   | Description                            |
| ------ | -------------------------------------- |
| `Path` | Path where the configuration was saved |

Raises:

| Type                  | Description                          |
| --------------------- | ------------------------------------ |
| `ServerNotFoundError` | If server not found in configuration |
| `ConfigError`         | If YAML parsing or writing fails     |

Source code in `src/holodeck/config/loader.py`

```
def remove_mcp_server_from_global(
    server_name: str,
    global_path: Path | None = None,
) -> Path:
    """Remove an MCP server from global config mcp_servers list.

    Args:
        server_name: Name of the MCP server to remove
        global_path: Optional custom path (defaults to ~/.holodeck/config.yaml)

    Returns:
        Path where the configuration was saved

    Raises:
        ServerNotFoundError: If server not found in configuration
        ConfigError: If YAML parsing or writing fails
    """
    if global_path is None:
        global_path = Path.home() / ".holodeck" / "config.yaml"

    loader = ConfigLoader()
    if global_path.exists():
        raw_config = loader.parse_yaml(str(global_path))
        if raw_config is None:
            raise ServerNotFoundError(server_name, "global configuration")

        mcp_servers_raw = [
            s for s in raw_config.get("mcp_servers", []) if s.get("type") == "mcp"
        ]

        original_len = len(mcp_servers_raw)
        mcp_servers_raw = [s for s in mcp_servers_raw if s.get("name") != server_name]

        if len(mcp_servers_raw) == original_len:
            raise ServerNotFoundError(server_name, "global configuration")

        mcp_servers = (
            [MCPTool(**s) for s in mcp_servers_raw] if mcp_servers_raw else None
        )

        global_config = GlobalConfig(
            providers=raw_config.get("providers"),
            vectorstores=raw_config.get("vectorstores"),
            execution=raw_config.get("execution"),
            deployment=raw_config.get("deployment"),
            mcp_servers=mcp_servers,
        )
    else:
        raise ServerNotFoundError(server_name, "global configuration")

    return save_global_config(global_config, global_path)
```

## `get_mcp_servers_from_agent(agent_path)`

Get all MCP servers from agent.yaml tools list.

Parameters:

| Name         | Type   | Description             | Default    |
| ------------ | ------ | ----------------------- | ---------- |
| `agent_path` | `Path` | Path to agent.yaml file | *required* |

Returns:

| Type            | Description                                                            |
| --------------- | ---------------------------------------------------------------------- |
| `list[MCPTool]` | List of MCPTool objects from agent config (empty list if no MCP tools) |

Raises:

| Type                | Description                     |
| ------------------- | ------------------------------- |
| `FileNotFoundError` | If agent file doesn't exist     |
| `ConfigError`       | If agent config is invalid YAML |

Source code in `src/holodeck/config/loader.py`

```
def get_mcp_servers_from_agent(agent_path: Path) -> list[MCPTool]:
    """Get all MCP servers from agent.yaml tools list.

    Args:
        agent_path: Path to agent.yaml file

    Returns:
        List of MCPTool objects from agent config (empty list if no MCP tools)

    Raises:
        FileNotFoundError: If agent file doesn't exist
        ConfigError: If agent config is invalid YAML
    """
    loader = ConfigLoader()

    try:
        agent_config = loader.parse_yaml(str(agent_path))
    except FileNotFoundError as e:
        raise FileNotFoundError(
            str(agent_path),
            f"Agent file not found: {agent_path}",
        ) from e

    if agent_config is None:
        return []

    tools = agent_config.get("tools", [])
    if not tools:
        return []

    mcp_servers: list[MCPTool] = []
    for tool in tools:
        if not isinstance(tool, dict):
            continue
        if tool.get("type") != "mcp":
            continue

        try:
            mcp_tool = MCPTool(**tool)
            mcp_servers.append(mcp_tool)
        except PydanticValidationError as e:
            logger.warning(
                f"Failed to parse MCP tool '{tool.get('name', 'unknown')}': {e}"
            )
            continue

    return mcp_servers
```

## `get_mcp_servers_from_global(global_path=None)`

Get all MCP servers from global config.

Parameters:

| Name          | Type   | Description | Default                                                           |
| ------------- | ------ | ----------- | ----------------------------------------------------------------- |
| `global_path` | \`Path | None\`      | Optional path to global config (default: ~/.holodeck/config.yaml) |

Returns:

| Type            | Description                                                                     |
| --------------- | ------------------------------------------------------------------------------- |
| `list[MCPTool]` | List of MCPTool objects from global config (empty list if no config or servers) |

Source code in `src/holodeck/config/loader.py`

```
def get_mcp_servers_from_global(global_path: Path | None = None) -> list[MCPTool]:
    """Get all MCP servers from global config.

    Args:
        global_path: Optional path to global config (default: ~/.holodeck/config.yaml)

    Returns:
        List of MCPTool objects from global config (empty list if no config or servers)
    """
    loader = ConfigLoader()

    global_config = loader.load_global_config()

    if global_config is None:
        return []

    if global_config.mcp_servers is None:
        return []

    return global_config.mcp_servers
```

______________________________________________________________________

## Environment Variable Utilities

Support for dynamic configuration using environment variables with the `${VAR_NAME}` pattern.

## `substitute_env_vars(text)`

Substitute environment variables in text using ${VAR_NAME} pattern.

Replaces all occurrences of ${VAR_NAME} with the corresponding environment variable value. Raises ConfigError if a referenced variable does not exist.

Parameters:

| Name   | Type  | Description                                      | Default    |
| ------ | ----- | ------------------------------------------------ | ---------- |
| `text` | `str` | Text potentially containing ${VAR_NAME} patterns | *required* |

Returns:

| Type  | Description                                     |
| ----- | ----------------------------------------------- |
| `str` | Text with all environment variables substituted |

Raises:

| Type          | Description                                         |
| ------------- | --------------------------------------------------- |
| `ConfigError` | If a referenced environment variable does not exist |

Example

> > > import os os.environ["API_KEY"] = "secret123" substitute_env_vars("key: ${API_KEY}") 'key: secret123'

Source code in `src/holodeck/config/env_loader.py`

```
def substitute_env_vars(text: str) -> str:
    """Substitute environment variables in text using ${VAR_NAME} pattern.

    Replaces all occurrences of ${VAR_NAME} with the corresponding environment
    variable value. Raises ConfigError if a referenced variable does not exist.

    Args:
        text: Text potentially containing ${VAR_NAME} patterns

    Returns:
        Text with all environment variables substituted

    Raises:
        ConfigError: If a referenced environment variable does not exist

    Example:
        >>> import os
        >>> os.environ["API_KEY"] = "secret123"
        >>> substitute_env_vars("key: ${API_KEY}")
        'key: secret123'
    """
    # Short-circuit: skip regex if no env var markers present
    if "${" not in text:
        return text

    # Pattern to match ${VAR_NAME} - captures alphanumeric, underscore
    pattern = r"\$\{([A-Za-z_][A-Za-z0-9_]*)\}"

    def replace_var(match: re.Match[str]) -> str:
        """Replace a single ${VAR_NAME} pattern with env value.

        Args:
            match: Regex match object for ${VAR_NAME}

        Returns:
            Environment variable value

        Raises:
            ConfigError: If variable does not exist
        """
        var_name = match.group(1)
        if var_name not in os.environ:
            raise ConfigError(
                var_name,
                f"Environment variable '{var_name}' not found. "
                f"Please set it and try again.",
            )
        return os.environ[var_name]

    return re.sub(pattern, replace_var, text)
```

## `get_env_var(key, default=None)`

Get environment variable with optional default.

Parameters:

| Name      | Type  | Description                       | Default    |
| --------- | ----- | --------------------------------- | ---------- |
| `key`     | `str` | Environment variable name         | *required* |
| `default` | `Any` | Default value if variable not set | `None`     |

Returns:

| Type  | Description                           |
| ----- | ------------------------------------- |
| `Any` | Environment variable value or default |

Source code in `src/holodeck/config/env_loader.py`

```
def get_env_var(key: str, default: Any = None) -> Any:
    """Get environment variable with optional default.

    Args:
        key: Environment variable name
        default: Default value if variable not set

    Returns:
        Environment variable value or default
    """
    return os.environ.get(key, default)
```

## `load_env_file(path)`

Load environment variables from a .env file.

Parameters:

| Name   | Type  | Description       | Default    |
| ------ | ----- | ----------------- | ---------- |
| `path` | `str` | Path to .env file | *required* |

Returns:

| Type             | Description                                |
| ---------------- | ------------------------------------------ |
| `dict[str, str]` | Dictionary of loaded environment variables |

Raises:

| Type          | Description            |
| ------------- | ---------------------- |
| `ConfigError` | If file cannot be read |

Source code in `src/holodeck/config/env_loader.py`

```
def load_env_file(path: str) -> dict[str, str]:
    """Load environment variables from a .env file.

    Args:
        path: Path to .env file

    Returns:
        Dictionary of loaded environment variables

    Raises:
        ConfigError: If file cannot be read
    """
    try:
        env_vars = {}
        with open(path) as f:
            for line in f:
                line = line.strip()
                if not line or line.startswith("#"):
                    continue
                if "=" in line:
                    key, value = line.split("=", 1)
                    env_vars[key.strip()] = value.strip()
        return env_vars
    except OSError as e:
        raise ConfigError("env_file", f"Cannot read environment file: {e}") from e
```

______________________________________________________________________

## Configuration Validation

Utility for converting Pydantic validation errors into user-friendly messages.

## `flatten_pydantic_errors(exc)`

Flatten Pydantic ValidationError into human-readable messages.

Converts Pydantic's nested error structure into a flat list of user-friendly error messages that include field names and descriptions.

Parameters:

| Name  | Type              | Description                        | Default    |
| ----- | ----------------- | ---------------------------------- | ---------- |
| `exc` | `ValidationError` | Pydantic ValidationError exception | *required* |

Returns:

| Type        | Description                                                |
| ----------- | ---------------------------------------------------------- |
| `list[str]` | List of human-readable error messages, one per field error |

Example

> > > from pydantic import BaseModel, ValidationError class Model(BaseModel): ... name: str try: ... Model(name=123) ... except ValidationError as e: ... msgs = flatten_pydantic_errors(e) ... # msgs contains human-readable descriptions

Source code in `src/holodeck/config/validator.py`

```
def flatten_pydantic_errors(exc: PydanticValidationError) -> list[str]:
    """Flatten Pydantic ValidationError into human-readable messages.

    Converts Pydantic's nested error structure into a flat list of
    user-friendly error messages that include field names and descriptions.

    Args:
        exc: Pydantic ValidationError exception

    Returns:
        List of human-readable error messages, one per field error

    Example:
        >>> from pydantic import BaseModel, ValidationError
        >>> class Model(BaseModel):
        ...     name: str
        >>> try:
        ...     Model(name=123)
        ... except ValidationError as e:
        ...     msgs = flatten_pydantic_errors(e)
        ...     # msgs contains human-readable descriptions
    """
    errors: list[str] = []

    for error in exc.errors():
        loc = error.get("loc", ())
        field_path = ".".join(str(item) for item in loc) if loc else "unknown"

        msg = error.get("msg", "Unknown error")
        error_type = error.get("type", "")

        if error_type == "value_error":
            input_val = error.get("input")
            formatted = f"Field '{field_path}': {msg} (received: {input_val!r})"
        else:
            formatted = f"Field '{field_path}': {msg}"

        errors.append(formatted)

    return errors if errors else ["Validation failed with unknown error"]
```

______________________________________________________________________

## Default Configuration

Built-in default constants and embedding dimension resolution.

### Constants

| Constant                     | Type   | Description                                                                          |
| ---------------------------- | ------ | ------------------------------------------------------------------------------------ |
| `OLLAMA_DEFAULTS`            | `dict` | Default Ollama provider settings (endpoint, temperature, max_tokens, top_p, api_key) |
| `OLLAMA_EMBEDDING_DEFAULTS`  | `dict` | Default Ollama embedding model (`nomic-embed-text:latest`)                           |
| `DEFAULT_EXECUTION_CONFIG`   | `dict` | Default execution settings (timeouts, cache, verbosity)                              |
| `EMBEDDING_MODEL_DIMENSIONS` | `dict` | Known embedding model dimension mappings (OpenAI and Ollama models)                  |

### get_embedding_dimensions

## `get_embedding_dimensions(model_name, provider='openai')`

Get embedding dimensions for a model.

Resolution order:

1. Known model in EMBEDDING_MODEL_DIMENSIONS
1. Provider default (openai: 1536, ollama: 768)
1. Fallback to 1536 with warning

Parameters:

| Name         | Type  | Description                                       | Default                                               |
| ------------ | ----- | ------------------------------------------------- | ----------------------------------------------------- |
| `model_name` | \`str | None\`                                            | Embedding model name (e.g., "text-embedding-3-small") |
| `provider`   | `str` | LLM provider ("openai", "azure_openai", "ollama") | `'openai'`                                            |

Returns:

| Type  | Description                        |
| ----- | ---------------------------------- |
| `int` | Embedding dimensions for the model |

Source code in `src/holodeck/config/defaults.py`

```
def get_embedding_dimensions(
    model_name: str | None,
    provider: str = "openai",
) -> int:
    """Get embedding dimensions for a model.

    Resolution order:
    1. Known model in EMBEDDING_MODEL_DIMENSIONS
    2. Provider default (openai: 1536, ollama: 768)
    3. Fallback to 1536 with warning

    Args:
        model_name: Embedding model name (e.g., "text-embedding-3-small")
        provider: LLM provider ("openai", "azure_openai", "ollama")

    Returns:
        Embedding dimensions for the model
    """
    if model_name and model_name in EMBEDDING_MODEL_DIMENSIONS:
        return EMBEDDING_MODEL_DIMENSIONS[model_name]

    if provider == "ollama":
        if model_name:
            logger.warning(
                f"Unknown Ollama model '{model_name}', assuming 768 dimensions. "
                "Set 'embedding_dimensions' explicitly if different."
            )
        return 768

    if model_name:
        logger.warning(
            f"Unknown embedding model '{model_name}', assuming 1536 dimensions. "
            f"Supported: {', '.join(EMBEDDING_MODEL_DIMENSIONS.keys())}. "
            "Set 'embedding_dimensions' explicitly if different."
        )
    return 1536
```

______________________________________________________________________

## Configuration Context

Request-scoped context variable for passing the agent base directory through async call stacks without explicit parameter threading.

### agent_base_dir

```
from contextvars import ContextVar

agent_base_dir: ContextVar[str | None] = ContextVar("agent_base_dir", default=None)
```

Set at CLI entry points (e.g., `holodeck test`, `holodeck chat`) to the parent directory of `agent.yaml`. Read downstream by tools and resolvers that need to resolve relative file paths.

**Usage:**

```
# At CLI entry point:
from holodeck.config.context import agent_base_dir
agent_base_dir.set(str(Path(agent_yaml_path).parent))

# Anywhere downstream:
base_dir = agent_base_dir.get()  # Returns str | None
```

______________________________________________________________________

## ConfigManager

Manager class for configuration file operations: creating defaults, resolving paths, generating YAML, and writing config files.

## `ConfigManager`

Manager for configuration operations to improve testability.

### `create_default_config()`

Create a default GlobalConfig with sample settings.

Source code in `src/holodeck/config/manager.py`

```
@staticmethod
def create_default_config() -> GlobalConfig:
    """Create a default GlobalConfig with sample settings."""
    # Create a default LLM provider
    default_provider = LLMProvider(
        provider=ProviderEnum.OPENAI,
        name="gpt-4",
        temperature=0.3,
        max_tokens=1000,
        api_key=SecretStr("your-openai-api-key-here"),
        endpoint=None,
    )

    # Create a default vectorstore config
    default_vectorstore = VectorstoreConfig(
        provider="postgres",
        connection_string="postgresql://user:password@localhost:5432/vectorstore",
        options={"sslmode": "prefer"},
    )

    # Create a default execution config
    default_execution = ExecutionConfig(
        file_timeout=30,
        llm_timeout=30,
        download_timeout=30,
        cache_enabled=True,
        cache_dir=".cache",
        verbose=False,
        quiet=False,
    )

    # Create a default deployment config
    default_deployment = DeploymentConfig(
        registry=RegistryConfig(url="docker.io", repository="holodeck/agent"),
        target=CloudTargetConfig(
            provider=CloudProvider.AWS,
            aws=AWSAppRunnerConfig(region="us-east-1"),
        ),
    )

    # Create the global config
    return GlobalConfig(
        providers={"openai": default_provider},
        vectorstores={"postgres": default_vectorstore},
        execution=default_execution,
        deployment=default_deployment,
        mcp_servers=None,
    )
```

### `get_config_path(global_config, project_config)`

Determine the configuration file path and type.

Parameters:

| Name             | Type   | Description                           | Default    |
| ---------------- | ------ | ------------------------------------- | ---------- |
| `global_config`  | `bool` | Whether to use global configuration.  | *required* |
| `project_config` | `bool` | Whether to use project configuration. | *required* |

Returns:

| Type               | Description                              |
| ------------------ | ---------------------------------------- |
| `tuple[Path, str]` | Tuple of (config_path, config_type_name) |

Source code in `src/holodeck/config/manager.py`

```
@staticmethod
def get_config_path(global_config: bool, project_config: bool) -> tuple[Path, str]:
    """Determine the configuration file path and type.

    Args:
        global_config: Whether to use global configuration.
        project_config: Whether to use project configuration.

    Returns:
        Tuple of (config_path, config_type_name)
    """
    if global_config:
        return Path.home() / ".holodeck" / "config.yaml", "global"
    else:
        # Default to project config if neither or project specified
        return Path.cwd() / "config.yaml", "project"
```

### `generate_config_content(config)`

Generate YAML content for the configuration.

Source code in `src/holodeck/config/manager.py`

```
@staticmethod
def generate_config_content(config: GlobalConfig) -> str:
    """Generate YAML content for the configuration."""
    config_dict = config.model_dump(exclude_unset=True, mode="json")
    return yaml.dump(config_dict, default_flow_style=False, sort_keys=False)
```

### `write_config(path, content)`

Write configuration content to file.

Source code in `src/holodeck/config/manager.py`

```
@staticmethod
def write_config(path: Path, content: str) -> None:
    """Write configuration content to file."""
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        f.write(content)
```

______________________________________________________________________

## Schema Validation

JSON Schema validation for LLM response formats, aligned with OpenAI structured output requirements. Only Basic JSON Schema keywords are supported.

### ALLOWED_KEYWORDS

```
ALLOWED_KEYWORDS = {
    "type", "properties", "required", "additionalProperties",
    "items", "enum", "default", "description", "minimum", "maximum",
}
```

### SchemaValidator

## `SchemaValidator`

Validates JSON Schema definitions for response formats.

### `validate_schema(schema, schema_name='schema')`

Validate a JSON schema against Basic JSON Schema specification.

Parameters:

| Name          | Type             | Description                                       | Default                                |
| ------------- | ---------------- | ------------------------------------------------- | -------------------------------------- |
| `schema`      | \`dict[str, Any] | str\`                                             | Schema as dict (inline) or JSON string |
| `schema_name` | `str`            | Name for error messages (e.g., "response_format") | `'schema'`                             |

Returns:

| Type             | Description                    |
| ---------------- | ------------------------------ |
| `dict[str, Any]` | Validated schema as dictionary |

Raises:

| Type         | Description                                       |
| ------------ | ------------------------------------------------- |
| `ValueError` | If schema is invalid or uses unsupported keywords |

Source code in `src/holodeck/config/schema.py`

```
@staticmethod
def validate_schema(
    schema: dict[str, Any] | str, schema_name: str = "schema"
) -> dict[str, Any]:
    """Validate a JSON schema against Basic JSON Schema specification.

    Args:
        schema: Schema as dict (inline) or JSON string
        schema_name: Name for error messages (e.g., "response_format")

    Returns:
        Validated schema as dictionary

    Raises:
        ValueError: If schema is invalid or uses unsupported keywords
    """
    # Convert string to dict if needed
    if isinstance(schema, str):
        try:
            parsed = json.loads(schema)
            if not isinstance(parsed, dict):
                raise ValueError(f"Invalid JSON in {schema_name}: must be object")
            schema_dict = parsed
        except json.JSONDecodeError as e:
            raise ValueError(f"Invalid JSON in {schema_name}: {str(e)}") from e
    else:
        schema_dict = schema

    # Validate schema structure using our custom validation
    # We don't use jsonschema's check_schema() because it validates against
    # the full JSON Schema metaschema, which is stricter than what OpenAI's
    # structured output requires (e.g., additionalProperties: false is valid
    # for OpenAI but fails Draft 4/7/2020-12 metaschema validation)
    try:
        SchemaValidator._check_allowed_keywords(schema_dict, schema_name)
        SchemaValidator._validate_schema_structure(schema_dict, schema_name)
    except ValueError:
        raise
    except Exception as e:
        raise ValueError(f"Invalid {schema_name} schema: {str(e)}") from e

    return schema_dict
```

### `load_schema_from_file(file_path, base_dir=None)`

Load and validate a JSON schema from file.

Parameters:

| Name        | Type  | Description                                            | Default    |
| ----------- | ----- | ------------------------------------------------------ | ---------- |
| `file_path` | `str` | Path to schema file (relative to base_dir or absolute) | *required* |
| `base_dir`  | \`str | Path                                                   | None\`     |

Returns:

| Type             | Description                    |
| ---------------- | ------------------------------ |
| `dict[str, Any]` | Validated schema as dictionary |

Raises:

| Type                | Description                  |
| ------------------- | ---------------------------- |
| `FileNotFoundError` | If schema file doesn't exist |
| `ValueError`        | If schema is invalid         |

Source code in `src/holodeck/config/schema.py`

```
@staticmethod
def load_schema_from_file(
    file_path: str, base_dir: str | Path | None = None
) -> dict[str, Any]:
    """Load and validate a JSON schema from file.

    Args:
        file_path: Path to schema file (relative to base_dir or absolute)
        base_dir: Base directory for relative paths (defaults to cwd)

    Returns:
        Validated schema as dictionary

    Raises:
        FileNotFoundError: If schema file doesn't exist
        ValueError: If schema is invalid
    """
    # Resolve file path
    if base_dir is None:
        # Try to get from context variable
        from holodeck.config.context import agent_base_dir

        base_dir = agent_base_dir.get()

    base_dir = Path.cwd() if base_dir is None else Path(base_dir)

    path = Path(file_path)
    if not path.is_absolute():
        path = base_dir / file_path

    # Check file exists
    if not path.exists():
        raise FileNotFoundError(
            f"Schema file not found: {path}\n" f"Expected file at: {path.resolve()}"
        )

    # Read and parse JSON
    try:
        with open(path, encoding="utf-8") as f:
            loaded = json.load(f)
            if not isinstance(loaded, dict):
                raise ValueError(f"Schema file {path} must be JSON object")
            schema_dict = loaded
    except json.JSONDecodeError as e:
        raise ValueError(f"Invalid JSON in schema file {path}: {str(e)}") from e
    except OSError as e:
        raise FileNotFoundError(
            f"Failed to read schema file {path}: {str(e)}"
        ) from e

    # Validate schema
    SchemaValidator.validate_schema(schema_dict, f"schema file {path}")

    return schema_dict
```

______________________________________________________________________

## Related Documentation

- [Data Models](https://docs.useholodeck.ai/api/models/index.md): Configuration model definitions
- [CLI Commands](https://docs.useholodeck.ai/api/cli/index.md): CLI API reference
- [YAML Schema](https://docs.useholodeck.ai/guides/agent-configuration/index.md): Agent configuration YAML format

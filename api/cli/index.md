# CLI API Reference

HoloDeck provides a command-line interface for project initialization, agent testing, interactive chat, HTTP serving, deployment, MCP server management, and configuration. This section documents the programmatic CLI API -- every public class, function, and exception exposed by the `holodeck.cli` package.

______________________________________________________________________

## Main CLI

Entry point for the HoloDeck CLI application using Click. Registers all seven subcommands and loads `.env` files on startup.

## `main(ctx)`

HoloDeck - Experimentation platform for AI agents.

Commands

init Initialize a new agent project test Run agent test cases chat Interactive chat session with an agent serve Start an HTTP server exposing an agent deploy Build and deploy agent containers

Initialize and manage AI agent projects with YAML configuration.

Source code in `src/holodeck/cli/main.py`

```
@click.group(invoke_without_command=True)
@click.version_option(version=__version__, prog_name="holodeck")
@click.pass_context
def main(ctx: click.Context) -> None:
    """HoloDeck - Experimentation platform for AI agents.

    Commands:
        init    Initialize a new agent project
        test    Run agent test cases
        chat    Interactive chat session with an agent
        serve   Start an HTTP server exposing an agent
        deploy  Build and deploy agent containers

    Initialize and manage AI agent projects with YAML configuration.
    """
    # Show help if no command is provided
    if ctx.invoked_subcommand is None:
        click.echo(ctx.get_help())
```

## `_load_dotenv_files()`

Load .env files from current directory and user home.

Priority (highest to lowest):

1. Shell environment variables (never overwritten)
1. .env in CWD (project-level config)
1. ~/.holodeck/.env (user-level defaults)

With override=False, the first value set wins. So we load project .env first, then home .env fills any remaining gaps.

Source code in `src/holodeck/cli/main.py`

```
def _load_dotenv_files() -> None:
    """Load .env files from current directory and user home.

    Priority (highest to lowest):
    1. Shell environment variables (never overwritten)
    2. .env in CWD (project-level config)
    3. ~/.holodeck/.env (user-level defaults)

    With override=False, the first value set wins. So we load
    project .env first, then home .env fills any remaining gaps.
    """
    # Load project-level .env first (higher priority of .env files)
    project_env = Path.cwd() / ".env"
    if project_env.exists():
        load_dotenv(project_env, override=False)

    # Load user-level .env second (fills gaps, never overrides)
    user_env = Path.home() / ".holodeck" / ".env"
    if user_env.exists():
        load_dotenv(user_env, override=False)
```

______________________________________________________________________

## CLI Commands

### Init Command

Initialize a new HoloDeck project with bundled templates and an interactive wizard.

## `init(project_name, template, description, author, force, llm, vectorstore, evals_arg, mcp_arg, non_interactive, verbose, quiet)`

Initialize a new HoloDeck agent project.

Creates a new project directory with all required configuration files, example instructions, tools templates, test cases, and data files.

The generated project includes agent.yaml (main configuration), instructions/ (system prompts), tools/ (custom function templates), data/ (sample datasets), and tests/ (evaluation test cases).

TEMPLATES:

```
conversational  - General-purpose conversational agent (default)
research        - Research/analysis agent with vector search examples
customer-support - Customer support agent with function tools
```

INTERACTIVE MODE (default):

```
When run without --non-interactive, the wizard prompts for:
- Agent name
- LLM provider (Ollama, OpenAI, Azure OpenAI, Anthropic)
- Vector store (ChromaDB, Qdrant, In-Memory)
- Evaluation metrics
- MCP servers
```

NON-INTERACTIVE MODE:

```
Use --non-interactive with --name to skip prompts and use defaults:

    holodeck init --name my-agent --non-interactive

Or override specific values:

    holodeck init --name my-agent --llm openai --vectorstore qdrant
```

EXAMPLES:

```
Basic project with interactive wizard:

    holodeck init

Quick setup with defaults (no prompts):

    holodeck init --name my-agent --non-interactive

Custom LLM and vector store:

    holodeck init --name my-agent --llm openai --vectorstore qdrant

Full customization without prompts:

    holodeck init --name my-agent --llm anthropic \
        --vectorstore chromadb --evals rag-faithfulness,rag-answer_relevancy \
        --mcp brave-search,memory --non-interactive
```

For more information, see: https://useholodeck.ai/docs/getting-started

Source code in `src/holodeck/cli/commands/init.py`

```
@click.command(name="init")
@click.option(
    "--name",
    "project_name",
    default=None,
    help="Agent/project name (required in non-interactive mode)",
)
@click.option(
    "--template",
    default="conversational",
    type=str,
    callback=validate_template,
    help="Project template: conversational (default), research, or customer-support",
)
@click.option(
    "--description",
    default=None,
    help="Brief description of what the agent does",
)
@click.option(
    "--author",
    default=None,
    help="Name of the project creator or organization",
)
@click.option(
    "--force",
    is_flag=True,
    help="Overwrite existing project directory without prompting",
)
@click.option(
    "--llm",
    type=click.Choice(sorted(VALID_LLM_PROVIDERS)),
    default=None,
    help="LLM provider (skips interactive prompt)",
)
@click.option(
    "--vectorstore",
    type=click.Choice(sorted(VALID_VECTOR_STORES)),
    default=None,
    help="Vector store (skips interactive prompt)",
)
@click.option(
    "--evals",
    "evals_arg",
    default=None,
    help="Comma-separated evaluation metrics (skips interactive prompt)",
)
@click.option(
    "--mcp",
    "mcp_arg",
    default=None,
    help="Comma-separated MCP servers (skips interactive prompt)",
)
@click.option(
    "--non-interactive",
    is_flag=True,
    help="Skip all interactive prompts (use defaults or flag values)",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress INFO logging output",
)
def init(
    project_name: str | None,
    template: str,
    description: str | None,
    author: str | None,
    force: bool,
    llm: str | None,
    vectorstore: str | None,
    evals_arg: str | None,
    mcp_arg: str | None,
    non_interactive: bool,
    verbose: bool,
    quiet: bool,
) -> None:
    """Initialize a new HoloDeck agent project.

    Creates a new project directory with all required configuration files,
    example instructions, tools templates, test cases, and data files.

    The generated project includes agent.yaml (main configuration), instructions/
    (system prompts), tools/ (custom function templates), data/ (sample datasets),
    and tests/ (evaluation test cases).

    TEMPLATES:

        conversational  - General-purpose conversational agent (default)
        research        - Research/analysis agent with vector search examples
        customer-support - Customer support agent with function tools

    INTERACTIVE MODE (default):

        When run without --non-interactive, the wizard prompts for:
        - Agent name
        - LLM provider (Ollama, OpenAI, Azure OpenAI, Anthropic)
        - Vector store (ChromaDB, Qdrant, In-Memory)
        - Evaluation metrics
        - MCP servers

    NON-INTERACTIVE MODE:

        Use --non-interactive with --name to skip prompts and use defaults:

            holodeck init --name my-agent --non-interactive

        Or override specific values:

            holodeck init --name my-agent --llm openai --vectorstore qdrant

    EXAMPLES:

        Basic project with interactive wizard:

            holodeck init

        Quick setup with defaults (no prompts):

            holodeck init --name my-agent --non-interactive

        Custom LLM and vector store:

            holodeck init --name my-agent --llm openai --vectorstore qdrant

        Full customization without prompts:

            holodeck init --name my-agent --llm anthropic \\
                --vectorstore chromadb --evals rag-faithfulness,rag-answer_relevancy \\
                --mcp brave-search,memory --non-interactive

    For more information, see: https://useholodeck.ai/docs/getting-started
    """
    # Initialize logging
    setup_logging(verbose=verbose, quiet=quiet)
    logger.debug(
        f"Init command invoked: project_name={project_name}, template={template}, "
        f"non_interactive={non_interactive}"
    )

    try:
        # Get current working directory as output directory
        output_dir = Path.cwd()

        # Parse comma-separated arguments
        evals_list = _parse_comma_arg(evals_arg)
        mcp_list = _parse_comma_arg(mcp_arg)

        # Validate evals if provided
        if evals_list:
            invalid_evals = [e for e in evals_list if e not in VALID_EVALS]
            if invalid_evals:
                valid = ", ".join(sorted(VALID_EVALS))
                invalid_str = ", ".join(invalid_evals)
                click.secho(
                    f"Warning: Invalid eval(s): {invalid_str}. Valid: {valid}",
                    fg="yellow",
                )
                evals_list = [e for e in evals_list if e in VALID_EVALS]

        # Validate MCP servers if provided
        if mcp_list:
            invalid_mcp = [s for s in mcp_list if s not in VALID_MCP_SERVERS]
            if invalid_mcp:
                valid = ", ".join(sorted(VALID_MCP_SERVERS))
                click.secho(
                    f"Warning: Invalid MCP server(s): {', '.join(invalid_mcp)}. "
                    f"Valid options: {valid}",
                    fg="yellow",
                )
                mcp_list = [s for s in mcp_list if s in VALID_MCP_SERVERS]

        # Determine if we should run wizard
        if non_interactive or not is_interactive():
            # Non-interactive mode: --name is required
            if not project_name:
                click.secho(
                    "Error: --name is required in non-interactive mode",
                    fg="red",
                )
                raise click.Abort()

            # Use defaults or flag values
            selected_llm = llm or "ollama"

            # Create provider config for providers that require endpoint
            provider_config = None
            if selected_llm == "azure_openai":
                # Use env var placeholders for Azure OpenAI
                provider_config = ProviderConfig(
                    endpoint="${AZURE_OPENAI_ENDPOINT}",
                )

            wizard_result = WizardResult(
                agent_name=project_name,
                template=template,
                llm_provider=selected_llm,
                provider_config=provider_config,
                vector_store=vectorstore or "chromadb",
                evals=evals_list if evals_list else get_default_evals(),
                mcp_servers=mcp_list if mcp_list else get_default_mcp_servers(),
            )
        else:
            # Interactive mode: run wizard
            # Skip template prompt if --template was provided (not default)
            wizard_result = run_wizard(
                skip_agent_name=project_name is not None,
                skip_template=template != "conversational",
                skip_llm=llm is not None,
                skip_vectorstore=vectorstore is not None,
                skip_evals=evals_arg is not None,
                skip_mcp=mcp_arg is not None,
                agent_name_default=project_name,
                template_default=template,
                llm_default=llm or "ollama",
                vectorstore_default=vectorstore or "chromadb",
                evals_defaults=evals_list if evals_list else None,
                mcp_defaults=mcp_list if mcp_list else None,
            )

        # Use agent_name from wizard result as project name
        final_project_name = wizard_result.agent_name

        # Check if project directory already exists (unless force)
        project_dir = output_dir / final_project_name
        if project_dir.exists() and not force:
            # Prompt user for confirmation
            if click.confirm(
                f"Project directory '{final_project_name}' already exists. "
                "Do you want to overwrite it?",
                default=False,
            ):
                force = True
            else:
                click.echo("Initialization cancelled.")
                return

        # Create project initialization input
        init_input = ProjectInitInput(
            project_name=final_project_name,
            template=wizard_result.template,
            description=description,
            author=author,
            output_dir=str(output_dir),
            overwrite=force,
            agent_name=wizard_result.agent_name,
            llm_provider=wizard_result.llm_provider,
            provider_config=wizard_result.provider_config,
            vector_store=wizard_result.vector_store,
            evals=wizard_result.evals,
            mcp_servers=wizard_result.mcp_servers,
        )

        # Initialize project
        initializer = ProjectInitializer()
        result = initializer.initialize(init_input)

        # Handle result
        if result.success:
            # Display success message
            click.echo()  # Blank line for readability
            click.secho("Project initialized successfully!", fg="green", bold=True)
            click.echo()
            click.echo(f"Project: {result.project_name}")
            click.echo(f"Location: {result.project_path}")
            click.echo(f"Template: {result.template_used}")
            click.echo()
            click.echo("Configuration:")
            click.echo(f"  Agent Name: {wizard_result.agent_name}")
            click.echo(f"  Template: {wizard_result.template}")
            click.echo(f"  LLM Provider: {wizard_result.llm_provider}")
            click.echo(f"  Vector Store: {wizard_result.vector_store}")
            click.echo(f"  Evals: {', '.join(wizard_result.evals) or 'none'}")
            click.echo(
                f"  MCP Servers: {', '.join(wizard_result.mcp_servers) or 'none'}"
            )
            click.echo()
            click.echo(f"Time: {result.duration_seconds:.2f}s")

            # Show created files (first 10, then summary)
            if result.files_created:
                click.echo()
                click.echo("Files created:")
                # Show key files first (config, instructions, tools, data)
                key_files = [
                    f
                    for f in result.files_created
                    if "agent.yaml" in f
                    or "system-prompt" in f
                    or "tools" in f
                    or "data" in f
                ]
                for file_path in key_files[:5]:
                    click.echo(f"  - {file_path}")
                if len(result.files_created) > 5:
                    remaining = len(result.files_created) - 5
                    click.echo(f"  ... and {remaining} more file(s)")

            click.echo()
            click.echo("Next steps:")
            click.echo(f"  1. cd {result.project_name}")
            click.echo("  2. Edit agent.yaml to configure your agent")
            click.echo("  3. Edit instructions/system-prompt.md to customize behavior")
            click.echo("  4. Add tools in tools/ directory")
            click.echo("  5. Update test_cases in agent.yaml")
            click.echo("  6. Run tests with: holodeck test agent.yaml")
            click.echo()
        else:
            # Display error message
            click.secho("Project initialization failed", fg="red", bold=True)
            click.echo()
            for error in result.errors:
                click.secho(f"Error: {error}", fg="red")
            click.echo()
            raise click.Abort()

    except WizardCancelledError as e:
        # Handle wizard cancellation gracefully
        click.echo()
        click.secho("Wizard cancelled.", fg="yellow")
        raise click.Abort() from e

    except KeyboardInterrupt as e:
        # Handle Ctrl+C gracefully with cleanup
        click.echo()
        click.secho("Initialization cancelled by user.", fg="yellow")
        raise click.Abort() from e

    except (ValidationError, InitError) as e:
        # Handle known errors
        click.secho(f"Error: {str(e)}", fg="red")
        raise click.Abort() from e

    except Exception as e:
        # Handle unexpected errors
        click.secho(f"Unexpected error: {str(e)}", fg="red")
        raise click.Abort() from e
```

## `validate_template(ctx, param, value)`

Validate template parameter and provide helpful error messages.

Parameters:

| Name    | Type        | Description                    | Default    |
| ------- | ----------- | ------------------------------ | ---------- |
| `ctx`   | `Context`   | Click context                  | *required* |
| `param` | `Parameter` | Click parameter                | *required* |
| `value` | `str`       | Template name provided by user | *required* |

Returns:

| Type  | Description                 |
| ----- | --------------------------- |
| `str` | The validated template name |

Raises:

| Type           | Description            |
| -------------- | ---------------------- |
| `BadParameter` | If template is invalid |

Source code in `src/holodeck/cli/commands/init.py`

```
def validate_template(
    ctx: click.Context,  # noqa: ARG001
    param: click.Parameter,  # noqa: ARG001
    value: str,
) -> str:
    """Validate template parameter and provide helpful error messages.

    Args:
        ctx: Click context
        param: Click parameter
        value: Template name provided by user

    Returns:
        The validated template name

    Raises:
        click.BadParameter: If template is invalid
    """
    available = TemplateRenderer.list_available_templates()
    if value not in available:
        raise click.BadParameter(
            f"Unknown template '{value}'. Available templates: {', '.join(available)}"
        )
    return value
```

## `_parse_comma_arg(value)`

Parse a comma-separated argument into a list.

Parameters:

| Name    | Type  | Description | Default                         |
| ------- | ----- | ----------- | ------------------------------- |
| `value` | \`str | None\`      | Comma-separated string or None. |

Returns:

| Type        | Description                         |
| ----------- | ----------------------------------- |
| `list[str]` | List of stripped, non-empty values. |

Source code in `src/holodeck/cli/commands/init.py`

```
def _parse_comma_arg(value: str | None) -> list[str]:
    """Parse a comma-separated argument into a list.

    Args:
        value: Comma-separated string or None.

    Returns:
        List of stripped, non-empty values.
    """
    if not value:
        return []
    return [v.strip() for v in value.split(",") if v.strip()]
```

______________________________________________________________________

### Test Command

Run tests for a HoloDeck agent with evaluation metrics and report generation.

## `test()`

Execute agent test cases with evaluation metrics.

Default subcommand is `run` — so `holodeck test agent.yaml` stays valid shorthand for `holodeck test run agent.yaml`. Subcommands:

run Execute test cases (default). view Launch the Dash evaluation dashboard.

Source code in `src/holodeck/cli/commands/test.py`

```
@click.group(cls=_TestGroup)
def test() -> None:
    """Execute agent test cases with evaluation metrics.

    Default subcommand is `run` — so `holodeck test agent.yaml` stays valid
    shorthand for `holodeck test run agent.yaml`. Subcommands:

      run    Execute test cases (default).
      view   Launch the Dash evaluation dashboard.
    """
```

## `SpinnerThread(progress)`

Bases: `Thread`

Background thread for spinner animation.

Initialize spinner thread.

Parameters:

| Name       | Type                | Description                | Default    |
| ---------- | ------------------- | -------------------------- | ---------- |
| `progress` | `ProgressIndicator` | ProgressIndicator instance | *required* |

Source code in `src/holodeck/cli/commands/test.py`

```
def __init__(self, progress: ProgressIndicator) -> None:
    """Initialize spinner thread.

    Args:
        progress: ProgressIndicator instance
    """
    super().__init__(daemon=True)
    self.progress = progress
    self._stop_event = threading.Event()
    self._running = False
```

### `run()`

Run spinner animation loop.

Source code in `src/holodeck/cli/commands/test.py`

```
def run(self) -> None:
    """Run spinner animation loop."""
    self._running = True
    while not self._stop_event.is_set():
        line = self.progress.get_spinner_line()
        if line:
            # Use \r to overwrite line, flush to ensure display
            sys.stdout.write(f"\r{line}")
            sys.stdout.flush()
        time.sleep(0.1)
    self._running = False
```

### `stop()`

Stop spinner animation.

Source code in `src/holodeck/cli/commands/test.py`

```
def stop(self) -> None:
    """Stop spinner animation."""
    self._stop_event.set()
    if self._running:
        # Clear spinner line
        sys.stdout.write("\r" + " " * 60 + "\r")
        sys.stdout.flush()
```

## `_save_report(report, output, format)`

Save test report to file in specified format.

Parameters:

| Name     | Type         | Description                 | Default                                                             |
| -------- | ------------ | --------------------------- | ------------------------------------------------------------------- |
| `report` | `TestReport` | TestReport instance to save | *required*                                                          |
| `output` | `str`        | Output file path            | *required*                                                          |
| `format` | \`str        | None\`                      | Report format (json/markdown). If None, auto-detect from extension. |

Source code in `src/holodeck/cli/commands/test.py`

```
def _save_report(report: TestReport, output: str, format: str | None) -> None:
    """Save test report to file in specified format.

    Args:
        report: TestReport instance to save
        output: Output file path
        format: Report format (json/markdown). If None, auto-detect from extension.
    """
    output_path = Path(output)

    # Determine format if not specified
    if format is None:
        if output.endswith(".json"):
            format = "json"
        elif output.endswith(".md") or output.endswith(".markdown"):
            format = "markdown"
        else:
            format = "json"  # Default to JSON
        logger.debug(f"Auto-detected report format: {format}")

    # Generate report content
    logger.debug(f"Generating {format} report")
    if format == "json":
        # Use pydantic's model_dump_json method
        content = report.model_dump_json(indent=2)
    else:  # markdown
        # Generate markdown format using reporter module
        content = generate_markdown_report(report)

    # Write to file (overwrites if exists)
    try:
        output_path.parent.mkdir(parents=True, exist_ok=True)
        output_path.write_text(content, encoding="utf-8")
        click.echo(f"Report saved to {output}")
    except OSError as e:
        logger.error(f"Failed to write report to {output}: {e}")
        raise
```

______________________________________________________________________

### Chat Command

Start an interactive multi-turn chat session with an agent.

## `chat(agent_config, verbose, quiet, observability, max_messages, force_ingest)`

Start an interactive chat session with an agent.

AGENT_CONFIG is the path to the agent.yaml configuration file.

Example:

```
holodeck chat examples/weather-agent.yaml

holodeck chat examples/assistant.yaml --verbose --max-messages 100
```

Chat Session Commands:

```
Type 'exit' or 'quit' to end the session.
Press Ctrl+C to interrupt.
```

Options:

```
--verbose / -v      Show detailed tool execution parameters and results
--quiet / -q        Suppress logging output (enabled by default)
--observability / -o    Enable OpenTelemetry tracing for debugging
--max-messages / -m     Set max messages before context warning (default: 50)
```

Source code in `src/holodeck/cli/commands/chat.py`

```
@click.command()
@click.argument("agent_config", type=click.Path(exists=True), default="agent.yaml")
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Show detailed logging and tool execution (parameters, internal state)",
)
@click.option(
    "--quiet/--no-quiet",
    "-q/-Q",
    default=False,
    help="Suppress INFO logging output. Use -q or --quiet to hide logs.",
)
@click.option(
    "--observability",
    "-o",
    is_flag=True,
    help="Enable OpenTelemetry tracing and metrics",
)
@click.option(
    "--max-messages",
    "-m",
    type=int,
    default=50,
    help="Maximum conversation messages before warning",
)
@click.option(
    "--force-ingest",
    "-f",
    is_flag=True,
    help="Force re-ingestion of all vector store source files",
)
def chat(
    agent_config: str,
    verbose: bool,
    quiet: bool,
    observability: bool,
    max_messages: int,
    force_ingest: bool,
) -> None:
    """Start an interactive chat session with an agent.

    AGENT_CONFIG is the path to the agent.yaml configuration file.

    Example:

        holodeck chat examples/weather-agent.yaml

        holodeck chat examples/assistant.yaml --verbose --max-messages 100

    Chat Session Commands:

        Type 'exit' or 'quit' to end the session.
        Press Ctrl+C to interrupt.

    Options:

        --verbose / -v      Show detailed tool execution parameters and results
        --quiet / -q        Suppress logging output (enabled by default)
        --observability / -o    Enable OpenTelemetry tracing for debugging
        --max-messages / -m     Set max messages before context warning (default: 50)
    """
    # Initialize observability context (will be set if observability enabled)
    obs_context: ObservabilityContext | None = None
    effective_quiet = quiet and not verbose

    try:
        # Load agent config and resolve execution config in one call
        from holodeck.config.loader import load_agent_with_config

        cli_config = ExecutionConfig(
            verbose=verbose if verbose else None,
            quiet=quiet if quiet else None,
        )

        agent, resolved_config, _loader = load_agent_with_config(
            agent_config, cli_config=cli_config
        )

        # Determine logging strategy: OTel replaces setup_logging when enabled
        if agent.observability and agent.observability.enabled:
            obs_context = initialize_observability(
                agent.observability, agent.name, verbose=verbose, quiet=quiet
            )
        else:
            setup_logging(verbose=verbose, quiet=effective_quiet)

        logger.info(
            f"Chat command invoked: config={agent_config}, "
            f"verbose={verbose}, quiet={quiet}, observability={observability}, "
            f"max_messages={max_messages}, force_ingest={force_ingest}"
        )
        logger.debug(f"Loading agent configuration from {agent_config}")
        logger.info(f"Agent configuration loaded successfully: {agent.name}")

        logger.debug(
            f"Resolved execution config: verbose={resolved_config.verbose}, "
            f"quiet={resolved_config.quiet}, llm_timeout={resolved_config.llm_timeout}"
        )

        # Run async chat session
        logger.debug("Starting chat session runtime")
        asyncio.run(
            _run_chat_session(
                agent=agent,
                agent_config_path=Path(agent_config),
                verbose=resolved_config.verbose or False,
                quiet=resolved_config.quiet or False,
                enable_observability=observability,
                max_messages=max_messages,
                force_ingest=force_ingest,
                llm_timeout=resolved_config.llm_timeout,
                observability_enabled=obs_context is not None,
            )
        )

        # Normal exit (user typed exit/quit)
        logger.info("Chat session ended normally")
        sys.exit(0)

    except ConfigError as e:
        logger.error(f"Configuration error: {e}", exc_info=True)
        click.secho("Error: Failed to load agent configuration", fg="red", err=True)
        click.echo(f"  {str(e)}", err=True)
        sys.exit(1)
    except AgentInitializationError as e:
        logger.error(f"Agent initialization error: {e}", exc_info=True)
        click.secho("Error: Failed to initialize agent", fg="red", err=True)
        click.echo(f"  {str(e)}", err=True)
        sys.exit(2)
    except KeyboardInterrupt:
        logger.info("Chat interrupted by user (Ctrl+C)")
        click.echo()
        click.secho("Goodbye!", fg="yellow")
        sys.exit(130)
    except ExecutionError as e:
        logger.error(f"Execution error: {e}", exc_info=True)
        click.secho(f"Error: {str(e)}", fg="red", err=True)
        sys.exit(1)
    except Exception as e:
        logger.error(f"Unexpected error: {e}", exc_info=True)
        click.secho(f"Error: {str(e)}", fg="red", err=True)
        sys.exit(1)
    finally:
        # Shutdown observability if it was initialized
        if obs_context:
            shutdown_observability(obs_context)
```

## `_run_chat_session(agent, agent_config_path, verbose, quiet, enable_observability, max_messages, force_ingest=False, llm_timeout=None, observability_enabled=False)`

Run the interactive chat session.

Parameters:

| Name                    | Type    | Description                                     | Default                         |
| ----------------------- | ------- | ----------------------------------------------- | ------------------------------- |
| `agent`                 | `Agent` | Loaded Agent configuration                      | *required*                      |
| `agent_config_path`     | `Path`  | Path to agent.yaml file                         | *required*                      |
| `verbose`               | `bool`  | Enable detailed tool execution display          | *required*                      |
| `quiet`                 | `bool`  | Suppress logging output                         | *required*                      |
| `enable_observability`  | `bool`  | Enable OpenTelemetry tracing                    | *required*                      |
| `max_messages`          | `int`   | Maximum messages before warning                 | *required*                      |
| `force_ingest`          | `bool`  | Force re-ingestion of vector store source files | `False`                         |
| `llm_timeout`           | \`int   | None\`                                          | LLM API call timeout in seconds |
| `observability_enabled` | `bool`  | Whether OTel tracing is enabled                 | `False`                         |

Raises:

| Type                | Description                   |
| ------------------- | ----------------------------- |
| `KeyboardInterrupt` | When user interrupts (Ctrl+C) |

Source code in `src/holodeck/cli/commands/chat.py`

```
async def _run_chat_session(
    agent: Agent,
    agent_config_path: Path,
    verbose: bool,
    quiet: bool,
    enable_observability: bool,
    max_messages: int,
    force_ingest: bool = False,
    llm_timeout: int | None = None,
    observability_enabled: bool = False,
) -> None:
    """Run the interactive chat session.

    Args:
        agent: Loaded Agent configuration
        agent_config_path: Path to agent.yaml file
        verbose: Enable detailed tool execution display
        quiet: Suppress logging output
        enable_observability: Enable OpenTelemetry tracing
        max_messages: Maximum messages before warning
        force_ingest: Force re-ingestion of vector store source files
        llm_timeout: LLM API call timeout in seconds
        observability_enabled: Whether OTel tracing is enabled

    Raises:
        KeyboardInterrupt: When user interrupts (Ctrl+C)
    """
    # Create parent span for chat command if observability is enabled
    if observability_enabled:
        from holodeck.lib.observability import get_tracer

        tracer = get_tracer(__name__)
        span_context: Any = tracer.start_as_current_span("holodeck.cli.chat")
    else:
        span_context = nullcontext()

    with span_context:
        # Initialize session manager
        try:
            chat_config = ChatConfig(
                agent_config_path=Path(agent_config_path),
                verbose=verbose,
                enable_observability=enable_observability,
                max_messages=max_messages,
                force_ingest=force_ingest,
                llm_timeout=llm_timeout,
            )
            session_manager = ChatSessionManager(
                agent_config=agent,
                config=chat_config,
            )
        except Exception as e:
            logger.error(f"Failed to initialize session: {e}", exc_info=True)
            raise AgentInitializationError(agent.name, str(e)) from e

        # Start session
        try:
            logger.debug("Starting chat session")
            await session_manager.start()
        except Exception as e:
            logger.error(f"Failed to start session: {e}", exc_info=True)
            raise AgentInitializationError(agent.name, str(e)) from e

        try:
            # Display welcome message
            click.secho(f"\nStarting chat with {agent.name}...", fg="green", bold=True)
            click.echo("Type 'exit' or 'quit' to end session.")
            click.echo()

            # Initialize progress indicator
            progress = ChatProgressIndicator(
                max_messages=max_messages,
                quiet=quiet,
                verbose=verbose,
            )

            # REPL loop
            while True:
                try:
                    # Get user input
                    user_input = click.prompt("You", default="").strip()

                    # Check for exit commands
                    if user_input.lower() in ("exit", "quit"):
                        click.secho("Goodbye!", fg="yellow")
                        break

                    # Skip empty messages (validation handled in session)
                    if not user_input:
                        continue

                    try:
                        logger.debug(f"Processing user message: {user_input[:50]}...")

                        # Eager-init so the backend's tool_event_queue is
                        # live before we spawn the drainer (mirrors the
                        # AGUI protocol path in holodeck serve).
                        executor = session_manager._executor
                        if executor is not None:
                            await _ensure_executor_ready(executor)
                        queue = (
                            executor.tool_event_queue if executor is not None else None
                        )

                        panel = ToolsPanel()
                        composer = LiveComposer()
                        stop_event = asyncio.Event()
                        drain_task: asyncio.Task[None] | None = None
                        if queue is not None:
                            drain_task = asyncio.create_task(
                                _drain_tool_events(queue, panel, stop_event)
                            )

                        await composer.begin()
                        click.echo("Agent: ", nl=False)
                        sys.stdout.flush()
                        paint_task: asyncio.Task[None] = asyncio.create_task(
                            _paint_panel_loop(panel, progress, composer, stop_event)
                        )

                        start_time = time.time()
                        chunks: list[str] = []
                        try:
                            async for (
                                chunk
                            ) in session_manager.process_message_streaming(user_input):
                                await composer.write_text(chunk)
                                chunks.append(chunk)
                        finally:
                            stop_event.set()
                            await paint_task
                            if drain_task is not None:
                                try:
                                    await asyncio.wait_for(drain_task, timeout=0.25)
                                except asyncio.TimeoutError:
                                    drain_task.cancel()
                                    with contextlib.suppress(asyncio.CancelledError):
                                        await drain_task
                            if queue is not None:
                                _drain_remaining(queue, panel)
                            await composer.end()

                        elapsed = time.time() - start_time

                        click.echo()  # newline after streamed content

                        # Build minimal AgentResponse for progress tracking.
                        # Token usage and tool details unavailable via streaming.
                        response = AgentResponse(
                            content="".join(chunks),
                            tool_executions=[],
                            tokens_used=None,
                            execution_time=elapsed,
                        )

                        # Update progress
                        progress.update(response)
                        progress.set_active_snapshot(panel.snapshot())

                        # Display status
                        if verbose:
                            click.echo(progress.get_status_panel())
                        else:
                            status = progress.get_status_inline()
                            click.echo(f"{status}\n")

                        logger.debug(f"Streamed response in {elapsed:.2f}s")

                        # Check for context limit warning
                        if session_manager.should_warn_context_limit():
                            click.secho(
                                "⚠️  Approaching context limit. Consider a new session.",
                                fg="yellow",
                            )
                            click.echo()

                    except Exception as e:
                        # Display error but continue session (don't crash)
                        logger.warning(f"Error processing message: {e}")
                        click.secho(f"Error: {str(e)}", fg="red")
                        click.echo()

                except EOFError:
                    # Handle Ctrl+D
                    click.echo()
                    click.secho("Goodbye!", fg="yellow")
                    break

        except KeyboardInterrupt:
            # Handle Ctrl+C
            click.echo()
            click.secho("Goodbye!", fg="yellow")
            raise
        finally:
            # Cleanup
            try:
                logger.debug("Terminating chat session")
                await session_manager.terminate()
            except Exception as e:
                logger.warning(f"Error during session cleanup: {e}")
```

______________________________________________________________________

### Config Command

Manage HoloDeck global and project configuration files.

## `config()`

Manage HoloDeck configuration.

Source code in `src/holodeck/cli/commands/config.py`

```
@click.group(name="config")
def config() -> None:
    """Manage HoloDeck configuration."""
    pass
```

## `init(global_config, project_config, force, verbose, quiet)`

Initialize HoloDeck global or project configuration.

Creates a new configuration file with default settings. By default, this command will prompt you to choose between global (~/.holodeck/config.yaml) or project (config.yaml) configuration initialization.

EXAMPLES:

```
Initialize global configuration:
    holodeck config init -g

Initialize project configuration:
    holodeck config init -p

Overwrite existing configuration:
    holodeck config init -g --force
```

For more information, see: https://useholodeck.ai/docs/config

Source code in `src/holodeck/cli/commands/config.py`

```
@config.command(name="init")
@click.option(
    "-g",
    "--global",
    "global_config",
    is_flag=True,
    help="Initialize global configuration in ~/.holodeck/config.yaml",
)
@click.option(
    "-p",
    "--project",
    "project_config",
    is_flag=True,
    help="Initialize project configuration in config.yaml",
)
@click.option(
    "--force",
    is_flag=True,
    help="Overwrite existing configuration file without prompting",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress INFO logging output",
)
def init(
    global_config: bool,
    project_config: bool,
    force: bool,
    verbose: bool,
    quiet: bool,
) -> None:
    """Initialize HoloDeck global or project configuration.

    Creates a new configuration file with default settings. By default, this command
    will prompt you to choose between global (~/.holodeck/config.yaml) or project
    (config.yaml) configuration initialization.

    EXAMPLES:

        Initialize global configuration:
            holodeck config init -g

        Initialize project configuration:
            holodeck config init -p

        Overwrite existing configuration:
            holodeck config init -g --force

    For more information, see: https://useholodeck.ai/docs/config
    """
    # Initialize logging
    setup_logging(verbose=verbose, quiet=quiet)
    logger.debug(
        f"Config init command invoked: global={global_config}, project={project_config}"
    )

    # Determine which config to initialize
    if not global_config and not project_config:
        # Prompt user to choose
        choice = click.prompt(
            "Initialize global (~/.holodeck/config.yaml) or "
            "project (config.yaml) configuration?",
            type=click.Choice(["g", "p"]),
            default="g",
        )
        if choice == "g":
            global_config = True
        else:
            project_config = True

    # Determine config file path
    config_path, config_type = ConfigManager.get_config_path(
        global_config, project_config
    )

    # Check if config file already exists
    # Check if config file already exists
    if (
        config_path.exists()
        and not force
        and not click.confirm(
            f"Configuration file '{config_path}' already exists. "
            "Do you want to overwrite it?",
            default=False,
        )
    ):
        click.echo("Initialization cancelled.")
        return

    try:
        # Create default configuration
        default_config = ConfigManager.create_default_config()

        # Generate YAML content
        yaml_content = ConfigManager.generate_config_content(default_config)

        # Write to file
        ConfigManager.write_config(config_path, yaml_content)

        click.secho(
            f"✓ {config_type.capitalize()} configuration initialized successfully!",
            fg="green",
            bold=True,
        )
        click.echo(f"Configuration saved to: {config_path}")
        click.echo()
        click.echo("Next steps:")
        click.echo("  1. Edit the configuration file to customize settings")
        click.echo("  2. Use the configuration in your agent projects")
        click.echo()

    except Exception as e:
        click.secho(f"✗ Failed to initialize configuration: {str(e)}", fg="red")
        raise click.Abort() from e
```

______________________________________________________________________

### Deploy Command

Build container images and deploy agents to cloud providers.

## `deploy(ctx)`

Deploy HoloDeck agents to container registries and cloud providers.

Subcommands:

```
build   Build a container image for the agent
run     Deploy a container image to the cloud
status  Check deployment status
destroy Destroy a deployment
```

Example:

```
holodeck deploy build

holodeck deploy build agent.yaml --dry-run
```

Source code in `src/holodeck/cli/commands/deploy.py`

```
@click.group(name="deploy", invoke_without_command=True)
@click.pass_context
def deploy(ctx: click.Context) -> None:
    """Deploy HoloDeck agents to container registries and cloud providers.

    Subcommands:

        build   Build a container image for the agent
        run     Deploy a container image to the cloud
        status  Check deployment status
        destroy Destroy a deployment

    Example:

        holodeck deploy build

        holodeck deploy build agent.yaml --dry-run
    """
    # Ensure context object exists
    ctx.ensure_object(dict)

    # If no subcommand is provided, show help
    if ctx.invoked_subcommand is None:
        click.echo(ctx.get_help())
```

## `build(agent_config, tag, no_cache, dry_run, verbose, quiet)`

Build a container image for the agent.

AGENT_CONFIG is the path to the agent.yaml configuration file.

Generates a Dockerfile from the agent configuration and builds a container image using Docker.

Example:

```
holodeck deploy build

holodeck deploy build agent.yaml --tag v1.0.0

holodeck deploy build --dry-run
```

Source code in `src/holodeck/cli/commands/deploy.py`

```
@deploy.command()
@click.argument(
    "agent_config",
    type=click.Path(exists=True),
    default="agent.yaml",
    required=False,
)
@click.option(
    "--tag",
    type=str,
    default=None,
    help="Custom tag for the image (overrides tag_strategy)",
)
@click.option(
    "--no-cache",
    is_flag=True,
    help="Build without using cache",
)
@click.option(
    "--dry-run",
    is_flag=True,
    help="Show what would be done without executing",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress progress output",
)
def build(
    agent_config: str,
    tag: str | None,
    no_cache: bool,
    dry_run: bool,
    verbose: bool,
    quiet: bool,
) -> None:
    """Build a container image for the agent.

    AGENT_CONFIG is the path to the agent.yaml configuration file.

    Generates a Dockerfile from the agent configuration and builds
    a container image using Docker.

    Example:

        holodeck deploy build

        holodeck deploy build agent.yaml --tag v1.0.0

        holodeck deploy build --dry-run
    """
    # Setup logging based on verbosity
    if not quiet:
        setup_logging(verbose=verbose, quiet=quiet)

    try:
        # Load agent configuration using ConfigLoader (same as chat, test, serve)
        from holodeck.config.loader import ConfigLoader

        agent_path = Path(agent_config).resolve()
        agent_dir = agent_path.parent

        if not quiet:
            click.echo(f"Loading agent configuration from {agent_config}...")

        # Load and validate agent using standard ConfigLoader. The build is
        # structural (it just needs the YAML shape — tools, deployment block,
        # file paths); secrets aren't required, so skip env-var substitution
        # to avoid forcing the operator to export every runtime var.
        loader = ConfigLoader()
        agent = loader.load_agent_yaml(agent_config, substitute_env=False)
        agent_name = agent.name

        # Get deployment configuration from agent (merged by ConfigLoader)
        if not agent.deployment:
            raise ConfigError(
                field="deployment",
                message="No 'deployment' section found in agent configuration",
            )
        deployment_config = agent.deployment

        # Determine tag
        if tag:
            # Custom tag from CLI overrides config
            image_tag = tag
        else:
            # Use tag strategy from config
            from holodeck.deploy.builder import generate_tag

            image_tag = generate_tag(
                deployment_config.registry.tag_strategy,
                deployment_config.registry.custom_tag,
            )

        # Full image name
        registry_url = deployment_config.registry.url
        repository = deployment_config.registry.repository
        image_name = f"{registry_url}/{repository}"
        full_image_name = f"{image_name}:{image_tag}"

        if not quiet:
            click.echo()
            click.secho("Build Configuration:", bold=True)
            click.echo(f"  Agent:     {agent_name}")
            click.echo(f"  Image:     {full_image_name}")
            click.echo(f"  Platform:  {deployment_config.platform}")
            click.echo(f"  Protocol:  {deployment_config.protocol.value}")
            click.echo(f"  Port:      {deployment_config.port}")
            click.echo()

        if dry_run:
            click.secho("[DRY RUN] Would build image:", fg="yellow")
            click.echo(f"  Image:    {full_image_name}")
            click.echo(f"  Platform: {deployment_config.platform}")

            # Show generated Dockerfile
            dockerfile_content = _generate_dockerfile_content(
                agent,
                deployment_config,
                image_tag,
                test_cases_file=_read_raw_test_cases_file(agent_path),
                base_dir=agent_dir,
            )
            click.echo()
            click.secho("Generated Dockerfile:", bold=True)
            for line in dockerfile_content.split("\n"):
                click.echo(f"  {line}")

            click.echo()
            click.secho("[DRY RUN] No image was built", fg="yellow")
            sys.exit(0)

        # Create build context
        if not quiet:
            click.echo("Preparing build context...")

        build_dir = _prepare_build_context(
            agent, deployment_config, agent_dir, image_tag
        )

        try:
            # Initialize builder
            if not quiet:
                click.echo("Connecting to Docker...")

            from holodeck.deploy.builder import ContainerBuilder, get_oci_labels

            builder = ContainerBuilder()

            # Generate OCI labels
            labels = get_oci_labels(
                agent_name=agent_name,
                version=image_tag,
            )

            # Build image
            if not quiet:
                click.echo(f"Building image {full_image_name}...")
                click.echo()

            build_kwargs: dict[str, Any] = {}
            if no_cache:
                build_kwargs["nocache"] = True

            result = builder.build(
                build_context=str(build_dir),
                image_name=image_name,
                tag=image_tag,
                labels=labels,
                platform=deployment_config.platform,
                **build_kwargs,
            )

            # Display build logs if verbose
            if verbose and result.log_lines:
                click.secho("Build Output:", bold=True)
                for line in result.log_lines:
                    if line.strip():
                        click.echo(f"  {line}")
                click.echo()

            # Success message
            _display_build_success(result, quiet)

        finally:
            # Cleanup build context
            shutil.rmtree(build_dir, ignore_errors=True)

    except DockerNotAvailableError as e:
        logger.error(f"Docker not available: {e}")
        click.secho("Error: Docker is not available", fg="red", err=True)
        click.echo(str(e), err=True)
        sys.exit(3)

    except ConfigError as e:
        logger.error(f"Configuration error: {e}")
        click.secho("Error: Configuration error", fg="red", err=True)
        click.echo(f"  {e.message}", err=True)
        sys.exit(2)

    except DeploymentError as e:
        logger.error(f"Deployment error: {e}")
        click.secho(f"Error: {e.operation} failed", fg="red", err=True)
        click.echo(f"  {e.message}", err=True)
        sys.exit(3)

    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
        click.secho(f"Error: {e}", fg="red", err=True)
        sys.exit(3)
```

## `run(agent_config, dry_run, verbose, quiet)`

Deploy an agent image to the configured cloud provider.

Source code in `src/holodeck/cli/commands/deploy.py`

```
@deploy.command()
@click.argument(
    "agent_config",
    type=click.Path(exists=True),
    default="agent.yaml",
    required=False,
)
@click.option(
    "--dry-run",
    is_flag=True,
    help="Show what would be done without executing",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress progress output",
)
def run(
    agent_config: str,
    dry_run: bool,
    verbose: bool,
    quiet: bool,
) -> None:
    """Deploy an agent image to the configured cloud provider."""
    if not quiet:
        setup_logging(verbose=verbose, quiet=quiet)

    with handle_deployment_errors():
        agent, deployment_config, agent_path = _load_agent_and_deployment(agent_config)
        _ensure_azure_provider(deployment_config, operation="deploy")

        image_uri, image_tag = _resolve_image_uri(deployment_config)

        if not quiet:
            click.echo()
            click.secho("Deploy Configuration:", bold=True)
            click.echo(f"  Agent:     {agent.name}")
            click.echo(f"  Image:     {image_uri}")
            click.echo(f"  Tag:       {image_tag}")
            click.echo(f"  Platform:  {deployment_config.platform}")
            click.echo(f"  Provider:  {deployment_config.target.provider.value}")
            click.echo(f"  Port:      {deployment_config.port}")
            click.echo()

        if dry_run:
            click.secho("[DRY RUN] Would deploy image:", fg="yellow")
            click.echo(f"  Image:    {image_uri}")
            click.echo(f"  Platform: {deployment_config.platform}")
            click.secho("[DRY RUN] No deployment was created", fg="yellow")
            sys.exit(0)

        deployer = create_deployer(deployment_config.target)
        result = deployer.deploy(
            service_name=agent.name,
            image_uri=image_uri,
            port=deployment_config.port,
            env_vars=deployment_config.environment,
            health_check_path=deployment_config.health_check_path,
        )

        state_path = get_state_path(agent_path)
        record = DeploymentRecord(
            provider=deployment_config.target.provider,
            service_id=result.service_id,
            service_name=result.service_name,
            url=result.url,
            status=result.status,
            image_uri=image_uri,
            config_hash=compute_config_hash(deployment_config),
        )
        record = update_deployment_record(state_path, agent.name, record)

        if quiet:
            click.echo(result.url or "")
            sys.exit(0)

        click.echo()
        click.secho("Deployment Successful!", fg="green", bold=True)
        click.echo(f"  Service:   {record.service_name}")
        click.echo(f"  Status:    {record.status}")
        if record.url:
            click.echo(f"  URL:       {record.url}")
            click.echo(
                f"  Health:    {record.url}{deployment_config.health_check_path}"
            )
        else:
            click.echo("  URL:       (not available yet)")
        click.echo()
```

## `status(agent_config, verbose, quiet)`

Check deployment status for an agent.

Source code in `src/holodeck/cli/commands/deploy.py`

```
@deploy.command()
@click.argument(
    "agent_config",
    type=click.Path(exists=True),
    default="agent.yaml",
    required=False,
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress progress output",
)
def status(agent_config: str, verbose: bool, quiet: bool) -> None:
    """Check deployment status for an agent."""
    if not quiet:
        setup_logging(verbose=verbose, quiet=quiet)

    with handle_deployment_errors():
        agent, deployment_config, agent_path = _load_agent_and_deployment(agent_config)
        _ensure_azure_provider(deployment_config, operation="status")

        state_path = get_state_path(agent_path)
        record = get_deployment_record(state_path, agent.name)
        if record is None:
            raise ConfigError(
                field="deployment_state",
                message="No deployment record found. Run `holodeck deploy run` first.",
            )

        deployer = create_deployer(deployment_config.target)
        status_result = deployer.get_status(record.service_id)

        updated_record = record.model_copy(
            update={
                "status": status_result.status or record.status,
                "url": status_result.url or record.url,
            }
        )
        record = update_deployment_record(state_path, agent.name, updated_record)

        if quiet:
            click.echo(record.status or "UNKNOWN")
            sys.exit(0)

        click.echo()
        click.secho("Deployment Status", bold=True)
        click.echo(f"  Service:   {record.service_name}")
        click.echo(f"  Provider:  {deployment_config.target.provider.value}")
        click.echo(f"  Status:    {record.status}")
        if record.url:
            click.echo(f"  URL:       {record.url}")
        if record.updated_at:
            click.echo(f"  Updated:   {record.updated_at.isoformat()}")
        click.echo()
```

## `destroy(agent_config, force, verbose, quiet)`

Destroy a deployed agent.

Source code in `src/holodeck/cli/commands/deploy.py`

```
@deploy.command()
@click.argument(
    "agent_config",
    type=click.Path(exists=True),
    default="agent.yaml",
    required=False,
)
@click.option(
    "--force",
    is_flag=True,
    help="Skip confirmation prompt",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress progress output",
)
def destroy(agent_config: str, force: bool, verbose: bool, quiet: bool) -> None:
    """Destroy a deployed agent."""
    if not quiet:
        setup_logging(verbose=verbose, quiet=quiet)

    with handle_deployment_errors():
        agent, deployment_config, agent_path = _load_agent_and_deployment(agent_config)
        _ensure_azure_provider(deployment_config, operation="destroy")

        state_path = get_state_path(agent_path)
        record = get_deployment_record(state_path, agent.name)
        if record is None:
            raise ConfigError(
                field="deployment_state",
                message="No deployment record found. Run `holodeck deploy run` first.",
            )

        service_name = record.service_name or agent.name

        if not force:
            confirm = click.confirm(
                f"Destroy deployment '{service_name}'?", default=False
            )
            if not confirm:
                click.secho("Destroy aborted.", fg="yellow")
                sys.exit(0)

        deployer = create_deployer(deployment_config.target)
        deployer.destroy(record.service_id)

        updated_record = record.model_copy(update={"status": "DELETED"})
        record = update_deployment_record(state_path, agent.name, updated_record)

        if quiet:
            click.echo("deleted")
            sys.exit(0)

        click.echo()
        click.secho("Deployment Destroyed", fg="green", bold=True)
        click.echo(f"  Service:   {record.service_name}")
        click.echo(f"  Status:    {record.status}")
        click.echo()
```

## `handle_deployment_errors()`

Context manager for consistent error handling in deployment commands.

Catches and handles ConfigError, DeploymentError, and unexpected exceptions with appropriate logging, user feedback, and exit codes.

Exit codes

2: Configuration error 3: Deployment/execution error

Source code in `src/holodeck/cli/commands/deploy.py`

```
@contextmanager
def handle_deployment_errors() -> Generator[None, None, None]:
    """Context manager for consistent error handling in deployment commands.

    Catches and handles ConfigError, DeploymentError, and unexpected exceptions
    with appropriate logging, user feedback, and exit codes.

    Exit codes:
        2: Configuration error
        3: Deployment/execution error
    """
    try:
        yield
    except ConfigError as e:
        logger.error(f"Configuration error: {e}")
        click.secho("Error: Configuration error", fg="red", err=True)
        click.echo(f"  {e.message}", err=True)
        sys.exit(2)
    except DeploymentError as e:
        logger.error(f"Deployment error: {e}")
        click.secho(f"Error: {e.operation} failed", fg="red", err=True)
        click.echo(f"  {e.message}", err=True)
        sys.exit(3)
    except Exception as e:
        logger.exception(f"Unexpected error: {e}")
        click.secho(f"Error: {e}", fg="red", err=True)
        sys.exit(3)
```

## `_generate_dockerfile_content(agent, deployment_config, version, test_cases_file=None, base_dir=None)`

Generate Dockerfile content for the agent.

Parameters:

| Name                | Type               | Description                      | Default                                                                                                 |
| ------------------- | ------------------ | -------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `agent`             | `Agent`            | Loaded Agent configuration model | *required*                                                                                              |
| `deployment_config` | `DeploymentConfig` | Deployment configuration         | *required*                                                                                              |
| `version`           | `str`              | Version/tag for OCI labels       | *required*                                                                                              |
| `test_cases_file`   | \`str              | None\`                           | Optional raw test_cases_file path from agent YAML.                                                      |
| `base_dir`          | \`Path             | None\`                           | Agent directory; when provided, credential-shaped files in the COPY surface are flagged with a warning. |

Returns:

| Type  | Description                  |
| ----- | ---------------------------- |
| `str` | Generated Dockerfile content |

Source code in `src/holodeck/cli/commands/deploy.py`

```
def _generate_dockerfile_content(
    agent: Agent,
    deployment_config: DeploymentConfig,
    version: str,
    test_cases_file: str | None = None,
    base_dir: Path | None = None,
) -> str:
    """Generate Dockerfile content for the agent.

    Args:
        agent: Loaded Agent configuration model
        deployment_config: Deployment configuration
        version: Version/tag for OCI labels
        test_cases_file: Optional raw test_cases_file path from agent YAML.
        base_dir: Agent directory; when provided, credential-shaped files in
            the COPY surface are flagged with a warning.

    Returns:
        Generated Dockerfile content
    """
    from holodeck.deploy.dockerfile import generate_dockerfile
    from holodeck.models.llm import ProviderEnum
    from holodeck.models.tool import (
        FunctionTool,
        HierarchicalDocumentToolConfig,
        VectorstoreTool,
    )

    # Collect leaf files needing an identical `COPY src /app/src` directive:
    # instructions, function tool Python files, single-file tool sources, and
    # the test_cases_file (eagerly resolved by the loader at serve time).
    instruction_files: list[str] = []
    if agent.instructions.file:
        instruction_files.append(agent.instructions.file)
    for tool in agent.tools or []:
        if isinstance(tool, FunctionTool):
            instruction_files.append(tool.file)

    # Collect data directories (from vectorstore + hierarchical_document tools).
    # Skip remote sources (az://, s3://) — those are fetched at runtime.
    data_directories: list[str] = []
    data_files: list[str] = []
    if agent.tools:
        for tool in agent.tools:
            if isinstance(tool, VectorstoreTool | HierarchicalDocumentToolConfig):
                source_path = tool.source
                if not source_path or "://" in source_path:
                    continue
                path = Path(source_path)
                if path.suffix:
                    # Single file source — copy the file directly.
                    data_files.append(source_path)
                else:
                    data_directories.append(str(path) + "/")

    # test_cases_file is resolved eagerly by the loader at serve time, so it
    # has to be present in the image even though serve never uses it.
    if test_cases_file and "://" not in test_cases_file:
        data_files.append(test_cases_file)

    # Treat data_files as instruction_files (same `COPY src /app/src` shape).
    instruction_files.extend(data_files)
    instruction_files = sorted(set(instruction_files))

    # Remove duplicates
    data_directories = list(set(data_directories))

    # Warn if credential-shaped files will be baked into the image.
    if base_dir is not None:
        _warn_if_credential_files_in_copy_surface(
            instruction_files=instruction_files,
            data_directories=data_directories,
            base_dir=base_dir,
        )

    # Node.js is only needed if an MCP tool spawns a Node interpreter.
    needs_nodejs = _agent_needs_nodejs(agent)

    # Detect required extras from agent config. Vector-store provider names
    # match the optional-dependency group names in pyproject.toml 1:1.
    vectorstore_extras = {"chromadb", "qdrant", "pinecone", "postgres"}
    extras: list[str] = []
    for tool in agent.tools or []:
        if isinstance(tool, VectorstoreTool | HierarchicalDocumentToolConfig):
            provider = (
                getattr(tool.database, "provider", None) if tool.database else None
            )
            if provider in vectorstore_extras:
                extras.append(provider)
            if tool.source:
                if tool.source.startswith("az://"):
                    extras.append("azure-blob")
                elif tool.source.startswith("s3://"):
                    extras.append("s3")
    if agent.model.provider == ProviderEnum.ANTHROPIC:
        extras.append("claude-otel")
    extras = sorted(set(extras))

    return generate_dockerfile(
        agent_name=agent.name,
        port=deployment_config.port,
        protocol=deployment_config.protocol.value,
        version=version,
        instruction_files=instruction_files if instruction_files else None,
        data_directories=data_directories if data_directories else None,
        environment=deployment_config.environment or None,
        needs_nodejs=needs_nodejs,
        extras=extras if extras else None,
    )
```

## `_prepare_build_context(agent, deployment_config, agent_dir, version)`

Prepare build context directory with all required files.

Parameters:

| Name                | Type               | Description                      | Default    |
| ------------------- | ------------------ | -------------------------------- | ---------- |
| `agent`             | `Agent`            | Loaded Agent configuration model | *required* |
| `deployment_config` | `DeploymentConfig` | Deployment configuration         | *required* |
| `agent_dir`         | `Path`             | Directory containing agent.yaml  | *required* |
| `version`           | `str`              | Version for Dockerfile labels    | *required* |

Returns:

| Type   | Description                               |
| ------ | ----------------------------------------- |
| `Path` | Path to temporary build context directory |

Source code in `src/holodeck/cli/commands/deploy.py`

```
def _prepare_build_context(
    agent: Agent,
    deployment_config: DeploymentConfig,
    agent_dir: Path,
    version: str,
) -> Path:
    """Prepare build context directory with all required files.

    Args:
        agent: Loaded Agent configuration model
        deployment_config: Deployment configuration
        agent_dir: Directory containing agent.yaml
        version: Version for Dockerfile labels

    Returns:
        Path to temporary build context directory
    """
    from holodeck.models.tool import (
        FunctionTool,
        HierarchicalDocumentToolConfig,
        VectorstoreTool,
    )

    # Create temporary build directory
    build_dir = Path(tempfile.mkdtemp(prefix="holodeck-build-"))

    # Generate and write Dockerfile
    test_cases_file = _read_raw_test_cases_file(agent_dir / "agent.yaml")
    dockerfile_content = _generate_dockerfile_content(
        agent,
        deployment_config,
        version,
        test_cases_file=test_cases_file,
        base_dir=agent_dir,
    )
    (build_dir / "Dockerfile").write_text(dockerfile_content)

    # Copy agent.yaml
    shutil.copy2(agent_dir / "agent.yaml", build_dir / "agent.yaml")

    # Collect leaf files: instructions, function tool sources, single-file
    # tool sources (hierarchical_document / vectorstore), and test_cases_file.
    files_to_copy: list[str] = []
    if agent.instructions.file:
        files_to_copy.append(agent.instructions.file)
    for tool in agent.tools or []:
        if isinstance(tool, FunctionTool):
            files_to_copy.append(tool.file)
        elif isinstance(tool, VectorstoreTool | HierarchicalDocumentToolConfig):
            source_path = tool.source
            if source_path and "://" not in source_path and Path(source_path).suffix:
                files_to_copy.append(source_path)
    if test_cases_file and "://" not in test_cases_file:
        files_to_copy.append(test_cases_file)

    for rel_path in sorted(set(files_to_copy)):
        src_path = agent_dir / rel_path
        if src_path.exists():
            dst_path = build_dir / rel_path
            dst_path.parent.mkdir(parents=True, exist_ok=True)
            shutil.copy2(src_path, dst_path)

    # Copy data directories (directory-typed tool sources)
    if agent.tools:
        for tool in agent.tools:
            if isinstance(tool, VectorstoreTool | HierarchicalDocumentToolConfig):
                source_path = tool.source
                if (
                    source_path
                    and "://" not in source_path
                    and not Path(source_path).suffix
                ):
                    src = agent_dir / source_path
                    if src.exists() and src.is_dir():
                        dst = build_dir / source_path
                        shutil.copytree(src, dst, dirs_exist_ok=True)

    # Create entrypoint script
    entrypoint_content = """#!/bin/bash
set -e

# Start the HoloDeck agent server
exec holodeck serve /app/agent.yaml \\
    --host 0.0.0.0 \\
    --port "${HOLODECK_PORT:-8080}" \\
    --protocol "${HOLODECK_PROTOCOL:-rest}"
"""
    (build_dir / "entrypoint.sh").write_text(entrypoint_content)

    return build_dir
```

## `_display_build_success(result, quiet)`

Display build success message.

Parameters:

| Name     | Type          | Description                       | Default    |
| -------- | ------------- | --------------------------------- | ---------- |
| `result` | `BuildResult` | Build result with image details   | *required* |
| `quiet`  | `bool`        | If True, only show minimal output | *required* |

Source code in `src/holodeck/cli/commands/deploy.py`

```
def _display_build_success(result: BuildResult, quiet: bool) -> None:
    """Display build success message.

    Args:
        result: Build result with image details
        quiet: If True, only show minimal output
    """
    if quiet:
        click.echo(result.full_name)
        return

    click.echo()
    click.secho("=" * 60, fg="green")
    click.secho("  Build Successful!", fg="green", bold=True)
    click.secho("=" * 60, fg="green")
    click.echo()
    click.echo(f"  Image:    {result.full_name}")
    click.echo(f"  ID:       {result.image_id[:19]}...")
    click.echo()
    click.secho("  Next steps:", bold=True)
    click.echo(f"    Run locally:  docker run -p 8080:8080 {result.full_name}")
    click.echo(f"    Push to registry:  docker push {result.full_name}")
    click.echo()
```

## `_load_agent_and_deployment(agent_config)`

Source code in `src/holodeck/cli/commands/deploy.py`

```
def _load_agent_and_deployment(
    agent_config: str,
) -> tuple[Agent, DeploymentConfig, Path]:
    from holodeck.config.loader import ConfigLoader

    loader = ConfigLoader()
    agent = loader.load_agent_yaml(agent_config)

    if not agent.deployment:
        raise ConfigError(
            field="deployment",
            message="No 'deployment' section found in agent configuration",
        )

    return agent, agent.deployment, Path(agent_config).resolve()
```

## `_ensure_azure_provider(deployment_config, operation)`

Source code in `src/holodeck/cli/commands/deploy.py`

```
def _ensure_azure_provider(deployment_config: DeploymentConfig, operation: str) -> None:
    provider = deployment_config.target.provider
    if provider != CloudProvider.AZURE:
        raise DeploymentError(
            operation=operation,
            message=(
                f"Cloud provider '{provider.value}' is not supported yet. "
                "Azure Container Apps is the only supported provider for now."
            ),
        )
```

## `_resolve_image_uri(deployment_config)`

Source code in `src/holodeck/cli/commands/deploy.py`

```
def _resolve_image_uri(deployment_config: DeploymentConfig) -> tuple[str, str]:
    from holodeck.deploy.builder import generate_tag

    tag = generate_tag(
        deployment_config.registry.tag_strategy,
        deployment_config.registry.custom_tag,
    )
    registry_url = deployment_config.registry.url
    repository = deployment_config.registry.repository
    image_name = f"{registry_url}/{repository}"
    return f"{image_name}:{tag}", tag
```

______________________________________________________________________

### MCP Command

Manage MCP (Model Context Protocol) servers -- search, add, list, and remove.

## `mcp()`

Manage MCP (Model Context Protocol) servers.

Search the official MCP registry, add servers to your agent configuration, and manage installed servers.

MCP servers extend your agent's capabilities by providing access to external tools and data sources. Use 'holodeck mcp search' to discover available servers, then 'holodeck mcp add' to install them.

 EXAMPLES:

```
Search for filesystem-related servers:
    holodeck mcp search filesystem

Add a server to your agent:
    holodeck mcp add io.github.modelcontextprotocol/server-filesystem

List installed servers:
    holodeck mcp list

Remove a server:
    holodeck mcp remove filesystem
```

For more information, see: https://useholodeck.ai/docs/mcp

Source code in `src/holodeck/cli/commands/mcp.py`

```
@click.group(name="mcp")
def mcp() -> None:
    """Manage MCP (Model Context Protocol) servers.

    Search the official MCP registry, add servers to your agent configuration,
    and manage installed servers.

    MCP servers extend your agent's capabilities by providing access to
    external tools and data sources. Use 'holodeck mcp search' to discover
    available servers, then 'holodeck mcp add' to install them.

    \b
    EXAMPLES:

        Search for filesystem-related servers:
            holodeck mcp search filesystem

        Add a server to your agent:
            holodeck mcp add io.github.modelcontextprotocol/server-filesystem

        List installed servers:
            holodeck mcp list

        Remove a server:
            holodeck mcp remove filesystem

    For more information, see: https://useholodeck.ai/docs/mcp
    """
    pass
```

## `search(query, limit, as_json, verbose, quiet)`

Search the MCP registry for available servers.

QUERY is an optional search term to filter servers by name. If not provided, lists all available servers.

 EXAMPLES:

```
Search for filesystem servers:
    holodeck mcp search filesystem

List all servers (first page):
    holodeck mcp search

Get results as JSON:
    holodeck mcp search --json
```

Source code in `src/holodeck/cli/commands/mcp.py`

```
@mcp.command(name="search")
@click.argument("query", required=False)
@click.option(
    "--limit",
    default=25,
    type=click.IntRange(min=1, max=100),
    help="Maximum number of results to return (1-100, default: 25)",
)
@click.option(
    "--json",
    "as_json",
    is_flag=True,
    help="Output results as JSON",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress INFO logging output",
)
def search(
    query: str | None, limit: int, as_json: bool, verbose: bool, quiet: bool
) -> None:
    """Search the MCP registry for available servers.

    QUERY is an optional search term to filter servers by name.
    If not provided, lists all available servers.

    \b
    EXAMPLES:

        Search for filesystem servers:
            holodeck mcp search filesystem

        List all servers (first page):
            holodeck mcp search

        Get results as JSON:
            holodeck mcp search --json
    """
    # Initialize logging
    setup_logging(verbose=verbose, quiet=quiet)
    logger.debug(f"MCP search command invoked: query={query}, limit={limit}")

    try:
        with MCPRegistryClient() as client:
            result = client.search(query=query, limit=limit)

            if as_json:
                _output_json(result)
            else:
                _output_table(result)

            # Show pagination hint if more results available
            if result.next_cursor and not as_json:
                click.echo(f"\n{result.total_count} total results. More available.")

    except RegistryConnectionError as e:
        click.secho(f"Error: Registry unavailable - {e}", fg="red", err=True)
        raise SystemExit(1) from e
    except RegistryAPIError as e:
        msg = f"Error: Registry service error - {e}"
        click.secho(msg, fg="red", err=True)
        raise SystemExit(1) from e
```

## `list_cmd(agent_file, global_only, show_all, as_json, verbose, quiet)`

List installed MCP servers.

By default, shows servers from the agent configuration in the current directory. Use -g to show global servers, or --all for both.

 EXAMPLES:

```
List servers in agent.yaml:
    holodeck mcp list

List global servers:
    holodeck mcp list -g

List all servers with source labels:
    holodeck mcp list --all
```

Source code in `src/holodeck/cli/commands/mcp.py`

```
@mcp.command(name="list")
@click.option(
    "--agent",
    "agent_file",
    default="agent.yaml",
    type=click.Path(),
    help="Path to agent configuration file (default: agent.yaml)",
)
@click.option(
    "-g",
    "--global",
    "global_only",
    is_flag=True,
    help="Show only global configuration",
)
@click.option(
    "--all",
    "show_all",
    is_flag=True,
    help="Show both agent and global configurations",
)
@click.option(
    "--json",
    "as_json",
    is_flag=True,
    help="Output results as JSON",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress INFO logging output",
)
def list_cmd(
    agent_file: str,
    global_only: bool,
    show_all: bool,
    as_json: bool,
    verbose: bool,
    quiet: bool,
) -> None:
    """List installed MCP servers.

    By default, shows servers from the agent configuration in the current
    directory. Use -g to show global servers, or --all for both.

    \b
    EXAMPLES:

        List servers in agent.yaml:
            holodeck mcp list

        List global servers:
            holodeck mcp list -g

        List all servers with source labels:
            holodeck mcp list --all
    """
    # Initialize logging
    setup_logging(verbose=verbose, quiet=quiet)
    logger.debug(
        f"MCP list command invoked: agent_file={agent_file}, global_only={global_only}"
    )

    servers: list[tuple[MCPTool, str]] = []

    try:
        if show_all:
            # Show both agent and global servers
            agent_path = Path(agent_file)
            if agent_path.exists():
                agent_servers = get_mcp_servers_from_agent(agent_path)
                servers.extend((s, "agent") for s in agent_servers)

            global_servers = get_mcp_servers_from_global()
            servers.extend((s, "global") for s in global_servers)

        elif global_only:
            # Show only global servers
            global_servers = get_mcp_servers_from_global()
            servers.extend((s, "global") for s in global_servers)

        else:
            # Default: show agent servers
            agent_path = Path(agent_file)
            agent_servers = get_mcp_servers_from_agent(agent_path)
            servers.extend((s, "agent") for s in agent_servers)

        # Output results
        if as_json:
            _list_output_json(servers, show_source=show_all)
        else:
            _list_output_table(servers, show_source=show_all)

    except FileNotFoundError as e:
        if show_all:
            # For --all, continue with global servers if agent not found
            global_servers = get_mcp_servers_from_global()
            servers = [(s, "global") for s in global_servers]
            if as_json:
                _list_output_json(servers, show_source=True)
            else:
                _list_output_table(servers, show_source=True)
        else:
            click.secho(f"Error: {e.message}", fg="red", err=True)
            raise SystemExit(1) from e
    except ConfigError as e:
        click.secho(f"Error: {e.message}", fg="red", err=True)
        raise SystemExit(1) from e
```

## `add(server, agent_file, global_install, server_version, transport, custom_name, verbose, quiet)`

Add an MCP server to your configuration.

SERVER is the server name from the MCP registry (e.g., io.github.modelcontextprotocol/server-filesystem).

By default, adds to agent.yaml in the current directory. Use -g to add to global configuration (~/.holodeck/config.yaml).

 EXAMPLES:

```
Add filesystem server to agent:
    holodeck mcp add io.github.modelcontextprotocol/server-filesystem

Add to global config:
    holodeck mcp add io.github.modelcontextprotocol/server-github -g

Add specific version:
    holodeck mcp add io.github.example/server --version 1.2.0
```

Source code in `src/holodeck/cli/commands/mcp.py`

```
@mcp.command(name="add")
@click.argument("server", required=True)
@click.option(
    "--agent",
    "agent_file",
    default="agent.yaml",
    type=click.Path(),
    help="Path to agent configuration file (default: agent.yaml)",
)
@click.option(
    "-g",
    "--global",
    "global_install",
    is_flag=True,
    help="Add to global configuration (~/.holodeck/config.yaml)",
)
@click.option(
    "--version",
    "server_version",
    default="latest",
    help="Server version to install (default: latest)",
)
@click.option(
    "--transport",
    default="stdio",
    type=click.Choice(["stdio", "sse", "http"]),
    help="Transport type (default: stdio)",
)
@click.option(
    "--name",
    "custom_name",
    default=None,
    help="Custom name for the server (overrides default short name)",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress INFO logging output",
)
def add(
    server: str,
    agent_file: str,
    global_install: bool,
    server_version: str,
    transport: str,
    custom_name: str | None,
    verbose: bool,
    quiet: bool,
) -> None:
    """Add an MCP server to your configuration.

    SERVER is the server name from the MCP registry (e.g.,
    io.github.modelcontextprotocol/server-filesystem).

    By default, adds to agent.yaml in the current directory.
    Use -g to add to global configuration (~/.holodeck/config.yaml).

    \b
    EXAMPLES:

        Add filesystem server to agent:
            holodeck mcp add io.github.modelcontextprotocol/server-filesystem

        Add to global config:
            holodeck mcp add io.github.modelcontextprotocol/server-github -g

        Add specific version:
            holodeck mcp add io.github.example/server --version 1.2.0
    """
    # Initialize logging
    setup_logging(verbose=verbose, quiet=quiet)
    logger.debug(f"MCP add command invoked: server={server}, version={server_version}")

    try:
        # 1. Fetch server from registry
        with MCPRegistryClient() as client:
            registry_server = client.get_server(server, server_version)

        # 2. Find STDIO package (HoloDeck only supports stdio transport)
        stdio_pkg = find_stdio_package(registry_server)
        if stdio_pkg is None:
            available = {p.transport.type for p in registry_server.packages}
            click.secho(
                f"Error: Server '{server}' does not support stdio transport.\n"
                f"Available transports: {', '.join(sorted(available))}\n"
                "HoloDeck currently only supports stdio transport.",
                fg="red",
                err=True,
            )
            raise SystemExit(1)

        # 3. Validate registry type is supported
        if stdio_pkg.registry_type not in SUPPORTED_REGISTRY_TYPES:
            click.secho(
                f"Error: Server uses unsupported package type "
                f"'{stdio_pkg.registry_type}'.\n"
                f"Supported types: {', '.join(sorted(SUPPORTED_REGISTRY_TYPES))}.",
                fg="red",
                err=True,
            )
            raise SystemExit(1)

        # 4. Convert to MCPTool (pass specific package)
        mcp_tool = registry_to_mcp_tool(registry_server, package=stdio_pkg)

        # 5. Apply custom name if provided
        if custom_name:
            mcp_tool = mcp_tool.model_copy(update={"name": custom_name})

        # 6. Add to config (agent or global)
        if global_install:
            add_mcp_server_to_global(mcp_tool)
            target_display = "~/.holodeck/config.yaml"
        else:
            agent_path = Path(agent_file)
            add_mcp_server_to_agent(agent_path, mcp_tool)
            target_display = agent_file

        # 7. Success message
        click.secho(f"Added '{mcp_tool.name}' to {target_display}", fg="green")

        # 8. Display required environment variables
        env_vars = stdio_pkg.environment_variables
        if env_vars:
            click.echo("\nRequired environment variables:")
            for ev in env_vars:
                required_marker = " (required)" if ev.required else " (optional)"
                desc = f" - {ev.description}" if ev.description else ""
                click.echo(f"  {ev.name}{required_marker}{desc}")
            click.echo("\nSet these in your .env file or shell environment.")

    except RegistryConnectionError as e:
        click.secho(f"Error: Registry unavailable - {e}", fg="red", err=True)
        raise SystemExit(1) from e
    except RegistryAPIError as e:
        click.secho(f"Error: Registry error - {e}", fg="red", err=True)
        raise SystemExit(1) from e
    except ServerNotFoundError as e:
        click.secho(
            f"Error: Server '{server}' not found in registry", fg="red", err=True
        )
        raise SystemExit(1) from e
    except DuplicateServerError as e:
        click.secho(f"Error: {e}", fg="red", err=True)
        raise SystemExit(1) from e
    except FileNotFoundError as e:
        click.secho(f"Error: {e.message}", fg="red", err=True)
        raise SystemExit(1) from e
    except ConfigError as e:
        click.secho(f"Error: {e.message}", fg="red", err=True)
        raise SystemExit(1) from e
```

## `remove(server, agent_file, global_remove, verbose, quiet)`

Remove an MCP server from your configuration.

SERVER is the name of the server to remove (e.g., 'filesystem').

By default, removes from agent.yaml in the current directory. Use -g to remove from global configuration.

 EXAMPLES:

```
Remove from agent config:
    holodeck mcp remove filesystem

Remove from global config:
    holodeck mcp remove github -g
```

Source code in `src/holodeck/cli/commands/mcp.py`

```
@mcp.command(name="remove")
@click.argument("server", required=True)
@click.option(
    "--agent",
    "agent_file",
    default="agent.yaml",
    type=click.Path(),
    help="Path to agent configuration file (default: agent.yaml)",
)
@click.option(
    "-g",
    "--global",
    "global_remove",
    is_flag=True,
    help="Remove from global configuration",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress INFO logging output",
)
def remove(
    server: str,
    agent_file: str,
    global_remove: bool,
    verbose: bool,
    quiet: bool,
) -> None:
    """Remove an MCP server from your configuration.

    SERVER is the name of the server to remove (e.g., 'filesystem').

    By default, removes from agent.yaml in the current directory.
    Use -g to remove from global configuration.

    \b
    EXAMPLES:

        Remove from agent config:
            holodeck mcp remove filesystem

        Remove from global config:
            holodeck mcp remove github -g
    """
    # Initialize logging
    setup_logging(verbose=verbose, quiet=quiet)
    logger.debug(f"MCP remove command invoked: server={server}, global={global_remove}")

    try:
        if global_remove:
            remove_mcp_server_from_global(server)
            target_display = "~/.holodeck/config.yaml"
        else:
            agent_path = Path(agent_file)
            remove_mcp_server_from_agent(agent_path, server)
            target_display = agent_file

        click.secho(f"Removed '{server}' from {target_display}", fg="green")

    except ServerNotFoundError as e:
        click.secho(f"Error: {e}", fg="red", err=True)
        raise SystemExit(1) from e

    except FileNotFoundError as e:
        click.secho(f"Error: {e}", fg="red", err=True)
        raise SystemExit(1) from e

    except ConfigError as e:
        click.secho(f"Error: {e}", fg="red", err=True)
        raise SystemExit(1) from e
```

#### MCP Helper Functions

## `_truncate(text, max_len)`

Truncate text with ellipsis if too long.

Parameters:

| Name      | Type  | Description                        | Default    |
| --------- | ----- | ---------------------------------- | ---------- |
| `text`    | `str` | The text to truncate.              | *required* |
| `max_len` | `int` | Maximum length including ellipsis. | *required* |

Returns:

| Type  | Description                                       |
| ----- | ------------------------------------------------- |
| `str` | Truncated text with ellipsis if exceeded max_len. |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _truncate(text: str, max_len: int) -> str:
    """Truncate text with ellipsis if too long.

    Args:
        text: The text to truncate.
        max_len: Maximum length including ellipsis.

    Returns:
        Truncated text with ellipsis if exceeded max_len.
    """
    if len(text) <= max_len:
        return text
    return text[: max_len - 3] + "..."
```

## `_get_transports(server)`

Get comma-separated transport types from server packages.

Parameters:

| Name     | Type             | Description                               | Default    |
| -------- | ---------------- | ----------------------------------------- | ---------- |
| `server` | `RegistryServer` | Registry server with package information. | *required* |

Returns:

| Type  | Description                                                |
| ----- | ---------------------------------------------------------- |
| `str` | Comma-separated transport types, or "stdio" if none found. |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _get_transports(server: RegistryServer) -> str:
    """Get comma-separated transport types from server packages.

    Args:
        server: Registry server with package information.

    Returns:
        Comma-separated transport types, or "stdio" if none found.
    """
    transports: set[str] = set()
    for pkg in server.packages:
        transports.add(pkg.transport.type)
    return ", ".join(sorted(transports)) or "stdio"
```

## `_get_transport_list(server)`

Get list of transport types for JSON output.

Parameters:

| Name     | Type             | Description                               | Default    |
| -------- | ---------------- | ----------------------------------------- | ---------- |
| `server` | `RegistryServer` | Registry server with package information. | *required* |

Returns:

| Type        | Description                                                 |
| ----------- | ----------------------------------------------------------- |
| `list[str]` | Sorted list of transport types, or ["stdio"] if none found. |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _get_transport_list(server: RegistryServer) -> list[str]:
    """Get list of transport types for JSON output.

    Args:
        server: Registry server with package information.

    Returns:
        Sorted list of transport types, or ["stdio"] if none found.
    """
    transports: set[str] = set()
    for pkg in server.packages:
        transports.add(pkg.transport.type)
    return sorted(transports) if transports else ["stdio"]
```

## `_get_version_display(server)`

Get version display string for table output.

Shows single version if only one, or latest version with count for multiple.

Parameters:

| Name     | Type             | Description                    | Default    |
| -------- | ---------------- | ------------------------------ | ---------- |
| `server` | `RegistryServer` | Registry server with versions. | *required* |

Returns:

| Type  | Description                                             |
| ----- | ------------------------------------------------------- |
| `str` | Version display string (e.g., "1.0.0" or "1.0.0 (+2)"). |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _get_version_display(server: RegistryServer) -> str:
    """Get version display string for table output.

    Shows single version if only one, or latest version with count for multiple.

    Args:
        server: Registry server with versions.

    Returns:
        Version display string (e.g., "1.0.0" or "1.0.0 (+2)").
    """
    if server.versions:
        latest = server.versions[0].version or server.version or "-"
        if len(server.versions) == 1:
            return latest
        # Show latest version with additional count
        return f"{latest} (+{len(server.versions) - 1})"
    return server.version or "-"
```

## `_output_table(result)`

Format search results as a table.

Parameters:

| Name     | Type           | Description                          | Default    |
| -------- | -------------- | ------------------------------------ | ---------- |
| `result` | `SearchResult` | Search result from the MCP registry. | *required* |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _output_table(result: SearchResult) -> None:
    """Format search results as a table.

    Args:
        result: Search result from the MCP registry.
    """
    if not result.servers:
        click.echo("No servers found.")
        return

    # Calculate column widths based on content
    name_width = min(40, max(len(s.name) for s in result.servers))
    version_width = 12
    desc_width = 35

    # Header
    click.echo(
        f"{'NAME':<{name_width}}  {'VERSION':<{version_width}}  "
        f"{'DESCRIPTION':<{desc_width}}  TRANSPORT"
    )
    click.echo("-" * (name_width + version_width + desc_width + 18))

    # Rows
    for server in result.servers:
        name = _truncate(server.name, name_width)
        version = _get_version_display(server)
        desc = _truncate(server.description, desc_width)
        transports = _get_transports(server)
        click.echo(
            f"{name:<{name_width}}  {version:<{version_width}}  "
            f"{desc:<{desc_width}}  {transports}"
        )
```

## `_output_json(result)`

Format search results as JSON.

Parameters:

| Name     | Type           | Description                          | Default    |
| -------- | -------------- | ------------------------------------ | ---------- |
| `result` | `SearchResult` | Search result from the MCP registry. | *required* |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _output_json(result: SearchResult) -> None:
    """Format search results as JSON.

    Args:
        result: Search result from the MCP registry.
    """
    output = {
        "servers": [
            {
                "name": s.name,
                "description": s.description,
                "transports": _get_transport_list(s),
                "versions": _format_version_for_json(s),
            }
            for s in result.servers
        ],
        "total_count": result.total_count,
        "has_more": result.next_cursor is not None,
    }
    click.echo(json.dumps(output, indent=2))
```

## `_format_version_for_json(server)`

Format version details for JSON output.

Parameters:

| Name     | Type             | Description                    | Default    |
| -------- | ---------------- | ------------------------------ | ---------- |
| `server` | `RegistryServer` | Registry server with versions. | *required* |

Returns:

| Type                      | Description                          |
| ------------------------- | ------------------------------------ |
| `list[dict[str, object]]` | List of version detail dictionaries. |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _format_version_for_json(server: RegistryServer) -> list[dict[str, object]]:
    """Format version details for JSON output.

    Args:
        server: Registry server with versions.

    Returns:
        List of version detail dictionaries.
    """
    if not server.versions:
        # Fallback if versions not populated
        return [{"version": server.version}]

    versions_output: list[dict[str, object]] = []
    for v in server.versions:
        version_info: dict[str, object] = {
            "version": v.version,
            "packages": [
                {
                    "registry_type": p.registry_type,
                    "identifier": p.identifier,
                    "version": p.version,
                    "transport": p.transport.type,
                }
                for p in v.packages
            ],
        }
        # Add metadata if available
        if v.meta:
            version_info["published_at"] = (
                v.meta.published_at.isoformat() if v.meta.published_at else None
            )
            version_info["is_latest"] = v.meta.is_latest
            version_info["status"] = v.meta.status
        versions_output.append(version_info)

    return versions_output
```

## `_extract_version_from_args(mcp_tool)`

Extract version from MCP tool args.

Parses the args list to find version specifiers like:

- @modelcontextprotocol/server-filesystem@1.0.0 -> 1.0.0
- package-name@2.3.4-beta -> 2.3.4-beta

Parameters:

| Name       | Type      | Description      | Default    |
| ---------- | --------- | ---------------- | ---------- |
| `mcp_tool` | `MCPTool` | MCPTool instance | *required* |

Returns:

| Type  | Description                        |
| ----- | ---------------------------------- |
| `str` | Version string or "-" if not found |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _extract_version_from_args(mcp_tool: MCPTool) -> str:
    """Extract version from MCP tool args.

    Parses the args list to find version specifiers like:
    - @modelcontextprotocol/server-filesystem@1.0.0 -> 1.0.0
    - package-name@2.3.4-beta -> 2.3.4-beta

    Args:
        mcp_tool: MCPTool instance

    Returns:
        Version string or "-" if not found
    """
    if not mcp_tool.args:
        return "-"

    for arg in mcp_tool.args:
        match = VERSION_PATTERN.search(arg)
        if match:
            return match.group(1)

    return "-"
```

## `_list_output_table(servers, show_source)`

Format installed servers list as a table.

Parameters:

| Name          | Type                        | Description                                                          | Default    |
| ------------- | --------------------------- | -------------------------------------------------------------------- | ---------- |
| `servers`     | `list[tuple[MCPTool, str]]` | List of (MCPTool, source) tuples where source is "agent" or "global" | *required* |
| `show_source` | `bool`                      | Whether to show SOURCE column (for --all mode)                       | *required* |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _list_output_table(servers: list[tuple[MCPTool, str]], show_source: bool) -> None:
    """Format installed servers list as a table.

    Args:
        servers: List of (MCPTool, source) tuples where source is "agent" or "global"
        show_source: Whether to show SOURCE column (for --all mode)
    """
    if not servers:
        click.echo(
            "No MCP servers configured. "
            "Use 'holodeck mcp search' to find available servers."
        )
        return

    # Calculate column widths based on content
    name_width = min(25, max(len(s[0].name) for s in servers))
    version_width = 12
    transport_width = 10
    desc_width = 40 if not show_source else 30

    # Header
    header = (
        f"{'NAME':<{name_width}}  {'VERSION':<{version_width}}  "
        f"{'TRANSPORT':<{transport_width}}  {'DESCRIPTION':<{desc_width}}"
    )
    if show_source:
        header += "  SOURCE"
    click.echo(header)

    # Separator
    sep_width = name_width + version_width + transport_width + desc_width + 8
    if show_source:
        sep_width += 8
    click.echo("-" * sep_width)

    # Rows
    for mcp_tool, source in servers:
        name = _truncate(mcp_tool.name, name_width)
        version = _extract_version_from_args(mcp_tool)
        transport = mcp_tool.transport.value if mcp_tool.transport else "stdio"
        desc = _truncate(mcp_tool.description, desc_width)

        row = (
            f"{name:<{name_width}}  {version:<{version_width}}  "
            f"{transport:<{transport_width}}  {desc:<{desc_width}}"
        )
        if show_source:
            row += f"  {source}"
        click.echo(row)
```

## `_list_output_json(servers, show_source)`

Format installed servers list as JSON.

Parameters:

| Name          | Type                        | Description                      | Default    |
| ------------- | --------------------------- | -------------------------------- | ---------- |
| `servers`     | `list[tuple[MCPTool, str]]` | List of (MCPTool, source) tuples | *required* |
| `show_source` | `bool`                      | Whether to include source field  | *required* |

Source code in `src/holodeck/cli/commands/mcp.py`

```
def _list_output_json(servers: list[tuple[MCPTool, str]], show_source: bool) -> None:
    """Format installed servers list as JSON.

    Args:
        servers: List of (MCPTool, source) tuples
        show_source: Whether to include source field
    """
    output_servers = []
    for mcp_tool, source in servers:
        server_dict: dict[str, object] = {
            "name": mcp_tool.name,
            "description": mcp_tool.description,
            "version": _extract_version_from_args(mcp_tool),
            "transport": mcp_tool.transport.value if mcp_tool.transport else "stdio",
        }

        # Include additional fields if present
        if mcp_tool.command:
            server_dict["command"] = mcp_tool.command.value
        if mcp_tool.args:
            server_dict["args"] = mcp_tool.args
        if mcp_tool.registry_name:
            server_dict["registry_name"] = mcp_tool.registry_name
        if show_source:
            server_dict["source"] = source

        output_servers.append(server_dict)

    output = {
        "servers": output_servers,
        "total_count": len(output_servers),
    }
    click.echo(json.dumps(output, indent=2))
```

______________________________________________________________________

### Serve Command

Start an HTTP server exposing an agent via AG-UI or REST protocol.

## `serve(agent_config, port, host, protocol, verbose, quiet, cors_origins)`

Start an HTTP server exposing an agent.

AGENT_CONFIG is the path to the agent.yaml configuration file.

Example:

```
holodeck serve examples/weather-agent.yaml

holodeck serve examples/assistant.yaml --port 9000 --protocol ag-ui
```

The server exposes the agent via HTTP with the specified protocol.

Protocols:

```
ag-ui   AG-UI protocol (streaming SSE events)
rest    REST API (JSON request/response)
```

Options:

```
--port / -p         Port to listen on (default: 8000)
--host / -h         Host to bind to (default: 127.0.0.1)
--protocol          Protocol to use: ag-ui or rest (default: ag-ui)
--verbose / -v      Enable verbose debug logging
--quiet / -q        Suppress INFO logging output
--cors-origins      Comma-separated CORS origins (default: *)
```

Source code in `src/holodeck/cli/commands/serve.py`

```
@click.command()
@click.argument(
    "agent_config",
    type=click.Path(exists=True),
    default="agent.yaml",
    required=False,
)
@click.option(
    "--port",
    "-p",
    type=int,
    default=8000,
    help="Port to listen on (default: 8000)",
)
@click.option(
    "--host",
    "-h",
    type=str,
    default="127.0.0.1",
    help="Host to bind to (default: 127.0.0.1 for local-only access)",
)
@click.option(
    "--protocol",
    type=click.Choice(["ag-ui", "rest"]),
    default="ag-ui",
    help="Protocol to use (default: ag-ui)",
)
@click.option(
    "--verbose",
    "-v",
    is_flag=True,
    help="Enable verbose debug logging",
)
@click.option(
    "--quiet",
    "-q",
    is_flag=True,
    help="Suppress INFO logging output",
)
@click.option(
    "--cors-origins",
    type=str,
    default="http://localhost:3000",
    help="Comma-separated list of allowed CORS origins (default: http://localhost:3000)",
)
def serve(
    agent_config: str,
    port: int,
    host: str,
    protocol: str,
    verbose: bool,
    quiet: bool,
    cors_origins: str,
) -> None:
    """Start an HTTP server exposing an agent.

    AGENT_CONFIG is the path to the agent.yaml configuration file.

    Example:

        holodeck serve examples/weather-agent.yaml

        holodeck serve examples/assistant.yaml --port 9000 --protocol ag-ui

    The server exposes the agent via HTTP with the specified protocol.

    Protocols:

        ag-ui   AG-UI protocol (streaming SSE events)
        rest    REST API (JSON request/response)

    Options:

        --port / -p         Port to listen on (default: 8000)
        --host / -h         Host to bind to (default: 127.0.0.1)
        --protocol          Protocol to use: ag-ui or rest (default: ag-ui)
        --verbose / -v      Enable verbose debug logging
        --quiet / -q        Suppress INFO logging output
        --cors-origins      Comma-separated CORS origins (default: *)
    """
    # Initialize observability context (will be set if observability enabled)
    obs_context: ObservabilityContext | None = None

    try:
        # Load agent config and resolve execution config in one call
        from holodeck.config.loader import load_agent_with_config

        agent, resolved_config, _loader = load_agent_with_config(agent_config)

        # Determine logging strategy: OTel replaces setup_logging when enabled
        if agent.observability and agent.observability.enabled:
            # OTel handles all logging - skip setup_logging
            # Console exporter is NOT force-enabled; use agent.yaml config
            # to enable it explicitly if needed.
            obs_context = initialize_observability(
                agent.observability, agent.name, verbose=verbose, quiet=quiet
            )
        else:
            # Traditional logging
            setup_logging(verbose=verbose, quiet=quiet)

        logger.info(
            f"Serve command invoked: config={agent_config}, "
            f"port={port}, host={host}, protocol={protocol}, verbose={verbose}"
        )
        logger.debug(f"Loading agent configuration from {agent_config}")
        logger.info(f"Agent configuration loaded successfully: {agent.name}")

        logger.debug(
            f"Resolved execution config: llm_timeout={resolved_config.llm_timeout}, "
            f"file_timeout={resolved_config.file_timeout}"
        )

        # Parse CORS origins
        origins = [o.strip() for o in cors_origins.split(",") if o.strip()]

        # Map protocol string to ProtocolType
        from holodeck.serve.models import ProtocolType

        protocol_type = ProtocolType.AG_UI if protocol == "ag-ui" else ProtocolType.REST

        # Determine if observability is enabled for span creation
        observability_enabled = obs_context is not None

        # Create and run server
        asyncio.run(
            _run_server(
                agent=agent,
                host=host,
                port=port,
                protocol=protocol_type,
                cors_origins=origins,
                verbose=verbose,
                execution_config=resolved_config,
                observability_enabled=observability_enabled,
            )
        )

    except ConfigError as e:
        logger.error(f"Configuration error: {e}", exc_info=True)
        click.secho("Error: Failed to load agent configuration", fg="red", err=True)
        click.echo(f"  {str(e)}", err=True)
        sys.exit(1)
    except KeyboardInterrupt:
        logger.info("Server interrupted by user (Ctrl+C)")
        click.echo()
        click.secho("Server stopped.", fg="yellow")
        sys.exit(130)
    except Exception as e:
        logger.error(f"Unexpected error: {e}", exc_info=True)
        click.secho(f"Error: {str(e)}", fg="red", err=True)
        sys.exit(1)
    finally:
        # Shutdown observability if it was initialized
        if obs_context:
            shutdown_observability(obs_context)
```

## `_run_server(agent, host, port, protocol, cors_origins, verbose, execution_config, observability_enabled=False)`

Run the HTTP server.

Parameters:

| Name                    | Type              | Description                               | Default    |
| ----------------------- | ----------------- | ----------------------------------------- | ---------- |
| `agent`                 | `Agent`           | Loaded Agent configuration.               | *required* |
| `host`                  | `str`             | Host to bind to.                          | *required* |
| `port`                  | `int`             | Port to listen on.                        | *required* |
| `protocol`              | `ProtocolType`    | Protocol type (AG-UI or REST).            | *required* |
| `cors_origins`          | `list[str]`       | List of allowed CORS origins.             | *required* |
| `verbose`               | `bool`            | Enable verbose debug logging.             | *required* |
| `execution_config`      | `ExecutionConfig` | Resolved execution configuration.         | *required* |
| `observability_enabled` | `bool`            | Enable OpenTelemetry per-request tracing. | `False`    |

Source code in `src/holodeck/cli/commands/serve.py`

```
async def _run_server(
    agent: Agent,
    host: str,
    port: int,
    protocol: ProtocolType,
    cors_origins: list[str],
    verbose: bool,
    execution_config: ExecutionConfig,
    observability_enabled: bool = False,
) -> None:
    """Run the HTTP server.

    Args:
        agent: Loaded Agent configuration.
        host: Host to bind to.
        port: Port to listen on.
        protocol: Protocol type (AG-UI or REST).
        cors_origins: List of allowed CORS origins.
        verbose: Enable verbose debug logging.
        execution_config: Resolved execution configuration.
        observability_enabled: Enable OpenTelemetry per-request tracing.
    """
    # Create parent span for serve command if observability is enabled
    if observability_enabled:
        from holodeck.lib.observability import get_tracer

        tracer = get_tracer(__name__)
        span_context: Any = tracer.start_as_current_span("holodeck.cli.serve")
    else:
        span_context = nullcontext()

    with span_context:
        import uvicorn

        from holodeck.serve.server import AgentServer

        # Create server
        server = AgentServer(
            agent_config=agent,
            protocol=protocol,
            host=host,
            port=port,
            cors_origins=cors_origins,
            debug=verbose,
            execution_config=execution_config,
            observability_enabled=observability_enabled,
        )

        # Create app
        app = server.create_app()

        # Start server lifecycle
        await server.start()

        # Display startup info
        _display_startup_info(agent, protocol, host, port)

        # Configure uvicorn
        config = uvicorn.Config(
            app=app,
            host=host,
            port=port,
            log_level="debug" if verbose else "info",
        )
        server_instance = uvicorn.Server(config)

        try:
            await server_instance.serve()
        finally:
            await server.stop()
```

## `_display_startup_info(agent, protocol, host, port)`

Display server startup information.

Parameters:

| Name       | Type           | Description                      | Default    |
| ---------- | -------------- | -------------------------------- | ---------- |
| `agent`    | `Agent`        | Agent configuration.             | *required* |
| `protocol` | `ProtocolType` | Protocol type.                   | *required* |
| `host`     | `str`          | Host the server is bound to.     | *required* |
| `port`     | `int`          | Port the server is listening on. | *required* |

Source code in `src/holodeck/cli/commands/serve.py`

```
def _display_startup_info(
    agent: Agent,
    protocol: ProtocolType,
    host: str,
    port: int,
) -> None:
    """Display server startup information.

    Args:
        agent: Agent configuration.
        protocol: Protocol type.
        host: Host the server is bound to.
        port: Port the server is listening on.
    """
    from holodeck.serve.models import ProtocolType

    click.echo()
    click.secho("=" * 60, fg="cyan")
    click.secho("  HoloDeck Agent Server", fg="cyan", bold=True)
    click.secho("=" * 60, fg="cyan")
    click.echo()
    click.echo(f"  Agent:    {agent.name}")
    click.echo(f"  Protocol: {protocol.value}")
    click.echo(f"  URL:      http://{host}:{port}")
    click.echo()
    click.secho("  Endpoints:", bold=True)

    if protocol == ProtocolType.AG_UI:
        click.echo(
            "    POST /awp                               AG-UI protocol endpoint"
        )
    else:
        click.echo(f"    POST /agent/{agent.name}/chat               Sync chat (JSON)")
        click.echo(f"    POST /agent/{agent.name}/chat/stream        Stream (SSE)")
        click.echo(f"    POST /agent/{agent.name}/chat/multipart     Sync (multipart)")
        click.echo(
            f"    POST /agent/{agent.name}/chat/stream/multipart  Stream (multipart)"
        )
        click.echo("    DELETE /sessions/{session_id}               Delete session")
        click.echo("    GET  /docs                                  OpenAPI docs")

    click.echo("    GET  /health                           Health check")
    click.echo("    GET  /ready                            Readiness check")
    click.echo()
    click.secho("  Press Ctrl+C to stop", fg="yellow")
    click.secho("=" * 60, fg="cyan")
    click.echo()
```

______________________________________________________________________

## CLI Utilities

### Project Initializer

Project initialization and scaffolding logic.

## `ProjectInitializer()`

Handles project initialization logic.

Provides methods to:

- Validate user inputs (project name, template, permissions)
- Load and validate template manifests
- Initialize new agent projects with all required files

Initialize the ProjectInitializer.

Source code in `src/holodeck/cli/utils/project_init.py`

```
def __init__(self) -> None:
    """Initialize the ProjectInitializer."""
    self.template_renderer = TemplateRenderer()
    # Get available templates from discovery function
    self.available_templates = set(TemplateRenderer.list_available_templates())
```

### `validate_inputs(input_data)`

Validate user inputs for project initialization.

Checks:

- Project name format (alphanumeric, hyphens, underscores, no leading digits)
- Project name is not empty and within length limits
- Template exists in available templates
- Output directory is writable
- Project directory doesn't already exist (unless overwrite is True)

Parameters:

| Name         | Type               | Description                                | Default    |
| ------------ | ------------------ | ------------------------------------------ | ---------- |
| `input_data` | `ProjectInitInput` | ProjectInitInput with user-provided values | *required* |

Raises:

| Type              | Description                   |
| ----------------- | ----------------------------- |
| `ValidationError` | If any validation checks fail |

Source code in `src/holodeck/cli/utils/project_init.py`

```
def validate_inputs(self, input_data: ProjectInitInput) -> None:
    """Validate user inputs for project initialization.

    Checks:
    - Project name format (alphanumeric, hyphens, underscores, no leading digits)
    - Project name is not empty and within length limits
    - Template exists in available templates
    - Output directory is writable
    - Project directory doesn't already exist (unless overwrite is True)

    Args:
        input_data: ProjectInitInput with user-provided values

    Raises:
        ValidationError: If any validation checks fail
    """
    project_name = input_data.project_name.strip()

    # Check project name is not empty
    if not project_name:
        raise ValidationError("Project name cannot be empty")

    # Check project name length
    if len(project_name) > self.MAX_PROJECT_NAME_LENGTH:
        raise ValidationError(
            f"Project name cannot exceed {self.MAX_PROJECT_NAME_LENGTH} characters"
        )

    # Check project name format
    if not re.match(self.PROJECT_NAME_PATTERN, project_name):
        raise ValidationError(
            f"Invalid project name: '{project_name}'. "
            "Project names must start with a letter or underscore, "
            "and contain only alphanumeric characters, hyphens, and underscores."
        )

    # Check template exists
    if input_data.template not in self.available_templates:
        templates_list = ", ".join(sorted(self.available_templates))
        raise ValidationError(
            f"Unknown template: '{input_data.template}'. "
            f"Available templates: {templates_list}"
        )

    # Check output directory is writable
    output_dir = Path(input_data.output_dir)
    if not output_dir.exists():
        raise ValidationError(f"Output directory does not exist: {output_dir}")

    if not output_dir.is_dir():
        raise ValidationError(f"Output path is not a directory: {output_dir}")

    try:
        # Test write permissions by attempting to check access
        if not os.access(str(output_dir), os.W_OK):
            raise ValidationError(f"Output directory is not writable: {output_dir}")
    except OSError as e:
        raise ValidationError(f"Cannot access output directory: {e}") from e

    # Check project directory doesn't already exist (unless force)
    project_dir = output_dir / project_name
    if project_dir.exists() and not input_data.overwrite:
        raise ValidationError(
            f"Project directory already exists: {project_dir}. "
            "Use --force to overwrite."
        )
```

### `load_template(template_name)`

Load and validate a template manifest.

Loads the manifest.yaml file from a template directory and validates it against the TemplateManifest schema.

Parameters:

| Name            | Type  | Description                                   | Default    |
| --------------- | ----- | --------------------------------------------- | ---------- |
| `template_name` | `str` | Name of the template (e.g., 'conversational') | *required* |

Returns:

| Name               | Type               | Description                            |
| ------------------ | ------------------ | -------------------------------------- |
| `TemplateManifest` | `TemplateManifest` | Parsed and validated template manifest |

Raises:

| Type                | Description                               |
| ------------------- | ----------------------------------------- |
| `FileNotFoundError` | If template or manifest file not found    |
| `InitError`         | If manifest cannot be parsed or validated |

Source code in `src/holodeck/cli/utils/project_init.py`

```
def load_template(self, template_name: str) -> TemplateManifest:
    """Load and validate a template manifest.

    Loads the manifest.yaml file from a template directory and validates
    it against the TemplateManifest schema.

    Args:
        template_name: Name of the template (e.g., 'conversational')

    Returns:
        TemplateManifest: Parsed and validated template manifest

    Raises:
        FileNotFoundError: If template or manifest file not found
        InitError: If manifest cannot be parsed or validated
    """
    # Get template directory
    # Templates are bundled in src/holodeck/templates/
    template_dir = Path(__file__).parent.parent.parent / "templates" / template_name

    if not template_dir.exists():
        raise FileNotFoundError(f"Template directory not found: {template_dir}")

    manifest_path = template_dir / "manifest.yaml"

    if not manifest_path.exists():
        raise FileNotFoundError(f"Template manifest not found: {manifest_path}")

    try:
        with open(manifest_path) as f:
            manifest_data = yaml.safe_load(f)

        if not manifest_data:
            raise InitError(f"Template manifest is empty: {manifest_path}")

        # Validate against TemplateManifest schema
        manifest = TemplateManifest.model_validate(manifest_data)
        return manifest

    except yaml.YAMLError as e:
        raise InitError(f"Template manifest contains invalid YAML: {e}") from e
    except Exception as e:
        if isinstance(e, ValidationError | InitError):
            raise
        raise InitError(f"Failed to load template manifest: {e}") from e
```

### `initialize(input_data)`

Initialize a new agent project.

Creates a new project directory with all required files and templates. Follows all-or-nothing semantics: either the entire project is created successfully, or no files are created and the directory is cleaned up.

Parameters:

| Name         | Type               | Description                                 | Default    |
| ------------ | ------------------ | ------------------------------------------- | ---------- |
| `input_data` | `ProjectInitInput` | ProjectInitInput with validated user inputs | *required* |

Returns:

| Name                | Type                | Description                                       |
| ------------------- | ------------------- | ------------------------------------------------- |
| `ProjectInitResult` | `ProjectInitResult` | Result of initialization with status and metadata |

Raises:

| Type        | Description                                    |
| ----------- | ---------------------------------------------- |
| `InitError` | If initialization fails (will attempt cleanup) |

Source code in `src/holodeck/cli/utils/project_init.py`

```
def initialize(self, input_data: ProjectInitInput) -> ProjectInitResult:
    """Initialize a new agent project.

    Creates a new project directory with all required files and templates.
    Follows all-or-nothing semantics: either the entire project is created
    successfully, or no files are created and the directory is cleaned up.

    Args:
        input_data: ProjectInitInput with validated user inputs

    Returns:
        ProjectInitResult: Result of initialization with status and metadata

    Raises:
        InitError: If initialization fails (will attempt cleanup)
    """
    start_time = time.time()
    project_name = input_data.project_name.strip()
    output_dir = Path(input_data.output_dir)
    project_dir = output_dir / project_name

    files_created = []

    try:
        # Validate inputs first
        self.validate_inputs(input_data)

        # Load template manifest
        template = self.load_template(input_data.template)

        # Create project directory
        if project_dir.exists() and input_data.overwrite:
            # Remove existing directory if force flag is set
            shutil.rmtree(project_dir)

        project_dir.mkdir(parents=True, exist_ok=False)
        files_created.append(str(project_dir))

        # Prepare provider-specific config
        provider_config = input_data.provider_config
        endpoint_env_var = get_provider_endpoint_env_var(input_data.llm_provider)

        # Determine endpoint value
        llm_endpoint = None
        if provider_config and provider_config.endpoint:
            llm_endpoint = provider_config.endpoint
        elif endpoint_env_var:
            # Use environment variable placeholder as default
            llm_endpoint = f"${{{endpoint_env_var}}}"

        # Prepare template variables
        template_vars = {
            "project_name": project_name,
            "description": input_data.description or "TODO: Add agent description",
            "author": input_data.author or "",
            # Wizard configuration fields
            "agent_name": input_data.agent_name,
            "llm_provider": input_data.llm_provider,
            "llm_model": get_model_for_provider(input_data.llm_provider),
            "llm_endpoint": llm_endpoint,
            "llm_api_key_env_var": get_provider_api_key_env_var(
                input_data.llm_provider
            ),
            "vector_store": input_data.vector_store,
            "vector_store_endpoint": get_vectorstore_endpoint(
                input_data.vector_store
            ),
            "evals": input_data.evals,
            "mcp_servers": [
                get_mcp_server_config(s) for s in input_data.mcp_servers
            ],
        }

        # Add template-specific defaults from manifest
        if template.defaults:
            template_vars.update(template.defaults)

        # Create files from template
        template_dir = (
            Path(__file__).parent.parent.parent / "templates" / input_data.template
        )

        # Process each file in the template manifest
        if template.files:
            for file_spec in template.files.values():
                if not file_spec.required:
                    continue

                file_path = project_dir / file_spec.path
                file_path.parent.mkdir(parents=True, exist_ok=True)

                if file_spec.template:
                    # Render Jinja2 template
                    template_file = template_dir / f"{file_spec.path}.j2"
                    if not template_file.exists():
                        # Try without .j2 extension
                        template_file = template_dir / file_spec.path

                    if file_path.suffix == ".yaml" or file_path.suffix == ".yml":
                        # Validate YAML files against schema
                        content = self.template_renderer.render_and_validate(
                            str(template_file), template_vars
                        )
                    else:
                        # Render non-YAML files normally
                        content = self.template_renderer.render_template(
                            str(template_file), template_vars
                        )

                    file_path.write_text(content)
                else:
                    # Copy static files directly
                    source_file = template_dir / file_spec.path
                    if source_file.exists():
                        shutil.copy2(source_file, file_path)

                files_created.append(str(file_path.relative_to(output_dir)))

        # Also copy .gitignore if it exists
        gitignore_src = template_dir / ".gitignore"
        if gitignore_src.exists():
            gitignore_dst = project_dir / ".gitignore"
            shutil.copy2(gitignore_src, gitignore_dst)
            files_created.append(str(gitignore_dst.relative_to(output_dir)))

        duration = time.time() - start_time

        return ProjectInitResult(
            success=True,
            project_name=project_name,
            project_path=str(project_dir),
            template_used=input_data.template,
            files_created=files_created,
            warnings=[],
            errors=[],
            duration_seconds=duration,
        )

    except (ValidationError, InitError) as e:
        # Clean up partial directory on error
        if project_dir.exists():
            with contextlib.suppress(Exception):
                shutil.rmtree(project_dir)

        duration = time.time() - start_time

        return ProjectInitResult(
            success=False,
            project_name=project_name,
            project_path=str(project_dir),
            template_used=input_data.template,
            files_created=[],
            warnings=[],
            errors=[str(e)],
            duration_seconds=duration,
        )

    except Exception as e:
        # Clean up partial directory on unexpected error
        if project_dir.exists():
            with contextlib.suppress(Exception):
                shutil.rmtree(project_dir)

        duration = time.time() - start_time

        return ProjectInitResult(
            success=False,
            project_name=project_name,
            project_path=str(project_dir),
            template_used=input_data.template,
            files_created=[],
            warnings=[],
            errors=[f"Unexpected error: {str(e)}"],
            duration_seconds=duration,
        )
```

## `get_model_for_provider(provider)`

Get the default model for an LLM provider.

Parameters:

| Name       | Type  | Description                                         | Default    |
| ---------- | ----- | --------------------------------------------------- | ---------- |
| `provider` | `str` | LLM provider identifier (e.g., 'ollama', 'openai'). | *required* |

Returns:

| Type  | Description                          |
| ----- | ------------------------------------ |
| `str` | Default model name for the provider. |

Source code in `src/holodeck/cli/utils/project_init.py`

```
def get_model_for_provider(provider: str) -> str:
    """Get the default model for an LLM provider.

    Args:
        provider: LLM provider identifier (e.g., 'ollama', 'openai').

    Returns:
        Default model name for the provider.
    """
    for choice in LLM_PROVIDER_CHOICES:
        if choice.value == provider:
            return choice.default_model
    return "gpt-oss:20b"  # Fallback to Ollama default
```

## `get_mcp_server_config(server_id)`

Get configuration for an MCP server.

Parameters:

| Name        | Type  | Description                                             | Default    |
| ----------- | ----- | ------------------------------------------------------- | ---------- |
| `server_id` | `str` | MCP server identifier (e.g., 'brave-search', 'memory'). | *required* |

Returns:

| Type             | Description                                                    |
| ---------------- | -------------------------------------------------------------- |
| `dict[str, str]` | Dictionary with server configuration (name, package, command). |

Source code in `src/holodeck/cli/utils/project_init.py`

```
def get_mcp_server_config(server_id: str) -> dict[str, str]:
    """Get configuration for an MCP server.

    Args:
        server_id: MCP server identifier (e.g., 'brave-search', 'memory').

    Returns:
        Dictionary with server configuration (name, package, command).
    """
    for server in MCP_SERVER_CHOICES:
        if server.value == server_id:
            return {
                "name": sanitize_tool_name(server.value),
                "display_name": server.display_name,
                "description": server.description,
                "package": server.package_identifier,
                "command": server.command,
            }
    return {
        "name": sanitize_tool_name(server_id),
        "display_name": server_id,
        "description": "",
        "package": server_id,
        "command": "npx",
    }
```

## `get_vectorstore_endpoint(store)`

Get the default endpoint for a vector store.

Parameters:

| Name    | Type  | Description                                           | Default    |
| ------- | ----- | ----------------------------------------------------- | ---------- |
| `store` | `str` | Vector store identifier (e.g., 'chromadb', 'qdrant'). | *required* |

Returns:

| Type  | Description |
| ----- | ----------- |
| \`str | None\`      |

Source code in `src/holodeck/cli/utils/project_init.py`

```
def get_vectorstore_endpoint(store: str) -> str | None:
    """Get the default endpoint for a vector store.

    Args:
        store: Vector store identifier (e.g., 'chromadb', 'qdrant').

    Returns:
        Default endpoint URL or None if not applicable.
    """
    for choice in VECTOR_STORE_CHOICES:
        if choice.value == store:
            return choice.default_endpoint
    return None
```

## `get_provider_api_key_env_var(provider)`

Get the API key environment variable name for an LLM provider.

Parameters:

| Name       | Type  | Description                                               | Default    |
| ---------- | ----- | --------------------------------------------------------- | ---------- |
| `provider` | `str` | LLM provider identifier (e.g., 'openai', 'azure_openai'). | *required* |

Returns:

| Type  | Description |
| ----- | ----------- |
| \`str | None\`      |

Source code in `src/holodeck/cli/utils/project_init.py`

```
def get_provider_api_key_env_var(provider: str) -> str | None:
    """Get the API key environment variable name for an LLM provider.

    Args:
        provider: LLM provider identifier (e.g., 'openai', 'azure_openai').

    Returns:
        Environment variable name for API key, or None if not required.
    """
    for choice in LLM_PROVIDER_CHOICES:
        if choice.value == provider:
            return choice.api_key_env_var
    return None
```

## `get_provider_endpoint_env_var(provider)`

Get the endpoint environment variable name for an LLM provider.

Parameters:

| Name       | Type  | Description                                     | Default    |
| ---------- | ----- | ----------------------------------------------- | ---------- |
| `provider` | `str` | LLM provider identifier (e.g., 'azure_openai'). | *required* |

Returns:

| Type  | Description |
| ----- | ----------- |
| \`str | None\`      |

Source code in `src/holodeck/cli/utils/project_init.py`

```
def get_provider_endpoint_env_var(provider: str) -> str | None:
    """Get the endpoint environment variable name for an LLM provider.

    Args:
        provider: LLM provider identifier (e.g., 'azure_openai').

    Returns:
        Environment variable name for endpoint, or None if not required.
    """
    for choice in LLM_PROVIDER_CHOICES:
        if choice.value == provider:
            return choice.endpoint_env_var
    return None
```

______________________________________________________________________

### Interactive Wizard

Interactive configuration wizard for `holodeck init`.

## `run_wizard(skip_agent_name=False, skip_template=False, skip_llm=False, skip_provider_config=False, skip_vectorstore=False, skip_evals=False, skip_mcp=False, agent_name_default=None, template_default='conversational', llm_default='ollama', provider_config_default=None, vectorstore_default='chromadb', evals_defaults=None, mcp_defaults=None)`

Run interactive configuration wizard.

Prompts user for agent name, template, LLM provider, provider-specific config, vector store, evaluation metrics, and MCP server selections. Skips prompts for values provided via CLI flags (when skip\_\* is True).

Parameters:

| Name                      | Type             | Description                                                 | Default                                              |
| ------------------------- | ---------------- | ----------------------------------------------------------- | ---------------------------------------------------- |
| `skip_agent_name`         | `bool`           | Skip agent name prompt (use agent_name_default).            | `False`                                              |
| `skip_template`           | `bool`           | Skip template prompt (use template_default).                | `False`                                              |
| `skip_llm`                | `bool`           | Skip LLM prompt (use llm_default).                          | `False`                                              |
| `skip_provider_config`    | `bool`           | Skip provider config prompts (use provider_config_default). | `False`                                              |
| `skip_vectorstore`        | `bool`           | Skip vectorstore prompt (use vectorstore_default).          | `False`                                              |
| `skip_evals`              | `bool`           | Skip evals prompt (use evals_defaults).                     | `False`                                              |
| `skip_mcp`                | `bool`           | Skip MCP prompt (use mcp_defaults).                         | `False`                                              |
| `agent_name_default`      | \`str            | None\`                                                      | Default agent name value.                            |
| `template_default`        | `str`            | Default template value (default: "conversational").         | `'conversational'`                                   |
| `llm_default`             | `str`            | Default LLM provider value (default: "ollama").             | `'ollama'`                                           |
| `provider_config_default` | \`ProviderConfig | None\`                                                      | Default provider config (endpoint, deployment name). |
| `vectorstore_default`     | `str`            | Default vector store value (default: "chromadb").           | `'chromadb'`                                         |
| `evals_defaults`          | \`list[str]      | None\`                                                      | Default evaluation metrics list.                     |
| `mcp_defaults`            | \`list[str]      | None\`                                                      | Default MCP server list.                             |

Returns:

| Type           | Description                                 |
| -------------- | ------------------------------------------- |
| `WizardResult` | WizardResult with all validated selections. |

Raises:

| Type                   | Description                                |
| ---------------------- | ------------------------------------------ |
| `WizardCancelledError` | If user cancels with Ctrl+C at any prompt. |

Source code in `src/holodeck/cli/utils/wizard.py`

```
def run_wizard(
    skip_agent_name: bool = False,
    skip_template: bool = False,
    skip_llm: bool = False,
    skip_provider_config: bool = False,
    skip_vectorstore: bool = False,
    skip_evals: bool = False,
    skip_mcp: bool = False,
    agent_name_default: str | None = None,
    template_default: str = "conversational",
    llm_default: str = "ollama",
    provider_config_default: ProviderConfig | None = None,
    vectorstore_default: str = "chromadb",
    evals_defaults: list[str] | None = None,
    mcp_defaults: list[str] | None = None,
) -> WizardResult:
    """Run interactive configuration wizard.

    Prompts user for agent name, template, LLM provider, provider-specific config,
    vector store, evaluation metrics, and MCP server selections. Skips
    prompts for values provided via CLI flags (when skip_* is True).

    Args:
        skip_agent_name: Skip agent name prompt (use agent_name_default).
        skip_template: Skip template prompt (use template_default).
        skip_llm: Skip LLM prompt (use llm_default).
        skip_provider_config: Skip provider config prompts
            (use provider_config_default).
        skip_vectorstore: Skip vectorstore prompt (use vectorstore_default).
        skip_evals: Skip evals prompt (use evals_defaults).
        skip_mcp: Skip MCP prompt (use mcp_defaults).
        agent_name_default: Default agent name value.
        template_default: Default template value (default: "conversational").
        llm_default: Default LLM provider value (default: "ollama").
        provider_config_default: Default provider config (endpoint, deployment name).
        vectorstore_default: Default vector store value (default: "chromadb").
        evals_defaults: Default evaluation metrics list.
        mcp_defaults: Default MCP server list.

    Returns:
        WizardResult with all validated selections.

    Raises:
        WizardCancelledError: If user cancels with Ctrl+C at any prompt.
    """
    try:
        # Step 1: Agent name
        if skip_agent_name and agent_name_default:
            agent_name = agent_name_default
        else:
            agent_name = _prompt_agent_name(default=agent_name_default)

        # Step 2: Template selection
        if skip_template:
            template = template_default
        else:
            template = _prompt_template(default=template_default)

        # Step 3: LLM provider
        if skip_llm:
            llm_provider = llm_default
        else:
            llm_provider = _prompt_llm_provider(default=llm_default)

        # Step 3b: Provider-specific configuration (e.g., Azure endpoint)
        if skip_provider_config:
            provider_config = provider_config_default
        else:
            provider_config = _prompt_provider_config(llm_provider)

        # Step 4: Vector store
        if skip_vectorstore:
            vector_store = vectorstore_default
        else:
            vector_store = _prompt_vectorstore(default=vectorstore_default)

        # Step 5: Evaluation metrics
        if skip_evals:
            evals = (
                evals_defaults if evals_defaults is not None else get_default_evals()
            )
        else:
            evals = _prompt_evals(defaults=evals_defaults)

        # Step 6: MCP servers
        if skip_mcp:
            mcp_servers = (
                mcp_defaults if mcp_defaults is not None else get_default_mcp_servers()
            )
        else:
            mcp_servers = _prompt_mcp_servers(defaults=mcp_defaults)

        # Create and validate result
        return WizardResult(
            agent_name=agent_name,
            template=template,
            llm_provider=llm_provider,
            provider_config=provider_config,
            vector_store=vector_store,
            evals=evals,
            mcp_servers=mcp_servers,
        )

    except KeyboardInterrupt as e:
        raise WizardCancelledError("Wizard cancelled by user") from e
```

## `is_interactive()`

Check if terminal supports interactive prompts.

Checks whether both stdin and stdout are connected to a TTY (terminal). This is used to determine if the wizard can run interactively or should fall back to non-interactive mode.

Returns:

| Type   | Description                                              |
| ------ | -------------------------------------------------------- |
| `bool` | True if stdin and stdout are both TTYs, False otherwise. |

Source code in `src/holodeck/cli/utils/wizard.py`

```
def is_interactive() -> bool:
    """Check if terminal supports interactive prompts.

    Checks whether both stdin and stdout are connected to a TTY
    (terminal). This is used to determine if the wizard can run
    interactively or should fall back to non-interactive mode.

    Returns:
        True if stdin and stdout are both TTYs, False otherwise.
    """
    return sys.stdin.isatty() and sys.stdout.isatty()
```

## `WizardCancelledError`

Bases: `Exception`

Raised when user cancels the wizard (Ctrl+C).

This exception is raised when the user presses Ctrl+C during any interactive prompt in the wizard flow. The caller should handle this exception to clean up any partial state.

______________________________________________________________________

## CLI Exceptions

CLI-specific exception hierarchy. All exceptions inherit from `CLIError`.

## `CLIError`

Bases: `Exception`

Base exception for all CLI errors.

This is the parent class for all exceptions raised by the CLI module. Users can catch this to handle any CLI error generically.

## `ValidationError`

Bases: `CLIError`

Raised when user input validation fails.

This exception is raised when:

- Project name is invalid (special characters, leading digits, etc.)
- Template choice doesn't exist
- Directory permissions are insufficient
- Input constraints are violated

Attributes:

| Name      | Type | Description                           |
| --------- | ---- | ------------------------------------- |
| `message` |      | Description of the validation failure |

## `InitError`

Bases: `CLIError`

Raised when project initialization fails.

This exception is raised when:

- Directory creation fails
- File writing fails
- Cleanup fails after partial creation
- Unexpected errors occur during initialization

Attributes:

| Name      | Type | Description                               |
| --------- | ---- | ----------------------------------------- |
| `message` |      | Description of the initialization failure |

## `TemplateError`

Bases: `CLIError`

Raised when template processing fails.

This exception is raised when:

- Template manifest is malformed or missing
- Jinja2 rendering fails
- Generated YAML doesn't validate against schema
- Template variables are missing or invalid

Attributes:

| Name      | Type | Description                         |
| --------- | ---- | ----------------------------------- |
| `message` |      | Description of the template failure |

## `ChatConfigError(message)`

Bases: `CLIError`

Raised when chat command cannot load agent configuration.

Initialize the error with a human-readable message.

Source code in `src/holodeck/cli/exceptions.py`

```
def __init__(self, message: str) -> None:
    """Initialize the error with a human-readable message."""
    self.message = message
    super().__init__(message)
```

## `ChatAgentInitError(message)`

Bases: `CLIError`

Raised when agent initialization fails for chat sessions.

Initialize the error with a human-readable message.

Source code in `src/holodeck/cli/exceptions.py`

```
def __init__(self, message: str) -> None:
    """Initialize the error with a human-readable message."""
    self.message = message
    super().__init__(message)
```

## `ChatRuntimeError(message, exit_code=None)`

Bases: `CLIError`

Raised for runtime chat failures that should exit the CLI.

Initialize the error with optional exit code override.

Source code in `src/holodeck/cli/exceptions.py`

```
def __init__(self, message: str, exit_code: int | None = None) -> None:
    """Initialize the error with optional exit code override."""
    self.exit_code = exit_code if exit_code is not None else self.exit_code
    self.message = message
    super().__init__(message)
```

## `ChatValidationError(message)`

Bases: `CLIError`

Raised for recoverable user input validation errors during chat.

Initialize the error with a human-readable message.

Source code in `src/holodeck/cli/exceptions.py`

```
def __init__(self, message: str) -> None:
    """Initialize the error with a human-readable message."""
    self.message = message
    super().__init__(message)
```

______________________________________________________________________

## Usage from Python

You can invoke CLI commands programmatically:

```
from holodeck.cli.main import main
from click.testing import CliRunner

runner = CliRunner()

# Initialize a new project
result = runner.invoke(main, ['init', '--template', 'conversational', '--name', 'my-agent'])
print(result.output)

# Run tests
result = runner.invoke(main, ['test', 'path/to/agent.yaml'])
print(result.output)

# Start an interactive chat session
result = runner.invoke(main, ['chat', 'agent.yaml'])
print(result.output)

# Start an HTTP server
result = runner.invoke(main, ['serve', 'agent.yaml', '--port', '9000'])
print(result.output)

# Build a container image
result = runner.invoke(main, ['deploy', 'build', 'agent.yaml', '--dry-run'])
print(result.output)

# Deploy to cloud
result = runner.invoke(main, ['deploy', 'run', 'agent.yaml'])
print(result.output)

# Check deployment status
result = runner.invoke(main, ['deploy', 'status', 'agent.yaml'])
print(result.output)

# Destroy a deployment
result = runner.invoke(main, ['deploy', 'destroy', 'agent.yaml', '--force'])
print(result.output)

# Search MCP registry
result = runner.invoke(main, ['mcp', 'search', 'filesystem'])
print(result.output)

# Add an MCP server
result = runner.invoke(main, ['mcp', 'add', 'io.github.modelcontextprotocol/server-filesystem'])
print(result.output)

# List installed MCP servers
result = runner.invoke(main, ['mcp', 'list'])
print(result.output)

# Remove an MCP server
result = runner.invoke(main, ['mcp', 'remove', 'filesystem'])
print(result.output)

# Initialize configuration
result = runner.invoke(main, ['config', 'init', '-g'])
print(result.output)
```

## CLI Entry Point

The CLI is registered as the `holodeck` command via `pyproject.toml`:

```
[project.scripts]
holodeck = "holodeck.cli.main:main"
```

After installation, use from terminal:

```
# Initialize a new project with interactive wizard
holodeck init

# Quick non-interactive setup
holodeck init --name my-agent --non-interactive

# Run tests (defaults to agent.yaml in current directory)
holodeck test

# Or specify explicit path
holodeck test agent.yaml --verbose --output report.md

# Interactive chat session
holodeck chat agent.yaml

# Start HTTP server with AG-UI protocol
holodeck serve agent.yaml --port 8000 --protocol ag-ui

# Build and deploy containers
holodeck deploy build agent.yaml --tag v1.0.0
holodeck deploy run agent.yaml
holodeck deploy status agent.yaml
holodeck deploy destroy agent.yaml

# MCP server management
holodeck mcp search filesystem
holodeck mcp add io.github.modelcontextprotocol/server-filesystem
holodeck mcp list
holodeck mcp list --all
holodeck mcp remove filesystem

# Configuration management
holodeck config init -g
holodeck config init -p
```

## Related Documentation

- [Template Models](https://docs.useholodeck.ai/api/models/#template-models): Template manifest and metadata models
- [Configuration Loading](https://docs.useholodeck.ai/api/config-loader/index.md): Configuration system
- [Test Runner](https://docs.useholodeck.ai/api/test-runner/index.md): Test execution

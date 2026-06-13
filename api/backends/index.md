# Backend Abstraction Layer

The backend abstraction layer provides a provider-agnostic interface for agent execution. Downstream consumers (test runner, chat session, serve endpoint) depend only on the protocols defined in `holodeck.lib.backends.base` -- no provider-specific types leak through.

## Routing

`BackendSelector` inspects `model.provider` and instantiates the correct backend automatically:

| Provider                 | Backend               |
| ------------------------ | --------------------- |
| `openai`, `azure_openai` | `OpenAIAgentsBackend` |
| `anthropic`, `ollama`    | `ClaudeBackend`       |

______________________________________________________________________

## `holodeck.lib.backends.base` -- Core Protocols & Data Classes

Defines the provider-agnostic contracts that every backend must satisfy and the unified result types returned to callers.

### ExecutionResult

The unified result type returned by every backend. Fields: `response`, `tool_calls`, `tool_results`, `token_usage`, `structured_output`, `num_turns`, `is_error`, `error_reason`, and `thinking` (extended-thinking text, empty when disabled or unsupported by the active backend).

## `ExecutionResult(response, tool_calls=list(), tool_results=list(), token_usage=TokenUsage.zero(), structured_output=None, num_turns=1, is_error=False, error_reason=None, thinking='')`

Provider-agnostic result of a single agent turn.

Attributes:

| Name                | Type                   | Description                                                                                                                                         |
| ------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `response`          | `str`                  | The text response from the agent.                                                                                                                   |
| `tool_calls`        | `list[dict[str, Any]]` | List of tool call records made during execution.                                                                                                    |
| `tool_results`      | `list[dict[str, Any]]` | List of tool result records returned during execution.                                                                                              |
| `token_usage`       | `TokenUsage`           | Token consumption metadata for this turn.                                                                                                           |
| `structured_output` | \`Any                  | None\`                                                                                                                                              |
| `num_turns`         | `int`                  | Number of turns taken to produce this result.                                                                                                       |
| `is_error`          | `bool`                 | Whether the execution ended in an error state.                                                                                                      |
| `error_reason`      | \`str                  | None\`                                                                                                                                              |
| `thinking`          | `str`                  | Extended-thinking text emitted by the model, concatenated in arrival order. Empty when extended thinking is disabled or unsupported by the backend. |

### ToolEvent

## `ToolEvent(kind, tool_name, tool_use_id, tool_input=None, tool_response=None, error=None, parent_tool_use_id=None, text=None)`

Real-time tool execution event from the backend.

Emitted by backends that support hook-based tool observation (e.g. Claude Agent SDK). Events are pushed onto an `asyncio.Queue` that consumers can drain concurrently during agent execution.

Attributes:

| Name                 | Type                                                                              | Description                                                                                                                                                                                                                                                                                                                                                                                                                         |
| -------------------- | --------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `kind`               | `Literal['start', 'end', 'error', 'subagent_message', 'parent_link', 'thinking']` | Event type — "start" before execution, "end" after success, "error" after failure, "subagent_message" for an assistant text snapshot streamed by an in-flight subagent (Task tool), "parent_link" to declare that a tool invocation was spawned by a subagent (so the panel can annotate it), "thinking" for an extended-thinking block streamed mid-turn (one event per ThinkingBlock, ahead of any tool call the block precedes). |
| `tool_name`          | `str`                                                                             | Name of the tool being invoked. Empty for "thinking" (no tool involved).                                                                                                                                                                                                                                                                                                                                                            |
| `tool_use_id`        | `str`                                                                             | Unique identifier correlating start/end/error for the same invocation. For "subagent_message" events this is the parent Task's tool_use_id so consumers can attach the snapshot to the corresponding active entry. For "parent_link" this is the child tool's id. For "thinking" this is a per-block id callers can thread through any downstream protocol (e.g. AG-UI REASONING\_\* message_id).                                   |
| `tool_input`         | \`dict[str, Any]                                                                  | None\`                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `tool_response`      | \`str                                                                             | None\`                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `error`              | \`str                                                                             | None\`                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `parent_tool_use_id` | \`str                                                                             | None\`                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `text`               | \`str                                                                             | None\`                                                                                                                                                                                                                                                                                                                                                                                                                              |

### AgentSession

## `AgentSession`

Bases: `Protocol`

Stateful multi-turn conversation session.

Implementations maintain conversation history across multiple `send` calls. Callers must invoke `close` when the session is no longer needed to release any held resources (connections, subprocesses, etc.).

### `close()`

Release session resources (connections, subprocesses, etc.).

Source code in `src/holodeck/lib/backends/base.py`

```
async def close(self) -> None:
    """Release session resources (connections, subprocesses, etc.)."""
    ...
```

### `prepare()`

Connect or otherwise ready the session before the first `send`.

Called by callers that need the session's underlying transport opened in a specific async task context (e.g. `_TaskBoundSession` requires the SDK's `connect()` to run in the actor task so the anyio task group binds correctly).

Default behavior is a no-op. Implementations that have lazy connect semantics — like `ClaudeSession` — should override this to perform the connect explicitly. Implementations that connect at construction time can leave this as a no-op.

Source code in `src/holodeck/lib/backends/base.py`

```
async def prepare(self) -> None:
    """Connect or otherwise ready the session before the first ``send``.

    Called by callers that need the session's underlying transport
    opened in a specific async task context (e.g.
    ``_TaskBoundSession`` requires the SDK's ``connect()`` to run in
    the actor task so the anyio task group binds correctly).

    Default behavior is a no-op. Implementations that have lazy
    connect semantics — like ``ClaudeSession`` — should override
    this to perform the connect explicitly. Implementations that
    connect at construction time can leave this as a no-op.
    """
    return None
```

### `send(message)`

Send a message and receive a single-turn result.

Parameters:

| Name      | Type  | Description                            | Default    |
| --------- | ----- | -------------------------------------- | ---------- |
| `message` | `str` | The user message to send to the agent. | *required* |

Returns:

| Type              | Description                                                 |
| ----------------- | ----------------------------------------------------------- |
| `ExecutionResult` | ExecutionResult containing the agent response and metadata. |

Source code in `src/holodeck/lib/backends/base.py`

```
async def send(self, message: str) -> ExecutionResult:
    """Send a message and receive a single-turn result.

    Args:
        message: The user message to send to the agent.

    Returns:
        ExecutionResult containing the agent response and metadata.
    """
    ...
```

### `send_streaming(message)`

Send a message and stream the agent response token by token.

Parameters:

| Name      | Type  | Description                            | Default    |
| --------- | ----- | -------------------------------------- | ---------- |
| `message` | `str` | The user message to send to the agent. | *required* |

Yields:

| Type                        | Description                                     |
| --------------------------- | ----------------------------------------------- |
| `AsyncGenerator[str, None]` | Successive string chunks of the agent response. |

Source code in `src/holodeck/lib/backends/base.py`

```
async def send_streaming(self, message: str) -> AsyncGenerator[str, None]:
    """Send a message and stream the agent response token by token.

    Args:
        message: The user message to send to the agent.

    Yields:
        Successive string chunks of the agent response.
    """
    # Protocol stub — concrete implementations use `yield`
    yield ""  # pragma: no cover
```

### AgentBackend

## `AgentBackend`

Bases: `Protocol`

Provider backend factory.

Each backend encapsulates provider-specific initialisation logic and exposes a uniform surface for single-turn invocations (`invoke_once`) and stateful sessions (`create_session`). Callers must call `initialize` before any other method and `teardown` when done.

### `create_session(*, eager_connect=True)`

Create a new stateful multi-turn session.

Parameters:

| Name            | Type   | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Default |
| --------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------- |
| `eager_connect` | `bool` | When True (default), the backend may open its underlying transport before returning. When False, the backend must return a session whose transport is opened lazily — typically by AgentSession.prepare() invoked from a specific async task. Used by callers that need connection lifecycle bound to a different task than the one calling create_session (e.g. \_TaskBoundSession actor). Backends without lazy connect semantics may accept and ignore this argument. | `True`  |

Returns:

| Type           | Description                                          |
| -------------- | ---------------------------------------------------- |
| `AgentSession` | A fresh AgentSession instance bound to this backend. |

Raises:

| Type                  | Description                                        |
| --------------------- | -------------------------------------------------- |
| `BackendInitError`    | If the backend was not initialised before calling. |
| `BackendSessionError` | If the session cannot be created.                  |

Source code in `src/holodeck/lib/backends/base.py`

```
async def create_session(self, *, eager_connect: bool = True) -> AgentSession:
    """Create a new stateful multi-turn session.

    Args:
        eager_connect: When True (default), the backend may open its
            underlying transport before returning. When False, the
            backend must return a session whose transport is opened
            lazily — typically by ``AgentSession.prepare()`` invoked
            from a specific async task. Used by callers that need
            connection lifecycle bound to a different task than the
            one calling ``create_session`` (e.g.
            ``_TaskBoundSession`` actor). Backends without lazy
            connect semantics may accept and ignore this argument.

    Returns:
        A fresh AgentSession instance bound to this backend.

    Raises:
        BackendInitError: If the backend was not initialised before calling.
        BackendSessionError: If the session cannot be created.
    """
    ...
```

### `initialize()`

Prepare the backend for use.

Raises:

| Type               | Description                                                                          |
| ------------------ | ------------------------------------------------------------------------------------ |
| `BackendInitError` | If the backend cannot be initialised (e.g. missing API key, unavailable subprocess). |

Source code in `src/holodeck/lib/backends/base.py`

```
async def initialize(self) -> None:
    """Prepare the backend for use.

    Raises:
        BackendInitError: If the backend cannot be initialised (e.g.
            missing API key, unavailable subprocess).
    """
    ...
```

### `invoke_once(message, context=None)`

Execute a single stateless agent turn.

Parameters:

| Name      | Type                     | Description                            | Default                                    |
| --------- | ------------------------ | -------------------------------------- | ------------------------------------------ |
| `message` | `str`                    | The user message to send to the agent. | *required*                                 |
| `context` | \`list\[dict[str, Any]\] | None\`                                 | Optional list of prior conversation turns. |

Returns:

| Type              | Description                                                 |
| ----------------- | ----------------------------------------------------------- |
| `ExecutionResult` | ExecutionResult containing the agent response and metadata. |

Raises:

| Type                  | Description                                   |
| --------------------- | --------------------------------------------- |
| `BackendSessionError` | If the invocation fails at runtime.           |
| `BackendTimeoutError` | If the invocation exceeds configured timeout. |

Source code in `src/holodeck/lib/backends/base.py`

```
async def invoke_once(
    self,
    message: str,
    context: list[dict[str, Any]] | None = None,
) -> ExecutionResult:
    """Execute a single stateless agent turn.

    Args:
        message: The user message to send to the agent.
        context: Optional list of prior conversation turns.

    Returns:
        ExecutionResult containing the agent response and metadata.

    Raises:
        BackendSessionError: If the invocation fails at runtime.
        BackendTimeoutError: If the invocation exceeds configured timeout.
    """
    ...
```

### `teardown()`

Release all backend resources.

Source code in `src/holodeck/lib/backends/base.py`

```
async def teardown(self) -> None:
    """Release all backend resources."""
    ...
```

### ContextGenerator

## `ContextGenerator`

Bases: `Protocol`

Backend-agnostic contextual embedding generation.

Implementations produce situating context for document chunks by summarising each chunk's role within the larger document. Both the existing Semantic Kernel generator and future Claude SDK generator should satisfy this protocol.

### `contextualize_batch(chunks, document_text, concurrency=None)`

Generate contextual descriptions for a batch of chunks.

Parameters:

| Name            | Type                  | Description                       | Default                                 |
| --------------- | --------------------- | --------------------------------- | --------------------------------------- |
| `chunks`        | `list[DocumentChunk]` | Document chunks to contextualize. | *required*                              |
| `document_text` | `str`                 | Full text of the source document. | *required*                              |
| `concurrency`   | \`int                 | None\`                            | Maximum number of concurrent LLM calls. |

Returns:

| Type        | Description                                              |
| ----------- | -------------------------------------------------------- |
| `list[str]` | A list of contextual description strings, one per chunk. |

Source code in `src/holodeck/lib/backends/base.py`

```
async def contextualize_batch(
    self,
    chunks: list[DocumentChunk],
    document_text: str,
    concurrency: int | None = None,
) -> list[str]:
    """Generate contextual descriptions for a batch of chunks.

    Args:
        chunks: Document chunks to contextualize.
        document_text: Full text of the source document.
        concurrency: Maximum number of concurrent LLM calls.

    Returns:
        A list of contextual description strings, one per chunk.
    """
    ...
```

### Exceptions

## `BackendError`

Bases: `HoloDeckError`

Base exception for all backend errors.

Catch this to handle any backend-related failure without needing to know the specific subtype.

## `BackendInitError`

Bases: `BackendError`

Raised during `initialize()` — startup validation failures.

Examples include a missing API key, an unreachable subprocess, or an incompatible runtime environment.

## `BackendSessionError`

Bases: `BackendError`

Raised during `send()` — session-level failures.

Examples include unexpected disconnections, malformed responses, or provider-reported errors during an active session.

## `BackendTimeoutError`

Bases: `BackendError`

Raised when a single invocation exceeds the configured timeout.

Callers may choose to retry with a longer timeout or surface this as a user-visible error.

______________________________________________________________________

## `holodeck.lib.backends.selector` -- Backend Routing

Routes an `Agent` configuration to the correct backend based on `model.provider`.

### BackendSelector

## `BackendSelector`

Selects and initializes the appropriate backend for an agent configuration.

### `select(agent, tool_instances=None, mode='test')`

Select and initialize the appropriate backend for the given agent.

Parameters:

| Name             | Type             | Description                                          | Default                                        |
| ---------------- | ---------------- | ---------------------------------------------------- | ---------------------------------------------- |
| `agent`          | `Agent`          | Agent configuration with model provider information. | *required*                                     |
| `tool_instances` | \`dict[str, Any] | None\`                                               | Initialized tool instances for Claude backend. |
| `mode`           | `str`            | Execution mode ("test" or "chat").                   | `'test'`                                       |

Returns:

| Type           | Description                                         |
| -------------- | --------------------------------------------------- |
| `AgentBackend` | An initialized AgentBackend instance ready for use. |

Raises:

| Type               | Description                                               |
| ------------------ | --------------------------------------------------------- |
| `BackendInitError` | If the provider is not supported or initialization fails. |

Source code in `src/holodeck/lib/backends/selector.py`

```
@staticmethod
async def select(
    agent: Agent,
    tool_instances: dict[str, Any] | None = None,
    mode: str = "test",
) -> AgentBackend:
    """Select and initialize the appropriate backend for the given agent.

    Args:
        agent: Agent configuration with model provider information.
        tool_instances: Initialized tool instances for Claude backend.
        mode: Execution mode (``"test"`` or ``"chat"``).

    Returns:
        An initialized AgentBackend instance ready for use.

    Raises:
        BackendInitError: If the provider is not supported or initialization fails.
    """
    provider = agent.model.provider

    if provider in (ProviderEnum.OPENAI, ProviderEnum.AZURE_OPENAI):
        # Lazy import keeps the optional openai-agents SDK off the import
        # path for non-OpenAI providers (SC-005).
        from holodeck.lib.backends.openai_agents_backend import OpenAIAgentsBackend

        openai_backend = OpenAIAgentsBackend(agent=agent)
        await openai_backend.initialize()
        return openai_backend

    if provider in (ProviderEnum.ANTHROPIC, ProviderEnum.OLLAMA):
        claude_backend = ClaudeBackend(
            agent=agent,
            tool_instances=tool_instances,
            mode=mode,
        )
        await claude_backend.initialize()
        return claude_backend

    raise BackendInitError(f"Unsupported provider: {provider}")
```

______________________________________________________________________

## `holodeck.lib.backends.openai_agents_backend` -- OpenAI Agents Backend

Implements the backend for `provider: openai` and `provider: azure_openai` natively on the OpenAI Agents SDK, behind the provider-agnostic backend interfaces.

### OpenAIAgentsBackend

## `OpenAIAgentsBackend(agent, base_dir=None)`

OpenAI Agents SDK backend implementing the `AgentBackend` protocol.

Wraps an SDK `Agent` (built from the HoloDeck agent config) and drives it through `Runner.run` for single-turn invocations and `SQLiteSession`- backed multi-turn sessions.

Initialize the backend with agent configuration.

Parameters:

| Name       | Type    | Description                       | Default                                                                                                     |
| ---------- | ------- | --------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `agent`    | `Agent` | The HoloDeck agent configuration. | *required*                                                                                                  |
| `base_dir` | \`Path  | None\`                            | Directory for resolving relative tool/instruction paths. Falls back to the agent_base_dir context variable. |

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
def __init__(self, agent: Agent, base_dir: Path | None = None) -> None:
    """Initialize the backend with agent configuration.

    Args:
        agent: The HoloDeck agent configuration.
        base_dir: Directory for resolving relative tool/instruction paths.
            Falls back to the ``agent_base_dir`` context variable.
    """
    self._agent_config = agent
    self._base_dir = base_dir
    self._sdk_agent: Any | None = None
    self._has_structured_output = False
    self._tool_instances: dict[str, Any] = {}
    self._owned_tools: list[Any] = []
    self._mcp_servers: list[Any] = []
```

### `create_session(*, eager_connect=True)`

Create a stateful multi-turn session backed by a fresh SQLiteSession.

Parameters:

| Name            | Type   | Description                                                                                  | Default |
| --------------- | ------ | -------------------------------------------------------------------------------------------- | ------- |
| `eager_connect` | `bool` | Accepted for protocol compatibility; the SQLite session is created synchronously regardless. | `True`  |

Returns:

| Type           | Description                                               |
| -------------- | --------------------------------------------------------- |
| `AgentSession` | An OpenAIAgentsSession bound to this backend's SDK agent. |

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
async def create_session(self, *, eager_connect: bool = True) -> AgentSession:
    """Create a stateful multi-turn session backed by a fresh SQLiteSession.

    Args:
        eager_connect: Accepted for protocol compatibility; the SQLite
            session is created synchronously regardless.

    Returns:
        An ``OpenAIAgentsSession`` bound to this backend's SDK agent.
    """
    del eager_connect  # no lazy-connect transport for this backend
    sdk_agent = self._require_agent()
    from agents import SQLiteSession

    session_id = f"holodeck-{uuid.uuid4().hex}"
    session = SQLiteSession(session_id)
    return OpenAIAgentsSession(
        sdk_agent=sdk_agent,
        sqlite_session=session,
        agent_config=self._agent_config,
        group_id=session_id,
        max_turns=_max_turns(self._agent_config),
        budget_usd=_max_budget_usd(self._agent_config),
        structured_output=self._has_structured_output,
    )
```

### `initialize()`

Build the SDK `Agent` — validating credentials and tools.

Raises:

| Type               | Description                                                |
| ------------------ | ---------------------------------------------------------- |
| `BackendInitError` | If credentials are missing or the provider is unsupported. |
| `ConfigError`      | If a tool config is unsupported or fails to load.          |

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
async def initialize(self) -> None:
    """Build the SDK ``Agent`` — validating credentials and tools.

    Raises:
        BackendInitError: If credentials are missing or the provider is
            unsupported.
        ConfigError: If a tool config is unsupported or fails to load.
    """
    from agents import Agent as SDKAgent

    from holodeck.lib.backends.openai_agents_output import (
        build_output_schema,
        load_response_format_schema,
    )
    from holodeck.lib.backends.openai_agents_tool_adapters import build_sdk_tools
    from holodeck.lib.backends.validators import validate_openai_agents
    from holodeck.lib.instruction_resolver import resolve_instructions

    # FR-034 / FR-110: fail load on credential gaps and allow ∩ disallow
    # conflicts (all problems surfaced together) before any SDK side effects.
    validate_openai_agents(self._agent_config)

    # Install the OTel-mirroring TracingProcessor once, before any run emits
    # spans (H1). Gated on the agent's observability/tracing being enabled.
    _install_tracing_mirror(self._agent_config)

    base_dir = self._resolve_base_dir()
    disallowed = _disallowed_tool_names(self._agent_config)
    model = _build_model(self._agent_config)
    instructions = resolve_instructions(
        self._agent_config.instructions, base_dir=base_dir
    )
    await self._initialize_tool_instances()
    tools = build_sdk_tools(
        self._agent_config.tools,
        base_dir,
        tool_instances=self._tool_instances,
        disallowed=disallowed,
    )
    mcp_servers = await self._initialize_mcp_servers(base_dir, disallowed)

    model_settings = _build_model_settings(
        self._agent_config.model, self._agent_config.openai
    )

    # FR-004: wire response_format (dict | str path | None) to output_type.
    schema = load_response_format_schema(
        self._agent_config.response_format, base_dir
    )
    output_type = build_output_schema(schema) if schema is not None else None
    self._has_structured_output = output_type is not None

    self._sdk_agent = SDKAgent(
        name=self._agent_config.name,
        instructions=instructions,
        model=model,
        tools=tools,
        mcp_servers=mcp_servers,
        model_settings=model_settings,
        output_type=output_type,
    )
```

### `invoke_once(message, context=None)`

Execute a single stateless agent turn.

Parameters:

| Name      | Type                     | Description                            | Default                                   |
| --------- | ------------------------ | -------------------------------------- | ----------------------------------------- |
| `message` | `str`                    | The user message to send to the agent. | *required*                                |
| `context` | \`list\[dict[str, Any]\] | None\`                                 | Optional prior turns (unused in the MVP). |

Returns:

| Type              | Description                   |
| ----------------- | ----------------------------- |
| `ExecutionResult` | ExecutionResult for the turn. |

Raises:

| Type                  | Description                      |
| --------------------- | -------------------------------- |
| `BackendSessionError` | If the SDK run fails at runtime. |

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
async def invoke_once(
    self,
    message: str,
    context: list[dict[str, Any]] | None = None,
) -> ExecutionResult:
    """Execute a single stateless agent turn.

    Args:
        message: The user message to send to the agent.
        context: Optional prior turns (unused in the MVP).

    Returns:
        ExecutionResult for the turn.

    Raises:
        BackendSessionError: If the SDK run fails at runtime.
    """
    del context  # not threaded into the SDK loop for the MVP
    sdk_agent = self._require_agent()
    from agents import Runner

    try:
        result = await Runner.run(
            sdk_agent,
            message,
            max_turns=_max_turns(self._agent_config),
            run_config=_build_run_config(self._agent_config),
            hooks=self._invoke_hooks(),
        )
    except BackendBudgetExceededError as exc:
        # Surface the budget abort as an error result so the partial response
        # and accumulated cost are preserved (FR-032), not lost to a raise.
        return _budget_error_result(exc)
    except Exception as exc:  # noqa: BLE001 - re-raised as backend error
        raise BackendSessionError(
            f"OpenAI Agents run failed: {type(exc).__name__}: {exc}"
        ) from exc
    return _to_execution_result(result, structured=self._has_structured_output)
```

### `teardown()`

Release backend resources, cleaning up RAG tools and MCP servers.

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
async def teardown(self) -> None:
    """Release backend resources, cleaning up RAG tools and MCP servers."""
    for tool_inst in self._owned_tools:
        cleanup = getattr(tool_inst, "cleanup", None)
        if callable(cleanup):
            try:
                await cleanup()
            except Exception as exc:  # noqa: BLE001 - best-effort cleanup
                logger.warning("Error cleaning up tool: %s", exc)
    self._owned_tools = []
    self._tool_instances = {}

    for server in self._mcp_servers:
        try:
            await server.cleanup()
        except Exception as exc:  # noqa: BLE001 - best-effort cleanup
            logger.warning("Error cleaning up MCP server: %s", exc)
    self._mcp_servers = []
```

### OpenAIAgentsSession

## `OpenAIAgentsSession(sdk_agent, sqlite_session, *, agent_config=None, group_id=None, max_turns=20, budget_usd=None, structured_output=False)`

Stateful multi-turn session backed by an SDK `SQLiteSession`.

Each `send` runs the SDK agent loop with the shared `SQLiteSession` so the SDK persists turn history. Idle sessions are SQLite rows, not held processes.

Bind the session to an SDK agent and its SQLite-backed history.

Parameters:

| Name                | Type    | Description                                                                                           | Default                                                                                                                                                                     |
| ------------------- | ------- | ----------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `sdk_agent`         | `Any`   | The built SDK Agent.                                                                                  | *required*                                                                                                                                                                  |
| `sqlite_session`    | `Any`   | The SDK SQLiteSession persisting turn history.                                                        | *required*                                                                                                                                                                  |
| `agent_config`      | \`Agent | None\`                                                                                                | The HoloDeck agent config used to build the per-run RunConfig. When None no RunConfig is attached.                                                                          |
| `group_id`          | \`str   | None\`                                                                                                | The session id, carried as RunConfig.group_id so the session's turns correlate in the trace.                                                                                |
| `max_turns`         | `int`   | The agent-loop cap passed to Runner.run.                                                              | `20`                                                                                                                                                                        |
| `budget_usd`        | \`float | None\`                                                                                                | The configured max_budget_usd. When set, a single cost accountant is shared across every turn so the budget covers the whole session; when None no cost hooks are attached. |
| `structured_output` | `bool`  | Whether the agent has an output schema, so each turn's final_output is parsed into structured_output. | `False`                                                                                                                                                                     |

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
def __init__(
    self,
    sdk_agent: Any,
    sqlite_session: Any,
    *,
    agent_config: Agent | None = None,
    group_id: str | None = None,
    max_turns: int = 20,
    budget_usd: float | None = None,
    structured_output: bool = False,
) -> None:
    """Bind the session to an SDK agent and its SQLite-backed history.

    Args:
        sdk_agent: The built SDK ``Agent``.
        sqlite_session: The SDK ``SQLiteSession`` persisting turn history.
        agent_config: The HoloDeck agent config used to build the per-run
            ``RunConfig``. When ``None`` no ``RunConfig`` is attached.
        group_id: The session id, carried as ``RunConfig.group_id`` so the
            session's turns correlate in the trace.
        max_turns: The agent-loop cap passed to ``Runner.run``.
        budget_usd: The configured ``max_budget_usd``. When set, a single
            cost accountant is shared across every turn so the budget covers
            the whole session; when ``None`` no cost hooks are attached.
        structured_output: Whether the agent has an output schema, so each
            turn's ``final_output`` is parsed into ``structured_output``.
    """
    self._sdk_agent = sdk_agent
    self._session = sqlite_session
    self._agent_config = agent_config
    self._group_id = group_id
    self._max_turns = max_turns
    self._budget_usd = budget_usd
    self._structured_output = structured_output
    # One accountant shared across the session's turns (FR-032).
    self._accountant: Any | None = None
```

### `close()`

Release the SQLite session connection, if any.

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
async def close(self) -> None:
    """Release the SQLite session connection, if any."""
    close = getattr(self._session, "close", None)
    if callable(close):
        maybe = close()
        if hasattr(maybe, "__await__"):
            await maybe
```

### `prepare()`

No-op. The SQLite session is ready at construction time.

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
async def prepare(self) -> None:
    """No-op. The SQLite session is ready at construction time."""
    return None
```

### `send(message)`

Run one turn against the persistent session.

Parameters:

| Name      | Type  | Description                            | Default    |
| --------- | ----- | -------------------------------------- | ---------- |
| `message` | `str` | The user message to send to the agent. | *required* |

Returns:

| Type              | Description                                                        |
| ----------------- | ------------------------------------------------------------------ |
| `ExecutionResult` | ExecutionResult for this turn. Runtime failures are returned as an |
| `ExecutionResult` | error result (is_error=True) rather than raised, so the multi-turn |
| `ExecutionResult` | executor can record per-turn failures.                             |

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
async def send(self, message: str) -> ExecutionResult:
    """Run one turn against the persistent session.

    Args:
        message: The user message to send to the agent.

    Returns:
        ExecutionResult for this turn. Runtime failures are returned as an
        error result (``is_error=True``) rather than raised, so the multi-turn
        executor can record per-turn failures.
    """
    from agents import Runner

    try:
        result = await Runner.run(
            self._sdk_agent,
            message,
            session=self._session,
            max_turns=self._max_turns,
            run_config=self._run_config(),
            hooks=self._hooks(),
        )
    except BackendBudgetExceededError as exc:
        return _budget_error_result(exc)
    except Exception as exc:  # noqa: BLE001 - surfaced via ExecutionResult
        return ExecutionResult(
            response="",
            is_error=True,
            error_reason=f"{type(exc).__name__}: {exc}",
        )
    return _to_execution_result(result, structured=self._structured_output)
```

### `send_streaming(message)`

Stream the agent response token by token.

Runs the SDK agent loop via `Runner.run_streamed` and forwards each model text delta as it arrives. Text deltas surface as raw-response events carrying a `ResponseTextDeltaEvent`; tool-call and lifecycle events are ignored for the streamed text channel.

Parameters:

| Name      | Type  | Description                            | Default    |
| --------- | ----- | -------------------------------------- | ---------- |
| `message` | `str` | The user message to send to the agent. | *required* |

Yields:

| Type                        | Description                                                     |
| --------------------------- | --------------------------------------------------------------- |
| `AsyncGenerator[str, None]` | String chunks of the agent response as the model produces them. |

Source code in `src/holodeck/lib/backends/openai_agents_backend.py`

```
async def send_streaming(self, message: str) -> AsyncGenerator[str, None]:
    """Stream the agent response token by token.

    Runs the SDK agent loop via ``Runner.run_streamed`` and forwards each
    model text delta as it arrives. Text deltas surface as raw-response
    events carrying a ``ResponseTextDeltaEvent``; tool-call and lifecycle
    events are ignored for the streamed text channel.

    Args:
        message: The user message to send to the agent.

    Yields:
        String chunks of the agent response as the model produces them.
    """
    from agents import Runner
    from openai.types.responses import ResponseTextDeltaEvent

    result = Runner.run_streamed(
        self._sdk_agent,
        message,
        session=self._session,
        max_turns=self._max_turns,
        run_config=self._run_config(),
        hooks=self._hooks(),
    )
    try:
        async for event in result.stream_events():
            if event.type != "raw_response_event":
                continue
            data = event.data
            if isinstance(data, ResponseTextDeltaEvent) and data.delta:
                yield data.delta
    except BackendBudgetExceededError:
        # The budget tripped mid-stream; the deltas produced so far have
        # already been yielded, so end the stream gracefully (FR-032).
        return
```

______________________________________________________________________

## `holodeck.lib.backends.claude_backend` -- Claude Agent SDK Backend

Implements the backend for `provider: anthropic` (and local `provider: ollama` models). Single-turn invocations use the top-level `query()` SDK function; multi-turn chat sessions use `ClaudeSDKClient`.

### ClaudeBackend

## `ClaudeBackend(agent, tool_instances=None, mode='test')`

Backend implementation for the Claude Agent SDK.

Implements the `AgentBackend` protocol. Both single-turn invocations and multi-turn sessions are built on the top-level `query()` function; multi-turn state is threaded via `resume=<sdk_session_id>` inside `ClaudeSession`.

The constructor stores config only — no I/O, no subprocess spawned. Initialization is deferred to `initialize()` (called lazily on first use).

Store configuration without performing any I/O.

Parameters:

| Name             | Type             | Description                        | Default                                                  |
| ---------------- | ---------------- | ---------------------------------- | -------------------------------------------------------- |
| `agent`          | `Agent`          | Agent configuration.               | *required*                                               |
| `tool_instances` | \`dict[str, Any] | None\`                             | Initialized vectorstore/hierarchical-doc tool instances. |
| `mode`           | `str`            | Execution mode ("test" or "chat"). | `'test'`                                                 |

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
def __init__(
    self,
    agent: Agent,
    tool_instances: dict[str, Any] | None = None,
    mode: str = "test",
) -> None:
    """Store configuration without performing any I/O.

    Args:
        agent: Agent configuration.
        tool_instances: Initialized vectorstore/hierarchical-doc tool instances.
        mode: Execution mode (``"test"`` or ``"chat"``).
    """
    self._agent = agent
    self._tool_instances = tool_instances or {}
    self._mode = mode
    self._initialized = False
    self._options: ClaudeAgentOptions | None = None
    self._owned_tools: list[Any] = []  # Tools created during initialize()
    self._instrumentor: Any = None
```

### `create_session(*, eager_connect=True)`

Create a new multi-turn session.

Automatically initializes if not yet done. Under spec 034 P4 the session no longer holds a persistent `ClaudeSDKClient`; each turn opens its own subprocess via `query(resume=session_id)`. `eager_connect` is retained as a no-op for API compatibility — `ClaudeSession.prepare()` is itself a no-op under P4.

Parameters:

| Name            | Type   | Description                                                            | Default |
| --------------- | ------ | ---------------------------------------------------------------------- | ------- |
| `eager_connect` | `bool` | Retained for backwards compatibility; has no effect under spec 034 P4. | `True`  |

Returns:

| Type            | Description                   |
| --------------- | ----------------------------- |
| `ClaudeSession` | A new ClaudeSession instance. |

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def create_session(self, *, eager_connect: bool = True) -> ClaudeSession:
    """Create a new multi-turn session.

    Automatically initializes if not yet done. Under spec 034 P4 the
    session no longer holds a persistent ``ClaudeSDKClient``; each
    turn opens its own subprocess via ``query(resume=session_id)``.
    ``eager_connect`` is retained as a no-op for API compatibility —
    ``ClaudeSession.prepare()`` is itself a no-op under P4.

    Args:
        eager_connect: Retained for backwards compatibility; has no
            effect under spec 034 P4.

    Returns:
        A new ``ClaudeSession`` instance.
    """
    await self._ensure_initialized()
    if self._options is None:
        raise BackendInitError("Backend options not set after initialization")
    session = ClaudeSession(options=self._options)
    if eager_connect:
        await session.prepare()
    return session
```

### `initialize()`

Initialize the backend — validate config, build options.

Idempotent: calling multiple times is a no-op after the first.

Raises:

| Type               | Description                             |
| ------------------ | --------------------------------------- |
| `BackendInitError` | On validation or configuration failure. |

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def initialize(self) -> None:
    """Initialize the backend — validate config, build options.

    Idempotent: calling multiple times is a no-op after the first.

    Raises:
        BackendInitError: On validation or configuration failure.
    """
    if self._initialized:
        return

    try:
        agent = self._agent
        claude = agent.claude

        # 1. Node.js prerequisite
        validate_nodejs(agent)

        # 2. Credentials
        auth_env = validate_credentials(agent.model)

        # 3. Embedding provider (vectorstore tools)
        validate_embedding_provider(agent)

        # 4. Auto-initialize vectorstore/hierarchical-doc tools if needed
        await self._initialize_tools()

        # 5. Tool adapters
        from pathlib import Path

        from holodeck.config.context import agent_base_dir as _agent_base_dir

        _base_dir_str = _agent_base_dir.get()
        _base_dir = Path(_base_dir_str) if _base_dir_str else None
        adapters = create_tool_adapters(
            tool_configs=agent.tools or [],
            tool_instances=self._tool_instances,
            base_dir=_base_dir,
        )
        tool_server, tool_names = build_holodeck_sdk_server(adapters)

        # 6. External MCP configs
        mcp_tools = [t for t in (agent.tools or []) if isinstance(t, MCPTool)]
        mcp_configs = build_claude_mcp_configs(mcp_tools)

        # 7. OTel env vars
        otel_env: dict[str, str] = {}
        if agent.observability:
            otel_env = translate_observability(agent.observability)

        # 8. Build options
        self._options = build_options(
            agent=agent,
            tool_server=tool_server if adapters else None,
            tool_names=tool_names,
            mcp_configs=mcp_configs,
            auth_env=auth_env,
            otel_env=otel_env,
            mode=self._mode,
        )

        # 9. Working directory collision check
        wd = claude.working_directory if claude else None
        validate_working_directory(wd)

        # 10. Response format validation
        validate_response_format(agent.response_format)

        # 11. GenAI instrumentation (optional, non-blocking)
        self._activate_instrumentation()

        self._initialized = True

    except Exception as exc:
        if not isinstance(exc, BackendInitError):
            raise BackendInitError(
                f"Claude backend initialization failed: {exc}"
            ) from exc
        raise
```

### `invoke_once(message, context=None)`

Invoke the agent for a single turn.

Automatically initializes if not yet done. Retries on `ProcessError` (subprocess crash) with exponential backoff.

Parameters:

| Name      | Type                     | Description        | Default                                                    |
| --------- | ------------------------ | ------------------ | ---------------------------------------------------------- |
| `message` | `str`                    | User message text. | *required*                                                 |
| `context` | \`list\[dict[str, Any]\] | None\`             | Optional conversation context (unused for Claude backend). |

Returns:

| Type              | Description                                |
| ----------------- | ------------------------------------------ |
| `ExecutionResult` | ExecutionResult with the agent's response. |

Raises:

| Type                  | Description                  |
| --------------------- | ---------------------------- |
| `BackendSessionError` | After max retries exhausted. |

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def invoke_once(
    self, message: str, context: list[dict[str, Any]] | None = None
) -> ExecutionResult:
    """Invoke the agent for a single turn.

    Automatically initializes if not yet done. Retries on ``ProcessError``
    (subprocess crash) with exponential backoff.

    Args:
        message: User message text.
        context: Optional conversation context (unused for Claude backend).

    Returns:
        ``ExecutionResult`` with the agent's response.

    Raises:
        BackendSessionError: After max retries exhausted.
    """
    await self._ensure_initialized()

    last_error: BaseException | None = None
    for attempt in range(_MAX_RETRIES):
        try:
            return await self._invoke_query(message)
        except (ProcessError, CLIConnectionError, MessageParseError) as exc:
            last_error = exc
        except BaseExceptionGroup as exc:
            # anyio TaskGroup wraps subprocess errors in ExceptionGroup
            last_error = exc

        if attempt < _MAX_RETRIES - 1:
            backoff = _BACKOFF_BASE_SECONDS * (2**attempt)
            logger.warning(
                "Claude subprocess error (attempt %d/%d), retrying in %ds: %s",
                attempt + 1,
                _MAX_RETRIES,
                backoff,
                last_error,
            )
            await asyncio.sleep(backoff)

    raise BackendSessionError(
        f"Claude subprocess failed after {_MAX_RETRIES} retries: {last_error}"
    )
```

### `teardown()`

Reset backend state, releasing any built options.

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def teardown(self) -> None:
    """Reset backend state, releasing any built options."""
    # Deactivate GenAI instrumentation
    if self._instrumentor is not None:
        try:
            self._instrumentor.uninstrument()
        except Exception as exc:
            logger.warning("Error deactivating GenAI instrumentation: %s", exc)
        self._instrumentor = None

    # Cleanup owned tools
    for tool_inst in self._owned_tools:
        if hasattr(tool_inst, "cleanup"):
            try:
                await tool_inst.cleanup()
            except Exception as exc:
                logger.warning("Error cleaning up tool: %s", exc)
    self._owned_tools = []
    self._tool_instances = {}
    self._initialized = False
    self._options = None
```

### ClaudeSession

## `ClaudeSession(options)`

Stateful multi-turn session backed by `query(resume=...)` (spec 034 P4).

Each `send()` / `send_streaming()` opens a fresh CLI subprocess via the top-level `query()` function. Turn 1 has no `resume`; the CLI assigns a session id which is captured from `ResultMessage` and stored on `_sdk_session_id`. Subsequent turns pass that id via `options.resume` so the CLI rehydrates the JSONL transcript at `~/.claude/projects/<encoded-cwd>/<sdk_session_id>.jsonl`.

The `_base_options` reference is **never mutated**. Turn-specific options are created as new `ClaudeAgentOptions` instances.

Initialize session with base options.

Parameters:

| Name      | Type                 | Description                                                  | Default    |
| --------- | -------------------- | ------------------------------------------------------------ | ---------- |
| `options` | `ClaudeAgentOptions` | Base options (immutable reference for the session lifetime). | *required* |

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
def __init__(self, options: ClaudeAgentOptions) -> None:
    """Initialize session with base options.

    Args:
        options: Base options (immutable reference for the session lifetime).
    """
    self._base_options = options
    self._turn_count: int = 0
    self._tool_event_queue: asyncio.Queue[ToolEvent] = asyncio.Queue()
    # spec 034 P4 — hybrid-session state.
    # CLI-assigned conversation id; captured from ResultMessage on turn 1
    # and fed into ``options.resume`` on turn 2+ so each fresh subprocess
    # rehydrates the JSONL transcript at
    # ``~/.claude/projects/<encoded-cwd>/<sdk_session_id>.jsonl``.
    self._sdk_session_id: str | None = None
    # Serialises concurrent send() / send_streaming() on the same session.
    # Two concurrent turns with the same resume= would race the transcript.
    self._send_lock: asyncio.Lock = asyncio.Lock()
```

### `tool_events`

Queue of real-time tool events emitted via SDK hooks.

### `close()`

Delete the on-disk JSONL transcript and clear session state.

Under spec 034 P4 the session has no persistent subprocess to disconnect. Conversation state lives on disk at `~/.claude/projects/<encoded-cwd>/<sdk_session_id>.jsonl`. Closing the session permanently discards that transcript so the next open of the same threadId starts fresh.

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def close(self) -> None:
    """Delete the on-disk JSONL transcript and clear session state.

    Under spec 034 P4 the session has no persistent subprocess to
    disconnect. Conversation state lives on disk at
    ``~/.claude/projects/<encoded-cwd>/<sdk_session_id>.jsonl``. Closing
    the session permanently discards that transcript so the next
    open of the same threadId starts fresh.
    """
    if self._sdk_session_id is not None:
        cwd_value = getattr(self._base_options, "cwd", None)
        path = _transcript_path(self._sdk_session_id, cwd=cwd_value)
        try:
            path.unlink()
        except FileNotFoundError:
            pass
        except OSError as exc:
            logger.warning("Failed to delete transcript %s: %s", path, exc)
        self._sdk_session_id = None
```

### `prepare()`

No-op under spec 034 P4.

Retained for backwards compatibility with the chat executor's `_TaskBoundSession`. Under the hybrid-session model the SDK's anyio task group is created inside each `query()` call frame, so there is no task-binding to do up front.

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def prepare(self) -> None:
    """No-op under spec 034 P4.

    Retained for backwards compatibility with the chat executor's
    ``_TaskBoundSession``. Under the hybrid-session model the SDK's
    anyio task group is created inside each ``query()`` call frame,
    so there is no task-binding to do up front.
    """
    return None
```

### `release_transport()`

No-op under spec 034 P4.

Retained for backwards compatibility with the chat executor's `_TaskBoundSession`. Under the hybrid-session model each turn's subprocess is created and torn down inside `query()`; there is no persistent transport to release between turns.

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def release_transport(self) -> None:
    """No-op under spec 034 P4.

    Retained for backwards compatibility with the chat executor's
    ``_TaskBoundSession``. Under the hybrid-session model each turn's
    subprocess is created and torn down inside ``query()``; there is no
    persistent transport to release between turns.
    """
    return None
```

### `send(message)`

Send a message and collect the full response.

Parameters:

| Name      | Type  | Description        | Default    |
| --------- | ----- | ------------------ | ---------- |
| `message` | `str` | User message text. | *required* |

Returns:

| Type              | Description                                |
| ----------------- | ------------------------------------------ |
| `ExecutionResult` | ExecutionResult with the agent's response. |

Raises:

| Type                  | Description                 |
| --------------------- | --------------------------- |
| `BackendSessionError` | On subprocess or SDK error. |

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def send(self, message: str) -> ExecutionResult:
    """Send a message and collect the full response.

    Args:
        message: User message text.

    Returns:
        ``ExecutionResult`` with the agent's response.

    Raises:
        BackendSessionError: On subprocess or SDK error.
    """
    async with self._send_lock:
        turn_no = self._turn_count + 1
        logger.debug(
            "[trace] ClaudeSession.send turn=%d: entering, resume=%s",
            turn_no,
            self._sdk_session_id,
        )
        try:
            options = self._options_with_hooks()
            if self._sdk_session_id is not None:
                import dataclasses

                try:
                    options = dataclasses.replace(
                        options, resume=self._sdk_session_id
                    )
                except TypeError:
                    # Fallback for non-dataclass options (test mocks).
                    options.resume = self._sdk_session_id

            text_parts: list[str] = []
            tool_calls: list[dict[str, Any]] = []
            tool_results: list[dict[str, Any]] = []
            thinking_parts: list[str] = []
            token_usage = TokenUsage.zero()
            num_turns = 1
            structured_output: Any = None

            msg_count = 0
            async for msg in claude_agent_sdk.query(
                prompt=_streaming_user_envelope(message), options=options
            ):
                msg_count += 1
                logger.debug(
                    "[trace] ClaudeSession.send turn=%d: msg #%d type=%s",
                    turn_no,
                    msg_count,
                    msg.__class__.__name__,
                )
                _maybe_emit_thinking_blocks(msg, self._tool_event_queue)
                _maybe_emit_subagent_message(msg, self._tool_event_queue)
                text_parts, tool_calls, tool_results, thinking_parts = (
                    _process_message(
                        msg,
                        text_parts,
                        tool_calls,
                        tool_results,
                        thinking_parts,
                    )
                )
                if msg.__class__.__name__ == "ResultMessage":
                    rm = cast(Any, msg)
                    usage = rm.usage or {}
                    prompt_tokens = usage.get("input_tokens", 0)
                    completion = usage.get("output_tokens", 0)
                    token_usage = TokenUsage(
                        prompt_tokens=prompt_tokens,
                        completion_tokens=completion,
                        total_tokens=prompt_tokens + completion,
                    )
                    num_turns = rm.num_turns
                    structured_output = rm.structured_output
                    if self._sdk_session_id is None:
                        captured = getattr(rm, "session_id", None)
                        if isinstance(captured, str) and captured:
                            self._sdk_session_id = captured

            logger.debug(
                "[trace] ClaudeSession.send turn=%d: exited, "
                "msg_count=%d, num_turns=%d, sdk_session_id=%s",
                turn_no,
                msg_count,
                num_turns,
                self._sdk_session_id,
            )
            self._turn_count += 1

            _enrich_tool_results(tool_calls, tool_results)

            response_text = "".join(text_parts)
            if structured_output is not None:
                response_text = json.dumps(structured_output)

            return ExecutionResult(
                response=response_text,
                tool_calls=tool_calls,
                tool_results=tool_results,
                token_usage=token_usage,
                structured_output=structured_output,
                num_turns=num_turns,
                thinking="".join(thinking_parts),
            )
        except (ProcessError, CLIConnectionError) as exc:
            raise BackendSessionError(
                f"subprocess terminated unexpectedly: {exc}"
            ) from exc
```

### `send_agui(input_data, message_override=None)`

Send an AG-UI request through this Claude session.

Yields AG-UI events translated directly from Claude SDK stream messages.

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def send_agui(
    self,
    input_data: RunAgentInput,
    message_override: str | None = None,
) -> AsyncGenerator[BaseEvent, None]:
    """Send an AG-UI request through this Claude session.

    Yields AG-UI events translated directly from Claude SDK stream messages.
    """
    async with self._send_lock:
        thread_id = input_data.thread_id
        run_id = input_data.run_id
        options = self._options_for_agui(input_data)
        prompt = _agui_prompt_from_input(input_data, message_override)
        frontend_tool_names = set(_agui_tool_names(input_data.tools))
        current_state = input_data.state
        result_data: dict[str, Any] | None = None

        yield RunStartedEvent(
            type=EventType.RUN_STARTED,
            thread_id=thread_id,
            run_id=run_id,
            parent_run_id=input_data.parent_run_id,
            input=input_data,
        )
        if input_data.state is not None:
            yield StateSnapshotEvent(
                type=EventType.STATE_SNAPSHOT,
                snapshot=input_data.state,
            )

        try:
            message_stream = claude_agent_sdk.query(
                prompt=_streaming_user_envelope(prompt),
                options=options,
            )
            async for event in self._stream_agui_messages(
                message_stream=message_stream,
                thread_id=thread_id,
                run_id=run_id,
                input_data=input_data,
                frontend_tool_names=frontend_tool_names,
                current_state=current_state,
            ):
                if isinstance(event, _AguiStreamStateUpdate):
                    current_state = event.state
                    continue
                if isinstance(event, _AguiStreamResult):
                    result_data = event.result
                    continue
                yield event

            self._turn_count += 1
            yield RunFinishedEvent(
                type=EventType.RUN_FINISHED,
                thread_id=thread_id,
                run_id=run_id,
                result=result_data,
            )
        except (ProcessError, CLIConnectionError) as exc:
            raise BackendSessionError(
                f"subprocess terminated unexpectedly: {exc}"
            ) from exc
```

### `send_streaming(message)`

Send a message and yield text chunks progressively.

Parameters:

| Name      | Type  | Description        | Default    |
| --------- | ----- | ------------------ | ---------- |
| `message` | `str` | User message text. | *required* |

Yields:

| Type                        | Description                              |
| --------------------------- | ---------------------------------------- |
| `AsyncGenerator[str, None]` | Text chunks as they arrive from the SDK. |

Raises:

| Type                  | Description                 |
| --------------------- | --------------------------- |
| `BackendSessionError` | On subprocess or SDK error. |

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
async def send_streaming(self, message: str) -> AsyncGenerator[str, None]:
    """Send a message and yield text chunks progressively.

    Args:
        message: User message text.

    Yields:
        Text chunks as they arrive from the SDK.

    Raises:
        BackendSessionError: On subprocess or SDK error.
    """
    async with self._send_lock:
        options = self._options_with_hooks()
        if self._sdk_session_id is not None:
            import dataclasses

            try:
                options = dataclasses.replace(options, resume=self._sdk_session_id)
            except TypeError:
                # Fallback for non-dataclass options (test mocks).
                options.resume = self._sdk_session_id

        try:
            async for msg in claude_agent_sdk.query(
                prompt=_streaming_user_envelope(message), options=options
            ):
                _maybe_emit_thinking_blocks(msg, self._tool_event_queue)
                _maybe_emit_subagent_message(msg, self._tool_event_queue)
                if msg.__class__.__name__ == "AssistantMessage":
                    for block in cast(Any, msg).content:
                        if block.__class__.__name__ == "TextBlock" and block.text:
                            yield block.text
                elif msg.__class__.__name__ == "ResultMessage":
                    self._turn_count += 1
                    if self._sdk_session_id is None:
                        captured = getattr(msg, "session_id", None)
                        if isinstance(captured, str) and captured:
                            self._sdk_session_id = captured
        except (ProcessError, CLIConnectionError) as exc:
            raise BackendSessionError(
                f"subprocess terminated unexpectedly: {exc}"
            ) from exc
```

### build_options

## `build_options(*, agent, tool_server, tool_names, mcp_configs, auth_env, otel_env, mode)`

Assemble `ClaudeAgentOptions` from agent config and bridge outputs.

Parameters:

| Name          | Type                 | Description                                                  | Default                                                       |
| ------------- | -------------------- | ------------------------------------------------------------ | ------------------------------------------------------------- |
| `agent`       | `Agent`              | The agent configuration.                                     | *required*                                                    |
| `tool_server` | \`McpSdkServerConfig | None\`                                                       | In-process MCP server for vectorstore/hierarchical-doc tools. |
| `tool_names`  | `list[str]`          | Allowed tool names from the in-process server.               | *required*                                                    |
| `mcp_configs` | `dict[str, Any]`     | External MCP server configs from build_claude_mcp_configs(). | *required*                                                    |
| `auth_env`    | `dict[str, str]`     | Auth env vars from validate_credentials().                   | *required*                                                    |
| `otel_env`    | `dict[str, str]`     | OTel env vars from translate_observability().                | *required*                                                    |
| `mode`        | `str`                | Execution mode ("test" or "chat").                           | *required*                                                    |

Returns:

| Type                 | Description                                      |
| -------------------- | ------------------------------------------------ |
| `ClaudeAgentOptions` | Configured ClaudeAgentOptions ready for query(). |

Source code in `src/holodeck/lib/backends/claude_backend.py`

```
def build_options(
    *,
    agent: Agent,
    tool_server: McpSdkServerConfig | None,
    tool_names: list[str],
    mcp_configs: dict[str, Any],
    auth_env: dict[str, str],
    otel_env: dict[str, str],
    mode: str,
) -> ClaudeAgentOptions:
    """Assemble ``ClaudeAgentOptions`` from agent config and bridge outputs.

    Args:
        agent: The agent configuration.
        tool_server: In-process MCP server for vectorstore/hierarchical-doc tools.
        tool_names: Allowed tool names from the in-process server.
        mcp_configs: External MCP server configs from ``build_claude_mcp_configs()``.
        auth_env: Auth env vars from ``validate_credentials()``.
        otel_env: OTel env vars from ``translate_observability()``.
        mode: Execution mode (``"test"`` or ``"chat"``).

    Returns:
        Configured ``ClaudeAgentOptions`` ready for ``query()``.
    """
    claude = agent.claude
    system_prompt = resolve_instructions(agent.instructions)

    # MCP servers
    mcp_servers: dict[str, Any] = dict(mcp_configs)
    if tool_server is not None:
        mcp_servers["holodeck_tools"] = tool_server

    # Env vars — unset CLAUDECODE to prevent the "nested session" guard
    # when HoloDeck runs inside a terminal with Claude Code active.
    # SDK merges: {**os.environ, **options.env}, so "" overrides "1".
    env: dict[str, str] = {"CLAUDECODE": "", **auth_env, **otel_env}

    # Custom base URL — when a custom auth provider is used with an explicit
    # endpoint, forward it as ANTHROPIC_BASE_URL so the SDK subprocess targets
    # the third-party endpoint.  Only inject for auth_provider=custom to avoid
    # accidentally overriding the default Anthropic URL for proxy/rate-limit
    # setups that use endpoint with a standard auth provider.
    if agent.model.endpoint and agent.model.auth_provider == AuthProvider.custom:
        env["ANTHROPIC_BASE_URL"] = agent.model.endpoint

    # Permission mode (raises ConfigError for acceptAll without unsafe opt-in)
    perm_mode = _build_permission_mode(claude, mode)

    # Extended thinking
    max_thinking_tokens = None
    if claude and claude.extended_thinking and claude.extended_thinking.enabled:
        max_thinking_tokens = claude.extended_thinking.budget_tokens

    # Allowed tools
    allowed_tools: list[str] = list(tool_names)
    if claude and claude.allowed_tools:
        allowed_tools.extend(claude.allowed_tools)

    # Built-in capabilities
    if claude and claude.web_search:
        allowed_tools.append("WebSearch")

    # Output format
    output_format = _build_output_format(agent.response_format)

    # Working directory — fall back to agent.yaml's directory so that
    # relative paths in MCP args (e.g. "./data") resolve correctly.
    from holodeck.config.context import agent_base_dir

    cwd = claude.working_directory if claude else None
    if cwd is None:
        cwd = agent_base_dir.get()

    # Max turns — default to _DEFAULT_MAX_TURNS when unset (spec 034 P1a)
    max_turns = (
        claude.max_turns
        if claude and claude.max_turns is not None
        else _DEFAULT_MAX_TURNS
    )

    # Disallowed tools (P1b — spec 034). Auto-disallow risky built-in SDK
    # tools (Bash, Write, Edit, WebFetch) that the operator has not declared
    # via the dedicated schema fields or claude.allowed_tools. This is the
    # real fail-closed lever — permission_mode is only a tiebreaker for tools
    # not covered by either list.
    declared = _declared_builtin_tools(claude)
    auto_disallow = sorted(_RISKY_BUILTIN_TOOLS - declared)
    explicit_disallow = (
        list(claude.disallowed_tools) if claude and claude.disallowed_tools else []
    )
    disallowed_tools: list[str] | None = (
        sorted(set(explicit_disallow) | set(auto_disallow))
        if (explicit_disallow or auto_disallow)
        else None
    )
    if auto_disallow:
        logger.info(
            "Auto-disallowed risky built-in SDK tools not declared in agent "
            "config: %s. Declare them via claude.bash.enabled, "
            "claude.file_system.*, or claude.allowed_tools to opt in.",
            ", ".join(auto_disallow),
        )

    # Default [] → CLI subprocess never inherits ~/.claude plugins/skills/hooks.
    setting_sources: list[str] = (
        list(claude.setting_sources)
        if claude is not None and claude.setting_sources is not None
        else []
    )

    # Build the options dict
    opts_kwargs: dict[str, Any] = {
        "model": agent.model.name,
        "system_prompt": system_prompt,
        "permission_mode": perm_mode,
        "max_turns": max_turns,
        "mcp_servers": mcp_servers,
        "allowed_tools": allowed_tools,
        "env": env,
        "cwd": cwd,
        "output_format": output_format,
        "setting_sources": setting_sources,
    }
    if disallowed_tools is not None:
        opts_kwargs["disallowed_tools"] = disallowed_tools

    if max_thinking_tokens is not None:
        opts_kwargs["max_thinking_tokens"] = max_thinking_tokens

    if claude is not None:
        if claude.effort is not None:
            opts_kwargs["effort"] = claude.effort
        if claude.max_budget_usd is not None:
            opts_kwargs["max_budget_usd"] = claude.max_budget_usd
        if claude.fallback_model is not None:
            opts_kwargs["fallback_model"] = claude.fallback_model
        if claude.agents:
            opts_kwargs["agents"] = {
                name: AgentDefinition(
                    description=spec.description,
                    prompt=spec.prompt or "",
                    tools=spec.tools,
                    model=spec.model,
                )
                for name, spec in claude.agents.items()
            }

    # Spec 034 P2b — merge HoloDeck default hooks (credential-redaction
    # PostToolUse) ahead of any user-supplied hooks. Opt-out by setting
    # `claude.disable_default_hooks: true` in agent.yaml. Bash hardening
    # is handled by SDK permission rules + P1b auto-disallow.
    from holodeck.lib.backends.claude_hooks import build_default_hooks

    user_hooks: dict[Any, list[Any]] = opts_kwargs.get("hooks") or {}
    if claude is not None and claude.disable_default_hooks:
        logger.warning(
            "HoloDeck default hooks DISABLED for agent '%s' (credential "
            "redaction PostToolUse hook will NOT run). OTel attribute "
            "redaction is unaffected. To re-enable, remove "
            "`claude.disable_default_hooks: true` from agent.yaml.",
            agent.name,
        )
        if user_hooks:
            opts_kwargs["hooks"] = user_hooks
    else:
        default_hooks = build_default_hooks()
        merged: dict[Any, list[Any]] = {}
        # Default hooks first so they short-circuit when they deny.
        for event, matchers in default_hooks.items():
            merged[event] = list(matchers)
        for event, matchers in user_hooks.items():
            if event in merged:
                merged[event] = merged[event] + list(matchers)
            else:
                merged[event] = list(matchers)
        if merged:
            opts_kwargs["hooks"] = merged

    # Spec 034 P2b — default-on subprocess env scrubbing. These two flags
    # tell the Claude CLI to strip Anthropic/cloud creds from tool
    # subprocesses (Bash, etc.) and to spawn stdio MCP servers with only
    # their declared env (not the full inherited shell env). They don't
    # affect the SDK subprocess's own env (which inherits everything from
    # this serve process — see P3 for the structural fix). Opt out via
    # `claude.disable_subprocess_env_scrub: true` in agent.yaml.
    env_overrides = dict(opts_kwargs.get("env") or {})
    if claude is not None and claude.disable_subprocess_env_scrub:
        logger.warning(
            "Subprocess env scrubbing DISABLED for agent '%s' "
            "(tool subprocesses and stdio MCP servers will inherit the "
            "full agent container env including credentials). To re-enable, "
            "remove `claude.disable_subprocess_env_scrub: true` from "
            "agent.yaml.",
            agent.name,
        )
    else:
        env_overrides.setdefault("CLAUDE_CODE_SUBPROCESS_ENV_SCRUB", "1")
        # Note: this scrubs the SDK-spawned subprocess env for stdio MCP servers.
        # Operators must declare any inherited env vars (HOME, PATH, provider creds)
        # on the MCP tool's `env` block — see docs/security/prompt-injection-defenses.md
        # §"Operator footgun".
        env_overrides.setdefault("CLAUDE_CODE_MCP_ALLOWLIST_ENV", "1")
    if env_overrides:
        opts_kwargs["env"] = env_overrides

    return ClaudeAgentOptions(**opts_kwargs)
```

______________________________________________________________________

## `holodeck.lib.backends.tool_adapters` -- Claude SDK Tool Adapters

Wraps HoloDeck vectorstore and hierarchical-document tools as `@tool`-decorated functions, bundles them into an in-process MCP server, and provides a factory for `ClaudeBackend` to call during initialization.

### VectorStoreToolAdapter

## `VectorStoreToolAdapter(config, instance)`

Wraps a `VectorStoreTool` for use with the Claude Agent SDK.

Parameters:

| Name       | Type              | Description                                             | Default    |
| ---------- | ----------------- | ------------------------------------------------------- | ---------- |
| `config`   | `VectorstoreTool` | The vectorstore tool configuration from the agent YAML. | *required* |
| `instance` | `VectorStoreTool` | An initialized VectorStoreTool instance.                | *required* |

Source code in `src/holodeck/lib/backends/tool_adapters.py`

```
def __init__(
    self,
    config: VectorstoreToolConfig,
    instance: VectorStoreTool,
) -> None:
    self.config = config
    self.instance = instance
```

### `to_sdk_tool()`

Return an `SdkMcpTool` backed by this adapter's search method.

Source code in `src/holodeck/lib/backends/tool_adapters.py`

```
def to_sdk_tool(self) -> SdkMcpTool[Any]:
    """Return an ``SdkMcpTool`` backed by this adapter's search method."""
    name = f"{self.config.name}_search"
    desc = _truncate_description(
        f"Search {self.config.name}: {self.config.description}"
    )
    return _make_vectorstore_search_fn(self.instance, name, desc)
```

### HierarchicalDocToolAdapter

## `HierarchicalDocToolAdapter(config, instance)`

Wraps a `HierarchicalDocumentTool` for use with the Claude Agent SDK.

Parameters:

| Name       | Type                             | Description                                                       | Default    |
| ---------- | -------------------------------- | ----------------------------------------------------------------- | ---------- |
| `config`   | `HierarchicalDocumentToolConfig` | The hierarchical document tool configuration from the agent YAML. | *required* |
| `instance` | `HierarchicalDocumentTool`       | An initialized HierarchicalDocumentTool instance.                 | *required* |

Source code in `src/holodeck/lib/backends/tool_adapters.py`

```
def __init__(
    self,
    config: HierarchicalDocumentToolConfig,
    instance: HierarchicalDocumentTool,
) -> None:
    self.config = config
    self.instance = instance
```

### `to_sdk_tool()`

Return an `SdkMcpTool` backed by this adapter's search method.

Source code in `src/holodeck/lib/backends/tool_adapters.py`

```
def to_sdk_tool(self) -> SdkMcpTool[Any]:
    """Return an ``SdkMcpTool`` backed by this adapter's search method."""
    name = f"{self.config.name}_search"
    desc = _truncate_description(
        f"Search {self.config.name}: {self.config.description}"
    )
    return _make_hierarchical_search_fn(self.instance, name, desc)
```

### create_tool_adapters

## `create_tool_adapters(tool_configs, tool_instances, base_dir=None)`

Build adapters for vectorstore, hierarchical-document, and function tools.

Filters *tool_configs* for supported types, matches each to its initialized instance (vectorstore / hierarchical) or loads the Python callable (function tools) and returns adapter objects.

Parameters:

| Name             | Type                         | Description                                  | Default                                                                                                                                               |
| ---------------- | ---------------------------- | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `tool_configs`   | `list[ToolUnion]`            | All tool configurations from the agent YAML. | *required*                                                                                                                                            |
| `tool_instances` | \`dict\[str, VectorStoreTool | HierarchicalDocumentTool\]\`                 | Initialized tool instances keyed by config name.                                                                                                      |
| `base_dir`       | \`Path                       | None\`                                       | Directory used to resolve relative FunctionTool.file paths. Typically the agent project root. Unused by vectorstore / hierarchical-document adapters. |

Returns:

| Type                           | Description                |
| ------------------------------ | -------------------------- |
| \`list\[VectorStoreToolAdapter | HierarchicalDocToolAdapter |

Raises:

| Type               | Description                                          |
| ------------------ | ---------------------------------------------------- |
| `BackendInitError` | If a supported tool config has no matching instance. |
| `ConfigError`      | If a function tool fails to load.                    |

Source code in `src/holodeck/lib/backends/tool_adapters.py`

```
def create_tool_adapters(
    tool_configs: list[ToolUnion],
    tool_instances: dict[str, VectorStoreTool | HierarchicalDocumentTool],
    base_dir: Path | None = None,
) -> list[VectorStoreToolAdapter | HierarchicalDocToolAdapter | FunctionToolAdapter]:
    """Build adapters for vectorstore, hierarchical-document, and function tools.

    Filters *tool_configs* for supported types, matches each to its
    initialized instance (vectorstore / hierarchical) or loads the Python
    callable (function tools) and returns adapter objects.

    Args:
        tool_configs: All tool configurations from the agent YAML.
        tool_instances: Initialized tool instances keyed by config name.
        base_dir: Directory used to resolve relative ``FunctionTool.file``
            paths. Typically the agent project root. Unused by vectorstore /
            hierarchical-document adapters.

    Returns:
        List of adapter objects ready for ``build_holodeck_sdk_server()``.

    Raises:
        BackendInitError: If a supported tool config has no matching instance.
        ConfigError: If a function tool fails to load.
    """
    adapters: list[
        VectorStoreToolAdapter | HierarchicalDocToolAdapter | FunctionToolAdapter
    ] = []

    for cfg in tool_configs:
        if isinstance(cfg, VectorstoreToolConfig):
            instance = tool_instances.get(cfg.name)
            if instance is None:
                raise BackendInitError(
                    f"No initialized instance found for tool '{cfg.name}' "
                    f"(type: {cfg.type}). Ensure tool initialization "
                    "completed before creating adapters."
                )
            adapters.append(
                VectorStoreToolAdapter(
                    config=cfg,
                    instance=instance,  # type: ignore[arg-type]
                )
            )
        elif isinstance(cfg, HierarchicalDocumentToolConfig):
            instance = tool_instances.get(cfg.name)
            if instance is None:
                raise BackendInitError(
                    f"No initialized instance found for tool '{cfg.name}' "
                    f"(type: {cfg.type}). Ensure tool initialization "
                    "completed before creating adapters."
                )
            adapters.append(
                HierarchicalDocToolAdapter(
                    config=cfg,
                    instance=instance,  # type: ignore[arg-type]
                )
            )
        elif isinstance(cfg, FunctionTool):
            func = load_function_tool(cfg, base_dir=base_dir)
            adapters.append(FunctionToolAdapter(config=cfg, callable=func))

    return adapters
```

### build_holodeck_sdk_server

## `build_holodeck_sdk_server(adapters)`

Bundle adapters into an in-process MCP server for the Claude subprocess.

Parameters:

| Name       | Type                           | Description                | Default                 |
| ---------- | ------------------------------ | -------------------------- | ----------------------- |
| `adapters` | \`list\[VectorStoreToolAdapter | HierarchicalDocToolAdapter | FunctionToolAdapter\]\` |

Returns:

| Type                                   | Description                                                     |
| -------------------------------------- | --------------------------------------------------------------- |
| `McpSdkServerConfig`                   | A tuple of (server_config, allowed_tool_names) where            |
| `list[str]`                            | server_config is a McpSdkServerConfig TypedDict and             |
| `tuple[McpSdkServerConfig, list[str]]` | allowed_tool_names are the fully-qualified MCP tool names.      |
| `tuple[McpSdkServerConfig, list[str]]` | Search-backed adapters contribute <name>\_search; function-tool |
| `tuple[McpSdkServerConfig, list[str]]` | adapters contribute the raw tool name.                          |

Source code in `src/holodeck/lib/backends/tool_adapters.py`

```
def build_holodeck_sdk_server(
    adapters: list[
        VectorStoreToolAdapter | HierarchicalDocToolAdapter | FunctionToolAdapter
    ],
) -> tuple[McpSdkServerConfig, list[str]]:
    """Bundle adapters into an in-process MCP server for the Claude subprocess.

    Args:
        adapters: Adapter objects produced by ``create_tool_adapters()``.

    Returns:
        A tuple of ``(server_config, allowed_tool_names)`` where
        *server_config* is a ``McpSdkServerConfig`` TypedDict and
        *allowed_tool_names* are the fully-qualified MCP tool names.
        Search-backed adapters contribute ``<name>_search``; function-tool
        adapters contribute the raw tool name.
    """
    sdk_tools: list[SdkMcpTool[Any]] = [a.to_sdk_tool() for a in adapters]

    server_config: McpSdkServerConfig = create_sdk_mcp_server(
        name=_SERVER_NAME,
        tools=sdk_tools,
    )

    allowed_tools: list[str] = []
    for a in adapters:
        if isinstance(a, FunctionToolAdapter):
            allowed_tools.append(f"mcp__{_SERVER_NAME}__{a.config.name}")
        else:
            allowed_tools.append(f"mcp__{_SERVER_NAME}__{a.config.name}_search")

    return server_config, allowed_tools
```

______________________________________________________________________

## `holodeck.lib.backends.mcp_bridge` -- MCP Configuration Bridge

Translates HoloDeck `MCPTool` configurations into Claude Agent SDK `McpStdioServerConfig` format for subprocess-based MCP servers. Only `stdio` transport tools are supported.

### build_claude_mcp_configs

## `build_claude_mcp_configs(mcp_tools)`

Translate HoloDeck MCPTool configs to Claude SDK MCP server configs.

Only stdio transport tools are supported by the Claude subprocess. Non-stdio tools are skipped with a warning.

Parameters:

| Name        | Type            | Description                                     | Default    |
| ----------- | --------------- | ----------------------------------------------- | ---------- |
| `mcp_tools` | `list[MCPTool]` | List of MCPTool configurations from agent YAML. | *required* |

Returns:

| Type                              | Description                                                       |
| --------------------------------- | ----------------------------------------------------------------- |
| `dict[str, McpStdioServerConfig]` | Dictionary mapping tool names to McpStdioServerConfig TypedDicts. |

Source code in `src/holodeck/lib/backends/mcp_bridge.py`

```
def build_claude_mcp_configs(
    mcp_tools: list[MCPTool],
) -> dict[str, McpStdioServerConfig]:
    """Translate HoloDeck MCPTool configs to Claude SDK MCP server configs.

    Only stdio transport tools are supported by the Claude subprocess.
    Non-stdio tools are skipped with a warning.

    Args:
        mcp_tools: List of MCPTool configurations from agent YAML.

    Returns:
        Dictionary mapping tool names to McpStdioServerConfig TypedDicts.
    """
    from holodeck.config.context import agent_base_dir

    base_dir = agent_base_dir.get()

    configs: dict[str, McpStdioServerConfig] = {}

    for tool in mcp_tools:
        if tool.transport != TransportType.STDIO:
            logger.warning(
                "Skipping MCP tool '%s': %s transport is not supported by "
                "Claude subprocess (only stdio is supported)",
                tool.name,
                tool.transport.value,
            )
            continue

        command = tool.command.value if tool.command else "npx"
        args = _resolve_relative_args(tool.args or [], base_dir)
        env = _resolve_mcp_env(tool)

        entry: McpStdioServerConfig = {
            "command": command,
            "args": args,
        }

        if env:
            entry["env"] = env

        configs[tool.name] = entry

    return configs
```

______________________________________________________________________

## `holodeck.lib.backends.otel_bridge` -- Observability Bridge

Translates HoloDeck `ObservabilityConfig` into environment variable dicts that configure OpenTelemetry for the Claude subprocess.

### translate_observability

## `translate_observability(config)`

Translate ObservabilityConfig to env vars for the Claude subprocess.

Produces a dict of environment variable key-value pairs that configure OpenTelemetry in the Claude subprocess. All values are strings.

Parameters:

| Name     | Type                  | Description                                           | Default    |
| -------- | --------------------- | ----------------------------------------------------- | ---------- |
| `config` | `ObservabilityConfig` | HoloDeck observability configuration from agent YAML. | *required* |

Returns:

| Type             | Description                                                |
| ---------------- | ---------------------------------------------------------- |
| `dict[str, str]` | Dictionary of environment variable names to string values. |
| `dict[str, str]` | Empty dict if observability is disabled.                   |

Source code in `src/holodeck/lib/backends/otel_bridge.py`

```
def translate_observability(config: ObservabilityConfig) -> dict[str, str]:
    """Translate ObservabilityConfig to env vars for the Claude subprocess.

    Produces a dict of environment variable key-value pairs that configure
    OpenTelemetry in the Claude subprocess. All values are strings.

    Args:
        config: HoloDeck observability configuration from agent YAML.

    Returns:
        Dictionary of environment variable names to string values.
        Empty dict if observability is disabled.
    """
    if not config.enabled:
        return {}

    env: dict[str, str] = {}

    # Core telemetry enablement
    env["CLAUDE_CODE_ENABLE_TELEMETRY"] = "1"

    # Disable subprocess trace export when Python-side GenAI instrumentation
    # is active.  The instrumentor's hooks produce the authoritative
    # invoke_agent / execute_tool spans in-process; letting the subprocess
    # also export traces creates duplicate, unlinked trace trees in the
    # collector and confuses dashboards like Aspire.
    if config.traces.enabled:
        env["OTEL_TRACES_EXPORTER"] = "none"

    # OTLP exporter configuration
    otlp = config.exporters.otlp
    otlp_enabled = otlp is not None and otlp.enabled

    # Metrics exporter
    if otlp_enabled and config.metrics.enabled:
        env["OTEL_METRICS_EXPORTER"] = "otlp"
    else:
        env["OTEL_METRICS_EXPORTER"] = "none"

    # Logs exporter
    if otlp_enabled and config.logs.enabled:
        env["OTEL_LOGS_EXPORTER"] = "otlp"
    else:
        env["OTEL_LOGS_EXPORTER"] = "none"

    # Protocol and endpoint (only if OTLP is enabled)
    if otlp_enabled and otlp is not None:
        env["OTEL_EXPORTER_OTLP_PROTOCOL"] = _PROTOCOL_MAP[otlp.protocol]
        env["OTEL_EXPORTER_OTLP_ENDPOINT"] = otlp.endpoint

    # Export intervals
    interval = str(config.metrics.export_interval_ms)
    env["OTEL_METRIC_EXPORT_INTERVAL"] = interval
    env["OTEL_LOGS_EXPORT_INTERVAL"] = interval

    # Privacy controls (FR-038: default off)
    if config.traces.capture_content:
        env["OTEL_LOG_USER_PROMPTS"] = "true"
        env["OTEL_LOG_TOOL_DETAILS"] = "true"

    # Warn about unsupported fields
    unsupported = _collect_unsupported_fields(config)
    if unsupported:
        logger.warning(
            "Claude subprocess does not support these observability settings "
            "(they will be ignored): %s",
            ", ".join(unsupported),
        )

    return env
```

______________________________________________________________________

## `holodeck.lib.backends.validators` -- Startup Validators

Pre-flight checks called by `ClaudeBackend.initialize()` before spawning the Claude subprocess. These surface configuration errors at startup rather than at runtime.

### validate_nodejs

## `validate_nodejs(agent)`

Validate that Node.js is available on PATH when the agent needs it.

Node.js is only required when at least one MCP tool spawns a Node interpreter (node, npx, yarn, pnpm). Agents without such tools skip this check entirely.

Parameters:

| Name    | Type    | Description                                              | Default    |
| ------- | ------- | -------------------------------------------------------- | ---------- |
| `agent` | `Agent` | Agent configuration to inspect for Node-dependent tools. | *required* |

Raises:

| Type          | Description                                          |
| ------------- | ---------------------------------------------------- |
| `ConfigError` | If node is not found on PATH and the agent needs it. |

Source code in `src/holodeck/lib/backends/validators.py`

```
def validate_nodejs(agent: Agent) -> None:
    """Validate that Node.js is available on PATH when the agent needs it.

    Node.js is only required when at least one MCP tool spawns a Node
    interpreter (node, npx, yarn, pnpm).  Agents without such tools skip
    this check entirely.

    Args:
        agent: Agent configuration to inspect for Node-dependent tools.

    Raises:
        ConfigError: If node is not found on PATH and the agent needs it.
    """
    if not agent_needs_nodejs(agent):
        return  # No Node MCP tools — installation unnecessary.

    if shutil.which("node") is None:
        raise ConfigError(
            "nodejs",
            "Node.js is required to run Claude Agent SDK but was not found on "
            "PATH. Install Node.js from https://nodejs.org/ and ensure it is "
            "on your PATH.",
        )

    stdout = ""
    try:
        result = subprocess.run(  # noqa: S603 # nosec B603 B607
            ["node", "--version"],  # noqa: S607
            capture_output=True,
            text=True,
            timeout=5,
        )
        stdout = result.stdout.strip()
        version = stdout.lstrip("v")
        major = int(version.split(".")[0])
    except subprocess.TimeoutExpired as e:
        raise ConfigError(
            "nodejs",
            "Node.js version check timed out. Ensure 'node --version' "
            "completes quickly.",
        ) from e
    except subprocess.CalledProcessError as e:
        raise ConfigError(
            "nodejs",
            "Failed to determine Node.js version. Ensure 'node --version' " "works.",
        ) from e
    except ValueError as e:
        raise ConfigError(
            "nodejs",
            f"Could not parse Node.js version from output: {stdout}",
        ) from e

    if major < NODEJS_MIN_VERSION:
        raise ConfigError(
            "nodejs",
            f"Node.js version {version} found but >= {NODEJS_MIN_VERSION} is "
            "required by Claude Agent SDK. Download from https://nodejs.org/",
        )

    logger.debug(f"Node.js version {version} validated (>= {NODEJS_MIN_VERSION})")
```

### validate_credentials

## `validate_credentials(model)`

Validate authentication credentials for the LLM provider.

Checks that the required environment variables are present for the configured auth_provider, including cloud routing context for Bedrock, Vertex, and Foundry. Returns a dict of environment variables to inject into the Claude subprocess.

Parameters:

| Name    | Type          | Description                 | Default    |
| ------- | ------------- | --------------------------- | ---------- |
| `model` | `LLMProvider` | LLM provider configuration. | *required* |

Returns:

| Type             | Description                                              |
| ---------------- | -------------------------------------------------------- |
| `dict[str, str]` | Dict of environment variables to set for the subprocess. |

Raises:

| Type          | Description                         |
| ------------- | ----------------------------------- |
| `ConfigError` | If required credentials are absent. |

Source code in `src/holodeck/lib/backends/validators.py`

```
def validate_credentials(model: LLMProvider) -> dict[str, str]:
    """Validate authentication credentials for the LLM provider.

    Checks that the required environment variables are present for the
    configured auth_provider, including cloud routing context for Bedrock,
    Vertex, and Foundry. Returns a dict of environment variables to inject
    into the Claude subprocess.

    Args:
        model: LLM provider configuration.

    Returns:
        Dict of environment variables to set for the subprocess.

    Raises:
        ConfigError: If required credentials are absent.
    """
    auth = model.auth_provider or AuthProvider.api_key

    if auth == AuthProvider.api_key:
        key = _get_required_env_var(
            "ANTHROPIC_API_KEY",
            "ANTHROPIC_API_KEY environment variable is not set. "
            "Set it with: export ANTHROPIC_API_KEY=sk-ant-...",
        )
        return {"ANTHROPIC_API_KEY": key}

    if auth == AuthProvider.oauth_token:
        token = _get_required_env_var(
            "CLAUDE_CODE_OAUTH_TOKEN",
            "CLAUDE_CODE_OAUTH_TOKEN environment variable is not set. "
            "Run `claude setup-token` to authenticate with OAuth.",
        )
        return {"CLAUDE_CODE_OAUTH_TOKEN": token}

    if auth == AuthProvider.bedrock:
        bedrock_region = _get_first_present_env_var(_BEDROCK_REGION_ENV_CANDIDATES)
        if bedrock_region is None:
            raise ConfigError(
                "AWS_REGION",
                "Missing AWS region for auth_provider: bedrock. Set either "
                "AWS_REGION or AWS_DEFAULT_REGION (for example: "
                "export AWS_REGION=us-east-1).",
            )
        return {
            "CLAUDE_CODE_USE_BEDROCK": "1",
            bedrock_region[0]: bedrock_region[1],
        }

    if auth == AuthProvider.vertex:
        region = _get_required_env_var(
            "CLOUD_ML_REGION",
            "CLOUD_ML_REGION environment variable is not set for "
            "auth_provider: vertex. Set it with: "
            "export CLOUD_ML_REGION=us-east5",
        )
        project_context = _get_first_present_env_var(_VERTEX_PROJECT_ENV_CANDIDATES)
        if project_context is None:
            raise ConfigError(
                "ANTHROPIC_VERTEX_PROJECT_ID",
                "Missing Vertex project context for auth_provider: vertex. "
                "Set one of: ANTHROPIC_VERTEX_PROJECT_ID, GCLOUD_PROJECT, "
                "GOOGLE_CLOUD_PROJECT, or GOOGLE_APPLICATION_CREDENTIALS.",
            )
        return {
            "CLAUDE_CODE_USE_VERTEX": "1",
            "CLOUD_ML_REGION": region,
            project_context[0]: project_context[1],
        }

    if auth == AuthProvider.custom:
        token = _get_required_env_var(
            "ANTHROPIC_AUTH_TOKEN",
            "ANTHROPIC_AUTH_TOKEN environment variable is not set. "
            "Set it for custom endpoint authentication "
            "(e.g., export ANTHROPIC_AUTH_TOKEN=ollama).",
        )
        return {"ANTHROPIC_AUTH_TOKEN": token}

    # AuthProvider.foundry (fallthrough)
    foundry_target = _get_first_present_env_var(_FOUNDRY_TARGET_ENV_CANDIDATES)
    if foundry_target is None:
        raise ConfigError(
            "ANTHROPIC_FOUNDRY_RESOURCE",
            "Missing Foundry target for auth_provider: foundry. "
            "Set one of: ANTHROPIC_FOUNDRY_RESOURCE or "
            "ANTHROPIC_FOUNDRY_BASE_URL.",
        )
    return {"CLAUDE_CODE_USE_FOUNDRY": "1", foundry_target[0]: foundry_target[1]}
```

### validate_embedding_provider

## `validate_embedding_provider(agent)`

Validate embedding provider configuration for vectorstore tools.

Anthropic does not support generating embeddings, so an external embedding_provider must be specified when using vectorstore tools with the Anthropic LLM provider.

Parameters:

| Name    | Type    | Description                      | Default    |
| ------- | ------- | -------------------------------- | ---------- |
| `agent` | `Agent` | Agent configuration to validate. | *required* |

Raises:

| Type          | Description                                             |
| ------------- | ------------------------------------------------------- |
| `ConfigError` | If embedding configuration is invalid for the provider. |

Source code in `src/holodeck/lib/backends/validators.py`

```
def validate_embedding_provider(agent: Agent) -> None:
    """Validate embedding provider configuration for vectorstore tools.

    Anthropic does not support generating embeddings, so an external
    embedding_provider must be specified when using vectorstore tools
    with the Anthropic LLM provider.

    Args:
        agent: Agent configuration to validate.

    Raises:
        ConfigError: If embedding configuration is invalid for the provider.
    """
    if not agent.tools:
        return

    has_vectorstore = any(
        isinstance(tool, VectorstoreTool | HierarchicalDocumentToolConfig)
        for tool in agent.tools
    )
    if not has_vectorstore:
        return

    # Anthropic cannot be used as an embedding provider
    if agent.embedding_provider is not None:
        if agent.embedding_provider.provider == ProviderEnum.ANTHROPIC:
            raise ConfigError(
                "embedding_provider",
                "Anthropic cannot generate embeddings. Use a different provider "
                "such as openai or azure_openai for embedding_provider when "
                "using vectorstore tools.",
            )
        return

    # Anthropic LLM + vectorstore tool + no embedding_provider
    if agent.model.provider == ProviderEnum.ANTHROPIC:
        raise ConfigError(
            "embedding_provider",
            "embedding_provider is required when using vectorstore tools with "
            "provider: anthropic. Add an embedding_provider using openai or "
            "azure_openai.",
        )
```

### validate_working_directory

## `validate_working_directory(path)`

Warn if CLAUDE.md in working directory may conflict with agent instructions.

Detects a CLAUDE.md file that contains a '# CLAUDE.md' header, which is the standard format used by Claude Code project instructions. Such a file may override or conflict with the agent's configured instructions.

Parameters:

| Name   | Type  | Description | Default                                             |
| ------ | ----- | ----------- | --------------------------------------------------- |
| `path` | \`str | None\`      | Working directory path, or None to skip validation. |

Source code in `src/holodeck/lib/backends/validators.py`

```
def validate_working_directory(path: str | None) -> None:
    """Warn if CLAUDE.md in working directory may conflict with agent instructions.

    Detects a CLAUDE.md file that contains a '# CLAUDE.md' header, which
    is the standard format used by Claude Code project instructions. Such
    a file may override or conflict with the agent's configured instructions.

    Args:
        path: Working directory path, or None to skip validation.
    """
    if path is None:
        return

    claude_md = Path(path) / "CLAUDE.md"
    if not claude_md.exists():
        return

    content = claude_md.read_text()
    if "# CLAUDE.md" in content:
        logger.warning(
            "A CLAUDE.md file with a '# CLAUDE.md' header was found in the "
            "working directory '%s'. This may conflict with the agent's system "
            "instructions. Review the file to avoid unexpected behavior.",
            path,
        )
```

### validate_response_format

## `validate_response_format(response_format)`

Validate response format schema is serializable and accessible.

Parameters:

| Name              | Type             | Description | Default |
| ----------------- | ---------------- | ----------- | ------- |
| `response_format` | \`dict[str, Any] | str         | None\`  |

Raises:

| Type          | Description                                               |
| ------------- | --------------------------------------------------------- |
| `ConfigError` | If the schema is not JSON-serializable or file not found. |

Source code in `src/holodeck/lib/backends/validators.py`

```
def validate_response_format(response_format: dict[str, Any] | str | None) -> None:
    """Validate response format schema is serializable and accessible.

    Args:
        response_format: Inline schema dict, file path string, or None.

    Raises:
        ConfigError: If the schema is not JSON-serializable or file not found.
    """
    if response_format is None:
        return

    if isinstance(response_format, str):
        schema_path = Path(response_format)
        if not schema_path.exists():
            raise ConfigError(
                "response_format",
                f"response_format file not found: {response_format}",
            )
        try:
            json.loads(schema_path.read_text())
        except (json.JSONDecodeError, OSError) as e:
            raise ConfigError(
                "response_format",
                f"response_format file is not valid JSON: {e}",
            ) from e
        return

    # Must be a dict — verify it is JSON-serializable
    try:
        json.dumps(response_format)
    except (TypeError, ValueError) as e:
        raise ConfigError(
            "response_format",
            f"response_format contains non-JSON-serializable values: {e}",
        ) from e
```

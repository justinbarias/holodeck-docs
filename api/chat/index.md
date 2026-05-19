# Chat API Reference

The chat subsystem provides interactive, multi-turn conversation capabilities for HoloDeck agents. It coordinates message validation, agent execution via the provider-agnostic backend layer, streaming responses, tool execution progress tracking, and session lifecycle management.

## Module: `holodeck.chat.executor`

Orchestrates agent execution for chat sessions using the backend abstraction layer (`AgentBackend` / `AgentSession`). Supports both synchronous turn-based and streaming response modes, with lazy backend initialization and optional task-bound session wrapping for HTTP server contexts.

### AgentResponse

## `AgentResponse(content, tool_executions, tokens_used, execution_time)`

Response from agent execution.

Contains the agent's text response, any tool executions performed, token usage tracking, and execution timing information.

### AgentExecutor

## `AgentExecutor(agent_config, backend=None, on_execution_start=None, on_execution_complete=None, release_transport_after_turn=False, llm_timeout=None)`

Coordinates agent execution for chat sessions.

Uses the provider-agnostic AgentBackend/AgentSession abstractions to execute user messages and manage conversation history.

Initialize executor with agent configuration.

No I/O is performed during construction — backend and session are lazily created on the first `execute_turn()` call.

Parameters:

| Name                           | Type                                | Description                                                                                                                                                                                               | Default                                                      |
| ------------------------------ | ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| `agent_config`                 | `Agent`                             | Agent configuration with model and instructions.                                                                                                                                                          | *required*                                                   |
| `backend`                      | \`AgentBackend                      | None\`                                                                                                                                                                                                    | Optional pre-initialized backend (bypasses BackendSelector). |
| `on_execution_start`           | \`Callable\[[str], None\]           | None\`                                                                                                                                                                                                    | Optional callback before agent execution.                    |
| `on_execution_complete`        | \`Callable\[[AgentResponse], None\] | None\`                                                                                                                                                                                                    | Optional callback after agent execution.                     |
| `release_transport_after_turn` | `bool`                              | If True, wrap the backend session in a \_TaskBoundSession actor so the SDK client stays in a single background task. Required for HTTP servers where each request runs in a different async task context. | `False`                                                      |
| `llm_timeout`                  | \`int                               | float                                                                                                                                                                                                     | None\`                                                       |

Source code in `src/holodeck/chat/executor.py`

```
def __init__(
    self,
    agent_config: Agent,
    backend: AgentBackend | None = None,
    on_execution_start: Callable[[str], None] | None = None,
    on_execution_complete: Callable[[AgentResponse], None] | None = None,
    release_transport_after_turn: bool = False,
    llm_timeout: int | float | None = None,
) -> None:
    """Initialize executor with agent configuration.

    No I/O is performed during construction — backend and session
    are lazily created on the first ``execute_turn()`` call.

    Args:
        agent_config: Agent configuration with model and instructions.
        backend: Optional pre-initialized backend (bypasses BackendSelector).
        on_execution_start: Optional callback before agent execution.
        on_execution_complete: Optional callback after agent execution.
        release_transport_after_turn: If True, wrap the backend session
            in a ``_TaskBoundSession`` actor so the SDK client stays in
            a single background task.  Required for HTTP servers where
            each request runs in a different async task context.
        llm_timeout: Per-turn LLM invocation timeout in seconds. When
            set, ``session.send`` / ``session.send_streaming`` are
            wrapped with ``asyncio.wait_for(timeout=llm_timeout)``.
            Mirrors the pattern used by ``TestExecutor``.
    """
    self.agent_config = agent_config
    self._backend: AgentBackend | None = backend
    self._session: AgentSession | None = None
    self._history: list[dict[str, Any]] = []
    self.on_execution_start = on_execution_start
    self.on_execution_complete = on_execution_complete
    self._use_task_bound_session = release_transport_after_turn
    self._llm_timeout = float(llm_timeout) if llm_timeout else None

    logger.info(f"AgentExecutor initialized for agent: {agent_config.name}")
```

### `tool_event_queue`

The tool event queue from the underlying session, if available.

### `clear_history()`

Clear conversation history and close the current session.

Resets the agent's chat history to start fresh conversation. The next `execute_turn()` will create a new session.

Source code in `src/holodeck/chat/executor.py`

```
async def clear_history(self) -> None:
    """Clear conversation history and close the current session.

    Resets the agent's chat history to start fresh conversation.
    The next ``execute_turn()`` will create a new session.
    """
    logger.debug("Clearing chat history and closing session")
    if self._session is not None:
        await self._session.close()
        self._session = None
    self._history = []
```

### `execute_turn(message)`

Execute a single turn of agent conversation.

Sends a user message to the agent, captures the response, extracts tool calls, and tracks token usage.

Parameters:

| Name      | Type  | Description                        | Default    |
| --------- | ----- | ---------------------------------- | ---------- |
| `message` | `str` | User message to send to the agent. | *required* |

Returns:

| Type            | Description                                                      |
| --------------- | ---------------------------------------------------------------- |
| `AgentResponse` | AgentResponse with content, tool executions, tokens, and timing. |

Raises:

| Type           | Description               |
| -------------- | ------------------------- |
| `RuntimeError` | If agent execution fails. |

Source code in `src/holodeck/chat/executor.py`

```
async def execute_turn(self, message: str) -> AgentResponse:
    """Execute a single turn of agent conversation.

    Sends a user message to the agent, captures the response,
    extracts tool calls, and tracks token usage.

    Args:
        message: User message to send to the agent.

    Returns:
        AgentResponse with content, tool executions, tokens, and timing.

    Raises:
        RuntimeError: If agent execution fails.
    """
    start_time = time.time()

    try:
        logger.debug(f"Executing turn for agent: {self.agent_config.name}")

        # Call pre-execution callback
        if self.on_execution_start:
            self.on_execution_start(message)

        # Lazy initialize backend and session
        await self._ensure_backend_and_session()

        # Invoke agent via backend session (timeout-wrapped when configured)
        if self._llm_timeout is not None:
            result: ExecutionResult = await asyncio.wait_for(
                self._session.send(message),  # type: ignore[union-attr]
                timeout=self._llm_timeout,
            )
        else:
            result = await self._session.send(message)  # type: ignore[union-attr]
        elapsed = time.time() - start_time

        # Extract content from execution result
        content = result.response

        # Convert tool calls to ToolExecution models
        tool_executions = self._convert_tool_calls(result.tool_calls)

        # Extract token usage (always a TokenUsage, never None)
        tokens_used = result.token_usage

        logger.debug(
            f"Turn executed successfully: content={len(content)} chars, "
            f"tools={len(tool_executions)}, time={elapsed:.2f}s"
        )

        # Track history
        self._history.append({"role": "user", "content": message})
        self._history.append({"role": "assistant", "content": content})

        response = AgentResponse(
            content=content,
            tool_executions=tool_executions,
            tokens_used=tokens_used,
            execution_time=elapsed,
        )

        # Call post-execution callback
        if self.on_execution_complete:
            self.on_execution_complete(response)

        return response

    except BackendSessionError:
        raise
    except BackendInitError as e:
        logger.error(f"Agent execution failed: {e}", exc_info=True)
        raise RuntimeError(f"Agent execution failed: {e}") from e
    except asyncio.TimeoutError:
        # llm_timeout exceeded — let the caller distinguish timeout
        # from crash instead of wrapping it as a generic RuntimeError.
        logger.warning(
            "Agent invocation exceeded llm_timeout=%ss",
            self._llm_timeout,
        )
        raise
    except RuntimeError:
        raise
    except Exception as e:
        logger.error(f"Agent execution failed: {e}", exc_info=True)
        raise RuntimeError(f"Agent execution failed: {e}") from e
```

### `execute_turn_streaming(message)`

Stream agent response token by token.

Parameters:

| Name      | Type  | Description                        | Default    |
| --------- | ----- | ---------------------------------- | ---------- |
| `message` | `str` | User message to send to the agent. | *required* |

Yields:

| Type                        | Description                                     |
| --------------------------- | ----------------------------------------------- |
| `AsyncGenerator[str, None]` | Successive string chunks of the agent response. |

Raises:

| Type           | Description               |
| -------------- | ------------------------- |
| `RuntimeError` | If agent execution fails. |

Source code in `src/holodeck/chat/executor.py`

```
async def execute_turn_streaming(self, message: str) -> AsyncGenerator[str, None]:
    """Stream agent response token by token.

    Args:
        message: User message to send to the agent.

    Yields:
        Successive string chunks of the agent response.

    Raises:
        RuntimeError: If agent execution fails.
    """
    try:
        await self._ensure_backend_and_session()
        collected: list[str] = []
        stream = self._session.send_streaming(message)  # type: ignore[union-attr]
        # Enforce llm_timeout as a deadline across the whole stream —
        # each chunk must arrive within the remaining budget; otherwise
        # the next __anext__ is cancelled and TimeoutError propagates.
        deadline = (
            time.monotonic() + self._llm_timeout
            if self._llm_timeout is not None
            else None
        )
        while True:
            if deadline is not None:
                remaining = deadline - time.monotonic()
                if remaining <= 0:
                    raise asyncio.TimeoutError(
                        f"streaming response exceeded llm_timeout="
                        f"{self._llm_timeout}s"
                    )
                try:
                    chunk = await asyncio.wait_for(
                        stream.__anext__(), timeout=remaining
                    )
                except StopAsyncIteration:
                    break
            else:
                try:
                    chunk = await stream.__anext__()
                except StopAsyncIteration:
                    break
            collected.append(chunk)
            yield chunk
        # Update history after stream completes
        self._history.append({"role": "user", "content": message})
        self._history.append({"role": "assistant", "content": "".join(collected)})
    except BackendSessionError:
        raise
    except BackendInitError as e:
        raise RuntimeError(f"Agent streaming failed: {e}") from e
```

### `get_history()`

Get current conversation history.

Returns:

| Type                   | Description                                                        |
| ---------------------- | ------------------------------------------------------------------ |
| `list[dict[str, Any]]` | Serialized conversation history as a list of dicts, or empty list. |

Source code in `src/holodeck/chat/executor.py`

```
def get_history(self) -> list[dict[str, Any]]:
    """Get current conversation history.

    Returns:
        Serialized conversation history as a list of dicts, or empty list.
    """
    return list(self._history)
```

### `shutdown()`

Cleanup executor resources.

Called when ending a chat session to release any held resources. Closes the session and tears down the backend.

Source code in `src/holodeck/chat/executor.py`

```
async def shutdown(self) -> None:
    """Cleanup executor resources.

    Called when ending a chat session to release any held resources.
    Closes the session and tears down the backend.
    """
    try:
        logger.debug("AgentExecutor shutting down")
        if self._session is not None:
            await self._session.close()
            self._session = None
        if self._backend is not None:
            await self._backend.teardown()
            self._backend = None
        logger.debug("AgentExecutor shutdown complete")
    except Exception as e:
        logger.error(f"Error during shutdown: {e}")
```

## Module: `holodeck.chat.message`

Validates user messages before they reach the agent, enforcing content standards such as empty-message detection, size limits, control-character filtering, and UTF-8 validation.

### MessageValidator

## `MessageValidator(max_length=10000)`

Validates user messages before sending to the agent.

Uses ValidationPipeline to enforce content standards including empty message detection, size limits, control character filtering, and UTF-8 validation.

Initialize validator with length constraints.

Parameters:

| Name         | Type  | Description                                               | Default |
| ------------ | ----- | --------------------------------------------------------- | ------- |
| `max_length` | `int` | Maximum message length in characters. Defaults to 10,000. | `10000` |

Source code in `src/holodeck/chat/message.py`

```
def __init__(self, max_length: int = 10_000) -> None:
    """Initialize validator with length constraints.

    Args:
        max_length: Maximum message length in characters. Defaults to 10,000.
    """
    self._pipeline = ValidationPipeline(max_length=max_length)
```

### `validate(message)`

Validate a message and return validation status.

Parameters:

| Name      | Type  | Description | Default                                                 |
| --------- | ----- | ----------- | ------------------------------------------------------- |
| `message` | \`str | None\`      | User message to validate (None, empty, or any content). |

Returns:

| Type               | Description                                  |
| ------------------ | -------------------------------------------- |
| `bool`             | Tuple of (is_valid: bool, error_message: str |
| \`str              | None\`                                       |
| \`tuple\[bool, str | None\]\`                                     |

Validation checks:

- Message is not None or empty
- Message does not exceed max_length
- Message contains no control characters
- Message is valid UTF-8

Source code in `src/holodeck/chat/message.py`

```
def validate(self, message: str | None) -> tuple[bool, str | None]:
    """Validate a message and return validation status.

    Args:
        message: User message to validate (None, empty, or any content).

    Returns:
        Tuple of (is_valid: bool, error_message: str | None).
        If valid, error_message is None.
        If invalid, error_message describes the validation failure.

    Validation checks:
    - Message is not None or empty
    - Message does not exceed max_length
    - Message contains no control characters
    - Message is valid UTF-8
    """
    return self._pipeline.validate(message)
```

## Module: `holodeck.chat.session`

Manages chat session lifecycle and state, coordinating between message validation, agent execution, token tracking, and session statistics.

### ChatSessionManager

## `ChatSessionManager(agent_config, config)`

Maintains chat session lifecycle and state management.

Coordinates between message validation, agent execution, and session state tracking.

Initialize session manager with configuration.

Parameters:

| Name           | Type         | Description                               | Default    |
| -------------- | ------------ | ----------------------------------------- | ---------- |
| `agent_config` | `Agent`      | Agent configuration to use for execution. | *required* |
| `config`       | `ChatConfig` | Chat runtime configuration.               | *required* |

Source code in `src/holodeck/chat/session.py`

```
def __init__(self, agent_config: Agent, config: ChatConfig) -> None:
    """Initialize session manager with configuration.

    Args:
        agent_config: Agent configuration to use for execution.
        config: Chat runtime configuration.
    """
    self.agent_config = agent_config
    self.config = config
    self.session: ChatSession | None = None
    self._executor: AgentExecutor | None = None
    self._validator = MessageValidator(max_length=10_000)

    # Token tracking
    self.total_tokens = TokenUsage(
        prompt_tokens=0,
        completion_tokens=0,
        total_tokens=0,
    )
    self.session_start_time = datetime.now()

    logger.debug(f"ChatSessionManager initialized for agent: {agent_config.name}")
```

### `get_session()`

Get current chat session.

Returns:

| Type          | Description |
| ------------- | ----------- |
| \`ChatSession | None\`      |

Source code in `src/holodeck/chat/session.py`

```
def get_session(self) -> ChatSession | None:
    """Get current chat session.

    Returns:
        ChatSession instance, or None if not started.
    """
    return self.session
```

### `get_session_stats()`

Get current session statistics.

Returns:

| Type             | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| `dict[str, Any]` | Dict with message_count, total_tokens, and session_duration. |

Source code in `src/holodeck/chat/session.py`

```
def get_session_stats(self) -> dict[str, Any]:
    """Get current session statistics.

    Returns:
        Dict with message_count, total_tokens, and session_duration.
    """
    session_duration = (datetime.now() - self.session_start_time).total_seconds()
    return {
        "message_count": self.session.message_count if self.session else 0,
        "total_tokens": self.total_tokens,
        "session_duration": session_duration,
    }
```

### `process_message(message)`

Process a user message through validation and execution.

Parameters:

| Name      | Type  | Description              | Default    |
| --------- | ----- | ------------------------ | ---------- |
| `message` | `str` | User message to process. | *required* |

Returns:

| Type            | Description                   |
| --------------- | ----------------------------- |
| `AgentResponse` | AgentResponse from the agent. |

Raises:

| Type           | Description                                |
| -------------- | ------------------------------------------ |
| `RuntimeError` | If session not started or execution fails. |
| `ValueError`   | If message validation fails.               |

Source code in `src/holodeck/chat/session.py`

```
async def process_message(self, message: str) -> AgentResponse:
    """Process a user message through validation and execution.

    Args:
        message: User message to process.

    Returns:
        AgentResponse from the agent.

    Raises:
        RuntimeError: If session not started or execution fails.
        ValueError: If message validation fails.
    """
    if self.session is None or self._executor is None:
        raise RuntimeError("Session not started. Call start() first.")

    # Validate message
    is_valid, error = self._validator.validate(message)
    if not is_valid:
        raise ValueError(f"Invalid message: {error}")

    # Execute turn
    try:
        logger.debug(f"Processing message ({len(message)} chars)")
        response = await self._executor.execute_turn(message)

        # Increment message count
        self.session.message_count += 1

        # Accumulate token usage
        if response.tokens_used:
            self.total_tokens = TokenUsage(
                prompt_tokens=self.total_tokens.prompt_tokens
                + response.tokens_used.prompt_tokens,
                completion_tokens=self.total_tokens.completion_tokens
                + response.tokens_used.completion_tokens,
                total_tokens=self.total_tokens.total_tokens
                + response.tokens_used.total_tokens,
            )

        logger.debug(
            f"Message processed: count={self.session.message_count}, "
            f"tools={len(response.tool_executions)}"
        )

        return response

    except Exception as e:
        logger.error(f"Message processing failed: {e}", exc_info=True)
        raise RuntimeError(f"Message processing failed: {e}") from e
```

### `process_message_streaming(message)`

Stream a user message through validation and agent execution.

Parameters:

| Name      | Type  | Description              | Default    |
| --------- | ----- | ------------------------ | ---------- |
| `message` | `str` | User message to process. | *required* |

Yields:

| Type                        | Description                                     |
| --------------------------- | ----------------------------------------------- |
| `AsyncGenerator[str, None]` | Successive string chunks of the agent response. |

Raises:

| Type           | Description                                |
| -------------- | ------------------------------------------ |
| `RuntimeError` | If session not started or execution fails. |
| `ValueError`   | If message validation fails.               |

Source code in `src/holodeck/chat/session.py`

```
async def process_message_streaming(
    self, message: str
) -> AsyncGenerator[str, None]:
    """Stream a user message through validation and agent execution.

    Args:
        message: User message to process.

    Yields:
        Successive string chunks of the agent response.

    Raises:
        RuntimeError: If session not started or execution fails.
        ValueError: If message validation fails.
    """
    if self.session is None or self._executor is None:
        raise RuntimeError("Session not started. Call start() first.")

    # Validate message (same as process_message)
    is_valid, error = self._validator.validate(message)
    if not is_valid:
        raise ValueError(f"Invalid message: {error}")

    logger.debug(f"Streaming message ({len(message)} chars)")
    async for chunk in self._executor.execute_turn_streaming(message):
        yield chunk

    # Increment message count after stream completes
    self.session.message_count += 1
    logger.debug(f"Streamed message: count={self.session.message_count}")
```

### `should_warn_context_limit()`

Check if conversation is approaching context limit.

Returns:

| Type   | Description                                                    |
| ------ | -------------------------------------------------------------- |
| `bool` | True if message count >= 80% of max_messages, False otherwise. |

Source code in `src/holodeck/chat/session.py`

```
def should_warn_context_limit(self) -> bool:
    """Check if conversation is approaching context limit.

    Returns:
        True if message count >= 80% of max_messages, False otherwise.
    """
    if self.session is None:
        return False

    threshold = int(self.config.max_messages * 0.8)
    should_warn = self.session.message_count >= threshold
    return should_warn
```

### `start()`

Start a new chat session.

Initializes the agent executor, creates a chat session, and transitions state to ACTIVE.

Raises:

| Type           | Description                      |
| -------------- | -------------------------------- |
| `RuntimeError` | If session initialization fails. |

Source code in `src/holodeck/chat/session.py`

```
async def start(self) -> None:
    """Start a new chat session.

    Initializes the agent executor, creates a chat session,
    and transitions state to ACTIVE.

    Raises:
        RuntimeError: If session initialization fails.
    """
    logger.info(f"Starting chat session for agent: {self.agent_config.name}")

    # Constructor does no I/O — errors surface on first turn (lazy-init)
    self._executor = AgentExecutor(
        self.agent_config,
        llm_timeout=self.config.llm_timeout,
    )

    # Create chat session with empty history
    self.session = ChatSession(
        agent_config=self.agent_config,
        history=[],
        state=SessionState.ACTIVE,
    )

    logger.info(f"Chat session started: session_id={self.session.session_id}")
```

### `terminate()`

Terminate the chat session.

Cleans up resources and transitions state to TERMINATED.

Source code in `src/holodeck/chat/session.py`

```
async def terminate(self) -> None:
    """Terminate the chat session.

    Cleans up resources and transitions state to TERMINATED.
    """
    try:
        if self.session is not None:
            self.session.state = SessionState.TERMINATED
            logger.info(
                f"Chat session terminated: session_id={self.session.session_id}"
            )

        if self._executor is not None:
            await self._executor.shutdown()

    except Exception as e:
        logger.error(f"Error during session termination: {e}", exc_info=True)
```

## Module: `holodeck.chat.streaming`

Streams tool execution events to callers in real time, allowing UIs to display progress as tools start, run, and complete (or fail).

### ToolExecutionStream

## `ToolExecutionStream(verbose=False)`

Streams tool execution events to the caller.

Emits ToolEvent instances as a tool executes, allowing callers to display real-time progress. Supports both standard and verbose modes.

Initialize the stream with verbosity preference.

Parameters:

| Name      | Type   | Description                                                                                                              | Default |
| --------- | ------ | ------------------------------------------------------------------------------------------------------------------------ | ------- |
| `verbose` | `bool` | If True, include detailed execution data (parameters, results). If False, emit minimal data (tool name, status, timing). | `False` |

Source code in `src/holodeck/chat/streaming.py`

```
def __init__(self, verbose: bool = False) -> None:
    """Initialize the stream with verbosity preference.

    Args:
        verbose: If True, include detailed execution data (parameters, results).
                If False, emit minimal data (tool name, status, timing).
    """
    self.verbose = verbose
```

### `stream_execution(tool_call)`

Stream execution events for a tool call.

Simulates the execution lifecycle by emitting events:

1. STARTED - Tool execution begins
1. PROGRESS - (optional, for long operations)
1. COMPLETED or FAILED - Execution finished

Parameters:

| Name        | Type            | Description                                 | Default    |
| ----------- | --------------- | ------------------------------------------- | ---------- |
| `tool_call` | `ToolExecution` | Tool execution with status and result data. | *required* |

Yields:

| Type                       | Description                                             |
| -------------------------- | ------------------------------------------------------- |
| `AsyncIterator[ToolEvent]` | ToolEvent instances representing execution progression. |

Source code in `src/holodeck/chat/streaming.py`

```
async def stream_execution(
    self, tool_call: ToolExecution
) -> AsyncIterator[ToolEvent]:
    """Stream execution events for a tool call.

    Simulates the execution lifecycle by emitting events:
    1. STARTED - Tool execution begins
    2. PROGRESS - (optional, for long operations)
    3. COMPLETED or FAILED - Execution finished

    Args:
        tool_call: Tool execution with status and result data.

    Yields:
        ToolEvent instances representing execution progression.
    """
    # Emit STARTED event
    yield ToolEvent(
        event_type=ToolEventType.STARTED,
        tool_name=tool_call.tool_name,
        timestamp=datetime.utcnow(),
        data={"parameters": tool_call.parameters} if self.verbose else {},
    )

    # Emit COMPLETED or FAILED based on status
    if tool_call.status == ToolStatus.SUCCESS:
        yield ToolEvent(
            event_type=ToolEventType.COMPLETED,
            tool_name=tool_call.tool_name,
            timestamp=datetime.utcnow(),
            data={
                "result": tool_call.result if self.verbose else None,
                "execution_time": tool_call.execution_time or 0,
            },
        )
    elif tool_call.status == ToolStatus.FAILED:
        yield ToolEvent(
            event_type=ToolEventType.FAILED,
            tool_name=tool_call.tool_name,
            timestamp=datetime.utcnow(),
            data={
                "error": tool_call.error_message or "Unknown error",
            },
        )
    else:
        # For PENDING or RUNNING, emit as PROGRESS
        yield ToolEvent(
            event_type=ToolEventType.PROGRESS,
            tool_name=tool_call.tool_name,
            timestamp=datetime.utcnow(),
            data={"status": tool_call.status.value},
        )
```

### ToolEvent and ToolEventType

`ToolEvent` and `ToolEventType` are re-exported from `holodeck.models.tool_event`. See the [Models API Reference](https://docs.useholodeck.ai/api/models/#tool-execution-and-events) for full documentation.

## Module: `holodeck.chat.progress`

Tracks and displays chat session progress with animated spinners and adaptive status output (inline for default mode, rich panel for verbose mode).

### ChatProgressIndicator

## `ChatProgressIndicator(max_messages, quiet, verbose)`

Bases: `SpinnerMixin`

Track and display chat session progress with spinner and status information.

Provides animated spinner during agent execution and adaptive status display (minimal in default mode, rich in verbose mode). Tracks message count, tokens, session time, response timing, and tool executions.

Inherits spinner animation from SpinnerMixin.

Initialize progress indicator.

Parameters:

| Name           | Type   | Description                                      | Default    |
| -------------- | ------ | ------------------------------------------------ | ---------- |
| `max_messages` | `int`  | Maximum messages for session before warning.     | *required* |
| `quiet`        | `bool` | Suppress status display (spinner still shows).   | *required* |
| `verbose`      | `bool` | Show rich status panel instead of inline status. | *required* |

Source code in `src/holodeck/chat/progress.py`

```
def __init__(self, max_messages: int, quiet: bool, verbose: bool) -> None:
    """Initialize progress indicator.

    Args:
        max_messages: Maximum messages for session before warning.
        quiet: Suppress status display (spinner still shows).
        verbose: Show rich status panel instead of inline status.
    """
    self.max_messages = max_messages
    self.quiet = quiet
    self.verbose = verbose

    # State tracking
    self.current_messages = 0
    self.total_tokens = TokenUsage.zero()
    self.last_response_time: float | None = None
    self.session_start = datetime.now()
    self._spinner_index = 0

    # Tool execution tracking
    self.last_tool_executions: list[ToolExecution] = []
    self.total_tool_calls: int = 0
    # Snapshot of tools observed via the live panel for the verbose
    # post-turn summary.  Populated by :meth:`set_active_snapshot`.
    self._panel_snapshot: list[tuple[str, bool, str | None]] = []
```

### `get_spinner_char()`

Get current spinner character and advance rotation.

Returns:

| Type  | Description                                          |
| ----- | ---------------------------------------------------- |
| `str` | Current spinner character from the braille sequence. |

Source code in `src/holodeck/lib/ui/spinner.py`

```
def get_spinner_char(self) -> str:
    """Get current spinner character and advance rotation.

    Returns:
        Current spinner character from the braille sequence.
    """
    char = self.SPINNER_CHARS[self._spinner_index % len(self.SPINNER_CHARS)]
    self._spinner_index += 1
    return char
```

### `get_spinner_line()`

Get current spinner animation frame.

Returns:

| Type  | Description                                        |
| ----- | -------------------------------------------------- |
| `str` | Animated spinner text, or empty string if not TTY. |

Source code in `src/holodeck/chat/progress.py`

```
def get_spinner_line(self) -> str:
    """Get current spinner animation frame.

    Returns:
        Animated spinner text, or empty string if not TTY.
    """
    if not is_tty():
        return ""

    spinner_char = self.get_spinner_char()
    return f"{spinner_char} Thinking..."
```

### `get_status_inline()`

Get minimal inline status for default mode.

Format: [messages_current/messages_max | execution_time]

Returns:

| Type  | Description           |
| ----- | --------------------- |
| `str` | Inline status string. |

Source code in `src/holodeck/chat/progress.py`

```
def get_status_inline(self) -> str:
    """Get minimal inline status for default mode.

    Format: [messages_current/messages_max | execution_time]

    Returns:
        Inline status string.
    """
    status_parts = []

    # Message count
    status_parts.append(f"{self.current_messages}/{self.max_messages}")

    # Execution time
    if self.last_response_time is not None:
        time_str = f"{self.last_response_time:.1f}s"
        status_parts.append(time_str)

    return f"[{' | '.join(status_parts)}]" if status_parts else ""
```

### `get_status_panel()`

Get rich status panel for verbose mode.

Returns:

| Type  | Description                     |
| ----- | ------------------------------- |
| `str` | Multi-line status panel string. |

Source code in `src/holodeck/chat/progress.py`

```
def get_status_panel(self) -> str:
    """Get rich status panel for verbose mode.

    Returns:
        Multi-line status panel string.
    """
    lines = []
    content_width = 39  # Width of content area (excluding "│ " and " │")

    # Top border
    lines.append("╭─── Chat Status ─────────────────────────╮")

    # Session time
    session_duration = (datetime.now() - self.session_start).total_seconds()
    hours = int(session_duration // 3600)
    minutes = int((session_duration % 3600) // 60)
    seconds = int(session_duration % 60)
    time_str = f"{hours:02d}:{minutes:02d}:{seconds:02d}"
    content = f"Session Time: {time_str}"
    lines.append(f"│ {content:<{content_width}} │")

    # Message count with percentage
    percentage = int((self.current_messages / self.max_messages) * 100)
    msg_str = f"{self.current_messages} / {self.max_messages} ({percentage}%)"
    content = f"Messages: {msg_str}"
    lines.append(f"│ {content:<{content_width}} │")

    # Token usage
    total_str = f"{self.total_tokens.total_tokens:,}"
    content = f"Total Tokens: {total_str}"
    lines.append(f"│ {content:<{content_width}} │")

    # Token breakdown - prompt
    prompt_str = f"{self.total_tokens.prompt_tokens:,}"
    content = f"  ├─ Prompt: {prompt_str}"
    lines.append(f"│ {content:<{content_width}} │")

    # Token breakdown - completion
    completion_str = f"{self.total_tokens.completion_tokens:,}"
    content = f"  └─ Completion: {completion_str}"
    lines.append(f"│ {content:<{content_width}} │")

    # Last response time
    if self.last_response_time is not None:
        time_str = f"{self.last_response_time:.1f}s"
        content = f"Last Response: {time_str}"
        lines.append(f"│ {content:<{content_width}} │")

    # Tool execution summary
    content = f"Tool Calls (Total): {self.total_tool_calls}"
    lines.append(f"│ {content:<{content_width}} │")

    # Last tool executions (if any) — prefer the live-panel snapshot
    # when available since streaming responses never populate
    # ``last_tool_executions``.
    if self._panel_snapshot:
        lines.append(f"│ {'Last Tools Called:':<{content_width}} │")
        status_icon = self._get_status_icon(ToolStatus.SUCCESS)
        for tool_name, is_subagent, subagent_type in self._panel_snapshot:
            label = (
                f"Task[{subagent_type}]"
                if is_subagent and subagent_type
                else tool_name
            )
            max_name_len = content_width - 6
            if len(label) > max_name_len:
                label = label[: max_name_len - 3] + "..."
            content = f"  └─ {status_icon} {label}"
            lines.append(f"│ {content:<{content_width}} │")
    elif self.last_tool_executions:
        lines.append(f"│ {'Last Tools Called:':<{content_width}} │")
        for tool_exec in self.last_tool_executions:
            # Get status indicator
            status_icon = self._get_status_icon(tool_exec.status)
            # Truncate tool name if too long
            tool_name = tool_exec.tool_name
            max_name_len = content_width - 6  # Account for "  └─ " and icon
            if len(tool_name) > max_name_len:
                tool_name = tool_name[: max_name_len - 3] + "..."
            content = f"  └─ {status_icon} {tool_name}"
            lines.append(f"│ {content:<{content_width}} │")

    # Bottom border
    lines.append("╰─────────────────────────────────────────╯")

    return "\n".join(lines)
```

### `set_active_snapshot(entries)`

Record the panel's view of tools that ran this turn.

Streaming responses don't surface `tool_executions` (they always return `[]`), so the verbose summary panel uses this snapshot from :class:`holodeck.chat.tools_panel.ToolsPanel` instead.

Parameters:

| Name      | Type  | Description                                                                                                                                      | Default    |
| --------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| `entries` | `Any` | Iterable of objects exposing tool_name, is_subagent, and subagent_type attributes (i.e. the \_Active records returned by ToolsPanel.snapshot()). | *required* |

Source code in `src/holodeck/chat/progress.py`

```
def set_active_snapshot(self, entries: Any) -> None:
    """Record the panel's view of tools that ran this turn.

    Streaming responses don't surface ``tool_executions`` (they always
    return ``[]``), so the verbose summary panel uses this snapshot
    from :class:`holodeck.chat.tools_panel.ToolsPanel` instead.

    Args:
        entries: Iterable of objects exposing ``tool_name``,
            ``is_subagent``, and ``subagent_type`` attributes (i.e. the
            ``_Active`` records returned by ``ToolsPanel.snapshot()``).
    """
    self._panel_snapshot = [
        (e.tool_name, e.is_subagent, e.subagent_type) for e in entries
    ]
    # Count every tool the panel saw towards the cumulative total.
    self.total_tool_calls += len(self._panel_snapshot)
```

### `update(response)`

Update progress after agent response.

Parameters:

| Name       | Type  | Description                                                                 | Default    |
| ---------- | ----- | --------------------------------------------------------------------------- | ---------- |
| `response` | `Any` | AgentResponse object with execution_time, tokens_used, and tool_executions. | *required* |

Source code in `src/holodeck/chat/progress.py`

```
def update(self, response: Any) -> None:
    """Update progress after agent response.

    Args:
        response: AgentResponse object with execution_time, tokens_used,
            and tool_executions.
    """
    # Update message count
    self.current_messages += 1

    # Update execution time
    if hasattr(response, "execution_time"):
        self.last_response_time = response.execution_time

    # Accumulate token usage
    if hasattr(response, "tokens_used") and response.tokens_used:
        self.total_tokens = self.total_tokens + response.tokens_used

    # Track tool executions
    if hasattr(response, "tool_executions") and response.tool_executions:
        self.last_tool_executions = response.tool_executions
        self.total_tool_calls += len(response.tool_executions)
    else:
        self.last_tool_executions = []
```

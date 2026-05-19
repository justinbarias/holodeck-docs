# Serve (Agent Local Server)

The `holodeck.serve` package provides HTTP server functionality for exposing HoloDeck agents via AG-UI (default) or REST protocols. It includes session management, middleware, multimodal file handling, and protocol adapters.

______________________________________________________________________

## Server

The main entry point. `AgentServer` wraps a FastAPI application and manages the full server lifecycle -- initialization, request routing, session cleanup, and graceful shutdown.

## `AgentServer(agent_config, protocol=ProtocolType.AG_UI, host='127.0.0.1', port=8000, cors_origins=None, debug=False, execution_config=None, observability_enabled=False, max_concurrent_init_jobs=4)`

HTTP server for exposing a single HoloDeck agent.

The AgentServer wraps a FastAPI application and manages the server lifecycle, including session management and protocol handling.

Attributes:

| Name           | Type | Description                                   |
| -------------- | ---- | --------------------------------------------- |
| `agent_config` |      | The agent configuration to serve.             |
| `protocol`     |      | The protocol to use (AG-UI or REST).          |
| `host`         |      | The hostname to bind to.                      |
| `port`         |      | The port to listen on.                        |
| `sessions`     |      | The session store for managing conversations. |
| `state`        |      | The current server state.                     |

Initialize the agent server.

Parameters:

| Name                    | Type              | Description                                                                                                 | Default                                         |
| ----------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------- | ----------------------------------------------- |
| `agent_config`          | `Agent`           | The agent configuration to serve.                                                                           | *required*                                      |
| `protocol`              | `ProtocolType`    | The protocol to use (default: AG-UI).                                                                       | `AG_UI`                                         |
| `host`                  | `str`             | The hostname to bind to (default: 127.0.0.1 for security). Use 0.0.0.0 to expose to all network interfaces. | `'127.0.0.1'`                                   |
| `port`                  | `int`             | The port to listen on (default: 8000).                                                                      | `8000`                                          |
| `cors_origins`          | \`list[str]       | None\`                                                                                                      | List of allowed CORS origins (default: ["\*"]). |
| `debug`                 | `bool`            | Enable debug logging (default: False).                                                                      | `False`                                         |
| `execution_config`      | \`ExecutionConfig | None\`                                                                                                      | Resolved execution configuration for timeouts.  |
| `observability_enabled` | `bool`            | Enable OpenTelemetry per-request tracing.                                                                   | `False`                                         |

Source code in `src/holodeck/serve/server.py`

```
def __init__(
    self,
    agent_config: Agent,
    protocol: ProtocolType = ProtocolType.AG_UI,
    host: str = "127.0.0.1",
    port: int = 8000,
    cors_origins: list[str] | None = None,
    debug: bool = False,
    execution_config: ExecutionConfig | None = None,
    observability_enabled: bool = False,
    max_concurrent_init_jobs: int = 4,
) -> None:
    """Initialize the agent server.

    Args:
        agent_config: The agent configuration to serve.
        protocol: The protocol to use (default: AG-UI).
        host: The hostname to bind to (default: 127.0.0.1 for security).
              Use 0.0.0.0 to expose to all network interfaces.
        port: The port to listen on (default: 8000).
        cors_origins: List of allowed CORS origins (default: ["*"]).
        debug: Enable debug logging (default: False).
        execution_config: Resolved execution configuration for timeouts.
        observability_enabled: Enable OpenTelemetry per-request tracing.
    """
    self.agent_config = agent_config
    self.protocol = protocol
    self.host = host
    self.port = port
    self.cors_origins = cors_origins or ["*"]
    self.debug = debug
    self.execution_config = execution_config
    self.observability_enabled = observability_enabled

    # Warn if binding to all interfaces
    if host == "0.0.0.0":  # noqa: S104  # nosec B104
        logger.warning(
            "Server binding to 0.0.0.0 exposes it to all network interfaces. "
            "Use 127.0.0.1 for local-only access."
        )

    # Claude SDK's anyio task group binds to the task that called
    # connect(). HTTP requests run in different tasks, so the
    # executor must release transport after each turn and reconnect
    # on the next request (session_id preserves conversation state).
    self._release_transport = (
        self.agent_config.model.provider == ProviderEnum.ANTHROPIC
    )

    # Spec 034 P4 changed the binding constraint: the SDK subprocess
    # is now spawned per *turn* (not per session), so memory-bounded
    # capacity must gate concurrent active turns rather than open
    # session count. We derive the active-turn cap from the cgroup
    # memory limit using the same formula previously used for the
    # session cap — only the semantic shifted.
    self._active_turn_cap: int | None = None
    self._active_turn_semaphore: asyncio.BoundedSemaphore | None = None
    if self.agent_config.model.provider == ProviderEnum.ANTHROPIC:
        claude = self.agent_config.claude
        explicit_cap = claude.max_concurrent_sessions if claude else None
        per_turn_mib = claude.session_memory_estimate_mib if claude else 200
        per_turn_bytes = per_turn_mib * 1024 * 1024
        mem_bytes = memory_limit_bytes()
        logger.info(
            "Claude cgroup memory limit: %s",
            f"{mem_bytes} bytes" if mem_bytes is not None else "unbounded",
        )
        if explicit_cap is not None:
            self._active_turn_cap = explicit_cap
            logger.info(
                "Claude active-turn cap: %d (explicit from "
                "claude.max_concurrent_sessions)",
                self._active_turn_cap,
            )
        elif mem_bytes is not None:
            self._active_turn_cap = derived_session_cap_from_memory(
                mem_bytes, per_session_bytes=per_turn_bytes
            )
            logger.info(
                "Claude active-turn cap: %d (derived from %d MiB memory "
                "limit @ %d MiB/turn)",
                self._active_turn_cap,
                mem_bytes // (1024 * 1024),
                per_turn_mib,
            )
        else:
            self._active_turn_cap = _FALLBACK_ACTIVE_TURN_CAP
            logger.info(
                "Claude active-turn cap: %d (fallback — no cgroup memory "
                "limit detected)",
                self._active_turn_cap,
            )
        self._active_turn_semaphore = asyncio.BoundedSemaphore(
            self._active_turn_cap
        )
    self.sessions = SessionStore(max_sessions=_OPEN_SESSION_CEILING)
    self._tool_init_manager = ToolInitManager(
        agent=self.agent_config, max_concurrent=max_concurrent_init_jobs
    )
    self.state = ServerState.INITIALIZING
    self._app: FastAPI | None = None
    self._start_time: datetime | None = None
```

### `is_ready`

Check if the server is ready to accept requests.

### `uptime_seconds`

Return server uptime in seconds.

### `create_app()`

Create and configure the FastAPI application.

Returns:

| Type      | Description                              |
| --------- | ---------------------------------------- |
| `FastAPI` | Configured FastAPI application instance. |

Source code in `src/holodeck/serve/server.py`

```
def create_app(self) -> FastAPI:
    """Create and configure the FastAPI application.

    Returns:
        Configured FastAPI application instance.
    """
    agent_name = self.agent_config.name
    protocol_name = self.protocol.value
    app = FastAPI(
        title=f"HoloDeck Agent: {agent_name}",
        description=f"Agent Local Server exposing {agent_name} via {protocol_name}",
        version="0.1.0",
        docs_url="/docs" if self.protocol == ProtocolType.REST else None,
        redoc_url="/redoc" if self.protocol == ProtocolType.REST else None,
    )

    # Add CORS middleware
    app.add_middleware(
        CORSMiddleware,
        allow_origins=self.cors_origins,
        allow_credentials=True,
        allow_methods=["*"],
        allow_headers=["*"],
    )

    # Add custom middleware
    # Middleware order matters: Starlette executes in reverse order of addition.
    # Request flow:  Logging -> ErrorHandling -> CORS -> Handler
    # Response flow: Handler -> CORS -> ErrorHandling -> Logging
    #
    # This order ensures:
    # 1. LoggingMiddleware logs all requests/responses including error responses
    # 2. ErrorHandlingMiddleware catches handler exceptions and returns RFC 7807
    # 3. CORS headers are added to all responses including errors
    app.add_middleware(ErrorHandlingMiddleware, debug=self.debug)
    app.add_middleware(
        LoggingMiddleware,
        debug=self.debug,
        observability_enabled=self.observability_enabled,
    )

    # Register health endpoints
    self._register_health_endpoints(app)

    # Register tool init endpoints (protocol-agnostic)
    from holodeck.serve.tool_init_routes import router as tool_init_router

    app.state.tool_init_manager = self._tool_init_manager
    app.include_router(tool_init_router)

    # Register protocol-specific endpoints
    if self.protocol == ProtocolType.AG_UI:
        self._register_agui_endpoints(app)
    elif self.protocol == ProtocolType.REST:
        self._register_rest_endpoints(app)

    # Store reference
    self._app = app
    self.state = ServerState.READY

    logger.info(
        f"FastAPI app created for agent '{self.agent_config.name}' "
        f"with {self.protocol.value} protocol"
    )

    return app
```

### `start()`

Start the server and begin accepting requests.

This method should be called after create_app() to transition the server to the RUNNING state. Also starts the background session cleanup task.

Source code in `src/holodeck/serve/server.py`

```
async def start(self) -> None:
    """Start the server and begin accepting requests.

    This method should be called after create_app() to transition
    the server to the RUNNING state. Also starts the background
    session cleanup task.
    """
    if self._app is None:
        self.create_app()

    # Validate backend prerequisites before accepting requests
    await self._validate_backend_prerequisites()

    # Clean up orphaned temp directories from previous runs
    from holodeck.lib.source_resolver import SourceResolver

    orphans_removed = await SourceResolver.cleanup_orphans()
    if orphans_removed:
        logger.info(
            "Cleaned up %d orphaned init temp directories",
            orphans_removed,
        )

    self._start_time = datetime.now(timezone.utc)
    self.state = ServerState.RUNNING

    # Start automatic session cleanup
    await self.sessions.start_cleanup_task()

    logger.info(
        f"Agent server started at http://{self.host}:{self.port} "
        f"serving agent '{self.agent_config.name}'"
    )
```

### `stop()`

Stop the server gracefully.

Transitions through SHUTTING_DOWN to STOPPED state, stopping the cleanup task and clearing all sessions.

Source code in `src/holodeck/serve/server.py`

```
async def stop(self) -> None:
    """Stop the server gracefully.

    Transitions through SHUTTING_DOWN to STOPPED state,
    stopping the cleanup task and clearing all sessions.
    """
    self.state = ServerState.SHUTTING_DOWN

    # Shutdown tool init manager (cancel running jobs)
    await self._tool_init_manager.shutdown()

    # Stop cleanup task
    await self.sessions.stop_cleanup_task()

    # Shutdown all session executors (stops actor tasks, SDK subprocesses)
    session_count = self.sessions.active_count
    session_ids = list(self.sessions.sessions.keys())
    if session_ids:
        await asyncio.gather(
            *(self.sessions.delete(sid) for sid in session_ids),
            return_exceptions=True,
        )

    self.state = ServerState.STOPPED

    logger.info(f"Agent server stopped. Cleaned up {session_count} sessions.")
```

______________________________________________________________________

## Models

Pydantic request/response models shared across both AG-UI and REST protocols, plus health-check and error-response schemas.

### ProtocolType

## `ProtocolType`

Bases: `str`, `Enum`

Protocol types supported by the Agent Local Server.

### ServerState

## `ServerState`

Bases: `str`, `Enum`

Server lifecycle states.

### FileContent

## `FileContent`

Bases: `BaseModel`

Binary file content for multimodal inputs.

Files are base64-encoded for JSON transport. Supported file types include images, PDFs, Office documents, and text files.

### `validate_mime_type(v)`

Validate MIME type is supported.

Source code in `src/holodeck/serve/models.py`

```
@field_validator("mime_type")
@classmethod
def validate_mime_type(cls, v: str) -> str:
    """Validate MIME type is supported."""
    if v not in SUPPORTED_MIME_TYPES:
        raise ValueError(f"Unsupported MIME type: {v}")
    return v
```

### `validate_base64(v)`

Validate content is valid base64.

Source code in `src/holodeck/serve/models.py`

```
@field_validator("content")
@classmethod
def validate_base64(cls, v: str) -> str:
    """Validate content is valid base64."""
    try:
        base64.b64decode(v, validate=True)
    except Exception:
        raise ValueError("content must be valid base64") from None
    return v
```

### ChatRequest

## `ChatRequest`

Bases: `BaseModel`

Request payload for chat endpoints.

Used by both synchronous and streaming chat endpoints in the REST protocol.

### `message_not_blank(v)`

Validate message is not just whitespace.

Source code in `src/holodeck/serve/models.py`

```
@field_validator("message")
@classmethod
def message_not_blank(cls, v: str) -> str:
    """Validate message is not just whitespace."""
    if not v.strip():
        raise ValueError("message cannot be blank")
    return v
```

### `valid_ulid(v)`

Validate session_id is valid ULID format if provided.

Source code in `src/holodeck/serve/models.py`

```
@field_validator("session_id")
@classmethod
def valid_ulid(cls, v: str | None) -> str | None:
    """Validate session_id is valid ULID format if provided."""
    if v is not None:
        try:
            ULID.from_str(v)
        except (ValueError, TypeError):
            raise ValueError("session_id must be valid ULID") from None
    return v
```

### ChatResponse

## `ChatResponse`

Bases: `BaseModel`

Response payload for synchronous chat endpoint.

Contains the agent's response, tool calls, and execution metadata.

### `valid_message_id_ulid(v)`

Validate message_id is valid ULID format.

Source code in `src/holodeck/serve/models.py`

```
@field_validator("message_id")
@classmethod
def valid_message_id_ulid(cls, v: str) -> str:
    """Validate message_id is valid ULID format."""
    try:
        ULID.from_str(v)
    except (ValueError, TypeError):
        raise ValueError("message_id must be valid ULID") from None
    return v
```

### `valid_session_id_ulid(v)`

Validate session_id is valid ULID format.

Source code in `src/holodeck/serve/models.py`

```
@field_validator("session_id")
@classmethod
def valid_session_id_ulid(cls, v: str) -> str:
    """Validate session_id is valid ULID format."""
    try:
        ULID.from_str(v)
    except (ValueError, TypeError):
        raise ValueError("session_id must be valid ULID") from None
    return v
```

### ToolCallInfo

## `ToolCallInfo`

Bases: `BaseModel`

Tool execution information in response.

Contains details about a tool call made by the agent during message processing.

### HealthResponse

## `HealthResponse`

Bases: `BaseModel`

Response for health check endpoints.

Provides server and agent status information.

### ProblemDetail

## `ProblemDetail`

Bases: `BaseModel`

RFC 7807 Problem Details error response.

Standard format for HTTP API error responses.

### SUPPORTED_MIME_TYPES

## `SUPPORTED_MIME_TYPES = {'image/png', 'image/jpeg', 'image/gif', 'image/webp', 'application/pdf', 'application/vnd.openxmlformats-officedocument.wordprocessingml.document', 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet', 'application/vnd.openxmlformats-officedocument.presentationml.presentation', 'text/plain', 'text/csv', 'text/markdown'}`

______________________________________________________________________

## Middleware

Cross-cutting concerns for the FastAPI application: structured logging with optional OpenTelemetry tracing, and RFC 7807 error handling.

### LoggingMiddleware

## `LoggingMiddleware(app, debug=False, observability_enabled=False)`

Bases: `BaseHTTPMiddleware`

Middleware for request/response logging and tracing.

Captures request metadata including timestamp, endpoint, session ID, and latency for each request. When observability is enabled, creates per-request trace spans with HTTP attributes.

Initialize logging middleware.

Parameters:

| Name                    | Type       | Description                                              | Default    |
| ----------------------- | ---------- | -------------------------------------------------------- | ---------- |
| `app`                   | `Callable` | The ASGI application.                                    | *required* |
| `debug`                 | `bool`     | Enable verbose logging of full request/response content. | `False`    |
| `observability_enabled` | `bool`     | Enable OpenTelemetry per-request tracing.                | `False`    |

Source code in `src/holodeck/serve/middleware.py`

```
def __init__(
    self, app: Callable, debug: bool = False, observability_enabled: bool = False
) -> None:
    """Initialize logging middleware.

    Args:
        app: The ASGI application.
        debug: Enable verbose logging of full request/response content.
        observability_enabled: Enable OpenTelemetry per-request tracing.
    """
    super().__init__(app)
    self.debug = debug
    self.observability_enabled = observability_enabled
```

### `dispatch(request, call_next)`

Process request, log metadata, and create trace span if enabled.

Parameters:

| Name        | Type              | Description                               | Default    |
| ----------- | ----------------- | ----------------------------------------- | ---------- |
| `request`   | `Request`         | The incoming HTTP request.                | *required* |
| `call_next` | `RequestCallNext` | The next middleware/handler in the chain. | *required* |

Returns:

| Type       | Description        |
| ---------- | ------------------ |
| `Response` | The HTTP response. |

Source code in `src/holodeck/serve/middleware.py`

```
async def dispatch(self, request: Request, call_next: RequestCallNext) -> Response:
    """Process request, log metadata, and create trace span if enabled.

    Args:
        request: The incoming HTTP request.
        call_next: The next middleware/handler in the chain.

    Returns:
        The HTTP response.
    """
    start_time = time.perf_counter()

    # Extract session ID from various sources
    session_id = self._extract_session_id(request)

    # Create span context for per-request tracing
    if self.observability_enabled:
        from opentelemetry import trace

        from holodeck.lib.observability import get_tracer

        tracer = get_tracer(__name__)
        span_context: Any = tracer.start_as_current_span(
            "holodeck.serve.request",
            kind=trace.SpanKind.SERVER,
        )
    else:
        span_context = nullcontext()

    with span_context as span:
        # Set initial span attributes if span exists
        if span:
            span.set_attribute("http.method", request.method)
            span.set_attribute("http.route", request.url.path)
            if session_id:
                span.set_attribute("session.id", session_id)

        # Log request
        logger.info(
            "Request started",
            extra={
                "method": request.method,
                "path": request.url.path,
                "session_id": session_id,
            },
        )

        if self.debug:
            logger.debug(
                "Request details",
                extra={
                    "headers": dict(request.headers),
                    "query_params": dict(request.query_params),
                },
            )

        # Process request
        response = await call_next(request)

        # Set response status on span
        if span:
            span.set_attribute("http.status_code", response.status_code)

        # Calculate latency
        latency_ms = (time.perf_counter() - start_time) * 1000

        # Log response
        logger.info(
            "Request completed",
            extra={
                "method": request.method,
                "path": request.url.path,
                "status_code": response.status_code,
                "latency_ms": round(latency_ms, 2),
                "session_id": session_id,
            },
        )

        return response
```

### ErrorHandlingMiddleware

## `ErrorHandlingMiddleware(app, debug=False)`

Bases: `BaseHTTPMiddleware`

Middleware for standardized error handling.

Catches unhandled exceptions and returns RFC 7807 Problem Details formatted error responses.

Initialize error handling middleware.

Parameters:

| Name    | Type       | Description                              | Default    |
| ------- | ---------- | ---------------------------------------- | ---------- |
| `app`   | `Callable` | The ASGI application.                    | *required* |
| `debug` | `bool`     | Include stack traces in error responses. | `False`    |

Source code in `src/holodeck/serve/middleware.py`

```
def __init__(self, app: Callable, debug: bool = False) -> None:
    """Initialize error handling middleware.

    Args:
        app: The ASGI application.
        debug: Include stack traces in error responses.
    """
    super().__init__(app)
    self.debug = debug
```

### `dispatch(request, call_next)`

Process request and handle errors.

Parameters:

| Name        | Type              | Description                               | Default    |
| ----------- | ----------------- | ----------------------------------------- | ---------- |
| `request`   | `Request`         | The incoming HTTP request.                | *required* |
| `call_next` | `RequestCallNext` | The next middleware/handler in the chain. | *required* |

Returns:

| Type       | Description                                           |
| ---------- | ----------------------------------------------------- |
| `Response` | The HTTP response, or an error response on exception. |

Source code in `src/holodeck/serve/middleware.py`

```
async def dispatch(self, request: Request, call_next: RequestCallNext) -> Response:
    """Process request and handle errors.

    Args:
        request: The incoming HTTP request.
        call_next: The next middleware/handler in the chain.

    Returns:
        The HTTP response, or an error response on exception.
    """
    try:
        return await call_next(request)
    except Exception as e:
        return self._create_error_response(request, e)
```

______________________________________________________________________

## Session Store

In-memory session storage with TTL-based expiration. Sessions maintain conversation context (via `AgentExecutor`) across multiple HTTP requests.

### ServerSession

## `ServerSession(agent_executor, session_id=(lambda: str(ULID()))(), created_at=(lambda: datetime.now(timezone.utc))(), last_activity=(lambda: datetime.now(timezone.utc))(), message_count=0)`

Individual conversation session with an agent.

Maintains state for a single conversation, including the agent executor instance that preserves conversation history.

Attributes:

| Name             | Type            | Description                                        |
| ---------------- | --------------- | -------------------------------------------------- |
| `session_id`     | `str`           | Unique identifier in ULID format.                  |
| `agent_executor` | `AgentExecutor` | Agent execution context with conversation history. |
| `created_at`     | `datetime`      | UTC timestamp when session was created.            |
| `last_activity`  | `datetime`      | UTC timestamp of last request in session.          |
| `message_count`  | `int`           | Number of messages exchanged in session.           |

### SessionStore

## `SessionStore(ttl_seconds=1800, max_sessions=1000, cleanup_interval_seconds=300)`

In-memory session storage with TTL-based cleanup.

Manages conversation sessions for the Agent Local Server. Sessions expire after a configurable TTL period of inactivity. Includes optional automatic background cleanup and max session limits.

Attributes:

| Name                       | Type                       | Description                                                 |
| -------------------------- | -------------------------- | ----------------------------------------------------------- |
| `sessions`                 | `dict[str, ServerSession]` | Dictionary mapping session IDs to ServerSession objects.    |
| `ttl_seconds`              |                            | Time-to-live for sessions in seconds (default: 30 minutes). |
| `max_sessions`             |                            | Maximum number of sessions allowed (default: 1000).         |
| `cleanup_interval_seconds` |                            | Interval for automatic cleanup (default: 300).              |

Initialize session store.

Parameters:

| Name                       | Type  | Description                                                  | Default |
| -------------------------- | ----- | ------------------------------------------------------------ | ------- |
| `ttl_seconds`              | `int` | Session timeout in seconds. Default is 1800 (30 minutes).    | `1800`  |
| `max_sessions`             | `int` | Maximum sessions before rejecting new ones. Default is 1000. | `1000`  |
| `cleanup_interval_seconds` | `int` | Interval for auto-cleanup. Default is 300 (5 min).           | `300`   |

Source code in `src/holodeck/serve/session_store.py`

```
def __init__(
    self,
    ttl_seconds: int = 1800,
    max_sessions: int = 1000,
    cleanup_interval_seconds: int = 300,
) -> None:
    """Initialize session store.

    Args:
        ttl_seconds: Session timeout in seconds. Default is 1800 (30 minutes).
        max_sessions: Maximum sessions before rejecting new ones. Default is 1000.
        cleanup_interval_seconds: Interval for auto-cleanup. Default is 300 (5 min).
    """
    self.sessions: dict[str, ServerSession] = {}
    self.ttl_seconds = ttl_seconds
    self.max_sessions = max_sessions
    self.cleanup_interval_seconds = cleanup_interval_seconds
    self._cleanup_task: asyncio.Task[None] | None = None
```

### `active_count`

Return count of active sessions.

### `get(session_id)`

Retrieve a session by ID.

Parameters:

| Name         | Type  | Description                        | Default    |
| ------------ | ----- | ---------------------------------- | ---------- |
| `session_id` | `str` | The session identifier to look up. | *required* |

Returns:

| Type            | Description |
| --------------- | ----------- |
| \`ServerSession | None\`      |

Source code in `src/holodeck/serve/session_store.py`

```
def get(self, session_id: str) -> ServerSession | None:
    """Retrieve a session by ID.

    Args:
        session_id: The session identifier to look up.

    Returns:
        The ServerSession if found, None otherwise.
    """
    return self.sessions.get(session_id)
```

### `get_all()`

Retrieve all active sessions.

Returns:

| Type                  | Description                               |
| --------------------- | ----------------------------------------- |
| `list[ServerSession]` | List of all active ServerSession objects. |

Source code in `src/holodeck/serve/session_store.py`

```
def get_all(self) -> list[ServerSession]:
    """Retrieve all active sessions.

    Returns:
        List of all active ServerSession objects.
    """
    return list(self.sessions.values())
```

### `create(agent_executor, session_id=None)`

Create a new session with the given agent executor.

Parameters:

| Name             | Type            | Description                                  | Default                                                                                                                                        |
| ---------------- | --------------- | -------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `agent_executor` | `AgentExecutor` | The AgentExecutor instance for this session. | *required*                                                                                                                                     |
| `session_id`     | \`str           | None\`                                       | Optional custom session ID. If not provided, a new ULID will be generated. Useful for mapping external IDs (like AG-UI thread_id) to sessions. |

Returns:

| Type            | Description                      |
| --------------- | -------------------------------- |
| `ServerSession` | The newly created ServerSession. |

Raises:

| Type           | Description                       |
| -------------- | --------------------------------- |
| `RuntimeError` | If max_sessions limit is reached. |
| `ValueError`   | If session_id already exists.     |

Source code in `src/holodeck/serve/session_store.py`

```
def create(
    self,
    agent_executor: AgentExecutor,
    session_id: str | None = None,
) -> ServerSession:
    """Create a new session with the given agent executor.

    Args:
        agent_executor: The AgentExecutor instance for this session.
        session_id: Optional custom session ID. If not provided, a new
            ULID will be generated. Useful for mapping external IDs
            (like AG-UI thread_id) to sessions.

    Returns:
        The newly created ServerSession.

    Raises:
        RuntimeError: If max_sessions limit is reached.
        ValueError: If session_id already exists.
    """
    if len(self.sessions) >= self.max_sessions:
        raise RuntimeError(
            f"Maximum session limit ({self.max_sessions}) reached. "
            "Try again later or increase max_sessions."
        )

    if session_id is not None and session_id in self.sessions:
        raise ValueError(f"Session with ID '{session_id}' already exists")

    # Pass session_id directly to constructor to avoid mutation
    if session_id is not None:
        session = ServerSession(
            agent_executor=agent_executor, session_id=session_id
        )
    else:
        session = ServerSession(agent_executor=agent_executor)

    self.sessions[session.session_id] = session
    return session
```

### `delete(session_id)`

Delete a session by ID, shutting down its executor.

Parameters:

| Name         | Type  | Description                       | Default    |
| ------------ | ----- | --------------------------------- | ---------- |
| `session_id` | `str` | The session identifier to delete. | *required* |

Returns:

| Type   | Description                                      |
| ------ | ------------------------------------------------ |
| `bool` | True if session was deleted, False if not found. |

Source code in `src/holodeck/serve/session_store.py`

```
async def delete(self, session_id: str) -> bool:
    """Delete a session by ID, shutting down its executor.

    Args:
        session_id: The session identifier to delete.

    Returns:
        True if session was deleted, False if not found.
    """
    session = self.sessions.pop(session_id, None)
    if session is None:
        return False
    try:
        await asyncio.wait_for(
            session.agent_executor.shutdown(),
            timeout=self._SHUTDOWN_TIMEOUT,
        )
    except asyncio.TimeoutError:
        logger.warning(
            "Executor shutdown timed out for session %s after %.1fs",
            session_id,
            self._SHUTDOWN_TIMEOUT,
        )
    except Exception as e:
        logger.warning(
            "Error shutting down executor for session %s: %s",
            session_id,
            e,
        )
    return True
```

### `touch(session_id)`

Update the last_activity timestamp for a session.

This should be called on each request to prevent session expiration.

Parameters:

| Name         | Type  | Description                       | Default    |
| ------------ | ----- | --------------------------------- | ---------- |
| `session_id` | `str` | The session identifier to update. | *required* |

Source code in `src/holodeck/serve/session_store.py`

```
def touch(self, session_id: str) -> None:
    """Update the last_activity timestamp for a session.

    This should be called on each request to prevent session expiration.

    Args:
        session_id: The session identifier to update.
    """
    session = self.sessions.get(session_id)
    if session:
        session.last_activity = datetime.now(timezone.utc)
```

### `cleanup_expired()`

Remove all expired sessions, shutting down their executors.

Sessions are considered expired if their last_activity timestamp is older than the configured TTL.

Returns:

| Type  | Description                 |
| ----- | --------------------------- |
| `int` | Number of sessions removed. |

Source code in `src/holodeck/serve/session_store.py`

```
async def cleanup_expired(self) -> int:
    """Remove all expired sessions, shutting down their executors.

    Sessions are considered expired if their last_activity timestamp
    is older than the configured TTL.

    Returns:
        Number of sessions removed.
    """
    now = datetime.now(timezone.utc)
    cutoff = now - timedelta(seconds=self.ttl_seconds)

    expired_ids = [
        session_id
        for session_id, session in self.sessions.items()
        if session.last_activity < cutoff
    ]

    await asyncio.gather(
        *(self.delete(sid) for sid in expired_ids),
        return_exceptions=True,
    )

    return len(expired_ids)
```

### `start_cleanup_task()`

Start the background cleanup task.

This should be called when the server starts to enable automatic cleanup of expired sessions.

Source code in `src/holodeck/serve/session_store.py`

```
async def start_cleanup_task(self) -> None:
    """Start the background cleanup task.

    This should be called when the server starts to enable
    automatic cleanup of expired sessions.
    """
    if self._cleanup_task is None or self._cleanup_task.done():
        self._cleanup_task = asyncio.create_task(self._cleanup_loop())
        logger.info(
            f"Started session cleanup task (interval: "
            f"{self.cleanup_interval_seconds}s, TTL: {self.ttl_seconds}s)"
        )
```

### `stop_cleanup_task()`

Stop the background cleanup task.

This should be called when the server stops to cleanly terminate the background task.

Source code in `src/holodeck/serve/session_store.py`

```
async def stop_cleanup_task(self) -> None:
    """Stop the background cleanup task.

    This should be called when the server stops to cleanly
    terminate the background task.
    """
    if self._cleanup_task is not None and not self._cleanup_task.done():
        self._cleanup_task.cancel()
        with contextlib.suppress(asyncio.CancelledError):
            await self._cleanup_task
        logger.info("Stopped session cleanup task")
    self._cleanup_task = None
```

______________________________________________________________________

## File Utilities

Shared utilities for multimodal content processing across REST and AG-UI protocols, including MIME-type mappings, temporary file management, and binary content extraction.

### Constants

## `MAX_FILE_SIZE_MB = 50`

## `MAX_TOTAL_SIZE_MB = 100`

## `MIME_TO_FILE_TYPE = {'image/png': 'image', 'image/jpeg': 'image', 'image/gif': 'image', 'image/webp': 'image', 'application/pdf': 'pdf', _WORD_MIME: 'word', _EXCEL_MIME: 'excel', _PPTX_MIME: 'powerpoint', 'text/plain': 'text', 'text/csv': 'csv', 'text/markdown': 'text'}`

## `MIME_TO_EXTENSION = {'image/png': '.png', 'image/jpeg': '.jpg', 'image/gif': '.gif', 'image/webp': '.webp', 'application/pdf': '.pdf', _WORD_MIME: '.docx', _EXCEL_MIME: '.xlsx', _PPTX_MIME: '.pptx', 'text/plain': '.txt', 'text/csv': '.csv', 'text/markdown': '.md'}`

### Functions

## `create_temp_file_from_bytes(content_bytes, mime_type, description=None)`

Create a temporary file from binary content and return FileInput.

Parameters:

| Name            | Type    | Description                           | Default                                |
| --------------- | ------- | ------------------------------------- | -------------------------------------- |
| `content_bytes` | `bytes` | Binary content to write to temp file. | *required*                             |
| `mime_type`     | `str`   | MIME type of the content.             | *required*                             |
| `description`   | \`str   | None\`                                | Optional description (e.g., filename). |

Returns:

| Type        | Description                                          |
| ----------- | ---------------------------------------------------- |
| `FileInput` | FileInput suitable for FileProcessor.process_file(). |

Source code in `src/holodeck/serve/file_utils.py`

```
def create_temp_file_from_bytes(
    content_bytes: bytes,
    mime_type: str,
    description: str | None = None,
) -> FileInput:
    """Create a temporary file from binary content and return FileInput.

    Args:
        content_bytes: Binary content to write to temp file.
        mime_type: MIME type of the content.
        description: Optional description (e.g., filename).

    Returns:
        FileInput suitable for FileProcessor.process_file().
    """
    file_type = MIME_TO_FILE_TYPE.get(mime_type, "text")
    extension = MIME_TO_EXTENSION.get(mime_type, ".bin")

    with tempfile.NamedTemporaryFile(suffix=extension, delete=False, mode="wb") as tmp:
        tmp.write(content_bytes)
        tmp_path = tmp.name

    logger.debug(
        "Created temp file: %s (type=%s, size=%d bytes)",
        tmp_path,
        file_type,
        len(content_bytes),
    )

    return FileInput(
        path=tmp_path,
        url=None,
        type=file_type,
        description=description,
        pages=None,
        sheet=None,
        range=None,
        cache=None,
    )
```

## `convert_file_content_to_file_input(file_content)`

Convert REST FileContent (base64) to FileProcessor-compatible FileInput.

Creates a temporary file with the decoded content and returns a FileInput pointing to it.

Parameters:

| Name           | Type          | Description                                         | Default    |
| -------------- | ------------- | --------------------------------------------------- | ---------- |
| `file_content` | `FileContent` | FileContent with base64-encoded data and MIME type. | *required* |

Returns:

| Type        | Description                                          |
| ----------- | ---------------------------------------------------- |
| `FileInput` | FileInput suitable for FileProcessor.process_file(). |

Source code in `src/holodeck/serve/file_utils.py`

```
def convert_file_content_to_file_input(file_content: FileContent) -> FileInput:
    """Convert REST FileContent (base64) to FileProcessor-compatible FileInput.

    Creates a temporary file with the decoded content and returns a FileInput
    pointing to it.

    Args:
        file_content: FileContent with base64-encoded data and MIME type.

    Returns:
        FileInput suitable for FileProcessor.process_file().
    """
    content_bytes = base64.b64decode(file_content.content)
    return create_temp_file_from_bytes(
        content_bytes=content_bytes,
        mime_type=file_content.mime_type,
        description=file_content.filename,
    )
```

## `convert_binary_dict_to_file_input(binary_content)`

Convert AG-UI binary content dict to FileProcessor-compatible FileInput.

Handles two transport options:

- data: Inline base64-encoded content (supported)
- url: Remote URL reference (NOT supported - SSRF security risk)
- id: File ID reference (NOT supported)

Note: URL downloads are intentionally disabled to prevent SSRF attacks. Clients must provide file content inline as base64-encoded data.

Parameters:

| Name             | Type             | Description                               | Default    |
| ---------------- | ---------------- | ----------------------------------------- | ---------- |
| `binary_content` | `dict[str, Any]` | Dict with type, mimeType, and data field. | *required* |

Returns:

| Type        | Description |
| ----------- | ----------- |
| \`FileInput | None\`      |

Raises:

| Type         | Description               |
| ------------ | ------------------------- |
| `ValueError` | If base64 decoding fails. |

Source code in `src/holodeck/serve/file_utils.py`

```
def convert_binary_dict_to_file_input(
    binary_content: dict[str, Any],
) -> FileInput | None:
    """Convert AG-UI binary content dict to FileProcessor-compatible FileInput.

    Handles two transport options:
    - data: Inline base64-encoded content (supported)
    - url: Remote URL reference (NOT supported - SSRF security risk)
    - id: File ID reference (NOT supported)

    Note: URL downloads are intentionally disabled to prevent SSRF attacks.
    Clients must provide file content inline as base64-encoded data.

    Args:
        binary_content: Dict with type, mimeType, and data field.

    Returns:
        FileInput suitable for FileProcessor, or None if not processable.

    Raises:
        ValueError: If base64 decoding fails.
    """
    mime_type = binary_content.get("mimeType", "")
    filename = binary_content.get("filename")

    logger.debug(
        "Converting binary content to FileInput: mime=%s, filename=%s",
        mime_type,
        filename,
    )

    # Handle inline base64 data
    if binary_content.get("data"):
        data = binary_content["data"]
        logger.debug("Processing base64 data (length=%d chars)", len(data))
        try:
            content_bytes = base64.b64decode(data)
        except (ValueError, binascii.Error) as e:
            logger.error("Base64 decode failed: %s", e, exc_info=True)
            raise ValueError(f"Invalid base64 data: {e}") from e

        return create_temp_file_from_bytes(
            content_bytes=content_bytes,
            mime_type=mime_type,
            description=filename,
        )

    # Handle URL reference (DISABLED for security - SSRF risk)
    if binary_content.get("url"):
        url = binary_content["url"]
        logger.warning(
            "URL file references are disabled for security (SSRF prevention). "
            "Please provide file content as inline base64 data instead. url=%s",
            url,
        )
        return None

    # Handle file ID reference (not supported)
    if binary_content.get("id"):
        file_id = binary_content["id"]
        logger.warning(
            "File ID references are not supported. "
            "Use inline base64 data instead. id=%s",
            file_id,
        )
        return None

    logger.warning(
        "Binary content has no data, url, or id field - skipping. Content keys: %s",
        list(binary_content.keys()),
    )
    return None
```

## `cleanup_temp_file(file_input)`

Clean up temporary file created during file conversion.

Parameters:

| Name         | Type        | Description                            | Default    |
| ------------ | ----------- | -------------------------------------- | ---------- |
| `file_input` | `FileInput` | FileInput with path to temporary file. | *required* |

Source code in `src/holodeck/serve/file_utils.py`

```
def cleanup_temp_file(file_input: FileInput) -> None:
    """Clean up temporary file created during file conversion.

    Args:
        file_input: FileInput with path to temporary file.
    """
    if file_input.path:
        try:
            Path(file_input.path).unlink(missing_ok=True)
            logger.debug("Cleaned up temp file: %s", file_input.path)
        except Exception as e:
            logger.warning("Failed to cleanup temp file %s: %s", file_input.path, e)
```

## `cleanup_temp_files(file_inputs)`

Clean up multiple temporary files.

Parameters:

| Name          | Type              | Description                            | Default    |
| ------------- | ----------------- | -------------------------------------- | ---------- |
| `file_inputs` | `list[FileInput]` | List of FileInput objects to clean up. | *required* |

Source code in `src/holodeck/serve/file_utils.py`

```
def cleanup_temp_files(file_inputs: list[FileInput]) -> None:
    """Clean up multiple temporary files.

    Args:
        file_inputs: List of FileInput objects to clean up.
    """
    for file_input in file_inputs:
        cleanup_temp_file(file_input)
```

## `process_multimodal_files(files, execution_config=None, is_agui_format=False)`

Process multimodal files and return combined content with cleanup list.

This is the unified file processing function for both REST and AG-UI protocols.

Parameters:

| Name               | Type                | Description                                         | Default                                                             |
| ------------------ | ------------------- | --------------------------------------------------- | ------------------------------------------------------------------- |
| `files`            | \`list[FileContent] | list\[dict[str, Any]\]\`                            | List of FileContent objects (REST) or binary content dicts (AG-UI). |
| `execution_config` | \`ExecutionConfig   | None\`                                              | Optional execution configuration for FileProcessor.                 |
| `is_agui_format`   | `bool`              | If True, treat files as AG-UI binary content dicts. | `False`                                                             |

Returns:

| Type                          | Description                                                            |
| ----------------------------- | ---------------------------------------------------------------------- |
| `tuple[str, list[FileInput]]` | Tuple of (combined_markdown_content, list_of_file_inputs_for_cleanup). |

Source code in `src/holodeck/serve/file_utils.py`

```
def process_multimodal_files(
    files: list[FileContent] | list[dict[str, Any]],
    execution_config: ExecutionConfig | None = None,
    is_agui_format: bool = False,
) -> tuple[str, list[FileInput]]:
    """Process multimodal files and return combined content with cleanup list.

    This is the unified file processing function for both REST and AG-UI protocols.

    Args:
        files: List of FileContent objects (REST) or binary content dicts (AG-UI).
        execution_config: Optional execution configuration for FileProcessor.
        is_agui_format: If True, treat files as AG-UI binary content dicts.

    Returns:
        Tuple of (combined_markdown_content, list_of_file_inputs_for_cleanup).
    """
    from holodeck.lib.file_processor import FileProcessor

    if not files:
        return "", []

    # Create FileProcessor
    if execution_config:
        processor = FileProcessor.from_execution_config(execution_config)
    else:
        processor = FileProcessor()

    file_inputs: list[FileInput] = []
    file_contents: list[str] = []

    for idx, file_item in enumerate(files):
        file_input: FileInput | None = None
        filename: str | None = None

        try:
            # Convert to FileInput based on format
            if is_agui_format:
                # AG-UI binary content dict
                binary_dict: dict[str, Any] = file_item  # type: ignore[assignment]
                filename = binary_dict.get("filename")
                file_input = convert_binary_dict_to_file_input(binary_dict)
            else:
                # REST FileContent Pydantic model
                file_content: FileContent = file_item  # type: ignore[assignment]
                filename = file_content.filename
                file_input = convert_file_content_to_file_input(file_content)

            if file_input is None:
                logger.debug(
                    "File %d/%d: conversion returned None, skipping",
                    idx + 1,
                    len(files),
                )
                continue

            # Add to cleanup list immediately after creation
            file_inputs.append(file_input)

            # Process file
            result = processor.process_file(file_input)

            if result.error:
                logger.warning(
                    "File %d/%d processing error: %s",
                    idx + 1,
                    len(files),
                    result.error,
                )
                file_contents.append(f"[File processing error: {result.error}]")
            elif result.markdown_content:
                # Add filename header for AG-UI format
                if is_agui_format and filename:
                    file_contents.append(
                        f"## File: {filename}\n\n{result.markdown_content}"
                    )
                else:
                    file_contents.append(result.markdown_content)
            else:
                logger.debug(
                    "File %d/%d: FileProcessor returned no content",
                    idx + 1,
                    len(files),
                )

        except Exception as e:
            logger.error(
                "File %d/%d: exception during processing: %s",
                idx + 1,
                len(files),
                e,
                exc_info=True,
            )
            file_contents.append(f"[Error processing file: {e}]")

    return "\n\n".join(file_contents), file_inputs
```

## `extract_binary_parts_from_content(content)`

Extract binary content parts from AG-UI message content list.

Filters the content list for binary type parts and validates MIME types. Handles both dict format and AG-UI Pydantic objects (BinaryInputContent).

Parameters:

| Name      | Type                   | Description | Default                                           |
| --------- | ---------------------- | ----------- | ------------------------------------------------- |
| `content` | \`list\[dict[str, Any] | Any\]\`     | List of content parts (text, binary, or strings). |

Returns:

| Type                   | Description                                                               |
| ---------------------- | ------------------------------------------------------------------------- |
| `list[dict[str, Any]]` | List of binary content dicts with type, mimeType, and data/url/id fields. |

Source code in `src/holodeck/serve/file_utils.py`

```
def extract_binary_parts_from_content(
    content: list[dict[str, Any] | Any],
) -> list[dict[str, Any]]:
    """Extract binary content parts from AG-UI message content list.

    Filters the content list for binary type parts and validates MIME types.
    Handles both dict format and AG-UI Pydantic objects (BinaryInputContent).

    Args:
        content: List of content parts (text, binary, or strings).

    Returns:
        List of binary content dicts with type, mimeType, and data/url/id fields.
    """
    binary_parts: list[dict[str, Any]] = []

    logger.debug(
        "Scanning content list for binary parts (total items: %d)",
        len(content),
    )

    for idx, part in enumerate(content):
        # Handle dict format
        if isinstance(part, dict):
            if part.get("type") != "binary":
                continue

            mime_type = part.get("mimeType", "")
            if mime_type not in SUPPORTED_MIME_TYPES:
                logger.warning(
                    "Skipping binary content with unsupported MIME type: %s",
                    mime_type,
                )
                continue

            logger.debug(
                "Found binary content (dict): idx=%d, mime=%s, filename=%s",
                idx,
                mime_type,
                part.get("filename"),
            )
            binary_parts.append(part)

        # Handle AG-UI Pydantic object (BinaryInputContent)
        elif hasattr(part, "type") and getattr(part, "type", None) == "binary":
            # AG-UI uses 'mime_type' attribute (snake_case), not 'mimeType'
            mime_type = getattr(part, "mime_type", "")
            if mime_type not in SUPPORTED_MIME_TYPES:
                logger.warning(
                    "Skipping binary content with unsupported MIME type: %s",
                    mime_type,
                )
                continue

            # Convert Pydantic object to dict for consistent handling
            binary_parts.append(
                {
                    "type": "binary",
                    "mimeType": mime_type,
                    "data": getattr(part, "data", None),
                    "url": getattr(part, "url", None),
                    "id": getattr(part, "id", None),
                    "filename": getattr(part, "filename", None),
                }
            )
            logger.debug(
                "Found binary content (Pydantic): idx=%d, mime=%s, filename=%s",
                idx,
                mime_type,
                getattr(part, "filename", None),
            )

    logger.debug("Extracted %d binary parts from content", len(binary_parts))
    return binary_parts
```

______________________________________________________________________

## Protocols

### Base Protocol

Abstract base class that both AG-UI and REST protocol adapters implement.

## `Protocol`

Bases: `ABC`

Abstract base class for server protocols.

Both AG-UI and REST protocols implement this interface to handle incoming requests and stream responses back to clients.

### `name`

Return the protocol name.

Returns:

| Type  | Description                                         |
| ----- | --------------------------------------------------- |
| `str` | Protocol identifier string (e.g., 'ag-ui', 'rest'). |

### `content_type`

Return the content type for responses.

Returns:

| Type  | Description                                        |
| ----- | -------------------------------------------------- |
| `str` | MIME type string for response Content-Type header. |

### `handle_request(request, session)`

Handle incoming request and yield response chunks.

Parameters:

| Name      | Type            | Description                                        | Default    |
| --------- | --------------- | -------------------------------------------------- | ---------- |
| `request` | `Any`           | The incoming request (format depends on protocol). | *required* |
| `session` | `ServerSession` | The server session for this request.               | *required* |

Yields:

| Type                          | Description                                           |
| ----------------------------- | ----------------------------------------------------- |
| `AsyncGenerator[bytes, None]` | Response chunks as bytes for streaming to the client. |

Source code in `src/holodeck/serve/protocols/base.py`

```
@abstractmethod
def handle_request(
    self,
    request: Any,
    session: ServerSession,
) -> AsyncGenerator[bytes, None]:
    """Handle incoming request and yield response chunks.

    Args:
        request: The incoming request (format depends on protocol).
        session: The server session for this request.

    Yields:
        Response chunks as bytes for streaming to the client.
    """
    ...
```

### REST Protocol

REST protocol adapter providing synchronous and streaming (SSE) chat endpoints, plus multipart file upload support.

## `RESTProtocol`

Bases: `Protocol`

REST protocol implementation with sync and streaming endpoints.

Handles:

- Synchronous requests: handle_sync_request() → ChatResponse
- Streaming requests: handle_request() → AsyncGenerator[bytes, None] (SSE)

### `name`

Return the protocol name.

Returns:

| Type  | Description                 |
| ----- | --------------------------- |
| `str` | Protocol identifier string. |

### `content_type`

Return the content type for streaming responses.

Returns:

| Type  | Description                                        |
| ----- | -------------------------------------------------- |
| `str` | MIME type string for response Content-Type header. |

### `handle_request(request, session)`

Handle streaming request and generate SSE events.

Processes the ChatRequest, executes the agent, and yields encoded SSE events for streaming to the client.

Parameters:

| Name      | Type            | Description                        | Default    |
| --------- | --------------- | ---------------------------------- | ---------- |
| `request` | `Any`           | ChatRequest from client.           | *required* |
| `session` | `ServerSession` | Server session with AgentExecutor. | *required* |

Yields:

| Type                          | Description                  |
| ----------------------------- | ---------------------------- |
| `AsyncGenerator[bytes, None]` | Encoded SSE events as bytes. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
async def handle_request(
    self,
    request: Any,
    session: ServerSession,
) -> AsyncGenerator[bytes, None]:
    """Handle streaming request and generate SSE events.

    Processes the ChatRequest, executes the agent, and yields
    encoded SSE events for streaming to the client.

    Args:
        request: ChatRequest from client.
        session: Server session with AgentExecutor.

    Yields:
        Encoded SSE events as bytes.
    """
    start_time = time.time()
    message_id = str(ULID())
    chat_request: ChatRequest = request

    try:
        # 1. Emit stream_start event
        yield SSEEvent.stream_start(session.session_id, message_id).encode("utf-8")

        # 2. Execute agent
        logger.debug(f"Executing agent for session {session.session_id}")
        response = await session.agent_executor.execute_turn(chat_request.message)

        # 3. Emit tool call events (if any)
        for tool_exec in response.tool_executions:
            tool_call_id = str(ULID())
            logger.debug(f"Emitting tool call events for: {tool_exec.tool_name}")

            # Tool call start
            yield SSEEvent.tool_call_start(
                tool_call_id=tool_call_id,
                name=tool_exec.tool_name,
                message_id=message_id,
            ).encode("utf-8")

            # Tool call args (send full args at once for non-streaming)
            args_json = json.dumps(tool_exec.parameters)
            yield SSEEvent.tool_call_args(
                tool_call_id=tool_call_id,
                args_delta=args_json,
            ).encode("utf-8")

            # Tool call end
            status = "success" if tool_exec.status.value == "success" else "error"
            yield SSEEvent.tool_call_end(
                tool_call_id=tool_call_id,
                status=status,
            ).encode("utf-8")

        # 4. Emit message delta (send full content at once for non-streaming)
        if response.content:
            yield SSEEvent.message_delta(
                delta=response.content,
                message_id=message_id,
            ).encode("utf-8")

        # 5. Calculate execution time and tokens
        execution_time_ms = int((time.time() - start_time) * 1000)
        tokens_used = None
        if response.tokens_used:
            tokens_used = {
                "prompt_tokens": response.tokens_used.prompt_tokens,
                "completion_tokens": response.tokens_used.completion_tokens,
                "total_tokens": response.tokens_used.total_tokens,
            }

        # 6. Emit stream_end event
        yield SSEEvent.stream_end(
            message_id=message_id,
            tokens_used=tokens_used,
            execution_time_ms=execution_time_ms,
        ).encode("utf-8")

        logger.debug(
            f"Completed streaming request for session {session.session_id}"
        )

    except BackendSessionError as e:
        logger.error(f"Backend session error: {e}", exc_info=True)
        yield SSEEvent.error(
            type="backend_error",
            title="Backend Error",
            status=502,
            detail=(
                "Claude Agent SDK subprocess terminated "
                "unexpectedly. Start a new session to retry."
            ),
        ).encode("utf-8")

    except Exception as e:
        logger.error(f"Error processing request: {e}", exc_info=True)
        # Emit error event
        yield SSEEvent.error(
            type="about:blank",
            title="Agent Error",
            status=500,
            detail=str(e),
        ).encode("utf-8")
```

### `handle_sync_request(request, session)`

Handle synchronous request and return complete response.

Parameters:

| Name      | Type            | Description                        | Default    |
| --------- | --------------- | ---------------------------------- | ---------- |
| `request` | `ChatRequest`   | ChatRequest from client.           | *required* |
| `session` | `ServerSession` | Server session with AgentExecutor. | *required* |

Returns:

| Type           | Description                         |
| -------------- | ----------------------------------- |
| `ChatResponse` | ChatResponse with agent's response. |

Raises:

| Type        | Description               |
| ----------- | ------------------------- |
| `Exception` | If agent execution fails. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
async def handle_sync_request(
    self,
    request: ChatRequest,
    session: ServerSession,
) -> ChatResponse:
    """Handle synchronous request and return complete response.

    Args:
        request: ChatRequest from client.
        session: Server session with AgentExecutor.

    Returns:
        ChatResponse with agent's response.

    Raises:
        Exception: If agent execution fails.
    """
    start_time = time.time()
    message_id = str(ULID())

    logger.debug(f"Executing sync request for session {session.session_id}")

    # Execute agent
    response = await session.agent_executor.execute_turn(request.message)

    # Calculate execution time
    execution_time_ms = int((time.time() - start_time) * 1000)

    # Build tool_calls list
    tool_calls = []
    for tool_exec in response.tool_executions:
        status = "success" if tool_exec.status.value == "success" else "error"
        tool_calls.append(
            ToolCallInfo(
                name=tool_exec.tool_name,
                arguments=tool_exec.parameters,
                status=status,
            )
        )

    # Build tokens_used dict
    tokens_used = None
    if response.tokens_used:
        tokens_used = {
            "prompt_tokens": response.tokens_used.prompt_tokens,
            "completion_tokens": response.tokens_used.completion_tokens,
            "total_tokens": response.tokens_used.total_tokens,
        }

    logger.debug(
        f"Completed sync request for session {session.session_id} "
        f"in {execution_time_ms}ms"
    )

    return ChatResponse(
        message_id=message_id,
        content=response.content,
        session_id=session.session_id,
        tool_calls=tool_calls,
        tokens_used=tokens_used,
        execution_time_ms=execution_time_ms,
    )
```

### `process_files(files, execution_config=None)`

Process base64 files through FileProcessor and return combined text.

Parameters:

| Name               | Type                | Description                                   | Default                                        |
| ------------------ | ------------------- | --------------------------------------------- | ---------------------------------------------- |
| `files`            | `list[FileContent]` | List of FileContent with base64-encoded data. | *required*                                     |
| `execution_config` | \`ExecutionConfig   | None\`                                        | Optional execution configuration for timeouts. |

Returns:

| Type  | Description                                         |
| ----- | --------------------------------------------------- |
| `str` | Combined markdown content from all processed files. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
async def process_files(
    self,
    files: list[FileContent],
    execution_config: ExecutionConfig | None = None,
) -> str:
    """Process base64 files through FileProcessor and return combined text.

    Args:
        files: List of FileContent with base64-encoded data.
        execution_config: Optional execution configuration for timeouts.

    Returns:
        Combined markdown content from all processed files.
    """
    if not files:
        return ""

    combined_content, file_inputs = process_multimodal_files(
        files=files,
        execution_config=execution_config,
        is_agui_format=False,
    )

    # Clean up temporary files
    cleanup_temp_files(file_inputs)

    return combined_content
```

## `SSEEvent`

SSE event serializer following sse-events.md specification.

Event format

event: {type} data: {json}

Keepalive format

: keepalive

### `format(event_type, data)`

Format an SSE event with type and JSON data.

Parameters:

| Name         | Type             | Description                           | Default    |
| ------------ | ---------------- | ------------------------------------- | ---------- |
| `event_type` | `str`            | The event type name.                  | *required* |
| `data`       | `dict[str, Any]` | Dictionary to serialize as JSON data. | *required* |

Returns:

| Type  | Description                                                |
| ----- | ---------------------------------------------------------- |
| `str` | SSE formatted string: "event: {type}\\ndata: {json}\\n\\n" |

Source code in `src/holodeck/serve/protocols/rest.py`

```
@staticmethod
def format(event_type: str, data: dict[str, Any]) -> str:
    """Format an SSE event with type and JSON data.

    Args:
        event_type: The event type name.
        data: Dictionary to serialize as JSON data.

    Returns:
        SSE formatted string: "event: {type}\\ndata: {json}\\n\\n"
    """
    json_data = json.dumps(data, separators=(",", ":"))
    return f"event: {event_type}\ndata: {json_data}\n\n"
```

### `stream_start(session_id, message_id)`

Create stream_start event.

Parameters:

| Name         | Type  | Description                | Default    |
| ------------ | ----- | -------------------------- | ---------- |
| `session_id` | `str` | Session identifier (ULID). | *required* |
| `message_id` | `str` | Message identifier (ULID). | *required* |

Returns:

| Type  | Description                       |
| ----- | --------------------------------- |
| `str` | SSE formatted stream_start event. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
@staticmethod
def stream_start(session_id: str, message_id: str) -> str:
    """Create stream_start event.

    Args:
        session_id: Session identifier (ULID).
        message_id: Message identifier (ULID).

    Returns:
        SSE formatted stream_start event.
    """
    return SSEEvent.format(
        "stream_start",
        {
            "session_id": session_id,
            "message_id": message_id,
        },
    )
```

### `message_delta(delta, message_id)`

Create message_delta event with text chunk.

Parameters:

| Name         | Type  | Description                         | Default    |
| ------------ | ----- | ----------------------------------- | ---------- |
| `delta`      | `str` | Text content chunk.                 | *required* |
| `message_id` | `str` | Message identifier for correlation. | *required* |

Returns:

| Type  | Description                        |
| ----- | ---------------------------------- |
| `str` | SSE formatted message_delta event. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
@staticmethod
def message_delta(delta: str, message_id: str) -> str:
    """Create message_delta event with text chunk.

    Args:
        delta: Text content chunk.
        message_id: Message identifier for correlation.

    Returns:
        SSE formatted message_delta event.
    """
    return SSEEvent.format(
        "message_delta",
        {
            "delta": delta,
            "message_id": message_id,
        },
    )
```

### `tool_call_start(tool_call_id, name, message_id)`

Create tool_call_start event.

Parameters:

| Name           | Type  | Description                  | Default    |
| -------------- | ----- | ---------------------------- | ---------- |
| `tool_call_id` | `str` | Unique tool call identifier. | *required* |
| `name`         | `str` | Tool name being called.      | *required* |
| `message_id`   | `str` | Parent message identifier.   | *required* |

Returns:

| Type  | Description                          |
| ----- | ------------------------------------ |
| `str` | SSE formatted tool_call_start event. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
@staticmethod
def tool_call_start(tool_call_id: str, name: str, message_id: str) -> str:
    """Create tool_call_start event.

    Args:
        tool_call_id: Unique tool call identifier.
        name: Tool name being called.
        message_id: Parent message identifier.

    Returns:
        SSE formatted tool_call_start event.
    """
    return SSEEvent.format(
        "tool_call_start",
        {
            "tool_call_id": tool_call_id,
            "name": name,
            "message_id": message_id,
        },
    )
```

### `tool_call_args(tool_call_id, args_delta)`

Create tool_call_args event with argument fragment.

Parameters:

| Name           | Type  | Description                           | Default    |
| -------------- | ----- | ------------------------------------- | ---------- |
| `tool_call_id` | `str` | Tool call identifier for correlation. | *required* |
| `args_delta`   | `str` | JSON fragment of tool arguments.      | *required* |

Returns:

| Type  | Description                         |
| ----- | ----------------------------------- |
| `str` | SSE formatted tool_call_args event. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
@staticmethod
def tool_call_args(tool_call_id: str, args_delta: str) -> str:
    """Create tool_call_args event with argument fragment.

    Args:
        tool_call_id: Tool call identifier for correlation.
        args_delta: JSON fragment of tool arguments.

    Returns:
        SSE formatted tool_call_args event.
    """
    return SSEEvent.format(
        "tool_call_args",
        {
            "tool_call_id": tool_call_id,
            "args_delta": args_delta,
        },
    )
```

### `tool_call_end(tool_call_id, status)`

Create tool_call_end event.

Parameters:

| Name           | Type  | Description                              | Default    |
| -------------- | ----- | ---------------------------------------- | ---------- |
| `tool_call_id` | `str` | Tool call identifier.                    | *required* |
| `status`       | `str` | Execution status ("success" or "error"). | *required* |

Returns:

| Type  | Description                        |
| ----- | ---------------------------------- |
| `str` | SSE formatted tool_call_end event. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
@staticmethod
def tool_call_end(tool_call_id: str, status: str) -> str:
    """Create tool_call_end event.

    Args:
        tool_call_id: Tool call identifier.
        status: Execution status ("success" or "error").

    Returns:
        SSE formatted tool_call_end event.
    """
    return SSEEvent.format(
        "tool_call_end",
        {
            "tool_call_id": tool_call_id,
            "status": status,
        },
    )
```

### `stream_end(message_id, tokens_used, execution_time_ms)`

Create stream_end event.

Parameters:

| Name                | Type             | Description                           | Default                                     |
| ------------------- | ---------------- | ------------------------------------- | ------------------------------------------- |
| `message_id`        | `str`            | Message identifier.                   | *required*                                  |
| `tokens_used`       | \`dict[str, int] | None\`                                | Token consumption statistics (may be None). |
| `execution_time_ms` | `int`            | Total execution time in milliseconds. | *required*                                  |

Returns:

| Type  | Description                     |
| ----- | ------------------------------- |
| `str` | SSE formatted stream_end event. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
@staticmethod
def stream_end(
    message_id: str,
    tokens_used: dict[str, int] | None,
    execution_time_ms: int,
) -> str:
    """Create stream_end event.

    Args:
        message_id: Message identifier.
        tokens_used: Token consumption statistics (may be None).
        execution_time_ms: Total execution time in milliseconds.

    Returns:
        SSE formatted stream_end event.
    """
    return SSEEvent.format(
        "stream_end",
        {
            "message_id": message_id,
            "tokens_used": tokens_used,
            "execution_time_ms": execution_time_ms,
        },
    )
```

### `error(type, title, status, detail=None)`

Create error event following RFC 7807 ProblemDetail.

Parameters:

| Name     | Type  | Description                       | Default                            |
| -------- | ----- | --------------------------------- | ---------------------------------- |
| `type`   | `str` | Error type URI.                   | *required*                         |
| `title`  | `str` | Short human-readable description. | *required*                         |
| `status` | `int` | HTTP status code.                 | *required*                         |
| `detail` | \`str | None\`                            | Detailed error message (optional). |

Returns:

| Type  | Description                |
| ----- | -------------------------- |
| `str` | SSE formatted error event. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
@staticmethod
def error(
    type: str,
    title: str,
    status: int,
    detail: str | None = None,
) -> str:
    """Create error event following RFC 7807 ProblemDetail.

    Args:
        type: Error type URI.
        title: Short human-readable description.
        status: HTTP status code.
        detail: Detailed error message (optional).

    Returns:
        SSE formatted error event.
    """
    data: dict[str, Any] = {
        "type": type,
        "title": title,
        "status": status,
    }
    if detail is not None:
        data["detail"] = detail

    return SSEEvent.format("error", data)
```

### `keepalive()`

Create keepalive comment.

Returns:

| Type  | Description                          |
| ----- | ------------------------------------ |
| `str` | SSE comment format: ": keepalive\\n" |

Source code in `src/holodeck/serve/protocols/rest.py`

```
@staticmethod
def keepalive() -> str:
    """Create keepalive comment.

    Returns:
        SSE comment format: ": keepalive\\n"
    """
    return ": keepalive\n"
```

## `convert_upload_file_to_file_content(upload_file, content_bytes=None)`

Convert FastAPI UploadFile to FileContent model.

Reads the uploaded file content and encodes it as base64.

Parameters:

| Name            | Type         | Description                                  | Default                                                                                  |
| --------------- | ------------ | -------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `upload_file`   | `UploadFile` | FastAPI UploadFile from multipart form-data. | *required*                                                                               |
| `content_bytes` | \`bytes      | None\`                                       | Pre-read file content to avoid redundant I/O. If provided, the file won't be read again. |

Returns:

| Type          | Description                              |
| ------------- | ---------------------------------------- |
| `FileContent` | FileContent with base64-encoded content. |

Raises:

| Type         | Description                            |
| ------------ | -------------------------------------- |
| `ValueError` | If file content type is not supported. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
async def convert_upload_file_to_file_content(
    upload_file: UploadFile,
    content_bytes: bytes | None = None,
) -> FileContent:
    """Convert FastAPI UploadFile to FileContent model.

    Reads the uploaded file content and encodes it as base64.

    Args:
        upload_file: FastAPI UploadFile from multipart form-data.
        content_bytes: Pre-read file content to avoid redundant I/O.
            If provided, the file won't be read again.

    Returns:
        FileContent with base64-encoded content.

    Raises:
        ValueError: If file content type is not supported.
    """
    # Use pre-read content if available, otherwise read from file
    if content_bytes is None:
        content_bytes = await upload_file.read()

    # Encode as base64
    content_b64 = base64.b64encode(content_bytes).decode("utf-8")

    # Get MIME type from content_type or filename
    mime_type = upload_file.content_type or "application/octet-stream"

    # Validate MIME type is supported (uses shared constant from models)
    if mime_type not in SUPPORTED_MIME_TYPES:
        raise ValueError(f"Unsupported MIME type: {mime_type}")

    return FileContent(
        content=content_b64,
        mime_type=mime_type,
        filename=upload_file.filename,
    )
```

## `process_multipart_files(files)`

Process multipart file uploads and convert to FileContent list.

Parameters:

| Name    | Type               | Description                         | Default    |
| ------- | ------------------ | ----------------------------------- | ---------- |
| `files` | `list[UploadFile]` | List of FastAPI UploadFile objects. | *required* |

Returns:

| Type                | Description                                              |
| ------------------- | -------------------------------------------------------- |
| `list[FileContent]` | List of FileContent objects with base64-encoded content. |

Raises:

| Type         | Description                                                   |
| ------------ | ------------------------------------------------------------- |
| `ValueError` | If any file has unsupported MIME type or exceeds size limits. |

Source code in `src/holodeck/serve/protocols/rest.py`

```
async def process_multipart_files(
    files: list[UploadFile],
) -> list[FileContent]:
    """Process multipart file uploads and convert to FileContent list.

    Args:
        files: List of FastAPI UploadFile objects.

    Returns:
        List of FileContent objects with base64-encoded content.

    Raises:
        ValueError: If any file has unsupported MIME type or exceeds size limits.
    """
    file_contents: list[FileContent] = []
    total_size = 0

    for upload_file in files:
        # Read file content once
        content = await upload_file.read()
        file_size = len(content)

        if file_size > MAX_FILE_SIZE_BYTES:
            raise ValueError(
                f"File '{upload_file.filename}' exceeds maximum size of "
                f"{MAX_FILE_SIZE_MB}MB"
            )

        total_size += file_size
        if total_size > MAX_TOTAL_SIZE_BYTES:
            raise ValueError(
                f"Total file size exceeds maximum of {MAX_TOTAL_SIZE_MB}MB"
            )

        # Convert to FileContent, passing pre-read content to avoid redundant I/O
        file_content = await convert_upload_file_to_file_content(
            upload_file, content_bytes=content
        )
        file_contents.append(file_content)

    return file_contents
```

### AG-UI Protocol

AG-UI protocol adapter implementing the [ag-ui-protocol](https://github.com/ag-ui-protocol/ag-ui) event-driven streaming pattern for agent interaction.

## `AGUIProtocol(accept_header=None, execution_config=None)`

Bases: `Protocol`

AG-UI protocol implementation.

Handles /awp endpoint requests, converting between AG-UI events and HoloDeck's AgentExecutor.

The AG-UI protocol follows an event-driven streaming pattern:

1. RunStartedEvent - Signals execution start
1. TextMessageStartEvent - Opens message stream
1. ToolCall\* events - For any tool invocations
1. TextMessageContentEvent - Streams response text
1. TextMessageEndEvent - Closes message stream
1. RunFinishedEvent/RunErrorEvent - Signals completion

Initialize the AG-UI protocol.

Parameters:

| Name               | Type              | Description | Default                                                                                                                                                                                     |
| ------------------ | ----------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `accept_header`    | \`str             | None\`      | HTTP Accept header for format negotiation.                                                                                                                                                  |
| `execution_config` | \`ExecutionConfig | None\`      | Execution configuration forwarded to process_multimodal_files so execution.file_timeout and execution.download_timeout from agent.yaml reach the FileProcessor used for binary attachments. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def __init__(
    self,
    accept_header: str | None = None,
    execution_config: ExecutionConfig | None = None,
) -> None:
    """Initialize the AG-UI protocol.

    Args:
        accept_header: HTTP Accept header for format negotiation.
        execution_config: Execution configuration forwarded to
            ``process_multimodal_files`` so ``execution.file_timeout``
            and ``execution.download_timeout`` from agent.yaml reach
            the FileProcessor used for binary attachments.
    """
    self._accept_header = accept_header
    self._execution_config = execution_config
```

### `name`

Return the protocol name.

Returns:

| Type  | Description                 |
| ----- | --------------------------- |
| `str` | Protocol identifier string. |

### `content_type`

Return the content type for responses.

Returns:

| Type  | Description                                        |
| ----- | -------------------------------------------------- |
| `str` | MIME type string for response Content-Type header. |

### `handle_request(request, session)`

Handle AG-UI request and generate event stream.

Processes the RunAgentInput, executes the agent, and yields encoded AG-UI events for streaming to the client.

Parameters:

| Name      | Type            | Description                        | Default    |
| --------- | --------------- | ---------------------------------- | ---------- |
| `request` | `Any`           | RunAgentInput from client.         | *required* |
| `session` | `ServerSession` | Server session with AgentExecutor. | *required* |

Yields:

| Type                          | Description                    |
| ----------------------------- | ------------------------------ |
| `AsyncGenerator[bytes, None]` | Encoded AG-UI events as bytes. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
async def handle_request(
    self,
    request: Any,
    session: ServerSession,
) -> AsyncGenerator[bytes, None]:
    """Handle AG-UI request and generate event stream.

    Processes the RunAgentInput, executes the agent, and yields
    encoded AG-UI events for streaming to the client.

    Args:
        request: RunAgentInput from client.
        session: Server session with AgentExecutor.

    Yields:
        Encoded AG-UI events as bytes.
    """
    # Extract components from RunAgentInput
    input_data: RunAgentInput = request
    thread_id = input_data.thread_id
    run_id = input_data.run_id

    # Create event encoder
    encoder = AGUIEventStream(accept_header=self._accept_header)
    message_id = str(ULID())

    # Track file inputs for cleanup
    file_inputs_to_cleanup: list[FileInput] = []

    try:
        # Extract user message and binary content from input
        logger.debug(
            "Starting request processing for run_id=%s, thread_id=%s",
            run_id,
            thread_id,
        )
        text_message, binary_parts = extract_message_and_files_from_input(
            input_data
        )
        logger.debug(
            "Request contains: text=%d chars, binary_parts=%d",
            len(text_message) if text_message else 0,
            len(binary_parts),
        )

        # Process binary files if present
        file_content = ""
        if binary_parts:
            logger.debug(
                "Processing %d binary parts for message context",
                len(binary_parts),
            )
            file_content, file_inputs_to_cleanup = process_multimodal_files(
                files=binary_parts,
                execution_config=self._execution_config,
                is_agui_format=True,
            )
            if file_content:
                logger.debug(
                    "File processing complete: %d files -> %d chars of context",
                    len(file_inputs_to_cleanup),
                    len(file_content),
                )
            elif binary_parts:
                logger.warning(
                    "File processing returned no content for %d files",
                    len(binary_parts),
                )

        # Combine text message with file content
        if file_content and text_message:
            full_message = f"{text_message}\n\n{file_content}"
            logger.debug(
                "Combined message: text + files = %d chars total",
                len(full_message),
            )
        elif file_content:
            full_message = file_content
            logger.debug(
                "Message is file content only: %d chars",
                len(full_message),
            )
        else:
            full_message = text_message

        # 1. Emit RunStartedEvent
        yield encoder.encode(create_run_started_event(thread_id, run_id))

        # 2. Eagerly initialize backend so tool_event_queue is available
        #    before execution starts. Gracefully degrade if the executor
        #    does not expose this method (e.g. in tests with mocks).
        executor = session.agent_executor
        logger.debug(
            "[trace] agui.handle_request: session=%s, executor_id=%s, "
            "session_attached=%s",
            session.session_id,
            id(executor),
            getattr(executor, "_session", None) is not None,
        )
        _ensure = getattr(executor, "_ensure_backend_and_session", None)
        if callable(_ensure) and asyncio.iscoroutinefunction(_ensure):
            await _ensure()
        tool_queue = getattr(executor, "tool_event_queue", None)
        logger.debug(
            "[trace] agui.handle_request: ensure done, tool_queue=%s",
            (
                "Queue"
                if isinstance(tool_queue, asyncio.Queue)
                else type(tool_queue).__name__
            ),
        )

        # 3. Execute agent — drain tool events concurrently if supported.
        #    Each tool call is wrapped in its own assistant message so
        #    CopilotKit renders a separate card per tool invocation.
        logger.debug(
            "[trace] agui.handle_request: executing agent for session %s",
            session.session_id,
        )
        tool_msg_ids: dict[str, str] = {}

        if isinstance(tool_queue, asyncio.Queue):
            # Real-time path: run execute_turn as a task, drain events
            execute_task = asyncio.create_task(executor.execute_turn(full_message))

            poll_count = 0
            while not execute_task.done():
                poll_count += 1
                if poll_count % 50 == 0:
                    logger.debug(
                        "[trace] agui.handle_request: still polling "
                        "tool_queue (%d iters, ~%.1fs)",
                        poll_count,
                        poll_count * 0.1,
                    )
                try:
                    event = await asyncio.wait_for(tool_queue.get(), timeout=0.1)
                    for agui_evt in _tool_event_to_agui(event, tool_msg_ids):
                        yield encoder.encode(agui_evt)
                except asyncio.TimeoutError:
                    continue

            logger.debug(
                "[trace] agui.handle_request: execute_task done, awaiting result"
            )
            response = await execute_task
            logger.debug(
                "[trace] agui.handle_request: execute_task result received"
            )

            # Drain any events that arrived between last poll and completion
            while not tool_queue.empty():
                event = tool_queue.get_nowait()
                for agui_evt in _tool_event_to_agui(event, tool_msg_ids):
                    yield encoder.encode(agui_evt)
        else:
            # Fallback path (e.g. SK backend): post-hoc tool events
            response = await executor.execute_turn(full_message)
            for tool_exec in response.tool_executions:
                logger.debug(
                    "Emitting tool call events for: %s", tool_exec.tool_name
                )
                for evt in create_tool_call_events(tool_exec, message_id):
                    yield encoder.encode(evt)

        # 4. Emit text response as its own assistant message
        yield encoder.encode(create_text_message_start(message_id))
        yield encoder.encode(
            create_text_message_content(message_id, response.content)
        )
        yield encoder.encode(create_text_message_end(message_id))

        # 5. Emit RunFinishedEvent
        yield encoder.encode(create_run_finished_event(thread_id, run_id))

        logger.debug("Completed request for run %s", run_id)

    except BackendSessionError as e:
        logger.error("Backend session error: %s", e, exc_info=True)
        yield encoder.encode(
            create_run_error_event(
                "Claude Agent SDK subprocess terminated unexpectedly.",
            )
        )

    except Exception as e:
        logger.error("Error processing request: %s", e, exc_info=True)
        # Emit error event
        yield encoder.encode(create_run_error_event(str(e)))

    finally:
        # Clean up temporary files
        if file_inputs_to_cleanup:
            logger.debug(
                "Cleaning up %d temporary files",
                len(file_inputs_to_cleanup),
            )
            cleanup_temp_files(file_inputs_to_cleanup)
```

## `AGUIEventStream(accept_header=None)`

Wrapper for AG-UI event encoding and streaming.

Handles format negotiation based on HTTP Accept header and encodes events for streaming to clients.

Initialize event stream with format negotiation.

Parameters:

| Name            | Type  | Description | Default                                                                       |
| --------------- | ----- | ----------- | ----------------------------------------------------------------------------- |
| `accept_header` | \`str | None\`      | HTTP Accept header for format selection. Defaults to text/event-stream (SSE). |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def __init__(self, accept_header: str | None = None) -> None:
    """Initialize event stream with format negotiation.

    Args:
        accept_header: HTTP Accept header for format selection.
                     Defaults to text/event-stream (SSE).
    """
    # EventEncoder requires a string, default to SSE format
    self.encoder = EventEncoder(accept=accept_header or "text/event-stream")
```

### `content_type`

Get the content type for the streaming response.

Returns:

| Type  | Description                                        |
| ----- | -------------------------------------------------- |
| `str` | MIME type string for response Content-Type header. |

### `encode(event)`

Encode a single AG-UI event.

Parameters:

| Name    | Type        | Description            | Default    |
| ------- | ----------- | ---------------------- | ---------- |
| `event` | `BaseEvent` | AG-UI event to encode. | *required* |

Returns:

| Type    | Description                             |
| ------- | --------------------------------------- |
| `bytes` | Encoded bytes for SSE or binary format. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def encode(self, event: BaseEvent) -> bytes:
    """Encode a single AG-UI event.

    Args:
        event: AG-UI event to encode.

    Returns:
        Encoded bytes for SSE or binary format.
    """
    encoded = self.encoder.encode(event)
    # Ensure we return bytes for streaming
    # EventEncoder returns str for SSE format, bytes for binary
    # Note: mypy doesn't know EventEncoder can return bytes
    if isinstance(encoded, bytes):  # type: ignore[unreachable]
        return encoded  # type: ignore[unreachable]
    result: bytes = encoded.encode("utf-8")
    return result
```

#### Event Factory Functions

## `extract_message_and_files_from_input(input_data)`

Extract text message and binary content parts from RunAgentInput.

Parameters:

| Name         | Type            | Description                           | Default    |
| ------------ | --------------- | ------------------------------------- | ---------- |
| `input_data` | `RunAgentInput` | AG-UI input containing messages list. | *required* |

Returns:

| Type                               | Description                                 |
| ---------------------------------- | ------------------------------------------- |
| `tuple[str, list[dict[str, Any]]]` | Tuple of (text_message, binary_parts_list). |

Raises:

| Type         | Description                |
| ------------ | -------------------------- |
| `ValueError` | If no user messages found. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def extract_message_and_files_from_input(
    input_data: RunAgentInput,
) -> tuple[str, list[dict[str, Any]]]:
    """Extract text message and binary content parts from RunAgentInput.

    Args:
        input_data: AG-UI input containing messages list.

    Returns:
        Tuple of (text_message, binary_parts_list).

    Raises:
        ValueError: If no user messages found.
    """
    messages = input_data.messages or []
    logger.debug(
        "Extracting message and files from input (total messages: %d)",
        len(messages),
    )

    # Find the last user message
    for message in reversed(messages):
        # Messages can be dicts or Message objects
        if isinstance(message, dict):  # type: ignore[unreachable]
            role = message.get("role", "")  # type: ignore[unreachable]
            content = message.get("content", "")
        else:
            role = getattr(message, "role", "")
            content = getattr(message, "content", "")

        if role == "user":
            logger.debug(
                "Found user message, content type: %s",
                type(content).__name__,
            )

            # Content can be a string or a list of content parts
            if isinstance(content, str):
                logger.debug(
                    "User message is plain string (%d chars), no files",
                    len(content),
                )
                return content, []
            elif isinstance(content, list):
                logger.debug(
                    "User message has list content (%d parts)",
                    len(content),
                )

                # Extract text and binary parts
                text_parts: list[str] = []
                binary_parts = extract_binary_parts_from_content(content)

                for part in content:
                    # Handle dict format
                    if isinstance(part, dict):
                        part_type = part.get("type", "unknown")
                        if part_type == "text":
                            text_parts.append(part.get("text", ""))
                    # Handle AG-UI Pydantic object (TextInputContent)
                    elif (
                        hasattr(part, "type") and getattr(part, "type", None) == "text"
                    ):
                        text_parts.append(getattr(part, "text", ""))
                    # Handle plain string
                    elif isinstance(part, str):
                        text_parts.append(part)

                # Validate that we have at least some content (text or binary)
                if not text_parts and not binary_parts:
                    logger.error(
                        "No content found in user message - "
                        "neither text nor binary parts"
                    )
                    raise ValueError(
                        "No content found in user message. "
                        "Message contained no text or binary content parts."
                    )

                text_message = " ".join(text_parts) if text_parts else ""
                logger.debug(
                    "Extracted: text=%d chars, binary_parts=%d",
                    len(text_message),
                    len(binary_parts),
                )
                return text_message, binary_parts

    logger.error("No user messages found in input")
    raise ValueError("No user messages found in input")
```

## `extract_message_from_input(input_data)`

Extract the last user message from RunAgentInput.

Parameters:

| Name         | Type            | Description                           | Default    |
| ------------ | --------------- | ------------------------------------- | ---------- |
| `input_data` | `RunAgentInput` | AG-UI input containing messages list. | *required* |

Returns:

| Type  | Description                                |
| ----- | ------------------------------------------ |
| `str` | The text content of the last user message. |

Raises:

| Type         | Description                |
| ------------ | -------------------------- |
| `ValueError` | If no user messages found. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def extract_message_from_input(input_data: RunAgentInput) -> str:
    """Extract the last user message from RunAgentInput.

    Args:
        input_data: AG-UI input containing messages list.

    Returns:
        The text content of the last user message.

    Raises:
        ValueError: If no user messages found.
    """
    messages = input_data.messages or []

    # Find the last user message
    for message in reversed(messages):
        # Messages can be dicts or Message objects
        # Note: At runtime, JSON deserialization may produce dicts
        if isinstance(message, dict):  # type: ignore[unreachable]
            role = message.get("role", "")  # type: ignore[unreachable]
            content = message.get("content", "")
        else:
            role = getattr(message, "role", "")
            content = getattr(message, "content", "")

        if role == "user":
            # Content can be a string or a list of content parts
            if isinstance(content, str):
                return content
            elif isinstance(content, list):
                # Extract text from content parts
                text_parts = []
                for part in content:
                    if isinstance(part, dict):
                        part_type = part.get("type", "unknown")
                        if part_type == "text":
                            text_parts.append(part.get("text", ""))
                        else:
                            # Non-text content (image, file, etc) not supported
                            logger.warning(
                                "Skipping non-text content part (type: %s). "
                                "Only 'text' content parts are supported.",
                                part_type,
                            )
                    elif isinstance(part, str):
                        text_parts.append(part)

                # Validate that we have at least some text content
                if not text_parts:
                    raise ValueError(
                        "No text content found in user message. "
                        "Message contained only non-text content parts."
                    )
                return " ".join(text_parts)

    raise ValueError("No user messages found in input")
```

## `map_session_id(thread_id)`

Map AG-UI thread_id to HoloDeck session_id.

The thread_id from AG-UI is used directly as the session_id.

Parameters:

| Name        | Type  | Description                           | Default    |
| ----------- | ----- | ------------------------------------- | ---------- |
| `thread_id` | `str` | AG-UI conversation thread identifier. | *required* |

Returns:

| Type  | Description                           |
| ----- | ------------------------------------- |
| `str` | Session ID (uses thread_id directly). |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def map_session_id(thread_id: str) -> str:
    """Map AG-UI thread_id to HoloDeck session_id.

    The thread_id from AG-UI is used directly as the session_id.

    Args:
        thread_id: AG-UI conversation thread identifier.

    Returns:
        Session ID (uses thread_id directly).
    """
    return thread_id
```

## `generate_run_id()`

Generate a unique run ID for this request.

Returns:

| Type  | Description                  |
| ----- | ---------------------------- |
| `str` | New ULID string for the run. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def generate_run_id() -> str:
    """Generate a unique run ID for this request.

    Returns:
        New ULID string for the run.
    """
    return str(ULID())
```

## `create_run_started_event(thread_id, run_id)`

Create RunStartedEvent for stream beginning.

Parameters:

| Name        | Type  | Description                     | Default    |
| ----------- | ----- | ------------------------------- | ---------- |
| `thread_id` | `str` | Conversation thread identifier. | *required* |
| `run_id`    | `str` | Unique run identifier.          | *required* |

Returns:

| Type              | Description               |
| ----------------- | ------------------------- |
| `RunStartedEvent` | RunStartedEvent instance. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_run_started_event(thread_id: str, run_id: str) -> RunStartedEvent:
    """Create RunStartedEvent for stream beginning.

    Args:
        thread_id: Conversation thread identifier.
        run_id: Unique run identifier.

    Returns:
        RunStartedEvent instance.
    """
    return RunStartedEvent(
        type=EventType.RUN_STARTED,
        thread_id=thread_id,
        run_id=run_id,
    )
```

## `create_run_finished_event(thread_id, run_id)`

Create RunFinishedEvent for successful completion.

Parameters:

| Name        | Type  | Description                     | Default    |
| ----------- | ----- | ------------------------------- | ---------- |
| `thread_id` | `str` | Conversation thread identifier. | *required* |
| `run_id`    | `str` | Unique run identifier.          | *required* |

Returns:

| Type               | Description                |
| ------------------ | -------------------------- |
| `RunFinishedEvent` | RunFinishedEvent instance. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_run_finished_event(thread_id: str, run_id: str) -> RunFinishedEvent:
    """Create RunFinishedEvent for successful completion.

    Args:
        thread_id: Conversation thread identifier.
        run_id: Unique run identifier.

    Returns:
        RunFinishedEvent instance.
    """
    return RunFinishedEvent(
        type=EventType.RUN_FINISHED,
        thread_id=thread_id,
        run_id=run_id,
    )
```

## `create_run_error_event(message, code=None)`

Create RunErrorEvent for failure.

Parameters:

| Name      | Type  | Description                           | Default                                 |
| --------- | ----- | ------------------------------------- | --------------------------------------- |
| `message` | `str` | Error message describing the failure. | *required*                              |
| `code`    | \`str | None\`                                | Optional error code for categorization. |

Returns:

| Type            | Description             |
| --------------- | ----------------------- |
| `RunErrorEvent` | RunErrorEvent instance. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_run_error_event(message: str, code: str | None = None) -> RunErrorEvent:
    """Create RunErrorEvent for failure.

    Args:
        message: Error message describing the failure.
        code: Optional error code for categorization.

    Returns:
        RunErrorEvent instance.
    """
    return RunErrorEvent(
        type=EventType.RUN_ERROR,
        message=message,
        code=code,
    )
```

## `create_text_message_start(message_id)`

Create TextMessageStartEvent to open message stream.

Parameters:

| Name         | Type  | Description                | Default    |
| ------------ | ----- | -------------------------- | ---------- |
| `message_id` | `str` | Unique message identifier. | *required* |

Returns:

| Type                    | Description                     |
| ----------------------- | ------------------------------- |
| `TextMessageStartEvent` | TextMessageStartEvent instance. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_text_message_start(message_id: str) -> TextMessageStartEvent:
    """Create TextMessageStartEvent to open message stream.

    Args:
        message_id: Unique message identifier.

    Returns:
        TextMessageStartEvent instance.
    """
    return TextMessageStartEvent(
        type=EventType.TEXT_MESSAGE_START,
        message_id=message_id,
        role="assistant",
    )
```

## `create_text_message_content(message_id, delta)`

Create TextMessageContentEvent with text chunk.

Parameters:

| Name         | Type  | Description                         | Default    |
| ------------ | ----- | ----------------------------------- | ---------- |
| `message_id` | `str` | Message identifier for correlation. | *required* |
| `delta`      | `str` | Text chunk to stream.               | *required* |

Returns:

| Type                      | Description                       |
| ------------------------- | --------------------------------- |
| `TextMessageContentEvent` | TextMessageContentEvent instance. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_text_message_content(message_id: str, delta: str) -> TextMessageContentEvent:
    """Create TextMessageContentEvent with text chunk.

    Args:
        message_id: Message identifier for correlation.
        delta: Text chunk to stream.

    Returns:
        TextMessageContentEvent instance.
    """
    return TextMessageContentEvent(
        type=EventType.TEXT_MESSAGE_CONTENT,
        message_id=message_id,
        delta=delta,
    )
```

## `create_text_message_end(message_id)`

Create TextMessageEndEvent to close message stream.

Parameters:

| Name         | Type  | Description                         | Default    |
| ------------ | ----- | ----------------------------------- | ---------- |
| `message_id` | `str` | Message identifier for correlation. | *required* |

Returns:

| Type                  | Description                   |
| --------------------- | ----------------------------- |
| `TextMessageEndEvent` | TextMessageEndEvent instance. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_text_message_end(message_id: str) -> TextMessageEndEvent:
    """Create TextMessageEndEvent to close message stream.

    Args:
        message_id: Message identifier for correlation.

    Returns:
        TextMessageEndEvent instance.
    """
    return TextMessageEndEvent(
        type=EventType.TEXT_MESSAGE_END,
        message_id=message_id,
    )
```

## `create_tool_call_start(tool_call_id, tool_call_name, parent_message_id=None)`

Create ToolCallStartEvent to initiate tool execution.

Parameters:

| Name                | Type  | Description                    | Default                             |
| ------------------- | ----- | ------------------------------ | ----------------------------------- |
| `tool_call_id`      | `str` | Unique tool call identifier.   | *required*                          |
| `tool_call_name`    | `str` | Name of the tool being called. | *required*                          |
| `parent_message_id` | \`str | None\`                         | Optional parent message identifier. |

Returns:

| Type                 | Description                  |
| -------------------- | ---------------------------- |
| `ToolCallStartEvent` | ToolCallStartEvent instance. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_tool_call_start(
    tool_call_id: str,
    tool_call_name: str,
    parent_message_id: str | None = None,
) -> ToolCallStartEvent:
    """Create ToolCallStartEvent to initiate tool execution.

    Args:
        tool_call_id: Unique tool call identifier.
        tool_call_name: Name of the tool being called.
        parent_message_id: Optional parent message identifier.

    Returns:
        ToolCallStartEvent instance.
    """
    return ToolCallStartEvent(
        type=EventType.TOOL_CALL_START,
        tool_call_id=tool_call_id,
        tool_call_name=tool_call_name,
        parent_message_id=parent_message_id,
    )
```

## `create_tool_call_args(tool_call_id, delta)`

Create ToolCallArgsEvent with argument fragment.

Parameters:

| Name           | Type  | Description                           | Default    |
| -------------- | ----- | ------------------------------------- | ---------- |
| `tool_call_id` | `str` | Tool call identifier for correlation. | *required* |
| `delta`        | `str` | JSON fragment of arguments.           | *required* |

Returns:

| Type                | Description                 |
| ------------------- | --------------------------- |
| `ToolCallArgsEvent` | ToolCallArgsEvent instance. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_tool_call_args(tool_call_id: str, delta: str) -> ToolCallArgsEvent:
    """Create ToolCallArgsEvent with argument fragment.

    Args:
        tool_call_id: Tool call identifier for correlation.
        delta: JSON fragment of arguments.

    Returns:
        ToolCallArgsEvent instance.
    """
    return ToolCallArgsEvent(
        type=EventType.TOOL_CALL_ARGS,
        tool_call_id=tool_call_id,
        delta=delta,
    )
```

## `create_tool_call_end(tool_call_id)`

Create ToolCallEndEvent to complete argument transmission.

Parameters:

| Name           | Type  | Description                           | Default    |
| -------------- | ----- | ------------------------------------- | ---------- |
| `tool_call_id` | `str` | Tool call identifier for correlation. | *required* |

Returns:

| Type               | Description                |
| ------------------ | -------------------------- |
| `ToolCallEndEvent` | ToolCallEndEvent instance. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_tool_call_end(tool_call_id: str) -> ToolCallEndEvent:
    """Create ToolCallEndEvent to complete argument transmission.

    Args:
        tool_call_id: Tool call identifier for correlation.

    Returns:
        ToolCallEndEvent instance.
    """
    return ToolCallEndEvent(
        type=EventType.TOOL_CALL_END,
        tool_call_id=tool_call_id,
    )
```

## `create_tool_call_events(tool_execution, message_id)`

Create complete tool call event sequence from ToolExecution.

Each tool call is wrapped in its own assistant message so CopilotKit renders a separate card per tool invocation.

Parameters:

| Name             | Type            | Description                               | Default    |
| ---------------- | --------------- | ----------------------------------------- | ---------- |
| `tool_execution` | `ToolExecution` | HoloDeck tool execution result.           | *required* |
| `message_id`     | `str`           | Unused (kept for backward compatibility). | *required* |

Returns:

| Type              | Description                              |
| ----------------- | ---------------------------------------- |
| `list[BaseEvent]` | List of AG-UI events for this tool call. |

Source code in `src/holodeck/serve/protocols/agui.py`

```
def create_tool_call_events(
    tool_execution: ToolExecution,
    message_id: str,  # noqa: ARG001 – kept for API compat
) -> list[BaseEvent]:
    """Create complete tool call event sequence from ToolExecution.

    Each tool call is wrapped in its own assistant message so CopilotKit
    renders a separate card per tool invocation.

    Args:
        tool_execution: HoloDeck tool execution result.
        message_id: Unused (kept for backward compatibility).

    Returns:
        List of AG-UI events for this tool call.
    """
    tool_call_id = str(ULID())
    parent_msg_id = str(ULID())

    result_content = tool_execution.result or ""

    events: list[BaseEvent] = [
        create_text_message_start(parent_msg_id),
        create_tool_call_start(
            tool_call_id=tool_call_id,
            tool_call_name=tool_execution.tool_name,
            parent_message_id=parent_msg_id,
        ),
        create_tool_call_args(
            tool_call_id=tool_call_id,
            delta=json.dumps(tool_execution.parameters),
        ),
        create_tool_call_end(tool_call_id=tool_call_id),
        ToolCallResultEvent(
            type=EventType.TOOL_CALL_RESULT,
            message_id=str(ULID()),
            tool_call_id=tool_call_id,
            content=result_content,
            role="tool",
        ),
        create_text_message_end(parent_msg_id),
    ]

    return events
```

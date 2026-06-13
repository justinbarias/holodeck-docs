# Observability

The observability subsystem provides OpenTelemetry instrumentation for HoloDeck agents, following GenAI semantic conventions. It manages traces, metrics, and logs through a unified initialization lifecycle and supports multiple exporters (console, OTLP, with Prometheus and Azure Monitor planned).

## Package entry point

## `observability`

OpenTelemetry observability module for HoloDeck.

This module provides telemetry instrumentation for HoloDeck agents, following OpenTelemetry GenAI semantic conventions.

Public API

initialize_observability: Initialize all telemetry providers shutdown_observability: Gracefully shutdown providers get_tracer: Get a tracer for creating spans get_meter: Get a meter for creating metrics enable_semantic_kernel_telemetry: Enable SK's native GenAI instrumentation ObservabilityContext: Container for initialized providers

Errors

ObservabilityError: Base exception for observability errors ObservabilityConfigError: Invalid configuration error

Example

> > > from holodeck.lib.observability import initialize_observability from holodeck.lib.observability import shutdown_observability from holodeck.models.observability import ObservabilityConfig
> > >
> > > config = ObservabilityConfig(enabled=True) context = initialize_observability(config, agent_name="my-agent") try: ... # Run agent ... pass ... finally: ... shutdown_observability(context)

Task: T053 - Export public API from **init**.py

### `ObservabilityContext(tracer_provider, meter_provider, logger_provider, exporters=list(), resource=Resource.create())`

Container for initialized observability components.

Holds references to all telemetry providers and tracks which exporters are active. Used for lifecycle management and provider access.

Attributes:

| Name              | Type             | Description                                                |
| ----------------- | ---------------- | ---------------------------------------------------------- |
| `tracer_provider` | \`TracerProvider | None\`                                                     |
| `meter_provider`  | \`MeterProvider  | None\`                                                     |
| `logger_provider` | `Any`            | OpenTelemetry LoggerProvider instance                      |
| `exporters`       | `list[str]`      | List of enabled exporter names (e.g., ["console", "otlp"]) |
| `resource`        | `Resource`       | Shared OpenTelemetry Resource                              |

#### `get_resource()`

Get the shared OpenTelemetry resource.

Returns:

| Type       | Description                                    |
| ---------- | ---------------------------------------------- |
| `Resource` | The Resource instance shared by all providers. |

Source code in `src/holodeck/lib/observability/providers.py`

```
def get_resource(self) -> Resource:
    """Get the shared OpenTelemetry resource.

    Returns:
        The Resource instance shared by all providers.
    """
    return self.resource
```

#### `is_enabled()`

Check if observability is active.

Returns:

| Type   | Description                                             |
| ------ | ------------------------------------------------------- |
| `bool` | True if all providers are initialized, False otherwise. |

Source code in `src/holodeck/lib/observability/providers.py`

```
def is_enabled(self) -> bool:
    """Check if observability is active.

    Returns:
        True if all providers are initialized, False otherwise.
    """
    return (
        self.tracer_provider is not None
        and self.meter_provider is not None
        and self.logger_provider is not None
    )
```

### `ObservabilityError(message)`

Bases: `HoloDeckError`

Base exception for all observability-related errors.

All observability-specific exceptions inherit from this class, enabling centralized exception handling for telemetry operations.

Attributes:

| Name      | Type | Description                  |
| --------- | ---- | ---------------------------- |
| `message` |      | Human-readable error message |

Initialize ObservabilityError with message.

Parameters:

| Name      | Type  | Description               | Default    |
| --------- | ----- | ------------------------- | ---------- |
| `message` | `str` | Descriptive error message | *required* |

Source code in `src/holodeck/lib/observability/errors.py`

```
def __init__(self, message: str) -> None:
    """Initialize ObservabilityError with message.

    Args:
        message: Descriptive error message
    """
    self.message = message
    super().__init__(message)
```

### `ObservabilityConfigError(field, message)`

Bases: `ObservabilityError`

Exception raised for observability configuration errors.

Raised when observability configuration is invalid or incomplete, such as missing required fields or invalid exporter settings.

Attributes:

| Name      | Type | Description                                   |
| --------- | ---- | --------------------------------------------- |
| `field`   |      | The configuration field that caused the error |
| `message` |      | Human-readable error message                  |

Initialize ObservabilityConfigError with field and message.

Parameters:

| Name      | Type  | Description                                   | Default    |
| --------- | ----- | --------------------------------------------- | ---------- |
| `field`   | `str` | Configuration field name where error occurred | *required* |
| `message` | `str` | Descriptive error message                     | *required* |

Source code in `src/holodeck/lib/observability/errors.py`

```
def __init__(self, field: str, message: str) -> None:
    """Initialize ObservabilityConfigError with field and message.

    Args:
        field: Configuration field name where error occurred
        message: Descriptive error message
    """
    self.field = field
    full_message = f"Observability configuration error in '{field}': {message}"
    super().__init__(full_message)
```

### `initialize_observability(config, agent_name, verbose=False, quiet=False)`

Initialize all telemetry providers based on configuration.

Parameters:

| Name         | Type                  | Description                                                | Default    |
| ------------ | --------------------- | ---------------------------------------------------------- | ---------- |
| `config`     | `ObservabilityConfig` | Observability configuration from agent.yaml                | *required* |
| `agent_name` | `str`                 | Agent name from agent.yaml (used for default service name) | *required* |
| `verbose`    | `bool`                | If True, set log level to DEBUG                            | `False`    |
| `quiet`      | `bool`                | If True, set log level to WARNING (overrides verbose)      | `False`    |

Returns:

| Type                   | Description                                     |
| ---------------------- | ----------------------------------------------- |
| `ObservabilityContext` | ObservabilityContext with initialized providers |

Raises:

| Type                       | Description                 |
| -------------------------- | --------------------------- |
| `ObservabilityConfigError` | If configuration is invalid |

Note

Initialization order is critical:

1. Configure logging first (prevents double logging)
1. Set up logging provider
1. Set up tracing provider
1. Set up metrics provider

Example

> > > from holodeck.lib.observability import initialize_observability from holodeck.models.observability import ObservabilityConfig
> > >
> > > config = ObservabilityConfig(enabled=True) context = initialize_observability(config, agent_name="my-agent")

Source code in `src/holodeck/lib/observability/providers.py`

```
def initialize_observability(
    config: ObservabilityConfig,
    agent_name: str,
    verbose: bool = False,
    quiet: bool = False,
) -> ObservabilityContext:
    """Initialize all telemetry providers based on configuration.

    Args:
        config: Observability configuration from agent.yaml
        agent_name: Agent name from agent.yaml (used for default service name)
        verbose: If True, set log level to DEBUG
        quiet: If True, set log level to WARNING (overrides verbose)

    Returns:
        ObservabilityContext with initialized providers

    Raises:
        ObservabilityConfigError: If configuration is invalid

    Note:
        Initialization order is critical:
        1. Configure logging first (prevents double logging)
        2. Set up logging provider
        3. Set up tracing provider
        4. Set up metrics provider

    Example:
        >>> from holodeck.lib.observability import initialize_observability
        >>> from holodeck.models.observability import ObservabilityConfig
        >>>
        >>> config = ObservabilityConfig(enabled=True)
        >>> context = initialize_observability(config, agent_name="my-agent")
    """
    global _observability_context

    from holodeck.lib.observability.config import configure_exporters, configure_logging

    # 1. Create shared resource
    resource = create_resource(config, agent_name)

    # 2. Configure exporters (returns span, metric, log exporters)
    span_exporters, metric_readers, log_exporters, exporter_names = configure_exporters(
        config
    )

    # 3. Configure logging (prevents double logging with console exporter)
    configure_logging(config)

    # 4. Set up logging (must be first per OTel docs)
    logger_provider = set_up_logging(config, resource, log_exporters, verbose, quiet)

    # 5. Set up tracing
    tracer_provider = set_up_tracing(config, resource, span_exporters)

    # 6. Set up metrics
    meter_provider = set_up_metrics(config, resource, metric_readers)

    # 7. Create and store context
    _observability_context = ObservabilityContext(
        tracer_provider=tracer_provider,
        meter_provider=meter_provider,
        logger_provider=logger_provider,
        exporters=exporter_names,
        resource=resource,
    )

    # 8. Enable Semantic Kernel telemetry (GenAI semantic conventions)
    from holodeck.lib.observability.instrumentation import (
        enable_litellm_telemetry,
        enable_semantic_kernel_telemetry,
    )

    enable_semantic_kernel_telemetry(config)

    # 9. Enable LiteLLM telemetry (RAG embeddings + contextual retrieval).
    # Must run after set_up_tracing so the callback reuses the global
    # tracer provider (and its RedactingSpanProcessor).
    enable_litellm_telemetry(config)

    return _observability_context
```

### `shutdown_observability(context)`

Flush pending telemetry and shutdown providers.

Parameters:

| Name      | Type                   | Description                                        | Default    |
| --------- | ---------------------- | -------------------------------------------------- | ---------- |
| `context` | `ObservabilityContext` | ObservabilityContext from initialize_observability | *required* |

Note

Should be called during application shutdown. Blocks until all pending spans/metrics are flushed.

Source code in `src/holodeck/lib/observability/providers.py`

```
def shutdown_observability(context: ObservabilityContext) -> None:
    """Flush pending telemetry and shutdown providers.

    Args:
        context: ObservabilityContext from initialize_observability

    Note:
        Should be called during application shutdown.
        Blocks until all pending spans/metrics are flushed.
    """
    global _observability_context

    # Force flush all pending data before shutdown
    if context.tracer_provider:
        context.tracer_provider.force_flush()

    if context.meter_provider:
        context.meter_provider.force_flush()

    if context.logger_provider:
        context.logger_provider.force_flush()

    # Shutdown in reverse order of initialization
    if context.meter_provider:
        context.meter_provider.shutdown()

    if context.tracer_provider:
        context.tracer_provider.shutdown()

    if context.logger_provider:
        context.logger_provider.shutdown()

    _observability_context = None
```

### `get_tracer(name)`

Get an OpenTelemetry tracer instance.

Parameters:

| Name   | Type  | Description                  | Default    |
| ------ | ----- | ---------------------------- | ---------- |
| `name` | `str` | Tracer name (typically name) | *required* |

Returns:

| Type     | Description                   |
| -------- | ----------------------------- |
| `Tracer` | OpenTelemetry Tracer instance |

Example

> > > tracer = get_tracer(**name**) with tracer.start_as_current_span("my-operation"): ... # do work

Source code in `src/holodeck/lib/observability/providers.py`

```
def get_tracer(name: str) -> Tracer:
    """Get an OpenTelemetry tracer instance.

    Args:
        name: Tracer name (typically __name__)

    Returns:
        OpenTelemetry Tracer instance

    Example:
        >>> tracer = get_tracer(__name__)
        >>> with tracer.start_as_current_span("my-operation"):
        ...     # do work
    """
    return trace.get_tracer(name)
```

### `get_meter(name)`

Get an OpenTelemetry meter instance.

Parameters:

| Name   | Type  | Description                 | Default    |
| ------ | ----- | --------------------------- | ---------- |
| `name` | `str` | Meter name (typically name) | *required* |

Returns:

| Type    | Description                  |
| ------- | ---------------------------- |
| `Meter` | OpenTelemetry Meter instance |

Example

> > > meter = get_meter(**name**) counter = meter.create_counter("requests") counter.add(1)

Source code in `src/holodeck/lib/observability/providers.py`

```
def get_meter(name: str) -> Meter:
    """Get an OpenTelemetry meter instance.

    Args:
        name: Meter name (typically __name__)

    Returns:
        OpenTelemetry Meter instance

    Example:
        >>> meter = get_meter(__name__)
        >>> counter = meter.create_counter("requests")
        >>> counter.add(1)
    """
    return metrics.get_meter(name)
```

### `get_observability_context()`

Return the current ObservabilityContext, or None if not initialized.

Thread-safety note: This accessor reads module-level state that is set by `initialize_observability()` in the CLI layer's main thread, before `asyncio.run()` is called. All async tasks (including `ClaudeBackend.initialize()`) run in the same thread, so no synchronization is needed. If future code introduces background task spawning that accesses this state, thread synchronization will be required.

Source code in `src/holodeck/lib/observability/providers.py`

```
def get_observability_context() -> ObservabilityContext | None:
    """Return the current ObservabilityContext, or None if not initialized.

    Thread-safety note: This accessor reads module-level state that is set
    by ``initialize_observability()`` in the CLI layer's main thread, before
    ``asyncio.run()`` is called. All async tasks (including
    ``ClaudeBackend.initialize()``) run in the same thread, so no
    synchronization is needed. If future code introduces background task
    spawning that accesses this state, thread synchronization will be
    required.
    """
    return _observability_context
```

### `enable_semantic_kernel_telemetry(config)`

Enable Semantic Kernel's native OpenTelemetry instrumentation.

Sets environment variables that Semantic Kernel reads to enable telemetry. SK provides comprehensive GenAI semantic convention support, automatically capturing attributes like:

- gen_ai.operation.name (e.g., "chat.completions")
- gen_ai.system (e.g., "openai")
- gen_ai.request.model (e.g., "gpt-4o")
- gen_ai.response.id, gen_ai.response.finish_reason
- gen_ai.usage.prompt_tokens, gen_ai.usage.completion_tokens

When sensitive diagnostics is enabled, SK also captures:

- gen_ai.content.prompt (via span events)
- gen_ai.content.completion (via span events)

Parameters:

| Name     | Type                  | Description                              | Default    |
| -------- | --------------------- | ---------------------------------------- | ---------- |
| `config` | `ObservabilityConfig` | ObservabilityConfig with traces settings | *required* |

Note

This function must be called BEFORE any Semantic Kernel operations. SK reads these environment variables at initialization time.

Example

> > > from holodeck.models.observability import ObservabilityConfig config = ObservabilityConfig(enabled=True) enable_semantic_kernel_telemetry(config)
> > >
> > > #### SK will now emit GenAI semantic convention spans

Source code in `src/holodeck/lib/observability/instrumentation.py`

```
def enable_semantic_kernel_telemetry(config: ObservabilityConfig) -> None:
    """Enable Semantic Kernel's native OpenTelemetry instrumentation.

    Sets environment variables that Semantic Kernel reads to enable telemetry.
    SK provides comprehensive GenAI semantic convention support, automatically
    capturing attributes like:

    - gen_ai.operation.name (e.g., "chat.completions")
    - gen_ai.system (e.g., "openai")
    - gen_ai.request.model (e.g., "gpt-4o")
    - gen_ai.response.id, gen_ai.response.finish_reason
    - gen_ai.usage.prompt_tokens, gen_ai.usage.completion_tokens

    When sensitive diagnostics is enabled, SK also captures:
    - gen_ai.content.prompt (via span events)
    - gen_ai.content.completion (via span events)

    Args:
        config: ObservabilityConfig with traces settings

    Note:
        This function must be called BEFORE any Semantic Kernel operations.
        SK reads these environment variables at initialization time.

    Example:
        >>> from holodeck.models.observability import ObservabilityConfig
        >>> config = ObservabilityConfig(enabled=True)
        >>> enable_semantic_kernel_telemetry(config)
        >>> # SK will now emit GenAI semantic convention spans
    """
    # Always enable basic GenAI diagnostics when observability is on
    os.environ[SK_OTEL_DIAGNOSTICS_ENV] = "true"

    # Enable sensitive content capture if explicitly configured
    # This captures prompts and completions in span events
    if config.traces.capture_content:
        os.environ[SK_OTEL_SENSITIVE_ENV] = "true"
```

______________________________________________________________________

## Providers (`providers`)

Core provider setup and lifecycle management. Creates the OpenTelemetry `TracerProvider`, `MeterProvider`, and `LoggerProvider`, and exposes helper accessors.

## `ObservabilityContext(tracer_provider, meter_provider, logger_provider, exporters=list(), resource=Resource.create())`

Container for initialized observability components.

Holds references to all telemetry providers and tracks which exporters are active. Used for lifecycle management and provider access.

Attributes:

| Name              | Type             | Description                                                |
| ----------------- | ---------------- | ---------------------------------------------------------- |
| `tracer_provider` | \`TracerProvider | None\`                                                     |
| `meter_provider`  | \`MeterProvider  | None\`                                                     |
| `logger_provider` | `Any`            | OpenTelemetry LoggerProvider instance                      |
| `exporters`       | `list[str]`      | List of enabled exporter names (e.g., ["console", "otlp"]) |
| `resource`        | `Resource`       | Shared OpenTelemetry Resource                              |

### `get_resource()`

Get the shared OpenTelemetry resource.

Returns:

| Type       | Description                                    |
| ---------- | ---------------------------------------------- |
| `Resource` | The Resource instance shared by all providers. |

Source code in `src/holodeck/lib/observability/providers.py`

```
def get_resource(self) -> Resource:
    """Get the shared OpenTelemetry resource.

    Returns:
        The Resource instance shared by all providers.
    """
    return self.resource
```

### `is_enabled()`

Check if observability is active.

Returns:

| Type   | Description                                             |
| ------ | ------------------------------------------------------- |
| `bool` | True if all providers are initialized, False otherwise. |

Source code in `src/holodeck/lib/observability/providers.py`

```
def is_enabled(self) -> bool:
    """Check if observability is active.

    Returns:
        True if all providers are initialized, False otherwise.
    """
    return (
        self.tracer_provider is not None
        and self.meter_provider is not None
        and self.logger_provider is not None
    )
```

## `create_resource(config, agent_name)`

Create OpenTelemetry resource with service name and attributes.

Service name resolution order:

1. config.service_name (if provided)
1. f"holodeck-{agent_name}" (default)

Parameters:

| Name         | Type                  | Description                                                | Default    |
| ------------ | --------------------- | ---------------------------------------------------------- | ---------- |
| `config`     | `ObservabilityConfig` | Observability configuration from agent.yaml                | *required* |
| `agent_name` | `str`                 | Agent name from agent.yaml (used for default service name) | *required* |

Returns:

| Type       | Description                                                    |
| ---------- | -------------------------------------------------------------- |
| `Resource` | OpenTelemetry Resource with service name and custom attributes |

Example

> > > config = ObservabilityConfig(enabled=True) resource = create_resource(config, agent_name="customer-support")
> > >
> > > ### Service name is "holodeck-customer-support"

Source code in `src/holodeck/lib/observability/providers.py`

```
def create_resource(config: ObservabilityConfig, agent_name: str) -> Resource:
    """Create OpenTelemetry resource with service name and attributes.

    Service name resolution order:
    1. config.service_name (if provided)
    2. f"holodeck-{agent_name}" (default)

    Args:
        config: Observability configuration from agent.yaml
        agent_name: Agent name from agent.yaml (used for default service name)

    Returns:
        OpenTelemetry Resource with service name and custom attributes

    Example:
        >>> config = ObservabilityConfig(enabled=True)
        >>> resource = create_resource(config, agent_name="customer-support")
        >>> # Service name is "holodeck-customer-support"
    """
    service_name = config.service_name or f"holodeck-{agent_name}"

    attributes: dict[str, Any] = {
        "service.name": service_name,
        **config.resource_attributes,
    }

    return Resource.create(attributes)
```

## `set_up_logging(config, resource, log_exporters, verbose=False, quiet=False)`

Set up OpenTelemetry LoggerProvider and bridge Python logging.

Must be called FIRST before tracing and metrics per OTel Python docs.

Parameters:

| Name            | Type                  | Description                                           | Default    |
| --------------- | --------------------- | ----------------------------------------------------- | ---------- |
| `config`        | `ObservabilityConfig` | Observability configuration                           | *required* |
| `resource`      | `Resource`            | Shared OpenTelemetry Resource                         | *required* |
| `log_exporters` | `list[Any]`           | List of log exporters to add                          | *required* |
| `verbose`       | `bool`                | If True, set log level to DEBUG                       | `False`    |
| `quiet`         | `bool`                | If True, set log level to WARNING (overrides verbose) | `False`    |

Returns:

| Type  | Description                        |
| ----- | ---------------------------------- |
| `Any` | Configured LoggerProvider instance |

Source code in `src/holodeck/lib/observability/providers.py`

```
def set_up_logging(
    config: ObservabilityConfig,
    resource: Resource,
    log_exporters: list[Any],
    verbose: bool = False,
    quiet: bool = False,
) -> Any:
    """Set up OpenTelemetry LoggerProvider and bridge Python logging.

    Must be called FIRST before tracing and metrics per OTel Python docs.

    Args:
        config: Observability configuration
        resource: Shared OpenTelemetry Resource
        log_exporters: List of log exporters to add
        verbose: If True, set log level to DEBUG
        quiet: If True, set log level to WARNING (overrides verbose)

    Returns:
        Configured LoggerProvider instance
    """
    import logging

    from opentelemetry._logs import set_logger_provider
    from opentelemetry.sdk._logs import LoggerProvider, LoggingHandler
    from opentelemetry.sdk._logs.export import BatchLogRecordProcessor

    # Determine log level based on flags (same logic as setup_logging)
    if quiet:
        log_level = logging.WARNING
    elif verbose:
        log_level = logging.DEBUG
    else:
        log_level = logging.INFO

    logger_provider = LoggerProvider(resource=resource)

    # Add all log exporters
    for exporter in log_exporters:
        logger_provider.add_log_record_processor(BatchLogRecordProcessor(exporter))

    set_logger_provider(logger_provider)

    # Bridge Python's logging module to OTel
    # This handler captures Python log records and sends them to OTel
    otel_handler = LoggingHandler(
        level=log_level,
        logger_provider=logger_provider,
    )

    # Add OTel handler to root logger so all Python logs are captured
    # Note: Child loggers (like "holodeck") propagate to root by default,
    # so we only need to add the handler here to avoid duplicate records.
    logging.getLogger().addHandler(otel_handler)
    logging.getLogger().setLevel(log_level)

    # Configure third-party loggers to respect verbosity settings
    from holodeck.lib.logging_config import configure_third_party_loggers

    configure_third_party_loggers(log_level)

    return logger_provider
```

## `set_up_tracing(config, resource, span_exporters)`

Set up OpenTelemetry TracerProvider with span exporters.

Creates a TracerProvider with the resource and registers it globally. If a TracerProvider was already set by another library, we use that existing provider and add our span processors to it.

Parameters:

| Name             | Type                  | Description                   | Default    |
| ---------------- | --------------------- | ----------------------------- | ---------- |
| `config`         | `ObservabilityConfig` | Observability configuration   | *required* |
| `resource`       | `Resource`            | Shared OpenTelemetry Resource | *required* |
| `span_exporters` | `list[Any]`           | List of span exporters to add | *required* |

Returns:

| Type             | Description                        |
| ---------------- | ---------------------------------- |
| `TracerProvider` | Configured TracerProvider instance |

Note

This must be called before any code that creates spans. The SEMANTICKERNEL_EXPERIMENTAL_GENAI_ENABLE_OTEL_DIAGNOSTICS env var should be set in main.py before any SK imports.

Source code in `src/holodeck/lib/observability/providers.py`

```
def set_up_tracing(
    config: ObservabilityConfig,
    resource: Resource,
    span_exporters: list[Any],
) -> TracerProvider:
    """Set up OpenTelemetry TracerProvider with span exporters.

    Creates a TracerProvider with the resource and registers it globally.
    If a TracerProvider was already set by another library, we use that
    existing provider and add our span processors to it.

    Args:
        config: Observability configuration
        resource: Shared OpenTelemetry Resource
        span_exporters: List of span exporters to add

    Returns:
        Configured TracerProvider instance

    Note:
        This must be called before any code that creates spans. The
        SEMANTICKERNEL_EXPERIMENTAL_GENAI_ENABLE_OTEL_DIAGNOSTICS env var
        should be set in main.py before any SK imports.
    """
    from opentelemetry.sdk.trace.export import BatchSpanProcessor

    from holodeck.lib.backends.otel_redaction import RedactingSpanProcessor

    # Check if a real TracerProvider was already set by another library
    existing_provider = trace.get_tracer_provider()

    if isinstance(existing_provider, TracerProvider):
        # Another library already set a TracerProvider - use it
        # We can still add our span processors to capture telemetry
        tracer_provider = existing_provider
    else:
        # No real provider set yet - create ours with the resource
        tracer_provider = TracerProvider(resource=resource)
        trace.set_tracer_provider(tracer_provider)

    # Register RedactingSpanProcessor first so it runs before any exporter.
    # Guard against duplicates: if this provider was already configured by a
    # prior set_up_tracing call, skip adding a second instance.
    existing_processors = _get_span_processors(tracer_provider)
    if not any(isinstance(p, RedactingSpanProcessor) for p in existing_processors):
        tracer_provider.add_span_processor(RedactingSpanProcessor())

    # Configure batch processor settings from config
    for exporter in span_exporters:
        processor = BatchSpanProcessor(
            exporter,
            max_queue_size=config.traces.max_queue_size,
            max_export_batch_size=config.traces.max_export_batch_size,
            schedule_delay_millis=config.traces.schedule_delay_millis,
        )
        tracer_provider.add_span_processor(processor)

    return tracer_provider
```

## `set_up_metrics(config, resource, metric_readers)`

Set up OpenTelemetry MeterProvider with metric readers.

Parameters:

| Name             | Type                  | Description                   | Default    |
| ---------------- | --------------------- | ----------------------------- | ---------- |
| `config`         | `ObservabilityConfig` | Observability configuration   | *required* |
| `resource`       | `Resource`            | Shared OpenTelemetry Resource | *required* |
| `metric_readers` | `list[Any]`           | List of metric readers to add | *required* |

Returns:

| Type            | Description                       |
| --------------- | --------------------------------- |
| `MeterProvider` | Configured MeterProvider instance |

Source code in `src/holodeck/lib/observability/providers.py`

```
def set_up_metrics(
    config: ObservabilityConfig,
    resource: Resource,
    metric_readers: list[Any],
) -> MeterProvider:
    """Set up OpenTelemetry MeterProvider with metric readers.

    Args:
        config: Observability configuration
        resource: Shared OpenTelemetry Resource
        metric_readers: List of metric readers to add

    Returns:
        Configured MeterProvider instance
    """
    meter_provider = MeterProvider(
        resource=resource,
        metric_readers=metric_readers,
    )

    metrics.set_meter_provider(meter_provider)

    return meter_provider
```

## `initialize_observability(config, agent_name, verbose=False, quiet=False)`

Initialize all telemetry providers based on configuration.

Parameters:

| Name         | Type                  | Description                                                | Default    |
| ------------ | --------------------- | ---------------------------------------------------------- | ---------- |
| `config`     | `ObservabilityConfig` | Observability configuration from agent.yaml                | *required* |
| `agent_name` | `str`                 | Agent name from agent.yaml (used for default service name) | *required* |
| `verbose`    | `bool`                | If True, set log level to DEBUG                            | `False`    |
| `quiet`      | `bool`                | If True, set log level to WARNING (overrides verbose)      | `False`    |

Returns:

| Type                   | Description                                     |
| ---------------------- | ----------------------------------------------- |
| `ObservabilityContext` | ObservabilityContext with initialized providers |

Raises:

| Type                       | Description                 |
| -------------------------- | --------------------------- |
| `ObservabilityConfigError` | If configuration is invalid |

Note

Initialization order is critical:

1. Configure logging first (prevents double logging)
1. Set up logging provider
1. Set up tracing provider
1. Set up metrics provider

Example

> > > from holodeck.lib.observability import initialize_observability from holodeck.models.observability import ObservabilityConfig
> > >
> > > config = ObservabilityConfig(enabled=True) context = initialize_observability(config, agent_name="my-agent")

Source code in `src/holodeck/lib/observability/providers.py`

```
def initialize_observability(
    config: ObservabilityConfig,
    agent_name: str,
    verbose: bool = False,
    quiet: bool = False,
) -> ObservabilityContext:
    """Initialize all telemetry providers based on configuration.

    Args:
        config: Observability configuration from agent.yaml
        agent_name: Agent name from agent.yaml (used for default service name)
        verbose: If True, set log level to DEBUG
        quiet: If True, set log level to WARNING (overrides verbose)

    Returns:
        ObservabilityContext with initialized providers

    Raises:
        ObservabilityConfigError: If configuration is invalid

    Note:
        Initialization order is critical:
        1. Configure logging first (prevents double logging)
        2. Set up logging provider
        3. Set up tracing provider
        4. Set up metrics provider

    Example:
        >>> from holodeck.lib.observability import initialize_observability
        >>> from holodeck.models.observability import ObservabilityConfig
        >>>
        >>> config = ObservabilityConfig(enabled=True)
        >>> context = initialize_observability(config, agent_name="my-agent")
    """
    global _observability_context

    from holodeck.lib.observability.config import configure_exporters, configure_logging

    # 1. Create shared resource
    resource = create_resource(config, agent_name)

    # 2. Configure exporters (returns span, metric, log exporters)
    span_exporters, metric_readers, log_exporters, exporter_names = configure_exporters(
        config
    )

    # 3. Configure logging (prevents double logging with console exporter)
    configure_logging(config)

    # 4. Set up logging (must be first per OTel docs)
    logger_provider = set_up_logging(config, resource, log_exporters, verbose, quiet)

    # 5. Set up tracing
    tracer_provider = set_up_tracing(config, resource, span_exporters)

    # 6. Set up metrics
    meter_provider = set_up_metrics(config, resource, metric_readers)

    # 7. Create and store context
    _observability_context = ObservabilityContext(
        tracer_provider=tracer_provider,
        meter_provider=meter_provider,
        logger_provider=logger_provider,
        exporters=exporter_names,
        resource=resource,
    )

    # 8. Enable Semantic Kernel telemetry (GenAI semantic conventions)
    from holodeck.lib.observability.instrumentation import (
        enable_litellm_telemetry,
        enable_semantic_kernel_telemetry,
    )

    enable_semantic_kernel_telemetry(config)

    # 9. Enable LiteLLM telemetry (RAG embeddings + contextual retrieval).
    # Must run after set_up_tracing so the callback reuses the global
    # tracer provider (and its RedactingSpanProcessor).
    enable_litellm_telemetry(config)

    return _observability_context
```

## `shutdown_observability(context)`

Flush pending telemetry and shutdown providers.

Parameters:

| Name      | Type                   | Description                                        | Default    |
| --------- | ---------------------- | -------------------------------------------------- | ---------- |
| `context` | `ObservabilityContext` | ObservabilityContext from initialize_observability | *required* |

Note

Should be called during application shutdown. Blocks until all pending spans/metrics are flushed.

Source code in `src/holodeck/lib/observability/providers.py`

```
def shutdown_observability(context: ObservabilityContext) -> None:
    """Flush pending telemetry and shutdown providers.

    Args:
        context: ObservabilityContext from initialize_observability

    Note:
        Should be called during application shutdown.
        Blocks until all pending spans/metrics are flushed.
    """
    global _observability_context

    # Force flush all pending data before shutdown
    if context.tracer_provider:
        context.tracer_provider.force_flush()

    if context.meter_provider:
        context.meter_provider.force_flush()

    if context.logger_provider:
        context.logger_provider.force_flush()

    # Shutdown in reverse order of initialization
    if context.meter_provider:
        context.meter_provider.shutdown()

    if context.tracer_provider:
        context.tracer_provider.shutdown()

    if context.logger_provider:
        context.logger_provider.shutdown()

    _observability_context = None
```

## `get_tracer(name)`

Get an OpenTelemetry tracer instance.

Parameters:

| Name   | Type  | Description                  | Default    |
| ------ | ----- | ---------------------------- | ---------- |
| `name` | `str` | Tracer name (typically name) | *required* |

Returns:

| Type     | Description                   |
| -------- | ----------------------------- |
| `Tracer` | OpenTelemetry Tracer instance |

Example

> > > tracer = get_tracer(**name**) with tracer.start_as_current_span("my-operation"): ... # do work

Source code in `src/holodeck/lib/observability/providers.py`

```
def get_tracer(name: str) -> Tracer:
    """Get an OpenTelemetry tracer instance.

    Args:
        name: Tracer name (typically __name__)

    Returns:
        OpenTelemetry Tracer instance

    Example:
        >>> tracer = get_tracer(__name__)
        >>> with tracer.start_as_current_span("my-operation"):
        ...     # do work
    """
    return trace.get_tracer(name)
```

## `get_meter(name)`

Get an OpenTelemetry meter instance.

Parameters:

| Name   | Type  | Description                 | Default    |
| ------ | ----- | --------------------------- | ---------- |
| `name` | `str` | Meter name (typically name) | *required* |

Returns:

| Type    | Description                  |
| ------- | ---------------------------- |
| `Meter` | OpenTelemetry Meter instance |

Example

> > > meter = get_meter(**name**) counter = meter.create_counter("requests") counter.add(1)

Source code in `src/holodeck/lib/observability/providers.py`

```
def get_meter(name: str) -> Meter:
    """Get an OpenTelemetry meter instance.

    Args:
        name: Meter name (typically __name__)

    Returns:
        OpenTelemetry Meter instance

    Example:
        >>> meter = get_meter(__name__)
        >>> counter = meter.create_counter("requests")
        >>> counter.add(1)
    """
    return metrics.get_meter(name)
```

## `get_observability_context()`

Return the current ObservabilityContext, or None if not initialized.

Thread-safety note: This accessor reads module-level state that is set by `initialize_observability()` in the CLI layer's main thread, before `asyncio.run()` is called. All async tasks (including `ClaudeBackend.initialize()`) run in the same thread, so no synchronization is needed. If future code introduces background task spawning that accesses this state, thread synchronization will be required.

Source code in `src/holodeck/lib/observability/providers.py`

```
def get_observability_context() -> ObservabilityContext | None:
    """Return the current ObservabilityContext, or None if not initialized.

    Thread-safety note: This accessor reads module-level state that is set
    by ``initialize_observability()`` in the CLI layer's main thread, before
    ``asyncio.run()`` is called. All async tasks (including
    ``ClaudeBackend.initialize()``) run in the same thread, so no
    synchronization is needed. If future code introduces background task
    spawning that accesses this state, thread synchronization will be
    required.
    """
    return _observability_context
```

______________________________________________________________________

## Configuration (`config`)

Exporter configuration and logging coordination. Prevents double logging when the console exporter is active, and builds the exporter lists consumed by the provider setup functions.

## `is_console_exporter_active(config)`

Check if console exporter is active.

Console exporter is active when:

- Explicitly enabled in configuration, OR
- No other exporters are enabled (console is the default fallback)

Parameters:

| Name     | Type                  | Description                 | Default    |
| -------- | --------------------- | --------------------------- | ---------- |
| `config` | `ObservabilityConfig` | Observability configuration | *required* |

Returns:

| Type   | Description                                          |
| ------ | ---------------------------------------------------- |
| `bool` | True if console exporter is active, False otherwise. |

Source code in `src/holodeck/lib/observability/config.py`

```
def is_console_exporter_active(config: ObservabilityConfig) -> bool:
    """Check if console exporter is active.

    Console exporter is active when:
    - Explicitly enabled in configuration, OR
    - No other exporters are enabled (console is the default fallback)

    Args:
        config: Observability configuration

    Returns:
        True if console exporter is active, False otherwise.
    """
    # Explicitly enabled
    if config.exporters.console and config.exporters.console.enabled:
        return True

    # Default to console when no other exporters are configured
    return not _any_exporter_enabled(config)
```

## `configure_logging(config)`

Configure logging to prevent duplicates with console exporter.

When console exporter is active, removes default StreamHandlers from the holodeck logger to prevent duplicate output.

Parameters:

| Name     | Type                  | Description                 | Default    |
| -------- | --------------------- | --------------------------- | ---------- |
| `config` | `ObservabilityConfig` | Observability configuration | *required* |

Note

Called automatically by initialize_observability().

Source code in `src/holodeck/lib/observability/config.py`

```
def configure_logging(config: ObservabilityConfig) -> None:
    """Configure logging to prevent duplicates with console exporter.

    When console exporter is active, removes default StreamHandlers
    from the holodeck logger to prevent duplicate output.

    Args:
        config: Observability configuration

    Note:
        Called automatically by initialize_observability().
    """
    if not is_console_exporter_active(config):
        return

    holodeck_logger = logging.getLogger("holodeck")

    # Remove console handlers to prevent double logging
    for handler in holodeck_logger.handlers[:]:
        if isinstance(handler, logging.StreamHandler) and handler.stream in (
            sys.stdout,
            sys.stderr,
        ):
            holodeck_logger.removeHandler(handler)
```

## `configure_exporters(config)`

Configure all explicitly enabled exporters.

Parameters:

| Name     | Type                  | Description                 | Default    |
| -------- | --------------------- | --------------------------- | ---------- |
| `config` | `ObservabilityConfig` | Observability configuration | *required* |

Returns:

| Type                                                | Description                                                              |
| --------------------------------------------------- | ------------------------------------------------------------------------ |
| `tuple[list[Any], list[Any], list[Any], list[str]]` | Tuple of (span_exporters, metric_readers, log_exporters, exporter_names) |

Note

Only exporters that are explicitly enabled in configuration are added. The serve command enables console exporter by default for server logging.

Source code in `src/holodeck/lib/observability/config.py`

```
def configure_exporters(
    config: ObservabilityConfig,
) -> tuple[list[Any], list[Any], list[Any], list[str]]:
    """Configure all explicitly enabled exporters.

    Args:
        config: Observability configuration

    Returns:
        Tuple of (span_exporters, metric_readers, log_exporters, exporter_names)

    Note:
        Only exporters that are explicitly enabled in configuration are added.
        The serve command enables console exporter by default for server logging.
    """
    from holodeck.lib.observability.exporters.console import create_console_exporters

    span_exporters: list[Any] = []
    metric_readers: list[Any] = []
    log_exporters: list[Any] = []
    exporter_names: list[str] = []

    # OTLP exporter (Phase 5 - US2)
    if config.exporters.otlp and config.exporters.otlp.enabled:
        from holodeck.lib.observability.exporters.otlp import create_otlp_exporters

        otlp_span, otlp_metric_reader, otlp_log = create_otlp_exporters(
            config.exporters.otlp
        )
        span_exporters.append(otlp_span)
        metric_readers.append(otlp_metric_reader)
        log_exporters.append(otlp_log)
        exporter_names.append("otlp")

    # Prometheus exporter (Phase 6 - US3)
    if config.exporters.prometheus and config.exporters.prometheus.enabled:
        # TODO: Will be implemented in Phase 6 (US3)
        exporter_names.append("prometheus")

    # Azure Monitor exporter (Phase 7 - US4)
    if config.exporters.azure_monitor and config.exporters.azure_monitor.enabled:
        # TODO: Will be implemented in Phase 7 (US4)
        exporter_names.append("azure_monitor")

    # Console exporter (explicitly enabled or default when no others configured)
    console_explicitly_enabled = (
        config.exporters.console and config.exporters.console.enabled
    )
    console_as_default = not _any_exporter_enabled(config)

    if console_explicitly_enabled or console_as_default:
        from holodeck.models.observability import ConsoleExporterConfig

        console_config = config.exporters.console or ConsoleExporterConfig()
        console_span, console_metric_reader, console_log = create_console_exporters(
            console_config
        )
        span_exporters.append(console_span)
        metric_readers.append(console_metric_reader)
        log_exporters.append(console_log)
        if "console" not in exporter_names:
            exporter_names.append("console")

    return span_exporters, metric_readers, log_exporters, exporter_names
```

______________________________________________________________________

## Instrumentation (`instrumentation`)

Wires up the internal instrumentation layer so it emits GenAI semantic-convention spans. Sets the environment variables that the underlying connector library reads at startup to turn on diagnostic and sensitive-content tracing.

## `enable_semantic_kernel_telemetry(config)`

Enable Semantic Kernel's native OpenTelemetry instrumentation.

Sets environment variables that Semantic Kernel reads to enable telemetry. SK provides comprehensive GenAI semantic convention support, automatically capturing attributes like:

- gen_ai.operation.name (e.g., "chat.completions")
- gen_ai.system (e.g., "openai")
- gen_ai.request.model (e.g., "gpt-4o")
- gen_ai.response.id, gen_ai.response.finish_reason
- gen_ai.usage.prompt_tokens, gen_ai.usage.completion_tokens

When sensitive diagnostics is enabled, SK also captures:

- gen_ai.content.prompt (via span events)
- gen_ai.content.completion (via span events)

Parameters:

| Name     | Type                  | Description                              | Default    |
| -------- | --------------------- | ---------------------------------------- | ---------- |
| `config` | `ObservabilityConfig` | ObservabilityConfig with traces settings | *required* |

Note

This function must be called BEFORE any Semantic Kernel operations. SK reads these environment variables at initialization time.

Example

> > > from holodeck.models.observability import ObservabilityConfig config = ObservabilityConfig(enabled=True) enable_semantic_kernel_telemetry(config)
> > >
> > > ### SK will now emit GenAI semantic convention spans

Source code in `src/holodeck/lib/observability/instrumentation.py`

```
def enable_semantic_kernel_telemetry(config: ObservabilityConfig) -> None:
    """Enable Semantic Kernel's native OpenTelemetry instrumentation.

    Sets environment variables that Semantic Kernel reads to enable telemetry.
    SK provides comprehensive GenAI semantic convention support, automatically
    capturing attributes like:

    - gen_ai.operation.name (e.g., "chat.completions")
    - gen_ai.system (e.g., "openai")
    - gen_ai.request.model (e.g., "gpt-4o")
    - gen_ai.response.id, gen_ai.response.finish_reason
    - gen_ai.usage.prompt_tokens, gen_ai.usage.completion_tokens

    When sensitive diagnostics is enabled, SK also captures:
    - gen_ai.content.prompt (via span events)
    - gen_ai.content.completion (via span events)

    Args:
        config: ObservabilityConfig with traces settings

    Note:
        This function must be called BEFORE any Semantic Kernel operations.
        SK reads these environment variables at initialization time.

    Example:
        >>> from holodeck.models.observability import ObservabilityConfig
        >>> config = ObservabilityConfig(enabled=True)
        >>> enable_semantic_kernel_telemetry(config)
        >>> # SK will now emit GenAI semantic convention spans
    """
    # Always enable basic GenAI diagnostics when observability is on
    os.environ[SK_OTEL_DIAGNOSTICS_ENV] = "true"

    # Enable sensitive content capture if explicitly configured
    # This captures prompts and completions in span events
    if config.traces.capture_content:
        os.environ[SK_OTEL_SENSITIVE_ENV] = "true"
```

## `SK_OTEL_DIAGNOSTICS_ENV = 'SEMANTICKERNEL_EXPERIMENTAL_GENAI_ENABLE_OTEL_DIAGNOSTICS'`

## `SK_OTEL_SENSITIVE_ENV = 'SEMANTICKERNEL_EXPERIMENTAL_GENAI_ENABLE_OTEL_DIAGNOSTICS_SENSITIVE'`

______________________________________________________________________

## Errors (`errors`)

Custom exception hierarchy for observability failures.

## `ObservabilityError(message)`

Bases: `HoloDeckError`

Base exception for all observability-related errors.

All observability-specific exceptions inherit from this class, enabling centralized exception handling for telemetry operations.

Attributes:

| Name      | Type | Description                  |
| --------- | ---- | ---------------------------- |
| `message` |      | Human-readable error message |

Initialize ObservabilityError with message.

Parameters:

| Name      | Type  | Description               | Default    |
| --------- | ----- | ------------------------- | ---------- |
| `message` | `str` | Descriptive error message | *required* |

Source code in `src/holodeck/lib/observability/errors.py`

```
def __init__(self, message: str) -> None:
    """Initialize ObservabilityError with message.

    Args:
        message: Descriptive error message
    """
    self.message = message
    super().__init__(message)
```

## `ObservabilityConfigError(field, message)`

Bases: `ObservabilityError`

Exception raised for observability configuration errors.

Raised when observability configuration is invalid or incomplete, such as missing required fields or invalid exporter settings.

Attributes:

| Name      | Type | Description                                   |
| --------- | ---- | --------------------------------------------- |
| `field`   |      | The configuration field that caused the error |
| `message` |      | Human-readable error message                  |

Initialize ObservabilityConfigError with field and message.

Parameters:

| Name      | Type  | Description                                   | Default    |
| --------- | ----- | --------------------------------------------- | ---------- |
| `field`   | `str` | Configuration field name where error occurred | *required* |
| `message` | `str` | Descriptive error message                     | *required* |

Source code in `src/holodeck/lib/observability/errors.py`

```
def __init__(self, field: str, message: str) -> None:
    """Initialize ObservabilityConfigError with field and message.

    Args:
        field: Configuration field name where error occurred
        message: Descriptive error message
    """
    self.field = field
    full_message = f"Observability configuration error in '{field}': {message}"
    super().__init__(full_message)
```

______________________________________________________________________

## Exporters

### Console exporter (`exporters.console`)

Development/debugging exporter that writes telemetry to stdout. Used as the default fallback when no other exporters are configured.

## `create_console_exporters(config)`

Create all console exporters (spans, metrics, logs).

Factory function that creates all three exporter types for the console exporter configuration.

Parameters:

| Name     | Type                    | Description                    | Default    |
| -------- | ----------------------- | ------------------------------ | ---------- |
| `config` | `ConsoleExporterConfig` | Console exporter configuration | *required* |

Returns:

| Type                                                                                  | Description                                           |
| ------------------------------------------------------------------------------------- | ----------------------------------------------------- |
| `tuple[ConsoleSpanExporter, PeriodicExportingMetricReader, ConsoleLogRecordExporter]` | Tuple of (span_exporter, metric_reader, log_exporter) |

Example

> > > from holodeck.models.observability import ConsoleExporterConfig config = ConsoleExporterConfig() span_exp, metric_reader, log_exp = create_console_exporters(config)

Source code in `src/holodeck/lib/observability/exporters/console.py`

```
def create_console_exporters(
    config: ConsoleExporterConfig,
) -> tuple[
    ConsoleSpanExporter, PeriodicExportingMetricReader, ConsoleLogRecordExporter
]:
    """Create all console exporters (spans, metrics, logs).

    Factory function that creates all three exporter types for
    the console exporter configuration.

    Args:
        config: Console exporter configuration

    Returns:
        Tuple of (span_exporter, metric_reader, log_exporter)

    Example:
        >>> from holodeck.models.observability import ConsoleExporterConfig
        >>> config = ConsoleExporterConfig()
        >>> span_exp, metric_reader, log_exp = create_console_exporters(config)
    """
    span_exporter = create_console_span_exporter(config)
    metric_reader = create_console_metric_reader(config)
    log_exporter = create_console_log_exporter(config)

    return span_exporter, metric_reader, log_exporter
```

## `create_console_span_exporter(config)`

Create a console span exporter.

Parameters:

| Name     | Type                    | Description                    | Default    |
| -------- | ----------------------- | ------------------------------ | ---------- |
| `config` | `ConsoleExporterConfig` | Console exporter configuration | *required* |

Returns:

| Type                  | Description                             |
| --------------------- | --------------------------------------- |
| `ConsoleSpanExporter` | Configured ConsoleSpanExporter instance |

Source code in `src/holodeck/lib/observability/exporters/console.py`

```
def create_console_span_exporter(config: ConsoleExporterConfig) -> ConsoleSpanExporter:
    """Create a console span exporter.

    Args:
        config: Console exporter configuration

    Returns:
        Configured ConsoleSpanExporter instance
    """
    return ConsoleSpanExporter()
```

## `create_console_metric_reader(config)`

Create a console metric reader.

Parameters:

| Name     | Type                    | Description                    | Default    |
| -------- | ----------------------- | ------------------------------ | ---------- |
| `config` | `ConsoleExporterConfig` | Console exporter configuration | *required* |

Returns:

| Type                            | Description                                              |
| ------------------------------- | -------------------------------------------------------- |
| `PeriodicExportingMetricReader` | PeriodicExportingMetricReader with ConsoleMetricExporter |

Source code in `src/holodeck/lib/observability/exporters/console.py`

```
def create_console_metric_reader(
    config: ConsoleExporterConfig,
) -> PeriodicExportingMetricReader:
    """Create a console metric reader.

    Args:
        config: Console exporter configuration

    Returns:
        PeriodicExportingMetricReader with ConsoleMetricExporter
    """
    exporter = ConsoleMetricExporter()
    return PeriodicExportingMetricReader(exporter)
```

## `create_console_log_exporter(config)`

Create a console log exporter.

Parameters:

| Name     | Type                    | Description                    | Default    |
| -------- | ----------------------- | ------------------------------ | ---------- |
| `config` | `ConsoleExporterConfig` | Console exporter configuration | *required* |

Returns:

| Type                       | Description                                  |
| -------------------------- | -------------------------------------------- |
| `ConsoleLogRecordExporter` | Configured ConsoleLogRecordExporter instance |

Source code in `src/holodeck/lib/observability/exporters/console.py`

```
def create_console_log_exporter(
    config: ConsoleExporterConfig,
) -> ConsoleLogRecordExporter:
    """Create a console log exporter.

    Args:
        config: Console exporter configuration

    Returns:
        Configured ConsoleLogRecordExporter instance
    """
    return ConsoleLogRecordExporter()
```

### OTLP exporter (`exporters.otlp`)

Exports telemetry via OTLP (gRPC or HTTP) to any compatible collector such as Jaeger, Honeycomb, or Datadog.

## `create_otlp_exporters(config)`

Create all OTLP exporters (spans, metrics, logs).

Factory function that creates all three exporter types for the OTLP exporter configuration.

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type                                             | Description                                           |
| ------------------------------------------------ | ----------------------------------------------------- |
| `tuple[Any, PeriodicExportingMetricReader, Any]` | Tuple of (span_exporter, metric_reader, log_exporter) |

Example

> > > from holodeck.models.observability import OTLPExporterConfig config = OTLPExporterConfig(endpoint="http://localhost:4317") span_exp, metric_reader, log_exp = create_otlp_exporters(config)

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_exporters(
    config: OTLPExporterConfig,
) -> tuple[Any, PeriodicExportingMetricReader, Any]:
    """Create all OTLP exporters (spans, metrics, logs).

    Factory function that creates all three exporter types for
    the OTLP exporter configuration.

    Args:
        config: OTLP exporter configuration

    Returns:
        Tuple of (span_exporter, metric_reader, log_exporter)

    Example:
        >>> from holodeck.models.observability import OTLPExporterConfig
        >>> config = OTLPExporterConfig(endpoint="http://localhost:4317")
        >>> span_exp, metric_reader, log_exp = create_otlp_exporters(config)
    """
    span_exporter = create_otlp_span_exporter(config)
    metric_reader = create_otlp_metric_reader(config)
    log_exporter = create_otlp_log_exporter(config)

    return span_exporter, metric_reader, log_exporter
```

## `create_otlp_span_exporter(config)`

Create OTLP span exporter based on protocol.

Dispatches to gRPC or HTTP implementation based on config.protocol.

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type  | Description                              |
| ----- | ---------------------------------------- |
| `Any` | OTLPSpanExporter instance (gRPC or HTTP) |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_span_exporter(config: OTLPExporterConfig) -> Any:
    """Create OTLP span exporter based on protocol.

    Dispatches to gRPC or HTTP implementation based on config.protocol.

    Args:
        config: OTLP exporter configuration

    Returns:
        OTLPSpanExporter instance (gRPC or HTTP)
    """
    from holodeck.models.observability import OTLPProtocol

    if config.protocol == OTLPProtocol.GRPC:
        return create_otlp_span_exporter_grpc(config)
    else:
        return create_otlp_span_exporter_http(config)
```

## `create_otlp_span_exporter_grpc(config)`

Create OTLP span exporter using gRPC protocol.

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type               | Description                      |
| ------------------ | -------------------------------- |
| `OTLPSpanExporter` | OTLPSpanExporter (gRPC) instance |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_span_exporter_grpc(config: OTLPExporterConfig) -> OTLPSpanExporterGRPC:
    """Create OTLP span exporter using gRPC protocol.

    Args:
        config: OTLP exporter configuration

    Returns:
        OTLPSpanExporter (gRPC) instance
    """
    endpoint = adjust_endpoint_for_protocol(config.endpoint, config.protocol)
    headers = resolve_headers(config.headers) if config.headers else None
    grpc_headers = _headers_to_grpc_metadata(headers)
    compression = get_compression_grpc(config.compression)
    timeout_seconds = config.timeout_ms / 1000.0

    return OTLPSpanExporterGRPC(
        endpoint=endpoint,
        insecure=config.insecure,
        headers=grpc_headers,
        timeout=timeout_seconds,
        compression=compression,
    )
```

## `create_otlp_span_exporter_http(config)`

Create OTLP span exporter using HTTP protocol.

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type               | Description                      |
| ------------------ | -------------------------------- |
| `OTLPSpanExporter` | OTLPSpanExporter (HTTP) instance |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_span_exporter_http(config: OTLPExporterConfig) -> OTLPSpanExporterHTTP:
    """Create OTLP span exporter using HTTP protocol.

    Args:
        config: OTLP exporter configuration

    Returns:
        OTLPSpanExporter (HTTP) instance
    """
    endpoint = adjust_endpoint_for_protocol(config.endpoint, config.protocol)

    # HTTP endpoint needs /v1/traces suffix
    if not endpoint.endswith("/v1/traces"):
        endpoint = f"{endpoint.rstrip('/')}/v1/traces"

    headers = resolve_headers(config.headers) if config.headers else None
    compression = get_compression_http(config.compression)
    timeout_seconds = config.timeout_ms / 1000.0

    return OTLPSpanExporterHTTP(
        endpoint=endpoint,
        headers=headers,
        timeout=timeout_seconds,
        compression=compression,
    )
```

## `create_otlp_metric_reader(config)`

Create OTLP metric reader (wraps exporter in PeriodicExportingMetricReader).

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type                            | Description                                      |
| ------------------------------- | ------------------------------------------------ |
| `PeriodicExportingMetricReader` | PeriodicExportingMetricReader with OTLP exporter |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_metric_reader(
    config: OTLPExporterConfig,
) -> PeriodicExportingMetricReader:
    """Create OTLP metric reader (wraps exporter in PeriodicExportingMetricReader).

    Args:
        config: OTLP exporter configuration

    Returns:
        PeriodicExportingMetricReader with OTLP exporter
    """
    from holodeck.models.observability import OTLPProtocol

    exporter: OTLPMetricExporterGRPC | OTLPMetricExporterHTTP
    if config.protocol == OTLPProtocol.GRPC:
        exporter = create_otlp_metric_exporter_grpc(config)
    else:
        exporter = create_otlp_metric_exporter_http(config)

    return PeriodicExportingMetricReader(exporter)
```

## `create_otlp_metric_exporter_grpc(config)`

Create OTLP metric exporter using gRPC protocol.

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type                 | Description                        |
| -------------------- | ---------------------------------- |
| `OTLPMetricExporter` | OTLPMetricExporter (gRPC) instance |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_metric_exporter_grpc(
    config: OTLPExporterConfig,
) -> OTLPMetricExporterGRPC:
    """Create OTLP metric exporter using gRPC protocol.

    Args:
        config: OTLP exporter configuration

    Returns:
        OTLPMetricExporter (gRPC) instance
    """
    endpoint = adjust_endpoint_for_protocol(config.endpoint, config.protocol)
    headers = resolve_headers(config.headers) if config.headers else None
    grpc_headers = _headers_to_grpc_metadata(headers)
    compression = get_compression_grpc(config.compression)
    timeout_seconds = config.timeout_ms / 1000.0

    return OTLPMetricExporterGRPC(
        endpoint=endpoint,
        insecure=config.insecure,
        headers=grpc_headers,
        timeout=timeout_seconds,
        compression=compression,
    )
```

## `create_otlp_metric_exporter_http(config)`

Create OTLP metric exporter using HTTP protocol.

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type                 | Description                        |
| -------------------- | ---------------------------------- |
| `OTLPMetricExporter` | OTLPMetricExporter (HTTP) instance |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_metric_exporter_http(
    config: OTLPExporterConfig,
) -> OTLPMetricExporterHTTP:
    """Create OTLP metric exporter using HTTP protocol.

    Args:
        config: OTLP exporter configuration

    Returns:
        OTLPMetricExporter (HTTP) instance
    """
    endpoint = adjust_endpoint_for_protocol(config.endpoint, config.protocol)

    # HTTP endpoint needs /v1/metrics suffix
    if not endpoint.endswith("/v1/metrics"):
        endpoint = f"{endpoint.rstrip('/')}/v1/metrics"

    headers = resolve_headers(config.headers) if config.headers else None
    compression = get_compression_http(config.compression)
    timeout_seconds = config.timeout_ms / 1000.0

    return OTLPMetricExporterHTTP(
        endpoint=endpoint,
        headers=headers,
        timeout=timeout_seconds,
        compression=compression,
    )
```

## `create_otlp_log_exporter(config)`

Create OTLP log exporter based on protocol.

Dispatches to gRPC or HTTP implementation based on config.protocol.

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type  | Description                             |
| ----- | --------------------------------------- |
| `Any` | OTLPLogExporter instance (gRPC or HTTP) |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_log_exporter(config: OTLPExporterConfig) -> Any:
    """Create OTLP log exporter based on protocol.

    Dispatches to gRPC or HTTP implementation based on config.protocol.

    Args:
        config: OTLP exporter configuration

    Returns:
        OTLPLogExporter instance (gRPC or HTTP)
    """
    from holodeck.models.observability import OTLPProtocol

    if config.protocol == OTLPProtocol.GRPC:
        return create_otlp_log_exporter_grpc(config)
    else:
        return create_otlp_log_exporter_http(config)
```

## `create_otlp_log_exporter_grpc(config)`

Create OTLP log exporter using gRPC protocol.

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type              | Description                     |
| ----------------- | ------------------------------- |
| `OTLPLogExporter` | OTLPLogExporter (gRPC) instance |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_log_exporter_grpc(config: OTLPExporterConfig) -> OTLPLogExporterGRPC:
    """Create OTLP log exporter using gRPC protocol.

    Args:
        config: OTLP exporter configuration

    Returns:
        OTLPLogExporter (gRPC) instance
    """
    endpoint = adjust_endpoint_for_protocol(config.endpoint, config.protocol)
    headers = resolve_headers(config.headers) if config.headers else None
    grpc_headers = _headers_to_grpc_metadata(headers)
    compression = get_compression_grpc(config.compression)
    timeout_seconds = config.timeout_ms / 1000.0

    return OTLPLogExporterGRPC(
        endpoint=endpoint,
        insecure=config.insecure,
        headers=grpc_headers,
        timeout=timeout_seconds,
        compression=compression,
    )
```

## `create_otlp_log_exporter_http(config)`

Create OTLP log exporter using HTTP protocol.

Parameters:

| Name     | Type                 | Description                 | Default    |
| -------- | -------------------- | --------------------------- | ---------- |
| `config` | `OTLPExporterConfig` | OTLP exporter configuration | *required* |

Returns:

| Type              | Description                     |
| ----------------- | ------------------------------- |
| `OTLPLogExporter` | OTLPLogExporter (HTTP) instance |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def create_otlp_log_exporter_http(config: OTLPExporterConfig) -> OTLPLogExporterHTTP:
    """Create OTLP log exporter using HTTP protocol.

    Args:
        config: OTLP exporter configuration

    Returns:
        OTLPLogExporter (HTTP) instance
    """
    endpoint = adjust_endpoint_for_protocol(config.endpoint, config.protocol)

    # HTTP endpoint needs /v1/logs suffix
    if not endpoint.endswith("/v1/logs"):
        endpoint = f"{endpoint.rstrip('/')}/v1/logs"

    headers = resolve_headers(config.headers) if config.headers else None
    compression = get_compression_http(config.compression)
    timeout_seconds = config.timeout_ms / 1000.0

    return OTLPLogExporterHTTP(
        endpoint=endpoint,
        headers=headers,
        timeout=timeout_seconds,
        compression=compression,
    )
```

## `resolve_headers(headers)`

Resolve environment variable references in header values.

Substitutes ${VAR_NAME} patterns with environment variable values.

Parameters:

| Name      | Type             | Description                                                    | Default    |
| --------- | ---------------- | -------------------------------------------------------------- | ---------- |
| `headers` | `dict[str, str]` | Dictionary of header names to values (may contain ${VAR} refs) | *required* |

Returns:

| Type             | Description                                        |
| ---------------- | -------------------------------------------------- |
| `dict[str, str]` | Dictionary with all environment variables resolved |

Raises:

| Type          | Description                                         |
| ------------- | --------------------------------------------------- |
| `ConfigError` | If a referenced environment variable does not exist |

Example

> > > import os os.environ["API_KEY"] = "secret123" resolve_headers({"Authorization": "Bearer ${API_KEY}"}) {'Authorization': 'Bearer secret123'}

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def resolve_headers(headers: dict[str, str]) -> dict[str, str]:
    """Resolve environment variable references in header values.

    Substitutes ${VAR_NAME} patterns with environment variable values.

    Args:
        headers: Dictionary of header names to values (may contain ${VAR} refs)

    Returns:
        Dictionary with all environment variables resolved

    Raises:
        ConfigError: If a referenced environment variable does not exist

    Example:
        >>> import os
        >>> os.environ["API_KEY"] = "secret123"
        >>> resolve_headers({"Authorization": "Bearer ${API_KEY}"})
        {'Authorization': 'Bearer secret123'}
    """
    return {key: substitute_env_vars(value) for key, value in headers.items()}
```

## `adjust_endpoint_for_protocol(endpoint, protocol)`

Adjust endpoint port based on protocol if using default localhost.

OTLP conventions:

- gRPC default port: 4317
- HTTP default port: 4318

Only adjusts ports for localhost/127.0.0.1 when using standard OTLP ports.

Parameters:

| Name       | Type           | Description                  | Default    |
| ---------- | -------------- | ---------------------------- | ---------- |
| `endpoint` | `str`          | Original endpoint URL        | *required* |
| `protocol` | `OTLPProtocol` | OTLP protocol (grpc or http) | *required* |

Returns:

| Type  | Description                           |
| ----- | ------------------------------------- |
| `str` | Endpoint with adjusted port if needed |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def adjust_endpoint_for_protocol(
    endpoint: str,
    protocol: OTLPProtocol,
) -> str:
    """Adjust endpoint port based on protocol if using default localhost.

    OTLP conventions:
    - gRPC default port: 4317
    - HTTP default port: 4318

    Only adjusts ports for localhost/127.0.0.1 when using standard OTLP ports.

    Args:
        endpoint: Original endpoint URL
        protocol: OTLP protocol (grpc or http)

    Returns:
        Endpoint with adjusted port if needed
    """
    from holodeck.models.observability import OTLPProtocol

    parsed = urlparse(endpoint)

    # Only adjust if it's localhost
    if parsed.hostname not in ("localhost", "127.0.0.1"):
        return endpoint

    current_port = parsed.port

    # Adjust port based on protocol if using standard OTLP ports
    if protocol == OTLPProtocol.HTTP and current_port == GRPC_DEFAULT_PORT:
        netloc = f"{parsed.hostname}:{HTTP_DEFAULT_PORT}"
        return urlunparse(parsed._replace(netloc=netloc))
    elif protocol == OTLPProtocol.GRPC and current_port == HTTP_DEFAULT_PORT:
        netloc = f"{parsed.hostname}:{GRPC_DEFAULT_PORT}"
        return urlunparse(parsed._replace(netloc=netloc))

    return endpoint
```

## `get_compression_grpc(compression)`

Convert compression string to gRPC Compression enum.

Parameters:

| Name          | Type  | Description | Default                                              |
| ------------- | ----- | ----------- | ---------------------------------------------------- |
| `compression` | \`str | None\`      | Compression algorithm name ("gzip", "deflate", None) |

Returns:

| Type          | Description |
| ------------- | ----------- |
| \`Compression | None\`      |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def get_compression_grpc(compression: str | None) -> grpc.Compression | None:
    """Convert compression string to gRPC Compression enum.

    Args:
        compression: Compression algorithm name ("gzip", "deflate", None)

    Returns:
        grpc.Compression enum value or None
    """
    if compression is None:
        return None

    compression_map = {
        "gzip": grpc.Compression.Gzip,
        "deflate": grpc.Compression.Deflate,
    }
    return compression_map.get(compression.lower())
```

## `get_compression_http(compression)`

Convert compression string to HTTP Compression enum.

Parameters:

| Name          | Type  | Description | Default                                              |
| ------------- | ----- | ----------- | ---------------------------------------------------- |
| `compression` | \`str | None\`      | Compression algorithm name ("gzip", "deflate", None) |

Returns:

| Type          | Description |
| ------------- | ----------- |
| \`Compression | None\`      |

Source code in `src/holodeck/lib/observability/exporters/otlp.py`

```
def get_compression_http(compression: str | None) -> Compression | None:
    """Convert compression string to HTTP Compression enum.

    Args:
        compression: Compression algorithm name ("gzip", "deflate", None)

    Returns:
        opentelemetry Compression enum value or None
    """
    if compression is None:
        return None

    compression_map = {
        "gzip": Compression.Gzip,
        "deflate": Compression.Deflate,
    }
    return compression_map.get(compression.lower())
```

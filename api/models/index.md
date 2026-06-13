# Data Models API Reference

HoloDeck uses [Pydantic v2](https://docs.pydantic.dev/) models for all configuration validation. This page documents the complete data model hierarchy, backend protocols, and exception classes used throughout the platform.

______________________________________________________________________

## Agent Configuration

Root-level models that define an AI agent instance.

## `Agent`

Bases: `BaseModel`

Agent configuration entity.

Root configuration for a single AI agent instance, defining model, instructions, tools, evaluations, and test cases.

### `validate_author(v)`

Validate author is not empty if provided.

Source code in `src/holodeck/models/agent.py`

```
@field_validator("author")
@classmethod
def validate_author(cls, v: str | None) -> str | None:
    """Validate author is not empty if provided."""
    if v is not None and (not v or not v.strip()):
        raise ValueError("author must be non-empty if provided")
    return v
```

### `validate_description(v)`

Validate description is not empty if provided.

Source code in `src/holodeck/models/agent.py`

```
@field_validator("description")
@classmethod
def validate_description(cls, v: str | None) -> str | None:
    """Validate description is not empty if provided."""
    if v is not None and (not v or not v.strip()):
        raise ValueError("description must be non-empty if provided")
    return v
```

### `validate_name(v)`

Validate name is not empty.

Source code in `src/holodeck/models/agent.py`

```
@field_validator("name")
@classmethod
def validate_name(cls, v: str) -> str:
    """Validate name is not empty."""
    if not v or not v.strip():
        raise ValueError("name must be a non-empty string")
    return v
```

### `validate_response_format(v)`

Validate response_format is dict, string path, or None.

Source code in `src/holodeck/models/agent.py`

```
@field_validator("response_format")
@classmethod
def validate_response_format(
    cls, v: dict[str, Any] | str | None
) -> dict[str, Any] | str | None:
    """Validate response_format is dict, string path, or None."""
    return v
```

### `validate_test_cases(v)`

Validate test cases list.

Source code in `src/holodeck/models/agent.py`

```
@field_validator("test_cases")
@classmethod
def validate_test_cases(
    cls, v: list[TestCaseModel] | None
) -> list[TestCaseModel] | None:
    """Validate test cases list."""
    if v is not None and len(v) > 1000:
        raise ValueError("Maximum 1000 test cases per agent")
    return v
```

### `validate_tool_name_uniqueness()`

Validate that all tool names are unique across tool types.

Raises:

| Type         | Description                                      |
| ------------ | ------------------------------------------------ |
| `ValueError` | If duplicate tool names are found, listing them. |

Source code in `src/holodeck/models/agent.py`

```
@model_validator(mode="after")
def validate_tool_name_uniqueness(self) -> Self:
    """Validate that all tool names are unique across tool types.

    Raises:
        ValueError: If duplicate tool names are found, listing them.
    """
    if not self.tools:
        return self
    counts = Counter(tool.name for tool in self.tools)
    duplicates = sorted(name for name, count in counts.items() if count > 1)
    if duplicates:
        raise ValueError(
            f"Duplicate tool names found: {', '.join(duplicates)}. "
            "Each tool must have a unique name."
        )
    return self
```

### `validate_tools(v)`

Validate tools list.

Source code in `src/holodeck/models/agent.py`

```
@field_validator("tools")
@classmethod
def validate_tools(cls, v: list[ToolUnion] | None) -> list[ToolUnion] | None:
    """Validate tools list."""
    if v is not None and len(v) > 50:
        raise ValueError("Maximum 50 tools per agent")
    return v
```

## `Instructions`

Bases: `BaseModel`

System instructions specification (file or inline).

Represents the system prompt for an agent, supporting both file references and inline text.

### `model_post_init(__context)`

Validate file and inline mutual exclusivity.

Source code in `src/holodeck/models/agent.py`

```
def model_post_init(self, __context: Any) -> None:
    """Validate file and inline mutual exclusivity."""
    if self.file and self.inline:
        raise ValueError("Cannot provide both 'file' and 'inline'")
    if not self.file and not self.inline:
        raise ValueError("Must provide either 'file' or 'inline'")
```

### `validate_file(v)`

Validate file is not empty if provided.

Source code in `src/holodeck/models/agent.py`

```
@field_validator("file")
@classmethod
def validate_file(cls, v: str | None) -> str | None:
    """Validate file is not empty if provided."""
    if v is not None and (not v or not v.strip()):
        raise ValueError("file must be non-empty if provided")
    return v
```

### `validate_inline(v)`

Validate inline is not empty if provided.

Source code in `src/holodeck/models/agent.py`

```
@field_validator("inline")
@classmethod
def validate_inline(cls, v: str | None) -> str | None:
    """Validate inline is not empty if provided."""
    if v is not None and (not v or not v.strip()):
        raise ValueError("inline must be non-empty if provided")
    return v
```

______________________________________________________________________

## LLM Provider

Language model provider configuration supporting OpenAI, Azure OpenAI, Anthropic, and Ollama.

## `ProviderEnum`

Bases: `str`, `Enum`

Supported LLM providers.

## `LLMProvider`

Bases: `BaseModel`

LLM provider configuration.

Specifies which LLM provider and model to use, along with model parameters.

### `check_auth_provider_relevance()`

Warn when auth_provider is set for non-Anthropic providers.

Source code in `src/holodeck/models/llm.py`

```
@model_validator(mode="after")
def check_auth_provider_relevance(self) -> "LLMProvider":
    """Warn when auth_provider is set for non-Anthropic providers."""
    if self.auth_provider is not None and self.provider != ProviderEnum.ANTHROPIC:
        logger.warning(
            "auth_provider '%s' is only relevant for Anthropic provider; "
            "it will be ignored for provider '%s'",
            self.auth_provider.value,
            self.provider.value,
        )
    return self
```

### `check_custom_auth_needs_endpoint()`

Warn when auth_provider is 'custom' but no endpoint is set.

Source code in `src/holodeck/models/llm.py`

```
@model_validator(mode="after")
def check_custom_auth_needs_endpoint(self) -> "LLMProvider":
    """Warn when auth_provider is 'custom' but no endpoint is set."""
    if self.auth_provider == AuthProvider.custom and not self.endpoint:
        logger.warning(
            "auth_provider 'custom' is typically used with a custom endpoint "
            "(e.g., endpoint: http://localhost:11434/v1). "
            "No endpoint is currently configured."
        )
    return self
```

### `check_endpoint_required()`

Validate endpoint is provided for Azure OpenAI and Ollama.

Source code in `src/holodeck/models/llm.py`

```
@model_validator(mode="after")
def check_endpoint_required(self) -> "LLMProvider":
    """Validate endpoint is provided for Azure OpenAI and Ollama."""
    if self.provider in (ProviderEnum.AZURE_OPENAI,) and (
        not self.endpoint or not self.endpoint.strip()
    ):
        raise ValueError(f"endpoint is required for {self.provider.value} provider")
    return self
```

### `validate_max_tokens(v)`

Validate max_tokens is positive.

Source code in `src/holodeck/models/llm.py`

```
@field_validator("max_tokens")
@classmethod
def validate_max_tokens(cls, v: int | None) -> int | None:
    """Validate max_tokens is positive."""
    if v is not None and v <= 0:
        raise ValueError("max_tokens must be positive")
    return v
```

### `validate_name(v)`

Validate name is not empty.

Source code in `src/holodeck/models/llm.py`

```
@field_validator("name")
@classmethod
def validate_name(cls, v: str) -> str:
    """Validate name is not empty."""
    if not v or not v.strip():
        raise ValueError("name must be a non-empty string")
    return v
```

### `validate_temperature(v)`

Validate temperature is in valid range.

Source code in `src/holodeck/models/llm.py`

```
@field_validator("temperature")
@classmethod
def validate_temperature(cls, v: float | None) -> float | None:
    """Validate temperature is in valid range."""
    if v is not None and (v < 0.0 or v > 2.0):
        raise ValueError("temperature must be between 0.0 and 2.0")
    return v
```

### `validate_top_p(v)`

Validate top_p is in valid range.

Source code in `src/holodeck/models/llm.py`

```
@field_validator("top_p")
@classmethod
def validate_top_p(cls, v: float | None) -> float | None:
    """Validate top_p is in valid range."""
    if v is not None and (v < 0.0 or v > 1.0):
        raise ValueError("top_p must be between 0.0 and 1.0")
    return v
```

______________________________________________________________________

## Claude Agent SDK

Configuration models for the Claude Agent SDK integration. All capabilities default to disabled (least-privilege).

## `AuthProvider`

Bases: `str`, `Enum`

Authentication method for Anthropic provider.

## `PermissionMode`

Bases: `str`, `Enum`

Level of autonomous action for Claude Agent SDK.

## `ClaudeConfig`

Bases: `BaseModel`

Claude Agent SDK-specific settings.

All fields optional. All capabilities default to disabled (least-privilege).

## `ExtendedThinkingConfig`

Bases: `BaseModel`

Extended reasoning (deep thinking) configuration.

## `BashConfig`

Bases: `BaseModel`

Shell command execution settings.

## `FileSystemConfig`

Bases: `BaseModel`

File read/write/edit access settings.

## `SubagentSpec`

Bases: `BaseModel`

A single subagent definition for multi-agent orchestration.

Each entry under `claude.agents` becomes an `claude_agent_sdk.types.AgentDefinition` on `ClaudeAgentOptions.agents`. The SDK uses `description` for routing.

US3 validators (prompt/prompt_file mutual exclusion and file resolution) are deliberately omitted here — they belong to the US3 task slice.

______________________________________________________________________

## OpenAI Agents SDK

Configuration models for the `openai:` block, applied when `model.provider` is `openai` or `azure_openai`. `OpenAIConfig` is the sibling of `ClaudeConfig`: it carries serve sizing (`max_concurrent_sessions`, `session_memory_estimate_mib`), the agent-loop bound (`max_turns`), reasoning `effort`, the `max_budget_usd` spend cap, `fallback_model`, tool allow/deny lists, and the safety/redaction opt-outs. All fields are optional. See the [OpenAI Backend guide](https://docs.useholodeck.ai/guides/openai-backend/index.md) for end-to-end usage.

## `OpenAIConfig`

Bases: `BaseModel`

OpenAI Agents SDK-specific settings.

All fields optional. Applicable only when `model.provider` is `openai` or `azure_openai`.

## `OpenAIPermissionsConfig`

Bases: `BaseModel`

Tool permission lists for the OpenAI Agents backend.

______________________________________________________________________

## Backend Abstraction Layer

Provider-agnostic protocols and data classes for agent execution are documented in the [Backends API Reference](https://docs.useholodeck.ai/api/backends/index.md). Key types: `AgentBackend`, `AgentSession`, `ContextGenerator`, `ExecutionResult`, `ToolEvent`.

______________________________________________________________________

## Token Usage

## `TokenUsage`

Bases: `BaseModel`

Token usage metadata.

Supports arithmetic operations for accumulating token counts

total = TokenUsage.zero() total = total + new_usage # or total += new_usage

### `validate_total(value, info)`

Ensure total_tokens >= prompt + completion.

Relaxed from strict equality: cache reads may push total above prompt+completion at some providers (data-model.md §10a).

Source code in `src/holodeck/models/token_usage.py`

```
@field_validator("total_tokens")
@classmethod
def validate_total(cls, value: int, info: ValidationInfo) -> int:
    """Ensure total_tokens >= prompt + completion.

    Relaxed from strict equality: cache reads may push total above
    prompt+completion at some providers (data-model.md §10a).
    """
    prompt = info.data.get("prompt_tokens", 0)
    completion = info.data.get("completion_tokens", 0)
    if value < prompt + completion:
        raise ValueError(
            f"total_tokens ({value}) must be >= "
            f"prompt_tokens ({prompt}) + completion_tokens ({completion})"
        )
    return value
```

### `zero()`

Create a TokenUsage instance with all counts at zero.

Source code in `src/holodeck/models/token_usage.py`

```
@classmethod
def zero(cls) -> "TokenUsage":
    """Create a TokenUsage instance with all counts at zero."""
    return cls(
        prompt_tokens=0,
        completion_tokens=0,
        total_tokens=0,
        cache_creation_tokens=0,
        cache_read_tokens=0,
    )
```

______________________________________________________________________

## Tool Models

Six tool types are supported via a discriminated union (`ToolUnion`): vectorstore, hierarchical document, function, MCP, prompt, and plugin.

### Base and Union

## `Tool`

Bases: `BaseModel`

Base tool model with discriminated union for subtypes.

### Vectorstore Tool

## `VectorstoreTool`

Bases: `BaseModel`

Vectorstore tool for semantic search over documents.

### `validate_chunk_overlap(v)`

Validate chunk_overlap is non-negative if provided.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("chunk_overlap")
@classmethod
def validate_chunk_overlap(cls, v: int | None) -> int | None:
    """Validate chunk_overlap is non-negative if provided."""
    if v is not None and v < 0:
        raise ValueError("chunk_overlap must be non-negative")
    return v
```

### `validate_chunk_size(v)`

Validate chunk_size is positive if provided.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("chunk_size")
@classmethod
def validate_chunk_size(cls, v: int | None) -> int | None:
    """Validate chunk_size is positive if provided."""
    if v is not None and v <= 0:
        raise ValueError("chunk_size must be positive")
    return v
```

### `validate_database(v)`

Validate database is not empty string if provided as string.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("database")
@classmethod
def validate_database(
    cls, v: DatabaseConfig | str | None
) -> DatabaseConfig | str | None:
    """Validate database is not empty string if provided as string."""
    if isinstance(v, str) and not v.strip():
        raise ValueError("database reference must be a non-empty string")
    return v
```

### `validate_embedding_dimensions(v)`

Validate embedding_dimensions is positive and reasonable if provided.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("embedding_dimensions")
@classmethod
def validate_embedding_dimensions(cls, v: int | None) -> int | None:
    """Validate embedding_dimensions is positive and reasonable if provided."""
    if v is not None:
        if v <= 0:
            raise ValueError("embedding_dimensions must be positive")
        if v > 10000:
            raise ValueError("embedding_dimensions unreasonably large (max 10000)")
    return v
```

### `validate_min_similarity_score(v)`

Validate min_similarity_score is between 0.0 and 1.0 if provided.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("min_similarity_score")
@classmethod
def validate_min_similarity_score(cls, v: float | None) -> float | None:
    """Validate min_similarity_score is between 0.0 and 1.0 if provided."""
    if v is not None and not (0.0 <= v <= 1.0):
        raise ValueError("min_similarity_score must be between 0.0 and 1.0")
    return v
```

### `validate_source(v)`

Validate source is not empty and has a supported URI scheme.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("source")
@classmethod
def validate_source(cls, v: str) -> str:
    """Validate source is not empty and has a supported URI scheme."""
    if not v or not v.strip():
        raise ValueError("source must be a non-empty path")
    return _validate_source_scheme(v)
```

### `validate_structured_config()`

Validate structured data configuration.

When vector_field is set (structured data mode), id_field becomes required to enable record identification for upserts and deduplication.

Source code in `src/holodeck/models/tool.py`

```
@model_validator(mode="after")
def validate_structured_config(self) -> "VectorstoreTool":
    """Validate structured data configuration.

    When vector_field is set (structured data mode), id_field becomes required
    to enable record identification for upserts and deduplication.
    """
    if self.vector_field is not None and self.id_field is None:
        raise ValueError(
            "id_field is required when vector_field is specified "
            "(structured data mode)"
        )
    return self
```

### `validate_top_k(v)`

Validate top_k is a positive integer.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("top_k")
@classmethod
def validate_top_k(cls, v: int) -> int:
    """Validate top_k is a positive integer."""
    if v <= 0:
        raise ValueError("top_k must be a positive integer")
    if v > 100:
        raise ValueError("top_k should not exceed 100")
    return v
```

### Function Tool

## `FunctionTool`

Bases: `BaseModel`

Function tool for calling Python functions.

### `validate_file(v)`

Validate file is not empty.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("file")
@classmethod
def validate_file(cls, v: str) -> str:
    """Validate file is not empty."""
    if not v or not v.strip():
        raise ValueError("file must be a non-empty path")
    return v
```

### `validate_function(v)`

Validate function is not empty.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("function")
@classmethod
def validate_function(cls, v: str) -> str:
    """Validate function is not empty."""
    if not v or not v.strip():
        raise ValueError("function must be a non-empty identifier")
    return v
```

### MCP Tool

## `MCPTool`

Bases: `BaseModel`

MCP (Model Context Protocol) tool for standardized integrations.

Supports four transport types:

- stdio (default): Local MCP servers via subprocess
- sse: Remote servers via Server-Sent Events
- websocket: Bidirectional WebSocket communication
- http: Streamable HTTP transport

For stdio transport, only npx, uvx, or docker commands are allowed for security reasons.

### `validate_request_timeout(v)`

Validate request_timeout is positive.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("request_timeout")
@classmethod
def validate_request_timeout(cls, v: int) -> int:
    """Validate request_timeout is positive."""
    if v <= 0:
        raise ValueError("request_timeout must be positive")
    return v
```

### `validate_transport_fields()`

Validate transport-specific required fields.

- stdio transport requires 'command'
- sse, websocket, http transports require 'url'

Source code in `src/holodeck/models/tool.py`

```
@model_validator(mode="after")
def validate_transport_fields(self) -> "MCPTool":
    """Validate transport-specific required fields.

    - stdio transport requires 'command'
    - sse, websocket, http transports require 'url'
    """
    if self.transport == TransportType.STDIO:
        if self.command is None:
            raise ValueError("'command' is required for stdio transport")
    elif (
        self.transport
        in (TransportType.SSE, TransportType.WEBSOCKET, TransportType.HTTP)
        and self.url is None
    ):
        raise ValueError(f"'url' is required for {self.transport.value} transport")
    return self
```

### `validate_url_scheme(v)`

Validate URL scheme for HTTP-based transports.

Allows http:// only for localhost, requires https:// for remote URLs. WebSocket URLs can use wss:// or ws://.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("url")
@classmethod
def validate_url_scheme(cls, v: str | None) -> str | None:
    """Validate URL scheme for HTTP-based transports.

    Allows http:// only for localhost, requires https:// for remote URLs.
    WebSocket URLs can use wss:// or ws://.
    """
    if v is None:
        return v
    # Allow http:// for localhost, require https:// otherwise
    if v.startswith("http://"):
        localhost_prefixes = (
            "http://localhost",
            "http://127.0.0.1",
            "http://[::1]",
        )
        if not any(v.startswith(prefix) for prefix in localhost_prefixes):
            raise ValueError("'url' must use https:// (or http:// for localhost)")
    elif not v.startswith(("https://", "wss://", "ws://")):
        raise ValueError("'url' must use https://, wss://, or ws:// scheme")
    return v
```

### Prompt Tool

## `PromptTool`

Bases: `BaseModel`

Prompt-based tool for AI-powered semantic functions.

### `check_template_or_file(v, info)`

Validate that exactly one of template or file is provided.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("parameters", mode="before")
@classmethod
def check_template_or_file(cls, v: Any, info: Any) -> Any:
    """Validate that exactly one of template or file is provided."""
    data = info.data
    template = data.get("template")
    file_path = data.get("file")

    if not template and not file_path:
        raise ValueError("Exactly one of 'template' or 'file' must be provided")
    if template and file_path:
        raise ValueError("Cannot provide both 'template' and 'file'")

    return v
```

### `validate_file(v)`

Validate file is not empty if provided.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("file")
@classmethod
def validate_file(cls, v: str | None) -> str | None:
    """Validate file is not empty if provided."""
    if v is not None and (not v or not v.strip()):
        raise ValueError("file must be non-empty if provided")
    return v
```

### `validate_parameters(v)`

Validate parameters is not empty.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("parameters")
@classmethod
def validate_parameters(
    cls, v: dict[str, dict[str, Any]]
) -> dict[str, dict[str, Any]]:
    """Validate parameters is not empty."""
    if not v:
        raise ValueError("parameters must have at least one parameter")
    return v
```

### `validate_template(v)`

Validate template is not empty if provided.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("template")
@classmethod
def validate_template(cls, v: str | None) -> str | None:
    """Validate template is not empty if provided."""
    if v is not None and (not v or not v.strip()):
        raise ValueError("template must be non-empty if provided")
    return v
```

### Hierarchical Document Tool

## `HierarchicalDocumentToolConfig`

Bases: `BaseModel`

Configuration for the HierarchicalDocumentTool.

A specialized document search tool that preserves document structure, supports multiple search modalities (semantic, keyword, exact), and enables context-aware embeddings for improved retrieval quality.

Features:

- Structure-aware chunking that respects markdown headings
- Hybrid search combining semantic, keyword, and exact matching
- Contextual embeddings that include document/section context
- Automatic definition and cross-reference extraction
- Optional reranking for improved result quality

### `validate_finite_weights(v)`

Validate search weights are finite numeric values.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("semantic_weight", "keyword_weight", "exact_weight")
@classmethod
def validate_finite_weights(cls, v: float) -> float:
    """Validate search weights are finite numeric values."""
    if not math.isfinite(v):
        raise ValueError("search weights must be finite numbers")
    return v
```

### `validate_min_score(v)`

Validate min_score is between 0.0 and 1.0 if provided.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("min_score")
@classmethod
def validate_min_score(cls, v: float | None) -> float | None:
    """Validate min_score is between 0.0 and 1.0 if provided."""
    if v is not None and not (0.0 <= v <= 1.0):
        raise ValueError("min_score must be between 0.0 and 1.0")
    return v
```

### `validate_reranker()`

Validate reranker_model is provided when enable_reranking=True.

Source code in `src/holodeck/models/tool.py`

```
@model_validator(mode="after")
def validate_reranker(self) -> "HierarchicalDocumentToolConfig":
    """Validate reranker_model is provided when enable_reranking=True."""
    if self.enable_reranking and self.reranker_model is None:
        raise ValueError("reranker_model is required when enable_reranking=True")
    return self
```

### `validate_source(v)`

Validate source is not empty and has a supported URI scheme.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("source")
@classmethod
def validate_source(cls, v: str) -> str:
    """Validate source is not empty and has a supported URI scheme."""
    if not v or not v.strip():
        raise ValueError("source must be a non-empty path")
    return _validate_source_scheme(v)
```

### `validate_weights()`

Validate hybrid weights are well-formed for deterministic fusion.

Source code in `src/holodeck/models/tool.py`

```
@model_validator(mode="after")
def validate_weights(self) -> "HierarchicalDocumentToolConfig":
    """Validate hybrid weights are well-formed for deterministic fusion."""
    if self.search_mode == SearchMode.HYBRID:
        total = self.semantic_weight + self.keyword_weight + self.exact_weight
        if not math.isclose(total, 1.0, rel_tol=0.0, abs_tol=1e-6):
            raise ValueError(
                f"Hybrid search weights must sum to 1.0, got {total:.6f} "
                "(semantic_weight + keyword_weight + exact_weight)"
            )

        if (self.semantic_weight + self.keyword_weight) <= 0:
            raise ValueError(
                "Hybrid search requires semantic_weight + keyword_weight > 0"
            )
    return self
```

### Supporting Enums

## `TransportType`

Bases: `str`, `Enum`

Supported MCP transport types.

Defines the communication protocol used to connect to MCP servers:

- STDIO: Local process via stdin/stdout (default, most common)
- SSE: Server-Sent Events over HTTP
- WEBSOCKET: WebSocket for bidirectional communication
- HTTP: Streamable HTTP transport

## `CommandType`

Bases: `str`, `Enum`

Allowed stdio commands for MCP servers (security constraint).

Only these commands are permitted for spawning MCP server processes to prevent command injection attacks:

- NPX: Node.js/npm package runner
- NODE: Direct Node.js script runner
- UVX: Python/uv package runner
- DOCKER: Docker container runner

## `SearchMode`

Bases: `str`, `Enum`

Search modality options for HierarchicalDocumentTool.

Defines which search indices are used during query:

- SEMANTIC: Dense embeddings only (conceptual similarity)
- KEYWORD: BM25 sparse index only (term frequency)
- HYBRID: Keyword + semantic combined with RRF fusion

## `ChunkingStrategy`

Bases: `str`, `Enum`

Document chunking approach for HierarchicalDocumentTool.

Defines how documents are split into chunks:

- STRUCTURE: Parse markdown headings and split at structural boundaries
- TOKEN: Fixed token-based splitting with overlap (fallback for flat text)

## `DocumentDomain`

Bases: `str`, `Enum`

Predefined document structure patterns for subsection detection.

Enables YAML-based configuration of domain-specific parsing rules for hierarchical document tools. Each domain defines subsection patterns that recognize implicit headings in documents.

Attributes:

| Name             | Type | Description                                            |
| ---------------- | ---- | ------------------------------------------------------ |
| `NONE`           |      | No subsection patterns (default, backward compatible). |
| `US_LEGISLATIVE` |      | US Code style: (a), (1), (A), (i) hierarchy.           |
| `AU_LEGISLATIVE` |      | Australian style: (1), (a), (i), (A) hierarchy.        |
| `ACADEMIC`       |      | Academic papers: 1., 1.1, 1.1.1 numbered sections.     |
| `TECHNICAL`      |      | Technical manuals: Step 1, 1.1, Note:, Warning:.       |
| `LEGAL_CONTRACT` |      | Legal contracts: Article I, Section 1, (a) clauses.    |

## `KeywordIndexProvider`

Bases: `str`, `Enum`

Keyword index backend provider for sparse/BM25 search.

### Database and Index Configuration

## `DatabaseConfig`

Bases: `BaseModel`

Vector database connection configuration.

Supports all Semantic Kernel vector store providers including PostgreSQL, Azure AI Search, Qdrant, Weaviate, ChromaDB, FAISS, Pinecone, and more.

Provider-specific parameters are passed via the config dict:

- postgres: connection_string
- azure-ai-search: connection_string, api_key
- qdrant: url, api_key (optional)
- weaviate: url, api_key (optional)
- chromadb: path or host
- faiss: path
- pinecone: api_key, index_name
- And more...

### `validate_connection_string(v)`

Validate connection string is not empty if provided.

Source code in `src/holodeck/models/tool.py`

```
@field_validator("connection_string")
@classmethod
def validate_connection_string(cls, v: SecretStr | None) -> SecretStr | None:
    """Validate connection string is not empty if provided."""
    if v is not None and not v.get_secret_value().strip():
        raise ValueError("connection_string must be non-empty if provided")
    return v
```

## `KeywordIndexConfig`

Bases: `BaseModel`

Keyword index configuration for sparse/BM25 search backend.

Two providers:

- in-memory: rank_bm25 in-process (default, dev/local)
- opensearch: OpenSearch endpoint (production)

When provider='opensearch', endpoint and index_name are required (validated at model construction time).

### `validate_opensearch_fields()`

Validate that endpoint and index_name are set for opensearch provider.

Source code in `src/holodeck/models/tool.py`

```
@model_validator(mode="after")
def validate_opensearch_fields(self) -> "KeywordIndexConfig":
    """Validate that endpoint and index_name are set for opensearch provider."""
    if self.provider == "opensearch":
        if not self.endpoint:
            raise ValueError("endpoint is required when provider is 'opensearch'")
        if not self.index_name:
            raise ValueError("index_name is required when provider is 'opensearch'")
    return self
```

______________________________________________________________________

## Tool Execution and Events

Runtime models for tracking tool execution status and streaming events.

## `ToolStatus`

Bases: `str`, `Enum`

Tool execution status.

## `ToolExecution`

Bases: `BaseModel`

Tool execution metadata.

### `sanitize_result(value)`

Sanitize tool output before display.

Source code in `src/holodeck/models/tool_execution.py`

```
@field_validator("result")
@classmethod
def sanitize_result(cls, value: str | None) -> str | None:
    """Sanitize tool output before display."""
    if value is None:
        return None
    return sanitize_tool_output(value)
```

### `validate_error_message(value, info)`

Require error_message when status is FAILED.

Source code in `src/holodeck/models/tool_execution.py`

```
@field_validator("error_message")
@classmethod
def validate_error_message(cls, value: str | None, info: Any) -> str | None:
    """Require error_message when status is FAILED."""
    if info.data.get("status") == ToolStatus.FAILED and not value:
        raise ValueError("error_message required when status=FAILED")
    return value
```

### `validate_failure()`

Ensure failed executions include an error message.

Source code in `src/holodeck/models/tool_execution.py`

```
@model_validator(mode="after")
def validate_failure(self) -> ToolExecution:
    """Ensure failed executions include an error message."""
    if self.status == ToolStatus.FAILED and not self.error_message:
        raise ValueError("error_message required when status=FAILED")
    return self
```

## `ToolEventType`

Bases: `str`, `Enum`

Type of tool execution event.

## `ToolEvent`

Bases: `BaseModel`

Event emitted during tool execution.

Represents a single event in the lifecycle of a tool execution, used for streaming updates about tool progress and results.

______________________________________________________________________

## Evaluation Models

Metrics and evaluation framework configuration. Three metric families are supported via a discriminated union (`MetricType`): standard NLP, G-Eval custom criteria, and RAG pipeline metrics.

## `EvaluationConfig`

Bases: `BaseModel`

Evaluation framework configuration.

Container for evaluation metrics with optional default model configuration. Supports standard EvaluationMetric, GEvalMetric (custom criteria), and RAGMetric (RAG pipeline evaluation).

### `validate_metrics(v)`

Validate metrics list is not empty.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("metrics")
@classmethod
def validate_metrics(
    cls, v: list[EvaluationMetric | GEvalMetric | RAGMetric | CodeMetric]
) -> list[EvaluationMetric | GEvalMetric | RAGMetric | CodeMetric]:
    """Validate metrics list is not empty."""
    if not v:
        raise ValueError("metrics must have at least one metric")
    return v
```

## `EvaluationMetric`

Bases: `BaseModel`

Evaluation metric configuration.

Represents a single evaluation metric with flexible model configuration, including per-metric LLM model overrides.

### `validate_custom_prompt(v)`

Validate custom_prompt is not empty if provided.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("custom_prompt")
@classmethod
def validate_custom_prompt(cls, v: str | None) -> str | None:
    """Validate custom_prompt is not empty if provided."""
    if v is not None and (not v or not v.strip()):
        raise ValueError("custom_prompt must be non-empty if provided")
    return v
```

### `validate_enabled(v)`

Validate enabled is boolean.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("enabled")
@classmethod
def validate_enabled(cls, v: bool) -> bool:
    """Validate enabled is boolean."""
    if not isinstance(v, bool):
        raise ValueError("enabled must be boolean")
    return v
```

### `validate_fail_on_error(v)`

Validate fail_on_error is boolean.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("fail_on_error")
@classmethod
def validate_fail_on_error(cls, v: bool) -> bool:
    """Validate fail_on_error is boolean."""
    if not isinstance(v, bool):
        raise ValueError("fail_on_error must be boolean")
    return v
```

### `validate_response_path(v)`

Validate response_path matches the supported grammar.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("response_path")
@classmethod
def validate_response_path(cls, v: str | None) -> str | None:
    """Validate response_path matches the supported grammar."""
    if v is None:
        return v
    # Local import — keeps the model layer free of evaluator runtime
    # imports during cold-start config loads.
    from holodeck.lib.evaluators.response_path import is_valid_path

    if not v.strip() or not is_valid_path(v):
        raise ValueError(
            "response_path must match dotted/bracketed grammar "
            "(e.g. 'answer', 'result.value', 'items[0].score'); "
            f"got {v!r}"
        )
    return v
```

### `validate_retry_on_failure(v)`

Validate retry_on_failure is in valid range.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("retry_on_failure")
@classmethod
def validate_retry_on_failure(cls, v: int | None) -> int | None:
    """Validate retry_on_failure is in valid range."""
    if v is not None and (v < 1 or v > 3):
        raise ValueError("retry_on_failure must be between 1 and 3")
    return v
```

### `validate_scale(v)`

Validate scale is positive.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("scale")
@classmethod
def validate_scale(cls, v: int | None) -> int | None:
    """Validate scale is positive."""
    if v is not None and v <= 0:
        raise ValueError("scale must be positive")
    return v
```

### `validate_threshold(v)`

Validate threshold is numeric if provided.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("threshold")
@classmethod
def validate_threshold(cls, v: float | None) -> float | None:
    """Validate threshold is numeric if provided."""
    if v is not None and not isinstance(v, int | float):
        raise ValueError("threshold must be numeric")
    return v
```

### `validate_timeout_ms(v)`

Validate timeout_ms is positive.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("timeout_ms")
@classmethod
def validate_timeout_ms(cls, v: int | None) -> int | None:
    """Validate timeout_ms is positive."""
    if v is not None and v <= 0:
        raise ValueError("timeout_ms must be positive")
    return v
```

## `GEvalMetric`

Bases: `BaseModel`

G-Eval custom criteria metric configuration.

Uses discriminator pattern with type="geval" to distinguish from standard EvaluationMetric instances in a discriminated union.

G-Eval enables custom evaluation criteria defined in natural language, using chain-of-thought prompting with LLM-based scoring.

Example

> > > metric = GEvalMetric( ... name="Professionalism", ... criteria="Evaluate if the response uses professional language", ... threshold=0.7 ... )

### `validate_criteria(v)`

Validate criteria is not empty.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("criteria")
@classmethod
def validate_criteria(cls, v: str) -> str:
    """Validate criteria is not empty."""
    if not v or not v.strip():
        raise ValueError("criteria must be a non-empty string")
    return v
```

### `validate_evaluation_params(v)`

Validate evaluation_params contains valid values.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("evaluation_params")
@classmethod
def validate_evaluation_params(cls, v: list[str]) -> list[str]:
    """Validate evaluation_params contains valid values."""
    if not v:
        raise ValueError("evaluation_params must not be empty")
    invalid_params = set(v) - VALID_EVALUATION_PARAMS
    if invalid_params:
        raise ValueError(
            f"Invalid evaluation_params: {sorted(invalid_params)}. "
            f"Valid options: {sorted(VALID_EVALUATION_PARAMS)}"
        )
    return v
```

### `validate_name(v)`

Validate name is not empty.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("name")
@classmethod
def validate_name(cls, v: str) -> str:
    """Validate name is not empty."""
    if not v or not v.strip():
        raise ValueError("name must be a non-empty string")
    return v
```

### `validate_threshold(v)`

Validate threshold is in valid range.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("threshold")
@classmethod
def validate_threshold(cls, v: float | None) -> float | None:
    """Validate threshold is in valid range."""
    if v is not None and (v < 0.0 or v > 1.0):
        raise ValueError("threshold must be between 0.0 and 1.0")
    return v
```

## `RAGMetric`

Bases: `BaseModel`

RAG pipeline evaluation metric configuration.

Uses discriminator pattern with type="rag" to distinguish from standard EvaluationMetric and GEvalMetric instances in a discriminated union.

RAG metrics evaluate the quality of retrieval-augmented generation pipelines:

- Faithfulness: Detects hallucinations by comparing response to context
- ContextualRelevancy: Measures relevance of retrieved chunks to query
- ContextualPrecision: Evaluates ranking quality of retrieved chunks
- ContextualRecall: Measures retrieval completeness against expected output

Example

> > > metric = RAGMetric( ... metric_type=RAGMetricType.FAITHFULNESS, ... threshold=0.8 ... )

### `validate_threshold(v)`

Validate threshold is in valid range.

Source code in `src/holodeck/models/evaluation.py`

```
@field_validator("threshold")
@classmethod
def validate_threshold(cls, v: float) -> float:
    """Validate threshold is in valid range."""
    if v < 0.0 or v > 1.0:
        raise ValueError("threshold must be between 0.0 and 1.0")
    return v
```

## `RAGMetricType`

Bases: `str`, `Enum`

RAG pipeline evaluation metric types.

These metrics evaluate the quality of Retrieval-Augmented Generation (RAG) pipelines by assessing various aspects of retrieval and response generation.

## `MetricType = Annotated[EvaluationMetric | GEvalMetric | RAGMetric | CodeMetric, Field(discriminator='type')]`

______________________________________________________________________

## Test Case Models

Test case definitions with multimodal file input support.

## `TestCaseModel`

Bases: `BaseModel`

Test case for agent evaluation.

Represents a single test scenario. Either legacy single-turn (with `input`) or multi-turn (with `turns`), but not both.

### `validate_input(v)`

Input, when supplied, must be non-empty.

Source code in `src/holodeck/models/test_case.py`

```
@field_validator("input")
@classmethod
def validate_input(cls, v: str | None) -> str | None:
    """Input, when supplied, must be non-empty."""
    if v is not None and (not v or not v.strip()):
        raise ValueError("input must be a non-empty string")
    return v
```

## `FileInput`

Bases: `BaseModel`

File input for multimodal test cases.

Represents a single file reference for test case inputs, supporting both local files and remote URLs with optional extraction parameters.

### `check_path_or_url(v, info)`

Validate that exactly one of path or url is provided.

Source code in `src/holodeck/models/test_case.py`

```
@field_validator("path", "url", mode="before")
@classmethod
def check_path_or_url(cls, v: Any, info: Any) -> Any:
    """Validate that exactly one of path or url is provided."""
    # This runs before validation, so we check in root_validator
    return v
```

### `model_post_init(__context)`

Validate path and url mutual exclusivity after initialization.

Source code in `src/holodeck/models/test_case.py`

```
def model_post_init(self, __context: Any) -> None:
    """Validate path and url mutual exclusivity after initialization."""
    if self.path and self.url:
        raise ValueError("Cannot provide both 'path' and 'url'")
    if not self.path and not self.url:
        raise ValueError("Must provide either 'path' or 'url'")
```

### `validate_pages(v)`

Validate pages are positive integers.

Source code in `src/holodeck/models/test_case.py`

```
@field_validator("pages")
@classmethod
def validate_pages(cls, v: list[int] | None) -> list[int] | None:
    """Validate pages are positive integers."""
    if v is not None and not all(isinstance(p, int) and p > 0 for p in v):
        raise ValueError("pages must be positive integers")
    return v
```

### `validate_type(v)`

Validate file type is supported.

Source code in `src/holodeck/models/test_case.py`

```
@field_validator("type")
@classmethod
def validate_type(cls, v: str) -> str:
    """Validate file type is supported."""
    valid_types = {"image", "pdf", "text", "excel", "word", "powerpoint", "csv"}
    if v not in valid_types:
        raise ValueError(f"type must be one of {valid_types}, got {v}")
    return v
```

______________________________________________________________________

## Test Result Models

Models for representing test execution results, metric outcomes, and reports.

## `ProcessedFileInput`

Bases: `BaseModel`

Processed file input with extracted content.

Represents a file that has been processed (converted to markdown) and is ready for use in agent prompts.

## `MetricResult`

Bases: `BaseModel`

Result of evaluating a single metric.

Represents the outcome of running an evaluation metric (e.g., groundedness, relevance) on a test case response.

## `TestResult`

Bases: `BaseModel`

Result of executing a single test case.

Contains the test input, agent response, tool calls, metric results, and overall pass/fail status along with any errors encountered.

Readers prefer `tool_invocations` (structured `{name, args, result, bytes}`) when non-empty; `tool_calls: list[str]` (name-only) remains for back-compat with legacy runs predating Phase 2b.

## `ReportSummary`

Bases: `BaseModel`

Summary statistics for a test execution run.

Aggregates statistics across all test results including pass rates, metric scores, and timing information.

### `validate_test_counts()`

Validate that passed + failed equals total_tests.

Source code in `src/holodeck/models/test_result.py`

```
@model_validator(mode="after")
def validate_test_counts(self) -> "ReportSummary":
    """Validate that passed + failed equals total_tests."""
    if self.passed + self.failed != self.total_tests:
        raise ValueError(
            f"passed ({self.passed}) + failed ({self.failed}) "
            f"must equal total_tests ({self.total_tests})"
        )
    return self
```

## `TestReport`

Bases: `BaseModel`

Complete test execution report.

Contains all test results, summary statistics, and metadata about the test run including agent name, version, and environment.

### `validate_results_count()`

Validate that summary total_tests matches results count.

Source code in `src/holodeck/models/test_result.py`

```
@model_validator(mode="after")
def validate_results_count(self) -> "TestReport":
    """Validate that summary total_tests matches results count."""
    if self.summary.total_tests != len(self.results):
        raise ValueError(
            f"summary.total_tests ({self.summary.total_tests}) "
            f"must match number of results ({len(self.results)})"
        )
    return self
```

______________________________________________________________________

## Chat Models

Interactive chat session and message models.

## `SessionState`

Bases: `str`, `Enum`

Chat session lifecycle state.

## `MessageRole`

Bases: `str`, `Enum`

Role of a chat message.

## `Message`

Bases: `BaseModel`

Chat message model.

### `validate_content(value, info)`

Strip content and enforce size limits.

Source code in `src/holodeck/models/chat.py`

```
@field_validator("content")
@classmethod
def validate_content(cls, value: str, info: Any) -> str:
    """Strip content and enforce size limits."""
    cleaned = value.strip()
    if not cleaned:
        raise ValueError("Message content cannot be empty")
    role = info.data.get("role")
    if role == MessageRole.USER and len(cleaned) > 10_000:
        raise ValueError("User message exceeds 10,000 character limit")
    return cleaned
```

### `validate_tool_calls(calls, info)`

Ensure tool calls are only attached to assistant messages.

Source code in `src/holodeck/models/chat.py`

```
@field_validator("tool_calls")
@classmethod
def validate_tool_calls(
    cls, calls: list[ToolExecution], info: Any
) -> list[ToolExecution]:
    """Ensure tool calls are only attached to assistant messages."""
    if calls and info.data.get("role") != MessageRole.ASSISTANT:
        raise ValueError("Tool calls only allowed for assistant messages")
    return calls
```

## `ChatSession`

Bases: `BaseModel`

Interactive chat session model.

### `validate_started_at(value)`

Ensure started_at is not in the future.

Source code in `src/holodeck/models/chat.py`

```
@field_validator("started_at")
@classmethod
def validate_started_at(cls, value: datetime) -> datetime:
    """Ensure started_at is not in the future."""
    if value > datetime.utcnow():
        raise ValueError("started_at cannot be in the future")
    return value
```

## `ChatConfig`

Bases: `BaseModel`

Runtime configuration for chat sessions.

### `validate_path(value)`

Ensure the agent config path exists and is a file.

Source code in `src/holodeck/models/chat.py`

```
@field_validator("agent_config_path")
@classmethod
def validate_path(cls, value: Path) -> Path:
    """Ensure the agent config path exists and is a file."""
    if not value.exists():
        raise ValueError(f"Agent config not found: {value}")
    if not value.is_file():
        raise ValueError(f"Path is not a file: {value}")
    return value
```

______________________________________________________________________

## Global Configuration

Project-wide settings stored in `~/.holodeck/config.yaml` for sharing defaults across multiple agents.

## `ExecutionConfig`

Bases: `BaseModel`

Test execution configuration.

Specifies execution parameters for agent test runs, including timeouts, caching, and logging options.

## `VectorstoreConfig`

Bases: `BaseModel`

Vectorstore configuration for global defaults.

Specifies connection details and options for a specific vectorstore backend.

### `validate_connection_string(v)`

Validate connection_string is not empty.

Source code in `src/holodeck/models/config.py`

```
@field_validator("connection_string")
@classmethod
def validate_connection_string(cls, v: str) -> str:
    """Validate connection_string is not empty."""
    if not v or not v.strip():
        raise ValueError("connection_string must be a non-empty string")
    return v
```

### `validate_provider(v)`

Validate provider is not empty.

Source code in `src/holodeck/models/config.py`

```
@field_validator("provider")
@classmethod
def validate_provider(cls, v: str) -> str:
    """Validate provider is not empty."""
    if not v or not v.strip():
        raise ValueError("provider must be a non-empty string")
    return v
```

## `GlobalConfig`

Bases: `BaseModel`

Global configuration entity.

Configuration stored in ~/.holodeck/config.yaml for sharing defaults across multiple agents, including LLM providers, vectorstores, execution, and deployment settings.

______________________________________________________________________

## Deployment Models

Configuration and result models for containerized agent deployment. The canonical deployment config lives at [`holodeck.models.deployment.DeploymentConfig`](#holodeck.models.deployment.DeploymentConfig).

### Enums

## `TagStrategy`

Bases: `str`, `Enum`

Strategy for generating container image tags.

## `CloudProvider`

Bases: `str`, `Enum`

Supported cloud providers for deployment.

## `RuntimeType`

Bases: `str`, `Enum`

Container runtime types.

## `ProtocolType`

Bases: `str`, `Enum`

API protocol types for agent serving.

### Registry

## `RegistryConfig`

Bases: `BaseModel`

Container registry configuration.

Attributes:

| Name                     | Type          | Description                                      |
| ------------------------ | ------------- | ------------------------------------------------ |
| `url`                    | `str`         | Registry URL (e.g., ghcr.io, docker.io, ECR URL) |
| `repository`             | `str`         | Repository name (e.g., org/agent-name)           |
| `tag_strategy`           | `TagStrategy` | Strategy for generating image tags               |
| `custom_tag`             | \`str         | None\`                                           |
| `credentials_env_prefix` | \`str         | None\`                                           |

### `validate_custom_tag()`

Validate that custom_tag is provided when tag_strategy is CUSTOM.

Source code in `src/holodeck/models/deployment.py`

```
@model_validator(mode="after")
def validate_custom_tag(self) -> "RegistryConfig":
    """Validate that custom_tag is provided when tag_strategy is CUSTOM."""
    if self.tag_strategy == TagStrategy.CUSTOM and not self.custom_tag:
        raise ValueError("custom_tag is required when tag_strategy is 'custom'")
    return self
```

### `validate_repository(v)`

Validate repository name pattern.

Source code in `src/holodeck/models/deployment.py`

```
@field_validator("repository")
@classmethod
def validate_repository(cls, v: str) -> str:
    """Validate repository name pattern."""
    if not REPOSITORY_PATTERN.match(v):
        raise ValueError(
            f"Invalid repository name: {v}. "
            "Must contain only lowercase letters, numbers, '.', '_', '/', '-'"
        )
    return v
```

### Cloud Provider Targets

## `AWSAppRunnerConfig`

Bases: `BaseModel`

AWS App Runner deployment configuration.

Attributes:

| Name                | Type  | Description                                                |
| ------------------- | ----- | ---------------------------------------------------------- |
| `region`            | `str` | AWS region for deployment                                  |
| `cpu`               | `int` | vCPU allocation (1, 2, or 4)                               |
| `memory`            | `int` | Memory allocation in MB (2048, 3072, 4096, 8192, or 12288) |
| `ecr_role_arn`      | \`str | None\`                                                     |
| `health_check_path` | `str` | Path for health checks                                     |
| `auto_scaling_min`  | `int` | Minimum instances                                          |
| `auto_scaling_max`  | `int` | Maximum instances                                          |

### `validate_ecr_role_arn(v)`

Validate ECR role ARN format.

Source code in `src/holodeck/models/deployment.py`

```
@field_validator("ecr_role_arn")
@classmethod
def validate_ecr_role_arn(cls, v: str | None) -> str | None:
    """Validate ECR role ARN format."""
    if v is not None and not AWS_ARN_PATTERN.match(v):
        raise ValueError(
            f"Invalid ECR role ARN: {v}. "
            "Must match pattern: arn:aws:iam::<account-id>:role/<role-name>"
        )
    return v
```

### `validate_scaling_range()`

Validate that auto_scaling_min \<= auto_scaling_max.

Source code in `src/holodeck/models/deployment.py`

```
@model_validator(mode="after")
def validate_scaling_range(self) -> "AWSAppRunnerConfig":
    """Validate that auto_scaling_min <= auto_scaling_max."""
    if self.auto_scaling_min > self.auto_scaling_max:
        raise ValueError(
            f"auto_scaling_min ({self.auto_scaling_min}) must be <= "
            f"auto_scaling_max ({self.auto_scaling_max})"
        )
    return self
```

## `GCPCloudRunConfig`

Bases: `BaseModel`

GCP Cloud Run deployment configuration.

Attributes:

| Name            | Type  | Description                              |
| --------------- | ----- | ---------------------------------------- |
| `project_id`    | `str` | GCP project ID                           |
| `region`        | `str` | GCP region for deployment                |
| `memory`        | `str` | Memory allocation (e.g., 512Mi, 1Gi)     |
| `cpu`           | `int` | vCPU allocation (1, 2, or 4)             |
| `concurrency`   | `int` | Maximum concurrent requests per instance |
| `timeout`       | `int` | Request timeout in seconds               |
| `min_instances` | `int` | Minimum instances                        |
| `max_instances` | `int` | Maximum instances                        |

### `validate_memory(v)`

Validate memory format (e.g., 512Mi, 1Gi).

Source code in `src/holodeck/models/deployment.py`

```
@field_validator("memory")
@classmethod
def validate_memory(cls, v: str) -> str:
    """Validate memory format (e.g., 512Mi, 1Gi)."""
    if not GCP_MEMORY_PATTERN.match(v):
        raise ValueError(
            f"Invalid memory format: {v}. Must be a number followed by Mi or Gi."
        )
    return v
```

### `validate_project_id(v)`

Validate GCP project ID format.

Source code in `src/holodeck/models/deployment.py`

```
@field_validator("project_id")
@classmethod
def validate_project_id(cls, v: str) -> str:
    """Validate GCP project ID format."""
    if not GCP_PROJECT_ID_PATTERN.match(v):
        raise ValueError(
            f"Invalid GCP project ID: {v}. "
            "Must be 6-30 lowercase letters, numbers, and hyphens, "
            "starting with a letter and not ending with a hyphen."
        )
    return v
```

## `AzureContainerAppsConfig`

Bases: `BaseModel`

Azure Container Apps deployment configuration.

Attributes:

| Name               | Type    | Description                   |
| ------------------ | ------- | ----------------------------- |
| `subscription_id`  | `str`   | Azure subscription ID (UUID)  |
| `resource_group`   | `str`   | Azure resource group name     |
| `environment_name` | \`str   | None\`                        |
| `location`         | `str`   | Azure region for deployment   |
| `cpu`              | `float` | vCPU allocation               |
| `memory`           | `str`   | Memory allocation (e.g., 2Gi) |
| `ingress_external` | `bool`  | Whether ingress is external   |
| `min_replicas`     | `int`   | Minimum replicas              |
| `max_replicas`     | `int`   | Maximum replicas              |

### `validate_cpu(v)`

Validate CPU is a valid Azure Container Apps value.

Source code in `src/holodeck/models/deployment.py`

```
@field_validator("cpu")
@classmethod
def validate_cpu(cls, v: float) -> float:
    """Validate CPU is a valid Azure Container Apps value."""
    valid_cpus = [0.25, 0.5, 0.75, 1.0, 1.25, 1.5, 1.75, 2.0]
    if v not in valid_cpus:
        raise ValueError(f"Invalid CPU value: {v}. Must be one of: {valid_cpus}")
    return v
```

### `validate_subscription_id(v)`

Validate Azure subscription ID is a valid UUID.

Source code in `src/holodeck/models/deployment.py`

```
@field_validator("subscription_id")
@classmethod
def validate_subscription_id(cls, v: str) -> str:
    """Validate Azure subscription ID is a valid UUID."""
    if not AZURE_UUID_PATTERN.match(v):
        raise ValueError(
            f"Invalid Azure subscription ID: {v}. Must be a valid UUID."
        )
    return v
```

## `CloudTargetConfig`

Bases: `BaseModel`

Cloud deployment target configuration.

Uses a discriminated union pattern to ensure provider-specific configuration matches the selected provider.

Attributes:

| Name       | Type                       | Description                      |
| ---------- | -------------------------- | -------------------------------- |
| `provider` | `CloudProvider`            | Cloud provider (aws, gcp, azure) |
| `aws`      | \`AWSAppRunnerConfig       | None\`                           |
| `gcp`      | \`GCPCloudRunConfig        | None\`                           |
| `azure`    | \`AzureContainerAppsConfig | None\`                           |

### `validate_provider_config()`

Validate that provider-specific config is provided and matches.

Source code in `src/holodeck/models/deployment.py`

```
@model_validator(mode="after")
def validate_provider_config(self) -> "CloudTargetConfig":
    """Validate that provider-specific config is provided and matches."""
    if self.provider == CloudProvider.AWS:
        if self.aws is None:
            raise ValueError("aws configuration is required when provider is 'aws'")
        if self.gcp is not None or self.azure is not None:
            raise ValueError(
                "Only aws configuration should be provided when provider is 'aws'"
            )
    elif self.provider == CloudProvider.GCP:
        if self.gcp is None:
            raise ValueError("gcp configuration is required when provider is 'gcp'")
        if self.aws is not None or self.azure is not None:
            raise ValueError(
                "Only gcp configuration should be provided when provider is 'gcp'"
            )
    elif self.provider == CloudProvider.AZURE:
        if self.azure is None:
            raise ValueError(
                "azure configuration is required when provider is 'azure'"
            )
        if self.aws is not None or self.gcp is not None:
            raise ValueError(
                "Only azure config should be set when provider is 'azure'"
            )
    return self
```

### Main Config and Results

## `DeploymentConfig`

Bases: `BaseModel`

Main deployment configuration model.

Attributes:

| Name                | Type                                    | Description                                                          |
| ------------------- | --------------------------------------- | -------------------------------------------------------------------- |
| `runtime`           | `RuntimeType`                           | Runtime type (currently only container)                              |
| `registry`          | `RegistryConfig`                        | Container registry configuration                                     |
| `target`            | `CloudTargetConfig`                     | Cloud deployment target configuration                                |
| `protocol`          | `ProtocolType`                          | API protocol type                                                    |
| `port`              | `Annotated[int, Field(ge=1, le=65535)]` | Container port to expose                                             |
| `health_check_path` | `str`                                   | HTTP path for health checks (e.g., /health, /healthz)                |
| `environment`       | `dict[str, str]`                        | Environment variables for the container                              |
| `platform`          | `str`                                   | Target platform for container image (e.g., linux/amd64, linux/arm64) |

## `DeployResult`

Bases: `BaseModel`

Result of a deployment operation.

Attributes:

| Name           | Type  | Description                                                           |
| -------------- | ----- | --------------------------------------------------------------------- |
| `service_id`   | `str` | Unique identifier for the deployed service (e.g., Azure resource ID)  |
| `service_name` | `str` | Human-readable name of the deployed service                           |
| `url`          | \`str | None\`                                                                |
| `status`       | `str` | Current deployment status (e.g., "Running", "Provisioning", "Failed") |

## `StatusResult`

Bases: `BaseModel`

Result of a status check operation.

Attributes:

| Name     | Type  | Description                                                      |
| -------- | ----- | ---------------------------------------------------------------- |
| `status` | `str` | Current deployment status (e.g., "Running", "Stopped", "Failed") |
| `url`    | \`str | None\`                                                           |

______________________________________________________________________

## Deployment State

Persisted deployment records stored on disk for tracking active deployments.

## `DeploymentRecord`

Bases: `BaseModel`

Persisted deployment record for a single agent.

## `DeploymentState`

Bases: `BaseModel`

Top-level deployment state stored on disk.

______________________________________________________________________

## Observability Models

OpenTelemetry observability configuration following no-code-first principles.

## `ObservabilityConfig`

Bases: `BaseModel`

Top-level observability configuration.

Root configuration for OpenTelemetry observability features. Service name defaults to 'holodeck-{agent.name}' if not explicitly set.

## `TracingConfig`

Bases: `BaseModel`

Tracing-specific configuration.

Controls distributed tracing settings including sampling rate, content capture, and redaction patterns for sensitive data.

### `validate_queue_size_batch_size()`

Validate max_queue_size >= max_export_batch_size.

Source code in `src/holodeck/models/observability.py`

```
@model_validator(mode="after")
def validate_queue_size_batch_size(self) -> "TracingConfig":
    """Validate max_queue_size >= max_export_batch_size."""
    if self.max_queue_size < self.max_export_batch_size:
        raise ValueError(
            f"max_queue_size ({self.max_queue_size}) must be >= "
            f"max_export_batch_size ({self.max_export_batch_size})"
        )
    return self
```

### `validate_redaction_patterns(v)`

Validate each pattern is valid regex.

Source code in `src/holodeck/models/observability.py`

```
@field_validator("redaction_patterns")
@classmethod
def validate_redaction_patterns(cls, v: list[str]) -> list[str]:
    """Validate each pattern is valid regex."""
    for pattern in v:
        try:
            re.compile(pattern)
        except re.error as e:
            raise ValueError(f"Invalid regex pattern '{pattern}': {e}") from e
    return v
```

## `MetricsConfig`

Bases: `BaseModel`

Metrics-specific configuration.

Controls metrics collection settings including export interval and Semantic Kernel metrics inclusion.

## `LogsConfig`

Bases: `BaseModel`

Logging-specific configuration.

Controls structured logging settings including log level, trace context inclusion, and namespace filtering.

## `LogLevel`

Bases: `str`, `Enum`

Supported log levels for observability.

## `OTLPProtocol`

Bases: `str`, `Enum`

OTLP transport protocol.

### Exporter Configuration

## `ExportersConfig`

Bases: `BaseModel`

Container for exporter configurations.

Note: If no exporters are explicitly enabled, console exporter is used as the default for development/debugging purposes.

### `get_enabled_exporters()`

Return list of enabled exporter names.

Source code in `src/holodeck/models/observability.py`

```
def get_enabled_exporters(self) -> list[str]:
    """Return list of enabled exporter names."""
    enabled = []
    if self.console and self.console.enabled:
        enabled.append("console")
    if self.otlp and self.otlp.enabled:
        enabled.append("otlp")
    if self.prometheus and self.prometheus.enabled:
        enabled.append("prometheus")
    if self.azure_monitor and self.azure_monitor.enabled:
        enabled.append("azure_monitor")
    return enabled
```

### `uses_console_as_default()`

Check if console exporter will be used as default.

Returns True if no other exporters are explicitly enabled, indicating console exporter should be used as fallback.

Source code in `src/holodeck/models/observability.py`

```
def uses_console_as_default(self) -> bool:
    """Check if console exporter will be used as default.

    Returns True if no other exporters are explicitly enabled,
    indicating console exporter should be used as fallback.
    """
    return len(self.get_enabled_exporters()) == 0
```

## `ConsoleExporterConfig`

Bases: `BaseModel`

Console exporter configuration.

Used as the default exporter when no other exporters are explicitly enabled. Outputs telemetry to stdout for development/debugging purposes.

## `OTLPExporterConfig`

Bases: `BaseModel`

OTLP exporter configuration.

Configures export to OTLP collectors (Jaeger, Honeycomb, Datadog, etc.) via gRPC or HTTP protocol.

## `PrometheusExporterConfig`

Bases: `BaseModel`

Prometheus exporter configuration.

Exposes metrics via HTTP endpoint for Prometheus scraping.

## `AzureMonitorExporterConfig`

Bases: `BaseModel`

Azure Monitor exporter configuration.

Exports telemetry directly to Azure Monitor / Application Insights.

### `validate_connection_string_required()`

Ensure connection_string is provided when enabled.

Source code in `src/holodeck/models/observability.py`

```
@model_validator(mode="after")
def validate_connection_string_required(self) -> "AzureMonitorExporterConfig":
    """Ensure connection_string is provided when enabled."""
    if self.enabled and not self.connection_string:
        raise ValueError(
            "Azure Monitor connection string is required when enabled. "
            "Set via config or APPLICATIONINSIGHTS_CONNECTION_STRING env var."
        )
    return self
```

______________________________________________________________________

## MCP Registry Models

Data models for the MCP Registry API at `registry.modelcontextprotocol.io`.

## `RegistryServer`

Bases: `BaseModel`

Complete MCP server representation from registry.

The main model representing an MCP server as returned by the registry API. Contains all metadata needed to discover, display, and install a server.

## `RegistryServerPackage`

Bases: `BaseModel`

Package distribution information from MCP registry.

Describes how to install and run an MCP server, including the package manager, package identifier, and required environment variables.

## `RegistryServerMeta`

Bases: `BaseModel`

Registry metadata for an MCP server.

Contains administrative information about the server's status and publication history in the registry.

## `ServerVersion`

Bases: `BaseModel`

Version-specific information for an MCP server.

Contains the version string along with packages and metadata specific to that version. Used in aggregated search results.

## `TransportConfig`

Bases: `BaseModel`

Transport configuration for MCP server communication.

Defines how to connect to an MCP server, supporting stdio (local process), SSE (server-sent events), or streamable HTTP transports.

## `EnvVarConfig`

Bases: `BaseModel`

Environment variable requirement from MCP registry.

Describes an environment variable that an MCP server requires for configuration (e.g., API keys, credentials).

## `RepositoryInfo`

Bases: `BaseModel`

Source repository information for an MCP server.

Links to the source code repository where the MCP server is maintained.

## `SearchResult`

Bases: `BaseModel`

Search result from MCP registry with pagination.

Returned by the registry search endpoint with a list of matching servers and pagination information for fetching additional results.

______________________________________________________________________

## Template Models

Models for template management, manifests, and project scaffolding.

## `TemplateManifest`

Bases: `BaseModel`

Template metadata and validation rules.

Describes a project template including its variables, defaults, and file structure.

Attributes:

| Name           | Type                        | Description                                                      |
| -------------- | --------------------------- | ---------------------------------------------------------------- |
| `name`         | `str`                       | Template identifier (conversational, research, customer-support) |
| `display_name` | `str`                       | Human-readable name for CLI output                               |
| `description`  | `str`                       | One-line description of template purpose                         |
| `category`     | `str`                       | Use case category (conversational-ai, research-analysis, etc.)   |
| `version`      | `str`                       | Template version (semver format)                                 |
| `variables`    | `dict[str, VariableSchema]` | Allowed template variables with constraints                      |
| `defaults`     | `dict[str, Any]`            | Template-specific default values                                 |
| `files`        | `dict[str, FileMetadata]`   | Files in template and how to process them                        |

### `validate_semver(v)`

Validate version is in semver format.

Parameters:

| Name | Type  | Description                    | Default    |
| ---- | ----- | ------------------------------ | ---------- |
| `v`  | `str` | The version string to validate | *required* |

Returns:

| Type  | Description                  |
| ----- | ---------------------------- |
| `str` | The validated version string |

Raises:

| Type         | Description                    |
| ------------ | ------------------------------ |
| `ValueError` | If version is not valid semver |

Source code in `src/holodeck/models/template_manifest.py`

```
@field_validator("version")
@classmethod
def validate_semver(cls, v: str) -> str:
    """Validate version is in semver format.

    Args:
        v: The version string to validate

    Returns:
        The validated version string

    Raises:
        ValueError: If version is not valid semver
    """
    parts = v.split(".")
    if len(parts) != 3:
        raise ValueError(f"Version must be semver format (MAJOR.MINOR.PATCH): {v}")
    try:
        for part in parts:
            int(part)
    except ValueError as e:
        raise ValueError(f"Version parts must be integers: {v}") from e
    return v
```

## `VariableSchema`

Bases: `BaseModel`

Schema for template variables.

Defines what values a template variable can accept, including type constraints, defaults, and allowed values.

Attributes:

| Name             | Type        | Description                                   |
| ---------------- | ----------- | --------------------------------------------- |
| `type`           | `str`       | Variable type (string, number, boolean, enum) |
| `description`    | `str`       | Description of what the variable controls     |
| `default`        | `Any`       | Default value if not provided                 |
| `required`       | `bool`      | Whether variable must be provided             |
| `allowed_values` | \`list[Any] | None\`                                        |

## `FileMetadata`

Bases: `BaseModel`

Metadata for template files.

Defines how each file in a template should be processed (e.g., Jinja2 rendering vs. direct copy).

Attributes:

| Name       | Type   | Description                            |
| ---------- | ------ | -------------------------------------- |
| `path`     | `str`  | Relative path in generated project     |
| `template` | `bool` | Whether this file is a Jinja2 template |
| `required` | `bool` | Whether this file is always included   |

______________________________________________________________________

## Project Initialization Models

Models for the `holodeck init` command input and output.

## `ProjectInitInput`

Bases: `BaseModel`

User-provided input for project initialization.

This model validates and stores the parameters passed to the init command, ensuring required fields are present and optional fields are properly typed.

Attributes:

| Name              | Type             | Description                                                         |
| ----------------- | ---------------- | ------------------------------------------------------------------- |
| `project_name`    | `str`            | Name of the project to create (alphanumeric, hyphens, underscores)  |
| `template`        | `str`            | Template choice (conversational, research, customer-support)        |
| `description`     | \`str            | None\`                                                              |
| `author`          | \`str            | None\`                                                              |
| `output_dir`      | `str`            | Target directory (currently CWD, but model allows future extension) |
| `overwrite`       | `bool`           | Whether to overwrite existing project                               |
| `agent_name`      | `str`            | Name of the agent to create (from wizard)                           |
| `llm_provider`    | `str`            | Selected LLM provider (from wizard)                                 |
| `provider_config` | \`ProviderConfig | None\`                                                              |
| `vector_store`    | `str`            | Selected vector store (from wizard)                                 |
| `evals`           | `list[str]`      | List of selected evaluation metrics (from wizard)                   |
| `mcp_servers`     | `list[str]`      | List of selected MCP servers (from wizard)                          |

### `validate_agent_name(v)`

Validate agent name format using shared validator.

Parameters:

| Name | Type  | Description                | Default    |
| ---- | ----- | -------------------------- | ---------- |
| `v`  | `str` | The agent name to validate | *required* |

Returns:

| Type  | Description              |
| ----- | ------------------------ |
| `str` | The validated agent name |

Raises:

| Type         | Description              |
| ------------ | ------------------------ |
| `ValueError` | If agent name is invalid |

Source code in `src/holodeck/models/project_config.py`

```
@field_validator("agent_name")
@classmethod
def validate_agent_name(cls, v: str) -> str:
    """Validate agent name format using shared validator.

    Args:
        v: The agent name to validate

    Returns:
        The validated agent name

    Raises:
        ValueError: If agent name is invalid
    """
    return _validate_agent_name(v)
```

### `validate_author(v)`

Validate author field.

Parameters:

| Name | Type  | Description | Default                     |
| ---- | ----- | ----------- | --------------------------- |
| `v`  | \`str | None\`      | The author name to validate |

Returns:

| Type  | Description |
| ----- | ----------- |
| \`str | None\`      |

Raises:

| Type         | Description                |
| ------------ | -------------------------- |
| `ValueError` | If author name is too long |

Source code in `src/holodeck/models/project_config.py`

```
@field_validator("author")
@classmethod
def validate_author(cls, v: str | None) -> str | None:
    """Validate author field.

    Args:
        v: The author name to validate

    Returns:
        The validated author name

    Raises:
        ValueError: If author name is too long
    """
    if v is not None and len(v) > 256:
        raise ValueError("Author name must be 256 characters or less")
    return v
```

### `validate_description(v)`

Validate description field.

Parameters:

| Name | Type  | Description | Default                     |
| ---- | ----- | ----------- | --------------------------- |
| `v`  | \`str | None\`      | The description to validate |

Returns:

| Type  | Description |
| ----- | ----------- |
| \`str | None\`      |

Raises:

| Type         | Description                |
| ------------ | -------------------------- |
| `ValueError` | If description is too long |

Source code in `src/holodeck/models/project_config.py`

```
@field_validator("description")
@classmethod
def validate_description(cls, v: str | None) -> str | None:
    """Validate description field.

    Args:
        v: The description to validate

    Returns:
        The validated description

    Raises:
        ValueError: If description is too long
    """
    if v is not None and len(v) > 1000:
        raise ValueError("Description must be 1000 characters or less")
    return v
```

### `validate_evals(v)`

Validate evaluation metrics choices.

Parameters:

| Name | Type        | Description                                | Default    |
| ---- | ----------- | ------------------------------------------ | ---------- |
| `v`  | `list[str]` | The list of evaluation metrics to validate | *required* |

Returns:

| Type        | Description                              |
| ----------- | ---------------------------------------- |
| `list[str]` | The validated list of evaluation metrics |

Raises:

| Type         | Description                          |
| ------------ | ------------------------------------ |
| `ValueError` | If any eval metric is not recognized |

Source code in `src/holodeck/models/project_config.py`

```
@field_validator("evals")
@classmethod
def validate_evals(cls, v: list[str]) -> list[str]:
    """Validate evaluation metrics choices.

    Args:
        v: The list of evaluation metrics to validate

    Returns:
        The validated list of evaluation metrics

    Raises:
        ValueError: If any eval metric is not recognized
    """
    invalid = [e for e in v if e not in VALID_EVALS]
    if invalid:
        valid = ", ".join(sorted(VALID_EVALS))
        raise ValueError(
            f"Invalid evaluation metric(s): {', '.join(invalid)}. "
            f"Valid options: {valid}"
        )
    return v
```

### `validate_llm_provider(v)`

Validate LLM provider choice.

Parameters:

| Name | Type  | Description                  | Default    |
| ---- | ----- | ---------------------------- | ---------- |
| `v`  | `str` | The LLM provider to validate | *required* |

Returns:

| Type  | Description                |
| ----- | -------------------------- |
| `str` | The validated LLM provider |

Raises:

| Type         | Description                       |
| ------------ | --------------------------------- |
| `ValueError` | If LLM provider is not recognized |

Source code in `src/holodeck/models/project_config.py`

```
@field_validator("llm_provider")
@classmethod
def validate_llm_provider(cls, v: str) -> str:
    """Validate LLM provider choice.

    Args:
        v: The LLM provider to validate

    Returns:
        The validated LLM provider

    Raises:
        ValueError: If LLM provider is not recognized
    """
    if v not in VALID_LLM_PROVIDERS:
        valid = ", ".join(sorted(VALID_LLM_PROVIDERS))
        raise ValueError(f"Invalid LLM provider: {v}. Valid options: {valid}")
    return v
```

### `validate_mcp_servers(v)`

Validate MCP server choices.

Parameters:

| Name | Type        | Description                         | Default    |
| ---- | ----------- | ----------------------------------- | ---------- |
| `v`  | `list[str]` | The list of MCP servers to validate | *required* |

Returns:

| Type        | Description                       |
| ----------- | --------------------------------- |
| `list[str]` | The validated list of MCP servers |

Raises:

| Type         | Description                         |
| ------------ | ----------------------------------- |
| `ValueError` | If any MCP server is not recognized |

Source code in `src/holodeck/models/project_config.py`

```
@field_validator("mcp_servers")
@classmethod
def validate_mcp_servers(cls, v: list[str]) -> list[str]:
    """Validate MCP server choices.

    Args:
        v: The list of MCP servers to validate

    Returns:
        The validated list of MCP servers

    Raises:
        ValueError: If any MCP server is not recognized
    """
    invalid = [s for s in v if s not in VALID_MCP_SERVERS]
    if invalid:
        valid = ", ".join(sorted(VALID_MCP_SERVERS))
        raise ValueError(
            f"Invalid MCP server(s): {', '.join(invalid)}. Valid options: {valid}"
        )
    return v
```

### `validate_project_name(v)`

Validate project name format.

Parameters:

| Name | Type  | Description                  | Default    |
| ---- | ----- | ---------------------------- | ---------- |
| `v`  | `str` | The project name to validate | *required* |

Returns:

| Type  | Description                |
| ----- | -------------------------- |
| `str` | The validated project name |

Raises:

| Type         | Description                |
| ------------ | -------------------------- |
| `ValueError` | If project name is invalid |

Source code in `src/holodeck/models/project_config.py`

```
@field_validator("project_name")
@classmethod
def validate_project_name(cls, v: str) -> str:
    """Validate project name format.

    Args:
        v: The project name to validate

    Returns:
        The validated project name

    Raises:
        ValueError: If project name is invalid
    """
    if not v:
        raise ValueError("Project name cannot be empty")
    if len(v) > 64:
        raise ValueError("Project name must be 64 characters or less")
    if v[0].isdigit():
        raise ValueError("Project name cannot start with a digit")
    if not all(c.isalnum() or c in "-_" for c in v):
        msg = (
            "Project name can only contain alphanumeric characters, "
            "hyphens, and underscores"
        )
        raise ValueError(msg)
    return v
```

### `validate_template(v)`

Validate template choice.

Parameters:

| Name | Type  | Description                   | Default    |
| ---- | ----- | ----------------------------- | ---------- |
| `v`  | `str` | The template name to validate | *required* |

Returns:

| Type  | Description                 |
| ----- | --------------------------- |
| `str` | The validated template name |

Raises:

| Type         | Description                   |
| ------------ | ----------------------------- |
| `ValueError` | If template is not recognized |

Source code in `src/holodeck/models/project_config.py`

```
@field_validator("template")
@classmethod
def validate_template(cls, v: str) -> str:
    """Validate template choice.

    Args:
        v: The template name to validate

    Returns:
        The validated template name

    Raises:
        ValueError: If template is not recognized
    """
    from holodeck.lib.template_engine import TemplateRenderer

    available_templates = set(TemplateRenderer.list_available_templates())
    if v not in available_templates:
        templates_list = ", ".join(sorted(available_templates))
        msg = f"Unknown template: {v}. Valid templates: {templates_list}"
        raise ValueError(msg)
    return v
```

### `validate_vector_store(v)`

Validate vector store choice.

Parameters:

| Name | Type  | Description                  | Default    |
| ---- | ----- | ---------------------------- | ---------- |
| `v`  | `str` | The vector store to validate | *required* |

Returns:

| Type  | Description                |
| ----- | -------------------------- |
| `str` | The validated vector store |

Raises:

| Type         | Description                       |
| ------------ | --------------------------------- |
| `ValueError` | If vector store is not recognized |

Source code in `src/holodeck/models/project_config.py`

```
@field_validator("vector_store")
@classmethod
def validate_vector_store(cls, v: str) -> str:
    """Validate vector store choice.

    Args:
        v: The vector store to validate

    Returns:
        The validated vector store

    Raises:
        ValueError: If vector store is not recognized
    """
    if v not in VALID_VECTOR_STORES:
        valid = ", ".join(sorted(VALID_VECTOR_STORES))
        raise ValueError(f"Invalid vector store: {v}. Valid options: {valid}")
    return v
```

## `ProjectInitResult`

Bases: `BaseModel`

Outcome of project initialization.

This model captures the result of a project initialization attempt, including success status, paths, file list, and any errors or warnings.

Attributes:

| Name               | Type        | Description                                   |
| ------------------ | ----------- | --------------------------------------------- |
| `success`          | `bool`      | Whether initialization completed successfully |
| `project_name`     | `str`       | Name of created project                       |
| `project_path`     | `str`       | Absolute path to created project directory    |
| `template_used`    | `str`       | Which template was applied                    |
| `files_created`    | `list[str]` | List of relative paths of created files       |
| `warnings`         | `list[str]` | Non-blocking issues (e.g., permission notes)  |
| `errors`           | `list[str]` | Blocking errors that prevented creation       |
| `duration_seconds` | `float`     | Time taken for initialization                 |

______________________________________________________________________

## Wizard Models

Interactive initialization wizard state, choices, and results.

### State and Results

## `WizardStep`

Bases: `str`, `Enum`

Steps in the interactive wizard flow.

Each step represents a prompt in the wizard sequence.

## `WizardState`

Bases: `BaseModel`

Runtime state tracking for the wizard flow.

This model tracks the user's progress through the wizard, storing intermediate selections until the wizard completes.

Attributes:

| Name           | Type         | Description                           |
| -------------- | ------------ | ------------------------------------- |
| `current_step` | `WizardStep` | The current step in the wizard flow   |
| `agent_name`   | \`str        | None\`                                |
| `llm_provider` | \`str        | None\`                                |
| `vector_store` | \`str        | None\`                                |
| `evals`        | `list[str]`  | List of selected evaluation metrics   |
| `mcp_servers`  | `list[str]`  | List of selected MCP servers          |
| `is_cancelled` | `bool`       | Whether the user cancelled the wizard |

## `WizardResult`

Bases: `BaseModel`

Final validated result from the wizard.

This immutable model contains all validated selections from the wizard, ready to be used for project initialization.

Attributes:

| Name              | Type             | Description                                               |
| ----------------- | ---------------- | --------------------------------------------------------- |
| `agent_name`      | `str`            | Validated agent name (alphanumeric, hyphens, underscores) |
| `template`        | `str`            | Selected project template                                 |
| `llm_provider`    | `str`            | Selected LLM provider                                     |
| `provider_config` | \`ProviderConfig | None\`                                                    |
| `vector_store`    | `str`            | Selected vector store                                     |
| `evals`           | `list[str]`      | List of selected evaluation metrics (can be empty)        |
| `mcp_servers`     | `list[str]`      | List of selected MCP servers (can be empty)               |

### `validate_agent_name(v)`

Validate agent name format using shared validator.

Parameters:

| Name | Type  | Description                | Default    |
| ---- | ----- | -------------------------- | ---------- |
| `v`  | `str` | The agent name to validate | *required* |

Returns:

| Type  | Description              |
| ----- | ------------------------ |
| `str` | The validated agent name |

Raises:

| Type         | Description              |
| ------------ | ------------------------ |
| `ValueError` | If agent name is invalid |

Source code in `src/holodeck/models/wizard_config.py`

```
@field_validator("agent_name")
@classmethod
def validate_agent_name(cls, v: str) -> str:
    """Validate agent name format using shared validator.

    Args:
        v: The agent name to validate

    Returns:
        The validated agent name

    Raises:
        ValueError: If agent name is invalid
    """
    return _validate_agent_name(v)
```

### `validate_evals(v)`

Validate evaluation metrics choices.

Parameters:

| Name | Type        | Description                                | Default    |
| ---- | ----------- | ------------------------------------------ | ---------- |
| `v`  | `list[str]` | The list of evaluation metrics to validate | *required* |

Returns:

| Type        | Description                              |
| ----------- | ---------------------------------------- |
| `list[str]` | The validated list of evaluation metrics |

Raises:

| Type         | Description                          |
| ------------ | ------------------------------------ |
| `ValueError` | If any eval metric is not recognized |

Source code in `src/holodeck/models/wizard_config.py`

```
@field_validator("evals")
@classmethod
def validate_evals(cls, v: list[str]) -> list[str]:
    """Validate evaluation metrics choices.

    Args:
        v: The list of evaluation metrics to validate

    Returns:
        The validated list of evaluation metrics

    Raises:
        ValueError: If any eval metric is not recognized
    """
    invalid = [e for e in v if e not in VALID_EVALS]
    if invalid:
        valid = ", ".join(sorted(VALID_EVALS))
        raise ValueError(
            f"Invalid evaluation metric(s): {', '.join(invalid)}. "
            f"Valid options: {valid}"
        )
    return v
```

### `validate_llm_provider(v)`

Validate LLM provider choice.

Parameters:

| Name | Type  | Description                  | Default    |
| ---- | ----- | ---------------------------- | ---------- |
| `v`  | `str` | The LLM provider to validate | *required* |

Returns:

| Type  | Description                |
| ----- | -------------------------- |
| `str` | The validated LLM provider |

Raises:

| Type         | Description                       |
| ------------ | --------------------------------- |
| `ValueError` | If LLM provider is not recognized |

Source code in `src/holodeck/models/wizard_config.py`

```
@field_validator("llm_provider")
@classmethod
def validate_llm_provider(cls, v: str) -> str:
    """Validate LLM provider choice.

    Args:
        v: The LLM provider to validate

    Returns:
        The validated LLM provider

    Raises:
        ValueError: If LLM provider is not recognized
    """
    if v not in VALID_LLM_PROVIDERS:
        valid = ", ".join(sorted(VALID_LLM_PROVIDERS))
        raise ValueError(f"Invalid LLM provider: {v}. Valid options: {valid}")
    return v
```

### `validate_mcp_servers(v)`

Validate MCP server choices.

Parameters:

| Name | Type        | Description                         | Default    |
| ---- | ----------- | ----------------------------------- | ---------- |
| `v`  | `list[str]` | The list of MCP servers to validate | *required* |

Returns:

| Type        | Description                       |
| ----------- | --------------------------------- |
| `list[str]` | The validated list of MCP servers |

Raises:

| Type         | Description                         |
| ------------ | ----------------------------------- |
| `ValueError` | If any MCP server is not recognized |

Source code in `src/holodeck/models/wizard_config.py`

```
@field_validator("mcp_servers")
@classmethod
def validate_mcp_servers(cls, v: list[str]) -> list[str]:
    """Validate MCP server choices.

    Args:
        v: The list of MCP servers to validate

    Returns:
        The validated list of MCP servers

    Raises:
        ValueError: If any MCP server is not recognized
    """
    invalid = [s for s in v if s not in VALID_MCP_SERVERS]
    if invalid:
        valid = ", ".join(sorted(VALID_MCP_SERVERS))
        raise ValueError(
            f"Invalid MCP server(s): {', '.join(invalid)}. Valid options: {valid}"
        )
    return v
```

### `validate_template(v)`

Validate template is available.

Parameters:

| Name | Type  | Description                   | Default    |
| ---- | ----- | ----------------------------- | ---------- |
| `v`  | `str` | The template name to validate | *required* |

Returns:

| Type  | Description                 |
| ----- | --------------------------- |
| `str` | The validated template name |

Raises:

| Type         | Description                   |
| ------------ | ----------------------------- |
| `ValueError` | If template is not recognized |

Source code in `src/holodeck/models/wizard_config.py`

```
@field_validator("template")
@classmethod
def validate_template(cls, v: str) -> str:
    """Validate template is available.

    Args:
        v: The template name to validate

    Returns:
        The validated template name

    Raises:
        ValueError: If template is not recognized
    """
    from holodeck.lib.template_engine import TemplateRenderer

    templates = TemplateRenderer.get_available_templates()
    available = [t["value"] for t in templates]
    if v not in available:
        valid = ", ".join(sorted(available))
        raise ValueError(f"Invalid template: {v}. Valid options: {valid}")
    return v
```

### `validate_vector_store(v)`

Validate vector store choice.

Parameters:

| Name | Type  | Description                  | Default    |
| ---- | ----- | ---------------------------- | ---------- |
| `v`  | `str` | The vector store to validate | *required* |

Returns:

| Type  | Description                |
| ----- | -------------------------- |
| `str` | The validated vector store |

Raises:

| Type         | Description                       |
| ------------ | --------------------------------- |
| `ValueError` | If vector store is not recognized |

Source code in `src/holodeck/models/wizard_config.py`

```
@field_validator("vector_store")
@classmethod
def validate_vector_store(cls, v: str) -> str:
    """Validate vector store choice.

    Args:
        v: The vector store to validate

    Returns:
        The validated vector store

    Raises:
        ValueError: If vector store is not recognized
    """
    if v not in VALID_VECTOR_STORES:
        valid = ", ".join(sorted(VALID_VECTOR_STORES))
        raise ValueError(f"Invalid vector store: {v}. Valid options: {valid}")
    return v
```

## `ProviderConfig`

Bases: `BaseModel`

Provider-specific configuration collected from wizard prompts.

This model holds provider-specific settings like endpoint URLs that are collected via follow-up prompts after selecting certain LLM providers (e.g., Azure OpenAI).

Attributes:

| Name       | Type  | Description |
| ---------- | ----- | ----------- |
| `endpoint` | \`str | None\`      |

### Choice Models

## `TemplateChoice`

Bases: `BaseModel`

Template option for the wizard.

This model represents a single project template choice displayed in the wizard selection prompt.

Attributes:

| Name           | Type  | Description                                              |
| -------------- | ----- | -------------------------------------------------------- |
| `value`        | `str` | Template identifier (e.g., 'conversational', 'research') |
| `display_name` | `str` | Human-readable name shown in the prompt                  |
| `description`  | `str` | Help text explaining the template purpose                |

## `LLMProviderChoice`

Bases: `BaseModel`

LLM provider option for the wizard.

This model represents a single LLM provider choice displayed in the wizard selection prompt.

Attributes:

| Name                | Type   | Description                                    |
| ------------------- | ------ | ---------------------------------------------- |
| `value`             | `str`  | Provider identifier (e.g., 'ollama', 'openai') |
| `display_name`      | `str`  | Human-readable name shown in the prompt        |
| `description`       | `str`  | Help text explaining the provider              |
| `is_default`        | `bool` | Whether this is the default selection          |
| `default_model`     | `str`  | Default model name for this provider           |
| `requires_api_key`  | `bool` | Whether an API key is required                 |
| `api_key_env_var`   | \`str  | None\`                                         |
| `requires_endpoint` | `bool` | Whether a custom endpoint is needed (Azure)    |
| `endpoint_env_var`  | \`str  | None\`                                         |

## `VectorStoreChoice`

Bases: `BaseModel`

Vector store option for the wizard.

This model represents a single vector store choice displayed in the wizard selection prompt.

Attributes:

| Name                  | Type   | Description                                  |
| --------------------- | ------ | -------------------------------------------- |
| `value`               | `str`  | Store identifier (e.g., 'chromadb', 'redis') |
| `display_name`        | `str`  | Human-readable name shown in the prompt      |
| `description`         | `str`  | Help text explaining the store               |
| `is_default`          | `bool` | Whether this is the default selection        |
| `default_endpoint`    | \`str  | None\`                                       |
| `persistence`         | `str`  | Storage type (local, remote, none)           |
| `connection_required` | `bool` | Whether a connection is needed at runtime    |

## `EvalChoice`

Bases: `BaseModel`

Evaluation metric option for the wizard.

This model represents a single evaluation metric choice displayed in the wizard multi-select prompt.

Attributes:

| Name           | Type   | Description                                                |
| -------------- | ------ | ---------------------------------------------------------- |
| `value`        | `str`  | Metric identifier (e.g., 'rag-faithfulness')               |
| `display_name` | `str`  | Human-readable name shown in the prompt                    |
| `description`  | `str`  | Help text explaining what the metric measures              |
| `is_default`   | `bool` | Whether this metric is pre-selected by default             |
| `metric_type`  | `str`  | Type of metric ('ai' for LLM-based, 'nlp' for traditional) |

## `MCPServerChoice`

Bases: `BaseModel`

MCP server option for the wizard.

This model represents a single MCP (Model Context Protocol) server choice displayed in the wizard multi-select prompt.

Attributes:

| Name                 | Type   | Description                                     |
| -------------------- | ------ | ----------------------------------------------- |
| `value`              | `str`  | Server identifier (e.g., 'brave-search')        |
| `display_name`       | `str`  | Human-readable name shown in the prompt         |
| `description`        | `str`  | Help text explaining the server's functionality |
| `is_default`         | `bool` | Whether this server is pre-selected by default  |
| `package_identifier` | `str`  | NPM package name for the MCP server             |
| `command`            | `str`  | Execution command (defaults to 'npx')           |

______________________________________________________________________

## Error Hierarchy

Custom exception classes from `holodeck.lib.errors` and `holodeck.lib.backends.base`. All exceptions inherit from `HoloDeckError`.

### Base Exception

## `HoloDeckError`

Bases: `Exception`

Base exception for all HoloDeck errors.

All HoloDeck-specific exceptions inherit from this class, enabling centralized exception handling and error tracking.

### Configuration and Validation Errors

## `ConfigError(field, message)`

Bases: `HoloDeckError`

Exception raised for configuration errors.

This exception is raised when configuration loading or parsing fails. It includes field-specific information to help users identify and fix configuration issues.

Attributes:

| Name      | Type | Description                                       |
| --------- | ---- | ------------------------------------------------- |
| `field`   |      | The configuration field that caused the error     |
| `message` |      | Human-readable error message describing the issue |

Initialize ConfigError with field and message.

Parameters:

| Name      | Type  | Description                                   | Default    |
| --------- | ----- | --------------------------------------------- | ---------- |
| `field`   | `str` | Configuration field name where error occurred | *required* |
| `message` | `str` | Descriptive error message                     | *required* |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, field: str, message: str) -> None:
    """Initialize ConfigError with field and message.

    Args:
        field: Configuration field name where error occurred
        message: Descriptive error message
    """
    self.field = field
    self.message = message
    super().__init__(f"Configuration error in '{field}': {message}")
```

## `ValidationError(field, message, expected, actual)`

Bases: `HoloDeckError`

Exception raised for validation errors during configuration parsing.

Provides detailed information about what was expected versus what was received, enabling users to quickly understand and fix validation issues.

Attributes:

| Name       | Type | Description                              |
| ---------- | ---- | ---------------------------------------- |
| `field`    |      | The field that failed validation         |
| `message`  |      | Description of the validation failure    |
| `expected` |      | Human description of expected value/type |
| `actual`   |      | The actual value that failed validation  |

Initialize ValidationError with detailed information.

Parameters:

| Name       | Type  | Description                                                           | Default    |
| ---------- | ----- | --------------------------------------------------------------------- | ---------- |
| `field`    | `str` | Field that failed validation (can use dot notation for nested fields) | *required* |
| `message`  | `str` | Description of what went wrong                                        | *required* |
| `expected` | `str` | Human-readable description of expected value                          | *required* |
| `actual`   | `str` | The actual value that failed                                          | *required* |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(
    self,
    field: str,
    message: str,
    expected: str,
    actual: str,
) -> None:
    """Initialize ValidationError with detailed information.

    Args:
        field: Field that failed validation (can use dot notation for nested fields)
        message: Description of what went wrong
        expected: Human-readable description of expected value
        actual: The actual value that failed
    """
    self.field = field
    self.message = message
    self.expected = expected
    self.actual = actual
    full_message = (
        f"Validation error in '{field}': {message}\n"
        f"  Expected: {expected}\n"
        f"  Got: {actual}"
    )
    super().__init__(full_message)
```

## `FileNotFoundError(path, message)`

Bases: `HoloDeckError`

Exception raised when a configuration file is not found.

Includes the file path and helpful suggestions for resolving the issue.

Attributes:

| Name      | Type | Description                         |
| --------- | ---- | ----------------------------------- |
| `path`    |      | Path to the file that was not found |
| `message` |      | Human-readable error message        |

Initialize FileNotFoundError with path and message.

Parameters:

| Name      | Type  | Description                                            | Default    |
| --------- | ----- | ------------------------------------------------------ | ---------- |
| `path`    | `str` | Path to the file that was not found                    | *required* |
| `message` | `str` | Descriptive error message, optionally with suggestions | *required* |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, path: str, message: str) -> None:
    """Initialize FileNotFoundError with path and message.

    Args:
        path: Path to the file that was not found
        message: Descriptive error message, optionally with suggestions
    """
    self.path = path
    self.message = message
    super().__init__(f"File not found: {path}\n{message}")
```

### Execution Errors

## `ExecutionError`

Bases: `HoloDeckError`

Exception raised when test execution fails.

Covers timeout, agent invocation errors, and other runtime failures during test execution.

Attributes:

| Name      | Type | Description                  |
| --------- | ---- | ---------------------------- |
| `message` |      | Human-readable error message |

## `EvaluationError`

Bases: `HoloDeckError`

Exception raised when metric evaluation fails.

Covers failures in evaluator initialization or metric calculation.

Attributes:

| Name      | Type | Description                  |
| --------- | ---- | ---------------------------- |
| `message` |      | Human-readable error message |

### Agent Errors

## `AgentInitializationError(agent_name, message)`

Bases: `HoloDeckError`

Exception raised when an agent fails to initialize.

Create an initialization error with context.

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, agent_name: str, message: str) -> None:
    """Create an initialization error with context."""
    self.agent_name = agent_name
    self.message = message
    super().__init__(f"Agent '{agent_name}' failed to initialize: {message}")
```

### Chat Errors

## `ChatValidationError(message)`

Bases: `HoloDeckError`

Exception raised when chat input validation fails.

Create a validation error for chat messages.

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, message: str) -> None:
    """Create a validation error for chat messages."""
    self.message = message
    super().__init__(message)
```

## `ChatSessionError(message)`

Bases: `HoloDeckError`

Exception raised for chat session lifecycle failures.

Create a chat session error.

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, message: str) -> None:
    """Create a chat session error."""
    self.message = message
    super().__init__(message)
```

### MCP Registry Errors

## `RegistryConnectionError(url, original_error=None)`

Bases: `HoloDeckError`

Exception raised when connection to MCP registry fails.

Raised for network timeouts, DNS failures, and connection refused errors.

Attributes:

| Name      | Type | Description                                           |
| --------- | ---- | ----------------------------------------------------- |
| `url`     |      | The registry URL that failed                          |
| `message` |      | Human-readable error message with resolution guidance |

Initialize RegistryConnectionError with URL and optional cause.

Parameters:

| Name             | Type        | Description                             | Default                                          |
| ---------------- | ----------- | --------------------------------------- | ------------------------------------------------ |
| `url`            | `str`       | The registry URL that failed to connect | *required*                                       |
| `original_error` | \`Exception | None\`                                  | The underlying exception that caused the failure |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, url: str, original_error: Exception | None = None) -> None:
    """Initialize RegistryConnectionError with URL and optional cause.

    Args:
        url: The registry URL that failed to connect
        original_error: The underlying exception that caused the failure
    """
    self.url = url
    message = (
        f"Failed to connect to MCP registry at {url}.\n"
        f"Check your network connection and try again."
    )
    if original_error:
        message += f"\nOriginal error: {original_error}"
    super().__init__(message)
```

## `RegistryAPIError(url, status_code, detail=None)`

Bases: `HoloDeckError`

Exception raised when MCP registry returns an error response.

Raised for HTTP error status codes from the registry API.

Attributes:

| Name          | Type | Description                              |
| ------------- | ---- | ---------------------------------------- |
| `url`         |      | The registry URL that returned the error |
| `status_code` |      | HTTP status code returned                |
| `message`     |      | Human-readable error message             |

Initialize RegistryAPIError with response details.

Parameters:

| Name          | Type  | Description                              | Default                                  |
| ------------- | ----- | ---------------------------------------- | ---------------------------------------- |
| `url`         | `str` | The registry URL that returned the error | *required*                               |
| `status_code` | `int` | HTTP status code                         | *required*                               |
| `detail`      | \`str | None\`                                   | Optional error detail from response body |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, url: str, status_code: int, detail: str | None = None) -> None:
    """Initialize RegistryAPIError with response details.

    Args:
        url: The registry URL that returned the error
        status_code: HTTP status code
        detail: Optional error detail from response body
    """
    self.url = url
    self.status_code = status_code

    if status_code == 429:
        message = "Rate limited by MCP registry. Please wait and try again."
    elif status_code >= 500:
        message = "MCP registry service error. Try again later."
    else:
        message = f"MCP registry error (HTTP {status_code})"

    if detail:
        message += f": {detail}"

    super().__init__(message)
```

## `ServerNotFoundError(server_name, location=None)`

Bases: `HoloDeckError`

Exception raised when requested MCP server is not found.

Can be raised in two contexts:

1. Server not found in MCP registry (during search/add)
1. Server not found in local configuration (during remove)

Attributes:

| Name          | Type | Description                                                            |
| ------------- | ---- | ---------------------------------------------------------------------- |
| `server_name` |      | The server name that was not found                                     |
| `location`    |      | Optional location context (e.g., "agent.yaml", "global configuration") |
| `message`     |      | Human-readable error message                                           |

Initialize ServerNotFoundError with server name.

Parameters:

| Name          | Type  | Description                               | Default                                     |
| ------------- | ----- | ----------------------------------------- | ------------------------------------------- |
| `server_name` | `str` | The name of the server that was not found | *required*                                  |
| `location`    | \`str | None\`                                    | Optional location where server was expected |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, server_name: str, location: str | None = None) -> None:
    """Initialize ServerNotFoundError with server name.

    Args:
        server_name: The name of the server that was not found
        location: Optional location where server was expected
    """
    self.server_name = server_name
    self.location = location
    if location:
        message = f"Server '{server_name}' not found in {location}."
    else:
        message = f"Server '{server_name}' not found in MCP registry."
    super().__init__(message)
```

## `DuplicateServerError(server_name, registry_name=None, existing_registry_name=None)`

Bases: `HoloDeckError`

Exception raised when attempting to add an MCP server that already exists.

Raised when a server with the same registry_name (for registry servers) or name (for manual servers) is already configured.

Attributes:

| Name                     | Type | Description                                      |
| ------------------------ | ---- | ------------------------------------------------ |
| `server_name`            |      | The short name of the server being added         |
| `registry_name`          |      | The full registry name of the server being added |
| `existing_registry_name` |      | The registry name of the existing server         |

Initialize DuplicateServerError with server details.

Parameters:

| Name                     | Type  | Description                              | Default                                          |
| ------------------------ | ----- | ---------------------------------------- | ------------------------------------------------ |
| `server_name`            | `str` | The short name of the server being added | *required*                                       |
| `registry_name`          | \`str | None\`                                   | The full registry name of the server being added |
| `existing_registry_name` | \`str | None\`                                   | The registry name of the existing server         |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(
    self,
    server_name: str,
    registry_name: str | None = None,
    existing_registry_name: str | None = None,
) -> None:
    """Initialize DuplicateServerError with server details.

    Args:
        server_name: The short name of the server being added
        registry_name: The full registry name of the server being added
        existing_registry_name: The registry name of the existing server
    """
    self.server_name = server_name
    self.registry_name = registry_name
    self.existing_registry_name = existing_registry_name

    if registry_name and registry_name == existing_registry_name:
        # Exact duplicate (same registry server)
        message = (
            f"Server '{registry_name}' is already configured. "
            f"Use --version to install a different version, "
            f"or remove the existing server first."
        )
    elif existing_registry_name:
        # Name conflict between different registry servers
        message = (
            f"A server named '{server_name}' already exists "
            f"(from '{existing_registry_name}'). "
            f"Use --name to specify a different name."
        )
    else:
        # Manual server with same name
        message = (
            f"A server named '{server_name}' already exists. "
            f"Use --name to specify a different name, "
            f"or remove the existing server first."
        )

    super().__init__(message)
```

## `RecordPathError(path, available_keys, message)`

Bases: `HoloDeckError`

Exception raised when navigating a record path in JSON structure fails.

Raised when a configured record_path cannot be resolved in the source data. Provides helpful information about available keys at the point of failure.

Attributes:

| Name             | Type | Description                                   |
| ---------------- | ---- | --------------------------------------------- |
| `path`           |      | The record_path that failed to navigate       |
| `available_keys` |      | List of available keys at the failure point   |
| `message`        |      | Detailed error message describing the failure |

Initialize RecordPathError with navigation details.

Parameters:

| Name             | Type        | Description                                      | Default    |
| ---------------- | ----------- | ------------------------------------------------ | ---------- |
| `path`           | `str`       | The record_path that failed (e.g., "data.items") | *required* |
| `available_keys` | `list[str]` | Keys available at the point of failure           | *required* |
| `message`        | `str`       | Detailed description of what went wrong          | *required* |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(
    self,
    path: str,
    available_keys: list[str],
    message: str,
) -> None:
    """Initialize RecordPathError with navigation details.

    Args:
        path: The record_path that failed (e.g., "data.items")
        available_keys: Keys available at the point of failure
        message: Detailed description of what went wrong
    """
    self.path = path
    self.available_keys = available_keys
    self.message = message
    super().__init__(
        f"Failed to navigate path '{path}': {message}. "
        f"Available keys: {available_keys}"
    )
```

### Deployment Errors

## `DeploymentError(operation, message)`

Bases: `HoloDeckError`

Base exception for deployment operations.

Parent class for all deployment-related errors including Docker, registry, and cloud provider operations.

Attributes:

| Name        | Type | Description                          |
| ----------- | ---- | ------------------------------------ |
| `operation` |      | The deployment operation that failed |
| `message`   |      | Human-readable error message         |

Initialize DeploymentError with operation context.

Parameters:

| Name        | Type  | Description                          | Default    |
| ----------- | ----- | ------------------------------------ | ---------- |
| `operation` | `str` | The deployment operation that failed | *required* |
| `message`   | `str` | Descriptive error message            | *required* |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, operation: str, message: str) -> None:
    """Initialize DeploymentError with operation context.

    Args:
        operation: The deployment operation that failed
        message: Descriptive error message
    """
    self.operation = operation
    self.message = message
    super().__init__(f"Deployment error during '{operation}': {message}")
```

## `DockerNotAvailableError(operation='docker')`

Bases: `DeploymentError`

Exception raised when Docker daemon is not available.

Raised when Docker operations fail due to the daemon not running or Docker not being installed.

Attributes:

| Name        | Type | Description                                           |
| ----------- | ---- | ----------------------------------------------------- |
| `operation` |      | The operation that required Docker                    |
| `message`   |      | Human-readable error message with resolution guidance |

Initialize DockerNotAvailableError.

Parameters:

| Name        | Type  | Description                        | Default    |
| ----------- | ----- | ---------------------------------- | ---------- |
| `operation` | `str` | The operation that required Docker | `'docker'` |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, operation: str = "docker") -> None:
    """Initialize DockerNotAvailableError.

    Args:
        operation: The operation that required Docker
    """
    message = (
        "Docker daemon is not available.\n"
        "Ensure Docker is installed and running:\n"
        "  - macOS/Windows: Start Docker Desktop\n"
        "  - Linux: sudo systemctl start docker"
    )
    super().__init__(operation, message)
```

## `RegistryAuthError(registry, detail=None)`

Bases: `DeploymentError`

Exception raised when container registry authentication fails.

Raised when pushing images to a registry fails due to authentication or authorization issues.

Attributes:

| Name        | Type | Description                                 |
| ----------- | ---- | ------------------------------------------- |
| `registry`  |      | The registry URL that failed authentication |
| `operation` |      | The operation that failed                   |
| `message`   |      | Human-readable error message                |

Initialize RegistryAuthError with registry details.

Parameters:

| Name       | Type  | Description                                 | Default                          |
| ---------- | ----- | ------------------------------------------- | -------------------------------- |
| `registry` | `str` | The registry URL that failed authentication | *required*                       |
| `detail`   | \`str | None\`                                      | Optional additional error detail |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, registry: str, detail: str | None = None) -> None:
    """Initialize RegistryAuthError with registry details.

    Args:
        registry: The registry URL that failed authentication
        detail: Optional additional error detail
    """
    self.registry = registry
    message = f"Authentication failed for registry '{registry}'."
    if detail:
        message += f"\n{detail}"
    message += (
        "\nEnsure credentials are configured:\n"
        "  - AWS ECR: aws ecr get-login-password | docker login\n"
        "  - GCP: gcloud auth configure-docker\n"
        "  - Azure: az acr login --name <registry>"
    )
    super().__init__("push", message)
```

## `CloudSDKNotInstalledError(provider, sdk_name)`

Bases: `DeploymentError`

Exception raised when required cloud SDK is not installed.

Raised when deployment to a cloud provider fails because the required CLI/SDK is not available on the system.

Attributes:

| Name        | Type | Description                                             |
| ----------- | ---- | ------------------------------------------------------- |
| `provider`  |      | The cloud provider (aws, gcp, azure)                    |
| `sdk_name`  |      | The name of the missing SDK                             |
| `operation` |      | The operation that required the SDK                     |
| `message`   |      | Human-readable error message with installation guidance |

Initialize CloudSDKNotInstalledError with provider details.

Parameters:

| Name       | Type  | Description                          | Default    |
| ---------- | ----- | ------------------------------------ | ---------- |
| `provider` | `str` | The cloud provider (aws, gcp, azure) | *required* |
| `sdk_name` | `str` | The name of the missing SDK          | *required* |

Source code in `src/holodeck/lib/errors.py`

```
def __init__(self, provider: str, sdk_name: str) -> None:
    """Initialize CloudSDKNotInstalledError with provider details.

    Args:
        provider: The cloud provider (aws, gcp, azure)
        sdk_name: The name of the missing SDK
    """
    self.provider = provider
    self.sdk_name = sdk_name

    install_instructions = {
        "aws": "pip install 'holodeck-ai[deploy-aws]'",
        "gcp": "pip install 'holodeck-ai[deploy-gcp]'",
        "azure": "pip install 'holodeck-ai[deploy-azure]'",
    }

    install_cmd = install_instructions.get(provider, "Check provider docs")
    message = (
        f"Cloud SDK '{sdk_name}' is not installed for provider '{provider}'.\n"
        f"Install it with: {install_cmd}"
    )
    super().__init__("deploy", message)
```

### Backend Errors

Backend-specific errors (`BackendError`, `BackendInitError`, `BackendSessionError`, `BackendTimeoutError`) are documented in the [Backends API Reference](https://docs.useholodeck.ai/api/backends/#exceptions).

______________________________________________________________________

## Related Documentation

- [Configuration Loading](https://docs.useholodeck.ai/api/config-loader/index.md): How to load and validate configurations
- [Test Runner](https://docs.useholodeck.ai/api/test-runner/index.md): Test execution framework using these models
- [Evaluation Framework](https://docs.useholodeck.ai/api/evaluators/index.md): Evaluation system using these models

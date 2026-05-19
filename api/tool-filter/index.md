# Tool Filter API Reference

The tool filter subsystem implements [Anthropic's Tool Search pattern](https://docs.anthropic.com/en/docs/build-with-claude/tool-use#tool-search) for reducing token usage by dynamically filtering tools per request based on semantic similarity to the user's query. Instead of sending every registered tool to the LLM on each call, only the most relevant tools are included.

The subsystem lives in `holodeck.lib.tool_filter` and exposes four public symbols: `ToolMetadata`, `ToolFilterConfig`, `ToolIndex`, and `ToolFilterManager`.

______________________________________________________________________

## Configuration

`ToolFilterConfig` is the Pydantic model that controls filtering behavior. It is typically embedded in an agent's YAML configuration.

```
tool_filter:
  enabled: true
  top_k: 5
  similarity_threshold: 0.3
  search_method: hybrid        # semantic | bm25 | hybrid
  always_include:
    - get_user_context
  always_include_top_n_used: 3
```

## `ToolFilterConfig`

Bases: `BaseModel`

Configuration for automatic tool filtering.

Defines how tools are filtered per request to reduce token usage. When enabled, only the most relevant tools (based on semantic similarity to the user query) are included in each LLM call.

Attributes:

| Name                        | Type                                    | Description                                              |
| --------------------------- | --------------------------------------- | -------------------------------------------------------- |
| `enabled`                   | `bool`                                  | Enable or disable tool filtering globally.               |
| `top_k`                     | `int`                                   | Maximum number of tools to include per request.          |
| `similarity_threshold`      | `float`                                 | Minimum similarity score for tool inclusion.             |
| `always_include`            | `list[str]`                             | Tool names that are always included regardless of score. |
| `always_include_top_n_used` | `int`                                   | Number of most-used tools to always include.             |
| `search_method`             | `Literal['semantic', 'bm25', 'hybrid']` | Method for tool search (semantic, bm25, or hybrid).      |

### `validate_always_include(v)`

Validate always_include entries are non-empty strings.

Source code in `src/holodeck/lib/tool_filter/models.py`

```
@field_validator("always_include")
@classmethod
def validate_always_include(cls, v: list[str]) -> list[str]:
    """Validate always_include entries are non-empty strings."""
    for tool_name in v:
        if not tool_name or not tool_name.strip():
            raise ValueError("always_include entries must be non-empty strings")
    return v
```

______________________________________________________________________

## Tool Metadata

`ToolMetadata` represents a single tool inside the index. It is created automatically by `ToolIndex.build_from_kernel` and carries the embedding vector (when available), parameter descriptions, and runtime usage counts.

## `ToolMetadata`

Bases: `BaseModel`

Metadata for a single tool used in semantic search and filtering.

Stores information about tools extracted from the Semantic Kernel, including embeddings for semantic search and usage statistics for adaptive optimization.

Attributes:

| Name            | Type          | Description                                             |
| --------------- | ------------- | ------------------------------------------------------- |
| `name`          | `str`         | Tool function name (e.g., "search", "get_user").        |
| `plugin_name`   | `str`         | Plugin namespace (e.g., "vectorstore", "mcp_weather").  |
| `full_name`     | `str`         | Combined identifier as "plugin_name-function_name".     |
| `description`   | `str`         | Human-readable description for semantic search.         |
| `parameters`    | `list[str]`   | List of parameter descriptions for enhanced matching.   |
| `defer_loading` | `bool`        | If True, exclude from initial context (load on-demand). |
| `embedding`     | \`list[float] | None\`                                                  |
| `usage_count`   | `int`         | Number of times this tool has been invoked.             |

### `validate_description(v)`

Validate description is not empty.

Source code in `src/holodeck/lib/tool_filter/models.py`

```
@field_validator("description")
@classmethod
def validate_description(cls, v: str) -> str:
    """Validate description is not empty."""
    if not v or not v.strip():
        raise ValueError("description must be a non-empty string")
    return v
```

### `validate_full_name(v)`

Validate full_name is not empty.

Source code in `src/holodeck/lib/tool_filter/models.py`

```
@field_validator("full_name")
@classmethod
def validate_full_name(cls, v: str) -> str:
    """Validate full_name is not empty."""
    if not v or not v.strip():
        raise ValueError("full_name must be a non-empty string")
    return v
```

### `validate_name(v)`

Validate name is not empty.

Source code in `src/holodeck/lib/tool_filter/models.py`

```
@field_validator("name")
@classmethod
def validate_name(cls, v: str) -> str:
    """Validate name is not empty."""
    if not v or not v.strip():
        raise ValueError("name must be a non-empty string")
    return v
```

### `validate_usage_count(v)`

Validate usage_count is non-negative.

Source code in `src/holodeck/lib/tool_filter/models.py`

```
@field_validator("usage_count")
@classmethod
def validate_usage_count(cls, v: int) -> int:
    """Validate usage_count is non-negative."""
    if v < 0:
        raise ValueError("usage_count must be non-negative")
    return v
```

______________________________________________________________________

## Tool Index

`ToolIndex` is the in-memory search index that holds all `ToolMetadata` entries and supports three search strategies:

| Method     | Description                                           |
| ---------- | ----------------------------------------------------- |
| `semantic` | Cosine similarity over embedding vectors              |
| `bm25`     | Classic BM25 keyword scoring (no embeddings required) |
| `hybrid`   | Reciprocal Rank Fusion of semantic and BM25 results   |

When the embedding service is unavailable, semantic search automatically falls back to BM25.

## `ToolIndex()`

In-memory index for fast tool searching.

Maintains a collection of ToolMetadata objects and supports multiple search methods for finding relevant tools based on user queries.

Attributes:

| Name              | Type                      | Description                                   |
| ----------------- | ------------------------- | --------------------------------------------- |
| `tools`           | `dict[str, ToolMetadata]` | Dictionary mapping full_name to ToolMetadata. |
| `_idf_cache`      | `dict[str, float]`        | Cached IDF values for BM25 search.            |
| `_doc_lengths`    | `dict[str, int]`          | Document lengths for BM25 normalization.      |
| `_avg_doc_length` | `float`                   | Average document length for BM25.             |

Initialize an empty tool index.

Source code in `src/holodeck/lib/tool_filter/index.py`

```
def __init__(self) -> None:
    """Initialize an empty tool index."""
    self.tools: dict[str, ToolMetadata] = {}
    self._idf_cache: dict[str, float] = {}
    self._doc_lengths: dict[str, int] = {}
    self._avg_doc_length: float = 0.0
```

### `build_from_kernel(kernel, embedding_service=None, defer_loading_map=None)`

Build index from Semantic Kernel plugins.

Extracts all registered functions from the kernel's plugins and creates ToolMetadata entries with optional embeddings.

Parameters:

| Name                | Type                     | Description                              | Default                                                                                                |
| ------------------- | ------------------------ | ---------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `kernel`            | `Kernel`                 | Semantic Kernel with registered plugins. | *required*                                                                                             |
| `embedding_service` | \`EmbeddingGeneratorBase | None\`                                   | Optional TextEmbedding service for generating embeddings.                                              |
| `defer_loading_map` | \`dict[str, bool]        | None\`                                   | Optional mapping of tool names to defer_loading flags. Defaults to True for all tools if not provided. |

Source code in `src/holodeck/lib/tool_filter/index.py`

```
async def build_from_kernel(
    self,
    kernel: Kernel,
    embedding_service: EmbeddingGeneratorBase | None = None,
    defer_loading_map: dict[str, bool] | None = None,
) -> None:
    """Build index from Semantic Kernel plugins.

    Extracts all registered functions from the kernel's plugins
    and creates ToolMetadata entries with optional embeddings.

    Args:
        kernel: Semantic Kernel with registered plugins.
        embedding_service: Optional TextEmbedding service for generating embeddings.
        defer_loading_map: Optional mapping of tool names to defer_loading flags.
                           Defaults to True for all tools if not provided.
    """
    defer_loading_map = defer_loading_map or {}
    documents_for_bm25: list[tuple[str, str]] = []

    # Get all plugins and their functions
    plugins: dict[str, KernelPlugin] = getattr(kernel, "plugins", {})
    if not plugins:
        logger.debug("No plugins found in kernel")
        return

    for plugin_name, plugin in plugins.items():
        functions: dict[str, KernelFunction] = getattr(plugin, "functions", {})
        for func_name, func in functions.items():
            # Build full name
            full_name = f"{plugin_name}-{func_name}" if plugin_name else func_name

            # Extract description
            description = getattr(func, "description", "") or ""
            if not description:
                # Try to get from metadata
                metadata = getattr(func, "metadata", None)
                if metadata:
                    description = getattr(metadata, "description", "") or ""

            if not description:
                description = f"Function {func_name} from plugin {plugin_name}"

            # Extract parameter descriptions
            parameters: list[str] = []
            try:
                func_params: list[KernelParameterMetadata] | None = getattr(
                    func, "parameters", None
                )
                if func_params:
                    for param in func_params:
                        if param.description:
                            parameters.append(f"{param.name}: {param.description}")
                        elif param.name:
                            parameters.append(param.name)
            except Exception as e:
                logger.debug(f"Could not extract parameters for {full_name}: {e}")

            # Determine defer_loading
            defer_loading = defer_loading_map.get(full_name, True)

            # Create metadata
            tool_metadata = ToolMetadata(
                name=func_name,
                plugin_name=plugin_name,
                full_name=full_name,
                description=description,
                parameters=parameters,
                defer_loading=defer_loading,
            )

            self.tools[full_name] = tool_metadata

            # Collect for BM25
            doc_text = self._create_searchable_text(tool_metadata)
            documents_for_bm25.append((full_name, doc_text))

            logger.debug(
                f"Indexed tool: {full_name} | "
                f"searchable_text: {doc_text[:200]}..."
            )

    # Build BM25 index
    self._build_bm25_index(documents_for_bm25)

    # Generate embeddings if service provided
    if embedding_service and self.tools:
        await self._generate_embeddings(embedding_service)

    logger.info(f"Built tool index with {len(self.tools)} tools")
```

### `get_all_tool_names()`

Get all tool full names in the index.

Returns:

| Type        | Description                  |
| ----------- | ---------------------------- |
| `list[str]` | List of all tool full names. |

Source code in `src/holodeck/lib/tool_filter/index.py`

```
def get_all_tool_names(self) -> list[str]:
    """Get all tool full names in the index.

    Returns:
        List of all tool full names.
    """
    return list(self.tools.keys())
```

### `get_tool(full_name)`

Get a tool by its full name.

Parameters:

| Name        | Type  | Description                                   | Default    |
| ----------- | ----- | --------------------------------------------- | ---------- |
| `full_name` | `str` | Tool's full name (plugin_name-function_name). | *required* |

Returns:

| Type           | Description |
| -------------- | ----------- |
| \`ToolMetadata | None\`      |

Source code in `src/holodeck/lib/tool_filter/index.py`

```
def get_tool(self, full_name: str) -> ToolMetadata | None:
    """Get a tool by its full name.

    Args:
        full_name: Tool's full name (plugin_name-function_name).

    Returns:
        ToolMetadata if found, None otherwise.
    """
    return self.tools.get(full_name)
```

### `get_top_n_used(n)`

Get the N most frequently used tools.

Parameters:

| Name | Type  | Description                    | Default    |
| ---- | ----- | ------------------------------ | ---------- |
| `n`  | `int` | Number of top tools to return. | *required* |

Returns:

| Type                 | Description                                            |
| -------------------- | ------------------------------------------------------ |
| `list[ToolMetadata]` | List of ToolMetadata sorted by usage_count descending. |

Source code in `src/holodeck/lib/tool_filter/index.py`

```
def get_top_n_used(self, n: int) -> list[ToolMetadata]:
    """Get the N most frequently used tools.

    Args:
        n: Number of top tools to return.

    Returns:
        List of ToolMetadata sorted by usage_count descending.
    """
    if n <= 0:
        return []

    sorted_tools = sorted(
        self.tools.values(), key=lambda t: t.usage_count, reverse=True
    )
    return sorted_tools[:n]
```

### `search(query, top_k, method='semantic', threshold=0.0, embedding_service=None)`

Search for relevant tools based on query.

Parameters:

| Name                | Type                     | Description                                | Default                                               |
| ------------------- | ------------------------ | ------------------------------------------ | ----------------------------------------------------- |
| `query`             | `str`                    | User query to match against tools.         | *required*                                            |
| `top_k`             | `int`                    | Maximum number of results to return.       | *required*                                            |
| `method`            | `str`                    | Search method (semantic, bm25, or hybrid). | `'semantic'`                                          |
| `threshold`         | `float`                  | Minimum score threshold for inclusion.     | `0.0`                                                 |
| `embedding_service` | \`EmbeddingGeneratorBase | None\`                                     | TextEmbedding service (required for semantic search). |

Returns:

| Type                               | Description                                               |
| ---------------------------------- | --------------------------------------------------------- |
| `list[tuple[ToolMetadata, float]]` | List of (ToolMetadata, score) tuples sorted by relevance. |

Source code in `src/holodeck/lib/tool_filter/index.py`

```
async def search(
    self,
    query: str,
    top_k: int,
    method: str = "semantic",
    threshold: float = 0.0,
    embedding_service: EmbeddingGeneratorBase | None = None,
) -> list[tuple[ToolMetadata, float]]:
    """Search for relevant tools based on query.

    Args:
        query: User query to match against tools.
        top_k: Maximum number of results to return.
        method: Search method (semantic, bm25, or hybrid).
        threshold: Minimum score threshold for inclusion.
        embedding_service: TextEmbedding service (required for semantic search).

    Returns:
        List of (ToolMetadata, score) tuples sorted by relevance.
    """
    if not self.tools:
        return []

    if method == "semantic":
        results = await self._semantic_search(query, embedding_service)
    elif method == "bm25":
        results = self._bm25_search(query)
    elif method == "hybrid":
        results = await self._hybrid_search(query, embedding_service)
    else:
        logger.warning(f"Unknown search method: {method}, falling back to semantic")
        results = await self._semantic_search(query, embedding_service)

    # Sort all results by score descending
    results.sort(key=lambda x: x[1], reverse=True)

    # Log ALL tool scores for debugging (helps diagnose ranking issues)
    logger.debug(
        f"Tool search ({method}) all scores: "
        f"{[(t.full_name, f'{s:.4f}') for t, s in results]}"
    )

    # Filter by threshold
    filtered = [(tool, score) for tool, score in results if score >= threshold]

    top_results = filtered[:top_k]

    # Log top matches for visibility
    if top_results:
        top_matches = [(t.full_name, f"{s:.3f}") for t, s in top_results[:3]]
        logger.info(f"Tool search ({method}): top matches {top_matches}")

    return top_results
```

### `update_usage(tool_name)`

Increment usage count for a tool.

Parameters:

| Name        | Type  | Description                          | Default    |
| ----------- | ----- | ------------------------------------ | ---------- |
| `tool_name` | `str` | Full name of the tool that was used. | *required* |

Source code in `src/holodeck/lib/tool_filter/index.py`

```
def update_usage(self, tool_name: str) -> None:
    """Increment usage count for a tool.

    Args:
        tool_name: Full name of the tool that was used.
    """
    if tool_name in self.tools:
        self.tools[tool_name].usage_count += 1
        logger.debug(
            f"Updated usage for {tool_name}: {self.tools[tool_name].usage_count}"
        )
```

______________________________________________________________________

## Tool Filter Manager

`ToolFilterManager` is the main orchestrator. It wires together the `ToolIndex`, the embedding service, and Semantic Kernel's `FunctionChoiceBehavior` to transparently reduce the tool set on every agent invocation.

### Typical lifecycle

```
from holodeck.lib.tool_filter import ToolFilterConfig, ToolFilterManager

config = ToolFilterConfig(
    enabled=True,
    top_k=5,
    similarity_threshold=0.3,
    search_method="hybrid",
)

manager = ToolFilterManager(config, kernel, embedding_service)
await manager.initialize()

# Per-request filtering
filtered_tool_names = await manager.filter_tools("What's the weather?")

# Or apply directly to execution settings
settings = await manager.prepare_execution_settings(query, base_settings)

# After execution, record which tools the model actually called
manager.record_tool_usage(result.tool_calls)
```

## `ToolFilterManager(config, kernel, embedding_service=None)`

Manages tool filtering for agent invocations.

Coordinates between the ToolIndex (for semantic search) and Semantic Kernel's FunctionChoiceBehavior (for tool filtering) to reduce token usage by only including relevant tools.

Attributes:

| Name                | Type | Description                                 |
| ------------------- | ---- | ------------------------------------------- |
| `config`            |      | ToolFilterConfig with filtering parameters. |
| `kernel`            |      | Semantic Kernel with registered plugins.    |
| `embedding_service` |      | TextEmbedding service for semantic search.  |
| `index`             |      | ToolIndex for fast tool searching.          |
| `_initialized`      |      | Whether the manager has been initialized.   |

Initialize the tool filter manager.

Parameters:

| Name                | Type                     | Description                              | Default                                             |
| ------------------- | ------------------------ | ---------------------------------------- | --------------------------------------------------- |
| `config`            | `ToolFilterConfig`       | Tool filtering configuration.            | *required*                                          |
| `kernel`            | `Kernel`                 | Semantic Kernel with registered plugins. | *required*                                          |
| `embedding_service` | \`EmbeddingGeneratorBase | None\`                                   | Optional TextEmbedding service for semantic search. |

Source code in `src/holodeck/lib/tool_filter/manager.py`

```
def __init__(
    self,
    config: ToolFilterConfig,
    kernel: Kernel,
    embedding_service: EmbeddingGeneratorBase | None = None,
) -> None:
    """Initialize the tool filter manager.

    Args:
        config: Tool filtering configuration.
        kernel: Semantic Kernel with registered plugins.
        embedding_service: Optional TextEmbedding service for semantic search.
    """
    self.config = config
    self.kernel = kernel
    self.embedding_service = embedding_service
    self.index = ToolIndex()
    self._initialized = False

    logger.debug(
        f"ToolFilterManager created: enabled={config.enabled}, "
        f"top_k={config.top_k}, method={config.search_method}"
    )
```

### `create_function_choice_behavior(filtered_tools)`

Create FunctionChoiceBehavior with filtered tool list.

Uses Semantic Kernel's native filtering mechanism to restrict which functions are available to the LLM.

Parameters:

| Name             | Type        | Description                         | Default    |
| ---------------- | ----------- | ----------------------------------- | ---------- |
| `filtered_tools` | `list[str]` | List of tool full_names to include. | *required* |

Returns:

| Type                     | Description                                                    |
| ------------------------ | -------------------------------------------------------------- |
| `FunctionChoiceBehavior` | FunctionChoiceBehavior configured with the filtered tool list. |

Source code in `src/holodeck/lib/tool_filter/manager.py`

```
def create_function_choice_behavior(
    self, filtered_tools: list[str]
) -> FunctionChoiceBehavior:
    """Create FunctionChoiceBehavior with filtered tool list.

    Uses Semantic Kernel's native filtering mechanism to restrict
    which functions are available to the LLM.

    Args:
        filtered_tools: List of tool full_names to include.

    Returns:
        FunctionChoiceBehavior configured with the filtered tool list.
    """
    return FunctionChoiceBehavior.Auto(
        filters={"included_functions": filtered_tools}
    )
```

### `filter_tools(query)`

Filter tools based on query relevance.

Returns a list of tool names that should be included in the LLM call based on semantic similarity to the query.

Parameters:

| Name    | Type  | Description               | Default    |
| ------- | ----- | ------------------------- | ---------- |
| `query` | `str` | User query for filtering. | *required* |

Returns:

| Type        | Description                                        |
| ----------- | -------------------------------------------------- |
| `list[str]` | List of tool full_names to include in the request. |

Source code in `src/holodeck/lib/tool_filter/manager.py`

```
async def filter_tools(self, query: str) -> list[str]:
    """Filter tools based on query relevance.

    Returns a list of tool names that should be included in
    the LLM call based on semantic similarity to the query.

    Args:
        query: User query for filtering.

    Returns:
        List of tool full_names to include in the request.
    """
    if not self._initialized:
        logger.warning("ToolFilterManager not initialized, returning all tools")
        return self.index.get_all_tool_names()

    # Start with always_include tools
    included_tools: set[str] = set()

    # Add always_include tools
    for tool_name in self.config.always_include:
        # Match against full_name or just function name
        for full_name in self.index.get_all_tool_names():
            if tool_name == full_name or full_name.endswith(f"-{tool_name}"):
                included_tools.add(full_name)
                break

    # Add top-N most used tools
    if self.config.always_include_top_n_used > 0:
        top_used = self.index.get_top_n_used(self.config.always_include_top_n_used)
        for tool in top_used:
            included_tools.add(tool.full_name)

    # Search for relevant tools
    remaining_slots = max(0, self.config.top_k - len(included_tools))

    if remaining_slots > 0:
        search_results = await self.index.search(
            query=query,
            top_k=remaining_slots + len(included_tools),  # Over-fetch to filter
            method=self.config.search_method,
            threshold=self.config.similarity_threshold,
            embedding_service=self.embedding_service,
        )

        for tool, score in search_results:
            if len(included_tools) >= self.config.top_k:
                break
            # Skip if already included
            if tool.full_name in included_tools:
                continue
            # Skip deferred tools if below threshold
            if tool.defer_loading and score < self.config.similarity_threshold:
                continue
            included_tools.add(tool.full_name)
            logger.debug(f"Included tool {tool.full_name} (score={score:.3f})")

    logger.info(
        f"Tool filtering: {len(included_tools)}/{len(self.index.tools)} tools "
        f"selected for query: '{query[:50]}...'"
    )
    logger.info(f"Selected tools: {sorted(included_tools)}")

    return list(included_tools)
```

### `get_filter_stats()`

Get statistics about tool filtering.

Returns:

| Type             | Description                           |
| ---------------- | ------------------------------------- |
| `dict[str, Any]` | Dictionary with filtering statistics. |

Source code in `src/holodeck/lib/tool_filter/manager.py`

```
def get_filter_stats(self) -> dict[str, Any]:
    """Get statistics about tool filtering.

    Returns:
        Dictionary with filtering statistics.
    """
    return {
        "enabled": self.config.enabled,
        "total_tools": len(self.index.tools),
        "top_k": self.config.top_k,
        "similarity_threshold": self.config.similarity_threshold,
        "search_method": self.config.search_method,
        "always_include": self.config.always_include,
        "always_include_top_n_used": self.config.always_include_top_n_used,
    }
```

### `initialize(defer_loading_map=None)`

Initialize the tool index from kernel plugins.

Must be called after all tools are registered on the kernel and before any filtering operations.

Parameters:

| Name                | Type              | Description | Default                                                |
| ------------------- | ----------------- | ----------- | ------------------------------------------------------ |
| `defer_loading_map` | \`dict[str, bool] | None\`      | Optional mapping of tool names to defer_loading flags. |

Source code in `src/holodeck/lib/tool_filter/manager.py`

```
async def initialize(
    self,
    defer_loading_map: dict[str, bool] | None = None,
) -> None:
    """Initialize the tool index from kernel plugins.

    Must be called after all tools are registered on the kernel
    and before any filtering operations.

    Args:
        defer_loading_map: Optional mapping of tool names to defer_loading flags.
    """
    if self._initialized:
        logger.debug("ToolFilterManager already initialized")
        return

    logger.debug("Initializing ToolFilterManager index")

    await self.index.build_from_kernel(
        kernel=self.kernel,
        embedding_service=self.embedding_service,
        defer_loading_map=defer_loading_map,
    )

    self._initialized = True
    logger.info(f"ToolFilterManager initialized with {len(self.index.tools)} tools")
```

### `prepare_execution_settings(query, base_settings)`

Prepare execution settings with filtered tools.

Filters tools based on the query and creates new execution settings with the appropriate FunctionChoiceBehavior.

Parameters:

| Name            | Type                      | Description                          | Default                            |
| --------------- | ------------------------- | ------------------------------------ | ---------------------------------- |
| `query`         | `str`                     | User query for filtering.            | *required*                         |
| `base_settings` | \`PromptExecutionSettings | dict[str, PromptExecutionSettings]\` | Base execution settings to modify. |

Returns:

| Type                      | Description                          |
| ------------------------- | ------------------------------------ |
| \`PromptExecutionSettings | dict[str, PromptExecutionSettings]\` |

Source code in `src/holodeck/lib/tool_filter/manager.py`

```
async def prepare_execution_settings(
    self,
    query: str,
    base_settings: PromptExecutionSettings | dict[str, PromptExecutionSettings],
) -> PromptExecutionSettings | dict[str, PromptExecutionSettings]:
    """Prepare execution settings with filtered tools.

    Filters tools based on the query and creates new execution
    settings with the appropriate FunctionChoiceBehavior.

    Args:
        query: User query for filtering.
        base_settings: Base execution settings to modify.

    Returns:
        Modified execution settings with filtered function choice behavior.
    """
    if not self.config.enabled:
        return base_settings

    # Filter tools
    filtered_tools = await self.filter_tools(query)

    # Create function choice behavior
    function_choice = self.create_function_choice_behavior(filtered_tools)

    # Handle both single settings and dict of settings
    if isinstance(base_settings, dict):
        # Clone and modify each settings object
        modified_settings: dict[str, PromptExecutionSettings] = {}
        for key, settings in base_settings.items():
            cloned = self._clone_settings(settings)
            if hasattr(cloned, "function_choice_behavior"):
                cloned.function_choice_behavior = function_choice
            modified_settings[key] = cloned
        return modified_settings
    else:
        # Single settings object
        cloned = self._clone_settings(base_settings)
        if hasattr(cloned, "function_choice_behavior"):
            cloned.function_choice_behavior = function_choice
        return cloned
```

### `record_tool_usage(tool_calls)`

Record tool usage for adaptive optimization.

Updates usage counts in the index based on which tools were actually called during agent execution.

Parameters:

| Name         | Type                   | Description                              | Default    |
| ------------ | ---------------------- | ---------------------------------------- | ---------- |
| `tool_calls` | `list[dict[str, Any]]` | List of tool call dicts with 'name' key. | *required* |

Source code in `src/holodeck/lib/tool_filter/manager.py`

```
def record_tool_usage(self, tool_calls: list[dict[str, Any]]) -> None:
    """Record tool usage for adaptive optimization.

    Updates usage counts in the index based on which tools
    were actually called during agent execution.

    Args:
        tool_calls: List of tool call dicts with 'name' key.
    """
    for call in tool_calls:
        tool_name = call.get("name", "")
        if tool_name:
            self.index.update_usage(tool_name)
```

______________________________________________________________________

## Module-Level Helpers

The `index` module also exposes two private helper functions used internally by `ToolIndex`. They are not part of the public API but are documented here for completeness.

## `_cosine_similarity(vec_a, vec_b)`

Compute cosine similarity between two vectors.

Parameters:

| Name    | Type          | Description              | Default    |
| ------- | ------------- | ------------------------ | ---------- |
| `vec_a` | `list[float]` | First embedding vector.  | *required* |
| `vec_b` | `list[float]` | Second embedding vector. | *required* |

Returns:

| Type    | Description                                   |
| ------- | --------------------------------------------- |
| `float` | Cosine similarity score between -1.0 and 1.0. |

Source code in `src/holodeck/lib/tool_filter/index.py`

```
def _cosine_similarity(vec_a: list[float], vec_b: list[float]) -> float:
    """Compute cosine similarity between two vectors.

    Args:
        vec_a: First embedding vector.
        vec_b: Second embedding vector.

    Returns:
        Cosine similarity score between -1.0 and 1.0.
    """
    if len(vec_a) != len(vec_b):
        return 0.0

    dot_product = sum(a * b for a, b in zip(vec_a, vec_b, strict=False))
    norm_a = math.sqrt(sum(a * a for a in vec_a))
    norm_b = math.sqrt(sum(b * b for b in vec_b))

    if norm_a == 0.0 or norm_b == 0.0:
        return 0.0

    return dot_product / (norm_a * norm_b)
```

## `_tokenize(text)`

Simple tokenizer for BM25 search.

Splits text on non-alphanumeric characters INCLUDING underscores, so that tool names like "brave_web_search" become ["brave", "web", "search"]. This enables matching individual terms like "web" against tool names.

Parameters:

| Name   | Type  | Description             | Default    |
| ------ | ----- | ----------------------- | ---------- |
| `text` | `str` | Input text to tokenize. | *required* |

Returns:

| Type        | Description               |
| ----------- | ------------------------- |
| `list[str]` | List of lowercase tokens. |

Source code in `src/holodeck/lib/tool_filter/index.py`

```
def _tokenize(text: str) -> list[str]:
    """Simple tokenizer for BM25 search.

    Splits text on non-alphanumeric characters INCLUDING underscores,
    so that tool names like "brave_web_search" become ["brave", "web", "search"].
    This enables matching individual terms like "web" against tool names.

    Args:
        text: Input text to tokenize.

    Returns:
        List of lowercase tokens.
    """
    # Split on non-alphanumeric characters (excluding underscores from word chars)
    # This ensures "brave_web_search" -> ["brave", "web", "search"]
    tokens = re.findall(r"[a-zA-Z0-9]+", text.lower())
    return tokens
```

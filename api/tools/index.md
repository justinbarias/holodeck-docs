# Tools

The `holodeck.tools` package provides tool implementations that extend agent capabilities with semantic search, hierarchical document retrieval, and MCP server integration.

## Module Overview

| Module                                      | Description                                            |
| ------------------------------------------- | ------------------------------------------------------ |
| `holodeck.tools`                            | Package exports for tools, mixins, and utilities       |
| `holodeck.tools.base_tool`                  | Mixin classes for embedding and database configuration |
| `holodeck.tools.common`                     | Shared constants and pure utility functions            |
| `holodeck.tools.vectorstore_tool`           | Semantic search over unstructured and structured data  |
| `holodeck.tools.hierarchical_document_tool` | Structure-aware document retrieval with hybrid search  |
| `holodeck.tools.mcp`                        | MCP server integration (factory, utilities, errors)    |

______________________________________________________________________

## `holodeck.tools.base_tool`

Mixin classes that encapsulate common functionality shared between `VectorStoreTool` and `HierarchicalDocumentTool`. Mixins are used instead of inheritance because the tools have fundamentally different record types and core behaviors.

### `EmbeddingServiceMixin`

## `EmbeddingServiceMixin`

Mixin for embedding service injection.

Provides the set_embedding_service method used by AgentFactory to inject a Semantic Kernel TextEmbedding service for generating real embeddings.

Required instance attributes (set by subclass **init**): \_embedding_service: Any - stores the injected service config: Any - tool configuration with a .name attribute

### `set_embedding_service(service)`

Set the embedding service for generating embeddings.

This method allows AgentFactory to inject a Semantic Kernel TextEmbedding service for generating real embeddings instead of placeholder zeros.

Parameters:

| Name      | Type  | Description                                                                                 | Default    |
| --------- | ----- | ------------------------------------------------------------------------------------------- | ---------- |
| `service` | `Any` | Semantic Kernel TextEmbedding service instance (OpenAITextEmbedding or AzureTextEmbedding). | *required* |

Source code in `src/holodeck/tools/base_tool.py`

```
def set_embedding_service(self, service: Any) -> None:
    """Set the embedding service for generating embeddings.

    This method allows AgentFactory to inject a Semantic Kernel TextEmbedding
    service for generating real embeddings instead of placeholder zeros.

    Args:
        service: Semantic Kernel TextEmbedding service instance
            (OpenAITextEmbedding or AzureTextEmbedding).
    """
    self._embedding_service = service
    tool_name = getattr(getattr(self, "config", None), "name", "unknown")
    logger.debug(f"Embedding service set for tool: {tool_name}")
```

### `DatabaseConfigMixin`

## `DatabaseConfigMixin`

Mixin for database configuration resolution and collection creation.

Provides methods for resolving database configuration from various formats (None, string reference, DatabaseConfig object) and creating vector store collections with automatic fallback to in-memory storage.

Required instance attributes (set by subclass **init**): config: Any - tool configuration with .name and .database attributes \_provider: str - stores the resolved provider name \_collection: Any - stores the created collection instance \_embedding_dimensions: int | None - embedding dimensions

### `_resolve_database_config(database)`

Resolve database configuration to provider and connection kwargs.

Handles three types of database configuration:

1. None - use in-memory storage
1. String reference - unresolved reference, warn and use in-memory
1. DatabaseConfig object - extract provider and connection parameters

Parameters:

| Name       | Type             | Description | Default |
| ---------- | ---------------- | ----------- | ------- |
| `database` | \`DatabaseConfig | str         | None\`  |

Returns:

| Type                         | Description                                 |
| ---------------------------- | ------------------------------------------- |
| `tuple[str, dict[str, Any]]` | Tuple of (provider_name, connection_kwargs) |

Example

> > > provider, kwargs = self.\_resolve_database_config(None) provider 'in-memory' kwargs {}

Source code in `src/holodeck/tools/base_tool.py`

```
def _resolve_database_config(
    self, database: DatabaseConfig | str | None
) -> tuple[str, dict[str, Any]]:
    """Resolve database configuration to provider and connection kwargs.

    Handles three types of database configuration:
    1. None - use in-memory storage
    2. String reference - unresolved reference, warn and use in-memory
    3. DatabaseConfig object - extract provider and connection parameters

    Args:
        database: Database configuration (from tool config)

    Returns:
        Tuple of (provider_name, connection_kwargs)

    Example:
        >>> provider, kwargs = self._resolve_database_config(None)
        >>> provider
        'in-memory'
        >>> kwargs
        {}
    """
    tool_name = getattr(getattr(self, "config", None), "name", "unknown")

    if isinstance(database, str):
        # Unresolved string reference - this shouldn't happen if merge_configs
        # was called, but fall back to in-memory with a warning
        logger.warning(
            f"Tool '{tool_name}' has unresolved database "
            f"reference '{database}'. Falling back to in-memory storage."
        )
        return "in-memory", {}

    if database is not None:
        # DatabaseConfig object - use its settings
        provider = database.provider
        connection_kwargs: dict[str, Any] = {}
        if database.connection_string:
            connection_kwargs["connection_string"] = (
                database.connection_string.get_secret_value()
            )
        # Add extra fields from DatabaseConfig (extra="allow")
        if hasattr(database, "model_extra"):
            extra_fields = database.model_extra or {}
            connection_kwargs.update(extra_fields)
        return provider, connection_kwargs

    # None - use in-memory
    return "in-memory", {}
```

### `_create_collection_with_fallback(provider, dimensions, connection_kwargs, record_class=None, definition=None)`

Create a vector store collection with fallback to in-memory.

Attempts to create a collection with the specified provider. If creation fails (e.g., database unreachable), falls back to in-memory storage.

Parameters:

| Name                | Type             | Description                             | Default                                                      |
| ------------------- | ---------------- | --------------------------------------- | ------------------------------------------------------------ |
| `provider`          | `str`            | Vector store provider name              | *required*                                                   |
| `dimensions`        | `int`            | Embedding dimensions for the collection | *required*                                                   |
| `connection_kwargs` | `dict[str, Any]` | Provider-specific connection parameters | *required*                                                   |
| `record_class`      | \`type[Any]      | None\`                                  | Optional custom record class for the collection              |
| `definition`        | \`Any            | None\`                                  | Optional VectorStoreCollectionDefinition for structured data |

Returns:

| Type  | Description                 |
| ----- | --------------------------- |
| `Any` | Created collection instance |

Raises:

| Type        | Description                                         |
| ----------- | --------------------------------------------------- |
| `Exception` | If in-memory provider also fails (shouldn't happen) |

Source code in `src/holodeck/tools/base_tool.py`

```
def _create_collection_with_fallback(
    self,
    provider: str,
    dimensions: int,
    connection_kwargs: dict[str, Any],
    record_class: type[Any] | None = None,
    definition: Any | None = None,
) -> Any:
    """Create a vector store collection with fallback to in-memory.

    Attempts to create a collection with the specified provider. If creation
    fails (e.g., database unreachable), falls back to in-memory storage.

    Args:
        provider: Vector store provider name
        dimensions: Embedding dimensions for the collection
        connection_kwargs: Provider-specific connection parameters
        record_class: Optional custom record class for the collection
        definition: Optional VectorStoreCollectionDefinition for structured data

    Returns:
        Created collection instance

    Raises:
        Exception: If in-memory provider also fails (shouldn't happen)
    """
    from holodeck.lib.vector_store import get_collection_factory

    try:
        factory = get_collection_factory(
            provider=provider,
            dimensions=dimensions,
            record_class=record_class,
            definition=definition,
            **connection_kwargs,
        )
        collection = factory()
        logger.info(
            f"Vector store connected: provider={provider}, "
            f"dimensions={dimensions}"
        )
        self._provider = provider
        return collection

    except (ImportError, ConnectionError, Exception) as e:
        # Fall back to in-memory storage for non-in-memory providers
        if provider != "in-memory":
            logger.warning(
                f"Failed to connect to {provider}: {e}. "
                "Falling back to in-memory storage."
            )
            factory = get_collection_factory(
                provider="in-memory",
                dimensions=dimensions,
                record_class=record_class,
                definition=definition,
            )
            collection = factory()
            logger.info("Using in-memory vector storage (fallback)")
            self._provider = "in-memory"
            return collection
        else:
            # Don't catch errors for in-memory provider
            raise
```

______________________________________________________________________

## `holodeck.tools.common`

Shared constants and pure utility functions used by `VectorStoreTool`, `HierarchicalDocumentTool`, and other tool implementations. Follows the DRY principle for file handling, path resolution, and embedding generation.

### Constants

#### `SUPPORTED_EXTENSIONS`

## `SUPPORTED_EXTENSIONS = frozenset({'.txt', '.md', '.pdf', '.csv', '.json'})`

#### `FILE_TYPE_MAPPING`

## `FILE_TYPE_MAPPING = {'.txt': 'text', '.md': 'text', '.pdf': 'pdf', '.csv': 'csv', '.json': 'text'}`

### Functions

#### `get_file_type`

## `get_file_type(path)`

Get FileInput type from file extension.

Maps file extensions to the appropriate type value for FileProcessor. Defaults to "text" for unknown extensions.

Parameters:

| Name   | Type  | Description | Default                           |
| ------ | ----- | ----------- | --------------------------------- |
| `path` | \`str | Path\`      | File path (string or Path object) |

Returns:

| Type  | Description                                        |
| ----- | -------------------------------------------------- |
| `str` | FileInput type string ("text", "pdf", "csv", etc.) |

Example

> > > get_file_type("document.pdf") 'pdf' get_file_type(Path("data/file.csv")) 'csv' get_file_type("unknown.xyz") 'text'

Source code in `src/holodeck/tools/common.py`

```
def get_file_type(path: str | Path) -> str:
    """Get FileInput type from file extension.

    Maps file extensions to the appropriate type value for FileProcessor.
    Defaults to "text" for unknown extensions.

    Args:
        path: File path (string or Path object)

    Returns:
        FileInput type string ("text", "pdf", "csv", etc.)

    Example:
        >>> get_file_type("document.pdf")
        'pdf'
        >>> get_file_type(Path("data/file.csv"))
        'csv'
        >>> get_file_type("unknown.xyz")
        'text'
    """
    if isinstance(path, str):
        path = Path(path)
    extension = path.suffix.lower()
    return FILE_TYPE_MAPPING.get(extension, "text")
```

#### `resolve_source_path`

## `resolve_source_path(source, base_dir=None)`

Resolve a source path relative to a base directory.

This function handles path resolution in priority order:

1. If source is absolute, return as-is
1. If base_dir is provided, resolve relative to base_dir
1. Try agent_base_dir context variable
1. Fall back to current working directory

Parameters:

| Name       | Type  | Description                               | Default                                              |
| ---------- | ----- | ----------------------------------------- | ---------------------------------------------------- |
| `source`   | `str` | Source path to resolve (from tool config) | *required*                                           |
| `base_dir` | \`str | None\`                                    | Optional base directory for relative path resolution |

Returns:

| Type   | Description                          |
| ------ | ------------------------------------ |
| `Path` | Resolved absolute Path to the source |

Example

> > > resolve_source_path("/absolute/path/file.txt") PosixPath('/absolute/path/file.txt') resolve_source_path("relative/file.txt", "/base") PosixPath('/base/relative/file.txt')

Source code in `src/holodeck/tools/common.py`

```
def resolve_source_path(source: str, base_dir: str | None = None) -> Path:
    """Resolve a source path relative to a base directory.

    This function handles path resolution in priority order:
    1. If source is absolute, return as-is
    2. If base_dir is provided, resolve relative to base_dir
    3. Try agent_base_dir context variable
    4. Fall back to current working directory

    Args:
        source: Source path to resolve (from tool config)
        base_dir: Optional base directory for relative path resolution

    Returns:
        Resolved absolute Path to the source

    Example:
        >>> resolve_source_path("/absolute/path/file.txt")
        PosixPath('/absolute/path/file.txt')
        >>> resolve_source_path("relative/file.txt", "/base")
        PosixPath('/base/relative/file.txt')
    """
    source_path = Path(source)

    # If path is absolute, use it directly
    if source_path.is_absolute():
        return source_path

    # Resolve relative to base directory
    # Priority: explicit base_dir > context var > cwd
    effective_base = base_dir
    if effective_base is None:
        # Try to get from context variable
        from holodeck.config.context import agent_base_dir

        effective_base = agent_base_dir.get()

    if effective_base:
        return (Path(effective_base) / source).resolve()

    return source_path.resolve()
```

#### `discover_files`

## `discover_files(source_path, extensions=None)`

Discover files to ingest from a source path.

Recursively traverses directories and filters by supported extensions. For single files, validates the extension is supported.

Parameters:

| Name          | Type             | Description                                        | Default                                                     |
| ------------- | ---------------- | -------------------------------------------------- | ----------------------------------------------------------- |
| `source_path` | `Path`           | Resolved path (file or directory) to discover from | *required*                                                  |
| `extensions`  | \`frozenset[str] | None\`                                             | Set of supported extensions (default: SUPPORTED_EXTENSIONS) |

Returns:

| Type         | Description                                                               |
| ------------ | ------------------------------------------------------------------------- |
| `list[Path]` | List of Path objects for files to process, sorted for deterministic order |

Note

This function does not validate file existence - that should be checked before calling this function.

Example

> > > discover_files(Path("/docs")) [PosixPath('/docs/file1.md'), PosixPath('/docs/subdir/file2.txt')]

Source code in `src/holodeck/tools/common.py`

```
def discover_files(
    source_path: Path,
    extensions: frozenset[str] | None = None,
) -> list[Path]:
    """Discover files to ingest from a source path.

    Recursively traverses directories and filters by supported extensions.
    For single files, validates the extension is supported.

    Args:
        source_path: Resolved path (file or directory) to discover from
        extensions: Set of supported extensions (default: SUPPORTED_EXTENSIONS)

    Returns:
        List of Path objects for files to process, sorted for deterministic order

    Note:
        This function does not validate file existence - that should be
        checked before calling this function.

    Example:
        >>> discover_files(Path("/docs"))
        [PosixPath('/docs/file1.md'), PosixPath('/docs/subdir/file2.txt')]
    """
    if extensions is None:
        extensions = SUPPORTED_EXTENSIONS

    if source_path.is_file():
        # Single file - check if supported
        if source_path.suffix.lower() in extensions:
            return [source_path]
        logger.warning(
            f"File {source_path} has unsupported extension "
            f"{source_path.suffix}. Supported: {extensions}"
        )
        return []

    if source_path.is_dir():
        # Directory - recursively find all supported files
        discovered: list[Path] = []
        for file_path in source_path.rglob("*"):
            if file_path.is_file():
                if file_path.suffix.lower() in extensions:
                    discovered.append(file_path)
                else:
                    logger.debug(
                        f"Skipping unsupported file: {file_path} "
                        f"(extension: {file_path.suffix})"
                    )
        # Sort for deterministic ordering
        return sorted(discovered)

    # Path doesn't exist - return empty list
    return []
```

#### `generate_placeholder_embeddings`

## `generate_placeholder_embeddings(count, dimensions=1536)`

Generate placeholder embedding vectors for testing.

Creates zero-valued embedding vectors when no embedding service is available. Useful for testing and development without LLM API access.

Parameters:

| Name         | Type  | Description                                 | Default    |
| ------------ | ----- | ------------------------------------------- | ---------- |
| `count`      | `int` | Number of embeddings to generate            | *required* |
| `dimensions` | `int` | Embedding vector dimensions (default: 1536) | `1536`     |

Returns:

| Type                | Description                           |
| ------------------- | ------------------------------------- |
| `list[list[float]]` | List of zero-valued embedding vectors |

Example

> > > embeddings = generate_placeholder_embeddings(3, 768) len(embeddings) 3 len(embeddings[0]) 768

Source code in `src/holodeck/tools/common.py`

```
def generate_placeholder_embeddings(
    count: int,
    dimensions: int = 1536,
) -> list[list[float]]:
    """Generate placeholder embedding vectors for testing.

    Creates zero-valued embedding vectors when no embedding service is available.
    Useful for testing and development without LLM API access.

    Args:
        count: Number of embeddings to generate
        dimensions: Embedding vector dimensions (default: 1536)

    Returns:
        List of zero-valued embedding vectors

    Example:
        >>> embeddings = generate_placeholder_embeddings(3, 768)
        >>> len(embeddings)
        3
        >>> len(embeddings[0])
        768
    """
    logger.debug(f"Generated {count} placeholder embeddings, dim={dimensions}")
    return [[0.0] * dimensions for _ in range(count)]
```

______________________________________________________________________

## `holodeck.tools.vectorstore_tool`

Provides semantic search over files and directories containing text data or structured data (CSV, JSON, JSONL files with field mapping). Supports automatic file discovery, text chunking, embedding generation, vector storage, and modification-time tracking for incremental ingestion.

### `VectorStoreTool`

## `VectorStoreTool(config, base_dir=None, execution_config=None)`

Bases: `EmbeddingServiceMixin`, `DatabaseConfigMixin`

Vectorstore tool for semantic search over unstructured data.

This tool enables agents to perform semantic search over documents by:

1. Discovering files from configured source (file or directory)
1. Converting files to markdown using FileProcessor
1. Chunking text for optimal embedding generation
1. Generating embeddings via Semantic Kernel services
1. Storing document chunks in a vector database
1. Performing similarity search on queries

Inherits from

EmbeddingServiceMixin: Provides set_embedding_service() method DatabaseConfigMixin: Provides database config resolution and collection creation

Attributes:

| Name               | Type       | Description                           |
| ------------------ | ---------- | ------------------------------------- |
| `config`           |            | Tool configuration from agent.yaml    |
| `is_initialized`   | `bool`     | Whether the tool has been initialized |
| `document_count`   | `int`      | Number of document chunks stored      |
| `last_ingest_time` | \`datetime | None\`                                |

Example

> > > config = VectorstoreTool( ... name="knowledge_base", ... description="Search product docs", ... source="data/docs/" ... ) tool = VectorStoreTool(config) await tool.initialize() results = await tool.search("How do I authenticate?")

Initialize VectorStoreTool with configuration.

Parameters:

| Name               | Type              | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Default                                                                                                                       |
| ------------------ | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| `config`           | `VectorstoreTool` | VectorstoreTool configuration from agent.yaml containing: - name: Tool identifier - description: Tool description - source: File or directory path to ingest - embedding_model: Optional custom embedding model - database: Optional database configuration - top_k: Number of results to return (default: 5) - min_similarity_score: Minimum score threshold (optional) - chunk_size: Text chunk size in tokens (optional) - chunk_overlap: Chunk overlap in tokens (optional) | *required*                                                                                                                    |
| `base_dir`         | \`str             | None\`                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Base directory for resolving relative source paths. If None, source paths are resolved relative to current working directory. |
| `execution_config` | \`ExecutionConfig | None\`                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Execution configuration for file processing timeouts and caching. If None, default FileProcessor settings are used.           |

Source code in `src/holodeck/tools/vectorstore_tool.py`

```
def __init__(
    self,
    config: VectorstoreToolConfig,
    base_dir: str | None = None,
    execution_config: ExecutionConfig | None = None,
) -> None:
    """Initialize VectorStoreTool with configuration.

    Args:
        config: VectorstoreTool configuration from agent.yaml containing:
            - name: Tool identifier
            - description: Tool description
            - source: File or directory path to ingest
            - embedding_model: Optional custom embedding model
            - database: Optional database configuration
            - top_k: Number of results to return (default: 5)
            - min_similarity_score: Minimum score threshold (optional)
            - chunk_size: Text chunk size in tokens (optional)
            - chunk_overlap: Chunk overlap in tokens (optional)
        base_dir: Base directory for resolving relative source paths.
            If None, source paths are resolved relative to current
            working directory.
        execution_config: Execution configuration for file processing
            timeouts and caching. If None, default FileProcessor
            settings are used.
    """
    self.config = config
    self._base_dir = base_dir
    self._execution_config = execution_config

    # State tracking
    self.is_initialized: bool = False
    self.document_count: int = 0
    self.last_ingest_time: datetime | None = None

    # Embedding dimensions (resolved during initialization)
    self._embedding_dimensions: int | None = None

    # Initialize components (lazy initialization for some)
    chunk_size = config.chunk_size or TextChunker.DEFAULT_CHUNK_SIZE
    chunk_overlap = config.chunk_overlap or TextChunker.DEFAULT_CHUNK_OVERLAP
    self._text_chunker = TextChunker(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap,
    )
    self._file_processor: FileProcessor | None = None

    # Embedding service (initialized lazily by AgentFactory)
    self._embedding_service: Any = None

    # Persistent collection instance for vector store operations
    self._collection: Any = None
    self._provider: str = "in-memory"

    # Source context for stable record keys (remote sources)
    self._source_root: Path | None = None
    self._is_remote: bool = False

    logger.debug(
        f"VectorStoreTool initialized: name={config.name}, "
        f"source={config.source}, base_dir={base_dir}, top_k={config.top_k}"
    )
```

### `initialize(force_ingest=False, provider_type=None, progress_callback=None)`

Initialize tool and ingest source files.

Discovers files from the configured source, processes them into chunks, generates embeddings, and stores them in the vector database. Source path is resolved relative to base_dir if set.

For structured data mode (when vector_field is configured), loads structured data from CSV/JSON/JSONL files with field mapping.

Parameters:

| Name                | Type                   | Description                                                   | Default                                                                           |
| ------------------- | ---------------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `force_ingest`      | `bool`                 | If True, re-ingest all files regardless of modification time. | `False`                                                                           |
| `provider_type`     | \`str                  | None\`                                                        | LLM provider for dimension auto-detection (defaults to "openai" if not specified) |
| `progress_callback` | \`Callable\[\[int, int | None\], None\]                                                | None\`                                                                            |

Raises:

| Type                | Description                                                   |
| ------------------- | ------------------------------------------------------------- |
| `FileNotFoundError` | If the source path doesn't exist.                             |
| `RuntimeError`      | If no supported files are found in source.                    |
| `ConfigError`       | If configured fields don't exist in source (structured mode). |

Source code in `src/holodeck/tools/vectorstore_tool.py`

```
async def initialize(
    self,
    force_ingest: bool = False,
    provider_type: str | None = None,
    progress_callback: Callable[[int, int | None], None] | None = None,
) -> None:
    """Initialize tool and ingest source files.

    Discovers files from the configured source, processes them into chunks,
    generates embeddings, and stores them in the vector database.
    Source path is resolved relative to base_dir if set.

    For structured data mode (when vector_field is configured), loads
    structured data from CSV/JSON/JSONL files with field mapping.

    Args:
        force_ingest: If True, re-ingest all files regardless of modification time.
        provider_type: LLM provider for dimension auto-detection
            (defaults to "openai" if not specified)
        progress_callback: Optional callback invoked after each file is
            processed (or skipped). Called as ``callback(current, total)``
            where *current* is the 1-based file index and *total* is the
            total number of discovered files.

    Raises:
        FileNotFoundError: If the source path doesn't exist.
        RuntimeError: If no supported files are found in source.
        ConfigError: If configured fields don't exist in source (structured mode).
    """
    # Default to openai if not specified
    if provider_type is None:
        provider_type = "openai"
        logger.debug(
            f"Defaulting to '{provider_type}' for dimension auto-detection"
        )

    # Branch to structured mode if vector_field is configured
    if self._is_structured_mode():
        await self._initialize_structured(provider_type)
        return

    source_path = self._resolve_source_path()

    # Validate source exists (T035)
    if not source_path.exists():
        # Get effective base_dir for error message
        from holodeck.config.context import agent_base_dir

        effective_base_dir = self._base_dir or agent_base_dir.get()
        raise FileNotFoundError(
            f"Source path does not exist: {source_path} "
            f"(configured source: {self.config.source}, "
            f"base_dir: {effective_base_dir})"
        )

    # Set up collection instance with provider type before processing files
    self._setup_collection(provider_type)

    # Discover files
    files = self._discover_files()

    if not files:
        logger.warning(
            f"No supported files found in source: {self.config.source}. "
            f"Supported extensions: {SUPPORTED_EXTENSIONS}"
        )
        # Still mark as initialized even with no files
        self.is_initialized = True
        self.document_count = 0
        self.last_ingest_time = datetime.now()
        return

    logger.info(f"Discovered {len(files)} files for ingestion")

    # Process each file with mtime checking
    total_chunks = 0
    skipped_files = 0
    processed_count = 0
    total_files = len(files)

    for file_path in files:
        # Check if file needs re-ingestion (unless force_ingest)
        if not force_ingest:
            needs_reingest = await self._needs_reingest(file_path)
            if not needs_reingest:
                logger.debug(f"Skipping unchanged file: {file_path}")
                skipped_files += 1
                processed_count += 1
                if progress_callback is not None:
                    progress_callback(processed_count, total_files)
                continue
        else:
            # Force ingest: delete existing records first
            await self._delete_file_records(file_path)

        source_file = await self._process_file(file_path)
        if source_file is None:
            processed_count += 1
            if progress_callback is not None:
                progress_callback(processed_count, total_files)
            continue

        # Generate embeddings
        embeddings = await self._embed_chunks(source_file.chunks)

        # Store chunks
        chunks_stored = await self._store_chunks(source_file, embeddings)
        total_chunks += chunks_stored

        processed_count += 1
        if progress_callback is not None:
            progress_callback(processed_count, total_files)

    self.document_count = total_chunks
    self.is_initialized = True
    self.last_ingest_time = datetime.now()

    logger.info(
        f"VectorStoreTool initialized: {len(files)} files "
        f"({skipped_files} skipped, up-to-date), {total_chunks} chunks indexed"
    )
```

### `search(query)`

Execute semantic search and return formatted results.

Parameters:

| Name    | Type  | Description                    | Default    |
| ------- | ----- | ------------------------------ | ---------- |
| `query` | `str` | Natural language search query. | *required* |

Returns:

| Type  | Description                                                        |
| ----- | ------------------------------------------------------------------ |
| `str` | Formatted string with search results including scores and sources. |

Raises:

| Type           | Description              |
| -------------- | ------------------------ |
| `RuntimeError` | If tool not initialized. |
| `ValueError`   | If query is empty.       |

Source code in `src/holodeck/tools/vectorstore_tool.py`

```
async def search(self, query: str) -> str:
    """Execute semantic search and return formatted results.

    Args:
        query: Natural language search query.

    Returns:
        Formatted string with search results including scores and sources.

    Raises:
        RuntimeError: If tool not initialized.
        ValueError: If query is empty.
    """
    # Validation
    if not self.is_initialized:
        raise RuntimeError(
            "VectorStoreTool must be initialized before search. "
            "Call initialize() first."
        )

    if not query or not query.strip():
        raise ValueError("Search query cannot be empty")

    # Generate query embedding
    query_embeddings = await self._embed_chunks([query])
    query_embedding = query_embeddings[0]

    # Branch to structured search if in structured mode
    if self._is_structured_mode():
        structured_results = await self._search_structured(query_embedding)

        # Apply min_similarity_score filter
        if self.config.min_similarity_score is not None:
            structured_results = [
                r
                for r in structured_results
                if r.score >= self.config.min_similarity_score
            ]

        # Apply top_k limit
        structured_results = structured_results[: self.config.top_k]

        return self._format_structured_results(structured_results, query)

    # Unstructured mode: search document chunks
    doc_results = await self._search_documents(query_embedding)

    # Apply min_similarity_score filter
    if self.config.min_similarity_score is not None:
        doc_results = [
            r for r in doc_results if r.score >= self.config.min_similarity_score
        ]

    # Apply top_k limit (T037)
    doc_results = doc_results[: self.config.top_k]

    # Format results (T034)
    return self._format_results(doc_results, query)
```

### `set_embedding_service(service)`

Set the embedding service for generating embeddings.

This method allows AgentFactory to inject a Semantic Kernel TextEmbedding service for generating real embeddings instead of placeholder zeros.

Parameters:

| Name      | Type  | Description                                                                                 | Default    |
| --------- | ----- | ------------------------------------------------------------------------------------------- | ---------- |
| `service` | `Any` | Semantic Kernel TextEmbedding service instance (OpenAITextEmbedding or AzureTextEmbedding). | *required* |

Source code in `src/holodeck/tools/base_tool.py`

```
def set_embedding_service(self, service: Any) -> None:
    """Set the embedding service for generating embeddings.

    This method allows AgentFactory to inject a Semantic Kernel TextEmbedding
    service for generating real embeddings instead of placeholder zeros.

    Args:
        service: Semantic Kernel TextEmbedding service instance
            (OpenAITextEmbedding or AzureTextEmbedding).
    """
    self._embedding_service = service
    tool_name = getattr(getattr(self, "config", None), "name", "unknown")
    logger.debug(f"Embedding service set for tool: {tool_name}")
```

______________________________________________________________________

## `holodeck.tools.hierarchical_document_tool`

Provides intelligent document search that understands document structure, extracts definitions, and generates optimized context for LLM consumption. Supports semantic, keyword (BM25), and hybrid search modes with configurable weights.

### `HierarchicalDocumentTool`

## `HierarchicalDocumentTool(config, base_dir=None, execution_config=None)`

Bases: `EmbeddingServiceMixin`, `DatabaseConfigMixin`

Semantic Kernel tool for hierarchical document retrieval.

This tool provides intelligent document search that understands document structure, extracts definitions, and generates optimized context for LLM consumption.

Inherits from

EmbeddingServiceMixin: Provides set_embedding_service() method DatabaseConfigMixin: Provides database config resolution and collection creation

Attributes:

| Name     | Type | Description                                             |
| -------- | ---- | ------------------------------------------------------- |
| `config` |      | Tool configuration from HierarchicalDocumentToolConfig. |
| `chunks` |      | Indexed document chunks.                                |

Example

> > > from holodeck.models.tool import HierarchicalDocumentToolConfig config = HierarchicalDocumentToolConfig( ... name="doc_search", ... description="Search policy documents", ... source="./docs/policy.md" ... ) tool = HierarchicalDocumentTool(config) await tool.initialize() results = await tool.search("What are the reporting requirements?")

Initialize the hierarchical document tool.

Parameters:

| Name               | Type                             | Description                                             | Default                                                                                                                                                                                                                                                                                                                                                   |
| ------------------ | -------------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `config`           | `HierarchicalDocumentToolConfig` | Tool configuration from HierarchicalDocumentToolConfig. | *required*                                                                                                                                                                                                                                                                                                                                                |
| `base_dir`         | \`str                            | None\`                                                  | Optional base directory for resolving relative source paths. If None, source paths are resolved relative to current working directory or agent_base_dir context variable.                                                                                                                                                                                 |
| `execution_config` | \`ExecutionConfig                | None\`                                                  | Execution configuration for file processing timeouts and caching. When provided, the lazy FileProcessor used by \_convert_to_markdown is built via FileProcessor.from_execution_config so that execution.file_timeout (and download_timeout / cache_dir) from agent.yaml are honored. When None, FileProcessor's hardcoded defaults (30s timeouts) apply. |

Source code in `src/holodeck/tools/hierarchical_document_tool.py`

```
def __init__(
    self,
    config: HierarchicalDocumentToolConfig,
    base_dir: str | None = None,
    execution_config: ExecutionConfig | None = None,
) -> None:
    """Initialize the hierarchical document tool.

    Args:
        config: Tool configuration from HierarchicalDocumentToolConfig.
        base_dir: Optional base directory for resolving relative source paths.
            If None, source paths are resolved relative to current working
            directory or agent_base_dir context variable.
        execution_config: Execution configuration for file processing
            timeouts and caching. When provided, the lazy FileProcessor
            used by ``_convert_to_markdown`` is built via
            ``FileProcessor.from_execution_config`` so that
            ``execution.file_timeout`` (and download_timeout / cache_dir)
            from agent.yaml are honored. When None, FileProcessor's
            hardcoded defaults (30s timeouts) apply.
    """
    self.config = config
    self._base_dir = base_dir
    self._execution_config = execution_config
    self._chunker: StructuredChunker | None = None
    self._searcher: Any = None  # HybridSearcher (future)
    self._context_generator: ContextGenerator | None = None
    self._glossary: dict[str, DefinitionEntry] | None = None
    self._chunks: list[DocumentChunk] = []
    self._initialized = False

    # Service injection attributes
    self._embedding_service: Any = None
    self._chat_service: Any = None
    self._collection: Any = None
    self._provider: str = "in-memory"
    self._embedding_dimensions: int | None = None
    self._qdrant_indexes_ensured: bool = False

    # Lazily constructed FileProcessor (see _get_file_processor)
    self._file_processor: FileProcessor | None = None

    # Hybrid search executor (initialized during ingestion)
    self._hybrid_executor: HybridSearchExecutor | None = None

    # Source context for stable record keys (remote sources)
    self._source_root: Path | None = None
    self._is_remote: bool = False
```

### `initialize(force_ingest=True, provider_type=None, progress_callback=None)`

Initialize the tool by processing all configured documents.

This method should be called before any search operations. It loads documents, chunks them, extracts definitions, and indexes content for search.

Uses mtime-based incremental ingestion to skip unchanged files. Files are only re-ingested if their modification time is newer than the stored record's mtime.

Parameters:

| Name                | Type                   | Description                                                                                                         | Default                                                                            |
| ------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `force_ingest`      | `bool`                 | If True, re-ingest all files regardless of modification time. Existing records will be deleted before re-ingestion. | `True`                                                                             |
| `provider_type`     | \`str                  | None\`                                                                                                              | LLM provider for dimension auto-detection (defaults to "openai" if not specified). |
| `progress_callback` | \`Callable\[\[int, int | None\], None\]                                                                                                      | None\`                                                                             |

Raises:

| Type                | Description                      |
| ------------------- | -------------------------------- |
| `FileNotFoundError` | If a document file is not found. |

Source code in `src/holodeck/tools/hierarchical_document_tool.py`

```
async def initialize(
    self,
    force_ingest: bool = True,
    provider_type: str | None = None,
    progress_callback: Callable[[int, int | None], None] | None = None,
) -> None:
    """Initialize the tool by processing all configured documents.

    This method should be called before any search operations.
    It loads documents, chunks them, extracts definitions, and
    indexes content for search.

    Uses mtime-based incremental ingestion to skip unchanged files.
    Files are only re-ingested if their modification time is newer
    than the stored record's mtime.

    Args:
        force_ingest: If True, re-ingest all files regardless of
            modification time. Existing records will be deleted
            before re-ingestion.
        provider_type: LLM provider for dimension auto-detection
            (defaults to "openai" if not specified).
        progress_callback: Optional callback invoked after each file is
            processed (or skipped). Called as ``callback(current, total)``
            where *current* is the 1-based file index and *total* is the
            total number of discovered files.

    Raises:
        FileNotFoundError: If a document file is not found.
    """
    # Default to openai if not specified
    if provider_type is None:
        provider_type = "openai"
        logger.debug(
            f"Defaulting to '{provider_type}' for dimension auto-detection"
        )

    # Set up collection with provider type for dimension resolution
    self._setup_collection(provider_type)

    # Ingest all documents (with incremental check)
    skipped_files = await self._ingest_documents(
        force_ingest=force_ingest, progress_callback=progress_callback
    )

    # Always set up the hybrid executor — search routes through it.
    self._setup_hybrid_executor()

    # Decide whether to reload the corpus and rebuild the in-process
    # keyword index. Only the in-process BM25 fallback actually needs
    # this: it's ephemeral and dies with the process. For native-hybrid
    # providers (qdrant), keyword retrieval happens inside the store —
    # the in-process index is never read, so loading every chunk into
    # Python on every restart is pure waste. For in-memory providers,
    # the collection itself is reseeded by _ingest_documents (nothing
    # is "skipped"), so there's no reload to do.
    from holodeck.lib.keyword_search import NATIVE_HYBRID_PROVIDERS

    is_native_hybrid = self._provider in NATIVE_HYBRID_PROVIDERS
    needs_corpus_reload = (
        skipped_files > 0 and self._provider != "in-memory" and not is_native_hybrid
    )

    if needs_corpus_reload:
        stored_chunks = await self._load_chunks_from_store()
        if stored_chunks:
            self._chunks = stored_chunks
            await self._build_hybrid_indices()
    elif is_native_hybrid:
        logger.debug(
            "Skipped startup corpus reload for native-hybrid provider "
            f"'{self._provider}'; lazy chunk fetch on miss."
        )

    self._initialized = True
    logger.info(
        f"HierarchicalDocumentTool initialized: "
        f"{len(self._chunks)} chunks from {self.config.source}"
    )
```

### `search(query, top_k=None)`

Search documents for content relevant to query.

Parameters:

| Name    | Type  | Description          | Default                    |
| ------- | ----- | -------------------- | -------------------------- |
| `query` | `str` | Search query string. | *required*                 |
| `top_k` | \`int | None\`               | Override configured top_k. |

Returns:

| Type                 | Description                   |
| -------------------- | ----------------------------- |
| `list[SearchResult]` | List of SearchResult objects. |

Raises:

| Type           | Description                 |
| -------------- | --------------------------- |
| `RuntimeError` | If tool is not initialized. |
| `ValueError`   | If query is empty.          |

Source code in `src/holodeck/tools/hierarchical_document_tool.py`

```
async def search(
    self,
    query: str,
    top_k: int | None = None,
) -> list[SearchResult]:
    """Search documents for content relevant to query.

    Args:
        query: Search query string.
        top_k: Override configured top_k.

    Returns:
        List of SearchResult objects.

    Raises:
        RuntimeError: If tool is not initialized.
        ValueError: If query is empty.
    """
    if not self._initialized:
        raise RuntimeError(
            "HierarchicalDocumentTool must be initialized before search. "
            "Call initialize() first."
        )

    if not query or not query.strip():
        raise ValueError("Search query cannot be empty")

    effective_top_k = top_k or self.config.top_k

    # Generate query embedding (needed for semantic and hybrid modes)
    query_embedding = await self._embed_query(query)

    # Route to appropriate search mode
    if self.config.search_mode == SearchMode.KEYWORD:
        results = await self._keyword_search(query, effective_top_k)
    elif self.config.search_mode == SearchMode.HYBRID:
        results = await self._hybrid_search(query, query_embedding, effective_top_k)
    else:
        # SEMANTIC mode (default)
        results = await self._semantic_search(query_embedding, effective_top_k)

    # Filter by min_score if configured
    if self.config.min_score is not None:
        results = [r for r in results if r.fused_score >= self.config.min_score]

    return results
```

### `get_context(query, max_tokens=None)`

Get LLM-ready context for a query.

This is a convenience method that searches and formats results into a single context string suitable for LLM prompts.

Parameters:

| Name         | Type  | Description               | Default                                        |
| ------------ | ----- | ------------------------- | ---------------------------------------------- |
| `query`      | `str` | Query to get context for. | *required*                                     |
| `max_tokens` | \`int | None\`                    | Maximum tokens for context (currently unused). |

Returns:

| Type  | Description               |
| ----- | ------------------------- |
| `str` | Formatted context string. |

Raises:

| Type           | Description                 |
| -------------- | --------------------------- |
| `RuntimeError` | If tool is not initialized. |

Source code in `src/holodeck/tools/hierarchical_document_tool.py`

```
async def get_context(self, query: str, max_tokens: int | None = None) -> str:
    """Get LLM-ready context for a query.

    This is a convenience method that searches and formats results
    into a single context string suitable for LLM prompts.

    Args:
        query: Query to get context for.
        max_tokens: Maximum tokens for context (currently unused).

    Returns:
        Formatted context string.

    Raises:
        RuntimeError: If tool is not initialized.
    """
    results = await self.search(query)

    if not results:
        return f"No relevant context found for: {query}"

    lines = [f"Context for query: {query}", ""]
    for i, result in enumerate(results, 1):
        lines.append(f"[{i}] {result.format()}")
        lines.append("")

    return "\n".join(lines).rstrip()
```

### `get_definition(term)`

Look up a term's definition.

Parameters:

| Name   | Type  | Description      | Default    |
| ------ | ----- | ---------------- | ---------- |
| `term` | `str` | Term to look up. | *required* |

Returns:

| Type             | Description |
| ---------------- | ----------- |
| \`dict[str, str] | None\`      |

Source code in `src/holodeck/tools/hierarchical_document_tool.py`

```
def get_definition(self, term: str) -> dict[str, str] | None:
    """Look up a term's definition.

    Args:
        term: Term to look up.

    Returns:
        Dictionary with 'term' and 'definition' keys, or None.
    """
    if self._glossary is None:
        return None

    normalized = term.lower().replace(" ", "_")
    entry = self._glossary.get(normalized)
    if entry:
        return {"term": entry.term, "definition": entry.definition_text}
    return None
```

### `set_embedding_service(service)`

Set the embedding service for generating embeddings.

This method allows AgentFactory to inject a Semantic Kernel TextEmbedding service for generating real embeddings instead of placeholder zeros.

Parameters:

| Name      | Type  | Description                                                                                 | Default    |
| --------- | ----- | ------------------------------------------------------------------------------------------- | ---------- |
| `service` | `Any` | Semantic Kernel TextEmbedding service instance (OpenAITextEmbedding or AzureTextEmbedding). | *required* |

Source code in `src/holodeck/tools/base_tool.py`

```
def set_embedding_service(self, service: Any) -> None:
    """Set the embedding service for generating embeddings.

    This method allows AgentFactory to inject a Semantic Kernel TextEmbedding
    service for generating real embeddings instead of placeholder zeros.

    Args:
        service: Semantic Kernel TextEmbedding service instance
            (OpenAITextEmbedding or AzureTextEmbedding).
    """
    self._embedding_service = service
    tool_name = getattr(getattr(self, "config", None), "name", "unknown")
    logger.debug(f"Embedding service set for tool: {tool_name}")
```

### `set_context_generator(generator)`

Set the context generator for contextual embeddings.

Accepts any ContextGenerator protocol implementation (LLMContextGenerator, ClaudeSDKContextGenerator, etc.).

Parameters:

| Name        | Type  | Description                                 | Default    |
| ----------- | ----- | ------------------------------------------- | ---------- |
| `generator` | `Any` | A ContextGenerator protocol implementation. | *required* |

Source code in `src/holodeck/tools/hierarchical_document_tool.py`

```
def set_context_generator(self, generator: Any) -> None:
    """Set the context generator for contextual embeddings.

    Accepts any ContextGenerator protocol implementation
    (LLMContextGenerator, ClaudeSDKContextGenerator, etc.).

    Args:
        generator: A ContextGenerator protocol implementation.
    """
    self._context_generator = generator
```

### `set_chat_service(service)`

Set the chat service for LLM context generation.

This enables contextual embeddings via the LLMContextGenerator. When contextual_embeddings is enabled in config, this also creates the LLMContextGenerator instance.

Parameters:

| Name      | Type  | Description                                      | Default    |
| --------- | ----- | ------------------------------------------------ | ---------- |
| `service` | `Any` | Semantic Kernel ChatCompletion service instance. | *required* |

Source code in `src/holodeck/tools/hierarchical_document_tool.py`

```
def set_chat_service(self, service: Any) -> None:
    """Set the chat service for LLM context generation.

    This enables contextual embeddings via the LLMContextGenerator.
    When contextual_embeddings is enabled in config, this also creates
    the LLMContextGenerator instance.

    Args:
        service: Semantic Kernel ChatCompletion service instance.
    """
    self._chat_service = service

    # Create LLMContextGenerator if contextual embeddings are enabled
    if self.config.contextual_embeddings and service is not None:
        from holodeck.lib.llm_context_generator import LLMContextGenerator

        self._context_generator = LLMContextGenerator(
            chat_service=service,
            max_context_tokens=self.config.context_max_tokens,
            concurrency=self.config.context_concurrency,
        )
        logger.info(
            f"LLMContextGenerator created for {self.config.name} "
            f"(max_tokens={self.config.context_max_tokens}, "
            f"concurrency={self.config.context_concurrency})"
        )
    else:
        logger.debug("Chat service set for HierarchicalDocumentTool")
```

### `to_semantic_kernel_function()`

Convert this tool to a Semantic Kernel function.

Returns:

| Type  | Description                       |
| ----- | --------------------------------- |
| `Any` | Semantic Kernel function wrapper. |

Source code in `src/holodeck/tools/hierarchical_document_tool.py`

```
def to_semantic_kernel_function(self) -> Any:
    """Convert this tool to a Semantic Kernel function.

    Returns:
        Semantic Kernel function wrapper.
    """

    # This would require Semantic Kernel decorators
    # For now, return a simple wrapper
    async def sk_search_function(query: str) -> str:
        results = await self.search(query)
        if not results:
            return "No results found."
        return "\n\n".join(r.format() for r in results)

    return sk_search_function
```

______________________________________________________________________

## `holodeck.tools.mcp`

MCP (Model Context Protocol) tool module for integrating external MCP servers. Provides a factory for creating Semantic Kernel MCP plugins, utility functions, and a dedicated error hierarchy.

### `holodeck.tools.mcp.factory`

#### `create_mcp_plugin`

## `create_mcp_plugin(config)`

Create an SK MCP plugin based on transport type.

This factory function creates the appropriate Semantic Kernel MCP plugin based on the transport type specified in the configuration. Each transport type maps to a specific SK plugin:

Transport mapping:

- stdio -> MCPStdioPlugin
- sse -> MCPSsePlugin
- websocket -> MCPWebsocketPlugin
- http -> MCPStreamableHttpPlugin

Parameters:

| Name     | Type      | Description                            | Default    |
| -------- | --------- | -------------------------------------- | ---------- |
| `config` | `MCPTool` | MCP tool configuration from agent.yaml | *required* |

Returns:

| Type             | Description                                                           |
| ---------------- | --------------------------------------------------------------------- |
| `MCPStdioPlugin` | MCPStdioPlugin instance. Other transport types (SSE, WebSocket, HTTP) |
| `MCPStdioPlugin` | will return their respective plugin types when implemented.           |

Raises:

| Type             | Description                                               |
| ---------------- | --------------------------------------------------------- |
| `MCPConfigError` | If transport type is not supported or not yet implemented |

Example

> > > config = MCPTool( ... name="filesystem", ... description="File operations", ... command=CommandType.NPX, ... args=["-y", "@modelcontextprotocol/server-filesystem"], ... ) plugin = create_mcp_plugin(config)
> > >
> > > ### plugin is an MCPStdioPlugin instance

Source code in `src/holodeck/tools/mcp/factory.py`

```
def create_mcp_plugin(config: MCPTool) -> "MCPStdioPlugin":
    """Create an SK MCP plugin based on transport type.

    This factory function creates the appropriate Semantic Kernel MCP plugin
    based on the transport type specified in the configuration. Each transport
    type maps to a specific SK plugin:

    Transport mapping:
    - stdio -> MCPStdioPlugin
    - sse -> MCPSsePlugin
    - websocket -> MCPWebsocketPlugin
    - http -> MCPStreamableHttpPlugin

    Args:
        config: MCP tool configuration from agent.yaml

    Returns:
        MCPStdioPlugin instance. Other transport types (SSE, WebSocket, HTTP)
        will return their respective plugin types when implemented.

    Raises:
        MCPConfigError: If transport type is not supported or not yet implemented

    Example:
        >>> config = MCPTool(
        ...     name="filesystem",
        ...     description="File operations",
        ...     command=CommandType.NPX,
        ...     args=["-y", "@modelcontextprotocol/server-filesystem"],
        ... )
        >>> plugin = create_mcp_plugin(config)
        >>> # plugin is an MCPStdioPlugin instance
    """
    # Resolve environment variables (env_file + explicit env + config passthrough)
    resolved_env = _resolve_env_vars(config)

    if config.transport == TransportType.STDIO:
        # Import SK plugin lazily to avoid hard dependency
        try:
            from semantic_kernel.connectors.mcp import MCPStdioPlugin
        except ImportError as e:
            raise MCPConfigError(
                field="transport",
                message=(
                    "Semantic Kernel MCP support not installed. "
                    "Install with: pip install semantic-kernel[mcp]"
                ),
            ) from e

        return MCPStdioPlugin(
            name=config.name,
            command=config.command.value if config.command else "npx",
            args=(config.args or []),
            env=resolved_env if resolved_env else None,
            encoding=config.encoding or DEFAULT_STDIO_ENCODING,
        )

    elif config.transport == TransportType.SSE:
        # TODO: Implement in T022 (Phase 7 - User Story 6)
        # from semantic_kernel.connectors.mcp import MCPSsePlugin
        # return MCPSsePlugin(
        #     name=config.name,
        #     url=config.url,
        #     headers=config.headers,
        #     timeout=config.timeout,
        #     sse_read_timeout=config.sse_read_timeout,
        # )
        raise MCPConfigError(
            field="transport",
            message="SSE transport not yet implemented. Coming in Phase 7.",
        )

    elif config.transport == TransportType.WEBSOCKET:
        # TODO: Implement in T025 (Phase 8 - User Story 7)
        # from semantic_kernel.connectors.mcp import MCPWebsocketPlugin
        # return MCPWebsocketPlugin(
        #     name=config.name,
        #     url=config.url,
        # )
        raise MCPConfigError(
            field="transport",
            message="WebSocket transport not yet implemented. Coming in Phase 8.",
        )

    elif config.transport == TransportType.HTTP:
        # TODO: Implement in T027 (Phase 9 - User Story 8)
        # from semantic_kernel.connectors.mcp import MCPStreamableHttpPlugin
        # return MCPStreamableHttpPlugin(
        #     name=config.name,
        #     url=config.url,
        #     headers=config.headers,
        #     timeout=config.timeout,
        #     sse_read_timeout=config.sse_read_timeout,
        #     terminate_on_close=config.terminate_on_close,
        # )
        raise MCPConfigError(
            field="transport",
            message="HTTP transport not yet implemented. Coming in Phase 9.",
        )

    else:
        raise MCPConfigError(
            field="transport",
            message=f"Unknown transport type: {config.transport}",
        )
```

### `holodeck.tools.mcp.utils`

#### `normalize_tool_name`

## `normalize_tool_name(name)`

Normalize tool name by replacing invalid characters with '-'.

Per Semantic Kernel pattern, tool names must be valid identifiers. This method replaces any character that is not alphanumeric or underscore with a hyphen.

Parameters:

| Name   | Type  | Description                        | Default    |
| ------ | ----- | ---------------------------------- | ---------- |
| `name` | `str` | Original tool name from MCP server | *required* |

Returns:

| Type  | Description                                |
| ----- | ------------------------------------------ |
| `str` | Normalized name safe for use as identifier |

Example

> > > normalize_tool_name("read.file") 'read-file' normalize_tool_name("read/write") 'read-write' normalize_tool_name("read_file_v2") 'read_file_v2'

Source code in `src/holodeck/tools/mcp/utils.py`

```
def normalize_tool_name(name: str) -> str:
    """Normalize tool name by replacing invalid characters with '-'.

    Per Semantic Kernel pattern, tool names must be valid identifiers.
    This method replaces any character that is not alphanumeric or
    underscore with a hyphen.

    Args:
        name: Original tool name from MCP server

    Returns:
        Normalized name safe for use as identifier

    Example:
        >>> normalize_tool_name("read.file")
        'read-file'
        >>> normalize_tool_name("read/write")
        'read-write'
        >>> normalize_tool_name("read_file_v2")
        'read_file_v2'
    """
    return re.sub(r"[^a-zA-Z0-9_]", "-", name)
```

### `holodeck.tools.mcp.errors`

MCP-specific error hierarchy extending the base HoloDeck error types.

```
HoloDeckError
├── ConfigError
│   └── MCPConfigError          # Invalid MCP configuration
└── MCPError                    # Base MCP runtime error
    ├── MCPConnectionError      # Failed to connect to server
    │   └── MCPTimeoutError     # Connection/request timeout
    ├── MCPProtocolError        # Protocol-level error from server
    └── MCPToolNotFoundError    # Tool not found on server
```

#### `MCPConfigError`

## `MCPConfigError(field, message)`

Bases: `ConfigError`

MCP configuration error (invalid transport, missing fields, etc.).

Initialize MCP configuration error.

Parameters:

| Name      | Type  | Description                                   | Default    |
| --------- | ----- | --------------------------------------------- | ---------- |
| `field`   | `str` | The configuration field that caused the error | *required* |
| `message` | `str` | Descriptive error message                     | *required* |

Source code in `src/holodeck/tools/mcp/errors.py`

```
def __init__(self, field: str, message: str) -> None:
    """Initialize MCP configuration error.

    Args:
        field: The configuration field that caused the error
        message: Descriptive error message
    """
    super().__init__(field, message)
```

#### `MCPError`

## `MCPError(message, server=None)`

Bases: `HoloDeckError`

Base exception for MCP runtime errors.

Initialize MCP error.

Parameters:

| Name      | Type  | Description               | Default                          |
| --------- | ----- | ------------------------- | -------------------------------- |
| `message` | `str` | Descriptive error message | *required*                       |
| `server`  | \`str | None\`                    | MCP server identifier (optional) |

Source code in `src/holodeck/tools/mcp/errors.py`

```
def __init__(self, message: str, server: str | None = None) -> None:
    """Initialize MCP error.

    Args:
        message: Descriptive error message
        server: MCP server identifier (optional)
    """
    self.server = server
    super().__init__(message)
```

#### `MCPConnectionError`

## `MCPConnectionError(message, server=None, command=None)`

Bases: `MCPError`

Failed to connect to MCP server.

Initialize MCP connection error.

Parameters:

| Name      | Type  | Description               | Default                               |
| --------- | ----- | ------------------------- | ------------------------------------- |
| `message` | `str` | Descriptive error message | *required*                            |
| `server`  | \`str | None\`                    | MCP server identifier (optional)      |
| `command` | \`str | None\`                    | Command that was attempted (optional) |

Source code in `src/holodeck/tools/mcp/errors.py`

```
def __init__(
    self, message: str, server: str | None = None, command: str | None = None
) -> None:
    """Initialize MCP connection error.

    Args:
        message: Descriptive error message
        server: MCP server identifier (optional)
        command: Command that was attempted (optional)
    """
    self.command = command
    super().__init__(message, server)
```

#### `MCPTimeoutError`

## `MCPTimeoutError(message, server=None, timeout=None)`

Bases: `MCPConnectionError`

MCP server connection or request timeout.

Initialize MCP timeout error.

Parameters:

| Name      | Type    | Description               | Default                                    |
| --------- | ------- | ------------------------- | ------------------------------------------ |
| `message` | `str`   | Descriptive error message | *required*                                 |
| `server`  | \`str   | None\`                    | MCP server identifier (optional)           |
| `timeout` | \`float | None\`                    | Timeout value that was exceeded (optional) |

Source code in `src/holodeck/tools/mcp/errors.py`

```
def __init__(
    self,
    message: str,
    server: str | None = None,
    timeout: float | None = None,
) -> None:
    """Initialize MCP timeout error.

    Args:
        message: Descriptive error message
        server: MCP server identifier (optional)
        timeout: Timeout value that was exceeded (optional)
    """
    self.timeout = timeout
    super().__init__(message, server)
```

#### `MCPProtocolError`

## `MCPProtocolError(message, server=None, error_code=None)`

Bases: `MCPError`

MCP protocol-level error returned by server.

Initialize MCP protocol error.

Parameters:

| Name         | Type  | Description               | Default                            |
| ------------ | ----- | ------------------------- | ---------------------------------- |
| `message`    | `str` | Descriptive error message | *required*                         |
| `server`     | \`str | None\`                    | MCP server identifier (optional)   |
| `error_code` | \`int | None\`                    | MCP protocol error code (optional) |

Source code in `src/holodeck/tools/mcp/errors.py`

```
def __init__(
    self,
    message: str,
    server: str | None = None,
    error_code: int | None = None,
) -> None:
    """Initialize MCP protocol error.

    Args:
        message: Descriptive error message
        server: MCP server identifier (optional)
        error_code: MCP protocol error code (optional)
    """
    self.error_code = error_code
    super().__init__(message, server)
```

#### `MCPToolNotFoundError`

## `MCPToolNotFoundError(tool_name, server=None)`

Bases: `MCPError`

Tool not found on MCP server.

Initialize MCP tool not found error.

Parameters:

| Name        | Type  | Description                         | Default                          |
| ----------- | ----- | ----------------------------------- | -------------------------------- |
| `tool_name` | `str` | Name of the tool that was not found | *required*                       |
| `server`    | \`str | None\`                              | MCP server identifier (optional) |

Source code in `src/holodeck/tools/mcp/errors.py`

```
def __init__(self, tool_name: str, server: str | None = None) -> None:
    """Initialize MCP tool not found error.

    Args:
        tool_name: Name of the tool that was not found
        server: MCP server identifier (optional)
    """
    self.tool_name = tool_name
    super().__init__(f"Tool '{tool_name}' not found on server", server)
```
